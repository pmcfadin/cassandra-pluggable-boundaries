# 08 - Prior research in this repo: boundaries proposed, prototyped, and lessons learned

Scope: everything this team already wrote or built about extension boundaries in Cassandra. Sources are `pluggable-consistency/`, `query-optimizer/`, `accord/`, `tcm/`, `docs/plans/`, and the prototype branch in the `cassandra` submodule (`codex/pluggable-consistency-prototype`).

Baselines:
- Research baseline: Cassandra `b029c7bed7`, Accord `fff32de2`.
- Prototype base: `0691eb3319`, Accord `56b7710a`.
- Prototype commit: `5afdb580a7`, published at `pmcfadin/cassandra` (see `pluggable-consistency/README.md` and `REVIEW-HANDOFF.md`).
- Prototype diff: `git diff --stat 0691eb3319 5afdb580a7` reports **104 files, +13,771 / -434**.

Maturity scale:
- **idea**: a sentence or table row.
- **design**: a written contract with invariants.
- **prototype**: code exists, with only focused local tests.
- **tested-in-CI**: the upstream or EKS CI pipeline has run it.

**Nothing in this repo is tested-in-CI.** All evidence comes from local focused suites:
- Phase 1: 51 tests.
- Phase 2: 227 passed / 2 skipped.
- Phase 3.1: 273.
- Phase 3.2: 310.
- Phase 3.4: 325.
- Phase-3 completion: 342 passed / 2 existing skips, plus a 310-step coordinator campaign and a 196-event recovery campaign.

All of these are under `pluggable-consistency/evidence/`.

---

## 1. Boundary catalog

### B1. Statement-level provider dispatch (Phase 1)
- **Shape:** `TransactionProvider { String id(); Set<Capability> capabilities(); TransactionOutcome execute(TransactionPlan, TransactionExecutionContext); enum Capability { STRICT_SERIALIZABLE, READ, WRITE, CONDITIONAL, MULTI_TABLE } }`. Source: `cassandra/src/java/org/apache/cassandra/service/consensus/txn/TransactionProvider.java`, 49 lines.
- **Cut point:** `TransactionStatement.execute`, before `createTxn`. The pre-existing Accord eligibility checks are still in `TransactionStatement` at about lines 327, 547 and 658 (`phase-3/COMPUTE-INTERFACE.md`).
- **Registry:** `TransactionProviderRegistry` and `TransactionProviders` are process-local test state, not a production registry.
- **Maturity:** prototype.
- **Evidence:** `PROTOTYPE-PLAN.md`, `PROTOTYPE-RESULTS.md`, `evidence/prototype/`.
- **Risks the team flagged:**
  - A delegating test provider proves only dispatch, not independence.
  - The capabilities are coarse labels, not constraints.

### B2. Coordinator execution seam: TransactionPlan -> TransactionOutcome (Phase 2)
- **Shape:** an immutable `TransactionPlan` (261 lines) that carries read slots, conditions, operations, references and values. It includes 19 deferred operation kinds (`TransactionOperation`, `TransactionCondition`, `TransactionReadSlot`, `TransactionReference`, `TransactionValue`).
  - The context is `TransactionExecutionContext`.
  - The result is a `TransactionOutcome` algebra.
  - `AccordTransactionCompiler` (258 lines) lowers the plan to an Accord `Txn` behind `AccordTransactionProvider`.
  - `ScalarTransactionProvider` is a test-only, independent, in-memory interpreter.
- **Cut point:** after CQL semantic preparation and binding, before any engine-specific transaction object exists.
- **Maturity:** prototype. The Accord adapter and the independent scalar interpreter both pass the same plan suite.
- **Evidence:** `PHASE-2-PLAN.md`, `PHASE-2-RESULTS.md`, `phase-2/BOUNDARY.md`, `phase-2/PLAN-DESIGN.md`, `phase-2/OUTCOME-DESIGN.md`, `phase-2/EXECUTOR-DESIGN.md`, `phase-2/REVIEW.md`.
- **Risks the team flagged:**
  - The plan still carries Cassandra objects (`SinglePartitionReadCommand`, `PartitionUpdate`, `TableMetadata`), so it is a Cassandra-internal seam, not a wire ABI.
  - Residual coupling remains in table eligibility, preparation helpers, timestamps and storage representation.
  - The outcome has no conditional-success flag.
  - Read-only heap `ByteBuffer`s crashed Cassandra's unsafe comparator, which forced defensive array copies.
  - Accord's key collector is consumed by `build()`, so plan reuse was unsafe.

