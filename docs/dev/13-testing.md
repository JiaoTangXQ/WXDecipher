# 13 · 测试计划与验收标准

## 1. 分层测试策略

| 层 | 手段 | 可 CI |
|---|---|---|
| 纯业务单元（decryptor 参数、parsers、platform 归一化、classify） | pytest 单测 | 是 |
| 数据层查询 | 对 P1 解出的脱敏小样本库做只读查询断言 | 是（带 fixture） |
| Keychain / lldb / SIP | 抽象后端 + 内存实现单测；真实后端手动/集成 | 部分 |
| GUI | `QT_QPA_PLATFORM=offscreen` 冒烟 | 是 |
| MCP | `--self-test` + 工具调用契约测试 | 是 |
| 端到端 | 真机：取密钥→解密→查看→MCP | 手动 |

## 2. 各期验收标准（Definition of Done）

### P1 · 核心闭环
- [ ] `platform_mac.discover()` 在本机正确发现 WeChat.app、3 个账号、三库齐全状态。
- [ ] 路径归一化对 5 种输入（账号目录/db_storage/message 子目录/多账号根/单账号上级）行为正确。
- [ ] `decryptor.calibrate()` 确定本机微信 4.1 的 `CipherParams` 并回填 `docs/dev/04`、`docs/dev/11`。
- [ ] `verify_key` 对正确密钥返回真、错误密钥返回假。
- [ ] `decrypt_db` 解出的 `message_0/contact/session` 明文库：`PRAGMA integrity_check=ok`，可列表，核心表行数 > 0。
- [ ] `verify.verify_decrypted` 报告三库表/行数/大小。
- [ ] CLI（`scripts/cli.py decrypt/verify`）可跑通。
- [ ] 全程日志不含原始密钥。

### P2 · 查看器
- [ ] GUI 启动、控制台/查看器切换正常。
- [ ] 会话列表加载、过滤、统计正确。
- [ ] 选中会话加载文本消息，分页（向更早）正常，不卡主线程。
- [ ] 群消息发送者正确剥离显示。
- [ ] 会话内搜索命中。
- [ ] QML 无 undefined 绑定告警（契约 09 全覆盖）。

### P3 · 媒体
- [ ] 图片 `.dat` 解码并预览（至少旧格式魔数反推可用）。
- [ ] 语音 SILK→WAV 可播放。
- [ ] 视频有 MP4 可打开、仅缩略图正确降级。
- [ ] 文件附件定位/提示正确。
- [ ] 头像显示，缺失占位。
- [ ] 媒体全部按需加载，缓存写工具目录，原始只读。

### P4 · 完善交付
- [ ] SIP 检测正确；关 SIP 后 lldb 一键取密钥成功并存 Keychain。
- [ ] Keychain 存/取/删/多账号正确，UI 只显指纹。
- [ ] MCP HTTP/stdio 自测通过；核心工具（status/list/read/search/verify/decrypt）契约正确。
- [ ] `.app` 在另一台干净 Mac（未装 Python）能启动、能引导 Full Disk Access、能手动导入密钥并查看。

## 3. 关键测试用例

- **解密正确性**：对比解密库某表行数与微信自带 `message_fts` 或已知条数量级一致。
- **密钥/账号错配**：用账号 A 的 key 解账号 B 的库 → `KeyMismatchError`。
- **只读保护**：解密/查看/媒体全流程后，`xwechat_files` 下文件 mtime/大小不变（断言原始目录未被写）。
- **脱敏**：grep 日志/MCP 输出无 64 hex 长密钥串。
- **大库性能**：最大会话首屏加载 < 2s（后台线程），滚动不卡。

## 4. 测试数据与隐私

- 使用开发者本人账号真实库，测试断言只针对**结构与数量**，不硬编码聊天内容。
- fixtures 若需入库，先脱敏（截取 schema + 少量合成行），不提交真实聊天数据。
- CI 不接触真实微信数据；真实数据测试仅在本机手动执行。

## 5. 回归

- 每期完成跑全量 `pytest` + offscreen 冒烟 + `py_compile`。
- 微信版本升级后重点回归：`decryptor.calibrate`、`keycapture` 定位、schema 变化。

## 6. 逐模块 pytest 用例骨架（精细化）

目录 `tests/`。约定：不硬编码聊天内容；涉及真实库的用例用 `@pytest.mark.integration` 标记，CI 默认跳过（`-m "not integration"`）。

