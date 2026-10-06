# 03 — Cluster distribution, membership, replication, consistency, data movement

Scope: partitioners, location/proximity, replication strategies, token allocation, TCM, gossip, failure detection, coordinator read/write paths, CL, speculative retry, read repair, Paxos, Accord, hints, batchlog, repair, streaming, internode messaging, CEP-45.

**Source baseline.** `cassandra/` HEAD `5afdb580a7` on `codex/pluggable-consistency-prototype`. The branch is one commit on top of trunk merge `0691eb3319`, which is the merge-base with local `trunk`. `git diff 0691eb3319...HEAD`: 104 files, +13,771/−434. Paths are relative to `src/java/org/apache/cassandra/` unless noted. **[BRANCH]** marks anything that exists only on the prototype branch. Everything else is trunk at `0691eb3319`. CEP-45 code lives only on `origin/cep-45-mutation-tracking` (tip `2af323f09d`, 2026-09-17), which is not merged into this tree; it is marked **[CEP-45 BRANCH]**.

The team's consistency research is in `pluggable-consistency/`, especially `reports/03-execution-seams.md`, `05-metadata-configuration-wire.md`, `06-topology-repair-streaming.md` and `11-accord-library-and-spi.md`. This document cites that work and does not summarize it again.

Legend for status: `stable-public` (documented config/schema surface, third parties can rely on it), `internal-seam` (an interface exists, but only in-tree code selects or constructs it), `partial` (a pluggable entry point exists, but core code special-cases in-tree implementations), `proposed` (no seam yet).

---

## Summary table

| # | Boundary | Status | Grade | One-line why |
|---|---|---|---|---|
| 1 | `IPartitioner` | stable-public (frozen) | C | Clean interface. It cannot change for the life of a cluster, it is serialized by class name in TCM and schema, and Accord, MerkleTree and SASI special-case in-tree classes. |
| 2 | `IEndpointSnitch` (legacy) | stable-public, deprecated | B− | Long-lived, has external implementations. Now wrapped by `SnitchAdapter`. Mixes four concerns. |
| 3 | `NodeProximity` / `InitialLocationProvider` / `NodeAddressConfig` | stable-public (new) | A− | A good split. Location is sourced from TCM after registration, so a provider only affects bootstrap. |
| 4 | Dynamic snitch | internal (boolean) | C | A hard-coded decorator class. It cannot be replaced, only toggled. |
| 5 | `AbstractReplicationStrategy` (custom `class` in CQL) | partial | C+ | Loadable by FQCN from DDL, but 31 `instanceof NetworkTopologyStrategy` checks gate CL, token allocation, leave and streaming. Every node must ship the class. |
| 6 | `PlacementProvider` (TCM placement) | internal-seam | C | A clean interface. It is hard-wired to `new UniformRangePlacement()` in about 9 places. |
| 7 | Token allocation | internal | D | Supports only NTS or Simple. It has no factory or config. |
| 8 | TCM `Processor` / `LogStorage` (metadata consensus backend) | internal-seam | C− | `Processor` is a two-method interface, but the selection is an `if` on a test sysprop. The Paxos log is hard-coded. |
| 9 | TCM `CustomTransformation` + `ExtensionKey/Value` | internal-seam (test-grade) | C | It is a real extension hook in the metadata log. Registration is a static map with no lifecycle, and an unknown extension asserts on replay. |
| 10 | Gossip (`IGossiper`, `IEndpointStateChangeSubscriber`) | internal-seam (residual) | D+ | Post-TCM it only carries liveness and advisory state. `Gossiper.instance` has 151 references. |
| 11 | `IFailureDetector` (`failure_detector`) | stable-public (thin) | C+ | A config key exists. `FailureDetector.instance` is a static final, and 29 call sites bypass the interface. |
| 12 | `SeedProvider` (`seed_provider`) | stable-public | B+ | The cleanest ParameterizedClass in the area. Widely implemented externally (inference). |
| 13 | `ConsistencyLevel` | closed enum | F (as a seam) | Wire-coded in the native protocol, with NTS-specific logic inside the enum. |
| 14 | Coordinator write path (`StorageProxy`, `AbstractWriteResponseHandler`) | partial | D | The one hook is the overridable `getWriteResponseHandler` on the strategy. The rest is static `StorageProxy`. |
| 15 | Coordinator read path (`AbstractReadExecutor`, `ReadCoordinator`) | internal-seam | C− | The `ReadCoordinator` interface exists for the Accord and eventually-consistent split, with a static factory. |
| 16 | `SpeculativeRetryPolicy` (`speculative_retry`) | stable-public (closed) | C | A clean interface, but `fromString` is a closed if-chain over five built-ins. |
| 17 | `ReadRepairStrategy` (`read_repair`) | stable-public (closed) | C | A `ReadRepair.Factory` interface exists, but the option is a two-value enum. |
| 18 | Paxos v1/v2 (`paxos_variant`) | stable-public (closed) | C | A global yaml enum. Routing happens in `ConsensusRequestRouter` across a three-value enum. |
| 19 | Accord library SPI (`accord.api.*`) | stable seam (library-facing) | B+ | The best boundary in the area. Cassandra implements Accord's `Agent`/`DataStore`/`Journal`/`MessageSink`/`TopologyService`, and accord-maelstrom proves a second host exists. |
| 20 | `transactional_mode` + consensus migration | stable-public (closed) | C | A per-table option, but the migration target is the enum `{paxos, accord}`. |
| 21 | **[BRANCH]** `TransactionProvider` SPI and durable domains | proposed / prototype | B− (as a prototype) | The right shape and TCM-durable. The registry is static and hard-coded, and admission had to be bolted onto 25 files. |
| 22 | Hinted handoff | internal | D | Final classes, a static mutable `instance`, and no SPI. |
| 23 | Batchlog | internal | D | `BatchlogManager.instance` static final, with no SPI. |
| 24 | Repair: `KeyspaceRepairManager`/`TableRepairManager`/`IValidationManager`/`SharedContext` | internal-seam | C | The interfaces exist (storage-engine and simulator seams). They are constructed with `new` and the wire format is MerkleTree-specific. |
| 25 | Auto-repair `IAutoRepairTokenRangeSplitter` | stable-public | B | A ParameterizedClass with a documented constructor contract. |
| 26 | Streaming `TableStreamManager`/`IncomingStream`/`OutgoingStream`/`StreamReceiver` | internal-seam | C+ | A good per-table abstraction from the pluggable-storage work. It is constructed with `new` in CFS and the `StreamOperation` enum is closed. |
| 27 | `StreamingChannel.Factory` (transport) and `StreamHook` | internal-seam / sysprop | C | `unsafeSet` exists for tests only. `cassandra.stream_hook` is an observer hook. |
| 28 | Internode messaging (`Verb`, `IVerbHandler`, `MessageDelivery`, sinks) | internal-seam | D+ | The `Verb` enum has a CUSTOM kind and a reflective `unsafeSetHandler`. The inbound and outbound sinks accept filters. The transport is not replaceable. |
| 29 | **[CEP-45 BRANCH]** Mutation tracking (`replication_type`) | partial (unmerged) | C (provisional) | A keyspace-level enum `{untracked, tracked}` gated at TCM V9. It is a second replication mode, not a plugin point. |