### B3. Consistency-domain ownership in TCM (Phase 3.1, 3.2, 3.4)
- **Shape:**
  - The `ConsistencyDomains` core TCM component (242 lines) holds `TransactionDomainDescriptor`s (343 lines).
  - Descriptor states are PREPARED, ACTIVE and REVOKED.
  - The transformations are `PrepareTransactionDomain` (atomic create plus a create-closed reservation) and `ActivateTransactionDomain`.
  - `TransactionDomainGuard` is a schema-extension marker.
  - Metadata versions: V11 adds domains, V12 adds the external binding, V13 adds binding kinds ETCD/RATIS and a replicated attest command (`tcm/serialization/Version`, `ClusterMetadata` +75, `MetadataKeys`).
- **Designed record** (`design/ARCHITECTURE.md`): `DomainPolicy{domainId, tableIds, providerId, providerProtocolVersion, semanticProfileId, semanticProfileVersion, storageFormatVersion, generation, metadataEpoch, configuration, migrationState}`.
- **Cut point:** cluster metadata. A table or table-set is owned by exactly one provider generation. The ownership generation is a fence separate from both the TCM epoch and the schema epoch.
- **Maturity:** prototype.
- **Evidence:** `phase-3/OWNERSHIP.md`, `phase-3/IMPLEMENTATION-3.1.md`, `PHASE-3.1-RESULTS.md`, `PHASE-3.2-ETCD-RESULTS.md`, `PHASE-3.4-RATIS-CQL-RESULTS.md`, `phase-3/ratis/REVIEW-3.4-METADATA.md`.
- **Risks the team flagged:**
  - There is no reservation-removal path.
  - Join, leave, move and replace are blocked while reservations exist.
  - Fixed bugs: an empty V11 snapshot omitted its count; pre-TCM nodes escaped the common-version check.
  - Activation is gated only via `eligibleToCommit` and the Directory min-version calculation.
  - A TCM transformation must not perform nondeterministic disk inspection.
  - `ActivateTransactionDomain` hardcodes `"etcd-scalar"` and `"ratis-scalar"` (around lines 83-102), so "the metadata and lifecycle seam is less general" (`PHASE-3.4-RATIS-CQL-RESULTS.md`).
- **Rejected alternative:** the TCM `ExtensionKey` mechanism was rejected because it serializes a Java class name reflectively. A dedicated core component was chosen instead (`phase-3/OWNERSHIP.md`).

### B4. Admission and maintenance gate (TransactionDomainGuard)
- **Shape:** static methods on `TransactionDomainGuard` (239 lines) that read `ClusterMetadata.currentNullable()`. The design calls for five distinct gates (`phase-3/ADMISSION.md`):
  1. Coordinator admission.
  2. Participant arrival.
  3. Durable provider apply.
  4. Authorized recovery.
  5. SSTable and maintenance publication.
- **Cut point:** every path that can change visibility of a domain-owned table. That includes CQL write/read/CAS, internal CQL, hints, batchlog, commitlog replay, read repair, streaming, import, repair, TRUNCATE and flush.
  - In the prototype there are about 62 guard call sites across 31 files outside `txn/`.
  - The largest are ColumnFamilyStore 8, TransactionStatement 7, StreamSession 6, StorageProxy 5, ModificationStatement 4 and BatchlogManager 4.
- **Maturity:** prototype for the paths it covers. The full inventory is still design.
- **Evidence:** `phase-3/ADMISSION.md`, `phase-3/ADMISSION-MATRIX-3.1.md`, `phase-3/ADMISSION-COMPLETION-NOTES.md`, `PHASE-3.1-RESULTS.md`.
- **Risks the team flagged:**
  - `PHASE-3.1-RESULTS.md` (50 files changed) states that no single Cassandra interception point covers ordinary mutations, CAS, replay, direct SSTable publication and topology.
  - `Keyspace.applyInternal` appends to the commitlog in `beginWrite` *before* `validateSafeToExecuteNonTransactionally` runs (Keyspace.java:549-552).
  - `PotentialTxnConflicts.ALLOW` short-circuits the check, and ALLOW is not ownership evidence.
  - The `ReadCommand` check is conditional on DISALLOW.
  - `CassandraStreamReceiver.finished -> cfs.addSSTables` bypasses the mutation path.
  - Internal CQL bypasses `StorageProxy`.
  - The Mutation and ReadCommand wire formats carry only a conflict bit, not domain or generation.
  - TRUNCATE reached replicas before it was rejected (fixed).
  - A forbidden commitlog record stops replay (open).
  - `Tracker.replaceFlushed` is outside the guards (open).

