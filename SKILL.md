---
name: openclaw-diagnosis
description: |
  诊断和修复远程 OpenClaw 实例的问题。当用户提到"诊断openclaw"、"openclaw问题"、
  "openclaw挂了"、"openclaw不响应"、"openclaw状态"、"检查openclaw"时触发此技能。
  也适用于用户提到特定机器名（zhangjiabo、yuchao、zhangyang、danni、liucongying、刘聪颖）并涉及服务问题时触发。
  即使用户只是随口说"openclaw怎么了"或"看看XX的openclaw"也应该触发。
  当用户说"巡检"、"巡检所有机器"、"检查使用情况"时，进入批量巡检模式。
  也适用于用户提到"guoyaya"或"果呀呀"并涉及服务问题时触发。
---

# OpenClaw 远程诊断与修复

通过 SSH 连接远程 OpenClaw 实例，执行标准化诊断流程，自动识别并修复常见问题，
最终输出完整的诊断报告和修复记录。

支持两种模式：
- **单机诊断**：深度诊断指定实例的问题
- **批量巡检**：快速检查所有实例的健康状态和使用情况

## 已知实例

| 名称 | Tailscale IP | SSH 用户 | 备注 |
|------|-------------|----------|------|
| zhangjiabo | 100.124.248.95 | openclaw | openclaws-mac-mini |
| yuchao | 100.107.214.102 | yuchao | gateway 可能从终端启动而非 launchctl |
| zhangyang | 100.67.1.75 | zhangyang | |
| danni | 100.99.28.33 | danni | |
| liucongying | 100.75.6.48 | openclaw | |
| kai | 100.67.177.68 | kai | 经常 SSH 不可达 |
| guoyaya | 100.112.34.65 | claw_mango | 多用户活跃实例 |

用户可能用名称或 IP 指定目标，如果没指定则询问要诊断哪台。
如果用户说"巡检"或"所有"，则对全部实例执行批量巡检。

## 连接准备

SSH 凭据存储在 memory 中（`openclaw_ssh_usernames.md`），优先从 memory 读取，
只在 memory 中找不到时才向用户询问。

使用 `sshpass` + `-o PubkeyAuthentication=no` 避免密钥认证干扰：
```bash
sshpass -p '<password>' ssh -o ConnectTimeout=10 -o StrictHostKeyChecking=no -o PreferredAuthentications=password -o PubkeyAuthentication=no <user>@<ip> '<command>'
```

重要注意事项：
- **超时设置**：ConnectTimeout 至少 10 秒，部分机器延迟较高（如 liucongying 可达 2s+）
- **必须指定 PreferredAuthentications=password**：否则多次 pubkey 尝试后会触发 "Too many authentication failures"
- **不要并行 SSH**：并行 Bash 调用时，若其中一个超时/报错会导致其他调用被取消；改为逐台串行执行
- **SSH/ping 超时必须重试**：Tailscale 网络偶尔抖动，单次 SSH 超时不代表机器不可达。任何 SSH 连接失败（包括 ConnectTimeout、Operation timed out）都必须至少重试 2 次再判定为离线。连通性预检和数据采集阶段都适用此规则。
- PATH 可能不含 homebrew，远程命令中始终前置：
```bash
export PATH="/opt/homebrew/bin:/usr/local/bin:$PATH"
```

## 批量巡检模式

当用户要求巡检所有机器时，按以下步骤执行：

### 步骤 1：连通性预检

用 SSH 探测（不用 ping，部分机器禁了 ICMP），逐台执行（不要并行）：
```bash
sshpass -p '<password>' ssh -o ConnectTimeout=10 -o StrictHostKeyChecking=no -o PreferredAuthentications=password -o PubkeyAuthentication=no <user>@<ip> 'echo OK' 2>&1
```

**重试规则**：如果 SSH 返回超时或连接失败，必须至少重试 2 次（共 3 次尝试）再判定为离线。
Tailscale 网络偶尔抖动，首次超时很常见但重试通常能成功。数据采集阶段同理。

### 步骤 2：逐台采集数据

对每台可达的机器，SSH 执行以下检查（合并为一条 SSH 命令减少连接次数）：

