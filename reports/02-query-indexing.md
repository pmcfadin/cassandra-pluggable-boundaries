# 02 — Pluggable boundaries: query language, execution, indexing, types, functions

Scope: CQL front end, prepared-statement pipeline, coordinator read/write path, query planning and index selection, secondary indexes (legacy, SASI, SAI, vector), triggers, UDF/UDA, native functions, custom types, constraints (CEP-42), masking (CEP-20), virtual tables, guardrails, paging.

Source: `cassandra`, branch `codex/pluggable-consistency-prototype`. Paths below are relative to `src/java/org/apache/cassandra/` unless noted. Merge base with `trunk` is `0691eb3319`. The branch differs from trunk in this area only in the Accord/consistency prototype: `cql3/statements/{Transaction,Modification,Batch,Select}Statement.java`, `cql3/transactions/*`, and **`service/StorageProxy.java`, where it adds `TransactionDomainGuard.check(...)` calls in `cas`, `mutateWithTriggers`, `read`, and `truncateBlocking`** (`git diff trunk...HEAD`). These calls are branch-only, and StorageProxy line numbers below are about 5–13 lines higher than on trunk. Everything else cited also exists on trunk.

Prior work reused here, not redone: `query-optimizer/` (ASSESSMENT.md, research/cep-39-and-reusable-hooks.md, design/plan-and-statistics-contract.md, research/legal-alternatives-and-barriers.md). Its conclusions are cited as **[QO]**.

Legend: **F** = verified fact from source at the cited line. **I** = inference or unverified external claim.

---

## Summary table

| # | Boundary | Status | Entry point | Grade |
|---|---|---|---|---|
| 1 | Secondary index API (`Index`, `Index.Group`, `QueryPlan`) | stable-public (de facto) | `CREATE CUSTOM INDEX … USING 'fqcn'`, `expr(idx,'…')`, `default_secondary_index` | **C** |
| 2 | SAI (CEP-7) as an internal platform (on-disk `Version`/`OnDiskFormat`) | internal-seam | `USING 'sai'`, `-Dcassandra.sai.latest_version` | **C-** |
| 3 | Vector/ANN engine inside SAI | internal-seam (hard-wired to jvector) | `vector<float,n>` + SAI + `ORDER BY v ANN OF …` | **D** |
| 4 | SASI | stable-public but experimental / frozen | `USING 'SASIIndex'`, `sasi_indexes_enabled` | **D** (do not extend) |
| 5 | `QueryHandler` | stable-public (de facto) | `-Dcassandra.custom_query_handler_class` | **C-** |
| 6 | CQL parse/prepare pipeline (`CQLStatement`, `Raw`, Parser.g) | internal-seam | none (ANTLR grammar compiled in) | **E** |
| 7 | Triggers (`ITrigger`) | stable-public, legacy | `CREATE TRIGGER … USING 'fqcn'`, `triggers_dir`, `triggers_policy` | **C** |
| 8 | Java UDF/UDA | stable-public, sandbox at risk | `CREATE FUNCTION … LANGUAGE java`, `user_defined_functions_enabled` | **C** |
| 9 | Native/system function registry | internal-seam | none (hard-coded list) | **E** |
| 10 | Custom types (`AbstractType` by class name) | stable-public, legacy-ish | `'org.foo.MyType'` in CQL, `TypeParser` | **D** |
| 11 | Constraints SPI (CEP-42 `ConstraintProvider`) | partial (new) | `ServiceLoader<ConstraintProvider>`, `CHECK fn(...)` | **B-** |
| 12 | Data masking functions (CEP-20) | partial | `MASKED WITH fn(...)`; UDF masks | **B-** |
| 13 | Virtual tables (`VirtualTable`/`VirtualKeyspace`) | internal-seam (good API, no loader) | none — hard-coded in `CassandraDaemon` | **C+** as API / **F** as plugin point |
| 14 | Coordinator read/write path (`StorageProxy`) | internal-seam | none (static methods) | **F** |
| 15 | Query planning / restrictions / index selection | internal-seam | `WITH included_indexes/excluded_indexes` hints only | **D** |
| 16 | Result paging (`QueryPager`, `PagingState`) | internal-seam | none | **D** |
| 17 | Guardrails as a policy hook | partial | `-Dcassandra.custom_guardrails_config_provider_class`; `CustomGuardrail` + `validator_class_name` | **C** |
| 18 | `QueryEvents.Listener` (observer) | internal-seam (used by audit/FQL) | none (programmatic `registerListener`) | **C** |

Grade scale: A = versioned SPI with a narrow, internal-free interface, a loader, and a TCK; F = no seam (patch the fork).

---

## 1. Secondary index API — `org.apache.cassandra.index.Index`

