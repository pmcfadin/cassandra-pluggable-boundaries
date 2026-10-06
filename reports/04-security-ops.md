# 04 — Pluggable Boundaries: Security, Observability, and In-Node Operations

Source: `cassandra/` at branch `codex/pluggable-consistency-prototype` (HEAD `5afdb580a7`, merge-base with trunk `0691eb3319`).
**Branch-only check:** `git diff --name-only 0691eb3319 HEAD` touches nothing in `auth/`, `security/`, `audit/`, `fql/`, `diag/`, `metrics/`, `tracing/`, `config/`, `db/guardrails/`, `tools/nodetool/`. Only `tcm/transformations/Startup.java` is near this area, and it is unrelated. **So every finding below is trunk behavior, not a branch artifact.** Paths are relative to `src/java/org/apache/cassandra/` unless prefixed `conf/` or `bin/`.

Legend: **[F]** fact verified in source; **[I]** inference or recollection that was not checked against the source tree (especially external implementations).

---

## 0. Cross-cutting facts about how plugins load today

- **There is no plugin framework.** Every boundary loads by reflection from the flat JVM classpath. `bin/cassandra.in.sh` builds `CLASSPATH` from `lib/*.jar` plus `$EXTRA_CLASSPATH` (line ~62). There is no `plugins/` directory, no classloader isolation, no version handshake, and no manifest. [F]
- **There are five loader idioms, with different constructor contracts:**
  1. `ParameterizedClass.newInstance(pc, searchPackages, expectedType)` (`config/ParameterizedClass.java:80`). It tries a `Map` constructor first, then a no-arg constructor (`:119-126`). Used by auth (`auth/AuthConfig.java:169-199`) and by mTLS validators (`auth/MutualTlsAuthenticator.java:103`, `auth/MutualTlsInternodeAuthenticator.java:106`).
  2. `FBUtilities.newXxx(className, Map)` requires a `Map` constructor and prepends a default package when the name has no dot. Variants: audit `utils/FBUtilities.java:685`, SSL `:702`, crypto `:722`.
  3. `FBUtilities.construct(className, readable, expectedType)` (`utils/FBUtilities.java:877/890`) requires a no-arg constructor or a static `instance` field. Used by every `-Dcassandra.custom_*` hook.
  4. Hand-rolled `getConstructor(X.class)`: KeyProvider takes `TransparentDataEncryptionOptions` (`security/CipherFactory.java:76-78`); SeedProvider takes `Map` (`config/DatabaseDescriptor.java:1687-1689`); ValueValidator/ValueGenerator take `CustomGuardrailConfig` (`db/guardrails/ValueValidator.java:137`, `ValueGenerator.java:153`).
  5. `java.util.ServiceLoader`, used in only three places: `StartupCheck` (`service/StartupChecks.java:172`), `SSTableFormat.Factory` (`config/DatabaseDescriptor.java:2006`), and `ConstraintProvider` (`cql3/constraints/ConstraintResolver.java:41`).
- **Validation is inconsistent.** The `expectedType`/`classForNameWithoutInitialization` type check (`utils/FBUtilities.java:772-800`) is applied fairly widely. Default-package resolution differs per loader: auth searches `""` and `org.apache.cassandra.auth`; audit uses `org.apache.cassandra.audit.`; SSL and crypto use `org.apache.cassandra.security.`; guardrail validators use `org.apache.cassandra.db.guardrails`; the seed provider requires an FQCN. [F]
- **Precedent for isolation exists but is unused for plugins.** `triggers/CustomClassLoader.java:47` (`URLClassLoader` over `conf/triggers/`) and `security/ThreadAwareSecurityManager.java:48` (UDF sandbox) show the codebase already knows how to do both. [F]
- **System properties vs yaml.** Ops hooks (tracing, guardrails provider, disk errors, query handler, config loader, storage and stream hooks, clocks, MBean wrapper) are set only through `-D` properties (`config/CassandraRelevantProperties.java:133-135,200,204,207-210,414,550,601,605`). They are invisible to `system_views.settings`, cannot be reloaded, and are undocumented in `cassandra.yaml`. [F]

---

## 1. `cassandra.yaml` class entries — full enumeration and area ownership

Config field lines are in `config/Config.java`.

| yaml line | key | default class | Config field | Area owner |
|---|---|---|---|---|
| 125 | `hints_compression` (commented) | LZ4Compressor | `:544` | storage/IO (other) |
| 198-205 | `authenticator` (+ mTLS `validator_class_name`) | AllowAllAuthenticator | `:86` | **this doc** |
| 218 | `authorizer` | AllowAllAuthorizer | `:87` | **this doc** |
| 234 | `role_manager` | CassandraRoleManager | `:88` | **this doc** |
| 256 | `default_role_initializer` (commented) | PasswordDefaultRoleInitializer | `:89` | **this doc** (new, CASSANDRA-21546) |
| 276 | `network_authorizer` | AllowAllNetworkAuthorizer | `:91` | **this doc** |
| 292 | `cidr_authorizer` | AllowAllCIDRAuthorizer | `:92` | **this doc** |
| 420 | `partitioner` (string) | Murmur3Partitioner | `:118` | data model/cluster (other) |
| 584 | `row_cache_class_name` | OHCProvider | `:566` | storage/caches (other) |
| 696 | `commitlog_compression` | — | `:484` | storage/commitlog (other) |
| 755-760 | `seed_provider` | SimpleSeedProvider | `:130` | **this doc** (ops); cluster-membership owners should also review |
| 843-846 | `memtable.configurations.*.class_name` | SkipList/Trie | `:274` | storage engine (other, CEP-11) |
| 1074-1082 | `internode_authenticator` (+ validator) | AllowAllInternodeAuthenticator | `:294` | **this doc** |
| 1267 | `default_compaction` | STCS | `:1301` | compaction (other) |
| 1621 / 1675 / 1692 / 1700 | `endpoint_snitch`, `initial_location_provider`, `node_proximity`, `addresses_config` (strings) | — | `:510-520` | topology/locator (other) |
| 1770-1773 | `crypto_provider` | DefaultCryptoProvider | `:90` | **this doc** |
| 1821 / 1891 | `server_/client_encryption_options.ssl_context_factory` | DefaultSslContextFactory | `config/EncryptionOptions.java:123` | **this doc** |
| 1937-1995 | `jmx_server_options.authorizer`, `login_config_name` | AuthorizationProxy / JAAS `CassandraLogin` | `:528` | **this doc** |
| 2054-2055 | `transparent_data_encryption_options.key_provider` | JKSKeyProvider | `:491` | **this doc** |
| 2205-2206 | `audit_logging_options.logger` | BinAuditLogger | `:816`; `audit/AuditLogOptions.java:43` | **this doc** |
| 2636-2642 | `role_name_policy.validator_class_name` / `generator_class_name` | none / UUIDRoleNameGenerator | `:1104` | **this doc** (guardrail, CEP-55) |
| 2662-2667 | `password_policy.validator_class_name` / `generator_class_name` | CassandraPasswordValidator/Generator | `:1103` | **this doc** (guardrail, CEP-24) |
| 2809 | `startup_checks` (config map, not a class) | — | `:1090` | **this doc** |
| 2922 / 3029 | `auto_repair...token_range_splitter.class_name` | RepairTokenRangeSplitter | (auto-repair config) | repair (other) |
| 3065-3079 | `compressor_providers` | DefaultCompressionProvider | `:93` | storage/compression (other) |
| — (not in yaml) | `back_pressure_strategy` | — | `:782` | messaging (other; likely vestigial [I]) |

