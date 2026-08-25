# 微信聊天记录解密工具 · Mac 版 · 主设计文档（Master Spec）

- 日期：2026-08-24
- 状态：待用户评审
- 文档版本：v2（全面版）

## 0. 文档导航

本设计采用分册结构。本文件是**总纲**，详细开发规格见 `docs/dev/` 各分册：

| 分册 | 内容 |
|---|---|
| [`docs/dev/README.md`](../../dev/README.md) | 开发文档索引与阅读顺序 |
| [`docs/dev/01-architecture.md`](../../dev/01-architecture.md) | 架构、分层、数据流、进程模型 |
| [`docs/dev/02-project-structure.md`](../../dev/02-project-structure.md) | 目录结构、文件清单、依赖与版本 |
| [`docs/dev/03-platform-mac.md`](../../dev/03-platform-mac.md) | 路径发现、账号枚举、环境探测 |
| [`docs/dev/04-decryptor.md`](../../dev/04-decryptor.md) | SQLCipher 分页解密算法（含伪代码） |
| [`docs/dev/05-keyring-keychain.md`](../../dev/05-keyring-keychain.md) | 密钥存储（Keychain） |
| [`docs/dev/06-keycapture-lldb-sip.md`](../../dev/06-keycapture-lldb-sip.md) | 首次取密钥（lldb + SIP 引导） |
| [`docs/dev/07-data-adapter.md`](../../dev/07-data-adapter.md) | 只读数据适配层 |
| [`docs/dev/08-media.md`](../../dev/08-media.md) | 图片/表情/语音/视频/文件媒体解析 |
| [`docs/dev/09-ui-qml-contract.md`](../../dev/09-ui-qml-contract.md) | QML↔Python 完整接口契约 |
| [`docs/dev/10-mcp-server.md`](../../dev/10-mcp-server.md) | 本地 MCP 服务与工具 JSON schema |
| [`docs/dev/11-db-schema-appendix.md`](../../dev/11-db-schema-appendix.md) | 数据库结构附录 |
| [`docs/dev/12-build-packaging.md`](../../dev/12-build-packaging.md) | 构建、打包、分发 |
| [`docs/dev/13-testing.md`](../../dev/13-testing.md) | 测试计划与验收标准 |
| [`docs/dev/14-glossary-faq.md`](../../dev/14-glossary-faq.md) | 术语表、Windows→Mac 对照、FAQ |
| [`docs/dev/15-implementation-plan.md`](../../dev/15-implementation-plan.md) | 分期任务分解与依赖关系 |
| [`docs/dev/16-error-catalog.md`](../../dev/16-error-catalog.md) | 领域异常与用户文案目录 |
| [`docs/dev/17-logging-and-i18n.md`](../../dev/17-logging-and-i18n.md) | 日志、脱敏、文案本地化 |
| [`docs/dev/18-cli-spec.md`](../../dev/18-cli-spec.md) | 命令行规格 |
| [`docs/dev/19-p1-calibration-runbook.md`](../../dev/19-p1-calibration-runbook.md) | P1 参数/schema 校准操作手册 |
| [`docs/dev/20-coding-standards.md`](../../dev/20-coding-standards.md) | 编码规范与工程约定 |
| [`docs/dev/21-security-privacy-threat-model.md`](../../dev/21-security-privacy-threat-model.md) | 安全与隐私威胁模型 |

## 1. 项目背景

微信在 macOS 上没有提供导出或备份聊天记录的官方入口：用户自己的消息、图片、语音、文件全部只存在本地一个 **SQLCipher 加密**的数据库里，既不能整体导出，也无法跨全部历史做全文检索，更没有任何程序化访问自身数据的手段。

这带来几个真实痛点：

- **换机/重装即丢**：迁移设备、重装系统或清理数据时，多年的聊天记录很容易丢失，且无官方手段找回。
- **无法检索与归档**：想在自己的历史记录里检索一段话、按会话归档、或长期离线保存，官方客户端都做不到。
- **数据不在自己手里**：用户对自己产生的数据缺乏可控的读取和备份方式。

**本项目要解决的，就是"让用户在自己的 Mac 上、以只读方式访问并备份自己微信账号的本地数据"这一真实需求**：本地发现账号、在用户授权下获取密钥、离线解密出明文副本、只读浏览与检索会话/媒体，并可选提供本地 MCP 服务供本机 AI 客户端在授权下检索。全流程本地完成，不上传任何数据。

## 2. 目标与非目标

### 2.1 目标

