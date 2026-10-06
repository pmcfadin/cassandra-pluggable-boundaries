# 01 - Storage engine and local persistence: pluggable boundary catalog

Scope: memtable, SSTable format, compaction, compression, commit log/CDC, caches, file system/disk layout, disk boundaries, `org.apache.cassandra.journal`, tries, and whether a whole-table "storage engine" boundary (CASSANDRA-13474 style) is feasible.

Source: `cassandra`, branch `codex/pluggable-consistency-prototype`. All paths below are relative to `src/java/org/apache/cassandra/` unless prefixed.

**Branch vs trunk (verified):** `git merge-base HEAD trunk` = `0691eb3319`, which is the local `trunk` tip. In the storage area the branch only adds `TransactionDomainGuard.check(...)` calls (11 files, +61 lines), for example `db/ColumnFamilyStore.java` (apply, addSSTables, import, scrub, truncate), `db/Keyspace.java:456`, `db/commitlog/CommitLog.java`, `db/commitlog/CommitLogReplayer.java`, `db/lifecycle/Tracker.java`, and `db/streaming/CassandraStreamReceiver.java`. `service/consensus/txn/TransactionDomainGuard.java` exists **only on this branch**. Everything else in this document exists on trunk. The guard sites are still useful here: they mark the choke points a table-storage boundary would have to own (section 3).

Legend. **V** = verified in the tree (file:line or git). **I** = inference or public knowledge not checked in this tree.

---

## 1. Summary table

| # | Boundary | Status | Selection | Loader | Grade | Recommendation |
|---|---|---|---|---|---|---|
| 1 | Memtable API (CEP-11) | stable-public (documented, per-table) | yaml `memtable.configurations` + table option `memtable = '<key>'` | reflection: static `factory(Map)` or `FACTORY` field | **B** | harden |
| 2 | SSTable format API (CEP-17) | stable-public (node-wide only) | yaml `sstable.selected_format`, `sstable.format.<name>.*` | `ServiceLoader<SSTableFormat.Factory>` | **C+** | harden + formalize |
| 3 | Compaction strategy | stable-public (de facto, since 1.x) | table `compaction = {'class': ...}`, yaml `default_compaction` | reflection ctor `(ColumnFamilyStore, Map)` | **C** | formalize (new interface) |
| 3a | UCS `Controller` | internal (concrete) | UCS options only | `Controller.fromOptions` static | **D** | new sub-boundary |
| 4 | Compression `ICompressor` + CEP-49 providers | stable-public | table `compression = {'class': ...}`, yaml `compressor_providers`, `commitlog_compression` | reflection static `create(Map)`; provider by `class_name` | **B+** | keep, small hardening |
| 5 | Commit log segment manager | internal-seam (test hook) | none (`cdc_enabled` picks one of two) | static `Function` in `DatabaseDescriptor` | **D** | new boundary (WAL SPI) |
| 5a | CDC consumer (`CommitLogReader`/`CommitLogReadHandler`) | partial (de facto external use) | `cdc_enabled`, `cdc_raw_directory`, table `cdc=true` | library embedding | **C-** | formalize as a reader SDK |
| 5b | Commit log archiver | stable-public (shell hooks) | `conf/commitlog_archiving.properties` | shell commands | **C** | keep |
| 6 | Row cache provider | stable-public (small) | yaml `row_cache_class_name` | `FBUtilities.classForNameWithoutInitialization` + `newInstance()` | **B-** | keep |
| 7 | Key cache | not a boundary | `key_cache_size` | hard-coded Caffeine | **F** | leave internal (format-owned) |
| 8 | Chunk cache | not a boundary | `file_cache_enabled`, `file_cache_size` | static singleton | **F** | new boundary (cache tier) |
| 9 | File system / `File` / `ChannelProxy` (CEP-36 area) | internal-seam (test-only swap) | none | `File.unsafeSetFilesystem` static | **D** | new boundary (CEP-36) |
| 10 | `Directories` / data dirs | config only | `data_file_directories`, `local_system_data_file_directory` | n/a | **D** | new boundary (placement) |
| 11 | Disk boundaries / allocation | none | implicit (JBOD) | concrete `DiskBoundaryManager` | **F** | new boundary (placement policy) |
| 12 | Disk access tuning (`disk_access_mode`, `disk_optimization_strategy`) | config enums | yaml | enum switch | n/a | keep |
| 13 | `journal` package (used by Accord) | internal-seam (generic, one user) | Accord yaml only | constructor injection | **B-** | formalize as a library |
| 14 | Tries (`db/tries`, `io/tries`) | internal library | n/a | n/a | B (library), not a boundary | keep internal |
| 15 | Whole-table storage engine (13474) | partial (write/stream/repair seams only) | none | hard-wired `new Cassandra*` | **D** | new boundary, staged |

