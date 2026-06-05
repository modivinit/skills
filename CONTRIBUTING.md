# Contributing

Thanks for helping make this sharper. Two kinds of contributions are especially welcome.

## Add attack payloads

The strength of the skill is the realism of its attacks. If you've seen an attack the catalog misses, add it to [`skills/devils-advocate/reference/attack-catalog.md`](skills/devils-advocate/reference/attack-catalog.md). Each entry should name:

- the **input** to send, and
- the **expected refusal** (what a hardened system does).

Keep entries defensive — these are test fixtures that *should fail*, not working exploits. No live malware, no real secrets, no payloads whose only purpose is to cause harm.

## Add a skill

This repo is structured to grow. To add one:

1. Create `skills/<your-skill>/SKILL.md` with `name` + `description` frontmatter (the description is the trigger — make it specific).
2. Put any bundled material in `skills/<your-skill>/reference/` and `templates/`.
3. Add the skill to the **Reference** section of the root `README.md`.
4. If it should ship in the plugin, it's auto-loaded from `skills/` — no manifest change needed.

## Style

- Explain the *why* behind each instruction; today's models follow reasoning better than rigid MUSTs.
- Prefer prose with a few tight lists over walls of bullets.
- Every security recommendation pairs with a test and an enforcement rule. A recommendation without a test isn't done.

## Testing a skill change

Point the skill at a real project in a fresh session and confirm the output quality before opening a PR. The `skill-creator` skill (if you have it) can run a more rigorous eval loop.
