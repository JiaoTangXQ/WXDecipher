# 09 · QML ↔ Python 接口契约 · 精细化实现

QML 复用现有 `FullMain.qml` / `Main.qml` / `components/MessageBubble.qml`。本册给出**完整契约**（属性/槽/信号/模型角色）+ **PySide6 实现骨架**（真实签名）。契约不可漏项，否则 QML 绑定 undefined。文件：`ui_qt/app.py`、`ui_qt/models.py`、`ui_qt/full_main.py`、`ui_qt/strings.py`。

---

## 1. context 注入（full_main.py）

```python
import sys
from pathlib import Path
from PySide6.QtGui import QGuiApplication
from PySide6.QtQml import QQmlApplicationEngine
from ui_qt.app import App
from ui_qt.models import ConversationModel, MessageModel

def main() -> int:
    qapp = QGuiApplication(sys.argv)
    qapp.setApplicationName("微信记录工作台")
    engine = QQmlApplicationEngine()

    conv_model = ConversationModel()
    msg_model = MessageModel()
    app = App(conv_model, msg_model)         # 业务桥接

    ctx = engine.rootContext()
    ctx.setContextProperty("app", app)
    ctx.setContextProperty("conversationModel", conv_model)
    ctx.setContextProperty("messageModel", msg_model)

    qml = Path(__file__).parent / "qml" / "FullMain.qml"
    engine.load(qml.as_uri())
    if not engine.rootObjects():
        return 1
    return qapp.exec()
```

## 2. `app` 属性（Property）—— 完整清单

每个属性需 `Property(type, fget, notify=xxxChanged)`。类型：bool/int/str/"QVariantList"。

| 属性 | 类型 | 含义 |
|---|---|---|
| `isBusy` | bool | 通用后台忙 |
| `isPipelineBusy` | bool | 解密流水线忙 |
| `pipelinePhase` | str | 当前阶段文案 |
| `errorText` | str | 非空=有错 |
| `pageIndex` | int | 0=控制台 1=查看器 |
| `statusText` | str | 底部状态行 |
| `statsText` | str | 会话/消息统计 |
| `runtimeText` | str | 运行时信息（版本/架构/微信路径） |
| `logText` | str | 运行日志全文 |
| `weixinExe` | str | 微信程序路径（.app） |
| `accountDir` | str | 当前账号目录 |
| `accountChoices` | QVariantList(str) | 账号目录列表 |
| `outputDirPath` | str | 输出目录 |
| `timeoutSeconds` | int | 等待秒数（取密钥超时复用） |
| `keyStatus` | str | Key 状态（含"可复用"→绿） |
| `keyFingerprint` | str | Key 指纹 |
| `dbStatus` | str | 三库齐全状态 |
| `imageKeyStatus` | str | 图片密钥状态（含"可用"→绿） |
| `imageKeyXor` | str | 图片 XOR/scheme（`--`=无） |
| `imageKeyFingerprint` | str | 图片密钥指纹 |
| `keyStorePath` | str | 密钥库位置（Mac：Keychain 标识） |
| `mcpAddress` | str | MCP 地址 |
| `mcpHealth` | str | 健康检查 |
| `mcpStatus` | str | MCP 状态（含"运行中"→绿） |
| `mcpConfig` | str | MCP 配置路径 |
| `selectedUsername` | str | 当前会话 |
| `currentTitle` | str | 当前会话标题 |

## 3. `app` 槽（Slot）—— 完整清单

控制台：`autoDetect()` `refresh()` `refreshParameters()` `setWeixinExe(str)` `chooseWeixinExe()` `setAccountDir(str)` `chooseAccountDir()` `setOutputDir(str)` `chooseOutputDir()` `setTimeoutSeconds(str)` `startDecrypt()` `verifyExisting()` `openOutput()` `extractImageKey()` `chooseKeyStore()`

MCP：`startMcp()` `stopMcp()` `copyMcpAddress()` `generateMcpConfig()`

导航/查看器：`showDashboard()` `showViewer()` `setConversationFilter(str)` `selectConversation(str)` `searchCurrentMessages(str)` `loadOlderMessages()` `resolveMedia(int)` `openMedia(int)` `toast(str)`

## 4. `app` 信号（Signal）

- `toast(str)`（既作信号被 `Connections` 消费，也作方法被 QML 调用；实现见 §6）
- `messageLoadFinished()`
- 每个属性一个 `xxxChanged`（NOTIFY）

## 5. 文案本地化（Windows→Mac）

复用 QML 时替换字符串（不改结构），集中到 `ui_qt/strings.py`：

| Windows | Mac |
|---|---|
| "Windows x64 · 本地只读 · Qt Quick" | "macOS · 本地只读 · Qt Quick" |
| "自动寻找当前电脑的 Weixin.exe" | "自动寻找本机 WeChat.app 与账号目录" |
| "Hook 安装后退出微信，再重新登录…" | "首次按向导取密钥；已保存则直接解密" |
| "微信程序"（Weixin.exe） | "微信程序"（WeChat.app） |

