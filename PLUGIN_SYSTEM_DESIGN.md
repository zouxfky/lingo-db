# LingoDB MLIR Extension System Design

## 1. Purpose

This document defines a PostgreSQL-style extension system for LingoDB. An
extension is installed as a shared library, enabled per database, and loaded at
runtime without rebuilding the LingoDB executable.

The first implementation only accepts plugin operations through MLIR input. It
does not add domain-specific frontends such as Cypher, ONNX, or a SQL syntax for
plugin operations. The only SQL addition in scope is administrative DDL such as
`CREATE EXTENSION` and `DROP EXTENSION`.

The extension system is also the execution substrate for a later LLM-driven
framework that generates dialect definitions, optimization passes, lowering
passes, runtime functions, and tests from workload specifications.

## 2. Decisions for Version 1

- Extensions are discovered only in configured extension directories.
- An extension is enabled per database with `CREATE EXTENSION`.
- Plugin IR is supplied as textual MLIR.
- One shared library contains the compiler and runtime portions initially.
- A plugin can register dialects, pipeline hooks, and runtime symbols.
- Plugins are tied to an exact LingoDB and LLVM/MLIR build identity.
- A loaded shared library remains mapped until process exit.
- `DROP EXTENSION` disables the extension logically but does not call
  `dlclose()`.
- Existing queries continue running when an extension is disabled. New queries
  cannot use it.
- The first implementation supports the default CPU LLVM JIT backend. Other
  backends are added after the core path is stable.
- Plugins cannot define persistent Catalog entry subclasses in version 1.
- Plugins must lower all plugin-specific IR to built-in LingoDB or standard
  MLIR before their declared deadline.

## 3. Non-goals for Version 1

- Hot unloading of compiler plugins.
- Binary compatibility across LingoDB or LLVM/MLIR releases.
- Domain frontends or SQL translation into plugin operations.
- Downloading extensions from the network.
- Installing an arbitrary shared library path from SQL.
- Untrusted or sandboxed native extensions.
- Plugin-defined Catalog serialization formats.
- Multiple simultaneously active versions of the same extension in one
  database.
- GPU, C, profiling, or TPDE baseline backend integration.

## 4. User Experience

An installed extension has this layout:

```text
$LINGODB_EXTENSION_PATH/
└── vector/
    └── 1.0.0/
        ├── extension.json
        └── liblingodb_vector.so
```

On macOS, the library may use the `.dylib` suffix.

The database administrator enables it with:

```sql
CREATE EXTENSION vector;
CREATE EXTENSION IF NOT EXISTS vector VERSION '1.0.0';
```

Subsequent MLIR queries can use its dialect:

```mlir
module {
  func.func @main() {
    %value = arith.constant 42 : i64
    demo.print %value : i64
    func.return
  }
}
```

The extension is disabled with:

```sql
DROP EXTENSION vector;
```

`DROP EXTENSION` removes the database-level activation immediately. The shared
library remains mapped until process exit because MLIR contexts, pass builders,
and generated code may still contain addresses owned by the library.

## 5. High-level Architecture

```text
extension.json + plugin shared library
                 |
                 v
          ExtensionManager
                 |
       +---------+----------+
       |         |          |
       v         v          v
   Dialects   Pipelines   Runtime symbols
       |         |          |
       +---------+----------+
                 |
                 v
MLIR input -> plugin optimization/lowering -> LingoDB IR -> LLVM JIT
```

The implementation adds four core components:

1. `ExtensionManager`: discovery, validation, dependency resolution, loading,
   activation, and process-wide lifetime.
2. `DialectRegistry`: plugin callbacks used while constructing an MLIR context.
3. `PipelineRegistry`: ordered plugin pass builders at well-defined execution
   phases.
4. `RuntimeSymbolRegistry`: plugin-owned function addresses exposed to the JIT.

## 6. Extension Manifest

Use JSON because the repository already vendors a structured JSON parser. Do
not parse the manifest with ad hoc string processing.

Example:

