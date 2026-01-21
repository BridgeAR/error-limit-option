# ECMAScript Proposal: Error option `limit`

## Status

Champion: Ruben Bridgewater

Author: Ruben Bridgewater <ruben@bridgewater.de>

Stage: 1

## Overview

This proposal introduces a new per-error `Error` options property, `limit`, that specifies the maximum number of implementation defined stack frames to capture for that specific error instance.

- `limit` is a numeric option interpreted using ToIntegerOrInfinity.
- If provided, the local `limit` overrides any global stack trace limit (e.g., V8's `Error.stackTraceLimit`) for that error only.
- Negative values throw a RangeError; `limit` must be a non-negative integer, `undefined`, or `Infinity`.
- Conceptually, this behaves like temporarily setting the V8 specific `Error.stackTraceLimit` immediately before creating the error and restoring it right after, but without any global side effects.

This provides a standardized, cross-engine way to control stack depth per error instance and avoids manipulating a global knob that affects unrelated errors.

## Proposed API

```js
new Error(message, { limit: 3 })
new TypeError(message, { limit: 0 })
new RangeError(message, { limit: 50 })
new AggregateError(iterable, message, { limit: Infinity })
// ... applies to all built-in Error constructors that accept an options bag
```

### Validation

- If `options` is provided and has a `limit` property whose value is not `undefined`, the engine computes `n = ToIntegerOrInfinity(limit)`.
  - If `n < 0`, throw a `RangeError`.
  - Otherwise, record `n` on the error instance.
- If `limit` is `undefined` or absent, the default behavior is unchanged (the global limit, if any, applies).

## Rationale: a local limit is more intuitive

- The global `Error.stackTraceLimit` defined in V8 is coarse-grained and impacts all errors, including those that do not need deeper stacks.
- A per-error `limit` expresses intent exactly where it matters and avoids surprising global side effects.
- The local limit always overrides the global limit for that specific error instance, which matches developer expectations when tailoring diagnostics.
- Eliminates boilerplate patterns that temporarily modify the global `Error.stackTraceLimit` just to construct one error.

### Why the global `Error.stackTraceLimit` defined in V8 is error-prone

- Changes are process-wide and time-based, so unrelated errors can inherit the wrong limit.
- Code between “set” and “reset” can throw synchronously, skipping the reset.
- Asynchronous boundaries make it easy to affect other in-flight work during `await`.
- People sometimes simply forget to reset the global knob.

Examples:

1. Synchronous throw during error creation skips reset

```js
// Global mutation that never resets if message coercion throws
const prev = Error.stackTraceLimit;
Error.stackTraceLimit = 2;

// toString can throw during Error(message) coercion
const msg = { toString() { throw new TypeError('format failed'); } };
new Error(msg); // throws before any reset runs; global now stuck at 2
```

With a local limit, there’s no global mutation:

```js
throw new Error('boom', { limit: 2 });
```

1. Asynchronous hazard: unrelated errors use your temporary limit

```js
async function makeErrorLater() {
  const prev = Error.stackTraceLimit;
  Error.stackTraceLimit = Infinity; // intend: deep trace for this one error

  // Meanwhile, other tasks executing during this await will also see Infinity.
  await doWorkElsewhere();

  const err = new Error('x'); // gets deep trace...
  Error.stackTraceLimit = prev; // ...but other errors during await did too.
}
```

With a local limit, only the intended error is affected:

```js
async function makeErrorLater() {
  await doWorkElsewhere();
  throw new Error('x', { limit: Infinity });
}
```

1. Simple omission: forgetting to reset

```js
Error.stackTraceLimit = 1;
// ... time passes, code evolves ...
// reset was forgotten; all subsequent errors now have too-short stacks
```

Per-error `limit` avoids this entire class of mistakes.

## Use cases

- Performance-sensitive hot paths where only the first few frames are needed (e.g., `limit: 2`) to minimize overhead.
- Focused deep debugging for specific error types or code paths (e.g., `limit: 100` or `Infinity`) without changing behavior for other errors.
- Library and framework utilities that want predictable stack depth for their own diagnostic errors without affecting user code globally.

## In the wild (prior art)

Developers frequently simulate per-error control by temporarily changing the global `Error.stackTraceLimit` before creating an error and resetting it afterward. Examples and references:

- MDN: `Error.stackTraceLimit` (documents temporary adjustment patterns) — `https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/stackTraceLimit`
- Node.js documentation: Errors — `https://nodejs.org/api/errors.html`
- GitHub code search: “Error.stackTraceLimit = Infinity” — `https://github.com/search?q=%22Error.stackTraceLimit+%3D+Infinity%22&type=code`
- longjohn (async stack tracing) sets very high/Infinity limits — `https://github.com/mattinsler/longjohn`
- Mocha full trace option (ecosystem precedent for “full stacks”) — `https://mochajs.org/#--full-trace`

These demonstrate that developers already rely on per-error or context-local stack depth control; standardizing a per-error `limit` removes the need for global mutations.

## Semantics

- Let `L` be the recorded `limit` for the error instance, if provided; otherwise `undefined`.
- When the host captures the stack for the error:
  - If `L` is `undefined`, apply the host’s default/global stack trace limit.
  - Otherwise, apply `L` as the limit for this error only.
- Formatting and any other host-defined stack processing are unchanged.

A `stack frame` is implementation defined.

## Specification text (outline)

1. In each `Error` (and subclass) constructor that accepts an options argument:
   - Let `options` be the second (or third, for `AggregateError`) argument.
   - If `options` is not `undefined`:
     - Let `limit` be ? Get(options, "limit").
     - If `limit` is not `undefined`:
       - Let `n` be ? ToIntegerOrInfinity(`limit`).
       - If `n < 0`, throw a `RangeError`.
       - Record `n` on the error instance in an internal slot for later use during stack capture.
2. When capturing the stack for an error instance `E`:
   - Let `S` be the sequence of frames that would be captured for `E` absent this option.
   - Let `n` be `E.[[StackTraceLimitOverride]]`, if present; otherwise `undefined`.
   - Apply the stack length limit:
     - If `n` is `undefined`, apply the host’s global/default limit.
     - Otherwise, apply `n` to `S`.
   - Produce the stack string from `S` as usual.

See the full spec draft:

- [spec.emu](./spec.emu)

## Non-goals and edge cases

- This proposal does not standardize stack string formatting, source map application, or async stack joining.
- If `limit` is `0`, no frames are included for the error beyond the error header (implementation defined).
- The option is per-error and does not affect other errors or future stack captures.

## Alternatives considered

- Continue using global `Error.stackTraceLimit`. Rejected for lack of precision, surprising global side effects, and boilerplate required to simulate per-error control.
- Add new host-specific APIs. Rejected; standardizing a language option is more portable and interoperable.

## Conclusion

`limit` provides an explicit, local way to control stack depth per error, matching developer expectations, improving ergonomics, and avoiding global side effects while aligning with ecosystem practice.
