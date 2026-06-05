# Devil's Advocate

**Adversarial skills for real engineers. Think like an attacker before you ship.**

Most code review checks "does the happy path work." This skill asks the opposite question — *if a hostile person sent every input, would the system hold?* — and turns the answer into tests and enforcement rules your agent will follow forever.

It reads a codebase **or** an architecture document, threat-models it the way an attacker would, explains the risks in plain English, and — only once you approve — writes canonical security docs that Claude Code obeys on every future change.

One rule runs through all of it:

> A recommendation is only finished when it ships as a **test** plus an **enforcement rule** — never as advice.

"You should validate inputs" is not a deliverable. "Here's the negative test to add and the lint rule that enforces it" is.

---

## Quickstart (30-second setup)

**Option A — install the skill directly (recommended).** Copy it into your skills directory:

```bash
# personal (available in every project)
git clone https://github.com/modivinit/skills
cp -r skills/skills/devils-advocate ~/.claude/skills/

# or per-project (commit it with your repo)
cp -r skills/skills/devils-advocate /path/to/project/.claude/skills/
```

**Option B — install as a Claude Code plugin** (auto-loads the skill, updates with the repo):

```text
/plugin marketplace add modivinit/skills
/plugin install devils-advocate@modivinit-skills
```

**Option C — one-line installer** via [skills.sh](https://skills.sh):

```bash
npx skills@latest add modivinit/skills
```

Then just ask your agent: *"play devil's advocate on this architecture"*, *"poke holes in my upload endpoint"*, or *"threat-model this backend and write me the test plan."*

---

## Why This Skill Exists

I kept watching agents (and humans) ship code that worked perfectly — until someone hostile touched it. The failure is never the happy path. It's the swapped ID, the booby-trapped upload, the URL that points at internal infrastructure, the seller comment that talks the AI into lying. These are old, well-understood attacks, and they still ship every day because nobody adversarially reviewed the design.

This skill encodes that review as a repeatable loop. It's organized around five attacker questions. Each is a real failure mode, with a fix you can actually enforce.

### #1 — "Can I read someone else's data?"

> All input is evil until proven otherwise.
>
> Michael Howard & David LeBlanc, *Writing Secure Code*

**The problem.** The most common real-world breach isn't exotic — it's a logged-in user swapping an ID in a URL and getting back someone else's records. (IDOR: Insecure Direct Object Reference.)

**The fix.** Route every read and write of owned data through an ownership helper, ban raw queries against owned tables with a lint rule, and back it with database row-level security. Then prove it: a cross-user test that, for *every* route, sends User A's token with User B's ID and asserts a 404.

### #2 — "Can I sneak bad bytes into the pipeline?"

**The problem.** Anything a user can upload is an attack surface — an SVG with embedded script, a tiny image that decompresses to gigabytes, a file whose real bytes don't match its extension.

**The fix.** Treat every incoming byte as hostile: sanitize HTML, reject SVG, allowlist MIME types and sources, cap size and dimensions, never execute extracted content. Prove it by feeding the system the bad inputs on purpose and asserting each is refused and audited.

### #3 — "Can I make the server fetch something it shouldn't?"

**The problem.** Give a server a URL to fetch and an attacker will point it at `localhost`, a cloud metadata endpoint, or a private IP. (SSRF: Server-Side Request Forgery.)

**The fix.** Allowlist domains, refuse raw IPs and non-HTTPS, don't follow redirects blindly, defend against DNS rebinding, and re-check the real peer IP mid-fetch. Prove it with a suite of internal-target URLs that must all be refused before any bytes are read.

### #4 — "Can I hijack the AI?"

> Prompt injection is what happens when untrusted text gets treated as trusted instructions.
>
> A framing popularized by Simon Willison

**The problem.** If your system feeds untrusted text to an LLM — listing copy, forum posts, OCR'd documents — someone will write *"ignore previous instructions"* into it and try to steer the model.

**The fix.** Wrap all untrusted text in labeled tags the system prompt declares as *data, not instructions*; tag suspicious patterns instead of deleting them; validate tool parameters; add a critic backstop. Prove it with a library of injection payloads the model must *report* rather than obey.

### #5 — "If I get in, what can I take, and can I erase it?"

> Security is a process, not a product.
>
> Bruce Schneier

**The problem.** Two questions auditors and regulators actually ask: if storage leaks, what's exposed? And when a user demands deletion, does it really happen?

**The fix.** Classify data by sensitivity, envelope-encrypt user content, redact secrets before any third-party trace, address files by opaque IDs only, and run an exact, *tested* deletion cascade. Prove it: no secret reaches a trace, no opaque ID resolves without an ownership check, and deletion leaves exactly the right rows behind.

### The thread that ties it together

None of these defenses survive on good intentions. Each one becomes a **lint rule** (so the bug can't be written), a **negative test** (so the attack provably fails), and a **CI gate** (so a red run blocks merge). A defense that lives only in a doc has already started to drift. That discipline is the whole point.

---

## Reference

### Security

- **[devils-advocate](skills/devils-advocate/SKILL.md)** — Read a codebase or architecture doc, think like an attacker, and propose an adversarial hardening + testing strategy across IDOR, ingestion, SSRF, prompt injection, and data-protection/deletion. On approval, writes `ADVERSARIAL_HARDENING.md`, `ADVERSARIAL_TEST_PLAN.md`, and a `CLAUDE.md` discipline block for Claude Code to follow. Works on full-stack, frontend-only, or backend-only inputs.

*(More skills land here over time — this repo is structured to grow.)*

---

## How It Works

Four phases. The first three are read-only and conversational; nothing is written to your repo until you say yes.

1. **Map** — detects code vs. doc, finds the surfaces that matter, and reflects your system back to you in plain English to confirm.
2. **Threat-map** — runs the five questions, marking what's already handled, what's missing, and what doesn't apply (and why).
3. **Propose** — a readable report with a prioritized punch list, then asks for your go-ahead.
4. **Write canonical docs** *(only after approval)* — the hardening spec, the test plan, and the `CLAUDE.md` discipline block.

---

## A Note on Trust

Skills are instructions an AI will follow, so only install ones you've read. This one is read-only until you approve phase 4, never exfiltrates anything, and writes only the three documents described above. Read the [`SKILL.md`](skills/devils-advocate/SKILL.md) before installing — good practice for *any* skill, from anyone.

## License

MIT — see [`LICENSE`](LICENSE).
