# pp-node

**Prototype Pollution Gadget Detector** — A Node.js fork

Automatically detects and reports Prototype Pollution gadget reads during
Node.js runtime execution.

## Overview

Prototype Pollution (PP) is an attack technique that injects arbitrary
properties into `Object.prototype`. This project modifies V8's property lookup
pipeline to catch the exact moment a **non-built-in property is read from
`Object.prototype`** and reports it to stderr with the full JavaScript stack
trace.

This allows you to run real Node.js applications and discover which code paths
consume PP gadgets at runtime.

## How It Works

### V8 Property Access Architecture

When JavaScript accesses `obj.prop`, V8 takes the following internal path:

```
obj.prop
  → Ignition bytecode handler (LdaNamedProperty)
  → LoadIC_BytecodeHandler (CSA machine code)
  → IC hit?  → handler executes directly (no C++ involved)
  → IC miss? → GenericPropertyLoad (CSA machine code)
               → lookup_prototype_chain loop
               → Walks up the prototype chain in machine code
               → Found on Object.prototype?
                  → return_value: Returns directly — never calls C++ Object::GetProperty()
```

**Key insight**: V8's CodeStubAssembler (CSA) generates machine code that
traverses the prototype chain and returns values **without ever entering C++**.
This means hooking `Object::GetProperty()` alone is insufficient — the CSA fast
path bypasses it entirely.

### Interception Point

The detection hook is placed inside `GenericPropertyLoad()` in
`accessor-assembler.cc`, at the `return_value` label of the
`lookup_prototype_chain` loop. When a property is found on `Object.prototype`, a
C++ runtime callback (`Runtime_ReportPPGadget`) is called to perform the
filtering and logging.

### Filtered Built-in Properties

The following standard `Object.prototype` properties are **not** reported as PP
gadgets:

`constructor`, `toString`, `valueOf`, `toLocaleString`, `hasOwnProperty`,
`isPrototypeOf`, `propertyIsEnumerable`, `__defineGetter__`, `__defineSetter__`,
`__lookupGetter__`, `__lookupSetter__`, `__proto__`

## Example Output

```
[PP-GADGET] Read from Object.prototype detected!
  Property: polluted

==== JS stack trace =========================================

    at myFunction (test.js:5:19)
    at main (test.js:10:3)
=====================
```

## Usage

1. Build (same as standard Node.js build)
2. Run:

```bash
./node your_app.js
```

No special V8 flags required. Detected PP gadget reads are printed to stderr.

## Test

```js
// test_pp.js
Object.prototype.polluted = "hacked";

const obj = {};
const val = obj.polluted; // ← [PP-GADGET] reported
console.log("val =", val);

// These are built-in properties, NOT reported
obj.toString;
obj.constructor;
```

## Modified Files

| File                                    | What                                                                                                                                                   | Why                                                                                                                                                                                                         |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `deps/v8/src/ic/accessor-assembler.cc`  | Added Object.prototype check in `GenericPropertyLoad()`'s `lookup_prototype_chain` → `return_value` path. Calls `Runtime_ReportPPGadget` when matched. | This is where V8's CSA machine code traverses the prototype chain and returns values directly without entering C++. It is the only point where prototype chain lookups can be intercepted in the fast path. |
| `deps/v8/src/runtime/runtime-object.cc` | Added `Runtime_ReportPPGadget` runtime function. Filters built-in property names, logs property name and JS stack trace to stderr.                     | CSA machine code cannot perform complex operations like string comparison against a list or stderr I/O. The heavy lifting is delegated to this C++ runtime callback.                                        |
| `deps/v8/src/runtime/runtime.h`         | Added `F(ReportPPGadget, 1, 1)` to `FOR_EACH_INTRINSIC_INTERNAL` macro.                                                                                | V8 requires runtime functions to be registered in this macro to generate a `Runtime::kReportPPGadget` ID that CSA code can use with `CallRuntime()`.                                                        |

## Limitations

- **DATA case only**: ACCESSOR case (getter/setter on `Object.prototype`) is not
  detected
- **Always on**: No flag to toggle on/off (proof of concept)
- **stderr only**: No file logging

## TODO

- [ ] Add a V8 flag (e.g. `--pp-detect`) to toggle gadget detection on/off
- [ ] Add an option to write detection output to a file instead of stderr
