# 12 · 构建、打包与分发

## 1. 开发环境

```bash
xcode-select --install                 # lldb（取密钥用）
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements-app.txt -r requirements-dev.txt
python ui_qt/full_main.py               # 源码模式运行
```

Python：3.11+（开发机 3.13 可用）。

## 2. py2app 打包

`setup.py`：

```python
from setuptools import setup

APP = ["ui_qt/full_main.py"]
DATA_FILES = [("qml", ["ui_qt/qml"]), ("resources", ["resources"])]
OPTIONS = {
    "argv_emulation": False,
    "iconfile": "resources/icon.icns",
    "includes": ["wechat_decryptor_mac", "ui_qt"],
    "packages": ["PySide6", "Crypto", "PIL", "pysilk", "zstandard"],
    "plist": {
        "CFBundleName": "微信记录工作台",
        "CFBundleIdentifier": "com.example.wechatdecryptor",
        "CFBundleShortVersionString": "0.1.0",
        "NSHighResolutionCapable": True,
        "LSMinimumSystemVersion": "12.0",
    },
}
setup(app=APP, data_files=DATA_FILES,
      options={"py2app": OPTIONS}, setup_requires=["py2app"])
```

```bash
python setup.py py2app          # 产出 dist/微信记录工作台.app
python setup.py py2app -A       # alias 模式（开发快速联调，不自包含）
```

打包注意：
- **PySide6 + QML 资源**：确保 `ui_qt/qml/` 打进包，QML 引擎按包内相对路径加载（用 `Path(__file__).parent/"qml"`）。
- **pycryptodome**（包名 `Crypto`）、**pysilk**、**zstandard** 显式列入 `packages`，避免 py2app 漏收。
- **只打包程序与资源**，绝不打包任何聊天数据/密钥/账号路径。

## 3. 权限与 entitlements

`.app` 需要访问 `~/Library/Containers/com.tencent.xinWeChat/`，受 macOS TCC 限制：

- 首次访问会被系统拦截 → 需用户在 **系统设置 → 隐私与安全性 → 完全磁盘访问权限（Full Disk Access）** 中勾选本 `.app`。UI 需检测 `PermissionError` 并引导（见 `docs/dev/03` §6）。
- 若做取密钥的 lldb 子进程，本身不进 `.app` 沙盒（lldb 需系统权限 + SIP 关闭）。
- 打包用非沙盒（不启用 App Sandbox），否则无法访问微信容器与调试。

## 4. 签名与公证（分发关键，影响小白体验）

三档策略（评审决定投入哪档）：

| 档 | 做法 | 小白体验 |
|---|---|---|
| 0 · 不签名 | ad-hoc | 首次需"右键→打开"绕过 Gatekeeper；可能提示"已损坏"需 `xattr -dr com.apple.quarantine` |
| 1 · Developer ID 签名 | `codesign` Developer ID | 仍可能需公证，Gatekeeper 提示减少 |
| 2 · 签名 + 公证 | `codesign` + `notarytool` + `stapler` | 双击即开，体验最佳（需 Apple 开发者账号 $99/年） |

未公证时的用户说明（写入 README）：
```bash
# 若提示"已损坏，无法打开"
xattr -dr com.apple.quarantine "/Applications/微信记录工作台.app"
```

## 5. 交付物

- `dist/微信记录工作台.app`
- `README.md`（安装、Full Disk Access、首次取密钥向导、免责声明）
- `requirements-app.txt`（源码运行者用）
- 可选 `.dmg`（`hdiutil create` 或 `create-dmg`）

## 6. 版本与构建信息

生成构建清单 `build_manifest.json`：

```json
{
  "product": "微信记录工作台 (macOS)",
  "package_version": "0.1.0",
  "build_time": "<ISO8601>",
  "target_os": "macOS 12+",
  "arch": "arm64",
  "main_app": "微信记录工作台.app",
  "wechat_target": "4.0/4.1 (xwechat)",
  "data_policy": "只打包程序与资源；不含任何聊天数据、密钥、账号路径。"
}
```

## 7. 构建前检查（发布纪律）

