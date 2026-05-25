# Skill 安装与使用（Cursor / Claude / Codex）

本仓库**没有**单独的 JSON 配置文件。各工具在启动时**自动扫描**约定目录下的 `SKILL.md`；把本仓库放到对应路径即可。

Skill 标识（frontmatter `name`）：**`sensecraft-auth`**  
→ 安装目录名必须是 **`sensecraft-auth`**（不是 `sensecraft-auth-skill`）。

---

## 1. Cursor

### 扫描路径

| 作用域 | 路径 |
|--------|------|
| 全局（推荐团队统一） | `~/.cursor/skills/sensecraft-auth/` |
| 项目级 | `<项目根>/.cursor/skills/sensecraft-auth/` |
| 兼容 | `~/.agents/skills/`、`.agents/skills/` |

### 安装（全局）

```bash
git clone https://github.com/skye-xiao/sensecraft-auth-skill.git \
  ~/.cursor/skills/sensecraft-auth
```

### 安装（项目级，monorepo 示例）

```bash
# 在 respeaker-app 或 your-app 根目录
mkdir -p .cursor/skills
ln -sfn ../../sensecraft-auth-skill .cursor/skills/sensecraft-auth
```

或使用 submodule（见 [README.md](README.md)）。

### 使用

| 方式 | 操作 |
|------|------|
| 手动引用 | Agent 输入框输入 `@sensecraft-auth` |
| 斜杠命令 | 输入 `/sensecraft-auth` |
| 自动匹配 | Agent 根据 `SKILL.md` 的 `description` 在登录/OAuth 相关任务时自动加载 |

### 确认已加载

1. **Cursor Settings** → **Rules** → **Agent Decides** 区域应出现 `sensecraft-auth`
2. Agent 聊天框输入 `@`，列表中应有 `sensecraft-auth`
3. 若无：重启 Cursor，或检查目录是否为 `.../sensecraft-auth/SKILL.md`

### 从 GitHub 导入（可选）

**Settings → Rules → Add Rule → Remote Rule (Github)**，填本仓库 URL（与 clone 二选一即可）。

---

## 2. Claude Code（CLI / 桌面）

### 扫描路径

| 作用域 | 路径 |
|--------|------|
| 全局 | `~/.claude/skills/sensecraft-auth/` |
| 项目级 | `<项目根>/.claude/skills/sensecraft-auth/` |

Cursor 文档说明 Claude 与 Cursor **共用同一套 Skill 目录规范**；Claude Code 读 `.claude/skills/`。

### 安装

```bash
git clone https://github.com/skye-xiao/sensecraft-auth-skill.git \
  ~/.claude/skills/sensecraft-auth
```

项目级：

```bash
mkdir -p .claude/skills
ln -sfn ../../sensecraft-auth-skill .claude/skills/sensecraft-auth
```

### 使用

- 在 Claude Code 中提及 SenseCraft 登录、OAuth、authapi 等，Agent 应自动关联 Skill
- 或显式说明：「按 sensecraft-auth skill 处理」
- 项目内可同时放 `CLAUDE.md` 指向本 Skill：`登录/OAuth 见 sensecraft-auth skill`

---

## 3. Codex

### 扫描路径

| 作用域 | 路径 |
|--------|------|
| 全局 | `~/.codex/skills/sensecraft-auth/` |
| 项目级 | `<项目根>/.codex/skills/sensecraft-auth/` |

（Cursor 官方文档：为兼容 Codex，会扫描 `.codex/skills/` 与 `~/.codex/skills/`。）

### 安装

```bash
git clone https://github.com/skye-xiao/sensecraft-auth-skill.git \
  ~/.codex/skills/sensecraft-auth
```

项目级：

```bash
mkdir -p .codex/skills
ln -sfn ../../sensecraft-auth-skill .codex/skills/sensecraft-auth
```

### 使用

- 在 Codex 任务描述中引用 SenseCraft auth / OAuth
- 若支持 `@skill` 或 slash command，名称同为 **`sensecraft-auth`**
- 安装后若未出现，重启 Codex CLI / IDE 插件

---

## 4. 目录结构要求

工具期望的结构：

```
sensecraft-auth/          ← 文件夹名 = frontmatter name
├── SKILL.md              ← 必须
├── reference.md
├── examples.md
├── INTEGRATION.md
└── ...
```

**常见错误**

| 错误 | 后果 |
|------|------|
| 目录叫 `sensecraft-auth-skill` | Agent 找不到或 name 不匹配 |
| 只有仓库根没有 `SKILL.md` 在子目录里 | 未被识别为 Skill |
| clone 到 `~/.cursor/skills-cursor/` | 会被 Cursor 内置 Skill 覆盖，勿用 |

---

## 5. 更新 Skill

```bash
cd ~/.cursor/skills/sensecraft-auth   # 或 ~/.claude/skills/... / ~/.codex/skills/...
git pull
```

submodule 方式：`git submodule update --remote vendor/sensecraft-auth-skill`

---

## 6. 与本 monorepo 的推荐布局

```
reSpeaker_app/
├── sensecraft-auth-skill/          ← 本仓库（git 独立 push）
└── respeaker-app/
    └── .cursor/skills/sensecraft-auth → ../../sensecraft-auth-skill
```

Claude / Codex 用户可在 `respeaker-app` 下同样建 `.claude/skills/`、`.codex/skills/` 符号链接。

---

## 7. 仍看不到 Skill 时

1. 确认路径：`ls ~/.cursor/skills/sensecraft-auth/SKILL.md`
2. 确认 frontmatter `name: sensecraft-auth` 与文件夹名一致
3. 重启 Cursor / Claude Code / Codex
4. Cursor：**Settings → Rules** 查看 Agent Decides 列表
5. 手动试：`@sensecraft-auth 帮我排查 Google 登录 400`
