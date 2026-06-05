# Repo context for agents

This repo distributes Claude skills. It is both a **plugin marketplace** and a plain **skills collection** — install either way (see `README.md`).

## Layout

```
.
├── .claude-plugin/
│   ├── marketplace.json   # lists this repo's plugin(s)
│   └── plugin.json        # plugin metadata; skills/ is auto-loaded
├── skills/
│   └── devils-advocate/   # one folder per skill, each with a SKILL.md
├── CLAUDE.md              # this file
├── CONTRIBUTING.md
├── LICENSE
└── README.md             # the showcase / docs
```

## Conventions

- **A skill is a folder with a `SKILL.md`** whose frontmatter has `name` + `description`. The `description` is the trigger — keep it specific and slightly pushy so the skill fires when it's genuinely useful.
- **Bundled material** lives under each skill in `reference/` (docs loaded on demand) and `templates/` (files the skill fills in). Keep `SKILL.md` lean and point to those.
- **Every security recommendation pairs with a test and an enforcement rule.** A recommendation in prose only is incomplete — that discipline is the product, not a nicety.
- **The root `README.md` Reference section lists every skill.** Add a line when you add a skill.

## When editing

- Don't rename a published skill folder or its `name` field without a deprecation note — installs reference it.
- Bump `version` in `.claude-plugin/plugin.json` on a release and tag it (`v0.1.0`, …).
- Keep docs generic — no project-specific or private details in skill content.
