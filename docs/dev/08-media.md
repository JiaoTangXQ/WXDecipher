# 08 · media（媒体解析：图片/表情/语音/视频/文件）· 精细化实现

按可见消息**按需**解析，不首屏全量。原始文件只读，解码副本写 `.media_cache/`。模块目录：`wechat_decryptor_mac/media/`。

本机实测布局：
```text
<account>/msg/attach/<md5>/YYYY-MM/Img/<hash>_t.dat   # 图片，_t=缩略图，V2 加密
<account>/msg/file/...                                 # 文件
<account>/msg/video/...                                # 视频 + *_thumb.jpg
<account>/business/emoticon/...                        # 表情缓存
db_storage/hardlink/hardlink.db                        # 媒体路径映射
db_storage/head_image/head_image.db                    # 头像
db_storage/emoticon/emoticon.db                        # 表情元数据
```
实测 `.dat` 头 16 字节：`0708 5632 0807 0004 0000 6015 0000 0141 …` → 魔数 `07 08 'V' '2'`（V2 格式）。

---

## 1. 统一结果与入口

```python
@dataclass
class MediaResult:
    available: bool = False
    renderable: bool = False     # 有可直接显示的图/缩略图
    playable: bool = False       # 有可播放/打开的本地文件
    url: str = ""                # file:// 主资源
    path: str = ""               # 本地绝对路径
    thumbnail_url: str = ""
    thumbnail_path: str = ""
    reason: str = ""             # 不可用原因（人话）
    attachment: bool = False

class MediaResolver:
    def __init__(self, account_dir: Path, cache_dir: Path,
                 image_key: ImageKey | None, hardlink: "HardlinkIndex"): ...
    def resolve(self, msg: MessageRow, mode: str = "thumbnail") -> MediaResult:
        return {
            "image": self._image, "emoji": self._emoji, "voice": self._voice,
            "video": self._video, "file": self._file,
        }.get(msg.kind, lambda m, md: MediaResult(reason="不支持的媒体"))(msg, mode)
```
`mode` ∈ `metadata|thumbnail|playable|export`，对应 MCP `wechat_media_resolve`。

## 2. hardlink 映射（media/hardlink.py）

微信 4.0 用 hardlink 去重：消息里存媒体 md5/文件名，实际文件在 `msg/attach|video|file` 下，映射记录在解密后的 `hardlink.db`。

```python
class HardlinkIndex:
    def __init__(self, hardlink_db_decrypted: Path, account_dir: Path):
        self._con = sqlite3.connect(f"file:{hardlink_db_decrypted}?mode=ro&immutable=1", uri=True)
        self._cache: dict[str, Path | None] = {}
    def resolve(self, md5_or_name: str) -> Path | None:
        # 查映射表得相对目录/文件名 → 拼绝对路径
        # 【P3 实测回填表名与字段】示例：
        #   SELECT dir, name FROM file_hardlink WHERE md5=?
        ...
```
- 表结构 P3 回填 `docs/dev/11`。
- 进程内 LRU 缓存，避免逐条查库。
- 找不到映射时回退到按 md5 目录约定拼路径（`msg/attach/<md5>/YYYY-MM/Img/...`）。

## 3. 图片 `.dat` 解码（media/image_dat.py）

图片密钥 ≠ 数据库主密钥。两类格式：

### 3.1 旧格式（无魔数，整文件单字节 XOR）
用已知图片魔数反推 XOR key（无需外部密钥）：
```python
MAGICS = {
    0xFF: ("jpg", b"\xFF\xD8\xFF"),
    0x89: ("png", b"\x89PNG\r\n\x1a\n"),
    0x47: ("gif", b"GIF8"),
    0x42: ("bmp", b"BM"),
}
def guess_xor(dat_head: bytes) -> tuple[int, str] | None:
    b0 = dat_head[0]
    for first, (ext, magic) in MAGICS.items():
        k = b0 ^ first                       # 首字节异或已知魔数首字节
        if bytes(x ^ k for x in dat_head[:len(magic)]) == magic:
            return k, ext
    return None
def decode_xor(data: bytes, k: int) -> bytes:
    return bytes(b ^ k for b in data)
```

### 3.2 V2 格式（本机实测，魔数 `07 08 'V' '2'`）
头部结构（部分字段 P3 实测确认）：
```text
偏移0: 07 08 56 32           magic "\x07\x08V2"
偏移4: .. .. .. ..           版本/类型字段            【待实测回填】
偏移8: 长度字段(小端)         加密段长度 / 原始长度      【待实测回填】
...    AES 加密段 + 可能的 XOR/明文尾部                 【待实测回填】
```
解码框架（scheme 可插拔，参数来自 `ImageKey`）：
```python
def decode_v2(data: bytes, img: ImageKey) -> tuple[bytes, str]:
    assert data[:4] == b"\x07\x08V2"
    # scheme=="aes": 用 img.aes_key 对加密段 AES 解密（模式/分段长 P3 实测）
    # scheme=="aes+xor": 解密段 + 尾部 XOR
    # 解出后用 pillow 识别真实格式
    ...
```
> V2 的确切 AES 模式（ECB/CBC）、加密段长度、是否混合 XOR，需在 P3 用真实图片密钥 + 已知明文样本校准（做法同 `decryptor.calibrate`：拿一张能对上魔数的样本反推）。校准前先支持旧格式与缩略图显示。

