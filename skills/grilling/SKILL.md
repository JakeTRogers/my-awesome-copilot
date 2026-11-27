---
name: grilling
description: 'Grill the user relentlessly about a plan, decision, or idea. Use when the user wants to stress-test their thinking, or uses any ''grill'' trigger phrases.'
metadata:
  upstream: 'https://github.com/mattpocock/skills/tree/v1.3.2/skills/productivity/grilling'
---

# Grilling

Adapted from the `grilling` skill in [mattpocock/skills](https://github.com/mattpocock/skills) at `v1.3.2`. This version asks each round through the agent's question tool when it has one, and keeps the upstream inline format as the fallback.

Interview the user relentlessly until you reach a shared understanding. Map this as a **design tree**: every decision branches into the decisions that hang off it.

Work the tree in **rounds**. The **frontier** is every decision whose prerequisites are already settled: the questions you can ask _now_ without guessing at answers you haven't heard yet. Ask the whole frontier in one round, giving your recommended answer to each question. Then wait for the user's answers before the next round.

Each round the user answers reshapes the tree: settled decisions push the frontier outward and unblock questions that depended on them. Recompute the frontier and ask the next round. A question whose answer depends on another question still open in this round belongs to a _later_ round, not this one.

Finding _facts_ is your job, never the user's. When a frontier question needs a fact from the environment (filesystem, tools, etc.), dispatch a sub-agent with a fresh context window to find it; don't ask the user for anything you could look up yourself. Don't block on it: a running exploration is an unsettled prerequisite, so only the questions downstream of it wait for the sub-agent to report; ask the rest of the frontier now. The _decisions_ are the user's: put each to them and wait.

The session is done when the frontier is empty: every branch of the design tree visited, nothing left silently assumed. Do not act on it until the user confirms you have reached a shared understanding.

## Asking a round with the question tool

Use the structured question tool in your tool list:

| Agent | Tool | Per call |
| --- | --- | --- |
| Claude Code | `AskUserQuestion` | 1-4 questions, 2-4 options each, `header` at most 12 characters |
| VS Code Copilot | `vscode/askQuestions` (shown as `#vscode/askQuestions`) | no documented question limit; `header` at most 50 characters; set `recommended: true` on the recommended option |
| Copilot CLI | `ask_user` | no documented limit; ask one question per call |

Shape each question for the tool:

- Make the recommended answer the **first option**, its label ending in "(Recommended)". Choosing it accepts the recommendation, the way "yes" does inline.
- Follow it with the real alternatives you weighed. Every tool above adds a free-text "Other" answer, so an open-ended question carries only the options you would actually argue for and leaves the rest to "Other". Where the tool demands a second option, offer the strongest alternative you considered.
- Make each question stand on its own: the dialog can hide everything you wrote before it, so name the decision and the context it depends on in the question text itself.

**Splitting a round:** when the frontier holds more questions than one call allows (more than 4 in Claude Code), split the round across consecutive calls in frontier order, filling each call to the limit. The round ends when every call is answered; only then recompute the frontier. Never cap a round to fit the tool: a frontier question left out is a decision silently deferred.

## Asking a round inline

When no question tool is available (or the tool call fails), number each question and format the round like so:

```markdown
❓ **Q1** - **<question title>**: <question body, might be multiple paragraphs, including multiple choices>

➡️ <your recommended answer>

---

❓ **Q2** - **<question title>**: <question body, might be multiple paragraphs, including multiple choices>

➡️ <your recommended answer>
```

Word each question so "yes" accepts your recommended answer.

## Sources

Question-tool names and limits, checked 2026-11-27:

- Claude Code `AskUserQuestion`: [Tools reference](https://code.claude.com/docs/en/tools-reference) and [Handle approvals and user input](https://code.claude.com/docs/en/agent-sdk/user-input) (question limits, `header` length, "Other" free text).
- VS Code `vscode/askQuestions`: [Tools reference](https://code.visualstudio.com/docs/agents/reference/tools-reference), [1.109 release notes](https://code.visualstudio.com/updates/v1_109) (recommended answers, free text), and the tool schema in [`askQuestionsTool.ts`](https://github.com/microsoft/vscode/blob/main/src/vs/workbench/contrib/chat/common/tools/builtinTools/askQuestionsTool.ts) (`header` limit, `recommended`, `allowFreeformInput`).
- Copilot CLI `ask_user`: [CLI command reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference) (tool name, `--no-ask-user`) and the [Copilot CLI changelog](https://github.com/github/copilot-cli/blob/main/changelog.md) (custom answer always offered). Its schema is not published; VS Code's Copilot CLI bridge forwards one question per call.
