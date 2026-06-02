# SenseCraft Auth — Reference

> 配合 [SKILL.md](SKILL.md)。宿主 App 集成见 [INTEGRATION.md](INTEGRATION.md)。

## authapi 接口矩阵

Base = `authDomain()`（须尾斜杠）。

| 场景 | Method | Path | Content-Type | 鉴权 |
|------|--------|------|--------------|------|
| 发送邮箱验证码 | POST | `/api/v1/auth/getEmailCode` | JSON | 无 |
| 邮箱登录 | POST | `/api/v1/auth/email/login` | **multipart** | 无 |
| 邮箱注册 | POST | `/api/v1/auth/registerByEmail` | JSON | 无 |
| 重置密码 | PUT | `/api/v1/auth/resetPassword` | JSON | 无 |
| 第三方登录 | POST | `/api/v1/auth/oauth/mobile` | JSON | 无 |
| 刷新 access | GET | `/api/v1/auth/refreshToken` | — | Bearer refresh |
| 修改密码 | PUT | `/api/v1/user/changePassword` | JSON | access |
| 登出 | POST | `/api/v1/user/signOut` | — | 建议带 token |
| 注销账号 | POST | `/api/v1/user/logout` | — | 需登录 |
| 用户与组织 | GET | `/api/v1/user/getUserOrgInfo` | — | access |
| 安全/绑定 | GET | `/api/v1/user/getSecurityInfo` | — | 需登录 |
| 更新资料 | PUT | `/api/v1/user/updateProfile` | JSON | 需登录 |

### getEmailCode

```json
{ "email": "user@example.com", "language": "en", "type": 4, "domain": 2 }
```

| type | 用途 |
|------|------|
| 1 | 注册 |
| 2 | 重置密码 |
| 3 | 绑定 |
| 4 | 登录验证码 |

### email/login（multipart）

**Content-Type：`multipart/form-data`**（不是 JSON）

| 字段 | 说明 |
|------|------|
| `account` | 邮箱 |
| `password` | MD5 hex（密码登录） |
| `loginCode` | 验证码（与 password **二选一**） |

### registerByEmail（JSON）

```json
{
  "email": "user@example.com",
  "registerCode": "123456",
  "userName": "user@example.com",
  "password": "e10adc3949ba59abbe56e057f20f883e"
}
```

| 字段 | 注意 |
|------|------|
| `registerCode` | **必须是 JSON 字符串** `"123456"`，不要传数字 |
| `password` | MD5 32 位小写 hex |
| 成功 `data` | 通常只有 `userId`，**无 token** → 需再调 email/login |

常见错误：`11008` 验证码错误 · `11002` 账号不存在 · `11013`/`11014` 已注册 → 引导登录

### resetPassword（JSON）

```json
{
  "email": "user@example.com",
  "code": "123456",
  "password": "<md5 hex>"
}
```

`code` 来自 getEmailCode **type=2**。常见：`11010`/`11008` 码错或过期 · `11002` 账号不存在

### changePassword（JSON，已登录）

```json
{
  "oldPassword": "<md5 hex>",
  "password": "<md5 hex>"
}
```

鉴权：`Authorization: <access_token>`（无 Bearer）

### oauth/mobile

**Content-Type：`application/json`**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `accountType` | string | 是 | `google` · `apple` · `github`（**小写**） |
| `platform` | string | 是 | `ios` · `android` |
| `idToken` | string | Google/Apple | OpenID JWT；Apple 传 `identityToken` |
| `code` | string | GitHub | 授权码；与 `idToken` **二选一**（按厂商） |

**不要**传 `accessToken`、`redirect_uri`、`code_verifier`（除非 OpenAPI 明确要求）。

#### Google（iOS / Android）

```http
POST /api/v1/auth/oauth/mobile
Content-Type: application/json

{
  "accountType": "google",
  "platform": "ios",
  "idToken": "<Google Sign-In 返回的 id_token>"
}
```

#### Apple

```http
POST /api/v1/auth/oauth/mobile
Content-Type: application/json

{
  "accountType": "apple",
  "platform": "ios",
  "idToken": "<Sign in with Apple 的 identityToken>"
}
```

#### GitHub

```http
POST /api/v1/auth/oauth/mobile
Content-Type: application/json

{
  "accountType": "github",
  "platform": "android",
  "code": "<OAuth 回调 URL 中的 code，一次性>"
}
```

