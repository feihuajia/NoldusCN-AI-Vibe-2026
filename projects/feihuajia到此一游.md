# RemoteControl 工具（rcs_toggle）功能介绍

> 一句话：**手动操作 BrainVision Recorder 采集脑电的同时，自动同步启停 Noldus FaceReader 面部表情采集。**
>
> 触发方式：**按住一个键盘键 + 鼠标左击**，即向 FaceReader 发送「开始 / 停止」命令。左击照常传给你点击的程序（比如 Recorder 的 Start/Stop 按钮），手动操作与自动触发互不影响。

---

## 1. 解决什么问题

在生理/情绪多模态实验里，经常需要同时记录**脑电（EEG，BrainVision Recorder）**和**面部表情视频（Noldus FaceReader）**。理想情况下两者必须**同时开始、同时结束**，否则数据后续对不齐。

现实困难：
- Recorder 和 FaceReader 是两套独立软件，各自有 Start/Stop 按钮，手动分别点击很难做到真正同时（误差常在零点几秒到几秒）。
- 让程序自动控制 Recorder 比较麻烦（需要 Recorder 的 Remote Control Server 授权/安装），且在某些实验流程里研究员本来就**想亲自手动点 Recorder 的按钮**来确认状态。

**本工具的思路**：不动 Recorder（由你手动点），只「旁路」同步 FaceReader。当你手指按住指定键去点 Recorder 按钮的同一时刻，工具自动给 FaceReader 也发一条启停命令，从而实现毫秒级对齐。

---

## 2. 核心功能

| 功能 | 说明 |
|------|------|
| **FaceReader 同步启停** | 通过 FaceReader 的 External Control API 发送 `Start Analyzing` / `Stop Analyzing` 命令 |
| **组合键触发** | `按住 F9 + 左击 = 开始`，`按住 F10 + 左击 = 停止`（键可在 cfg 改） |
| **左击透传不拦截** | 鼠标左击会原样传给被点击的程序，你可以照常手动点 Recorder 按钮 |
| **常驻 TCP + 自动重连** | 与 FaceReader 保持长连接，断线自动重连，命令发出即返回 |
| **去抖动** | 同一动作 350ms 内去重，防止左键抖动/连点发一堆重复命令 |
| **文本配置无需重打包** | 端口、host、热键全写在 `rcs_toggle.cfg`，改完重启 exe 即生效 |

> 注意：**当前版本不再自动控制 Recorder**。Recorder 的 Start/Stop 全部由你手动点击完成，工具只负责「顺带」同步 FaceReader。

---

## 3. 工作原理

```
                   ┌──────────────────────────────────────┐
   你的手 ──► 按住 F9/F10 + 鼠标左击
                   │  左击照常传递                         │
                   ▼                                      ▼
        ┌──────────────────┐               ┌─────────────────────────┐
        │  Recorder 按钮    │  (你手动点击)  │  rcs_toggle.exe          │
        │  Recorder 开始/停 │ ◄───────────  │  全局鼠标钩子捕获左击     │
        └──────────────────┘               │  + 读键盘物理状态         │
                                            │  → 组装 FaceReader 帧     │
                                            └────────────┬─────────────┘
                                                         │ TCP (二进制帧+XML)
                                                         ▼
                                            ┌─────────────────────────┐
                                            │  FaceReader API          │
                                            │  Start/Stop Analyzing    │
                                            └─────────────────────────┘
```

关键设计（踩坑后的选择）：

1. **左击捕获**：用 `mouse` 库的 `WH_MOUSE_LL` 全局低层钩子（实测可靠）。
2. **键盘状态读取**：**不**用 `keyboard` 库的 `is_pressed`——在部分会话下它的钩子会静默失效（能注册但收不到事件）。改用 Win32 `GetAsyncKeyState` 直接读物理键状态，微秒级、无时序问题、不依赖任何钩子线程。
3. **FaceReader 协议**：Noldus 私有协议，二进制帧 + XML，小端 int32：
   - 帧 = `[int32 总长] [int32 TypeNameLen] [TypeName] [Xml]`，均 UTF-8
   - `TypeName = "FaceReaderAPI.Messages.ActionMessage"`
   - `Action = FaceReader_Start_Analyzing / FaceReader_Stop_Analyzing`
   - 回报为 `ResponseMessage`（`FaceReader_Sends_Success` / `_Error`），按长度前缀逐帧解析
4. **必须管理员权限运行**：全局鼠标钩子否则不生效。

---

## 4. 文件结构