---

## 2. Per-boundary detail

### 2.1 Memtable API (CEP-11)

- **Status:** stable-public. Per-table selection, documented in `db/memtable/Memtable_API.md` and in the `conf/cassandra.yaml:820-848` comments. Added by commit `e4e19e33fa` "Add memtable API (CEP-11)" (V).
- **Entry point:** yaml `memtable.configurations.<key>.class_name/parameters/inherits` (`conf/cassandra.yaml:841-848`). The table option `memtable = '<key>'` names a yaml key, not a class. If a schema references a key the node lacks, the node logs an error and falls back to the default (`schema/MemtableParams.java` `getWithFallback` :122-131) (V).
- **Loading:** `schema/MemtableParams.java:223-262`. A simple class name is prefixed with `org.apache.cassandra.db.memtable.`. The loader first tries a static `factory(Map)` method, then a static `FACTORY` field, and rejects unconsumed parameters (V).
- **Interfaces:** `db/memtable/Memtable.java:60` (`Memtable extends Comparable, UnfilteredSource, CellSourceIdentifier`), `Memtable.Factory` `:77` (`create(...)` `:88`, plus `writesShouldSkipCommitLog` `:107`, `writesAreDurable` `:120`, `streamToMemtable` `:133`, `streamFromMemtable` `:149`), `Memtable.Owner` `:160`, `FlushablePartitionSet` (~`:356-390`). Bases: `AbstractMemtable`, `AbstractAllocatorMemtable`, `AbstractShardedMemtable`. In-tree implementations: `SkipListMemtable`, `ShardedSkipListMemtable`, `TrieMemtable` (V).
- **External implementations (I):** the trie and sharded memtables came from DataStax (in-tree). Persistent-memory (PMem) memtable work motivated the `writesAreDurable`/`writesShouldSkipCommitLog` hooks. No widely used out-of-tree memtable is known.
- **Leakage/coupling (V):**
  - `Owner.signalFlushRequired(Memtable, ColumnFamilyStore.FlushReason)` (`Memtable.java:163`) and `shouldSwitch(ColumnFamilyStore.FlushReason, TableMetadata)` (`:465`) expose a `ColumnFamilyStore` nested enum.
  - The write signature `put(PartitionUpdate, UpdateTransaction, OpOrder.Group, boolean)` (`:207`) ties implementers to `OpOrder`, the index `UpdateTransaction`, and `CommitLogPosition` bounds (`switchOut` `:407`, `accepts` `:420`, `mayContainDataBefore` `:434`).
  - Implementations reach for `DatabaseDescriptor` and memory pools (`AbstractAllocatorMemtable` holds 8 `DatabaseDescriptor`/static references). Memory accounting is shared and global (`memtable_heap_space`, `memtable_allocation_type`).
  - `ColumnFamilyStore implements Memtable.Owner, SSTable.Owner` (`db/ColumnFamilyStore.java:189`), so the owner side is not substitutable.
- **Compatibility:** no versioning of the Java API. On-disk risk is low because memtables flush through the SSTable writer. Tests: `test/unit/.../db/memtable/MemtableQuickTest`, `MemtableSizeTestBase` (parameterized), and a `TestMemtable` factory. `ant test-latest` runs the full unit suite with TrieMemtable via `test/conf/latest_diff.yaml:27-45`. That covers in-tree implementations only. No packaged TCK exists.
- **Grade B:** narrow, documented, per-table, with two real implementations. It leaks `ColumnFamilyStore` types and OpOrder/commit-log mechanics.
- **Recommendation: harden.**
  1. Move `FlushReason` out of `ColumnFamilyStore` into `Memtable` or a neutral package.
  2. Publish `MemtableQuickTest`/`MemtableSizeTestBase` as an abstract conformance suite that third parties can subclass.
  3. Add schema-to-yaml agreement checks across the cluster, for example a startup or TCM validation warning when nodes lack a referenced key.
  4. Annotate the API (`@Stable`/`@Evolving`) and adopt a deprecation policy.

### 2.2 SSTable format API (CEP-17)