### B5. External provider binding and readiness attestation (Phase 3.2, 3.4)
- **Shape:** `ExternalTransactionDomainBinding` (226 lines) records the binding kind, the external identity and an attestation. Readiness is a third fact, distinct from the epoch and the generation.
- **Cut point:** between Cassandra TCM and an external consensus system (etcd, Ratis).
- **Maturity:** prototype.
- **Evidence:** `PHASE-3.2-ETCD-RESULTS.md`, `phase-3/etcd/IMPLEMENTATION.md`, `PHASE-3.4-RATIS-CQL-RESULTS.md`.
- **Risks the team flagged:**
  - Readiness is a *trusted assertion*, not verified.
  - Provisioning is not atomic across the two systems, so the activation gap shows up as unavailable time. Production reconciliation is still needed.
  - Fixed: an activation race paired an old image with a new revision.
  - Fixed: registry precedence let a local assignment override TCM.
  - Fixed: projection identity and field-order bugs.

### B6. Durable participant seam: submit/resolve envelope plus outcome algebra (Phase 3.2-3.4)
- **Shape:** `submit(DomainRequest) -> Submission` and `resolve(DomainRef, RequestId) -> Decision`.
  - Envelope fields: `RequestId(128-bit), AttemptId, DomainId, ProviderId/ProfileId, OwnershipGeneration, SchemaRef, Route, DeadlineBudget, PayloadVersion, Payload`.
  - Outcomes: COMMITTED, REJECTED, PENDING, UNKNOWN, STALE_GENERATION, UNAVAILABLE, NOT_READY.
- **Cut point:** between the coordinator and the provider's durable state machine. It is a serialized, versioned envelope, not Java objects.
- **Maturity:** prototype, but **only inside test adapters**. The common `TransactionProvider` interface still exposes only `execute`, and durable identity and lookup are absent from the core contract.
- **Evidence:** `phase-3/COMPUTE-INTERFACE.md`, `phase-3/REQUEST-CONTRACT.md`, `phase-3/REPLAY-CONTRACT-REVIEW.md`, `phase-3/ratis/RATIS-CONTRACT.md`, `phase-3/ratis/REVIEW-3.4-TRANSPORT.md`.
- **Rules the team set:**
  - No generic prepare/commit verbs, because that would make a shallow seam.
  - Never serialize live Java objects or `nanoTime`.
  - Never persist enum ordinals.
  - An unknown version must not fall through to Accord or to an ordinary mutation.
  - Absolute assignments are not deduplication: replaying `SET 7` after `SET 9` restores stale data.

### B7. Provider-owned durability applied to Cassandra storage (Phase 3.3)
- **Shape:** an Apache Ratis 3.3.0 state machine, running in three processes, applies into a Cassandra storage fixture. The recovery authority is the Raft log plus snapshots, and Cassandra commitlog durability is disabled for the domain.
- **Cut point:** a second boundary, separate from B2. B2 is coordinator execution; B7 is durable participant application.
- **Maturity:** prototype (fixture).
- **Evidence:** `PHASE-3.3-PLAN.md`, `PHASE-3.3-RATIS-RESULTS.md`, `phase-3/DURABILITY.md`, `phase-3/ratis/CASSANDRA-STORAGE-NOTES.md`, `phase-3/ratis/IMPLEMENTATION.md`.
- **Risks the team flagged:**
  - The whole-image blob is not a general row layout. An incremental storage contract with watermarks is still needed.
  - Restart must not reformat.
  - A responding admin endpoint is not proof of a functioning quorum.
  - A snapshot does not force a transfer.
  - Inherited lifecycle methods are no-ops.
  - Checksum publication ordering matters.

