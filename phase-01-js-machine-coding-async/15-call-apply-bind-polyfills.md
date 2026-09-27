# call/apply/bind Polyfills

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| Borrow `this` via a temporary property | Attach the function as a temp property on the target object, invoke it as `obj.fn(...)`, then delete it | This is the actual mechanism that makes `this` binding work in JS — method calls bind `this` to "whatever object the function was called *as a property of*," so temporarily making the function a property is how you control that |
| `apply` takes an array, `call` takes a spread list | Otherwise identical to each other | The only functional difference is argument-passing shape; this is easy to state but the *implementation* of `apply` from scratch requires converting an arbitrary-length array into an argument list, which is the actual interesting part |
| `bind` returns a new function, doesn't invoke immediately | The returned function captures `thisArg` and any pre-bound args in a closure, applying them plus any later-supplied args when eventually called | `bind` is deferred — the whole point is producing a reusable, pre-configured function, not executing anything right away |
| `bind`+`new` interaction | A bound function used as a constructor (`new boundFn()`) ignores the bound `this` and uses the newly created object instead | This is the trickiest correctness requirement — `bind`'s `this`-override doesn't survive being used with `new`, by spec design |

## The Scenario

"Implement your own versions of `Function.prototype.call`, `.apply`, and `.bind` — without using the native ones. Show that they behave identically to the built-ins, including edge cases like calling with no `thisArg`, and using a `bind`-returned function as a constructor with `new`."

## Clarifying Questions

- **Should the polyfills be attached to `Function.prototype` (true polyfills, usable as `fn.myCall(...)`), or written as standalone helper functions?** Given the phrasing ("implement your own versions of `Function.prototype.call`"), I'd default to attaching them to `Function.prototype` under different names (`myCall`, `myApply`, `myBind`) so they coexist with the natives for comparison, unless a standalone-function version is specifically wanted.
- **Does `myBind`'s returned function need to correctly support being called with `new`** (ignoring the bound `this` in that case, using the newly constructed object instead, per spec)? This is the single trickiest part of `bind` to get right, and it's very commonly the actual thing being tested when this question comes up — I'd confirm it's in scope, since a "simple" `bind` that ignores this case will look correct in every normal call but silently misbehave the moment someone does `new BoundClass()`.
- **What should happen when `thisArg` is `null` or `undefined`** — should the function's `this` become the global object (`window`/`globalThis`) as in **sloppy mode**, or stay `null`/`undefined` as in **strict mode**? Native behavior actually differs here depending on whether the *called function* is strict or sloppy mode, which is a subtlety worth surfacing — I'd default to matching strict-mode behavior (pass `thisArg` through as-is, including `null`/`undefined`) since virtually all modern code (ES modules, class bodies) is implicitly strict, and note the sloppy-mode auto-boxing-to-global-object behavior as a historical edge case rather than the primary path to implement.
- **Does `myBind` need to support partial application (binding some arguments now, supplying the rest at call time)?** Yes, this is core `bind` behavior, not an extension — `fn.bind(obj, a, b)` returns a function that, when later called with `(c, d)`, invokes the original as `fn.call(obj, a, b, c, d)`. I'd confirm this is understood as required, not optional, since it's easy to build a `bind` that only fixes `this` and forgets that pre-supplied arguments must also be prepended.

## Approach & Trade-offs

The unifying trick behind `call` and `apply` is the same: **you can't directly force an arbitrary function to run with an arbitrary `this` value except by making the call happen as a method call on that object.** In JavaScript, `this` inside a regular function call is determined by *how the function was invoked*, not where it was defined — specifically, `obj.method()` binds `this` to `obj` for the duration of that call. So to implement `myCall(thisArg, ...args)` on some function `fn`, the approach is: temporarily attach `fn` as a property of `thisArg` (using a key unlikely to collide, ideally a `Symbol` to guarantee no collision at all), invoke it as `thisArg[tempKey](...args)` — which correctly binds `this` to `thisArg` by the very mechanism just described — capture the return value, then delete the temporary property so `thisArg` isn't left mutated afterward.

