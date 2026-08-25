# 06 · keycapture（首次取密钥：SIP 引导 + lldb 抓取）· 精细化实现

**项目唯一需要特权的环节。** 取到一次后存 Keychain 永久复用。模块目录：`wechat_decryptor_mac/keycapture/`。

---

## 0. 为什么必须特权（不可绕开）

主密钥只在微信进程内存。macOS 读加固进程（hardened runtime、无 `get-task-allow`）内存，`task_for_pid()` 在 SIP 开启时被内核拒绝。故首次取密钥必须：A) 一次性关 SIP，或 B) 重签名微信加调试授权。取到后存 Keychain，日常零特权。

## 1. SIP 检测（keycapture/sip.py）

```python
from enum import Enum
import subprocess

class SipStatus(Enum):
    ENABLED = "enabled"; DISABLED = "disabled"; UNKNOWN = "unknown"

def sip_status() -> SipStatus:
    try:
        out = subprocess.run(["csrutil", "status"], capture_output=True,
                             text=True, timeout=5).stdout.lower()
    except Exception:
        return SipStatus.UNKNOWN
    if "disabled" in out: return SipStatus.DISABLED
    if "enabled"  in out: return SipStatus.ENABLED
    return SipStatus.UNKNOWN
```
（本机实测 `System Integrity Protection status: enabled.` → ENABLED。）

`can_debug()`：`SipStatus.DISABLED` 或（已重签名微信）时返回 True。

## 2. 取密钥总入口

```python
@dataclass
class CaptureResult:
    key: bytes
    strategy: str          # "breakpoint" | "coredump-scan" | "offset"
    fingerprint: str

def capture_key(pid: int, sample_db: Path, params: CipherParams,
                strategy: str = "auto", progress=None) -> CaptureResult:
    if sip_status() is SipStatus.ENABLED and not _wechat_is_resigned():
        raise SipEnabledError()
    if not _lldb_available():
        raise LldbUnavailableError()
    order = _strategy_order(strategy)          # ["breakpoint","coredump-scan","offset"]
    last = None
    for s in order:
        try:
            key = _STRATEGIES[s](pid, sample_db, params, progress)
            if key and verify_key_bytes(sample_db, key, params):
                return CaptureResult(key, s, fingerprint(key))
        except CaptureRecoverable as e:
            last = e; continue
    raise CaptureFailedError(detail=str(last))
```

**校验即判据**：任何策略产出的候选都必须过 `decryptor.verify_key`（页 HMAC）。把"定位准不准"与"密钥对不对"解耦——扫到就能确认，避免误报。

## 3. lldb 可用性

```python
def _lldb_available() -> bool:
    return shutil.which("lldb") is not None
```
缺失 → `LldbUnavailableError`（提示 `xcode-select --install`）。本机实测 `/usr/bin/lldb` 版本 `lldb-1600.0.39.3`。

## 4. 策略 A：断点法（首选，最稳）

在微信打开数据库时命中设置密钥的函数，直接读参数。

### 4.1 目标符号（按序尝试）
WeChat 静态链接 WCDB/SQLCipher，符号可能被裁剪。候选断点：
```text
sqlite3_key            # (db, pKey, nKey) —— nKey=32, pKey 指向密钥
sqlite3_key_v2         # (db, zDbName, pKey, nKey)
sqlcipher_codec_ctx_set_pass
```
`image lookup -rn '^sqlite3_key'` 查符号是否存在。

### 4.2 lldb Python 脚本（keycapture/lldb_script.py 生成）
```python
# 运行环境：lldb 内置 python。仅做"读内存/取参数→打印 hex"，不做验证。
import lldb, struct

def capture(debugger, pid, out_path):
    debugger.SetAsync(False)
    target = debugger.CreateTarget("")
    err = lldb.SBError()
    process = target.AttachToProcessWithID(debugger.GetListener(), pid, err)
    if not err.Success(): print("ATTACH_FAIL", err.GetCString()); return

    bp = target.BreakpointCreateByName("sqlite3_key")
    if bp.GetNumLocations() == 0:
        bp = target.BreakpointCreateByName("sqlite3_key_v2")
    process.Continue()                       # 等微信触发 DB 访问（用户打开聊天）

    thread = process.GetSelectedThread()
    frame = thread.GetFrameAtIndex(0)
    # arm64 调用约定：x0=db, x1=pKey, x2=nKey（sqlite3_key）
    #                 x0=db, x1=zDbName, x2=pKey, x3=nKey（_v2）
    regs = frame.GetRegisters()
    x1 = _reg(frame, "x1"); x2 = _reg(frame, "x2"); x3 = _reg(frame, "x3")
    # 依函数判定 pKey/nKey
    pkey, nkey = (x1, x2) if bp.name=="sqlite3_key" else (x2, x3)
    if nkey in (32,):                        # 只取 32 字节密钥
        e = lldb.SBError()
        data = process.ReadMemory(pkey, 32, e)
        if e.Success():
            open(out_path,"w").write(data.hex())
    process.Detach()
```
父进程以子进程运行（见 §7），从 `out_path` 读取 hex 候选并 `verify_key`。

### 4.3 触发 DB 访问
断点法需微信真正打开数据库。向导提示：**"抓取中，请在微信里点开任意一个聊天会话"**。命中后立即读参、detach。

## 5. 策略 B：核心转储 + 离线扫描（稳健回退）

不依赖符号。先用 lldb dump 内存，再在本工具环境（有 pycryptodome）离线扫描。

### 5.1 dump
```bash
lldb -p <pid> \
  -o "process save-core --style stack /tmp/wx_<pid>.core" \
  -o "detach" -o "quit" --batch
```
> `--style stack` 只存栈/可写区可显著减小体积；若命中率低改用完整 dump（无 `--style`）。dump 文件权限 600，用后安全删除。

