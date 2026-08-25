# 16 · 错误目录（领域异常与用户文案）

统一异常体系。业务层抛领域异常，界面层/MCP 捕获后转成人话文案（对齐 Windows 版措辞风格），绝不向用户抛裸栈。

## 1. 异常基类与层级

```python
class WeChatDecryptorError(Exception):
    """所有领域异常基类。带 code、user_message、detail。"""
    code: str
    user_message: str          # 面向用户的中文提示
    detail: str = ""           # 面向日志的细节（脱敏）

# 环境/路径
class WeChatNotFoundError(WeChatDecryptorError): ...      # 未找到微信程序
class DataRootNotFoundError(WeChatDecryptorError): ...    # 未找到 xwechat_files
class AccountNotFoundError(WeChatDecryptorError): ...     # 无有效账号
class IncompleteDbError(WeChatDecryptorError): ...        # 三库不齐
class FullDiskAccessError(WeChatDecryptorError): ...      # TCC 未授权

# 密钥/解密
class KeyMismatchError(WeChatDecryptorError): ...         # 密钥/账号/参数不匹配
class KeyFormatError(WeChatDecryptorError): ...           # 手动导入格式错
class CipherParamsError(WeChatDecryptorError): ...        # 参数校准失败
class DecryptIOError(WeChatDecryptorError): ...           # 读写/大小异常

# 取密钥
class SipEnabledError(WeChatDecryptorError): ...          # SIP 开启无法调试
class LldbUnavailableError(WeChatDecryptorError): ...     # lldb 不可用
class CaptureFailedError(WeChatDecryptorError): ...       # 抓取未命中密钥
class WeChatNotRunningError(WeChatDecryptorError): ...    # 微信未运行

# 密钥存储
class KeychainError(WeChatDecryptorError): ...            # Keychain 读写失败

# 数据/媒体
class DecryptedDbMissingError(WeChatDecryptorError): ...  # 未先解密
class MediaUnavailableError(WeChatDecryptorError): ...    # 媒体不可用（含原因）

# MCP
class McpAuthError(WeChatDecryptorError): ...             # token 校验失败
class McpLimitError(WeChatDecryptorError): ...            # 超返回上限
```

## 2. 错误目录表

| code | 异常 | user_message（中文） | 处置建议 |
|---|---|---|---|
| `E_WECHAT_NOT_FOUND` | WeChatNotFoundError | 未找到 WeChat.app，请手动选择或先安装微信 | 手动选路径 |
| `E_DATA_ROOT` | DataRootNotFoundError | 未找到微信数据目录，请手动选择账号目录 | 手动选 xwechat_files |
| `E_NO_ACCOUNT` | AccountNotFoundError | 未发现已登录账号，请先登录微信 | 登录微信 |
| `E_DB_INCOMPLETE` | IncompleteDbError | 数据库目录不完整，缺少 {missing} | 检查账号目录 |
| `E_FDA` | FullDiskAccessError | 需在"系统设置→隐私与安全性→完全磁盘访问权限"授权本工具 | 引导授权并重启应用 |
| `E_KEY_MISMATCH` | KeyMismatchError | 密钥与账号不匹配（页 HMAC 未通过）；请确认账号目录或重新取密钥 | 换账号/重取 |
| `E_KEY_FORMAT` | KeyFormatError | 密钥格式错误，应为 64 位十六进制（32 字节） | 重新输入 |
| `E_CIPHER_PARAMS` | CipherParamsError | 无法确定加密参数，可能是不支持的微信版本 | 上报版本 |
| `E_SIP_ENABLED` | SipEnabledError | 系统完整性保护（SIP）已开启，无法读取微信内存取密钥 | 走关 SIP 向导 |
| `E_LLDB` | LldbUnavailableError | 未找到 lldb，请安装 Xcode 命令行工具（xcode-select --install） | 安装 CLT |
| `E_CAPTURE_FAIL` | CaptureFailedError | 未能抓到有效密钥；请在微信里打开一个聊天后重试 | 触发 DB 访问后重试 |
| `E_WECHAT_NOT_RUNNING` | WeChatNotRunningError | 微信未运行，取密钥需要微信处于登录状态 | 启动并登录微信 |
| `E_KEYCHAIN` | KeychainError | 钥匙串访问失败，请在弹窗中允许访问 | 允许 Keychain |
| `E_NO_DECRYPTED` | DecryptedDbMissingError | 尚未解密，请先完成解密 | 先解密 |
| `E_MEDIA` | MediaUnavailableError | {reason}（如：缓存不可预览/需重新导出/仅缩略图） | 视原因 |
| `E_MCP_AUTH` | McpAuthError | MCP 鉴权失败，请使用最新生成的配置/Token | 重新生成配置 |
| `E_MCP_LIMIT` | McpLimitError | 返回超出上限，请缩小范围或分页/导出到文件 | 用分页/导出 |

## 3. 处理约定

- 业务层：只抛异常，不打印、不弹窗。
- 界面层：`try/except WeChatDecryptorError` → 设 `app.errorText = e.user_message`，追加运行日志 `[code] detail`，必要时 `toast`。
- MCP：返回 JSON-RPC error，`code`/`message=user_message`/`data.detail`（脱敏）。
- 未预期异常（非领域异常）：记完整栈到 `logs/`，对用户显示"发生未知错误，详见运行日志"。
- 所有 `detail` 与日志：密钥脱敏（见 `docs/dev/17`）。

## 4. 与 Windows 版文案对齐

沿用 Windows 版关键提示风格（如"页面 HMAC 未通过通常优先检查账号目录是否选错"），仅把 Windows 专有名词替换为 Mac 等价（Hook→取密钥、DPAPI→Keychain）。