**Why a `Symbol` key instead of a string key for the temporary property.** Using a string key like `'__fn__'` risks colliding with a real existing property on `thisArg` of the same name, which would be silently overwritten during the call and not properly restored afterward (only deleted, not "restored to its original value") — a `Symbol()` is guaranteed unique, so this collision risk doesn't exist at all, which is a meaningfully more correct implementation, not just a stylistic preference.

**`myApply` differs from `myCall` only in how the remaining arguments are shaped** — an array (or array-like, or `null`/`undefined` meaning "no arguments") versus a spread parameter list. Once the "borrow `this` via temporary property" trick is written once, `myApply` is nearly identical to `myCall`, just spreading an array instead of accepting `...args` directly — I'd implement `myCall` first, then either share the core logic via a small internal helper or just duplicate the ~5 lines, since the duplication here is small enough that extracting a shared helper doesn't obviously pay for itself, but I'd mention the option.

**`myBind` is the most involved of the three**, because it has to satisfy several distinct requirements simultaneously:
1. Return a **new function** immediately, without invoking the original — `bind` is about creating a reusable, pre-configured callable, not about running anything right away.
2. Support **partial application** — arguments passed to `bind` itself are prepended to whatever arguments the returned function is eventually called with.
3. The returned function, when called normally, invokes the original with the bound `this` (via `myApply`/`myCall`, built on the same borrowing trick).
4. The returned function, when called **with `new`**, must behave like a normal constructor call using the *original* function — meaning the bound `this` is **ignored**, and `this` instead becomes the newly created instance, with the prototype chain set up as if `new OriginalFn(...)` had been called directly. This is the part every naive implementation misses, and it's the reason `myBind`'s returned function needs to check, internally, "was I just invoked via `new`?" (detectable via `this instanceof boundFn`, exploiting the fact that `new` sets the new object's prototype to the constructor's `.prototype`, so `this instanceof boundFn` is `true` precisely when called via `new`) and branch accordingly.
5. The returned function should also correctly expose the original function's `.prototype` for the `instanceof` check above to work, and ideally set the bound function's own prototype chain (`Object.create(originalFn.prototype)`) so that objects constructed via the bound function correctly inherit from the original function's prototype.

## Solution

```javascript
Function.prototype.myCall = function (thisArg, ...args) {
  if (typeof this !== 'function') {
    throw new TypeError('myCall must be invoked on a function');
  }

  // null/undefined thisArg: strict-mode-like passthrough. A real global-object
  // fallback would apply here for sloppy-mode functions — omitted as a historical
  // edge case, noted explicitly rather than silently ignored.
  const context = thisArg == null ? globalThis : Object(thisArg);

  const key = Symbol('fn'); // guaranteed no collision with existing properties
  context[key] = this;
  try {
    return context[key](...args); // invoked as a method call — this is what binds `this`
  } finally {
    delete context[key]; // never leave the temporary property behind, even if fn throws
  }
};

Function.prototype.myApply = function (thisArg, argsArray) {
  if (typeof this !== 'function') {
    throw new TypeError('myApply must be invoked on a function');
  }
  const args = argsArray == null ? [] : Array.from(argsArray); // supports array-likes too
  return this.myCall(thisArg, ...args);
};

Function.prototype.myBind = function (thisArg, ...boundArgs) {
  if (typeof this !== 'function') {
    throw new TypeError('myBind must be invoked on a function');
  }
  const originalFn = this;

  function boundFn(...callArgs) {
    const allArgs = [...boundArgs, ...callArgs];

    // Detect construction via `new`: when `new boundFn()` runs, `this` is a fresh
    // object whose prototype is boundFn.prototype, so this check is true only then.
    if (this instanceof boundFn) {
      // Ignore the bound thisArg entirely — construct using the *original* function,
      // with `this` as the newly created instance, matching native `new` semantics.
      return originalFn.myApply(this, allArgs);
    }

    return originalFn.myApply(thisArg, allArgs);
  }

  // Preserve the prototype chain so `new boundFn()` produces instances that are
  // correctly `instanceof originalFn`, and so the `this instanceof boundFn` check above works.
  boundFn.prototype = Object.create(originalFn.prototype || Object.prototype);

  return boundFn;
};
```

