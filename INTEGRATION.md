# 宿主 App 集成

> 配合 [SKILL.md](SKILL.md)、[reference.md](reference.md)。本 Skill 描述 **SenseCraft authapi 通用契约**；各 App 的工程路径、路由、业务 JWT 由宿主仓库自行维护。

---

## Agent 接到任务时读什么

```
1. sensecraft-auth-skill（本仓库）— API、IdP、错误码、排错
2. 宿主 App 仓库 — auth 实现、路由、环境切换、产品 JWT（若有）
```

---

## 宿主 App 应提供的文档 / 代码

在宿主仓库内应能查到以下内容（路径因框架而异，**以宿主仓库为准**）：

| 项 | 说明 | Flutter 常见位置（示例） |
|----|------|-------------------------|
| Auth 模块 | 登录、OAuth、token 持久化 | `lib/src/features/auth/` |
| 路由文档 | 登录页、OAuth 回调、绑定邮箱等 | `docs/APP_ROUTES.md` 或等价文档 |
| 环境配置 | authapi Base URL、PROD/DEV 切换 | env / `--dart-define` / 设置页 |
| IdP 配置 | Google Web `serverClientId`、GitHub Client ID、Apple Capability | 编译变量或配置文件 |
| 业务 JWT（可选） | SenseCraft token 之后的 Portal / 产品网关登录 | 宿主 auth 或 gateway 模块 |

本 Skill **不包含**上述路径的字面量；Agent 在宿主仓库内搜索 `authDomain`、`oauth/mobile`、`GoogleSignIn` 等关键字定位实现。

---

## SenseCraft 层 vs 产品层

| 层级 | 终点 | 本 Skill |
|------|------|----------|
| SenseCraft authapi | `data.token` + `data.refresh_token` | ✅ 在范围内 |
| 产品业务 JWT / Portal | 各 App 自有接口（路径因产品而异） | ❌ 不在范围内 |

---

## Google OAuth：组织级 vs App 级

| 类型 | 谁建 | 写进 App 代码？ |
|------|------|----------------|
| **Web Client**（PROD/DEV） | Seeed Google Cloud 项目，**按 authapi 档位**（见 [SKILL.md](SKILL.md) §环境 × authapi × Web Client ID） | ✅ 作 `serverClientId` |
| **iOS Client** | 各 App 在 Console 建，Bundle ID 与 App 一致 | ✅ `GIDClientID` + reversed URL scheme |
| **Android Client** | 各 App 在 Console 建，包名 + SHA-1 | ❌ 仅 Console；Play Services 自动匹配 |

**常见错误**：把 **Web Client ID** 当作 iOS/Android 主 Client → Google 报 `Custom scheme URIs are not allowed for 'WEB' client type`。

---

## 已知宿主 App

| App | 包名 / Bundle ID | Auth 实现 | 路由 / 说明 |
|-----|------------------|-----------|-------------|
| SenseCraft Voice | `cc.seeed.voice` | 宿主仓库 `lib/src/features/auth/` | [IDP_SETUP.md](IDP_SETUP.md) · `docs/app_routes.md` |
| Seeedash | `cc.seeed.seeedash` | Seeedash 仓库 auth 模块 | Seeedash 仓库路由文档 |

新增 App 接入时：在本表增加一行，并在该 App 仓库维护 auth 文档链接。

---

## SenseCraft Voice：错误码与国际化

authapi 返回非 0 `code` 时，UI **不要**直接展示原始 `msg`（多为英文且与系统语言不一致）。应走统一本地化：

| 文件 | 作用 |
|------|------|
| `lib/src/core/server/sensecraft_auth/sensecraft_error_codes.dart` | 与 authapi `HttpResponse` 对齐的常量（含 11014 手机、17005–17013 OAuth 等） |
| `lib/src/core/server/server_error_localizer.dart` | `serverErrorMessage(context, e)`：`bizCode` → `AppLocalizations` |
| `lib/src/core/l10n/app_localizations.dart` | 中英 `error*` 文案 |
| `lib/src/core/server/auth/auth_email_conflict.dart` | `11013` / `11014` → 已注册，引导登录而非通用失败 |

登录相关页（`password_login_page`、`email_login_page`、`register_*`、`forgot_password_page`、`third_party_authorize_page` 等）已调用 `serverErrorMessage`。

**改 authapi 错误码或新增 code 时**：

1. 更新 authapi `HttpResponse` 常量（服务端）
2. 同步 `sensecraft_error_codes.dart`
3. 在 `server_error_localizer.dart` 的 `_messageForBizCode` 增加 case
4. 在 `app_localizations.dart` 增加 getter + `en` / `zh` 条目
5. 更新本 Skill [reference.md](reference.md) §错误码速查

---

## 宿主 App 自检清单

```
- [ ] authapi 请求打在 authDomain()（非 Portal / 产品 API 误用 auth 路径）
- [ ] 登录环境（PROD/DEV）与 Web serverClientId 一致
- [ ] iOS：Console iOS Client + GIDClientID + URL Scheme（非 Web ID）
- [ ] Android：Console Android Client（包名 + 当前签名 SHA-1）
- [ ] GitHub：callback URL 与 App redirect_uri 完全一致
- [ ] SenseCraft token 持久化后，若有业务 JWT，在宿主模块单独实现
- [ ] 改 authapi 行为时同步更新 sensecraft-auth-skill
- [ ] 新增/变更 bizCode 时同步 Voice：`sensecraft_error_codes.dart` + `server_error_localizer.dart` + l10n
```

---

## 编译期变量（常见）

| 变量 | 作用 |
|------|------|
| `AUTH_BASE_URL` | 覆盖 authapi Base URL（须尾斜杠） |
| `GOOGLE_SERVER_CLIENT_ID` | Web `serverClientId` |
| `GITHUB_CLIENT_ID` | GitHub OAuth App |
| `APP_ENV` | 环境 → 选择 PROD/DEV Web Client |

详见 [SKILL.md](SKILL.md) §编译期变量。
