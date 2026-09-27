# Design a Schema-driven Form Builder

## Quick Reference

| Decision Point | Choice | Why |
|---|---|---|
| Field rendering | A field-type registry (mirroring the widget registry pattern from the Configurable Dashboard scenario) mapping schema field types to components, driven entirely by data, not hardcoded per-form JSX | The entire point of "schema-driven" is that adding a new form, or changing an existing one's fields, is a data change (editing the schema), not a code change — a registry is what makes new field *types* addable without touching the rendering engine itself |
| Validation | Validation rules declared in the schema (required, pattern, min/max, cross-field rules) and interpreted by a generic validation engine, not per-form hand-written validation logic | Hand-written per-form validation reintroduces exactly the "changing a form requires a code change" problem the schema-driven approach exists to avoid |
| Conditional fields (show/hide based on another field's value) | Declarative `condition` expressions in the schema, evaluated against current form values by a generic engine, re-evaluated on every relevant value change | Conditional visibility is one of the most common real-world form requirements and must be expressible in data (the schema) for the same reason validation must be — otherwise every form needing conditional logic falls back to hardcoded per-form code |
| Form state management | A single normalized values/errors/touched state object keyed by field name, independent of how many or which fields the current schema defines | The state shape must not hardcode knowledge of specific field names — the whole engine has to work for an arbitrary, schema-defined field set, including one that changes at runtime |
| Schema versioning / evolution | Schemas are versioned data; a previously-submitted form's data should remain interpretable even after the schema it was built from changes | A form schema is a living document — fields get added, renamed, removed over a product's lifetime — and historical submissions need to stay readable/auditable against the schema version they were actually submitted under |

## The Scenario

"Design a form-building system where forms are defined by a schema (JSON or similar) rather than hand-coded — think something like Typeform, Google Forms' backend, or an internal tool for building intake/survey forms without writing new code for each one. A non-engineer should be able to define a new form's fields, validation, and layout through configuration, and the renderer needs to turn that schema into a fully working, validated form. Walk me through the architecture."

## Clarifying Questions

- **Who authors the schema — is there a visual form-builder UI producing the schema as an output, or is the schema hand-written/edited directly (e.g., by a product manager editing JSON, or via a CMS-like admin tool)?** This matters less for the rendering engine itself (which only needs to consume a valid schema regardless of how it was produced) but significantly affects whether schema *validation* (catching an authoring mistake — a malformed schema — before it ever reaches an end user) needs to be a first-class, immediate-feedback concern in an authoring UI, versus something that can fail more quietly for a directly-hand-edited schema.
- **What range of field types needs to be supported, and is that range fixed or expected to grow over time?** Directly determines whether a simple, closed switch-statement-style field renderer suffices or whether the registry-based extensibility pattern (allowing new field types to be added without modifying the core rendering engine) is actually load-bearing — the same consideration as the Configurable Dashboard's widget registry.
- **How complex does conditional logic need to be — simple single-field show/hide ("show field B only if field A equals X"), or arbitrary multi-field boolean expressions ("show field C if (A equals X AND B is not empty) OR D is greater than 10")?** Single-condition show/hide can be expressed with a small, simple declarative shape; arbitrary boolean expression trees require either a more general expression-evaluation mechanism or a deliberately constrained expression language, which is a materially bigger design surface, including safety considerations if the expressions are ever evaluated as arbitrary executable code.
- **Does the schema also define layout (field ordering, grouping into steps/pages, columns), or only fields/validation, with layout handled separately (e.g., a fixed single-column layout regardless of schema content)?** A multi-step/paginated form driven by schema is a meaningfully bigger scope (needing to track "which step is active," per-step validation gating advancement) than a single flat list of fields in one column.
- **What happens to previously-submitted form responses when the schema they were submitted against later changes (a field renamed, removed, or its validation tightened)?** This is an easy-to-overlook but real concern for a schema-driven system that's expected to evolve over time — the design needs an explicit answer for whether/how historical submissions remain interpretable against a schema version different from the current one.
- **Does validation need to run purely client-side for immediate feedback, or does the same schema-derived validation logic also need to run server-side (re-validating on submission, independent of client-side checks)?** A trustworthy system re-validates server-side regardless of client-side validation (never trusting client-side checks as the sole gate) — worth confirming this is in scope, since it affects whether the validation-rule interpretation logic needs to be implementable identically on both client and server (or shared as common code) rather than existing only in the browser.

## Approach & Trade-offs

**The core architectural move is a field-type registry, directly analogous to the widget registry from the Configurable Dashboard scenario, and recognizing that structural similarity is itself a useful thing to name out loud.** Just as that dashboard's shell knows nothing about any specific widget's internals — only a common contract every registered widget type implements — this form renderer's engine should know nothing about any specific *field type's* internals, only a common contract (a schema field's `type` maps to a registered component that knows how to render itself, given its own config, and how to validate its own value against its own declared rules). Adding a new field type (say, a file-upload field, or a rich rating-scale input) becomes an isolated registration, not a change to the core rendering/validation engine — which is the property that actually makes a system "schema-driven" in a meaningful sense, rather than merely "schema-configured" for a fixed, closed set of field types that still require engine code changes to extend.

**Validation rules must be interpretable data, not per-form hand-written imperative code, or the system silently reintroduces the exact problem it exists to eliminate.** A schema's field definition declares its validation constraints (`required: true`, `pattern: '^[0-9]{5}$'`, `min: 0`, `max: 100`, or a cross-field rule referencing another field's value) as data the generic validation engine interprets uniformly across every field, regardless of form — rather than, for instance, a form-specific validation function hand-written in code for each new form as it's created. The moment adding or changing a form's validation requires writing or editing code (rather than editing schema data), the system has stopped being genuinely schema-driven for that concern, even if field rendering itself remains schema-driven — validation is just as central a piece of "form behavior" as field rendering, and needs the identical data-driven treatment.

**Conditional field visibility needs a declarative condition representation evaluated by a generic engine against live form values — and the complexity ceiling of that representation is a real, scenario-defining trade-off.** A simple, common case ("show field B only when field A equals a specific value") can be expressed compactly and safely as a small, constrained declarative shape (e.g., `{ field: 'A', operator: 'equals', value: 'X' }`) that a generic interpreter evaluates by looking up the referenced field's current value and applying the named operator — this is both simple to implement and inherently safe (no arbitrary code execution risk, since the "language" is a small, closed set of operators the engine explicitly supports). A materially more ambitious requirement (arbitrary boolean expression trees combining many fields with AND/OR/NOT) either extends this same declarative shape into a small recursive expression tree (still safe, still a closed, well-defined interpreter) or, if truly unbounded flexibility is required, could be tempted toward evaluating author-supplied expression strings as actual executable code (e.g., a raw JavaScript expression stored in the schema and run via something like `new Function()` or `eval`) — which introduces a genuine code-injection risk if schema authorship isn't a fully trusted, controlled process, and should be treated as a serious security trade-off to flag explicitly rather than a convenient implementation shortcut.

**Form state must be a generic, field-name-keyed structure with zero hardcoded knowledge of any specific field, because the whole point is supporting an arbitrary, schema-defined field set that can differ completely between forms and can itself change at runtime (conditional fields appearing/disappearing).** The values, validation-error, and touched/dirty state for a rendered form should each be a plain keyed object (`Record<fieldName, value>`, `Record<fieldName, errorMessage | null>`, etc.) populated dynamically by iterating the schema's field list — never a state shape with named, hardcoded properties per field (`{ email: string, age: number, ... }`), since that would require the state management code itself to change for every new schema, defeating the purpose. When a conditionally-hidden field becomes hidden again after being filled in, the engine needs an explicit, deliberate decision about whether to retain its now-irrelevant value in state (in case the condition becomes true again later and the user's prior input should reappear) or clear it (avoiding submission of a value for a field the user can no longer even see) — both are defensible, product-dependent choices, but the decision should be explicit rather than accidental.

