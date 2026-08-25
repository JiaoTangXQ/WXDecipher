# 01 · 架构与数据流

## 1. 分层

三层：**界面层（ui_qt/QML）→ 业务服务层 → 基础设施**。业务层不 import Qt，可脱离界面用 CLI/测试直接调用。

```text
界面层     ui_qt/app.py (QObject 桥接) + qml/*.qml
             │  只做：属性/信号/槽 ↔ 业务服务调用 + 线程调度
             ▼
业务服务层  platform_mac  decryptor  verify  data_adapter  media
            keyring_store keycapture app_paths mcp_server  models
             │  纯 Python，可单测，无 Qt 依赖（models 除外由 UI 包装）
             ▼
基础设施    sqlite3(ro) · pycryptodome · Keychain(security) · lldb · py2app
```

**依赖方向**：界面层 → 业务层 → 基础设施。禁止反向依赖。`mcp_server` 与 `ui_qt` 是两个平行的"前端"，都消费同一套业务服务。

## 2. 模块职责与接口边界

| 模块 | 单一职责 | 关键入口（签名见各分册） | 依赖 |
|---|---|---|---|
| `app_paths` | 解析工具数据目录、输出/缓存/日志路径 | `AppPaths.resolve() -> AppPaths` | 标准库 |
| `platform_mac` | 发现微信程序、账号、校验三库 | `discover() -> Environment`；`enumerate_accounts(root) -> list[Account]` | 标准库 |
| `keyring_store` | 主/图片密钥的 Keychain 读写 | `load_key(account) -> KeyRecord|None`；`save_key(account, key)` | `security` |
| `keycapture` | SIP 检测 + lldb 取密钥 | `sip_status() -> SipStatus`；`capture_key(pid) -> bytes` | lldb |
| `decryptor` | SQLCipher 分页解密、页 HMAC 校验 | `verify_key(db, key) -> bool`；`decrypt_db(src, dst, key)` | pycryptodome |
| `verify` | 明文库结构/行数校验 | `verify_decrypted(dir) -> VerifyReport` | sqlite3 |
| `data_adapter` | 会话/联系人/消息只读查询 | `Repository(decrypted_dir)`（见 07） | sqlite3 |
| `media` | 媒体定位与解码 | `MediaResolver(account, cache)`（见 08） | pillow, pysilk |
| `models` | Qt 列表模型包装 data_adapter | `ConversationModel`, `MessageModel` | PySide6 |
| `ui_qt.app` | QML 桥接、线程调度 | `App(QObject)`（见 09） | PySide6 |
| `mcp_server` | 本地 MCP HTTP/stdio | `serve_http()`, `serve_stdio()`（见 10） | 标准库 |

## 3. 进程与线程模型

- **GUI 主线程**：只跑 Qt 事件循环与 QML 渲染。禁止在主线程做解密/IO/lldb 等阻塞操作。
- **工作线程池**：解密、数据库查询、媒体解码、图片密钥扫描、索引构建都在 `QThreadPool`/`concurrent.futures` 后台执行，完成后通过信号回主线程更新属性/模型。
- **lldb 取密钥**：以子进程方式调用（`subprocess` 跑 lldb 脚本），避免把调试器嵌进主进程；结果经 stdout 解析（只回传密钥指纹到日志，密钥本体直接进 Keychain）。
- **MCP HTTP**：独立后台线程/子进程，绑定 `127.0.0.1` 随机端口；随主程序生命周期启停。
- **MCP stdio**：由 AI 客户端拉起独立进程（`python -m wechat_decryptor_mac.mcp --mode stdio --workspace <dir>`）。

线程安全：sqlite3 连接**每线程独立**打开（只读、immutable），不跨线程共享 connection。

## 4. 端到端时序（解密→查看）

```text
用户点"自动检测本机"
  App.autoDetect() ── 后台 ──▶ platform_mac.discover()
                                     └─▶ Environment{app_path, accounts[], data_root}
  ◀── 信号回填 ── accountChoices / weixinExe / dbStatus

用户点"开始解密" App.startDecrypt() ── 后台 ──▶
  1) keyring_store.load_key(account)
     命中 → decryptor.verify_key(session.db, key)
        通过 → 直接用
     未命中/不通过 →
        keycapture 引导（P4）/ 手动导入 → verify_key → keyring_store.save_key
  2) for db in [message_0, contact, session, ...]:
        decryptor.decrypt_db(src, decrypted/<name>_decrypted.db, key)
  3) verify.verify_decrypted(decrypted_dir) → VerifyReport
  ◀── 日志/状态/keyFingerprint/dbStatus 回填；成功后可 showViewer()

用户进入查看器 App.showViewer()
  data_adapter.Repository(decrypted_dir) 打开明文库
  ConversationModel 填充 ◀── list_conversations()
  选中会话 → MessageModel 填充 ◀── read_messages(username, page)
  可见气泡 onCompleted → App.resolveMedia(index) ── 后台 ──▶ media.resolve() → 回填模型行
```

## 5. 错误处理策略

- 业务层抛领域异常（`DecryptError`, `KeyMismatchError`, `AccountNotFoundError`, `SipEnabledError` 等），不吞异常。
- 界面层捕获领域异常 → 转为 `errorText` + 运行日志人话提示（对齐 Windows 文案风格），不弹裸栈。
- 只读/只写边界违例（试图写原始目录）视为编程错误，直接 assert 失败于开发期。
- 关键动作（解密、取密钥、MCP 启停）全程写 `logs/`，密钥脱敏。

## 6. 与 Windows 版的架构对应

| Windows | Mac | 说明 |
|---|---|---|
| `wx_key.dll` 注入 Weixin.exe | `keycapture` lldb 读内存 | 取密钥机制替换 |
| DPAPI `*.dpapi` | Keychain | 密钥存储替换 |
| 注册表/App Paths/%LOCALAPPDATA% | `platform_mac` + `app_paths` | 路径发现替换 |
| `微信解密核心.exe`（worker） | 同进程后台线程 + `mcp` 子进程模式 | Mac 不强制拆双 exe |
| PyInstaller onedir | py2app `.app` | 打包替换 |
| QML 界面 | **原样复用** | 无需改 |
| 解密/适配/MCP 业务 | **移植** | 逻辑一致 |
