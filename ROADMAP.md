# Pluggable boundaries for Apache Cassandra: catalog and proposed roadmap

Status: draft synthesis, 2026-10-03. Built from eight research reports in [`reports/`](reports/) (01–08). Every grade, count, and `file:line` claim comes from those reports, which carry the citations. Mick Semb Wever's Apache Cassandra feature mindmap (not published here) is mapped onto the catalog in §6a.

| Report | Area |
|---|---|
| [01](reports/01-storage-engine.md) | Storage engine and local persistence |
| [02](reports/02-query-indexing.md) | Query, indexing, types, functions |
| [03](reports/03-distribution-consistency.md) | Distribution, membership, replication, consistency, data movement |
| [04](reports/04-security-ops.md) | Security, observability, in-node operations |
| [05](reports/05-edges-protocol-modularity.md) | Client protocol, internode, serialization, CDC, build modularity |
| [06](reports/06-upstream-history.md) | CEPs, JIRA, dev@ stance, analogous projects |
| [07](reports/07-ecosystem.md) | Sidecar, Analytics, drivers, third-party plugins and their upgrade pain |
| [08](reports/08-prior-research.md) | This repo's pluggable-consistency and optimizer work |

---

## 1. The thesis in one paragraph

Cassandra already has about **100 extension points**, but almost none of them is a *boundary*. A boundary is something a new idea can build against without forking, without reaching into internals, and without breaking on the next patch release. Today the project has (a) no definition of which Java interfaces are public, (b) five incompatible ways of loading a plugin from a flat classpath, (c) no way for the cluster to know that every node has the plugin a table depends on, and (d) no test kit an out-of-tree implementer can run. The consequence is visible in the ecosystem: Analytics shades a whole server per major, the Management API and the exporters patch bytecode and break on patch releases, Stargate gave up at 5.0, and Rocksandra and Elassandra died as forks. The roadmap therefore leads with a **thin plugin platform** (policy, artifact, loader, cluster registry, test kit, maturity ladder). It then hardens the seams that already work, and only after that opens new seams. Each new seam is paired with a named consumer whose per-version fork it would delete.

## Top 10 boundaries to create

Ranked in build order. Earlier items unblock later ones and carry less risk. Each is a slot or stage of one of the six cuts in §6, so building all ten requires six core refactors, not ten.

| # | Boundary | Cut (§6) | Who uses it first | Mindmap ideas it unlocks |
|---|---|---|---|---|
| 1 | **Plugin platform**: `cassandra-api` artifact, isolated plugin loader, TCM plugin registry, conformance test kit | 0 | Every plugin below; pilots are `StartupCheck`, `IAuditLogger`, CEP-42 constraints | In-process upgrades |
| 2 | **UCS compaction controller SPI** | 0 (slot) | Adaptive and cost-model compaction research | Adaptive UCS |
| 3 | **Observability SPI**: metrics reporters with an OpenTelemetry bridge, tracing (CEP-32), event sinks, lifecycle hooks | 0 (slots) | cassandra-exporter, OTel agent, Management API: removes bytecode agents | Sidecar, K8ssandra |
| 4 | **Change-stream SPI** (CDC v2): typed post-apply change records with an origin tag; fixes Accord writes being invisible to CDC | 3 | Debezium, DataStax CDC agent, Sidecar CDC (CEP-44) | Point-in-time backups, zero-downtime migrations, mutation-based repair |
| 5 | **`cassandra-storage-format` library**: versioned SSTable and commitlog reader/writer usable outside the server | 3 | Cassandra Analytics (deletes per-version bridges), Medusa | Off-process Spark Analytics, attached Parquet, SQL reads |
| 6 | **Management API**: one command model behind nodetool, JMX, CQL and Sidecar REST (CEP-38), plus Sidecar modules | 5 | cass-operator / Management API, Reaper | Sidecar, K8ssandra donation |
| 7 | **Request pipeline**: admission check before the first durable write, plus a client front-end SPI | 1 | The domain guard from the consistency prototype; an HTTP or gRPC front end | Full SQL, Mongo, Redis, Data API, QUIC protocol, thin drivers |
| 8 | **TransactionProvider**: pluggable execution engine on the request pipeline | 1 (slot) | Accord, plus the etcd and Ratis prototype providers | Interactive transactions, snapshot isolation, expressive consistency |
| 9 | **Storage substrate** (CEP-36): per-directory channel factory, data placement, cache tier | 4 | An out-of-tree object-store implementation | Object storage, compute–storage separation |
| 10 | **Read planner + statistics SPI with `VectorEngine`** (CEP-39) | 2 | The query-optimizer experiment; jvector plus a second vector engine | SAI, OR/NOT, sortable indexes, vector search |