```bash
# 系统状态
uptime

# 进程检查
ps aux | grep openclaw-gateway | grep -v grep

# 今日日志量
grep "<today-date>" ~/.openclaw/logs/gateway.log 2>/dev/null | wc -l

# 今日消息统计（飞书收到的消息数）
grep "<today-date>" ~/.openclaw/logs/gateway.log 2>/dev/null | grep "received message" | wc -l

# 独立用户数（去重 ou_ ID）
grep "<today-date>" ~/.openclaw/logs/gateway.log 2>/dev/null | grep "received message from" | sed "s/.*from \(ou_[a-f0-9]*\).*/\1/" | sort -u | wc -l

# 今日错误数
grep "<today-date>" ~/.openclaw/logs/gateway.log 2>/dev/null | grep -i "error\|failed" | wc -l

# Session 文件大小（膨胀检测）
ls -lhS ~/.openclaw/agents/main/sessions/*.jsonl 2>/dev/null | head -3

# 关键配置完整性检查
python3 -c "
import json
d=json.load(open('$HOME/.openclaw/openclaw.json'))
defaults = d.get('agents',{}).get('defaults',{})
session = d.get('session',{})
model = defaults.get('model',{})
msgs = d.get('messages',{})
print('dmScope:', session.get('dmScope','未配置'))
print('session.reset:', json.dumps(session.get('reset','未配置')))
print('contextTokens:', defaults.get('contextTokens','未配置'))
print('model.fallbacks:', json.dumps(model.get('fallbacks',[])))
print('compaction.mode:', defaults.get('compaction',{}).get('mode','未配置'))
"

# Daily-memory 检查
ls -lt ~/.openclaw/workspace/memory/ 2>/dev/null | head -5

# Daily-memory 双保险检查（hooks + AGENTS.md 提示词）
python3 -c "
import json
d=json.load(open('$HOME/.openclaw/openclaw.json'))
sm=d.get('hooks',{}).get('internal',{}).get('entries',{}).get('session-memory',{})
print('session-memory-hook:', 'enabled' if sm.get('enabled') else '未配置')
"
grep -c "memory/YYYY-MM-DD" ~/.openclaw/workspace/AGENTS.md 2>/dev/null || echo "agents_md_memory:0"

# Cron 健康检查
grep "<today-date>" ~/.openclaw/logs/gateway.log 2>/dev/null | grep "cron.*Channel is required" | wc -l

# rate_limit 检查
grep "<today-date>" ~/.openclaw/logs/gateway.log 2>/dev/null | grep -i "rate_limit\|rate limit" | wc -l

# 会话锁文件检测（陈旧锁文件会阻止 session reset）
find ~/.openclaw/agents/main/sessions/ -name "*.lock" -mmin +30 2>/dev/null | head -5
echo "stale_locks:$(find ~/.openclaw/agents/main/sessions/ -name '*.lock' -mmin +30 2>/dev/null | wc -l | tr -d ' ')"

# 状态目录云同步检测（iCloud 同步会损坏数据）
xattr -l ~/.openclaw/ 2>/dev/null | grep -c "com.apple.icloud" || echo "icloud_sync:0"

# Embedding provider 可用性（memory sync 报错检测）
grep "<today-date>" ~/.openclaw/logs/gateway.err.log 2>/dev/null | grep -i "embedding\|memory.*sync.*failed" | tail -3

# 引导文件大小检查（超过 context 预算会被截断）
wc -c ~/.openclaw/workspace/AGENTS.md 2>/dev/null || echo "AGENTS.md: 不存在"
wc -c ~/.openclaw/workspace/SYSTEM.md 2>/dev/null || echo "SYSTEM.md: 不存在"
```

### 步骤 3：输出巡检报告

汇总为表格：

```
## OpenClaw 巡检报告 — <日期>

| 实例 | 状态 | 今日日志 | 消息数 | 独立用户 | 错误数 | 备注 |
|------|------|---------|--------|---------|--------|------|
| xxx  | 运行中/离线 | N行 | N条 | N人 | N条 | 异常说明 |

### Daily-Memory 状态（双保险）
| 实例 | 今日 daily-memory | hooks session-memory | AGENTS.md 指令 | 状态 |
|------|-------------------|---------------------|---------------|------|
| xxx  | 有/无             | ✅/❌              | ✅/❌          | 双保险/单保险/无保险 |

### 配置完整性
| 实例 | dmScope | session.reset | contextTokens | fallbacks | cron 错误 | rate_limit | 需关注 |
|------|---------|---------------|---------------|-----------|-----------|------------|--------|
| xxx  | ✅/❌  | daily/未配置   | 200000/未配置  | N个/空    | N次       | N次        | 说明   |

### Session 健康
| 实例 | 最大 session 文件 | 大小 | 状态 |
|------|------------------|------|------|
| xxx  | 文件名            | N MB | 正常/⚠️膨胀 |

### 环境健康（来自 doctor 检查项）
| 实例 | 陈旧锁文件 | iCloud 同步 | Embedding 报错 | 引导文件大小 | 需关注 |
|------|-----------|------------|---------------|-------------|--------|
| xxx  | N个/无    | 是/否      | 有/无          | N KB        | 说明   |

### 需关注的问题
1. 具体问题和建议
```

## 单机诊断流程

