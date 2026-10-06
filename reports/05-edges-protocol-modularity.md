# 05 — The Node's Edges: Client Protocol, Internode Transport, Serialization, Change Data Out, Module Structure

Scope: native protocol, internode messaging, versioned serialization (messaging, SSTable, commitlog, TCM, Accord), CDC and mutation observation, build/module structure, dtest-api, and the offline tools that Cassandra Analytics/Sidecar use.

Source: `cassandra/` on branch `codex/pluggable-consistency-prototype`, HEAD `5afdb580a7`, merge-base with trunk `0691eb3319`. Paths below are relative to `cassandra/src/java/org/apache/cassandra/` unless they start with `cassandra/`.

Conventions:
- **[F]** means verified in this tree.
- **[I]** means inference, or external knowledge not checked here. This covers Stargate, Scylla, Debezium, Analytics and Sidecar internals.
- **[B]** means the item exists only on this branch, not on trunk.

---

## 0. Executive summary

| # | Boundary | Status | Grade |
|---|---|---|---|
| 1 | Native protocol listener (`transport.Server`, `NativeTransportService`) | internal-seam | D |
| 2 | Native protocol messages (`Message.Type`, `Message.Codec`) | internal-seam (closed enum) | D |
| 3 | Protocol versions / beta (`ProtocolVersion`) | stable-public (wire spec) | B |
| 4 | Frame / envelope compression (lz4 / snappy) | internal-seam (hardcoded) | D |
| 5 | Request execution: `QueryHandler` (`cassandra.custom_query_handler_class`) | partial | C |
| 6 | Dispatcher / `ClientResourceLimits` / overload | internal-seam (static) | D |
| 7 | Custom payload (protocol v4+ `customPayload`) | stable-public (wire) | B- |
| 8 | Internode `Verb` enum incl. `Kind.CUSTOM` | internal-seam, fork-oriented | D+ |
| 9 | Internode message params (`ParamType`, `CUSTOM_MAP`) | partial | C+ |
| 10 | `InboundSink` / `OutboundSink` filters | internal-seam (test-oriented) | C- |
| 11 | `IVersionedSerializer` + `MessagingService.Version` | internal-seam | C |
| 12 | Accord embedded versioned serializers (`LATEST` / `CLUSTER_SAFE` / `DOWNGRADE_SAFE`) | internal-seam (best pattern in tree) | B |
| 13 | TCM `serialization.Version` + `Transformation.Kind.introducedIn` | internal-seam | B- |
| 14 | TCM extensions (`ExtensionKey` / `CustomTransformation.registerExtension`) | partial (test-only users) | C |
| 15 | SSTable format SPI (`SSTableFormat.Factory`, ServiceLoader) + `StorageCompatibilityMode` | partial | B- |
| 16 | CDC (commitlog hard-links to `cdc_raw` + `_cdc.idx`) | stable-public (file contract), weak | C- |
| 17 | `CommitLogReader` / `CommitLogReadHandler` as a library API | internal-seam (de-facto public) | D+ |
| 18 | Mutation write path observer (`KeyspaceWriteHandler`) | internal-seam (hardcoded) | D |
| 19 | Build / module structure (ant, `modules/accord`, checkstyle) | partial | D |
| 20 | in-JVM dtest API (`dtest-api` 0.0.18) | stable-public (versioned API jar) | A- |
| 21 | `CQLSSTableWriter` / `cassandra-sstableloader` artifact | partial | C |
| 22 | Offline SSTable/commitlog reading for Analytics/Sidecar | internal-seam (bridge-per-version) | D |

The main finding: Cassandra has strong versioning machinery but almost no runtime extension registries.

- **Versioning is strong.** Messaging versions, SSTable format versions, TCM `Version`, Accord's self-describing embedded versions and `StorageCompatibilityMode` exist and work.
- **Registries are almost absent.** Protocol opcodes, verbs, compressors, listeners and write-path observers are closed Java `enum`s or hardcoded `new` calls.
- **Exceptions.** The only real runtime registries are:
  - `SSTableFormat.Factory` (ServiceLoader)
  - `QueryHandler` (one system property)
  - `CustomTransformation.registerExtension` (TCM, test-only today)
  - the message `CUSTOM_MAP` param
- **External consumers embed `cassandra-all`.** CDC readers, Analytics and Sidecar take the whole jar and call internals. Their compatibility is paid for with per-version "bridge" modules. [I]
- **Two good models to copy.** The dtest-api is the in-tree example of how to do this properly: a tiny separately-versioned interface jar, with a per-branch implementation behind an isolating classloader. Accord's `EmbeddedAsymmetricVersionedSerializer` is the right pattern for plugin payloads in mixed-version clusters.

---

## 1. Native protocol listener — `transport.Server` / `NativeTransportService`

- **Status:** internal-seam.
- **Entry point:**
  - `CassandraDaemon.startNativeTransport()` (`service/CassandraDaemon.java:938`) constructs one `NativeTransportService` (`:673-674`).
  - `NativeTransportService.initialize()` (`service/NativeTransportService.java:62`) builds exactly one `transport.Server` on `native_transport_port` / `rpc_address` (`:78-86`).
- **Interfaces [F]:**
  - `CassandraDaemon.Server` (`service/CassandraDaemon.java:1039`) is a generic `start/stop/isRunning/clearConnectionHistory` lifecycle.
  - `transport.Server implements CassandraDaemon.Server` (`transport/Server.java:72`).
  - `Server.Builder.withPipelineConfigurator(...)` (`transport/Server.java:264`) and `withEventNotifier` (`:270`) are real injection points. However, only tests and `SimpleClient` paths use them. Production always takes the default `PipelineConfigurator` (`:115-119`).