`Config.java` `ParameterizedClass`/class-string fields not exposed in yaml: `failure_detector` (`:523`), also owned by other areas.

---

## 2. Boundary catalog

### 2.1 IAuthenticator (password, mTLS / CEP-34, mTLS+password fallback)
- **Status:** stable-public.
- **Entry and loading:** `authenticator: {class_name, parameters}` → `AuthConfig.applyAuth()` → `authInstantiate` (`auth/AuthConfig.java:65,169-190`). Short names resolve against `org.apache.cassandra.auth`.
- **Interface:** `auth/IAuthenticator.java:34`. Required methods: `requireAuthentication` `:40`, `protectedResources` `:67`, `validateConfiguration` `:97`, `setup` `:104`, `newSaslNegotiator(InetAddress)` `:127`, `legacyAuthenticate` `:172`. Defaults: `supportsEarlyAuthentication` `:57`, `getSupportedRoleOptions`/`getAlterableRoleOptions` `:75/:87`, `getAuthenticateMessage(ClientState)` `:112`, cert-aware `newSaslNegotiator(..., Certificate[])` `:144`, `getSupportedAuthenticationModes` `:154`. Nested `SaslNegotiator` `:198-239` and `AuthenticationMode` `:256`.
- **In-tree implementations:** `AllowAllAuthenticator`, `PasswordAuthenticator` (bcrypt, `CredentialsCache` `auth/PasswordAuthenticator.java:362`), `MutualTlsAuthenticator` (`auth/MutualTlsAuthenticator.java:76`, `IdentityCache` `:257`), `MutualTlsWithPasswordFallbackAuthenticator` (`auth/MutualTlsWithPasswordFallbackAuthenticator.java:37`).
- **mTLS sub-SPI:** `MutualTlsCertificateValidator` (`auth/MutualTlsCertificateValidator.java:32`, methods `isValidCertificate` `:49`, `identity` `:65`). It loads from the `validator_class_name` parameter (`MutualTlsAuthenticator.java:103`). Shipped implementation: `SpiffeCertificateValidator` `:51`. Optional `MutualTlsCertificateValidityPeriodValidator` (CASSANDRA-18951).
- **External implementations [I]:** Instaclustr `cassandra-ldap` (LDAPAuthenticator); Instaclustr `cassandra-kerberos`; DataStax DSE unified auth (proprietary, not OSS-pluggable); assorted SAML/OIDC experiments.
- **Leakage [F]:**
  - The interface imports `service.ClientState` and `transport.messages.AuthenticateMessage`, so the native protocol leaks into the SPI.
  - It throws Cassandra `exceptions.*`.
  - `AuthConfig` hard-codes `instanceof PasswordAuthenticator || instanceof MutualTlsAuthenticator` for cache-config warnings (`:70`). It also hard-requires `CassandraRoleManager` when `PasswordAuthenticator` is used (`:107-108`), so a third-party password-compatible authenticator cannot reuse that coupling rule cleanly.
  - mTLS identity → role resolution goes through the global `DatabaseDescriptor.getRoleManager().roleForIdentity` (`MutualTlsAuthenticator.java:270`).
- **Compatibility:** evolved by adding `default` methods (cert-aware SASL, auth modes, early auth). That is binary compatible in practice, but nothing documents a promise.
- **Grade: B.** It is the most mature SPI in the area and has a real external ecosystem. It loses points for transport-type leakage and `instanceof` special-casing.
- **Recommendation:** freeze a documented "auth SPI v1" (javadoc `@since`, compatibility policy). Replace the `instanceof` checks with capability methods (`usesCredentialsCache()`, `requiredRoleManager()`). Move `AuthenticateMessage` construction out of the interface.

### 2.2 IAuthorizer
- **Status:** stable-public. `authorizer:` → `AuthConfig.java:83`. Interface `auth/IAuthorizer.java:30` extends `AuthCache.BulkLoader` (`auth/AuthCache.java:451`). Methods: `authorize` `:53`, `grant`/`revoke`/`list` `:72/:92/:114`, `revokeAllFrom`/`revokeAllOn` `:127/:138`, `protectedResources` `:145`, `validateConfiguration` `:152`, `setup` `:159`.
- **In-tree:** `AllowAllAuthorizer`, `CassandraAuthorizer` (stored in `system_auth.role_permissions`).
- **External [I]:** Instaclustr LDAP-group authorizers; Apache Ranger-style plugins (no known OSS Ranger plugin for Cassandra was verified).
- **Leakage:** `IResource` and `Permission` are closed enums/hierarchies, so plugins cannot add new resource types or permissions without forking. It also uses `utils.Pair`.
- **Grade: B−.** Clean for "store grants elsewhere"; closed for "new permission model".
- **Recommendation:** make `IResource` registration open (a resource-type registry). Consider a policy-decision hook (ABAC) separate from grant storage. See §3.4.

