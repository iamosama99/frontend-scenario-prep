# Curry, Compose & Pipe

## Quick Reference

| Concept | Mechanism | Why it matters |
|---|---|---|
| Curry | Transform `f(a, b, c)` into `f(a)(b)(c)` — and also `f(a, b)(c)` / `f(a, b, c)` | Enables partial application: fix some arguments now, supply the rest later |
| Compose | `compose(f, g, h)(x) === f(g(h(x)))` — right-to-left | Matches mathematical function composition notation; used by Redux's `compose` |
| Pipe | `pipe(f, g, h)(x) === h(g(f(x)))` — left-to-right | Reads in the order operations actually happen — often preferred for data-transformation chains |
| `fn.length` | Reports the number of *named, non-default, non-rest* parameters | The mechanism curry uses to know when "enough" arguments have arrived — and its exact limitation |

## The Scenario

"Implement a generic `curry` function that works for a function of any arity — so `curry(add)(1)(2)(3)`, `curry(add)(1, 2)(3)`, and `curry(add)(1, 2, 3)` should all work the same for a 3-argument `add`. Then implement `compose` and `pipe` for combining unary functions — and I want you to be precise about which direction each one applies functions in, because I've seen people get that backwards."

## Clarifying Questions

- **Should curry rely on `fn.length`, and what happens if the target function has default parameters or a rest parameter?** This is worth raising proactively rather than waiting to be asked, since it's the single biggest limitation of `fn.length`-based curry (`function f(a, b = 2) {}`.length is `1`, not `2` — default parameters after the first stop being counted; `function f(...args) {}`.length is `0`). I'd confirm the target functions are "normal," fixed-arity functions, which is the realistic case for something like a curried `add(a, b, c)`.
- **For compose/pipe — right-to-left vs. left-to-right — which is "compose" and which is "pipe"?** I want to state this explicitly rather than assume, since it's genuinely a common mix-up: `compose(f, g)(x) = f(g(x))` (rightmost function runs first — matches mathematical notation `f∘g`), while `pipe(f, g)(x) = g(f(x))` (leftmost function runs first — reads top-to-bottom like a pipeline).
- **Do compose/pipe need to support multi-argument first functions, or strictly unary functions chained together?** Standard implementations only guarantee correctness for a multi-arg *first* function in the chain (its result becomes the sole input to the next), since every function after the first only ever receives one value — the previous function's single return value. I'd state that constraint rather than silently assume it.
- **Is there a real use case here, or is this purely an algorithm exercise?** Worth naming: curry underlies point-free functional composition patterns (build a `filter(isEven)` once, reuse it everywhere), and `compose` specifically is what Redux's `applyMiddleware` uses internally to chain middleware — this isn't an academic-only construct.

## Approach & Trade-offs

