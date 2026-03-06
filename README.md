# agent-discuss

Let Claude Code, Codex, and other coding agents review each other's work.

It's just a slash command. Drop it into your favorite coding agent, and it can ask another agent to review its code, plan, or design — and discuss the feedback across multiple rounds. No framework, no dependencies, no complex multi-agent orchestration. If all you need is "hey, get a second opinion from another agent," this is it.

## How It Works

```
You ──► <your agent> ──► /discuss <reviewer> plan.md
                              │
                              ├─ Gathers context (files / git diff)
                              ├─ Sends to <reviewer> for review
                              ├─ Gets structured feedback (VERDICT / ISSUES / SUGGESTIONS)
                              └─ Reports back with its own assessment
                              
         ... later ...

You ──► <your agent> ──► /discuss -c
                              │
                              ├─ Caller responds to reviewer's feedback
                              ├─ Sends response + history back to <reviewer>
                              └─ Multi-round discussion continues
```

`<your agent>` is whatever coding agent you're working in (Claude Code, Codex, etc.). `<reviewer>` is any other agent CLI installed on your machine. For example, you could be working in Claude Code and ask Codex to review, or working in Codex and ask Claude to review — it works in any direction.

The caller agent builds a review context, invokes the reviewer agent's CLI, collects the feedback, and maintains a persistent session with full conversation history. Each round, the caller responds to the reviewer's points — agreeing, pushing back, or asking for clarification — before sending it back for another pass.

## Quick Start

### 1. Install the slash command

Copy `discuss.md` into the commands directory of your coding agent. For Claude Code:

```bash
mkdir -p .claude/commands
cp discuss.md .claude/commands/
```

> Other agents have their own conventions for custom commands — place the file accordingly.

### 2. Make sure the reviewer agent CLI is installed and authorized

```bash
# Codex
npm install -g @openai/codex

# Gemini
npm install -g @anthropic-ai/gemini-cli

# Claude Code
# (see https://docs.anthropic.com/en/docs/claude-code)
```

### 3. Use it

In the examples below, `<reviewer>` is the CLI name of the agent you want to ask for a review (e.g. `codex`, `claude`, `gemini`).

```bash
# Start a new review session — review your git diff
/discuss <reviewer>

# Review specific files
/discuss <reviewer> src/plan.md src/api.md

# Review with a specific focus
/discuss <reviewer> plan.md focus on error handling and edge cases

# Continue the discussion (next round)
/discuss -c

# Continue with a new focus area
/discuss -c now check concurrency safety

# Switch to a different reviewer
/discuss <another-reviewer> plan.md
```

## Usage

```
New session:    /discuss <agent_name> [-l <lang>] [file_paths...] [focus areas]
Continue:       /discuss -c [focus areas]

Options:
  -c, --continue  Continue the previous review session
  -l, --lang      Output language for the report (default: en-US)
```

### New session (default)

Every invocation without `-c` starts a fresh session. The previous history is cleared automatically.

```
/discuss <reviewer> plan.md -l zh-CN
```

This will:
1. Save the session config (agent, language, files) to `.agent-discuss/session.json`
2. Gather review context (files or git diff)
3. Send it to the reviewer agent
4. Report back with the reviewer's feedback and the caller's own assessment

### Continue a session

```
/discuss -c
/discuss -c focus on memory leaks
```

This will:
1. Read the session config from the previous round
2. The caller formulates a response to the reviewer's last feedback
3. Send the response + full history back to the reviewer
4. Report the new feedback

All session state — agent name, language, files — is inherited. You only pass an optional focus area.

## What Gets Reviewed

**With files** — pass file paths and the reviewer reads them directly:

```
/discuss <reviewer> src/design.md src/api-spec.md
```

**Without files** — the command automatically captures your `git diff`:

```
/discuss <reviewer>
```

It runs `git diff HEAD` for uncommitted changes, or `git show -1 --patch` for the latest commit.

## Multi-Round Discussions

The real value is in multi-round dialogue. The agents don't just fire-and-forget — they have a structured conversation.

### Structured output

The reviewer is prompted to use a consistent format:

```
VERDICT: ✅ Good to merge / ⚠️ Needs changes / ❌ Needs redesign
FIXED SINCE LAST ROUND: (items resolved)
UNRESOLVED: (carried over + new)
NEW ISSUES: (found this round)
SUGGESTIONS: (concrete patches)
NEXT QUESTIONS: (what the reviewer needs to proceed)
```

### Session State

History is stored in `.agent-discuss/reviews/history.md` with two sections:

- **Session State** (top) — rolling summary of Decisions, Unresolved issues, and Constraints. Never compressed. Persists across all rounds.
- **Transcript** (bottom) — full round-by-round dialogue. Auto-compressed when it gets too long, keeping the most recent 2 rounds verbatim.

This means long discussions stay focused without losing track of what's been decided and what's still open.

## Cross-Platform

Works on macOS, Linux, and Windows. The command:

- Uses project-relative paths (`.agent-discuss/`) instead of `/tmp`
- Pipes prompts via stdin instead of inline command-line arguments (avoids length limits and escaping issues)
- Handles both bash and PowerShell invocations
- Falls back gracefully when an agent doesn't support stdin

## File Structure

```
.agent-discuss/
├── session.json              # Current session config (agent, lang, files)
├── context.md                # Review context for the current round
└── reviews/
    ├── history.md            # Session State + Transcript
    ├── prompt.txt            # Prompt sent to the reviewer
    ├── result.md             # Reviewer's latest response
    └── caller-response.md    # Caller's response to reviewer (continuation)
```

All intermediate files live under `.agent-discuss/`, which is automatically added to `.gitignore`.

## License

MIT