### 5.2 离线扫描算法（keycapture/lldb_capture.py）
32 字节高熵密钥。**核心难点**：`verify_key` 内含 PBKDF2 256000 次（每次约毫秒级），不能对上百万候选逐一验证。用多级预筛把候选降到可验证量级：

```python
def scan_core(core: Path, sample_db: Path, params: CipherParams,
              progress=None) -> bytes | None:
    data = core.read_bytes()                 # 或 mmap 分块
    seen: set[bytes] = set()
    tested = 0; CAP = 300_000                 # 验证次数上限，防失控
    for cand in _candidates(data):
        if cand in seen: continue
        seen.add(cand)
        if not _entropy_ok(cand): continue    # 预筛（廉价）
        tested += 1
        if verify_key_bytes(sample_db, cand, params):  # 昂贵，仅对通过预筛者
            return cand
        if progress: progress(tested, CAP)
        if tested >= CAP: break
    return None

def _candidates(data: bytes):
    # 先 8 字节对齐扫描（密钥多在结构体/指针对齐处），命中率高、数量少
    for step in (8, 1):                        # 8 对齐优先，失败再逐字节
        for off in range(0, len(data) - 32, step):
            yield data[off:off+32]

def _entropy_ok(b: bytes) -> bool:
    if b.count(0) > 4: return False           # 密钥几乎无连续/多零
    distinct = len(set(b))
    if distinct < 24: return False            # 32 随机字节期望 distinct≈30
    if all(0x20 <= x < 0x7f for x in b): return False  # 排除可打印 ASCII
    # 可选：Shannon 熵阈值
    return True
```

- **预筛把验证量从"内存字节数"降到几万级**，PBKDF2 可承受。
- 阈值（`distinct>=24`、`零<=4`）可调；宁可放宽预筛也不能漏真密钥，最终以 `verify_key` 为准。
- `CAP` 命中上限 + 进度回调，避免长时间无反馈。

## 6. 策略 C：版本偏移法（可选，最快，需维护）

维护"微信版本 → 密钥相对模块基址偏移"表：
```python
KEY_OFFSETS = { "4.1.12": {"module": "WeChat", "offset": 0x......} }  # 【需实测维护】
```
lldb 取模块基址 `image list -o WeChat` + 偏移直接读 32 字节。命中率最高但每次微信更新需重测，仅作加速项，默认不启用。

## 7. 子进程协议

lldb 不嵌入主进程，避免调试器状态污染。父子约定：

```python
def _run_lldb(argv: list[str], timeout: int) -> str:
    proc = subprocess.run(["lldb", "--batch", *argv],
                          capture_output=True, text=True, timeout=timeout)
    return proc.stdout
```
- 断点法：lldb 跑生成的 python 脚本，把 32 字节 hex 写入临时文件（600 权限）；父进程读 → `verify_key` → 成功即入 Keychain → 删除临时文件。
- 核心转储法：lldb 只 dump；父进程离线扫描。
- 超时用 `app.timeoutSeconds`（默认 240）。
- 输出解析只取 hex；密钥本体不进日志，仅记 `fingerprint`。

## 8. 引导向导状态机（KeyCaptureWizard.qml，见 09 §9）

```text
[检测 SIP]
  DISABLED / 已重签名 ─▶ [就绪] ─▶ 点"一键获取密钥"
     └─▶ 提示"在微信里打开一个聊天" ─▶ 断点法；失败→核心转储法
     └─▶ 成功：显示指纹，存 Keychain，提示"可重新开启 SIP"
  ENABLED ─▶ [关 SIP 图文步骤]
     1) 关机→长按电源进入启动选项（Apple 芯片）/ 开机按 ⌘R（Intel）
     2) 实用工具→终端： csrutil disable   （一键复制）
     3) 重启回系统 →「我已完成，重新检测」
  任意态：可切"手动导入密钥" / "了解重签名方案"
```

## 9. 手动导入（贯穿各期，兜底）

```python
def import_manual_key(text: str, sample_db: Path, params: CipherParams) -> bytes:
    key = _normalize_hex(text)               # 去空格/0x/换行；接受 64 hex
    if len(key) != 32:
        raise KeyFormatError("需 64 位十六进制（32 字节）")
    if not verify_key_bytes(sample_db, key, params):
        raise KeyMismatchError()
    return key
```

## 10. 重签名备选（方案 B，仅文档，不入默认 UI）
```bash
cat > /tmp/ent.plist <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
 <key>com.apple.security.get-task-allow</key><true/>
 <key>com.apple.security.cs.disable-library-validation</key><true/>
</dict></plist>
EOF
cp -R /Applications/WeChat.app /tmp/WeChat-debug.app
codesign -f -s - --entitlements /tmp/ent.plist --deep /tmp/WeChat-debug.app
```
风险：破坏签名/公证、触发微信自校验、更新失效。仅供高级用户，提示对副本操作、风险自负。

## 11. 安全
- 只读微信内存，不写微信进程/磁盘。
- 密钥抓到→校验→入 Keychain，内存临时副本尽力清零（`ctypes.memset` 或覆盖后删引用）。
- 核心转储/临时 hex 文件 600 权限，用后 `os.remove`；转储不留存。
- 明确提示关 SIP 的安全代价，抓完 `csrutil enable`。

## 12. 单测/验证
- `sip_status` 喂假输出（enabled/disabled/其它）断言。
- `_entropy_ok` 对真密钥样本=True、对零/ASCII/低熵=False。
- `scan_core` 用"注入已知密钥的小 dump"验证能命中且不误报。
- 断点法与真机 lldb 只能关 SIP 后手动验证；`scripts/cli.py capture-key`（见 18）独立调试。