- **Loading mechanism:** none. There is no list of servers, no config key for additional listeners and no ServiceLoader. The old Thrift second listener and `native_transport_port_ssl` were removed. No dual-port option remains in `config/Config.java:352-372`. [F]
- **External consumers [I]:**
  - Stargate v1 did not plug into this. It ran a separate JVM that joined the ring as a non-token coordinator ("persistence-cassandra-3.11/4.0/dse" modules). It embedded a version-specific `cassandra-all` and called `QueryHandler`/`StorageProxy`-level APIs.
  - Stargate v2 moved front-ends (REST, GraphQL, Docs, gRPC) out-of-process, talking gRPC to a "Bridge" coordinator.
  - Scylla Alternator is a second listener in the same process (HTTP/JSON on its own port). It translates directly to Scylla's internal storage proxy, bypassing CQL parsing.
- **Leakage/coupling:**
  - `Server` statically reaches `DatabaseDescriptor`, `StorageService.instance.register(notifier)` and `Schema.instance.registerListener(notifier)` (`transport/Server.java:114-127`).
  - The `Dispatcher` executor pools are `static` (`transport/Dispatcher.java:62,81`). Two servers would share them silently.
- **Compatibility story:** none needed today. There is one listener.
- **Grade: D.** The lifecycle interface exists, but there is no way to register a second front end without patching `NativeTransportService`/`CassandraDaemon`.
- **Recommendation:** add a `ClientFrontEnd` SPI. Details are in Proposal P1.
  - Discover it via ServiceLoader. Enable it with `client_front_ends: [{class_name, parameters}]` in yaml.
  - Lifecycle hooks the daemon already has: `start`, `stop`, `isRunning`, `drain` (used by `nodetool disablebinary`), and connection listing for `system_views.clients`.
  - The CQL native server becomes the first implementation. Make the `Dispatcher` pools per-server rather than static.

## 2. Native protocol message set — `Message.Type`, `Message.Codec`

- **Status:** internal-seam (closed).
- **Classes [F]:**
  - `transport/Message.java:64` (`abstract class Message`) and `:74` (`interface Codec<M> extends CBCodec<M>`).
  - `enum Type` (`:91-108`) hard-maps opcodes 0–16 to codecs.
  - `fromOpcode` throws `ProtocolException("Unknown opcode")`.
  - `Request.execute(QueryState, RequestTime)` (`:244-246`) is the per-message server hook.
  - Message codecs live in `transport/messages/*`.
- **Loading mechanism:** Java enum. Duplicate opcodes are rejected in the static initializer.
- **External consumers:** drivers implement the wire spec (`doc/native_protocol_v*.spec`). No code consumers. [I]
- **Leakage:** `Request.execute` takes `QueryState` (service) and returns transport `Response`. The request objects carry `QueryOptions` and `cql3` types.
- **Compatibility:**
  - Clients negotiate by protocol version.
  - Unknown opcodes are a hard protocol error. That is correct for a spec'd protocol, but it means "custom message types" are impossible without a protocol revision.
- **Grade: D** as an extension point. As a spec boundary it is fine (see §3).
- **Recommendation:** do not open the opcode space. Steer extension to:
  - `customPayload` (§7), and
  - a separate front-end (P1), so CQL stays a clean, versioned spec.

## 3. Protocol versions and beta — `ProtocolVersion`

- **Status:** stable-public (wire spec).
- **Classes [F]:**
  - `transport/ProtocolVersion.java:48-51`: V3, V4, V5, and `V6("v6-beta", true)`.
  - `SUPPORTED_VERSIONS` (`:70`), `CURRENT = V5` (`:87`), `BETA = Optional.of(V6)` (`:88`).
  - Beta versions require the envelope's `USE_BETA` flag (doc comment at `:36-38`).
  - `native_transport_allow_older_protocols` is at `config/Config.java:362`.
- **Compatibility:** good. Clients negotiate, beta is gated by opt-in, and old versions can be disabled.
- **Grade: B.** This is a mature, explicit evolution process: spec doc, beta flag, driver coordination. It is not extensible by third parties, and should not be.
- **Recommendation:** reuse the "beta flag + spec doc" governance model for any new front-end protocol and for the plugin API version (P5).

## 4. Frame / envelope compression

- **Status:** internal-seam (hardcoded).
- **Classes [F]:**
  - `transport/Compressor.java:32` (interface) with nested `SnappyCompressor` (`:57`) and `LZ4Compressor`.
  - `StartupMessage` hard-switches on `"snappy"`/`"lz4"` (`transport/messages/StartupMessage.java:100-115`).
  - The advertised list is hardcoded in `transport/InitialConnectionHandler.java:85-92`.
  - v5+ framing uses `net.FrameEncoderLZ4` / `FrameEncoderCrc` (`transport/PipelineConfigurator.java:303,392-410`). This is shared with internode framing.
- **Notable:** a code comment in `transport/Compressor.java:55` says it does not reuse `ICompressor` (the pluggable SSTable compressor API) "because the API doesn't expose enough".
- **Grade: D.** There are two compression abstractions and neither is pluggable at the protocol edge. Adding zstd for clients needs a protocol spec change anyway.
- **Recommendation:** low priority. If touched, unify on `ICompressor` with a name registry, and keep protocol-advertised names spec-governed.

## 5. Request execution — `QueryHandler`

- **Status:** partial. This is the closest thing to an "internal request API".
- **Entry point [F]:**
  - System property `cassandra.custom_query_handler_class`, loaded in `ClientState`'s static initializer (`service/ClientState.java:122-133`) via `FBUtilities.construct`.
  - Accessed through `ClientState.getCQLQueryHandler()` (`:370`).
  - Used by e.g. `transport/messages/QueryMessage.java:116`.
- **Interface [F]:** `cql3/QueryHandler.java:32-55`. Methods: `parse`, `process`, `prepare`, `getPrepared`, `processPrepared`, `processBatch`.
- **External consumers [I]:**
  - Stargate's persistence layer and DSE/Stargate auth wrappers used custom QueryHandlers.
  - Some tracing/auditing agents wrap `QueryProcessor`.
