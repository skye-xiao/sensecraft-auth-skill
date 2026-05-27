# IdP 完整配置 — SenseCraft Voice 参考实例

> **宿主 App**：SenseCraft Voice（`respeaker-app`）  
> **包名 / Bundle ID**：`cc.seeed.voice`  
> 其它 App 接入时复制流程，替换包名、Client ID、callback scheme；**Web PROD/DEV** 仍用 [SKILL.md](SKILL.md) 组织级 ID（与 authapi 档位绑定）。

---

## 总览：Console → App → authapi

```
Google Cloud / Apple Developer / GitHub OAuth App
        ↓
App 原生 + Flutter（取 idToken 或 code）
        ↓
POST {authDomain}/api/v1/auth/oauth/mobile
        ↓
SenseCraft token + refresh_token
```

| IdP | App 产物 | oauth/mobile 字段 |
|-----|----------|-------------------|
| Google | `id_token`（+ 本地 `accessToken`） | `accountType: google`, `platform`, **`idToken`** |
| Apple | `identityToken` | `accountType: apple`, `platform: ios`, **`idToken`** |
| GitHub | 浏览器回调 `code` | `accountType: github`, `platform`, **`code`** |

---

## 环境与 authapi（须与 Web Client 同档）

| App 环境 | authapi Base URL | Google Web `serverClientId` |
|----------|------------------|----------------------------|
| `release` / `prod` | `https://sensecraft-auth.seeed.cc/authapi/` | PROD（下表） |
| `cn` / `china` | `https://sensecraft-auth.seeed.cn/authapi/` | PROD |
| `dev` / `test` / `local` | `https://intranet-sensecap-env-expose-publicdns.seeed.cc/authapi/` | DEV（下表） |

---

## 1. Google

### 1.1 Google Cloud Console（Credentials）

Google Cloud 项目号：**721415563732**。为 `cc.seeed.voice` 创建 **三条** OAuth 2.0 Client ID（类型不同）：

| 类型 | Client ID | 配置项 | 写进 App？ |
|------|-----------|--------|------------|
| **Web application**（PROD） | `721415563732-gvsfu25trpg6buls5l6kvpf7fqhfrarg.apps.googleusercontent.com` | 无 custom scheme redirect | ✅ `serverClientId`（prod 环境） |
| **Web application**（DEV） | `721415563732-onmkav3p8u5ahq35265am22ulbm6kf9p.apps.googleusercontent.com` | 内网测试 | ✅ `serverClientId`（dev/test 环境） |
| **iOS** | `721415563732-74os5ug6p9ine9pkr32a9bvgl61tocsr.apps.googleusercontent.com` | Bundle ID = `cc.seeed.voice` | ✅ `GIDClientID` |
| **Android** | `721415563732-26oie3565ficf12vihkhr71i80l67str.apps.googleusercontent.com`（示例） | 包名 `cc.seeed.voice` + **当前签名 SHA-1** | ❌ 仅 Console |

**Android SHA-1**：Debug / Release / Play App signing **各建一条** Android Client（Client ID 可不同）。缺当前安装包 SHA-1 → **`DEVELOPER_ERROR (10)`**。

**勿混淆**：

- `serverClientId` = **Web** Client（PROD 或 DEV）
- `GIDClientID` = **iOS** Client（不是 Web ID）
- Android Client ID **不要**写进 Dart 代码

### 1.2 App 代码（SenseCraft Voice）

| 位置 | 内容 |
|------|------|
| `lib/src/core/server/config/google_oauth_config.dart` | PROD / DEV Web Client；`googleWebClientIdForAppEnv()` |
| `lib/src/features/auth/presentation/third_party_authorize_page.dart` | `GoogleSignIn.initialize(serverClientId: …)` → `authenticate()` → `idToken` |
| 覆盖 | `--dart-define=GOOGLE_SERVER_CLIENT_ID=…` |

### 1.3 iOS（`ios/Runner/Info.plist`）

```xml
<key>GIDClientID</key>
<string>721415563732-74os5ug6p9ine9pkr32a9bvgl61tocsr.apps.googleusercontent.com</string>

<!-- Google Sign-In URL callback -->
<key>CFBundleURLSchemes</key>
<array>
  <string>com.googleusercontent.apps.721415563732-74os5ug6p9ine9pkr32a9bvgl61tocsr</string>
</array>
```

（GitHub scheme `sensecraftvoice` 见 §3，与 Google scheme 分属不同 `CFBundleURLTypes` 条目。）

### 1.4 Android

| 位置 | 内容 |
|------|------|
| `android/app/build.gradle.kts` | `applicationId = "cc.seeed.voice"` |
| `android/app/src/main/res/values/strings.xml` | `default_web_client_id` = PROD Web Client（可选；Flutter 仍以 Dart 环境逻辑为准） |
| Google Cloud | Android OAuth Client：`cc.seeed.voice` + SHA-1 |

### 1.5 oauth/mobile 请求示例

```http
POST https://sensecraft-auth.seeed.cc/authapi/api/v1/auth/oauth/mobile
Content-Type: application/json

{
  "accountType": "google",
  "platform": "android",
  "idToken": "<Google Sign-In 返回的 id_token>"
}
```

**17001**：多为 `id_token.aud` 与当前环境 Web Client（PROD/DEV）不一致。

---

## 2. Apple

### 2.1 Apple Developer + Xcode

