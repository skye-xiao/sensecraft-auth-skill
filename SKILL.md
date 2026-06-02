---
name: sensecraft-auth
description: >-
  Integrates SenseCraft unified auth (authapi): email register/login, verification
  codes, password reset, Google/Apple/GitHub OAuth via oauth/mobile, token refresh,
  and user profile APIs. Use when modifying SenseCraft login, OAuth, authapi HTTP,
  IdP Client ID config, or debugging auth failures. Framework-agnostic; pair with the
  host app's auth module and route docs for code paths. Excludes product-specific business
  JWT exchange after SenseCraft login.
---

# SenseCraft 统一认证（authapi）

## 范围（必读）

**本 Skill 只覆盖 SenseCraft authapi**：注册、登录、第三方 OAuth、用户资料接口。

| 在范围内 | 不在范围内（除非用户明确要求） |
|----------|-------------------------------|
| `authDomain()` / **SenseCraft authapi** HTTP | **产品业务后端**（如 Voice 网关、`sensecraft-respeaker-service` 的 `/api/v1/user/app/login` 等） |
| 邮箱验证码、注册、登录、重置密码 | 产品自有 JWT / Portal 登录（各 App 路径不同，见 [INTEGRATION.md](INTEGRATION.md)） |
| `POST /api/v1/auth/oauth/mobile` | BLE、设备、非 auth 业务 API |
| SenseCraft `token` + `refresh_token` | — |
| Google / Apple / GitHub IdP 配置 | — |

**authapi ≠ 产品 API 后端**：本 Skill 的 Host 为 `sensecraft-auth.seeed.cc` / `.seeed.cn` / 内网 authapi；**不要**与产品服务（如 `sensecraft-respeaker-service`、Portal `portalapi`）的登录接口混用。OAuth 走 authapi 时提交 `oauth/mobile`，不是产品后端的 `app/login`。

**登录成功终点（SenseCraft 层）**：`data.token` + `data.refresh_token` 持久化。  
之后的产品业务 JWT / 网关登录见各宿主 App（见 [INTEGRATION.md](INTEGRATION.md) §SenseCraft 层 vs 产品层）。

---

## 接到任务时（工作流）

```
任务类型？
├─ 改 API/客户端   → reference.md 接口矩阵 + 宿主项目 Repository 实现
├─ 配 IdP/编译     → [IDP_SETUP.md](IDP_SETUP.md)（SenseCraft Voice 完整实例）+ reference.md §IdP
├─ 改 UI/路由      → 宿主 App auth 模块 + 路由文档（见 INTEGRATION.md）
├─ 排错            → §排错决策树 + examples.md
└─ 新增 OAuth 厂商 → §Checklist + examples.md §4
```

1. 读 [reference.md](reference.md)（API、错误码、IdP）
2. 若在当前 App 仓库内：读 auth 源码与路由（路径见 [INTEGRATION.md](INTEGRATION.md)）
3. 改 authapi 行为时：同步更新 **sensecraft-auth-skill** 仓库

---

## 架构速览

```
IdP（Google/Apple/GitHub）或邮箱
        ↓
SenseCraft authapi（authDomain）
        ↓
SenseCraft token + refresh_token
        ↓
（可选）宿主产品业务 JWT — 不在本 Skill
```

- **authapi ≠ Portal**：`/api/v1/auth/*` 与 auth 相关 `/api/v1/user/*` → 一律 `authDomain()`

---

## 环境与服务地址

### 环境 × authapi × Web Client ID（Google `serverClientId`）

三者须**同档配对**；`id_token.aud` 须与 SenseCraft 校验用的 Web Client 一致，否则 `oauth/mobile` 返回 **17001**。

| App 环境键 | authapi Base URL | Google Web `serverClientId` | 说明 |
|------------|------------------|----------------------------|------|
| `release` / `prod` / `production` | `https://sensecraft-auth.seeed.cc/authapi/` | **PROD** | 国际生产 |
| `cn` / `china` | `https://sensecraft-auth.seeed.cn/authapi/` | **PROD** | 国内生产（与国际共用 PROD Web Client aud） |
| `dev` / `test` / `local` / `debug` / `qa` / `uat` | `https://intranet-sensecap-env-expose-publicdns.seeed.cc/authapi/` | **DEV** | 内网测试 |

| 档位 | Web Client ID |
|------|---------------|
| **PROD** | `721415563732-gvsfu25trpg6buls5l6kvpf7fqhfrarg.apps.googleusercontent.com` |
| **DEV** | `721415563732-onmkav3p8u5ahq35265am22ulbm6kf9p.apps.googleusercontent.com` |

- 覆盖 authapi：`AUTH_BASE_URL=https://…/authapi/`（**须尾斜杠**）
- 覆盖 Web Client：`GOOGLE_SERVER_CLIENT_ID=…` 或宿主 App 环境选择逻辑
- **`cn` 与 `release` 同属 PROD 档**：authapi 域名不同，但 Google `serverClientId` 仍用 PROD（勿在国内生产误用 DEV Client）
- 验证码 `language`：authapi host 含 `.cn` → `cn`，否则 `en`（与 Web Client 档位独立）

---

## 通用约定

### 响应信封

- `{ code, msg, data }`；`code === 0` 或 `"0"` 为成功
- 密码上传：**MD5，32 位小写 hex**
- 验证码：`domain: 2`（craft）；`type` 1 注册 · 2 重置 · 3 绑定 · **4 登录**
- `language`：host 含 `.cn` → `cn`，否则 `en`

### Authorization 头（SenseCraft 移动客户端惯例）

