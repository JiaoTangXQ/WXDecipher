# 07 · data_adapter（只读数据适配层）· 精细化实现

把解密后的明文 SQLite 转成界面/MCP 用的领域对象。**只读**打开，每线程独立连接。模块目录：`wechat_decryptor_mac/data_adapter/`。

> 微信 4.1 真实表结构在 P1 执行 `docs/dev/19` 后回填 `docs/dev/11`。本册给出**接口、SQL 模板骨架、解析算法**；标 `【schema】` 处以真实字段替换。

---

## 1. 连接管理

```python
def open_ro(db: Path) -> sqlite3.Connection:
    con = sqlite3.connect(f"file:{db}?mode=ro&immutable=1", uri=True,
                          check_same_thread=True)   # 每线程独立，不共享
    con.row_factory = sqlite3.Row
    return con
```
- `immutable=1`：允许并发只读、跳过锁。
- 连接**不跨线程**；后台线程各自 `open_ro`。
- `Repository` 持有各库连接的 thread-local 工厂。

## 2. 领域对象（全字段，对应 MessageBubble.qml）

```python
@dataclass
class ConversationRow:
    username: str; display_name: str; initials: str
    summary: str; last_time: str; epoch: int
    unread_count: int; avatar_url: str = ""

@dataclass
class ContactRow:
    username: str; remark: str; nick_name: str; alias: str
    avatar_url: str = ""; is_group: bool = False

@dataclass
class MessageRow:
    local_id: int; table: str
    sender_name: str; sender_wxid: str; is_self: bool
    time: str; epoch: int
    kind: str; type_label: str
    content: str = ""; duration: str = ""
    # 媒体（P3 由 media 填充）
    media_available: bool = False; media_renderable: bool = False
    media_playable: bool = False
    media_url: str = ""; media_path: str = ""
    media_thumbnail_url: str = ""; media_thumbnail_path: str = ""
    media_reason: str = ""; media_attachment: bool = False
    # 结构化卡片
    structured_title: str = ""; structured_description: str = ""; structured_url: str = ""
    # 媒体解析用的原始线索（不进 UI）
    _img_md5: str = ""; _video_name: str = ""; _file_name: str = ""
    _emoji_cdn_url: str = ""; _raw_xml: str = ""

@dataclass
class Stats:
    conversations: int; message_tables: int; total_messages: int
```

## 3. Repository

```python
class Repository:
    def __init__(self, decrypted_dir: Path, account_dir: Path | None = None):
        self.dir = decrypted_dir; self.account_dir = account_dir
        self._contacts = ContactRepo(decrypted_dir / "contact_decrypted.db")
        self._sessions = SessionRepo(decrypted_dir / "session_decrypted.db", self._contacts)
        self._messages = MessageRepo(decrypted_dir, self._contacts)

    def list_conversations(self, query="", limit=100, offset=0) -> list[ConversationRow]:
        return self._sessions.list(query, limit, offset)
    def stats(self) -> Stats: ...
    def list_contacts(self, query="", limit=100, offset=0, include_avatar=False) -> list[ContactRow]:
        return self._contacts.list(query, limit, offset, include_avatar)
    def group_members(self, group, query="", limit=500, offset=0) -> list[ContactRow]:
        return self._contacts.group_members(group, query, limit, offset)
    def read_messages(self, conversation, limit=50, before_epoch=None, query="") -> list[MessageRow]:
        return self._messages.read(conversation, limit, before_epoch, query)
    def read_message(self, local_id, table) -> MessageRow | None: ...
    def search_messages(self, query, limit=50, cursor=None) -> "SearchPage":
        ...   # 委托 mcp.indexer（见 10）
```

## 4. SessionRepo（session.db）