按顺序执行以下 6 个阶段。每个阶段记录结果，用于最终报告。

### 阶段 1：连通性检查

逐项执行：
- `ping -c 3 -W 2 <ip>` — 网络可达性
- `nc -z -w 2 <ip> 22` — SSH 端口
- `nc -z -w 2 <ip> 18789` — Gateway 端口（默认）

如果 ping 不通，诊断终止，告知用户机器不可达。
如果 SSH 端口不通，诊断终止，告知用户 SSH 服务异常。

注意：Gateway 端口可能绑定在 localhost（127.0.0.1），此时远程 nc 检测会失败但 gateway 实际在运行。
不要因为 Gateway 端口不通就判定服务异常，需要 SSH 登录后进一步确认。

### 阶段 2：服务状态

SSH 登录后执行：

```bash
# 版本信息
openclaw --version
node --version

# Gateway 状态（最重要）
openclaw gateway status

# 进程检查
ps aux | grep openclaw | grep -v grep

# launchctl 服务状态
launchctl list 2>/dev/null | grep -i claw
```

关注点：
- Gateway 是否在运行（pid、state）
- 是通过 launchctl 管理还是手动启动
- 版本是否过旧（对比 `openclaw status` 中的 update 信息）

### 阶段 3：错误日志分析

```bash
# 最近错误日志（重点）
tail -100 ~/.openclaw/logs/gateway.err.log

# 今日运行日志中的错误
grep "<today-date>" ~/.openclaw/logs/gateway.log | grep -i "error\|failed" | tail -30
```

已知错误模式及含义：

| 错误模式 | 含义 | 严重程度 |
|---------|------|---------|
| `deactivated_workspace` | OpenAI Codex OAuth workspace 被停用 | 致命 — agent 无法调用 LLM |
| `No API key found for provider` | Fallback 模型缺少 API key | 致命 — 无可用模型 |
| `ERR_MODULE_NOT_FOUND` | OpenClaw 安装损坏，dist 文件缺失 | 致命 — gateway 无法启动 |
| `error=terminated rawError=terminated` | LLM 请求被终止（超时/profile 耗尽） | 高 — agent 无法完成回复 |
| `rotate_profile reason=timeout` | OAuth profile 超时，轮转到下一个账号 | 高 — 所有 profile 可能耗尽 |
| `surface_error reason=timeout` | 所有 profile 耗尽，错误暴露给用户 | 高 — 用户完全无法使用 |
| `LLM request timed out` | LLM 请求超时 | 高 — 可能是网络或提供商问题 |
| `fetch failed` / `ECONNABORTED` | 外部 API 网络不通 | 高 — 影响频道和 LLM |
| `timeout of 15000ms exceeded` + `[ws]` | WebSocket 连接超时 | 高 — 影响实时消息推送 |
| `tools cannot set both allow and alsoAllow` | tools 配置冲突，allow 和 alsoAllow 不能共存 | 高 — 可能导致 gateway 启动失败 |
| `rate_limit` / `usage limit` | ChatGPT Plus 账号配额耗尽，需等待冷却（通常 24-54h） | 高 — 所有请求失败 |
| `refresh_token_reused` | OAuth refresh token 被并发刷新导致失效 | 高 — 需到机器上 `openclaw configure` 重新授权 |
| `Channel is required` (cron) | Feishu WebSocket 老化，gateway 内部标记 channel 不健康 | 中 — 定时任务静默失败，重启 gateway 临时修复 |
| `allowlist contains unknown entries` | alsoAllow 中的工具名在当前 runtime 不存在 | 中 — 工具不会被注册，但不阻塞启动 |
| `ENOENT ... SKILL.md` | Skill 文件路径不存在（全局 vs workspace 路径不匹配） | 中 — 相关 skill 不可用 |
| `WebSocket connection closed with code 1006` | Discord WebSocket 异常断开 | 中 — 通常是网络环境问题 |
| `TypeError: ... is not a function` (plugin) | 插件与新版本不兼容 | 中 — 影响特定插件功能 |
| `skipping duplicate message` (memory dedup) | 重复消息被跳过 | 低 — 正常去重行为 |
| `plugins.allow is empty` | 插件信任白名单未设置 | 低 — 安全警告 |
| `missing transcripts` | Session 数据不完整 | 低 — 不影响运行 |

### terminated 错误深度分析

当日志中出现大量 `error=terminated rawError=terminated` 时，按以下顺序排查：

1. **检查 session 文件大小**：
```bash
ls -lhS ~/.openclaw/agents/main/sessions/*.jsonl | head -5
```
如果最大 session 超过 5MB，很可能是 context 过大导致 Codex API 处理超时。
大 session 通常由大量 toolResult（exec 命令返回的大输出）堆积造成。

