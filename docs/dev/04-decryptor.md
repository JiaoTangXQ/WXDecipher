# 04 · decryptor（SQLCipher 分页解密核心）· 精细化实现

项目技术核心。微信 4.0/4.1 数据库是 **SQLCipher 4 兼容**格式，用 pycryptodome 手工实现分页解密，不依赖系统 sqlcipher。本册给出字节级算法、明文重建、流式处理、WAL 处理、真实 pycryptodome 调用与全部边界。

模块文件：`wechat_decryptor_mac/decryptor.py`

---

## 1. SQLCipher 页物理布局（字节级）

数据库文件 = N 个连续页，每页 `page_size`（微信 4.x = 4096）字节。

单页字节布局（`reserve` = 页尾保留区长度）：

```text
偏移 0 ................................. page_size
┌───────────────┬────────┬──────────┬─────────┐
│  密文区(可变)  │ IV(16) │ HMAC(H)  │ 补零对齐 │
└───────────────┴────────┴──────────┴─────────┘
                 └──────────── reserve ─────────┘
密文区长度 = page_size - reserve            （第 i>1 页）
密文区长度 = page_size - reserve - 16       （第 1 页，前 16 字节是 salt）
```

- **第 1 页特殊**：文件最前 16 字节是 **salt**（明文、随机，逐库不同）。第 1 页密文从偏移 16 开始。（调试时可用 `xxd -l16 session.db` 查看本机实际 salt，勿写入文档。）
- `reserve` 计算：`reserve = ceil((16 + H) / 16) * 16`，H = HMAC 摘要字节数。
  - SHA512：H=64 → 16+64=80 → reserve=80。
  - SHA1：H=20 → 16+20=36 → reserve=48。
  - SHA256：H=32 → 16+32=48 → reserve=48。
- 各页的 IV 位于 `page_size - reserve`，HMAC 紧随其后。

## 2. 参数模型（CipherParams）

```python
from dataclasses import dataclass
from Crypto.Hash import SHA1, SHA256, SHA512

_HASH = {"sha1": SHA1, "sha256": SHA256, "sha512": SHA512}
_HSIZE = {"sha1": 20, "sha256": 32, "sha512": 64}

@dataclass(frozen=True)
class CipherParams:
    page_size: int = 4096
    kdf_algo: str = "sha512"      # 微信4.x=SHA512；3.x=SHA1
    kdf_iter: int = 256000        # 【P1 实测回填】微信4.x 候选 256000
    hmac_algo: str = "sha512"     # 微信4.x=SHA512
    hmac_kdf_iter: int = 2        # SQLCipher FAST_KDF_ITER，固定 2
    key_size: int = 32            # AES-256
    iv_size: int = 16
    salt_mask: int = 0x3a         # mac salt = db_salt XOR 0x3a（逐字节）

    @property
    def hmac_size(self) -> int:
        return _HSIZE[self.hmac_algo]

    @property
    def reserve(self) -> int:
        raw = self.iv_size + self.hmac_size
        return (raw + 15) // 16 * 16

    @property
    def kdf_hash(self):  return _HASH[self.kdf_algo]
    @property
    def hmac_hash(self): return _HASH[self.hmac_algo]
```

候选参数集（`calibrate` 按序尝试）：

```python
CANDIDATE_PARAMS = [
    CipherParams(kdf_algo="sha512", kdf_iter=256000, hmac_algo="sha512"),  # v4-A 首选
    CipherParams(kdf_algo="sha512", kdf_iter=256000, hmac_algo="sha256"),  # v4 变体
    CipherParams(kdf_algo="sha1",   kdf_iter=64000,  hmac_algo="sha1"),    # v3 回退
    CipherParams(kdf_algo="sha512", kdf_iter=4000,   hmac_algo="sha512"),  # 低迭代变体
]
```

## 3. 密钥派生（真实 pycryptodome 调用）

```python
from Crypto.Protocol.KDF import PBKDF2
from Crypto.Hash import HMAC

def derive_keys(raw_key: bytes, salt: bytes, p: CipherParams) -> tuple[bytes, bytes]:
    assert len(raw_key) == p.key_size, "主密钥必须 32 字节"
    assert len(salt) == 16, "salt 必须 16 字节"

    # 页加密 key：raw_key 作 password、文件 salt、PBKDF2-HMAC-<kdf_algo>
    enc_key = PBKDF2(raw_key, salt, dkLen=p.key_size, count=p.kdf_iter,
                     hmac_hash_module=p.kdf_hash)

    # HMAC key：enc_key 作 password、(salt XOR 0x3a) 作 salt、迭代 2 次
    mac_salt = bytes(b ^ p.salt_mask for b in salt)
    mac_key = PBKDF2(enc_key, mac_salt, dkLen=p.key_size, count=p.hmac_kdf_iter,
                     hmac_hash_module=p.hmac_hash)
    return enc_key, mac_key
```

