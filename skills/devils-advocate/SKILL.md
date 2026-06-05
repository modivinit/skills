---
name: devils-advocate
description: Read a codebase or architecture document, think like an attacker, and propose an adversarial hardening and testing strategy — then, on approval, write canonical security docs for Claude Code to follow. Use this whenever the user wants to harden a design, threat-model a system, plan security or penetration tests, do a security review, or asks about IDOR, SSRF, prompt injection, input/ingestion validation, data-leak, or deletion-correctness risks. Trigger even when the user only says "poke holes in this," "what could go wrong," "is this secure," "review my architecture for weaknesses," or "play devil's advocate" — and works on full-stack, frontend-only, or backend-only inputs (code or docs).
---

# Devil's Advocate — Adversarial Hardening & Testing

Attack the design before someone else does. Adversarial review for a codebase or an architecture doc — ending in tests and rules, not advice.

## What this skill is for

Most reviews check "does the happy path work." This one asks the opposite question: **if a hostile person sent every input, would the system hold?** It reads a design (code or an architecture doc), thinks like an attacker, and produces a concrete hardening + testing plan written so that both a non-engineer and a future Claude Code session can act on it.

**This is the skill: a recommendation is finished only when it ships as a test plus an enforcement rule. Everything else is advice theater.** "You should validate inputs" = not done. "Here's the negative test that fails on the attack, plus the lint rule that stops anyone re-adding the bug" = done. If you can't name the test, the recommendation isn't ready — say so, don't pad.

**Anti-pattern: advice theater.** A report full of "consider," "should," and "might want to," with no test and no rule attached. That's the exact failure mode this skill exists to kill.

## The method in one breath

Walk five attacker questions against the target. Each maps to a real attacker, a real defense, and a real test:

1. **"Can I read someone else's data?"** — tenant isolation / IDOR
2. **"Can I sneak bad bytes into the pipeline?"** — ingestion hardening
3. **"Can I make the server fetch something it shouldn't?"** — SSRF
4. **"Can I hijack the AI?"** — prompt injection (only if the system uses an LLM)
5. **"If I get in, what can I take, and can I erase it?"** — data protection & deletion

Full plain-English detail, with defenses and tests for each, is in `reference/five-questions.md`. Read it at the start of every run — it's the backbone, and skipping it is skipping the skill.

## The workflow — four phases

Phases 1–3 (and the optional 3.5) are **read-only and conversational**. Phase 4 writes files and runs **only after the user explicitly approves.** Do not skip the approval checkpoint; the value of this skill is that a human decides before anything is committed.

### Phase 1 — Detect and map (read-only)

1. Figure out what you're looking at: a codebase or an architecture document; full-stack, frontend-only, or backend-only. Say which, so the user can correct you.
2. Locate the surfaces that matter:
   - **In code:** route/handler definitions, file-upload paths, outbound HTTP/`fetch` calls, LLM/AI calls, auth and ownership checks, raw database queries.
   - **In a doc:** the sections that describe those same surfaces.
   Use search broadly here. If the target is large, spawn parallel search agents rather than reading everything yourself.
3. Write a short **"Here's what I think your system is"** summary in plain English — what it does, where data enters, who the users are, what's stored, what's trusted. Ask the user to confirm or correct it. This anchors everything and catches misreadings while they're cheap.

**Do not proceed to Phase 2 until the user confirms your system summary.** A misread system produces a confident, wrong threat model — the most expensive kind.

### Phase 2 — Threat-map against the five questions

For each of the five questions, record four things:

- **Applies?** Some questions won't (a static marketing site has no tenant-isolation surface; a no-LLM app skips prompt injection). **Silence is not safety** — a question you skip without a reason is a gap you're pretending isn't there. Either write *"not applicable, because…"* or treat it as applicable.
- **The attacker** for *this* system, in one sentence, concrete enough to test. "A paying user swaps another tenant's invoice ID into `GET /invoices/:id`." If you can't name the move, you don't understand the surface yet.
- **Already defended** — what the design already does right (say so; it builds trust and avoids noise).
- **Missing or weak** — the gap.