---

## 1. Placement and topology

### 1.1 IPartitioner — `stable-public` (frozen per cluster), grade **C**
- **Entry:** `partitioner:` in yaml (`config/Config.java:118`), overridable by sysprop `cassandra.partitioner` (`config/CassandraRelevantProperties.java:462`). Loaded in `DatabaseDescriptor.applyPartitioner` (`config/DatabaseDescriptor.java:1851-1866`) through `FBUtilities.newPartitioner` (`utils/FBUtilities.java:666`). A short name is resolved to the `org.apache.cassandra.dht.` package, and `LocalPartitioner` is special-cased with a comparator.
- **Interface:** `dht/IPartitioner.java:37`. It covers about 25 methods: token generation, splitting, ownership, and validators. It also has a static `IPartitioner.global()` (`:39`).
- **Implementations (in-tree):** `Murmur3Partitioner`, `RandomPartitioner`, `ByteOrderedPartitioner`, `OrderPreservingPartitioner`, `LocalPartitioner`, `ReversedLongLocalPartitioner`, and `MetaStrategy.partitioner`.
- **External implementations:** none known in common use (inference). The interface is public but rarely implemented outside the tree.
- **Coupling and leakage:**
  - The partitioner class name is serialized into every `ClusterMetadata` snapshot (`tcm/ClusterMetadata.java:1398,1406`, read back at `:1445,1457`) and into each `TableMetadata` (`schema/TableMetadata.java:2138,2174`). A custom class therefore has to be present on every node, on offline tools, and on future versions.
  - Accord capability is a default-false flag (`IPartitioner.java:196-203`), and only Murmur3, Random and ByteOrdered implement it (`dht/Murmur3Partitioner.java:351`, `RandomPartitioner.java:418`, `ByteOrderedPartitioner.java:364`). A custom partitioner silently cannot use Accord.
  - `MerkleTree` uses an off-heap path only for Murmur3 and Random (`utils/MerkleTree.java:740`). SASI requires Murmur3 (`index/sasi/SASIIndex.java:154`).
  - 79 references to `DatabaseDescriptor.getPartitioner()` or `IPartitioner.global()` assume one global partitioner, even though `TableMetadata.partitioner` exists per table.
- **Compatibility:** immutable for the life of the cluster. Mixed-version is safe only because the class never changes.
- **Recommendation:** keep the interface frozen and don't invite implementations. Next steps:
  - Make capability flags (`accordSupported`, off-heap Merkle support, order preservation) an explicit `PartitionerCapabilities` record that callers check. Today some callers check flags and others use `instanceof`.
  - Replace the global accessor with `TableMetadata.partitioner` wherever per-table partitioners already apply (system and index tables).

### 1.2 Snitch → NodeProximity / InitialLocationProvider / NodeAddressConfig — grades **B−** (legacy) and **A−** (new)
- **Entry:** either `endpoint_snitch` (legacy, deprecated at load with an INFO log) or both `node_proximity` and `initial_location_provider`, optionally with `addresses_config` (`config/Config.java:510,518-519`). Setting both or neither is rejected (`config/DatabaseDescriptor.java:1792-1797`). Classes are loaded with `FBUtilities.construct` through `createEndpointSnitch`, `createProximityImpl`, `createInitialLocationProvider` and `createAddressConfig` (`DatabaseDescriptor.java:2061-2091`). Short names resolve to `org.apache.cassandra.locator.`.
- **Interfaces:**
  - `locator/IEndpointSnitch.java:34`
  - `locator/NodeProximity.java:25`: `sortedByProximity`, `compareEndpoints`, `isWorthMergingForRangeQuery`
  - `locator/InitialLocationProvider.java:24`: `initialLocation()` and `validate(ClusterMetadata)`
  - `locator/NodeAddressConfig.java:23`
  - Legacy bridge: `locator/SnitchAdapter.java:30`, which implements all three new interfaces from one `IEndpointSnitch`.
  - Runtime location lookup is `locator/Locator.java:40`. It reads from `RegistrationStatus` and TCM's `Directory`.
- **Implementations:** in-tree cloud providers (Ec2, GCP, Azure, Alibaba, Cloudstack) as both a `*Snitch` and a `*LocationProvider`, plus `RackDCFileLocationProvider`, `SimpleLocationProvider`, `NetworkTopologyProximity`, `NoOpProximity` and `BaseProximity`. External: operator- and vendor-specific snitches are widely reported (inference, unverified).
- **Coupling:**
  - The key improvement is that location is sourced from `ClusterMetadata` after registration (`InitialLocationProvider.java` javadoc). A provider can no longer diverge across nodes at runtime, which removes the classic snitch-mismatch failure.
  - `applySnitch` is "definitely not safe for tools + clients — implicitly instantiates StorageService" (`DatabaseDescriptor.java:1789`).
  - 18 static `DatabaseDescriptor.getNodeProximity()`/`getEndpointSnitch()` call sites.
- **Compatibility:** legacy snitches keep working through the adapter. Swapping proximity is node-local and safe. Swapping location providers after registration has no effect on placement.
- **Recommendation:**
  - Promote this split as the documented model and publish a removal timeline for `endpoint_snitch`.
  - Accept `ParameterizedClass` for proximity and location providers. They currently take only a bare string, so there is no way to pass options.
  - Make the dynamic snitch a composable `NodeProximity` decorator chain. See 1.3.

### 1.3 Dynamic snitch — `internal`, grade **C**
- **Entry:** `dynamic_snitch: true|false` plus three tuning keys (`config/Config.java:511-516`). The wiring is `nodeProximity = conf.dynamic_snitch ? new DynamicEndpointSnitch(proximity) : proximity` (`DatabaseDescriptor.java:1823`). The class is `locator/DynamicEndpointSnitch.java:58`, and it subscribes to `LatencySubscribers` on `MessagingService`.
- **Gap:** it cannot be replaced, for example with a latency-plus-load-aware or zone-cost-aware sorter.
- **Recommendation:** add `node_proximity_decorators: [ParameterizedClass...]`. Keep the dynamic snitch as the default decorator. The change is local, with no wire or mixed-version impact.

