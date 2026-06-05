# devils-advocate

A Claude skill that reads a codebase **or** an architecture document, thinks like an attacker, and proposes a concrete **adversarial hardening + testing strategy** — then, once you approve, writes canonical security docs your Claude Code setup will follow on every change.

Most reviews check "does the happy path work." This one asks the opposite: *if a hostile person sent every input, would the system hold?*

## What it does

It walks five attacker questions against your design:

1. **Can I read someone else's data?** — tenant isolation / IDOR
2. **Can I sneak bad bytes into the pipeline?** — ingestion hardening
3. **Can I make the server fetch something it shouldn't?** — SSRF
4. **Can I hijack the AI?** — prompt injection (only if you use an LLM)
5. **If I get in, what can I take, and can I erase it?** — data protection & deletion

For each, it names a realistic attacker, the defense, the **test that proves it**, and the **lint rule / CI gate** that stops the defense from rotting. Its core principle: *a recommendation is only finished when it ships as a test plus an enforcement rule — never as advice.*

## How it works (four phases)

1. **Map (read-only)** — detects code vs. doc, finds the surfaces that matter, and reflects back a plain-English summary of your system for you to confirm.
2. **Threat-map** — runs the five questions, marking what's already handled, what's missing, and what doesn't apply (and why).
3. **Propose (in chat)** — a readable report with a prioritized punch list, then asks for your go-ahead.
4. **Write canonical docs (only after you say yes)** — generates `ADVERSARIAL_HARDENING.md` (the why), `ADVERSARIAL_TEST_PLAN.md` (the attacks to run), and a `CLAUDE.md` discipline block (the per-change rules).

Nothing is written to your repo until you approve.

## Install

This is a standard Claude skill — a folder with a `SKILL.md`. See the [repo README](../../README.md) for plugin and one-line installs; the manual paths are:

**Personal (all your projects):**

```
cp -r skills/devils-advocate ~/.claude/skills/
```

**Per-project (checked into a repo, shared with your team):**

```
cp -r skills/devils-advocate /path/to/your/project/.claude/skills/
```

Then start a Claude Code session and ask something like *"play devil's advocate on this architecture"* or *"review my backend for security weaknesses."* The skill triggers on hardening / threat-model / security-review / IDOR / SSRF / prompt-injection requests.

## Try it

Point it at any design and ask:

- "Poke holes in this architecture doc."
- "What could an attacker do to my upload endpoint?"
- "Threat-model this backend and write me the test plan."
- "Is this secure? Play devil's advocate."

## Layout

```
skills/devils-advocate/
├── SKILL.md                  # triggers + the four-phase workflow
├── README.md
├── reference/
│   ├── five-questions.md      # the method
│   ├── attack-catalog.md      # reusable adversarial inputs
│   └── enforcement-patterns.md# lint + CI recipes
└── templates/
    ├── ADVERSARIAL_HARDENING.md.template
    ├── ADVERSARIAL_TEST_PLAN.md.template
    └── CLAUDE-md-discipline-block.template
```

## Contributing

The attack catalog gets more valuable with more real-world cases. PRs that add payloads to `reference/attack-catalog.md` (with the expected refusal) are welcome.

## A note on trust

Skills are instructions an AI will follow, so only install ones you've read. This skill is read-only until you approve Phase 4, never exfiltrates anything, and writes only the three documents described above. Read `SKILL.md` before installing — that's good practice for *any* skill.

## License

MIT — see `LICENSE`.