**Schema versioning is what makes a schema-driven system durable over its actual product lifetime, not just at a single point in time — this is easy to underweight in initial design and a common shortcoming in real implementations.** Because the entire premise is that schemas change over time (new fields added, old ones removed or renamed, validation rules tightened), every form submission needs to be stored alongside a reference to the exact schema version it was submitted under, not merely "the current schema" — reviewing a submission from six months ago against today's schema (which may have since removed or renamed fields that submission's data still references) would otherwise misrender or misinterpret historical data. This pushes the design toward treating schemas themselves as versioned, immutable-once-published records (a new edit to a form produces a new schema version, rather than mutating the existing one in place), directly analogous to why immutable, addressable versions matter in other systems this repo touches on.

## Solution

**Field-type registry — the core extensibility mechanism, structurally mirroring the Configurable Dashboard's widget registry:**

```tsx
interface FieldDefinition<TConfig = unknown, TValue = unknown> {
  type: string;
  Component: React.ComponentType<{ config: TConfig; value: TValue; onChange: (v: TValue) => void; error?: string }>;
  validate?: (value: TValue, config: TConfig) => string | null; // returns an error message, or null if valid
}

const fieldRegistry = new Map<string, FieldDefinition<any, any>>();
function registerField(def: FieldDefinition<any, any>) { fieldRegistry.set(def.type, def); }

registerField({
  type: 'text',
  Component: TextFieldInput,
  validate: (value: string, config: { required?: boolean; pattern?: string }) => {
    if (config.required && !value?.trim()) return 'This field is required.';
    if (config.pattern && value && !new RegExp(config.pattern).test(value)) return 'Invalid format.';
    return null;
  },
});
```

