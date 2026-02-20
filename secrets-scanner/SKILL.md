---
name: secrets-scanner
description: "Claude Code 配置文件密钥泄露扫描与修复工具。扫描 settings.local.json 等配置文件中硬编码的 API Key、JWT Token、密码，并给出低成本修复方案。使用场景：(1) 用户说「扫描密钥泄露」「检查 settings 有没有泄露」「secrets scan」(2) 分享给他人做安全自查"
---

# Secrets Scanner — Claude Code 配置安全扫描

## 目标

扫描 Claude Code 本地配置文件中硬编码的敏感信息，给出具体修复方案，修复成本尽量低。

---

## 扫描范围

按顺序检查以下文件：

1. `~/.claude/settings.local.json` — 权限配置，最容易混入硬编码凭证
2. `~/.claude/settings.json` — 全局配置，可能含 API Key
3. `~/.claude/CLAUDE.md` — 用户指令文件，检查是否有明文密钥
4. 当前项目的 `CLAUDE.md`（如果存在）

---

## 扫描规则

### 高危模式（必须修复）

| 模式 | 说明 | 示例 |
|------|------|------|
| `eyJ[A-Za-z0-9+/]{20,}` | JWT Token（Supabase anon key、service key 等） | `eyJhbGciOiJIUzI1NiIs...` |
| `sk-[A-Za-z0-9]{20,}` | OpenAI / Anthropic API Key | `sk-ant-api03-...` |
| `AIza[A-Za-z0-9_-]{35}` | Google API Key | `AIzaSyXXXXXXXX...` |
| `ghp_[A-Za-z0-9]{36}` | GitHub Personal Access Token | `ghp_xxxx` |
| 密码出现在 `expect` 命令中 | SSH/服务器密码硬编码 | `send "mypassword\r"` |
| URL 中含 `token=` 或 `key=` | 凭证拼接在 URL 里 | `?key=AIzaSy...` |

### 中危模式（建议修复）

| 模式 | 说明 |
|------|------|
| IP 地址 + 端口出现在 Bash 权限条目中 | 服务器地址暴露 |
| 数据库连接字符串 | 含用户名密码的 DSN |
| 邮箱 + 密码组合 | 账号凭证 |

---

## 扫描步骤

```
1. 读取 ~/.claude/settings.local.json
2. 读取 ~/.claude/settings.json
3. 读取 ~/.claude/CLAUDE.md
4. 对每个文件，逐行匹配高危/中危模式
5. 输出：文件路径 + 行号 + 泄露类型 + 脱敏预览（只显示前8位）
6. 给出每条的修复方案
```

---

## 修复方案（低成本优先）

### 方案 A：直接删除（最低成本）

适用于：`settings.local.json` 的 `permissions.allow` 里，因为某次一次性操作被记录进去的含密钥条目。

这类条目通常是历史遗留，删掉不影响功能。

**操作**：从 `allow` 数组中删除包含密钥的那一行。

---

### 方案 B：替换为环境变量引用（推荐）

适用于：`settings.json` 的 `env` 块，或需要保留功能的场景。

**操作**：
1. 将密钥写入 `~/.secrets/.env.项目名`（权限设为 600）
2. 在 shell 配置（`~/.zshrc` 或 `~/.bashrc`）中加载：
   ```bash
   export $(grep -v '^#' ~/.secrets/.env.项目名 | xargs)
   ```
3. 代码中改用 `os.environ.get("KEY_NAME")` 或 `Deno.env.get("KEY_NAME")`

---

### 方案 C：密钥轮换（泄露已发生时）

如果密钥已经出现在对话记录（JSONL 文件）中，需要假设已泄露：

1. 立即在对应平台吊销旧密钥
2. 生成新密钥
3. 按方案 B 重新配置

---

## 输出格式

扫描完成后，输出：

```
## 扫描结果

### 高危发现
- [文件路径:行号] 类型：JWT Token | 预览：eyJhbGci... | 建议：方案A（直接删除）

### 中危发现
- [文件路径:行号] 类型：服务器IP | 预览：203.0.113.x | 建议：可保留，注意不要提交到公开仓库

### 无发现
- ✓ ~/.claude/settings.json 未发现敏感信息

## 修复清单
[ ] 删除 settings.local.json 第 XX 行（JWT Token）
[ ] 将 Google API Key 移入 ~/.secrets/.env.google
```

---

## 注意事项

- 扫描时只读取文件，不修改任何内容
- 输出时对密钥做脱敏处理（只显示前 8 位 + `...`）
- 修复操作需用户确认后再执行
- 不要将扫描结果（含完整密钥）输出到对话中
