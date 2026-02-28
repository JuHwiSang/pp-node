# pp-node

**Prototype Pollution Gadget Candidate Detector** — A Node.js fork

Automatically detects and reports Prototype Pollution gadget candidate reads
during Node.js runtime execution.

## Overview

Prototype Pollution (PP) is an attack technique that injects arbitrary
properties into `Object.prototype`. This project modifies V8's property lookup
pipeline to catch **two types of gadget candidate accesses** and reports them to
stderr with the full JavaScript stack trace:

1. **Non-built-in property read from `Object.prototype`** — A property that was
   added to `Object.prototype` (not a standard built-in) is read through an
   object. This means the pollution has already occurred and a gadget is active.

2. **Non-existent property access** — A property is not found anywhere in the
   prototype chain, returning `undefined`. If an attacker were to inject this
   property into `Object.prototype`, this access site would become a gadget.

This allows you to run real Node.js applications and discover which code paths
consume PP gadget candidates at runtime.

## How It Works

### V8 Property Access Architecture

When JavaScript accesses `obj.prop`, V8 takes the following internal path:

```
obj.prop
  → Ignition bytecode handler (LdaNamedProperty)
  → LoadIC_BytecodeHandler (CSA machine code)
  → IC hit?  → handler executes directly (no C++ involved)
              → kNonExistent handler? → report + return undefined (IP2)
  → IC miss? → C++ Runtime_LoadIC_Miss → LoadIC::Load
               → LookupIterator: walk prototype chain in C++
               → Not found? → report + return undefined (IP4)
               → Found? → return value (no PP hook on this path)
               → UpdateCaches: installs handler for next time
  → Megamorphic/Generic? → GenericPropertyLoad (CSA machine code)
                            → lookup_prototype_chain loop in machine code
                            → Found on Object.prototype?
                               → report (IP1) + return value
                            → Not found anywhere (proto == null)?
                               → report + return undefined (IP3)
```

**Key insight**: V8 has **three separate execution paths** for property access:

1. **IC hit** (CSA) — cached handler executes in machine code, no C++ involved
2. **IC miss** (C++) — first access at a given code site, falls through to C++
   `LoadIC::Load`. After the lookup, `UpdateCaches` installs a handler so the
   next access at the same site takes the IC hit path.
3. **Megamorphic/Generic** (CSA) — too many different receiver maps at one code
   site, or the `Builtins::kGetProperty` stub. Falls back to
   `GenericPropertyLoad` which walks the prototype chain in machine code.

All three paths must be hooked to catch every non-existent property access.

> **Coverage note**: Object.prototype pollution reads (type 1) are only detected
> on the megamorphic/generic CSA path (IP1). The IC miss C++ path (IP4) only
> detects non-existent property accesses (type 2). In practice this gap is small
> because after the first IC miss, `UpdateCaches` installs a handler and
> subsequent accesses go through the IC hit path (which has appropriate
> handlers) or eventually transition to the megamorphic path (where IP1 catches
> it).

### Interception Points

There are **four** interception points:

#### IP1: Property found on Object.prototype (CSA `return_value` label)

Located in `GenericPropertyLoad()` in `accessor-assembler.cc`, at the
`return_value` label of the `lookup_prototype_chain` loop. When a property is
found on `Object.prototype` (checked via `TaggedEqual` against
`INITIAL_OBJECT_PROTOTYPE_INDEX`), a C++ runtime callback
(`Runtime_ReportPPGadgetCandidateProto`) is called. The runtime function filters
out standard built-in properties (see list below) and logs the rest.

This catches: `Object.prototype.x = 1; ({}).x` — active pollution reads. This
path is taken when the IC is megamorphic or when `GenericPropertyLoad` is used
as a generic lookup fallback.

#### IP2: IC cached non-existent handler (CSA `nonexistent` label)

Located in `HandleLoadICSmiHandlerLoadNamedCase()` in `accessor-assembler.cc`.
After the first lookup determines a property doesn't exist, V8 caches a
`kNonExistent` handler in the IC (Inline Cache). Subsequent accesses to the same
property on objects with the same map hit this cached handler, returning
`undefined` directly from machine code. The hook calls
`Runtime_ReportPPGadgetCandidate` with the property name and receiver before
returning `undefined`. The runtime function checks if `Object.prototype` is in
the receiver's prototype chain — if not (e.g., `Object.create(null)`), the
report is skipped.

This catches: repeated `obj.nonExistent` accesses via the IC fast path.