### 2.3 IRoleManager + IDefaultRoleInitializer (new)
- **Status:** IRoleManager stable-public; IDefaultRoleInitializer stable-public and new on trunk (CASSANDRA-21546, commit `bd22f9d3ee`, `CHANGES.txt:5`; **not** branch-only).
- **IRoleManager:** `auth/IRoleManager.java:38`. It has 20+ methods, including role CRUD `:73-176`, `isSuper`/`canLogin` `:216/:227`, `getCustomOptions` `:238`, `setup(boolean)` `:277`, and identity defaults `roleForIdentity`/`addIdentity`/`dropIdentity` `:286-328` (mTLS). It also exposes `defaultRoleInitializer()` `:333`.
- **IDefaultRoleInitializer:** `auth/IDefaultRoleInitializer.java:38`, with `createDefaultRole` `:52`, `validateConfiguration` `:68`, `supportsRoleManager(IRoleManager)` `:86`, and a static `validateSupportedParams` helper `:88`. Implementations: `PasswordDefaultRoleInitializer` `:49`, `MutualTlsDefaultRoleInitializer` `:39`. It is instantiated before the role manager (`AuthConfig.java:95-100`) and pairing is validated at `:160`.
- **Leakage:** returns `transport.messages.ResultMessage` (`createRoleWithResult` `:87`). The identity-mapping API was bolted on as defaults, and `CassandraRoleManager` is the only realistic implementation.
- **External [I]:** LDAP role managers (Instaclustr) usually extend `CassandraRoleManager` instead of implementing the interface. That is evidence the interface is too large.
- **Grade: C+.** Pluggable on paper, but the surface is fat, and the identity and default-role concerns are entangled.
- **Recommendation:** split it into role *store*, role *membership resolution*, and *identity mapping* sub-SPIs. Keep `IDefaultRoleInitializer` as the template for small, focused SPIs (it is the best-shaped new interface in this area).

### 2.4 INetworkAuthorizer (DC-level) and ICIDRAuthorizer (CIDR)
- **Status:** both stable-public. Loading: `AuthConfig.java:120,133`. Both throw if authorization is required while authentication is not (`:124,:137`).
- **INetworkAuthorizer:** `auth/INetworkAuthorizer.java:23`. Small: `authorize(RoleResource)→DCPermissions` `:44`, `setRoleDatacenters` `:46`, `drop` `:52`.
- **ICIDRAuthorizer:** `auth/ICIDRAuthorizer.java:30`. It forces implementers to return concrete internals: `CIDRGroupsMappingManager` `:48` and `metrics.CIDRAuthorizerMetrics` `:50`. It also forces `initCaches` `:46` and `loadCidrGroupsCache` `:79`. Base class `AbstractCIDRAuthorizer`.
- **Grade: B (network), D+ (CIDR).** The CIDR SPI is effectively "subclass `AbstractCIDRAuthorizer`", because it leaks the in-tree cache, mapping manager, and metrics types.
- **Recommendation:** shrink ICIDRAuthorizer to `hasAccessFromIp(role, ip)` plus lifecycle. Move cache and metrics into a decorator owned by core. Long term, fold DC/CIDR/resource authorization into one "connection admission policy" SPI (§3.4).

### 2.5 Auth caches (AuthCache family)
- **Status:** internal-seam. `auth/AuthCache.java:62` (protected constructors `:118,:161`). Subclasses: `RolesCache`, `PermissionsCache`, `NetworkPermissionsCache`, `CIDRPermissionsCache`, `PasswordAuthenticator.CredentialsCache`, `MutualTlsAuthenticator.IdentityCache`, `AuthorizationProxy.JmxPermissionsCache` (`auth/jmx/AuthorizationProxy.java:580`). Plugins interact only through `BulkLoader.bulkLoader()` (`:451-457`, warm-up). The knobs `*_validity`, `*_update_interval`, `*_cache_max_entries`, `*_cache_active_update` live at `Config.java:95-112`. Exposed via JMX `AuthCacheMBean` and `AuthCacheService`.
- **Gap:** a plugin cannot get an invalidation event (for example, an LDAP group change). It can only wait for TTL or call JMX.
- **Grade: C.** Usable, but invisible to plugins except for bulk-load.
- **Recommendation:** add a public `AuthInvalidationBus` (an invalidate-by-role API callable from plugins), and expose cache sizing per plugin.

### 2.6 IInternodeAuthenticator
- **Status:** stable-public. `internode_authenticator:` (`Config.java:294`) → `AuthConfig.java:113`. Interface `auth/IInternodeAuthenticator.java:27`, with default `authenticate(addr, port)` `:39`, cert-aware `authenticate(addr, port, Certificate[], InternodeConnectionDirection)` `:62`, `validateConfiguration` `:73`, and `setupInternode` `:80`. Implementations: `AllowAllInternodeAuthenticator`, `MutualTlsInternodeAuthenticator` (`:83`, validator via `:106`).
- **Leakage:** minimal. It uses only JDK types plus `ConfigurationException`.
- **Grade: A−.** Small, JDK-typed, and evolved with defaults. This is the model shape.
- **Recommendation:** keep it. Document it as a reference SPI.

### 2.7 ISslContextFactory (`ssl_context_factory`)
- **Status:** stable-public (CASSANDRA-16666). Configured per `server_/client_encryption_options` (and `jmx_encryption_options`) → `EncryptionOptions.java:123,340` → `FBUtilities.newSslContextFactory` (`utils/FBUtilities.java:702`, `Map` constructor, default package `org.apache.cassandra.security`). Interface `security/ISslContextFactory.java:55`:
  - `createJSSESslContext(boolean)` `:67` is `@Deprecated(since="5.1")`.
  - The `ClientAuth` overloads are at `:77`.
  - `createNettySslContext(...)` `:103/:117`.
  - Hot reload: `initHotReloading` `:136`, `shouldReload` `:145`.
  - `hasKeystore`/`hasOutboundKeystore` `:152/:162`, `getAcceptedProtocols`/`getCipherSuites` `:173/:180`.
- **In-tree:** `DefaultSslContextFactory` → `FileBasedSslContextFactory` (JKS/PKCS12), `PEMBasedSslContextFactory`, `DisableSslContextFactory`. All extend `AbstractSslContextFactory`.
- **Leakage [F]:** the SPI imports Netty `io.netty.handler.ssl.SslContext` and `CipherSuiteFilter` (`:28-29`). Implementers are therefore pinned to Cassandra's bundled Netty version. Common SSL options are injected into the same `Map` as plugin parameters, which risks collisions (validation at `EncryptionOptions.java:288-295`).
- **External [I]:** Vault/KMS-backed SSL factories (Kubernetes cert-manager integrations, Instaclustr). The Apache Cassandra Sidecar ships its own keystore handling rather than this SPI.
- **Grade: B−.** Real and used, with hot-reload. The Netty type leak is the compatibility hazard.
- **Recommendation:** offer a JDK-only variant (`KeyManagerFactory`/`TrustManagerFactory` providers) and let core build the Netty context. That decouples plugins from Netty upgrades.

