# bnb-bounty-reviewer

`bnb-bounty-reviewer` is a review skill for triaging BNB Chain bug bounty reports against local BNB repositories such as `bsc`, `bsc-genesis-contract`, `greenfield-contracts`, `greenfield-cosmos-sdk`, `opbnb`, and related projects.

This repository is structured so the same skill can be used by both Claude and Codex:

- Claude uses the repository-level plugin layout in this repo.
- Codex uses the nested skill directory at `skills/bnb-bounty-reviewer/`.

## Repository Layout

```text
.
├── .claude-plugin/                 # Claude plugin metadata
├── evals/                          # Skill evaluation cases
└── skills/
    └── bnb-bounty-reviewer/
        └── SKILL.md                # Actual skill content
```

## Quick Install With An Agent

If your agent supports skill installation from GitHub, this is the easiest option.

### Codex

Ask Codex:

```text
Install the skill from GitHub repo MatusKysel/bnb-bounty-reviewer using path skills/bnb-bounty-reviewer.
```

Codex should install the nested skill into:

```text
~/.codex/skills/bnb-bounty-reviewer
```

### Claude

Ask Claude:

```text
Install or clone https://github.com/MatusKysel/bnb-bounty-reviewer into my Claude skills directory so I can use bnb-bounty-reviewer.
```

Claude should place the repository at:

```text
~/.claude/skills/bnb-bounty-reviewer
```

## Manual Install

### Claude

Claude expects the full repository layout, including `.claude-plugin/`.

```bash
git clone git@github.com:MatusKysel/bnb-bounty-reviewer.git ~/.claude/skills/bnb-bounty-reviewer
```

Or with HTTPS:

```bash
git clone https://github.com/MatusKysel/bnb-bounty-reviewer.git ~/.claude/skills/bnb-bounty-reviewer
```

### Codex

Codex only needs the actual skill directory.

Option 1: install through Codex from GitHub using:

```text
repo: MatusKysel/bnb-bounty-reviewer
path: skills/bnb-bounty-reviewer
```

Option 2: copy it manually:

```bash
mkdir -p ~/.codex/skills
cp -R skills/bnb-bounty-reviewer ~/.codex/skills/bnb-bounty-reviewer
```

If you are running that command from outside this repository, copy the nested directory from this repo:

```text
bnb-bounty-reviewer/skills/bnb-bounty-reviewer -> ~/.codex/skills/bnb-bounty-reviewer
```

## Updating

### Claude

If installed as a git checkout:

```bash
git -C ~/.claude/skills/bnb-bounty-reviewer pull --ff-only origin main
```

### Codex

If installed via git or copy, update by replacing the installed skill directory with the latest version of:

```text
skills/bnb-bounty-reviewer/
```

If your Codex environment supports GitHub skill install prompts, you can ask it to reinstall/update from the same repo and path.

## Notes

- The skill content lives in `skills/bnb-bounty-reviewer/SKILL.md`.
- Claude and Codex do not necessarily auto-refresh installed skills from GitHub.
- Starting a new session is the safest way to ensure updated skill metadata is reloaded.