> 关键点：即便 `raw_key` 已是 32 字节原始密钥，SQLCipher 仍以它为 password + 文件 salt 做一次 PBKDF2 得到真正的页加密 key。这是与"直接拿 raw_key 当 AES key"的常见误解不同之处。

## 4. 单页校验与解密（字节级）

```python
import struct, hmac as _hmac
from Crypto.Cipher import AES

def process_page(page: bytes, page_no: int, enc_key: bytes, mac_key: bytes,
                 p: CipherParams) -> bytes:
    """返回该页解密后的明文（不含 salt/不含 reserve）。page_no 从 1 计。"""
    start = 16 if page_no == 1 else 0
    reserve = p.reserve
    body_end = p.page_size - reserve            # 密文区结束 = IV 起点
    iv = page[body_end:body_end + p.iv_size]
    stored_hmac = page[body_end + p.iv_size: body_end + p.iv_size + p.hmac_size]
    ciphertext = page[start:body_end]

    # --- HMAC 校验：H(mac_key, 密文 + IV + le32(page_no)) ---
    h = HMAC.new(mac_key, digestmod=p.hmac_hash)
    h.update(page[start:body_end + p.iv_size])   # 密文区 + IV
    h.update(struct.pack("<I", page_no))         # 页号 4 字节小端
    if not _hmac.compare_digest(h.digest(), stored_hmac):
        raise KeyMismatchError(page_no=page_no)

    # --- AES-256-CBC 解密 ---
    if len(ciphertext) % 16 != 0:
        raise DecryptIOError(f"page {page_no} 密文长度非16倍数")
    return AES.new(enc_key, AES.MODE_CBC, iv).decrypt(ciphertext)
```

要点：
- HMAC 覆盖范围**含 IV**，且追加**小端 4 字节页号**（SQLCipher 规范）。
- 用 `hmac.compare_digest` 常量时间比较，避免时序侧信道。
- 空闲页（全 0 页）在 SQLCipher 中 HMAC 仍成立；若遇到全 0 页（`page == b"\x00"*page_size`），SQLCipher 约定输出全 0 明文，需特判：

```python
    if page_no != 1 and page == b"\x00" * p.page_size:
        return b"\x00" * (p.page_size - reserve)   # 空闲页直通
```

（此特判放在 HMAC 之前。第 1 页不会是空闲页。）

## 5. 明文重建（整库解密）

### 5.1 明文页组装规则
- **第 1 页**：输出 = `b"SQLite format 3\x00"`（重建标准 16 字节头） + `process_page(page1)`（长度 `page_size - reserve - 16`） + `reserve` 字节占位 → 合计 `page_size`。
- **第 i>1 页**：输出 = `process_page(page_i)`（长度 `page_size - reserve`） + `reserve` 字节占位 → 合计 `page_size`。
- 占位字节：写原页的 reserve 区原样（`page[page_size-reserve:]`）即可；SQLite 忽略每页保留区内容。**写零亦可**。

> 为什么明文库仍保留 reserve：解密后的 SQLite 头偏移 20 的"每页保留字节数"由 SQLCipher 写入为 `reserve`，因此明文库每页必须保留同样长度的尾部空间，页结构才合法。该字节位于第 1 页明文内（文件偏移 20 → 落在 `process_page(page1)` 结果的第 4 字节），无需手工修改；若 P1 实测 `integrity_check` 失败，再显式将输出偏移 20 置为 `reserve`（列为回退）。

### 5.2 流式实现（避免整库读入内存）

message_0.db 可能很大，逐页读写：

```python
def decrypt_db(src: Path, dst: Path, raw_key: bytes, p: CipherParams) -> DecryptStat:
    size = src.stat().st_size
    if size % p.page_size != 0:
        # 尾部残页（常见于 -wal 合并前）：只处理完整页，残页跳过并告警
        log.warning("库大小非页整数倍，尾部 %d 字节将跳过", size % p.page_size)
    n_pages = size // p.page_size

    dst.parent.mkdir(parents=True, exist_ok=True)
    with src.open("rb") as fin, dst.open("wb") as fout:
        head = fin.read(p.page_size)
        salt = head[:16]
        enc_key, mac_key = derive_keys(raw_key, salt, p)

        # 第 1 页
        plain1 = process_page(head, 1, enc_key, mac_key, p)
        fout.write(b"SQLite format 3\x00")
        fout.write(plain1)
        fout.write(head[p.page_size - p.reserve:])

        # 其余页
        for i in range(2, n_pages + 1):
            page = fin.read(p.page_size)
            plain = process_page(page, i, enc_key, mac_key, p)
            fout.write(plain)
            fout.write(page[p.page_size - p.reserve:])
    return DecryptStat(pages=n_pages, out=dst, fingerprint=fingerprint(raw_key))
```

