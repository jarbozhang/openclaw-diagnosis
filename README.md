# OpenClaw Diagnosis Skill

Claude Code skill for diagnosing and repairing remote [OpenClaw](https://github.com/nicepkg/openclaw) instances via SSH.

## Features

- **Single-machine deep diagnosis** — 6-phase inspection: connectivity, service status, error log analysis, model/auth checks, channel/session config, daily-memory & cron health
- **Bulk fleet inspection** — one-command health check across all managed instances with structured report output
- **Auto-repair** — fixes common issues automatically: session bloat, missing configs, stale lock files, tools conflicts, version upgrades
- **Daily-memory dual-safeguard audit** — verifies both hook-driven and prompt-driven memory generation mechanisms

## Managed Instances

| Name | Tailscale IP | Notes |
|------|-------------|-------|
| zhangjiabo | 100.124.248.95 | |
| yuchao | 100.107.214.102 | gateway may run from terminal |
| zhangyang | 100.67.1.75 | |
| danni | 100.99.28.33 | |
| liucongying | 100.75.6.48 | |
| kai | 100.67.177.68 | SSH sometimes unreachable |
| guoyaya | 100.112.34.65 | multi-user active instance |

## Diagnosis Phases

1. **Connectivity** — ping, SSH port, Gateway port
2. **Service Status** — version, gateway process, launchctl state
3. **Error Log Analysis** — pattern matching against 20+ known error signatures with severity levels
4. **Model & Auth** — API key validity, OAuth token expiration
5. **Channel & Session Config** — dmScope, session reset, contextTokens, fallbacks, tools allowlist
6. **Daily-Memory & Cron** — memory continuity, hook config, AGENTS.md prompts, cron errors

## Auto-Repair Capabilities

| Issue | Auto-fixable |
|-------|-------------|
| Session file bloat (>2MB) | Yes |
| Missing contextTokens | Yes |
| Daily-memory hook not enabled | Yes |
| Session dmScope not set | Yes |
| tools.allow / alsoAllow conflict | Yes |
| Feishu tool names mismatch | Yes |
| Stale session lock files | Yes |
| Version upgrade | Yes |
| OAuth token expired | No (requires browser) |
| iCloud sync corruption | No (requires local access) |

## Usage

Install as a Claude Code skill:

```bash
# Clone into Claude Code skills directory
git clone https://github.com/jarbozhang/openclaw-diagnosis.git ~/.claude/skills/openclaw-diagnosis
```

Then trigger in Claude Code by saying:

- "诊断 openclaw" / "openclaw 挂了"
- "巡检" / "巡检所有机器"
- "看看 guoyaya 的 openclaw"
- Any machine name + service issue context

## Requirements

- Claude Code with SSH access to target machines via Tailscale
- `sshpass` installed locally
- SSH credentials stored in Claude Code memory (`openclaw_ssh_usernames.md`)

## License

MIT
