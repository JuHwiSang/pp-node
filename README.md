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
              → kNonExistent handler? → return undefined (Interception Point 2)
  → IC miss? → GenericPropertyLoad (CSA machine code)
               → lookup_prototype_chain loop
               → Walks up the prototype chain in machine code
               → Found on Object.prototype?
                  → return_value: Calls Runtime_ReportPPGadgetCandidateProto (Interception Point 1)
               → Not found anywhere (proto == null)?
                  → return_undefined: Calls Runtime_ReportPPGadgetCandidate (Interception Point 3)
```

**Key insight**: V8's CodeStubAssembler (CSA) generates machine code that
traverses the prototype chain and returns values **without ever entering C++**.
This means hooking `Object::GetProperty()` alone is insufficient — the CSA fast
path bypasses it entirely.

### Interception Points

There are **three** interception points in V8's CSA-generated machine code:

#### IP1: Property found on Object.prototype (`return_value` label)

Located in `GenericPropertyLoad()` in `accessor-assembler.cc`, at the
`return_value` label of the `lookup_prototype_chain` loop. When a property is
found on `Object.prototype`, a C++ runtime callback (`Runtime_ReportPPGadget`)
is called to perform the filtering and logging.

This catches: `Object.prototype.x = 1; ({}).x` — _actual_ pollution reads.

#### IP2: IC cached non-existent handler (`nonexistent` label)

Located in `HandleLoadICSmiHandlerLoadNamedCase()` in `accessor-assembler.cc`.
After the first lookup determines a property doesn't exist, V8 caches a
`kNonExistent` handler in the IC (Inline Cache). Subsequent accesses to the same
property on objects with the same map hit this cached handler, returning
`undefined` directly from machine code. The hook calls
`Runtime_ReportPPGadgetCandidate` before returning `undefined`.

This catches: repeated `obj.nonExistent` accesses via the IC fast path.

#### IP3: Prototype chain miss (`return_undefined` label)

Located in `GenericPropertyLoad()` in `accessor-assembler.cc`, at the
`return_undefined` label. When the prototype chain walk reaches `null` (the end)
without finding the property, this path returns `undefined`. The hook calls
`Runtime_ReportPPGadgetCandidate` before returning.

This catches: first-time `obj.nonExistent` accesses via the slow path.

### Filtered Built-in Properties

The following standard `Object.prototype` properties are **not** reported as PP
gadget candidates (for IP1 only):

`constructor`, `toString`, `valueOf`, `toLocaleString`, `hasOwnProperty`,
`isPrototypeOf`, `propertyIsEnumerable`, `__defineGetter__`, `__defineSetter__`,
`__lookupGetter__`, `__lookupSetter__`, `__proto__`

## Example Output

```
[PP-GADGET-CANDIDATE] Read of non-built-in property from Object.prototype!
  Property: polluted

==== JS stack trace =========================================

    at myFunction (test.js:5:19)
    at main (test.js:10:3)
=====================

[PP-GADGET-CANDIDATE] Non-existent property access detected!
  Property: nonExistentProp

==== JS stack trace =========================================

    at myFunction (test.js:6:19)
    at main (test.js:10:3)
=====================
```

## Usage

1. Build (same as standard Node.js build)
2. Run:

```bash
./node your_app.js
```

No special V8 flags required. Detected PP gadget candidates are printed to
stderr.

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

| File                                    | What                                                                                                                                                                                                                        | Why                                                                                                                                                                                                                                                        |
| --------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `deps/v8/src/ic/accessor-assembler.cc`  | **IP1**: Added Object.prototype check in `GenericPropertyLoad()`'s `lookup_prototype_chain` → `return_value` path. Calls `Runtime_ReportPPGadgetCandidateProto` when a non-built-in property is read from Object.prototype. | This is where V8's CSA machine code traverses the prototype chain and returns values directly without entering C++. It is the only point where prototype chain lookups can be intercepted in the fast path.                                                |
| `deps/v8/src/ic/accessor-assembler.cc`  | **IP2**: Added `Runtime_ReportPPGadgetCandidate` call in `HandleLoadICSmiHandlerLoadNamedCase()`'s `nonexistent` label, before returning `undefined`.                                                                       | After the first lookup, V8 caches a `kNonExistent` handler in the IC. Subsequent accesses to the same non-existent property hit this cached handler, bypassing the prototype chain walk entirely. Without this hook, repeated accesses would be invisible. |
| `deps/v8/src/ic/accessor-assembler.cc`  | **IP3**: Added `Runtime_ReportPPGadgetCandidate` call in `GenericPropertyLoad()`'s `return_undefined` label, before returning `undefined`.                                                                                  | When the prototype chain walk reaches `null` without finding the property, this slow path returns `undefined`. This is where first-time non-existent property accesses are intercepted.                                                                    |
| `deps/v8/src/runtime/runtime-object.cc` | Added `Runtime_ReportPPGadgetCandidateProto` (filters built-in property names, logs to stderr) and `Runtime_ReportPPGadgetCandidate` (logs non-existent property accesses to stderr).                                       | CSA machine code cannot perform complex operations like string comparison against a list or stderr I/O. The heavy lifting is delegated to these C++ runtime callbacks.                                                                                     |
| `deps/v8/src/runtime/runtime.h`         | Added `F(ReportPPGadgetCandidateProto, 1, 1)` and `F(ReportPPGadgetCandidate, 1, 1)` to `FOR_EACH_INTRINSIC_INTERNAL` macro.                                                                                                | V8 requires runtime functions to be registered in this macro to generate `Runtime::kReportPPGadgetCandidateProto` and `Runtime::kReportPPGadgetCandidate` IDs that CSA code can use with `CallRuntime()`.                                                  |

## Limitations

- **DATA case only**: ACCESSOR case (getter/setter on `Object.prototype`) is not
  detected
- **Always on**: No flag to toggle on/off (proof of concept)
- **stderr only**: No file logging
- **No deduplication**: Same gadget candidate may be reported multiple times for
  repeated accesses

## TODO

- [ ] Add a V8 flag (e.g. `--pp-detect`) to toggle gadget detection on/off
- [ ] Add an option to write detection output to a file instead of stderr
- [ ] Add deduplication to suppress repeated reports for the same (property
      name, source location) pair
