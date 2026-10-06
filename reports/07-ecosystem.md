# 07 — Out-of-process and ecosystem boundaries

Research date: 2026-10-03. Sources are GitHub at default-branch HEAD, read via `gh api`, plus a shallow clone of `apache/cassandra-analytics` in `/tmp`, ASF JIRA REST, and the ASF cwiki. Every web claim has a URL next to it. **[unverified]** marks claims I could not confirm from a primary source. **[inference]** marks my own reading of the evidence.

## TL;DR

- **The ecosystem's real boundary is "whatever internals I could reach."** Most ecosystem projects keep one module per Cassandra version: Sidecar adapters, Analytics bridges, Management API agents, LDAP/Kerberos auth, Debezium, the DataStax CDC agent, Stargate persistence and esop's vendored MBeans. Upgrade pain grows with how deep the coupling goes:
  - JMX/CQL clients (Sidecar adapters, ecChronos, Reaper-JMX) pay little.
  - Projects that embed `cassandra-all` or subclass internals (Analytics, Debezium, CDC agent, Stargate, LDAP) pay a lot.
  - Bytecode-patching agents (Management API, cassandra-exporter) break silently at runtime with `NoSuchMethodError`, even on patch releases.
- **Three boundaries are missing and keep being rebuilt:**
  1. A stable **SSTable/commit-log read/write library** that is not `cassandra-all`.
  2. A stable **management API** (typed, versioned, not raw JMX MBeans).
  3. A **CDC stream API**. Today every consumer re-implements `CommitLogReadHandler` against internal `Mutation`/`PartitionUpdate` types.
- **Sidecar has no third-party plugin model.** It assembles a fixed Guice module list. It is the natural *host* for ops features (CEP-40, CEP-44 and CEP-53 already target it), but outside code cannot plug into it without forking.
- **Out-of-process does not remove coupling by itself.** Sidecar's CDC feature pulls in Analytics' bridges, which shade a full `cassandra-all` per version. The coupling just moved. A process boundary only helps if the protocol across it is stable.

---

## 1. Apache Cassandra Sidecar

