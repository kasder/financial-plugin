# financial-plugin

Claude Code plugin bundling the **Cabinet Chraibi** virtual analyst team — a multi-agent pipeline for serious business-plan reviews in French.

## What it does

Drops you into a 7-agent workflow when you run `/analyse` (or ask Claude to analyse a BP) inside a directory containing a PDF/DOCX/XLSX business plan:

1. **lead** does a reconnaissance read and dispatches the team.
2. **finance**, **market**, **investment**, **legal**, **risk** each draft their angle **in parallel**.
3. **challenger** tears all five drafts apart, writes a consolidated critique.
4. The five specialists revise **in parallel** with the full critique in hand.
5. **lead** synthesises the final French memo at `./analysis_<project>.md` and gives an explicit verdict (Go / Go conditionnel / No-go) plus 3-5 open questions.

After the memo lands, the team stays in standby — you can ask follow-up questions and lead routes them to the right specialist without re-running the whole pipeline.

## Contents

| Path | Purpose |
|---|---|
| `.claude-plugin/plugin.json` | Plugin manifest |
| `.claude-plugin/marketplace.json` | Marketplace entry (single-plugin marketplace) |
| `skills/analyse/SKILL.md` | The `analyse` skill — dispatch + bookkeeping logic |
| `commands/analyse.md` | `/analyse` slash command (thin wrapper over the skill) |
| `agents/{lead,finance,market,investment,legal,risk,challenger}.md` | The 7 subagents |

## Install

### From a local path

```
/plugin marketplace add /Users/kadser/Desktop/financial-plugin
/plugin install financial-plugin@financial-plugin-marketplace
```

### From a git repo

Push this directory to a git repo (e.g. `github.com/<you>/financial-plugin`) and:

```
/plugin marketplace add <you>/financial-plugin
/plugin install financial-plugin@financial-plugin-marketplace
```

Restart Claude Code (or run `/plugin reload`) so the agents and skill get picked up.

## Use

In a directory containing your BP file(s):

```
/analyse
```

…or just ask in natural language: *"analyse ce dossier"*, *"passe ça à l'équipe"*, *"run the analysts on this BP"*.

Pass a language hint to flip user-facing output to English (agent drafts stay French):

```
/analyse en anglais
```

## Output

- Final memo: `./analysis_<project>.md` (at the repo root)
- Intermediate drafts + critique: `./analysis_<project>/`

## Requirements

- Claude Code with plugin support (`/plugin` command available).
- All 7 subagents must load — the skill has a pre-flight guard that refuses to run with an incomplete team to avoid silently degraded output.

## Customising

- Edit specialist personas in `agents/<name>.md` — they're standalone Markdown subagent files.
- Tune the pipeline (parallelism, rounds, file layout) in `skills/analyse/SKILL.md`.
- The skill is opinionated about **language (French)** and **two-round critique-and-revise**; both can be relaxed by editing the SKILL.md.