```javascript
function greet(greeting, punctuation) {
  return `${greeting}, ${this.name}${punctuation}`;
}

greet.myCall({ name: 'Ada' }, 'Hello', '!');       // "Hello, Ada!"
greet.myApply({ name: 'Grace' }, ['Hi', '.']);      // "Hi, Grace."

const greetAda = greet.myBind({ name: 'Ada' }, 'Hey');
greetAda('?');                                      // "Hey, Ada?" — partial application

function Person(name) {
  this.name = name;
}
Person.prototype.sayName = function () { return `I am ${this.name}`; };

const BoundPerson = Person.myBind({ name: 'ignored' });
const p = new BoundPerson('Real Name');
p.sayName();               // "I am Real Name" — bound `this` correctly ignored under `new`
p instanceof Person;       // true — prototype chain preserved
```

> **Check yourself:** Why does `this instanceof boundFn` correctly detect "was this call made via `new`," and what would go wrong if `myBind` didn't set `boundFn.prototype = Object.create(originalFn.prototype)`?

## Why the "Borrow `this` as a Temporary Property" Trick Works

This is worth isolating because it's the single mechanism the entire scenario rests on, and articulating *why* it works (not just that it does) is the real signal. `this` inside a normal function invocation is not lexically determined — unlike arrow functions (which capture `this` from their enclosing scope at definition time), a regular `function`'s `this` is determined **dynamically, at call time, based on the call syntax used**. Specifically: `someObject.someMethod()` sets `this` to `someObject` for that invocation, because the function was invoked *as a property access off* `someObject` — this is a rule about the *call expression's shape*, not about anything intrinsic to the function itself. `call`/`apply` need to run an arbitrary, already-existing function with an arbitrary `this`, without being able to rewrite how the function itself was originally defined or called at its original site — so the trick is to *manufacture* exactly the call shape that produces the desired `this`: temporarily make the function a property of the desired `thisArg`, then call it through that property. This works for *any* object as `thisArg`, precisely because it doesn't rely on anything about the function except that it can be invoked, and doesn't rely on anything about the object except that properties can be temporarily attached to it.

## Gotchas

**Using a plain string key for the temporary property, risking collisions.** `context.__fn = this` risks silently overwriting a real existing property named `__fn` on `thisArg`, and — critically — `delete context.__fn` afterward doesn't *restore* whatever was there before, it just removes it entirely, permanently losing that original property. A `Symbol()` key sidesteps this because it's guaranteed unique against every other key, string or symbol, that could already exist on the object.

**Forgetting `finally` when deleting the temporary property, so a throwing function leaves `thisArg` polluted.** If the original function throws during the borrowed call, and the temporary-property deletion isn't wrapped in a `finally` block, the exception propagates immediately and skips the `delete` — leaving the temporary property (and a reference to the original function) attached to `thisArg` indefinitely, both a correctness bug (the object now has an unexpected extra property) and a minor memory-retention issue.

**`myBind`'s returned function failing to handle `new` correctly.** The most common incomplete `bind` implementation only ever calls `originalFn.apply(thisArg, args)`, regardless of how the *bound* function itself was invoked — this looks completely correct under every normal call, but the moment someone does `new BoundConstructor()`, the bound `this` (whatever was passed to `bind`) incorrectly overrides what should be the freshly constructed instance, breaking `new` entirely for any bound constructor function. This is specifically why native `bind`'s behavior under `new` is a favorite "gotcha" follow-up — it's very easy to build something that passes every test that doesn't include a `new` call.

**Not preserving the prototype chain on the bound function.** Even after correctly detecting a `new` call, if `boundFn.prototype` isn't set to inherit from `originalFn.prototype` (via `Object.create`), then `new boundFn()` produces objects that are *not* `instanceof originalFn`, and any prototype methods (like `Person.prototype.sayName` in the example) wouldn't be reachable on instances created through the bound constructor — breaking inheritance-dependent code silently.