**Schema shape — data describing fields, validation, and conditional visibility:**

```ts
interface FieldSchema {
  name: string;
  type: string;          // key into fieldRegistry
  label: string;
  config: unknown;       // field-type-specific, opaque to the engine
  condition?: FieldCondition; // when to show this field at all
}

interface FieldCondition {
  field: string;          // the OTHER field this condition depends on
  operator: 'equals' | 'notEquals' | 'contains' | 'greaterThan';
  value: unknown;
}

interface FormSchema {
  schemaId: string;
  version: number;        // immutable once published — a new edit produces a new version
  fields: FieldSchema[];
}
```

**The generic rendering + conditional-visibility engine — zero hardcoded field knowledge:**

```tsx
function SchemaForm({ schema }: { schema: FormSchema }) {
  const { values, errors, setValue, validateAll } = useFormEngine(schema);

  function isFieldVisible(field: FieldSchema): boolean {
    if (!field.condition) return true;
    const { field: depField, operator, value: expected } = field.condition;
    const actual = values[depField];
    switch (operator) {
      case 'equals': return actual === expected;
      case 'notEquals': return actual !== expected;
      case 'contains': return Array.isArray(actual) && actual.includes(expected);
      case 'greaterThan': return typeof actual === 'number' && actual > (expected as number);
    }
  }

  return (
    <form onSubmit={(e) => { e.preventDefault(); if (validateAll()) submitForm(schema.schemaId, schema.version, values); }}>
      {schema.fields.filter(isFieldVisible).map((field) => {
        const def = fieldRegistry.get(field.type);
        if (!def) return <UnknownFieldTypePlaceholder key={field.name} type={field.type} />;
        return (
          <def.Component
            key={field.name}
            config={field.config}
            value={values[field.name]}
            onChange={(v) => setValue(field.name, v)}
            error={errors[field.name]}
          />
        );
      })}
      <button type="submit">Submit</button>
    </form>
  );
}
```