Why this order: items 1–3 are small and prove the platform. Items 4–6 deliver the most value to existing ecosystem projects for the least core risk. Items 7–10 are where new engines (consistency, APIs, storage, planning) take off, and they depend on the platform being in place.

## 2. What history says will and will not work (from [06](reports/06-upstream-history.md))

- **What fails:** broad pluggability framed as a goal. The 2017 pluggable storage engine (13474/13475, open since 2017), CEP-18 "Improving Modularity" (withdrawn 2021), pluggable membership (17058), and the pluggable CommitLog thread all died on the same objections: interfaces with only one implementation, undefined semantics, and slower mainline development.
- **What succeeds:** a narrow seam that arrives *with* its second implementation. Examples: CEP-11 memtable + CEP-19 trie memtable, CEP-17 SSTable format + CEP-25 BTI, CEP-9 SSLContext, CEP-42 constraints, CEP-49 compression accelerators. CEP-49's Intel QAT plugin, hosted outside the ASF, is the strongest out-of-tree precedent.
- **The standing rule is Blake Eggleston's (2017):** every refactor must "make sense on its own" and stay "abandon-able".
- **The open gaps are the ones the community itself keeps naming:**
  - Capwell (2025): "We don't have any process to even know what is public API".
  - The 2024 experimental/beta/GA debate ended with no outcome.
  - CEP-65 explicitly *deferred* the plugin SPI surface as "a legitimate and separate problem".
  - Since 2025 there has been support for an interfaces jar as a Gradle subproject in the same repo (Benedict, Haddad, Shepherd) and for classloader isolation.
- **Do not lead with "pluggable storage engine."** It is the most-rejected framing in the project's history.

## 3. Design principles for every boundary

These are merged from the team's own lessons ([08](reports/08-prior-research.md) §2–3), the ecosystem evidence ([07](reports/07-ecosystem.md) L1–L7), and the dev@ objections ([06](reports/06-upstream-history.md) §3b).

