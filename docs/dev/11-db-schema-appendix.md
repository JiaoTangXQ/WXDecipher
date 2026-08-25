# 11 · 数据库结构附录

> **本册在 P1 解密成功后回填真实 schema。** 当前记录：已核实的加密库清单 + 期望结构（供开发前参考），真实字段以 `PRAGMA table_info` dump 为准。

## 1. db_storage 加密库清单（本机实测）

| 路径 | 用途 | 优先级 |
|---|---|---|
| `message/message_0.db` | 私聊/群聊消息（可能多分库 message_N） | P1 |
| `message/biz_message_0.db` | 公众号/服务号消息 | 后续 |
| `message/message_fts.db` | 微信自带全文索引 | 参考 |
| `message/message_resource.db` | 消息资源（媒体元数据） | P3 |
| `contact/contact.db` | 联系人、群、群成员 | P1 |
| `contact/contact_fts.db` | 联系人全文索引 | 参考 |
| `session/session.db` | 会话列表 | P1 |
| `head_image/head_image.db` | 头像 | P3 |
| `hardlink/hardlink.db` | 媒体文件去重映射 | P3 |
| `emoticon/emoticon.db` | 表情元数据 | P3 |
| `favorite/favorite.db` `favorite_fts.db` | 收藏 | 可选 |
| `sns/sns.db` | 朋友圈 | 可选 |
| `general/general.db` | 通用配置 | 可选 |
| `bizchat/bizchat.db` | 企业/服务会话 | 可选 |
| `solitaire/solitaire.db` | 接龙 | 可选 |

全部为 SQLCipher 加密（文件头随机，非 `SQLite format 3`）。

## 2. 回填模板（P1 执行）

解密后对每个库运行并粘贴结果：

```sql
-- 表清单
SELECT name FROM sqlite_master WHERE type='table' ORDER BY name;
-- 每表结构
PRAGMA table_info('<table>');
-- 行数
SELECT count(*) FROM '<table>';
```

回填区（P1 后补全）：

### session.db 【待实测回填】
- 表：
- 会话字段（对端 username / summary / timestamp / unread）：

### contact.db 【待实测回填】
- 联系人表字段（username / remark / nick_name / alias / 头像）：
- 群/群成员表：

### message_0.db 【待实测回填】
- 分表还是单表？表名规则（`Msg_<md5>` 或统一表 + talker 列）：
- 消息字段（local_id / talker / is_sender / create_time / type / sub_type / content / compress_content / 扩展）：
- 消息 type 数值 → kind 映射实测值：
- 正文压缩方式（zstd？哪些字段）：

### hardlink.db 【待实测回填】
- md5/文件名 → 相对路径映射表结构：

## 3. SQLCipher 参数（P1 校准后回填）

见 `docs/dev/04` §2；校准确定后把最终 `CipherParams` 记录于此：
- page_size：
- kdf_algo / kdf_iter：
- hmac_algo / reserve：

## 4. 已知与 Windows 版差异

- 库数量更多（Windows 文档仅提 3 个核心库，Mac 4.1 实测 16 个）。
- 媒体走 `hardlink.db` 映射（需专门解析）。
- 表结构预计与 Windows 4.0 高度一致（同一跨平台架构），但需实测确认字段名/类型细节。