**Curry**'s mechanism: return a function that accumulates arguments across calls. On each call, concatenate the newly-provided arguments to whatever's already been collected; if the total collected count is `>= fn.length`, invoke `fn` with everything collected so far; otherwise, return a new function that continues collecting (closing over what's been accumulated).

The key design decision is using `fn.length` as the arity signal rather than requiring the caller to specify arity explicitly — this is what makes `curry(fn)` a one-argument utility rather than `curry(fn, arity)`, matching how Lodash's `curry` behaves by default (though Lodash *does* accept an explicit arity as an escape hatch for exactly the default-parameter/rest-parameter limitation named above).

**Compose/pipe**'s mechanism: both are a straightforward `reduce` over the list of functions, differing only in reduce direction and initial application order. `compose` uses `reduceRight` (processing the function list from the end backward, matching "rightmost runs first"); `pipe` uses `reduce` (left to right, "leftmost runs first"). I implement `pipe` as `compose` with the arguments reversed, or vice versa, to make the relationship between them explicit in the code itself rather than two independently-written, easy-to-drift-apart implementations.

## Solution

```javascript
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    }
    return (...moreArgs) => curried.apply(this, [...args, ...moreArgs]);
  };
}
```

```javascript
function add(a, b, c) {
  return a + b + c;
}

const curriedAdd = curry(add);

curriedAdd(1)(2)(3);   // 6
curriedAdd(1, 2)(3);   // 6
curriedAdd(1, 2, 3);   // 6
curriedAdd(1)(2, 3);   // 6
```

```javascript
function compose(...fns) {
  return function (...args) {
    return fns.reduceRight((result, fn, index) => {
      // The last function (first to run, rightmost in the call) can take
      // multiple args; every subsequent function only ever gets one value.
      return index === fns.length - 1 ? fn(...result) : fn(result);
    }, args);
  };
}

function pipe(...fns) {
  return compose(...fns.reverse());
}
```

```javascript
const double = (x) => x * 2;
const increment = (x) => x + 1;
const square = (x) => x * x;

compose(square, increment, double)(3); // square(increment(double(3))) => square(7) => 49
pipe(double, increment, square)(3);    // square(increment(double(3))) => same result, read left-to-right
```

> **Check yourself:** Given `curriedAdd = curry((a, b, c) => a + b + c)`, why does `curriedAdd(1, 2, 3, 4)` still just return `6` (ignoring the `4`) rather than throwing or behaving differently — and is that the right behavior?

## Practical Use Case: Redux's `compose`

Redux's `applyMiddleware(mw1, mw2, mw3)` is, internally, a `compose(mw1, mw2, mw3)` call that wraps `dispatch` in layers — each middleware wraps the *next* middleware's dispatch, so the composed function calls `mw1`'s wrapper first, which calls `mw2`'s wrapper, which calls `mw3`'s wrapper, which finally calls the real `dispatch`. This is exactly why middleware order matters in Redux — `applyMiddleware(logger, thunk)` behaves differently from `applyMiddleware(thunk, logger)`, because composition order determines which middleware sees the action *first* and which one is closest to the real store.

## Gotchas

**`fn.length` breaks for default parameters.** `function f(a, b = 2, c) {}`.length` is `1`, not `3` — JavaScript only counts parameters *before* the first one with a default value (and a rest parameter contributes `0` regardless of position). A curry implementation relying on `fn.length` will think `f` only needs one argument and invoke it prematurely, before `c` has actually been supplied, silently passing `undefined` for `c`.

**`fn.length` is `0` for rest-parameter-only functions.** `function sum(...nums) {}`.length` is `0`, meaning `curry(sum)` invokes immediately on the first call, with zero accumulated arguments — curry fundamentally can't know when "enough" variadic arguments have arrived, since there's no fixed arity to compare against. Variadic functions need an explicit arity hint or an explicit "terminator" call (e.g., calling with zero arguments to signal "stop collecting and invoke now").

**Mixing up compose and pipe direction.** This is the single most common mistake in a live coding setting — confidently writing `compose` with left-to-right semantics (or vice versa) because the two terms don't inherently signal their direction to everyone equally. Stating the direction *out loud*, with a concrete two-function example, before writing code is the reliable way to avoid this in an interview.

**Assuming every function in a compose/pipe chain can be multi-argument.** Only the very first function to execute (the last one listed in `compose`, or the first one listed in `pipe`) can meaningfully receive multiple arguments — every function after it in the chain receives exactly one value: the previous function's return value. A candidate who writes generic multi-arg support for *every* step in the chain is solving a harder, differently-specified problem (and usually gets the plumbing subtly wrong).

**Placeholder arguments are an unstated advanced feature.** Lodash's curry supports a placeholder (`_`) that lets you skip a position now and fill it in later — `curriedAdd(_, 2, 3)(1)` — which requires tracking *which positions* are filled versus placeholder-pending, a meaningfully more complex data structure than a flat accumulated-args array. Not supporting it is a fine default; not knowing it's a gap is not.

## Follow-up Questions

**Q (High): Implement curry so that `add(1)(2)(3) === add(1,2)(3) === add(1,2,3)` for a 3-argument function. Walk through the accumulation logic.**

Answer: The returned `curried` function checks, on every call, whether the *total* arguments accumulated so far (previous calls' args plus this call's new args) meets or exceeds `fn.length`. If yes, invoke `fn` with everything collected. If no, return a new function that, when called, repeats the same check with the combined argument list — this is what allows `add(1)(2)(3)` (one argument added on each of three calls: after the third call, `1+1+1=3` args total, matching `fn.length`, triggering invocation) and `add(1,2)(3)` (two args, then one more: `2+1=3` args total, same trigger) to converge on the identical invocation, `fn(1,2,3)`, despite arriving via different call shapes. The recursion is structural, not arity-shape-specific — it doesn't care *how* the arguments were grouped across calls, only their cumulative count.

The trap: hardcoding logic for "exactly one argument per call" (only handling the `add(1)(2)(3)` shape) rather than genuinely accumulating an arbitrary number of arguments per call, which fails the `add(1,2)(3)` and `add(1,2,3)` cases the prompt explicitly requires.

---

**Q (High): What's the difference between `compose` and `pipe`? Implement both using `reduce`.**

Answer: `compose(f, g, h)(x)` evaluates as `f(g(h(x)))` — the rightmost function in the argument list runs *first*, and execution proceeds right-to-left, matching the mathematical composition notation `f ∘ g ∘ h`. `pipe(f, g, h)(x)` evaluates as `h(g(f(x)))` — the leftmost function runs first, execution proceeds left-to-right, reading like a literal pipeline/assembly line. Implementation-wise, `compose` is `fns.reduceRight((acc, fn) => fn(acc), initialArgs)` and `pipe` is `fns.reduce((acc, fn) => fn(acc), initialArgs)` — the *only* difference between the two is the reduce direction, which is why implementing `pipe` as `compose(...fns.reverse())` (or vice versa) is a clean way to express that they're the same mechanism, mirrored.

The trap: getting the direction backwards under interview pressure — this is common enough that explicitly narrating "rightmost-first for compose, leftmost-first for pipe" and verifying it against a two-function example before writing code is worth the 15 seconds it costs.

---

**Q (High): Why does `fn.length` break down for curry when the target function has default parameters or a rest parameter?**

Answer: `Function.prototype.length` is specified to count only the function's parameters up to (and not including) the first one with a default value, and rest parameters aren't counted at all regardless of their position — this is a deliberate spec choice reflecting that default and rest parameters represent "optional" or "variable" arity, which doesn't map cleanly onto a single integer. Concretely: `function f(a, b = 1, c) {}`.length` is `1` (only `a` is counted; `b`'s default value stops the count, and `c` isn't counted even though it has no default, because it comes *after* a defaulted parameter), and `function g(...rest) {}`.length` is `0`. A curry implementation that uses `fn.length` as "the number of arguments to wait for" will invoke `f` after just one argument has arrived, silently passing `undefined` for `b` and `c` rather than waiting for them — a real, silent correctness bug, not a crash, which makes it more dangerous.

The trap: assuming `fn.length` is a reliable general-purpose arity signal — it's specifically reliable *only* for functions with no default parameters and no rest parameter, and that constraint needs to be either enforced (validate/document it) or worked around (accept an explicit arity argument, as Lodash's `curry(fn, arity)` does).

