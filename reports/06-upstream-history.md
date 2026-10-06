# 06 - Upstream history and community stance on pluggability

Scope: CEPs that touch extension boundaries, historical attempts and removals, dev@ positions, the public-API policy gap, and plugin/incubation models in analogous projects. Researched 2026-10 against the Confluence REST API, JIRA REST, Ponymail (lists.apache.org), and local `CHANGES.txt`/`NEWS.txt`/git history in `cassandra/`.

URL conventions:
- Thread: `https://lists.apache.org/thread/<tid>`
- CEP page: `https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=<id>`
- CEP index: https://cwiki.apache.org/confluence/spaces/CASSANDRA/pages/95652201/Cassandra+Enhancement+Proposals+CEP
- JIRA: `https://issues.apache.org/jira/browse/CASSANDRA-<n>`

Marks: **[UNVERIFIED]** means I did not confirm the claim from a primary source this session. **[local]** means the claim comes from the repo's CHANGES.txt, NEWS.txt or git history.

---

## TL;DR

1. **The project already ships many plug points but does not commit to them.** It has ServiceLoader SPIs for SSTable formats, compression accelerators, constraints and startup checks, plus ParameterizedClass yaml hooks for memtables, auth, snitches, SSL, crypto providers, seed providers and disk-error handling. Compatibility on those surfaces is effectively undefined. The project has no list of public SPIs. Capwell put it this way in 2025: "We don't have any process to even know what is public API" (thread `11kto120lbb2k6kj9xt9t5r9b36jm71t`).
2. **Every attempt at broad pluggability has failed.** These are the 2017 pluggable storage engine (13474/13475), CEP-18 "Improving Modularity" (withdrawn in 2021), pluggable membership (17058) and the pmem memtable (13981, abandoned). Each failed on the same grounds:
   - it introduced abstractions with no second implementation;
   - it hurt mainline velocity;
   - the semantics behind the interfaces were not defined.