1. 打通完整价值链：**发现账号 → 获取/复用密钥 → 解密数据库 → 只读查看聊天与媒体 → 本地 MCP 服务**。
2. 面向包含非技术用户在内的最终用户，可作为 `.app` 分发。
3. 严格只读原始微信数据，不上传任何数据到远程。

### 2.2 非目标

- 不支持微信 3.x（旧 Mac `Message/msg_*.db` 结构）。
- 不做 Windows 兼容。
- 不修改原始微信数据库或微信程序（重签名方案仅作可选文档，不入默认流程）。

## 3. 已核实的本机事实（设计依据）

> 以下均在开发机（本机）实测确认，作为设计的事实基础。

- 平台：macOS（darwin 25.2.0），Apple Silicon（`arm64`）。
- `lldb`：`/usr/bin/lldb`，版本 `lldb-1600.0.39.3`，可用。
- SIP：`System Integrity Protection status: enabled`（当前开启）。
- 微信：`/Applications/WeChat.app`，`CFBundleShortVersionString = 4.1.12`，`CFBundleVersion = 269365`，bundle id `com.tencent.xinWeChat`，主进程 `/Applications/WeChat.app/Contents/MacOS/WeChat`。
- 数据根：`~/Library/Containers/com.tencent.xinWeChat/Data/Documents/xwechat_files/`。
- 账号：存在多个 `wxid_*`（本机 3 个）＋ `all_users`、`Backup`。
- `db_storage` 含 16 个 SQLCipher 加密库（全部实测文件头为随机字节，非明文 `SQLite format 3`）：
  ```text
  bizchat/bizchat.db          contact/contact.db      contact/contact_fts.db
  emoticon/emoticon.db        favorite/favorite.db    favorite/favorite_fts.db
  general/general.db          hardlink/hardlink.db    head_image/head_image.db
  message/message_0.db        message/biz_message_0.db
  message/message_fts.db      message/message_resource.db
  session/session.db          sns/sns.db              solitaire/solitaire.db
  ```
- 媒体布局（账号目录下）：
  ```text
  msg/attach/<md5>/YYYY-MM/Img/<hash>_t.dat   # 图片(_t 为缩略图)
  msg/file  msg/video  msg/migrate
  business/emoticon  cache  resource  config  temp
  ```

## 4. 核心技术决策（决议记录）

| 编号 | 决策点 | 结论 | 依据 |
|---|---|---|---|
| D1 | 目标微信版本 | 仅微信 4.0/4.1（xwechat 架构） | 用户确认；本机为 4.1.12 |
| D2 | 首次取密钥方式 | 引导式一次性关 SIP + lldb 一键抓取；保留手动导入 | 用户要求"便于其他用户使用"；Mac 无零特权方案 |
| D3 | 密钥存储 | macOS Keychain（替代 Windows DPAPI） | 平台等价物 |
| D4 | 解密实现 | pycryptodome 手工 SQLCipher 分页解密（不依赖系统 sqlcipher 二进制） | 与原 Windows 实现一致，`_internal` 内含 pycryptodome |
| D5 | 界面 | 复用现有 QML，仅重写 Python 后端 `app` 桥接 | Qt Quick 跨平台 |
| D6 | MCP | **保留**全套本地 MCP（HTTP+stdio） | 用户确认保留（2026-08-24） |
| D7 | 打包 | py2app 出 `.app`；v0.1 先走档 0（不公证 + 右键打开说明），后续可升级公证 | Mac 分发标准做法；用户暂未要求投入公证 |
| D8 | 重签名微信取密钥 | 仅文档说明，不做默认 UI | 对小白不友好、随更新失效 |
| D9 | 文档位置与语言 | 全部文档放当前仓库 `docs/` 下，**全中文** | 用户确认（2026-08-24） |
| D10 | 代码项目目录名 | `wechat_decryptor_mac/`（与现有 Windows 发行版目录并列） | 默认采用，未见异议 |

### 4.1 密钥获取的硬约束（务必理解）

微信 4.0 的 SQLCipher 主密钥仅存在于运行中进程内存，磁盘无明文。macOS 上读取加固进程（hardened runtime、无 `get-task-allow` 授权）内存，在 SIP 开启时被内核禁止（`task_for_pid` 失败）。因此**首次取密钥必然需要一次特权操作**（关一次 SIP 或重签名微信），这是平台限制，任何设计都无法绕开。取到一次后密钥稳定，存 Keychain 永久复用，日常使用零特权。详见 `docs/dev/06`。

## 5. 架构总览

