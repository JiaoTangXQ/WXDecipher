# 开发文档索引（微信记录工具 Mac 版）

本目录是 Mac 版的**开发级规格**，配合主设计文档一起阅读：

- 主设计（总纲）：[`../superpowers/specs/2026-08-24-wechat-decryptor-mac-design.md`](../superpowers/specs/2026-08-24-wechat-decryptor-mac-design.md)

## 阅读顺序

1. [`01-architecture.md`](01-architecture.md) — 先建立全局架构与数据流认知。
2. [`02-project-structure.md`](02-project-structure.md) — 目录、文件清单、依赖版本。
3. [`03-platform-mac.md`](03-platform-mac.md) — 路径发现（一切的入口）。
4. [`04-decryptor.md`](04-decryptor.md) — 解密核心（项目技术核心）。
5. [`05-keyring-keychain.md`](05-keyring-keychain.md) — 密钥存储。
6. [`06-keycapture-lldb-sip.md`](06-keycapture-lldb-sip.md) — 首次取密钥。
7. [`07-data-adapter.md`](07-data-adapter.md) — 只读数据层。
8. [`08-media.md`](08-media.md) — 媒体解析。
9. [`09-ui-qml-contract.md`](09-ui-qml-contract.md) — 界面接口契约。
10. [`10-mcp-server.md`](10-mcp-server.md) — MCP 服务。
11. [`11-db-schema-appendix.md`](11-db-schema-appendix.md) — 数据库结构附录。
12. [`12-build-packaging.md`](12-build-packaging.md) — 构建打包。
13. [`13-testing.md`](13-testing.md) — 测试与验收。
14. [`14-glossary-faq.md`](14-glossary-faq.md) — 术语、Windows→Mac 对照、FAQ。
15. [`15-implementation-plan.md`](15-implementation-plan.md) — 分期任务分解与依赖关系。
16. [`16-error-catalog.md`](16-error-catalog.md) — 领域异常与用户文案目录。
17. [`17-logging-and-i18n.md`](17-logging-and-i18n.md) — 日志、脱敏、文案本地化。
18. [`18-cli-spec.md`](18-cli-spec.md) — 命令行规格。
19. [`19-p1-calibration-runbook.md`](19-p1-calibration-runbook.md) — P1 参数/ schema 校准操作手册。
20. [`20-coding-standards.md`](20-coding-standards.md) — 编码规范与工程约定。
21. [`21-security-privacy-threat-model.md`](21-security-privacy-threat-model.md) — 安全与隐私威胁模型。

## 约定

- 语言：Python 3.11+（开发机可用 3.13）。
- 代码风格：类型注解齐全；模块内单一职责；纯函数优先，副作用集中在边界。
- 所有涉及"密钥/明文"处：日志只输出长度、来源、SHA-256 指纹前缀，绝不输出原始密钥。
- 所有对原始微信目录的访问：只读；写操作只允许发生在工具数据目录内。
- 文档中标注 `【待实测回填】` 的字段，需在 P1 解密成功后用真实 schema/参数更新。

## 术语速记

- **账号目录**：`.../xwechat_files/wxid_xxx/`，其下含 `db_storage/`。
- **db_storage**：微信 4.0 加密数据库根目录。
- **主密钥**：解 SQLCipher 库的 32 字节 key；与账号绑定。
- **图片密钥**：解 `.dat` 图片的独立 key/参数，≠ 主密钥。
- **页 HMAC 校验**：验证密钥是否正确、账号是否匹配的手段。
