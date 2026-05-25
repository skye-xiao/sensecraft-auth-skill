# sensecraft-auth-skill

> **SenseCraft authapi 通用 Agent Skill，供多 App 复用**

SenseCraft **authapi** Agent Skill：邮箱注册/登录、验证码、Google / Apple / GitHub OAuth、token 刷新、用户资料 API。

**一个仓库、一份 Skill** — 只覆盖 SenseCraft 统一认证（authapi），不包含各 App 的产品业务 JWT、BLE、设备协议等。具体代码路径与路由见各宿主 App 仓库（见 [INTEGRATION.md](INTEGRATION.md)）。

---

## 适用对象

任意接入 SenseCraft authapi 的移动 / Web App，例如：

| App | 包名 / Bundle ID | 宿主 auth 文档 |
|-----|------------------|--------------|
| SenseCraft Voice | `cc.seeed.voice` | 宿主仓库 `lib/src/features/auth/` · `docs/APP_ROUTES.md` |
| Seeedash | `cc.seeed.seeedash` | 见 Seeedash 仓库 auth 模块与路由文档 |

Agent 改登录时：**本 Skill** + **宿主 App 的 auth 源码与路由文档**。

---

## 安装

详细步骤（**Cursor / Claude Code / Codex** 路径、确认是否加载、常见错误）见 **[SETUP.md](SETUP.md)**。

> Skill **没有**单独 JSON 配置；各工具自动扫描 `~/.cursor/skills/`、`~/.claude/skills/`、`~/.codex/skills/` 等目录。安装文件夹名必须为 **`sensecraft-auth`**（与 `SKILL.md` 里 `name` 一致）。

### 任意项目（推荐，团队统一 — Cursor 全局）

```bash
git clone https://github.com/skye-xiao/sensecraft-auth-skill.git \
  ~/.cursor/skills/sensecraft-auth
```

> 仓库 canonical remote 为 `skye-xiao/sensecraft-auth-skill`；若组织迁移至 `Seeed-Studio/sensecraft-auth-skill`，将 URL 替换即可。

Cursor：`@sensecraft-auth` 或 `/sensecraft-auth`（见 [SETUP.md](SETUP.md) §Cursor）  
Claude Code / Codex：安装到 `~/.claude/skills/` / `~/.codex/skills/`（见 [SETUP.md](SETUP.md)）

更新：`cd ~/.cursor/skills/sensecraft-auth && git pull`

### 作为子模块挂在 App 仓库里

```bash
cd your-app
git submodule add https://github.com/skye-xiao/sensecraft-auth-skill.git vendor/sensecraft-auth-skill
ln -sfn ../../vendor/sensecraft-auth-skill .cursor/skills/sensecraft-auth
```

### 可选：与 App 仓库同级（本地 workspace 示例）

```
your-workspace/
├── sensecraft-auth-skill/     ← 本仓库
└── your-app/
    └── .cursor/skills/sensecraft-auth → ../../sensecraft-auth-skill
```

---

## 文件

| 文件 | 说明 |
|------|------|
| [SKILL.md](SKILL.md) | 主 Skill（Agent 读取） |
| [reference.md](reference.md) | API、错误码、IdP |
| [examples.md](examples.md) | 排错、新增 OAuth |
| [INTEGRATION.md](INTEGRATION.md) | 宿主 App 集成契约与检查清单 |
| [SETUP.md](SETUP.md) | Cursor / Claude / Codex 安装与排错 |
| [CHANGELOG.md](CHANGELOG.md) | 版本变更 |

---

## 与 App 仓库的关系

| 仓库 | 职责 |
|------|------|
| **sensecraft-auth-skill**（本仓库） | SenseCraft authapi 通用知识、OAuth 规则、排错 |
| **各宿主 App 仓库** | 具体实现、UI、路由、产品业务 JWT；**不复制 Skill 正文** |

---

## 维护

改 authapi / OAuth 通用规则 → 只改 **本仓库**，打 tag / push，全队 `git pull` 或更新 submodule。  
某 App 专属路径或路由变更 → 改该 App 仓库文档，并在 [INTEGRATION.md](INTEGRATION.md) 的「已知宿主 App」表更新链接（若适用）。
