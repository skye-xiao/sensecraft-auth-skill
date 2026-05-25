# SenseCraft Auth — Reference

> 补充 [SKILL.md](SKILL.md)。宿主 App 代码路径见各项目 `docs/AUTH_MODULE.md`。

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

常见错误：`11002`/`11008` 验证码问题 · `11013` 邮箱已注册

### resetPassword（JSON）

```json
{
  "email": "user@example.com",
  "code": "123456",
  "password": "<md5 hex>"
}
```

`code` 来自 getEmailCode **type=2**。常见：`11010` 码错/过期 · `11002` 账号不存在

### changePassword（JSON，已登录）

```json
{
  "oldPassword": "<md5 hex>",
  "password": "<md5 hex>"
}
```

鉴权：`Authorization: <access_token>`（无 Bearer）

### oauth/mobile

Google/Apple：`accountType`, `platform` (`ios`|`android`), `idToken`  
GitHub：`accountType`, `platform`, `code`

### 登录成功 data 常见字段

`token`, `refresh_token`, `user_id`, `account`, `email`, `nickname`, `org_id`

---

## 错误码速查

`code` 可能是 number 或 string `"0"`，比较前先归一化为 int。

| code | 含义 | App 处理 |
|------|------|----------|
| `0` | 成功 | — |
| `10000` | 服务异常 | 展示 msg |
| `10009` | accessKey 相关 | 检查 token |
| `11002` | 账号不存在 / 验证码无效 | 按场景提示 |
| `11005` | 密码错误 | |
| `11008` | 验证码错误 | |
| `11010` | 重置密码码错/过期 | |
| `11013` | 邮箱已注册 | → 引导登录 |
| `11025` | 账号冻结 | |
| `11202` | 参数错误 | 查 body 格式 |
| `11229` | 登录尝试过多 | 稍后重试 |
| `11101` | refresh 过期 | 重新登录 |
| `11102` | accessKey 无效 | 重新登录 |
| `11103` | refresh 无效 | 重新登录 |
| `17001` | OAuth 失败 | 查 IdP 凭证 / aud |
| `17002` | 不支持 accountType | 后端未开通 |
| `17003` | 用户信息错误 | refresh / 资料接口 |
| `17004` | 验证码仍有效 / 限流 | **继续用旧码** |

---

## OpenAPI / 权威字段

接口字段以 **线上 SenseCraft authapi OpenAPI** 为准（内网或运维提供的 swagger）。  
本 Skill 与宿主 `docs/AUTH_MODULE.md` 应与 OpenAPI 及当前 App 实现保持一致；冲突时：**OpenAPI + 线上行为 > 旧文档**。

---

## 第三方 IdP 配置

### Google

| 类型 | 作用 |
|------|------|
| **Web** | mobile `serverClientId` → `id_token.aud` |
| **Android** | 包名 + SHA-1（Debug + Release） |
| **iOS** | Bundle ID + `GIDClientID` + reversed scheme |

**Web Client ID ≠ iOS GIDClientID** — 同一 Cloud 项目下两个不同客户端。

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

本 Skill **不含**具体工程路径。在宿主 App 仓库内查阅人类文档，例如 reSpeaker：

- `docs/AUTH_MODULE.md` — 代码地图、路由、流程图
- `docs/THIRD_PARTY_LOGIN.md` — 本产品 Client ID / callback

Agent 任务 = 本 Skill + 宿主 `docs/AUTH_MODULE.md`。
