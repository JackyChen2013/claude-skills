# Claude Code Skills

可复用的 Claude Code skill 集合，直接 clone 到 `~/.claude/skills/` 即可使用。

## 安装方式

```bash
# 安装单个 skill
git clone git@github.com:JackyChen2013/claude-skills.git /tmp/claude-skills
cp -r /tmp/claude-skills/secrets-scanner ~/.claude/skills/
```

## Skills 列表

### secrets-scanner

扫描 Claude Code 配置文件中硬编码的 API Key、JWT Token、密码，给出低成本修复方案。

**触发方式**：对 Claude 说「扫描密钥泄露」或「secrets scan」

**扫描范围**：
- `~/.claude/settings.local.json`
- `~/.claude/settings.json`
- `~/.claude/CLAUDE.md`

**能发现什么**：
- JWT Token（Supabase anon key / service key）
- OpenAI / Anthropic API Key
- Google API Key
- GitHub Token
- SSH 密码硬编码在 expect 命令中