```python
# 【schema】表/字段以 P1 回填为准；以下为占位 SQL 模板
SQL_LIST_SESSIONS = """
SELECT username, summary, last_timestamp AS epoch, unread_count
FROM   SessionTable
WHERE  (:q = '' OR username LIKE :like OR summary LIKE :like)
ORDER BY last_timestamp DESC
LIMIT :limit OFFSET :offset
"""
class SessionRepo:
    def list(self, query, limit, offset) -> list[ConversationRow]:
        like = f"%{query}%"
        rows = self.con.execute(SQL_LIST_SESSIONS,
                 {"q": query, "like": like, "limit": limit, "offset": offset})
        out = []
        for r in rows:
            name = self.contacts.display_name(r["username"])   # 备注>昵称>username
            out.append(ConversationRow(
                username=r["username"], display_name=name,
                initials=_initials(name), summary=_clean(r["summary"]),
                epoch=r["epoch"], last_time=_fmt_time(r["epoch"]),
                unread_count=r["unread_count"] or 0,
                avatar_url=self.contacts.avatar_url(r["username"]),  # P3
            ))
        return out
```
- 显示名：join contact；`display_name = remark or nick_name or username`。
- `initials`：显示名首个可见字（中文取首字，英文取首字母大写）。
- `summary` 清洗：剥离控制字符、截断。

## 5. ContactRepo（contact.db）

```python
SQL_CONTACT = """
SELECT username, remark, nick_name, alias
FROM   Contact
WHERE  (:q='' OR remark LIKE :like OR nick_name LIKE :like OR username LIKE :like)
ORDER BY remark, nick_name LIMIT :limit OFFSET :offset
"""      # 【schema】

class ContactRepo:
    def __init__(self, db: Path):
        self.con = open_ro(db); self._name_cache: dict[str,str] = {}
    def display_name(self, username: str) -> str:
        if username in self._name_cache: return self._name_cache[username]
        r = self.con.execute("SELECT remark,nick_name FROM Contact WHERE username=?",
                             (username,)).fetchone()
        name = (r and (r["remark"] or r["nick_name"])) or username
        self._name_cache[username] = name; return name
    def group_members(self, group, query, limit, offset) -> list[ContactRow]:
        room = self._resolve_room_id(group)   # 群 wxid / 群名 / chat_room.id
        # 【schema】join 群成员表 + Contact
        ...
```
- LRU 缓存显示名，避免消息渲染时逐条查库。
- `group` 解析：接受群 `@chatroom`、群显示名、`chat_room.id`。

## 6. MessageRepo（message_*.db）

### 6.1 分表/分库定位
```python
# 微信 4.0 消息按会话哈希分表或分库【P1 实测确认：Msg_<md5> 分表 or 单表+talker列】
def table_for(username: str) -> str:
    return "Msg_" + hashlib.md5(username.encode()).hexdigest()
```
- 若为分库（message_0..N），需在多库中定位含该表的库；`MessageRepo` 维护 `表名→库` 映射（启动时扫描各库 `sqlite_master`）。

### 6.2 读取分页
```python
SQL_READ = """
SELECT local_id, sender, is_sender, create_time, msg_type, sub_type,
       content, compress_content, extra
FROM   "{table}"
WHERE  (:before IS NULL OR create_time < :before)
  AND  (:q='' OR content LIKE :like)
ORDER BY create_time DESC
LIMIT :limit
"""     # 【schema】列名占位
class MessageRepo:
    def read(self, conversation, limit, before_epoch, query) -> list[MessageRow]:
        table = self._locate(conversation)
        con = self._con_for(table)
        rows = con.execute(SQL_READ.format(table=table),
                 {"before": before_epoch, "q": query,
                  "like": f"%{query}%", "limit": limit}).fetchall()
        return [self._to_row(r, table) for r in reversed(rows)]  # 反转成时间正序
```
- 首屏 `ORDER BY create_time DESC LIMIT n` 取最近，反转为正序展示。
- `loadOlderMessages`：以当前最早 `epoch` 作 `before_epoch` 续查。