### B8. Client retry envelope (Phase 3 completion)
- **Shape:** an HTTP envelope with client states retained, absent, unknown and rejected. Receipt capacity is 128 with no expiry. The demonstration covers cold coordinator replacement and a generation fence from 1 to 2.
- **Cut point:** the client and coordinator boundary for idempotent retry.
- **Maturity:** prototype. The six new files are test code; "the common provider interface did not need to change".
- **Evidence:** `PHASE-3-COMPLETION-PLAN.md`, `PHASE-3-COMPLETION-RESULTS.md`, `phase-3/COMPLETION-DEMO.md`, `phase-3/PHASE-3-EXIT-AUDIT.md`.
- **Risks the team flagged:**
  - A timeout is not an abort.
  - Receipts must not be evicted without a tombstone, and receipt garbage collection and retention are undefined.
  - Native-protocol custom payloads are discarded by `QueryProcessor.process`, so there is no in-band carrier.

### B9. Provider lifecycle and engine contract (design only)
- **Shape:** `ProviderFactory.describe/recover` plus `Engine.validate/execute/resolve/advancePlacement/maintain/quiesce/checkpoint/importCheckpoint/closeAfterDurableStop`. The document explicitly says this is "not a Java ABI".
  - Completion states: success, condition not met, definite nonexecution, unavailable, unsupported, retryable metadata lag, unknown.
  - Every message carries domain, provider, generation, epoch and operation ID.
  - Providers are bundled and installed at startup, and unload is forbidden while dependencies exist.
  - `design/ARCHITECTURE.md` includes a responsibility table of what Cassandra owns versus what the provider owns.
- **Maturity:** design.
- **Evidence:** `design/ARCHITECTURE.md`. The exit audit lists the lifecycle verbs still to build: load, validate, activate, refresh, fence, recover, close. It also lists budgets and backpressure, errors, auth, and retention.
- **Risks the team flagged:**
  - R15: the interface may still assume Accord state, so a second-provider review is required before the SPI is frozen.
  - R16: the support matrix grows.

### B10. Paused-domain migration (design only)
- **Shape:** a state machine: SourceActive -> Prepared -> SourceFenced -> SourceResolved -> Checkpointed -> TargetReady -> TargetActive -> SourceRetired.
- **Cut point:** a whole-domain fence. Per-request engine choice on the same data is explicitly rejected.
- **Maturity:** design.
- **Evidence:** `design/ARCHITECTURE.md`, `RESEARCH-REPORT.md` (scope ladder), `design/ROADMAP-AND-RISKS.md`.
- **Risks the team flagged:**
  - R3: split authority.
  - R9: mixed versions.
  - Migration availability is an open decision.

### B11. Query frontend seam (design)
- **Shape:** four contracts (frontend, planner/optimizer, query-execution backend, transaction provider) that share catalog, read semantics and admission facts. The shared planning seam sits after semantic preparation and binding, and before concrete `ReadCommand`s or provider compilation.
- **Maturity:** design.
- **Evidence:** `pluggable-consistency/query-planning/ASSESSMENT.md`, `01-FRONTEND.md`, `04-DESIGN-CHECK.md`.
- **Risks the team flagged:**
  - `QueryHandler` is chosen statically in `ClientState:122`; it is not a live registry.
  - `BatchMessage:181` bypasses it via `QueryProcessor.parseAndPrepare`.
  - Paging state has no plan ID, so reoptimization cannot change an in-flight retry.

### B12. Optimizer plan/statistics contract (design plus native prototype tests)
- **Shape:** the value objects are `LogicalRead`, `BoundContext`, `PhysicalPlan` (with traits), `PlanSelection`, `ExecutionFeedback`, `Estimate` (with provenance), `ReconciliationTrait` and `PageContract`. The design is three stages: logical -> physical -> lowered into the *existing* `ReadCommand`.
- **Maturity:** design plus prototype tests: 18 native tests and 28 equal-result plan arms. No end-to-end speedup has been established.
- **Evidence:** `query-optimizer/ASSESSMENT.md`, `query-optimizer/design/plan-and-statistics-contract.md`, `query-optimizer/research/cassandra-query-optimizer-assessment.md`, `query-optimizer/research/cross-engine-paper-notes.md`, `pluggable-consistency/query-planning/02-OPTIMIZER.md`.
- **Rules the team set:**
  - Eligibility precedes cost.
  - Reconciliation is a legality trait, not a cost.
  - Each paging contract has exactly one physical shape.
  - The SAI `Long.MIN/MAX` sentinel is a preference, not a cardinality.
  - A learned estimator is a swappable component behind the contract and never defines which plans are legal.