```json
{
  "name": "demo",
  "version": "1.0.0",
  "plugin_abi": 1,
  "lingodb_build_id": "fab813e8-llvm20.1-debug",
  "llvm_version": "20.1",
  "library": "liblingodb_demo.so",
  "dialect_namespaces": ["demo"],
  "must_lower_before": "BeforeQueryOptimization",
  "dependencies": []
}
```

Required validation:

- The extension name uses a restricted identifier syntax.
- The resolved manifest and library remain below an approved extension root.
- The manifest name and descriptor name match.
- The ABI version and descriptor structure size match.
- The LingoDB build ID matches exactly.
- The LLVM/MLIR major and minor versions match exactly.
- Every dependency is installed and version-compatible.
- The dependency graph is acyclic.
- Dialect namespaces and runtime symbol names do not conflict.
- The shared library is a regular file with acceptable ownership and
  permissions.

## 7. Plugin ABI

Add `include/lingodb/plugin/PluginABI.h` with an explicitly exported entry
symbol:

```cpp
namespace lingodb::plugin {

inline constexpr uint32_t pluginAbiVersion = 1;

enum class PipelinePhase : uint32_t {
   AfterParse,
   BeforeQueryOptimization,
   AfterQueryOptimization,
   AfterRelAlgLowering,
   AfterSubOpLowering,
   BeforeBackend,
};

class ExtensionRegistration;

struct ExtensionDescriptorV1 {
   uint32_t abiVersion;
   uint32_t structSize;
   const char* name;
   const char* version;
   const char* buildId;
   void (*registerExtension)(ExtensionRegistration& registration);
};

} // namespace lingodb::plugin

extern "C" LINGODB_PLUGIN_EXPORT
const lingodb::plugin::ExtensionDescriptorV1* lingodbGetExtensionV1();
```

The entry point has C linkage so its name is stable. The descriptor callback
uses C++ and MLIR types, so the compiler plugin ABI is deliberately build-locked.
The loader must reject a different build ID rather than attempting best-effort
compatibility.

`ExtensionRegistration` writes into temporary registries. The manager validates
the complete staged registration before publishing it. This prevents a normal
validation failure from leaving half-registered symbols or hooks behind.

The registration surface should look like:

```cpp
registration.addDialect(
   "demo",
   [](mlir::DialectRegistry& registry) {
      registry.insert<demo::DemoDialect>();
   });

registration.addPipelineHook(
   PipelinePhase::AfterParse,
   0,
   [](mlir::OpPassManager& pm, const PipelineBuildContext& context) {
      pm.addPass(demo::createOptimizePass());
      pm.addPass(demo::createLowerToStdPass());
   });

registration.addRuntimeSymbol(
   "lingodb_ext_demo_v1_print_i64",
   reinterpret_cast<void*>(&demoPrintI64),
   "(i64)->void");
```

Runtime symbol names must contain the extension name and ABI version. This
avoids collisions and permits side-by-side process loading during an upgrade.

## 8. Extension Manager

Add these files:

```text
include/lingodb/plugin/ExtensionManager.h
include/lingodb/plugin/ExtensionManifest.h
include/lingodb/plugin/ExtensionRegistration.h
include/lingodb/plugin/PipelineRegistry.h
include/lingodb/plugin/RuntimeSymbolRegistry.h
include/lingodb/plugin/PluginABI.h

src/plugin/ExtensionManager.cpp
src/plugin/ExtensionManifest.cpp
src/plugin/ExtensionRegistration.cpp
src/plugin/PipelineRegistry.cpp
src/plugin/RuntimeSymbolRegistry.cpp
src/plugin/DynamicLibrary.cpp
src/plugin/CMakeLists.txt
```

Suggested public API:

```cpp
class ExtensionManager {
   public:
   static ExtensionManager& instance();

   InstalledExtension inspect(std::string_view name,
                              std::optional<std::string_view> version);
   void load(const InstalledExtension& extension);
   void activate(runtime::Session& session,
                 std::string_view name,
                 std::optional<std::string_view> version);
   void deactivate(runtime::Session& session, std::string_view name);

   void appendDialects(mlir::DialectRegistry& registry,
                       const ExtensionSet& extensions) const;
   mlir::LogicalResult runPipeline(PipelinePhase phase,
                                   mlir::ModuleOp module,
                                   const PipelineBuildContext& context) const;
   void visitRuntimeSymbols(const ExtensionSet& extensions,
                            const RuntimeSymbolVisitor& visitor) const;
};
```