- **Status:** stable-public, documented in `io/sstable/SSTable_API.md` (links to CEP-17). BIG and BTI ship in-tree. Selection is node-wide, not per-table.
- **Entry point:** yaml `sstable.selected_format` (default `big`, `config/Config.java:456-460`) and `sstable.format.<name>.<opt>`. The `storage_compatibility_mode` gate rejects BTI under `CASSANDRA_4` (`utils/StorageCompatibilityMode.java:83`) (V).
- **Loading:** `ServiceLoader.load(SSTableFormat.Factory.class)` at `config/DatabaseDescriptor.java:2006`. Names are validated as unique and lowercase (`:1944-1960`). The service file `src/resources/META-INF/services/org.apache.cassandra.io.sstable.format.SSTableFormat$Factory` lists `BigFormatFactory` and `BtiFormatFactory`. All discovered formats can **read**, and only the selected one **writes** (V).
- **Interfaces:** `io/sstable/format/SSTableFormat.java:45` (`name`, `getVersion`, writer and reader factories, component sets `:60-83`, `getKeyCacheValueSerializer` `:85`, `getScrubber(ColumnFamilyStore, LifecycleTransaction, ...)` `:91`, `delete`, `deleteOrphanedComponents`), `SSTableFormat.Factory` `:196`, `SSTableReaderFactory`, `SSTableWriterFactory`. `io/sstable/format/SSTableReader.java:160` is a ~78 KB **abstract class**, `public abstract class SSTableReader extends SSTable implements UnfilteredSource, SelfRefCounted<...>`. `io/sstable/format/Version.java:34` is an abstract class with about 18 `has*()` feature flags (`:47-118`). `io/sstable/Component.java:82` (`Type.createSingleton(..., formatClass)`) and `:206` (V).
- **External implementations (I):** BTI was contributed by DataStax (CEP-25). DataStax's serverless/CNDB work is the commonly cited motivation for CEP-17 (I). Cassandra Analytics' bulk writer and `CQLSSTableWriter` (`io/sstable/CQLSSTableWriter.java`) produce in-tree formats offline. No independent third-party format is known.
- **Leakage/coupling (V):**
  - Implementers must extend the giant `SSTableReader`/`SSTableWriter`/`SortedTableWriter` hierarchy rather than implement an interface. `SSTableReader` holds `protected final FileHandle dfile` (`SSTableReader.java:276`), which assumes a local chunked Data file.
  - `Version` forces every format to answer questions about shared Stats metadata (`hasCommitLogIntervals`, `hasPendingRepair`, `hasUIntDeletionTime`, and others). Serialization of `StatsMetadata`/`CompressionInfo` is shared, not format-owned.
  - `getScrubber` takes a concrete `ColumnFamilyStore` and `LifecycleTransaction`. `SSTable.Owner` (`io/sstable/SSTable.java:372`) needs `TableMetrics` and `CompressionDictionaryManager`.
  - Format special-casing leaks out of the package: `db/lifecycle/LogFile.java:546` (`BigFormat.is(...)`), `config/DatabaseDescriptor.java:1985` (defaults to `BigFormat.NAME`), and `utils/StorageCompatibilityMode.java:83`.
  - The `Descriptor` filename/directory regexes (`io/sstable/Descriptor.java:72-98`) and `Directories` tie the format to the local path layout.
  - Key-cache values are format-specific (`KeyCacheValueSerializer`, `:187`), and `AbstractRowIndexEntry` is shared across formats.
  - Zero-copy streaming (`db/streaming/ComponentManifest.java`) and SAI (`index/Index.java:392` `getFlushObserver`) assume component files.
- **Compatibility:** per-format `Version` strings and `isCompatible`/`isCompatibleForStreaming` (`Version.java:116-118`), plus `storage_compatibility_mode`. `ant test-oa` and `ant test-latest` (BTI via `test/conf/latest_diff.yaml:53-54`) cover in-tree formats only. There is no format conformance kit.
- **Grade C+:** the SPI and discovery are real and documented. The implementation burden (abstract base classes, shared metadata, local-file assumptions, node-wide only) makes a genuinely new format, such as a columnar or remote one, very expensive.
- **Recommendation: harden + formalize.**
  1. Extract `SSTableReader`'s public surface into an interface. Keep `SSTableReader` as the default base class.
  2. Move Stats/metadata version flags behind a format-owned `MetadataSerializer`.
  3. Remove `BigFormat.is` special cases from `LogFile`.
  4. Allow per-table `selected_format` as a schema option, gated by TCM so that all replicas understand it.
  5. Publish a format TCK: write/read/scrub/verify/stream/import round-trip tests parameterized by `SSTableFormat.Factory`.

### 2.3 Compaction strategy