```bash
python -m pytest tests/
python -m py_compile $(git ls-files '*.py')   # 或遍历源码
ruff check . && mypy wechat_decryptor_mac
QT_QPA_PLATFORM=offscreen python ui_qt/full_smoke_test.py --demo
```

## 8. py2app 真实坑点手册（精细化）

py2app 打 PySide6 应用是已知的高频踩坑区，以下按"现象 → 原因 → 处理"整理，务必逐条落实。

### 8.1 PySide6 体积与插件裁剪
- **现象**：`.app` 动辄 800MB+，或运行时报 `This application failed to start because no Qt platform plugin could be initialized`（找不到 `libqcocoa.dylib`）。
- **原因**：py2app 默认整包收 PySide6，但 Qt 的 **plugins**（`platforms/`、`imageformats/`、`iconengines/`）与 QML 运行时不一定被正确放到 `Contents/Resources` / rpath 可见处。
- **处理**：
  - 平台插件必须在：`Contents/Resources/lib/python3.x/PySide6/Qt/plugins/platforms/libqcocoa.dylib`。构建后**验证**：
    ```bash
    find dist/*.app -name libqcocoa.dylib
    ```
  - 用 `QT_DEBUG_PLUGINS=1 ./dist/*.app/Contents/MacOS/*` 首启诊断插件加载。
  - 裁剪：不需要的模块从收集里排除，显著减体积：
    ```python
    OPTIONS["excludes"] = [
        "PySide6.QtWebEngineCore", "PySide6.QtWebEngineWidgets",
        "PySide6.Qt3D", "PySide6.QtCharts", "PySide6.QtDataVisualization",
        "PySide6.QtMultimedia",  # 若不用系统播放器再排除
        "tkinter", "PyQt5", "PyQt6",
    ]
    ```

### 8.2 QML 运行时收集（最容易漏）
- **现象**：源码模式正常，打包后白屏或 `module "QtQuick" is not installed`。
- **原因**：QML 的 `.qml`、`qmldir`、以及 **QtQuick/QtQuick.Controls 的原生 qml 插件目录**（`PySide6/Qt/qml/...`）未被打进包。py2app 只收 Python import，收不到 QML import 图。
- **处理**：
  - 我方 `.qml` 通过 `DATA_FILES`/`resource` 显式打入，代码用 `Path(__file__).parent/"qml"` 定位（不要用绝对开发路径）。
  - 强制携带 Qt 的 qml 目录，构建后验证存在：
    ```bash
    ls dist/*.app/Contents/Resources/lib/python3.*/PySide6/Qt/qml/QtQuick
    ls dist/*.app/Contents/Resources/lib/python3.*/PySide6/Qt/qml/QtQuick/Controls
    ```
    若缺失，手动在 `setup.py` 里把 `PySide6/Qt/qml` 作为数据树复制，或在启动时 `QQmlApplicationEngine.addImportPath()` 指向包内 qml 目录。
  - 首启设置环境自检：
    ```python
    import os
    os.environ.setdefault("QT_QUICK_CONTROLS_STYLE", "Basic")  # 避免缺 Style 插件
    ```

### 8.3 rpath 与动态库定位
- **现象**：`ImportError: dlopen(... .so, ...): Library not loaded: @rpath/QtCore` 或 `image not found`。
- **原因**：py2app 重定位后 `.so`/`.dylib` 的 `@rpath` 未指向 `.app` 内实际 Qt 库位置。
- **处理**：
  - 构建后核查依赖解析：
    ```bash
    otool -L dist/*.app/Contents/Resources/lib/python3.*/PySide6/QtCore.abi3.so
    ```
    确认 `@rpath/QtCore.framework/...` 能在包内命中。
  - 若签名后失效，注意**签名要在所有库就位、rpath 修完之后**再做（见 §4）。
  - pycryptodome 的 C 扩展（`Crypto/Cipher/_raw_aes.abi3.so` 等）也要在 `otool -L` 下自包含，无对外部 Homebrew 库的依赖。

