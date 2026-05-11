---
name: analyse
description: Analyse a business plan (or similar long-form deliverable) inside the current repository using the Chraibi analyst team — lead agent orchestrates 5 specialists (finance, market, investment, legal, risk) plus a challenger, with a two-round critique-and-revise pipeline, and produces a final French memo at the repo root. Use whenever the user wants a BP analysed, asks for a "memo", "analyse ce dossier", "analyse this BP", "passe ça à l'équipe", "run the analysts", or invokes /analyse — even if they don't name the team or skill explicitly. Trigger when there is a PDF/DOCX/XLSX in the current working directory that looks like a business plan or investment file and the user wants opinions on it.
---

# Analyse

Multi-agent business-plan analysis driven by the Chraibi analyst team. Operates inside whatever repository the user is currently in — the team reads the files in the repo, produces drafts in a sibling `analysis_<project>/` folder, and lands the final memo as `analysis_<project>.md` at the repo root.

## Why this skill exists

A serious BP review touches finance, market, investment thesis, legal/regulatory exposure, and operational risk. One pass is rarely defensible — the first draft is always missing something a sharp critic would catch. This skill institutionalises the workflow M. Chraibi uses: each specialist drafts, a challenger tears the drafts apart, the specialists revise, and the lead synthesises a final memo. The skill handles dispatch and bookkeeping so the user stays in the conversation rather than wrangling agents.

## Pre-flight check

Before doing anything else, verify the team is dispatchable. The skill needs these 7 subagent types to be available via the Agent tool: `lead`, `finance`, `market`, `investment`, `legal`, `risk`, `challenger`.

The authoritative source of truth is the **runtime list of subagent types** visible in your own tool description for the Agent tool — not any disk path. Disk paths vary by environment (Claude Code uses `~/.claude/agents/`, Cowork has its own sandbox under `/sessions/<id>/mnt/.claude/`, plugin-bundled agents live under the plugin directory). If an agent is dispatchable, the runtime exposes it; if it's not exposed, dispatching it will fail regardless of what's on disk.

Check the agents by inspecting your own available subagent types. If any of the seven names above are missing from that list, fail loudly with a French message naming the missing agents and refusing to proceed:

> Erreur: l'équipe d'analystes Chraibi est incomplète. Agents manquants: {liste}. Vérifiez que le plugin `financial-plugin` est installé et activé dans l'environnement courant, ou installez les agents manuellement dans `~/.claude/agents/`. La skill refuse de tourner sans l'équipe complète pour éviter de produire un mémo silencieusement dégradé.

Helpful diagnostic to include when failing: list the subagent types you DO see, so the user can tell whether the plugin loaded partially, not at all, or under different names.

This guard exists because the rest of the workflow depends on every specialist being dispatchable. A missing agent would corrupt the critique pipeline silently if we let it slide — better to refuse outright.

## Language

User-facing output (file picker, confirmations, final summary, error messages) is **French by default**. The user can override at invocation by saying e.g. "in English", "en anglais", or by passing a language hint with the trigger. If they did, use that language for everything Claude says back to the user; the agents themselves are written in French and may continue to draft in French regardless.

## Step 1 — Identify the file(s)

The skill operates in the current working directory and figures out the target itself, but it never assumes — every selection is confirmed with the user.

1. List the cwd and pick out plausible candidates: PDF, DOCX, XLSX, ODT, KEY, PPT/PPTX. Skip lockfiles, dotfiles, and obviously unrelated extensions.
2. Decide based on what's there:
   - **One candidate**: surface it and ask for confirmation. *"J'ai trouvé `BP_Atlas.pdf`. C'est ce que vous voulez analyser ?"*
   - **Multiple candidates**: list them with short metadata (size, mtime) and ask which file is the main BP and which (if any) are companions like a financials Excel. Multi-select OK.
   - **None**: ask the user for the path explicitly. Don't guess.
3. Wait for explicit user confirmation before proceeding. The cost of a wrong-file analysis is wasted compute and a misleading memo, so the friction here is worth it.

Once confirmed, derive `PROJECT_NAME` from the main file: strip extension, replace spaces with underscores, keep the original case (`BP Mining Atlas.pdf` → `BP_Mining_Atlas`). This becomes the suffix for both the workspace folder and the final memo filename.

## Step 2 — Set up the workspace

Create `./analysis_{PROJECT_NAME}/` at the repo root. This holds intermediate drafts and the critique. Final memo lives one level up at `./analysis_{PROJECT_NAME}.md`.

Internal layout:

```
analysis_BP_Mining_Atlas/
├── reconnaissance.md          # lead's preliminary read
├── round1/
│   ├── finance.md
│   ├── market.md
│   ├── investment.md
│   ├── legal.md
│   └── risk.md
├── critique.md                # challenger's findings
└── round2/
    ├── finance.md
    ├── market.md
    ├── investment.md
    ├── legal.md
    └── risk.md
```

If `analysis_{PROJECT_NAME}/` or `analysis_{PROJECT_NAME}.md` already exists, ask the user before overwriting — prior runs are user state and must not be silently clobbered.

## Step 3 — Dispatch the lead