| 场景 | Header |
|------|--------|
| 已登录 authapi | `Authorization: <access_token>`（**无 Bearer**） |
| 刷新 | `GET /api/v1/auth/refreshToken`，`Authorization: Bearer <refresh_token>` |
| 登录/注册/OAuth | 匿名 path 不带 token（见 reference §Pre-login） |

其它：`Accept: application/json`；`channel: sensecap_app`（可问后端是否保留）；不用 Cookie。

---

## 邮箱注册 / 登录

| 场景 | 步骤 |
|------|------|
| 验证码登录 | getEmailCode(type=4) → email/login multipart(account, loginCode) |
| 密码登录 | email/login multipart(account, password=MD5) |
| 注册 | getEmailCode(type=1) → registerByEmail → **再密码登录**（注册响应通常无 token） |
| 忘记密码 | getEmailCode(type=2) → resetPassword |

- `17004` → **视为可继续**，用上一封邮件验证码
- `11013` / `11014` → 邮箱或手机已注册，引导登录（勿当 unknown error）
- 展示错误：宿主 App 用 `serverErrorMessage` 映射 bizCode → 中英 l10n（Voice 见 [INTEGRATION.md](INTEGRATION.md) §错误码与国际化）；完整 code 表见 [reference.md](reference.md) §错误码速查

---

## 第三方登录（OAuth）

**统一模式**：IdP 凭证 → `POST /api/v1/auth/oauth/mobile` → 存 SenseCraft token。

| 厂商 | App 侧获取 | 提交给 authapi |
|------|------------|----------------|
| Google | `idToken` | `accountType`, `platform`, **`idToken`** |
| Apple | `identityToken` | 同上 |
| GitHub | 浏览器 OAuth → `code` | `accountType`, `platform`, **`code`** |

### 实现要点

- **`oauth/mobile` 只传 `idToken` 或 `code`**；Google `accessToken` 通常仅本地校验
- GitHub **Client Secret 仅 SenseCraft 服务端**
- 同邮箱 OAuth **绑定已有账号**
- `17002` → 后端未支持该 `accountType`

### Google

- **Web Client ID** = mobile `serverClientId` → `id_token.aud`，须与上表 **PROD/DEV 档位**一致（见 §环境 × authapi × Web Client ID）
- Android：包名 + Debug/Release（及 Play App Signing）**SHA-1**；Cloud Console **每条 Android OAuth 客户端 = 一个 SHA-1**，多条客户端会有不同 Client ID，**仅 Console 登记，不写进 App 代码**
- App 代码只配 **Web** `serverClientId`（按 PROD/DEV 档位）；Google Play Services 按当前安装包签名自动匹配对应 Android 客户端
- iOS：Bundle ID + `GIDClientID` + reversed URL scheme
- **Web Client ID ≠ iOS GIDClientID ≠ Android Client ID**（见 reference.md）
- **不用 Firebase**

各 App 仍须在 Google Cloud Console 创建**自己的** iOS / Android OAuth Client（Bundle ID / 包名 + SHA-1），**不能**用 Web Client ID 替代。Web Client ID 见 §环境 × authapi × Web Client ID。**SenseCraft Voice 完整对照表**见 [IDP_SETUP.md](IDP_SETUP.md) §1。

### Apple

Sign in with Apple Capability；`idToken` = `identityToken`；常见失败 1000 / -7026。

### GitHub

- Callback URL 与 App 内 `redirect_uri` **完全一致**（scheme **小写**）
- Client ID 可编译注入；Secret 不进 App
- Scope 常用：`read:user user:email`
- SenseCraft Voice 实例：`Iv23liaPMJxOyKXEs8kt`，callback `sensecraftvoice://oauth-callback` — 见 [IDP_SETUP.md](IDP_SETUP.md) §3

---

## 新增 OAuth 厂商 Checklist

```
- [ ] 1. SenseCraft 后端：OpenAPI 确认 accountType；测试 oauth/mobile
- [ ] 2. 客户端：组装 idToken 或 code + platform
- [ ] 3. UI：IdP 流程 + 错误展示
- [ ] 4. IdP 控制台：Client ID / Callback / Capability
- [ ] 5. 原生：deep link / URL scheme（若浏览器回调）
- [ ] 6. 编译配置（勿提交 Secret）
- [ ] 7. 自测：Debug + Release（Google SHA-1）；17001/17002
- [ ] 8. 更新本 Skill reference + 宿主项目文档
```

详见 [examples.md](examples.md) §4。

---

## 编译期变量（常见）

| 变量 | 作用 |
|------|------|
| `AUTH_BASE_URL` | 覆盖 authapi |
| `GOOGLE_SERVER_CLIENT_ID` | Web `serverClientId` |
| `GITHUB_CLIENT_ID` | GitHub OAuth App |
| `APP_ENV` | 环境 → Web Client 选择 |

---

## 排错决策树

```
登录失败
├─ IdP 层？ → Google SHA-1 / serverClientId / Apple Capability / GitHub callback
├─ oauth/mobile？ → 17001 凭证；17002 accountType
├─ 验证码？ → 17004 用旧码；11008/11010/11015/11016 码错或过期
├─ 展示英文 msg？ → 查宿主是否走 serverErrorMessage，勿裸显 biz msg
└─ 401？ → authapi host；raw token；11101/11102/11103
```

更多：[examples.md](examples.md)（含 Apple §2、注册 §7）· API/错误码：[reference.md](reference.md)

---

## 修改约束

- 不硬编码 Client Secret
- 邮箱登录 **multipart**；`registerCode` 为 **字符串**
- 不确定字段 → SenseCraft OpenAPI + 宿主项目实现