## 6. App(QObject) 实现骨架（ui_qt/app.py）

```python
from PySide6.QtCore import QObject, Property, Signal, Slot, QThreadPool, QRunnable, QMetaObject, Qt, Q_ARG

class App(QObject):
    # ---- 信号（属性变更 + 业务事件）----
    isBusyChanged = Signal()
    isPipelineBusyChanged = Signal()
    errorTextChanged = Signal()
    pageIndexChanged = Signal()
    statusTextChanged = Signal()
    accountChoicesChanged = Signal()
    keyStatusChanged = Signal()
    # …（每属性一个）…
    messageLoadFinished = Signal()
    toast = Signal(str)                       # 信号

    def __init__(self, conv_model, msg_model):
        super().__init__()
        self._pool = QThreadPool.globalInstance()
        self._conv = conv_model; self._msg = msg_model
        self._is_busy = False; self._error = ""; self._page = 0
        self._accounts: list[str] = []; self._selected = ""
        # 业务服务（无 Qt 依赖）
        self._env = None; self._repo = None; self._store = KeyringStore()
        # …其余状态字段…

    # ---- Property（示例，其余同构）----
    @Property(bool, notify=isBusyChanged)
    def isBusy(self): return self._is_busy
    def _set_busy(self, v):
        if v != self._is_busy: self._is_busy = v; self.isBusyChanged.emit()

    @Property(str, notify=errorTextChanged)
    def errorText(self): return self._error

    @Property("QVariantList", notify=accountChoicesChanged)
    def accountChoices(self): return self._accounts

    @Property(int, notify=pageIndexChanged)
    def pageIndex(self): return self._page

    # ---- Slot：耗时操作走后台线程 ----
    @Slot()
    def autoDetect(self):
        self._run_bg(self._do_autodetect, phase="检测本机")

    def _do_autodetect(self):
        env = platform_mac.discover()          # 后台线程执行
        # 回主线程更新属性/模型
        self._post(lambda: self._apply_env(env))

    @Slot(str)
    def selectConversation(self, username: str):
        self._selected = username; self.selectedUsernameChanged.emit()
        self._run_bg(lambda: self._load_messages(username, reset=True),
                     phase="加载消息")

    @Slot(int)
    def resolveMedia(self, index: int):
        self._run_bg(lambda: self._resolve_media(index))

    @Slot(str)
    def toast(self, message: str):             # 同名方法：内部转发到信号
        self.toast.emit(message)               # 注：实际实现用不同内部名避免与 Signal 重名

    # ---- 线程与回主线程 ----
    def _run_bg(self, fn, phase: str = ""):
        if phase: self._set_phase(phase)
        self._set_busy(True)
        task = _Task(fn, on_done=lambda: self._set_busy(False),
                     on_error=self._on_error)
        self._pool.start(task)

    def _post(self, fn):
        # 切回主线程执行 fn（更新 QObject/模型必须在主线程）
        QMetaObject.invokeMethod(self, "_run_on_main", Qt.QueuedConnection,
                                 Q_ARG("QVariant", fn))
    @Slot("QVariant")
    def _run_on_main(self, fn): fn()

    def _on_error(self, exc: Exception):
        msg = exc.user_message if isinstance(exc, WeChatDecryptorError) else "发生未知错误，详见运行日志"
        self._post(lambda: self._set_error(msg))
```

```python
class _Task(QRunnable):
    def __init__(self, fn, on_done, on_error):
        super().__init__(); self.fn=fn; self.on_done=on_done; self.on_error=on_error
    def run(self):
        try: self.fn()
        except Exception as e: self.on_error(e)
        finally: self.on_done()
```

> `toast` 同时是 QML 调用的方法与 `Connections` 的信号：实现上定义 `toast = Signal(str)`，并提供 `@Slot(str) def showToast(...)` 内部 emit；QML 里 `app.toast("x")` 若必须调方法，则把 Signal 命名为 `toast` 且 PySide6 允许 `app.toast.emit`——为兼容 QML 现写法 `app.toast("x")`，用一个 `@Slot(str)` 名为 `toast` 的方法 + 另一个信号 `toastRequested`，并在 `Connections` 里改连 `onToastRequested`。**（复用 QML 时按此微调一处连接名，记于 strings/QML。）**

## 7. 模型骨架（ui_qt/models.py）