The process-wide plugin state machine is:

```text
DISCOVERED -> VALIDATED -> LOADED -> REGISTERED
                                  -> FAILED
```

The database-level state is separate:

```text
DISABLED -> ACTIVE -> DRAINING -> DISABLED
```

The same library may be loaded process-wide while active in only one of several
database sessions.

Loading and registration must be protected by a mutex. Query compilation reads
an immutable snapshot of the active extension set and registries.

## 9. MLIR Context Integration

The current built-in dialect registration in `src/execution/Frontend.cpp` must
be split into reusable functions:

```cpp
void registerBuiltinDialects(mlir::DialectRegistry& registry,
                             bool includeLLVM);

void initializeContext(mlir::MLIRContext& context,
                       bool includeLLVM,
                       const plugin::ExtensionSet& extensions);
```

`initializeContext` performs this sequence:

1. Register built-in LingoDB and standard MLIR dialects.
2. Ask `ExtensionManager` to append dialects for the active extension set.
3. Append the completed registry to the context.
4. Load the available dialects.
5. Configure the existing context options.

The scheduler currently has one LLVM and one non-LLVM context stack. Replace
that cache key with:

```cpp
struct ContextKey {
   bool includeLLVM;
   std::string extensionFingerprint;
};
```

The fingerprint is computed from the sorted list of
`name@version@buildId`. This prevents a context created for one database from
being reused for a database with a different extension set.

Context pre-warming must capture the same `ContextKey`. A stale context must be
destroyed instead of returning to a newer generation of the pool.

## 10. Pipeline Integration

Add a helper that runs plugin hooks through a normal MLIR `PassManager`:

```cpp
mlir::LogicalResult runExtensionPipeline(
   PipelinePhase phase,
   mlir::ModuleOp module,
   catalog::Catalog* catalog,
   const ExtensionSet& extensions,
   const std::shared_ptr<execution::SnapshotState>& snapshots,
   bool verify);
```

It must enable the existing verifier and instrumentation. Hooks are ordered by:

1. Dependency topological order.
2. Numeric priority.
3. Extension name as a deterministic tie-breaker.

Modify `src/execution/Execution.cpp` to invoke hooks at these boundaries:

```text
Frontend completes
  -> AfterParse
  -> BeforeQueryOptimization
Core relation optimization
  -> AfterQueryOptimization
RelAlg to SubOperator lowering
  -> AfterRelAlgLowering
SubOperator to imperative lowering
  -> AfterSubOpLowering
  -> BeforeBackend
LLVM backend
```

For the first milestone, require plugin-specific IR to be completely optimized
and lowered during `AfterParse`. The extra phases exist so later extensions can
interoperate more deeply without changing the ABI immediately.

After the declared lowering deadline, validate that the module contains no
plugin-owned operations, types, or attributes. A failure must report the first
remaining IR object and the responsible extension.

## 11. Runtime Symbol Integration

The default LLVM backend currently registers built-in `FunctionHelper` and bare
runtime functions. Extend `LLVMBackendOptions` with an immutable list of plugin
runtime symbols and register them in the same ORC JIT symbol map.

The backend obtains the symbols from the active extension set in the current
`ExecutionContext`/`Session`, not from every process-loaded plugin.

Version 1 should use explicit symbol registration rather than `RTLD_GLOBAL`.
This makes ownership and collisions visible and avoids accidental resolution
against an inactive extension.

The initial plugin lowering should target a normal `func.call` with an external
private declaration. This avoids modifying the DB runtime-function semantic
registry in the first implementation.

Later backend work must add the same plugin symbol registry to:

- C emission and dynamic loading.
- TPDE baseline external-function maps.
- GPU host runtime registration.
- Profiling and debug LLVM backends.

## 12. Catalog Persistence

Keep extension metadata core-owned. Add to `Catalog`:

```cpp
struct ExtensionRecord {
   std::string name;
   std::string version;
   std::string buildId;

   void serialize(utility::Serializer& serializer) const;
   static ExtensionRecord deserialize(utility::Deserializer& deserializer);
};

std::unordered_map<std::string, ExtensionRecord> extensions;
```

