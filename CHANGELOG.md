# Changelog

本仓库遵循 [Keep a Changelog](https://keepachangelog.com/) 风格。版本号见 Git tag。

## [Unreleased]

### Added

- Google：多 SHA-1 / 多 Android Client ID 与 Web `serverClientId` 分工（reference §Google、SKILL §Google、examples §1）
- Apple Sign in 排错示例（examples §2）
- `registerByEmail` / `resetPassword` / `changePassword` 请求体（reference）
- 扩展错误码表（10009、11102、17003 等）
- OpenAPI 权威来源说明（reference）
- GitHub callback 通用写法，具体 URL 指向宿主文档

### Changed

- 宿主集成说明：指向 auth 源码与 `APP_ROUTES.md`（替代已移除的 `AUTH_MODULE.md` 引用）
- examples 重组为 9 节（含注册流程、11013）

## [1.0.0] - 2026-05-25

### Added

- 初始发布：`SKILL.md`、`reference.md`、`examples.md`
- SenseCraft authapi 范围：邮箱登录/注册、OAuth（Google/Apple/GitHub）、token 刷新
- 排错决策树、新增 OAuth checklist
- 团队安装说明（`~/.cursor/skills`、submodule、monorepo 链接）

[Unreleased]: https://github.com/Seeed-Studio/sensecraft-auth-skill/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/Seeed-Studio/sensecraft-auth-skill/releases/tag/v1.0.0
