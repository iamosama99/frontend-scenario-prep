# GraphQL Client-side N+1 / Over-fetching

## Quick Reference

| Problem | Mechanism | Fix |
|---|---|---|
| Client-side N+1 | A list renders N children, each independently querying for its own related data | Fetch the related data as part of the parent list query (nested selection), not per-child |
| Over-fetching | A query requests more fields than the view actually renders | Trim the query's field selection to exactly what's used; fragment colocation makes this auditable |
| Under-fetching (the over-correction) | Trimming too aggressively breaks a sibling/future consumer that needed a field | Colocate fragments per-component so each consumer declares its own needs explicitly |
| Server-side N+1 (resolver-level) | Each field resolver independently hits the database per parent item | DataLoader-style batching on the server — a client-side concern to be aware of, not fix from the client |
| Waterfall from query dependency | One query's result gates when the next query can be sent | Structure as one query with nested selections where possible, or fire independent queries in parallel |

## The Scenario

"We migrated part of the app to GraphQL specifically to avoid REST over-fetching. But someone profiled the new comments-list feature and found it's making 47 network requests to render one page — one query for the list of comments, then one more query per comment to fetch its author's avatar and name. That's worse than the REST version it replaced. Explain why this happened despite using GraphQL, and fix it — plus, walk through how you'd prevent this specific mistake from recurring as the team adds more GraphQL-backed lists."

## Clarifying Questions