2. **检查 OAuth token 有效性**：
```bash
python3 -c "
import json, time
d=json.load(open('$HOME/.openclaw/agents/main/agent/auth-profiles.json'))
now=time.time()*1000
for k,v in d.items():
    if not isinstance(v,dict): continue
    exp=v.get('expires',0)
    if exp:
        remain_h=(exp-now)/3600000
        status='VALID' if exp>now else 'EXPIRED'
        print(k+': '+status+' ('+str(round(remain_h,1))+'h remaining)')
"
```

3. **检查 API 连通性**：
```bash
curl -s --connect-timeout 10 -o /dev/null -w "HTTP %{http_code} time=%{time_total}s" https://api.openai.com/v1/models
```

4. **因果链判断**：
   - token 有效 + API 通 + session 巨大 → session context 过大，重启 gateway 清理状态
   - token 有效 + API 通 + session 正常 → Codex 平台侧限流，等待或换 profile
   - token 过期 → 需到机器上 `openclaw configure` 重新授权
   - API 不通 → 网络/proxy 问题

### 阶段 4：模型与授权检查

```bash
# 当前模型配置和授权
python3 -c "
import sys,json
d=json.load(open('$HOME/.openclaw/openclaw.json'))
defaults = d.get('agents',{}).get('defaults',{})
print('Model:', json.dumps(defaults.get('model',{}), indent=2))
print('Auth:', json.dumps(d.get('auth',{}), indent=2))
"
```

验证主模型是否可用：如果错误日志中有 `deactivated_workspace` 或 `No API key`，
说明当前模型授权已失效。

同时检查 OAuth token 有效期：
```bash
python3 -c "
import json, time
d=json.load(open('$HOME/.openclaw/agents/main/agent/auth-profiles.json'))
now=time.time()*1000
for k,v in d.items():
    if not isinstance(v,dict): continue
    exp=v.get('expires',0)
    if exp:
        remain_h=(exp-now)/3600000
        status='VALID' if exp>now else 'EXPIRED'
        print(k+': '+status+' ('+str(round(remain_h,1))+'h remaining)')
    else:
        print(k+': type='+v.get('type','?'))
"
```

### 阶段 5：频道与 Session 配置检查

```bash
# 频道和 session 配置
python3 -c "
import sys,json
d=json.load(open('$HOME/.openclaw/openclaw.json'))
print('Channels:', json.dumps(d.get('channels',{}), indent=2))
print('Session:', json.dumps(d.get('session',{}), indent=2))
"

# 从日志中检查各频道状态
grep -E '\[(feishu|telegram|discord|openclaw-weixin)\]' ~/.openclaw/logs/gateway.log | tail -30
```

检查项：
- 各频道是否正常连接（看日志中是否有 `starting`、`logged in`、`ready` 等状态）
- `session.dmScope` 是否配置为 `per-channel-peer`（推荐值，确保每个用户独立 session）
- 如果未配置 dmScope，所有用户会共享同一个 session，应提醒修复
- `session.reset.mode` 是否为 `daily`（防止 session 膨胀的官方推荐方案）
- `agents.defaults.contextTokens` 是否设置为 200000（限制上下文窗口，触发更早的 compaction）
- `agents.defaults.model.fallbacks` 是否配置（单 profile 挂了就全挂）
- `dmPolicy` 设置：`pairing` 模式下只有已配对用户能对话，`open` 模式下所有人可对话
- `tools.alsoAllow` 是否包含飞书工具（如 feishu_chat、feishu_doc 等）
- `messages.ackReaction`、`ackReactionScope`、`statusReactions`：这些配置是可选的，**未配置也不影响功能**，飞书上仍会显示处理状态表情。巡检时不要将其标记为问题或建议修复。
- Session 文件大小：超过 2MB 需要关注，超过 5MB 需要立即处理

### 阶段 5.5：工具可用性检查

```bash
# tools 配置
python3 -c "
import json
d=json.load(open('$HOME/.openclaw/openclaw.json'))
tools = d.get('tools', {})
print('profile:', tools.get('profile'))
print('allow:', tools.get('allow'))
print('alsoAllow count:', len(tools.get('alsoAllow', [])))
"

# 检查 allow 和 alsoAllow 是否同时存在（不允许）
# 如果同时存在，需要合并 allow 到 alsoAllow 并删除 allow

# skill 目录
ls ~/.openclaw/workspace/skills/ 2>/dev/null

# 检查是否有 npm 正在更新 openclaw（会导致 dist 文件缺失）
ps aux | grep "npm i" | grep -v grep
```

关键配置陷阱：
- **`allow` 和 `alsoAllow` 不能同时存在**：否则 gateway 启动时报错 `tools cannot set both allow and alsoAllow`。
  解法：将 `allow` 的内容合并到 `alsoAllow`，然后删除 `allow` 字段。
