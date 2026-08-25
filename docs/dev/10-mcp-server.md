# 10 · mcp_server（本地 MCP 服务）· 精细化实现

让支持 MCP 的 AI 客户端读取已解密数据或显式触发解密。纯本机、只读优先。移植自 Windows 版，语义等价替换（Hook→取密钥）。模块目录：`wechat_decryptor_mac/mcp/`。

---

## 1. 传输模式

- **HTTP（默认）**：`127.0.0.1` + 系统自动分配端口；每次启动随机 Bearer Token。地址 `http://127.0.0.1:<port>/mcp`，健康检查 `/health`。
- **stdio**：AI 客户端拉起 `python scripts/mcp_server.py --mode stdio --workspace <dir>`，不监听 TCP。

配置文件：`~/Library/Application Support/WeChatDecryptor/mcp_servers.generated.json`
```json
{ "mcpServers": { "wechat-local": {
  "url": "http://127.0.0.1:<port>/mcp",
  "headers": { "Authorization": "Bearer <本次token>" } } } }
```

## 2. JSON-RPC 2.0 报文

实现方法：`initialize`、`tools/list`、`tools/call`、`resources/list`、`resources/templates/list`、`resources/read`。

请求/响应封装：
```python
@dataclass
class RpcRequest:  id: object; method: str; params: dict
@dataclass
class RpcError:    code: int; message: str; data: dict | None = None

def ok(id, result): return {"jsonrpc":"2.0","id":id,"result":result}
def err(id, code, message, data=None):
    return {"jsonrpc":"2.0","id":id,"error":{"code":code,"message":message,"data":data or {}}}
```
错误码：JSON-RPC 标准（-32600 非法请求、-32601 方法不存在、-32602 参数错、-32603 内部错）+ 领域错误经 `data.detail`（脱敏，映射 `docs/dev/16`）。

`initialize` 返回：
```json
{"protocolVersion":"2024-11-05",
 "capabilities":{"tools":{},"resources":{"subscribe":false}},
 "serverInfo":{"name":"wechat-local","version":"0.1.0"}}
```

## 3. 派发核心（mcp/server.py）

```python
class McpServer:
    def __init__(self, workspace: Path, repo_factory, decrypt_fn):
        self.tools = build_tools(workspace, repo_factory, decrypt_fn)  # name->callable
        self.resources = build_resources(workspace)

    def handle(self, req: RpcRequest):
        if req.method == "initialize":      return ok(req.id, INIT_RESULT)
        if req.method == "tools/list":      return ok(req.id, {"tools": TOOL_SPECS})
        if req.method == "tools/call":
            name = req.params["name"]; args = req.params.get("arguments", {})
            tool = self.tools.get(name)
            if not tool: return err(req.id, -32601, f"未知工具 {name}")
            try:
                content = tool(**args)       # 返回 MCP content 列表
                return ok(req.id, {"content": content})
            except WeChatDecryptorError as e:
                return err(req.id, -32000, e.user_message, {"detail": e.detail, "code": e.code})
        if req.method.startswith("resources/"):  return self._resources(req)
        return err(req.id, -32601, "方法不存在")
```
工具返回 MCP content：`[{"type":"text","text":...}]` 或 `[{"type":"resource","resource":{...}}]`。

## 4. HTTP 传输（mcp/server.py）

```python
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
import secrets, json

def serve_http(server: McpServer, host="127.0.0.1", port=0):
    token = secrets.token_urlsafe(32)
    class H(BaseHTTPRequestHandler):
        def _auth_ok(self):
            return self.headers.get("Authorization") == f"Bearer {token}"
        def do_GET(self):
            if self.path == "/health":
                return self._json(200, {"status":"ok"})
            return self._json(404, {"error":"not found"})
        def do_POST(self):
            if self.path != "/mcp":  return self._json(404, {"error":"not found"})
            if not self._auth_ok():  return self._json(401, {"error":"unauthorized"})
            body = json.loads(self.rfile.read(int(self.headers["Content-Length"])))
            req = RpcRequest(body.get("id"), body["method"], body.get("params", {}))
            self._json(200, server.handle(req))
        def _json(self, code, obj):
            data = json.dumps(obj).encode(); self.send_response(code)
            self.send_header("Content-Type","application/json")
            self.send_header("Content-Length",str(len(data))); self.end_headers()
            self.wfile.write(data)
        def log_message(self, *a): pass      # 静默，另走 mcp.log（token 脱敏）
    httpd = ThreadingHTTPServer((host, port), H)
    actual_port = httpd.server_address[1]
    return httpd, actual_port, token         # 写入 generated.json；UI 显示端口
```
- 端口 0 → 系统分配；`actual_port` 回填 UI 与配置。
- 仅绑 `127.0.0.1`；每启动新 token；关服务 token 失效。
- 日志记 token 指纹（`sha256(token)[:8]`），不记明文。

