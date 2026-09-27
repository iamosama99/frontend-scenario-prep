# Conflict Resolution for Concurrent Edits

## Quick Reference

| Strategy | Mechanism | Best For | Weakness |
|---|---|---|---|
| Last-write-wins (LWW) | Whoever saves last silently overwrites | Low-stakes, rarely-concurrent fields | Silently loses the other person's work with no warning |
| Optimistic locking (version/etag check) | Reject a write if the resource's version changed since it was loaded | Detecting conflict, letting a human resolve it | Doesn't resolve the conflict itself — needs a UX for what happens next |
| Field-level merge | Only reject/merge on the specific fields that actually changed | Forms with many independent fields (most business CRUD) | Requires knowing which fields are safe to merge automatically |
| Operational Transform / CRDTs | Mathematically mergeable operations, no data loss | Real-time collaborative text/rich editing (Google Docs-style) | Significant implementation complexity; usually a library, not hand-rolled |

## The Scenario

"Two support agents both open the same customer record and start editing it — one updates the phone number, the other updates the shipping address, at roughly the same time. What happens when they both hit save? Now make it harder: what if they'd both edited the *same* field — say, both changed the customer's email to different values? Design the conflict handling for both cases."

## Clarifying Questions

- **Is this record edited concurrently often enough to be a real, expected occurrence, or is it a rare edge case that mostly just needs to not silently corrupt data when it happens?** This changes the bar for the solution — a frequently-concurrent resource (a shared task board, a live document) justifies investing in a smoother merge/live-sync experience, while a rarely-concurrent one (most individual customer records, most of the time) can reasonably get a simpler "detect and ask a human" treatment without it feeling like a gap.
- **Does the backend currently support any form of conflict detection at all — a version number, an `updatedAt` timestamp, an ETag — or would that need to be added?** Optimistic locking requires the server to actually reject a stale write, which needs *some* server-side versioning mechanism; if none exists, that's a real, non-trivial piece of backend work needed before any client-side conflict UX can be built on top of it, not just a client-side design problem.
- **For the same-field conflict (both changed email) — is there a business rule for which value should "win" if the system had to pick automatically (e.g., most-recently-modified overall record, or a role-based priority), or does this always need explicit human resolution?** Some domains have a defensible automatic tiebreaker (e.g., "the record's primary owner's edit wins"); most don't, and pretending one exists to avoid building a resolution UI is a common shortcut that produces silent, confusing data loss.
- **What's the actual granularity of "the edit" — is the whole customer record saved as one PUT/PATCH on form submit, or does each field save independently/incrementally as it's changed (autosave-per-field)?** This fundamentally changes the shape of the conflict: whole-record-save conflicts need field-level diffing to even detect that the *particular* fields two people changed don't overlap; per-field autosave naturally scopes each conflict check to just that one field, which is simpler to reason about but has its own trade-offs (partial saves, more requests).
- **Should a detected conflict block the second save entirely until resolved, or should it save what it can (non-conflicting fields) and only flag the specifically-conflicting field(s)?** For the phone-number/shipping-address example, blocking the whole save because of a *different* field's concurrent edit would be needlessly heavy-handed if the two changes don't actually overlap — I'd want field-level granularity in the conflict check itself, not just record-level.

## Approach & Trade-offs

**The two examples in the scenario are deliberately different problems, and treating them identically is the main trap here: non-overlapping field edits (phone vs. address) should merge automatically with no conflict at all, while overlapping edits to the same field (both changed email) are a genuine conflict needing a decision.** A record-level "whoever saves last wins, full stop" approach gets the *first* case wrong by accident — if agent A's save happens to land after agent B's, and both submitted the full record, A's save can silently overwrite B's shipping-address change even though A never touched that field, purely because A's `PATCH` included B's already-stale copy of the address field. This is the single most important design point: **conflict detection and resolution need to operate at field granularity, not whole-record granularity**, specifically so unrelated concurrent edits never conflict with each other in the first place.

**Field-level granularity is achievable either via a `PATCH` that sends only the changed fields (not the whole record) plus per-field optimistic locking, or via a full-record `PUT` with a diff computed against the version the client originally loaded.** Sending only changed fields is the simpler and more directly correct approach: if agent A's client only ever sends `{ phone: '...' }` and agent B's only sends `{ shippingAddress: '...' }`, there's structurally no way for one to clobber the other's field, regardless of save order — no conflict-detection logic is even needed for the non-overlapping case, because the requests themselves don't overlap. This is a meaningfully better starting point than trying to detect-and-resolve overlapping writes after the fact with full-record saves, and it's worth stating as the first design choice, before getting into resolution strategy for genuine conflicts.