**Assuming `apply`'s second argument is always a real `Array`.** Native `apply` accepts any **array-like** object (has a `length` property and indexed properties — e.g., `arguments` objects, `NodeList`s), not strictly `Array` instances — an implementation that does `argsArray.map(...)` or relies on array-specific methods directly on the argument, rather than converting it first (`Array.from(argsArray)`), breaks for legitimate array-like inputs that aren't true arrays.

## Follow-up Questions

**Q (High): Why does the "temporarily attach the function as a property and call it that way" trick correctly set `this`, and why can't you just do `fn.this = thisArg` or similar?**

Answer: `this` is not a regular, assignable property of a function — it's a special binding resolved dynamically based on how a function is *invoked*, specifically the call expression's syntax. There is no `fn.this = value` mechanism because `this` isn't stored data on the function object; it's determined at call time by one of a small number of specific rules (plain function call → `undefined`/global object depending on strict mode; method call `obj.fn()` → `obj`; `new fn()` → the newly created object; arrow functions → lexically inherited from the enclosing scope, ignoring call-site rules entirely). Since `call`/`apply` need to run a function with an arbitrary chosen `this`, and the only spec-defined way to make `this` become a specific object is the method-call rule (`obj.fn()` → `this === obj`), the only available lever is to *manufacture* that exact call shape — attach `fn` as a property of the target object, invoke it through that property, then remove the property, since it was never meant to be a persistent addition to that object.

The trap: proposing `fn.this = thisArg` or any direct-assignment approach as if `this` were an ordinary mutable property — demonstrating that `this` binding is a call-site-determined dynamic mechanism, not stored state, is the core conceptual understanding this question is testing.

---

**Q (High): Walk through exactly what happens, step by step, when a function returned by your `myBind` is invoked with `new`, and why the bound `thisArg` must be ignored in that case.**

