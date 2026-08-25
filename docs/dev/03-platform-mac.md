# 03 · platform_mac（路径发现与账号枚举）· 精细化实现

在 macOS 上通过容器固定路径 / Spotlight / 进程探测三路互补发现。**不写死**任何用户名或绝对路径。模块文件：`wechat_decryptor_mac/platform_mac.py`。

---

## 1. 数据类型（全字段）

```python
from dataclasses import dataclass, field
from pathlib import Path

@dataclass(frozen=True)
class Account:
    wxid: str                 # 目录名，如 wxid_xxxxxxxx
    account_dir: Path         # .../xwechat_files/<wxid>
    db_storage: Path          # <account_dir>/db_storage
    has_core_dbs: bool        # message_0.db + contact.db + session.db 齐全
    missing: tuple[str, ...]  # 缺失的核心库相对名（用于 dbStatus 文案）
    display_hint: str         # UI 下拉显示（先用 wxid；解密后可换真实昵称）

@dataclass(frozen=True)
class Environment:
    app_path: Path | None     # /Applications/WeChat.app
    app_version: str | None   # 4.1.12
    app_build: str | None     # 269365
    running_pid: int | None   # 主进程 WeChat 的 pid
    data_root: Path | None    # .../xwechat_files
    accounts: tuple[Account, ...]
    sip_enabled: bool
    arch: str                 # arm64 / x86_64
    full_disk_access: bool    # 是否已能读取容器
```

核心库常量：

```python
CORE_DBS = (
    "message/message_0.db",
    "contact/contact.db",
    "session/session.db",
)
CONTAINER = "com.tencent.xinWeChat"
DATA_SUBPATH = "Data/Documents/xwechat_files"
WECHAT_BUNDLE_ID = "com.tencent.xinWeChat"
```

## 2. 顶层入口

```python
def discover() -> Environment:
    app_path, ver, build = _find_app()
    pid = _find_running_pid()
    data_root = _find_data_root(app_path)
    fda = True
    try:
        accounts = tuple(enumerate_accounts(data_root)) if data_root else ()
    except PermissionError:
        accounts, fda = (), False
    return Environment(
        app_path=app_path, app_version=ver, app_build=build, running_pid=pid,
        data_root=data_root, accounts=accounts,
        sip_enabled=(sip_status() is SipStatus.ENABLED),
        arch=platform.machine(), full_disk_access=fda,
    )
```

## 3. 发现微信程序 `_find_app()`

按序探测，命中即止：

### 3.1 运行中的主进程
```python
def _find_running_pid() -> int | None:
    # 主进程可执行名恰为 WeChat，路径以 .app/Contents/MacOS/WeChat 结尾
    out = subprocess.run(
        ["pgrep", "-f", r"/WeChat.app/Contents/MacOS/WeChat$"],
        capture_output=True, text=True).stdout
    for line in out.split():
        pid = int(line)
        exe = _proc_executable(pid)          # 见 3.4
        if exe and exe.name == "WeChat" and exe.parent.name == "MacOS":
            return pid
    return None
```
> 必须排除子进程 `WeChatHelper` / `WeChatAppEx` / `wxplayer` / `crashpad_handler`（本机实测这些都在跑）。判据：可执行名恰为 `WeChat` 且父目录名为 `MacOS`。

### 3.2 由 pid 回溯 .app
```python
def _app_from_pid(pid: int) -> Path | None:
    exe = _proc_executable(pid)   # /Applications/WeChat.app/Contents/MacOS/WeChat
    if exe:
        # 回溯三级：MacOS → Contents → WeChat.app
        app = exe.parents[2]
        return app if app.suffix == ".app" else None
    return None
```

### 3.3 标准位置与 Spotlight
```python
CANDIDATES = [Path("/Applications/WeChat.app"),
              Path.home() / "Applications/WeChat.app"]

def _find_app() -> tuple[Path|None, str|None, str|None]:
    pid = _find_running_pid()
    app = _app_from_pid(pid) if pid else None
    if not app:
        app = next((c for c in CANDIDATES if c.exists()), None)
    if not app:
        app = _mdfind_bundle(WECHAT_BUNDLE_ID)   # mdfind kMDItemCFBundleIdentifier==...
    if not app:
        return None, None, None
    ver, build = _read_version(app)
    return app, ver, build
```
```python
def _mdfind_bundle(bundle_id: str) -> Path | None:
    out = subprocess.run(
        ["mdfind", f"kMDItemCFBundleIdentifier == '{bundle_id}'"],
        capture_output=True, text=True).stdout.strip().splitlines()
    return Path(out[0]) if out else None
```

### 3.4 进程可执行路径
```python
def _proc_executable(pid: int) -> Path | None:
    # ps -o comm= 给完整路径（macOS）
    out = subprocess.run(["ps", "-o", "comm=", "-p", str(pid)],
                         capture_output=True, text=True).stdout.strip()
    return Path(out) if out else None
```

### 3.5 版本读取
```python
import plistlib
def _read_version(app: Path) -> tuple[str|None, str|None]:
    info = app / "Contents/Info.plist"
    with info.open("rb") as f:
        d = plistlib.load(f)
    return d.get("CFBundleShortVersionString"), d.get("CFBundleVersion")
```
（本机：`4.1.12` / `269365`。）

