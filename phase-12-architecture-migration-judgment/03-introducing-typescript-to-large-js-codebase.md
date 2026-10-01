# Introducing TypeScript to a Large Untyped JS Codebase

## Quick Reference

| Decision | Mechanism | Why it's the right call |
|---|---|---|
| Incremental, not big-bang | `allowJs` + `checkJs` off; rename files `.js → .ts(x)` as touched | Codebase keeps shipping; each step is mergeable |
| Start permissive, ratchet up | `strict: false` initially → per-flag/per-directory strictness; "no new `any`" rule | Prevents a thousand-error wall and demoralization |
| Type the boundaries first | API responses, shared utilities, design system, state/store | Highest leverage; types propagate inward |
| Enforce the direction | CI blocks new `.js` files and regressions in a strictness metric | Migration monotonically progresses |

## The Scenario

"We've got a 400k-line JavaScript React codebase, 60 engineers, no types. We keep hitting 'undefined is not a function' in production and onboarding is slow. The CTO wants TypeScript. How do you introduce it without stopping feature work?"

## Clarifying Questions

- **What problem are we solving — runtime errors, refactor fear, onboarding, API contract drift?** Types help most with some of these; knowing which tells me where to invest (e.g., API boundary types vs. internal helpers).
- **Is there buy-in from the team, or will this be resented?** A mandated migration with no support dies; I'd look for champions and decide on the support/training level.
- **What's the build setup (Babel, webpack, Vite) and monorepo structure?** TS can be compiled via Babel/esbuild (transpile only) with `tsc --noEmit` for checking — affects build time and rollout.
- **Test coverage and CI duration?** Typecheck adds CI time; large codebases need project references/incremental builds.
- **Are there generated sources of truth (OpenAPI/GraphQL schema)?** If yes, generated types give instant high-value coverage at the boundary.
- **Timeline and capacity?** Sets the pace: dedicated migration squad vs. opportunistic.

## Approach & Trade-offs

**Principle: make migration incremental, reversible, and monotonic.** No single PR converts 400k lines. TypeScript supports gradual adoption by design.

**Step 1 — Tooling without disruption.** Add `tsconfig.json` with `allowJs: true`, `checkJs: false`, `noEmit: true` (let the existing Babel/esbuild pipeline transpile; use `tsc --noEmit` only to check). JS and TS files interoperate immediately. Add editor/CI typecheck. Nothing breaks.

**Step 2 — Pick a starting strictness, and ratchet.** Beginning with `strict: true` on 400k lines of untyped JS yields thousands of errors; people give up. Start with `strict: false` (or `noImplicitAny: false`) and enable flags progressively (`noImplicitAny`, `strictNullChecks` — the highest-value one — `strictFunctionTypes`, …), or use per-directory `tsconfig`s so *new* and *migrated* areas are strict while the rest stays loose. Treat `strictNullChecks` as the main payoff for "undefined is not a function."

**Step 3 — Type from the outside in.** Highest leverage first: (a) API layer — generate types from OpenAPI/GraphQL so contract drift becomes a compile error; (b) shared utilities and the design-system components; (c) global store/state shapes; (d) then feature code, in the order it's touched. Types at boundaries propagate inferred types inward, giving disproportionate benefit.

**Step 4 — Convert files as they're touched, plus targeted pushes.** "If you edit a file substantially, rename it to `.ts`/`.tsx`" is a low-friction opportunistic rule. Supplement with dedicated sprints for hot shared modules. Codemods (`ts-migrate`) can mass-rename and add `// @ts-expect-error` placeholders, converting a huge error count to tracked TODOs — a valid bootstrap but with a caveat: it generates lots of `any`, so measure and burn down.

**Step 5 — Ratchets in CI.** Block new `.js` files; track `any` count and `@ts-expect-error` count and fail the build if they increase; measure % of files in TS. Visible progress sustains momentum.

**Trade-offs.** Pros: catches whole bug classes (null access, wrong prop shapes), safe refactors, self-documenting code, better IDE. Cons: learning curve; type-gymnastics for dynamic code; build/CI time; false sense of safety (types erase at runtime — API data isn't validated unless you validate); `any` creep. I'd explicitly say types are not a substitute for tests or runtime validation at trust boundaries (e.g., zod for external input).

**What to avoid.** Over-typing early (elaborate generics nobody understands), blanket `any` everywhere, and aiming for "100% strict immediately." Also avoid mandating a big-bang freeze.

## Solution

### Initial config

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "allowJs": true,
    "checkJs": false,
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": false,
    "noImplicitAny": false,
    "skipLibCheck": true,
    "incremental": true,
    "moduleResolution": "bundler",
    "baseUrl": "."
  },
  "include": ["src"]
}
```

```jsonc
// package.json
"scripts": { "typecheck": "tsc --noEmit" }
```

### Strict where it's new

```jsonc
// src/features/billing/tsconfig.json — migrated area is strict
{ "extends": "../../../tsconfig.json",
  "compilerOptions": { "strict": true, "composite": true },
  "include": ["."] }