#### 成功响应（`code === 0`）

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "token": "<access_token>",
    "refresh_token": "<refresh_token>",
    "user_id": 123,
    "account": "user@example.com",
    "email": "user@example.com",
    "nickname": "User",
    "org_id": 1
  }
}
```

常见失败：**17001**（idToken 无效 / aud 与 Web Client 档位不匹配）· **17002**（accountType 未开通）

### 登录成功 data 常见字段

`token`, `refresh_token`, `user_id`, `account`, `email`, `nickname`, `org_id`

---

## 错误码速查

与 authapi `HttpResponse` 常量一致。`code` 可能是 number 或 string `"0"`，比较前用 `parseSenseCraftCode` 归一化为 int。

**SenseCraft Voice 客户端**：常量见 `lib/src/core/server/sensecraft_auth/sensecraft_error_codes.dart`；展示文案经 `serverErrorMessage()` → `server_error_localizer.dart` + `AppLocalizations`（中英）。未映射的 code 仍 fallback 到响应 `msg`。

### 通用 (10xxx)

| code | 服务端 msg（摘要） | Voice l10n | 处理建议 |
|------|-------------------|------------|----------|
| `0` | success | — | 成功 |
| `10000` | server error | `errorInternalError` | 稍后重试 |
| `10002` | Remote called error | `errorRemoteCalled` | 远端服务异常 |
| `10007` | path not found | `errorPathNotFound` | 检查 path / 环境 |
| `10009` | Lack of necessary parameters | `errorMissingParams` | 缺 token/参数 → 重新登录 |

### 账号 / 邮箱 / 手机 / 密码 (110xx)

| code | 服务端 msg（摘要） | Voice l10n | 处理建议 |
|------|-------------------|------------|----------|
| `11002` | No account exists | `errorAccountNotFound` | 账号不存在或先注册 |
| `11005` | Password error | `errorPasswordIncorrect` | 密码错误 |
| `11008` | Verification code error | `errorVerifyCodeInvalid` | 验证码错误 |
| `11010` | Verification code expired… | `errorVerifyCodeExpired` | 验证码过期，重新获取 |
| `11013` | email registered | `errorEmailAlreadyRegistered` | **→ 引导登录** |
| `11014` | mobile registered | `errorMobileAlreadyRegistered` | **→ 引导登录**（国内） |
| `11015` | Sms code error | `errorVerifyCodeInvalid` | 短信验证码错误 |
| `11016` | Sms code expired… | `errorVerifyCodeExpired` | 短信码过期 |
| `11017` | Sms code has been sent | `errorSmsCodeAlreadySent` | 稍后再发 |
| `11018` | Mobile required | `errorMobileRequired` | 必填手机号 |
| `11019` | Invalid mobile format | `errorMobileFormatInvalid` | 格式错误 |
| `11020` | New password must differ… | `errorNewPasswordSameAsOld` | 新密码不能与旧密码相同 |
| `11025` | Account frozen | `errorAccountFrozen` | 联系客服 |

### Token (111xx)

| code | 服务端 msg（摘要） | Voice l10n | 处理建议 |
|------|-------------------|------------|----------|
| `11101` | Validation token expires | `errorTokenExpired` | 重新登录；客户端可尝试 refresh |
| `11102` | Invalid token | `errorTokenInvalid` | 重新登录 |
| `11103` | Refresh_token error | `errorTokenInvalid` | 重新登录 |

### 权限 / 参数 / OSS / 限流 (112xx)

| code | 服务端 msg（摘要） | Voice l10n | 处理建议 |
|------|-------------------|------------|----------|
| `11201` | no options permission | `errorForbidden` | 无权限 |
| `11202` | parameters invalid | `errorInvalidParams` | 查 JSON/multipart 格式 |
| `11229` | exceed login limitation | `errorTooManyLoginAttempts` | 稍后重试 |
| `11233` | OSS upload not configured | `errorOssUploadNotConfigured` | 上传未配置 |
| `11234` | Failed to generate upload URL | `errorOssPresignFailed` | 上传地址失败 |

### OAuth / 条款 / 绑定 (17xxx)

| code | 服务端 msg（摘要） | Voice l10n | 处理建议 |
|------|-------------------|------------|----------|
| `17000` | authorize code invalid | `errorAuthorizeCodeInvalid` | 授权码无效，重试 |
| `17001` | failed to get user info | `errorOauthFailed` | IdP 凭证 / `id_token.aud` |
| `17002` | Unsupported login methods | `errorUnsupportedOAuthProvider` | 后端未开通 accountType |
| `17003` | account invalid | `errorUserInfoError` | 重新登录 |
| `17004` | validating-code has sent… | `errorVerifyCodeNotExpired` | **继续用上一封邮件验证码** |
| `17005` | oauth2 context incorrect | `errorOauthStateMismatch` | 重新发起 OAuth |
| `17006` | Authorization code not provided | `errorOauthCodeMissing` | 缺少 code |
| `17007` | state not provided | `errorOauthStateMissing` | 缺少 state |
| `17008` | email from oauth2 has existed | `errorOauthAccountNeedBind` | 邮箱已注册 → 绑定/邮箱登录 |
| `17009` | child account can not logout | `errorChildAccountCannotDelete` | 子账号不可注销 |
| `17010` | accept Terms of Service | `errorTermsAcceptanceRequired` | 须同意服务条款 |
| `17011` | third-party linked elsewhere | `errorOauthForeignIdTaken` | 第三方账号已被占用 |
| `17012` | already linked other sign-in | `errorOauthOrgAlreadyBound` | 已绑定其它登录方式 |
| `17013` | WeChat no unionid | `errorOauthWechatNoUnionid` | 开放平台 / 授权范围 |

> **注意**：`11002` 在 SenseCraft 为「账号不存在」；部分**产品业务后端**可能复用同码表示其它含义，Voice 登录页以 SenseCraft 文案为准。

---

## OpenAPI / 权威字段

接口字段以 **线上 SenseCraft authapi OpenAPI** 为准（内网或运维提供的 swagger）。  
本 Skill 与宿主 App auth 实现及 **SenseCraft authapi OpenAPI** 保持一致；冲突时：**OpenAPI + 线上行为 > 旧文档**。

**勿与产品后端 OpenAPI 混淆**：`sensecraft-respeaker-service` 等产品的 `/api/v1/user/app/login` 属于另一套网关契约；本 Skill 仅覆盖 **authapi**（`oauth/mobile`、`getEmailCode` 等，Host 见 [SKILL.md](SKILL.md) §环境）。

---

## 第三方 IdP 配置

### Google

| 类型 | 作用 | 写进 App 代码？ |
|------|------|----------------|
| **Web** | mobile `serverClientId` → `id_token.aud`；须与 SenseCraft 环境（PROD/DEV）一致 | **是**（通常 1～2 个，按环境切换） |
| **Android** | 包名 + SHA-1；Google Play Services 按**当前安装包签名**自动选用匹配的 Android OAuth 客户端 | **否**（只在 Google Cloud Console 登记） |
| **iOS** | Bundle ID + `GIDClientID` + reversed scheme | **是**（`GIDClientID` 为 iOS 客户端 ID，不是 Web ID） |

**多个 SHA-1 怎么办**

- Debug keystore、Release keystore、Google Play **App signing certificate** 的 SHA-1 **往往各不相同**。
- 在 Google Cloud Console → Credentials → **Android OAuth 客户端**：每条凭证 = **一个包名 + 一个 SHA-1** → 会得到**多个不同的 Android Client ID**（正常）。
- **不要**因为 Android Client ID 变多个就去改 App 里的 `serverClientId`；代码里始终用 **Web Client ID**。
- 缺当前安装包 SHA-1 对应的 Android 客户端 → 选账号前失败，常见 **DEVELOPER_ERROR (10)** 或 **canceled**（IdP 层，未到 `oauth/mobile`）。
- Firebase 控制台可在**同一个 Android App** 下添加多个 SHA-1；OAuth Android 客户端在 Cloud Console 仍是「一条 SHA-1 一条凭证」。

**Web Client ID ≠ iOS GIDClientID ≠ Android Client ID** — 同一 Cloud 项目下三种不同 OAuth 客户端。

勿把 Web 客户端 redirect 设为 custom URL scheme。

### GitHub OAuth App

| 项 | 说明 |
|----|------|
| Client ID | 可公开，编译注入 |
| Client Secret | **仅 SenseCraft 服务端** |
| Callback URL | 与 App `redirect_uri` 完全一致，scheme 小写 |
| Scope | `read:user user:email` |

### Apple

App ID 开启 Sign in with Apple；Xcode Capability；传 `identityToken` 为 `idToken`。

---

## Pre-login 匿名路径

以下 path 请求**不应**附带 stale access token：

- `/api/v1/auth/getEmailCode`
- `/api/v1/auth/registerByEmail`
- `/api/v1/auth/email/login`
- `/api/v1/auth/oauth/mobile`
- `/api/v1/auth/resetPassword`

---

## signOut vs logout

| 动作 | Path |
|------|------|
| 登出 | `POST /api/v1/user/signOut` |
| 注销账号 | `POST /api/v1/user/logout` |

---

## 宿主 App 集成

本 Skill **不含**具体工程路径。在宿主 App 仓库内查阅 auth 模块、路由文档与环境配置；检查清单与已知 App 索引见 [INTEGRATION.md](INTEGRATION.md)。

Agent 任务 = **本 Skill** + **宿主 App auth 源码与路由文档**。