### 6.1 conftest.py（公共夹具）
```python
import pytest, sqlite3
from pathlib import Path

@pytest.fixture
def tmp_account(tmp_path: Path) -> Path:
    """构造假的 xwechat_files/wxid_x/db_storage 结构。"""
    acc = tmp_path/"xwechat_files"/"wxid_test_0001"
    db = acc/"db_storage"
    for rel in ("message/message_0.db","contact/contact.db","session/session.db"):
        p = db/rel; p.parent.mkdir(parents=True, exist_ok=True); p.write_bytes(b"\x00"*4096)
    return acc

@pytest.fixture
def real_account() -> Path:
    """本机真实账号（集成用）。缺失则跳过。"""
    p = Path.home()/("Library/Containers/com.tencent.xinWeChat/Data/Documents/"
                     "xwechat_files/wxid_xxxxxxxx")
    if not p.exists(): pytest.skip("无真实账号")
    return p

@pytest.fixture
def known_key() -> bytes:
    """本机主密钥，来自环境变量（不入库）。缺失则跳过集成解密测试。"""
    import os
    hexk = os.environ.get("WX_TEST_KEY")
    if not hexk: pytest.skip("未提供 WX_TEST_KEY")
    return bytes.fromhex(hexk)
```

### 6.2 test_platform_mac.py
```python
from wechat_decryptor_mac import platform_mac as pm

def test_enumerate_and_core_dbs(tmp_account):
    root = tmp_account.parent
    accs = pm.enumerate_accounts(root)
    assert len(accs) == 1 and accs[0].has_core_dbs and accs[0].missing == ()

def test_normalize_db_storage_level(tmp_account):
    a = pm.normalize_to_account(tmp_account/"db_storage")
    assert a and a.account_dir == tmp_account

def test_normalize_message_subdir(tmp_account):
    a = pm.normalize_to_account(tmp_account/"db_storage"/"message")
    assert a and a.account_dir == tmp_account

def test_normalize_multi_account_returns_list(tmp_path):
    # 造两个账号 → 返回 list，不自动选
    ...

@pytest.mark.integration
def test_discover_real(real_account):
    env = pm.discover()
    assert env.app_version and env.arch in ("arm64","x86_64")
    assert any(a.has_core_dbs for a in env.accounts)
```

### 6.3 test_decryptor.py
```python
from wechat_decryptor_mac import decryptor as dc

def test_derive_keys_stable():
    k = dc.derive_keys(b"\x11"*32, b"\x22"*16, dc.CipherParams())
    assert len(k[0]) == 32 and len(k[1]) == 32

@pytest.mark.integration
def test_verify_key_true_false(real_account, known_key):
    session = real_account/"db_storage/session/session.db"
    params = dc.calibrate(session, known_key); assert params
    assert dc.verify_key(session, known_key, params)
    assert not dc.verify_key(session, b"\x00"*32, params)

@pytest.mark.integration
def test_decrypt_integrity(tmp_path, real_account, known_key):
    session = real_account/"db_storage/session/session.db"
    params = dc.calibrate(session, known_key)
    out = tmp_path/"session_decrypted.db"
    dc.decrypt_db(session, out, known_key, params)
    import sqlite3
    con = sqlite3.connect(f"file:{out}?mode=ro", uri=True)
    assert con.execute("PRAGMA integrity_check").fetchone()[0] == "ok"
    assert con.execute("SELECT count(*) FROM sqlite_master").fetchone()[0] > 0
```

### 6.4 test_verify.py
```python
from wechat_decryptor_mac import verify
def test_missing_file(tmp_path):
    rep = verify.verify_decrypted(tmp_path, names=("session_decrypted.db",))
    assert not rep.all_ok and rep.dbs[0].error == "文件不存在"

def test_plain_sqlite_ok(tmp_path):
    import sqlite3
    db = tmp_path/"session_decrypted.db"
    con = sqlite3.connect(db); con.execute("CREATE TABLE t(x)"); con.execute("INSERT INTO t VALUES(1)")
    con.commit(); con.close()
    rep = verify.verify_decrypted(tmp_path, names=("session_decrypted.db",))
    assert rep.all_ok and rep.dbs[0].row_counts["t"] == 1
```

### 6.5 test_keyring.py（内存后端）
```python
from wechat_decryptor_mac.keyring_store import KeyringStore, MemoryBackend
def test_save_load_delete():
    s = KeyringStore(MemoryBackend())
    rec = s.save_key("wxid_a", b"\x33"*32, "manual")
    assert s.load_key("wxid_a").key == b"\x33"*32
    assert "wxid_a" in s.list_stored()
    s.delete_key("wxid_a"); assert s.load_key("wxid_a") is None
    assert len(rec.fingerprint) == 8
```