### 1.4 AbstractReplicationStrategy — `partial`, grade **C+**
- **Entry:** CQL `replication = {'class': '...'}`. `AbstractReplicationStrategy.getClass` (`locator/AbstractReplicationStrategy.java:336-349`) resolves short names to `org.apache.cassandra.locator.` and rejects `OldNetworkTopologyStrategy`. The class is stored as `ReplicationParams.klass` (`schema/ReplicationParams.java:58,209`). Options are validated through `validateOptions`, `recognizedOptions` and `validateExpectedOptions` (`:223,243,385`).
- **Abstract surface:**
  - `calculateNaturalReplicas(Token, ClusterMetadata)` (`:81`)
  - `calculateDataPlacement(Epoch, List<Range>, ClusterMetadata)` (`:83`)
  - `getReplicationFactor()` (`:149`)
  - an overridable `getWriteResponseHandler` (`:85-141`)
- **In-tree implementations:** `SimpleStrategy` (`locator/SimpleStrategy.java:54`) and `NetworkTopologyStrategy` (`:73`). The `SystemStrategy` base class covers `LocalStrategy` (`:34`), `RemoteStrategy` (`:23`, an empty subclass of Local) and `MetaStrategy` (`:44`, for the CMS keyspace). The CMS membership placement `CMSPlacementStrategy` (`:48`) is separate and not an ARS.
- **External implementations:** vendor "Everywhere"-style strategies are reported (inference, unverified).
- **Coupling:**
  - **31 `instanceof NetworkTopologyStrategy` checks**, including:
    - `db/ConsistencyLevel.java:103,115,164,299`: LOCAL_/EACH_QUORUM semantics live inside the CL enum
    - `locator/ReplicaPlans.java:118,176,829`
    - `AbstractReplicationStrategy.java:108`: EACH_QUORUM uses the DC-sync handler only for NTS
    - `service/DatacenterSyncWriteResponseHandler.java:59`
    - `tcm/transformations/PrepareLeave.java:147`
    - `dht/RangeStreamer.java:450`
    - `dht/tokenallocator/TokenAllocation.java:256-260`
  - A custom DC-aware strategy therefore gets no DC-local CL semantics, no token allocation, and degraded leave and streaming behavior.
  - TCM executes `AlterSchema` and placement calculation on every node during log replay, so the class must exist on every node. A node missing the class fails to replay the log and cannot start. This is stricter than pre-TCM schema behavior (inference from replay design, see `TransactionalClusterMetadata.md`).
  - Placements are materialized into `DataPlacements` and serialized in snapshots (`tcm/ClusterMetadata.java:1414-1415`). The output is data, but recomputation still needs code.
- **Compatibility:** no versioning of strategy semantics. Changing `calculateDataPlacement` output between releases changes placement on replay. That is a correctness hazard, but it is mitigated because snapshots carry placements.
- **Recommendation:**
  - Introduce a `DatacenterAwareReplication` capability interface (methods for per-DC RF and per-DC replica sets) and replace the `instanceof NetworkTopologyStrategy` checks with capability checks. This is mechanical, in-tree only, and has no wire impact.
  - Then document the ARS contract: determinism, primary-range ordering (`:69-79`), and placement stability across versions. Optionally add a `strategyVersion` to `ReplicationParams` behind a TCM `Version` bump.

### 1.5 PlacementProvider — `internal-seam`, grade **C**
- **Interface:** `tcm/ownership/PlacementProvider.java:31` (`calculatePlacements`, plus plan and transition methods). The only implementation is `UniformRangePlacement` (`tcm/ownership/UniformRangePlacement.java:83`), constructed with `new` in `ClusterMetadataService.java:308,351`, `tcm/Startup.java:226,504,614`, `tcm/compatibility/GossipHelper.java:435` and `tools/CMSOfflineTool.java:505,689`.
- **Why it matters:** this is where token ranges become placements and transition plans (join, leave, move, replace). It is the natural cut for **pluggable replica placement** (non-token placement, tablets, rack-pinned shards). See Proposed P3.
- **Coupling:** `PrepareJoin`, `PrepareMove` and `UnsafeJoin` hold a `PlacementProvider`. Transition plans are serialized as `PlacementDeltas` in sequences, so the provider must be deterministic across all CMS members and versions.
- **Recommendation:** one `ClusterMetadataService.placementProvider()` accessor, no direct `new`. Keep it internal until P3.

### 1.6 Token allocation — `internal`, grade **D**
- **Entry:** `allocate_tokens_for_keyspace` and `allocate_tokens_for_local_replication_factor` (`config/Config.java:143,145`). The code is in `dht/tokenallocator/` (`TokenAllocator.java:24`, `TokenAllocatorFactory.java:30`, `ReplicationAwareTokenAllocator`, `NoReplicationTokenAllocator`, `OfflineTokenAllocator`).
- **Coupling:** `TokenAllocation.createStrategy` throws for anything but NTS or Simple (`TokenAllocation.java:256-260`). There is no config-selectable allocator.
- **Recommendation:** after 1.4's capability interface, let a strategy supply its own `ReplicationStrategy<Unit>` adapter. This is low priority.

---

## 2. Cluster metadata and membership

### 2.1 TCM Processor / log backend — `internal-seam`, grade **C−**
- **Interfaces:**
  - `tcm/Processor.java:27`: `commit(Entry.Id, Transformation, Epoch, Retry)` and `fetchLogAndWait`
  - `tcm/AbstractLocalProcessor.java:40`: abstract `tryCommitOne`/`acceptCommit`/`fetchLogAndWait` at `:247-249`
  - `tcm/log/LogStorage.java:25`: `append`, `getPersistedLogState`, `getLogStateBetween`, with built-ins `SystemKeyspace` and `None` at `:39-40`
  - `tcm/ownership/...`, `tcm/MetadataValue.java:21`
- **Implementations:**
  - `PaxosBackedProcessor` (`tcm/PaxosBackedProcessor.java:57`) commits by LWT into `system_cluster_metadata.distributed_metadata_log` (`schema/DistributedMetadataLogKeyspace.java:58,90-116`). It is replicated by `MetaStrategy` over CMS members.
  - `AtomicLongBackedProcessor` (tests and single node)
  - `RemoteProcessor` (non-CMS nodes forward to CMS)
  - `GossipProcessor` (`tcm/migration/GossipProcessor.java:29`, refuses commits before `nodetool cms initialize`)
  - `SwitchableProcessor` (`ClusterMetadataService.java:1115`)