**For the case that's a genuine conflict — both changed the *same* field — the right default is optimistic locking with a version token, rejecting the second write and surfacing an explicit resolution UI, rather than silent last-write-wins or a guessed automatic merge.** Concretely: the server tracks a version (an incrementing integer, or simply comparing `updatedAt`) per record (or, more precisely, could be tracked per-field for maximal granularity, though a per-record version combined with field-level `PATCH` payloads covers the two examples given here without needing per-field versioning); when agent B's `PATCH` for `email` arrives referencing the version B loaded the record at, and the server's current version has already moved past that (because A's edit — to a *different* field — landed first), the server has to decide whether that's actually a conflict. If B's edit is to `email` and A's prior edit was to `phone`, no real conflict exists even though the version number moved — this is exactly why field-level change tracking on the server side (not just a single monotonic record version) is needed to correctly distinguish "the record changed, but not the field I'm touching" (no real conflict, safe to proceed) from "the record changed, specifically the field I'm touching" (real conflict).

**When a genuine same-field conflict is detected, the resolution UX should show both values and let the human decide — not attempt an automatic merge for a scalar field like email, where there's no sensible way to "merge" two different string values.** (Automatic merging is meaningful for structured/list-shaped data — e.g., two people adding different items to a shared list can genuinely merge both additions — but two different values for one scalar field have no meaningful middle ground; contrast with a rich-text document where operational transforms can merge concurrent character-level insertions without loss.) The conflict UI needs to show: what the current user was trying to save, what the other value now is (fetched fresh from the server), and who/when made that other change if available — then let the user choose "keep mine," "take theirs," or manually reconcile, and resubmit as a fresh, deliberate save rather than silently retrying the original stale write.

## Solution

**1. Change detection and payload shaping — only send what actually changed, and track the version the client last saw for the fields being changed:**

```tsx
function useEditCustomer(customerId: string) {
  const { data: customer } = useQuery({ queryKey: ['customer', customerId], queryFn: () => fetchCustomer(customerId) });
  const [draft, setDraft] = useState<Partial<Customer>>({});

  const changedFields = useMemo(() => {
    const changes: Partial<Customer> = {};
    for (const key of Object.keys(draft) as (keyof Customer)[]) {
      if (draft[key] !== customer?.[key]) changes[key] = draft[key];
    }
    return changes;
  }, [draft, customer]);

  async function save() {
    return updateCustomer(customerId, {
      fields: changedFields,
      baseVersion: customer?.version, // the version this client last saw, per field or per record
    });
  }

  return { customer, draft, setDraft, changedFields, save };
}
```

**2. Server-side conflict check (illustrative — the actual comparison logic lives server-side, but the client needs to understand and act on its shape):**

```
PATCH /customers/123
{
  "fields": { "email": "new@example.com" },
  "baseVersion": 7
}

// Server logic (conceptual):
// - current record version is 9 (moved because of an unrelated phone-number edit)
// - but check: was `email` specifically touched between version 7 and 9? No.
// -> no real conflict, apply the change, bump version to 10

// vs. a genuine conflict:
// - current record version is 9
// - `email` WAS changed between version 7 and 9 (by another agent)
// -> reject with 409, include the current server value for the conflicting field(s)
```

**3. Client handling of the 409 — surfacing an explicit resolution UI, not a silent retry:**

```tsx
async function handleSave() {
  const result = await save();

  if (result.status === 409) {
    setConflict({
      field: 'email',
      mine: draft.email,
      theirs: result.conflictingFields.email.currentValue,
      changedBy: result.conflictingFields.email.changedBy, // if available
    });
    return; // do not silently retry with a stale base version
  }
  // success path — clear draft, refetch, etc.
}
```

**4. The resolution UI itself:**

```tsx
function ConflictResolutionDialog({ conflict, onResolve }: { conflict: Conflict; onResolve: (value: string) => void }) {
  return (
    <Dialog>
      <p>Someone else changed {conflict.field} while you were editing.</p>
      <div>
        <button onClick={() => onResolve(conflict.mine)}>Keep my value: {conflict.mine}</button>
        <button onClick={() => onResolve(conflict.theirs)}>Use their value: {conflict.theirs}</button>
      </div>
      {/* optionally: an editable field pre-filled with `conflict.theirs`, letting the user manually reconcile into a third value */}
    </Dialog>
  );
}
```

> **Check yourself:** Why does the server need to check "was this *specific field* changed since the client's base version" rather than just "has the record's version changed at all" — walk through what goes wrong with the coarser check using the phone/address example.

