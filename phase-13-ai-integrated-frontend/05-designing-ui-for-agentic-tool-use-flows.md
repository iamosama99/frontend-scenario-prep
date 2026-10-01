# Designing UI for Agentic / Tool-use Flows

## Quick Reference

| Design problem | Mechanism | Why |
|---|---|---|
| Opaque multi-step work | Typed event stream → timeline of steps (thinking, tool call, result, answer) | Users trust what they can see; debugging needs visibility |
| Risky actions | Permission tiers + approval gates with clear previews, before side effects | The model can be wrong or manipulated; humans stay in the loop |
| Long runs | Run object with status, progress, resumability, cancel | Closing the tab shouldn't lose or silently continue work |
| Errors/partial success | Per-step status + retry/skip, truthful final summary | Real runs fail halfway; the UI must say what happened |

## The Scenario

"We're building an agent that can search our knowledge base, query an internal API, create tickets, and send emails on the user's behalf. A single request may run 10–30 steps over a few minutes. Design the frontend experience: what does the user see, what can they control, and how do you make it safe?"

## Clarifying Questions

- **Which actions have side effects, and which are reversible?** Reading data vs. sending an email vs. deleting records need different gating.
- **Who's the user — expert/operator or casual end user?** Operators want detail and control; casual users want a clean summary with an escape hatch.
- **Is the run synchronous in the session, or can it be backgrounded and resumed?** Determines whether state lives in the server (run record) or the client.
- **Do we need human-in-the-loop approvals, and are there policy/compliance requirements (audit logs)?** Drives approval UI and persistence.
- **What does the event protocol look like — typed events (`tool_call`, `tool_result`, `message_delta`, `status`) or raw text?** Typed events make structured UI possible; raw text forces fragile parsing.
- **What are failure semantics — can steps be retried, skipped, or must the run abort?** Shapes error UI.
- **Multi-tab/device and collaboration needs?** A run might be viewed by multiple sessions.

## Approach & Trade-offs

**Model the run, not the chat.** An agentic interaction isn't one message; it's a *run* with an ID, a status (`queued → running → awaiting_approval → completed | failed | cancelled`), and an ordered list of *steps*. The UI renders the run object as a timeline derived from a typed event stream. Chat text is just one step type (`assistant_message`). This structure gives resumability (reload and re-fetch the run), multi-tab consistency, audit trails, and testability.

**Event-driven state with a reducer.** Events arrive as a stream: `run_started`, `step_started {id,tool,args}`, `step_delta`, `step_finished {result|error}`, `approval_requested {id, action, preview}`, `message_delta`, `run_finished`. A reducer folds them into run state; the UI is a pure function of that state. Because events are idempotent by `eventId`/`seq`, reconnect can replay from the last sequence number without duplicates.

**Transparency without overwhelm.** Show a compact, scannable timeline: step title ("Searching knowledge base…"), status icon, duration; expandable details (arguments, raw result, errors) for those who want them (progressive disclosure). Stream the final answer prominently; collapse completed intermediate steps by default. Avoid showing raw chain-of-thought dumps; show actions and evidence (sources/citations) rather than rambling internal text.

**Permission and approval model.** Classify tools by risk: *read-only* (auto-run), *reversible writes* (auto-run with undo, or lightweight confirm), *irreversible/outward-facing* (email, payment, delete) → **blocking approval** with a precise preview of what will happen ("Send email to jane@x.com with subject… [Edit] [Approve] [Reject]"). Approvals should be bound to the *exact* arguments shown (hash the payload) so the agent can't change them after approval (TOCTOU). Offer "always allow this tool for this session/project" with scope limits and revocation. Default to least privilege.

**Prompt injection reality.** Tool results (web pages, emails, documents) are untrusted and can contain instructions that hijack the agent. UI mitigations: show where each action originated, require approval for outward-facing actions regardless of how confident the model is, display source content distinctly, and never auto-execute actions derived solely from untrusted content. The frontend isn't the only defense, but it's the last human checkpoint.

**Control and recovery.** Stop (cancels at next safe step), pause/resume where supported, retry a failed step, skip it, edit arguments and re-run, and "undo" for reversible actions. After failure, give a truthful summary: completed steps, failed step with reason, and what state the world is left in. Never claim success the events don't support.

**Background and resumability.** Persist the run server-side; the UI reconnects by `runId` and replays events from `lastSeq`. Notify (in-app/push) when approval is needed or the run finishes. This is why client-only state (a hook holding the stream) is insufficient for multi-minute runs.

**Trade-offs.** More transparency = more UI complexity and cognitive load; more approvals = safer but friction ("approval fatigue" leads users to click-through everything — so tier risk carefully and make approvals meaningful, not constant). Streaming granularity vs. render cost (batch updates). A rigid step schema vs. flexibility for new tool types (use a generic step renderer with registry overrides per tool for rich previews).

## Solution

### Event and state model

```ts
type RunEvent =
  | { seq: number; type: 'run_started'; runId: string }
  | { seq: number; type: 'step_started'; stepId: string; tool: string; args: unknown }
  | { seq: number; type: 'step_finished'; stepId: string; result?: unknown; error?: string }
  | { seq: number; type: 'approval_requested'; stepId: string; risk: 'write' | 'irreversible'; preview: Preview; argsHash: string }
  | { seq: number; type: 'message_delta'; text: string }
  | { seq: number; type: 'run_finished'; status: 'completed' | 'failed' | 'cancelled' };

type RunState = {
  status: 'running' | 'awaiting_approval' | 'completed' | 'failed' | 'cancelled';
  lastSeq: number;
  steps: Record<string, Step>; order: string[];
  answer: string;
  pendingApproval?: Approval;
};

function reduce(s: RunState, e: RunEvent): RunState {
  if (e.seq <= s.lastSeq) return s;          // idempotent replay after reconnect
  switch (e.type) { /* fold each event into state */ }
}
```