**Generic, field-name-keyed form state — no hardcoded field names anywhere:**

```ts
function useFormEngine(schema: FormSchema) {
  const [values, setValues] = useState<Record<string, unknown>>({});
  const [errors, setErrors] = useState<Record<string, string | null>>({});

  function setValue(fieldName: string, value: unknown) {
    setValues((prev) => ({ ...prev, [fieldName]: value }));
  }

  function validateAll(): boolean {
    const nextErrors: Record<string, string | null> = {};
    for (const field of schema.fields) {
      const def = fieldRegistry.get(field.type);
      nextErrors[field.name] = def?.validate?.(values[field.name], field.config) ?? null;
    }
    setErrors(nextErrors);
    return Object.values(nextErrors).every((e) => e === null);
  }

  return { values, errors, setValue, validateAll };
}
```

> **Check yourself:** Without looking above, explain why `values` and `errors` are generic `Record<string, unknown>` structures rather than typed objects with named field properties, and describe what would have to change about this engine if they weren't.

## Server-side Re-validation & Schema Versioning

```ts
// Conceptual server-side sketch: the SAME validation rule interpretation must run here too — never trust client-side checks alone.
async function handleFormSubmission(schemaId: string, version: number, values: Record<string, unknown>) {
  const schema = await getSchemaVersion(schemaId, version); // the EXACT version this submission was rendered from
  for (const field of schema.fields) {
    const def = fieldValidators[field.type]; // server-side equivalent of the client's fieldRegistry validators
    const error = def?.validate?.(values[field.name], field.config);
    if (error) throw new ValidationError(field.name, error);
  }
  await storeSubmission(schemaId, version, values); // stored WITH the schema version it was submitted under
}
```

Storing `version` alongside every submission means a submission from an older schema version remains fully interpretable later even after the form's schema has since evolved — reviewing it re-fetches and renders against *that* version's field definitions, not whatever the schema currently looks like.

## Gotchas

**Hardcoding per-form validation logic in code rather than expressing it as schema data interpreted by a generic engine.** Reintroduces exactly the "changing a form requires a code change" problem the entire schema-driven approach exists to eliminate — validation must be just as data-driven as field rendering.

**Evaluating author-supplied conditional-logic expressions as arbitrary executable code (`eval`/`new Function()`) rather than through a small, closed, safe declarative operator set.** A genuine code-injection risk if schema authorship isn't a fully trusted, tightly controlled process — a constrained declarative expression language, interpreted by a fixed, known-safe engine, avoids this risk entirely for the vast majority of real conditional-logic needs.

**Hardcoding form state shape with named per-field properties instead of a generic, field-name-keyed structure.** Breaks the moment the schema changes or a different schema needs to be rendered by the same engine — the whole design depends on the engine having zero baked-in knowledge of any specific field.

**No server-side re-validation, trusting client-side schema-derived validation as the sole gate.** A client can always be bypassed (a direct API call skipping the rendered form entirely) — the same validation-rule interpretation must run authoritatively server-side regardless of what client-side checks already passed.

**Mutating a schema in place rather than versioning it, and storing submissions without a reference to which schema version they were submitted under.** Makes historical submissions unreliably interpretable once the schema changes — a renamed or removed field silently breaks how an old submission's data is understood or displayed.

**No explicit decision about what happens to a conditionally-hidden field's already-entered value.** Silently either always clearing it (losing a user's input if the condition flips back) or always retaining it (risking submission of a value for a field the user can no longer see and may not have intended to keep) without the team ever having made this an explicit, considered product decision.

## Follow-up Questions

**Q (High): Why should conditional-visibility logic be expressed as a constrained declarative operator set rather than as raw, author-supplied executable expressions, even though the latter would support more arbitrary logic?**

