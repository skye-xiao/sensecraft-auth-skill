# sensecraft-auth-skill

SenseCraft **authapi** Agent Skill：邮箱注册/登录、验证码、Google / Apple / GitHub OAuth、token 刷新、用户资料 API。

**一个仓库、一份 Skill** — 不包含 Voice 业务 JWT、BLE 等产品逻辑。各 App 的代码路径见 auth 模块与路由文档（如 reSpeaker 的 `lib/src/features/auth/`、`docs/APP_ROUTES.md`）。

---

## 安装

### 任意项目（推荐团队统一）

```bash
git clone https://github.com/Seeed-Studio/sensecraft-auth-skill.git \
  ~/.cursor/skills/sensecraft-auth
```

Cursor / Claude：`@sensecraft-auth`  
更新：`cd ~/.cursor/skills/sensecraft-auth && git pull`

### 作为子模块挂在 App 仓库里

```bash
cd your-app
git submodule add https://github.com/Seeed-Studio/sensecraft-auth-skill.git vendor/sensecraft-auth-skill
ln -sfn ../../vendor/sensecraft-auth-skill .cursor/skills/sensecraft-auth
```

### 与 reSpeaker monorepo 同级（本 workspace）

```
reSpeaker_app/
├── sensecraft-auth-skill/     ← 本仓库（单独 push GitHub）
└── respeaker-app/
    └── .cursor/skills/sensecraft-auth → ../../sensecraft-auth-skill
```

---

## 文件

| 文件 | 说明 |
|------|------|
| [SKILL.md](SKILL.md) | 主 Skill |
| [reference.md](reference.md) | API、错误码、IdP |
| [examples.md](examples.md) | 排错、新增 OAuth |
| [CHANGELOG.md](CHANGELOG.md) | 版本变更 |

---

## 与 App 仓库的关系

| 仓库 | 职责 |
|------|------|
| **sensecraft-auth-skill**（本仓库） | SenseCraft authapi 通用知识 |
| **respeaker-app** 等 | 具体代码、路由、Mermaid 文档；**不复制 Skill** |

Agent 在 reSpeaker 改登录：读 **本 Skill** + **`respeaker-app/lib/src/features/auth/`** 与 **`docs/APP_ROUTES.md`**。

---

## 维护

改 authapi / OAuth 通用规则 → 只改 **本仓库**，打 tag / push，全队 `git pull` 或更新 submodule。
