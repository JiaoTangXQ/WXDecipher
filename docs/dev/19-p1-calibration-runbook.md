# 19 · P1 校准操作手册（回填真实参数与 schema）

本册是 P1 阶段在**开发机（本机，微信 4.1.12）**上一步步执行的操作手册，目的是把 `docs/dev/04`（SQLCipher 参数）与 `docs/dev/11`（数据库 schema）中的 `待实测回填` 项落实为真实值。

> 本手册描述"要做的操作步骤"，属于文档。真正执行需先具备密钥，涉及一次性关 SIP。执行时机由用户确认。

## 0. 前置

- 已登录微信、有真实数据（本机满足）。
- 账号目录（本机示例）：
  ```text
  ~/Library/Containers/com.tencent.xinWeChat/Data/Documents/xwechat_files/wxid_xxxxxxxx
  ```
- 需要该账号的 32 字节主密钥（下一步获取）。

## 1. 获取主密钥（一次性）

三选一：

- **手动导入**：若已从别处获得 64 位十六进制密钥，跳到第 2 步。
- **一次性关 SIP + lldb**（见 `docs/dev/06`）：
  1. 进恢复模式执行 `csrutil disable`，重启。
  2. 确认微信运行并登录，记下主进程 pid：
     ```bash
     pgrep -f '/Applications/WeChat.app/Contents/MacOS/WeChat$'
     ```
  3. 用 P4 的 `capture-key` 或调试脚本抓取；抓取期间在微信里打开一个聊天以触发数据库访问。
  4. 抓到后校验通过，记录指纹（不记明文）。
  5. 抓完 `csrutil enable` 开回 SIP。
- **重签名**（高级备选，`docs/dev/06` §6）。

## 2. 校准 SQLCipher 参数

对已知密钥，遍历候选参数集，找到令首页 HMAC 通过的一组：

```bash
python scripts/cli.py calibrate \
  --db "<账号目录>/db_storage/session/session.db" \
  --key <64hex>
```

期望输出（示例，实际以运行为准）：
```text
命中参数：page_size=4096 kdf=sha512 kdf_iter=256000 hmac=sha512 reserve=80
```

**回填**：把命中结果写入
- `docs/dev/04` §2 `CipherParams` 默认值；
- `docs/dev/11` §3 参数记录。

若无候选命中：记录微信版本，扩充 `CANDIDATE_PARAMS`（尝试其它 kdf_iter：如 4000/64000/256000；hmac：sha1/sha256/sha512；reserve 对齐/不对齐组合），再校准。

## 3. 解密三核心库

```bash
python scripts/cli.py decrypt \
  --account wxid_xxxxxxxx \
  --key <64hex> \
  --dbs message_0,contact,session \
  --params auto
```

产出：`~/Library/Application Support/WeChatDecryptor/decrypted/{message_0,contact,session}_decrypted.db`

## 4. 校验解密结果

```bash
python scripts/cli.py verify --dir "<解密输出目录>"
```

期望：三库 `integrity_check=ok`，能列表，核心表行数 > 0。这是 **P1 验收标准**。

## 5. 导出 schema 回填 11

```bash
python scripts/cli.py dump-schema --dir "<解密输出目录>" --out /tmp/wechat_schema.md
```

或手动用标准 sqlite3：
```bash
sqlite3 "<解密输出目录>/session_decrypted.db" ".tables"
sqlite3 "<解密输出目录>/session_decrypted.db" ".schema"
```

**回填 `docs/dev/11`**：
- `session.db`：会话表名与字段（对端 username / summary / timestamp / unread）。
- `contact.db`：联系人字段（username/remark/nick_name/alias/头像）、群与群成员表。
- `message_0.db`：确认分表还是单表；表名规则；消息字段（local_id/talker/is_sender/create_time/type/sub_type/content/compress_content）；type 数值→kind 映射；正文压缩方式。
- `hardlink.db`：媒体 md5/文件名→相对路径映射表。

## 6. 确认正文解析细节（回填 07）

抽样若干消息行，确认：
- 群消息正文是否有 `wxid:\n` 发送者前缀。
- 哪些字段是 zstd 压缩（`zstandard` 能否解开）。
- `type=49` appmsg XML 的结构（title/des/url/子类型）。

**回填 `docs/dev/07`** §4.1/§4.2 的映射表与解析规则。

## 7. 只读保护验证

执行完以上操作后，验证原始微信目录未被写入：
```bash
# 记录操作前后 db_storage 的 mtime/大小，应一致
find "<账号目录>/db_storage" -name '*.db' -newer /tmp/marker
```
应为空（无文件被修改）。

## 8. 完成标志

- `docs/dev/04` §2、`docs/dev/11` §3/§2、`docs/dev/07` §4 的 `待实测回填` 全部替换为真实值。
- CLI 全链路（detect→calibrate→decrypt→verify→dump-schema）跑通。
- 原始目录只读性验证通过。
