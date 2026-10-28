# ai-skils

The AI skills I use every day with Claude Code and Cursor.

Each skill is a folder with a `SKILL.md` that tells the agent when to use it and how.

## Skills

| Skill | What it does |
|---|---|
| [endpoint-mapping](skills/endpoint-mapping) | Documents every endpoint of an API server (routes, auth, roles, payloads, errors) straight from the backend code |
| [landing-page-design](skills/landing-page-design) | Plans, writes and designs high-converting landing pages, with a default visual system |
| [thermo-nuclear-code-quality-review](skills/thermo-nuclear-code-quality-review) | A very strict maintainability review: abstractions, giant files, tangled conditions. Runs only when called |
| [writing-skills](skills/writing-skills) | How to write, test and improve skills like these |
| [ponytail](skills/ponytail) | "Lazy senior dev" mode: the smallest change that fully solves the task |
| [ponytail-review](skills/ponytail-review) | Reviews a change for correctness, safety, tests, speed and unneeded code |
| [ponytail-audit](skills/ponytail-audit) | Audits a whole repo and ranks what to fix or delete |
| [ponytail-debt](skills/ponytail-debt) | Lists the shortcuts left in the code as a debt ledger |
| [ponytail-gain](skills/ponytail-gain) | Shows ponytail's measured savings |
| [ponytail-help](skills/ponytail-help) | Quick reference for the ponytail skills |

## Install

Link each skill folder into your skills directory, so edits here apply right away:

```bash
# Claude Code (all projects)
ln -s "$PWD/skills/endpoint-mapping" ~/.claude/skills/endpoint-mapping

# Cursor
ln -s "$PWD/skills/endpoint-mapping" ~/.cursor/skills/endpoint-mapping
```

Or link them all at once:

```bash
for s in skills/*/; do ln -s "$PWD/${s%/}" ~/.claude/skills/"$(basename "$s")"; done
```

Skill folders must sit directly inside the skills directory (`~/.claude/skills/<name>/SKILL.md`). Start a new session to load them.

## Credits

Some of these skills are my own, and some come from other people, sometimes adapted to how I work:

- **ponytail** skills: [DietrichGebert/ponytail](https://github.com/DietrichGebert/ponytail) (MIT)
- **writing-skills**: adapted from [superpowers](https://github.com/obra/superpowers) by Jesse Vincent (MIT)
- **landing-page-design**: taken from a public GitHub repo; parts adapted from [taste-skill](https://github.com/Leonxlnx/taste-skill) by Leon Lin (MIT, see `LICENSE-taste-skill`)

All credit for those goes to their original authors.