- **Selection:** `if (CassandraRelevantProperties.TCM_USE_ATOMIC_LONG_PROCESSOR.getBoolean())` → AtomicLong, else Paxos (`ClusterMetadataService.java:196-211`). The sysprop is `cassandra.test.use_atomic_long_processor` (`CassandraRelevantProperties.java:636`). **There is no pluggable consensus backend for metadata.** The constructor takes `Function<Processor, Processor> wrapProcessor` (`:188`), which is a decorator hook used by tests and the simulator.
- **Coupling:**
  - `ClusterMetadata.current()` has 383 references in 147 files.
  - `logBootstrapCallback` checks `instanceof PaxosBackedProcessor` (`:396`).
  - CMS membership is expressed as `MetaStrategy` placements, so the consensus group equals a Cassandra keyspace's replicas.
  - Commit verbs (`TCM_COMMIT_REQ`, `TCM_FETCH_CMS_LOG_REQ`, and others) are fixed `Verb` entries.
- **Mixed version:** gated by `Transformation.Kind.introducedIn` vs `Directory.commonSerializationVersion` (`tcm/Transformation.java:119,283-292`). **[BRANCH]** bumps `NodeVersion.CURRENT_METADATA_VERSION` to `V13` and adds `V11`–`V13` (`tcm/serialization/Version.java`).
- **Recommendation:** see Proposed P2. Short term:
  - Turn the processor `if` into a factory keyed by a non-test config option, restricted to `AtomicLong|Paxos`.
  - Remove the `instanceof` check.
  - Document that `Processor` plus `LogStorage` is the cut: total order of `Entry` by `Epoch`, and idempotent replay.

### 2.2 CustomTransformation / Extension metadata — `internal-seam`, grade **C**
- **Surface:**
  - `Transformation.Kind.CUSTOM(30, V0)` (`tcm/Transformation.java:261`)
  - `tcm/transformations/CustomTransformation.java:37`, with a static `Map<String, serializer> extensions` (`:43`), `registerExtension`/`unregisterExtension`/`clearExtensions` (`:94-107`), and in-tree demos `PokeString`, `PokeInt`, `ClearInt` (`:47-49,161+`)
  - Metadata values in `ClusterMetadata.extensions` (`tcm/ClusterMetadata.java:121`)
  - Keys in `tcm/extensions/ExtensionKey.java:31`, whose value class is loaded by name at `:62`
- **Value:** this is the only place where out-of-tree code could put durable, linearizable, cluster-wide state through TCM without a new `Kind`.
- **Gaps:**
  - Registration is a static mutable map with no config-driven loading.
  - An unknown extension on deserialize hits an `assert` (`:90`). With assertions disabled it causes an NPE. Either way, a node without the plugin cannot replay the log.
  - There is no per-extension version or eligibility gate beyond `child.eligibleToCommit`.
- **[BRANCH] contrast:** the prototype did *not* use CUSTOM. It added first-class kinds `PREPARE_TRANSACTION_DOMAIN(44, V11)` and `ACTIVATE_TRANSACTION_DOMAIN(45, V12)` (`tcm/Transformation.java`, branch) plus a core key `CONSISTENCY_DOMAINS` (`tcm/MetadataKeys.java`, branch). That is evidence that CUSTOM is not yet trusted for production state.
- **Recommendation:** harden CUSTOM into the extension mechanism:
  1. Load registrations through `ServiceLoader` at startup, before log replay.
  2. Give each extension a declared `introducedIn` version and a required-on-all-nodes flag, enforced through `Directory` like `Kind.introducedIn`.
  3. Make an unknown extension a typed startup failure with a clear message, or, for `optional` extensions, an opaque passthrough that keeps the bytes.

### 2.3 Gossip post-TCM — `internal-seam` (residual), grade **D+**
- **Role now:** membership, tokens and schema are owned by TCM. Gossip still carries liveness, `LOAD`, `SEVERITY`, `RPC_READY`, `SSTABLE_VERSIONS`, `DISK_USAGE`, `INDEX_STATUS`, and `NET_VERSION` ("derived from ClusterMetadata") (`gms/ApplicationState.java:33-74`). It also serves the upgrade path through `tcm/compatibility/GossipHelper.java`. `X1`–`X10` are reserved padding states (`:65-74`), which is a de-facto extension slot.
- **Interfaces:** `gms/IGossiper.java` (register, unregister, `getEndpointStateForEndpoint`, `applyStateLocally`) and `gms/IEndpointStateChangeSubscriber.java:30`. `Gossiper` implements `IGossiper` (`gms/Gossiper.java:141`). `repair/SharedContext.java:80` exposes `IGossiper`, so repair is already abstracted from the singleton.
- **Coupling:** `Gossiper.instance` has 151 references in 42 files.
- **Recommendation:** don't make gossip pluggable. Instead:
  - Shrink it into a `ClusterLiveness` and `NodeAdvisoryState` service with `IGossiper` as the only entry point.
  - Document `X1`–`X10` as reserved for extensions, with a registration API, so plugins can publish soft per-node state without new `ApplicationState` values.

### 2.4 IFailureDetector — `stable-public` (thin), grade **C+**
- **Entry:** `failure_detector: "FailureDetector"` (`config/Config.java:523`). `DatabaseDescriptor.createFailureDetector` (`:2093-2099`) resolves short names to `org.apache.cassandra.gms.` and is installed as a supplier (`:1834`).
- **Interface:** `gms/IFailureDetector.java:29` (`isAlive`, `interpret`, `report`, `remove`, `forceConviction`, listeners).
- **Coupling:** `public static final IFailureDetector instance = newFailureDetector()` (`gms/FailureDetector.java:97`). It is resolved once in a static initializer, so a test or reload can't swap it. Gossip pushes heartbeats into it, so a replacement still depends on gossip heartbeats. `HintsService` (`hints/HintsService.java:125,129`) and `SharedContext` (`repair/SharedContext.java:78`) take it by injection.
- **External implementations:** none known (inference).
- **Recommendation:** fine as is for experiments. Next steps:
  - Pass it through `SharedContext`-style injection everywhere (29 static call sites).
  - Decouple its input from gossip with a `HeartbeatSource` interface, which enables a FD fed by TCM or by an external health service.

### 2.5 SeedProvider — `stable-public`, grade **B+**
`seed_provider: {class_name, parameters}` (`config/Config.java:130`), loaded by `applySeedProvider` (`DatabaseDescriptor.java:1678-1690`). The interface is `locator/SeedProvider.java:22`. Seeds are now used mostly for TCM discovery (`tcm/discovery`). It is widely implemented by Kubernetes operators (inference, unverified). It already shows the right pattern: ParameterizedClass, node-local, no wire impact.

---

## 3. Consistency and coordination

