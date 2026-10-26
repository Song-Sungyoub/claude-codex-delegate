# codex-delegate

A Claude Code skill for a simple division of labor: Claude designs, plans and reviews; the OpenAI Codex CLI implements and runs verification; you approve scope.

## Install

Copy this folder to your skills directory:

```bash
# user-level
cp -r codex-delegate ~/.claude/skills/
# or per project
cp -r codex-delegate <repo>/.claude/skills/
```

The skill folder must be named `codex-delegate` (the repository itself is `claude-codex-delegate`):

```bash
git clone https://github.com/Song-Sungyoub/claude-codex-delegate ~/.claude/skills/codex-delegate
```

Requires the `codex` CLI, signed in (`codex login status`).

## Use

Ask Claude to hand a task to Codex ("have Codex implement this", "ask Codex to analyze the repo"). The skill covers:

- finding a Codex binary that accepts your model (the app-bundled CLI is often newer than the one on PATH),
- read-only vs. write invocations of `codex exec`,
- a brief template (scope, design, repo rules, done-when, report-back),
- a review checklist so Claude checks the real diff instead of trusting Codex's report,
- `report.schema.json`: a JSON schema for Codex's final report (`--output-schema`), including a field for extensions (MCP servers, plugins, skills) it would need,
- defaults and knobs (model, reasoning effort, sandbox limits, MCP policy).

## Notes

- Approval happens in chat before any write run; `codex exec` cannot ask mid-run.
- The skill is repo-agnostic. Project-specific rules belong in the repo's AGENTS.md, which the brief tells Codex to follow.

MIT licensed.
