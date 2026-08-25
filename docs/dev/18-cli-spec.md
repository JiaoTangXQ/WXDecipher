# 18 · 命令行（CLI）规格

`scripts/cli.py` 是 P1 的主要交付与验收载体，也是无 GUI 场景的运维入口。所有子命令只读原始库、只写工具数据目录。

## 1. 总览

```bash
python scripts/cli.py <command> [options]
```

| 子命令 | 用途 | 期 |
|---|---|---|
| `detect` | 发现微信/账号/环境 | P1 |
| `decrypt` | 解密指定账号的库 | P1 |
| `verify` | 校验已解密结果 | P1 |
| `calibrate` | 用已知密钥校准 SQLCipher 参数 | P1 |
| `dump-schema` | 导出解密库 schema（回填 11） | P1 |
| `capture-key` | 关 SIP 后抓取密钥（独立调试） | P4 |
| `mcp` | 启动 MCP（等价 scripts/mcp_server.py） | P4 |

全局选项：`--workspace <dir>`（工具数据目录，默认 `~/Library/Application Support/WeChatDecryptor`）、`--verbose`、`--json`（机器可读输出）。

## 2. `detect`

```bash
python scripts/cli.py detect [--json]
```
输出：微信路径与版本、数据根、账号列表（wxid + 三库齐全）、SIP 状态、架构。
退出码：0 成功；2 未发现微信/账号。

## 3. `calibrate`

```bash
python scripts/cli.py calibrate --db <session.db 路径> --key <hex|keychain:wxid>
```
遍历候选参数集，输出命中的 `CipherParams`（page_size/kdf/hmac/reserve）。
用于 P1 首次确定本机微信 4.1 参数，结果回填 `docs/dev/04`、`docs/dev/11`。
退出码：0 命中；3 无候选命中（疑似不支持版本）。

## 4. `decrypt`

```bash
python scripts/cli.py decrypt \
  --account <账号目录|wxid> \
  --key <hex | keychain[:wxid] | manual> \
  [--dbs message_0,contact,session] \
  [--out <输出目录>] \
  [--params <calibrate 结果或 auto>]
```
- `--key`：
  - `keychain[:wxid]`：从 Keychain 读（P4 起）。
  - 64 hex：直接用（先校验）。
  - `manual`：交互式提示输入（隐藏回显）。
- `--dbs`：默认三核心库；可指定子集或 `all`。
- 行为：校验密钥 → 逐库分页解密 → 写 `<out>/<name>_decrypted.db` → 自动 `verify`。
- 退出码：0 成功；4 密钥不匹配；5 解密 IO 错误。
- 日志：密钥仅指纹。

示例（P1 开发验收）：
```bash
python scripts/cli.py decrypt \
  --account wxid_xxxxxxxx \
  --key <你抓到的 64 hex> \
  --dbs message_0,contact,session
python scripts/cli.py verify --dir "$HOME/Library/Application Support/WeChatDecryptor/decrypted"
```

## 5. `verify`

```bash
python scripts/cli.py verify --dir <解密输出目录> [--json]
```
对每个 `*_decrypted.db`：`PRAGMA integrity_check`、表清单、行数、文件大小 → `VerifyReport`。
退出码：0 全通过；6 有库损坏/缺失。

## 6. `dump-schema`

```bash
python scripts/cli.py dump-schema --dir <解密输出目录> [--out schema.md]
```
对每个明文库导出：表清单、每表 `PRAGMA table_info`、行数。产出用于回填 `docs/dev/11`。

## 7. `capture-key`（P4，需先关 SIP）

```bash
sudo python scripts/cli.py capture-key --pid <WeChat pid> --db <session.db> [--strategy scan|breakpoint]
```
关 SIP 环境下用 lldb 抓密钥，校验通过后可 `--save-keychain wxid` 存入 Keychain。
退出码：0 成功；7 SIP 开启；8 抓取失败。

## 8. `mcp`

```bash
python scripts/cli.py mcp --mode http|stdio [--workspace <dir>] [--self-test]
```
等价 `scripts/mcp_server.py`；`--self-test` 见 `docs/dev/10` §7。

## 9. 通用约定

- `--json` 输出结构化结果，供脚本/测试消费。
- 所有子命令失败时 `stderr` 输出 `docs/dev/16` 的 `code + user_message`。
- 不接受任何会写原始微信目录的参数组合。
