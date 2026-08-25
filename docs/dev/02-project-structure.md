# 02 · 项目结构与依赖

## 1. 目录结构

项目根目录：`wechat_decryptor_mac/`（位于仓库根）。

```text
wechat_decryptor_mac/
├── README.md                     # 用户/开发者入口
├── requirements-app.txt          # 运行依赖（含版本）
├── requirements-dev.txt          # 开发/测试依赖
├── setup.py                      # py2app 打包配置
├── pyproject.toml                # 构建元数据、ruff/mypy 配置
│
├── wechat_decryptor_mac/         # 业务包（无 Qt 依赖，可单测）
│   ├── __init__.py
│   ├── app_paths.py              # 工具数据目录解析
│   ├── platform_mac.py           # 路径发现、账号枚举、环境探测（见 03）
│   ├── decryptor.py              # SQLCipher 分页解密（见 04）
│   ├── verify.py                 # 明文库校验
│   ├── keyring_store.py          # Keychain 密钥存储（见 05）
│   ├── keycapture/               # 首次取密钥（见 06）
│   │   ├── __init__.py
│   │   ├── sip.py                # csrutil 状态检测
│   │   ├── lldb_capture.py       # 子进程调用 lldb 抓密钥
│   │   └── lldb_script.py        # 生成的 lldb 脚本模板
│   ├── data_adapter/             # 只读数据层（见 07）
│   │   ├── __init__.py
│   │   ├── repository.py         # 统一查询入口
│   │   ├── session_repo.py       # session.db 会话
│   │   ├── contact_repo.py       # contact.db 联系人/群
│   │   ├── message_repo.py       # message_*.db 消息
│   │   └── parsers.py            # 消息 XML/zstd/protobuf 解析
│   ├── media/                    # 媒体解析（见 08）
│   │   ├── __init__.py
│   │   ├── resolver.py           # 媒体统一解析入口
│   │   ├── image_dat.py          # .dat 图片解码 + 图片密钥
│   │   ├── voice_silk.py         # SILK→WAV
│   │   ├── emoticon.py           # 表情（本地缓存/CDN）
│   │   ├── video.py              # 视频/缩略图
│   │   └── hardlink.py           # hardlink.db 媒体路径映射
│   └── mcp/                      # MCP 服务（见 10）
│       ├── __init__.py
│       ├── server.py             # JSON-RPC + HTTP/stdio 传输
│       ├── tools.py              # 工具实现
│       ├── resources.py          # Resource 实现
│       └── indexer.py            # 搜索索引（SQLite/FTS）
│
├── ui_qt/                        # 界面层
│   ├── full_main.py              # 主入口（对应 FullMain.qml）
│   ├── app.py                    # App(QObject) 桥接（见 09）
│   ├── models.py                 # ConversationModel / MessageModel
│   ├── full_smoke_test.py        # 无窗口冒烟
│   └── qml/                      # Qt Quick 界面
│       ├── FullMain.qml
│       ├── Main.qml
│       └── components/MessageBubble.qml
│
├── scripts/
│   ├── cli.py                    # 命令行解密/校验（P1 交付）
│   └── mcp_server.py             # MCP stdio 启动脚本
│
├── resources/                    # 图标、占位图、引导图
│   └── icon.icns
│
├── tests/                        # 单测/集成测试（见 13）
│   ├── fixtures/
│   ├── test_platform_mac.py
│   ├── test_decryptor.py
│   ├── test_verify.py
│   ├── test_data_adapter.py
│   └── test_mcp.py
│
└── docs/ -> (仓库根 docs/)        # 本设计文档
```

## 2. QML 界面说明

`ui_qt/qml/` 下的界面文件：

- `FullMain.qml`（主界面：解密控制台 + 查看器双页）
- `Main.qml`（精简查看器，可选保留）
- `components/MessageBubble.qml`（消息气泡）

QML 文案统一使用 macOS 语境（见 09 §5），结构/布局/主题保持一致。新增取密钥向导 QML：`components/KeyCaptureWizard.qml`（P4）。

## 3. 依赖与版本（`requirements-app.txt`）

> 版本以安装时最新稳定版为准，下列为建议下限参考。

```text
PySide6>=6.7            # Qt Quick/QML（Qt6）
pycryptodome>=3.23.0    # AES/HMAC/PBKDF2，SQLCipher 手工解密
pillow>=12.0.0          # 图片解码
silk-python>=0.2.8      # SILK 语音解码（pysilk）
zstandard>=0.25.0       # 消息正文 zstd 解压
```

