# Devil's Advocate

Think like an attacker before you ship.

Most code review asks "does this work?" The answer is usually yes — the happy path almost always works. That's not where things break. They break when someone hostile shows up: a logged-in user swaps an ID in a URL and reads another customer's data, an "image" upload turns out to be a script, a URL field gets pointed at your internal network, a product review quietly tells your AI assistant to ignore its instructions.

These aren't clever zero-days. They're old, boring, well-documented attacks. They keep shipping anyway, because nobody sat down and asked "what would I do if I wanted to break this?"

This skill does that, on a codebase or even just an architecture doc. It walks through five attacker questions, explains what it finds in plain English, and — only after you say go — writes the results down as security docs your Claude Code setup actually follows on every future change.

There's one rule it never breaks:

> A recommendation isn't finished until it ships as a **test** plus an **enforcement rule**. Never as advice.

"You should validate inputs" helps no one. "Here's the negative test that proves the bug is gone, and the lint rule that stops anyone from reintroducing it" — that's the deliverable.

## Get started

The fastest path is to drop the skill into your skills folder:

```bash
git clone https://github.com/modivinit/skills

# available in every project
cp -r skills/skills/devils-advocate ~/.claude/skills/

# or commit it with one repo so your team gets it too
cp -r skills/skills/devils-advocate /path/to/project/.claude/skills/
```

Prefer plugins? Install it as one and it'll update with the repo:

```text
/plugin marketplace add modivinit/skills
/plugin install devils-advocate@modivinit-skills
```

