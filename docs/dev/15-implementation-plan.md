# 15 · 实现任务分解（分期任务清单）

本册把 P1–P4 拆成可独立执行、可验收的任务。**本文档不含代码**，仅为开发时的任务地图。每个任务标注：依赖、产出、验收、对应分册。

> 编码前需用户另行确认启动。当前仅完善文档。

## 依赖关系总览

```text
P1 ── platform_mac ─┐
      app_paths ─────┼─▶ decryptor ─▶ verify ─▶ CLI ─▶ [P1 验收]
      (手动导入密钥) ─┘        │
                              └─▶ 11 schema 回填 + 04 参数校准
P2 ── data_adapter ─▶ models ─▶ ui_qt.app ─▶ QML ─▶ [P2 验收]
P3 ── media(hardlink→image/voice/video/emoji/avatar) ─▶ 接入 07/09 ─▶ [P3 验收]
P4 ── keychain + keycapture(lldb/SIP) + mcp(server/tools/resources/indexer) + py2app ─▶ [P4 验收]
```

## P1 · 核心闭环

| # | 任务 | 依赖 | 产出 | 验收 | 分册 |
|---|---|---|---|---|---|
| 1.1 | `app_paths` 工具数据目录解析 | — | `AppPaths` | 目录按约定创建/可写回退 | 02、01 |
| 1.2 | `platform_mac` 发现+枚举+归一化 | 1.1 | `Environment`/`Account` | 本机发现 3 账号、5 种归一化正确 | 03 |
| 1.3 | `decryptor` 参数模型+密钥派生+页处理 | — | `CipherParams`/`derive_keys`/`process_page` | 单页解密函数就位 | 04 |
| 1.4 | `decryptor.calibrate` 本机参数校准 | 1.3、手动密钥 | 确定的 `CipherParams` | 回填 04§2、11§3 | 04、19 |
| 1.5 | `decryptor.decrypt_db` 整库解密+明文重建 | 1.4 | 明文库 | `integrity_check=ok`、可列表 | 04 |
| 1.6 | `verify` 明文库校验 | 1.5 | `VerifyReport` | 三库表/行数/大小报告 | 01 |
| 1.7 | 手动导入密钥+校验 | 1.3 | `import_manual_key` | 正确密钥过、错误拒 | 06 |
| 1.8 | `scripts/cli.py` decrypt/verify | 1.5、1.6、1.7 | CLI | 命令行跑通全链路 | 18 |
| 1.9 | schema 回填 11 | 1.5 | 真实 schema | 11 各库结构补全 | 11、19 |

## P2 · 查看器

| # | 任务 | 依赖 | 产出 | 验收 | 分册 |
|---|---|---|---|---|---|
| 2.1 | `data_adapter.parsers`（分表定位/类型判定/正文解压/群发送者剥离） | 1.9 | `classify`/`parse_content` | 类型映射与解压正确 | 07 |
| 2.2 | `session_repo` 会话列表+统计 | 2.1 | `list_conversations`/`stats` | 会话排序/过滤/统计正确 | 07 |
| 2.3 | `contact_repo` 联系人+显示名 | 2.1 | `resolve_display_name` | 备注>昵称>username | 07 |
| 2.4 | `message_repo` 消息分页+会话内搜索 | 2.1 | `read_messages` | 分页/向更早/搜索正确 | 07 |
| 2.5 | `models` 两个 QAbstractListModel | 2.2–2.4 | `ConversationModel`/`MessageModel` | roles 全覆盖契约 | 09 |
| 2.6 | `ui_qt.app` 桥接+线程调度 | 2.5 | `App(QObject)` | 全部属性/槽/信号实现 | 09、01 |
| 2.7 | QML 接入+文案本地化 | 2.6 | 可运行 GUI | 无 undefined 绑定、Win 文案已替换 | 09、17 |

## P3 · 媒体

| # | 任务 | 依赖 | 产出 | 验收 | 分册 |
|---|---|---|---|---|---|
| 3.1 | `hardlink` 映射 | 1.9 | `HardlinkIndex` | md5→路径解析正确 | 08、11 |
| 3.2 | `image_dat` 解码+图片密钥 | 3.1 | `decode_dat` | 旧格式魔数反推可用 | 08 |
| 3.3 | `voice_silk` SILK→WAV | — | WAV 输出 | 可播放 | 08 |
| 3.4 | `video` 定位/缩略图降级 | 3.1 | `MediaResult` | MP4可开/仅缩略图降级 | 08 |
| 3.5 | `emoticon` 本地/CDN | 3.1 | 表情预览 | 本地优先/离线占位 | 08 |
| 3.6 | 头像 head_image | 1.9 | avatar_url | 显示/占位 | 08 |
| 3.7 | 接入 07 行填充 + 09 `resolveMedia`/`openMedia` | 3.1–3.6 | 按需刷新 | 可见气泡触发、原地刷新 | 07、09 |

## P4 · 完善交付

| # | 任务 | 依赖 | 产出 | 验收 | 分册 |
|---|---|---|---|---|---|
| 4.1 | `keyring_store` Keychain 后端 | — | 存/取/删/多账号 | 只显指纹 | 05 |
| 4.2 | `keycapture.sip` 检测 | — | `sip_status` | 解析正确 | 06 |
| 4.3 | `keycapture.lldb_capture` 抓取+校验 | 4.2、1.3 | `capture_key` | 关SIP后一键成功 | 06 |
| 4.4 | 取密钥向导 QML | 4.2、4.3 | `KeyCaptureWizard.qml` | 引导闭环 | 06、09 |
| 4.5 | `mcp.server` 传输+协议 | 2.x | HTTP/stdio | initialize/tools/list | 10 |
| 4.6 | `mcp.tools` + `resources` | 4.5、2.x、3.x | 全套工具 | 契约测试通过 | 10 |
| 4.7 | `mcp.indexer` FTS 索引 | 2.4 | `.index` | 增量构建/搜索 | 10 |
| 4.8 | MCP 接入 UI（地址/token/配置/启停） | 4.5 | MCP 卡功能 | 生成配置/复制/健康检查 | 09、10 |
| 4.9 | py2app 打包 + Full Disk Access 引导 | 全部 | `.app` | 干净机启动+授权 | 12 |

## 里程碑串联

- **M1（P1 结束）**：命令行能解出本机真实明文库 → 技术可行性坐实。
- **M2（P2 结束）**：GUI 能看文本聊天 → 可用最小闭环。
- **M3（P3 结束）**：媒体齐全 → 体验完整。
- **M4（P4 结束）**：一键取密钥 + MCP + `.app` → 可分发给其他用户。