1. **Consumer first.** No seam without a named consumer. Success is measured by deleting that consumer's per-version module ([07] L7).
2. **Two implementations before "Stable."** The default implementation plus a genuinely different second one. A delegating adapter proves nothing ([08] lesson 11; CEP-11/19 and CEP-17/25 are the models).
3. **Self-sufficient SPIs.** Signatures use only types the SPI package owns: no `db.rows.*`, no `cql3.statements.*`. The server passes callbacks and context in; plugins never call statics ([07] L3; LDAP and Stargate needed per-version modules despite using official interfaces).
4. **Explicit context, no singletons across the seam.** Pass the `ClusterMetadata` snapshot rather than calling `ClusterMetadata.current()`, which has 383 call sites. Accord bug #8 is the cautionary case. `repair/SharedContext` is the in-tree pattern to copy ([03] §5, [08] lesson 6).
5. **Cluster-wide presence through TCM.** Anything that affects schema, placement, ownership, or execution is activated by a TCM transformation that requires every node to advertise the plugin and its version. The plugin fails closed at DDL time, never at replay ([03] §5, [02] cross-cutting #1).
6. **Stable identities, not class names or ordinals.** Provider IDs are permanent strings. Enum ordinals are never persisted. Unknown versions fail closed ([08] lessons 5 and 8). Today compressor names in `CompressionInfo.db` and class names in schema make plugins permanent dependencies of the data they wrote ([01] cross-cutting #3).
7. **Self-describing versioned payloads.** Plugin data on disk or the wire uses Accord's embedded versioned serializer pattern (`LATEST` / `CLUSTER_SAFE` / `DOWNGRADE_SAFE`) ([05] §12).
8. **Gate before the first durable side effect.** Admission must come before the commitlog append, not after ([08] lesson 2).
9. **Failure paths in the acceptance suite.** Every boundary's test kit includes kill and restart and mixed-version cases ([08] lesson 14).
10. **Abandon-able refactors.** Every extraction must improve the core even if the plugin never ships ([06] §2a).
11. **Out-of-process only helps across a published contract.** Sidecar's CDC still pulls in `cassandra-all` through the Analytics bridges, so the coupling just moved ([07] L1).

## 4. Layer 0: the plugin platform (does not exist; prerequisite for everything)

| ID | Component | What it is | Evidence it is needed | Precedent to copy |
|---|---|---|---|---|
| **F1** | **Public SPI CEP + stability annotations** | A published list of supported SPIs; `@Stable` / `@Evolving` / `@Experimental` annotations; tie-in to the 2025 T-1 compatibility and deprecate-then-remove vote | Three project documents disagree on what is public ([06] §3e). No annotations exist anywhere ([02] cross-cutting #2) | Hadoop `InterfaceStability`, Spark `@DeveloperApi`, Kafka KIP public-interface sections |
| **F2** | **`cassandra-api` artifact** | A Gradle subproject in the same repo holding only SPIs and owned value types, with no `DatabaseDescriptor` or singletons. Plus a report-only, then ratcheting, import-layering check (ArchUnit or checkstyle ImportControl) | No enforced package boundaries; `utils` and `io` import `db` ([05] §19). Plugin authors compile against `cassandra-all` ([06] §3c) | **in-JVM `dtest-api` (A−)**: a tiny, separately versioned jar implemented per branch ([05] §20); `modules/accord` Gradle-in-Ant |
| **F3** | **Unified plugin loader** | A `plugins/<name>/plugin.yaml` descriptor (name, version, per-SPI api-version), a child-first classloader with an allowlist for `cassandra-api` and the JDK, a single constructor contract with `init/start/stop`, the `-Dcassandra.custom_*` hooks moved into yaml with aliases, and a `system_views.plugins` virtual table. Existing `class_name` keys keep working | Five loader idioms, a flat `lib/*.jar` classpath, sysprop hooks invisible to `system_views.settings` ([04] §0) | `triggers/CustomClassLoader`, UDF sandbox loader, Kafka Connect `plugin.path` |
| **F4** | **TCM plugin registry and capability vector** | Per-node plugin and version vectors in `Directory`. A "require plugin X ≥ v" transformation. DDL referencing an absent plugin is rejected. `ExtensionKey` / `CustomTransformation` keyed by stable IDs rather than class names. Plugin payloads use F7-style versioned serializers | Missing memtable classes on some nodes was Capwell's pain point ([06] §3b). `UDFunction.createBrokenFunction` papers over the same problem ([02]). The prototype had to bump core TCM V11–V13 for each provider kind ([08] lesson 8) | The CEP-42 "node fails to start if a constraint is missing" outcome; TCM `Kind.introducedIn` gating |
| **F5** | **Conformance kits (plugin TCK)** | Subclassable abstract suites per SPI, run on the in-JVM dtest framework, including kill/restart and mixed-version cases. Out-of-tree plugins run them in ASF CI against every release | No SPI has a TCK. `ant test-latest` already runs the full suite against alternate *in-tree* memtable, format, and compaction choices, which is a de facto TCK ([01] cross-cutting #1) | Lucene codec test framework; `test-latest` |
| **F6** | **Maturity ladder with a deletion path** | Experimental (off by default, removable without deprecation) → Beta (one major) → Stable → Default. Plus a `sandbox/` area in trunk with an explicit deletion rule | The 2024 debate was unresolved; "all new features disabled by default" is already agreed ([06] §3d) | Linux `staging/`, Lucene `sandbox`, OpenSearch `sandbox/` |
| **F7** | **Ecosystem tier** | A listing of out-of-tree plugins in a `cassandra-ecosystem` area (CEP-63), each "tested with X.Y.Z" in ASF CI | Vendors need a path that is neither fork nor merge (Jirsa 2017). In-tree vendor code rots (CloudstackSnitch) | spark-packages, PGXN, CEP-49 QAT |

F1 changes no code and can start immediately. F2–F5 together are the "plumbing" CEP.

---

## 5. Consolidated boundary catalog

**Status** values: `stable` (config- or schema-selectable, used externally); `seam` (an interface exists but is not a contract); `partial` (selectable, but core special-cases built-ins); `closed` (an enum or hard-coded); `none`.

**Action** values: **Keep**; **Harden** (annotate, add a TCK, fix leaks); **Formalize** (turn a seam into a contract); **New** (propose a seam).

### 5.1 Data plane: storage

| Boundary | Status | Grade | Action | Notes |
|---|---|---|---|---|
| Compression `ICompressor` + CEP-49 providers | stable | B+ | Keep / Harden | Persisted compressor names pin plugins to data |
| Memtable API (CEP-11) | stable | B | Harden | Leaks `ColumnFamilyStore` and commitlog mechanics |
| Row cache provider | stable | B− | Keep | |
| `journal` package (Accord's log) | seam | B− | Formalize as a library | Remove the `StorageService` call; reusable by new providers and mutation tracking |
| SSTable format API (CEP-17) | stable (node-wide only) | C+ | Harden | Huge abstract reader, shared version flags, local-file assumption |
| Compaction strategy | stable (de facto) | C | Formalize | Strategies get the whole CFS; core type-checks built-ins |
| UCS `Controller` | closed | D | **New (S)**: controller SPI | Adaptive, cost-model, or ML-tuned compaction without forking UCS |
| CDC consumer (`CommitLogReader`) | partial | C− / D+ | **New**: change-stream SPI (5.4) | |
| Commitlog segment manager / WAL | seam (test hook) | D | New (later): WAL SPI | |
| Chunk cache | none | F | **New (S)**: cache-tier SPI | Local-NVMe cache for remote SSTables |
| File system / `ChannelProxy` (CEP-36) | seam (test swap) | D | **New (M)**: storage channel factory per data directory | Split generic FS registration from remote semantics, as dev@ asked |
| Directories / disk boundaries | none | D / F | **New (M)**: data placement policy | Hot/cold tiering |
| SSTable lifecycle / remote SSTables | none | — | New (L) | `Tracker.subscribe` publish hook; UUID sstable IDs |
| Columnar / projection scan | none | — | New (L) | Batch and projection extension of `UnfilteredSource` |
| Compaction offload | none | — | New (L) | The strategy chooses *what*, the executor chooses *where* |
| Whole-table storage (13474 style) | partial (write/stream/repair seams, hard-wired) | D | New (XL, staged, **do not lead with it**) | The virtual-table path proves `ReadCommand.queryStorage` is a workable cut |

### 5.2 Data plane: query, indexing, functions

| Boundary | Status | Grade | Action | Notes |
|---|---|---|---|---|
| Constraints SPI (CEP-42) | partial | B− | Harden | Only the first provider is used; unversioned serialization |
| Data masking (CEP-20) | partial | B− | Keep | |
| Secondary index `Index` / `Index.Group` | stable (de facto) | C | Harden | About 32 internal types leak; self-reported cost estimates; replicas re-derive the plan from the first index only |
| SAI on-disk `Version` / `OnDiskFormat` | seam | C− | Formalize | |
| Vector / ANN engine | closed (jvector) | D | **New**: `VectorEngine` SPI inside SAI | Keeps hybrid filter + ANN and SSTable-attached lifecycle shared |
| `QueryHandler` | partial (one sysprop) | C− | Replace with request API (5.5) | Internal callers and `BatchMessage` bypass it |
| Triggers | stable, legacy | C | Keep (frozen) | |
| Java UDF/UDA | stable, sandbox at risk | C | **New, time-boxed**: `FunctionRuntime` SPI | The sandbox depends on SecurityManager, deprecated and reportedly disabled on JDK 24+. Must land before Cassandra supports such a JDK |
| Native function registry | closed | E | **New (S)**: `FunctionFactory` via loader | |
| Custom types (`AbstractType`) | stable, legacy | D | Keep, discourage | |
| Virtual tables | good API, no loader | C+ / F | **New (S)**: `VirtualKeyspaceProvider` | Also the read path for observability plugins |
| Query planning / index selection | seam | D | **New**: planner + statistics SPI (CEP-39) | Builds on `query-optimizer/`: shadow planner + `EXPLAIN` first |
| CQL parse / prepare | closed | E | New (long): logical-plan IR for alternative front ends | |
| Paging | seam | D | Formalize alongside the planner | Paging has no plan identity |
| `StorageProxy` coordinator | static | F | Behind admission and request API | |

### 5.3 Control plane: distribution, consistency, metadata

| Boundary | Status | Grade | Action | Notes |
|---|---|---|---|---|
| `NodeProximity` / `InitialLocationProvider` | stable (new) | A− | Keep; add parameters | Location comes from TCM after registration |
| Accord host SPI (`accord.api.*`) | stable seam | B+ | Keep | accord-maelstrom is a second host |
| `SeedProvider` | stable | B+ | Keep | Template for small yaml plugins |
| Auto-repair token-range splitter | stable | B | Keep | |
| `IEndpointSnitch` | deprecated | B− | Migrating | **Model migration**: deprecate, adapt, replace with a TCM-native model |
| Replication strategy (custom `class`) | partial | C+ | **Harden**: a strategy-capability interface replacing 31 `instanceof NetworkTopologyStrategy` checks | Unblocks the Everywhere strategy and others |
| `PlacementProvider` | seam | C | **New**: pluggable placement (CEP-60) | Replay placements as data so non-CMS nodes do not need the class |
| `IPartitioner` | stable, frozen | C | Keep | |
| Speculative retry / read repair | closed (if-chains) | C | **New (S)**: ParameterizedClass factories | Node-local, no wire impact |
| Paxos v1/v2 and `transactional_mode` | closed enums | C | → TransactionProvider | |
| TransactionProvider + durable domains | **branch prototype only** | B− (prototype) | Productize after admission + F4 | One `execute` verb versus about ten in the design; hard-coded registry |
| TCM `Processor` / `LogStorage` | seam (test sysprop) | C− | New (late): metadata-log backend | The root of trust. Stays internal until a second backend passes the simulator |
| TCM `CustomTransformation` / `ExtensionKey` | seam (test-only users) | C | **Harden → F4** | Unknown extension asserts on replay |
| Coordinator write path | partial | D | **New**: admission SPI (prerequisite) | The prototype needed guards in 25 files (about 62 call sites) for one invariant |
| Coordinator read path (`ReadCoordinator`) | seam | C− | Admission SPI | |
| Repair (`TableRepairManager`, MerkleTree wire) | seam | C | New: anti-entropy engine | CEP-45 log-offset reconciliation as a second implementation |
| Streaming (`TableStreamManager`, `StreamingChannel.Factory`) | seam | C+ / C | New: streaming transport | Object-store-mediated transfer |
| Internode `Verb` / `IVerbHandler` | seam (reflective CUSTOM slot) | D+ | **New**: `VerbRegistry` with TCM-committed IDs | |
| Hints, batchlog, gossip, failure detector | static internals | D / D+ / C+ | Leave internal | Not where new ideas need to plug in |
| `ConsistencyLevel` | closed, wire-coded | F | Leave closed | New semantics go through TransactionProvider, not new CLs |
| Mutation tracking (CEP-45) | unmerged branch | C (prov.) | Watch | A second replication mode; a substrate for CDC and repair |

### 5.4 Edges: protocol, CDC, offline formats

| Boundary | Status | Grade | Action | Notes |
|---|---|---|---|---|
| in-JVM dtest API | stable, versioned jar | **A−** | **Copy its pattern** | |
| Protocol versions / beta flag | stable (wire spec) | B | Keep | Governance model for the plugin API version |
| Accord embedded versioned serializers | seam | B | **Generalize** into plugin serializers | |
| Custom payload | stable (wire) | B− | → negotiated protocol extensions | Untyped; DSE forks the version space instead |
| SSTable format SPI + `StorageCompatibilityMode` | partial | B− | Harden | |
| TCM `Version` + `Kind.introducedIn` | seam | B− | Basis of F4 | |
| Message params `CUSTOM_MAP`, sinks | partial | C+ / C− | `MessageInterceptor` SPI | Tracing context propagation |
| CDC `cdc_raw` file contract | stable, weak | C− | **New: change-stream SPI** | **Accord writes skip the commitlog, so they never reach CDC** (`TxnWrite.java:144-150`; read from code, not exercised) |
| Native listener / message types / frame compression | closed | D | **New**: client front-end SPI with shared admission | REST, gRPC, Redis-, DynamoDB-, and Mongo-wire front ends |
| `CQLSSTableWriter` / sstableloader | partial | C | Fold into the storage-format library | |
| Offline SSTable and commitlog reading | bridge-per-version | D | **New: `cassandra-storage-format` library** | Highest-leverage ecosystem fix |
| Build / modules | partial | D | F2 | |

### 5.5 Security, observability, operations

| Boundary | Status | Grade | Action | Notes |
|---|---|---|---|---|
| `IInternodeAuthenticator` | stable | A− | Keep | |
| `StartupCheck` (ServiceLoader) | stable | B+ | **Template for F3** | |
| `IAuthenticator` / mTLS validator | stable | B | Harden: self-sufficient role and credential callbacks | cassandra-ldap runs `CreateRoleStatement` internally and uses split packages |
| `IAuditLogger`, crypto provider, `INetworkAuthorizer` | stable | B | Keep | |
| `IAuthorizer`, `ISslContextFactory` | stable | B− | Keep | |
| `IRoleManager` | stable | C+ | Harden (as above) | |
| Guardrails config provider / custom guardrails | partial / stable | C / B | **New**: policy engine (OPA or Cedar adapters out of tree) | 119-method `GuardrailsConfig` |
| Observability event buses | seam | C | **New (S)**: event-sink SPI unifying audit, FQL, diagnostics | |
| `ConfigurationLoader` | partial (sysprop) | C− | New: layered `ConfigSource` SPI | |
| CIDR authorizer, KeyProvider/TDE | partial | D+ | New: key-management SPI | TDE covers only commitlog and hints |
| Tracing | partial (sysprop) | D | **New**: OpenTelemetry tracing SPI (CEP-32) | Silent fallback on error |
| Metrics reporters | removed (18743) | D | **New (S)**: `metrics_reporters` + OTel bridge | Removes the reason for most javaagents |
| FQL / BinLog | seam | D | → event-sink | |
| nodetool / JMX commands | static | D | **New**: management-command SPI (CEP-38) | Three front ends: nodetool, JMX, CQL |
| Lifecycle hooks | none | — | **New (S)**: five hooks | Daemon lifecycle, admin channel, seed reload, superuser provisioning, system keyspace replication. These are exactly what the Management API patches with ByteBuddy ([07] L5) |

### 5.6 Out-of-process (Sidecar)

| Boundary | Status | Action | Consumer whose fork it deletes |
|---|---|---|---|
| Sidecar module system | fixed Guice list | **New**: ServiceLoader `SidecarModule` + stable `ICassandraAdapter` + OpenAPI compatibility test (CEP-63 annotations) | Management API, ecChronos, esop parallel sidecars |
| Versioned management API (REST or CQL, CEP-38) | none (raw MBeans) | **New** | cass-operator (via the Management API agent), Reaper, esop's vendored MBeans |
| Snapshot / backup manifest | none (directory parsing) | **New (S)** | Medusa (broke on 5.0 permissions and BTI), esop, CEP-40 |
| Lifecycle / config / seed control | none | **New** (CEP-53) | Management API agent |

---

## 6. Roadmap: six cuts, many slots

The catalog lists about 35 seams to open. Opening them one at a time would mean about 35 core refactors, 35 dev@ debates, and 35 rounds of mixed-version reasoning. That is the pattern that sank 13474 and CEP-18. This roadmap groups them instead.

- **A cut** is core surgery done once. It refactors call sites, defines a contract with its context objects, settles the mixed-version rules, and ships a TCK. Each cut needs a CEP.
- **A slot** is a named extension point inside an existing cut. Adding one means a factory interface, a yaml key, and a TCK subclass. It needs no new loader, no new TCM version, no new wire version, and no new call-site refactor. It is a normal JIRA ticket, not a CEP.

The goal is that the project opens **six cuts**, and every idea after that, including everything on Mick's mindmap, becomes a slot.

### 6.1 Rules that keep later ideas inside existing cuts

1. **Design each cut for its whole family.** Before freezing a cut, list every known slot from the catalog and the mindmap, and validate the contract against at least two very different slots, not just the first consumer.
2. **Extend through capabilities, not new interfaces.** Implementations declare `capabilities()`, and the core checks them (as `supports(FAST_PATH)`, `supports(COLUMNAR_SCAN)`, `supports(REMOTE_FILES)`). A new ability becomes a new capability constant, not a new boundary. This is the general form of replacing the 31 `instanceof NetworkTopologyStrategy` checks.
3. **Pass one context object per cut** (request context, storage context, cluster context) instead of individual arguments or singletons. Adding information later adds a field, not a signature change.
4. **One registry for everything the cluster must agree on.** Verbs, serializers, TCM extension keys, persisted provider IDs, and the per-node plugin version vector all go through the Cut 0 registry. No slot ever bumps a core TCM or messaging version.
5. **The default implementation is slot #1.** Each cut ships with today's code moved behind it, so the cut is abandon-able and proves itself on day one.

### 6.2 The six cuts

| Cut | Where it goes | Slots it opens (consumer or second implementation) | Catalog items absorbed |
|---|---|---|---|
| **0. Plugin platform and cluster registry** | Packaging, startup, TCM `Directory`, serializer layer | Every node-local slot: UCS controller, speculative retry, read repair, proximity, function factory, virtual keyspaces, metrics reporters/OTel, tracing, event sinks, lifecycle hooks, guardrail policy, key management, config source, startup checks, auth | F1–F7, `VerbRegistry`, plugin serializers, TCM `ExtensionKey` hardening, the node-local quick wins |
| **1. Request pipeline** | Coordinator entry: front end → logical request → admit → execute | Client front ends (CQL native as #1, HTTP/gRPC, QUIC, Mongo, Redis, SQL), admission policies (domain guard), per-request semantic options, execution engines (StorageProxy as #1, Accord, TransactionProvider) | Admission SPI, client front-end SPI, request API, `QueryHandler` replacement, TransactionProvider |
| **2. Read planning** | Between `SelectStatement.getQuery` and `ReadCommand`, sharing Cut 1's logical request type | Planners (shadow/CBO), statistics providers, index implementations, `VectorEngine`, paging strategies | Planner + stats (CEP-39), Index API hardening, VectorEngine, SAI version formalization |
| **3. Data out** | One versioned row/file model used outside the server | Change listeners (`cdc_raw` as #1, Debezium, Accord-origin changes), offline readers/writers, analytics bridges, snapshot/backup tools, exporters (Parquet, Iceberg), PITR, CEP-45 reconciliation | Storage-format library, change-stream SPI, snapshot manifest, `CQLSSTableWriter` |
| **4. Storage substrate** | Where bytes live and how they are maintained: per-directory channel factory, placement, cache tier, SSTable lifecycle publish, compaction executor | Object storage, hot/cold tiering, NVMe cache, remote SSTables, compaction offload, streaming over object store | CEP-36 split, placement policy, cache tier, remote lifecycle, compaction offload, streaming transport |
| **5. Operations surface** | One management command model behind nodetool, JMX, CQL, and Sidecar REST, plus the Sidecar module loader | Commands, Sidecar modules (Management API, ecChronos, backup), operator integrations | CEP-38 command SPI, Sidecar plugin model, the five lifecycle-hook consumers |

**Deliberately not cuts** (they stay internal; revisit only after the six land): metadata-log backend, whole-table storage, WAL, gossip/hints/batchlog, and new `ConsistencyLevel` codes. Repair-engine and placement-provider (CEP-60) plugins are expected to fit as slots on Cut 0 (TCM-replayed decisions) plus Cut 3 (reconciliation input). If review shows they don't, they would be the seventh cut.

### 6.3 Order

```
Cut 0 ──┬── Cut 3 (data out)        ┐ lowest risk, highest ecosystem payoff
        ├── Cut 5 (operations)      ┘
        ├── Cut 1 (request pipeline) ── Cut 2 (read planning)
        └── Cut 4 (storage substrate)
```

| Step | Cut | Why here | Exit criterion |
|---|---|---|---|
| 0 | Rules (no code) | Public-SPI CEP, annotation sweep, layering baseline. File the Accord-CDC gap and UDF/JDK risk | CEP vote; every existing seam labeled |
| 1 | **Cut 0** | Everything else uses its loader, registry, and TCK. Pilots: `StartupCheck`, `IAuditLogger`, CEP-42 constraints, memtable/SSTable TCK from `test-latest` | An out-of-tree plugin builds against `cassandra-api` only, passes its TCK in ASF CI, and is refused cleanly by a node that lacks it. Then three node-local slots land as ordinary tickets (UCS controller first) |
| 2 | **Cut 3** and **Cut 5** in parallel | Mostly additive; few hot-path changes; biggest external pain | One external project (Analytics, Debezium, or the Management API) deletes a per-version module. Accord writes appear in the change stream |
| 3 | **Cut 1** | Largest refactor (about 62 call sites). Designed once for front ends, admission, and engines together | CQL native and one second front end run through it. TransactionProvider and the domain guard plug in as slots |
| 3′ | **Cut 4** (parallel with Cut 1) | Independent code area; CEP-36 is already under discussion | Local FS and one out-of-tree object store pass the same TCK |
| 4 | **Cut 2** | Reuses Cut 1's logical request type | Shadow planner + `EXPLAIN` with a second index or vector engine as a slot |

### 6.4 What this saves

- **Core refactors:** about 35 separate seams → 6 cuts. Ideas after that land as slots.
- **CEPs:** one per cut, plus the rules CEP. Slots are tickets.
- **Mixed-version design:** done once in Cut 0's registry. Today each provider kind bumps core TCM (V11–V13 in the prototype).
- **Mindmap coverage:** every mindmap idea maps to a slot in one of the six cuts (§6a). The only non-slot items are core limits that aren't plugin problems.
- **Risk:** Cut 1 is the one large refactor. It should be staged within its single design, landing one stage at a time with the default implementation moved first, so it stays abandon-able.

---

## 6a. Mapping Mick's mindmap onto the boundaries

The mindmap groups ideas into four clusters: query features, transaction features, performance and cost, and known problems and limitations. Its purple-outlined "replacement" ideas (Full SQL, Mongo, Neo4j, Redis, SQL reads) are exactly the kind of idea that should grow as a plugin outside the mainline. The table shows which cut each idea plugs into as a slot. The point of the table is that **no idea on the map requires a seventh cut**.

| Mindmap idea | Cut | Slot |
|---|---|---|
| Full SQL (Calcite/Postgres), Mongo/Mongoose compatibility, Data API, Cypher / Neo4j replacement | 1 | Client front end producing the logical request |
| Redis replacement | 1 + 0 | Client front end + memtable implementation (reloadable memtables) |
| QUIC-based native protocol | 1 | Transport for the CQL front end |
| Thin, maintenance-free CQL drivers | 1 | Negotiated protocol extensions advertised by the front end, so new server features don't need driver releases |
| SAI, CQL OR / NOT, sortable indexes | 2 | Index implementations + planner |
| Astra/HCD vector search | 2 | `VectorEngine` |
| Expressive consistency levels | 1 | Per-request semantic options handled by the execution engine. **Tension:** the catalog says keep `ConsistencyLevel` wire codes closed |
| Accord, interactive transactions, snapshot isolation, fast-path electorates | 1 | Execution engine (TransactionProvider); fast path as a capability |
| MVCC, point-in-time backups | 3 | Change listener + snapshot manifest (MVCC itself is storage-internal) |
| Negotiated authentication (CEP-50) | 0 | Authenticator |
| UCS, adaptive UCS | 0 | UCS controller (**first slot after Cut 0 lands**) |
| Unbounded compaction throughput | 4 | Compaction executor |
| Unlimited off-heap memtables, reloadable memtables | 0 | Memtable implementation (existing SPI, hardened) |
| Object storage (CEP-36), compute–storage separation | 4 | Channel factory + placement + cache tier |
| Direct-representation internode messaging (CEP-57) | 0 | Registered verbs + versioned serializers |
| Cheap quorums and witness replicas | 0 | Replication strategy declaring capabilities (replaces `instanceof` checks) |
| Sidecar, K8ssandra donation | 5 | Sidecar modules + management commands |
| Off-process Spark Analytics, attached Parquet, SQL reads (Arrow, Iceberg, Presto) | 3 (+4 for columnar scan) | Offline reader + exporters; columnar scan as a capability |
| Self-scheduling repairs (CEP-37), mutation-based repair (CEP-45) | 0 + 3 | Repair scheduler / splitter; reconciliation from the change stream |
| Real-time elasticity, unlimited concurrent schema changes | 0 | Placement decisions replayed through TCM; core TCM work otherwise |
| Zero-downtime migrations | 3 + 1 | Change listener + proxying front end |
| In-process upgrades | 0 | Classloader isolation + plugin version vector |
| Unlimited partition sizes, unlimited tombstones, JDK 21 / virtual threads | — | **Not plugin problems.** Core engineering work; the UDF/JDK risk (step 0) is the only link |

What the comparison shows:
- Cut 1 (request pipeline) and Cut 3 (data out) carry most of the map: the replacement APIs on one side, analytics and SQL reads on the other. That is why Cut 3 goes early and Cut 1 is designed for the whole front-end family at once.
- "Expressive consistency levels" conflicts with the catalog's advice to keep `ConsistencyLevel` wire codes closed. Resolve this with Mick before proposing it.

## 7. What not to do

- Do not open a seam without its second implementation and a named consumer. CEP-18 and 13474 show how that ends.
- Do not frame anything as "pluggable storage engine," "pluggable consistency," or "make Cassandra modular." Frame each item as a narrow seam that improves the core on its own.
- Do not add new `ConsistencyLevel` values, new `-Dcassandra.custom_*` hooks, or new class-name-in-schema references. Each makes the platform problem worse.
- Do not let SPIs expose `ColumnFamilyStore`, `Mutation`, `PartitionUpdate`, `CQLStatement`, or `DatabaseDescriptor`.
- Do not move hints, batchlog, gossip, or the failure detector behind SPIs. Nobody is waiting on them.

## 8. Open questions for the project

1. Should the SPI artifact be one jar or one jar per area (storage, query, security, ops)?
2. Which plugins should live in-tree (`contrib/`) versus the ecosystem listing? What CI obligation comes with the listing?
3. Is classloader isolation mandatory for all plugins, or opt-in for those that bring dependencies?
4. Should Experimental features be removable within a major, or only at a major boundary?
5. Will dev@ accept Cut 1 (request pipeline) as one CEP that lands in stages, or insist on separate CEPs for front ends, admission, and execution engines? Separate CEPs would bring back the per-seam cost this roadmap is trying to avoid.
6. For this repo's own work: should TransactionProvider wait for the platform (F3/F4) and admission, or keep advancing on the branch? The team's exit audit already orders lifecycle work before topology and migration ([08] §4).

## 9. Confidence and gaps

- **Verified:** in-tree facts were checked against `cassandra/` at trunk `0691eb3319` (branch-only items are marked). CEP numbers come from the wiki. Ecosystem pain has URLs in [07].
- **Not verified:** the analogous-project mechanisms ([06] §4) were written from memory. The JDK SecurityManager timing, Instaclustr Minotaur, and several unread dev@ threads are listed in each report's open-items section.
- **Not run:** no builds or tests were run for this synthesis. The Accord-CDC gap is from code reading.
- **Mindmap:** cross-checked in §6a. The checkmark meanings come from its legend (5.0 and 6.0); the purple-outlined "replacement" nodes are read here as aspirational, not shipped.
