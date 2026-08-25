# 20 · 编码规范与工程约定

## 1. 语言与版本
- Python 3.11+（开发机 3.13）。
- 全量类型注解；`from __future__ import annotations` 视需要。
- `mypy` 严格模式（`disallow_untyped_defs`），`ruff` 负责 lint + 格式。

## 2. 分层纪律（硬性）
- 业务包 `wechat_decryptor_mac/` **不得 import Qt**（PySide6）。Qt 只出现在 `ui_qt/` 与 `models.py`。
- 依赖方向：`ui_qt`/`mcp` → 业务服务 → 基础设施。禁止反向。
- `mcp` 与 `ui_qt` 是平行前端，共用同一套业务服务，不互相依赖。

## 3. 模块与文件
- 单一职责；文件过大（> ~400 行）视为拆分信号。
- 每个模块开头 docstring 说明：做什么、怎么用、依赖什么。
- 对外类型一律 `@dataclass`（多数 `frozen=True`），字段带注解。

## 4. 纯函数优先
- 解密、解析、映射等核心逻辑写成纯函数（输入→输出，无副作用），便于单测。
- 副作用（文件 IO、subprocess、Keychain、网络）集中在明确的边界模块。

## 5. 只读/只写边界
- 原始微信目录：只读打开（`sqlite3` `?mode=ro&immutable=1`；`.dat`/媒体只读）。
- 写操作只允许发生在工具数据目录 `~/Library/Application Support/WeChatDecryptor/`。
- 提供 `assert_within_workspace(path)` 守卫，任何写入前校验路径在工作区内，越界即编程错误。

## 6. 密钥与脱敏（硬性）
- 密钥用 `bytes`；对外只暴露 `sha256(key)[:8]` 指纹。
- 禁止把密钥放进：日志、异常 message、MCP 返回、UI 属性、临时文件。
- 见 `docs/dev/17` §2 脱敏规则；`redact()` 作最后保险。

## 7. 并发
- sqlite3 连接每线程独立创建，不跨线程共享。
- GUI 主线程不阻塞；耗时进后台线程/线程池，完成用信号回主线程。
- 模型增删改只在主线程。

## 8. 异常
- 只抛 `docs/dev/16` 的领域异常，带 `code`/`user_message`/`detail`。
- 不吞异常、不 `except: pass`；未预期异常记完整栈到文件。

## 9. 路径与时间
- 一律 `pathlib.Path`。
- 时间在业务层用 epoch 秒（int），展示层格式化；注意微信时间戳单位（秒/毫秒，P1 实测确认）。

## 10. 依赖
- 用包管理器加最新稳定版；不臆造版本号。
- 尽量少引入原生依赖（解密用 pycryptodome，不引 sqlcipher 二进制），保证 `.app` 自包含。
- 新增依赖需在 `requirements-app.txt` 记录并说明用途。

## 11. 测试
- 纯逻辑必须有单测（见 `docs/dev/13`）。
- 涉及真实数据的测试只断言结构/数量，不硬编码聊天内容，不提交真实数据。
- 提交前：`pytest` + `ruff` + `mypy` + offscreen 冒烟。

## 12. 提交与文档
- 代码变更若影响接口/schema/参数，同步更新对应 `docs/dev/*`。
- `待实测回填` 项在获得真实数据后立即回填，不遗留。
- 注释只解释"为什么"，不复述"做什么"。

## 13. 安全默认
- 默认不联网（仅表情按消息自带 CDN 地址下载，且可关）。
- MCP 默认仅 `127.0.0.1` + 随机 token。
- 取密钥、解密、关 SIP 等敏感动作需用户显式确认，不静默执行。