### 3.1 ConsistencyLevel — closed enum, grade **F** as a seam
`db/ConsistencyLevel.java:35-52` has 12 public values plus four `UNSAFE_DELAY_*` test values (99–102). Codes are native-protocol wire codes, so a new CL needs a protocol change. Replica counting for DC-aware levels branches on `instanceof NetworkTopologyStrategy` inside the enum (`:103,115,164,299`). **Recommendation:** don't make CL extensible. Move the per-strategy blockFor and liveness math behind the strategy capability interface from 1.4, so the CL enum stays fixed and the semantics sit with the strategy.

### 3.2 Coordinator write path — `partial`, grade **D**
- **Entry points:** static `StorageProxy` methods: `mutate` (`service/StorageProxy.java:981`), `mutateWithTriggers` (`:1230`), `dispatchMutationsWithRetryOnDifferentSystem` (`:1278`, which routes between eventual and Accord), `mutateAtomically` (`:1445`), `performWrite` (`:1741`) and `cas` (`:370`).
- **Hooks:**
  - The only override point is `AbstractReplicationStrategy.getWriteResponseHandler` (`locator/AbstractReplicationStrategy.java:85-141`), so a custom strategy can return a custom `AbstractWriteResponseHandler` (`service/AbstractWriteResponseHandler.java:77`).
  - `ideal_consistency_level` attaches a shadow handler for metrics (`:117-138`).
  - Replica selection is the static `locator/ReplicaPlans.java`.
- **Leakage:**
  - `StorageService.instance` has 194 references in 85 files, and `MessagingService.instance()` has 158 in 84.
  - **[BRANCH] finding:** to enforce one invariant (tables owned by an external transaction domain must not take ordinary writes), the prototype added `TransactionDomainGuard.check` in **25 source files**. Among them are `StorageProxy`, `AbstractMutationVerbHandler`, `ReadCommandVerbHandler`, `CassandraKeyspaceWriteHandler`, `Hint`, `HintsDispatcher`, `HintsService`, `BatchlogManager`, `CommitLog`, `CommitLogReplayer`, `CassandraStreamReceiver`, `StreamReceiveTask`, `StreamSession`, `StreamingRepairTask`, `ActiveRepairService`, `SSTableImporter`, `Tracker` and `ColumnFamilyStore`, plus five CQL statements.
  - **There is no single mutation-admission chokepoint.** Writes enter storage through at least eight independent ingress paths: coordinator, replica verb, hints, batchlog, commitlog replay, streaming, import, and repair. This is the strongest structural argument for Proposed P1.
- **Recommendation:** P1 (write admission SPI) is a prerequisite for every other consistency plugin.

### 3.3 Coordinator read path — `internal-seam`, grade **C−**
- `service/reads/AbstractReadExecutor.java:66`, with the static factory `getReadExecutor(ClusterMetadata, ...)` at `:196`.
- `service/reads/ReadCoordinator.java:37` is the closest thing to a read SPI: `localReadSupported`, `forNonLocalStrategyTokenRead`, `sendReadCommand`, `sendReadRepairMutation`, `isEventuallyConsistent` (`:67-80`). It exists so Accord can reuse the replica-read machinery.
- Range reads are in `service/reads/range/`, which includes `AccordRangeResponse`.
- **Recommendation:** promote `ReadCoordinator` to be the read-side half of P1 and P4, and pass it explicitly instead of using static defaults.

### 3.4 SpeculativeRetryPolicy — `stable-public` (closed), grade **C**
Table option `speculative_retry` (also `additional_write_policy`). The interface is `service/reads/SpeculativeRetryPolicy.java:24`, with `calculateThreshold(SnapshottingTimer, long)` and `kind()`. The `Kind` enum is `NEVER, FIXED, PERCENTILE, HYBRID, ALWAYS` (`:26-29`). `fromString` is a closed if-chain (`:42-60`). Schema stores the string, so mixed-version is fine for the built-ins. **Recommendation:** allow `speculative_retry = 'class:com.x.Y(args)'` behind a `Kind.CUSTOM`. The policy is coordinator-local and touches no wire. The only risk is schema replay on nodes missing the class, so treat it as required-on-all-nodes the same way as 2.2.

### 3.5 Read repair strategy — `stable-public` (closed), grade **C**
Table option `read_repair` → `service/reads/repair/ReadRepairStrategy.java:29`, an enum `{NONE, BLOCKING}` that implements `ReadRepair.Factory` (`service/reads/repair/ReadRepair.java:41-45`). The factory interface is already right. **Recommendation:** same pattern as 3.4. Note that read repair mutations go out via `ReadCoordinator.sendReadRepairMutation` (`ReadCoordinator.java:75`), so a custom read repair is also a write ingress for P1.

### 3.6 Paxos v1/v2 — `stable-public` (closed), grade **C**
- `paxos_variant` is a global, volatile yaml value: `PaxosVariant` enum (`config/Config.java:1110-1190`), default `v1`, read into `Paxos.PAXOS_VARIANT` (`service/paxos/Paxos.java:240`), with the switch at `Paxos.useV2()` (`:1293`).
- Related config: `paxos_state_purging`, `paxos_repair_enabled`, `skip_paxos_repair_on_topology_change*` (`Config.java:1197-1283`).
- Per-request routing is `ConsensusRequestRouter.routeAndMaybeMigrate` (`service/consensus/migration/ConsensusRequestRouter.java:189`) over `ConsensusRoutingTarget {paxosV1, paxosV2, accord}` (`:91-96`).
- **Mixed version:** v2 requires all nodes to support it. The switch is cluster-wide but set per node, with no TCM-enforced agreement (inference: operators must coordinate the switch manually).
- **Recommendation:** move `paxos_variant` into TCM as cluster state, and turn `ConsensusRoutingTarget` into an open provider id. That is P1/P4 territory.

### 3.7 Accord integration — library SPI grade **B+**; Cassandra-side configuration grade **C**
- **Library boundary:** git submodule `modules/accord` (`.gitmodules`, `apache/cassandra-accord`). Accord defines host SPIs in `modules/accord/accord-core/src/main/java/accord/api/`: `Agent`, `DataStore`, `Journal`, `MessageSink`, `TopologyService`, `ProgressLog`, `Scheduler`, `Read`/`Update`/`Write`/`Query`/`Result`, `Key`/`RoutingKey`, `ViolationHandler`, and others.
- **Implementations of the host SPI:** Cassandra's `service/accord/api/*` (`AccordAgent`, `AccordTimeService`, `AccordViolationHandler`, `AccordTopologySorter`, and others) is one. `modules/accord/accord-maelstrom` is a verified second host. This is the inverse of a Cassandra plugin: a consensus engine that is pluggable into hosts.
- **Cassandra side:**
  - `accord.*` config (`config/AccordConfig.java`, `enabled` at `:44`, plus about 60 tuning keys) is attached at `Config.accord` (`config/Config.java:1303`).
  - The service interface is `service/accord/IAccordService.java:80`. Static holders in `AccordService` (`service/accord/AccordService.java:308-351`) hold `instance`, `unsafeInstance`, a `NOOP_SERVICE` and `requestInstance`/`replyInstance`.
  - Table option `transactional_mode` (`service/consensus/TransactionalMode.java:71-91`: `off`, `mixed_reads`, `full`, and others) and `transactional_migration_from` (`schema/TableParams.java:101-102`).
  - Migration state is in TCM (`ConsensusMigrationState`, kinds 37–38 at `tcm/Transformation.java:270-271`). The target enum is `ConsensusMigrationTarget {paxos, accord}` (`service/consensus/migration/ConsensusMigrationTarget.java:27-30`).