### 2.8 Crypto provider (`crypto_provider`, Amazon Corretto)
- **Status:** stable-public (CASSANDRA-18624). Flow: `DatabaseDescriptor.applyCryptoProvider()` (`config/DatabaseDescriptor.java:1642-1676`) → `FBUtilities.newCryptoProvider` (`:722`, `Map` constructor, unmodifiable params). A system property overrides yaml: `cassandra.crypto_provider_class_name` (`CassandraRelevantProperties.java:204`, applied `:1651-1653`). Abstract base `security/AbstractCryptoProvider.java:36` (`getProviderName` `:66`, `getProviderClassAsString` `:73`, `installator` `:80`, `isHealthyInstallation` `:87`, `install`/`uninstall` `:96/:186`). In-tree: `DefaultCryptoProvider` (Corretto, `security/DefaultCryptoProvider.java:33`), `JREProvider` (`security/JREProvider.java:27`, used when the yaml key is absent, `:1647-1648`). `fail_on_missing_provider` defaults to false (`:1662`).
- **Note:** it is an abstract class, not an interface, and installs into the JVM-global `java.security.Security` provider list at position 1.
- **External [I]:** BouncyCastle FIPS and other FIPS-provider wrappers are the obvious candidates. None verified.
- **Grade: B.** Narrow and well-scoped. The global JVM side effects are inherent.
- **Recommendation:** document FIPS mode as a first-class use case. Expose the active provider in `system_views.settings` or a vtable.

### 2.9 Transparent data encryption / KeyProvider
- **Status:** partial. `transparent_data_encryption_options.key_provider` (`config/TransparentDataEncryptionOptions.java:31`) → `security/CipherFactory.java:75-78`. The constructor **must** take `TransparentDataEncryptionOptions`, which is a Cassandra config class, not a `Map`. Interface `security/KeyProvider.java:30` has a single method `Key getSecretKey(String alias)` `:32`. The only implementation is `JKSKeyProvider` `:37`.
- **Scope [F]:** `EncryptionContext` is used only by commitlog (`db/commitlog/*`) and hints (`hints/HintsDescriptor.java`). **OSS Cassandra has no SSTable-at-rest encryption**, so the KeyProvider protects only commitlog and hints. Keys are cached in an unbounded-ish Caffeine cache (max 64) with no rotation hook.
- **External [I]:** HashiCorp Vault and AWS KMS key providers exist in vendor forks (DSE has its own TDE). None verified for OSS.
- **Grade: D+.** The interface is trivially small, but the constructor contract is odd, there is no rotation or versioning, and the scope is limited to commitlog and hints.
- **Recommendation:** promote it to a `KeyManagementService` SPI with key-version IDs, rotation callbacks, and a `Map` constructor. This is a prerequisite for any SSTable encryption CEP and would let Vault/KMS integrations live out of tree.

### 2.10 Audit logging (IAuditLogger)
- **Status:** stable-public. `audit_logging_options.logger` (`audit/AuditLogOptions.java:43`) → `FBUtilities.newAuditLogger` (`utils/FBUtilities.java:685`, `Map` constructor, default package `org.apache.cassandra.audit`). Interface `audit/IAuditLogger.java` has three methods: `isEnabled` `:23`, `log(AuditLogEntry)` `:29`, `stop` `:35`. In-tree: `BinAuditLogger` (Chronicle BinLog), `FileAuditLogger` (logback), `DiagnosticEventAuditLogger`, `NoOpAuditLogger`. Filtering is done by core (`AuditLogFilter`) before `log()`.
- **Driver:** `AuditLogManager` (`audit/AuditLogManager.java:74`) implements `QueryEvents.Listener`, `AuthEvents.Listener`, and `JmxInvocationListener`, and registers at `:252-253`. It is switchable at runtime via nodetool/JMX `AuditLogManagerMBean`.
- **Leakage:** `AuditLogEntry` is a concrete class carrying `InetAddressAndPort` and internal types. Only one logger is active at a time (no fan-out).
- **External [I]:** Ecosystem audit plugins (Splunk/Kafka shippers) generally tail the `FileAuditLogger` output rather than implement the SPI. Ecosystem tooling such as AxonOps reportedly does the same. Unverified.
- **Grade: B.** Small, runtime-switchable, and core-filtered. It lacks multi-sink fan-out and a stable entry schema.
- **Recommendation:** allow a list of loggers. Version `AuditLogEntry` as a JDK-typed record. This boundary is a good candidate to become the "event sink" template for FQL and diagnostics.

### 2.11 Full query logging (FQL / BinLog)
- **Status:** internal-seam (not pluggable). `fql/FullQueryLogger.java:60` implements `QueryEvents.Listener` and writes Chronicle Queue via `utils/binlog/BinLog.java`. The only extension point is the archiver: `BinLogArchiver` (`utils/binlog/BinLogArchiver.java`) with `DeletingArchiver` and `ExternalArchiver` (a shell `archive_command`, gated by `allow_nodetool_archive_command`, `conf/cassandra.yaml:2232-2236`).
- **Grade: D.** The sink format (Chronicle) is fixed and the archiver is shell-out only. Anyone wanting Kafka or OTel must write a `QueryEvents.Listener`, and there is no supported way to register one (see 2.12).
- **Recommendation:** make the FQL sink an SPI that shares the audit sink contract. Make `BinLogArchiver` loadable by class.

### 2.12 In-process event buses: QueryEvents, AuthEvents, DiagnosticEventService
- **Status:** internal-seam. All three are real, typed, multi-subscriber buses, but there is no supported registration path for out-of-tree code.
  - `cql3/QueryEvents.java:43` has `registerListener` `:57` and a `Listener` with default methods `:270-317` (query, execute, batch, prepare × success/failure).
  - `auth/AuthEvents.java:32` has `registerListener` `:46` and `Listener` `:82`.
  - `diag/DiagnosticEventService.java` has `publish` `:72`, typed `subscribe` `:109/:124`, `subscribeAll` `:149`, and `isEnabled` `:252`. It is gated by `diagnostic_events_enabled: false` (`conf/cassandra.yaml:2249`). Remote consumers use JMX `DiagnosticEventPersistence` and `LastEventIdBroadcaster` with an in-memory `diag/store/DiagnosticEventMemoryStore.java`.