## 5. stdio 传输
```python
def serve_stdio(server: McpServer):
    for line in sys.stdin:                    # 每行一个 JSON-RPC
        if not line.strip(): continue
        body = json.loads(line)
        req = RpcRequest(body.get("id"), body["method"], body.get("params",{}))
        sys.stdout.write(json.dumps(server.handle(req)) + "\n"); sys.stdout.flush()
```

## 6. 工具清单与 JSON Schema（契约）

> 返回均脱敏；密钥只给指纹。返回受 `max_bytes`/`max_body_length` 硬上限约束。

| 工具 | 关键参数 | 说明 |
|---|---|---|
| `wechat_status` | — | 三库齐全/大小/会话数/消息数/Keychain 可读/Key 指纹 |
| `wechat_list_key_stores` | — | 列 Keychain 本工具项（wxid+指纹，不给密钥） |
| `wechat_list_conversations` | query,limit,offset | 会话列表 |
| `wechat_list_contacts` | query,limit,offset,include_avatar | 联系人 |
| `wechat_group_members` | group,query,limit,offset | 群成员 |
| `wechat_read_messages` | conversation,limit,query,account_dir,before_epoch | 会话消息 |
| `wechat_search_messages` | query,limit,include_media,cursor,max_bytes,max_body_length | 全局搜索（FTS） |
| `wechat_index_build` | force,async | 建/刷新索引 |
| `wechat_read_message` | message_id | 单条 |
| `wechat_media_resolve` | message_id,mode(metadata/thumbnail/playable/export) | 媒体 |
| `wechat_export_start` | fmt(jsonl/csv/md/html/pdf),conversation,query,max_rows,destination,async | 流式导出 |
| `wechat_verify_decrypted` | — | 只读校验三库 |
| `wechat_decrypt` | account_dir,output_dir,wxid,timeout,dry_run,confirm_capture | 显式解密 |
| `wechat_start_decrypt`/`wechat_task_status`/`wechat_task_logs`/`wechat_task_cancel` | task_id | 异步任务 |

各工具完整 JSON Schema（`tools/list` 的 `inputSchema`）示例：
```json
// wechat_read_messages
{"type":"object","properties":{
  "conversation":{"type":"string"},"limit":{"type":"integer","default":100},
  "query":{"type":"string"},"account_dir":{"type":"string"},
  "before_epoch":{"type":"integer"}},"required":["conversation"]}
// wechat_media_resolve
{"type":"object","properties":{
  "message_id":{"type":"string"},
  "mode":{"enum":["metadata","thumbnail","playable","export"],"default":"thumbnail"}},
  "required":["message_id"]}
// wechat_decrypt（Mac 语义）
{"type":"object","properties":{
  "account_dir":{"type":"string"},"output_dir":{"type":"string"},
  "wxid":{"type":"string"},"timeout":{"type":"integer","default":240},
  "dry_run":{"type":"boolean","default":false},
  "confirm_capture":{"type":"boolean","default":false}}}
```
其余工具 schema 同 `docs/dev/10` 原表（本册保留 Windows 版全集，仅 `wechat_decrypt` 改 Mac 语义）。

### 6.1 wechat_decrypt Mac 语义
```text
Keychain 有 key 且过三库校验 → 直接复用解密（无特权）
key 缺失/不符 → 返回 requires_capture=true
   需用户确认后再传 confirm_capture:true 才触发 keycapture 引导
dry_run:true → 只查路径/密钥状态，不解密/不取密钥/不改文件
```
决不因 AI 单方请求静默取密钥或关 SIP。

## 7. 工具实现示例（mcp/tools.py）