## 4. 发现数据根 `_find_data_root(app_path)`

```python
def _find_data_root(app_path: Path | None) -> Path | None:
    primary = Path.home() / "Library/Containers" / CONTAINER / DATA_SUBPATH
    if primary.exists():
        return primary
    # 回退：容器 Data/ 下深度≤3 找 xwechat_files
    base = Path.home() / "Library/Containers" / CONTAINER / "Data"
    if base.exists():
        for depth1 in base.rglob("xwechat_files"):
            if depth1.is_dir() and _relative_depth(base, depth1) <= 3:
                return depth1
    return None
```
> 固定容器路径优先（本机实测命中 `~/Library/Containers/com.tencent.xinWeChat/Data/Documents/xwechat_files`）。**不扫描整块磁盘**，回退也限定容器内深度≤3。

## 5. 账号枚举 `enumerate_accounts(data_root)`

```python
EXCLUDE_DIRS = {"all_users", "Backup"}

def enumerate_accounts(data_root: Path) -> list[Account]:
    out: list[Account] = []
    for child in sorted(data_root.iterdir()):
        if not child.is_dir() or child.name in EXCLUDE_DIRS:
            continue
        if not child.name.startswith("wxid_"):
            continue
        out.append(_account_at(child))
    # 齐全的排前
    out.sort(key=lambda a: (not a.has_core_dbs, a.wxid))
    return out

def _account_at(account_dir: Path) -> Account:
    db = account_dir / "db_storage"
    missing = tuple(rel for rel in CORE_DBS if not (db / rel).exists())
    return Account(
        wxid=account_dir.name, account_dir=account_dir, db_storage=db,
        has_core_dbs=(len(missing) == 0), missing=missing,
        display_hint=account_dir.name,
    )
```
（本机 3 个 `wxid_*` 均应 `has_core_dbs=True`。）

## 6. 路径归一化 `normalize_to_account(path) -> Account | list[Account] | None`

用户可能选择不同层级，统一处理：

```python
def normalize_to_account(path: Path):
    p = path.resolve()
    # 情形1：直接是账号目录（含 db_storage）
    if (p / "db_storage").is_dir() and p.name.startswith("wxid_"):
        return _account_at(p)
    # 情形2：选了 db_storage 本身
    if p.name == "db_storage" and p.parent.name.startswith("wxid_"):
        return _account_at(p.parent)
    # 情形3：选了 db_storage 的子目录（message/contact/session）
    if p.parent.name == "db_storage" and p.parent.parent.name.startswith("wxid_"):
        return _account_at(p.parent.parent)
    # 情形4：xwechat_files 根 / 多账号上级
    accs = [a for a in enumerate_accounts(p)] if p.is_dir() else []
    if len(accs) == 1:
        return accs[0]                 # 单账号：自动选
    if len(accs) > 1:
        return accs                    # 多账号：交 UI 下拉，不自动选
    return None
```

归一化后由调用方校验 `has_core_dbs`；不齐全时用 `IncompleteDbError(missing=...)`（见 16），文案列出缺失库。

## 7. Full Disk Access 探测

访问 `~/Library/Containers/com.tencent.xinWeChat/` 在 `.app` 形态下受 TCC 限制。判定：

```python
def _probe_fda(data_root: Path | None) -> bool:
    if not data_root: return False
    try:
        next(iter(data_root.iterdir()), None)   # 能列目录即有权限
        return True
    except PermissionError:
        return False
```
无权限 → `FullDiskAccessError` → UI 引导"系统设置→隐私与安全性→完全磁盘访问权限"添加本 `.app`，并提示重启应用（TCC 授权后需重启进程生效）。详见 `docs/dev/12` §3。

## 8. UI 契约映射（→ 09）

| Environment/Account | app 属性 |
|---|---|
| `app_path` | `weixinExe`（值为 .app 路径） |
| `app_version`/`arch`/`app_path` | `runtimeText`（"WeChat 4.1.12 · arm64 · /Applications/WeChat.app"） |
| `accounts[].account_dir`（str 列表） | `accountChoices` |
| 选中账号 | `accountDir` |
| `has_core_dbs`/`missing` | `dbStatus`（"完整（3/3）"/"缺少 session.db"） |
| `sip_enabled` | 取密钥向导 `sipStatus` |

## 9. 边界与错误

| 情况 | 处理 |
|---|---|
| 未装微信 | `app_path=None`，提示手动选择/安装 |
| 未登录/无账号 | `accounts=()`，提示先登录 |
| 容器无权限 | `FullDiskAccessError`，引导授权 |
| 多账号 | 必须 UI 选择，禁止自动选 |
| 选错层级 | 归一化兜底；仍无效返回 None + 提示 |

## 10. 单测

- 临时目录构造 `xwechat_files/wxid_x/db_storage/{message,contact,session}` 结构，验证枚举、`has_core_dbs`、`missing`。
- 归一化 6 种输入（账号目录/db_storage/子目录/多账号根/单账号上级/无效）行为断言。
- `_find_running_pid` 用假 `pgrep`/`ps` 输出（依赖注入或 monkeypatch）验证主进程筛选、排除子进程。
- 本机集成：`discover()` 返回 3 账号、SIP=enabled、arch=arm64、version=4.1.12。