- **Status:** stable-public by convention (it has existed since 3.0 and is used by external projects), with no compatibility annotation. **F:** the `Index` javadoc says only that implementations are instantiated by reflection. It makes no stability promise (`index/Index.java:70-160`).
- **Entry point (F):**
  - `CREATE CUSTOM INDEX … USING '<fqcn or alias>' WITH OPTIONS = {...}` (`antlr/Parser.g:1099-1108`).
  - Aliases `sai`, `storageattachedindex`, and `SASIIndex` are registered in `schema/IndexMetadata.java:75-81`.
  - The class is loaded via `FBUtilities.classForNameWithoutInitialization` (`schema/IndexMetadata.java:151`, `index/SecondaryIndexManager.java:925-928`).
  - It is constructed reflectively with the ctor `(ColumnFamilyStore, IndexMetadata)` (`SecondaryIndexManager.java:930-955`).
  - Options are validated reflectively through a static `validateOptions` (`IndexMetadata.java:166`).
  - Plain `CREATE INDEX` uses `default_secondary_index` / `default_secondary_index_enabled` (`config/Config.java:1012-1013`).
  - Free-form query syntax is `WHERE expr(index_name, <term>)` (`Parser.g:535-541`), which becomes `RowFilter.CustomExpression`. Only one is allowed per query, and it forces that index's group (`SecondaryIndexManager.java:1313-1324`).
- **Interfaces (F):**
  - `Index` (`index/Index.java:160`)
  - `Index.IndexBuildingSupport` (`:188`)
  - `Index.Indexer` with write-path callbacks `insertRow/updateRow/removeRow/partitionDelete/rangeTombstone` (`:614-687`)
  - `Index.Searcher` (`:725`)
  - `Index.Group`, with `indexerFor`, `queryPlanFor`, `getFlushObserver`, and `getComponents` (`:762-916`)
  - `Index.QueryPlan extends Comparable` (`:940-1081`, including `isTopK` `:1081` and `postProcessor` `:1043`)
  - `Index.Status` (`:1090`)
  - Registration goes through `IndexRegistry` / `SecondaryIndexManager.registerIndex(index, groupKey, supplier)` (`SecondaryIndexManager.java:1419`). The default is a singleton group (`Index.java:292`). SAI registers one shared group per table (`index/sai/StorageAttachedIndex.java:320`).
- **Selection (F):** `getBestIndexQueryPlanFor` asks every group for a plan. It ranks plans first by index hints, then by `QueryPlan` natural order, i.e. `getEstimatedResultRows` (`SecondaryIndexManager.java:1308-1360`). The code comment admits that "some index implementations may return fixed values … to prioritize themselves" (`:1347-1349`). In other words, selectivity is self-reported and not comparable across implementations **[QO]**.
- **Distributed contract (F):** the coordinator picks the plan. `ReadCommand` serializes only the *first index's* `IndexMetadata` (`db/ReadCommand.java:481`). The replica re-derives the plan with `indexGroup.queryPlanFor(rowFilter)` (`ReadCommand.java:1465-1470`). Group-level `queryPlanFor` must therefore be deterministic across nodes, but the API does not state this.
- **Known external implementations:**
  - Stratio `cassandra-lucene-index` (`expr()` syntax, Lucene per node). **I:** archived. Later community forks (e.g., Instaclustr) targeted 3.11/4.0.
  - Elassandra `ElasticSecondaryIndex`. **I:** Elassandra was a *fork*, not a pure plugin, which suggests the API was insufficient.
  - DSE Search (Solr). **I:** DSE fork, closed source.
  - In-tree: `CassandraIndex` (legacy), SASI, SAI, and the test `StubIndex`/`StubIndexGroup` (`test/unit/org/apache/cassandra/index/`).
- **Leakage (F):** `Index.java` imports 32 internal classes. These include `ColumnFamilyStore`, `SSTableReader`, `Descriptor`, `ILifecycleTransaction`, `Memtable`, `ReadExecutionController`, `UnfilteredPartitionIterator`, `PartitionUpdate`, `RowFilter`, `ReducingKeyIterator`, `CollatedViewIndexBuilder`, and the CQL layer's `cql3.restrictions.Restriction` and `QueryOptions` (through `getPostQueryOrdering(Restriction, QueryOptions)`, `Index.java:493`). An implementer depends on the storage engine, the SSTable lifecycle, and CQL restriction internals. Operators are a closed enum (`cql3/Operator.java`). A custom index cannot add an operator, so anything beyond `= < > IN CONTAINS LIKE ANN` must go through the single `expr()` slot.
- **Compatibility (F):** `git log trunk` shows 29 commits to `Index.java` since 2019. No `@since`/stability markers exist. `CustomIndexTest` + `StubIndex` are the only harness, and there is no TCK. SSTable-attached indexes (`isSSTableAttached`, `getFlushObserver`, `getComponents`) couple the index to the SSTable format and component naming.
- **Grade: C.** It is real and has been used externally, but it leaks storage and CQL internals, its selection protocol cannot be compared across implementations, and there is no compatibility promise.
- **Recommendation:**
  1. Mark `Index`/`Index.Group`/`QueryPlan` as `@Evolving` public API, with a deprecation window per major release.
  2. Split it into `IndexWriteSPI` (row events, keyed on a stable row view) and `IndexReadSPI` (a plan over a logical predicate IR, not `RowFilter`/`Restriction`).
  3. Replace `getEstimatedResultRows` with the interval-plus-provenance estimate proposed in [QO] `design/plan-and-statistics-contract.md`.
  4. Document the replica re-planning determinism requirement.
  5. Publish `StubIndex` as a test-jar TCK.