- **Leakage (heavy):**
  - The signatures expose `CQLStatement`, `QueryState`, `QueryOptions`, `ClientState`, transport `ResultMessage`, `Dispatcher.RequestTime` (a transport class), `BatchStatement` and `MD5Digest`.
  - The only selection mechanism is one system property, read in a static initializer. There is no chaining or decoration.
- **Compatibility:** none. The signature changed across 4.0, 5.0 and trunk (e.g. `RequestTime` was added).
- **Grade: C.** It is a real, used seam, but at the wrong altitude: CQL-text-in, transport-message-out.
- **Recommendation:** split it into two layers (P2):
  1. A transport-neutral `RequestApi` (statement + options + principal → result rows/metadata, with deadline and overload signals).
  2. The CQL adapter on top.

  Make `QueryHandler` a decorator chain (yaml list) rather than a single replacement.

## 6. Dispatcher, `ClientResourceLimits`, overload

- **Status:** internal-seam (static singletons).
- **Classes [F]:**
  - `transport/Dispatcher.java:57`. Static `requestExecutor` (`:62`) and `authExecutor` (`:81`) come from `SharedExecutorPool`.
  - `transport/ClientResourceLimits.java:39`: static `GLOBAL_LIMIT` (`:43`), `PER_ENDPOINT_ALLOCATORS` (`:45`), `GLOBAL_REQUEST_LIMITER` (`:47`).
  - Related yaml options: `native_transport_max_request_data_in_flight[_per_ip]`, `native_transport_rate_limiting_enabled`, `native_transport_max_requests_per_second`, and `native_transport_throw_on_overload` / queue-age backoff (`config/Config.java:366-372,1618-1624`).
- **Grade: D.** Admission control is valuable shared infrastructure. Any new front end needs it, but it is tied to CQL's netty pipeline and static state.
- **Recommendation:** extract an `AdmissionController` (per-tenant/per-endpoint bytes and rate, queue-age backoff) that every front end (P1) and internal API caller (P2) uses. Front ends then cannot bypass overload protection. That is the biggest risk with "plug in a REST server".

## 7. Custom payload

- **Status:** stable-public (wire, protocol v4+).
- **Classes [F]:** `transport/Message.java:170,200-205` carries `Map<String,ByteBuffer> customPayload`, and it is passed into every `QueryHandler` method.
- **Consumers [I]:** DSE/Stargate used it for routing/auth hints. Drivers expose it.
- **Grade: B-.** This is the sanctioned way to attach out-of-band request metadata without a protocol change. Its weakness: there is no namespace convention, no registry of keys and no server-side dispatch.
- **Recommendation:** define a reserved key-prefix convention (`x-<plugin>/...`) and a server-side `PayloadInterceptor` hook in the request API (P2).

---

## 8. Internode `Verb` enum (incl. CUSTOM)

- **Status:** internal-seam. It is oriented to forks and vendors, not runtime plugins.
- **Classes [F]:**
  - `net/Verb.java:202` (`public enum Verb`). There are 139 constants with ids grouped informally by subsystem: 200/201 virtual mutation, 8xx TCM, 1xx Accord.
  - The comment "largest used ID: 116" at `:411` is stale.
  - `Kind { NORMAL, CUSTOM }` (`:430-433`). Custom ids count down from `CUSTOM_VERB_START = 2^14-1` (`:585`), capped by `MAX_CUSTOM_VERB_ID = 1000` (`:589`), with a collision check (`:486-495,616-635`).
  - The only custom constant is the placeholder `UNUSED_CUSTOM_VERB` (`:413-415`).
  - `fromId` throws `IllegalArgumentException("Unknown verb id")` (`:646-657`). It is called during header decode (`net/Message.java:1004,1047`).
  - `@VisibleForTesting unsafeSetHandler/unsafeSetSerializer/unsafeSetExpiration` (`:545-583`) mutate final fields by reflection. The dtests use them to swap handlers.
- **Loading mechanism:** enum constants only. The CUSTOM id space exists so that downstream forks (DSE-style) can add verbs in source without colliding with upstream ids. [I, design intent from comments at `:587-589`]
- **Leakage:** each Verb binds a `Stage`, a timeout function, a serializer supplier and a handler supplier at class-init time. Handlers are static singletons.
- **Compatibility:**
  - A node receiving an unknown verb id fails the decode. Senders must gate on the peer's messaging version (`net/EndpointMessagingVersions.java:33`) or, for TCM/Accord, on the cluster-min version.
  - There is no per-verb `introducedIn`, unlike TCM's `Transformation.Kind`.
- **Grade: D+.** The custom id space is good foresight, but there is no way to register at runtime, no per-verb version gating, and unknown verbs are fatal to the message rather than safely rejected.
- **Recommendation (P4):**
  - Add a `VerbRegistry` that accepts plugin verbs in the CUSTOM range, keyed by stable name → id (persist the id assignment in TCM so every node agrees).
  - Each verb declares `introducedIn` (plugin API version).
  - Unknown custom verbs should produce a typed `FAILURE_RSP` (or be dropped with a metric) instead of throwing during decode.

## 9. Message params — `ParamType` / `CUSTOM_MAP`

- **Status:** partial.
- **Classes [F]:**
  - `net/ParamType.java:43-62`, including `CUSTOM_MAP(14, CustomParamsSerializer)` (`:57`) and `ACCORD_TRACING(20)`.
  - `net/Message.java:715` `withCustomParam(String, byte[])`.
  - Forward compatibility: unknown param ids are length-skipped (`net/Message.java:1162-1172`, comment "forward compatibiliy with minor version changes").
- **Grade: C+.** The tolerant decoder is exactly the property plugins need. `CUSTOM_MAP` gives string-keyed opaque bytes.
- **Recommendation:** document `CUSTOM_MAP` as the plugin side-channel (tracing context, tenant id, domain tags). Reserve namespaced keys.