### 6.6 test_keycapture.py（可 CI 的纯逻辑）
```python
from wechat_decryptor_mac.keycapture import sip, lldb_capture as lc
def test_sip_parse(monkeypatch):
    monkeypatch.setattr(sip, "_raw", lambda: "System Integrity Protection status: enabled.")
    assert sip.sip_status() is sip.SipStatus.ENABLED

def test_entropy_filter():
    assert not lc._entropy_ok(b"\x00"*32)               # 全零
    assert not lc._entropy_ok(b"A"*32)                   # 低 distinct/ASCII
    assert lc._entropy_ok(bytes(range(32)))              # 高熵

def test_scan_core_hits(tmp_path, monkeypatch):
    key = bytes(range(100,132))
    core = tmp_path/"c.bin"; core.write_bytes(b"\x00"*1000 + key + b"\x00"*1000)
    monkeypatch.setattr(lc, "verify_key_bytes", lambda db,c,p: c == key)
    assert lc.scan_core(core, tmp_path/"s.db", object()) == key
```

### 6.7 test_parsers.py（data_adapter 纯逻辑）
```python
from wechat_decryptor_mac.data_adapter import parsers as ps
def test_classify_text(): assert ps.classify(1,0)[0] == "text"
def test_classify_image(): assert ps.classify(3,0)[0] == "image"
def test_split_group_sender():
    wxid, body = ps.split_group_sender("wxid_abc:\n你好", is_sender=0)
    assert wxid == "wxid_abc" and body == "你好"
def test_split_private_no_prefix():
    wxid, body = ps.split_group_sender("你好", is_sender=0)
    assert wxid == "" and body == "你好"
```

### 6.8 test_data_adapter.py（集成，真实解密库）
```python
@pytest.mark.integration
def test_conversations_and_messages(decrypted_dir):   # decrypted_dir 由解密测试产出
    from wechat_decryptor_mac.data_adapter.repository import Repository
    repo = Repository(decrypted_dir)
    convs = repo.list_conversations(limit=10)
    assert isinstance(convs, list)
    if convs:
        msgs = repo.read_messages(convs[0].username, limit=20)
        assert all(m.kind for m in msgs)              # 只断言结构，不断言内容
```

### 6.9 test_mcp.py（契约）
```python
from wechat_decryptor_mac.mcp.server import McpServer, RpcRequest
def test_initialize_and_tools_list(fake_workspace):
    srv = McpServer(fake_workspace, repo_factory=fake_repo, decrypt_fn=None)
    r = srv.handle(RpcRequest(1,"initialize",{}))
    assert r["result"]["serverInfo"]["name"] == "wechat-local"
    r = srv.handle(RpcRequest(2,"tools/list",{}))
    names = {t["name"] for t in r["result"]["tools"]}
    assert {"wechat_status","wechat_read_messages","wechat_decrypt"} <= names

def test_unknown_method():
    ...  # 返回 -32601
```

### 6.10 test_media.py（纯逻辑）
```python
from wechat_decryptor_mac.media import image_dat as im
def test_guess_xor_jpg():
    plain = b"\xFF\xD8\xFF" + b"..."
    k = 0x5A; enc = bytes(b ^ k for b in plain)
    assert im.guess_xor(enc)[0] == k
```

### 6.11 冒烟（GUI）
```bash
QT_QPA_PLATFORM=offscreen python ui_qt/full_smoke_test.py --demo
# 断言：QML 加载成功、无 "is not defined" / "Unable to assign" 告警
```

## 7. 安全/只读回归（专项，来自 21）
```python
@pytest.mark.integration
def test_original_readonly(real_account, known_key, tmp_path):
    import os
    before = {p: p.stat().st_mtime_ns for p in (real_account/"db_storage").rglob("*.db")}
    # …执行 decrypt + verify + 若干 media resolve（输出到 tmp_path）…
    after = {p: p.stat().st_mtime_ns for p in (real_account/"db_storage").rglob("*.db")}
    assert before == after                     # 原始库未被写

def test_no_key_in_logs(caplog, known_key):
    # 执行解密后，日志中不得出现 64 hex 密钥串
    import re
    assert not re.search(r"[0-9a-fA-F]{64}", caplog.text)
```

## 8. 运行命令
```bash
pytest -m "not integration"                 # CI：纯逻辑
WX_TEST_KEY=<hex> pytest -m integration      # 本机：真实数据
ruff check . && mypy wechat_decryptor_mac
QT_QPA_PLATFORM=offscreen python ui_qt/full_smoke_test.py --demo
```
