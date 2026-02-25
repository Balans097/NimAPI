# nimhcr.nim — Complete Reference

> **Module**: `system/nimhcr` — Nim's Runtime Library  
> **Full name**: Nim Hot Code Reloading (HCR) Runtime  
> **Compilation flag**: `-d:hotcodereloading` (user side) or `-d:createNimHcr` (library build)  
> **Delivered as**: A minimal, *never-reloaded* shared library (`nimhcr.dll` / `libnimhcr.so` / `libnimhcr.dylib`) that all reloadable modules link against.

---

## What is Hot Code Reloading?

Hot Code Reloading (HCR) lets you **modify, recompile, and reload individual modules of a running Nim program without restarting it**. The state of global variables is preserved across reloads; only the code (function bodies) and newly added symbols change.

This is useful for:
- Fast iteration on game/UI logic while keeping the application running.
- REPL-like development workflows.
- Live patching of long-running server processes.

---

## Architecture Overview

Understanding the three-layer design is essential before reading individual functions.

```
┌──────────────────────────────────────────────────────────────────┐
│  Main binary  (never reloaded — compiled once)                   │
│   • Calls hcrInit() once at startup                              │
│   • Calls hcrPerformCodeReload() when the user requests reload   │
└────────────────────────┬─────────────────────────────────────────┘
                         │ loads / manages
┌────────────────────────▼─────────────────────────────────────────┐
│  nimhcr shared library  (this file — also never reloaded)        │
│   • Owns the jump table  (permanent proc addresses)              │
│   • Owns the global heap  (permanent global variable storage)    │
│   • Owns the module import graph                                 │
└──────┬──────────────────────┬────────────────────────────────────┘
       │ registers            │ registers
┌──────▼──────┐        ┌──────▼──────┐   ...more reloadable modules
│  module_A   │        │  module_B   │
│  .so/.dll   │        │  .so/.dll   │   (rebuilt and reloaded
│  (reloadable│        │  (reloadable│    by hcrPerformCodeReload)
└─────────────┘        └─────────────┘
```

### How function pointers stay stable (trampolines)

Every function call between reloadable modules goes through a *trampoline*:

1. The trampoline is a tiny block of machine code (a `LongJumpInstruction`) stored in the permanently-mapped `jumpTable`.
2. At load time `hcrRegisterProc` writes a jump instruction into the trampoline pointing to the current `_actual` implementation.
3. On reload, `hcrRegisterProc` is called again: it *updates the existing trampoline* to point to the new `_actual`. All callers that hold a pointer to the trampoline automatically call the new code.

### How globals stay stable

1. On first load, `hcrRegisterGlobal` allocates `size` bytes on the heap and writes the address to the module's pointer variable. Returns `true` → initialise now.
2. On reload, the same call finds the existing record, writes the *same* address back, and returns `false` → skip initialisation. The variable value is preserved.

---

## Key Types

| Type | Description |
|------|-------------|
| `HcrProcGetter` | `proc(libHandle: pointer, procName: cstring): pointer` — callback used by modules to resolve symbols from the nimhcr shared library itself |
| `HcrGcMarkerProc` | `proc() {.nimcall, raises: [].}` — zero-argument GC marker callback registered per global |
| `HcrModuleInitializer` | `proc() {.nimcall.}` — signature of the generated `DatInit000`, `Init000`, `HcrCreateTypeInfos`, `HcrInit000` entry points inside each module DLL |

### Internal types (inside `createNimHcr`)

| Type | Description |
|------|-------------|
| `ShortJumpInstruction` | Packed 5-byte x86 relative jump: opcode `0xE9` + 32-bit offset |
| `LongJumpInstruction` | Packed 14-byte x86-64 indirect absolute jump: `0xFF 0x25` + 32-bit offset + 64-bit target pointer |
| `ProcSym` | Entry in a module's proc table: `jump` (pointer into jump table) + `gen` (generation counter) |
| `GlobalVarSym` | Entry in a module's global table: `p` (heap pointer) + `markerProc` + `gen` |
| `ModuleDesc` | Complete per-module record: proc table, global table, import list, DLL handle, content hash, generation, last-modification time, before/after handlers |

---

## Important Global State (inside `createNimHcr`)