- **On trunk there is no TransactionEngine-like SPI on the Cassandra side.** Execution selection is enums: `TransactionalMode`, `ConsensusMigrationTarget`, `ConsensusRoutingTarget`. See `pluggable-consistency/reports/03-execution-seams.md` and `11-accord-library-and-spi.md`.

### 3.8 [BRANCH] TransactionProvider SPI and durable transaction domains — prototype grade **B−**
- **New package:** `service/consensus/txn/` (18 main files, all added on the branch).
  - `TransactionProvider` (`id()`, `capabilities()`, `execute(TransactionPlan, TransactionExecutionContext)`; `Capability {STRICT_SERIALIZABLE, READ, WRITE, CONDITIONAL, MULTI_TABLE}`)
  - `TransactionPlan`/`TransactionOperation`/`TransactionCondition`/`TransactionReference`/`TransactionValue`/`TransactionReadSlot`/`TransactionOutcome`
  - `TransactionProviderRegistry` (`resolve`/`select` by table set and required capabilities)
  - `TransactionDomain`/`TransactionDomainDescriptor`/`ConsistencyDomains`
  - `ExternalTransactionDomainBinding`
  - `TransactionDomainGuard`
  - `AccordTransactionProvider` and `AccordTransactionCompiler`
- **TCM integration:** kinds `PREPARE_TRANSACTION_DOMAIN(44,V11)` and `ACTIVATE_TRANSACTION_DOMAIN(45,V12)`, the core key `CONSISTENCY_DOMAINS`, and a serialization version check for descriptors in `ClusterMetadata` (`tcm/ClusterMetadata.java:1395-1396`, branch).
- **Loading:** `TransactionProviders.defaultRegistry()` hard-codes only Accord (`service/consensus/txn/TransactionProviders.java:55-60`). `unsafeSetRegistryForTesting` swaps it (`:46`). The etcd and Ratis providers (`EtcdTransactionProvider`, `RatisTransactionProvider`, `Scalar*`) are **test-tree only** (`test/unit/.../consensus/txn/`). There is no production loading mechanism.
- **Assessment:** the shapes are right: a CQL-independent plan IR, capability negotiation, durable domain ownership in TCM, and an admission guard. The weak points are a static registry, no config or ServiceLoader, and an admission guard scattered across 25 files (3.2). Detailed evidence and results are in `pluggable-consistency/PHASE-3*-RESULTS.md` and `REVIEW-HANDOFF.md`.

### 3.9 [CEP-45 BRANCH] Mutation tracking / witnesses — `partial` (unmerged), provisional grade **C**
- The keyspace option `replication_type = 'tracked'|'untracked'` is an enum (`schema/ReplicationType.java:29-31` on `origin/cep-45-mutation-tracking`), serialized only from TCM `V9`.
- About 58 `isTracked`-style branch points.
- A new `replication/` package: `MutationTrackingService`, `CoordinatorLog`, `MutationJournal`, `Offsets`, `TrackedWriteRequest`, `ActiveLogReconciler`, transfers, and others.
- It is a second, hard-coded replication and reconciliation mode, not a plugin point. However, it introduces exactly the abstractions a pluggable anti-entropy engine needs: per-mutation ids, coordinator logs, and reconciled offsets. Track it as the P5 foundation. None of it is in this tree.

---

## 4. Data movement and durability side-paths

### 4.1 Hinted handoff — `internal`, grade **D**
- `hints/HintsService.java:75` is a final class with `public static HintsService instance = new HintsService()` (`:100`), which is mutable and not final. Other classes: `HintsDispatcher`, `Hint` (final), `HintVerbHandler` (final).
- The only config is `hinted_handoff_enabled` plus `hints_compression` (ParameterizedClass for the compressor, `config/Config.java:121,544`).
- The FD is injected (`:125-129`).
- **[BRANCH]:** `HintsDispatcher` must decode every hint when any domain is reserved, which disables the encoded fast path (`hints/HintsDispatcher.java`, branch diff). That shows the cost of an admission check that is not native to the path.
- **Recommendation:** no SPI is needed. Route hint application through P1.

### 4.2 Batchlog — `internal`, grade **D**
`batchlog/BatchlogManager.java:107,113` (`public static final instance`). **[BRANCH]** added `validateBatchMutations`, which decodes encoded remote mutations to admission-check them. **Recommendation:** same as hints. Route through P1. A pluggable "atomic multi-partition write" provider would belong under P4, not here.

### 4.3 Repair — `internal-seam`, grade **C**
- **Interfaces:**
  - `repair/KeyspaceRepairManager.java:34` and `repair/TableRepairManager.java:34`, constructed with `new CassandraKeyspaceRepairManager(this)` (`db/Keyspace.java:287,298`) and `new CassandraTableRepairManager(this)` (`db/ColumnFamilyStore.java:594`). These are storage-engine seams.
  - `repair/IValidationManager.java:25`, `repair/StreamExecutor.java:24`, `repair/Scheduler.java:35`, `repair/asymmetric/PreferedNodeFilter.java:25`.
  - `repair/SharedContext.java:56-86`: a DI context exposing messaging, FD, locator, gossiper, compaction, validation, stream executor and paxos repair state. It is used by the simulator, and it is the best-factored subsystem in this area.
- **Orchestration:** `repair/RepairCoordinator.java:98` and `service/ActiveRepairService.java:162` (`ActiveRepairService.instance`, 30 references). Incremental repair is in `repair/consistent/ConsistentSession.java:146`, and preview repair in `streaming/PreviewKind.java:32`.
- **Wire coupling:** validation produces `MerkleTrees` (`repair/Validator.java:48-77`, `utils/MerkleTree.java:77`) carried in fixed repair messages (`repair/messages/RepairMessage.java:62`). The anti-entropy algorithm is part of the wire protocol.
- **Auto-repair (CEP-37):** `token_range_splitter` is a `ParameterizedClass` per repair type (`repair/autorepair/AutoRepairConfig.java:325-420`, contract at `:538`) for `IAutoRepairTokenRangeSplitter` (`:38`), with in-tree `RepairTokenRangeSplitter` and `FixedSplitTokenRangeSplitter`. That is grade **B**, a proper public plugin.
- **Recommendation:** see P5.