## Gotchas

**Record-level (not field-level) optimistic locking, causing unrelated concurrent edits to falsely conflict.** If the server rejects any write where the record's version has moved at all, agent A editing phone and agent B editing address — genuinely non-overlapping changes — would still trigger a conflict for whichever saves second, which is a false positive that makes the system feel broken for the overwhelmingly common non-overlapping case.

**Silent last-write-wins with no version check at all.** The simplest to implement and the most dangerous — a second save with a stale full-record copy can silently revert or destroy the first save's changes with zero indication to either user that anything was lost.

**Retrying a rejected save by simply resubmitting the same stale data.** If a 409 response is handled by just calling the save function again without updating the base version or draft, it will fail identically every time — the client needs to actually fetch/merge the current server state before a retry can succeed.

**Attempting an automatic "merge" for a same-field scalar conflict (silently concatenating or picking one value with a heuristic) instead of asking a human.** Unlike a genuinely mergeable structure (a list two people added different items to), two different string values for `email` have no principled automatic resolution — guessing produces a result neither user actually intended.

**Not showing who/when made the conflicting change, when that information is available.** "Someone else changed this" without any context (which teammate, how recently) makes it harder for the user resolving the conflict to make an informed choice — showing attribution when available meaningfully improves the resolution UX with modest extra cost.

## Follow-up Questions

**Q (High): Why does record-level optimistic locking (a single version number for the whole record) produce false-positive conflicts, and how does field-level tracking fix it — walk through the phone/address example concretely with actual version numbers.**

Answer: Say the customer record starts at version 5. Agent A loads it (sees version 5), agent B loads it independently (also sees version 5). Agent A edits and saves `phone`, and the server accepts it, bumping the record to version 6. Agent B, still holding version 5 as their "base version," now saves `shippingAddress`. With record-level locking, the server compares B's base version (5) against the current version (6), sees a mismatch, and rejects it as a conflict — even though B's edit (address) has nothing to do with A's edit (phone) and there's no actual data-loss risk in applying both. With field-level tracking, the server instead asks a more precise question: "has the `shippingAddress` field specifically changed since version 5?" — and since only `phone` changed between 5 and 6, the answer is no, so B's save is accepted cleanly, bumping the version to 7, with both edits preserved. This is the concrete mechanism behind why field-granularity, not record-granularity, is the right unit for conflict detection here.

The trap: implementing "optimistic locking" correctly in the narrow textbook sense (comparing a version number) but at the wrong granularity for this use case — technically correct locking behavior that produces the wrong practical outcome (false conflicts on non-overlapping edits) for a multi-field record edited by multiple people.

---

**Q (High): For the same-field conflict (both changed email), why not just let the server pick a deterministic winner automatically — e.g., "the most recent write wins" — instead of building a whole resolution UI?**

Answer: "Most recent write wins" is exactly last-write-wins, and the entire reason it's unsatisfying here is that it silently discards one user's explicit, intentional edit with no indication to them that it happened — the losing agent walks away believing their change to the customer's email was saved, when it wasn't, and has no reason to go check. That's a worse failure mode than a conflict UI that costs the user one extra click, specifically because the alternative isn't "no conflict happens," it's "the conflict happens invisibly." A deterministic auto-resolution rule is defensible only when the business genuinely doesn't care which value wins for that field (rare) or when there's a clear priority signal that isn't arbitrary (e.g., an explicit "the record's assigned owner's edits take precedence over anyone else's" business rule, if such a rule actually exists and is deliberately chosen, not assumed) — and even then, I'd want the losing party to be notified their edit didn't stick, rather than staying silent about it.

The trap: presenting "pick a deterministic winner" as strictly simpler and therefore better — it's simpler to implement, but it changes the actual behavior from "flag and let a human decide" to "silently discard someone's real, intentional work," which is a much bigger behavioral trade-off than the implementation-complexity framing suggests.

---

**Q (High): The resolution dialog shows "their" current value by fetching it fresh from the server at the moment of conflict. What if, by the time the user picks "use their value" and resubmits, a *third* agent has changed the same field again in the meantime?**

Answer: The resubmission after conflict resolution should itself be treated as a brand-new save attempt, going through the exact same base-version check as any other write — so if a third edit landed in between, this resubmission would correctly conflict again, and the user would see a second resolution prompt with the newer current value. This isn't a special case needing separate handling; it falls out naturally as long as the resolution flow always re-fetches the current value and current version immediately before resubmitting (rather than reusing the version/value captured at the moment the *first* conflict was shown, which could itself now be stale) — the conflict-detection mechanism is inherently repeatable and doesn't assume a conflict can only happen once per edit session.