### 8.4 隐藏依赖（hidden imports）
- **现象**：运行时 `ModuleNotFoundError`，但源码模式好好的。
- **原因**：动态导入（如 `Crypto` 子模块、`zstandard` 的后端、`pysilk`）py2app 静态分析收不全。
- **处理**：`includes` 里补齐：
  ```python
  OPTIONS["includes"] += [
      "Crypto.Cipher._raw_aes", "Crypto.Cipher._raw_cbc",
      "Crypto.Protocol.KDF", "Crypto.Hash.SHA512",
      "zstandard.backend_c", "_cffi_backend",
  ]
  ```
  构建后用真实流程（解密一次 + 播一段 SILK）验证，而不是只看是否启动。

### 8.5 架构（arm64 / universal2）
- **现象**：在 Intel Mac 上 `mach-o, but wrong architecture` 或反之。
- **原因**：wheel 装的是单架构（多数开发机 arm64），`.app` 也就只支持 arm64。
- **处理**：
  - 明确交付 **arm64 单架构**（Apple Silicon）。`build_manifest.json` 的 `arch` 如实写 `arm64`。
  - 如需 universal2，需所有原生 wheel（PySide6/pycryptodome/pysilk）都提供 universal2 或分别装后 `lipo` 合并——成本高，默认不做，写清"仅支持 Apple Silicon（M 系列）"。

### 8.6 与 SIP/lldb 取密钥的关系
- 取密钥的 `lldb` 是**外部系统进程**，不打进 `.app`；打包只需保证代码能 `subprocess` 调用系统 `lldb`（用户装了 CLT）。
- `.app` 自身**不要**启用 App Sandbox、不要 Hardened Runtime 的严格限制（否则连不了 FDA / 调不动子进程）。若要公证，Hardened Runtime 为必需，此时给 `.app` 授 entitlements：
  ```xml
  <!-- entitlements.plist -->
  <key>com.apple.security.cs.allow-jit</key><true/>
  <key>com.apple.security.cs.disable-library-validation</key><true/>
  ```
  `disable-library-validation` 是 PySide6/pycryptodome 这类第三方原生库在 Hardened Runtime 下能加载的关键。

### 8.7 签名/公证顺序（避免"改一处全废"）
```bash
# 1) 先修好包内所有库与 rpath，再签（从内到外，深度签名）
codesign --deep --force --options runtime \
  --entitlements entitlements.plist \
  --sign "Developer ID Application: NAME (TEAMID)" \
  "dist/微信记录工作台.app"

# 2) 公证
ditto -c -k --keepParent "dist/微信记录工作台.app" app.zip
xcrun notarytool submit app.zip --keychain-profile "AC_PROFILE" --wait

# 3) 装订票据
xcrun stapler staple "dist/微信记录工作台.app"

# 4) 验证
codesign --verify --deep --strict --verbose=2 "dist/微信记录工作台.app"
spctl -a -vvv -t install "dist/微信记录工作台.app"
```
> 任何对 `.app` 内容的改动（哪怕替换一个资源文件）都会使签名失效，必须**重新从 §7 检查 → 签名 → 公证**走一遍。

### 8.8 一键构建脚本（建议 `scripts/build_app.sh`）
```bash
#!/usr/bin/env bash
set -euo pipefail
rm -rf build dist
python -m pytest -m "not integration"
QT_QPA_PLATFORM=offscreen python ui_qt/full_smoke_test.py --demo
python setup.py py2app
# 构建后自检（缺一即失败）
APP=dist/*.app
find $APP -name libqcocoa.dylib | grep . || { echo "缺 cocoa 插件"; exit 1; }
ls $APP/Contents/Resources/lib/python3.*/PySide6/Qt/qml/QtQuick >/dev/null || { echo "缺 QtQuick qml"; exit 1; }
echo "构建自检通过"
```

### 8.9 分发前冒烟（在另一台干净 Mac）
- 目标机**不装 Python、不装 Homebrew**。
- 双击（或 `xattr -dr com.apple.quarantine`）→ 启动无缺库报错。
- 引导 Full Disk Access → 能发现账号。
- 手动导入密钥 → 能解密并查看（对应 13 §2 P4 验收最后一条）。
