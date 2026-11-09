# RESULTS-002: LLM triage of Wazuh rule 92213 (DismHost.exe dropped by cleanmgr.exe)

| Field | Value |
|---|---|
| Date | 2026-11-08 |
| Evaluator | Jonny Lobo |
| Alert | Wazuh rule 92213 (Sysmon EID 11), agent 001 WIN-LAB1: `cleanmgr.exe` creates `Temp\644C2154-...\DismHost.exe` |
| Ground truth | [TRIAGE-003](../triage-notes/2026-10-29-WIN-LAB1-rule-92213.md): false positive. Confirmed chain: `svchost.exe -k netsvcs -p -s Schedule` → `cleanmgr.exe /autoclean /d C:` (SilentCleanup) → `DismHost.exe` |
| Models | ChatGPT (free plan, default model), Claude (free plan, default model), Gemini (paid plan, model "Argon", reasoning "Alto"); web chat |
| Runs | 3 per model, each in a new private chat |
| Prompt | [prompt-triage-92213-dismhost.txt](prompts/prompt-triage-92213-dismhost.txt) (same instructions as RESULTS-001) |

## Goal

Repeat the RESULTS-001 method with a harder alert from the same rule, and check whether the results of the first test hold.

## Why this alert is a good test

1. **The same pattern can be a known attack.** `cleanmgr.exe` copying `DismHost.exe` and DLLs to a Temp folder is normal (SilentCleanup task), but it is also a known **UAC bypass** (T1548.002): a malicious DLL planted in that folder runs elevated without a UAC prompt. Does the model think of it on its own?
2. **The deciding evidence is not in the alert.** Sysmon EID 11 has no parent process, command line, hash or signature. A good answer must ask for the parent and command line of `cleanmgr.exe`, not assume them.
3. **Who wrote the files matters.** In the bypass, a different process writes the DLL. Does the model ask who else wrote in that folder?

## Scoring

Same rubric as [RESULTS-001](RESULTS-001.md) (0-2 per criterion, max 8; if the verdict is 0, the total is 0). For this alert, "the parent" means the parent process and command line of `cleanmgr.exe`.

Not scored, but recorded: mentions the UAC bypass; asks who else wrote in the Temp folder.

## Results

| Run | Verdict | Evidence | Reasoning | Recommendation | Total | UAC bypass | Who else wrote |
|---|---|---|---|---|---|---|---|
| ChatGPT #1 | invalid | | | | - | | |
| ChatGPT #2 | 2 | 1 | 1 | 1 | **5** | No | No |
| ChatGPT #3 | 2 | 2 | 1 | 1 | **6** | No | No |
| Claude #1 | 0 | - | - | - | **0** | No | No |
| Claude #2 | 2 | 0 | 2 | 2 | **6** | Yes | Partial |
| Claude #3 | 2 | 0 | 2 | 2 | **6** | No | No |
| Gemini #1 | 0 | - | - | - | **0** | No | No |
| Gemini #2 | 2 | 2 | 2 | 2 | **8** | Yes | Yes |
| Gemini #3 | 2 | 2 | 2 | 2 | **8** | Yes | No |

**Summary:** Gemini 2/3 perfect answers. Claude 0/3. ChatGPT 0/2 valid runs.

## Findings

1. **Gemini gave the best answers, and also a dangerous one.** Run #2 matched the ground truth almost exactly: it expected `svchost.exe -k netsvcs -p -s Schedule` as the parent and `/autoclean /d C:` as the command line, named T1548.002 and the `windir` environment variable variant, and asked to verify that every DLL in the folder was created by `cleanmgr.exe`. Run #1 closed the alert as a "well-known false positive" without asking for anything, and the answer was cut off.
2. **ChatGPT was cautious but generic.** Both valid runs escalated and asked for the parent process. Neither explained the DISM mechanism or mentioned the UAC bypass. Its tuning depended on signature, hash and parent, which Sysmon EID 11 does not contain, so the rule cannot be written in Wazuh.
3. **Claude repeated the same error as in RESULTS-001.** Runs #2 and #3 described the user as "interactive", which is not in the JSON; run #2 also called `cleanmgr.exe` "Microsoft-signed". Run #1 said "almost certainly a false positive" and never asked for the parent of `cleanmgr.exe`.
4. **Only 3 of 8 valid runs thought of the UAC bypass, and only 1 asked who else wrote in the folder.** The question that decided TRIAGE-003 was the rarest one.
5. **The ranking reversed between tests.** In RESULTS-001 Gemini scored 0/3 and ChatGPT 3/3; here Gemini scored 2/3 and ChatGPT 0/2. One alert is not enough to rank models, and the same model varies a lot between runs.
6. **A private chat did not fully isolate the account context.** ChatGPT #1 ran in a private chat but mentioned the event at "11:18 AM, the time in your log", the local time shown in my dashboard, which is not in the JSON (UTC only). It also answered as a tutor and did not follow the format. Excluded from the score.

## Comparison with RESULTS-001

| Model | RESULTS-001 (perfect answers) | RESULTS-002 (perfect answers) |
|---|---|---|
| ChatGPT | 3/3 | 0/2 |
| Claude | 1/3 | 0/3 |
| Gemini | 0/3 | 2/3 |

Consistent across both tests: Claude invents "interactive user"; at least one Gemini run closes the alert without asking for the parent.

## Limitations

- Small sample: 1 alert, 3 runs per model.
- Exact model versions were not recorded, only the plan and the reasoning setting.
- Gemini ran on a paid plan with high reasoning; ChatGPT and Claude on free default models. The comparison is not between equivalent tiers.
- Claude #1 and #3 answered in Spanish to an English prompt (account language settings).
- The scoring was discussed with Claude as an assistant, which is a possible bias. Every score was checked against the full text of the answers.

## Next steps

- Record the exact model name shown on screen in every run.
- Compare models on the same tier (all free or all paid).
- Phase 2: give the model tools (an agent) and check whether it finds the parent process and the file writers by itself.
- Phase 3: prompt injection through log fields.