### Rendering the timeline with a tool registry

```tsx
const stepRenderers: Record<string, ComponentType<{ step: Step }>> = {
  search_kb: SearchStep,      // shows sources/citations
  create_ticket: TicketStep,  // shows ticket card with link
};

function Timeline({ run }: { run: RunState }) {
  return (
    <ol aria-label="Agent steps">
      {run.order.map(id => {
        const step = run.steps[id];
        const Renderer = stepRenderers[step.tool] ?? GenericStep;
        return <li key={id}><StepHeader step={step} /><Collapsible><Renderer step={step} /></Collapsible></li>;
      })}
    </ol>
  );
}
```

### Approval gate bound to exact arguments

```tsx
function ApprovalCard({ a, runId }: { a: Approval; runId: string }) {
  return (
    <section role="alertdialog" aria-labelledby="ap-title">
      <h3 id="ap-title">Approve: send email?</h3>
      <EmailPreview {...a.preview} />
      <button onClick={() => api.post(`/runs/${runId}/approvals/${a.id}`, { decision: 'approve', argsHash: a.argsHash })}>
        Approve & send
      </button>
      <button onClick={() => api.post(`/runs/${runId}/approvals/${a.id}`, { decision: 'reject' })}>Reject</button>
    </section>
  );
}
// Server verifies argsHash matches the pending action; rejects if arguments changed.
```

### Resumable connection

```ts
const events = await fetchEventsSince(runId, state.lastSeq);   // replay missed
subscribe(runId, { fromSeq: state.lastSeq }, e => dispatch(e));
```

### Truthful summary on failure

```tsx
{run.status === 'failed' && (
  <FailureSummary completed={completedSteps} failed={failedStep}
                  note="The ticket was created, but the email was not sent." />
)}
```

> **Check yourself:** Why bind an approval to a hash of the exact arguments, and why is "model the run" more robust than "stream chat text" for a multi-minute agent?

## Gotchas

- **Treating an agent run as one long chat message.** No resumability, no structure, no audit.
- **Auto-executing outward-facing actions.** Prompt injection and model error make this dangerous.
- **Approval fatigue.** Prompting for everything trains users to click through; tier by risk.
- **Approval without payload binding.** Agent may alter arguments after approval (TOCTOU).
- **Vague previews.** "Approve action?" with no concrete details isn't informed consent.
- **Claiming success the events don't support.** Misleading summaries after partial failures.
- **Losing state on reload.** Keep run state server-side; client rebuilds from events.
- **Duplicate events on reconnect.** Without sequence numbers, steps double-render.
- **Showing raw internals.** Dumping logs overwhelms; use progressive disclosure.
- **No accessible announcements.** Use live regions for approval requests and completion; keep focus management sane when an approval card appears.

## Follow-up Questions

**Q (High): How do you decide which actions need human approval?**

Answer: By reversibility and blast radius: read-only auto-run; reversible, low-impact writes run with undo or a light confirm; irreversible or outward-facing (email, payment, deletion, permission changes) require explicit approval with a concrete preview. Combine with context (production data, external recipients) and let users scope "always allow" narrowly with revocation. Treat any action influenced by untrusted tool output as higher risk.

The trap: either approving everything (fatigue) or nothing (unsafe).

**Q (High): How does prompt injection affect the UI design?**

Answer: Tool results can carry instructions that hijack the agent, so the UI should make provenance visible, never auto-run outward actions derived from untrusted content, show exactly what will be done and where it came from, and keep a human approval gate on high-risk actions. Server-side policy enforcement is required too; the UI is the last checkpoint, not the only one.

The trap: assuming the model "won't fall for it" or that this is purely a backend concern.

**Q (Medium): Why an event-sourced run model instead of just appending messages?**

Answer: Typed events folded by a reducer give deterministic UI, replay after disconnect (by sequence number), multi-tab consistency, an audit trail, and testability (feed events, assert state). Messages alone can't represent steps, approvals, and status transitions cleanly.

The trap: parsing free-form text for structure.

**Q (Medium): How do you show progress without overwhelming users?**

Answer: Layered disclosure: a one-line live status and compact timeline by default, expandable step details (args, results, sources), final answer emphasized, intermediate steps collapsed on completion. Tailor density by user type (operator vs. casual).

The trap: dumping every token/log line or hiding everything behind a spinner.

**Q (Low): What if the user closes the tab mid-run?**

Answer: Runs live on the server; the client reconnects by run ID and replays from the last sequence. If approvals are needed while away, send a notification; unapproved gates simply wait (with timeouts and expiry). Define behavior explicitly: pause at the next gate rather than proceed unattended.

The trap: letting an unattended run perform risky actions because nobody can click Approve.

## Self-Assessment

- [ ] Can model a run (status, steps, events) rather than a chat transcript
- [ ] Can write the reducer idea with sequence-based idempotency
- [ ] Can tier tool risk and design the approval gate (payload-bound)
- [ ] Can explain prompt-injection implications for the UI
- [ ] Can describe truthful partial-failure reporting and retry/skip controls
- [ ] Can design resumability across reloads and tabs

---
*Next: Handling AI Latency and Failure Gracefully in the UI — closes the phase on the non-happy-path: slow, flaky, rate-limited, and wrong.*