Answer: When `new boundFn(...)` executes, the JS runtime (per the `new` operator's own semantics) creates a brand-new plain object, sets that new object's internal prototype link to `boundFn.prototype`, and then invokes `boundFn` with `this` bound to that newly created object (not to whatever `thisArg` was captured by `bind`) — this happens *before* the body of `boundFn` even runs, as part of `new`'s own mechanics, which `myBind`'s implementation doesn't control directly but can *detect*. Inside `boundFn`'s body, checking `this instanceof boundFn` distinguishes this situation: when called via `new`, `this` is that freshly created object whose prototype chain includes `boundFn.prototype`, so the check is `true`; when called normally (`boundFn()` or `boundFn.call(x)`), `this` is whatever was actually passed (or the bound `thisArg` isn't involved in `this`'s value going into the function at all — it only matters once *inside*, when deciding what to forward to `originalFn`), and won't generally be `instanceof boundFn`. When the check is `true`, the implementation forwards the call to `originalFn` using `this` (the newly constructed object) instead of the captured `thisArg` — matching the semantic that a bound constructor should still construct correctly, with `bind`'s `this`-fixing being specifically a *call-time* override that `new`'s own object-creation semantics take precedence over.

The trap: describing this as "bind just ignores `new`" without being able to explain the actual detection mechanism (`this instanceof boundFn`, relying on `new`'s prototype-linking behavior) or the actual spec rationale (a bound function used as a constructor should still construct real instances of the original class, which wouldn't be possible if the artificially bound `this` silently won).

---

**Q (Medium): What's the actual difference between `call` and `apply`, and why does `apply` need `Array.from` (or equivalent) internally rather than just spreading the argument directly?**

Answer: The only semantic difference is how additional arguments are supplied: `call(thisArg, arg1, arg2, ...)` takes them as an explicit, individually-listed sequence, while `apply(thisArg, argsArray)` takes them bundled into a single array-like value. Internally, once `myCall` exists, `myApply` can be implemented almost entirely in terms of it — spread the array into individual arguments and delegate. The reason for `Array.from` (or similar array-like-to-array conversion) rather than directly spreading `argsArray` with `...argsArray` is that native `apply` explicitly supports **array-like** objects, not just true `Array` instances — most notably `arguments` objects (which have `length` and numeric indices but aren't real arrays) and, historically, `NodeList`s and other DOM collections. Spread syntax (`...x`) actually works on both arrays and any iterable, but array-*likes* that aren't iterable (some are, some historically weren't, depending on the exact object) need explicit conversion; `Array.from` handles both array-likes and iterables uniformly, which is the safer, more spec-faithful choice for a polyfill aiming to match native behavior across all the input shapes native `apply` accepts.

The trap: assuming `apply`'s second argument is always a genuine `Array` and using array-specific methods on it directly without conversion — this breaks for legitimate array-like (but non-Array) inputs that native `apply` is specified to accept.

---

**Q (Medium): Why does `myBind` need to support "partial application" (pre-supplying some arguments at bind time), and how do the bound and call-time arguments combine?**

Answer: `Function.prototype.bind`'s signature is `fn.bind(thisArg, ...boundArgs)` — any arguments after `thisArg` are captured at bind time and are always **prepended** to whatever arguments the returned function is later called with. So `const add5 = add.bind(null, 5)` followed by `add5(3)` invokes the original as `add(5, 3)`, not `add(3, 5)` and not `add(3)` — the bound args come first, in the order they were originally supplied to `bind`, followed by whatever the caller supplies to the bound function itself. This is genuinely useful for partial application / currying-adjacent patterns (fixing some configuration or context arguments once, letting call sites supply only the remaining, varying arguments) — it's not an obscure edge case, it's core, commonly-relied-upon `bind` behavior (e.g., `setTimeout(callback.bind(null, arg1, arg2), delay)` is an extremely common real-world pattern for passing extra arguments into a timer callback).

The trap: implementing `myBind` to only fix `this`, forgetting the argument-prepending behavior entirely — this produces a `bind` that works for the "just fix `this`" use case (the most commonly *tested* surface behavior) but silently drops or mishandles the equally common partial-application use case, which usually only gets caught if the interviewer specifically tests `bind` with extra bound arguments.

---

**Q (Low): How does `Function.prototype.name` and `.length` behave on a function returned by native `bind`, and would your polyfill match that?**

Answer: Native `bind` produces a function whose `.name` is prefixed with `"bound "` (e.g., `function greet(){}.bind(obj).name === "bound greet"`), and whose `.length` reflects the *original* function's declared parameter count minus however many arguments were pre-bound (floored at 0) — e.g., `function f(a,b,c){}.bind(null, 1).length === 2`. A from-scratch `myBind` as commonly implemented (a plain `function boundFn(...args) {...}`) won't automatically get either of these right — `.name` would be `"boundFn"` literally, and `.length` would be `0` (since `boundFn` is declared with a rest parameter, which doesn't count toward `.length`) rather than reflecting the original function's arity. Matching this exactly would require explicitly setting `Object.defineProperty(boundFn, 'name', { value: 'bound ' + (originalFn.name || '') })` and computing `.length` as `Math.max(originalFn.length - boundArgs.length, 0)` and defining it similarly, since both `.name` and `.length` are non-writable-but-configurable properties on functions by default.

The trap: assuming a functionally correct `bind` (right `this`, right arguments, right `new` handling) is a complete polyfill — native `bind` also carefully preserves metadata (`.name`, `.length`) that a straightforward implementation doesn't reproduce automatically, and while this is a lower-stakes detail than the `new`-handling behavior, being aware it exists (rather than being surprised by it) signals having actually compared a polyfill against the real spec rather than just against a few manual test calls.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can implement `myCall`, `myApply`, and `myBind` from memory, including the temporary-property `this`-borrowing trick
- [ ] Can explain precisely why that trick works, grounded in how `this` is actually resolved at call time (not lexically, for regular functions)
- [ ] Can implement and explain `myBind`'s `new`-detection branch (`this instanceof boundFn`) and why it's necessary
- [ ] Can explain why `boundFn.prototype` must be set to inherit from the original function's prototype
- [ ] Can explain `bind`'s partial-application argument-ordering behavior precisely
- [ ] Can name at least one detail (`.name`/`.length` metadata) that a functionally-correct polyfill still doesn't fully replicate

---
*Next: Event Loop Output Prediction — Tricky Async — shifts from implementing mechanisms to reading and predicting them: tracing exact execution order across the call stack, microtask queue, and macrotask queue, the conceptual foundation everything in this phase has been built on.*
