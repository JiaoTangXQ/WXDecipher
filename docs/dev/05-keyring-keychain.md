# 05 · keyring_store（macOS Keychain 密钥存储）· 精细化实现

替代 Windows DPAPI。密钥存 macOS 登录钥匙串，绑定当前用户，不落盘明文。模块文件：`wechat_decryptor_mac/keyring_store.py`。

---

## 1. 存储项设计

服务名固定 `WeChatDecryptor`；按 `wxid` 分账户；密钥 hex 编码为字符串值。

| 项 | account（-a） | service（-s） | 值（-w） |
|---|---|---|---|
| 数据库主密钥 | `dbkey:<wxid>` | `WeChatDecryptor` | 32 字节 → 64 hex |
| 图片密钥/参数 | `imgkey:<wxid>` | `WeChatDecryptor` | JSON 串 |

## 2. 数据类型

```python
@dataclass(frozen=True)
class KeyRecord:
    wxid: str
    key: bytes
    source: str          # "keychain" | "capture" | "manual"
    fingerprint: str     # sha256(key)[:8]

@dataclass(frozen=True)
class ImageKey:
    aes_key: bytes | None       # 新格式 AES key（可能无）
    xor_byte: int | None        # 旧格式单字节 XOR
    scheme: str                 # "xor" | "aes" | "aes+xor"
    def to_json(self) -> str: ...
    @classmethod
    def from_json(cls, s: str) -> "ImageKey": ...
```

## 3. 后端抽象（便于测试）

```python
class KeyBackend(Protocol):
    def get(self, account: str) -> str | None: ...
    def set(self, account: str, secret: str) -> None: ...
    def delete(self, account: str) -> None: ...
    def list_accounts(self) -> list[str]: ...

class KeychainBackend(KeyBackend):   # 生产：调用 security
    ...
class MemoryBackend(KeyBackend):     # 测试：内存 dict
    ...
```

`KeyringStore` 依赖 `KeyBackend`，默认注入 `KeychainBackend`。

## 4. security 命令真实调用（KeychainBackend）

### 4.1 写入（覆盖）
```python
def set(self, account: str, secret: str) -> None:
    r = subprocess.run(
        ["security", "add-generic-password",
         "-a", account, "-s", "WeChatDecryptor",
         "-w", secret, "-U",              # -U：存在则更新
         "-T", "",                        # 不预授权任何 app（每次访问弹窗，最安全）
         "-D", "WeChatDecryptor key"],    # kind 描述
        capture_output=True, text=True)
    if r.returncode != 0:
        raise KeychainError(detail=r.stderr.strip())
```
> `-w secret` 会出现在进程参数里（`ps` 可短暂看到）。更安全做法：省略 `-w`，改用 `security` 的交互输入或用 `-w -`（从 stdin 读）。实现采用 **stdin 方式**：

```python
    proc = subprocess.run(
        ["security", "add-generic-password", "-a", account,
         "-s", "WeChatDecryptor", "-U", "-T", "", "-w"],
        input=secret + "\n", capture_output=True, text=True)
```
（`-w` 不带值时 security 从 stdin 读，避免密钥进 argv。）

### 4.2 读取
```python
def get(self, account: str) -> str | None:
    r = subprocess.run(
        ["security", "find-generic-password",
         "-a", account, "-s", "WeChatDecryptor", "-w"],
        capture_output=True, text=True)
    if r.returncode == 44:      # errSecItemNotFound
        return None
    if r.returncode != 0:
        raise KeychainError(detail=r.stderr.strip())
    return r.stdout.strip()
```
（`-w` 使 `find-generic-password` 只打印密码到 stdout。首次读取 `.app` 会弹钥匙串授权。）

### 4.3 删除
```python
def delete(self, account: str) -> None:
    r = subprocess.run(
        ["security", "delete-generic-password",
         "-a", account, "-s", "WeChatDecryptor"],
        capture_output=True, text=True)
    if r.returncode not in (0, 44):
        raise KeychainError(detail=r.stderr.strip())
```

### 4.4 列举本工具的项
```python
def list_accounts(self) -> list[str]:
    r = subprocess.run(
        ["security", "dump-keychain"], capture_output=True, text=True)
    # 解析 svce = "WeChatDecryptor" 的条目，取 acct 值
    return _parse_accounts(r.stdout, service="WeChatDecryptor")
```
> `dump-keychain` 输出量大且需授权；仅用于"密钥库管理"界面，非热路径。

## 5. KeyringStore 高层接口

```python
class KeyringStore:
    def __init__(self, backend: KeyBackend | None = None):
        self.backend = backend or KeychainBackend()

    def load_key(self, wxid: str) -> KeyRecord | None:
        hexs = self.backend.get(f"dbkey:{wxid}")
        if not hexs: return None
        key = bytes.fromhex(hexs)
        return KeyRecord(wxid, key, "keychain", fingerprint(key))

    def save_key(self, wxid: str, key: bytes, source: str) -> KeyRecord:
        assert len(key) == 32
        self.backend.set(f"dbkey:{wxid}", key.hex())
        return KeyRecord(wxid, key, source, fingerprint(key))

    def delete_key(self, wxid: str) -> None:
        self.backend.delete(f"dbkey:{wxid}")

    def list_stored(self) -> list[str]:
        return [a.split(":",1)[1] for a in self.backend.list_accounts()
                if a.startswith("dbkey:")]

    def load_image_key(self, wxid: str) -> ImageKey | None:
        s = self.backend.get(f"imgkey:{wxid}")
        return ImageKey.from_json(s) if s else None

    def save_image_key(self, wxid: str, img: ImageKey) -> None:
        self.backend.set(f"imgkey:{wxid}", img.to_json())
```

## 6. 与解密流程衔接

```python
rec = store.load_key(wxid)
if rec and decryptor.verify_key(session_db, rec.key, params):
    key = rec.key                      # 复用，零特权
else:
    key = keycapture.capture_key(...).key   # 或 import_manual_key(...)
    if decryptor.verify_key(session_db, key, params):
        store.save_key(wxid, key, source)
    else:
        raise KeyMismatchError()
```

## 7. 授权弹窗与用户体验

- `.app` 首次读/写会弹"WeChatDecryptor 想使用钥匙串中的机密信息"。UI 需提示这是正常现象，可选"始终允许"。
- `-T ""`（不预授权任何程序）最安全但每次弹窗；若体验优先，可改 `-T <app 路径>` 预授权本 `.app`（权衡见 21 威胁模型，默认取安全）。

## 8. 错误映射（→ 16）
| security 返回码 | 含义 | 处理 |
|---|---|---|
| 0 | 成功 | — |
| 44 | 未找到项 | `get` 返回 None；`delete` 视为成功 |
| 45 | 交互被拒/取消 | `KeychainError`（提示允许访问） |
| 其它非 0 | 失败 | `KeychainError(detail=stderr)` |

## 9. UI 契约映射（→ 09）
- `keyStatus`：`未获取`/`已保存，可复用`/`正在校验`/`已导入`。
- `keyFingerprint`：`fingerprint`。
- `keyStorePath`：Mac 显示 `钥匙串：WeChatDecryptor`；`chooseKeyStore()` 打开"密钥库管理"（列 `list_stored()`）或置灰。
- `imageKeyStatus`/`imageKeyFingerprint`/`imageKeyXor`：图片密钥状态。

## 10. 单测
- 用 `MemoryBackend` 测 `KeyringStore` 存/取/删/列/多账号/图片密钥往返。
- `KeychainBackend` 走真实 `security`：集成/手动测（CI 跳过）。
- 断言任何路径都不把原始 key 写日志（配合 17 脱敏）。
