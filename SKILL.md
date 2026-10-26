---
name: codex-delegate
description: Delegate code implementation and verification to the OpenAI Codex CLI (`codex exec`) while Claude keeps design, planning and review. Use when the user wants Codex to write code, run builds/tests, or do a read-only repo analysis, or says "let Codex implement it". Covers locating a working Codex binary, read-only vs write invocations, the brief template, and the review checklist.
---

# Delegating to Codex

Split of roles: **Claude** designs, writes the brief and reviews. **Codex** implements and runs the project's own verification. **The user** approves scope and checks what no tool can (visual quality, product judgment).

## 1. Find a working Codex

```bash
codex --version && codex login status
codex exec -s read-only --skip-git-repo-check -o <tmpfile> "Run a shell command that prints the current directory name, then reply with it. Do not modify files."
```

If the probe fails with `model_not_found` / "model is not supported", the `codex` on PATH is probably older than the one the Codex desktop app bundles. Look for a newer binary (the app's own path is usually in `CODEX_CLI_PATH` inside `~/.codex/config.toml`, or in a versioned subfolder next to the app's `codex` binary), compare `--version`, and use the newest. Do not edit the user's `config.toml` to work around it; pass the binary path and `-m` per call instead.

The probe must make Codex run a shell command; a text-only probe ("Reply OK") is not enough. An install whose command-execution helper is missing still answers text, then fails every real task with `failed to spawn code-mode host ... codex-code-mode-host.exe` and still exits with code 0.

After a Codex app update the CLI appears in a new versioned folder next to the old ones, and `CODEX_CLI_PATH` in `config.toml` may keep pointing to a stale one. Pick the newest build by `--version` and build date, and on Windows check that its folder contains the helper executables (`codex-code-mode-host.exe`, `codex-command-runner.exe`). Re-run the probe after every app update.

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
- `--skip-git-repo-check` is required when the working directory is not a trusted git repository (no commits, an SVN working copy, a plain folder). Without it Codex stops before doing anything (`Not inside a trusted directory`).
- A brief can be given by path in the prompt ("Read <path> and carry it out"); Codex read a file outside `-C` fine (checked). Stdin (`-`) works too. When piping non-ASCII text from Windows PowerShell 5.1, set `$OutputEncoding = [Text.UTF8Encoding]::new($false)` first (the default is ASCII); a Korean brief arrived intact this way.
- Exit code 0 does not mean the work was done: a run whose tools failed exits 0 with an apologetic final message. Judge by the reply and by the files that actually changed.

## 3. Sessions: one Codex session per task

A task (one approved piece of work) is the unit, not a call, and follow-ups on the same feature stay in the same session while they build on what Codex just wrote: that saves re-reading files and keeps its reasoning. Every `codex exec` starts a fresh session that remembers nothing; the `session id:` line in its log identifies it.

**Follow-ups within the same task** continue that session:

```bash
cd <repo>   # resume has no -C; the current directory is used
codex exec resume <SESSION_ID> -c 'sandbox_mode="workspace-write"' -o <reply.txt> - < followup.md
```

Verified behavior of `resume`:
- Memory is kept (the resumed session recalled earlier context).
- Sandbox and working directory are **not** inherited, and `-s` / `-C` do not exist on `resume`. The resumed session runs in the directory you invoke it from (checked: it ignored the first run's directory). Without the `-c` override the sandbox falls back to the config default, which may be writable even if the first run was read-only. Always pass `sandbox_mode` explicitly; `read-only` really blocks writes.
- Use the session ID, never `--last`: it picks the newest session for the current directory, which is wrong as soon as two tasks run.

**Start a new session** (new brief that includes what was learned) when the task is new or unrelated, the direction changes, more than 3-4 follow-ups have piled up or the session's cumulative tokens pass about 150k (a starting value to calibrate; every run's log ends with a `tokens used` line, so sum them in the ledger), or Codex repeats a mistake you already rejected. Long sessions carry stale context and early wrong assumptions. `--ephemeral` is meant for throwaway read-only probes (it should skip recording the session; not verified).

**Follow-up briefs are files too.** Append `## Follow-up N` to the task's brief: what to change, which approved files remain in scope. A follow-up cannot widen the approved scope; that needs the user's approval first. Review the diff after every round, not only the last.

**Task ledger.** Keep one table outside the repository (a stray file would pollute the working copy):

