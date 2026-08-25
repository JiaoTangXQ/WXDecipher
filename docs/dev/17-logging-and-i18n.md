# 17 · 日志、脱敏与文案本地化

## 1. 日志

### 1.1 目标与位置
- 位置：`~/Library/Application Support/WeChatDecryptor/logs/`
- 文件：`app.log`（滚动）、`decrypt-<task_id>.log`（每次解密任务）、`mcp.log`。
- 同时把关键阶段回显到 GUI"运行日志"区（`app.logText`）与 MCP 任务日志（脱敏）。

### 1.2 级别
- `DEBUG`：仅文件，含内部细节（仍脱敏密钥）。
- `INFO`：阶段进展（发现账号、开始解密、解密某库完成、校验结果、MCP 启停）。
- `WARNING`：可恢复问题（残页、缓存缺失、网络不可用）。
- `ERROR`：领域异常 code + user_message + detail。
- `CRITICAL`：未预期异常 + 完整栈（仅文件）。

### 1.3 格式
```text
2026-08-24T17:00:00+08:00 INFO  [decrypt] task=ab12 库=session.db 页数=1234 结果=OK 指纹=9f3a1b2c
```
- 时间 ISO8601 带时区。
- 模块标签 `[decrypt] [platform] [keycapture] [mcp] [media]`。
- 任务串 `task=<id>` 贯穿一次操作。

## 2. 脱敏规则（强制）

**任何日志/UI/MCP 输出都不得包含原始密钥。**

- 密钥 → 只输出 `sha256(key)[:8]` 十六进制指纹 + 长度 + 来源。
- 禁止打印：`raw_key`、`enc_key`、`mac_key`、图片 AES key、手动输入的 hex、内存扫描候选。
- 路径脱敏：日志可含账号目录路径，但账号 wxid 建议截断（如 `wxid_ab12…cd34`）。
- 聊天正文：不写入日志；错误上下文最多记消息 id/类型，不记正文。
- 提供统一 `redact(text)` 工具：匹配 64 hex 连续串 → 替换为 `<redacted:指纹>`；作为日志最后一道保险。

## 3. 文案本地化（全中文）

### 3.1 策略
- 所有面向用户字符串集中到 `ui_qt/strings.py`（Python 侧）与 QML 顶部常量/`strings.js`（QML 侧），避免散落，便于统一维护与将来可能的多语言。
- 当前仅中文；不做 i18n 框架，但集中管理为将来留口。

### 3.2 必须替换的 Windows 文案（复用 QML 时）
见 `docs/dev/09` §5，核心：
- `Weixin.exe` → `WeChat.app`
- `Hook` / `安装 Hook` → `取密钥` / `读取内存密钥`
- `DPAPI` / `.dpapi 密钥库` → `钥匙串（Keychain）`
- `%LOCALAPPDATA%\WeChatDecryptor` → `~/Library/Application Support/WeChatDecryptor`
- `Windows 10/11 x64` → `macOS 12+`
- `退出微信重新登录（等待登录秒数）` → `按向导取密钥（首次）/ 已保存则直接解密`

### 3.3 文案风格
- 沿用 Windows 版"给小白"的直白风格与故障定位话术。
- 错误提示用 `docs/dev/16` 的 `user_message`，保持一致。

## 4. 关键动作的日志检查点

| 动作 | 必记 INFO |
|---|---|
| 自动检测 | 微信路径、版本、账号数、三库状态、SIP、架构 |
| 解密 | 每库：页数、结果、耗时、指纹；总体：成功/失败、输出目录 |
| 取密钥 | SIP 状态、策略、是否命中（不记密钥本体）、校验结果 |
| Keychain | 存/取/删 的账号与结果（不记密钥） |
| MCP | 启停、地址（端口）、生成配置路径（不记 token 明文，记 token 指纹） |
| 媒体 | 解析类型、命中/缺失原因、是否联网下载 |

## 5. 自查
- 发布前 grep 日志样本：无 64 位连续 hex；无聊天正文。
- 单测：`redact()` 对含密钥文本正确替换。
