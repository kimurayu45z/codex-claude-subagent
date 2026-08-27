---
name: claude-code-subagent
description: Deprecated compatibility skill for bounded Claude Code delegation. Prefer subagent-memory from kimurayu45z/subagent, which preserves this guidance and adds Codex plus durable cross-provider context. Use this copy only while migrating or when the replacement is unavailable.
metadata:
  short-description: Deprecated; migrate to subagent-memory
---

# Claude Code Subagent

> Deprecated: the maintained replacement is `subagent-memory` in
> <https://github.com/kimurayu45z/subagent>. If it is installed, use that skill
> and read its `references/claude-code.md` for this workflow. This file remains
> functional only as a migration fallback.

Use Claude Code as a separate worker for a concrete, bounded task. Codex remains responsible for scoping the delegation, reviewing the returned evidence and workspace changes, and reporting the result to the user.

## Use the authoritative references

Consult these pages when exact flags, compatibility, alias resolution, or permission behavior matters:

- [CLI reference](https://code.claude.com/docs/en/cli-reference)
- [Model configuration](https://code.claude.com/docs/en/model-config)
- [Headless mode](https://code.claude.com/docs/en/headless)
- [Permission modes](https://code.claude.com/docs/en/permission-modes)
- [Documentation index](https://code.claude.com/docs/llms.txt)

Run `claude --version` and `claude --help` to check the locally installed CLI. Do not treat `--help` as complete: the official CLI reference explicitly says it does not list every flag. If local behavior and the documentation differ, constrain the invocation to locally supported behavior or explain that an update is needed.

## Delegate deliberately

Before invoking Claude Code:

1. State one outcome, the relevant files or subsystem, and what is out of scope.
2. Specify whether the worker may edit files or must remain read-only.
3. Name the required verification and the expected return format.
4. Run from the intended trusted working directory. `claude -p` skips the workspace trust dialog.
5. Preserve the user's authorization boundary. Do not delegate commits, pushes, deployments, destructive operations, external messages, or unrelated cleanup unless the user authorized them.

The prompt should tell the worker to inspect before acting, preserve unrelated user changes, cite concrete files or evidence, and stop if the requested work requires broader authority. Avoid pasting secrets or large untrusted content into the shell command; refer to trusted local files instead.

## Select a model by alias

Prefer aliases unless the user explicitly needs a pinned model version. Alias targets vary by Claude Code version, provider, account access, and organization policy, and they change over time.

| Alias | Use |
| --- | --- |
| `haiku` | Small, fast, well-specified tasks |
| `sonnet` | Default for routine implementation, review, and investigation |
| `opus` | Difficult reasoning, architecture, or high-value review |
| `best` | Most capable model available under the current account and provider |
| `fable` | Long, difficult autonomous work when available |
| `opusplan` | Opus while planning, then Sonnet for execution |
| `sonnet[1m]` | Sonnet with extended context for unusually large inputs |
| `opus[1m]` | Opus with extended context for unusually large inputs |
| `default` | Clear an override and use the runtime default; it is a special value, not a model alias |

Do not record a concrete version as the permanent meaning of an alias. Check the official model configuration page when exact resolution or availability matters. Use `--model <alias>` for an invocation-specific choice.

## Control headless runs

Use `claude -p` for non-interactive delegation. For a result consumed by Codex, default to `--output-format json`.

`--max-turns` is an optional hard limit for print mode. Claude Code applies no turn limit when the flag is omitted and exits with an error if the limit is reached. Set it only when the caller actually needs a turn cap, choose the value for the expected amount of agentic work, and do not reuse a fixed example value across unrelated tasks. Treat a run that reaches the cap as incomplete.

`--max-budget-usd` is a separate optional spend ceiling for print mode; usage by Claude Code subagents counts toward it. Use turn and budget limits independently according to which resource the caller needs to bound.

```bash
claude -p \
  --model sonnet \
  --output-format json \
  "Inspect the current implementation, identify the root cause, and return evidence-backed findings. Do not edit files."
```

Use `--json-schema` when downstream logic needs validated fields. In JSON output, read the response from `result`, schema-constrained data from `structured_output`, and the resumable identifier from `session_id`. Check the process exit status as well as the JSON payload. Cost fields are estimates, not authoritative billing records.

Use `stream-json` only when the caller needs incremental events. Combine it with the flags required by the current CLI reference, and treat the final `result` event as completion.

## Restrict tools and permissions

These controls have different purposes:

- `--tools` limits which built-in tools are available.
- `--allowedTools` pre-approves matching operations; it does not restrict the available tool set.
- `--disallowedTools` denies tools or scoped operations. Deny rules remain useful as defense in depth.
- `--permission-mode dontAsk` allows only pre-approved operations and is the predictable default for locked-down headless work.
- `--permission-mode plan` is useful for exploration that must not edit.

Build the smallest allowlist that can complete the task. Authorize exact verification commands after inspecting the project rather than broadly allowing all shell commands.

Example read-only review:

```bash
claude -p \
  --model opus \
  --output-format json \
  --permission-mode dontAsk \
  --tools "Read,Bash" \
  --allowedTools "Read" "Bash(rg *)" "Bash(git status *)" "Bash(git diff *)" "Bash(git log *)" \
  --disallowedTools "Edit" "Write" "Bash(git push *)" "Bash(git reset *)" "Bash(rm *)" "mcp__*" \
  "Review the current changes for correctness and regressions. Do not modify the workspace. Cite file paths and verification evidence."
```

Example bounded implementation, after the user has authorized edits:

```bash
claude -p \
  --model sonnet \
  --output-format json \
  --permission-mode dontAsk \
  --tools "Read,Edit,Write,Bash" \
  --allowedTools "Read" "Edit" "Write" "Bash(git status *)" "Bash(git diff *)" "Bash(npm test *)" "Bash(npm run lint *)" \
  --disallowedTools "Bash(git push *)" "Bash(git reset *)" "Bash(rm *)" "mcp__*" \
  "Implement the requested fix only in the named subsystem. Preserve unrelated changes, run the allowed checks, and summarize changed files and remaining risks. Do not commit or push."
```

Adjust tool names and exact commands to the repository. If the allowlist blocks a necessary action, review that action and start a follow-up with a narrowly expanded rule; do not jump to unrestricted permissions.

Never use `--dangerously-skip-permissions` merely to avoid designing permissions. Use it only with explicit user authorization inside an appropriately isolated container or VM, preferably as a non-root user, and retain explicit deny rules. Do not use it in an ordinary host workspace.

## Decide whether to load project customizations

The normal `claude -p` path loads project and user configuration, including hooks, skills, MCP servers, and project instructions. Use it only in a trusted workspace.

`--bare` improves startup time and reproducibility and avoids most auto-discovered customization, but it also skips project instructions such as `CLAUDE.md`, hooks, plugins, MCP servers, and normal OAuth/keychain lookup. Use it only when that tradeoff is intentional, then pass the required context and narrow tools explicitly. Bare mode is not a substitute for OS-level isolation.

## Resume the right worker

For follow-up work, capture `session_id` from JSON and resume that exact session. Prefer `--resume <session-id>` over `--continue` when multiple Claude sessions may exist, because “most recent” is ambiguous. Repeat the model and permission controls when reproducibility matters; an explicit `--model` overrides the restored model.

```bash
claude -p \
  --resume "$CLAUDE_SESSION_ID" \
  --model sonnet \
  --output-format json \
  --permission-mode dontAsk \
  --tools "Read,Bash" \
  --allowedTools "Read" "Bash(git diff *)" "Bash(npm test *)" \
  --disallowedTools "Edit" "Write" "Bash(git push *)" "Bash(rm *)" \
  "Re-check the latest diff and tests, then report whether the earlier concern is resolved. Do not edit files."
```

Use `--fork-session` with `--resume` when the follow-up should branch without continuing the original session identity. Use `--no-session-persistence` only when no follow-up or audit trail is needed.

## Review the delegation

Claude Code's response is evidence, not automatic acceptance. Inspect any workspace diff, run proportionate checks independently, reconcile disagreements with the actual code and tool output, and disclose incomplete verification. Keep the user's original task as the final acceptance criterion.