- **`alsoAllow` 中的工具名必须匹配已注册的插件工具名**：内置 feishu plugin 注册的工具名
  可能与 openclaw-lark 插件不同。如果 err 日志出现 `allowlist contains unknown entries`，
  说明工具名不匹配，但不阻塞启动。
- **npm 正在更新 openclaw 时不要重启 gateway**：更新过程中 dist 文件被替换，会导致
  `ERR_MODULE_NOT_FOUND` 错误。等 npm 完成后再重启。

### 阶段 5.6：环境健康检查（Doctor 检查项）

参考 `openclaw doctor` 的检查项，执行以下环境级诊断：

```bash
# 1. 会话锁文件检测 — 陈旧锁文件（>30min）会阻止 session reset
echo "===STALE_LOCKS==="
find ~/.openclaw/agents/main/sessions/ -name "*.lock" -mmin +30 2>/dev/null
echo "stale_lock_count:$(find ~/.openclaw/agents/main/sessions/ -name '*.lock' -mmin +30 2>/dev/null | wc -l | tr -d ' ')"

# 2. 状态目录云同步检测 — iCloud 同步会损坏 session 数据和 SQLite
echo "===ICLOUD_CHECK==="
xattr -l ~/.openclaw/ 2>/dev/null | grep "com.apple.icloud" || echo "icloud_sync: none"
ls -la ~/Library/Mobile\ Documents/ 2>/dev/null | grep -i openclaw || echo "icloud_documents: none"

# 3. Embedding provider 可用性 — memory sync 需要正确的 embedding provider
echo "===EMBEDDING_ERRORS==="
grep "<today-date>" ~/.openclaw/logs/gateway.err.log 2>/dev/null | grep -i "embedding\|memory.*sync.*failed" | tail -5

# 4. 引导文件大小检查 — 超过 ~50KB 的引导文件会被截断，影响 agent 行为
echo "===BOOT_FILES==="
wc -c ~/.openclaw/workspace/AGENTS.md 2>/dev/null || echo "AGENTS.md: missing"
wc -c ~/.openclaw/workspace/SYSTEM.md 2>/dev/null || echo "SYSTEM.md: missing"
wc -c ~/.openclaw/workspace/CLAUDE.md 2>/dev/null || echo "CLAUDE.md: missing"
```

检查项：
- **陈旧锁文件**：超过 30 分钟的 .lock 文件说明写锁未被正常释放，可能阻止 session reset 和新消息写入。修复方式：删除陈旧锁文件。
- **iCloud 同步**：`.openclaw` 目录如果被 iCloud 同步，会导致 SQLite 数据库损坏、session 文件冲突。修复方式：将 `.openclaw` 排除出 iCloud 同步。
- **Embedding 报错**：`Unknown memory embedding provider` 说明 memory 配置引用了不存在的 embedding provider（如配了 packyapi 但它不支持 embedding）。影响语义记忆检索质量。
- **引导文件过大**：AGENTS.md/SYSTEM.md 超过 ~50KB 会被截断，agent 可能丢失关键指令。

### 阶段 6：Daily-Memory 与 Cron 健康检查

#### Daily-Memory 生成机制（双保险）

Daily-memory 有两种独立的生成机制，建议全部启用：

1. **hooks 机制（自动）**：`hooks.internal.entries.session-memory.enabled: true`
   - 触发时机：session reset（凌晨 `session.reset.atHour` 点）时自动触发
   - 由 gateway 内置 hook 自动将当天 session 摘要写入 `memory/YYYY-MM-DD.md`
   - 依赖条件：`session.reset.mode: daily` + hook 启用

2. **AGENTS.md 提示词驱动（agent 自主行为）**：
   - AGENTS.md 中有 memory 指令，agent 在对话中主动调用 `write` 工具写入 daily-memory
   - 不依赖任何 hooks 配置，但需要有实际对话活动
   - 可在日志中看到 `tool call: write params={"file_path":"...memory/YYYY-MM-DD.md"...}`

两种机制互相补充：hooks 保证每天 reset 时一定写入摘要；提示词驱动让 agent 在对话过程中更及时地记录。
如果只有一种机制，应在巡检报告中标记为"缺少双保险"并建议补全。

#### 检查命令

