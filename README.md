# OpenClaw 远程诊断 Skill

通过 SSH 连接远程 [OpenClaw](https://github.com/nicepkg/openclaw) 实例，执行标准化诊断流程，自动识别并修复常见问题。适用于 Claude Code。

## 功能

- **单机深度诊断** — 6 阶段检查：连通性、服务状态、错误日志分析、模型/授权、频道/会话配置、Daily-Memory 与 Cron 健康
- **批量巡检** — 一条命令检查所有托管实例，输出结构化巡检报告
- **自动修复** — 自动修复常见问题：Session 膨胀、配置缺失、陈旧锁文件、工具冲突、版本升级等
- **Daily-Memory 双保险审计** — 同时验证 Hook 驱动和提示词驱动两种记忆生成机制

## 托管实例

| 名称 | Tailscale IP | 备注 |
|------|-------------|------|
| zhangjiabo | 100.124.248.95 | |
| yuchao | 100.107.214.102 | Gateway 可能从终端启动 |
| zhangyang | 100.67.1.75 | |
| danni | 100.99.28.33 | |
| liucongying | 100.75.6.48 | |
| kai | 100.67.177.68 | SSH 偶尔不可达 |
| guoyaya | 100.112.34.65 | 多用户活跃实例 |

## 诊断阶段

1. **连通性检查** — ping、SSH 端口、Gateway 端口
2. **服务状态** — 版本、Gateway 进程、launchctl 状态
3. **错误日志分析** — 匹配 20+ 已知错误模式，按严重程度分级
4. **模型与授权** — API Key 有效性、OAuth Token 过期检测
5. **频道与 Session 配置** — dmScope、session reset、contextTokens、fallbacks、工具白名单
6. **Daily-Memory 与 Cron** — 记忆连续性、Hook 配置、AGENTS.md 提示词、Cron 错误

## 自动修复能力

| 问题 | 可自动修复 |
|------|-----------|
| Session 文件膨胀（>2MB） | 是 |
| 缺少 contextTokens 配置 | 是 |
| Daily-Memory Hook 未启用 | 是 |
| Session dmScope 未设置 | 是 |
| tools.allow / alsoAllow 冲突 | 是 |
| 飞书工具名不匹配 | 是 |
| 陈旧会话锁文件 | 是 |
| 版本升级 | 是 |
| OAuth Token 过期 | 否（需浏览器交互） |
| iCloud 同步导致数据损坏 | 否（需本地操作） |

## 使用方法

安装为 Claude Code skill：

```bash
git clone https://github.com/jarbozhang/openclaw-diagnosis.git ~/.claude/skills/openclaw-diagnosis
```

在 Claude Code 中通过以下方式触发：

- "诊断 openclaw" / "openclaw 挂了"
- "巡检" / "巡检所有机器"
- "看看 guoyaya 的 openclaw"
- 提到机器名 + 服务问题相关上下文

## 前置条件

- Claude Code，且可通过 Tailscale 网络 SSH 访问目标机器
- 本地已安装 `sshpass`
- SSH 凭据存储在 Claude Code memory 中（`openclaw_ssh_usernames.md`）

## License

MIT