### B13. Query-execution backend / DataFusion scan contract (design)
- **Shape:** a DataFusion `TableProvider` scan with exact, inexact or unsupported pushdown, and live, export or transactional-snapshot modes.
- **Maturity:** design.
- **Evidence:** `pluggable-consistency/query-planning/ANALYTICS-ARCHITECTURE.md`, `03-CONSISTENCY.md`.
- **Risks the team flagged:**
  - Analytics must not read around domain fences.
  - Live reads vs exports first is an open user decision.
  - Calcite vs DataFusion planner ownership is open.

### B14. Existing index/query hooks reused (inventory)
- **Shape:** `Index.QueryPlan`, `SecondaryIndexManager.getBestIndexQueryPlanFor`, `IndexHints`/`RowFilter`, SAI `QueryController`, and `QueryContext` metrics.
- **Maturity:** these exist upstream; the repo only inventoried them. CEP-39 is "Under Discussion".
- **Evidence:** `query-optimizer/research/cep-39-and-reusable-hooks.md`.

---

## 2. Lessons learned: what made boundaries hard (evidence-backed)

1. **There is no single choke point for writes or visibility.** The guard needed about 62 call sites in 31 files. Bypass paths include streaming `addSSTables`, internal CQL, commitlog replay, hints, batchlog, read repair, import and `Tracker.replaceFlushed`.
   - Plan for a gate at every visibility-changing path, plus an inventory test.
   - Sources: `phase-3/ADMISSION.md`, `PHASE-3.1-RESULTS.md`.
2. **Check before you make anything durable.** `Keyspace.applyInternal` appends to the commitlog before the safety check runs. A gate placed after the first durable side effect is not a gate.
   - Source: `phase-3/ADMISSION.md`, Keyspace.java:549-552.
3. **A permission bit is not ownership.** `PotentialTxnConflicts.ALLOW` and the one-bit conflict flag in Mutation and ReadCommand cannot carry domain or generation.
   - Messages must carry domain, provider, generation, epoch and operation ID.
   - Sources: `phase-3/ADMISSION.md`, `design/ARCHITECTURE.md`.
4. **Ownership needs a fence that is not the TCM epoch.** The ownership generation, the schema epoch and readiness are three independent facts. Collapsing them caused the etcd activation race (an old image paired with a new revision).
   - Source: `PHASE-3.2-ETCD-RESULTS.md`.
5. **Mixed-version safety is a first-class contract.** The team made these rules:
   - Never persist enum ordinals.
   - An unknown version must fail closed and never fall through.
   - Pre-TCM nodes must not escape the min-version gate.
   - Corroborating upstream evidence:
     - Accord #10: `TransactionalMode` is serialized by ordinal into the TCM log.
     - TCM SEQ-1 and SER-2: raw `.ordinal()` is used in `BootstrapAndJoin:542`, `Move:650`, `NodeState` and `TableParams:690`.
     - TCM SER-1: `Transformation.Kind.fromId()` is an unguarded array index on the boot path.
   - Sources: `phase-3/COMPUTE-INTERFACE.md`, `PHASE-3.1-RESULTS.md`, `accord/ACCORD-ASSESSMENT.md`, `tcm/TCM-ASSESSMENT.md`.
6. **Static singletons leak authority.** Accord #8: `TxnQuery` reads node-local `ClusterMetadata.current()` instead of execution-epoch metadata, which makes replicas nondeterministic.
   - The prototype's own `TransactionDomainGuard` (static, `currentNullable()`) and `AccordTransactionCompiler` (`current()`) repeat that pattern.
   - Pass the metadata snapshot explicitly.
   - Sources: `accord/ACCORD-ASSESSMENT.md`, the prototype source.