```

### Types at the boundary (highest ROI)

```ts
// generated: npx openapi-typescript api.yaml -o src/api/schema.d.ts
import type { paths } from './schema';
type User = paths['/users/{id}']['get']['responses']['200']['content']['application/json'];

export async function getUser(id: string): Promise<User> { /* ... */ }
```

### Runtime validation at trust boundaries

```ts
const UserSchema = z.object({ id: z.string(), email: z.string().email() });
export const parseUser = (data: unknown) => UserSchema.parse(data);   // types ≠ runtime safety
```

### Ratchet scripts in CI

```bash
# fail if new .js files appear under src/
git diff --name-status origin/main | grep '^A.*\.jsx\?$' && exit 1

# fail if 'any' count increased relative to main
node scripts/count-any.js > current.txt && diff-against-main
```

### Convenient escape hatch (tracked)

```ts
// @ts-expect-error TODO(TS-412): legacy shape, fix when billing is migrated
legacyFn(arg);
```

`@ts-expect-error` over `@ts-ignore`: it errors when the suppression becomes unnecessary, so debt self-reports.

> **Check yourself:** Why enable `strictNullChecks` before the rest of `strict`, and why prefer `@ts-expect-error` to `@ts-ignore`?

## Gotchas

- **Turning on `strict` repo-wide first.** Thousands of errors, team revolt, abandonment.
- **`any` creep.** Codemods and rushed conversions produce `any`; without ratchets the migration is cosmetic.
- **Types as runtime guarantees.** External data can still violate types; validate at boundaries.
- **Typecheck in the critical build path.** Slows CI; use `tsc --noEmit --incremental`, project references, or a separate parallel job; let esbuild/Babel transpile.
- **Over-clever types.** Deeply conditional generics nobody can maintain hurt more than they help.
- **Mandating without support.** Provide training, pairing, a style guide, and fast feedback; otherwise resentment.
- **Declaring `.d.ts` shims for everything.** Untyped dependency → `declare module` any-shim is fine initially, but track and replace.
- **Mixing `@ts-ignore` liberally.** Hides real errors forever.

## Follow-up Questions

**Q (High): How do you migrate incrementally without stopping feature work?**

Answer: Enable `allowJs` so JS and TS coexist, typecheck separately from the build, and convert files opportunistically when touched plus targeted pushes on shared modules. Keep strictness loose at first and ratchet per-directory/flag; block *new* JS and growth of `any` in CI so progress only moves forward.

The trap: proposing a feature freeze or a single mass-conversion PR.

**Q (High): Where do you start for the best payoff?**

Answer: Boundaries and shared foundations: generated API types, shared utilities, design system components, and store shapes. Types there flow into callers through inference, so you get coverage in untyped files too. Leaf feature code last.

The trap: starting with random leaf files or alphabetical order.

**Q (Medium): Types disappear at runtime — how do you stay safe with external data?**

Answer: Treat types as compile-time contracts only. Validate untrusted input (API responses, URL params, localStorage, postMessage) with a schema library (zod/valibot), deriving the TS type from the schema so there's one source of truth. Generated API types alone don't prove the server actually obeyed them.

The trap: assuming `as User` makes data safe.

**Q (Medium): How do you handle teammates who resist?**

Answer: Show value with a concrete incident class types would have caught, keep the on-ramp gentle (loose settings, pairing, templates, good editor setup), involve skeptics in setting conventions, avoid type-golf, and measure outcomes (bugs, refactor speed). Make the cost low before demanding the habit.

The trap: "It's mandatory" as the entire plan.

**Q (Low): Build speed concerns on a huge codebase?**

Answer: Split transpilation from checking (Babel/esbuild/swc transpile; `tsc --noEmit` in parallel CI), use `incremental`/project references, cache in CI, and keep editor language-server performance in mind with `skipLibCheck` and scoped includes.

The trap: letting `tsc` block every local build and then blaming TypeScript.

## Self-Assessment

- [ ] Can state the incremental config (`allowJs`, `noEmit`) and why
- [ ] Can describe a strictness ratchet (flags and per-directory)
- [ ] Can justify typing the API boundary first
- [ ] Can explain `@ts-expect-error` vs. `@ts-ignore`
- [ ] Can explain why runtime validation still matters
- [ ] Can list CI ratchets that keep the migration moving forward

---
*Next: Monolith Frontend to Micro-frontends — When and How — asks the opposite judgment question: when to split things apart, and whether you should at all.*