```python
@dataclass
class DecryptStat:
    pages: int
    out: Path
    fingerprint: str
```

### 5.3 输出位置
`~/Library/Application Support/WeChatDecryptor/decrypted/<name>_decrypted.db`。源库只读打开（`"rb"`），永不写回微信目录。

## 6. WAL / 附属文件处理

- 微信运行时 `db_storage/*/*.db-wal`、`*.db-shm` 可能存在，含尚未合并进主库的最新数据（WAL 帧同样 SQLCipher 加密）。
- **P1 策略**：只解密主 `.db`。若存在非空 `-wal`，日志告警"部分最新消息可能在 WAL 中，未纳入本次解密"。
- **改进（后续）**：解析 WAL 帧格式（每帧 = 24 字节帧头 + 一页密文），用同 `enc_key/mac_key` 逐帧解密并按 commit 记录重放到明文库；复杂度高，非 P1 范围，列 `docs/dev/11` 风险。
- 最稳做法（可选，需用户操作）：解密前确保微信已退出，触发过 checkpoint，使 WAL 并入主库。

## 7. 密钥校验与参数校准

```python
def verify_key(src: Path, raw_key: bytes, p: CipherParams) -> bool:
    """只读首页并校验 HMAC。快、不写盘。用于验证密钥/账号/参数三者匹配。"""
    with src.open("rb") as f:
        head = f.read(p.page_size)
    if len(head) < p.page_size:
        return False
    try:
        enc_key, mac_key = derive_keys(raw_key, head[:16], p)
        process_page(head, 1, enc_key, mac_key, p)
        return True
    except (KeyMismatchError, DecryptIOError):
        return False

def calibrate(src: Path, raw_key: bytes,
              candidates=CANDIDATE_PARAMS) -> CipherParams | None:
    for p in candidates:
        if verify_key(src, raw_key, p):
            return p
    return None
```

`calibrate` 产物：确定本机微信 4.1 的确切 `CipherParams`，回填本册 §2 默认值与 `docs/dev/11` §3。操作步骤见 `docs/dev/19`。

## 8. 校验通过后的完整性自检

解密后立即用标准库校验（不属于 `decryptor`，由 `verify` 做，见流程）：

```python
import sqlite3
def quick_check(dst: Path) -> bool:
    uri = f"file:{dst}?mode=ro&immutable=1"
    con = sqlite3.connect(uri, uri=True)
    try:
        return con.execute("PRAGMA integrity_check").fetchone()[0] == "ok"
    finally:
        con.close()
```

## 9. 待解密库清单与顺序

| 库 | 用途 | 期 |
|---|---|---|
| `message/message_0.db`（可能 message_N） | 消息 | P1 |
| `contact/contact.db` | 联系人/群 | P1 |
| `session/session.db` | 会话 | P1 |
| `head_image/head_image.db` | 头像 | P3 |
| `hardlink/hardlink.db` | 媒体映射 | P3 |
| `emoticon/emoticon.db` | 表情 | P3 |
| `message/message_resource.db`、`biz_message_0.db` | 资源/公众号 | 视需要 |

多分库：若存在 `message_1.db`…，全部解密，`data_adapter` 跨库合并。

## 10. 错误与边界（全量）

| 情况 | 处理 |
|---|---|
| 文件大小非页整数倍 | 告警，处理完整页，跳过尾部残页 |
| 首页 HMAC 失败 | `KeyMismatchError` → 上层提示密钥/账号/参数不符 |
| 密文长度非 16 倍 | `DecryptIOError` |
| 空闲全 0 页 | 直通输出全 0 明文（§4 特判） |
| 库为空/0 页 | `verify_key` 返回 False |
| `raw_key` 非 32 字节 | `KeyFormatError`（在导入层拦截） |
| 存在非空 -wal | 告警，仅解主库 |

全程禁止把 `raw_key/enc_key/mac_key` 写日志；仅记 `fingerprint = sha256(raw_key)[:8]`。

## 11. 单元测试（对应 13）

- `derive_keys`：给定固定 raw_key+salt+params，输出稳定（快照）。
- `process_page`：用真实 `session.db` 首页 + 正确密钥 → 不抛；错误密钥 → `KeyMismatchError`。
- `verify_key`：正确/错误密钥分别 True/False。
- `calibrate`：本机命中 v4-A（或实测参数）。
- `decrypt_db`：产物 `integrity_check=ok`、可 `.tables`、核心表行数>0。
- 空闲页特判、残页告警路径覆盖。