- **Is each comment's author query issued from a separate component (e.g., a `CommentAuthor` component that each `Comment` renders, each independently calling `useQuery`), or is it one query somehow looping N times in a single component?** This changes where the fix needs to happen — a per-child-component query pattern is a component-composition/data-fetching-architecture issue (the natural place a client-side N+1 arises in a component-tree-shaped UI), whereas a single component issuing N queries in a loop is more likely a straightforward "this should be one query with a list variable" bug.
- **Does the GraphQL schema already support requesting an author as a nested field on `Comment` (e.g., `comment { id text author { name avatarUrl } }`), or does fetching an author require a separate top-level query (`author(id: $id)`) with no nested path from `Comment`?** If the schema already supports nesting, this is purely a client-side query-authoring bug — the fix is rewriting the query. If the schema genuinely doesn't expose author as a nested field on comment, the fix needs a schema change too, which is a larger, cross-team piece of work with its own trade-offs (who owns the schema, how quickly can it change).
- **Is the underlying server-side resolver for `author` (once nested) itself naive per-parent-item (would 20 comments' nested author fields each trigger a separate database query server-side, an N+1 one layer down), or does the server already batch this correctly (DataLoader-style)?** The client-side fix (one nested query) can still hit an N+1 at the resolver/database layer if the server isn't batching — worth knowing whether this needs a client fix only, or a client fix plus verifying/fixing server-side batching too.
- **How is data-fetching structured on the client — is there a shared GraphQL client (Apollo, urql, Relay) with normalized caching already in place, or is this closer to hand-rolled `fetch` calls sending raw GraphQL query strings?** A normalized-cache client changes both how the bug likely arose (fragment colocation patterns, cache-read behavior) and what tooling is available to prevent recurrence (Apollo's `useFragment`, Relay's compiler-enforced fragment composition).
- **Beyond this one feature, is there a broader pattern across the app of components independently declaring and firing their own queries for data a parent already fetched (or could easily fetch) as part of a larger query?** If this is systemic rather than isolated to comments, the durable fix is closer to establishing a team-wide convention/lint rule than patching this one list — worth understanding the scope before proposing a fix sized only for this feature.

## Approach & Trade-offs

**First, name precisely why GraphQL didn't prevent this on its own — the tool solves over-fetching *within a single query*, not query *composition* across a component tree, and conflating the two is exactly how this bug happened.** GraphQL's core promise (client specifies exactly the fields it needs, server returns exactly that, no more) genuinely eliminates the classic REST over-fetching problem *for a single request*. It says nothing about how many *separate requests* the client chooses to issue, and a component-tree-shaped UI (a `CommentList` rendering N `Comment` components, each of which renders a `CommentAuthor` that independently calls `useQuery` for that comment's author) will happily issue N+1 separate, individually-well-shaped GraphQL queries if that's how the components are composed — each query is efficient in isolation (no wasted fields), but the client is still making the same N+1 network-round-trip mistake a REST implementation could make, just with prettier individual requests. This is a case where a genuinely useful tool got applied without addressing the actual architectural cause of the original problem it was brought in to solve.

**The fix is restructuring the query shape to match the actual data shape being rendered — nested selections, not per-child top-level queries.** GraphQL's nested-selection syntax exists specifically to let a client ask for a list and its related data in one round trip: `comments { id text author { name avatarUrl } }` returns each comment *with* its author embedded, resolved server-side in one request/response cycle, rather than requiring N follow-up requests from the client. This only works if the schema supports the nesting (author reachable as a field on comment) — if it currently doesn't, that's a real, second piece of work (a schema change, coordinated with whoever owns the backend/schema) rather than something fixable purely client-side, and worth flagging as a distinct, potentially blocking dependency of the fix.

**Moving the query up to encompass what child components need is a real trade-off against component encapsulation — worth naming, not just applying silently.** Each `Comment` component independently owning its data-fetching (including its author) is, from a component-design perspective, appealingly self-contained — a `Comment` can be dropped anywhere and fetches everything it needs on its own, no parent coordination required. Restructuring to one parent-level query that requests everything every child needs breaks that self-containment: the parent (`CommentList`) now needs to know about and request data on behalf of its children's rendering needs, coupling the list query to the shape of whatever `Comment` currently renders. GraphQL's fragment colocation pattern is the mechanism that resolves this tension: each component declares its *own* data requirements as a colocated fragment (`Comment` exports a `CommentFragment` naming exactly the fields it needs, including nested author fields), and the parent composes those fragments into its query without needing to know their contents — restoring most of the self-containment benefit while still resulting in one physical network request.

**Preventing recurrence needs to be a structural/tooling answer, not a one-time fix plus a wiki note — the same shape of mistake (a child component independently firing its own query for data a parent could have included) is exactly the kind of thing that quietly reappears feature by feature without something enforcing the pattern.** A code-review checklist item ("did you check if this data can be included in a parent query instead of a new one?") helps but relies on every reviewer remembering to ask it, every time, for as long as the team exists — for a mistake specifically caused by "each component was individually reasonable, the aggregate wasn't," relying purely on human vigilance for every future addition has a stated failure history exactly one occurrence long already (this bug). A fragment-colocation convention (see Prevention section) paired with either GraphQL codegen/compiler tooling that surfaces composed-query shape, or a lightweight lint/review habit of checking Network tab request count on new list features, are more durable answers than a purely process-based fix.

## Solution

**Step 1 — the buggy version: each `Comment` independently queries its author.**

```tsx
function CommentList({ postId }: { postId: string }) {
  const { data } = useQuery(GET_COMMENTS, { variables: { postId } });
  return <>{data?.comments.map((c) => <Comment key={c.id} comment={c} />)}</>;
}

const GET_COMMENTS = gql`
  query GetComments($postId: ID!) {
    comments(postId: $postId) { id text authorId }
  }
`;

function Comment({ comment }: { comment: { id: string; text: string; authorId: string } }) {
  const { data } = useQuery(GET_AUTHOR, { variables: { id: comment.authorId } }); // one request PER comment
  return (
    <div>
      <CommentAuthorBadge name={data?.author.name} avatarUrl={data?.author.avatarUrl} />
      <p>{comment.text}</p>
    </div>
  );
}
```

47 requests for a page of comments: 1 for the list, 1 per comment for its author — the pattern QA found.

**Step 2 — nest the author field directly in the comments query, assuming the schema supports it:**

```tsx
const GET_COMMENTS = gql`
  query GetComments($postId: ID!) {
    comments(postId: $postId) {
      id
      text
      author { name avatarUrl } # resolved server-side, in the same request
    }
  }
`;

function CommentList({ postId }: { postId: string }) {
  const { data } = useQuery(GET_COMMENTS, { variables: { postId } });
  return <>{data?.comments.map((c) => <Comment key={c.id} comment={c} />)}</>;
}

function Comment({ comment }: { comment: CommentWithAuthor }) {
  return (
    <div>
      <CommentAuthorBadge name={comment.author.name} avatarUrl={comment.author.avatarUrl} />
      <p>{comment.text}</p>
    </div>
  );
}
```

One request total, regardless of comment count — the N+1 is gone because the data shape being requested now matches the data shape being rendered.

**Step 3 — restore component self-containment using fragment colocation, so `Comment`/`CommentAuthorBadge` still declare their own data needs rather than the parent having to know their internals:**

```tsx
// CommentAuthorBadge.tsx
const COMMENT_AUTHOR_BADGE_FRAGMENT = gql`
  fragment CommentAuthorBadgeFields on User {
    name
    avatarUrl
  }
`;
function CommentAuthorBadge({ author }: { author: FragmentType<typeof COMMENT_AUTHOR_BADGE_FRAGMENT> }) {
  const data = useFragment(COMMENT_AUTHOR_BADGE_FRAGMENT, author);
  return <span><img src={data.avatarUrl} /> {data.name}</span>;
}

// Comment.tsx
const COMMENT_FRAGMENT = gql`
  fragment CommentFields on Comment {
    id
    text
    author { ...CommentAuthorBadgeFields }
  }
  ${COMMENT_AUTHOR_BADGE_FRAGMENT}
`;
function Comment({ comment }: { comment: FragmentType<typeof COMMENT_FRAGMENT> }) {
  const data = useFragment(COMMENT_FRAGMENT, comment);
  return (
    <div>
      <CommentAuthorBadge author={data.author} />
      <p>{data.text}</p>
    </div>
  );
}

// CommentList.tsx — composes child fragments without needing to know their field-level contents
const GET_COMMENTS = gql`
  query GetComments($postId: ID!) {
    comments(postId: $postId) { ...CommentFields }
  }
  ${COMMENT_FRAGMENT}
`;
```

Each component still owns and declares exactly the fields *it* needs (true over-fetching prevention, at the component level, not just the top-level query) — but composition still results in one physical query, since Apollo (or Relay/urql equivalents) flattens fragment composition into a single request at query-execution time. Adding a new field to `CommentAuthorBadge`'s needs means editing its own fragment, not hunting down and editing a top-level query somewhere else in the tree.

**Step 4 — verify the server-side resolver for the now-nested `author` field batches correctly, since a naive per-parent resolver would reintroduce N+1 one layer down (server-side, invisible from the client's Network tab, but real database load):**

```ts
// Server-side resolver, conceptually — batched via DataLoader
const authorLoader = new DataLoader(async (userIds: readonly string[]) => {
  const users = await db.users.findMany({ where: { id: { in: userIds } } });
  return userIds.map((id) => users.find((u) => u.id === id));
});

const resolvers = {
  Comment: {
    author: (comment) => authorLoader.load(comment.authorId), // batches all calls within one tick into one query
  },
};
```

Without this, 20 comments' nested `author` fields would each independently call the resolver, producing 20 database queries within the single GraphQL request — an N+1 that moved from "visible in the browser's Network tab" (Step 1's version) to "invisible from the client, but present in database query logs" (an unbatched Step 2/3 resolver) — worth explicitly checking, not assumed away just because the client-side request count dropped to one.

> **Check yourself:** If `CommentAuthorBadge` later needs a new field (say, `author.isVerified`), which file(s) does that change touch under the fragment-colocation design in Step 3, and why doesn't it require editing `CommentList.tsx`?

## Preventing Recurrence

**Establish fragment colocation as the default pattern for any component that renders data from a GraphQL query, not an opt-in practiced only where someone remembers to.** The core discipline: a component that renders GraphQL-sourced data declares a fragment naming exactly the fields it uses, and never issues its own top-level `useQuery` for data a reasonably nearby ancestor could include in its existing query — any new "does this component need to fetch its own data" decision should default to "extend an ancestor's query via a fragment" and treat "fire an independent query" as the exception requiring justification (e.g., data genuinely unrelated to anything an ancestor renders, or a component reused in contexts with no relevant ancestor query at all).

**A lightweight review/verification habit: check Network tab request count on any new list-rendering feature before calling it done**, specifically looking for "does request count scale with list length" — a list of 5 items making 5 more requests than a list of 3 items is close to definitionally this bug, and it's a fast, concrete check that doesn't require deep GraphQL tooling knowledge to apply, making it a reasonable review-checklist item even for reviewers less familiar with the codebase's GraphQL layer specifically.

**Where available, lean on compiler/codegen tooling that makes composed-query shape and unused-field detection automatic rather than manual.** Relay's compiler enforces fragment colocation structurally (a component can only access fields it declared in its own fragment, a build-time-checked guarantee, not just a convention) and can flag fields fetched but never read; Apollo's codegen can generate types from fragments that make an unused or missing field a type error rather than a silent runtime gap. Adopting whichever of these the team's GraphQL client supports turns "remember the convention" into "the build enforces the convention," which scales far better across a growing team than relying on every future PR author and reviewer independently remembering this specific past incident.

## Gotchas

**Fixing the client-side N+1 (Step 2/3) and declaring victory without verifying the server-side resolver batches correctly** — the visible symptom (47 client requests) disappears, but if the `author` resolver is naive per-parent, the same N+1 pattern now exists one layer down as N database queries within a single GraphQL request, invisible from the browser's Network tab and easy to miss without checking server-side logs/query counts specifically.

**Over-correcting into under-fetching** — after being burned by over-fetching/N+1, a common overcorrection is aggressively trimming every query to the absolute observed-minimum fields, without fragment-level ownership per component; the next developer adding a feature that needs one more field on `author` either has to hunt down and edit a shared top-level query they don't own conceptually, or — worse — adds their own new independent query for just that one field, reintroducing exactly this bug in miniature.

**Treating fragment colocation as purely a client-side/component-organization concern and not checking whether the composed query, once assembled from many small fragments, still ends up reasonably sized** — fragment composition prevents *duplicate* requests, but it's still possible for a deeply nested component tree's composed fragments to add up to a genuinely large single query if many components each add a few fields; worth periodically checking actual composed-query size/shape for very deep or wide component trees, not assuming colocation is a free, unbounded-depth pattern.

**Assuming this specific fix (nesting `author` under `comments`) generalizes automatically to every future list+related-data pattern without checking the schema supports the equivalent nesting each time** — a future list feature might relate to something the current schema genuinely doesn't expose as a nested field yet, requiring the same schema-change conversation this fix needed for `author`, rather than assuming every future case is a pure client-side query-rewrite.

**Not distinguishing a genuinely list-scoped batch fetch from a case where nesting isn't actually appropriate** — e.g., if `author` data were extremely large or rarely actually rendered (most comments' authors never scrolled into view), always eagerly nesting it into the list query trades N+1 network requests for one larger, possibly wasteful single request; worth confirming the fix direction (nest everything vs. lazily fetch on demand for only what's visible) matches the actual rendering/virtualization behavior of the list, not applying "always nest" as a blind rule.

## Follow-up Questions

**Q (High): Explain precisely why GraphQL, despite its core value proposition being "no over-fetching," didn't prevent this specific bug. What exactly does GraphQL guarantee, and what does it not guarantee?**

Answer: GraphQL's over-fetching guarantee operates at the level of a single query's field selection — for any one request, the server returns exactly the fields named in that query's selection set, nothing more, which genuinely eliminates the classic REST problem of an endpoint returning a large, fixed response shape regardless of what the client actually needs. It says nothing about how many separate query *requests* a client chooses to make, or how a client structures its requests relative to how components in a UI are composed — a component tree where a parent renders N children, and each child independently issues its own (individually well-formed, non-over-fetching) `useQuery` call, results in N+1 total requests, each one efficient in isolation, but the aggregate is the same N+1-round-trip problem a REST client could produce. The bug in this scenario isn't that any individual query asked for too many fields — each one (`GetComments`, and each per-comment `GetAuthor`) is a perfectly minimal query — the bug is entirely about request *count* and *composition*, a dimension GraphQL's field-selection mechanism doesn't address at all; that dimension is addressed by how queries are structured (nested selections) and how components compose their data needs (fragment colocation), both client-architecture decisions GraphQL enables but doesn't enforce.

The trap: treating "we use GraphQL" as itself sufficient reasoning for why over-fetching/N+1 shouldn't be possible — GraphQL solves a specific, narrower problem (per-request field-selection efficiency) than "efficient data fetching" as a whole, and the request-count/composition dimension is a separate concern requiring separate, deliberate design (nesting, fragment colocation) that doesn't happen automatically just from adopting GraphQL.

---

**Q (High): What's the exact difference between fixing this by adding `author` as a nested field directly in the parent's top-level query, versus using fragment colocation — and why does the scenario's "prevent this from recurring" requirement favor the fragment approach specifically?**

Answer: Directly nesting `author { name avatarUrl }` in the parent's top-level query (Step 2) fixes the immediate N+1 — one request instead of N+1 — but leaves the parent (`CommentList`'s query) responsible for knowing and declaring exactly which author fields its descendant components (`Comment`, `CommentAuthorBadge`) actually render; every time a descendant's rendering needs change (a new field used, an old one dropped), someone has to remember to go edit the *parent's* query to match, which is exactly the kind of cross-file, easy-to-forget coordination that caused unrelated bugs in other contexts (analogous to the enumerated-invalidation-list gotcha in [[02-cache-invalidation-strategy]] — a list of things to keep in sync by hand tends to drift). Fragment colocation (Step 3) inverts this: each component declares its own fragment naming exactly the fields *it* needs, and the parent composes fragments without needing to know their contents — a `CommentAuthorBadge` needing a new field only requires editing `CommentAuthorBadge`'s own fragment; the composition automatically includes it in the final assembled query with no edit needed anywhere else in the tree. This directly serves the "prevent this from recurring" part of the prompt: the failure mode that caused the original bug (a component's data needs and the query structure serving it drifting out of sync, because keeping them in sync required a human to remember to edit the right file) is structurally addressed by colocation, not just patched for the current set of fields.

The trap: treating Step 2 (flat nesting in the parent query) as a complete answer to the full prompt — it fixes the *symptom* (47 requests) but not the *underlying maintainability property* the "prevent recurrence" part of the question is specifically asking about; a reviewer looking for the complete answer wants to see the fragment-colocation reasoning, not just the flattened query.

---

**Q (High): The client-side fix reduces the Network tab to one request. How would you verify whether the server-side resolver for the newly-nested `author` field is still doing an N+1 at the database layer, and why can't the client-side Network tab tell you this on its own?**

Answer: The browser's Network tab only shows requests the *client* makes — once the fix nests `author` inside a single GraphQL request, there's exactly one HTTP request/response visible regardless of what happens *inside* the server while resolving that request; if the `author` field's resolver naively queries the database once per parent `Comment` rather than batching, that's 20 separate database queries happening entirely server-side, inside the single request/response cycle the client observes as "one network call" — invisible from the client's perspective by construction, since the client only sees the outer request boundary, not what the server did to fulfill it. Verifying this requires server-side visibility: database query logs/count for a single GraphQL request handling a comments-with-authors query (a spike to 20+ queries for one logical request is the signature), an APM/tracing tool that shows resolver-level spans within a single GraphQL execution, or, more directly, checking whether the schema's resolver implementation for `Comment.author` uses a batching mechanism (DataLoader or equivalent) versus a naive per-call database lookup by reading the resolver code itself. This is a genuinely separate check from anything observable client-side, and skipping it (declaring victory once the client Network tab shows one request) is a common, easy-to-miss gap specifically because the client-visible symptom that originally revealed the bug (request count in the Network tab) has fully resolved, removing the obvious signal that anything might still be wrong.

The trap: treating "Network tab shows one request now" as proof the N+1 problem is fully resolved — it proves the *client-side* N+1 (request count) is fixed, but says nothing about a resolver-level N+1 happening server-side within that one request, which needs its own, separate verification via server-side tooling, not inferred from client-observable behavior.

---

**Q (Medium): Someone on the team proposes: "let's just always fetch every field on every type, on every query, so we never under-fetch again." What's wrong with this as a response to having just been burned by an N+1/over-fetching incident?**

Answer: This reintroduces REST-style over-fetching deliberately, as an overcorrection — it guarantees every query returns the maximum possible payload regardless of what any given view actually renders, which is precisely the waste GraphQL was adopted to eliminate in the first place, and at any reasonable schema size/field count, this materially increases response payload size and server-side computation (resolving and serializing fields nobody asked for) for every single query in the app, permanently, as a blanket policy rather than a targeted fix. It also doesn't actually address the *lesson* from this incident — the bug wasn't caused by a query having too few fields per se, it was caused by a mismatch between how components were composed (each independently fetching) and how the query was structured (not nested to match); "fetch everything, always" sidesteps having to think about that composition question correctly, rather than solving it, and papers over exactly the discipline (each component declaring precisely what it needs, composed correctly) that fragment colocation is meant to instill and that will matter for the next several kinds of bugs, not just this one.

The trap: reaching for a blanket, maximally-defensive rule ("just fetch everything") as a reaction to a specific incident, without recognizing that it solves the wrong axis of the problem (breadth of a single query's fields) rather than the axis that was actually broken (request composition/count across components) — and that it reintroduces the original problem GraphQL was adopted to solve, as a permanent, unconditional cost rather than a temporary fix.

---

**Q (Medium): How would a normalized GraphQL client cache (Apollo's InMemoryCache, for instance) change this scenario if, instead of a comments list, this were a page showing the same author's info in two different, unrelated components (e.g., a comment and a separate "recently active users" sidebar)?**

Answer: A normalized cache stores entities by a stable identity (typically `__typename` + `id`) rather than nesting each query's response as an isolated, disconnected blob — so if the comments query and the sidebar's query both include `author { id name avatarUrl }` for the same underlying user, the normalized cache recognizes both responses as referring to the same entity (`User:123`) and stores exactly one copy, with both the comment's rendering and the sidebar's rendering reading from that single normalized entry; if one of the two queries happens to fire second and the entity's already in the cache from the first, the client may be able to satisfy that portion of the second query from cache without a redundant network round trip for already-known fields (behavior varies by client and cache policy, but this is the mechanism that makes it possible). This is directly analogous to the normalized-cache discussion in [[02-cache-invalidation-strategy]] applied to GraphQL specifically — one entity, one canonical cache entry, many consumers reading through it — and it's a meaningfully different mechanism than fragment colocation (which addresses request *composition*/count) though the two work together well: colocation ensures each component asks for exactly what it needs; normalization ensures that if two components happen to need overlapping data about the same entity, that overlap isn't fetched or stored redundantly.

The trap: conflating fragment colocation and normalized caching as solving the same problem — colocation is about how a component's data needs are declared and composed into queries (preventing the N+1/composition bug in this scenario); normalization is about how the *results* of potentially multiple queries touching the same entity are stored and deduplicated client-side (preventing redundant fetches/storage for the same entity referenced from multiple places) — both matter, but they're separate mechanisms addressing separate inefficiencies.

---

**Q (Low): If this were a REST API instead of GraphQL, what would the equivalent fix look like, and is there anything about the REST version of this fix that's actually simpler or harder than the GraphQL version?**

Answer: The REST equivalent of the bug is a comments-list endpoint (`GET /posts/:id/comments`) that returns comments with only an `authorId`, forcing the client to make a separate `GET /users/:id` call per comment — the identical N+1-in-request-count pattern, just without GraphQL's field-selection syntax involved at all. The equivalent fix is a server-side change: either have the comments endpoint embed a denormalized author object directly in each comment in the response (`{ id, text, author: { name, avatarUrl } }`, computed server-side, analogous to the nested GraphQL selection), or provide a batch endpoint (`GET /users?ids=1,2,3,...`) the client can call once with every needed author ID collected from the comments response, trading one extra round trip (batch users) for N. Compared to the GraphQL version, the REST fix is arguably *simpler* to reason about in one sense (no query-shape/fragment-composition abstraction to design — it's a fairly direct "change what this endpoint returns" or "add a batch endpoint" decision) but *harder* in another: REST's response shape is fixed per endpoint for all consumers (unlike GraphQL, where different queries against the same schema can each request different nested shapes as needed), so denormalizing the author into the comments response benefits this one view but might over-fetch for a different consumer of the same comments endpoint that doesn't need author data at all — a trade-off GraphQL's per-query field selection specifically avoids, which is worth naming as a genuine advantage of GraphQL here even though it didn't prevent the *request-count* dimension of this particular bug.

The trap: concluding from this scenario that "GraphQL didn't help, so it's no better than REST for this class of problem" — GraphQL's field-selection benefit (avoiding a fixed response shape forcing over-fetching for consumers who need less) is real and distinct from the request-composition/count problem that caused this specific bug; the two axes (field-selection efficiency vs. request-count/composition efficiency) are independent, and GraphQL genuinely helps with one of them while requiring separate, deliberate design (nesting, colocation) for the other.

---

## Self-Assessment

- [ ] Can explain precisely what GraphQL's over-fetching guarantee does and does not cover (field selection per request vs. request count/composition)
- [ ] Can rewrite a client-side N+1 query pattern into a single nested-selection query and explain why request count stops scaling with list length
- [ ] Can implement and justify fragment colocation as the mechanism that restores component self-containment after flattening a query
- [ ] Can explain why a client-side fix isn't sufficient on its own and knows to verify server-side resolver batching (DataLoader) separately
- [ ] Can identify the under-fetching overcorrection risk and explain why "just fetch everything" isn't the right lesson to take from this incident
- [ ] Can distinguish fragment colocation (request composition) from normalized caching (entity deduplication) as two separate GraphQL client mechanisms

---
*This closes Phase 7 — Networking & Data Layer Scenarios. Phase 8, CSS & Layout Debugging Scenarios, shifts from data-fetching and network resilience to visual correctness — diagnosing and fixing layout bugs that only surface in specific browsers, viewports, or content states, starting with the Holy Grail Layout.*