### 6.3 行 → MessageRow（parsers.py）
```python
def _to_row(self, r, table) -> MessageRow:
    kind, label = classify(r["msg_type"], r["sub_type"])
    sender_wxid, body = split_group_sender(r["content"], r["is_sender"])
    body = decompress_if_needed(body, r["compress_content"])
    row = MessageRow(local_id=r["local_id"], table=table,
        sender_wxid=sender_wxid, is_self=bool(r["is_sender"]),
        sender_name=self.contacts.display_name(sender_wxid) if sender_wxid else ("我" if r["is_sender"] else ""),
        epoch=r["create_time"], time=_fmt_time(r["create_time"]),
        kind=kind, type_label=label)
    if kind == "text":
        row.content = body
    elif kind in ("link","file"):
        _fill_appmsg(row, body)          # 解析 XML 卡片
    elif kind == "image":
        row._img_md5 = _extract_img_md5(body, r["extra"])
    elif kind == "voice":
        row.duration = _voice_duration(r["extra"])
    elif kind == "video":
        row._video_name = _extract_video_name(body, r["extra"])
    elif kind == "emoji":
        row._emoji_cdn_url = _extract_emoji_url(body)
    return row
```

### 6.4 类型判定（classify）
```python
# 【P1 实测回填 type 数值】
TYPE_MAP = {
    1:  ("text","文本"), 3:("image","图片"), 34:("voice","语音"),
    43: ("video","视频"), 47:("emoji","表情"),
    49: None,                              # appmsg，按 sub_type 细分
    10000:("system","系统"), 10002:("system","系统"),
}
APPMSG_SUB = {5:("link","链接"), 6:("file","文件"), 19:("link","合并转发"),
              33:("link","小程序"), 57:("link","引用")}   # 【实测回填】
def classify(msg_type, sub_type):
    m = TYPE_MAP.get(msg_type)
    if m: return m
    if msg_type == 49: return APPMSG_SUB.get(sub_type, ("link","卡片"))
    return ("system", f"类型{msg_type}")
```

### 6.5 群消息发送者剥离
```python
def split_group_sender(content: str, is_sender: int) -> tuple[str, str]:
    # 群消息正文常见前缀 "wxid_xxx:\n<正文>"；私聊无前缀
    if not is_sender and ":\n" in content[:64]:
        head, _, rest = content.partition(":\n")
        if head.startswith("wxid_") or head.endswith("@chatroom") or _looks_like_id(head):
            return head, rest
    return "", content
```
（是否有该前缀在 P1 抽样确认，回填。）

### 6.6 正文解压
```python
import zstandard
def decompress_if_needed(body, compress_blob) -> str:
    if compress_blob:                     # 【实测确认哪些字段用 zstd】
        try: return zstandard.ZstdDecompressor().decompress(compress_blob).decode("utf-8","replace")
        except zstandard.ZstdError: pass
    return body if isinstance(body, str) else body.decode("utf-8","replace")
```

### 6.7 appmsg 卡片解析
```python
import xml.etree.ElementTree as ET
def _fill_appmsg(row, xml_text):
    try: root = ET.fromstring(xml_text)
    except ET.ParseError: row.content = xml_text; return
    app = root.find(".//appmsg")
    if app is None: row.content = xml_text; return
    row.structured_title = (app.findtext("title") or "").strip()
    row.structured_description = (app.findtext("des") or "").strip()
    row.structured_url = (app.findtext("url") or "").strip()
    att = app.find("appattach")
    if att is not None:
        row._file_name = (att.findtext("fileext") and app.findtext("title")) or ""
```

## 7. 时间与格式
```python
def _fmt_time(epoch: int) -> str:
    # 【P1 确认单位：秒 or 毫秒】；本模板按秒
    return datetime.fromtimestamp(epoch).strftime("%Y-%m-%d %H:%M")
```

## 8. 搜索
- 会话内：`read_messages(conversation, query=...)`，SQL `content LIKE`。
- 全局：`search_messages` 委托 `mcp.indexer` 的 FTS5 索引（见 10 §5），返回 `SearchPage{rows, next_cursor}`，带硬上限。

## 9. 性能
- 首屏 `limit` 条 + 媒体按需（可见气泡触发 `resolveMedia`）。
- 显示名/头像 LRU 缓存。
- `immutable=1` 允许并发只读；后台线程查询不阻塞 UI。
- 分库表名→库映射启动时构建一次。

## 10. 单测
- 用 P1 解出的真实库（脱敏小样本）验证：会话列表排序/过滤/统计、消息分页与向更早、`classify` 各类型、群发送者剥离、zstd 解压、appmsg 解析。
- 断言不硬编码任何聊天正文，只断言结构/数量/字段存在。