开发依赖（`requirements-dev.txt`）：

```text
pytest>=8
ruff>=0.6
mypy>=1.11
py2app>=0.28            # 打包
```

系统依赖：
- macOS 12+（建议 13+）。
- `lldb`（随 Xcode Command Line Tools，`xcode-select --install`）——仅取密钥用。
- `security` 命令（系统自带）——Keychain 读写。

> 不使用 `pysqlcipher3`/系统 `sqlcipher` 二进制：解密全部用 pycryptodome 手工实现，避免额外原生依赖，保证 `.app` 自包含。

## 4. 运行方式

```bash
# 源码模式（开发）
cd wechat_decryptor_mac
python -m pip install -r requirements-app.txt
python ui_qt/full_main.py

# 命令行解密（P1）
python scripts/cli.py decrypt --account <账号目录> --key <hex 或 keychain> --out <输出目录>
python scripts/cli.py verify --dir <输出目录>

# 无窗口冒烟
QT_QPA_PLATFORM=offscreen python ui_qt/full_smoke_test.py --demo
```

## 5. 命名与编码约定

- 包名 `wechat_decryptor_mac`，模块小写下划线。
- 所有对外类型用 `@dataclass`（`Environment`、`Account`、`KeyRecord`、`VerifyReport`、`ConversationRow`、`MessageRow`、`MediaResult` 等），字段带类型注解。
- 时间统一用 epoch 秒（int）在业务层流转，展示层格式化。
- 二进制密钥用 `bytes`；对外展示用 `sha256(key)[:8]` 十六进制指纹。

## 6. app_paths（工具数据目录解析）· 精细化实现

模块文件：`wechat_decryptor_mac/app_paths.py`。集中管理工具**可写数据目录**（`~/Library/Application Support/WeChatDecryptor`）。业务层任何写操作只允许落在这些路径内。

```python
from dataclasses import dataclass
from pathlib import Path
import os, tempfile

APP_DIR_NAME = "WeChatDecryptor"

@dataclass(frozen=True)
class AppPaths:
    root: Path                 # ~/Library/Application Support/WeChatDecryptor
    decrypted: Path            # root/decrypted
    media_cache: Path          # root/.media_cache
    index: Path                # root/.index
    logs: Path                 # root/logs
    exports: Path              # root/exports
    mcp_config: Path           # root/mcp_servers.generated.json

    @classmethod
    def resolve(cls) -> "AppPaths":
        root = cls._writable_root()
        p = cls(root=root,
                decrypted=root/"decrypted", media_cache=root/".media_cache",
                index=root/".index", logs=root/"logs", exports=root/"exports",
                mcp_config=root/"mcp_servers.generated.json")
        for d in (p.root, p.decrypted, p.media_cache, p.index, p.logs, p.exports):
            d.mkdir(parents=True, exist_ok=True)
        return p

    @staticmethod
    def _writable_root() -> Path:
        # 首选 Application Support；不可写时多级回退
        candidates = [
            Path.home()/"Library/Application Support"/APP_DIR_NAME,
            Path.home()/f".{APP_DIR_NAME.lower()}",
            Path(tempfile.gettempdir())/APP_DIR_NAME,
        ]
        for c in candidates:
            try:
                c.mkdir(parents=True, exist_ok=True)
                probe = c/".w"; probe.write_text("ok"); probe.unlink()
                return c
            except OSError:
                continue
        raise RuntimeError("找不到可写的工具数据目录")

def assert_within_workspace(path: Path, paths: AppPaths) -> None:
    """任何写操作前调用；越界即编程错误。"""
    rp = path.resolve()
    allowed = [paths.root]
    if not any(str(rp).startswith(str(a.resolve())) for a in allowed):
        raise AssertionError(f"写入越界（非工作区）：{rp}")
```

要点：
- `decrypted/`、`.media_cache/`、`.index/`、`logs/`、`exports/`、`mcp_servers.generated.json` 全在 `root` 下，删 `root` 即彻底清理（隐私，见 21）。
- `assert_within_workspace` 是只读/只写边界的强制守卫（见 20 §5），媒体解码、导出、索引写盘前都要过。
- 回退顺序：`Application Support` → 家目录隐藏目录 → 临时目录。

## 7. verify（明文库校验）· 精细化实现

模块文件：`wechat_decryptor_mac/verify.py`。解密后只读校验，是 P1 验收的核心判据。