## 2. SAI (CEP-7) as an internal platform

- **Status:** internal-seam.
- **F:**
  - SAI is one `Index.Group` per table (`StorageAttachedIndexGroup implements Index.Group`, `index/sai/StorageAttachedIndexGroup.java:77`).
  - On-disk formats are abstracted by `OnDiskFormat` (`index/sai/disk/format/OnDiskFormat.java:55`). They are selected by `Version`, with a static `AA → V1OnDiskFormat` mapping (`Version.java:45`), a static `ALL` set (`:49`), and `LATEST` taken from `CassandraRelevantProperties.SAI_LATEST_VERSION` (`:56`).
  - Analyzers include only `NonTokenizingAnalyzer` (`index/sai/analyzer/`), with no class-name loading.
  - Planning lives in `index/sai/plan/{StorageAttachedIndexQueryPlan,QueryController,Operation,FilterTree,VectorTopKProcessor}.java`.
- **Leakage:** SAI deliberately splits `indexFilter` from `postIndexFilter` and exposes `KeyRangeIterator.getMaxKeys` and `QueryContext` counters. These are the best existing optimizer hooks **[QO cep-39-and-reusable-hooks.md table]**. However, none are reachable by a third party without forking SAI.
- **External:** **I:** DataStax's fork carries newer SAI versions, Lucene analyzers, and BM25. Those features have not reached trunk here (no Lucene analyzer classes exist under `index/sai/analyzer/`).
- **Grade: C-.** The internal layering is clean (`Version`/`OnDiskFormat`/analyzer), but it is closed: the format and analyzer lists are static.
- **Next steps:**
  - Make `Version` registration ServiceLoader-based, following the `SSTableFormat.Factory` precedent (`config/DatabaseDescriptor.java:2006`).
  - Expose an `Analyzer` SPI chosen by an index option.
  - Keep SAI the reference implementation of the read SPI in #1.

## 3. Vector / ANN engine

- **Status:** internal-seam, hard-wired.
- **F:**
  - `VectorType` (`db/marshal/VectorType.java:46`) and `vector_type_enabled`/`vector_dimensions` guardrails (`db/guardrails/Guardrails.java:516,525`).
  - The CQL `ORDER BY col ANN OF term` (`Parser.g:547`) becomes `Ordering.Ann` (`cql3/Ordering.java:85-180`).
  - Restriction checks reject ANN without an index (`cql3/restrictions/StatementRestrictions.java:87-93`).
  - Top-K queries bypass normal paging and forbid aggregation/SERIAL (`cql3/statements/SelectStatement.java:391-396,430,649`).
  - The engine is jvector, imported directly: `io.github.jbellis.jvector.*` appears in `index/sai/disk/v1/vector/DiskAnn.java:34-41` and `OnHeapGraph.java:60`. The index option `similarity_function` parses into jvector's `VectorSimilarityFunction` enum (`index/sai/disk/v1/IndexWriterConfig.java:47-48,154-156`), so a third-party type leaks into the schema option surface.
- **Coupling:** ANN is reachable only via SAI's `QueryPlan.isTopK()` and `postProcessor`. A competing engine (DiskANN variant, IVF-PQ, GPU, external service) must either fork SAI or ship a full custom `Index` and reimplement top-K/paging semantics itself.
- **Grade: D.**
- **Next steps:** see Proposed boundary P3.

## 4. SASI

- **F:**
  - Disabled by default (`sasi_indexes_enabled=false`, `config/Config.java:717`; enforced at `cql3/statements/schema/CreateIndexStatement.java:151-152`).
  - Flagged "experimental and … not recommended for production" (`index/sasi/SASIIndex.java:87`).
- **Grade: D.**
- **Recommendation:** keep it frozen. Its existence as a plugin-shaped `Index` is evidence that #1 can host full engines, but new work should target SAI or the #1 SPI.

## 5. `QueryHandler`

- **Status:** stable-public in practice, undocumented.
- **Entry (F):**
  - System property `cassandra.custom_query_handler_class` (`config/CassandraRelevantProperties.java:209`), read once in a `ClientState` static initializer.
  - Constructed via `FBUtilities.construct`. Any failure kills the JVM (`service/ClientState.java:122-141`).
  - It is used only by native-protocol `QueryMessage`/`ExecuteMessage`/`BatchMessage` (`transport/messages/QueryMessage.java:116`, `ExecuteMessage.java:137`, `BatchMessage.java:181`).