```
RemoteControl/
├── rcs_toggle.py          # 主程序源码（核心：FaceReaderLink + 鼠标钩子触发）
├── rcs_toggle.cfg         # 配置模板（host / port / 热键）
├── dist/
│   ├── rcs_toggle.exe     # 已打包可执行（拷到采集机直接用）
│   └── rcs_toggle.cfg     # 随 exe 分发的配置
├── fake_rcs_6700.py       # 本机自检：模拟 RCS 在 6700 端口监听（不用装真 Recorder）
├── test_fr_link.py        # 单测：用假 FaceReader 验证帧编码/发送/解析往返
├── test_rcs_link.py       # 单测（遗留）：验证旧的 RCSLink 逻辑（见下方说明）
├── DEPLOY.md              # 采集机部署前置清单（更偏「怎么跑起来」的步骤）
└── Setup/
    ├── Application Programming Interface - FaceReader 10 Technical Note.pdf
    ├── RCS_BrainVisionRecorder_20190523.pdf
    ├── RemoteControlClient_Installation.msi      # 官方 RemoteControl 客户端安装包
    └── RemoteControlServer_Installation.msi      # 官方 RemoteControl 服务端安装包
```

### 关于遗留的 RCS 代码
项目名里的 **RCS = Remote Control Server**（BrainVision Recorder 的远程控制服务端）。早期版本曾通过 RCS 自动控制 Recorder 的启停，`test_rcs_link.py` 和 `fake_rcs_6700.py` 都是那个时期的自检脚本。当前主线（`rcs_toggle.py`）已**移除 `RCSLink`，改为只同步 FaceReader、Recorder 全手动**，因此：
- `test_fr_link.py` → 现在仍可用，覆盖 FaceReaderLink 全流程。
- `test_rcs_link.py` → 会因 `from rcs_toggle import RCSLink` 失败而报错，属历史遗留，仅供参考。
- `fake_rcs_6700.py` → 仍是「不装 Recorder 也能看连上效果」的轻量演示。

`Setup/` 里的官方 MSI 和 PDF 是 RCS 方案的原始资料，如果将来想恢复「全自动控制 Recorder」可在此基础上扩展。

---

## 5. 配置文件 rcs_toggle.cfg

改文本即可，**无需重新打包**：

```ini
[facereader]
enabled = true          # false 则本工具完全不触发（纯手动）
host = 127.0.0.1        # 同机用 127.0.0.1；跨机填目标机 IP 并放行防火墙
port = 9090             # FaceReader API 端口；以 Settings>Data Export>External communication 显示为准

[hotkeys]
start = f9              # 按住此键 + 左击 = 开始
stop = f10              # 按住此键 + 左击 = 停止
                        # 支持 F1-F24 / A-Z / 0-9 / space / ctrl / shift / alt
```

规则：
- 左击时**未按住**任何指定键 → 什么都不发生。
- 同时按住 start/stop 两键 → **开始优先**。

---

## 6. 怎么跑起来（采集机）

1. **Recorder** 开 → 加载 workspace → 进入 Monitoring（手动操作，工具不介入）。
2. **FaceReader** 开 → 打开一个 Analysis → `Settings > Data Export > External communication (API)` 勾选 **Enable external control**，记下显示的端口。
3. 把 `dist/rcs_toggle.exe` + `rcs_toggle.cfg` 拷到采集机同一目录，确认 cfg 端口与上一步一致。
4. **以管理员身份**运行 `rcs_toggle.exe`。
5. 手动点 Recorder 的 Start 按钮时**按住 F9** → Recorder 开始 + FaceReader 同步开始；点 Stop 时**按住 F10** → 同步停止。`Ctrl+C` 退出。

### 运行时日志含义（节选）

| 现象 | 含义 |
|------|------|
| `[fr ] 已连接 127.0.0.1:xxxx` | FaceReader 链路连上 ✓ |
| 按住 F9 左击 → `>>> [FR ] 开始: 已发送`，随后 `[fr ] < FaceReader_Sends_Success` | FaceReader 开始成功 ✓ |
| `[fr ] < FaceReader_Sends_Error ...` | FaceReader 没打开 Analysis / 没启用 external control |
| 一直 `连接失败` | 端口不对 / FaceReader 没起 API / 防火墙 |
| 按住键左击完全没反应 | **没用管理员身份运行**（全局钩子未生效） |

更详细的分步部署清单见同目录 **`DEPLOY.md`**。

---

## 7. 依赖与打包

- **运行依赖**：`mouse`（需管理员权限，否则全局鼠标钩子不生效）；`ctypes` 等均为标准库。
- **打包命令**（仅当改了 `rcs_toggle.py` 源码才需要；改 cfg 不用）：

  ```bash
  python -m PyInstaller --onefile --console --name rcs_toggle --hidden-import=mouse rcs_toggle.py
  ```

---

## 8. 已知限制

- **延迟**：命令发出本身是毫秒级；Recorder 进入 Recording 状态由其自身决定，工具**不等待** Recorder（Recorder 是你手动点的，工具只对齐 FaceReader 的触发时刻）。
- **权限**：必须管理员运行，否则鼠标钩子无效、组合无反应。
- **前提**：FaceReader 必须已打开一个 Analysis 并启用 external control，否则会收到 `FaceReader_Sends_Error`。