- **De facto registration hack [I, plausible]:** a `ServiceLoader`-discovered `StartupCheck` (2.17) runs in-process early, during `DatabaseDescriptor.applyStartupChecks` (`config/DatabaseDescriptor.java:1467-1472`) and `CassandraDaemon.runStartupChecks` (`:460`). It could register listeners. Otherwise, javaagents are the only route.
- **Leakage:** `QueryEvents.Listener` exposes `CQLStatement`, `QueryState`, `QueryOptions`, and `QueryHandler.Prepared`, which are deep internals. `DiagnosticEvent` subclasses expose internal state as `Map<String, Serializable>` (OK-ish).
- **Grade: C.** The plumbing is good and the contract is not.
- **Recommendation:** this is the cheapest high-value boundary. Add a supported `NodeListener`/`EventSink` SPI (ServiceLoader or yaml list) that receives JDK-typed event records (query fingerprint, keyspace/table, consistency, latency, user, outcome). Audit, FQL, and diagnostics become in-tree implementations.

### 2.13 Tracing (`-Dcassandra.custom_tracing_class`)
- **Status:** partial. `tracing/Tracing.java:111-130`: a static initializer constructs the class named in `CUSTOM_TRACING_CLASS` (`CassandraRelevantProperties.java:210`) via `FBUtilities.construct`, and **silently falls back** to `TracingImpl` on error (`:124-128`). The abstract base `Tracing` (`:53`) **extends `ExecutorLocals.Impl`**. Abstract methods: `stopSessionImpl` `:219`, `begin` `:242`, `newTraceState` `:316`, `trace(ByteBuffer,String,int)` `:368`. There is also `TraceState` (`tracing/TraceState.java:41`, `traceImpl` `:183`).
- **Leakage [F]:** implementers subclass an internal executor-locals class and deal with `TimeUUID`, `InetAddressAndPort`, and the internode tracing message parameters. The trace data model is "session + free-text events written to `system_traces`".
- **External [I]:** The `cassandra-zipkin-tracing` plugin (thelastpickle/smacke, Cassandra 3.x era) is the known user. OpenTelemetry users today rely on the OTel javaagent's driver-side instrumentation, not server spans.
- **Grade: D.** It works, but it is a system property plus a fat abstract class plus silent fallback, with no structured span model.
- **Recommendation:** replace it with an OpenTelemetry-shaped `TracingProvider` SPI (spans, attributes, context propagation via native-protocol custom payload and internode params), configured in yaml. See §3.2.

### 2.14 Metrics (CassandraMetricsRegistry / Dropwizard)
- **Status:** internal-seam. `metrics/CassandraMetricsRegistry.java:89` extends Dropwizard `MetricRegistry`, with a singleton `Metrics` `:100`. Every metric is also registered as a JMX MBean through `MBeanWrapper` (`:439,:553-584`). Dropwizard versions: `metrics-core` 4.2.28, `metrics-jvm`/`metrics-logback` 4.2.19 (`.build/parent-maven-pom.xml:714-745`).
- **`metrics-reporter-config` was removed [F]** (CASSANDRA-18743, `NEWS.txt:459-460`; `CHANGES.txt:820`; reporter-config3 removed CASSANDRA-18907 `:807`). The `cassandra.metricsReporterConfigFile` property is **still declared** (`CassandraRelevantProperties.java:420`) but has **zero usages**: it is dead config. NEWS says to use Dropwizard reporters "e.g. CsvReporter", but there is no yaml hook to instantiate one.
- **Virtual-table exposure:** keyspace `system_metrics` is built from the registry (`service/CassandraDaemon.java:645`, `createMetricsKeyspaceTables`).
- **External [I]:** Instaclustr `cassandra-exporter` (javaagent that reads the registry), Prometheus `jmx_exporter` (javaagent over JMX), DataStax MCAC (javaagent with collectd), and Apache Cassandra Sidecar (reads via JMX). Every popular metrics integration is a **javaagent**, which shows the missing boundary.
- **Grade: D.** The data is excellent, but there is no supported reporter SPI after the 18743 removal, so the ecosystem is forced into agents and JMX scraping.
- **Recommendation:**
  - Add a `metrics_reporters:` yaml list of `ParameterizedClass` implementing a tiny `MetricsReporterFactory(Map) → start(MetricRegistry)/stop()`.
  - Ship Prometheus and OTel adapters out of tree.
  - Delete `METRICS_REPORTER_CONFIG_FILE`.

### 2.15 Virtual tables for observability
- **Status:** internal-seam. These are registered only in core, in `CassandraDaemon.setupVirtualKeyspaces()` (`service/CassandraDaemon.java:641-651`): `system_virtual_schema`, `system_views` (29 tables, `db/virtual/SystemViewsKeyspace.java`, including `clients` `:37`, `settings` `:38`, `queries` `:64`, `system_logs` `:65`, `slow_queries` `:66`), `system_metrics`, and Accord debug keyspaces. `system_logs` is fed by the logback `VirtualTableAppender` (`:654-660`). There is no ServiceLoader or yaml hook to add a virtual keyspace.
- **Grade: C.** The `VirtualTable` / `AbstractVirtualTable` / `CollectionVirtualTableAdapter` API is clean enough to be public, but registration is closed.
- **Recommendation:** add `VirtualKeyspaceProvider` via ServiceLoader, restricted to a plugin-owned keyspace namespace (`plugin_<name>`). Read-only first. This is the CQL-native alternative to JMX for plugins. (Cross-check with the CQL/schema area owner.)