Answer: A raw executable expression (an author-typed JavaScript string evaluated via `eval` or `new Function()`) genuinely does support more arbitrary logic than a fixed, closed operator set — but it also means anyone able to author or edit a schema effectively gains the ability to run arbitrary code in the context of whatever renders that schema, which is a serious injection risk unless schema authorship is restricted to a fully trusted, tightly access-controlled process (and even then, an accidental authoring mistake — a typo producing an infinite loop or an unintended side effect in an expression — is a much scarier failure mode for executable code than for declarative data, which can at worst be malformed and simply fail to evaluate meaningfully). A small, closed set of well-defined operators (`equals`, `notEquals`, `contains`, `greaterThan`, and boolean combinators like `and`/`or` if needed) covers the substantial majority of real-world conditional-form-logic needs, is trivially safe to evaluate (there's no code being run, only data being interpreted by fixed, known logic), and is easy to validate/lint at authoring time in a way arbitrary code cannot be.

The trap: dismissing the executable-expression approach as simply "more powerful, so better" without weighing the security and authoring-safety cost against how rarely genuinely unbounded expressive power is actually needed in practice — recognizing that a closed declarative language covers the realistic requirement while remaining safe is the stronger, more senior answer than reflexively reaching for maximum flexibility.

---

**Q (High): A schema is edited to remove a field that several existing (already-submitted) form responses have data for. What should happen to those historical submissions, and how does your design prevent this from being a problem?**

Answer: Because every submission is stored alongside the exact schema version it was submitted under (not merely "the current schema" as an implicit, mutable reference), removing a field from a *new* schema version has no effect on how a previously-submitted response is interpreted — reviewing that older submission re-fetches and renders it against its own stored version's field definitions, which still include the now-removed field, so its data remains fully interpretable and displayable exactly as it was at submission time. This is precisely why schemas need to be treated as versioned, immutable-once-published records rather than mutated in place — a schema "edit" is really "publish a new version," leaving every previously-referenced version (and every submission tied to it) permanently intact and interpretable.

The trap: describing schema edits as in-place mutations of a single schema document, with submissions referencing only "the form" (not a specific version) — this loses the ability to correctly interpret historical data the moment any edit removes, renames, or changes the meaning of a field that older submissions' data still depends on.

---

**Q (High): Should the exact same field-registry validation functions run on both client and server, or is it acceptable for them to be separately (re-)implemented in each environment?**

Answer: Ideally the identical validation logic runs in both places, implemented once as shared code (e.g., a package/module importable by both the client bundle and the server, if the stack allows sharing code between them) — separately reimplementing the same rules in two places is a real, ongoing maintenance risk, since any future change to a validation rule (tightening a pattern, adding a new operator) now requires remembering to update both implementations identically, and any drift between them produces the confusing failure mode of a submission that passed client-side validation but is then rejected server-side (or, worse, one that's incorrectly accepted server-side despite the schema's stated rules, if the server's reimplementation is looser than the client's). Where genuinely sharing code isn't practical (e.g., a client in TypeScript/JavaScript and a server in an entirely different language), the two implementations should at minimum be driven by the exact same schema-interpretation logic conceptually and covered by shared test fixtures asserting both produce identical validation results for the same inputs — the goal is that "what counts as valid" is a property of the schema and its declared rules, not of which environment happens to be checking it.

The trap: treating client-side validation as sufficient on its own with server-side validation as a lower-effort afterthought ("just check `required` fields aren't empty, roughly") — the server-side check needs to be a faithful, complete re-implementation of the same declared rules, not a looser approximation, since it's the only validation a client can't simply bypass.

---

**Q (Medium): How would you support a multi-step (paginated) form driven entirely by schema, where advancing to the next step is gated on the current step's fields being valid?**

Answer: The schema needs an additional structural layer grouping fields into named steps (e.g., `steps: [{ stepId, fields: [...] }, ...]` alongside or instead of a flat `fields` array), and the rendering engine tracks which step is currently active as its own piece of state, independent of the field-values/errors state already described. Advancing to the next step re-uses the exact same `validateAll`-style logic already built, scoped to only the current step's field subset (validating only the fields belonging to the active step, not the entire form's fields at once) — gating the "next" action on that subset passing validation, while the overall submit action at the final step still validates the complete form across all steps' fields (catching, for instance, a conditionally-revealed field on an earlier step that became invalid due to a later step's input, if such cross-step conditions are supported). This extends the existing engine's structure rather than requiring a fundamentally different one — steps are simply a grouping/gating concern layered on top of the same generic, schema-driven field rendering and validation already in place.