## 10. `InboundSink` / `OutboundSink`

- **Status:** internal-seam (test-oriented).
- **Classes [F]:**
  - `net/InboundSink.java:54,154` `add(Predicate<Message<?>>)`.
  - `net/OutboundSink.java:34,122,132` `add(Filter)` / `add(AsyncFilter)`.
  - These are used by in-JVM dtest message filters.
- **Grade: C-.** This is a working interception chain on both directions, but it is only predicate/drop, unordered, and has no production config.
- **Recommendation:** promote it to a `MessageInterceptor` SPI (observe, annotate params, reject). It would serve tracing, multi-tenant tagging and fault injection without patching handlers.

## 11. `IVersionedSerializer` + `MessagingService.Version`

- **Status:** internal-seam.
- **Classes [F]:**
  - Serializer interfaces:
    - `io/IVersionedAsymmetricSerializer.java:29`
    - `io/IVersionedSerializer.java:25`
    - `io/ISerializer.java:25`
  - Messaging versions:
    - `net/MessagingService.java:216-228`: `VERSION_40(12)`, `VERSION_50(13)`, `VERSION_60(14)`, with `MIN_ACCORD_VERSION = VERSION_60`.
    - Int constants at `:262-270`. Their comment says to "try to keep Version enum in-sync". This is a known duplication.
    - `currentVersion()` derives from `storage_compatibility_mode` (`:310-312`).
    - `accept_messaging` / `accept_streaming` ranges (`:276-289`).
- **How new on-wire formats are introduced:**
  1. Bump the messaging version (8 bits, "don't waste versions", `:215`).
  2. Branch on the version in every affected serializer.
  3. Gate senders on the peer version.

  The version is global to the node, so a plugin cannot get its own.
- **Grade: C.** It is a coherent, battle-tested scheme, but monolithic: any plugin format change would need a core messaging-version bump.
- **Recommendation:** plugins should not use `MessagingService.Version` as their version axis. Use the Accord pattern (§12).

## 12. Accord embedded versioned serializers (the pattern to copy)

- **Status:** internal-seam. This is the best-designed versioning in the tree.
- **Classes [F]:**
  - `service/accord/serializers/Version.java:33-52`: `enum Version implements MessageVersionProvider`, `V1(1, MessagingService.Version.VERSION_60)`.
  - Three explicit policies:
    - `LATEST`
    - `DOWNGRADE_SAFE_VERSION`, for disk
    - `CLUSTER_SAFE_VERSION`, for messaging in mixed clusters
  - `net/Verb.java:668-670` `accordEmbedded(...)` wraps every Accord verb serializer as `AccordSerializers.embedded(Version.CLUSTER_SAFE_VERSION, delegate)` (`service/accord/serializers/AccordSerializers.java:57-60`). The payload carries its own version header, independent of the messaging version. The comment at `:34-36` says a messaging bump needs no Accord change.
- **Grade: B.** Self-describing, independently versioned payloads with explicit disk-vs-wire safety levels. The gap: `CLUSTER_SAFE` is a static constant, not computed from cluster-min. TCM's `commonSerializationVersion` could drive it.
- **Recommendation:** generalize it as `PluginVersionedSerializer<T>`: a namespace id plus `{latest, clusterSafe(), downgradeSafe()}`. Derive `clusterSafe()` from a per-plugin version recorded in TCM's `Directory` (P4/P5).

## 13. TCM serialization `Version` and `Transformation.Kind`

- **Status:** internal-seam.
- **Classes [F]:**
  - `tcm/serialization/Version.java:30-105`: `OLD(-1)`, `V0`…`V10` on trunk, and `UNKNOWN`. **[B]** V11 (durable transaction-domain reservations), V12 (external transaction-domain binding) and V13 (Ratis binding) exist only on this branch.
  - `NodeVersion.CURRENT_METADATA_VERSION` is `V10` on trunk and `V13` on the branch (`tcm/membership/NodeVersion.java:39`). **[B]**
  - `minCommonSerializationVersion()` reads `Directory.commonSerializationVersion` (`tcm/membership/Directory.java:83`). This is cluster-wide agreement on the oldest serializer every node understands.
  - `Transformation.Kind` (`tcm/Transformation.java:218`) carries `introducedIn` per kind, and the default eligibility check (`:103-113`) refuses to commit a kind that the lowest-version node cannot deserialize. `CUSTOM(30, V0, CustomTransformation.serializer)` is at `:261`.
- **Grade: B-.** This is the most principled mixed-version gating in Cassandra: per-kind introduction version plus cluster-min agreement. It is limited by a single linear core version, so a plugin's metadata changes still need a core enum bump. The branch's V11–V13 bumps for a prototype "provider" show that cost.
- **Recommendation:** add plugin-scoped version vectors to `Directory` (node → {plugin → version}) so plugin transformations gate on plugin-min, not core-min (P4).

## 14. TCM extensions — `ExtensionKey` / `CustomTransformation`

- **Status:** partial (exists, but only tests use it).
- **Classes [F]:**
  - `tcm/extensions/ExtensionKey.java:31`. It serializes `id` and the **value class name** (`:47-61`), and deserializes via `FBUtilities.classForNameWithoutInitialization`.
  - `ClusterMetadata.extensions` (`tcm/ClusterMetadata.java:121`) is an immutable map carried in every metadata snapshot.
  - `CustomTransformation.registerExtension/unregisterExtension(String, serializer)` (`tcm/transformations/CustomTransformation.java:43,94,99`) is a static name → serializer map. Built-ins are only `PokeString`/`PokeInt`/`ClearInt` (`:47-49`).
  - Callers outside core are only `test/distributed/.../log/RegisterTest.java` and `AlterTopologyTest.java`.