| Variable | Type | Description |
|----------|------|-------------|
| `jumpTable` | `ReservedMemSeq[LongJumpInstruction]` | The permanently-mapped, executable memory region holding all trampolines. Default capacity: 500 MB on x86-64 (configurable via `HOT_CODE_RELOADING_JUMP_TABLE_SIZE`). |
| `modules` | `Table[string, ModuleDesc]` | Maps file path → module descriptor. The authoritative module registry. |
| `root` | `string` | File path of the main module. |
| `system` | `string` | File path of the `system` module. |
| `mainDatInit` | `HcrModuleInitializer` | Pointer to the main module's `DatInit000`. Called on every reload. |
| `generation` | `int` | Monotonically increasing integer. Incremented on each `hcrPerformCodeReload`. Modules/symbols with `gen < generation` after a reload pass are considered stale and removed. |
| `hashToModuleMap` | `Table[string, string]` | Maps a module's content hash → file path. Used by `hcrHasModuleChanged`. |
| `currentModule` | `string` | File path of the module currently being initialised. Set during `initGlobalScope` so that `hcrAddEventHandler` registers handlers in the right module. |
| `hcrDynlibHandle` | `pointer` | Handle to the nimhcr DLL itself, passed to each module so it can call `HcrProcGetter` to resolve HCR symbols. |
| `getProcAddr` | `HcrProcGetter` | Callback for resolving symbols from nimhcr — set once by `hcrInit` and forwarded to all modules during `initHcrData`. |

---

## Public API — Functions

All functions below are exported from the nimhcr shared library. On the *library side* (`-d:createNimHcr`) they are the implementations; on the *user side* (`-d:hotcodereloading`) they are imported declarations.

---

### `hcrInit`

```nim
proc hcrInit*(moduleList: ptr pointer, main, sys: cstring,
              datInit: HcrModuleInitializer,
              handle: pointer, gpa: HcrProcGetter)
```

**The single entry point called by the main module at program startup.** It bootstraps the entire HCR system:

1. Stores the main module path, system module path, `datInit` pointer, DLL handle, and proc-getter.
2. Reads `moduleList` (a null-terminated array of `cstring`s) to discover the direct imports of the main module.
3. Calls `recursiveDiscovery` to perform a DFS traversal of the full import graph, loading every module DLL.
4. Calls `initModules` to run the four-pass initialisation sequence (see [Initialisation Passes](#initialisation-passes)).
5. Sets `currentModule = root` so that top-level code in the main module registers its handlers correctly.

The `main` module itself is never loaded as a DLL (it is already the host binary); only its imports are.

```nim
# Generated by the Nim compiler in the main module's C code:
hcrInit(addr moduleList, "main", "system", datInit000, nimhcrHandle, getProcAddr)
```

---

### `hcrRegisterProc`

```nim
proc hcrRegisterProc*(module: cstring, name: cstring, fn: pointer): pointer
```

**Registers (or updates) a function in the permanent jump table.** Called from `DatInit000` of every reloadable module for each of its exported functions.

- **`module`**: file path of the owning module.
- **`name`**: the function's mangled name.
- **`fn`**: pointer to the `_actual` implementation in the just-loaded DLL.
- **Returns**: pointer to the trampoline entry (stable across reloads).

**First registration**: a new `LongJumpInstruction` slot is appended to `jumpTable`; the jump is written.  
**Subsequent reloads**: the *existing* jump entry is found, its jump is *overwritten* to point to the new `fn`. All existing callers holding the trampoline address now transparently call the new code.

```nim
# Generated C (simplified):
myFunc_ptr = (MyFuncType) hcrRegisterProc("mymodule.nim", "myFunc", myFunc_actual);
```

---

### `hcrGetProc`

```nim
proc hcrGetProc*(module: cstring, name: cstring): pointer
```

**Retrieves the trampoline address of a function registered by another module.** Used when module A needs a pointer to a function defined in module B. Unlike `hcrRegisterProc`, this does not allocate a new trampoline; it simply looks up the existing one.

Returns `nil` (an empty `ProcSym().jump`) if the symbol does not exist yet (deferred initialisation order).

```nim
# Generated C (simplified):
otherFunc_ptr = (OtherFuncType) hcrGetProc("othermodule.nim", "otherFunc");
```

---

### `hcrRegisterGlobal`

```nim
proc hcrRegisterGlobal*(module: cstring, name: cstring, size: Natural,
                        gcMarker: HcrGcMarkerProc, outPtr: ptr pointer): bool
```

**Registers (or retrieves) a global variable's permanent heap location.** Called from `Init000` (or `DatInit000`) of every reloadable module for each module-level variable.

- **`module`**: owning module file path.
- **`name`**: the global's mangled name.
- **`size`**: `sizeof` the variable's type — used to allocate space on first registration.
- **`gcMarker`**: a GC tracing callback for this global; may be `nil` for non-GC types.
- **`outPtr`**: output parameter — set to the permanent heap address.
- **Returns**: `true` on first registration (caller must execute initialisation code), `false` on subsequent calls (global already has its value — skip init).

The `true`/`false` return is the mechanism that prevents re-running initialisation on every reload while still making the global accessible:

```nim
# Generated Nim (conceptual):
var myGlobal_ptr: ptr MyType
if hcrRegisterGlobal("mymodule.nim", "myGlobal", sizeof(MyType), nil,
                     cast[ptr pointer](addr myGlobal_ptr)):
  myGlobal_ptr[] = MyType(field: initialValue)  # run only once
```

---

### `hcrGetGlobal`

```nim
proc hcrGetGlobal*(module: cstring, name: cstring): pointer
```

**Retrieves the heap address of a global registered by another module.** The returned pointer remains valid for the lifetime of the program (the global is never moved). Used when module A references a global defined in module B.

```nim
# Generated C (simplified):
otherGlobal = (OtherType*) hcrGetGlobal("othermodule.nim", "otherGlobal");
```

---

### `hcrHasModuleChanged`

```nim
proc hcrHasModuleChanged*(moduleHash: string): bool
```

**Returns `true` if the on-disk DLL for the module with the given content hash is newer than when it was last loaded.** Looks up the module path from `hashToModuleMap`, then compares `lastModification` against `getLastModificationTime`.

`moduleHash` is the build-time content hash of the module's source, embedded by the compiler. The indirection through hashes makes the query stable even if module paths change between builds.

```nim
if hcrHasModuleChanged(myModuleHash):
  echo "myModule has been rebuilt"
```

---

### `hcrReloadNeeded`

```nim
proc hcrReloadNeeded*(): bool
```

**Returns `true` if *any* registered module has changed on disk.** Iterates over `hashToModuleMap` and calls `hcrHasModuleChanged` for each entry. Short-circuits on the first changed module.

The user-facing name (via the `hotcodereloading` standard module) is `hasAnyModuleChanged`.

```nim
# Typical game loop pattern:
while running:
  update()
  render()
  if hasAnyModuleChanged():
    performCodeReload()
```

---

### `hcrPerformCodeReload`

```nim
proc hcrPerformCodeReload*()
```

**The main reload entry point.** Call this from the main module (or any non-reloadable code) when you want to apply changes. **Never call it while any reloadable function is on the call stack.**

Sequence of operations:

1. If `hcrReloadNeeded()` returns `false`, returns immediately.
2. **Disables the GC** (`GC_disable` or `GC_disableOrc`) for the duration of the reload to prevent the GC from tracing stale type info or marker procs mid-swap. Re-enabled via `defer`.
3. Increments `generation`.
4. Executes all *before-reload handlers* in DFS order (leaves first, root last).
5. Re-runs `recursiveDiscovery` + `initModules` to load changed DLLs and re-register their symbols.
6. Executes all *after-reload handlers* in the same DFS order.
7. Cleans up stale modules and symbols (those with `gen < generation` were not touched during the reload and are therefore no longer in the import graph).

```nim
# hotcodereloading module wrapper:
proc performCodeReload*() =
  hcrPerformCodeReload()
```

---

### `hcrAddEventHandler`

```nim
proc hcrAddEventHandler*(isBefore: bool, cb: proc () {.nimcall.})
```

**Registers a before- or after-reload callback for the currently initialising module.** The `isBefore` flag distinguishes the two phases:

- `isBefore = true` → callback runs just *before* the module is reloaded (good for teardown, saving state).
- `isBefore = false` → callback runs just *after* the reload (good for re-registration, restore state).

Handlers are stored in `modules[currentModule].handlers` and are automatically cleared at the start of each reload (so the module re-registers them fresh when its top-level scope re-executes).

The user-facing API is via the `hotcodereloading` module:

```nim
import hotcodereloading

beforeCodeReload:
  echo "saving state before reload..."
  saveMyState()

afterCodeReload:
  echo "restoring state after reload..."
  restoreMyState()
```

These macros expand into `hcrAddEventHandler` calls.

---

### `hcrAddModule`

```nim
proc hcrAddModule*(module: cstring)
```

**Ensures a module entry exists in the `modules` table.** Creates a blank `ModuleDesc` for `module` if it is not already registered. Called by the compiler-generated code for modules that are imported but not yet loaded (deferred discovery).

---

### `hcrGeneration`

```nim
proc hcrGeneration*(): int
```

**Returns the current reload generation counter.** Starts at 0, incremented by 1 on each `hcrPerformCodeReload`. Useful in diagnostics and tests to verify that a reload has occurred.

```nim
let before = hcrGeneration()
performCodeReload()
assert hcrGeneration() == before + 1
```

---

### `hcrMarkGlobals`

```nim
proc hcrMarkGlobals*() {.compilerproc, exportc, dynlib, nimcall, gcsafe.}
```

**GC root marker for all HCR-managed globals.** Called by the GC to prevent HCR-owned global memory from being collected. Iterates over every `GlobalVarSym` in every registered module and calls its `markerProc` (if non-nil).

This function is registered with the Nim runtime as a global root marker via `nimRegisterGlobalMarker`. It runs on the main thread only.

---

## Internal Functions

These are not exported. They are documented here to explain the reload lifecycle.

---

### `writeJump(jumpTableEntry, targetFn)`

Writes either a **short relative jump** (5 bytes, `0xE9`) or a **long indirect absolute jump** (14 bytes, `0xFF 0x25`) into `jumpTableEntry`, depending on whether the target address is within ±2 GB of the trampoline.

- Short jump: used when the DLL and jump table are close in memory (common with ASLR off or on small programs).
- Long jump: required on x86-64 when DLLs are loaded far from the jump table.

On **x86**: writes the absolute address of `absoluteAddr` directly.  
On **x86-64**: writes `offset = 0` so the CPU reads the pointer from the adjacent `absoluteAddr` field using RIP-relative addressing.

ARM support (8 or 16 bytes) is noted but not yet implemented.

---

### `loadDll(name)`

Loads a module's shared library:

1. Unloads the old handle (if any).
2. Makes a `.copy.{dllExt}` of the DLL — *this copy* is what gets loaded. This prevents file-locking on Windows and ensures the original can be overwritten by the build system.
3. Calls `loadLib` on the copy.
4. Reads `HcrGetImportedModules` from the DLL to get its import list.
5. Reads `HcrGetSigHash` to get the content hash; updates `hashToModuleMap`.
6. Clears the module's event handlers (they will be re-registered by top-level code).

---

### `initHcrData(name)`

Calls the `HcrInit000` entry point of the named module's DLL. This pass gives the module its `hcrDynlibHandle` and `getProcAddr` so it can call back into the nimhcr library.

---

### `initTypeInfoGlobals(name)`

Calls `HcrCreateTypeInfos` in the module's DLL. This pass allocates type-info globals (e.g. `TNimType` descriptors) before any `DatInit` runs, because `DatInit` may reference those descriptors.

---

### `initPointerData(name)`

Calls `DatInit000` in the module's DLL. This is where `hcrRegisterProc` and `hcrGetProc` calls for all the module's functions occur.

---

### `initGlobalScope(name)`

Calls `Init000` in the module's DLL. This is where:
- `hcrRegisterGlobal` calls are made (first call → initialise, subsequent → skip).
- Top-level Nim statements execute (only on first load, because `hcrRegisterGlobal` returns `false` on reload).
- Before/after handlers are re-registered via `hcrAddEventHandler`.

Sets `currentModule` to `name` so that `hcrAddEventHandler` knows which module to attach to.

---

### `recursiveDiscovery(dlls)`

DFS traversal of the module import graph starting from a list of direct imports. For each module:

- Already at current generation → skip (already processed).
- On-disk DLL not modified → update generation (keep alive), recurse into its imports.
- Modified (or new) → call `loadDll`, recurse, add to `modulesToInit` and `allModulesOrderedByDFS`.

This procedure fills the two sequences that drive `initModules`.

---

### `initModules()`

Runs the four-pass initialisation sequence over all modules in `modulesToInit`:

| Pass | Entry point | What it does |
|------|------------|-------------|
| 1 | `HcrInit000` | Binds HCR function pointers in the module |
| 2 | `HcrCreateTypeInfos` | Registers type info globals |
| 3 | `DatInit000` | Registers proc trampolines |
| 4 | `Init000` | Registers globals, runs top-level code |

The `system` module always runs passes 3 and 4 before any other module, because other modules may depend on system globals (e.g. the allocator state).

---

### `cleanupSymbols(module)`

Removes stale symbols (procs and globals) from a module's tables — those whose `gen` is less than the current `generation`. For each stale global, `dealloc` is called to free the heap memory. For procs, only the table entry is removed (the trampoline slot is not yet reclaimed — see the free-list TODO in the source).

---

### `cleanupGlobal(module, name)`

Deletes a single global symbol from the module's table and calls `dealloc` on its heap pointer.

---

### `unloadDll(name)`

Calls `unloadLib` on the module's DLL handle (if loaded). After this the trampolines still exist but point to invalid code — they must be rewritten before being called again.

---

## Initialisation Passes

Each loaded module DLL exposes four generated entry points that nimhcr calls in a fixed sequence:

```
HcrInit000        ← Pass 1: bind pointers to nimhcr functions
HcrCreateTypeInfos ← Pass 2: allocate type-info globals
DatInit000        ← Pass 3: register procs (hcrRegisterProc / hcrGetProc)
Init000           ← Pass 4: register globals + execute top-level code
```

The sequence is designed so that later passes can safely use symbols registered in earlier passes.

---

## Environment Variables

| Variable | Default | Effect |
|----------|---------|--------|
| `HOT_CODE_RELOADING_JUMP_TABLE_SIZE` | `500` (MB on x86-64, `50` on x86) | Controls the maximum number of trampoline slots. Each slot is `sizeof(LongJumpInstruction)` bytes. Larger values reserve more virtual address space. |

---

## Compile-Time Flags

| Flag | Meaning |
|------|---------|
| `-d:createNimHcr` | Build this file as the nimhcr shared library. Activates all implementation code. |
| `-d:hotcodereloading` | Build a user program with HCR support. Imports the nimhcr symbols from the shared library. |
| `-d:testNimHcr` | Enables both the implementation and the client stubs simultaneously for unit tests. Also redacts timestamps and paths in trace output. |
| `-d:traceHcr` | Prints verbose trace lines (module loads, symbol registrations, etc.) to stdout. |

---

## Reload Lifecycle: Step-by-Step

```
User rebuilds module_B.nim
          │
          ▼
User calls performCodeReload()
          │
          ├─ hcrReloadNeeded() ── false → return early
          │
          ├─ true → GC_disable()
          │
          ├─ inc(generation)   # generation = N+1
          │
          ├─ recursiveExecuteHandlers(isBefore=true, root)
          │    └─ DFS: C handlers → B handlers → A handlers
          │         └─ each: call cb()
          │
          ├─ recursiveDiscovery(root.imports)
          │    └─ module_B is newer → loadDll(module_B)
          │         └─ copies module_B.so → module_B.so.copy.so
          │         └─ loads the copy
          │         └─ updates hashToModuleMap
          │
          ├─ initModules()   # 4-pass init for changed modules
          │    ├─ Pass 1 (HcrInit000)        ← bind HCR pointers
          │    ├─ Pass 2 (HcrCreateTypeInfos) ← type info globals
          │    ├─ Pass 3 (DatInit000)         ← update trampolines
          │    └─ Pass 4 (Init000)            ← re-register globals
          │         └─ hcrRegisterGlobal returns false → skip init
          │         └─ hcrAddEventHandler re-registers handlers
          │
          ├─ recursiveExecuteHandlers(isBefore=false, root)
          │    └─ DFS: C handlers → B handlers → A handlers
          │
          └─ cleanup stale modules & symbols (gen < N+1)
               └─ dealloc removed globals
               └─ unloadLib old DLL copies
               └─ GC_enable()
```

---

## Constraints and Gotchas

**No reloadable call on the stack during reload**  
Old DLLs are unloaded during `hcrPerformCodeReload`. If any function from a reloaded module is currently on the call stack, the return will jump to freed memory and crash. Always trigger reloads from code in the main (non-reloadable) module.

**Global initialisers run once only**  
`hcrRegisterGlobal` returns `false` on every reload, so initialisation code never re-runs. To change a global's value on reload, use a `beforeCodeReload` / `afterCodeReload` handler.

**Removing a global, then re-adding it, resets it**  
If you remove a global, reload (the global is `dealloc`'d), then add it back and reload again, it will be freshly initialised. This is the only way to reset a global's value.

**Top-level code runs once**  
Top-level Nim statements (outside of procs) are only executed on the first load of a module, not on reloads. Use `afterCodeReload` for code that should run on every reload.

**Main module is never reloaded**  
Only imported modules can be reloaded. The main module's binary is the host process. Therefore calls that originate from the main module cannot be updated by HCR.

**ARM not yet supported**  
The trampoline writing code only handles x86 and x86-64. ARM/ARM64 constants are defined but `writeJump` is not implemented for them.

**The jump table has a fixed maximum capacity**  
Each proc registration reserves one slot permanently (free-list reclamation is a TODO). Programs with very many unique functions may need to increase `HOT_CODE_RELOADING_JUMP_TABLE_SIZE`.