- **Status:** stable-public by long practice. TWCS began as an out-of-tree jar before it merged (I, public history). UCS (CEP-26) arrived in `957eca2fb5` (V).
- **Entry point:** table option `compaction = {'class': 'X', ...}` and yaml `default_compaction` (`conf/cassandra.yaml:1260-1272`). Simple names resolve to the `org.apache.cassandra.db.compaction.` package (V).
- **Loading:** `schema/CompactionParams.java:314` (`FBUtilities.classForNameWithoutInitialization(..., AbstractCompactionStrategy.class)`), then `db/ColumnFamilyStore.java:990-997` reflects on the constructor `(ColumnFamilyStore, Map)`. Callers: `db/compaction/CompactionStrategyHolder.java:72` and `PendingRepairManager.java:117` (V).
- **Interfaces:** `db/compaction/AbstractCompactionStrategy.java:67` is an **abstract class** with `protected final ColumnFamilyStore cfs` (`:87`). Abstract methods: `getNextBackgroundTasks` `:185`, `getMaximalTasks` `:196`, `getUserDefinedTask` `:207`, `getEstimatedRemainingTasks`, `getMaxSSTableBytes`, `addSSTable` `:296`, `removeSSTable` `:310`, `getSSTables` `:325`. Overridable hooks: `getScanners`, `groupSSTablesForAntiCompaction` `:529`, `createSSTableMultiWriter` `:558`, `supportsEarlyOpen` `:582`, static `validateOptions` `:451`. Tasks extend `AbstractCompactionTask`/`CompactionTask`.
- **Leakage/coupling (V):**
  - The strategy receives the whole `ColumnFamilyStore`.
  - The manager special-cases built-ins: `CompactionStrategyManager.java:330, 630, 656, 679, 769` (`instanceof LeveledCompactionStrategy`) and `:807` (`instanceof UnifiedCompactionStrategy`). The same happens in `tools/StandaloneScrubber.java:209` and `cql3/statements/schema/AlterSchemaStatement.java:236` (TWCS).
  - Repaired, unrepaired, pending, and transient split ownership lives in `AbstractStrategyHolder`/`PendingRepairManager`, not in the strategy.
  - The UCS sharding and disk machinery (`ShardManager*`, `DiskBoundaries`) is not exposed as a reusable service.
- **Compatibility:** no API versioning. Options are free-form string maps validated by the strategy, and there is no TCK. Strategy changes are schema changes, propagated by TCM.
- **Grade C:** a widely used and long-lived seam, but implemented by subclassing a stateful base class bound to `ColumnFamilyStore`, with core code type-switching on concrete strategies.
- **Recommendation: formalize.**
  1. Define a `CompactionStrategy` interface over a narrow `CompactionContext` (live sstable view, disk boundaries, metrics, task factory) instead of `ColumnFamilyStore`.
  2. Replace the `instanceof` checks with capability methods such as `levelFanout()` and `supportsShardedWriter()`.
  3. Publish a strategy harness that simulates flush and compaction streams over synthetic sstable metadata (the UCS simulation tests are a starting point).
- **3a. UCS `Controller`** (`db/compaction/unified/Controller.java:45`, `public class`, `fromOptions(ColumnFamilyStore, Map)` `:426`) is concrete, not swappable, and not selectable by option. **Grade D.** Making `Controller` an interface, selected by a `controller_class` UCS option, would let operators experiment with adaptive or cost-model controllers without forking UCS. This is high value at low cost.

### 2.4 Compression (`ICompressor`, CEP-49 providers)

- **Status:** stable-public. CEP-49 "Hardware-accelerated compression" (`c68dd59d4a`, V) added the provider layer.
- **Entry points:** table `compression = {'class': ..., 'chunk_length_in_kb': ...}`, yaml `commitlog_compression`, `hints_compression`, and `compressor_providers.<compressor>: {class_name, parameters.fail_on_missing_provider}` (`conf/cassandra.yaml:3065-3080`, `config/Config.java:93`). Dictionary compression settings are at `Config.java:579-582` (V).
- **Loading:** `schema/CompressionParams.java:305-334`. The class is resolved with `classForNameWithoutInitialization`, then `CompressorRegistry.getCompressor` (`io/compress/CompressorRegistry.java:155`) delegates to a provider. `DefaultCompressionProvider` reflects on a static `create(Map)` (`io/compress/DefaultCompressionProvider.java:66`). Providers can only override built-in compressors (yaml comment and `CompressorRegistry.java:184, 228`). On failure the registry falls back to the default unless `fail_on_missing_provider` is set (`:162-175`) (V).
- **Interfaces:** `io/compress/ICompressor.java:27` (small: compress/uncompress, `preferredBufferType`, `supports(BufferType)`, `supportedOptions`, `Uses` enum `:35`), `IDictionaryCompressor.java:32`, `AbstractCompressionProvider` (V).
- **External implementations (I):** hardware-offload providers (for example Intel QAT-style accelerators) are the target of CEP-49. Custom codecs (Brotli and others) have appeared in community repos, unverified.
- **Leakage/coupling (V):**
  - `CompressionMetadata.java:500` writes `getSstableCompressor().serializedAs().getSimpleName()` into `CompressionInfo.db`. The compressor's class identity is therefore baked into every sstable, so a codec must stay on the classpath under that name for as long as such sstables exist.
  - The dictionary path is wired to `db/compression/CompressionDictionaryManager` and a messaging verb (`CompressionDictionaryUpdateVerbHandler`), so it is not pluggable.
  - The chunk format (checksums, `maxCompressedLength`) is fixed by core.