```python
from dataclasses import dataclass, field
from pathlib import Path
import sqlite3

@dataclass
class DbReport:
    name: str                  # message_0_decrypted.db
    ok: bool                   # integrity_check == "ok"
    tables: list[str] = field(default_factory=list)
    row_counts: dict[str, int] = field(default_factory=dict)
    size_bytes: int = 0
    error: str = ""

@dataclass
class VerifyReport:
    dbs: list[DbReport]
    @property
    def all_ok(self) -> bool: return all(d.ok for d in self.dbs)

CORE_DECRYPTED = ("message_0_decrypted.db", "contact_decrypted.db", "session_decrypted.db")

def verify_decrypted(decrypted_dir: Path,
                     names: tuple[str, ...] = CORE_DECRYPTED) -> VerifyReport:
    reports = []
    for name in names:
        db = decrypted_dir / name
        reports.append(_verify_one(db))
    return VerifyReport(dbs=reports)

def _verify_one(db: Path) -> DbReport:
    if not db.exists():
        return DbReport(name=db.name, ok=False, error="文件不存在")
    r = DbReport(name=db.name, ok=False, size_bytes=db.stat().st_size)
    try:
        con = sqlite3.connect(f"file:{db}?mode=ro&immutable=1", uri=True)
        con.row_factory = sqlite3.Row
        r.ok = con.execute("PRAGMA integrity_check").fetchone()[0] == "ok"
        r.tables = [row[0] for row in con.execute(
            "SELECT name FROM sqlite_master WHERE type='table' ORDER BY name")]
        for t in r.tables:
            try:
                r.row_counts[t] = con.execute(f'SELECT count(*) FROM "{t}"').fetchone()[0]
            except sqlite3.DatabaseError:
                r.row_counts[t] = -1        # 单表损坏不影响整体报告
    except sqlite3.DatabaseError as e:
        r.error = str(e)                    # 通常是密钥/参数错导致的非法明文库
    finally:
        try: con.close()
        except Exception: pass
    return r
```

要点：
- 只读、`immutable=1` 打开；绝不写。
- `integrity_check` 是"解密是否真正正确"的权威判据：密钥/参数错会解出结构非法的字节流，`sqlite3` 打开或 `integrity_check` 失败。
- 逐表行数，单表损坏降级为 `-1`，不中断整体报告。
- `VerifyReport.all_ok` 供 CLI 退出码与 UI `dbStatus`。

## 8. 跨模块公开 API 索引（速查）

一页速查所有业务模块的公开类型与入口（详见各分册）。

### 数据类（@dataclass）
| 类型 | 模块 | 分册 |
|---|---|---|
| `AppPaths` | app_paths | 02§6 |
| `Account` / `Environment` | platform_mac | 03 |
| `CipherParams` / `DecryptStat` | decryptor | 04 |
| `DbReport` / `VerifyReport` | verify | 02§7 |
| `KeyRecord` / `ImageKey` | keyring_store | 05 |
| `CaptureResult` / `SipStatus` | keycapture | 06 |
| `ConversationRow` / `ContactRow` / `MessageRow` / `Stats` | data_adapter | 07 |
| `MediaResult` / `HardlinkIndex` | media | 08 |
| `RpcRequest` / `RpcError` / `Task` | mcp | 10 |

### 关键入口函数/方法
| 入口 | 模块 |
|---|---|
| `AppPaths.resolve()` / `assert_within_workspace()` | app_paths |
| `discover()` / `enumerate_accounts()` / `normalize_to_account()` | platform_mac |
| `derive_keys()` / `process_page()` / `decrypt_db()` / `verify_key()` / `calibrate()` | decryptor |
| `verify_decrypted()` | verify |
| `KeyringStore.load_key/save_key/delete_key/list_stored/load_image_key/save_image_key` | keyring_store |
| `sip_status()` / `capture_key()` / `import_manual_key()` | keycapture |
| `Repository.list_conversations/list_contacts/group_members/read_messages/read_message/search_messages/stats` | data_adapter |
| `MediaResolver.resolve()` / `HardlinkIndex.resolve()` | media |
| `App(QObject)` + 全部 Property/Slot/Signal | ui_qt.app |
| `ConversationModel` / `MessageModel` | ui_qt.models |
| `McpServer.handle()` / `serve_http()` / `serve_stdio()` / `build_index()` | mcp |

### 领域异常
全部继承 `WeChatDecryptorError`，清单与错误码见 `docs/dev/16`。