`reference/attack-catalog.md` holds reusable, concrete attack inputs (SSRF URLs, injection strings, malformed-upload cases, the cross-user ID-swap recipe). Pull from it so the threats are specific, not hand-wavy.

### Phase 3 — Propose the strategy (the report, in chat)

Present a readable proposal. For each applicable area give:

- the attacker, one sentence;
- the recommended defense(s), in plain language;
- the **test** that proves the defense — the adversarial input that *should fail*;
- a **mechanical-enforcement** note: the lint rule / CI gate that stops the defense from silently rotting (see `reference/enforcement-patterns.md`).

End with a **prioritized punch list** (what to fix first, ordered by risk × effort) and an explicit approval question, e.g. *"Want me to write these up as canonical docs your Claude Code setup will follow?"*

Keep the whole report dual-audience: simple enough for a founder to follow, precise enough for Claude Code to execute. Avoid jargon without a one-line gloss.

### Phase 3.5 — Cross-model review (optional, off by default)

Claude reviewing its own threat model shares its own blind spots — the same weights that read the code make the same assumptions about it. For high-stakes targets (auth, payments, deletion, anything regulated) or when the user asks for a second opinion, hand the analysis to an **independent** model and have it attack the conclusions.

This runs **only** if the user opts in *and* a reviewer is configured via the `DEVILS_ADVOCATE_REVIEWER` environment variable — any CLI that reads a prompt on stdin and prints a review on stdout (Codex, Gemini, a local model, whatever they set). If it's unset, say so and continue without this step rather than failing.

The flow: fill `templates/external-reviewer-prompt.md.template` into a self-contained brief (the other model can't see the repo, so inline the relevant excerpts), pipe it to the configured reviewer **read-only**, then show its findings **verbatim** before reconciling them into *confirmed / already-covered / disagreed*. Re-confirm the punch list before Phase 4. Full procedure and guardrails are in `reference/cross-model-review.md` — read it before running this step.

### Phase 4 — Write canonical docs (only after "yes")

Generate the artifacts below from the templates in `templates/`, filling them with the findings from Phases 2–3:

1. **`ADVERSARIAL_HARDENING.md`** — the strategy and rationale (the "why"). From `templates/ADVERSARIAL_HARDENING.md.template`.
2. **`ADVERSARIAL_TEST_PLAN.md`** — the concrete attack catalog: every negative test to write, grouped by area, each with the input to send and the expected refusal (the "prove it"). From `templates/ADVERSARIAL_TEST_PLAN.md.template`.
3. **A `CLAUDE.md` discipline block** — short imperative rules a future Claude Code session obeys on every change. From `templates/CLAUDE-md-discipline-block.template`.
4. **A CI gate** *(offer it; write it if the project uses GitHub and the user wants it)* — `.github/workflows/adversarial-ci.yml`, which runs the lint rules and negative suites on every PR so a red run blocks merge. From `templates/adversarial-ci.yml.template`. The discipline only holds if it's mechanically enforced; the workflow is what turns the test plan into a gate. Fill in the project's real security command and tell the user to mark the check **required** in branch protection.

Placement rules:
- Put the two `.md` files where the project keeps docs (repo root or a `docs/` or `security/` folder — ask if unclear).
- If the project already has a `CLAUDE.md`, **append** the discipline block under a clear heading; never overwrite the existing file.
- Put the workflow at `.github/workflows/adversarial-ci.yml`; if a security workflow already exists, fold the steps in rather than clobbering it.
- Tell the user exactly what you wrote and where, and what Claude Code will now do differently.

## Output style notes

- Prefer prose with a few tight lists over walls of bullets.
- Name realistic attackers, not abstractions ("a paying user swapping an ID," not "a threat actor").
- Every defense in the final docs must be paired with a test and an enforcement mechanism. If you can't name a test for a recommendation, it's not ready — say so rather than padding the report.
- **Don't invent vulnerabilities to look thorough.** "This area is well-handled" is a finding, and a valuable one. A padded report trains the user to ignore the real ones.

## When NOT to over-apply this skill

If the user asks a narrow factual security question ("what's a safe bcrypt cost factor?"), just answer it — don't launch the four-phase workflow. The workflow is for *reviewing a design*, not for every security-adjacent question.