Spawn the `lead` agent with the Agent tool (`subagent_type: lead`). The lead is the only agent the skill talks to directly; lead then orchestrates everyone else.

Hand the lead a prompt along these lines (in French, since lead operates in French):

```
Nouveau dossier à analyser.

Fichiers sources: {chemins absolus}
Workspace: {repo_root}/analysis_{PROJECT_NAME}/
Mémo final: {repo_root}/analysis_{PROJECT_NAME}.md

Protocole:
1. Lis les fichiers sources nativement (PDF, DOCX, XLSX — Claude lit
   tous ces formats sans script). Produis ta reconnaissance dans
   reconnaissance.md (faits clés, secteur, demande, premiers red flags).
2. Décide quels fichiers chaque spécialiste doit recevoir, en fonction
   de ta reconnaissance. Tous les spécialistes ne lisent pas tout —
   par exemple, le XLSX financier n'est utile à finance et investment.
3. Round 1: dispatche en PARALLÈLE finance, market, investment, legal,
   risk (5 sub-agents en parallèle, dans le même tour). Chacun écrit
   son draft dans round1/{nom}.md.
4. Round 2 (critique): une fois les 5 drafts revenus, dispatche le
   challenger seul. Le challenger lit les 5 drafts et écrit critique.md
   avec ses remarques par spécialiste plus les contradictions
   transversales.
5. Round 3 (révision): redispatche en PARALLÈLE les 5 mêmes
   spécialistes. Chacun reçoit la critique COMPLÈTE (pas seulement la
   sienne) afin de voir les enjeux transversaux. Drafts révisés dans
   round2/{nom}.md.
6. Synthèse: assemble le mémo final en français dans
   analysis_{PROJECT_NAME}.md à la racine du repo. Termine par un
   verdict explicite (Go / Go conditionnel / No-go) et 3-5 questions
   ouvertes pour M. Chraibi.

Les spécialistes et le challenger sont dispo via Agent tool. Tous les
sous-agents tournent en parallèle quand c'est possible (Round 1 et
Round 3) — c'est important pour la latence.

Quand le mémo est prêt, retourne au tour principal un résumé de 5-8
lignes en français + le verdict.
```

Pass the absolute file paths and absolute workspace path so lead doesn't have to resolve them.

## Step 4 — Surface the result

When the lead returns:

1. Print lead's 5-8-line summary verbatim.
2. Add a confirmation block:

```
Mémo complet : {repo_root}/analysis_{PROJECT_NAME}.md
Verdict : {verdict}
Workspace intermédiaire : {repo_root}/analysis_{PROJECT_NAME}/
```

3. Switch to **standby mode**: tell the user the team is available for follow-up — *"L'équipe reste dispo. Tapez vos questions, demandes de brainstorm, ou relancez la skill sur un autre fichier."* From this point on, route follow-up questions back to the appropriate specialist (or to lead for cross-cutting questions) on demand. No need to redispatch the full pipeline for a single question.

## Native file parsing

This skill does NOT bundle any preprocessing scripts. Claude's Read tool handles PDF, DOCX, XLSX directly in the model environment — there is no init_dossier.py, no parse_bp.py. Lead and specialists open files via Read like any other text. If a file is too large for one Read call, page it (Claude knows how). If a PDF is image-only and produces no text, lead notifies the user in French and asks whether to (a) wait for a text version, (b) proceed without that source, or (c) abort.

## Parallelism

The two parallel phases are non-negotiable for latency reasons:
- **Round 1**: lead dispatches finance + market + investment + legal + risk in a single tool-use block (5 simultaneous Agent calls).
- **Round 3**: lead dispatches the same 5 specialists again in a single block, each receiving the full critique.

Challenger runs alone (single dispatch) since no other agent is doing related work in that phase. Lead is the only agent that does NOT run in parallel — it's the sequential orchestrator.

## Common errors

| Situation | French message |
|---|---|
| Aucun fichier candidat dans le repo | "Aucun fichier analysable détecté dans le répertoire courant. Indiquez-moi le chemin du BP." |
| Plusieurs candidats | "Plusieurs fichiers possibles : {liste}. Lequel est le BP principal ? Y a-t-il un Excel financier compagnon ?" |
| Workspace déjà présent | "`analysis_{nom}/` existe déjà. Voulez-vous (a) écraser, (b) renommer la nouvelle analyse, (c) annuler ?" |
| Agent manquant | "Erreur: équipe incomplète. Agents manquants: {liste}." |
| PDF scanné | "Attention : `{nom}.pdf` semble être un scan (aucun texte extrait). (a) attendre une version texte, (b) procéder sans ce fichier, (c) annuler ?" |
| Format inattendu | "Format `.{ext}` non reconnu. Confirmez que je dois quand même tenter de le lire ?" |

## What this skill is not

- It does not run the analysis itself — the skill is dispatch + bookkeeping. The thinking happens in the lead and specialists.
- It does not bundle Python scripts or external tools. Native file parsing only.
- It does not assume `~/Chraibi/Dossiers/` or any global directory. Everything stays inside the repo where the skill was invoked.
- It does not skip the user-confirmation step on file selection, even when there's only one obvious candidate. Every analysis run is explicit.