---

**Q (Medium): How is `compose` used in a real library like Redux, and why does composition order matter there?**

Answer: Redux's `applyMiddleware(...middlewares)` builds an enhanced `dispatch` by composing each middleware's `(next) => (action) => {...}` wrapper function, using `compose` internally so that the *first* middleware listed ends up as the outermost wrapper around `dispatch` — meaning it's the first to see every dispatched action and the last to see the call chain "unwind." Concretely, with `applyMiddleware(logger, thunk)`, `logger` wraps `thunk`, which wraps the real `dispatch` — so `logger` logs the *thunk function itself* being dispatched (before `thunk` middleware has a chance to intercept and execute it), which is why the conventional Redux middleware ordering puts `thunk` (or similar action-transforming middleware) *before* `logger`, so `logger` sees the fully-resolved action, not the unresolved thunk. This is a direct, practical consequence of compose's right-to-left/outermost-first semantics leaking into an ordering requirement that trips up real projects.

The trap: treating middleware order as arbitrary or "just a convention" — it's a direct, mechanical consequence of how `compose` nests function calls, and getting it backwards causes a specific, debuggable class of bug (middleware seeing the wrong shape of data), not a vague "best practice" violation.

---

**Q (Medium): How would you support placeholder arguments in curry (like Lodash's `_`) to allow skipping positions?**

Answer: Instead of a flat array of accumulated arguments, you need a data structure that also tracks which positions are still "open" (placeholder-pending) versus filled. On each call, merge the new arguments into the existing accumulated array by position — a placeholder marker in the *new* call's arguments means "don't fill this slot yet, leave whatever was there (or leave it open)," while a placeholder in the *existing* accumulated array at a position that the new call provides a real value for means "fill this now." The invocation-readiness check then isn't simply "count >= `fn.length`" — it has to check that *every* position from `0` to `fn.length - 1` is filled with a real (non-placeholder) value, since a caller could technically supply more total arguments than the arity while still leaving specific positions open via placeholders.

The trap: treating placeholders as just "a value to filter out before counting" — that loses positional information (which specific slot was skipped), which is the entire point of a placeholder (`add(_, 2, 3)` needs to remember that position `0` specifically is still open, not just that "one slot" is open somewhere).

---

**Q (Low): How would you make `compose`/`pipe` work with functions that return Promises — async composition?**

Answer: Change the reducer's combining step from direct function application to `.then()`-chaining: instead of `(acc, fn) => fn(acc)`, use `(accPromise, fn) => accPromise.then(fn)`, seeding the initial accumulator with `Promise.resolve(initialArgs)`. This works uniformly whether or not any individual function actually returns a promise, since `.then()` auto-unwraps returned promises (and passes through plain values as already-resolved) — so a mixed chain of sync and async functions composes correctly without the caller needing to know which steps are async. The composed function's overall return value becomes a `Promise` (even if every step happens to be synchronous), which is a meaningful behavioral change callers need to be aware of — they now need `await`/`.then()` on the composed result even for an all-sync chain, which is the natural price of building in async support unconditionally.

The trap: trying to keep the composed function's return type "sync when possible, async when needed" — that's not really achievable cleanly in JavaScript's promise model (a function can't conditionally be sync or async based on runtime values without leaking that distinction to the caller), so committing to "the composed result is always awaitable" is the correct, honest trade-off to name.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Can implement curry using `fn.length` and closures, supporting all valid argument-grouping shapes, from memory
- [ ] Can state, without hesitating, which direction compose runs and which direction pipe runs, with a two-function example
- [ ] Can implement both compose and pipe using `reduce`/`reduceRight`
- [ ] Can explain precisely why default parameters and rest parameters break `fn.length`-based curry
- [ ] Can explain why Redux middleware order matters in terms of compose's execution order
- [ ] Can explain the core idea behind placeholder-argument support, even if not implementing it fully

---
*Next: Memoize With TTL & Cache Eviction — another closures-based utility, this time about caching function results correctly (including the concurrent-async-call trap).*