| Task | Session ID | Brief file | Approved files | Tokens used (sum) | State |
|---|---|---|---|---|---|
| short name | uuid | path | list or glob | number | briefed / running / in review / approved / dropped |

Update it on every call; a lost session ID means a lost session. Close the session (state `approved` or `dropped`) when the user accepts or abandons the task.

**Parallel tasks** are fine only if their approved file sets are disjoint and every shared resource (one working copy, one editor or dev server, one database) is used by one job at a time. Run code-writing in parallel and shared-resource verification one after another. When unsure, run serially.

## 4. Brief template

Write it to a file; Codex knows only what the brief and the repo tell it.

```markdown
# Goal
<one paragraph: what and why>

# Scope (approved by the user)
- Files you may change: <exact list, docs included; no open-ended clauses such as "update any other mentions">
- Files you must NOT touch: everything else, including docs that merely mention the changed code (report stale mentions instead of editing them). If another file is needed, stop and report instead of editing.

# Design
<the decisions already made; interfaces, data shapes, behavior>

# Repo rules
Follow the repository's AGENTS.md / CONTRIBUTING / style docs: <name the exact files to read first>.
Repository docs and the existing architecture take precedence over any global guidance in ~/.codex/AGENTS.md.
Do not run VCS state-changing commands (commit, push, update, revert, clean, lock) unless this brief says so.
Text files: preserve each file's existing encoding and line endings (check for a BOM and LF/CRLF first). Write UTF-8 without BOM and keep LF unless the file already differs. In Windows PowerShell 5.1, `Set-Content` and `Out-File -Encoding UTF8` add a BOM and can rewrite line endings; use `[IO.File]::WriteAllText($path, $text, [Text.UTF8Encoding]::new($false))` or the patch tool, and report the first three bytes of every edited text file.

# Done when
- <observable criteria>
- <exact build / test / lint commands to run, and the result expected>

# Report back
Your final reply must be JSON matching `report.schema.json` (Claude passes it with `--output-schema`): status, files changed with a summary each, commands run with exit code and real result, what you could not verify, open questions, and any extension (MCP server, plugin or skill) you would need.
```

## 5. Verification belongs in the brief