### 4.4 Streaming — `internal-seam`, grade **C+**
- **Per-table abstraction:**
  - `streaming/TableStreamManager.java:37` (`createStreamReceiver`, `prepareIncomingStream` `:47`, `createOutgoingStreams`)
  - `streaming/IncomingStream.java:32`, `OutgoingStream.java:36`, `StreamReceiver.java:26`
  - Dispatch goes through `cfs.getStreamManager()` (`streaming/StreamReceiveTask.java:72`, `StreamSession.java:515`, `streaming/messages/IncomingStreamMessage.java:48`)
  - The implementation is `new CassandraStreamManager(this)` (`db/ColumnFamilyStore.java:593`)
  - `db/streaming/*`: `CassandraOutgoingFile`, `CassandraEntireSSTableStreamWriter/Reader` (zero-copy), `ComponentManifest`, `CassandraStreamReceiver`
- **Zero-copy streaming:** `stream_entire_sstables: true` (`config/Config.java:812`), decided per file in `CassandraOutgoingFile.computeShouldStreamEntireSSTables` (`db/streaming/CassandraOutgoingFile.java:69`). It is tied to the SSTable format and component manifest.
- **Closed parts:** `StreamOperation` enum (`streaming/StreamOperation.java:20-29`) and the `StreamMessage.Type` enum (`streaming/messages/StreamMessage.java:62`).
- **Transport:** `streaming/StreamingChannel.java:34`. `Factory.Global` holds a static `NettyStreamingConnectionFactory`, with an `unsafeSet` at `:40-46`. `StreamSession` accepts a `StreamingChannel.Factory` (`streaming/StreamSession.java:280`). The SSTableLoader overrides it (`io/sstable/SSTableLoader.java:305`).
- **Observer:** `StreamHook` from sysprop `cassandra.stream_hook` (`streaming/StreamHook.java:29-38`, `CassandraRelevantProperties.java:605`).
- **Versioning:** `accept_streaming` requires `current_version` on both ends (`net/MessagingService.java:283,288`), so streaming between mixed majors is refused. That is a strong mixed-version constraint.
- **Recommendation:** P6. Expose `StreamingChannel.Factory` and `TableStreamManager` as factories selected per table and per storage engine (coordinate with the storage-engine researcher). Keep the `StreamMessage` protocol fixed.

### 4.5 Internode messaging — `internal-seam`, grade **D+**
- **Surface:**
  - `net/Verb.java:202` is an enum of about 169 verbs, with `Kind {NORMAL, CUSTOM}` (`:430-433`). The CUSTOM id space starts at `(1<<14)-1`, with at most 1000 custom ids (`:585-589`), and holds a single placeholder, `UNUSED_CUSTOM_VERB` (`:414`).
  - Handlers, serializers and expirations can be swapped by reflection: `unsafeSetHandler`/`unsafeSetSerializer`/`unsafeSetExpiration` (`:545-571`).
  - `net/IVerbHandler.java:27`.
  - `net/MessageDelivery.java:49` is an interface implemented by `MessagingService` (`:211`) and injected through `SharedContext`.
  - Filter chains: `inboundSink.add(Predicate)` (`net/InboundSink.java:154`) and `outboundSink.add(Filter|AsyncFilter)` (`net/OutboundSink.java:122-132`). These are used by in-JVM dtests and the simulator.
- **Versioning:** `VERSION_40/50/60` (12/13/14), with `minimum_version = VERSION_40` (`net/MessagingService.java:262-275`). Verb ids are wire constants.
- **Gap:** an enum cannot gain values at runtime, so a plugin cannot add verbs without editing `Verb.java`. The CUSTOM range is reserved but unusable without source changes.
- **Recommendation:** P7.

---

## 5. Cross-cutting coupling inventory (trunk counts, `rg -c` over `src/java`)

| Singleton | Refs | Files |
|---|---|---|
| `ClusterMetadata.current()` | 383 | 147 |
| `StorageService.instance` | 194 | 85 |
| `MessagingService.instance()` | 158 | 84 |
| `Gossiper.instance` | 151 | 42 |
| `ActiveRepairService.instance` | 30 | 15 |
| `FailureDetector.instance` | 29 | 16 |
| `DatabaseDescriptor.getPartitioner()` / `IPartitioner.global()` | 79 | — |
| `instanceof NetworkTopologyStrategy` (and casts) | 31 | ~10 |

`repair/SharedContext.java` is the existing in-tree template for de-singletonizing a subsystem. It is the pattern to replicate.

**Mixed-version mechanisms available to any new boundary:**
1. TCM `Version` plus `Kind.introducedIn`, gated on `Directory.commonSerializationVersion` (`tcm/Transformation.java:119`).
2. `MessagingService` version, negotiated per connection (`net/MessagingService.java:262-288`).
3. Schema options as strings, which old nodes may reject.
4. Gossip `X1`–`X10` padding states.

Only (1) gives cluster-wide, linearizable "everyone can do X now" semantics. **Every plugin that affects placement, ownership or execution must register through TCM, with a per-plugin version, before it is activated.**

---

## 6. Proposed new boundaries

Each proposal names the cut, the contract, and the mixed-version safety rule.

### P1. Mutation and read admission SPI (prerequisite for everything else)
- **Cut:** one `MutationAdmission.check(Mutation, IngressPath)` and `ReadAdmission.check(ReadCommand, IngressPath)` called at a single choke point per path. `IngressPath` is an enum: COORDINATOR, REPLICA_VERB, HINT, BATCHLOG, COMMITLOG_REPLAY, STREAM, IMPORT, REPAIR, READ_REPAIR, PAXOS_COMMIT. Natural homes are `Keyspace.apply` / `CassandraKeyspaceWriteHandler` for writes and `ReadCommand.executeLocally` for reads, plus pre-apply checks for the paths that bypass them (streaming and import operate on SSTables, not mutations, so they need `SSTableAdmission`).
- **Evidence:** **[BRANCH]** needed 25 files for one invariant (3.2).
- **Mixed version:** admission policy derives only from TCM state (e.g. `CONSISTENCY_DOMAINS`), so every node evaluates the same epoch. Activation waits on a `ProgressBarrier` (`tcm/sequences/ProgressBarrier.java`) so no lagging node keeps accepting writes. The prototype's two-step PREPARE/ACTIVATE domain kinds already model this.

