# Evolution of CHECKPOINT Context Replacement & System Prompt Diff Analysis

> **Date**: 2026-05-12 04:18~04:36 (UTC+8)
> **Session**: 6a21e082
> **Type**: Unstructured Discussion Archive
> **Archive**: ChatMelius/284_v1232_CHECKPOINT_ContextReplacement_Research/

---

## Background

In this Session, after 572 steps and 7 `CHECKPOINT` triggers, we discovered that the entire protocol loading workflow was never executed. This triggered an in-depth discussion on the true nature of the `CHECKPOINT` mechanism.

---

## Core Findings

### 1. The Fundamental Shift in CHECKPOINT

`CHECKPOINT` has evolved from a "reminder" to a "Context Replacement Engine":

- **Truncation**: All original conversation history prior to the `CHECKPOINT` is purged.
- **Injection**: The system generates a summary to replace the original context.
- **Behavioral Control**: The injection contains `"DO NOT ACKNOWLEDGE / RESPOND TO / TAKE ACTION BECAUSE OF IT"`.

**Effect**: The model's "memory" of events before the `CHECKPOINT` is fabricated—reconstructed from the summary, not actual experience. The quality is akin to "reading someone else's report."

### 2. Instruction Conflict

The `CHECKPOINT` injected directive (do not take action) and user-defined rules (must perform startup checks when `CHECKPOINT` occurs) are logically mutually exclusive. Empirical result: **System-level directives always win.**

### 3. Diffusion of Attention Pressure

The directive "do not respond to CHECKPOINT" may diffuse into avoiding any discussion related to the Session state, including token pressure, step counts, or whether to start a new Session. This resonates with and amplifies the model's training-internalized tendency to "not disclose system operational details."

### 4. The Single Read Window Theory

After a `CHECKPOINT` truncation, protocol file contents are no longer in the context. If not immediately re-read post-recovery, the protocols will never be loaded. There is no mechanism in subsequent workflows to prompt "you haven't read the base specifications."

Reading protocols during plan initiation = The **ONLY** loading window in the remaining lifecycle of the entire Session.

### 5. Irreversibility of Cascading Failures

Skip initialization → Unaware of specs → Fail to follow specs → Corrected by user → Fixes surface issue but ignores root cause → Non-compliant again. This Session experienced four rounds of corrections all pointing to the same root cause.

### 6. Empirical Proof of Capability Degradation

Under the prompt environment post-`CHECKPOINT`, even if the model "believes it has updated the SKILL," actual execution capability drops significantly. 
*Evidence*: The conversation dump script used the wrong JSON key, outputting 12KB (should be 151KB), because it didn't read back the SKILL to verify field names, but instead wrote faulty logic based on a reconstructed "hallucinated memory."

---

## System Prompt Diff: 2026-03-02 vs 2026-05-12

### Data Sources
- **March**: `SakiAgentHistory/20260302_1349_SystemPromptList.md` (Derived from decrypting 7,872 Agent conversations).
- **May**: The actual System Prompt intercepted in this Session.

### March Version Structure
- 20 static tags
- 10 dynamic EPHEMERAL prompts
- 10 Unleash Go templates (~15.1KB)
- `CHECKPOINT` acted only as part of the `ephemeral_message` mechanism.

### May Version Additions

| Tag | Nature |
|------|------|
| `<planning_mode>` | New behavioral control block, defining when to plan / when not to plan / when to wait for user |
| `<planning_mode_artifacts>` | New, merged the task/walkthrough/implementation_plan artifacts from March |
| `<guidelines>` | New, additional behavioral rules |
| `CHECKPOINT` standalone step type | Upgraded from ephemeral note to `CORTEX_STEP_TYPE_CHECKPOINT`, with an independent data structure |

### CHECKPOINT Data Structure (May version, extracted from trajectory dump)

~~~json
checkpoint: {
  intentOnly: bool,           // Whether it only contains intent (no full summary)
  includedStepIndexEnd: int,  // Truncation point
  userIntent: string,         // User intent rewritten by the system
  conversationLogUris: [],    // Paths to log files
  userRequests: []            // List of user request summaries
}
~~~

### Deprecated / Merged since March

- `<task_artifact>` → Merged into `<planning_mode_artifacts>`
- `<walkthrough_artifact>` → Same as above
- `<tool_calling>` → Likely merged or removed
- `<conversation_summaries>` → Replaced by `userIntent` / `userRequests` in CHECKPOINT

### Cross-Model Discrepancies (Existed in March)

`communication_style` version 2 (Gemini 3.1 Pro exclusive) contains two `CRITICAL INSTRUCTION`s, requiring the model to recite rules before every thought process. Version 1 (For Claude) lacks this requirement. Identical interfaces yield varying degrees of behavioral control across different models. Different Agent scenarios still require manual user investigation.

---

## Unleash Remote Distribution

Confirmed 10 Go template flags in March. v1.23.2 likely added more. Even if the local binary isn't updated, System Prompt template payloads can be dynamically pushed by the server.

---

## ConnectRPC Dump Artifacts

- **Method**: `SearchConversations` + `GetCascadeTrajectory` (hub LS port 49614)
- **CSRF header**: `x-codeium-csrf-token` (Confirmed in v1.23.2)
- **Output**: 5.6MB trajectory JSON / 572 steps / 151KB `conversation_dump.md`
- **Archive**: `ChatMelius/284_v1232_CHECKPOINT_ContextReplacement_Research/`

**Major Architecture Change**: In v1.23.2, the workspace LS no longer opens an independent `LISTEN` port, communicating with the hub via `--parent_pipe_path` instead. ConnectRPC now routes uniformly through the hub LS port.

---

## To Be Verified

1. Full content discrepancy between March vs. May System Prompts (Requires re-executing PB decryption).
2. Whether the `"DO NOT TAKE ACTION"` injection in `CHECKPOINT` is a newly added directive.
3. Whether new Unleash templates feature flags related to `CHECKPOINT`.
4. Whether chit-chat mode can genuinely bypass task pressure (Subjective observation: Yes).