```bash
# Daily-memory 文件
ls -lt ~/.openclaw/workspace/memory/ 2>/dev/null | head -10

# 今日 daily-memory
cat ~/.openclaw/workspace/memory/<today-date>.md 2>/dev/null | head -30

# MEMORY.md（长期记忆）
cat ~/.openclaw/workspace/MEMORY.md 2>/dev/null | head -30

# hooks 配置检查（双保险检查）
python3 -c "
import json
d=json.load(open('$HOME/.openclaw/openclaw.json'))
hooks=d.get('hooks',{})
internal=hooks.get('internal',{})
entries=internal.get('entries',{})
sm=entries.get('session-memory',{})
print('hooks.internal.enabled:', internal.get('enabled','未配置'))
print('session-memory:', 'enabled' if sm.get('enabled') else 'disabled' if 'enabled' in sm else '未配置')
print('boot-md:', 'enabled' if entries.get('boot-md',{}).get('enabled') else '未配置')
print('command-logger:', 'enabled' if entries.get('command-logger',{}).get('enabled') else '未配置')
"

# AGENTS.md 是否有 memory 指令
grep -c "memory/YYYY-MM-DD" ~/.openclaw/workspace/AGENTS.md 2>/dev/null || echo "AGENTS.md 无 memory 指令"

# Cron 任务执行情况
grep "<today-date>" ~/.openclaw/logs/gateway.log | grep -i "cron" | head -20

# Cron 错误统计
grep "<today-date>" ~/.openclaw/logs/gateway.log | grep "cron.*Channel is required" | wc -l
```

检查项：
- daily-memory 是否有今日记录（workspace/memory/<today-date>.md 文件是否存在且非空）
- daily-memory 是否连续（检查最近几天的文件）
- **双保险检查**：hooks session-memory 是否启用 + AGENTS.md 是否有 memory 指令，两者都应该有
- Cron 任务是否有 "Channel is required" 错误（说明定时任务缺少 delivery 配置）
- MEMORY.md 是否已建立（长期记忆索引）

## 自动修复

诊断完成后，根据发现的问题自动执行修复。修复前告知用户将要做什么。

### 修复：Session 膨胀（.jsonl 文件过大）

Session 文件超过 2MB 就应该关注，超过 5MB 会导致 LLM API 超时。

**应急处理**：删除膨胀的 session 文件
```bash
# 找到最大的 session 文件
ls -lhS ~/.openclaw/agents/main/sessions/*.jsonl | head -3
# 删除膨胀的 session（用户会开始新 session，不丢失长期记忆）
rm ~/.openclaw/agents/main/sessions/<session-id>.jsonl
```

**根本修复**：配置 session.reset.mode 防止再次膨胀
```bash
python3 -c "
import json
path = '$HOME/.openclaw/openclaw.json'
with open(path) as f:
    config = json.load(f)
config.setdefault('session', {})['reset'] = {'mode': 'daily', 'atHour': 4}
with open(path, 'w') as f:
    json.dump(config, f, indent=2, ensure_ascii=False)
print('done - session.reset:', json.dumps(config['session']['reset']))
"
```

### 修复：缺少 contextTokens 配置

官方推荐路径为 `agents.defaults.contextTokens`（不是 `agents.defaults.model.contextTokens`）：
```bash
python3 -c "
import json
path = '$HOME/.openclaw/openclaw.json'
with open(path) as f:
    config = json.load(f)
config.setdefault('agents', {}).setdefault('defaults', {})['contextTokens'] = 200000
with open(path, 'w') as f:
    json.dump(config, f, indent=2, ensure_ascii=False)
print('done')
"
```

### 修复：Daily-Memory 双保险缺失（hooks 未配置）

当巡检发现 `hooks.internal.entries.session-memory` 未启用时，补上完整的 hooks 配置：

```bash
python3 -c "
import json
path = '$HOME/.openclaw/openclaw.json'
with open(path) as f:
    config = json.load(f)
config['hooks'] = {
    'internal': {
        'enabled': True,
        'entries': {
            'boot-md': {'enabled': True},
            'command-logger': {'enabled': True},
            'session-memory': {'enabled': True}
        }
    }
}
with open(path, 'w') as f:
    json.dump(config, f, indent=2, ensure_ascii=False)
print('done')
"
```

修改后需要重启 gateway。重启后日志应显示 `[hooks] loaded 4 internal hook handlers`。

注意：如果已有 `hooks` 配置（如自定义 hook），应合并而非覆盖。上面的脚本会覆盖整个 `hooks` 字段。

### 修复：Session dmScope 未配置

```bash
python3 -c "
import json
path = '$HOME/.openclaw/openclaw.json'
with open(path) as f:
    config = json.load(f)
config['session'] = config.get('session', {})
config['session']['dmScope'] = 'per-channel-peer'
with open(path, 'w') as f:
    json.dump(config, f, indent=2, ensure_ascii=False)
print('done')
"
```

修改后需要重启 gateway。

### 修复：tools.allow 和 alsoAllow 冲突

如果配置中同时存在 `allow` 和 `alsoAllow`，需要合并：