### P2. Pluggable metadata consensus backend (`Processor` + `LogStorage`)
- **Cut:** `Processor` (`tcm/Processor.java:27`) plus `LogStorage` (`tcm/log/LogStorage.java:25`) form the cut. Add a `MetadataLogBackend` factory (`metadata_log_backend: {class_name, parameters}`) that supplies both. Paxos-on-`distributed_metadata_log` stays the default. Candidates are external Raft (Ratis), etcd, and Accord itself.
- **Contract:**
  - Gap-free, totally ordered `Entry` by `Epoch`.
  - `commit` is linearizable and idempotent by `Entry.Id`.
  - `fetchLogAndWait` returns a consecutive prefix.
  - Snapshots are compatible with `MetadataSnapshots`.
  - CMS membership changes (`ReconfigureCMS`) must be expressible in the backend, or delegated to it, which would replace `MetaStrategy` placements for the CMS.
- **Mixed version:** the backend is cluster-wide state. Switching requires a migration sequence analogous to `GossipProcessor` → `PaxosBackedProcessor` (`nodetool cms initialize`): freeze at epoch N, copy the log, then commit a `SWITCH_BACKEND` transformation in the old backend whose replay tells nodes to switch. All nodes must have the backend on the classpath before the switch, enforced through a `Directory` capability.
- **Risk:** this is the root of trust for the whole cluster. Keep it `internal-seam` until a second backend passes the TCM simulator suites.

### P3. Pluggable replica placement
- **Cut:** a public `PlacementProvider` (`tcm/ownership/PlacementProvider.java:31`), selected per keyspace (`replication = {'placement': '...'}`) or per cluster. It is the generalization of `UniformRangePlacement`. ARS becomes "replica count and DC policy" and the PlacementProvider becomes "which ranges go to which nodes, and how transitions happen".
- **Contract:** deterministic over `(ClusterMetadata, Keyspaces)`. Transition plans must emit `PlacementDeltas` that respect `LockedRanges` and the existing `ProgressBarrier` streaming steps.
- **Mixed version:** CMS members compute placements once at commit, and they are serialized in the log as `DataPlacements` (`tcm/ClusterMetadata.java:1414-1415`). Non-CMS replicas could replay placements as data without the provider class, if `AlterSchema` stops recomputing locally. Removing that recompute-on-replay is what makes this boundary safe for staggered rollouts.

### P4. TransactionEngine / execution provider SPI (productize the [BRANCH] prototype)
- **Cut:** `TransactionProvider` (branch) at the CQL-to-plan boundary, plus `ReadCoordinator` and `getWriteResponseHandler` for non-transactional CL paths, plus the `ConsensusRoutingTarget` enum becoming an open provider id.
- **Loading:** `ServiceLoader`, plus `transaction_providers: [ParameterizedClass]` in yaml. Ownership of domains stays in TCM (`CONSISTENCY_DOMAINS`).
- **Mixed version:** a domain can only be ACTIVATEd when every node reports the provider id and version in `Directory`, or in a `NodeVersion` capability set. Old nodes reject statements for unknown domains with a typed error, not a fallback to Paxos or Accord. Admission (P1) prevents bypass. See `pluggable-consistency/design/ARCHITECTURE.md` and `ROADMAP-AND-RISKS.md`.

### P5. Pluggable anti-entropy / repair engine
- **Cut:** above `RepairCoordinator` at the `RepairTask`/`SyncTask` level (`repair/RepairTask.java:25`, `repair/SyncTask.java:43`), and above `Validator` with an `AntiEntropySummary` abstraction that replaces direct `MerkleTrees`. Default: MerkleTree. Alternatives: CEP-45 log-offset reconciliation, or range-hash and bloom digests. Keep `KeyspaceRepairManager`/`TableRepairManager` as the storage-side seam.
- **Mixed version:** repair sessions negotiate an engine id in the prepare message. They fall back to MerkleTree unless all participants advertise the engine, either through the messaging version plus a capability bit or a gossip `X*` state. The incremental-repair `pending_repair` marking must be engine-agnostic.

### P6. Streaming as a transport abstraction
- **Cut:** make `StreamingChannel.Factory` (`streaming/StreamingChannel.java:36-46`) a config-selected factory, for example object-store-mediated transfer or RDMA. Make `TableStreamManager` come from the table's storage engine and not `new CassandraStreamManager`. Keep `StreamMessage` framing as the control plane.
- **Mixed version:** streaming already requires an identical `current_version` (`net/MessagingService.java:288`). Alternative transports negotiate in `StreamInitMessage` and fall back to Netty.

### P7. Internode extension verbs and transport
- **Cut (verbs):** replace the reflective CUSTOM verb slot with `VerbRegistry.registerCustom(int customId, Stage, serializer, handler)` during startup, before messaging opens. The ids live in the existing CUSTOM range (`net/Verb.java:585-589`), so plugins can add RPCs without editing `Verb.java`.
- **Cut (transport):** `MessageDelivery` (`net/MessageDelivery.java:49`) is already the outbound interface consumed through `SharedContext`, so promote it as the injection point everywhere and pass it instead of `MessagingService.instance()` (158 references).
- **Mixed version:** a custom verb must declare a minimum messaging version and an "unknown-at-peer" policy (drop or fail). Peers advertise their registered custom-verb set (gossip `X*` or a `Directory` capability) so senders never emit verbs that a peer cannot deserialize. Today an unknown verb id fails deserialization on the connection.

### P8. Node-local policy plugins (low risk, quick wins)
Make these ParameterizedClass-loadable, following the `SeedProvider` and `token_range_splitter` pattern:
- custom `speculative_retry` (3.4)
- custom `read_repair` factory (3.5)
- `NodeProximity` decorators (1.3)
- `NodeProximity` and `InitialLocationProvider` with parameters (1.2)
- FD heartbeat source (2.4)

None of these touch the wire or TCM, except that table options are replayed from schema, so missing classes must fail clearly at DDL time and be required-on-all-nodes.

---

## 7. Priority order (inference, author's judgement)
1. **P8** quick wins, plus the strategy capability interface that removes `instanceof NetworkTopologyStrategy` (1.4). Mechanical, no wire impact.
2. **P1** admission SPI. The branch proves it is needed, and it unblocks P4 and P5.
3. Harden **CustomTransformation** (2.2) into the TCM extension mechanism with versioned, required-on-all-nodes registration. Every other plugin needs durable cluster state.
4. **P4**, which productizes the prototype on top of 1–3.
5. **P3** and **P5**, which need TCM replay to treat placements and summaries as data.
6. **P2** and **P7**, which have the highest blast radius. Keep them internal until simulator coverage exists.