Tell Codex to run whatever the project already uses (build, unit tests, linters, the engine's headless CLI, a running dev environment), and to say plainly what it could not run. Look for the tooling in the repo's docs or agent-skill folders before writing the brief, and name the commands in "Done when".

Codex may start the editor or dev server a verification needs and stop what it started. One that was already running when the task began belongs to the user and may hold unsaved work: Codex must ask (through Claude) before closing or restarting it.

Make checks easy to run and read. Give a checks class a `Run()` that catches exceptions and returns a result string (and a `Main()` that throws on failure): `Console.WriteLine` output is not captured by editor commands. Ad-hoc editor scripts go to `$env:TEMP` (outside the repo) and run with `run_script --file`; Windows PowerShell 5.1 splits double quotes inside native-command arguments, so avoid them in `eval` expressions.

Example (Unity): `unity status` to detect a live Editor, `recompile` + `recompile_status` for compile errors, `run_tests`, and driving scenes through the CLI instead of hand-editing `.unity`/`.prefab` YAML. If compile errors put the Editor in Safe Mode, the CLI cannot connect; fix the C# and restart the Editor.

## 6. Review (Claude's job, do not skip)

Codex's report is a claim, not proof.

1. Read the actual diff. Confirm only approved files changed and nothing else (stray files, generated output, formatting churn). No usable VCS (for example an SVN working copy without the `svn` CLI)? Hash the relevant files and keep copies of the files you expect to change before the run, then compare after; `git diff --no-index old new` gives a readable diff. Also check line endings and BOM of edited text files against the originals.
2. Check it against the design in the brief and the repo's rules (comment style, docs that must be kept in sync, naming).
3. Re-run the key verification yourself when cheap (build, the specific failing test), or say it was not re-run.
4. Report to the user: files changed, what changed, verification results (Codex's vs. your own), and what only the user can check.

Methods that worked in practice:
- **Boundary proof**: fingerprint (path, size, modification time) everything outside the allowed area before the run and compare after; confirm any hit by hash. It is cheap and catches editor side effects.
- **Encoding proof**: compare BOM and CRLF/LF counts of every edited text file against the backups. Codex once rewrote whole docs to CRLF and added a BOM, which makes a line diff show the entire file as changed.
- **Unity YAML (prefab/scene)**: compare by object block (anchor id), not by line. Unity reorders override entries, so a line diff can look huge while the real change is one added component.
- **Own checks**: write an independent check with hand-computed expectations (boundary values, ring-buffer wrap, interpolation endpoints) instead of only re-running Codex's checks.
- **Scope honesty**: what the user approved is the file list in the brief. If Codex touched a file outside it, even helpfully, say so in the report and get approval before keeping the change.

If the result is wrong, send a narrower follow-up brief rather than fixing it silently; for a small fix, say that Claude made it.

## 7. Known pitfalls

- A model name that works in the Codex app may be rejected by an older CLI; see section 1.
- A failed run is often an environment problem (stale binary, missing helper, trust check), not a task problem. Fix the environment and start a fresh session with the same brief; do not `resume` a session that never did any work. Record such attempts in the ledger as dropped with the reason.
- The first read-only probe is the cheapest way to prove auth, model and sandbox before spending a real run.
- Codex may read AGENTS.md from the working directory upward; keep `-C` inside the repo so repo rules apply.
- Do not paste secrets into briefs; briefs may be logged.
- Open-ended brief clauses ("update any other mentions") widen the scope silently. List the exact files and ask Codex to report the rest.
- Windows PowerShell 5.1 traps: double quotes inside native-command arguments are split (also in `git commit -m`; use `-F file` written without a BOM); `$name:` must be written `${name}:`; variable names are case-insensitive, so `$R` and `$r` are the same variable.
- Be explicit with paths. A prompt saying "the Skills folder" was read twice as Codex's own `~/.codex/skills` instead of the repository's `./Skills`; write paths relative to `-C`.

## 8. Defaults and knobs

**Defaults.** Model `gpt-6.1-sol` (leave `-m` off unless overriding). Reasoning effort `low` for docs, searches and mechanical edits, `medium` for implementation and debugging; go higher only when the user asks. Set it per call with `-c model_reasoning_effort=medium` (checked: the log header shows the value) and leave the user's `config.toml` alone. Do not use the fast service tier (2x speed, more usage) unless the user asks.

**What the model offers** (Codex 0.164 local catalog, `codex debug models`; re-read it after app updates):
- Context window 500k tokens (older models 272k); text and image input, so UI mockups can go along with `-i <image>`.
- Effort levels low, medium, high, xhigh, max, ultra. Default is whatever `config.toml` says.
- Tool mode "code mode only": commands run through a helper executable, which is why a stale CLI folder fails every task (section 1).
- Freeform `apply_patch`, optional live web search (`--search`), optional fast service tier, and its own sub-agents (multi-agent v2).

**Structured report.** Pass `--output-schema <skill-dir>/report.schema.json` and the final message is JSON with `status`, `files_changed`, `commands_run`, `not_verified`, `open_questions`, `extension_requests` (checked: valid JSON came back at low and medium). Parse it, then compare with your own diff; it is a claim, not proof. `--json` streams events (`thread.started`, `item.started`, `item.completed`, `turn.completed`) if you need to see tool failures directly.

**Global guidance.** `~/.codex/AGENTS.md` is loaded into every Codex run and may hold the user's own design preferences that Claude does not see. Read it once, and in each brief say which wins (see the brief template).

**Sandbox limits.** Network is restricted and writes stop at the working directory (plus temp). Codex cannot install packages, reach the web (except `--search`) or edit its own configuration.

**Extensions (MCP servers, plugins, skills).** An MCP server is one tool server (`codex mcp add`), a plugin is an installable bundle from a marketplace that can carry skills, MCP servers and apps (`codex plugin add PLUGIN@MARKETPLACE`), a skill is a folder of instructions under `~/.codex/skills`. Codex should not install any of them itself: its configuration lives outside the workspace and they run external code. It lists what it needs in `extension_requests` (kind, name, reason, install_command). Claude first checks what is already installed (`codex mcp list`, `codex plugin list`) and the source, then installs outside the sandbox once the user agrees; sources the user pre-approved may be installed without asking again. Never grant `--add-dir` on `~/.codex` (it holds the login).

**Not verified yet:** `-i` image input, `queue`, `fork`, `--add-dir`, and whether `--ephemeral` really skips recording a session.