### 2.16 Configuration system (Config, DatabaseDescriptor, ConfigurationLoader)
- **Status:** partial.
  - `-Dcassandra.config.loader` (`CassandraRelevantProperties.java:200`) → `FBUtilities.construct` (`config/DatabaseDescriptor.java:519-524`).
  - Interface `config/ConfigurationLoader.java:22` has a single method `Config loadConfig()`. The default is `YamlConfigurationLoader`, which supports `-Dcassandra.settings.*` overrides (`YamlConfigurationLoader.java:95`, gated by `CONFIG_ALLOW_SYSTEM_PROPERTIES`) and env-var overrides (`:225`, gated by `CASSANDRA_ALLOW_CONFIG_ENVIRONMENT_VARIABLES`).
  - The property model is `config/Loader.java` / `Properties.java` / `Replacements` (`@Replaces` renames), used by the `system_views.settings` updatable properties.
- **Leakage:** a loader must return the full mutable `Config` POJO (`config/Config.java`, 1652 lines). Validation and derivation happen in `DatabaseDescriptor` (6540 lines, all static). There is no layering (base plus overlay), no remote source, and no change notification beyond per-property setters.
- **External [I]:** K8ssandra/cass-operator uses `cass-config-builder` to render yaml (out of process). The Management API passes config via yaml and env vars. Nobody is known to ship a custom `ConfigurationLoader`.
- **Grade: C−.** The hook exists and is tiny, but the contract is "produce our whole internal POJO", and static `DatabaseDescriptor` makes runtime change unsafe.
- **Recommendation:** add a `ConfigSource` SPI (ordered, layered key/value sources: yaml, env, sysprops, file-watch, remote) that feeds the existing `Loader`/`Properties` model. Expose provenance in `system_views.settings`. This is a long-term prerequisite for any config-change listener.

### 2.17 Startup checks (StartupCheck, `startup_checks:`)
- **Status:** stable-public. This is one of only three ServiceLoader SPIs. `service/StartupCheck.java` has `name()` `:44`, `execute(StartupChecksConfiguration)` `:56`, `isConfigurable` `:63`, `isDisabledByDefault` `:76`, and `postAction` `:86`. Discovery is at `service/StartupChecks.java:164-190` (duplicate names cause startup to fail; CASSANDRA-21409 moved exception handling). Per-check config comes from the `startup_checks:` map (`Config.java:1090`, `conf/cassandra.yaml:2809`).
- **Grade: B+.** It has ServiceLoader discovery, named and configurable checks, and yaml config by name. It is the closest thing to a model plugin contract in the codebase.
- **Recommendation:** use this pattern (ServiceLoader + name + yaml map keyed by name) as the template for the unified loader (§3.1). Document that a check must not have side effects; today it is the de facto bootstrap hook (2.12).

### 2.18 Guardrails: GuardrailsConfigProvider, custom value guardrails (password / role-name policy)
- **GuardrailsConfigProvider — status: partial.** `db/guardrails/GuardrailsConfigProvider.java:48-69`: a static singleton chosen by `-Dcassandra.custom_guardrails_config_provider_class` (`CassandraRelevantProperties.java:208`). The legacy constant is `@Deprecated(since="5.0")` `:45`. Its single method `getOrCreate(@Nullable ClientState)` `:58` returns `GuardrailsConfig`, which is a ~119-method interface (`db/guardrails/GuardrailsConfig.java`, 760 lines). Per-client config is possible (tenant-specific limits), but implementers must implement every threshold getter, and the interface grows each release.
- **Custom value guardrails — status: stable-public (CEP-24, CEP-55).** yaml `password_policy` / `role_name_policy` (`Config.java:1103-1104`, `CustomGuardrailConfig extends HashMap`). Plugin classes extend abstract `ValueValidator<V>` (`ValueValidator.java:40`) / `ValueGenerator<V>` (`ValueGenerator.java:37`) with a `CustomGuardrailConfig` constructor (`:137/:153`). The default package is `org.apache.cassandra.db.guardrails`. They can be reconfigured at runtime via `GuardrailsMBean` (`role_name_policy_reconfiguration_enabled`). Framework: `CustomGuardrail` (`CustomGuardrail.java:43`).
- **Grade: C (provider), B (custom value guardrails).** The provider is a system property plus a fat, ever-growing interface. Custom value guardrails are a good, reusable pattern.
- **Recommendation:** generalize `CustomGuardrail` into a policy-engine SPI (§3.4). Deprecate GuardrailsConfigProvider in favor of `GuardrailPolicy.evaluate(context) → Allow/Warn/Fail`.

### 2.19 Disk failure policy hook (DiskErrorsHandler)
- **Status:** partial / new (CASSANDRA-20363; fail-on-init-error CASSANDRA-20614). `-Dcassandra.custom_disk_error_handler` (`CassandraRelevantProperties.java:207`) → `service/DiskErrorsHandlerService.java:76-82`. Interface `service/DiskErrorsHandler.java:24` extends `AutoCloseable` and has `init`, `handleCorruptSSTable`, `handleFSError`, `handleCommitError`, `handleStartupFSError`, `inspectDiskError`, and `inspectCommitLogError` (`:26-38`). The default is `DefaultDiskErrorsHandler`, which implements `disk_failure_policy` / `commit_failure_policy` (`conf/cassandra.yaml:502,519`).
- **Leakage:** uses `CorruptSSTableException` and `FSError` (internal exceptions). It is a sysprop, not yaml.
- **Grade: C+.** Useful (for example, auto-replace or orchestrator-signalling policies), but sysprop-only and internally typed.
- **Recommendation:** move it to yaml (`disk_error_handler: {class_name, parameters}`) and pass a JDK-typed `DiskErrorEvent`.

### 2.20 JMX authentication/authorization and nodetool
- **JMX — status: stable-public (JDK-standard).**
  - `jmx_server_options` (`config/JMXServerOptions.java:45`): `login_config_name`/`login_config_file` (`:58-59`, JAAS; `conf/cassandra-jaas.config`, `auth/CassandraLoginModule.java`).
  - `authorizer` (`:70`) is any `InvocationHandler` → `utils/JMXServerUtils.java:215-229` (wrapped as an `MBeanServerForwarder` proxy). The default is `auth/jmx/AuthorizationProxy`, which delegates to IAuthorizer.
  - JMX auth is therefore pluggable through standard JAAS LoginModules. Kerberos and LDAP LoginModules from the JDK work [I].
  - Grade **B**.