- **Compatibility:** implicit. Old sstables require the old class. There is no codec-ID registry.
- **Grade B+:** small, clean, config-selectable, with a provider override. The persisted class name is the main wart.
- **Recommendation: keep, with small hardening.**
  1. Introduce stable codec IDs, separate from class names, with an alias table.
  2. Document the `ICompressor` thread-safety and buffer contract.
  3. Add a codec conformance test (round-trip, buffer types, corruption detection).

### 2.5 Commit log: segment manager, CDC, archiver

- **Segment manager status: internal-seam (D).**
  - `db/commitlog/CommitLog.java:81` declares `public static final CommitLog instance = CommitLog.construct()`.
  - `construct()` (`:94-96`) takes `DatabaseDescriptor.getCommitLogSegmentMgrProvider()`, a static `Function<CommitLog, AbstractCommitLogSegmentManager>` defined at `config/DatabaseDescriptor.java:300` that chooses `CommitLogSegmentManagerCDC` or `CommitLogSegmentManagerStandard` based on `cdc_enabled`. A public `setCommitLogSegmentMgrProvider` exists at `:5351`, but it is a test hook with no config key.
  - `AbstractCommitLogSegmentManager.java:71` has package-private abstract `discard(...)` (`:290`), and `CommitLogSegment.java:68` has package-private abstract `write`/`flush` (`:419-421`). External implementations are therefore impossible without split packages.
  - The sync mode is a hard `switch` (`CommitLog.java:118-130`). Segment types (`MemoryMappedSegment`, `CompressedSegment`, `EncryptedSegment`, `DirectIOSegment`) are chosen internally from `commitlog_disk_access_mode`, compression, and encryption.
  - Memtable flush bookkeeping is expressed in `CommitLogPosition`, so the WAL position type leaks into the memtable and sstable metadata (`Version.hasCommitLogIntervals`).
- **CDC consumer (C-, partial).** CDC writes hard links or segments to `cdc_raw_directory` (`Config.java:497-507`). Consumers parse segments with `db/commitlog/CommitLogReader.java:57` and `readCommitLogSegment(CommitLogReadHandler, File, ...)` (`:144-161`), implementing `CommitLogReadHandler.java:25`.
  - External consumers (I, public knowledge): the Debezium Cassandra connector, DataStax CDC for Apache Cassandra, and Cassandra Analytics/Sidecar CDC (CEP-44).
  - These embed the full server jar and must initialize `DatabaseDescriptor` and schema to deserialize mutations. There is no stable consumer API, and the format is versioned only by `CommitLogDescriptor` versions.
- **Archiver (C, stable-public):** `CommitLogArchiver.java:56` reads `conf/commitlog_archiving.properties` (`archive_command`, `restore_command`, `:128-129`). It runs shell hooks, which is crude but stable.
- **Recommendation:**
  1. Create a `WriteAheadLog` SPI (allocate/append/sync/discard-up-to/replay) selected by yaml `commitlog.class_name`, and make `CommitLog` an implementation rather than a singleton.
  2. Keep `CommitLogPosition` as an opaque, comparable token.
  3. Long term, evaluate backing the commit log with `journal` (2.11).
  4. For CDC, publish a standalone `cassandra-cdc-reader` artifact with a versioned mutation-decoding API and schema supplied by the caller, not by `DatabaseDescriptor`.

### 2.6 Row cache provider

- **Status:** stable-public, small. Yaml `row_cache_class_name` (`Config.java:566`, default `OHCProvider`; `conf/cassandra.yaml:584`) is loaded at `service/CacheService.java:145-158` through `classForNameWithoutInitialization(..., CacheProvider.class)` and `newInstance()`. `NopCacheProvider` is used when the size is 0 (V).
- **Interfaces:** `cache/CacheProvider.java:20` (`ICache<K,V> create()`), `cache/ICache.java:27`. Implementations: `OHCProvider` (wraps the external OHC library, I: author Robert Stupp) and `SerializingCacheProvider` (V).
- **Leakage:** the key and value types (`RowCacheKey`, `IRowCacheEntry`) are internal, and saving/loading goes through `AutoSavingCache`. Invalidation semantics are owned by `ColumnFamilyStore`.
- **Grade B-:** a tiny, honest SPI for a feature that is rarely recommended.
- **Recommendation: keep.** Do not invest beyond docs.

### 2.7 Key cache

- `service/CacheService.java:119-129` hard-codes `CaffeineCache.create(...)` wrapped in `AutoSavingCache<KeyCacheKey, AbstractRowIndexEntry>`. Values are format-specific via `SSTableFormat.getKeyCacheValueSerializer()`, and BTI does not use the key cache.
- **Grade F as a boundary.** That is intentional, and it should remain owned by the format (BIG) rather than become a separate SPI.