- **Interface (F):** `cql3/QueryHandler.java:32-60`, with `parse`, `process`, `prepare`, `getPrepared`, `processPrepared`, and `processBatch`. Every method traffics in internal `CQLStatement`, `BatchStatement`, `QueryState`, `QueryOptions`, `ResultMessage`, `Dispatcher.RequestTime`, and `MD5Digest`, plus the `QueryHandler.Prepared` POJO (`:60-90`).
- **Leakage (F):**
  - Internal callers bypass it and use `QueryProcessor.instance` directly, e.g. `auth/CassandraAuthorizer.java:450` and prepared-statement management in `service/StorageService.java:2831-2839`. A handler therefore sees only client traffic.
  - Only one handler is allowed, so handlers cannot be chained.
  - A handler that wants new syntax must still produce a `CQLStatement`, which means using the internal statement classes or reimplementing them.
- **External:**
  - In-tree `CustomPayloadMirroringQueryHandler` (`cql3/CustomPayloadMirroringQueryHandler.java:36`, used by `test/unit/.../transport/MessagePayloadTest.java`).
  - **I:** Stargate v1's `persistence-cassandra-*` modules installed a `StargateQueryHandler` to intercept auth and statements.
  - **I:** DSE ships its own handler.
- **Compatibility (F):** 7 commits since 2019, e.g. `dc17c29724` added `Dispatcher.RequestTime`, an incompatible signature change. There is no harness beyond the mirroring test.
- **Grade: C-.** It is the only front-end interception point, but it is a single global slot, sees only part of the traffic, and its signatures are internal.
- **Recommendation:**
  - Replace it with a chain of `QueryInterceptor`s: pre-parse, post-parse, pre-execute, and post-result, with `CQLStatement` exposed as a read-only descriptor.
  - Load it from yaml as well as the system property.
  - Make it the documented home for alternative front ends. Long-term, see P2.

## 6. CQL parse / prepare pipeline

- **F:**
  - The grammar is a single ANTLR3 grammar (`src/antlr/Parser.g`, 2487 lines; `Lexer.g`; `Cql.g`). `QueryProcessor implements QueryHandler` (`cql3/QueryProcessor.java:112`), and `getStatement` parses and prepares (`:950`).
  - The prepared cache is a Caffeine `Cache<MD5Digest, Prepared>` (`:123,143`) and is persisted to `system.prepared_statements`.
  - `CQLStatement` (`cql3/CQLStatement.java:32`) is `authorize / validate / execute / executeLocally / getAuditLogContext`, and `CQLStatement.Raw.prepare(ClientState)` is `:132-141`.
  - `SelectStatement` is 2082 lines and `StatementRestrictions` is a `final class` of 967 lines.
- **Leakage:** syntax, semantic analysis, planning (`getQuery` → `getRangeCommand`/`getSliceCommands`, `SelectStatement.java:494-514`), and execution are fused in each statement class. No logical plan IR exists between parse and `ReadCommand`.
- **Grade: E.** There is no seam. Extending the grammar means editing Parser.g.
- **Next steps:** see P1 (logical IR) and P2.

## 7. Triggers — `ITrigger`

- **Status:** stable-public, legacy.
- **F:**
  - `ITrigger.augment(Partition) → Collection<Mutation>` (`triggers/ITrigger.java`). It is unchanged since 2013 (0 commits since 2019), which is the most stable interface in this area.
  - Created with `CREATE TRIGGER … USING 'fqcn'` (`Parser.g:1171`).
  - Jars are loaded from `-Dcassandra.triggers_dir` (`CassandraRelevantProperties.java:732`) through an isolated `CustomClassLoader` (`triggers/CustomClassLoader.java:47`), with hot reload via `nodetool reloadtriggers` (`tools/nodetool/ReloadTriggers.java:24`).
  - `triggers_policy` = `enabled | disabled | forbidden` (`config/Config.java:1605`; enforced at `triggers/TriggerExecutor.java:241-250,318-325`).
  - Triggers execute on the coordinator in `StorageProxy.mutateWithTriggers` (`service/StorageProxy.java:1230-1262`, branch line numbers) and in the CAS path (`:487`).
  - An example lives at `examples/triggers/.../AuditTrigger.java`.
- **Leakage:** callers see the internal `Partition` and `Mutation` types. Triggers run synchronously on the write path and may produce cross-partition mutations, which conflicts with Accord/transactional domains.
- **Grade: C.** The interface is tiny, stable, and has a real classloader, but its types are internal and its semantics are underspecified.
- **Recommendation:**
  - Define a stable, read-only `RowChange` view and an explicit "augment-only same-token" mode.
  - Add a CDC-style async alternative so new ideas can leave the write path.
  - The CASSANDRA-17280 "hooks" note (`config/DatabaseDescriptor.java:1107-1109`) is the natural home for this work.