7. **Accord leaks everywhere.** 317 Java files reference Accord symbols, and 145 of them are outside `service/accord`.
   - `StorageProxy.cas` switches between Paxos V1, Paxos V2 and Accord (lines 388-429).
   - `TransactionStatement` builds the Accord `Txn` directly (585-609).
   - `IAccordService` exposes `Txn`, `TxnId`, `Ranges` and command stores (79-199).
   - `BootstrapAndJoin` waits for Accord readiness (382-405).
   - Any SPI must be reviewed against a second, non-Accord provider (R15).
   - Sources: `RESEARCH-REPORT.md`, `evidence/baseline-and-coupling.json`.
8. **TCM extension points are not plugin-friendly.**
   - `ExtensionKey` serializes reflective class names.
   - The alternative, core metadata versions (V11, V12, V13), means every new provider kind bumped a core version and edited `ActivateTransactionDomain`.
   - Transformations must be deterministic, with no disk inspection.
   - The local `SystemKeyspaceStorage` append tolerates failure, so it cannot be the authority.
   - Sources: `phase-3/OWNERSHIP.md`, `PHASE-3.4-RATIS-CQL-RESULTS.md`.
9. **Do not let Java objects or clocks cross the seam.** Read-only heap buffers crashed the unsafe comparator, and Accord's key collector is consumed by `build()`.
   - Serialize versioned values and never `nanoTime`.
   - Sources: `phase-2/REVIEW.md`, `phase-3/COMPUTE-INTERFACE.md`.
10. **Retry identity is its own contract.**
    - A timeout is not an abort.
    - Absolute writes are not idempotent across reordering.
    - Receipts need tombstoned retention.
    - Ownership and readiness, persistence, retry identity and result projection are four separate contracts.
    - Sources: `PHASE-3.2-ETCD-RESULTS.md`, `PHASE-3-COMPLETION-RESULTS.md`.
11. **A delegating adapter proves nothing.** Independence was shown only once a non-Accord interpreter (scalar, then etcd, then Ratis) passed the same suite.
    - Sources: `PROTOTYPE-RESULTS.md`, `PHASE-2-RESULTS.md`.
12. **Cross-system activation is not atomic.** The gap between TCM and the external system is unavailable time. Readiness must be attested and reconciled, not asserted.
    - Sources: `PHASE-3.2-ETCD-RESULTS.md`, `PHASE-3.4-RATIS-CQL-RESULTS.md`.
13. **Liveness facts lie.** A responding admin endpoint does not mean a working quorum, and restart must not reformat.
    - Source: `PHASE-3.3-RATIS-RESULTS.md`.
14. **Failure paths are the untested surface.**
    - Accord: #1 journal compaction loses dropped-table state; #2 a save failure is reported as durable; `CommandsForKey` is unversioned; migration-race dtests are `@Ignore`d; the burn test is an empty stub.
    - TCM: the only critical is SCHEMA-DIVERGENCE-1, where killing the `cms initialize` initiator between PreInitialize and Initialize leaves the cluster unrecoverable. No process-kill tests existed before Track D.
    - A boundary's acceptance suite must include kill and restart.
    - Sources: `accord/ACCORD-ASSESSMENT.md`, `tcm/TCM-ASSESSMENT.md`.
15. **Test-tree rule.** Accord reproducers must live in Cassandra's `test/` tree, never in `modules/accord`.
    - `ci-test` only searches `test/`, and the build passes `-x test` to the Accord Gradle build.
    - "Not reachable from Cassandra's test tree" is a valid finding.
    - Source: `docs/plans/2026-08-03-accord-tcm-verification-campaign-design.md`.
16. **Static frontend selection.** `QueryHandler` is fixed at startup, and `BatchMessage` bypasses it. Paging has no plan identity.
    - Source: `query-planning/ASSESSMENT.md`.
17. **Topology coupling.** Domain reservations had to block join, leave, move and replace. `DataPlacements` are not ownership.
    - Sources: `PHASE-3.1-RESULTS.md`, `phase-3/OWNERSHIP.md`.

## 3. Boundary design principles the team already articulated

- **Separate the semantics.** Consistency level, isolation, agreement and atomic commit are different things. So are deployment-time, schema-time, coexistence and live-migration selection (`RESEARCH-BRIEF.md`).
- **Climb the scope ladder in order:**
  1. Startup selection.
  2. New empty table.
  3. Paused-domain migration.
  - Per-request engine choice on shared data is rejected, and hot-load is excluded (`RESEARCH-REPORT.md`).