### 2.8 Chunk cache

- `cache/ChunkCache.java:52-53`: `private static boolean enabled = ...; public static final ChunkCache instance = enabled ? new ChunkCache(...) : null`. `maybeWrap(ChunkReader)` (`:192`) is the only integration point. There is no SPI.
- **Grade F.** It is, however, the natural place for a **tiered cache** (for example a local-NVMe cache in front of remote sstables). See section 4, P3.

### 2.9 File system abstraction (`File`, `PathUtils`, `ChannelProxy`; CEP-36 territory)

- **Verified state:**
  - `io/util/File.java:60` holds `private static FileSystem filesystem = FileSystems.getDefault()`, swapped only via `File.unsafeSetFilesystem(FileSystem)` (`:810`). Tests use it with Jimfs (`test/unit/.../io/util/FileSystems.java:57-61`, `utils/ConfigGenBuilderTest.java:43`) and the simulator (`test/simulator/.../AccordJournalSimulationTest.java:64`).
  - `File.java:127` carries a TODO noting that the URI constructor bypasses the custom FS.
  - `io/util/ChannelProxy.java:44` is a **`final class`**. `openChannel` uses `FileChannel.open(file.toPath())` (`:61`), and `MmappedRegions.java:380` uses `channel.map(...)`. mmap requires a default-FS file channel.
  - The read stack has proper interfaces: `ChunkReader` (`io/util/ChunkReader.java:31`), `RebuffererFactory`, `Rebufferer`, `ReaderFileProxy`. `FileHandle` (`io/util/FileHandle.java:50`, builds `new ChannelProxy(...)` at `:447`) is concrete.
- **CEP-36** ("configurable ChannelProxy" / external storage aliasing) has **no traces in this tree** (no `channel_proxy` or storage-provider config; `git log --grep` finds nothing) (V). Status: **proposed**. The proposer and current state on the dev list are not verified (I).
- **Grade D:** there is a single global swap point, but it is test-only, mmap defeats it, and `ChannelProxy` is final.
- **Recommendation: new boundary.** Make `ChannelProxy` creation go through a `StorageChannelFactory` keyed by data directory (or URI scheme), so that a directory can be backed by a non-default NIO `FileSystem`. Disable mmap (`disk_access_mode`) for such directories. This is the minimum change that unlocks the remote and tiered work in section 4.

### 2.10 `Directories` / data file directories

- Yaml `data_file_directories`, `local_system_data_file_directory` (`Config.java:454, 468`) feed `db/Directories.java:113` (`DataDirectory[] paths` `:204`, `getWriteableLocation*` `:363`, `getLocationForDisk` `:314`). These are local paths only, with free-space-based choice. `Descriptor` parses `<ks>/<table>-<id>/` layouts by regex.
- **Grade D:** configurable, but not pluggable.
- **Recommendation:** make `DataDirectory` carry a storage-tier attribute and a channel factory (see 2.9), and let placement be a policy (see 2.11).

### 2.11 Disk boundaries / allocation

- `db/DiskBoundaryManager.java:40` is a concrete class that computes per-disk token boundaries from local ranges (`getDiskBoundaries(ColumnFamilyStore)` `:45`, `getDiskBoundaryValue` `:108`). It is consumed by compaction writers, `ShardManagerDiskAware`, and flush. Neither config nor an interface exists.
- **Grade F.**
- **Recommendation:** a `DataPlacementPolicy` SPI (inputs: owned ranges, data directories with tier and capacity; outputs: boundaries and target directory per writer). This is required for hot/cold tiering.

### 2.12 Disk access tuning (context)

`disk_access_mode`, `background_write_disk_access_mode`, `commitlog_disk_access_mode`, and `compaction_read_disk_access_mode` (`Config.java:131, 417, 487-488`), plus `disk_optimization_strategy` (`Config.java:617`, enum to `SsdDiskOptimizationStrategy`/`SpinningDiskOptimizationStrategy` in `DatabaseDescriptor.java:105-110, 262`). These are config enums, not boundaries, and they should stay that way.

### 2.13 `org.apache.cassandra.journal` (Accord's durable log)

- **Status:** internal-seam, but a genuinely generic library. It was added for CEP-15 (`8dc82a6369`). Its only user is `service/accord/journal/AccordJournal.java:117` (`new Journal<>("AccordJournal", directory, params, JournalKey.SUPPORT, ...)`) (V).
- **API:** `journal/Journal.java:93` `Journal<K,V>`, with the constructor at `:180` taking `(name, directory, Params, KeySupport<K>, ValueSerializer<K,V>, SegmentCompactor<K,V>)`. Supporting types:
  - `Params.java:22` (segment size, flush mode, failure policies, `userVersion` `:76`)
  - `KeySupport.java:34` (fixed-size keys, comparator, checksum)
  - `ValueSerializer.java:25`
  - `SegmentCompactor.java:23`
  - `RecordConsumer`, `RecordPointer`
  - in-memory and on-disk indexes (`InMemoryIndex`, `OnDiskIndex`)