## 8. UDF / UDA

- **F:**
  - JavaScript was removed. `scripted_user_defined_functions_enabled` is deprecated (`config/Config.java:701-704`), and setting it fails startup with "removed in CASSANDRA-18252. Hooks are planned … CASSANDRA-17280" (`config/DatabaseDescriptor.java:1106-1109`). `UDFunction.create` now always builds `JavaBasedUDFunction` (`cql3/functions/UDFunction.java:260-270`).
  - The sandbox is a compile-time allow/deny list (`UDFunction.java:104-172`), bytecode checks (`UDFByteCodeVerifier.java:47`), a no-permission `ProtectionDomain` (`JavaBasedUDFunction.java:204`), and `ThreadAwareSecurityManager` installed via `System.setSecurityManager` (`security/ThreadAwareSecurityManager.java:48,95`). On JDK 21 it requires `-Djava.security.manager=allow` (`conf/jvm21-server.options:150`).
  - Config knobs: `user_defined_functions_enabled` (default false), `allow_insecure_udfs`, `allow_extra_insecure_udfs`, and warn/fail timeouts (`Config.java:698-773`).
  - Broken functions (class missing on some nodes) become placeholder "broken" functions (`UDFunction.java:272-290`).
- **Risk (I, high confidence):** the SecurityManager is deprecated (JEP 411) and permanently disabled in JDK 24 (JEP 486). The UDF sandbox therefore has no future on newer JDKs (current `java.supported` = 11,17,21, `build.xml:48`).
- **External:** user code only. No known library ecosystem.
- **Grade: C.** The CQL surface is clean and schema-replicated, but the isolation mechanism is end-of-life and only one language is supported.
- **Recommendation:**
  - Introduce a `FunctionRuntime` SPI keyed by `LANGUAGE`. Candidates: Java-in-isolated-classloader without SecurityManager plus a resource budget, WASM (e.g., Chicory, pure JVM), or out-of-process.
  - Keep the UDF/UDA schema and CQL unchanged.
  - Do this before the JDK 24+ move.

## 9. Native/system function registry

- **F:** `NativeFunctions.instance` is a hard-coded initializer that calls `XxxFcts.addFunctionsTo(this)` for 16 families, including masking, vector, and cluster-metadata functions (`cql3/functions/NativeFunctions.java:34-54`). `add(NativeFunction)` and `add(FunctionFactory)` are public (`:62-80`), but no loader exists.
- **Leakage:** implementers would use `NativeScalarFunction` with `ByteBuffer`-level `Arguments`.
- **Grade: E.**
- **Recommendation:** a `ServiceLoader<FunctionFactory>` contributing to a `system_ext`/plugin keyspace namespace (not `system`), version-gated through TCM so all nodes agree. This is roughly a two-file change.

## 10. Custom types — `AbstractType` by class name

- **F:**
  - A CQL `'org.foo.MyType'` becomes `CQL3Type.Custom(String className)` (`cql3/CQL3Type.java:177-186`).
  - `TypeParser.getAbstractTypeClass` loads the class with `classForNameWithoutInitialization`. Names without a dot get `org.apache.cassandra.db.marshal.` prepended (`db/marshal/TypeParser.java:492-500`). The singleton comes from a static `instance` field or `getInstance(TypeParser)` (`:475,505-510`).
  - `AbstractType` (`db/marshal/AbstractType.java:66`, 34 commits since 2019) requires `fromString`, `fromJSONObject`, `getSerializer`, comparison/byte-comparable semantics, multi-cell/freeze behavior, and so on.
  - No guardrail restricts custom types.
- **Leakage:**
  - The type's class name is persisted in schema (TCM) and in SSTable `SerializationHeader`s, so every node *and every offline tool* needs the jar.
  - Drivers see an opaque `custom` protocol type. **I:** most drivers return raw bytes.
  - Byte-comparable/ordering contracts are tied to storage-engine internals.
- **Grade: D.** It works, but it is a persistent, cluster-wide classpath dependency with no versioning.
- **Recommendation:** steer extension toward UDTs and vectors plus functions. If custom types are kept, gate them behind a guardrail, require a `TypeProvider` SPI with an explicit format version, and refuse schema changes when any node lacks the provider (via a TCM capability check).

## 11. Constraints — CEP-42 `ConstraintProvider`

- **Status:** partial (newest SPI in the area).
- **F:**
  - Syntax is `CHECK <constraint> AND …` (`Parser.g:993`) with `DROP CHECK` (`:1221`).
  - The SPI is `cql3/constraints/ConstraintProvider.java:28`, with `getUnaryConstraint`, `getConstraintFunction`, and satisfiability-checker lists. It is loaded by `ServiceLoader.load(ConstraintProvider.class).findFirst()` (`cql3/constraints/ConstraintResolver.java:41`), and **only the first provider wins**.
  - Built-ins are enums (`ConstraintResolver.java:49,62`).
  - An example lives at `examples/constraints/` (with `META-INF/services`), and a test at `test/unit/.../constraints/ConstraintsProviderTest.java`.