分层，模块单一职责、接口清晰、独立可测。完整说明见 `docs/dev/01`。

```text
┌─────────────────────────────────────────────────────────┐
│                    ui_qt (QML + app 桥接)                  │
├───────────────┬───────────────┬──────────────────────────┤
│   解密控制台   │   只读查看器   │   取密钥引导向导(Mac新增)  │
├───────────────┴───────────────┴──────────────────────────┤
│  业务服务层                                                │
│  platform_mac · decryptor · verify · data_adapter · media │
│  keyring_store · keycapture · app_paths · mcp_server       │
├───────────────────────────────────────────────────────────┤
│  基础设施：sqlite3(只读) · pycryptodome · Keychain · lldb  │
└───────────────────────────────────────────────────────────┘
```

数据流：
```text
platform_mac 发现账号/校验三库
        ↓
keyring_store 查 Keychain
   ├─ 命中且页HMAC校验通过 → 复用
   └─ 未命中 → keycapture 引导取密钥 / 手动导入 → 校验 → 存 Keychain
        ↓
decryptor 分页解密 → 明文 SQLite 写入工具数据目录
        ↓
verify 校验 → data_adapter/media 只读查看  ‖  mcp_server 本地服务
```

## 6. 目录与存储约定

- 工具数据目录：`~/Library/Application Support/WeChatDecryptor/`
  - `decrypted/`、`.media_cache/`、`.index/`、`mcp_servers.generated.json`、`logs/`
- 密钥：Keychain（服务名 `WeChatDecryptor`），不落盘明文。
- 原始微信目录：`sqlite3` 以 `?mode=ro&immutable=1` 打开；媒体 `.dat` 只读复制到缓存后解码，不改写原文件。

## 7. 实施分期（里程碑）

| 期 | 名称 | 交付 | 验收标准 |
|---|---|---|---|
| P1 | 核心闭环 | `platform_mac` + 手动导入密钥 + `decryptor` + `verify` + CLI | 命令行对本机真实库解出可被标准 sqlite3 打开的明文库，行数合理 |
| P2 | 查看器 | `data_adapter` + QML 移植 + `app` 桥接 | GUI 能列会话、看文本消息、搜索 |
| P3 | 媒体 | `media`（图片/表情/语音/视频/文件） | 图片可预览、语音可播放、视频/文件可定位 |
| P4 | 完善交付 | `keycapture`(lldb+SIP 引导) + `keyring_store`(Keychain) + `mcp_server` + py2app | 一键取密钥成功、MCP 自测通过、`.app` 可在干净机器启动 |

> P1 阶段 `keyring_store` 可用简单本地文件占位（仅开发用），P4 换 Keychain。取密钥在 P1/P2/P3 期间用"手动导入本机已抓到的密钥"支撑开发，lldb 自动抓取在 P4 完成。

## 8. 风险登记册

| 风险 | 影响 | 缓解 |
|---|---|---|
| SQLCipher 参数随微信小版本变化 | 解密失败 | P1 以本机真实库实测校准；参数集合化、可配置（见 `docs/dev/04`） |
| lldb 取密钥定位随版本漂移 | 自动取密钥失效 | 特征扫描 + 多策略回退 + 手动导入兜底（见 `docs/dev/06`） |
| SIP 开启时无法调试微信 | 首次取密钥受阻 | 引导式一次性关 SIP；文档化重签名备选 |
| 未公证 `.app` 触发 Gatekeeper | 小白打不开 | 提供右键打开说明；评估 Developer ID 公证 |
| 多账号误选 | 解错库 | 强制账号选择 UI，页 HMAC 校验绑定账号 |
| 微信 4.1 schema 与文档假设不一致 | 适配层报错 | P1 解密后 dump 真实 schema 回填 `docs/dev/11` |

## 9. 决策确认记录（2026-08-24 评审结论）

| 项 | 结论 |
|---|---|
| MCP | ✅ 保留（D6） |
| 分期顺序 | ✅ 维持 P1→P2→P3→P4 默认顺序 |
| Apple 公证 | ✅ v0.1 走档 0（不公证 + 右键打开说明）；后续按需升级（D7、`docs/dev/12` §4） |
| 项目目录名 | ✅ `wechat_decryptor_mac/`（D10） |
| 文档位置/语言 | ✅ 当前仓库 `docs/` 下，全中文（D9） |

> 全部评审项已定稿，无阻塞。后续文档以此为准继续完善；进入编码需用户另行确认（本轮仅完善文档，不写代码）。
