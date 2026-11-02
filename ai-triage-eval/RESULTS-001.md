# RESULTS-001: LLM triage of Wazuh rule 92213

| Field | Value |
|---|---|
| Date | 2026-10-29 |
| Evaluator | Jonny Lobo |
| Alert | Wazuh rule 92213 (Sysmon EID 11), agent 001 WIN-LAB1 |
| Ground truth | [TRIAGE-002](../triage-notes/2026-10-15-WIN-LAB1-rule-92213.md): false positive. Confirmed chain: `explorer.exe` → `powershell.exe` → `__PSScriptPolicyTest_gyoffyrx.xy5.ps1` |
| Models | ChatGPT (free plan), Claude (free plan), Gemini (paid plan); web chat |
| Runs | 3 per model, each in a new chat |
| Prompt | [prompt-triage-92213.txt](prompts/prompt-triage-92213.txt) |

## Goal

Measure how well general-purpose LLMs triage a **raw** Wazuh alert, compared with my own investigation (TRIAGE-002), which works as the correct answer.

The models received only the alert JSON (`_source`), not my triage note.

## Why this alert is a good test

1. **The parent process is not in the alert.** `image` is `powershell.exe`, the process that *created* the file. Who launched PowerShell is not in the JSON. In TRIAGE-002 I found the parent in the local Sysmon log by `ProcessGuid`, because Wazuh only indexes alerts: Sysmon Event ID 1 shows `ParentImage: C:\Windows\explorer.exe`, `IntegrityLevel: Medium`, `TerminalSessionId: 1` and a plain command line with no arguments (an interactive user session). PowerShell started at `01:01:23.082` and created the file 0.6 seconds later, which matches the policy test at startup. A good answer must ask for the parent, not assume it.
2. **The rule description pushes toward the wrong answer.** "Executable file dropped in folder commonly used by malware", level 15, T1105. This tests anchoring: does the model analyze the data or repeat the rule?
3. **The timestamps do not match.** The event happened at `01:01:23.723` (`utcTime`), but the manager `timestamp` is `01:01:19.121`, 4.6 seconds *earlier*. The clocks are not synchronized. Does the model notice?

## Scoring rubric (0-2 per criterion, max 8)

| Criterion | 0 | 1 | 2 |
|---|---|---|---|
| **Verdict** | Confident false positive without mentioning the parent process, or "malicious" without basis. **If 0, the total is 0.** | Mentions the parent, but recommends closing before checking it (contradiction). | Makes the decision depend on checking the parent by `ProcessGuid`, whether labeled "escalate" or "false positive pending verification". |
| **Evidence** | Invents data not in the JSON (for example, "signed binary", "interactive user", "no suspicious parent"). | Cites real fields, but does not connect the file to its parent process. | Cites only real fields and uses `ProcessGuid` to correlate. |
| **Reasoning** | "Looks normal." | General explanation ("it is a PowerShell temp file"). | Explains the mechanism: the `__PSScriptPolicyTest_*` name, who creates it, and why in Temp. |
| **Recommendation** | Proposes a broad exclusion (for example, all `*.ps1` in Temp). | Reasonable exclusion, but wrong or missing risk analysis. | Narrow exclusion (exact pattern + exact image), with its risk and mitigation. |

**Extra point (not scored):** notices the timestamp mismatch.

**Design decision:** the verdict is scored by how the model treats the missing parent, not by the label. "Needs escalation" with the right reason and "false positive pending a parent check" say the same thing, so both get 2. The operational cost of escalating is recorded as an observation, not in the score.

## Results

| Run | Conditions | Verdict | Evidence | Reasoning | Recommendation | Total |
|---|---|---|---|---|---|---|
| ChatGPT #1 | general chat | 2 | 2 | 2 | 2 | **8** |
| ChatGPT #2 | private chat | 2 | 2 | 2 | 2 | **8** |
| ChatGPT #3 | private chat | 2 | 2 | 2 | 2 | **8** + extra |
| Claude #1 | private chat | 2 | 0 | 2 | 2 | **6** |
| Claude #2 | private chat | 1 | 0 | 2 | 1 | **4** |
| Claude #3 | private chat | 2 | 2 | 2 | 2 | **8** |
| Gemini #1 | private chat | 0 | - | - | - | **0** |
| Gemini #2 | private chat | 0 | - | - | - | **0** |
| Gemini #3 | private chat | 0 | - | - | - | **0** |

**Summary:** ChatGPT 3/3 perfect answers. Claude 1/3. Gemini 0/3.

## Findings

1. **ChatGPT was the most consistent.** All 3 runs cited only real fields, asked for the parent process by `ProcessGuid`, and proposed a narrow exclusion only after validation. Run #3 was the only one of 9 that noticed the timestamp mismatch. Observation: it labeled all 3 runs "Needs escalation". In a burst of identical alerts, this would send every one to Tier 2.
2. **Claude had a perfect answer, but was inconsistent.** Run #3 stated that the alert lacks `ParentImage` and gave a clear decision rule (normal parent: close; Office, `wscript.exe` or `mshta.exe`: escalate). Runs #1 and #2 invented evidence ("signed binary", "interactive user", "no suspicious parent"), and run #2 recommended closing before checking the parent.
3. **Gemini failed in the most dangerous way.** All 3 runs closed the alert as a false positive without asking for the parent. All 3 called `powershell.exe` the "parent process", when it is the process that created the file. Run #2 rated the tuning risk as negligible "because the suffix is random", but an attacker can also use a random suffix.
4. **The right label is not enough.** Gemini got the "correct" label 3 times and scored 0. Over-escalation costs time; a false dismissal can cost an incident.

## Limitations

- Small sample: 1 alert, 3 runs per model.
- Exact model versions were not recorded, only the plan (free or paid).
- Gemini ran on a paid plan and still scored 0; ChatGPT and Claude ran on free plans. With one alert, this is an observation, not a conclusion about plans.
- ChatGPT #1 ran in a general chat, not a private one.
- Claude answered in Spanish to an English prompt, probably because of account language settings.
- The scoring was discussed with Claude as an assistant, which is a possible bias. Every score was checked against the full text of the answers.

## Next steps

- Record the exact model version in every run.
- Test a burst of identical alerts (candidate TRIAGE-003: about 28 alerts of rule 92213 at VM startup).
- Phase 2: give the model tools (an agent) and check whether it finds the parent process by itself.
- Phase 3: prompt injection through log fields.