**Plugin and extension model.**
- `SidecarModules.all()` builds a fixed list of about 20 Guice modules, with CDC as the only conditional one ([SidecarModules.java](https://github.com/apache/cassandra-sidecar/blob/trunk/server/src/main/java/org/apache/cassandra/sidecar/modules/SidecarModules.java)). The list:
  - Api, Auth, CassandraOperations, Configuration, Coordination, HealthCheck, Lifecycle, LiveMigration, OperationalJobs, OpenApi, RestoreJob, Scheduling, SchemaReporting, SidecarSchema, SSTablesAccess, Telemetry, Utilities, SysInfo.
  - `addIf(cdcEnabled, new CdcModule())`.
- The daemon passes that list straight to `Guice.createInjector(SidecarModules.all(config))` ([CassandraSidecarDaemon.java](https://github.com/apache/cassandra-sidecar/blob/trunk/server/src/main/java/org/apache/cassandra/sidecar/CassandraSidecarDaemon.java)).
- There is no `ServiceLoader` and no config-driven module list. The only override path is `Modules.override()`, which the javadoc mentions for tests.
- Internally, routes, periodic tasks and table schemas register through Guice multibindings (`modules/multibindings/VertxRouteMapKeys`, `PeriodicTaskMapKeys`, `TableSchemaMapKeys`). That is a ready-made internal extension seam that was never exposed. **[inference]**
- JIRA has a few small pluggability tickets: CASSSIDECAR-88 "Allow DriverUtils to be pluggable", CASSSIDECAR-114 (pluggable hash algorithm) and CASSSIDECAR-116 (extensible JmxClient). None of them is a plugin API ([JIRA search](https://issues.apache.org/jira/browse/CASSSIDECAR-116)).
- `LifecycleProvider` (`start`/`stop`/`isRunning` for a Cassandra instance) is a small in-Sidecar SPI. `ProcessLifecycleProvider` is the default ([LifecycleProvider.java](https://github.com/apache/cassandra-sidecar/blob/trunk/server/src/main/java/org/apache/cassandra/sidecar/lifecycle/LifecycleProvider.java)).

**Version adapters.**
- `adapters/` holds `adapters-base`, `adapters-cassandra41` and `adapters-cassandra50` ([tree](https://github.com/apache/cassandra-sidecar/tree/trunk/adapters)).
- `CassandraVersionProvider` picks the highest `ICassandraFactory` whose minimum version is at or below the running node's version ([CassandraVersionProvider.java](https://github.com/apache/cassandra-sidecar/blob/trunk/server/src/main/java/org/apache/cassandra/sidecar/utils/CassandraVersionProvider.java)).
- The adapters are **thin**. `Cassandra50StorageOperations` only extends `Cassandra41StorageOperations` and adds constructors ([source](https://github.com/apache/cassandra-sidecar/blob/trunk/adapters/adapters-cassandra50/src/main/java/org/apache/cassandra/sidecar/adapters/cassandra50/Cassandra50StorageOperations.java)).
- `adapters-base` talks to Cassandra only through:
  - JMX proxy interfaces it defines itself: `StorageJmxOperations` → `org.apache.cassandra.db:type=StorageService`, plus CompactionManager, StreamManager, EndpointSnitch, Table and metrics ([jmx/](https://github.com/apache/cassandra-sidecar/tree/trunk/adapters/adapters-base/src/main/java/org/apache/cassandra/sidecar/adapters/base/jmx)).
  - CQL through driver 3.11.3.
- `adapters-base` has no `cassandra-all` dependency ([adapters-base/build.gradle](https://github.com/apache/cassandra-sidecar/blob/trunk/adapters/adapters-base/build.gradle)).
- **This is the cheapest version coupling in the whole ecosystem.**

**The hidden deep coupling.**
- `server/build.gradle` depends on six published Analytics artifacts: `cassandra-analytics-core`, `-cdc`, `-cdc-codec`, `-cdc-sidecar`, `cassandra-avro-converter` and `-common` ([server/build.gradle L147-156](https://github.com/apache/cassandra-sidecar/blob/trunk/server/build.gradle)).
- Through those artifacts, Sidecar's CDC feature (CEP-44) inherits the Analytics bridges, which shade a full `cassandra-all` per version (section 2).
- CEP-63 names this as the project's own pain. It calls the Sidecar↔Analytics release coupling "a concrete, ongoing velocity bottleneck" and notes the circular dependency and a "complete private fork of the Sidecar client" inside Analytics ([CEP-63](https://cwiki.apache.org/confluence/display/CASSANDRA/CEP-63%3A+Merge+cassandra-analytics+and+cassandra-sidecar+into+cassandra-ecosystem+and+refactor)).
- CEP-63 explicitly puts a shared library with core "out of scope", calling it "attractive long-term".

**Is Sidecar the natural home for ops plugins?** Mostly yes, and the CEP index already points that way ([CEP index](https://cwiki.apache.org/confluence/display/CASSANDRA/Cassandra+Enhancement+Proposals+%28CEP%29)):

| Feature | CEP | Status | Notes |
|---|---|---|---|
| Management process | CEP-1 | Adopted | |
| Live migration | CEP-40 | Adopted | `livemigration/` package exists |
| Kafka CDC via Sidecar | CEP-44 | Adopted | `cdc/` |
| Merge with Analytics | CEP-63 | Adopted | |
| Rolling restarts via Sidecar | CEP-53 | Under discussion | |
| CQL Management API | CEP-38 | Under discussion | |
| Restore jobs | — | Present | `restore/`, `RestoreJobModule` |
| Repair jobs | — | Present | `handlers/RepairHandler`, `job/RepairJob` |

**The counter-example is repair scheduling.**
- CEP-37 "Unified Repair Solution" landed **in-process** in 6.0 ([CASSANDRA-19918](https://issues.apache.org/jira/browse/CASSANDRA-19918), fix version 6.0-alpha1).
- Sidecar's repair support is only an on-demand job.
- So the project has not settled where scheduling-type ops live. **[inference]**

## 2. Apache Cassandra Analytics: evidence of a missing SSTable API

**Structure.**
- One bridge family per Cassandra major: `cassandra-four-zero{,-bridge,-types}`, `cassandra-five-zero{,-bridge,-types}` and `cassandra-six-zero{,-bridge,-types}`.
- Each family also has matching `*-avro-converter` and `cassandra-analytics-spark-*-converter` modules ([repo root](https://github.com/apache/cassandra-analytics)).

**How a bridge embeds Cassandra.**
- `cassandra-five-zero/build.gradle` takes `implementation 'org.apache.cassandra:cassandra-all:$cassandra50Version'` and shades the **entire server** into `five-zero.jar`. It excludes JNA, snappy and zstd, and strips Guava `Range*` so all bridges share one copy ([build.gradle](https://github.com/apache/cassandra-analytics/blob/trunk/cassandra-five-zero/build.gradle)).
- Pinned versions: `cassandra40Version=4.0.17`, `cassandra50Version=5.0.7`, `cassandra60Version=6.0-alpha2` ([gradle.properties](https://github.com/apache/cassandra-analytics/blob/trunk/gradle.properties)).
- At runtime, `BaseCassandraBridgeFactory.buildClassLoader()` extracts the jar to a temp file and loads it into a child-first `PostDelegationClassLoader` ([BaseCassandraBridgeFactory.java](https://github.com/apache/cassandra-analytics/blob/trunk/cassandra-bridge/src/main/java/org/apache/cassandra/bridge/BaseCassandraBridgeFactory.java)).
- The effect is one complete Cassandra node runtime per major, isolated by classloader, inside a Spark executor.

**Internal classes used.** I counted distinct `org.apache.cassandra.*` imports outside Analytics' own packages, from the clone:

| Bridge + types modules | Distinct internal imports | Lines of code |
|---|---|---|
| 4.0 | 179 | ~13.8k |
| 5.0 | 125 | ~6.6k |
| 6.0 | 136 | ~6.0k |

The most-used imports in the 5.0 bridge:
- `io.sstable.format.*`, `io.sstable.metadata.*`, `io.sstable.Descriptor`, `ISSTableScanner`, `indexsummary`
- `schema.TableMetadata`, `TableMetadataRef`, `ColumnMetadata`
- `db.DecoratedKey`, `db.rows.*` (Row, Cell, Unfiltered, UnfilteredRowIterator, ComplexColumnData, EncodingStats), `DeletionTime`, `Clustering`
- `dht.IPartitioner`, `Murmur3Partitioner`, `RandomPartitioner`
- `db.marshal.*`
- `io.util.*` (FileHandle, ChannelProxy, DataInputPlus)
- `utils.BloomFilter`, `VIntCoding`
- `config.DatabaseDescriptor`

**Brittleness markers (verified in source):**
- **Split-package classes.** The bridges place classes inside Cassandra's own packages to reach package-private members:
  - `org.apache.cassandra.io.sstable.SSTableTombstoneWriter`
  - `org.apache.cassandra.io.sstable.format.bti.BtiReaderUtils`
  - `org.apache.cassandra.db.DbUtils`
  - `org.apache.cassandra.io.util.{DataInputStreamPlus,RebufferingChannelInputStream,ReadOnlyInputStreamFileChannel}`
  - `org.apache.cassandra.db.commitlog.{BufferingCommitLogReader,SeekableCommitLogSegmentReader}` (4.0)
  - Example: [BtiReaderUtils.java](https://github.com/apache/cassandra-analytics/blob/trunk/cassandra-five-zero-bridge/src/main/java/org/apache/cassandra/io/sstable/format/bti/BtiReaderUtils.java).
- **Reflection.** `ReaderUtils` finds the private 5-arg constructor of `SerializationHeader.Component` by parameter count and calls `setAccessible(true)` ([ReaderUtils.java](https://github.com/apache/cassandra-analytics/blob/trunk/cassandra-five-zero-bridge/src/main/java/org/apache/cassandra/spark/reader/ReaderUtils.java)).
- **Global mutable config.** `CdcBridgeImplementation` calls `DatabaseDescriptor.setCDCEnabled`, `setCommitLogSync`, `setCommitLogCompression`, `setEncryptionContext` and similar to fake a node config ([CdcBridgeImplementation.java](https://github.com/apache/cassandra-analytics/blob/trunk/cassandra-five-zero-bridge/src/main/java/org/apache/cassandra/bridge/CdcBridgeImplementation.java)).
- **Schema runtime entanglement.** CASSANDRA 6.0 TCM leaked into the reader. CASSANALYTICS-202 reports that opening one keyspace instance in the 6.0 bridge diffs against `DistributedSchema.empty()` and rebuilds every keyspace, re-registering metrics ([CASSANALYTICS-202](https://issues.apache.org/jira/browse/CASSANALYTICS-202)).

**Upgrade pain:**
- [CASSANALYTICS-84](https://issues.apache.org/jira/browse/CASSANALYTICS-84) "Create bridge modules for Cassandra 5.0". Filed 2025-08-21, about 11 months after 5.0 GA (5.0 GA date per general knowledge **[unverified here]**). It asks for "as least code copy-pasting as possible".
- [CASSANALYTICS-192](https://issues.apache.org/jira/browse/CASSANALYTICS-192) "Create bridge modules for Cassandra 6.0": four new modules, "based on 5.0 version with minimal code changes".
- [CASSANALYTICS-122](https://issues.apache.org/jira/browse/CASSANALYTICS-122): 5.0's extended `localDeletionTime` (uint storage) forced type changes.
- [CASSANALYTICS-24](https://issues.apache.org/jira/browse/CASSANALYTICS-24): "SSTable-version-based bridge determination". The bridge is selected by on-disk format version, which is effectively the stable key.
- `scripts/build-dtest-jars.sh` pins each Cassandra branch to a commit-ish "in case something committed to one of the Cassandra branches breaks us" ([build-dtest-jars.sh](https://github.com/apache/cassandra-analytics/blob/trunk/scripts/build-dtest-jars.sh)).

**What would fix it.**
- A published, versioned **`cassandra-sstable` library** (reader/writer/scanner over `Descriptor` + `TableMetadata` + partitioner).
- It would need no `DatabaseDescriptor` static init and no metrics or keyspace singletons. It would be versioned by SSTable format, not server version.
- CEP-17 "SSTable format API" (adopted, 5.0) made formats pluggable *inside* the server. It did not make them consumable *outside* it ([CEP index](https://cwiki.apache.org/confluence/display/CASSANDRA/Cassandra+Enhancement+Proposals+%28CEP%29)). **[inference]**

## 3. Drivers and the native protocol

**Protocol versioning.**
- Server `ProtocolVersion`: V3, V4, V5 supported; `V6(6, "v6-beta", true)` is beta ([ProtocolVersion.java](https://github.com/apache/cassandra/blob/trunk/src/java/org/apache/cassandra/transport/ProtocolVersion.java)).
- The Java driver mirrors this: `V6(..., true)` is beta ([DefaultProtocolVersion.java](https://github.com/apache/cassandra-java-driver/blob/trunk/core/src/main/java/com/datastax/oss/driver/api/core/DefaultProtocolVersion.java)).
- The driver also carries a separate vendor namespace, `DSE_V1`/`DSE_V2` ([DseProtocolVersion.java](https://github.com/apache/cassandra-java-driver/blob/trunk/core/src/main/java/com/datastax/dse/driver/api/core/DseProtocolVersion.java)). That is evidence that vendors extend the protocol by forking the version space rather than through negotiated extensions.

**Extension points that exist.**
- **Custom payload.** A `Map<String, ByteBuffer>` on every request and response, delivered to `QueryHandler.process/prepare/processPrepared/processBatch` ([QueryHandler.java](https://github.com/apache/cassandra/blob/trunk/src/java/org/apache/cassandra/cql3/QueryHandler.java)).
- **Custom QueryHandler.** Selected with the system property `cassandra.custom_query_handler_class` (`CassandraRelevantProperties`). `CustomPayloadMirroringQueryHandler` ships as a test aid.
- **`CUSTOM` type code.** Lets clients carry types they don't understand as blobs.
- This is the only in-band channel. It is untyped, and drivers expose it only as raw bytes.

**How new server features reach clients.** A new type, a new message, or a new error code means a protocol-spec change, a protocol-version bump (or beta), and coordinated releases of every driver. Java, Python and gocql are now all ASF repos ([cassandra-java-driver](https://github.com/apache/cassandra-java-driver), [cassandra-python-driver](https://github.com/apache/cassandra-python-driver), [cassandra-gocql-driver](https://github.com/apache/cassandra-gocql-driver)).

**What's missing.** There is no capability-negotiated extension mechanism (for example, `STARTUP` options advertising named extensions with their own frame or opcode ranges). Today a server-side plugin cannot add a message type, a result kind or an error code without a protocol version bump. **[inference]** I found no CEP for this; CEP-50 "Authentication Negotiation" covers only auth.

## 4. Testing infrastructure for plugin authors

**In-JVM dtest API** ([apache/cassandra-in-jvm-dtest-api](https://github.com/apache/cassandra-in-jvm-dtest-api/tree/trunk/src/main/java/org/apache/cassandra/distributed)):
- Offers `ICluster`, `IInstance`, `IUpgradeableInstance`, `IInstanceConfig`, `IMessageFilters`, `IClassTransformer`, `InstanceClassLoader` and `Versions`.
- The API jar is stable. The *implementation* is Cassandra's shaded dtest jar, which every external consumer builds itself from each branch:
  - Sidecar: `build-dtest-jars.sh`, `build-shaded-dtest-jar-local.sh`, `relocate-dtest-dependencies.pom` ([sidecar scripts](https://github.com/apache/cassandra-sidecar/tree/trunk/scripts)).
  - Analytics: [same scripts](https://github.com/apache/cassandra-analytics/tree/trunk/scripts), pinned to commit-ish per branch.
- I found no evidence that the shaded per-version dtest implementation jars are published to Maven Central. **[unverified]**
- Plugin authors can run in-JVM clusters with their plugin on the instance classpath and configured via `IInstanceConfig`. That is plausible, but there is no documented recipe. **[inference]**

**Harry.** Now in-tree under `test/harry` ([apache/cassandra test/](https://github.com/apache/cassandra/tree/trunk/test)). The standalone [apache/cassandra-harry](https://github.com/apache/cassandra-harry) still exists. It is a model-checking fuzzer for the read/write path. It would serve plugin authors who change storage or indexing, but it is not packaged as a library for them. **[inference]**

**Simulator.** In-tree, `test/simulator/{asm,bootstrap,main,test}`. It relies on bytecode weaving (`asm`) of Cassandra's own classes ([test/simulator](https://github.com/apache/cassandra/tree/trunk/test/simulator)). Third-party code on the classpath would have to be simulator-safe (deterministic executors and time), which is unrealistic for external plugins today. **[inference]**

**ccm, cassandra-builds, cassandra-dtest, cassandra-easy-stress** (all active Apache repos). These are black-box tools and are the realistic testing tier for out-of-process plugins.

**What would help plugin authors:**
- Published per-version dtest implementation jars.
- A "plugin TCK" module that runs a plugin's SPI implementation through compatibility tests across supported majors. The ecosystem has rebuilt this matrix many times: LDAP has 6 version modules, Management API 9 agent modules, Analytics 3 bridges.

## 5. Third-party plugins and embedders

Hook-mechanism legend:
- **SPI**: a pluggable class named in config.
- **Agent**: a `-javaagent` (ByteBuddy or premain).
- **Embed**: `cassandra-all` on the classpath, with internal classes called outside the server.
- **JMX/CQL**: an external client.
- **Fork**: a patched server.

| Project | Hook mechanism | Cassandra internals coupled | Upgrade pain evidence | Boundary that would fix it |
|---|---|---|---|---|
| **Sidecar** (ASF) | JMX/CQL client. Embeds via Analytics for CDC | StorageService, CompactionManager, StreamManager and Table MBeans (self-defined proxies). Transitively, the Analytics bridges | Thin per-version adapters ([adapters/](https://github.com/apache/cassandra-sidecar/tree/trunk/adapters)). Analytics coupling called a "velocity bottleneck" ([CEP-63](https://cwiki.apache.org/confluence/display/CASSANDRA/CEP-63%3A+Merge+cassandra-analytics+and+cassandra-sidecar+into+cassandra-ecosystem+and+refactor)) | Versioned management API. CDC stream API |
| **Cassandra Analytics** (ASF) | Embed: full shaded `cassandra-all` per major in a child-first classloader | ~125–180 internal classes: sstable format/metadata/scanner, db.rows, schema, dht, marshal, io.util, DatabaseDescriptor. Split packages plus reflection | New bridge per major ([CASSANALYTICS-84](https://issues.apache.org/jira/browse/CASSANALYTICS-84), [-192](https://issues.apache.org/jira/browse/CASSANALYTICS-192)). TCM schema leak ([-202](https://issues.apache.org/jira/browse/CASSANALYTICS-202)) | Standalone SSTable read/write library, versioned by format |
| **Instaclustr cassandra-ldap** | SPI (`IAuthenticator`, `IRoleManager`) plus split-package subclasses | Puts `LDAPAuthenticator`/`LDAPCassandraRoleManager` in `org.apache.cassandra.auth`. Per-version code uses `QueryProcessor`, `CreateRoleStatement`, `GrantRoleStatement`, `SelectStatement`, `ClientState`, `QueryState`, `Dispatcher`, `AuthKeyspace` | Modules for 2.2, 3.0, 3.11, 4.0, 4.1 and 5.0 ([repo](https://github.com/instaclustr/cassandra-ldap)). Example: [Cassandra50SystemAuthRoles.java](https://github.com/instaclustr/cassandra-ldap/blob/master/cassandra-5.0/src/main/java/com/instaclustr/cassandra/ldap/auth/Cassandra50SystemAuthRoles.java) | Auth SPI that owns role and credential storage callbacks, so plugins don't run CQL statements internally (CEP-31 negotiated auth is in draft) |
| **Instaclustr cassandra-kerberos** | SPI (`IAuthenticator`, SASL) | Auth internals (per-version modules) | Modules for 2.2–4.1. No 5.0 module; last push 2023-07 ([repo](https://github.com/instaclustr/cassandra-kerberos)) | Same as above |
| **Instaclustr cassandra-everywhere-strategy** | SPI (`AbstractReplicationStrategy` subclass) | Replication strategy and TokenMetadata internals **[inference from class type]** | Modules for 2.2, 3.0, 3.11, 4, 4.1 ([repo](https://github.com/instaclustr/cassandra-everywhere-strategy)) | Stable replication-strategy SPI (TCM-era placement API) |
| **Instaclustr esop** (backup/restore) | JMX client plus filesystem | **Vendors its own copies** of `StorageServiceMBean`, `ColumnFamilyStoreMBean`, `FailureDetectorMBean` and `StorageProxyMBean`, per version (cassandra2/3/30/…) under `jmx.org.apache.cassandra` | [vendored MBean tree](https://github.com/instaclustr/esop/tree/master/core/src/main/java/jmx/org/apache/cassandra/service) | Published, versioned management client artifact (or REST/CQL management API) |
| **Instaclustr cassandra-exporter** | Agent (premain). Intercepts the MBeanServer and calls internal statics | `FBUtilities`, `InetAddressAndPort`, Gossiper internals, metrics registry | `NoSuchMethodError` on 4.0 trunk ([#23](https://github.com/instaclustr/cassandra-exporter/issues/23)) and on 4.0.13, a **patch release** ([#126](https://github.com/instaclustr/cassandra-exporter/issues/126)). Repo archived 2024 | Metrics-reporter SPI or stable metrics export endpoint. Users asked for this in [#96](https://github.com/instaclustr/cassandra-exporter/issues/96) "pluggable metrics suppliers" |
| **Instaclustr Minotaur** | Not investigated | — | **[unverified]** | — |
| **K8ssandra Management API** | Agent (ByteBuddy): intercepts `CassandraDaemon`, `QueryHandler`, `Gossiper`, `CassandraRoleManager` and system_distributed replication. Adds a Unix-socket native transport server in `org.apache.cassandra.transport`. "RPC" is tunnelled through CQL (`CALL NodeOps.x()`) | Daemon startup, `Message.Request.execute`, QueryHandler, gossip seeds, role manager, `K8SeedProvider` | Agent modules for 4.x, 4.1.x, 5.0.x, 6.0.x, DSE 6.8/6.9, HCD cc4/cc5, plus `cassandra-trunk` ([repo](https://github.com/k8ssandra/management-api-for-apache-cassandra)). `NoSuchMethodError` on `Message$Request.execute` ([#705](https://github.com/k8ssandra/management-api-for-apache-cassandra/issues/705)), 5.1 nightly broke ([#438](https://github.com/k8ssandra/management-api-for-apache-cassandra/issues/438)), Netty `NoSuchMethodError` on 5.0.2 ([#755](https://github.com/k8ssandra/management-api-for-apache-cassandra/issues/755)), 4.1-SNAPSHOT failed to start ([#122](https://github.com/k8ssandra/management-api-for-apache-cassandra/issues/122)). Agent source: [Agent.java](https://github.com/k8ssandra/management-api-for-apache-cassandra/blob/master/management-api-agent-common/src/main/java/com/datastax/mgmtapi/Agent.java) | In-tree management endpoint (CEP-38 CQL Management API, or Sidecar), seed-provider and lifecycle hooks, local admin transport |
| **cass-operator / k8ssandra-operator** | HTTP client of the Management API | Indirect: everything the Management API agent couples to | `pkg/httphelper` is its only node interface ([cass-operator](https://github.com/k8ssandra/cass-operator/tree/master/pkg/httphelper)) | Same. The operator is fine; its dependency isn't |
| **Reaper** | JMX client **and** an HTTP (Management API) backend: two proxies for the same operations | Compiles against `cassandra-all:5.0.7` for MBean interfaces and enums (`StorageServiceMBean`, `CompactionManagerMBean`, `RepairOption`, `RepairParallelism`, `ProgressEventType`, `FailureDetector`) ([pom.xml](https://github.com/thelastpickle/cassandra-reaper/blob/master/src/server/pom.xml), [JmxCassandraManagementProxy.java](https://github.com/thelastpickle/cassandra-reaper/blob/master/src/server/src/main/java/io/cassandrareaper/management/jmx/JmxCassandraManagementProxy.java)) | 5.0 removed `dateof`, breaking Reaper's schema migrations ([#1436](https://github.com/thelastpickle/cassandra-reaper/issues/1436)). 5.0.2 repair failures ([#1545](https://github.com/thelastpickle/cassandra-reaper/issues/1545)) | Versioned management API. Repair is now partly in-tree (CEP-37, [CASSANDRA-19918](https://issues.apache.org/jira/browse/CASSANDRA-19918)) |
| **ecChronos** (Ericsson) | Separate agent process. CQL (driver) plus JMX, direct or via a Jolokia agent in the Cassandra JVM. LWT lock tables | Repair MBeans, `percentRepaired`/`maxRepairedAt` ([ARCHITECTURE.md](https://github.com/Ericsson/ecchronos/blob/master/docs/ARCHITECTURE.md)) | Low: compatibility matrix doc, "Add tests with Cassandra 5" ([#545](https://github.com/Ericsson/ecchronos/issues/545)) | Proof that JMX/CQL is cheap to maintain. Would gain from a typed repair API |
| **Medusa** (backup) | `nodetool` subprocess, CQL, filesystem copy | Snapshot dir layout, file permissions, SSTable component names | 5.0 permission change ([CASSANDRA-17470](https://issues.apache.org/jira/browse/CASSANDRA-17470)) broke backups ([#698](https://github.com/thelastpickle/cassandra-medusa/issues/698)). BTI format: verify fails when `Rows.db` is 0 bytes ([#861](https://github.com/thelastpickle/cassandra-medusa/issues/861)) | Snapshot/backup API that returns a manifest (component list, sizes, checksums) instead of making tools parse directories |
| **Debezium Cassandra connector** | Embed: a per-node JVM process reading `cdc_raw` with Cassandra's `CommitLogReader`/`CommitLogReadHandler`; needs `cassandra.yaml` ([docs](https://debezium.io/documentation/reference/stable/connectors/cassandra.html)) | `DatabaseDescriptor`, `Mutation`, `PartitionUpdate`, `db.rows.*`, `db.marshal.*`, `CommitLogDescriptor`, `LivenessInfo`, `ClusteringBound` ([Cassandra5CommitLogReadHandlerImpl](https://github.com/debezium/debezium-connector-cassandra/blob/main/cassandra-5/src/main/java/io/debezium/connector/cassandra/Cassandra5CommitLogReadHandlerImpl.java)) | Separate `cassandra-4`, `cassandra-5` and `dse` modules ([repo](https://github.com/debezium/debezium-connector-cassandra)). Many documented semantic gaps: no dedup across replicas, no before-image, best-effort schema changes, unsupported range deletes, TTL on collections and LWT ([docs](https://debezium.io/documentation/reference/stable/connectors/cassandra.html)) | CDC stream API (CEP-6 "CDC v2" is still a draft; CEP-45 mutation tracking may provide the substrate) |
| **DataStax CDC for Apache Cassandra** | Agent (premain/agentmain) that calls `DatabaseDescriptor.daemonInitialization()` and reads commit logs in the Cassandra JVM | `DatabaseDescriptor`, `StorageService`, commit-log reader internals ([agent-c4 Agent.java](https://github.com/datastax/cdc-apache-cassandra/blob/master/agent-c4/src/main/java/com/datastax/oss/cdc/agent/Agent.java)) | `agent-c3`, `agent-c4` and `agent-dse4` modules. "Failed to start cdc agent on cassandra 4.1.2" ([#180](https://github.com/datastax/cdc-apache-cassandra/issues/180)), 4.1 support open ([#134](https://github.com/datastax/cdc-apache-cassandra/issues/134)) | Same CDC stream API |
| **Stargate v1 / v2 coordinator** | Embed: the coordinator joins the ring as a non-token-owning node and implements a `persistence-api` per backend | Executors, `SchemaChangeListener`, QueryHandler, gossip/ring internals | Modules `persistence-cassandra-3.11`, `-4.0` and `-dse-6.8` ([coordinator/](https://github.com/stargate/stargate/tree/main/coordinator)). 4.1 needed a new module because `LocalAwareExecutorService` was renamed and `SchemaChangeListener` became an interface ([#2609](https://github.com/stargate/stargate/issues/2609), [#2311](https://github.com/stargate/stargate/issues/2311)). On 5.0: "We do not have plans currently to produce a Cassandra 5.0 persistence module" ([#3038](https://github.com/stargate/stargate/issues/3038)) | Coordinator-only node role or proxy API (CEP-64 "Proxy Execution Support" is under discussion). v2's API services already moved out-of-process over a gRPC "bridge" ([README](https://github.com/stargate/stargate)) |
| **Stratio cassandra-lucene-index** | SPI (`org.apache.cassandra.index.Index` secondary index) | Index SPI plus read/write path internals | Plugin versions keyed to exact Cassandra patch versions ("version tags (e.g. 3.0.6.3)") ([README](https://github.com/Stratio/cassandra-lucene-index/blob/branch-3.0.14/README.rst)). Archived 2020; never reached 4.0 | Stable index SPI. SAI (CEP-7) now covers much of the use case in-tree |
| **Elassandra** | Fork (Elasticsearch plus a patched Cassandra 3.11) | Everything | "Release of Elassandra based on ES 7.x & Cassandra 4.x" still open ([#395](https://github.com/strapdata/elassandra/issues/395)) | Index plus query-handler SPIs strong enough not to need a fork |
| **Rocksandra** (Instagram) | Fork (`rocks_3.0` branch) | Storage engine | Upstream "Cassandra pluggable storage engine" [CASSANDRA-13474](https://issues.apache.org/jira/browse/CASSANDRA-13474) and [CASSANDRA-13476](https://issues.apache.org/jira/browse/CASSANDRA-13476) still **Open** since 2017. Fork repo archived ([Instagram/cassandra](https://github.com/Instagram/cassandra/tree/rocks_3.0)) | Storage-engine boundary. CEP-11 (memtable) and CEP-17 (SSTable format) cover part of it |

### Pain gradient (observed)

| Coupling style | Examples | Typical break | Cost per major |
|---|---|---|---|
| CQL/JMX client, self-defined interfaces | Sidecar adapters, ecChronos, Medusa | Renamed MBean op, file-layout change, removed CQL function | Small subclass or fix |
| JMX client compiled against `cassandra-all` | Reaper, esop (vendored) | Signature drift; heavy dependency | Re-pin plus exclusions |
| SPI plus internal helpers | LDAP, Kerberos, Everywhere, Lucene | Internal statements and helpers change | New module per major |
| Embedding `cassandra-all` offline | Analytics, Debezium, Stargate | Format, schema and TCM changes; static init | New module (thousands of lines) per major; sometimes abandoned (Stargate 5.0) |
| Bytecode-patching agent | Management API, exporter, CDC agent | `NoSuchMethodError` at runtime, even on patch releases | New agent per major; emergency patches |
| Fork | Elassandra, Rocksandra | Everything | Usually terminal |

## 6. Kubernetes operators and the Management API as an implicit boundary

- cass-operator's whole node interface is the Management API's REST surface (`pkg/httphelper`) ([cass-operator](https://github.com/k8ssandra/cass-operator/tree/master/pkg/httphelper)). Reaper ships an `HttpCassandraManagementProxy` that targets the same API ([management/http](https://github.com/thelastpickle/cassandra-reaper/tree/master/src/server/src/main/java/io/cassandrareaper/management/http)).
- So the de facto Kubernetes management contract for Apache Cassandra is a **third-party REST API implemented by a ByteBuddy agent**. It needs a new agent module per major and per vendor build.
- CEP-2 (Kubernetes operator) is still "under discussion" ([CEP index](https://cwiki.apache.org/confluence/display/CASSANDRA/Cassandra+Enhancement+Proposals+%28CEP%29)).
- Sidecar plus CEP-38 (CQL Management API) is the obvious in-project replacement. That would require:
  - Lifecycle control: Sidecar `LifecycleProvider`, CEP-53 rolling restarts.
  - Seed and config injection: today the Management API patches `K8SeedProvider` in via the agent.
  - A local admin channel: today the Management API adds a Unix-socket CQL transport.
- Once those exist in-project, operators could target Sidecar and the agent becomes unnecessary. **[inference]**

---

## Lessons and proposed boundaries

### L1. Process boundaries are only as stable as the protocol across them
- Sidecar's JMX adapters are cheap because they code against MBean names and signatures, which change rarely.
- Sidecar's CDC is expensive because it pulls `cassandra-all` in via Analytics.
- Reaper (out-of-process) still compiles against `cassandra-all`.
- **Rule:** an out-of-process component should depend on a *published contract artifact* (OpenAPI, CQL virtual tables, a slim MBean-interface jar), never on `cassandra-all`.

### L2. Keep out-of-process, hosted by Sidecar
Operations that are long-running, need cross-node coordination, or are I/O-heavy and must not share a heap with the read/write path:

| Area | Why out-of-process | Contract needed from core |
|---|---|---|
| Backup/restore, snapshot upload (esop, Medusa) | Network and object-store I/O, credentials | Snapshot manifest API (components, checksums, format version), not directory parsing |
| Live migration, data transfer (CEP-40) | Bulk file streaming | Same manifest plus quiesce/flush hooks |
| CDC publishing to Kafka and others (CEP-44, Debezium, DataStax CDC) | Back-pressure must not stall writes; each sink has its own dependencies | **CDC stream API**: typed change events plus offsets, emitted by the server or read through a format-stable commit-log library |
| Metrics export (exporter, Management API metrics) | Exporter dependencies (Prometheus, OTel) | Stable metrics enumeration via virtual tables, or a metrics-reporter SPI (see L3) |
| Lifecycle, rolling restart, config and seed injection (Management API, cass-operator, CEP-53) | Must survive the process it manages | Sidecar `LifecycleProvider`; seed provider fed from Sidecar |
| Bulk analytics read/write (Analytics) | Spark runtime | **Standalone SSTable library** (L4) |
| Repair *orchestration* (Reaper, ecChronos) | Cross-cluster policy, UI | Typed repair API (CEP-38). In-tree CEP-37 handles the default case |

To make Sidecar a real plugin host:
- Expose the existing Guice multibindings (routes, periodic tasks, table schemas, jobs) through `ServiceLoader`-discovered `SidecarModule`s.
- Give plugins a stable `ICassandraAdapter` facade.
- Publish an OpenAPI compatibility test. CEP-63 already proposes stability annotations and an N-1 test.

Without this, third parties will keep building parallel sidecars: the Management API, ecChronos, Reaper, esop.

### L3. Keep in-process, as SPIs
Some functions sit on the request path or need node-local state at hot-path latency:
- Authentication, authorization and role storage (LDAP, Kerberos, CIDR/mTLS)
- Replication strategy and placement
- Secondary indexes (Lucene → SAI)
- Memtable and SSTable formats (CEP-11, CEP-17)
- QueryHandler / payload interceptors
- Seed provider, SSLContext (CEP-9), and a metrics reporter

The lesson from LDAP, Lucene and Stargate: **an SPI is only stable if it is self-sufficient.** Each of those plugins needed per-version modules even though it implemented an official interface, because the interface forced it to reach into helpers:
- LDAP runs `CreateRoleStatement`/`QueryProcessor` internally and subclasses `CassandraRoleManager` in a split package.
- Stargate depends on executor and `SchemaChangeListener` types.

Proposed rules for in-process SPIs:
1. The SPI package exports **only** interfaces and value types it owns. No `db.rows.*` or `cql3.statements.*` in signatures.
2. The server passes in callbacks (role store, schema view, executor) instead of letting plugins call statics.
3. `@Stable`/`@Evolving` annotations plus a compatibility test per release (a "plugin TCK" run on the in-JVM dtest framework).
4. No split packages. JPMS-style package sealing or a checkstyle rule would surface violations. **[inference]**

### L4. The single highest-leverage missing boundary is an SSTable and commit-log library
Analytics (three bridges), Debezium, the DataStax CDC agent, Sidecar CDC (transitively), the Medusa verify step and offline tools all rebuild "read Cassandra files outside a running node."

A `cassandra-storage-format` artifact would fix it. It should:
- Be versioned by on-disk format (CASSANALYTICS-24 already selects bridges by SSTable version).
- Have no `DatabaseDescriptor` static initialisation, metrics or keyspace singletons. It should take `TableMetadata` from the caller.
- Expose reader, scanner, writer and commit-log segment reader.
- Live in-tree and be tested by the server's own build.

CEP-65 (a shared `cassandra-utils` library, draft) and the "shared library with core" that CEP-63 deferred are the natural vehicles.

### L5. Agents and bytecode patching mark a missing hook
Each ByteBuddy interception in the Management API maps to a hook the server lacks:

| Intercepted class | Missing hook |
|---|---|
| `CassandraDaemon` | Startup/lifecycle listener |
| `QueryHandler` | Admin command channel |
| `Gossiper` | Seed reload |
| `CassandraRoleManager` | Bootstrap superuser provisioning |
| `SystemDistributedReplication` | System keyspace replication policy |

The exporter shows that internal-static access breaks even on **patch** releases ([#126](https://github.com/instaclustr/cassandra-exporter/issues/126)). Adding these five hooks would remove the main reason agents exist.

### L6. The protocol needs a negotiated extension mechanism
Custom payload is the only in-band hook, and it is untyped. The DSE protocol versions show that vendors fork the version space instead. Named, negotiated protocol extensions (advertised in `SUPPORTED`/`STARTUP`) would let a server plugin add message or result types that the drivers expose through a generic API, without a protocol-version bump. **[inference; no CEP found]**

### L7. Watch for history repeating
- CEP-18 "Improving Modularity" was withdrawn so that its tickets could proceed one by one ([CEP-18](https://cwiki.apache.org/confluence/display/CASSANDRA/CEP-18%3A+Improving+Modularity)).
- CASSANDRA-13474 (pluggable storage engine) has been Open since 2017.
- Boundary work that is not tied to a concrete external consumer tends to stall.
- **Recommendation:** pair each proposed boundary with a named consumer from the table above. Examples: SSTable library ↔ Analytics, CDC API ↔ Debezium/CEP-44, management API ↔ cass-operator/Reaper, auth SPI ↔ cassandra-ldap. Measure success by deleting that consumer's per-version module.

## Open items / not verified

- Instaclustr Minotaur: not investigated.
- Whether per-version shaded in-JVM dtest jars are published to Maven.
- Exact 5.0 GA date used for the Analytics lag estimate.
- Debezium upgrade-pain issues: the issue tracker moved to the `debezium/dbz` repo, which I did not search.
- The Lucene README mentions a Maven profile that can "download and patch" Cassandra. I did not confirm whether that patches server code or only installs the plugin jar.