- **Leakage (F):**
  - Schema serialization writes the constraint *kind* as an enum ordinal (`ColumnConstraint.java:52-56`, `ColumnConstraints.java:267`), and function constraints by name and args (`FunctionColumnConstraint.java:169-195`).
  - Deserialization resolves the name through the provider (`:195`). A node or tool without the provider jar therefore cannot rehydrate schema. **I:** this would likely fail during TCM log replay.
- **Grade: B-.** It has a real SPI, an example, and a test, and its types are mostly CQL-level. The weaknesses are single-provider-only, no version field, and a missing-provider failure mode.
- **Next steps:** support multiple providers with namespacing, add a provider version to the serialized form, gate `CREATE` through TCM on "all nodes have provider X", and add a test-jar TCK. This SPI is the template to copy for #9, #10, and #13.

## 12. Data masking (CEP-20)

- **F:**
  - `dynamic_data_masking_enabled` (default false, `Config.java:752`).
  - Syntax `MASKED WITH fn(args)` or `MASKED WITH DEFAULT` (`Parser.g:1025-1026`).
  - Native masks are `MaskingFunction` subclasses (`cql3/functions/masking/`).
  - `ColumnMask.findMaskingFunction` accepts native `MaskingFunction`s **or a UDF in the same keyspace** (`ColumnMask.java:236-255`), so UDFs are already the extension path.
