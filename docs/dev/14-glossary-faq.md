# 14 · 术语表 · Windows→Mac 对照 · FAQ

## 1. 术语表

| 术语 | 含义 |
|---|---|
| xwechat_files | 微信 4.0 数据根目录名 |
| db_storage | 加密数据库根目录 |
| SQLCipher | 微信数据库使用的加密 SQLite 方案（分页 AES + 每页 HMAC） |
| salt | 数据库文件前 16 字节，KDF 用 |
| 主密钥 / raw_key | 32 字节数据库密钥，与账号绑定 |
| enc_key / mac_key | 由 raw_key + salt 派生的页加密密钥与 HMAC 密钥 |
| reserve | 每页尾部保留区（IV + HMAC） |
| 页 HMAC 校验 | 判定密钥/账号/参数是否正确的唯一可靠手段 |
| 图片密钥 | 解 `.dat` 图片的独立密钥/参数，≠ 主密钥 |
| hardlink | 媒体去重存储的映射（hardlink.db） |
| SIP | macOS 系统完整性保护；开启时禁止调试加固进程 |
| lldb | macOS 调试器，用于读微信内存取密钥 |
| Keychain | macOS 钥匙串，存密钥（替代 DPAPI） |
| TCC / Full Disk Access | macOS 隐私授权；访问微信容器需要 |
| MCP | 让 AI 客户端读取本地数据的本机服务协议 |

## 2. Windows → Mac 能力对照

| 维度 | Windows 版 | Mac 版 |
|---|---|---|
| 取密钥 | `wx_key.dll` 注入 Weixin.exe（一键） | lldb 读内存（需一次性关 SIP 或重签名） |
| 密钥存储 | DPAPI `*.dpapi` 文件 | Keychain |
| 路径发现 | 注册表 / App Paths / %LOCALAPPDATA% | 容器路径 / Spotlight / pgrep |
| 数据目录 | `%LOCALAPPDATA%\WeChatDecryptor` | `~/Library/Application Support/WeChatDecryptor` |
| 打包 | PyInstaller onedir exe | py2app `.app` |
| 权限门槛 | 同用户/同权限运行 | Full Disk Access + 取密钥时 SIP |
| 数据库格式 | db_storage（4.0） | **相同**（4.0/4.1） |
| 解密算法 | SQLCipher 分页 | **相同** |
| QML 界面 | Qt Quick | **相同（复用）** |
| MCP | HTTP + stdio | **相同（移植）** |

## 3. FAQ

**Q：为什么 Mac 不能像 Windows 那样一键取密钥？**
A：密钥只在微信进程内存里。Windows 靠 DLL 注入读内存；Mac 上 SIP 开启时内核禁止调试微信这类加固进程，所以首次取密钥必须关一次 SIP（或重签名微信）。取到后存 Keychain，之后永久复用、零特权。

**Q：关了 SIP 安全吗？**
A：关 SIP 会降低系统防护。方案是"进恢复模式关 → 抓一次密钥 → 立刻开回"。抓完务必 `csrutil enable`。

**Q：能不能不碰 SIP？**
A：可以走"重签名 WeChat.app 加调试授权"，但对小白不友好、微信更新即失效，仅作高级备选文档。或直接手动导入你已从别处获得的密钥。

**Q：会改动我的微信数据吗？**
A：不会。所有原始库/媒体只读打开，解密产物与缓存只写工具自己的数据目录。

**Q：为什么打开 `.app` 提示"已损坏"？**
A：未公证应用被 Gatekeeper 拦截。执行 `xattr -dr com.apple.quarantine "/Applications/微信记录工作台.app"` 或右键→打开。

**Q：支持微信 3.x 旧版 Mac 吗？**
A：本项目只做 4.0/4.1。3.x 数据库结构与密钥方案完全不同，不在范围内。

**Q：多个微信账号怎么办？**
A：工具枚举所有 `wxid_*` 账号，需你在下拉框明确选择；密钥按账号分别存 Keychain。

## 4. 合规与免责

- 工具仅用于**用户对本人可访问的本地数据**进行解密与查看，不联网上传聊天数据。
- 首次取密钥涉及调试自己机器上的微信进程；请在符合当地法律与微信用户协议的前提下，仅对自己的数据使用。
- 分发时在 README 明确免责声明与使用边界。