- **MBean registration — status: internal-seam.** `-Dorg.apache.cassandra.mbean_registration_class` (`CassandraRelevantProperties.java:414`, `utils/MBeanWrapper.java:69-81`) is used by in-JVM dtests.
- **nodetool — status: internal-seam.** It was migrated from airline to **picocli** (CASSANDRA-17445, `CHANGES.txt:290`; `tools/NodeTool.java:47,196`). Subcommands are a **static** `@Command(subcommands={...})` list in `tools/nodetool/NodetoolCommand.java` (~170 command classes). There is no ServiceLoader or plugin subcommand discovery, and every command talks to the node via `NodeProbe` (JMX).
  - Grade **D** as a boundary. picocli makes plugin subcommands trivial to add (`CommandLine.addSubcommand` and ServiceLoader), but nothing does it.
- **CQL-based management (CEP-38) — status: proposed; not present [F].** There is no `COMMAND`/management grammar in `src/antlr/` (`K_EXECUTE` exists only as a permission at `Parser.g:1477`).
- **External [I]:**
  - The K8ssandra/DataStax **Management API for Apache Cassandra** loads a javaagent (`datastax-mgmtapi-agent`, ByteBuddy) to expose management over a unix-socket CQL channel.
  - Apache Cassandra **Sidecar** uses JMX plus CQL from out of process.
  - Ericsson **ecchronos** (repair scheduler) uses JMX.
  - Instaclustr **esop** (backup/restore) uses JMX plus filesystem.
  - All of them exist *because* there is no in-process management SPI.
- **Recommendation:** add a `ManagementCommand` SPI (name, typed params, result rows). Make the same command object reachable from nodetool (picocli), JMX (generic `execute` MBean), and a future CEP-38 CQL verb. Use ServiceLoader so plugins can add commands.

### 2.21 Seed provider (SeedProvider)
- **Status:** stable-public. `seed_provider` (`Config.java:130`) → `DatabaseDescriptor.applySeedProvider()` (`config/DatabaseDescriptor.java:1678-1697`). It requires an FQCN and a `Map` constructor. Interface `locator/SeedProvider.java` has one method, `List<InetAddressAndPort> getSeeds()` `:24`. In-tree: `SimpleSeedProvider`.
- **External [I]:** K8ssandra `K8SeedProvider`, cloud-API seed providers (EC2 tag/ASG), and Consul/DNS providers. Many exist in the wild.
- **Leakage:** `InetAddressAndPort` (internal type). The class is loaded and called once at startup, then periodically by gossip.
- **Grade: B+.** A one-method SPI with a large ecosystem. It should be JDK-typed (`InetSocketAddress`).
- **Recommendation:** keep it. Add an `InetSocketAddress` default bridge. Coordinate with the TCM/cluster-membership owner, because under TCM the seed role shrinks [I].

### 2.22 Logging (logback) and LoggingSupport
- **Status:** internal-seam. `utils/logging/LoggingSupportFactory.java:40-58` detects logback by `LoggerFactory` class-name sniffing and loads `LogbackLoggingSupport`. Otherwise it uses `NoOpFallbackLoggingSupport` and warns that this is an "unsupported deployment". `LoggingSupport` (`utils/logging/LoggingSupport.java:31`) backs nodetool `setlogginglevel`, the `system_logs` vtable appender, and JMX log level. `conf/logback.xml` is the real extension point: any logback appender jar on the classpath works.
- **Grade: B− (via logback), F (backend swap).** Logback config is a mature, external, de facto boundary.
- **Recommendation:** leave it. Document `VirtualTableAppender` and the slow-query logger names as stable.

### 2.23 Profiler and heap dump
- **Status:** internal-seam. `service/AsyncProfilerService.java` (embedded `one.profiler.AsyncProfiler`, `:49`; upgraded to 4.5, CASSANDRA-21630) is gated by `-Dcassandra.async_profiler.enabled` / `.unsafe_mode` (`CassandraRelevantProperties.java:70-71`). It is exposed via `profiler/AsyncProfilerMBean` and nodetool `AsyncProfileCommandGroup`/`ProfileLoad`. Heap dumps: `utils/HeapUtils.java`, `dump_heap_on_uncaught_exception` / `heap_dump_path` (`Config.java:1570-1571`).
- **Grade: C.** Hard-wired to async-profiler. JFR is unexposed [I].
- **Recommendation:** low priority. If a profiler SPI is ever needed, fold it into the ManagementCommand SPI.

### 2.24 JVM agents (ecosystem)
- **Fact:** the shipped scripts add only the `jamm` agent (`conf/cassandra-env.sh:200-201`, `bin/cassandra.in.sh:81-82`).
- **[I] Ecosystem agents and what they substitute for:**

  | Agent | Substitutes for |
  |---|---|
  | Prometheus `jmx_exporter` | metrics |
  | Instaclustr `cassandra-exporter` | metrics |
  | DataStax MCAC | metrics + events |
  | `datastax-mgmtapi-agent` | management |
  | OpenTelemetry java agent | tracing; mostly client side |
  | AxonOps / other APM agents | metrics + events |

- **Grade: n/a. This is a symptom.** Each agent marks a missing in-process boundary: metrics reporter, event sink, management command, or tracing.

### 2.25 Other `-Dcassandra.*` class hooks found (owned by other areas; listed for cross-check)
- `cassandra.custom_query_handler_class` (`CassandraRelevantProperties.java:209`, `service/ClientState.java:131`) → CQL/query area.
- `cassandra.storage_hook` (`:601`, `db/StorageHook.java:55-58`) and `cassandra.stream_hook` (`:605`, `streaming/StreamHook.java:37-40`) → storage/streaming.
- `cassandra.clock`, `cassandra.monotonic_clock.approx/.precise` (`:133-135`, `utils/Clock.java:66`, `utils/MonotonicClock.java:97`) → test/simulator infra.
- `cassandra.schema.update_handler_factory.class` (`:550`) → schema/TCM.
- `cassandra.test.accord.agent` (`:45`, `service/accord/AccordService.java:462`) → Accord/consensus.
- `cassandra.crypto_provider_class_name` (`:204`) → covered in 2.8.

---

## 3. Proposed new boundaries