- **One owner per domain generation.** Every message carries the fencing identity (`design/ARCHITECTURE.md`).
- **Layered identity.** Provider ID, profile ID and version, protocol version and storage version are separate compatibility axes. The provider ID is permanent and is not a class name.
- **Capabilities are structured constraints, not labels.** Eligibility is checked before cost; reconciliation is a legality trait (ARCHITECTURE; optimizer contract).
- **Deep seams over shallow verbs.** Submit/resolve over a versioned envelope, with no generic prepare/commit (`phase-3/COMPUTE-INTERFACE.md`).
- **Fail closed on unknowns.** Unknown version, provider or generation produces REJECTED, STALE_GENERATION or NOT_READY, never a fallback.
- **The outcome algebra must distinguish unknown from failed** (`phase-2/OUTCOME-DESIGN.md`).
- **Bundled providers, installed at startup.** No unload while dependencies exist.
- **Lower into existing execution.** The optimizer lowers into `ReadCommand`; analytics never read around fences.
- **The interface is not frozen until a second provider review is done** (R15), with stop conditions: if the "generic" interface keeps requiring Accord internals, stop (`design/ROADMAP-AND-RISKS.md`).
- **Contain the support matrix** (R16). Stage gates 0-6 have explicit exit criteria, and none are satisfied yet.

## 4. Contradictions and open decisions

**Contradictions in the prior work:**
- **Engine contract vs shipped interface.** `ARCHITECTURE.md` specifies about ten engine verbs, including resolve, quiesce and checkpoint. The prototype ships one method, `execute`.
  - Submit/resolve, durable request identity and lookup exist only in test adapters.
  - The claim that "the common interface did not need to change" (Phase-3 completion) holds only because durability lives outside it.
- **"Provider ID is permanent, not a class name" vs core TCM code.** Core TCM code hardcodes `etcd-scalar`, `ratis-scalar` and the ETCD/RATIS binding kinds. Adding a provider means a core metadata version bump, which is not plugin-like.
  - Avoiding ExtensionKey fixed reflection but gave up open extensibility. There is no stable plugin-registered metadata payload yet.
- **"Structured capabilities" vs the shipped enum.** The design calls for structured capabilities, but the prototype ships a five-value `Capability` enum.
- **STRICT_SERIALIZABLE vs the scalar providers.** STRICT_SERIALIZABLE is required for `BEGIN TRANSACTION`, yet the scalar providers admit ONE/SERIAL through an "experimental" consistency mapping.
- **"Verifiable attestation" vs the prototype.** The design wants verifiable attestation; the prototype records readiness as a trusted assertion.
- **"Bounded retention" vs the receipt store.** Bounded retention is required, but the receipt store has 128 slots and no expiry or GC.
- **Generality vs the storage layout.** The provider-owned storage is a whole-image blob per domain, not a granular row contract with watermarks.
- **Explicit-context principle vs the guard and compiler.** The guard and compiler use static `ClusterMetadata.current*()`, which is the same pattern as Accord bug #8.
- **Single gate vs the 62 call sites.** The 62 call sites contradict the goal of one clean admission seam. The team concluded that five distinct gates are inherent.
- **Reservations block topology changes**, yet there is no removal path, so a stuck reservation freezes cluster operations.
- **Commitlog replay** stops on a forbidden record instead of quarantining it.

**Open decisions** (`design/ROADMAP-AND-RISKS.md`, `query-planning/`):
1. The primary use case, which decides between RAMP and Raft profiles.
2. Granularity: a whole table vs table-sets.
3. The RAMP replication backend.
4. Raft group layout.
5. Compatibility promises.
6. Migration availability (paused vs online).
7. Whether table metadata alone is sufficient opt-in for weaker profiles.
8. Live reads vs exports first for analytics.
9. Calcite vs DataFusion planner ownership.
10. Whether a production native-protocol carrier for retry identity is needed, given that custom payloads are dropped.
11. The order of the remaining work, which the exit audit gives as lifecycle, then budgets, errors and auth, then incremental storage, and only then topology, migration and RAMP (`phase-3/PHASE-3-EXIT-AUDIT.md`).