- **Leakage/coupling:**
  - The serialized class name couples the on-disk/wire log to Java class names, which makes renames break replay.
  - Registration is static and has no version.
  - A node lacking the plugin cannot deserialize the log entry. There is no "opaque, skip" mode, so this would stall TCM log replay on that node. That is consistent with the unknown-kind behaviour. [I]
- **Grade: C.** This is the right idea (plugin state replicated by the cluster metadata log), with unsafe mechanics.
- **Recommendation:**
  - Replace class-name serialization with stable string ids plus a plugin version.
  - Require that every node has the plugin installed before the extension can be activated. Use a TCM-gated "plugin enabled" transformation.
  - Serialize extension values as length-prefixed opaque bytes, so tooling (`OfflineClusterMetadataDump`, `tools/OfflineClusterMetadataDump.java`) can skip what it doesn't understand.

## 15. SSTable format SPI and `StorageCompatibilityMode`

- **Status:** partial. This is the only ServiceLoader-based storage SPI at this edge.
- **Classes [F]:**
  - `io/sstable/format/SSTableFormat.java:196` `interface Factory`, loaded via `ServiceLoader.load(SSTableFormat.Factory.class, …)` (`config/DatabaseDescriptor.java:2006`), with name-uniqueness validation (`:1944-1960`).
  - Registered in `cassandra/src/resources/META-INF/services/org.apache.cassandra.io.sstable.format.SSTableFormat$Factory`.
  - Formats: `big` (`BigFormat.java:168`) with versions `ma` … `nb`/`oa`/`pa` chosen by compatibility mode (`:436-438`), and `bti` (`BtiFormat.java:65,290`, `current_version="ea"`).
  - `utils/StorageCompatibilityMode.java:29`: `CASSANDRA_4`, `CASSANDRA_5`, `UPGRADING`, `NONE`. This one knob also chooses messaging `current_version` and the commitlog descriptor version (`db/commitlog/CommitLogDescriptor.java:63-73`).
- **Grade: B-.** It is a genuine SPI and is versioned by a two-letter string. However, the `Factory` returns a full `SSTableFormat<R,W>` whose readers and writers extend deep internal classes (`SSTableReader`, `SSTableWriter`, `RowIndexEntry`…). A third-party format is a fork in disguise.
- **Recommendation:**
  - `StorageCompatibilityMode` is the right user-facing "one knob for downgrade safety". Plugins should hook into it (each plugin declares which modes permit its new formats).
  - A read-only "storage reader API" (P3) would be more useful externally than a writable format SPI.

---

## 16. CDC — commitlog segments in `cdc_raw`

- **Status:** stable-public as a *file-layout contract*. It is weak.
- **Classes/config [F]:**
  - `cdc_enabled`, `cdc_block_writes`, `cdc_on_repair_enabled`, `cdc_raw_directory`, `cdc_total_space`, `cdc_free_space_check_interval` (`config/Config.java:497-508`).
  - Per-table `cdc=true` becomes `Mutation.cdcEnabled` (`db/Mutation.java:90,131,338` `trackedByCDC()`).
  - `db/commitlog/CommitLogSegmentManagerCDC.java:51`: segments are hard-linked into `cdc_raw` (`:209,249`, CASSANDRA-12148), and the manager enforces space with blocking or non-blocking writes (`allocate` `:172`).
  - `CommitLogSegment` writes `<segment>_cdc.idx` holding the durable offset, and appends `COMPLETED` when the segment is done (`db/commitlog/CommitLogSegment.java:72-79,365-387,475`).
  - The consumer must delete files to free space.
- **External consumers [I]:**
  - The Debezium Cassandra connector runs as an agent on every node. It reads `cdc_raw` with `CommitLogReader` + a `CommitLogReadHandler`, requires `DatabaseDescriptor.toolInitialization()` and schema loaded from the node, and ships per-major artifacts (3.x / 4.x / 5.x).
  - The DataStax CDC agent for Pulsar/Kafka uses the same approach, plus a separate connector to fetch row images over CQL and dedupe RF copies.
- **Properties consumers must work around [F/I]:**
  - **RF copies.** Every replica emits the mutation. There is no cross-replica ordering or dedupe.
  - **No row images.** Mutations are partial-row deltas with no before or after image.
  - **Version coupling.** Mutation bytes use the commitlog's messaging version (`CommitLogDescriptor.getMessagingVersion()`, `:219`), so a reader needs the exact matching serializer code.
  - **Schema dependence.** Unknown table ids are counted and skipped (`CommitLogReader.java` `readMutation` `:425-460`, catches `UnknownTableException`).
  - **Accord writes bypass CDC. [F, significant]** Accord-applied writes call `mutation.apply(false, false)` with `durableWrites=false` (`service/accord/txn/TxnWrite.java:144-150`). `CassandraKeyspaceWriteHandler.beginWrite` only calls `CommitLog.instance.add` when `makeDurable` is set (`db/CassandraKeyspaceWriteHandler.java:43-55,105`). So **transactional (Accord) writes never reach `cdc_raw`**. Durability lives in the Accord journal. Treat this as verified by code path; it has not been exercised at runtime.
  - **[B]** On this branch, `TransactionDomainGuard.check` was added to `CommitLog.add` and replay (`db/commitlog/CommitLog.java:306`, `CommitLogReplayer.java:291-296`). Provider-owned domains add another class of writes that CDC consumers must reason about.
- **Grade: C-.** It works and is widely deployed, but the contract is "here are our internal log files". Every consumer re-implements dedupe and row-image reconstruction, and breaks on major upgrades. It is also already incomplete for Accord.
- **Recommendation:** add a Change-Stream SPI (P3) at the write-handler level, not the file level. Keep the file-based CDC as one implementation.

## 17. `CommitLogReader` / `CommitLogReadHandler`

