# pp-node

**Prototype Pollution Gadget Detector** — A Node.js fork

Automatically detects and reports Prototype Pollution gadget reads during
Node.js runtime execution.

## Overview

Prototype Pollution (PP) is an attack technique that injects arbitrary
properties into `Object.prototype`. This project modifies V8's property read
pipeline (`Object::GetProperty`) to catch the exact moment a **non-built-in
property is read from `Object.prototype`** and reports it to stderr.

This allows you to run real Node.js applications and discover which code paths
consume PP gadgets at runtime.

## How It Works

```
obj.someProperty
  → V8 LookupIterator traverses the prototype chain
  → Property found on Object.prototype
  → Not a built-in property
  → [PP-GADGET] logged to stderr with JS stack trace
```

### Interception Point

`Object::GetProperty()` in `deps/v8/src/objects/objects.cc`, at the `DATA` case.

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
./node --no-use-ic --no-opt your_app.js
```

Detected PP gadget reads are printed to stderr.

### Why `--no-use-ic --no-opt`?

V8 uses Inline Caches (IC) and optimizing compilers (TurboFan/Maglev) to speed
up property access. Once cached, property reads bypass `Object::GetProperty()`
entirely — which is where our detection hook lives. These flags force all
property accesses through the slow path so every read is intercepted.

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

| File                             | Changes                                                                                                          |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `deps/v8/src/objects/objects.cc` | Added `MaybePrintPPGadget()`, `IsBuiltinObjectPrototypeProperty()`. Called from `Object::GetProperty` DATA case. |

## Limitations

- **DATA case only**: ACCESSOR case (getter/setter on `Object.prototype`) is not
  detected
- **Requires `--no-use-ic --no-opt`**: Detection relies on the slow property
  lookup path; IC/optimized code bypasses it
- **Always on**: No flag to toggle on/off (proof of concept)
- **stderr only**: No file logging

## TODO

- [ ] Add a V8 flag (e.g. `--pp-detect`) to toggle gadget detection on/off
- [ ] Add an option to write detection output to a file instead of stderr
