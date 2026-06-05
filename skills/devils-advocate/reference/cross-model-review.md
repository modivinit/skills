# Cross-Model Review — getting a second model to check the work

Claude reviewing its own threat model has a blind spot: the same weights that
read your code form the same assumptions about it. A finding Claude misses on
the first pass, it tends to miss on the second pass too. The cheapest fix is to
hand the analysis to a *different* model and ask it to attack the conclusions.

This is **optional** and **off by default**. Use it when the stakes are high
(auth, payments, deletion, anything regulated) or when the user explicitly asks
for a second opinion. Skip it for a quick "poke holes in this" pass.

---

## Model-agnostic by design

The skill does not hard-wire any vendor. It shells out to whatever reviewer CLI
the user has configured, via a single environment variable:

```bash
# Whatever command reads a prompt on stdin and prints a review on stdout.
# Examples (the user picks one; the skill does not assume any of these exist):
export DEVILS_ADVOCATE_REVIEWER="codex exec -s read-only"
export DEVILS_ADVOCATE_REVIEWER="gemini -p"
export DEVILS_ADVOCATE_REVIEWER="ollama run llama3.3"
export DEVILS_ADVOCATE_REVIEWER="llm -m gpt-5"   # simonw/llm, any backend
```

The contract is deliberately minimal — the reviewer must:

1. accept a prompt (on stdin or as a trailing argument), and
2. print its review to stdout.

Anything that satisfies that contract works, including a local model. If the
variable is unset, the skill says so and continues without the cross-model step
rather than failing.

---

## How the skill runs it

The cross-model pass happens **after Phase 3** (the proposal is on the table)
and **before** writing any docs. The steps:

1. **Build a self-contained brief.** Fill `templates/external-reviewer-prompt.md.template`
   with the system summary, the per-question findings, and the punch list. The
   brief must stand alone — the other model has not seen your repo, so include
   the relevant code/architecture excerpts inline.

2. **Send it to the configured reviewer.** Write the brief to a temp file and
   pipe it in. Keep the reviewer **read-only** — it critiques, it does not touch
   the repo:

   ```bash
   tmp="$(mktemp -t devils-advocate-brief.XXXXXX.md)"
   # ... skill writes the brief into "$tmp" ...
   cat "$tmp" | ${DEVILS_ADVOCATE_REVIEWER:?set DEVILS_ADVOCATE_REVIEWER to enable cross-model review}
   rm -f "$tmp"
   ```

3. **Reconcile, don't rubber-stamp.** Show the reviewer's findings **verbatim**
   first (no paraphrasing — that's where Claude's blind spot would re-enter),
   then sort them into:
   - **Confirmed** — a real gap Claude's pass missed → add it to the punch list.
   - **Already covered** — note where it's handled, so the user sees the overlap.
   - **Disagreed** — say why, in one sentence; the user is the tiebreaker.

4. **Re-ask for approval.** The reconciled punch list may have grown, so confirm
   before moving to Phase 4.

---

## Guardrails

- **Read-only reviewer.** Never grant the external model write or exec access to
  the repo. It receives text and returns text. Prefer the CLI's read-only flag
  where one exists (e.g. `-s read-only`).
- **Mind what leaves the machine.** A hosted reviewer means code excerpts go to a
  third party. Say so before the first call, and honor an opt-out. For sensitive
  code, point the user at a local model command instead.
- **One strong disagreement beats five weak agreements.** A second model that
  only ever says "looks good" is adding latency, not safety. If it never pushes
  back across several runs, tell the user the cross-model step isn't earning its
  keep on this codebase.
- **Verbatim first.** Always surface the reviewer's raw output before your
  reconciliation. Summarizing first lets the same blind spot quietly drop the
  one finding that mattered.