- **Status:** internal-seam, but de-facto public (embedded by CDC agents [I]).
- **Classes [F]:**
  - `db/commitlog/CommitLogReadHandler.java:23-80`: `shouldSkipSegmentOnError`, `handleUnrecoverableError`, `handleMutation(Mutation, size, entryLocation, CommitLogDescriptor)`.
  - `db/commitlog/CommitLogReader.java:70,85,130,144-176`: public constructor and the `readAllFiles` / `readCommitLogSegment` overloads.
- **Leakage:** it hands out a live `org.apache.cassandra.db.Mutation` (`PartitionUpdate`, `Row`, `Cell`, `TableMetadata`), so the full storage-engine object model becomes the API. It also requires `DatabaseDescriptor` initialization (`toolInitialization`, `config/DatabaseDescriptor.java:363-376`).
- **Grade: D+.** The handler interface is small and stable-ish, but its payload type is the entire engine.
- **Recommendation:** in P3, provide a `ChangeRecord` view (keyspace, table id/name, partition key bytes, clustering, cell-level ops, timestamps, ttl, origin = {coordinator write, hint, repair, stream, accord, provider}). The internal-to-view converter then lives in core and is versioned with it.

## 18. Write-path observer — `KeyspaceWriteHandler`

- **Status:** internal-seam (hardcoded).
- **Classes [F]:**
  - `db/KeyspaceWriteHandler.java:23-29`, an interface with `beginWrite`, `createContextForIndexing` and `createContextForRead`.
  - It is always constructed as `new CassandraKeyspaceWriteHandler(this)` (`db/Keyspace.java:288,299`).