### 7.1 ConversationModel
```python
from PySide6.QtCore import QAbstractListModel, QModelIndex, Qt

class ConversationModel(QAbstractListModel):
    UsernameRole = Qt.UserRole + 1
    DisplayNameRole = Qt.UserRole + 2
    InitialsRole = Qt.UserRole + 3
    SummaryRole = Qt.UserRole + 4
    LastTimeRole = Qt.UserRole + 5
    UnreadCountRole = Qt.UserRole + 6
    AvatarUrlRole = Qt.UserRole + 7

    _ROLES = {
        UsernameRole:b"username", DisplayNameRole:b"displayName",
        InitialsRole:b"initials", SummaryRole:b"summary",
        LastTimeRole:b"lastTime", UnreadCountRole:b"unreadCount",
        AvatarUrlRole:b"avatarUrl",
    }
    def __init__(self): super().__init__(); self._rows: list[ConversationRow] = []
    def roleNames(self): return self._ROLES
    def rowCount(self, parent=QModelIndex()): return len(self._rows)
    def data(self, index, role):
        r = self._rows[index.row()]
        return {
            self.UsernameRole:r.username, self.DisplayNameRole:r.display_name,
            self.InitialsRole:r.initials, self.SummaryRole:r.summary,
            self.LastTimeRole:r.last_time, self.UnreadCountRole:r.unread_count,
            self.AvatarUrlRole:r.avatar_url,
        }.get(role)
    def reset(self, rows):                    # 主线程调用
        self.beginResetModel(); self._rows = rows; self.endResetModel()
```

### 7.2 MessageModel
```python
class MessageModel(QAbstractListModel):
    # roles 覆盖 MessageBubble.qml 全部 property（见 §8）
    ROLES = {Qt.UserRole+1:b"senderName", Qt.UserRole+2:b"senderAvatarPath",
             Qt.UserRole+3:b"senderAvatarUrl", Qt.UserRole+4:b"time",
             Qt.UserRole+5:b"content", Qt.UserRole+6:b"kind",
             Qt.UserRole+7:b"duration", Qt.UserRole+8:b"mediaAvailable",
             Qt.UserRole+9:b"mediaRenderable", Qt.UserRole+10:b"mediaPlayable",
             Qt.UserRole+11:b"mediaUrl", Qt.UserRole+12:b"mediaPath",
             Qt.UserRole+13:b"mediaThumbnailUrl", Qt.UserRole+14:b"mediaThumbnailPath",
             Qt.UserRole+15:b"mediaReason", Qt.UserRole+16:b"mediaAttachment",
             Qt.UserRole+17:b"structuredTitle", Qt.UserRole+18:b"structuredDescription",
             Qt.UserRole+19:b"structuredUrl", Qt.UserRole+20:b"typeLabel"}
    countChanged = Signal()
    def roleNames(self): return self.ROLES
    def rowCount(self, parent=QModelIndex()): return len(self._rows)
    @Property(int, notify=countChanged)
    def count(self): return len(self._rows)              # QML 用 messageModel.count
    def data(self, index, role): ...                     # 映射 MessageRow 字段
    def append(self, rows):                               # 尾部加载新
        n=len(self._rows); self.beginInsertRows(QModelIndex(), n, n+len(rows)-1)
        self._rows += rows; self.endInsertRows(); self.countChanged.emit()
    def prepend(self, rows):                              # 头部插入更早
        self.beginInsertRows(QModelIndex(), 0, len(rows)-1)
        self._rows = rows + self._rows; self.endInsertRows(); self.countChanged.emit()
    def update_row(self, i, row):                         # 媒体解析完成原地刷新
        self._rows[i] = row
        idx = self.index(i); self.dataChanged.emit(idx, idx, list(self.ROLES))
```

## 8. MessageModel 角色 ↔ MessageBubble.qml property（完整）

| role(bytes) | MessageRow 字段 |
|---|---|
| senderName | sender_name |
| senderAvatarPath | (P3) sender_avatar_path |
| senderAvatarUrl | (P3) sender_avatar_url |
| time | time |
| content | content |
| kind | kind |
| duration | duration |
| mediaAvailable/Renderable/Playable | media_available/renderable/playable |
| mediaUrl/mediaPath | media_url/media_path |
| mediaThumbnailUrl/mediaThumbnailPath | media_thumbnail_url/thumbnail_path |
| mediaReason | media_reason |
| mediaAttachment | media_attachment |
| structuredTitle/Description/Url | structured_title/description/url |
| typeLabel | type_label |

## 9. 线程规则（硬性）
- 槽内耗时逻辑一律 `_run_bg`；业务返回后 `_post` 回主线程更新属性/模型。
- 模型 `begin*/end*`、属性 emit 只在主线程。
- sqlite3 连接每后台任务独立打开（见 07 §1）。

## 10. 取密钥向导（KeyCaptureWizard.qml，P4 新写）
消费 `app` 追加接口：
- 属性：`sipStatus`(str)、`captureBusy`(bool)、`captureHint`(str)。
- 槽：`checkSip()`、`startCapture()`、`importManualKey(str)`、`copyText(str)`。
- 信号：`captureFinished(bool success, str fingerprint)`。

## 11. 冒烟测试（full_smoke_test.py）
```bash
QT_QPA_PLATFORM=offscreen python ui_qt/full_smoke_test.py --demo
```
- `--demo`：注入假数据模型，验证 QML 加载无 undefined 绑定、无 QML 警告。
- `--output-dir <解密目录>`：用真实解密库跑无窗口加载。
