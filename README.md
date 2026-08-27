# Codex Claude Code Subagent

> **Deprecated:** this skill has moved to
> [`kimurayu45z/subagent`](https://github.com/kimurayu45z/subagent) as
> `subagent-memory`. The replacement keeps this bounded Claude Code delegation
> guidance, adds equivalent Codex guidance, and uses the `subagent` CLI when a
> stable subordinate identity and durable history are useful. This repository
> remains available for migration and recovery but is no longer the active
> implementation.

## Migrate

Install and verify the replacement before removing this skill:

```bash
npx skills add kimurayu45z/subagent -g --agent codex \
  --skill subagent-memory -y --copy
npx skills remove claude-code-subagent -g -y
```

Restart Codex or begin a new turn after migration so skill discovery refreshes.

A Codex skill for delegating bounded coding, review, investigation, and verification work to the local Claude Code CLI. It covers non-interactive execution, model aliases, structured output, turn and cost bounds, explicit tool permissions, and resumable sessions.

The skill deliberately uses model aliases rather than pinning their current concrete versions. Alias resolution varies by provider, Claude Code version, account access, and organization policy.

## Legacy install for recovery only

From GitHub:

```bash
npx skills add kimurayu45z/codex-claude-subagent -g --agent codex -y
```

From a local checkout while developing the skill:

```bash
npx skills add . -g --agent codex -y --copy
```

Restart Codex or begin a new turn after installation so the skill is rediscovered. Invoke it explicitly as `$claude-code-subagent`, or ask Codex to use Claude Code as a subagent.

## Authoritative references

- [Claude Code CLI reference](https://code.claude.com/docs/en/cli-reference)
- [Claude Code model configuration](https://code.claude.com/docs/en/model-config)
- [Claude Code headless mode](https://code.claude.com/docs/en/headless)
- [Claude Code permission modes](https://code.claude.com/docs/en/permission-modes)
- [Claude Code documentation index](https://code.claude.com/docs/llms.txt)

## Requirements

- Claude Code CLI installed and authenticated for the intended provider
- A trusted target workspace, or an isolated environment for untrusted code
- Codex with skill support

This skill does not grant authorization to commit, push, deploy, delete data, send messages, or broaden the user's requested scope.