### 3.3 解码流程
```python
def _image(self, msg, mode) -> MediaResult:
    dat = self.hardlink.resolve(msg.img_md5) or self._guess_path(msg)
    if not dat or not dat.exists():
        return MediaResult(reason="图片文件未找到")
    head = dat.read_bytes()[:64]
    if head[:4] == b"\x07\x08V2":
        if not self.image_key:
            return MediaResult(renderable=False, reason="需先获取图片密钥")
        raw, ext = decode_v2(dat.read_bytes(), self.image_key)
    else:
        g = guess_xor(head)
        if not g: return MediaResult(reason="未知图片格式")
        k, ext = g; raw = decode_xor(dat.read_bytes(), k)
    out = self.cache_dir / "images" / f"{dat.stem}.{ext}"
    out.parent.mkdir(parents=True, exist_ok=True)
    out.write_bytes(raw)                     # 仅写缓存目录
    return MediaResult(available=True, renderable=True,
                       url=out.as_uri(), path=str(out))
```
用 pillow 兜底校验解出的确是有效图片（`Image.open(io.BytesIO(raw)).verify()`），失败则 `reason="解码结果非有效图片"`。

### 3.4 图片密钥获取（extractImageKey）
- 优先"魔数反推"覆盖旧格式，免密钥。
- V2 的 AES key：从内存扫描（同 keycapture 思路，用"能对上魔数的样本图"作校验判据），或已知参数。
- 结果存 Keychain（`imgkey:<wxid>`），显示指纹 + XOR/scheme（`app.imageKeyXor`）。

## 4. 语音（media/voice_silk.py）

微信语音是 SILK（`#!SILK_V3` 头）。
```python
import pysilk, wave
def silk_to_wav(silk_bytes: bytes, out: Path, rate=24000) -> Path:
    pcm = pysilk.decode(silk_bytes, sample_rate=rate)   # 返回 PCM s16le
    with wave.open(str(out), "wb") as w:
        w.setnchannels(1); w.setsampwidth(2); w.setframerate(rate)
        w.writeframes(pcm)
    return out
```
- 语音数据存位置 P3 实测确认（`message_resource.db` 或 `msg/` 下），可能带自定义头需剥离到 `#!SILK_V3`。
- 采样率常见 24000；若解码异常尝试 16000。
- 输出 `.media_cache/voice/<id>.wav`；时长取消息扩展字段填 `duration`。

## 5. 视频（media/video.py）
```python
def _video(self, msg, mode) -> MediaResult:
    mp4 = self.hardlink.resolve(msg.video_name) or self._video_path(msg)
    thumb = self._thumb_path(msg)            # *_thumb.jpg
    if mp4 and mp4.exists():
        return MediaResult(available=True, playable=True, renderable=bool(thumb),
                           path=str(mp4), url=mp4.as_uri(),
                           thumbnail_url=thumb.as_uri() if thumb else "")
    if thumb and thumb.exists():
        return MediaResult(available=True, renderable=True, playable=False,
                           thumbnail_url=thumb.as_uri(), reason="仅缩略图，未找到视频")
    return MediaResult(reason="视频与缩略图均未找到")
```
`openMedia` → `subprocess.run(["open", path])` 交系统播放器。

## 6. 表情（media/emoticon.py）
```python
def _emoji(self, msg, mode) -> MediaResult:
    local = self._emoji_local(msg)           # business/emoticon 或 emoticon.db 记录
    if local and local.exists():
        return MediaResult(available=True, renderable=True, url=local.as_uri(), path=str(local))
    if msg.emoji_cdn_url and net_enabled():   # 仅按消息自带地址、且联网开关开
        p = self._download(msg.emoji_cdn_url, self.cache_dir/"emoji")
        return MediaResult(available=True, renderable=True, url=p.as_uri(), path=str(p))
    return MediaResult(renderable=False, reason="缓存不可预览")
```
下载仅 GIF/WebP、写受限缓存；`net_enabled()` 受全局联网开关控制（默认可关，见 21）。

## 7. 文件（file）
```python
def _file(self, msg, mode) -> MediaResult:
    f = self.hardlink.resolve(msg.file_name) or self._file_path(msg)
    if f and f.exists() and f.suffix != ".dat":
        return MediaResult(available=True, playable=True, attachment=True,
                           path=str(f), url=f.as_uri())
    if f and f.suffix == ".dat":
        return MediaResult(available=True, attachment=True, playable=False,
                           reason="附件为加密缓存，需在微信重新导出")
    return MediaResult(attachment=True, reason="附件未找到")
```
不把加密 `.dat` 交系统 shell。

## 8. 头像（P3）
- `head_image.db` 存头像 blob/URL；解析当前账号头像写 `.media_cache/avatars/<wxid>.png`，缺失显示占位。
- 填 `ConversationRow.avatar_url`、`MessageRow.sender_avatar_url`。表结构 P3 回填 11。

## 9. 只读与缓存边界
- 解码产物全部写 `~/Library/Application Support/WeChatDecryptor/.media_cache/`，可安全删除。
- 原始 `.dat/.db/.mp4` 只读；绝不写回 `xwechat_files`（`assert_within_workspace` 守卫）。
- 大文件不内联进 MCP，只给本地路径/Resource（见 10）。

## 10. UI 契约映射（→ 09）
`MediaResult` 字段一一映射到 `MessageRow` 媒体字段 → `MessageBubble.qml` property。`app.resolveMedia(index)` 后台解析→模型 `dataChanged` 原地刷新该行；`app.openMedia(index)` → `open` 命令。

## 11. 单测
- `guess_xor`/`decode_xor`：构造 XOR 后的假 jpg/png，验证反推正确。
- `decode_v2`：P3 校准后用真实样本回归。
- `silk_to_wav`：小 SILK 样本 → 可读 WAV（帧数/采样率正确）。
- `HardlinkIndex.resolve`：对解密 hardlink.db 样本命中/未命中。
- 只读回归：解析后原始媒体文件未变。