| 步骤 | 操作 |
|------|------|
| 1 | [Apple Developer](https://developer.apple.com) → Identifiers → App ID `cc.seeed.voice` → 勾选 **Sign in with Apple** |
| 2 | Xcode → Target → **Signing & Capabilities** → **Sign in with Apple** |
| 3 | 重新下载 **Provisioning Profile**（旧描述文件无 Capability 会失败） |
| 4 | **真机**测试（模拟器行为可能不一致） |

Apple **没有**像 Google 那样在 App 里填单独的 OAuth Client ID 字符串；依赖 App ID + Capability。

### 2.2 App 代码（SenseCraft Voice）

| 位置 | 内容 |
|------|------|
| `lib/src/features/auth/presentation/third_party_authorize_page.dart` | `SignInWithApple.getAppleIDCredential()` → `identityToken` 作为 `idToken` 调后端 |
| 路由 | `/login/authorize?provider=apple` |

### 2.3 oauth/mobile 请求示例

```http
POST https://sensecraft-auth.seeed.cc/authapi/api/v1/auth/oauth/mobile
Content-Type: application/json

{
  "accountType": "apple",
  "platform": "ios",
  "idToken": "<Sign in with Apple 的 identityToken JWT>"
}
```

用户选「隐藏邮箱」时可能无 email → 宿主引导 `/link-identity` 绑定邮箱。

---

## 3. GitHub

### 3.1 GitHub OAuth App（Settings → Developer settings）

| 项 | SenseCraft Voice 值 |
|----|---------------------|
| **Client ID** | `Iv23liaPMJxOyKXEs8kt` |
| **Client Secret** | **仅 SenseCraft 服务端**配置，不进 App |
| **Authorization callback URL** | `sensecraftvoice://oauth-callback`（与 App **完全一致**，scheme **小写**） |

构建覆盖 Client ID：`--dart-define=GITHUB_CLIENT_ID=…`

### 3.2 App 代码（SenseCraft Voice）

| 常量 | 值 |
|------|-----|
| `_githubClientId` | `Iv23liaPMJxOyKXEs8kt`（或 dart-define） |
| `_githubCallbackScheme` | `sensecraftvoice` |
| `_githubRedirectUri` | `sensecraftvoice://oauth-callback` |

实现：`flutter_web_auth_2` 打开 GitHub 授权页 → 回调带 `code` → `oauth/mobile` **只传 `code`**（当前不传 `code_verifier` / `redirect_uri`，除非后端 OpenAPI 要求）。

### 3.3 Android（`AndroidManifest.xml`）

```xml
<activity
    android:name="com.linusu.flutter_web_auth_2.CallbackActivity"
    android:exported="true">
    <intent-filter android:label="flutter_web_auth_github_oauth">
        <action android:name="android.intent.action.VIEW"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <category android:name="android.intent.category.BROWSABLE"/>
        <data android:scheme="sensecraftvoice" android:host="oauth-callback"/>
    </intent-filter>
</activity>
```

**必须**用 `CallbackActivity`，不要只挂在 `MainActivity`（否则 `flutter_web_auth_2` 无法完成回调）。

### 3.4 iOS（`Info.plist`）

```xml
<key>CFBundleURLSchemes</key>
<array>
  <string>sensecraftvoice</string>
</array>
```

### 3.5 oauth/mobile 请求示例

```http
POST https://sensecraft-auth.seeed.cc/authapi/api/v1/auth/oauth/mobile
Content-Type: application/json

{
  "accountType": "github",
  "platform": "android",
  "code": "<OAuth 回调 URL 中的 code，一次性>"
}
```

**17001（GitHub）**：code 过期/已用，或 GitHub App Client ID 与 SenseCraft 服务端登记不一致。

---

## 4. 端到端自检（SenseCraft Voice）

```
Google
- [ ] Console：Web PROD/DEV + iOS Client + Android Client（包名 + 当前 SHA-1）
- [ ] Dart：prod → PROD Web serverClientId；dev/test → DEV Web
- [ ] iOS：GIDClientID = iOS Client；reversed URL scheme 已配
- [ ] 选账号成功；oauth/mobile 非 17001

Apple（仅 iOS）
- [ ] App ID + Xcode Capability + 新 Provisioning Profile
- [ ] identityToken 非空；oauth/mobile 成功

GitHub
- [ ] GitHub callback URL = sensecraftvoice://oauth-callback
- [ ] Android CallbackActivity + iOS URL scheme
- [ ] oauth/mobile 只传 code；SenseCraft 服务端已配 Client Secret

通用
- [ ] authapi Host 与 Web Client 档位一致
- [ ] pre-login 请求不带 stale Authorization
```

---

## 5. 相关宿主路径

| 文件 | 说明 |
|------|------|
| `lib/src/core/server/config/google_oauth_config.dart` | Google Web PROD/DEV |
| `lib/src/features/auth/presentation/third_party_authorize_page.dart` | Google / GitHub / Apple 登录流程 |
| `lib/src/core/server/sensecraft_auth/` | authapi 客户端 |
| `ios/Runner/Info.plist` | GIDClientID、URL Schemes |
| `android/app/src/main/AndroidManifest.xml` | GitHub CallbackActivity |
| `docs/APP_ROUTES.md` | `/login/authorize?provider=` |

排错见 [examples.md](examples.md) §1–§3。
