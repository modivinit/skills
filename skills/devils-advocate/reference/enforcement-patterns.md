# Enforcement Patterns — making defenses mechanical

A defense that lives only in a doc drifts the moment someone forgets it. The job of this file is to turn each recommendation into something a machine checks on every commit. **Pair every defense with (a) a negative test and (b) an enforcement rule.** These are language-agnostic recipes — adapt to the project's stack.

---

## The three enforcement primitives

1. **Lint / static rule** — bans the dangerous *pattern* so the bug can't be written.
2. **Negative test** — proves the *attack* fails (from `attack-catalog.md`).
3. **CI gate** — blocks the merge if either fails.

A defense is "done" only when all three exist for it.

---

## Lint-rule recipes

- **Ban raw queries on owned tables.** Forbid direct ORM/SQL calls (`db.select(...)`, `db.update(...)`, raw `SELECT`) against ownership-bearing tables outside the ownership-helper module. (ESLint `no-restricted-syntax` / `no-restricted-imports`; or a `ripgrep` denylist in CI for non-JS stacks.)
- **Funnel all LLM/observability calls through one wrapper.** Ban direct imports of the AI SDK / HTTP-trace client anywhere except the redaction wrapper, so redaction can't be bypassed.
- **Ban un-tagged external content.** Forbid string-concatenating retrieved content directly into a prompt; require it go through the tag-wrapping helper.
- **Ban raw storage keys in client-facing types.** Forbid the real storage-path field from appearing in any response schema, log line, or webhook payload — opaque IDs only.
- **Ban non-allowlisted outbound fetch.** Forbid direct `fetch`/`http.get` to a dynamic URL outside the `isFetchableUrl()`-guarded helper.

Each rule ships with a small passing/failing test corpus so the rule itself is tested.

---

## Negative-test suites

Mirror the catalog in `attack-catalog.md`. Suggested layout:

```
tests/security/
├── cross_user_idor.test.*        # every route: A's token + B's id → 404
├── ingestion_rejection.test.*    # SVG, bombs, MIME mismatch, oversize → rejected + audited
├── ssrf_guard.test.*             # localhost/metadata/private/redirect/rebind → refused
├── prompt_injection.test.*       # payloads reported, not obeyed; tool params clean
└── deletion_cascade.test.*       # post-delete row state + key revocation + no trace leak
```

Make them table-driven: one row per attack input, so adding a new attack is one line. New ingestion path or new route ⇒ add rows before merge.

---

## CI gate

A single command (e.g. `npm run security` / `make security`) that runs lint + the negative suites and exits non-zero on any failure. Wire it into:

- a **pre-commit hook** (fast lint subset), and
- the **CI pipeline** (full suite) as a required check, so a red run blocks merge.

If the project uses a reviewer agent, treat its security findings as **merge-blocking**, with a signed, expiring escape-hatch row for documented exceptions (never silent overrides).

---

## The discipline to leave behind

Encode this as a standing rule in the project's `CLAUDE.md` (see the template):

> *Every security defense ships with a lint rule AND a negative test. A defense described only in prose is incomplete and will be treated as a bug.*

This is what converts a one-time review into a property the codebase keeps.
