---
description: Run the Chraibi analyst team on a business plan in the current directory — produces a French memo with verdict.
argument-hint: "[optional: file path or language hint, e.g. 'en anglais']"
---

Invoke the `analyse` skill on the current working directory.

Arguments (optional): $ARGUMENTS

Follow the skill exactly:
1. Pre-flight: verify the 7 subagents (`lead`, `finance`, `market`, `investment`, `legal`, `risk`, `challenger`) are all dispatchable. If any are missing, fail with the French error message specified in the skill — do not proceed.
2. Identify candidate file(s) in the cwd (PDF/DOCX/XLSX/etc.) and confirm the target with the user before starting.
3. Set up the `analysis_{PROJECT_NAME}/` workspace.
4. Dispatch the `lead` agent with the protocol prompt from the skill (passing absolute paths).
5. Surface lead's summary + verdict + memo path when it returns, then go to standby mode.

If `$ARGUMENTS` contains a language hint (e.g. "en anglais", "in English"), use that language for user-facing output; agents keep their French drafting.

If `$ARGUMENTS` contains an explicit file path, skip the candidate scan and use that file as the main BP (still ask the user to confirm any companion files).