The trap: treating conflict resolution as a one-shot special path that, once handled, is assumed safe to resubmit unconditionally — the resubmission carries exactly the same staleness risk as the original save and needs the same version check applied to it, not an exemption.

---

**Q (Medium): How would this design change for a field that's a list (e.g., a customer's tags), where two agents concurrently add *different* tags?**

Answer: This is exactly the case where automatic merging is legitimate and preferable to a conflict prompt — if agent A adds tag "vip" and agent B adds tag "priority-support" concurrently, both additions are independently valid and there's no actual conflict of intent; the correct behavior is to merge both into the final tag list rather than rejecting one. This pushes the server-side representation toward an operation-based model for this field specifically ("add tag X" / "remove tag Y" as discrete operations, rather than "set the tags array to this snapshot") — two "add" operations targeting different values commute cleanly and can both apply, whereas two operations targeting the *same* value (both trying to remove the same tag, or one adding and one removing the same tag) would still need conflict handling, just with a much smaller and more specific surface area than a full record-level check. This is a good example of where the "field-level, not record-level" principle actually needs to go a level deeper for collection-shaped fields specifically — sub-field, operation-level granularity — rather than treating the whole tags array as one opaque scalar field for locking purposes.

The trap: treating every field uniformly as a scalar-overwrite conflict candidate — list/set-shaped fields often have real, low-risk automatic-merge opportunities that a one-size-fits-all "reject on version mismatch" policy would unnecessarily flag as conflicts.

---

**Q (Medium): Does adding field-level conflict detection meaningfully increase backend complexity — what would you need the server to actually track that a naive single-timestamp `updatedAt` approach doesn't give you?**

Answer: Yes, meaningfully — a single `updatedAt` timestamp per record only tells you *that* something changed and *when*, not *which field*, so it can't answer "was the specific field I'm trying to write actually touched since I last read it." Supporting the field-level check requires either per-field version/timestamp tracking (a `fieldVersions: { phone: 6, email: 4, shippingAddress: 7 }` map alongside the record, updated per field on write) or, more simply in many systems, an audit/change log the server can query ("has there been any write to `email` on this record since version 5") — the audit-log approach has a nice side benefit of also solving "show who changed it and when" for the resolution UI's attribution, for free, since that data needs to exist anyway for the check itself.

The trap: assuming field-level conflict detection is a purely client-side design choice with no backend cost — it's a real, non-trivial backend capability (per-field change tracking) that has to exist before the client-side flow described here is achievable, and it's worth flagging explicitly rather than presenting the whole design as client-only.

---

**Q (Low): If both agents are on the *same* team looking at the *same* customer record simultaneously as a matter of course (not a rare accident), would you reach for a different architecture entirely — closer to a live collaborative document than a form with occasional conflicts?**

Answer: Yes — if concurrent, simultaneous editing of the same record by multiple people is the expected common case rather than a rare accident, that's a strong signal to move toward a live-sync model (each keystroke/change broadcast and merged in near-real-time, similar to a collaborative document editor) rather than a "load, edit locally, save, maybe conflict" form-based model — at that point, the conflict-resolution-on-save design described here is solving the wrong problem, because the underlying assumption (edits happen in isolated sessions that occasionally overlap) no longer holds. That's a substantially larger investment (likely reaching for an existing CRDT or operational-transform library rather than hand-rolling it) and would only be justified by that actual usage pattern being real and common enough to matter, not by a single scenario description mentioning two agents happening to be in the same record at once.

The trap: over-architecting toward full live-collaborative-editing infrastructure for what's actually described as an occasional, not-the-common-case conflict — matching the solution's complexity to the actual frequency and stakes of concurrent editing, not to the theoretical maximum sophistication available.

---

## Self-Assessment

- [ ] Can explain why field-level (not record-level) conflict detection is the key design decision, with the phone/address example as concrete proof
- [ ] Can distinguish scalar-field conflicts (need human resolution) from list/collection-field conflicts (often auto-mergeable) and explain why
- [ ] Can describe what a resolution UI needs to show and why silent auto-resolution is usually the wrong default for scalar fields
- [ ] Can explain what backend capability (beyond a single `updatedAt`) field-level detection actually requires
- [ ] Can recognize when the whole form-based conflict-resolution model is the wrong fit and a live-collaborative architecture is warranted instead

---
*Next: Redux → Lightweight State — Migration Trade-offs — moves from resolving conflicts within a given architecture to deciding when an existing state-management architecture itself is the wrong fit and how to migrate off it safely.*