```python
def tool_read_messages(repo_factory, **kw):
    repo = repo_factory(kw.get("account_dir"))
    rows = repo.read_messages(kw["conversation"], kw.get("limit",100),
                              kw.get("before_epoch"), kw.get("query",""))
    text = render_rows_capped(rows, max_bytes=kw.get("max_bytes"))  # 超限截断+提示
    return [{"type":"text","text":text}]

def tool_status(repo_factory, store, **kw):
    repo = repo_factory(None); s = repo.stats()
    return [{"type":"text","text": json.dumps({
        "conversations": s.conversations, "messages": s.total_messages,
        "key": {"stored": bool(store.list_stored()),
                "fingerprint": store.load_key(kw.get("wxid",""))
                               and store.load_key(kw["wxid"]).fingerprint},
    }, ensure_ascii=False)}]
```
所有返回经 `cap(content, max_bytes)`：超限则截断并附"结果过大，请分页/导出"提示（`McpLimitError` 或 content 内说明）。

## 8. Resources（mcp/resources.py）

- `wechat://index/status`：索引状态
- `wechat://message/{message_id}`：单条消息
- `wechat://media/{message_id}`：媒体元数据
- `wechat://file/{encoded_abs_path}`：**白名单内**缩略图/语音/导出文件

```python
def read_resource(uri: str, workspace: Path):
    scheme, _, rest = uri.partition("://")
    if rest.startswith("file/"):
        path = Path(unquote(rest[5:]))
        assert_within(path, allow=[workspace/"decrypted", workspace/".media_cache",
                                   workspace/"exports", ACCOUNT_DIR])  # 白名单守卫
        if path.stat().st_size <= INLINE_LIMIT:
            return {"blob": base64.b64encode(path.read_bytes()).decode(),
                    "mimeType": guess_mime(path)}
        return {"uri": uri, "size": path.stat().st_size, "path": str(path)}  # 大文件只给元数据
    ...
```
路径越界（不在白名单）直接拒绝，防目录穿越。

## 9. 索引（mcp/indexer.py）

```python
def build_index(decrypted_dir: Path, index_db: Path, force=False, progress=None):
    con = sqlite3.connect(index_db)
    con.execute("""CREATE VIRTUAL TABLE IF NOT EXISTS msg_fts USING fts5(
        message_id UNINDEXED, conversation UNINDEXED, sender, epoch UNINDEXED,
        body, tokenize='unicode61')""")
    con.execute("""CREATE TABLE IF NOT EXISTS index_state(
        table_name TEXT PRIMARY KEY, last_epoch INTEGER)""")
    # 增量：对每个消息表，读 last_epoch 之后的新行插入 FTS
    for table in _message_tables(decrypted_dir):
        last = 0 if force else _last_epoch(con, table)
        for row in _iter_new(decrypted_dir, table, last):
            con.execute("INSERT INTO msg_fts VALUES(?,?,?,?,?)",
                        (row.id, row.conversation, row.sender, row.epoch, row.body))
        _set_last_epoch(con, table, _max_epoch(...))
    con.commit()
```
- 索引文件 `.index/wechat_index.sqlite`；源库只读。
- 中文用 `unicode61`（或后续接 jieba 外部分词，视召回需要）。
- `search_messages` 走 `SELECT ... FROM msg_fts WHERE body MATCH ? LIMIT ? `，游标分页，硬上限。
- 大库 `async=true`：`wechat_start_decrypt` 式任务化，轮询 `wechat_task_status`。

## 10. 任务系统（异步大操作）
```python
@dataclass
class Task: id: str; phase: str; done: bool; exit_code: int|None; tail: list[str]
TASKS: dict[str, Task] = {}
def start_task(fn) -> str:
    tid = secrets.token_hex(8); TASKS[tid] = Task(tid,"启动",False,None,[])
    threading.Thread(target=_run_task, args=(tid, fn), daemon=True).start()
    return tid
```
`wechat_task_logs` 只返回脱敏阶段/退出码/尾部（不含密钥/正文全文）。

## 11. 安全边界
- HTTP 仅 `127.0.0.1` + 随机 token；不监听局域网；不改系统代理。
- 只读源库；不上传远程；仅显式 `wechat_decrypt`/`start_decrypt` 触发解密；仅 `confirm_capture:true`+用户确认触发取密钥。
- 密钥/ token 只出指纹；返回硬上限；Resource 路径白名单守卫。
- 关主程序/删数据目录即清理。

## 12. 自测
```bash
python scripts/mcp_server.py --mode stdio --workspace <dir> --self-test
```
校验 `initialize` 握手、`tools/list`、解密库状态、Keychain 可读；不取密钥、不启动微信。契约测试：对每个工具跑最小参数，断言返回结构与上限生效。