Or use the one-line installer from [skills.sh](https://skills.sh):

```bash
npx skills@latest add modivinit/skills
```

Once it's in, just talk to your agent normally: "play devil's advocate on this architecture," "poke holes in my upload endpoint," "threat-model this backend and write me the test plan." It picks up on hardening, threat-model, security-review, and the usual suspects (IDOR, SSRF, prompt injection) on its own.

## The five questions

Everything is organized around five questions an attacker would ask. Each one maps to a real failure mode and a fix you can actually enforce.

**1. "Can I read someone else's data?"** The single most common breach isn't exotic — it's a logged-in user changing an ID in a URL and getting back records that aren't theirs. (The acronym is IDOR, insecure direct object reference, but the mechanic is just "swap the number.") The fix is to route every read and write of owned data through one ownership check, ban raw queries against owned tables with a lint rule, and back it with row-level security in the database. Then prove it: for every route, send User A's token with User B's ID and assert you get a 404.

**2. "Can I sneak bad bytes into the pipeline?"** Anything a user can upload is an attack surface. An SVG with a script inside. A tiny image that unzips to gigabytes. A file whose actual bytes don't match its extension. Treat every incoming byte as hostile — sanitize, reject what you can't vouch for, cap sizes, and never execute what you extracted. Then feed it the bad inputs on purpose and confirm each one bounces.

**3. "Can I make the server fetch something it shouldn't?"** Hand a server a URL and an attacker will point it at `localhost`, a cloud metadata endpoint, or a private IP. (This one's SSRF.) Allowlist the domains you trust, refuse raw IPs and plain HTTP, don't blindly follow redirects, and re-check who you're actually talking to mid-request. Prove it with a pile of internal-target URLs that all have to be refused before a single byte comes back.

**4. "Can I hijack the AI?"** If your system ever feeds untrusted text to a model — listing copy, forum posts, scanned documents — someone is going to write "ignore previous instructions" into it. Wrap untrusted text in labels your system prompt treats as data, not commands. Flag suspicious patterns instead of silently obeying them. Then keep a library of injection payloads around and confirm the model reports them rather than following them.

**5. "If I get in, what can I take, and can I erase it?"** Two questions auditors really do ask: if your storage leaks, what's exposed? And when a user demands deletion, does it actually happen? Classify data by how sensitive it is, encrypt user content, keep secrets out of third-party traces, and run a deletion cascade you've actually tested — so no orphaned rows survive and no background job quietly resurrects deleted data.

None of these defenses survive on good intentions. Each one has to become a lint rule (so the bug can't be written), a negative test (so the attack provably fails), and a CI gate (so a red run blocks the merge). The moment a defense lives only in a doc, it's already drifting. Closing that gap is the entire point.

## How a run goes

Four phases. The first three are read-only and happen in chat — nothing touches your repo until you approve.

1. **Map.** It works out whether it's looking at code or a doc, finds the parts that matter, and tells you in plain English what it thinks your system is. You confirm or correct it before anything else happens.
2. **Threat-map.** It runs the five questions, noting what you already handle well, what's missing, and what simply doesn't apply (and says why — silence is never treated as safety).
3. **Propose.** A readable report and a prioritized punch list: what to fix first, ordered by risk against effort. Then it asks for the go-ahead.
4. **Write it down** *(only after you say yes).* It generates `ADVERSARIAL_HARDENING.md` (the reasoning), `ADVERSARIAL_TEST_PLAN.md` (the attacks to run), and a `CLAUDE.md` discipline block (the rules every future change has to follow). On GitHub projects it can also drop in a CI workflow that runs the whole test plan on every PR.

## Want a second opinion?

There's a catch with any AI reviewing its own work: the same model that wrote or read the code tends to share its blind spots. So there's an optional cross-model step. Point the `DEVILS_ADVOCATE_REVIEWER` environment variable at any reviewer CLI — OpenAI Codex, Gemini, a local model through Ollama, whatever you've got — and the skill hands the whole analysis to that model and asks it to attack the conclusions. You see its findings raw, then a reconciliation against the first pass.

It's off by default and entirely optional. Reach for it on the high-stakes stuff — auth, payments, deletion, anything a regulator cares about. The setup and guardrails live in [`reference/cross-model-review.md`](skills/devils-advocate/reference/cross-model-review.md).

## Making it stick in CI

A test plan nobody runs is just a document. For GitHub repos, the skill can write a `.github/workflows/adversarial-ci.yml` that runs your lint rules and negative suites on every pull request. Mark it a required check in branch protection and a red run blocks the merge — which is the whole reason the test plan exists. The workflow also ships with a commented-out cross-model review job, so an independent model can weigh in on each diff once you're ready for it.

## What's in here

```
skills/devils-advocate/
├── SKILL.md                    # triggers + the four-phase workflow
├── README.md
├── reference/
│   ├── five-questions.md       # the method in detail
│   ├── attack-catalog.md       # reusable adversarial inputs
│   ├── enforcement-patterns.md # lint + test + CI recipes
│   └── cross-model-review.md   # running a second model against the analysis
└── templates/
    ├── ADVERSARIAL_HARDENING.md.template
    ├── ADVERSARIAL_TEST_PLAN.md.template
    ├── CLAUDE-md-discipline-block.template
    ├── external-reviewer-prompt.md.template  # the brief sent to the second model
    └── adversarial-ci.yml.template            # the GitHub Actions gate
```

More skills will land here over time; the repo is set up to grow.

## A word on trust

A skill is a set of instructions an AI will follow, so only install ones you've actually read. This one stays read-only until you approve phase 4, never sends your code anywhere on its own, and writes only the files described above. The one exception is the optional cross-model step, which by design sends excerpts to whatever reviewer you configured — it'll tell you before it does, and you can point it at a local model if that matters. Read [`SKILL.md`](skills/devils-advocate/SKILL.md) before installing. That's good practice for any skill, from anyone.

## Contributing

The attack catalog is the part that gets more useful the more real cases it has. If you've got a payload worth adding, a PR to [`reference/attack-catalog.md`](skills/devils-advocate/reference/attack-catalog.md) — with the input and the refusal it should trigger — is very welcome.

## License

MIT. See [`LICENSE`](LICENSE).