Do not add one `CatalogEntryType` per plugin. Do not ask a plugin to deserialize
state before the plugin has been loaded.

Update `src/catalog/Catalog.cpp`:

- Add a new serialized property for the extension map.
- Increment the Catalog binary version.
- Accept the previous Catalog version and initialize an empty extension map.
- Add create, drop, lookup, and list methods.

`Session::createSession` must activate all persisted extensions immediately
after loading the core Catalog and before any MLIR query context is created.
Missing or incompatible required extensions are startup errors for that
database, with a message naming the expected version and build ID.

## 13. CREATE and DROP EXTENSION

The SQL lexer already knows the `EXTENSION` keyword, but the parser has no
`CREATE EXTENSION` production.

Add:

- `CreateExtensionInfo` in the create AST.
- Parser productions for `CREATE EXTENSION` and `DROP EXTENSION`.
- Analyzer checks for duplicate activation and `IF [NOT] EXISTS` behavior.
- Translator support that emits a core runtime call.
- A serialized `CreateExtensionDef` containing only name and requested version.
- `RelationHelper` functions that call `ExtensionManager` and update Catalog.

`CREATE EXTENSION` affects subsequent queries. It does not make the new dialect
available inside the DDL query that performs the installation.

`DROP EXTENSION` performs this sequence:

1. Reject or cascade database objects that depend on the extension.
2. Mark the extension inactive for new queries.
3. Remove its Catalog record and persist the Catalog.
4. Stop returning matching contexts to the context pool.
5. Allow already-running queries to finish.
6. Keep the shared library mapped until process exit.

Version 1 has no plugin-defined persistent objects, so dependency handling can
initially be limited to explicit extension dependencies.

## 14. Build and Link Strategy

This is the highest-risk part of the implementation.

The project currently hides C++ symbols and mostly links static libraries. A
plugin must not bring a second independent copy of MLIR or a core LingoDB
dialect into the process. Duplicate type identities can make `isa`/`cast` fail
even when the printed operation names match.

Before implementing the full manager, build a technical spike that proves:

```text
external demo shared library
  -> host loads it
  -> plugin registers demo dialect
  -> host parses demo.print
  -> plugin lowers demo.print to standard MLIR
  -> LLVM JIT calls demo runtime function
```

The production direction should provide one shared compiler/runtime identity,
for example an exported `liblingodb.so`, linked by both LingoDB tools and
extensions. The exact target split can be refined after the spike, but all
plugin-visible MLIR and LingoDB type definitions must come from the same binary
instances as the host.

Build-system additions:

- Explicit `LINGODB_PLUGIN_EXPORT` visibility macros.
- Position-independent code for plugin-facing targets.
- An installed public plugin SDK include directory.
- `LingoDBPluginConfig.cmake` for out-of-tree extension builds.
- A `lingodb_add_extension()` CMake helper.
- A generated LingoDB/LLVM/MLIR build ID header.
- Correct Linux rpath and macOS `@rpath` handling.
- A CMake target for the demo extension.

Do not proceed to Catalog or SQL work until the dynamic type-identity spike
passes on the primary server platform.

## 15. Error Handling and Security

Extension loading executes native code in the database process. Version 1 must
treat extensions as trusted administrator-installed components.

Required safeguards:

- Resolve paths canonically and enforce configured roots.
- Reject absolute library paths in SQL and manifests.
- Reject symlink escapes and writable-by-everyone plugin files.
- Use `RTLD_NOW` behavior so missing symbols fail during activation.
- Validate everything possible before invoking plugin registration callbacks.
- Reject duplicate extension identities, dialect namespaces, hooks, and runtime
  symbols.
- Report errors with extension name, version, path, and failed validation.
- Do not catch crashes from arbitrary plugin native code as recoverable errors.
- Never call `dlclose()` in version 1.
- Require administrative permission for `CREATE/DROP EXTENSION` once LingoDB
  has an authorization model.

## 16. Tests

Add an out-of-tree-style demo extension under `test/plugins/demo` and build it
as a separate shared library.

Unit tests:

- Manifest parsing and validation.
- Search path containment.
- ABI and build-ID mismatch.
- Dependency ordering and cycle detection.
- Duplicate dialect and runtime-symbol detection.
- Deterministic pipeline-hook ordering.
- Extension-set fingerprinting.
- Catalog serialization and previous-version migration.

Process-level integration tests:

- Load, parse, lower, JIT, and execute `demo.print`.
- Reject `demo.*` IR when the database has not enabled the extension.
- Enable an extension, restart, and execute plugin IR.
- Drop an extension and reject subsequent plugin queries.
- Keep an in-flight query valid while dropping the extension.
- Use two databases with different extension sets.
- Reject remaining plugin IR after the declared lowering deadline.
- Reject a missing runtime symbol before execution.
- Verify Linux and macOS library discovery.

Use separate test processes for cases that modify process-global extension
state. Test order must not affect results.

## 17. Implementation Phases

### Phase 0: Dynamic linking and TypeID spike

- Build `demo.so` outside the core target graph.
- Load it before constructing `MLIRContext`.
- Register and parse one operation.
- Lower it to `func.call`.
- Register and invoke one runtime function through LLVM JIT.
- Decide and implement the shared-library/export strategy.

Exit criterion: the full demo query runs without statically compiling the demo
dialect into LingoDB.

### Phase 1: Core extension runtime

- Add manifest parsing and secure discovery.
- Add plugin ABI and staged registration.
- Add `ExtensionManager`.
- Add dialect, pipeline, and runtime-symbol registries.
- Add extension-aware MLIR context construction.
- Add extension-aware context-pool keys.
- Add pipeline hooks and lowering-deadline checks.
- Support default CPU LLVM JIT only.

Exit criterion: `run-mlir` can execute a demo extension selected explicitly by
an internal/test activation API.

### Phase 2: Database activation

- Add `ExtensionRecord` persistence.
- Activate extensions during `Session` creation.
- Add Catalog migration from the previous binary version.
- Add `CREATE EXTENSION` and `DROP EXTENSION`.
- Invalidate matching context pools after activation changes.

Exit criterion: enable once, restart the process, and execute plugin MLIR
without a command-line plugin path.

### Phase 3: SDK and operational quality

- Install headers and CMake package files.
- Add a plugin project template.
- Add list/inspect/verify administrative commands.
- Add dependency and upgrade handling.
- Add diagnostics, tracing, and plugin timing.
- Complete Linux and macOS tests.

Exit criterion: a plugin can be developed and built in a separate repository
against an installed LingoDB SDK.

### Phase 4: Additional backends and richer integration

- Integrate plugin symbols with C, baseline, debug, profiling, and GPU backends.
- Add plugin-owned Session state with explicit destruction hooks.
- Add opaque core-owned Catalog metadata for plugin configuration.
- Consider side-by-side active versions and generation-based upgrades.

## 18. LLM-generated Dialect Framework

The LLM framework is a separate project built on the stable plugin SDK. It
should generate a complete candidate extension, not only TableGen definitions:

```text
workload + semantics + reference implementation
  -> dialect and type definitions
  -> verifiers and canonicalization
  -> optimization passes
  -> lowering passes
  -> runtime adapters or kernels
  -> manifest and CMake
  -> positive, negative, and differential tests
```

Generated extensions must pass an automated loop:

```text
generate -> compile -> verify IR -> check complete lowering
         -> differential test -> fuzz/sanitize -> benchmark -> revise
```

The first generated workload should be a small vector operation family rather
than graph traversal or full LLM inference. Vector distance and top-k operations
exercise custom IR, lowering, runtime calls, Arrow-compatible data, and
performance evaluation while keeping the semantic oracle manageable.

## 19. Expected Scope

The plugin-system MVP is expected to add approximately 3,000 to 6,000 lines of
core code and tests, excluding generated TableGen output. The loader itself is
small; most work is in build identity, MLIR context isolation, deterministic
pipeline composition, JIT symbol ownership, persistence, and lifecycle tests.

The first engineering task on the server should be Phase 0. A successful
TypeID/JIT spike removes the largest uncertainty and determines the correct
shared-library layout before the rest of the implementation is committed.