- **Related (other researchers' areas):**
  - `triggers/ITrigger.java:41` (coordinator-side `augment`, class loaded from the triggers dir via `triggers/CustomClassLoader.java:47`).
  - `notifications/INotificationConsumer.java:20` (SSTable/memtable lifecycle events via `db/lifecycle/Tracker.java:96`).
  - `audit/IAuditLogger.java:21` and FQL (query-level, not data-level).
- **Grade: D.** This is the natural single choke point where *all* local writes pass with full context (commitlog or not, Accord or not), but it is not pluggable.
- **Recommendation:** P3 attaches here: `ChangeListener.onApply(ChangeRecord, origin, durable)` after a successful memtable apply. It is asynchronous, uses a bounded queue, and is shed under overload with a metric.

---

## 19. Build and module structure

- **Status:** partial.
- **Facts [F]:**
  - **Build system.** Ant `cassandra/build.xml` (121 KB) plus `cassandra/.build/*.xml` include files. There is no gradle or maven for core. Maven poms are generated for publishing (`.build/parent-maven-pom.xml`, `cassandra-deps-maven-pom.xml`, `sstableloader-deps-maven-pom.xml`).
  - **Accord.** `modules/accord` is a git submodule (`cassandra/.gitmodules`) built with **gradle**, invoked from ant (`.build/build-accord.xml:4-5`, exec `gradlew`). It produces `cassandra-accord-*.jar`.
    - `accord-core` has **zero** imports of `org.apache.cassandra` (checked with `rg`).
    - Cassandra implements Accord's interfaces in `service/accord/*`. This is the only true library boundary in the codebase, and it works.
  - **Separate tool builds:**
    - `tools/stress`
    - `tools/fqltool`
    - `tools/sstableloader`, which got its own `build.xml` and its own Maven artifact `cassandra-sstableloader` in CASSANDRA-20328 (commit `f80a234d1e`, "part of a broader effort to decouple java driver code from the server code").
  - **Published artifacts:** `cassandra-all` (single jar, `build.xml:974`), `cassandra-sstableloader`, and the dtest jar (`build.xml:2031-2048`).
  - **No enforced package boundaries.** `.build/checkstyle.xml:103-104` only bans specific classes (`java.io.File`, `CompletableFuture`, raw `Executors`, etc.) for concurrency and filesystem hygiene. There is no `ImportControl` and no ArchUnit (`rg -i "importcontrol|archunit"` finds nothing).
- **Package coupling (files containing ≥1 import, `rg` count; same-package refs not counted) [F]:**

| package | files | use `DatabaseDescriptor` | import `service.*` | import `db.*` | import `tcm.*` |
|---|---|---|---|---|---|
| transport | 47 | 14 | 15 | 7 | 0 |
| net | 76 | 11 | 9 | 5 | 7 |
| io | 253 | 23 | 10 | 97 | 1 |
| serializers | 31 | 0 | 0 | 29 | 0 |
| utils | 234 | 9 | 3 | 30 | 2 |
| cql3 | 286 | 32 | 86 | 172 | 46 |
| db | 533 | 87 | 83 | 334 | 30 |
| service | 394 | 74 | 200 | 208 | 64 |
| tcm | 145 | 25 | 36 | 57 | 125 |

- **Reading of the coupling table:**
  - `utils` → `db` (30 files) and `io` → `db` (97) are layering inversions. "Leaf" packages depend on the storage engine.
  - `DatabaseDescriptor` (global static config) is referenced across every layer, including `transport` and `net`. That is the single biggest obstacle to splitting jars, because any module that touches config pulls in the world.
  - The best-isolated candidates for extraction are `serializers` (no DD, no service) and `transport` (no tcm; its `service`/`cql3` dependence is concentrated in `Message.Request.execute` and `messages/*`).
- **Grade: D.** One monolithic jar, no enforced layering, and global static config. The one success (Accord as a gradle sub-library with an inverted dependency) shows the target shape.
- **Recommendation (P5):**
  1. Add a checkstyle `ImportControl` or ArchUnit test in *report-only* mode to baseline the current graph. Then ratchet it.
  2. Create a `cassandra-api` module (no `DatabaseDescriptor`, no singletons). It would hold plugin SPIs, change records, the request API, and the versioned-serializer contract.
  3. Make `cassandra-all` implement it, the same way it implements Accord interfaces.
  4. Use the sstableloader split (CASSANDRA-20328) as procedural precedent for adding artifacts.

## 20. in-JVM dtest API — the model stable boundary

- **Status:** stable-public. It is a separately released, versioned API jar.
- **Facts [F]:**
  - `org.apache.cassandra:dtest-api:0.0.18` (`.build/parent-maven-pom.xml:549-553`) is a separate Apache repo and release.
  - Each Cassandra branch carries the implementation: `test/distributed/org/apache/cassandra/distributed/impl/Instance.java:210` (`implements IInvokableInstance`).
  - Each instance and version runs in its own `InstanceClassLoader` (`impl/AbstractCluster.java:357`). An explicit `sharedClassPredicate` (`:153,1297-1302`) decides which classes, essentially only the API package, cross the loader boundary.
  - Messages cross as `IMessage` (verb id, bytes, version), and messaging versions are propagated for filters (`Instance.java:746,810`).
- **What it teaches:**
  1. Keep the API in its own artifact with its own semver cadence, and implement it per Cassandra branch.
  2. Let only API types and JDK types cross the boundary. Everything else is isolated by classloader, which is what makes cross-version upgrade tests possible.
  3. Data crosses as bytes plus version, not live engine objects.
  4. Use feature flags (`Feature.NETWORK`, `GOSSIP`, …) instead of method proliferation.
  5. Evolve additively. Old API versions keep working against new impls.
- **Grade: A-.** It is the existence proof that Cassandra can maintain a stable, cross-version API. The minus is that the impl still reaches into internals freely, which is fine for tests.
- **Recommendation:** use dtest-api as the explicit template for `cassandra-api` (P5) and for an Analytics storage-reader API (P3b): separate repo/artifact, per-branch impl, isolated classloader, bytes-plus-version at the seam.

## 21. `CQLSSTableWriter` and `cassandra-sstableloader`

- **Status:** partial. It is a public library API with internal types leaking through.
- **Classes [F]:**
  - `io/sstable/CQLSSTableWriter.java:137`, which calls `DatabaseDescriptor.clientInitialization(false)` in its static block (`:144`).
  - `Builder` methods (`:439-687`): `inDirectory`, `forTable`, `withType`, `using`, `sorted`, `withMaxSSTableSizeInMiB`, `withBuildIndexes`. These also take or expose internal types:
    - `withPartitioner(IPartitioner)` (`:513`)
    - `withSSTableProducedListener(Consumer<Collection<SSTableReader>>)` (`:645`)
    - `withFormat(SSTableFormat)` (`:669`)
    - `withSSTableIdBuilder(SSTableId.Builder)` (`:675`)
    - `withCompressionDictionary` (`:687`)
  - Sources for `cassandra-sstableloader`: `cassandra/tools/sstableloader/src/org/apache/cassandra/tools/BulkLoader.java`, `utils/NativeSSTableLoaderClient.java`.
- **Evolution evidence (CHANGES.txt) [F]:**
  - CASSANDRA-19800 (notify on SSTable production)
  - CASSANDRA-20609 (format selection)
  - CASSANDRA-18714 (build SAI)
  - CASSANDRA-20938 (dictionary compression)
  - CASSANDRA-21012 (SSTable id generator)
  - CASSANDRA-18941 (max size when sorted)

  These look like features requested by bulk-writer consumers. Attribution to Cassandra Analytics is [I].
- **Grade: C.** It is the most-used offline API, and an additive builder is a good shape. But `SSTableReader`/`SSTableFormat`/`IPartitioner` in its signature tie callers to exact internals, and its static `clientInitialization` makes multi-version-in-one-JVM impossible without classloader isolation.
- **Recommendation:**
  - Replace internal types in the Builder with API-module types: partitioner by name, and a produced-file callback with `{path, components, first/last token, size}`.
  - Move it into the `cassandra-sstableloader` or `cassandra-api`-dependent artifact.

## 22. Offline reading for Analytics / Sidecar

- **Status:** internal-seam, bridged per version.
- **Facts [I]:**
  - Apache Cassandra Analytics (Spark bulk reader and writer) has `cassandra-bridge` plus `cassandra-four-zero` / `cassandra-five-zero` modules. Each shades a specific `cassandra-all` and exposes a `CassandraBridge` interface: SSTable reader/writer, partitioner, type conversion, CQL schema parsing, and compaction-free scanning via `SSTableReader`/`ISSTableScanner`. Each major release needs a new bridge module.
  - Sidecar uses the bridge or `cassandra-all` for SSTable upload/import (`SSTableImporter` — note `db/SSTableImporter.java` is touched on this branch **[B]**), CDC and restore.
  - Neither project is in this tree, so they were not verified here.
- **What the in-tree side shows [F]:** there is no in-tree API designed for these consumers. They depend on `DatabaseDescriptor.clientInitialization`/`toolInitialization` (`config/DatabaseDescriptor.java:363-420`), schema parsing via `cql3`, and the `io.sstable` reader classes.
- **Grade: D.** It works only because the ecosystem pays the bridge tax. Every Cassandra release risks breaking Analytics, and the coupling is invisible in this repo (no tests here guard it).
- **Recommendation:** P3b. As a cheap first step, add a contract-test module in-tree that exercises exactly the classes and methods the bridges use. That makes breakage visible at commit time.

---

## Proposed new boundaries

**P1. Pluggable client front ends (`ClientFrontEnd` SPI) over a shared admission layer**

- **Design:**
  - ServiceLoader with a yaml `client_front_ends` list. Each entry has a lifecycle (start/stop/drain/isRunning), a connection registry feeding `system_views.clients`, and access to a shared `AdmissionController` (extracted from `ClientResourceLimits` + `Dispatcher` backpressure) plus the shared auth (`IAuthenticator` SASL).
  - The CQL native server becomes implementation #1. Its `Dispatcher` pools become per-front-end.
- **Front ends this enables:** REST/HTTP, gRPC, Redis-protocol, DynamoDB-compatible (Alternator-style) and Mongo-wire. They run in-process for latency, but behind the API module.
- **Comparison:** Stargate proved demand but paid for an out-of-process coordinator plus per-version persistence modules. Alternator shows the in-process variant works but must reach deep internals. With P2 underneath, Cassandra gets Alternator's latency without its coupling.
- **Next steps:**
  - (a) Turn `NativeTransportService` into a list.
  - (b) Make `Dispatcher` executors non-static.
  - (c) Make overload handling front-end-neutral.

**P2. Transport-neutral internal Request API**

- **Design:**
  - A `RequestApi` that takes `{principal, keyspace, statement (CQL text or prepared id, or a structured op such as GetItem/PutItem), bound values as bytes+type, consistency or transaction-domain hints, deadline, payload map}`. It returns `{rows as bytes + column specs, warnings, paging state}` and emits typed errors.
  - `QueryHandler` becomes the CQL adapter, with a decorator chain instead of one system property.
  - `customPayload` maps onto a namespaced payload map.
- **Next step:** define it in `cassandra-api` and implement it by delegating to `QueryProcessor`. Migrate `transport/messages/*Message.execute` onto it, so the CQL front end dogfoods it.

**P3. Change-Stream SPI (with P3b, a Storage-Reader API)**

- **P3 design:**
  - `ChangeListener` attaches at `KeyspaceWriteHandler`/post-apply and emits a stable `ChangeRecord` with an explicit `origin` (client write, hint, repair, stream, Accord apply, provider/domain apply).
  - Async with a bounded buffer, shed under overload, and per-table opt-in via the existing `cdc` table param.
  - It fixes the Accord blind spot (§16) and gives consumers origin tags for dedupe (e.g. ignore repair/stream).
  - Keep `cdc_raw` as a built-in listener for back-compat.
  - Optional later step: a coordinator-side or Accord-ordered stream for single-copy ordered CDC on Accord tables. The Accord journal already has a total order per key.
- **P3b design:**
  - A versioned `StorageReader` API (in the dtest-api style: separate artifact, per-branch impl, isolated classloader) for Analytics/Sidecar: open SSTable set → scan with token range / column projection → rows as bytes + schema; read commitlog segments → `ChangeRecord`.
  - This replaces the bridge modules' reliance on `SSTableReader` internals.
  - First step: an in-tree contract test of the classes Analytics bridges currently call.

**P4. Plugin-safe internode/metadata extension registry**

- **Verbs and serializers:**
  - A `VerbRegistry` over the existing `Kind.CUSTOM` id space. Name → id assignment is committed via TCM so every node agrees.
  - Per-verb `introducedIn(pluginVersion)` gating. Unknown custom verbs produce a typed failure response rather than a decode exception.
  - Payloads use a generalized Accord-style `EmbeddedVersionedSerializer` keyed by plugin namespace (§12).
- **Mixed-version metadata:**
  - TCM `Directory` gains per-node plugin version vectors, so `clusterSafe()` = min over nodes for that plugin.
  - `CustomTransformation`/`ExtensionKey` switch from Java class names to stable ids plus plugin version.
  - Activating a plugin is itself a TCM transformation that requires every node to advertise it. Removing it is the reverse, and is refused while state exists.
- **Message metadata:** `ParamType.CUSTOM_MAP` plus promoted `InboundSink`/`OutboundSink` give a `MessageInterceptor` SPI for metadata and tracing.
- **Branch relevance [B]:** the prototype's TCM V11–V13 bumps for transaction-domain/Ratis state are exactly the core-version churn this proposal would remove.

**P5. Modular build with a `cassandra-api` module and enforced layering**

- **Steps:**
  1. Report-only ArchUnit or checkstyle `ImportControl` baseline.
  2. Create `cassandra-api` with no `DatabaseDescriptor` or singletons. It holds the SPIs (P1–P4), `ChangeRecord`, the request API, `PluginVersionedSerializer` and a plugin descriptor (name, version, supported Cassandra API range).
  3. Ratchet rules: `api` imports nothing from core, `transport` may not import `db.*`, and `utils`/`io` may not import `service.*`.
  4. Add a plugin classloader (generalize `triggers/CustomClassLoader.java:47`) with a dtest-api-style shared-class predicate so plugins see only `cassandra-api` plus the JDK.
  5. Evaluate gradle for new modules following `modules/accord`. Ant can keep orchestrating, as it already does for Accord.
- **Governance:** apply the native-protocol model (spec + `-beta` flag + additive evolution) to the plugin API version.

## Facts vs inferences recap

- **Verified in tree:** every `path:line` cited, including:
  - the closed enums (`Message.Type`, `Verb`, `ParamType`, TCM `Version`, `Transformation.Kind`)
  - the hardcoded single native server
  - the `QueryHandler` system property
  - the ServiceLoader SSTable SPI
  - the CDC hard-link/idx mechanics
  - Accord writes skipping the commitlog (`durableWrites=false`)
  - no import-control/ArchUnit
  - Accord-core having no Cassandra imports
  - the sstableloader artifact split
  - dtest-api 0.0.18 with classloader isolation
- **Branch-only [B]:**
  - TCM `Version` V11–V13 and `CURRENT_METADATA_VERSION=V13`
  - `TransactionDomainGuard` in `CommitLog.add` / `CommitLogReplayer`
  - branch touches to `SSTableImporter`, streaming and hints
- **Inference / external, not verified here:**
  - Stargate v1/v2 architecture
  - Scylla Alternator internals
  - Debezium and DataStax CDC agent internals
  - Cassandra Analytics bridge modules
  - Sidecar dependencies
  - the claim that CUSTOM verbs exist for vendor forks
  - the claim that CQLSSTableWriter features were driven by Analytics
  - the TCM log stalling on an unknown custom transformation