### 3.1 Unified, versioned plugin loader (highest leverage)
- **Problem:** there are five loader idioms with inconsistent constructor contracts and package defaults, mixed yaml and sysprop entry points, and a flat classpath (§0).
- **Proposal:**
  - Introduce `org.apache.cassandra.plugin.PluginLoader`. It discovers `plugins/<name>/` directories, each with a `plugin.yaml` declaring `name`, `version`, `api-version` (per SPI), and `provides: [SPI → class]`.
  - Each plugin gets a child-first `URLClassLoader` with a parent-first allowlist for `org.apache.cassandra.spi.*` and the JDK. This avoids Netty/Guava/Jackson version clashes. Precedent: `triggers/CustomClassLoader.java:47`.
  - All existing `ParameterizedClass` keys keep working. When a `class_name` matches a plugin-provided class, the loader resolves it from that plugin's loader.
  - Standardize on one constructor contract, `(Map<String,String> params)` or `(PluginContext ctx)`, plus lifecycle `init/start/stop`.
  - Move all `-Dcassandra.custom_*` hooks into yaml keys and keep the sysprops as deprecated aliases.
  - Add a `system_views.plugins` vtable (name, version, SPI, api-version, status).
- **Compatibility:** each SPI declares `@SpiVersion(major.minor)`. The loader refuses a major mismatch and warns on minor.
- **Next step:** a CEP draft. Prototype with `StartupCheck` (already ServiceLoader-based) and `IAuditLogger` (small).

### 3.2 OpenTelemetry tracing and metrics SPI
- **Tracing:** add `TracingProvider { Span startSpan(name, parent ctx, attrs); propagation inject/extract }`. Context rides in native-protocol custom payload and internode message params. The existing `system_traces` model becomes one in-tree provider. This replaces `custom_tracing_class` (§2.13).
- **Metrics:** add a `metrics_reporters:` yaml list (`MetricsReporterFactory`), plus an optional bridge exposing `CassandraMetricsRegistry` through the OTel Metrics API. Ship out-of-tree Prometheus and OTLP exporters. This removes the javaagent need (§2.14, §2.24). Delete the dead `METRICS_REPORTER_CONFIG_FILE`.

### 3.3 Event-sink SPI (QueryEvents / AuthEvents / Diagnostic / Audit / FQL unified)
- `EventSink<E extends NodeEvent>`, registered by yaml list or plugin. It receives **JDK-typed, versioned** records and runs through bounded async delivery with a drop-or-block policy (reuse the BinLog `block`/`max_queue_weight` semantics). Audit, FQL, and diagnostics become built-in sinks. Today's buses (§2.12) are already the right shape internally.

### 3.4 Policy engine for guardrails and authorization
- `GuardrailPolicy` / `AdmissionPolicy`: `evaluate(OperationContext{user, roles, keyspace, table, op, client ip, driver, size estimates}) → Allow | Warn(msg) | Fail(msg)`.
- Built-in implementations wrap the existing thresholds (`Guardrails`), `CustomGuardrail` validators, DC/CIDR authorizers, and per-tenant limits (today: `GuardrailsConfigProvider.getOrCreate(ClientState)`).
- Out-of-tree candidates: OPA/Rego and Cedar adapters [I].
- This replaces the 119-method `GuardrailsConfig` as the extension surface (§2.18) and shrinks ICIDRAuthorizer (§2.4).

### 3.5 Pluggable config sources
- `ConfigSource` SPI (ordered layers, change notifications), feeding `config/Loader`/`Properties`. Expose provenance per key in `system_views.settings`. Keep `ConfigurationLoader` as a deprecated single-source adapter (§2.16).

### 3.6 Management command SPI (bridge to CEP-38)
- A single `ManagementCommand` abstraction with three front ends: picocli nodetool, a generic JMX `execute`, and CQL `EXECUTE COMMAND` (CEP-38). It is ServiceLoader-extensible, and authorization goes through IAuthorizer on a new `CommandResource` (§2.20).

### 3.7 Key management service SPI
- Versioned keys and rotation, with a `Map` constructor. This is a prerequisite for SSTable TDE, and Vault/KMS providers would live out of tree (§2.9).

### 3.8 Plugin virtual keyspaces
- `VirtualKeyspaceProvider` via ServiceLoader or plugin, namespaced `plugin_*`, read-only initially (§2.15).

---

## 4. Scorecard

| # | Boundary | Status | Loader | Grade |
|---|---|---|---|---|
| 2.1 | IAuthenticator (+mTLS validator) | stable-public | yaml ParameterizedClass | B |
| 2.2 | IAuthorizer | stable-public | yaml | B− |
| 2.3 | IRoleManager / IDefaultRoleInitializer | stable-public | yaml | C+ |
| 2.4 | INetworkAuthorizer / ICIDRAuthorizer | stable-public | yaml | B / D+ |
| 2.5 | Auth caches | internal-seam | — | C |
| 2.6 | IInternodeAuthenticator | stable-public | yaml | A− |
| 2.7 | ISslContextFactory | stable-public | yaml (Map constructor) | B− |
| 2.8 | Crypto provider | stable-public | yaml + sysprop | B |
| 2.9 | KeyProvider / TDE | partial | yaml (Options constructor) | D+ |
| 2.10 | IAuditLogger | stable-public | yaml (Map constructor) | B |
| 2.11 | FQL / BinLog | internal-seam | — (shell archiver) | D |
| 2.12 | QueryEvents / AuthEvents / Diagnostics | internal-seam | none | C |
| 2.13 | Tracing | partial | sysprop | D |
| 2.14 | Metrics reporters | internal-seam | none (removed) | D |
| 2.15 | Observability vtables | internal-seam | none | C |
| 2.16 | ConfigurationLoader | partial | sysprop | C− |
| 2.17 | StartupCheck | stable-public | ServiceLoader + yaml map | B+ |
| 2.18 | GuardrailsConfigProvider / custom value guardrails | partial / stable | sysprop / yaml | C / B |
| 2.19 | DiskErrorsHandler | partial | sysprop | C+ |
| 2.20 | JMX auth (JAAS, authz proxy) / nodetool / CEP-38 | stable / internal / proposed | yaml / static / — | B / D / — |
| 2.21 | SeedProvider | stable-public | yaml (Map constructor) | B+ |
| 2.22 | Logging (logback) | internal-seam | logback.xml | B− |
| 2.23 | Profiler / heap dump | internal-seam | sysprop | C |

**Priority order for the roadmap:**
1. Unified loader (3.1).
2. Event-sink and metrics-reporter SPIs (3.2/3.3). These are cheap because the buses and registry already exist, and they remove most javaagents.
3. Management command SPI (3.6).
4. Policy engine (3.4).
5. KMS (3.7).
6. Config sources (3.5).