#### IP3: Prototype chain miss (CSA `return_undefined` label)

Located in `GenericPropertyLoad()` in `accessor-assembler.cc`, at the
`return_undefined` label. When the prototype chain walk reaches `null` (via
`proto == null` check) without finding the property, this path returns
`undefined`. The hook calls `Runtime_ReportPPGadgetCandidate` with the property
name and receiver before returning. Same `Object.prototype` chain filter as IP2.

This catches: `obj.nonExistent` accesses via the `GenericPropertyLoad` CSA path
(megamorphic IC, or first-time generic lookup).

#### IP4: IC miss — C++ slow path (`LoadIC::Load`)

Located in `LoadIC::Load()` in `ic.cc`, at the
`!it.IsFound() &&
!ShouldThrowReferenceError()` branch. When the IC has no
cached handler (first access at a given code site, REPL input, new script
context), `Runtime_LoadIC_Miss` is invoked and the lookup happens entirely in
C++ via `LookupIterator`. When the property is not found (`!it.IsFound()`), the
code checks: (1) `IsString(*name)` to filter out Symbols, (2)
`IsJSReceiver(*object)` and walks the receiver's prototype chain via
`PrototypeIterator` to verify `Object.prototype` is present. Only if both checks
pass is the PP gadget candidate reported inline (using `PrintF` + `PrintStack`)
before returning `undefined`.

This catches: **first-time** `{}.test` accesses, REPL usage, and any case where
the IC is uninitialized. This is the **most commonly hit path** for non-existent
property detection because every property access starts as an IC miss before a
handler is installed.

### Filtered Built-in Properties

The following standard `Object.prototype` properties are **not** reported as PP
gadget candidates (for IP1 only):

`constructor`, `toString`, `valueOf`, `toLocaleString`, `hasOwnProperty`,
`isPrototypeOf`, `propertyIsEnumerable`, `__defineGetter__`, `__defineSetter__`,
`__lookupGetter__`, `__lookupSetter__`, `__proto__`

## Example Output

### stderr (default, no flag)

When running `test_pp.js` (see Test section below), output on stderr looks like:

```
[PP-GADGET-CANDIDATE] Read of non-built-in property from Object.prototype!
  Property: polluted

==== JS stack trace =========================================

    0: ExitFrame [pc: 0x...]
    1: StubFrame [pc: 0x...]
    2: /* anonymous */ [0x...] [test_pp.js:5] [bytecode=... offset=...]
    ...
=====================

[PP-GADGET-CANDIDATE] Non-existent property access detected!
  Property: nonExistent

==== JS stack trace =========================================

    0: ExitFrame [pc: 0x...]
    ...
=====================
```

### JSON Lines file (with `--pp-detect-output`)

When using `--pp-detect-output=result.jsonl`, each detection event is a single
JSON object on one line:

```jsonl
{"type":"Read of non-built-in property from Object.prototype!","property":"polluted","stack":"\n==== JS stack trace ...\n"}
{"type":"Non-existent property access detected!","property":"nonExistent","stack":"\n==== JS stack trace ...\n"}
```

| Field      | Description                                                                                                                                |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `type`     | Detection type: `"Read of non-built-in property from Object.prototype!"` (IP1) or `"Non-existent property access detected!"` (IP2/IP3/IP4) |
| `property` | Property name that was accessed                                                                                                            |
| `stack`    | V8 internal stack trace (`kPrintStackConcise` mode), JSON-escaped                                                                          |

> Note: Stack trace format is V8's internal `PrintStack(kPrintStackConcise)`,
> which includes native frames, bytecode offsets, and hex addresses — not the
> standard JavaScript `Error.stack` format.

## Usage

1. Build (same as standard Node.js build)
2. Run:

```bash
# Default: PP gadget candidates are printed to stderr as plain text
./node your_app.js

# File output: write detection results as JSON Lines to a file
./node --pp-detect-output=result.jsonl your_app.js
```

No special V8 flags required for detection itself — it is always on. The
`--pp-detect-output` flag only controls **where and how** results are written
(JSON Lines file vs. stderr plain text).

## Test

```js
// test_pp.js
Object.prototype.polluted = "hacked";

const obj = {};

// PP gadget candidate: property read from Object.prototype
const val = obj.polluted; // ← [PP-GADGET-CANDIDATE] reported
console.log("val =", val);

// PP gadget candidate: non-existent property access
obj.nonExistent; // ← [PP-GADGET-CANDIDATE] reported

// These are built-in properties, NOT reported
obj.toString;
obj.constructor;

// Own property, NOT reported
obj.x = 1;
obj.x;
```

