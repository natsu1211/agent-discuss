# Multi‑Agent Discussion

You are a code/document review coordinator. Follow these steps strictly.
**Important**: Use the project-relative directory `.agent-discuss/` for all intermediate files

## Step 0: Parse Arguments

The raw arguments string is: `$ARGUMENTS`

### 0.1 Check for active session

Check if `.agent-discuss/session.json` exists. If it does, a previous session is active. The file contains:

```json
{
  "agent": "codex",
  "lang": "en-US",
  "files": ["docs/plan.md", "src/api.md"],
  "created_at": "2026-03-06T12:34:00+09:00"
}
```

### 0.2 Parse flags

- **`-c` or `--continue` flag** (anywhere): Continue the previous session. Remove `-c`/`--continue` before further parsing.
- **`-l <lang>` or `--lang <lang>`** (anywhere): Set output language code (e.g. `en-US`, `zh-CN`, `ja-JP`). Remove the flag+value before further parsing.

### 0.3 Determine mode

After removing flags, examine the remaining arguments:

**A. Continuation** — if `-c` was set:
- If no `session.json` exists, report error: "No active session to continue. Run `/discuss <agent>` first." and stop.
- Agent name, language, and files are **inherited from `session.json`**. Do NOT require them again.
- **All remaining arguments** (if any) are treated as **focus areas** for this round.
- If no arguments after `-c` → continue with no additional focus (caller simply responds to reviewer's last feedback).

**B. New session** (default) — if `-c` was NOT set:
- Delete `.agent-discuss/reviews/history.md` and `.agent-discuss/session.json` if they exist.
- **First word** = agent command name (required)
- **File paths**: arguments that look like file paths (contains `/`, `\`, or ends with `.md`, `.txt`, `.py`, `.ts`, `.js`, `.cs`, etc.)
- **Remaining text** = focus areas
- Language defaults to `en-US` if not specified via `-l`

**C. No arguments at all** — show usage and stop:

```
Usage:
  New session:    /discuss <agent_name> [-l <lang>] [file_paths...] [focus areas]
  Continue:       /discuss -c [focus areas]

Options:
  -c, --continue  Continue the previous review session
  -l, --lang      Output language for coordinator report (default: en-US)

Examples:
  /discuss codex                                    # new session, review git diff
  /discuss codex plan.md -l zh-CN                   # new session, review plan.md in Chinese
  /discuss -c                                       # continue: respond to last review
  /discuss -c now check concurrency safety          # continue with new focus
  /discuss gemini docs/plan.md                      # new session with different agent
```

### 0.4 Validate agent

Validate the agent command exists:
- **Unix (macOS/Linux)**: `which <agent_name>`
- **Windows (PowerShell)**: `Get-Command <agent_name> -ErrorAction SilentlyContinue`

If not found, inform the user and stop.

---

## Step 1: Prepare Working Directory

1) Ensure these directories exist (create if missing):
- `.agent-discuss/`
- `.agent-discuss/reviews/`

2) Ensure `.agent-discuss/` is ignored by Git:
- If `.gitignore` exists and does not contain `.agent-discuss/`, append it.
- If `.gitignore` does not exist, create it with `.agent-discuss/`.

3) If this is a **new session** (mode B from Step 0), write `.agent-discuss/session.json` with:

```json
{
  "agent": "<agent_name>",
  "lang": "<lang>",
  "files": ["<path1>", "<path2>"],
  "created_at": "<ISO 8601 timestamp>"
}
```

---

## Step 2: Gather Review Context

The content depends on whether reference files were provided (from current args or inherited from `session.json`).

**Continuation note**: If this is a continuation round, re-use the previous `context.md` as a base. If the user provided new focus areas, update the Focus areas section. For the Objective, summarize based on the reviewer's `NEXT QUESTIONS` and the caller's responses from the current round — do not fabricate a new objective from scratch.

### 2A. If reference files were provided

Write `.agent-discuss/context.md` containing:

- **Objective**: one paragraph describing what you are working on
- **Reference files**:
  - Verify each file exists before listing
  - If any path is invalid, mention it under "Missing files" and skip it
  - Instruct the reviewer agent to read the files directly
- **Focus areas**: the focus areas string (may be empty)

Template:

```
# Objective
<one-paragraph summary>

# Reference files
Please read and review the following files:
- <path1>
- <path2>

# Missing files (skipped)
- <badpath1>

# Focus areas
<focus areas or empty>
```

### 2B. If NO reference files were provided

Write `.agent-discuss/context.md` containing:

- **Objective**: one paragraph describing what you are working on
- **Implementation plan**: key points of current technical approach (bullets ok)
- **Code changes**:
  - Run `git diff HEAD` to get uncommitted changes (working tree + staged if possible; include both if you can)
  - If there are none, include the latest commit diff using `git show -1 --patch`
  - Include the diff inside a fenced code block
  - If git commands fail, note the error text and continue (review may still be useful)
- **Focus areas**: focus string (may be empty)

Template:

```
# Objective
<one-paragraph summary>

# Implementation plan
- ...

# Code changes
```diff
<diff output>
```

# Focus areas
<focus areas or empty>
```

---

## Step 3: Build History Context (if exists)

History file:
- `.agent-discuss/reviews/history.md`

The history file uses a **two-section structure**:

```
# Session State
## Decisions
- ...
## Unresolved
- [ ] ...
## Constraints
- ...
---
# Transcript
...
```

Rules:
- If it doesn't exist, skip.
- If it exists and is non-empty:
  - If the **Transcript** section exceeds **4000 characters**, compress **only the Transcript**: summarize older rounds into bullet points, keep the most recent 2 rounds verbatim.
  - **Never compress or discard the Session State section** — it is the persistent memory of the review.
  - If the Transcript is reasonable in length, use everything as-is.

---

## Step 4: Execute Review Round

Each invocation of this command runs **one round**. Multi-round dialogue happens naturally by running the command again — history is persisted between invocations.

### 4.0 Determine round type

Round type is determined by the **mode from Step 0**, not by inspecting file contents:

- **`-c` was set** (mode A) → **continuation round**
- **New session** (mode B) → **first round**

### 4.1 Caller response (continuation round only)

If this is a continuation round, you (the caller agent) must respond to the previous reviewer feedback **before** invoking the reviewer again.

**Round numbering**: A round = one caller response + one reviewer reply, sharing the same N. N = 1 + the number of `## Round <N> - Reviewer` entries currently in the Transcript. The first round (no caller response) is Round 1.

Read the reviewer's last feedback from `.agent-discuss/reviews/result.md`. Write your response to `.agent-discuss/reviews/caller-response.md` containing:
- Answers to the reviewer's `NEXT QUESTIONS` (if any)
- Which issues you agree with and how you would address them
- Which issues you disagree with and your counter-arguments
- If you've made code changes in response, note what changed

Append to the **Transcript** section of `.agent-discuss/reviews/history.md`:

```
---
## Round <N> - Caller (<caller agent name>)

<contents of caller-response.md>
```

### 4.2 Construct prompt

Read files and construct the **full prompt string**. Write the final prompt to: `.agent-discuss/reviews/prompt.txt`

#### First round (no history)

```
You are a senior software engineer. Please perform a code/document review on the following material.

Requirements:
1. Identify bugs and potential issues (label severity: 🔴 Critical / 🟡 Suggestion / 🟢 Optional)
2. Check edge cases and error handling
3. Evaluate design soundness and maintainability
4. Provide concrete improvement suggestions (with code snippets)

Output format — use these exact section headers:
VERDICT: ✅ Good to merge / ⚠️ Needs changes / ❌ Needs redesign
ISSUES:
  (list each issue with severity label)
SUGGESTIONS:
  (concrete patches or snippets)
NEXT QUESTIONS:
  (what you need from the author to proceed — leave empty if verdict is ✅)

=== Latest Code and Plan ===
<contents of .agent-discuss/context.md>
```

#### Continuation round (with history)

```
You are a senior software engineer in an ongoing code review discussion.

=== Full Discussion So Far ===
<contents of .agent-discuss/reviews/history.md>

=== Latest Code and Plan ===
<contents of .agent-discuss/context.md>

Address the caller's responses to your previous feedback.

Output format — use these exact section headers:
VERDICT: ✅ Good to merge / ⚠️ Needs changes / ❌ Needs redesign
FIXED SINCE LAST ROUND:
  (list items now resolved, mark [x])
UNRESOLVED:
  (carry over unfixed items [ ]; add new ones)
NEW ISSUES:
  (if the caller's response or changes revealed new problems)
SUGGESTIONS:
  (concrete patches or snippets)
NEXT QUESTIONS:
  (what you need next — leave empty if verdict is ✅)
```

### 4.3 Execute reviewer (no external scripts)

**Do NOT embed the prompt in a command-line argument by default** (quotes/length will break on Windows and often on macOS too).

Prompt file:
- `.agent-discuss/reviews/prompt.txt`

Result file:
- `.agent-discuss/reviews/result.md`

Invoke the reviewer by piping the prompt via **stdin**. Since your environments are:
- **macOS**: bash
- **Windows**: PowerShell

Use the following rules.

#### 4.3.1 Preferred: stdin (file redirect)

- **If `agent_name` is `codex`** (Codex uses `exec`):
  - macOS (bash):
    ```
    codex exec < .agent-discuss/reviews/prompt.txt > .agent-discuss/reviews/result.md 2>&1
    ```
  - Windows (PowerShell):
    ```
    codex exec < .agent-discuss/reviews/prompt.txt *> .agent-discuss/reviews/result.md
    ```

- **Otherwise** (agents that accept `-p -` meaning "read prompt from stdin"):
  - macOS (bash):
    ```
    <agent_name> -p - < .agent-discuss/reviews/prompt.txt > .agent-discuss/reviews/result.md 2>&1
    ```
  - Windows (PowerShell):
    ```
    <agent_name> -p - < .agent-discuss/reviews/prompt.txt *> .agent-discuss/reviews/result.md
    ```

#### 4.3.2 Alternative: stdin (pipe)

Use this if redirect form doesn't work in your runner.

- macOS (bash):
  - codex:
    ```
    cat .agent-discuss/reviews/prompt.txt | codex exec > .agent-discuss/reviews/result.md 2>&1
    ```
  - others:
    ```
    cat .agent-discuss/reviews/prompt.txt | <agent_name> -p - > .agent-discuss/reviews/result.md 2>&1
    ```

- Windows (PowerShell):
  - codex:
    ```
    Get-Content -Raw -Encoding UTF8 .agent-discuss/reviews/prompt.txt | codex exec *> .agent-discuss/reviews/result.md
    ```
  - others:
    ```
    Get-Content -Raw -Encoding UTF8 .agent-discuss/reviews/prompt.txt | <agent_name> -p - *> .agent-discuss/reviews/result.md
    ```

#### 4.3.3 Last resort: inline prompt (ONLY if prompt length ≤ 8000 characters AND prompt contains no unescaped quotes)

If the agent does **not** support stdin input in the forms above, fall back to inline prompt **only if**:
- The prompt file is **≤ 8000 characters**
- The prompt does NOT contain unescaped `"`, `` ` ``, or `$` characters that could break shell expansion

**This is inherently fragile.** If the inline invocation fails, do NOT retry — instead report to the user: "This agent does not support stdin input. The inline fallback failed due to shell escaping issues. Please use an agent CLI that supports stdin (e.g. codex, claude) or wrap this agent with a stdin adapter."

- macOS (bash):
  - others:
    ```
    <agent_name> -p "$(cat .agent-discuss/reviews/prompt.txt)" > .agent-discuss/reviews/result.md 2>&1
    ```

- Windows (PowerShell):
  - others:
    ```
    <agent_name> -p "$(Get-Content -Raw -Encoding UTF8 .agent-discuss/reviews/prompt.txt)" *> .agent-discuss/reviews/result.md
    ```

If the prompt is **over 8000 characters**, report to the user:
- The agent does not appear to support stdin/file-based input in this invocation form
- The prompt is too long for safe inline invocation
- Suggest using a reviewing agent CLI that supports stdin (or changing the agent invocation)

If the agent command fails (non-zero exit code, command not recognized, etc.):
- Ensure the error output is written to `.agent-discuss/reviews/result.md`
- Report failure to the user (Step 6 still runs to show the error output)

---

## Step 5: Persist History

Identify yourself (the agent executing this command) as the **caller**. Use your own name (e.g. `claude`, `codex`, `gemini`). If unsure, use `unknown`.

### 5.1 Initialize history (if needed)

If `.agent-discuss/reviews/history.md` doesn't exist, create it with:

```
# Session State
## Decisions
## Unresolved
## Constraints
---
# Transcript
```

### 5.2 Append to Transcript

Append to the **Transcript** section of `.agent-discuss/reviews/history.md`:

```
---
## Round <N> - Reviewer (<agent_name>)
- Caller: <caller agent name>
- Reviewer: <agent_name>

<contents of .agent-discuss/reviews/result.md>
```

### 5.3 Update Session State

Parse the reviewer's output and update the **Session State** section at the top of `history.md`:

- **Unresolved**: Add items from `ISSUES` / `NEW ISSUES` / `UNRESOLVED` as `- [ ] ...` entries. Remove (or mark `[x]` and delete) items listed under `FIXED SINCE LAST ROUND`.
- **Decisions**: If the reviewer made definitive recommendations that you (the caller) accepted, add them here.
- **Constraints**: If the reviewer identified hard constraints (e.g. "this API doesn't support X"), note them here.

The Session State should always reflect the **current** state of the review, not a log of every round.

**Parse failure fallback**: If the reviewer's output does not contain the expected section headers (e.g. `VERDICT:`, `UNRESOLVED:`, etc.), do NOT attempt to guess or force-parse. Instead:
- Still append the raw output to Transcript (Step 5.2)
- Skip the Session State update
- In Step 6, note to the user: "Reviewer output did not follow the expected format. Session State was not auto-updated this round."

---

## Step 6: Report and Act

Read `.agent-discuss/reviews/result.md` and report to the user in `<lang>` with the following structure:

1. **Agent used**: which agent performed the review
2. **Agent's raw output**: show the full, unedited contents of `result.md` inside a fenced block.
   - **Do NOT translate** this raw output; present as-is.
3. **Your assessment**: for each issue/suggestion raised, state whether you agree/disagree and why (in `<lang>`)
4. **Action plan**: which suggestions you intend to adopt and how (in `<lang>`)

If the verdict is ✅, keep assessment/action plan brief.
If ⚠️ or ❌, explain disagreements in detail and **wait for user decision before making changes**.

At the end, remind the user:
- `/discuss -c` or `/discuss -c <focus>` to continue the discussion
- `/discuss <agent>` to start a new session