The trap: designing step-gating as an entirely separate, parallel validation mechanism from the field-level validation engine already built — the stronger answer reuses the same per-field validation logic, merely scoping which fields' validation is checked before allowing step advancement, rather than inventing a second validation pathway.

---

**Q (Medium): What should the engine do when it encounters a field `type` in the schema that isn't registered (e.g., a schema authored for a newer version of the system, referencing a field type this particular client build doesn't yet know about)?**

Answer: The rendering engine's lookup against the field registry must handle a miss explicitly (as sketched in the `SchemaForm` solution above via `UnknownFieldTypePlaceholder`) rather than crashing or silently omitting the field — rendering a clear, visible placeholder indicating that field type isn't supported by the current renderer build is both more honest to whoever's filling out the form (rather than a silently incomplete form with no indication a field is missing) and more debuggable for whoever's maintaining the system (a clear signal exactly which field type needs registering, rather than a vague "the form seems to be missing something" report). This is structurally the identical concern as the Configurable Dashboard scenario's handling of a deprecated/unrecognized widget type — the same defensive pattern applies to any registry-based, schema/config-driven rendering system.

The trap: treating an unrecognized field type as something that "shouldn't happen" and therefore not worth explicit handling — in a system where the schema and the rendering engine can genuinely be deployed/versioned independently (a very plausible situation — schemas might be authored or updated more frequently than the client application itself ships), an unrecognized type is a realistic, not merely theoretical, occurrence.

---

**Q (Low): How would you let a non-engineer preview exactly how a schema they're authoring will render, before publishing it live?**

Answer: Because the rendering engine is already fully driven by schema data with no code changes required per form, a live preview is simply rendering the in-progress (unpublished) schema draft through the exact same `SchemaForm` component used for a real, published form — feeding it the author's current in-progress edits as its `schema` prop, live-updating as they make changes, rather than maintaining any separate preview-specific rendering logic. This is the same benefit already discussed for the Configurable Dashboard's live widget-gallery preview — a genuinely data-driven rendering engine gets an accurate, always-in-sync preview essentially "for free," since preview and production rendering are the same code path operating on different (draft vs. published) schema data.

The trap: building a separate, simplified preview renderer distinct from the actual production rendering engine — this risks the preview drifting out of sync with how the form will actually render/behave once published, and is unnecessary extra work given the production engine can already render any valid schema, published or not.

---

## Self-Assessment

- [ ] Can design a field-type registry and explain its structural parallel to the Configurable Dashboard's widget registry
- [ ] Can explain why validation rules must be schema data interpreted generically, not per-form hand-written code
- [ ] Can design a safe, closed declarative conditional-visibility operator set and articulate why raw executable expressions are a real security risk by comparison
- [ ] Can design generic, field-name-keyed form state with no hardcoded knowledge of any specific field
- [ ] Can explain why schemas must be versioned and submissions tied to a specific version, with a concrete scenario showing what breaks without this
- [ ] Can explain why server-side re-validation must faithfully mirror client-side rules rather than being a looser afterthought

---
*Next: Design a Component Library / Design System From Scratch — the final scenario in this phase, and a different kind of system-design problem than any prior one in this phase: rather than a single product feature, it's the shared foundation many features are built on, where the central new concerns become API design for reusability, theming/tokens, versioning without breaking consumers, and accessibility built in by default rather than bolted on per-consumer.*