- **Coupling (V):** small. It imports only `db.TypeSizes` (`ActiveSegment`, `EntrySerializer`) and `service.StorageService` (`Journal.java:58`, used at `:965` for `stopTransports()` on failure policy). It uses `io.util.File`, so the Jimfs and simulator FS swaps work, which the Accord simulation test exploits.
- **Compatibility:** an on-disk `Descriptor` with a journal version plus a caller-defined `userVersion`. There are no docs beyond `package-info.java`.
- **Grade B-:** clean generics and narrow dependencies, but one consumer, no stability promise, and a direct `StorageService` call.
- **Recommendation: formalize as a library.** Replace the `StorageService` call with an injected failure handler and document the on-disk format and the `userVersion` contract. Candidates for reuse include mutation-tracking or witness logs, a commit-log reimplementation (2.5), hints, and pluggable consistency providers' durable participant logs (see `08-prior-research.md` B6/B7).

### 2.14 Tries (`db/tries`, `io/tries`)

`db/tries/Trie.java:61` (abstract), `InMemoryTrie.java:48`, `MergeTrie`, `SlicedTrie`, and others, with design docs `Trie.md`/`InMemoryTrie.md`. These are used by TrieMemtable, BTI, and SAI. This is a strong internal **library**, not an extension boundary. **Keep internal.** If anything, publish it as a separately versioned module for reuse by out-of-tree formats or memtables.

---

## 3. Whole-table storage engine boundary (CASSANDRA-13474 style)

**What already exists (V).** CASSANDRA-13474/13475 landed three abstraction commits (4.0 era):

| Commit | What it abstracted | Seam | Instantiated at |
|---|---|---|---|
| `d31ed0f51b` | write path | `db/KeyspaceWriteHandler.java:23`, `db/TableWriteHandler.java:23` (`write(PartitionUpdate, WriteContext, boolean)`), `db/WriteContext.java:27` | `new CassandraKeyspaceWriteHandler(this)` (`Keyspace.java:288, 299`), `new CassandraTableWriteHandler(this)` (`ColumnFamilyStore.java:592`) |
| `9714a7c817` | streaming | `streaming/TableStreamManager.java:37` | `new CassandraStreamManager(this)` (`ColumnFamilyStore.java:593`) |
| `c5a7fcaa8e` | repair | `repair/TableRepairManager.java:34` | `CassandraTableRepairManager` |

All of them are **hard-wired**: no factory, no config. The field type in `ColumnFamilyStore.java:323` is even the concrete `CassandraTableWriteHandler`. Instagram's RocksDB engine (Rocksandra, CASSANDRA-13476) remained a fork (I, public knowledge).

**What is missing (V):**
- **Read path.** `db/ReadCommand.java:437` declares `protected abstract UnfilteredPartitionIterator queryStorage(ColumnFamilyStore cfs, ReadExecutionController)`. `SinglePartitionReadCommand.java:556-734` (`queryMemtableAndDisk*`) walks the `ColumnFamilyStore` `View` (memtables plus sstables) directly.
  - The useful abstraction already present is `db/rows/UnfilteredSource.java:31`, implemented by both `Memtable` and `SSTableReader`.
  - Precedent: virtual tables bypass `ColumnFamilyStore` entirely through `VirtualTableSinglePartitionReadCommand` (`SinglePartitionReadCommand.java:1542`) and `VirtualTablePartitionRangeReadCommand` (`PartitionRangeReadCommand.java:628`). That proves `ReadCommand`/`UnfilteredPartitionIterator` is a workable cut line.
- **Breadth of coupling:** 264 source files reference `ColumnFamilyStore`, 200 reference `SSTableReader`, and 46 do `ColumnFamilyStore.getIfExists`/`Keyspace.open(...).getColumnFamilyStore` (rg counts, V). Snapshots, truncate, import, scrub, verify, cleanup, nodetool MBeans (`ColumnFamilyStoreMBean`), SAI and 2i, the caches, and the Paxos/Accord system tables all assume CFS.
- **Choke points identified on this branch.** The branch-only `TransactionDomainGuard` sites map exactly the table-level operations that a storage engine would have to own: mutation apply (`Keyspace.applyInternal`, `CFS.apply`, `CommitLog.add`), SSTable publication (`CFS.addSSTables`, `Tracker`), import, scrub, truncate, streaming receive, and commit-log replay.

**Feasibility (inference, informed by the above).** A full CFS replacement is not feasible in one step. A staged `TableStorage` SPI is feasible:

1. **Read seam.** Change `queryStorage(ColumnFamilyStore, ...)` to call `cfs.storage().query(command, controller)`, with the default implementation being today's memtable+sstable merge. The virtual-table subclassing shows the shape.
2. **Factory.** Add a `TableStorageFactory` keyed by a schema option (`storage = {'class': ...}`, TCM-validated) that produces `TableWriteHandler` + read + `TableStreamManager` + `TableRepairManager`. That removes the three hard-wired `new Cassandra*` sites.
3. **Operations.** Operations become capability-checked (`supportsSnapshot`, `supportsImport`, `supportsCompaction`) rather than assumed. Unsupported nodetool operations fail cleanly, using the guard sites listed above.
4. **Keep the shared lanes.** Commit log, schema, and index APIs remain shared core. A storage engine plugs in **under** the coordinator and replica verbs, not beside them.

Grade today: **D**. The interfaces exist for 3 of roughly 8 concerns, and none of them is selectable.

---

## 4. Proposed new boundaries

| ID | Proposal | Unlocks | Where to cut (V references) | Effort |
|---|---|---|---|---|
| P1 | **Storage channel factory** (CEP-36 minimal) | object or remote-backed directories, encryption-at-rest layers, test FSes without static swaps | replace `final ChannelProxy` creation in `FileHandle.java:447` and `ChannelProxy.openChannel` (`:57-61`), plus the `SequentialWriter` open path, with a per-`DataDirectory` factory; disable mmap there (`MmappedRegions.java:380`) | M |
| P2 | **Data placement policy** | hot/cold tiering, per-tier compaction, heterogeneous JBOD | `DiskBoundaryManager.java:40-108`, `Directories.getWriteableLocation` (`Directories.java:363`), `ShardManagerDiskAware` | M |
| P3 | **Cache tier SPI** | local-NVMe cache for remote sstables, alternative eviction | `ChunkCache.maybeWrap` (`ChunkCache.java:192`) becomes a configured `ChunkCacheProvider` wrapping `ChunkReader` | S |
| P4 | **SSTable object lifecycle / remote SSTables** | shared-storage or disaggregated replicas, cheap bootstrap via file-handle sharing, backup offload | `Tracker.subscribe` (`db/lifecycle/Tracker.java:609`) notifications as the publish hook; `LifecycleTransaction`/`LogFile` atomicity (`LogFile.java:546`); UUID sstable ids (`Config.java:947` `uuid_sstable_identifiers_enabled`, `SSTableIdFactory`); `Descriptor` regex decoupling | L |
| P5 | **Columnar / projection-aware format** | analytics scans, compression wins | CEP-17 alone is insufficient: the read path is row-at-a-time `UnfilteredRowIterator`. Extend `UnfilteredSource` with an optional batch/projection scan (`ColumnFilter` + `RowFilter` pushdown) used by range reads and Analytics; the format then advertises the capability | L |
| P6 | **WAL SPI** (2.5) | remote/replicated WALs, journal-backed commit log, PMem | `CommitLog.instance` (`CommitLog.java:81`), `DatabaseDescriptor.java:300`, package-private abstracts in `AbstractCommitLogSegmentManager`/`CommitLogSegment` | M-L |
| P7 | **Compaction offload** | remote or serverless compaction | `AbstractCompactionTask` execution plus `SSTableMultiWriter` outputs published via P4 lifecycle; strategy chooses *what*, executor chooses *where* | L |
| P8 | **UCS controller SPI** (3a) | adaptive, cost-model, or ML-tuned compaction without forking UCS | `Controller.java:45, 426` | S |
| P9 | **Journal as public library** (2.13) | durable logs for new subsystems (consistency providers, mutation tracking) | `Journal.java:180`, remove the `StorageService` call at `:965` | S |
| P10 | **Table storage SPI** (section 3) | alternative engines (LSM variants, B-tree, RocksDB-class) per table | `ReadCommand.queryStorage` (`ReadCommand.java:437`), `ColumnFamilyStore.java:592-593`, `Keyspace.java:288` | XL, staged |

**Cross-cutting gaps:**
1. **No conformance kits.** `ant test-latest` (`build.xml:1725`, `test/conf/latest_diff.yaml`) runs the whole suite against alternate *in-tree* memtable, format, and compaction choices, which is a de facto TCK pattern. Packaging it as subclassable abstract suites for out-of-tree implementations would be the cheapest high-leverage step.
2. **No stability annotations or deprecation policy** on any storage SPI.
3. **Persisted class identities.** Compressor simple names in `CompressionInfo.db` and format names in sstable filenames make plugins permanent dependencies of the data they wrote.

**Suggested order:** P8, P9, P3, and the conformance kits (cheap, low risk). Then P1 + P2 (the foundation for tiering). Then the SSTable format hardening (2.2) and P4. Then P6/P7. P10 last, staged.