```bash
python3 -c "
import json
path = '$HOME/.openclaw/openclaw.json'
with open(path) as f:
    config = json.load(f)
also = config['tools'].get('alsoAllow', [])
allow = config['tools'].get('allow', [])
for item in allow:
    if item not in also:
        also.append(item)
config['tools']['alsoAllow'] = also
if 'allow' in config['tools']:
    del config['tools']['allow']
with open(path, 'w') as f:
    json.dump(config, f, indent=2, ensure_ascii=False)
print('done')
"
```

修改后需要重启 gateway。

### ~~修复：缺少 ackReaction 配置~~ （已确认非问题）

`messages.ackReaction`、`ackReactionScope`、`statusReactions` 未配置时，飞书仍会正常显示处理状态表情。
**不需要修复，巡检时也不要标记为问题。**

### 修复：补充 alsoAllow 飞书工具

如果 tools 配置中缺少飞书相关工具，参照 danni 的配置补充（注意不能同时有 allow 和 alsoAllow）：

```bash
python3 -c "
import json
path = '$HOME/.openclaw/openclaw.json'
with open(path) as f:
    config = json.load(f)
feishu_tools = [
    'feishu_bitable_app', 'feishu_bitable_app_table', 'feishu_bitable_app_table_field',
    'feishu_bitable_app_table_record', 'feishu_bitable_app_table_view',
    'feishu_calendar_calendar', 'feishu_calendar_event', 'feishu_calendar_event_attendee',
    'feishu_calendar_freebusy', 'feishu_chat', 'feishu_chat_members',
    'feishu_create_doc', 'feishu_doc_comments', 'feishu_doc_media',
    'feishu_drive_file', 'feishu_fetch_doc', 'feishu_get_user',
    'feishu_im_bot_image', 'feishu_im_user_fetch_resource',
    'feishu_im_user_get_messages', 'feishu_im_user_get_thread_messages',
    'feishu_im_user_message', 'feishu_im_user_search_messages',
    'feishu_oauth', 'feishu_oauth_batch_auth', 'feishu_search_doc_wiki',
    'feishu_search_user', 'feishu_sheet', 'feishu_task_comment',
    'feishu_task_subtask', 'feishu_task_task', 'feishu_task_tasklist',
    'feishu_update_doc', 'feishu_wiki_space', 'feishu_wiki_space_node'
]
also = config['tools'].get('alsoAllow', config['tools'].get('allow', []))
for t in feishu_tools:
    if t not in also:
        also.append(t)
config['tools']['alsoAllow'] = also
config['tools'].pop('allow', None)
with open(path, 'w') as f:
    json.dump(config, f, indent=2, ensure_ascii=False)
print('done - alsoAllow count:', len(also))
"
```

### 修复：版本过旧

```bash
npm install -g openclaw@latest
```

升级后需要重启 gateway。
**注意**：升级过程中 gateway 会因为 dist 文件缺失而无法启动，等 npm 完成后再重启。

**升级后 feishu 配置兼容性问题**（2026.4.14+ 确认）：

新版本收紧了 `channels.feishu` 的 JSON Schema，以下字段不再被接受，会导致 gateway 启动失败并报
`channels.feishu: invalid config: must NOT have additional properties`：

- `streaming`
- `footer`（含 `footer.elapsed`、`footer.status`）
- `groupPolicy`
- `requireMention`
- `mediaLocalRoots`

**升级后必须检查并删除这些字段**，否则 gateway 无法启动。修复脚本：
```bash
python3 -c "
import json
path = '$HOME/.openclaw/openclaw.json'
with open(path) as f:
    config = json.load(f)
ch = config.get('channels',{}).get('feishu',{})
removed = []
for k in ['footer','streaming','groupPolicy','requireMention','mediaLocalRoots']:
    if k in ch:
        del ch[k]
        removed.append(k)
if removed:
    with open(path, 'w') as f:
        json.dump(config, f, indent=2, ensure_ascii=False)
    print('已删除:', removed)
else:
    print('无需修改')
"
```

修复后再重启 gateway。升级后还建议运行 `openclaw doctor --fix --non-interactive` 处理其他 legacy 配置迁移（如 `tools.web.search` 等）。

**从 openclaw-lark 切换到内置 feishu 时的群聊策略问题**：

内置 feishu 插件默认 `groupPolicy=allowlist`，如果之前用 openclaw-lark 时群聊是 open 模式，切换后群消息会被全部拦截（日志显示 `group xxx not in groupAllowFrom (groupPolicy=allowlist)`），DM 私聊不受影响。