3. **The successful pattern is narrow and use-case driven.** The plug points that landed are each a single boundary motivated by a concrete implementation, usually donated from DSE/Astra: CEP-9 SSLContext, CEP-11 memtable, CEP-17 SSTable format, CEP-21 TCM, CEP-49 compression accelerators and CEP-42 custom constraints. Blake Eggleston's 2017 rule still describes the consensus. A refactor must "make sense on its own" and stay "abandon-able", and pluggable storage is framed as "enabling experimentation", not as a supported product feature.
4. **Momentum since 2025 favours separating interfaces from the server.** Interest is converging on a separate interfaces/SPI artifact (Gradle subprojects, Benedict's "APIs in a separate jar"), on classloader isolation for plugins, and on an ecosystem repo (CEP-63 and CEP-65). CEP-65 explicitly *defers* the plugin SPI surface ("a legitimate and separate problem"), which leaves room for a dedicated CEP.
5. **The rules for graduating a feature are unresolved.** Experimental, beta, GA and default were debated in 2024 with no formal outcome. A 2025 vote set a release-level rule: majors are T-1 API compatible with deprecate-then-remove. Neither the vote nor the debate says whether that rule covers Java SPIs.

---

## 1. CEP catalog: entries that create or touch a boundary

The status and release values come from the CEP index's "Release" column and each page's state field, cross-checked against `CHANGES.txt` [local]. "Boundary" names the extension seam the CEP created or touched.

### 1a. Adopted / shipped

| CEP | Title | Status / release | Boundary created or touched | JIRA | Notes |
|---|---|---|---|---|---|
| 1 | Management process (sidecar) | Page now titled "Deprecated"; state "Implementing" | Out-of-process management (sidecar) | 14395 | Spawned the separate `cassandra-sidecar` repo |
| 3 | Guardrails | 4.1 | Guardrails framework; a pluggable guardrails config source | 17146 | Andrés de la Peña: the pluggable config "is not yet officially supported... might still change in a minor" (thread `15v04w8ht0c70f06cndqtlywrfxyf1jn`) |
| 7 | Storage Attached Index | 5.0 | Index SPI (`Index`, `Index.Group`) | 16052 | DSE donation |
| 8 | DataStax drivers donation | Accepted / done | Client ecosystem | - | Ecosystem, not a server SPI |
| 9 | Pluggable SSLContext | 4.1 | `ISslContextFactory` via yaml ParameterizedClass | 16666 | Clean, narrow SPI |
| 10 | Cluster/code simulations | 4.1 | Executors, time, File→Path wrapper, alternative streaming | 16921 | Test seams. Astra used the Path wrapper to send files to S3 (Henrik Ingo, 2022 thread) |
| 11 | Pluggable memtable | 4.1 | `Memtable.Factory`, configured in yaml and per table; persistent memtables allowed | 17034 | Non-goal: replacing the storage engine. 13981 (pmem) rebased onto it, then was abandoned |
| 13 | Partition denylist | 4.1 | Operational | - | Not a plug point |
| 14 | Paxos improvements | 4.1 | Consensus path | - | Not a plug point |
| 15 | Accord general-purpose transactions | 6.0 [local: CHANGES] | Accord lives in a separate repo pulled in as a submodule/library | 17092 | Precedent for an out-of-tree module released with core |
| 16 | cqlsh auth plugin | 4.1 | Python auth provider plugin for cqlsh | 16456 | |
| 17 | SSTable format API | 5.0 | `ServiceLoader<SSTableFormat.Factory>` (`DatabaseDescriptor.applySSTableFormats`) | 17056 | Jacek: the interfaces may change "even with a minor version release". Capwell accepted on the basis that it is scoped to testing. `@DeveloperApi` was proposed with no conclusion (threads `x90kpo8gbsh74hz4fwdx88s8v2cpcqh4`, `0vttvnbowz4qpmjsd62yyzkq68rtocm5`) |
| 19 | Trie memtable | 5.0 | First non-default implementation of CEP-11 | - | Shows a plug point being validated by a second implementation |
| 20 | Dynamic data masking | 5.0 | Masking functions | - | |
| 21 | Transactional Cluster Metadata | 6.0 | The page says TCM "at its core... is also pluggable, enabling Cassandra-derived projects to supply their own implementations" | 18330 | Also a natural home for plugin-presence metadata (see section 3) |
| 24 | Password validation/generation | 6.0 [local] | Pluggable password validator/generator | 17457 | |
| 25 | Trie-indexed SSTable (BTI) | 5.0 | Second implementation of CEP-17 | - | |
| 26 | Unified Compaction Strategy | 5.0 | `AbstractCompactionStrategy` (plug point dates from 1.0, CASSANDRA-1610) | - | Jeff Jirsa noted in 2024 that no production evidence had been posted |
| 28 | Spark bulk analytics | 5.0 | Separate `cassandra-analytics` repo; reads and writes SSTables outside the server | - | Out-of-tree consumer of internal SSTable code |
| 29 | NOT operator | Accepted | CQL | - | |
| 30 | ANN vector search | 5.0 | SAI index extension | - | |
| 33 | CIDR authorizer | 5.0 | `IAuthorizer` family | - | |
| 34 | mTLS authenticators | 5.0 | New `MutualTlsCertificateValidator` interface | - | Example of a new auth SPI |
| 37 | Unified repair | 6.0 | Repair scheduling | 19918 | |
| 40 | Sidecar live migration | Adopted | Sidecar | - | |
| 42 | Constraints framework | 6.0 [local] | Custom constraints via SPI | 19947, 20824 | Outcome of the 2025 SPI thread: a node fails to start if a constraint is missing |
| 43 | CREATE TABLE LIKE | 6.0 | CQL | - | |
| 44 | Kafka CDC via Sidecar | Adopted | CDC consumption out of process | - | |
| 45 | Mutation tracking | Adopted | Replication internals | - | |
| 49 | Hardware-accelerated compression | Accepted; implemented under 7.0 in CHANGES [local] | ServiceLoader accelerator plugins with fallback to the default compressor on error | 20975 | **Strongest out-of-tree plugin precedent.** The Intel QAT jar is hosted outside the ASF (Intel GitHub/Maven Central). Thread `dltwzp464xlczlnomh4nbcrkcb363312` (not read in full) |
| 50 | Authentication negotiation | Adopted | Auth mechanism negotiation | - | |
| 58 | Satellite DCs | Adopted | Replication topology | - | |
| 63 | Merge analytics and sidecar into `cassandra-ecosystem` | Accepted | Repo and ecosystem structure | - | Relevant to the question of where plugins should live |

### 1b. Under discussion, draft, or not in the index table

| CEP | Title | State | Boundary relevance |
|---|---|---|---|
| 2 | Kubernetes operator | Under discussion | Ecosystem |
| 6 | CDC v2 | Draft | CDC SPI |
| 12 | Diagnostics vtables | Under discussion | Observability |
| 27 | Generic API for exposing internal data collections | Draft | Virtual-table SPI |
| 31 | Negotiated authentication | Draft | Auth SPI |
| 32 | OpenTelemetry | The page says Accepted; tracked as 21433 | Tracing/metrics backends |
| 35 | PIP-installable cqlsh | Under discussion | Tooling distribution |
| 36 | "Configurable ChannelProxy to alias external storage locations" | Under discussion (2025 thread) | File I/O seam (`ChannelProxy`, NIO `FileSystemProvider`). A separate page, "CEP-36(DRAFT) Make Cassandra's data store on Object storage" (id 358812103), proposes a Hadoop-style SPI filesystem layer |
| 38 | CQL Management API | The page says ACCEPTED (vote thread 2025-10) | Management commands |
| 39 | Cost-based optimizer | Under discussion | Proposes a pluggable optimizer/cost model; notes that memtables, SSTables and partitioners are already pluggable |
| 41 | Unified rate limiter | Draft | Rate-limiter SPI |
| 46 | Transient replication/witnesses | Accepted (page only) | Replication |
| 47 | Tree-based configuration | Draft | Config |
| 51 | YAML include | Accepted | Config |
| 52 | Schema annotations | Accepted | Metadata that plugins could consume |
| 54 | ZSTD dictionary compression | Accepted | Compressor |
| 55 | Generated role names | In 6.0 CHANGES [local] | Auth |
| 57 | Flat keys / trie interfaces | Under discussion | Internal key interfaces |
| 60 | Flexible placements | Under discussion | The page says "placement strategy will be pluggable" with an efficient default. Thread `zzzzcz9chj720mdllw4k2nzs33mf6jzt` (not read) |
| 61 | Quota management | Under discussion | Possible SPI |
| 62 | Config management via Sidecar | Accepted | Out-of-process |
| 64 | Proxy execution | The page says Accepted | Coordinator path |
| 65 | `cassandra-utils` extraction | Target 7.0 | Explicitly out of scope: "AbstractType, Index, IAuthenticator, and the rest of the plugin SPI surface". It calls "plugin authors want to build against something smaller than cassandra-all" "a legitimate and separate problem". It says an I/O jar (SSTable/CommitLog) deserves its own CEP, and it warns of diamond-dependency risk |
| 66 | Zero-copy SSTable splitting | Draft | Page: "Pluggable components and SPIs: No change" |

### 1c. Discarded

| CEP | Title | Outcome |
|---|---|---|
| 18 | Improving Modularity | Withdrawn 2021-11. Proposed seams for schema, membership, compaction↔SSTable, failure detection and verbs. Henrik Ingo described it as "a bundle of APIs arising out of our recent work" on Astra Serverless. It produced 17044 (schema pluggability, fixed in 4.1) and 17058 (pluggable membership, never merged; Benedict warned it could conflict with removing the token ring). Discussion thread `2rv2w8m55nwwkngp149l6n229yzsj810` |
| 23 | Sparse serialization | Discarded; not boundary-relevant |

I found no CEP-22 page. CEP numbers suggested in the brief were not trusted; every number above was re-derived from the wiki.

---

## 2. Historical attempts, removals, donations

### 2a. Pluggable storage engine (2017 to present, still open)

| Ticket | What | State | Key facts |
|---|---|---|---|
| 13474 | Pluggable storage engine (umbrella) | Open since 2017 | Instagram's Rocksandra reported 3-4x lower P99 read latency. Jeremiah Jordan in 2020: "not planned for 4.0" |
| 13475 | Storage engine API design | Open, 43 comments | The design debate summarised below |
| 13476 | RocksDB engine | Open | DuyHai Doan raised the feature-matrix gap: collections, counters, LWT, MV and SASI |
| 13553 | Related subtask | Open | |
| 14117 | Refactor the read path (`StorageHandler` interface) | Open | Never landed |
| 14118 / 14116 / 14115 | Refactor write path / repair / streaming | Fixed in 4.0 | The refactor parts that "made sense on their own" landed. The pluggability part did not |

Key positions on 13475:
- **Blake Eggleston (2017-11-05).** Treat this as modularization "to enable experimentation with alternate storage ideas without having to resort to awful hacks, not 'add pluggable storage to Cassandra'". The work must stay "abandon-able". Each refactor must make sense on its own and must not make core development harder. A RocksDB engine would still face API churn in minors and still need a fork. "Cassandra is a sorted lsm, rocksdb is a sorted lsm, and we don't need 2 of them." He favoured extracting the interface from `ColumnFamilyStore`/`Keyspace` over designing a new StorageEngine API.
- **Sylvain Lebresne (2017-11-10).** The codebase's modularity is "a mess", and a pluggable storage engine is "not realistic at this point".
- **Josh McKenzie.** Estimated the work at more than a man-year.
- **Aleksey Yeschenko.** Listed what a real patch would need: abstracting Keyspace/CFS, metadata, metrics and CQL options.

dev@ threads:
- 2017-11 design proposal by Dikang Gu: `5f5c802brlk7md9q5vwnotm61qmvny27`
- Related threads: `qpvg39cp229vvy2cqxh31nplw0vrx977` and `2gqmb309fj5mnlfyvg1f8pjlty6vc2k7`
- RocksDB experiment: `pzc5vno73rgwt36swptydf80oqqxlpy4` (not read)

### 2b. Other pluggability attempts

| Ticket / thread | What | Outcome |
|---|---|---|
| 13981 | Persistent-memory (Optane) memtable | Rebased onto the CEP-11 API, then resolved **Abandoned** 2023-05 ("Optane is dead": Jonathan Ellis, Brandon Williams). Reference PR #2274. The plug point survived the hardware. This is the system working as intended |
| 13486 + thread `hgovd5qw8rzt1cz5ylvdfwrjpjgww32s` (2017-05) | IBM CAPI Flash vendor integration | Not merged in-tree. The discussion moved to the plugin model (section 3) |
| Thread `4poc8oy075p6yg1modbcdvoywyf3gsfh` (2017-10) | Pluggable CommitLog (SPDK/CAPI, Rei Odaira) | Not merged. Ariel Weisberg asked why it could not be a mode of the existing implementation |
| 17058 | Pluggable membership (from CEP-18) | Never merged |
| 17044 | Schema pluggability (from CEP-18) | Fixed in 4.1 |
| 1610 | Pluggable compaction | 1.0. The oldest and most-used plug point. LCS, DTCS, TWCS and UCS all ride on it |
| 3078 | Pluggable secondary indexes | 1.0. Later became the `Index` API used by SASI and SAI |
| 1669 | `SeedProvider` | 0.7 |
| 5401 | Internode authenticator | Shipped |
| 8096 | Pluggable cache serializers | Shipped |
| 10681 | Index build pluggability | Shipped |
| 13579 | Row cache plugin access | **[UNVERIFIED]** state |
| 18624 | Pluggable crypto providers (ACCP) | 5.0 |
| 20363 | `DiskErrorsHandler` pluggability | 6.0/7.0 [local] |
| 20824 | Custom constraints via SPI | 6.0 [local] |
| 21093 | Custom `StartupCheck` via SPI | [local] |
| 21409 | Pluggability item from 6.0/7.0 CHANGES [local] | Details **[UNVERIFIED]** |
| 21546 | Pluggable default role initialization | 7.x [local] |
| 20975 | Compression accelerator SPI (CEP-49) | 7.0 [local] |

### 2c. Removals and deprecations of pluggable things

| Item | When | Why / how | Lesson |
|---|---|---|---|
| Thrift, together with the pluggable `request_scheduler` | 4.0 (11115). Commit `4881d9c308` "Thrift removal" [local] | The protocol was removed and the scheduler plug point went with it | A plug point dies with its host subsystem |
| Non-Java/JS UDF languages | 3.x (12883) | Security and maintenance cost | |
| JavaScript UDFs | Deprecated in 4.0; scripted UDF internals removed in 5.0 (18252) with "hooks to be added later in CASSANDRA-17281". CVE-2021-44521 covered scripted UDFs | Security | **The pattern is "remove now, re-add as a hook later"** |
| `BackPressureStrategy` | Removed in 4.0 (15375) | Superseded | |
| DTCS | Removed in 5.0 (18043) | Superseded by TWCS | The compaction SPI survived; one implementation was retired |
| `index_interval` | Removed in 4.0 after deprecation in 2.0/2.1 | | Long deprecation windows are normal |
| `YamlFileNetworkTopologySnitch` | Removed (7917) | | |
| `CloudstackSnitch` | Deprecated (18438) | Unmaintained | In-tree vendor code rots. This supports the 2017 argument for keeping it out of tree |
| `IEndpointSnitch` | Deprecated in 6.0 (19488); thread `d11x08x7xfplvxxfo2j5s46owq7mpbcw` | TCM replaces it with a locator/initial-location model. Sam Tunnicliffe called it "one of the more widely used extension points"; existing custom snitches keep working | **The best precedent for migrating a widely used SPI without breaking it** |
| SASI, MVs | Moved behind experimental flags | Retroactively labelled experimental (Jon Haddad, 2024) | "Experimental" has been used as a post-hoc warning label |
| `IAuthenticator` / `IRoleManager` | Interface changes in 1.2, 2.2 and 3.x **[UNVERIFIED per-version details]** | Feature growth (roles, SASL) | The SPI has broken repeatedly, so third-party auth plugins are version-pinned in practice |

### 2d. DataStax donations and the "Astra pressure"

- CEP-18's API bundle came from Astra Serverless work on compute/storage separation and S3 (Henrik Ingo, thread `2rv2w8m55nwwkngp149l6n229yzsj810`).
- **SAI, the trie memtable, the BTI SSTable format and UCS** all came from DSE/Astra and arrived as implementations behind existing or new plug points (`Index`, `Memtable.Factory`, `SSTableFormat.Factory`, compaction strategy). The donation pattern works because the plug point and its second implementation arrive together.
- In the 2025 CEP-36 thread (2025-03-05) Mick linked DataStax's fork-side `StorageProvider`: https://github.com/datastax/cassandra/blob/main/src/java/org/apache/cassandra/io/storage/StorageProvider.java. The `RemoteStorageProvider` that would use it remains private.
- Dinesh's counterpoint: Astra usage is not evidence of maturity in OSS.

### 2e. Release-level compatibility policy (2025)

The [VOTE] "Simplifying our release versioning process" (2025-04-17, thread `3h9wnwvd171d9jl5yd7059sjvqtmx7qh`; DISCUSS thread `jy6vodbkh64plhdfwqz3l3364gsmh2lq`) passed with broad PMC +1s. Its terms:
- No more semver `.MINOR`.
- Online upgrades are supported from all GA-supported releases.
- **T-1 releases are guaranteed API compatible for non-deprecated features.**
- Deprecate in release N, remove in N+1.

Comments in the thread:
- Scott Andreas wants "a *very* high bar for user API-breaking changes".
- Josh and Benedict restated that any incompatible API change, and any addition, needs a [DISCUSS] thread.

The vote does **not** define whether Java SPIs count as "API".

---

## 3. Community stance on dev@

### 3a. Threads read

| Date | Thread (tid) | Topic | Outcome |
|---|---|---|---|
| 2017-05 | `hgovd5qw8rzt1cz5ylvdfwrjpjgww32s` | Integrating vendor-specific code and developing plugins | Consensus for plugins over in-tree vendor code. No process was adopted |
| 2017-10 | `4poc8oy075p6yg1modbcdvoywyf3gsfh` | Making CommitLog pluggable | Not pursued |
| 2021-10 | `2rv2w8m55nwwkngp149l6n229yzsj810` | CEP-18 Improving Modularity | Withdrawn |
| 2022 | `x90kpo8gbsh74hz4fwdx88s8v2cpcqh4`, `0vttvnbowz4qpmjsd62yyzkq68rtocm5` | CEP-17 SSTable API | Accepted as a "may change in minors" API |
| 2022-04 | `15v04w8ht0c70f06cndqtlywrfxyf1jn` | Pluggability improvements in 4.1 (Henrik Ingo, prompted by debates with Patrick McFadin) | Inventory of 4.1 seams: CEP-9, 10, 11, 17044, CEP-16, CEP-3 |
| 2022-12 | `3n8lb1syrsmx89d10kyqz94zzdqhz3o5` | API modifications and when to raise a thread | DISCUSS required for public API changes. The scope of "public" was left undefined |
| 2023-10 | `7mg7rwvjqzrft8g0ckk2yc3byxv2gg9b` | `@Deprecated(since, forRemoval)` | Mick: removal cannot be the default; `forRemoval` needs consensus. Doug Rohrer uses Cassandra as a library |
| 2024-05 | `o73rdvp5c3krjpk7y6nbgbt2h0471q47` | Experimental virtual tables | Restated "all new features disabled by default" |
| 2024-10 | `d11x08x7xfplvxxfo2j5s46owq7mpbcw` | IEndpointSnitch deprecation | Deprecated in 6.0, compatibility preserved |
| 2024-12 | `6vrckh4yo3k31zglzoy55y7cjbrg0qft` | Experimental flagging / feature lifecycle | **No formal outcome found** |
| 2025-03 | `r0nhyyn6...` (CEP-36) and `7x9wsn265mq5jd5q236y0nkjsw6ycbct` (Plugins and dependencies) | Remote storage; splitting out plugin interfaces | Enthusiasm, no consensus. The full CEP-36 tid is truncated in my notes **[UNVERIFIED full id]** |
| 2025-04 | `3h9wnwvd171d9jl5yd7059sjvqtmx7qh` | Versioning VOTE | Passed (section 2e) |
| 2025-08 | `11kto120lbb2k6kj9xt9t5r9b36jm71t` | Java SPI way of loading custom implementations | Stefan proceeded with SPI for constraints. A missing constraint makes the node fail to start |
| 2026-05 | `qtw0hrl3mj6l3tzz0q09kw9dhvyxs8tt` | Validating compaction and storage changes (Scott Andreas) | Josh: tools/dev "developers use at your own risk". Jaydeep: an "unofficial" area in trunk. Ekaterina: commit as experimental |
| 2026-06 | `66olocmvfzgz95dlmgkh540r2fmr4r0j` | Forking utilities | Led to CEP-65 |

Not read, but relevant:
- `8z2w751lhscmm5bzpss92f6jx0qnhj0o` (2021 release types/semver)
- `qw7bng55ol6qrbm2y4s3qb9gfl1nm5qt` (2025-11 cadence/alphas vote)
- `3vllktslsjw8vpwdn0b45wr74dp9vvyn` (2023 incremental feature work)
- `f8f4jjzt1ngpfn1dc9o3jd0fgh03gt9x` (CEP-11 discussion)

### 3b. Recurring objections

| Objection | Who / where | Representative statement |
|---|---|---|
| Premature abstraction slows the core | Benedict (CEP-18); Blake (13475) | "pluggability... can slow down development or prevent deeper integrations" |
| Interfaces without defined semantics are meaningless | Capwell (CEP-18) | You must "define semantics around the APIs and can't just create interfaces". Mocks that test atomic membership when the real implementation is not atomic make the tests meaningless |
| Single-implementation interfaces | Dinesh (CEP-18) | Also concerned about performance regressions and breaking back-compat for tools |
| Nobody knows what is public | Capwell (2021, 2025) | "We don't have any process to even know what is public API". The SSTable format is not public |
| Duplicating the core engine | Blake (13475) | "we don't need 2 of them" |
| Feature-matrix gaps in alternative engines | DuyHai Doan (13476) | Collections, counters, LWT, MV and SASI |
| "We can't remove things" | Jon Haddad (2024-12) | Strongly against early integration of immature features |
| Side-loaded plugins diverge across nodes | Capwell (2025-08) | A memtable implementation missing on some nodes was a pain point; plugin presence is not stored in TCM |
| Too much process cost | Benjamin Lerer, Andrés (2022-12) | A DISCUSS per API change would have meant about 70 threads for 4.1 |
| Dependency sprawl and vendor bias | Stefan, Benedict (CEP-36 2025) | "Why S3?" Licensing concerns. Split generic FS registration from remote semantics |
| Multi-repo overhead | Dinesh, Jeremiah (2025, 2026) | "Why not a folder in the same repo?" |

### 3c. Proponents' arguments

| Argument | Who / where |
|---|---|
| Vendors need a path that is not "fork or merge" | Jeff Jirsa (2017): "we should do more than just say 'the API exists, ship your own jar'". He likes the spark-packages and httpd models and set up a ppc64le Jenkins job |
| Plugins should not be tied to the release cycle | Jon Haddad (2017): spark-packages; Python's "standard lib is where modules go to die"; Rust nursery |
| Plugins declare "tested with X.Y.Z" and run in CI every release | J.D. Jordan (2017) |
| Three tiers: listing, contrib, in-tree; plus component maintainers | Jason Brown (2017): Mesos-style maintainers |
| Benefits outweigh costs if interfaces are derived from use cases or implementations | Josh McKenzie (CEP-18) |
| A separate API jar defines what is public | Benedict (2025-03): APIs in a separate jar/source tree would solve "how we define what our public APIs are". Plugins need their own classloader, possibly reusing the UDF loading logic. Modules within Cassandra proper should stay in the same repo |
| Gradle subprojects (accord, interfaces, server, sstable-reader) and a separate classpath | Jon Haddad (2025, 2026). Cites his snakeyaml conflict (CASSANDRA-19663) |
| Break out the interfaces first | Joel Shepherd (Amazon), Josh (2025-03). "plenty of enthusiasm... shortage of consensus" |
| Separate public SPIs from public APIs | Mick (2025-08); he also proposed categorising changes (2022-12) |
| A reusable plugin ecosystem repo | Dinesh (2025-08), for constraints |
| Extend, but do not break | Stefan Miklosovic (CEP-18): tooling relies on the fat jar |
| Release independently of the major cycle | Capwell (2026-06), for utilities |

### 3d. Feature-maturity ladder (unresolved)

Settled positions:
- **Experimental and disabled by default.** "We agreed a long time ago that all new features are disabled by default" (Capwell, 2024-05). J.D. Jordan: the project agreed that experimental features behind flags are fine in stable releases.

Positions from the 2024-12 debate (`6vrckh4yo3k31zglzoy55y7cjbrg0qft`):
- **Josh McKenzie** proposed Beta → GA → Default. Every new feature would spend one major in beta, and graduation would need a DISCUSS thread.
- **Aleksey** distinguished three meanings of "experimental":
  - retroactively broken;
  - flawed;
  - genuinely experimental and opt-in, which "might not make it at all".
- **Mick** wants both beta and experimental, and experimental features should be removable.
- **Ben Slater** said experimental and beta are distinct.
- **Dinesh** favours feature flagging.
- **Outcome.** No formal policy was found. The Release Lifecycle page (https://cwiki.apache.org/confluence/display/CASSANDRA/Release+Lifecycle) defines alpha, beta, RC and GA for **releases only**. It allows interface and format changes in alpha and says nothing about feature-level maturity or SPIs.

### 3e. The public-API policy gap (three documents disagree)

| Source | What it counts as compatibility-relevant |
|---|---|
| CEP index template, "Compatibility Concerns" (https://cwiki.apache.org/confluence/spaces/CASSANDRA/pages/95652201) | Includes "pluggable components (SPIs) like authorisation, triggers" |
| Website Code Style, "Public APIs" (https://cassandra.apache.org/_/development/code_style.html) | "CQL, virtual tables, JMX, yaml, system properties, etc." **Java SPIs are not listed.** API-strategy changes go to dev@ |
| CEP-17 thread / Capwell 2025 | SPIs such as SSTableFormat are explicitly *not* public and may change in minors |

Effect: a plugin author cannot tell which interfaces are stable. A separate SPI artifact with stability annotations would settle the question mechanically instead of by policy text. Benedict (2025-03) made this point directly.

---

## 4. Analogous projects: incubation and graduation mechanisms

**Verification status.** The mechanisms below are summarised from each project's canonical documentation as I knew it before this session. **I did not re-fetch these pages this session.** The URLs are the canonical doc locations, and every row should be treated as **[UNVERIFIED this session]** until spot-checked.

| Project | Plugin mechanism | Stability signalling | Incubation → graduation path | Canonical URL |
|---|---|---|---|---|
| PostgreSQL | Extensions (`CREATE EXTENSION`, versioned upgrade scripts), hooks, Table Access Methods (PG12+), FDW, custom WAL resource managers | The C API is unversioned. Extensions recompile per major release | `contrib/` ships with core but is optional. Mature modules can move into core (e.g. `pg_stat_statements` stayed in contrib; full-text search started as contrib tsearch2 and merged in 8.3). The out-of-tree ecosystem (PGXN, Citus, TimescaleDB) proves the hooks are real | https://www.postgresql.org/docs/current/extend-extensions.html ; https://www.postgresql.org/docs/current/tableam.html |
| Apache Lucene | `Codec`/`PostingsFormat` SPI via ServiceLoader with **versioned names**; old codecs move to `backward-codecs` | `@lucene.experimental` and `@lucene.internal` javadoc tags | The `sandbox` module holds new ideas, which graduate to core or other modules | https://lucene.apache.org/core/ (javadocs for `org.apache.lucene.codecs`) |
| Apache Kafka | Connect `plugin.path` with **isolated classloaders** per plugin; pluggable authorizer, quota and tiered-storage `RemoteStorageManager` (KIP-405) | Public interfaces are defined per KIP. `@InterfaceStability.Evolving/Unstable/Stable` annotations | The KIP process. Features ship as early access or preview, then become production-ready (e.g. KRaft, tiered storage) | https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Improvement+Proposals ; https://kafka.apache.org/documentation/#connect_plugindiscovery |
| Apache Hadoop | Pluggable FileSystem (`fs.<scheme>.impl` with ServiceLoader), schedulers, codecs | `@InterfaceAudience.Public/LimitedPrivate/Private` × `@InterfaceStability.Stable/Evolving/Unstable`; a documented compatibility guide | Features land behind config flags and are documented as alpha or beta in release notes | https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/InterfaceClassification.html ; https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/Compatibility.html |
| Apache Spark | DataSource V2 (`TableProvider`, catalog plugins), `spark.plugins`, spark-packages | `@Experimental`, `@DeveloperApi`, `@Evolving`, `@Stable` | Third-party packages are listed on spark-packages.org (cited by Jirsa and Haddad in 2017). APIs are promoted by removing the annotations | https://spark.apache.org/docs/latest/ ; https://spark-packages.org |
| Elasticsearch / OpenSearch | Plugins installed via a CLI, each with its own classloader | Plugins pin to the **exact** engine version | OpenSearch uses a `sandbox/` area in the main repo for experimental modules, then promotes them to `modules/` or `plugins/` | https://opensearch.org/docs/latest/install-and-configure/plugins/ |
| RocksDB | Pluggable options: memtable rep, table factory, compaction filter, merge operator, `Env`/`FileSystem` | The C++ API is not stable across majors | Contrib-style `utilities/`. Rocksandra was a fork, not a plugin | https://github.com/facebook/rocksdb/wiki |
| FoundationDB | The "layers" model: the core stays minimal and features live in client-side layers | Versioned client API (`api_version`) | Layers mature out of tree (Record Layer, Document Layer) | https://apple.github.io/foundationdb/layer-concept.html |
| Linux kernel | `drivers/staging/` | Staging carries explicit TODO files | Code moves out of staging once quality bars are met, or is deleted if it goes unmaintained. **A deletion path is built in** | https://www.kernel.org/doc/html/latest/ (staging docs) |
| Rust | rust-lang-nursery (cited by Haddad in 2017); now unstable features gated by `#![feature]` on nightly | Stability attributes on every API | The RFC process; nightly feature gate, then stabilisation | https://rustc-dev-guide.rust-lang.org/stability.html |
| Apache Mesos / ASF Incubator | The Mesos component-maintainers model (Jason Brown, 2017). The ASF Incubator is a governance-level incubation model | - | The Incubator applies graduation criteria to whole projects | https://incubator.apache.org/policy/process.html |

Mechanisms that transfer to Cassandra:
1. **Stability annotations on SPIs.** Hadoop, Spark, Kafka and Lucene all have them. `@DeveloperApi` was proposed in the CEP-17 thread and dropped. This is the cheapest fix for the gap in section 3e.
2. **A separate interfaces artifact.** Hadoop and Kafka keep client/API jars separate from server jars. This matches Benedict's and Haddad's 2025 proposals and the gap CEP-65 left open.
3. **Classloader isolation.** Kafka Connect `plugin.path` and Elasticsearch plugins do this. It addresses CASSANDRA-19663-style dependency conflicts, and Benedict suggested reusing the UDF classloader.
4. **A sandbox area with a deletion path.** Lucene `sandbox`, OpenSearch `sandbox/` and Linux `staging/` all have one. This matches the 2026 "tools/dev, use at your own risk" and "unofficial area in trunk" suggestions and Mick's "experimental must be removable".
5. **Versioned SPI names.** Lucene codec names carry versions. Cassandra's SSTable format version strings already behave this way, and the idea could generalise to other SPIs.
6. **A listing or ecosystem tier above out-of-tree plugins.** spark-packages and PGXN are examples. This matches Jason Brown's three tiers and CEP-63's `cassandra-ecosystem`.

---

## 5. Implications for the pluggable-boundaries roadmap

These are recommendations inferred from the history above, not project decisions.

1. **Lead with a "public SPI" CEP before any new seam.** It would publish an explicit list of supported SPIs, add stability annotations (Stable, Evolving, Experimental), and tie them to the 2025 T-1 compatibility and deprecate-then-remove rule. This is the gap everyone from Capwell to Mick to Dinesh has named since 2021. CEP-65 deferred it. It can be done without moving any code.
2. **Then publish an `cassandra-spi` (interfaces) artifact as a Gradle subproject in the same repo.** This addresses Benedict's and Jeremiah's preference for one repo and Haddad's subprojects. Plugin authors would stop compiling against `cassandra-all`, which was Joel Shepherd's complaint.
3. **Plugin classloader isolation plus cluster-wide presence checks.** Each plugin gets its own classloader, possibly by reusing the UDF loader. TCM would record which plugins and versions are required, so a node missing a plugin fails fast. That extends the constraints precedent and fixes Capwell's memtable pain point.
4. **Every new seam needs two implementations and contract tests.** That means a default implementation plus a second one, either real or a test implementation honouring the same semantics. The tests must exercise the interface contract, not mocks. This answers Dinesh's single-implementation objection and Capwell's semantics objection. CEP-11 with CEP-19 and CEP-17 with CEP-25 are the models.
5. **Refactors must stay "abandon-able".** Every extraction must improve the core on its own terms (Blake 2017). This explains why the 14115/14116/14118 refactors landed and the storage-engine SPI did not.
6. **Adopt a feature/SPI lifecycle (Experimental → Beta → GA → Default) with a removal path for Experimental.** This would resolve the open 2024-12 thread, and it can reuse the existing "disabled by default" consensus.
7. **Host vendor and accelerator plugins out of tree, with an ASF listing and CI.** CEP-49's Intel QAT pattern plus J.D. Jordan's "tested with X.Y.Z in ASF CI" would give vendors a path that is neither a fork nor an in-tree merge. CloudstackSnitch shows what happens to in-tree vendor code.
8. **Migrate widely used SPIs the IEndpointSnitch way.** Deprecate the old interface, keep existing implementations working through an adapter, and replace it with a TCM-native model.
9. **Do not lead with "pluggable storage engine".** It is the most-rejected framing in the project's history. Present storage work as narrow seams instead: file system/ChannelProxy (CEP-36, split per Benedict's proposal), memtable, SSTable format and compaction.

---

## 6. Unverified or incomplete items

- Every row in section 4 (analogous projects) is from documentation not re-fetched this session.
- The full thread id for the CEP-36 ChannelProxy discussion (`r0nhyyn6...`) was truncated in my notes.
- The details of CASSANDRA-21409 and the current state of 13579 were not confirmed.
- The per-version break details for `IAuthenticator`/`IRoleManager` (1.2, 2.2 and 3.x) were inferred from history, not diffed.
- CEP-32, CEP-38 and CEP-64 "Accepted" states come from the pages themselves. I did not cross-check vote threads, except CEP-38 (vote 2025-10).
- These threads were not read: 2021 release types, 2025-11 cadence vote, 2023 incremental feature work, CEP-11 discuss, CEP-49 discuss, CEP-60 discuss, 2017 RocksDB experiment.
- I found no formal outcome for the 2024-12 experimental/beta thread. A later decision may exist that I did not locate.
- The PostgreSQL contrib-to-core examples (tsearch2 merged in 8.3) are from memory.
