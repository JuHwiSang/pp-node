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
               → UpdateCaches: installs kNonExistent handler for next time
  → Megamorphic? → GenericPropertyLoad (CSA machine code)
                    → lookup_prototype_chain loop in machine code
                    → Found on Object.prototype?
                       → report (IP1)
                    → Not found anywhere (proto == null)?
                       → report + return undefined (IP3)
```

**Key insight**: V8 has **three separate execution paths** for property access:

1. **IC hit** (CSA) — cached handler executes in machine code, no C++ involved
2. **IC miss** (C++) — first access, falls through to C++ `LoadIC::Load`
3. **Megamorphic** (CSA) — too many different maps, uses `GenericPropertyLoad`

All three paths must be hooked to catch every non-existent property access.

### Interception Points

There are **four** interception points:

#### IP1: Property found on Object.prototype (CSA `return_value` label)

Located in `GenericPropertyLoad()` in `accessor-assembler.cc`, at the
`return_value` label of the `lookup_prototype_chain` loop. When a property is
found on `Object.prototype`, a C++ runtime callback
(`Runtime_ReportPPGadgetCandidateProto`) is called to filter built-in properties
and log.

This catches: `Object.prototype.x = 1; ({}).x` — active pollution reads via the
megamorphic CSA path.

#### IP2: IC cached non-existent handler (CSA `nonexistent` label)

Located in `HandleLoadICSmiHandlerLoadNamedCase()` in `accessor-assembler.cc`.
After the first lookup determines a property doesn't exist, V8 caches a
`kNonExistent` handler in the IC (Inline Cache). Subsequent accesses to the same
property on objects with the same map hit this cached handler, returning
`undefined` directly from machine code. The hook calls
`Runtime_ReportPPGadgetCandidate` before returning `undefined`.

This catches: repeated `obj.nonExistent` accesses via the IC fast path.

#### IP3: Prototype chain miss (CSA `return_undefined` label)

Located in `GenericPropertyLoad()` in `accessor-assembler.cc`, at the
`return_undefined` label. When the megamorphic prototype chain walk reaches
`null` without finding the property, this path returns `undefined`. The hook
calls `Runtime_ReportPPGadgetCandidate` before returning.

This catches: `obj.nonExistent` accesses via the megamorphic CSA slow path.

#### IP4: IC miss — C++ slow path (`LoadIC::Load`)

Located in `LoadIC::Load()` in `ic.cc`. When the IC has no cached handler (first
access, REPL input, new script context), `Runtime_LoadIC_Miss` is called and the
lookup happens entirely in C++. When the property is not found, the PP gadget
candidate is reported inline before returning `undefined`.

This catches: **first-time** `{}.test` accesses, REPL usage, and any case where
the IC is uninitialized.

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

| File                                    | What                                                                                                                                             | Why                                                                                                                                               |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| `deps/v8/src/ic/accessor-assembler.cc`  | **IP1**: Added Object.prototype check in `GenericPropertyLoad()`'s `return_value` path. Calls `Runtime_ReportPPGadgetCandidateProto`.            | Catches non-built-in property reads from Object.prototype via the megamorphic CSA path.                                                           |
| `deps/v8/src/ic/accessor-assembler.cc`  | **IP2**: Added `Runtime_ReportPPGadgetCandidate` call in `HandleLoadICSmiHandlerLoadNamedCase()`'s `nonexistent` label.                          | Catches repeated non-existent property accesses via the IC cached `kNonExistent` handler.                                                         |
| `deps/v8/src/ic/accessor-assembler.cc`  | **IP3**: Added `Runtime_ReportPPGadgetCandidate` call in `GenericPropertyLoad()`'s `return_undefined` label.                                     | Catches first-time non-existent property accesses via the megamorphic CSA slow path.                                                              |
| `deps/v8/src/ic/ic.cc`                  | **IP4**: Added inline PP gadget candidate reporting in `LoadIC::Load()` when `!it.IsFound()`.                                                    | Catches first-time non-existent property accesses via C++ IC miss path (REPL, new scripts, uninitialized IC). This is the most commonly hit path. |
| `deps/v8/src/runtime/runtime-object.cc` | Added `Runtime_ReportPPGadgetCandidateProto` (filters built-in properties) and `Runtime_ReportPPGadgetCandidate` (non-existent property access). | C++ runtime callbacks for CSA hooks (IP1-3).                                                                                                      |
| `deps/v8/src/runtime/runtime.h`         | Registered both runtime functions in `FOR_EACH_INTRINSIC_INTERNAL`.                                                                              | Required for CSA `CallRuntime()` to reference them.                                                                                               |

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
