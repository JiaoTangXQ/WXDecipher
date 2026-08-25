# WXDecipher · Local WeChat Chat History Decryptor & Viewer (macOS)

**English** | [中文](README.zh-CN.md)

> A local **macOS** tool that helps a user read and view the local chat-history data of **their own WeChat account**, on **their own device**, for personal backup, search and migration. All processing happens locally; no chat data is ever uploaded or transmitted over the network.

---

## ⚠️ Legal Notice & Disclaimer (Read Before Use)

**This project is provided for technical research and lawful personal use only. By downloading or using it, you acknowledge that you have read and agreed to all of the terms below.**

1. **Your own data only.** This tool may only be used to access the local data of a WeChat account that **you own and that is normally logged in on your own machine**. Using it to access, crack, or steal another person's chat history, or any data you are not authorized to access, is **strictly prohibited**.
2. **Local processing, no data leaves your device.** All decryption and viewing happen on the user's local device. This project **contains no server** and does not collect, upload, or share any of your data, keys, or personal information.
3. **Lawful and compliant use.** You must comply with the laws and regulations of your jurisdiction, including but not limited to applicable cybersecurity, personal-information-protection, data-security, and privacy laws. Using this tool for **any unlawful purpose** — including unauthorized data access, invasion of privacy, corporate espionage, or evidence forgery — is strictly prohibited. You bear sole legal responsibility for any such use.
4. **Respect others' privacy.** Chat histories typically contain the personal information of other parties and group members. Even for your own records, obtain the consent of the relevant parties before exporting or sharing content that involves them, to avoid infringing third-party privacy.
5. **No warranty, no liability.** This project is provided "AS IS", without any express or implied warranty as to availability, accuracy, or data safety. The author is not liable for any data loss, corruption, legal dispute, or other damages arising from use of this tool.
6. **Unofficial, no affiliation.** This is an **independent, third-party** open-source technical project with **no affiliation, endorsement, or partnership** with Tencent or with "WeChat". "WeChat" and related names are trademarks of their respective owners and are used here only to objectively describe the compatibility target.
7. **No purpose of circumventing copyright protection.** This project exists to help users access **their own data** and is not intended to circumvent or defeat any technical protection measure for the purpose of infringement.
8. **Clean up promptly.** Decrypted plaintext data is highly sensitive. Store it securely, delete it promptly after use, and never share or upload it publicly.

**If you do not agree with any of the above, stop using this project immediately and delete it along with any of its outputs.**

---

## Overview

On macOS, WeChat 4.x stores chat history in a **SQLCipher-encrypted** local database. With authorization on the user's own machine, this project provides a **local, read-only** decryption and viewing solution:

- **Discover** — locate the local WeChat app, its data directory, and logged-in accounts.
- **Key capture** — with the user's explicit authorization and understanding of the risks, obtain the account's database key from local process memory.
- **Decrypt** — produce a plaintext copy of the database offline using the standard SQLCipher paged algorithm (**the original WeChat directory is opened read-only and never written back**).
- **View** — browse conversations, messages, and contacts read-only, with local restoration of media such as text, images, voice, and video.
- **MCP service** — optionally expose a local [Model Context Protocol](https://modelcontextprotocol.io/) service, bound to the loopback address with random authentication, so a local AI client can search local data under authorization.

## Background

WeChat on macOS provides no official way to export or back up your chat history. Your messages, images, voice notes, and files live only inside an encrypted local database — you cannot export them as a whole, cannot search across your full history, and have no programmatic access to your own data.

This causes real problems: switching or resetting your Mac can wipe out years of conversations with no official way to recover them; you cannot archive or full-text search your own records; and your data is effectively out of your own hands. WXDecipher exists to solve exactly that — letting you unlock and read **your own** local WeChat data on **your own** Mac, so you can back it up, search it, and keep it under your control, entirely offline.

## Status

🚧 **Design-specification stage** — the repository currently contains **complete development-level design documents**; business code has not yet been written. The documentation is detailed enough to implement from directly.

- Design & development docs: see [`docs/dev/`](docs/dev/), starting from [`docs/dev/README.md`](docs/dev/README.md).
- Master design: [`docs/superpowers/specs/2026-08-24-wechat-decryptor-mac-design.md`](docs/superpowers/specs/2026-08-24-wechat-decryptor-mac-design.md)

## Planned Features

| Module | Description |
|---|---|
| Path discovery | Automatically locate `WeChat.app`, the data root, and accounts (`xwechat_files/wxid_*`) |
| SQLCipher decryption | Pure-Python paged decryption (PBKDF2 + AES + per-page HMAC verification), no native dependency |
| Key storage | Securely store keys in the macOS **Keychain** |
| Viewer | PySide6 + Qt Quick/QML UI; read-only browsing of conversations/messages/contacts |
| Media restoration | `.dat` image decoding, SILK voice → WAV, video/file resolution |
| Local MCP | Loopback-bound JSON-RPC service with a random token, callable by AI clients |
| CLI | Command-line entry points for decryption, verification, and parameter calibration |

## Tech Stack

- **Language:** Python 3.11+
- **UI:** PySide6 (Qt Quick / QML)
- **Crypto:** pycryptodome (SQLCipher-compatible implementation)
- **Compression/media:** zstandard, pysilk
- **Packaging:** py2app (self-contained `.app`)
- **Platform:** macOS 12+ (Apple Silicon)

## Requirements & Known Limitations

- The target WeChat account must be **logged in** on the local machine.
- Obtaining the database key may require temporarily adjusting macOS **System Integrity Protection (SIP)** and using `lldb` — a consequence of macOS's security restrictions on process memory; see the docs for the trade-offs and procedure.
- Accessing the WeChat data directory requires granting the app **Full Disk Access**.
- Apple Silicon (M-series) only.

## Privacy & Security Design

- The original WeChat data is accessed **read-only**; all writes occur only within the tool's own data directory.
- Keys are stored only in the Keychain; logs emit only a key fingerprint (`sha256` prefix) and never the raw key or chat content.
- The MCP service binds only to `127.0.0.1` with a random authentication token generated on each start.
- See [`docs/dev/21-security-privacy-threat-model.md`](docs/dev/21-security-privacy-threat-model.md).

## License

Released under the **MIT License**, provided that **your use also complies with the "Legal Notice & Disclaimer" above and with applicable local laws**. The license grants rights to use the code only and does not constitute authorization or endorsement of any unlawful use.