## Modified Files

| File                                    | What                                                                                                                                                                                                | Why                                                                                                                                                                                                                          |
| --------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `deps/v8/src/flags/flag-definitions.h`  | Added `DEFINE_STRING(pp_detect_output, ...)` V8 flag.                                                                                                                                               | Controls output destination: when set, detection results go to the specified file as JSON Lines; when unset (`nullptr`), falls back to stderr.                                                                               |
| `deps/v8/src/ic/accessor-assembler.cc`  | **IP1**: Added Object.prototype check in `GenericPropertyLoad()`'s `return_value` path. Calls `Runtime_ReportPPGadgetCandidateProto`.                                                               | Catches non-built-in property reads from Object.prototype via the megamorphic CSA path.                                                                                                                                      |
| `deps/v8/src/ic/accessor-assembler.cc`  | **IP2**: Added `Runtime_ReportPPGadgetCandidate(name, receiver)` call in `HandleLoadICSmiHandlerLoadNamedCase()`'s `nonexistent` label.                                                             | Catches repeated non-existent property accesses via the IC cached `kNonExistent` handler. Receiver passed for Object.prototype chain filtering.                                                                              |
| `deps/v8/src/ic/accessor-assembler.cc`  | **IP3**: Added `Runtime_ReportPPGadgetCandidate(name, receiver)` call in `GenericPropertyLoad()`'s `return_undefined` label.                                                                        | Catches first-time non-existent property accesses via the megamorphic CSA slow path. Receiver passed for Object.prototype chain filtering.                                                                                   |
| `deps/v8/src/ic/ic.cc`                  | **IP4**: `ReportPPGadget()` helper (anonymous namespace) + call in `LoadIC::Load()` when `!it.IsFound()`. Walks receiver's prototype chain via `PrototypeIterator` to check for `Object.prototype`. | Catches first-time non-existent property accesses via C++ IC miss path. Filters out `Object.create(null)` objects that are immune to PP. Helper is a **deliberate copy** of the one in `runtime-object.cc` (see note below). |
| `deps/v8/src/runtime/runtime-object.cc` | `ReportPPGadget()` helper (anonymous namespace) + `Runtime_ReportPPGadgetCandidateProto(name)` + `Runtime_ReportPPGadgetCandidate(name, receiver)`. Both runtime functions now call the helper.     | Centralized output logic for IP1-3. Supports JSON Lines file output (when `--pp-detect-output` is set) or stderr plain text (default).                                                                                       |
| `deps/v8/src/runtime/runtime.h`         | Registered both runtime functions in `FOR_EACH_INTRINSIC_INTERNAL`. `ReportPPGadgetCandidate` takes 2 args (name, receiver).                                                                        | Required for CSA `CallRuntime()` to reference them.                                                                                                                                                                          |

> **Helper duplication note**: The `ReportPPGadget()` helper function exists in
> two places — `runtime-object.cc` and `ic.cc` — each inside an anonymous
> `namespace {}`. This is intentional: extracting to a shared `.h` file would
> require adding a new V8 header and modifying build files, which introduces
> risk disproportionate to the ~50 lines of duplicated code. Both copies are
> identical and should be kept in sync. Each copy has a comment referencing the
> other.

## Limitations

- **DATA case only**: ACCESSOR case (getter/setter on `Object.prototype`) is not
  detected
- **Always on**: No flag to toggle on/off (proof of concept)
- **No deduplication**: Same gadget candidate may be reported multiple times for
  repeated accesses
- **REPL noise**: In the Node.js REPL, internal machinery (e.g. acorn parser in
  `getInputPreview`, tab completion) generates false positive reports for normal
  built-in-related property accesses
- **Object.prototype read coverage gap on IC miss path**: When a polluted
  property is first accessed (IC miss → C++ `LoadIC::Load`), the property IS
  found on Object.prototype so `it.IsFound()` is true — IP4 does not fire.
  However, after `UpdateCaches` installs a handler, subsequent accesses will hit
  the IC fast path or eventually the megamorphic path where IP1 catches it.

## TODO

- [ ] Add a V8 flag (e.g. `--pp-detect`) to toggle gadget detection on/off
- [x] Add an option to write detection output to a file instead of stderr
      (`--pp-detect-output=<file>`, JSON Lines format)
