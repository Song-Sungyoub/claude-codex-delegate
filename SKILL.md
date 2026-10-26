---
name: codex-delegate
description: Delegate code implementation and verification to the OpenAI Codex CLI (`codex exec`) while Claude keeps design, planning and review. Use when the user wants Codex to write code, run builds/tests, or do a read-only repo analysis, or says "let Codex implement it". Covers locating a working Codex binary, read-only vs write invocations, the brief template, and the review checklist.
---

# Delegating to Codex

Split of roles: **Claude** designs, writes the brief and reviews. **Codex** implements and runs the project's own verification. **The user** approves scope and checks what no tool can (visual quality, product judgment).

## 1. Find a working Codex

```bash
codex --version && codex login status
codex exec -s read-only --skip-git-repo-check -o <tmpfile> "Reply OK only. Do not modify files."
```

If the probe fails with `model_not_found` / "model is not supported", the `codex` on PATH is probably older than the one the Codex desktop app bundles. Look for a newer binary (the app's own path is usually in `CODEX_CLI_PATH` inside `~/.codex/config.toml`, or in a versioned subfolder next to the app's `codex` binary), compare `--version`, and use the newest. Do not edit the user's `config.toml` to work around it; pass the binary path and `-m` per call instead.

Treat the stderr noise from unrelated MCP servers (auth errors, `base_instructions` cache warnings) as harmless unless the final reply is missing.

## 2. Invoke

Run `codex exec --help` once; flags move between versions. Core shape:

```bash
# read-only analysis (no approval needed; changes nothing)
codex exec -s read-only -C <repo> -o <reply.txt> "<prompt>"

# implementation (only after the user approved the scope in chat)
codex exec -s workspace-write -C <repo> -o <reply.txt> - < brief.md
```

- `-o` writes only the final message to a file; read that instead of scraping the log. On Windows read it with UTF-8.
- Long briefs go through stdin (`-`), not the command line.
- Never use `--dangerously-bypass-approvals-and-sandbox` / full-access sandboxes unless the user explicitly asks.
- `codex exec` is non-interactive: Codex cannot ask for approval mid-run. All approval happens in chat **before** the call.
- Run long jobs in the background and wait for the completion notice; do not poll.

## 3. Brief template

Write it to a file; Codex knows only what the brief and the repo tell it.

```markdown
# Goal
<one paragraph: what and why>

# Scope (approved by the user)
- Files you may change: <list>
- Files you must NOT touch: everything else. If another file is needed, stop and report instead of editing.

# Design
<the decisions already made; interfaces, data shapes, behavior>

# Repo rules
Follow the repository's AGENTS.md / CONTRIBUTING / style docs: <name the exact files to read first>.
Do not run VCS state-changing commands (commit, push, update, revert, clean, lock) unless this brief says so.

# Done when
- <observable criteria>
- <exact build / test / lint commands to run, and the result expected>

# Report back
Files changed, what changed in each, commands run with their real output (pass/fail), anything you could not verify.
```

## 4. Verification belongs in the brief

Tell Codex to run whatever the project already uses (build, unit tests, linters, the engine's headless CLI, a running dev environment), and to say plainly what it could not run. Look for the tooling in the repo's docs or agent-skill folders before writing the brief, and name the commands in "Done when".

Example (Unity): `unity status` to detect a live Editor, `recompile` + `recompile_status` for compile errors, `run_tests`, and driving scenes through the CLI instead of hand-editing `.unity`/`.prefab` YAML. If compile errors put the Editor in Safe Mode, the CLI cannot connect; fix the C# and restart the Editor.

## 5. Review (Claude's job, do not skip)

Codex's report is a claim, not proof.

1. Read the actual diff. Confirm only approved files changed and nothing else (stray files, generated output, formatting churn).
2. Check it against the design in the brief and the repo's rules (comment style, docs that must be kept in sync, naming).
3. Re-run the key verification yourself when cheap (build, the specific failing test), or say it was not re-run.
4. Report to the user: files changed, what changed, verification results (Codex's vs. your own), and what only the user can check.

If the result is wrong, send a narrower follow-up brief rather than fixing it silently; for a small fix, say that Claude made it.

## 6. Known pitfalls

- A model name that works in the Codex app may be rejected by an older CLI; see section 1.
- The first read-only probe is the cheapest way to prove auth, model and sandbox before spending a real run.
- Codex may read AGENTS.md from the working directory upward; keep `-C` inside the repo so repo rules apply.
- Do not paste secrets into briefs; briefs may be logged.