- **Leakage:** custom masks inherit the UDF sandbox risk (#8). Masks are applied at result-set build, so they cannot push down.
- **Grade: B-.** It is the cleanest "extension via CQL objects" example.
- **Recommendation:** after #9 lands, allow masking functions from plugin function factories. No other change is needed.

## 13. Virtual tables

- **F:**
  - `db/virtual/VirtualTable.java:34` has `metadata`, `apply(PartitionUpdate)`, two `select(...)` overloads (`:71,82`), `truncate`, `allowFilteringImplicitly`, `sorted`, and `indexes()`. Helpers include `AbstractVirtualTable`, `AbstractLazyVirtualTable`, `AbstractMutableVirtualTable`, and `SimpleDataSet`.
  - The registry is `VirtualKeyspaceRegistry.register` (`db/virtual/VirtualKeyspaceRegistry.java:42`), but registration is **hard-coded** in `service/CassandraDaemon.java:643-650`.
  - No config key or ServiceLoader exists, and no plugin virtual keyspace is possible without patching.
- **Leakage:** `select` takes internal `DataRange`, `ClusteringIndexFilter`, `ColumnFilter`, `RowFilter`, and `DataLimits` and returns `UnfilteredPartitionIterator`. `AbstractVirtualTable`/`SimpleDataSet` hide most of this.
- **Grade:** C+ as an API, F as a plugin point.
- **Recommendation (cheap, high value):** `ServiceLoader<VirtualKeyspaceProvider>` invoked at `CassandraDaemon.java:~650`, restricted to node-local keyspaces whose names are not `system*`. This is the lowest-risk way for operational ideas (diagnostics, sidecar-like views, optimizer EXPLAIN tables) to grow outside mainline.

## 14. Coordinator read/write path — `StorageProxy`

- **F:**
  - Everything is a static method: `cas` (`service/StorageProxy.java:370`), `mutate` (`:981`), `mutateWithTriggers` (`:1230`), `read` (`:2185`), and `getRangeSlice` (`:2844`). Branch line numbers apply.
  - The only policy-like hooks are the partition denylist and guardrails. `WritePerformer` (`:3127`) is an internal strategy.
  - The branch adds `TransactionDomainGuard.check` at four entry points (branch-only, see header). This is itself evidence that cross-cutting coordinator policy currently requires source edits.
- **Grade: F.**
- **Recommendation:** add a `CoordinatorInterceptor` chain (`beforeRead`/`beforeWrite`/`beforeCas`, veto or annotate only, no rewriting) and move denylist, transaction-domain guard, and guardrail checks onto it. Rewriting the replica plan stays out of scope (it belongs to the consistency area's file).

## 15. Query planning, restrictions, index selection

- **F:**
  - `StatementRestrictions` (final) decides legality and `usesSecondaryIndexing` using `IndexRegistry` + `IndexHints` (`cql3/restrictions/StatementRestrictions.java:205-383`).
  - Hints are `SELECT … WITH included_indexes = … AND excluded_indexes = …` (`cql3/statements/SelectOptions.java:40-43`; `db/filter/IndexHints.java`). They are on trunk.
  - Group selection happens in `SecondaryIndexManager.getBestIndexQueryPlanFor` (`:1308`). Iterator selection happens inside SAI `QueryController`.
- **Reuse [QO]:** decisions are spread across these layers. Exclusion acts early, but the intersection clause limit acts late. Marginal statistics cannot pick the right plan. Any plan interface needs an explicit legality contract covering reconciliation, RFP, paging, static rows, and limits. CEP-39 is still "Under Discussion" (see [QO] `research/cep-39-and-reusable-hooks.md`, `ASSESSMENT.md` §§1–5).
- **Grade: D.** Hints are a user-facing knob, not an SPI.
- **Recommendation:** see P1. Start with a read-only plan descriptor and an EXPLAIN virtual table (needs #13), exactly as [QO] proposes.

## 16. Result paging

- **F:**
  - The `QueryPager` interface (`service/pager/QueryPager.java:49-157`) has concrete `SinglePartitionPager`, `PartitionRangeQueryPager`, `MultiPartitionPager`, and `AggregationQueryPager`. Pagers are built by `ReadQuery.getPager`, which is not pluggable.
  - `PagingState` is opaque bytes to the client.
  - Top-K queries opt out of paging (`SelectStatement.java:430,649`).
- **[QO]:** single-partition paging rebuilds the command, while range paging carries the existing plan, so a planner must preserve continuation semantics.
- **Grade: D.**
- **Recommendation:** version `PagingState` with a plan or engine tag so that a plugin index or engine can own its own continuation token (vector/top-K "next page" is the motivating case).

## 17. Guardrails as policy hooks

- **F:**
  - About 60 static guardrails in `db/guardrails/Guardrails.java` are invoked inline (e.g., `SelectStatement.java:366,378,617,831`).
  - `GuardrailsConfigProvider` is chosen by `-Dcassandra.custom_guardrails_config_provider_class` (`GuardrailsConfigProvider.java:48-49`). It returns a per-`ClientState` config, and its javadoc states the mechanism "not officially supported and may change in a minor release" (`:36-37`).
  - `CustomGuardrail` + `ValueValidator` are loaded by `validator_class_name` from yaml (`ValueValidator.java:44,138`), but only for `password_policy`/`role_name_policy` (`Config.java:1103-1104`).
  - Existing query-area guardrails include `allowFilteringEnabled`, `intersectFilteringQueryEnabled`, `saiSSTableIndexesPerQuery`, `nonPartitionRestrictedIndexQueryEnabled`, `vectorDimensions`, and `preparedStatementsRequireParameters`.
- **Grade: C.** Per-tenant config is pluggable, but new *rules* are not.
- **Recommendation:** generalize `CustomGuardrail` into a `QueryPolicy` SPI evaluated on the read-only statement descriptor from #5/P1. This is the natural landing spot for "reject plans like X" without touching mainline.

## 18. `QueryEvents.Listener`

- **F:** an observer with success/failure callbacks for query, execute, and batch (`cql3/QueryEvents.java:57,270-300`). It is used by `AuditLogManager` (`audit/AuditLogManager.java:252`) and `FullQueryLogger` (`fql/FullQueryLogger.java:104`). There is no external loader.
- **Grade: C.** It is a clean observer, but exposes internal types and has no loader.
- **Recommendation:** load listeners via ServiceLoader. It then becomes the passive half of the #5 interceptor chain (shadow planning and plan-feedback collection per [QO]).

---

## Cross-cutting findings

1. **Five loading mechanisms coexist (F):**
   - reflective class name stored in schema (index, trigger, type)
   - JVM system property (QueryHandler, GuardrailsConfigProvider)
   - yaml class name (`validator_class_name`)
   - `ServiceLoader` (ConstraintProvider, SSTableFormat.Factory, StartupCheck: `DatabaseDescriptor.java:2006`, `StartupChecks.java:172`)
   - hard-coded registration (functions, virtual tables)

   None checks plugin version, and none checks cluster-wide presence. **Recommendation:** one `PluginLoader` built on ServiceLoader, with a declared API version and a TCM-published "node capabilities" set. Schema DDL that references a plugin is then rejected unless all nodes advertise it. This removes the "class missing on some nodes" failure mode that `UDFunction.createBrokenFunction` (`UDFunction.java:272`) and constraint deserialization (#11) currently paper over.
2. **No `@Public/@Evolving` annotations exist anywhere in this area (F).** Change counts since 2019 (git log, trunk): `ITrigger` 0, `ConstraintProvider` 1, `GuardrailsConfigProvider` 5, `QueryHandler` 7, `VirtualTable` 8, `Index` 29, `AbstractType` 34.
3. **Schema-persisted plugin references outlive the plugin jar.** This affects indexes, triggers, types, constraints, and UDFs. Offline tools (sstabledump, sstableloader) need the jars for types and SSTable-attached indexes.
4. **Internal callers bypass the extension points** (`QueryProcessor.instance` used directly, #5). A "new idea" plugin cannot assume it sees all traffic.

---

## Proposed new boundaries

### P1. Pluggable query planner with a statistics SPI (aligned with CEP-39; builds on [QO])

- **Where to cut:**
  - Between `SelectStatement.getQuery` (`SelectStatement.java:494-514`) and `ReadCommand` construction, and around `SecondaryIndexManager.getBestIndexQueryPlanFor` (`:1308`).
  - Introduce an immutable `LogicalRead`: table, partition and clustering predicates, residual predicates, ordering (incl. ANN), limits, paging state, and a consistency/reconciliation trait. Introduce a `PhysicalCandidates` set enumerated from the existing `Index.Group.queryPlanFor`.
  - `QueryPlanner.choose(LogicalRead, Candidates, StatsProvider, RequestContext) → PhysicalChoice`. The default implementation reproduces today's comparator exactly.
  - `StatsProvider` returns estimates as intervals with provenance and freshness, never as correctness premises ([QO] `design/plan-and-statistics-contract.md`).
- **Guardrails:**
  - Eligibility precedes cost. The planner may only choose among candidates that `StatementRestrictions` already marked legal.
  - Replicas must re-derive the same choice, or the choice must be serialized in `ReadCommand` (today only the first `IndexMetadata` is, `ReadCommand.java:481`). This needs a messaging-version bump.
- **First deliverable:** read-only (shadow) planner plus an `EXPLAIN` virtual table, then the scalar two-index experiment in [QO] `experiments/FULL-CQL-FOLLOWUP.md`.

### P2. Pluggable query-language front ends over a logical-plan IR

- **Where to cut:**
  - Promote P1's `LogicalRead` and an equivalent `LogicalWrite` (mutations + conditions + TTL/timestamp) to a stable, versioned IR.
  - The CQL front end (Parser.g → `CQLStatement.Raw.prepare`) becomes one producer. A `FrontEnd` SPI (`parse(text, session) → LogicalStatement`) plus a protocol adapter lets a Postgres-wire, document/JSON, or GraphQL layer emit the same IR without producing internal `CQLStatement`s.
  - Execution, auth, audit (#18), guardrails (#17), and the planner (P1) all consume the IR, so new front ends inherit them.
- **Interim step (low risk):** a `QueryHandler` v2 (#5) that accepts a `LogicalStatement` from the handler instead of requiring a `CQLStatement`.
- **I:** Stargate v1 demonstrated demand for exactly this, and solved it by embedding Cassandra behind its own coordinator. A stable IR would have let it run as a plugin.
- **Non-goal:** joins, subqueries, and SQL federation stay out until the planner exists ([QO] ASSESSMENT "Full SQL federation … separate follow-on").

### P3. Pluggable vector index engines

- **Where to cut:**
  - Inside SAI, behind `VectorIndexSegmentSearcher` (`index/sai/disk/v1/segment/VectorIndexSegmentSearcher.java`) and the memtable `VectorMemoryIndex` (`index/sai/memory/VectorMemoryIndex.java`).
  - Define a `VectorEngine` SPI: `buildSegment(vectors, rowIds, options)`, `search(query, k, Bits acceptRows, budget) → scored rowIds`, `memtableIndex()`, `componentNames()`, `formatVersion`.
  - Move jvector behind it as the default engine. Replace the leaked `VectorSimilarityFunction` with a Cassandra enum (`IndexWriterConfig.java:47-48`).
  - Select the engine via an index option (`WITH OPTIONS = {'engine':'jvector'}`), registered through the SAI `Version`/`OnDiskFormat` ServiceLoader from #2.
- **Why inside SAI and not as a separate `Index`:** reusing SAI keeps hybrid filter+ANN, top-K paging/postprocessing, and SSTable-attached lifecycle shared. A standalone `Index` would have to re-solve all three (#3).

### P4. Small enabling SPIs (do first; low risk)

1. `VirtualKeyspaceProvider` via ServiceLoader (#13).
2. `FunctionFactory` via ServiceLoader into a plugin namespace (#9).
3. Multi-provider `ConstraintProvider` with versioned serialization (#11).
4. A `QueryEvents.Listener` loader (#18).
5. The `PluginLoader` with a TCM capability set (cross-cutting #1). This is the prerequisite for any schema-referenced plugin to be safe.

### P5. UDF runtime SPI (time-boxed by the JDK roadmap)

`FunctionRuntime` keyed by `LANGUAGE` (#8), with the SecurityManager-free Java runtime and WASM as candidates. It must land before Cassandra supports a JDK where `java.security.manager=allow` is gone (I: JDK 24+).

---

## Confidence / method

- **What was checked:** every F-line was checked with `rg`/`sed` against the working tree, branch differences were checked with `git diff trunk...HEAD`, and change counts come from `git log trunk --since=2019-01-01`.
- **What was not run:** no build or tests.
- **Unverified claims:** external implementations (Stratio, Elassandra, DSE, Stargate, DataStax SAI fork) and JDK SecurityManager removal timing come from background knowledge and are marked I.