切换插件后必须确认群聊策略：
```bash
python3 -c "
import json
path = '$HOME/.openclaw/openclaw.json'
with open(path) as f:
    config = json.load(f)
ch = config['channels']['feishu']
ch['groupPolicy'] = 'open'
ch['requireMention'] = False
with open(path, 'w') as f:
    json.dump(config, f, indent=2, ensure_ascii=False)
print('done')
"
```

**npm 权限问题**：部分机器可能报 `EACCES` 错误（npm cache 含 root-owned 文件），需先修复：
```bash
echo "<password>" | sudo -S chown -R $(id -u):$(id -g) ~/.npm
```

### 修复：插件不兼容

```bash
openclaw plugins update <plugin-name>
```

### 修复：Session 数据损坏

```bash
openclaw sessions cleanup --store ~/.openclaw/agents/main/sessions/sessions.json --enforce --fix-missing
```

### 修复：陈旧会话锁文件

锁文件超过 30 分钟未释放，说明写进程已崩溃但锁没被清理，会阻止 session reset 和新消息写入。

```bash
# 查找并删除陈旧锁文件（>30min）
find ~/.openclaw/agents/main/sessions/ -name "*.lock" -mmin +30 -exec rm -v {} \;
```

删除后通常不需要重启 gateway，session 会在下次写入时正常获取锁。

### 修复：iCloud 同步导致数据损坏

`.openclaw` 目录被 iCloud 同步会导致 SQLite 数据库损坏。这个问题无法远程自动修复，需要在目标机器上：
1. 将 `~/.openclaw` 移出 iCloud 同步范围
2. 或在 Finder 中右键 `.openclaw` 文件夹 → "从此 Mac 移除下载" 的反操作

### 修复：模型授权失效

这个问题无法远程自动修复（OAuth 需要浏览器交互）。告知用户需要在目标机器上：
```bash
openclaw configure
```
重新完成模型授权。

如果有其他已配置的 provider（如 openrouter），可以询问用户是否临时切换。
切换方式是修改 `~/.openclaw/openclaw.json` 中 `agents.defaults.model.primary`。

### 修复后：重启 Gateway

按以下优先级尝试重启：

**方式 1：launchctl（推荐）**
```bash
# 检查是否由 launchctl 管理
launchctl list 2>/dev/null | grep -i claw
```

如果显示 `ai.openclaw.gateway`：
```bash
launchctl kickstart -k gui/$(id -u)/ai.openclaw.gateway
```

如果 plist 存在但未加载（`ls ~/Library/LaunchAgents/ai.openclaw.gateway.plist` 存在但 `launchctl list` 不显示）：
```bash
launchctl load ~/Library/LaunchAgents/ai.openclaw.gateway.plist
```

**方式 2：kill 进程（从终端启动的情况）**

某些实例（如 yuchao）从终端而非 launchctl 启动 gateway：
```bash
# kill 现有进程
kill $(pgrep -f openclaw-gateway)
# 然后通过 launchctl 启动（更可靠，崩溃自动恢复）
launchctl load ~/Library/LaunchAgents/ai.openclaw.gateway.plist
```

**方式 3：手动重启（兜底）**
```bash
openclaw gateway stop; sleep 2; openclaw gateway start
```

重启后等待 ~5 秒，再检查进程和日志确认服务正常：
```bash
sleep 5
ps aux | grep openclaw-gateway | grep -v grep
tail -5 ~/.openclaw/logs/gateway.log
```

## 输出报告

诊断和修复完成后，输出结构化报告：

```
## OpenClaw 诊断报告 — <机器名> (<IP>)

### 环境信息
- OpenClaw 版本：xxx
- Node 版本：xxx
- Gateway 状态：running/stopped (PID xxx)
- 启动方式：launchctl / 手动

### 发现的问题
| # | 问题 | 严重程度 | 状态 |
|---|------|---------|------|
| 1 | 具体问题描述 | 致命/高/中/低 | 已修复/未修复/需人工处理 |

### 修复记录
| # | 操作 | 结果 |
|---|------|------|
| 1 | 具体执行了什么 | 成功/失败 + 详情 |

### 频道状态
| 频道 | 状态 |
|------|------|
| 飞书 | ✅/❌ + 详情 |
| Telegram | ✅/❌ + 详情 |
| Discord | ✅/❌ + 详情 |
| 微信 | ✅/❌ + 详情 |

### Session 配置
- dmScope: per-channel-peer / 未配置（⚠️ 所有用户共享 session）

### Daily-Memory 状态
- 今日记录: ✅/❌
- 连续天数: N天
- 最近记录: <日期>

### Cron 任务状态
- 今日执行: N次
- Channel 错误: N次（需修复 delivery channel 配置）

### 使用情况（今日）
- 日志行数: N
- 收到消息: N条
- 独立用户: N人
- 错误数: N条

### 仍需关注
- 无法自动修复的问题及建议操作
```
