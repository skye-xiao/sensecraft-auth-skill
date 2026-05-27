# SenseCraft Auth — Examples

> 配合 [SKILL.md](SKILL.md)。**从零配置 IdP** 见 [IDP_SETUP.md](IDP_SETUP.md)（SenseCraft Voice 完整实例）。宿主路径见 [INTEGRATION.md](INTEGRATION.md)。

---

## 示例 0：快速入口

| 需求 | 文档 |
|------|------|
| Google / Apple / GitHub **完整配置**（Console + App + oauth/mobile） | [IDP_SETUP.md](IDP_SETUP.md) |
| 仅排错 | 下文 §1–§3 |

---

## 示例 1：Google Release 无法登录（Debug 正常）

1. 确认 **Web serverClientId** 与 `APP_ENV` / authapi 集群一致（PROD vs DEV）
2. 检查 `GOOGLE_SERVER_CLIENT_ID` 是否误覆盖
3. Android：Cloud Console 为**当前安装包签名**新建/核对 **Android OAuth 客户端**（包名 + SHA-1）；Debug 与 Release（及 Play App signing）**各一条**，Client ID 不同属正常，**不必写进 App**
4. Android 包名与 OAuth 客户端一致
5. 上架 Play 后：在 Console 补充 **Play 控制台 → App signing key certificate** 的 SHA-1（不是只用 upload key）
6. iOS：`GIDClientID` 为 **iOS 客户端** ID（不是 Web ID）
7. 有 idToken 但 `oauth/mobile` 17001 → `id_token.aud` 与环境 Web Client 不匹配
8. 选账号前即失败 / **DEVELOPER_ERROR (10)** / **canceled** → 多为缺 Android 客户端或 SHA-1 不匹配（IdP 层）
9. Android `strings.xml` 的 `default_web_client_id`（若有）**不能**代替按环境的 Dart/配置逻辑

**不要**：在 Web 客户端上配置 custom scheme redirect；**不要**把 Android / iOS OAuth Client ID 当作 `serverClientId` 写进代码。

---

## 示例 1b：Google 报 `Custom scheme URIs are not allowed for 'WEB' client type`

**症状**：授权页 400 `invalid_request`，提示 Web client 不允许 custom scheme。

**原因**：App 原生登录误用了 **Web 类型** OAuth Client ID，或 redirect 走了 `com.googleusercontent.apps.xxx://` 却绑在 Web Client 上。

**步骤**：

1. Google Cloud Console 确认已建 **iOS / Android** Client（Bundle ID / 包名 + SHA-1 与 App 一致）
2. iOS：`GIDClientID` = **iOS Client ID**；`CFBundleURLSchemes` = Console 给出的 reversed scheme
3. `GoogleSignIn.initialize(serverClientId: ...)` 只用 **Web Client ID**（PROD/DEV，见 SKILL.md）
4. **不要**把 Web Client ID 填进 iOS `GIDClientID` 或当作 Android 主 Client

---

## 示例 2：Apple Sign in 失败（iOS）

**症状**：点 Apple 登录报错 1000、AKAuthenticationError -7026、或拿不到 `identityToken`。

**步骤**：

1. [Apple Developer](https://developer.apple.com) → Identifiers → App ID → 勾选 **Sign in with Apple**
2. Xcode → Target → **Signing & Capabilities** → 添加 **Sign in with Apple**
3. 重新生成 **Provisioning Profile**（旧描述文件不含 Capability 会失败）
4. 真机测试（模拟器行为与证书可能不一致）
5. `oauth/mobile`：`accountType: apple`，`idToken` = `identityToken`（JWT 字符串）
6. 用户选「隐藏邮箱」时，首次可能无 email — 由宿主 App 引导绑定（见宿主路由，如 `/link-identity`）

**oauth/mobile 17001**：检查 `identityToken` 是否为空、是否过期；authapi 集群是否与测试环境一致。

---

## 示例 3：GitHub 登录无 code

1. GitHub OAuth App **Authorization callback URL** === App 内 `redirect_uri`（**字符完全一致**，scheme **小写**）
2. 示例形态：`{your-scheme}://oauth-callback` — 具体值见宿主 App OAuth 配置（勿在通用 Skill 里写死）
3. Android：专用 **Callback Activity**（或等价）注册 deep link，不要只挂 MainActivity
4. iOS：`Info.plist` → `CFBundleURLSchemes` 含 callback 的 scheme
5. Flutter `flutter_web_auth_2`：scheme 须匹配 `^[a-z][a-z\d+.-]*$`
6. App 侧可生成 `state` 防 CSRF；`code` **一次性** — 勿复用
7. 当前常见实现：仅向 `oauth/mobile` 传 `code`（不传 `code_verifier` / `redirect_uri`，除非后端 OpenAPI 要求）

---

## 示例 4：oauth/mobile 17001

| accountType | 常见原因 |
|-------------|----------|
| google / apple | idToken 过期；aud 错误；时钟偏差 |
| github | code 过期/已用；Client ID 与 SenseCraft 服务端配置不一致 |

确认 body 仅含 `accountType`, `platform`, `idToken|code`；`AUTH_BASE_URL` 与 IdP 环境匹配。

---

## 示例 5：新增 OAuth 厂商

1. 后端 OpenAPI 确认 `accountType` 与字段（`idToken` vs `code`）
2. 客户端 IdP 流程 → `oauth/mobile`
3. IdP 控制台 + 原生 deep link（若需要）
4. 错误码 17002 / 17001 的用户提示
5. PR 更新 **sensecraft-auth-skill**

**Credential 决策**：

| IdP 返回 | oauth/mobile |
|----------|--------------|
| OpenID JWT | `idToken` |
| OAuth code | `code` |

---

## 示例 6：验证码登录

- 发码 `type: 4`，`domain: 2`
- 登录 **multipart**：`account` + `loginCode`（非 JSON）
- 无独立「仅验证验证码」接口 — 直接 email/login
- `17004` → 让用户用上一封邮件里的码

---

## 示例 7：注册流程

1. getEmailCode `type: 1`
2. registerByEmail — **`registerCode` 必须是字符串**
3. 成功通常只有 `userId` → **再 password login** 拿 token
4. `11013` → 引导登录，不要当 unknown error

---

## 示例 8：注册 / 发码 11013

注册场景 sendEmailCode 或 register 返回 **11013**（邮箱已注册）→ 导航到登录页，勿 SnackBar 通用失败。

---

## 示例 9：改完代码自检

```
- [ ] 请求打在 authapi（非 Portal）
- [ ] 密码 MD5 hex；registerCode 为字符串
- [ ] email/login 用 multipart
- [ ] pre-login 不带 stale Authorization
- [ ] 未把产品业务 JWT 混进 SenseCraft-only 文档
- [ ] 同步 sensecraft-auth-skill；宿主路径变更时更新 INTEGRATION.md 已知宿主表
```
