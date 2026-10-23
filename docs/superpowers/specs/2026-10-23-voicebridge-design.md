# VoiceBridge 设计规格

- 日期：2026-10-23
- 状态：待评审
- 决策来源：参考仓库 `../remote-mic-app`（SayAll / 无线麦）调研 + 五轮设计澄清

---

## 1. 背景与目标

### 1.1 要解决的问题

用户希望用 Android 手机**代替小米蓝牙遥控器**，充当 Mac 的语音键盘：

1. 手机麦克风的声音要成为 Mac 上某个语音输入法的音频来源；
2. 手机上的按键要能触发 Mac 的键盘快捷键；
3. 按下语音键时，要能把焦点切到当前 App 的输入框；
4. Mac 端需要人工批准手机的连接请求。

### 1.2 核心设计取舍

**Mac App 不认识、不适配任何具体输入法。**

参考项目 SayAll 为此维护了一张"遥控器 × 输入法"能力矩阵（5 种输入法 × 3 种遥控器 × 2 种收音模式 × 快捷键侧别），并为此反复返工：ChatterFly 在 1.9.21 被以"不再支持"从首次设置中移除（`remote-mic-app/Resources/zh-Hans.lproj/ReleaseHistory.md:17`），历史配置需迁移为未选中（`Sources/RemoteMic/AppSettings.swift:1032`）。

VoiceBridge 把这一层完全交给用户：

| 事项 | 谁负责 |
| --- | --- |
| 用哪个输入法 | 用户 |
| 输入法的收音快捷键是什么 | 用户 |
| 输入法的麦克风选哪个设备 | 用户 |
| 长按收音 / 按一次收音 | 用户 |
| 把手机声音变成虚拟麦克风 | VoiceBridge |
| 按下用户配置的触发键 | VoiceBridge |
| 执行用户配置的快捷键 | VoiceBridge |
| 切换输入框 | VoiceBridge |
| 批准/拒绝手机连接 | VoiceBridge |

### 1.3 成功标准

用户在目标 App 的输入框里，按住手机上的语音键说话，松开后文字出现在输入框里——全程不需要碰 Mac。

---

## 2. 范围

### 2.1 包含

- macOS App（Swift）：局域网服务、配对批准、快捷键录制与注入、AX 聚焦输入框、语音会话状态机、虚拟麦克风写入
- Android App（Kotlin/Compose）：前台录音服务、WebSocket 客户端、遥控 UI
- 虚拟麦克风驱动（C++17 + libASPL）：AudioServerPlugIn，transport 报 USB
- 跨端协议：控制帧 + PCM 音频帧

### 2.2 明确不包含

| 不做 | 原因 |
| --- | --- |
| 适配任何具体输入法 | 见 1.2，用户自行配置 |
| 输入法能力矩阵 / 工具识别 / 截图指引 | 上述决定的直接结果 |
| 网页版 / Apple Watch 版 / iOS 版 | 非本次范围 |
| 多客户端并发 | 只允许一个手机连接 |
| 音频压缩 | LAN 带宽无压力，PCM 直传最简单 |
| 跨互联网连接 | 局域网足够，且避免服务器运维 |
| 麦克风降噪 / 增益 / 降噪模型 | 非核心，可后续 |
| 按钮手势（双击 / 长按） | v1 不做 |
| 音频落盘 | 隐私边界：PCM 只存在于内存与共享内存 |

---

## 3. 术语

| 术语 | 含义 |
| --- | --- |
| 触发键 | 用于启动/停止目标输入法收音的单个修饰键（Fn / 左Command / 右Command / 左Option / 右Option） |
| 指令键 | 手机上除语音键外的语义按键，映射到 Mac 快捷键或系统动作 |
| 虚拟麦克风 | VoiceBridge 提供的 Core Audio 输入设备，把写入的 PCM 暴露为系统麦克风 |
| 语音会话 | 一次"按下 → 收音 → 松开 → 排空 → 释放" 的完整生命周期 |
| 预录 | Android 端在收到 voiceStart 前就录音并本地缓冲的行为 |
| 预卷 | Mac 端在 voiceReady 前收到的音频暂存，不写入虚拟麦克风 |

---

## 4. 系统架构

```
┌──────────────────┐   WebSocket（局域网）    ┌────────────────────────────────┐
│  Android App     │ ───────────────────────► │  Mac App (Swift)               │
│  Kotlin/Compose  │  控制帧 JSON / 音频帧    │  ├ PairingService  配对+批准   │
│                  │      二进制 PCM          │  ├ ProtocolCodec   编解码      │
│  AudioRecord     │ ───────────────────────► │  ├ KeyInjector     CGEvent 注入│
│  16k/mono/Int16  │                         │  ├ ShortcutCapture 快捷键录制  │
│  预录环形缓冲     │                         │  ├ ComposerFocus   AX 聚焦     │
└──────────────────┘                         │  ├ VoiceSession    会话状态机  │
                                             │  └ AudioBridge     共享内存写入│
                                             └───────────────┬────────────────┘
                                                             │ POSIX 共享内存（SPSC 无锁环）
                                                             ▼
                                             ┌────────────────────────────────┐
                                             │ VoiceBridge.driver             │
                                             │ AudioServerPlugIn (libASPL)    │
                                             │ 设备 "VoiceBridge Mic"         │
                                             │ transport = USB                │
                                             └───────────────┬────────────────┘
                                                             │ Core Audio
                                                             ▼
                                              用户的语音 App（ChatterFly / 豆包 / 微信…）
```

### 4.1 驱动路线决策

**选定：AudioServerPlugIn（HAL 插件）+ Apple 开源 libASPL，自写代码。**

决策过程与依据：

1. Apple 官方文档明确：**AudioDriverKit 仅支持物理音频设备**（"When creating a virtual device, best practice is to use an Audio Server Driver Plug-in instead… AudioDriverKit only supports physical audio devices."）。且 `AudioDriverKitTypes.h` 中不存在 `IOUserAudioTransportType Virtual`，dext 中设置 transport 为 virt 只改变图标。
2. BlackHole 上游为 GPL-3.0（官方 LICENSE 确认），fork 会把整个项目拖入 GPL 传染。
3. 存在 MIT 许可的同场景完整实现 `sheeshmohsin/noise-ai`：C++17 + AudioServerPlugIn (libASPL) + Swift/SwiftUI，IPC 为 POSIX 共享内存（音频）+ localhost TCP（控制）。**本项目的驱动结构以其为模板。**
4. AudioServerPlugIn 路线**不需要 DriverKit entitlement**，只需普通 Developer ID 签名 + 公证，省去付费账号与 Apple 审批排队。

代价：仍需少量 C++，且 transport=USB 需自行实现（一处属性赋值）。

### 4.2 libASPL 角色

libASPL（`ExistentialAudio/libASPL`，Apple 官方示例库）提供 AudioServerPlugIn 的样板代码（属性表、I/O 循环、设备对象注册），使自写驱动只需关注：设备属性（含 transport=USB）、共享内存环形缓冲、输入流填充。

### 4.3 仓库结构

```
voicebridge/
├── mac-app/                 Swift Package（可执行 target + 库 target）
├── android-app/             Gradle 工程（Kotlin + Compose）
├── driver/                  CMake 工程（C++17 + libASPL）
├── protocol/                协议定义（JSON schema + 帧格式 + 共享内存头）
└── docs/superpowers/specs/  设计规格
```

`protocol/` 下的协议定义是**跨端唯一事实源**：Mac 端、Android 端、驱动端都从这里取常量与布局，避免三处各写一遍导致漂移。

---

## 5. 组件设计

### 5.1 Mac App（Swift，macOS 14+）

| 模块 | 单一职责 | 关键约束 / 参考 |
| --- | --- | --- |
| `PairingService` | WebSocket 服务、握手、两位码批准、长期信任 | 只允许一个活动客户端；配对流程参考 `remote-mic-app/Sources/RemoteMic/BridgeAppModel.swift:6277` `requestPhoneApproval` |
| `ProtocolCodec` | 控制帧 JSON 编解码 + 音频帧二进制解包 | 纯函数，无 I/O，可单测 |
| `ShortcutRegistry` | 语义按钮 id → 快捷键 / 系统动作 的映射表 | 纯数据，可导入导出 |
| `KeyInjector` | CGEvent 注入，成对 down / up | 注入事件必须写 `eventSourceUserData` 标记，避免被 `ShortcutCapture` 当成用户输入捕获回去（参考 `remote-mic-app/Sources/RemoteMic/KeyboardInjector.swift:1878` `postKeyState`） |
| `ShortcutCapture` | event tap 录制任意快捷键 | 组合键走 `keyDown`；独立修饰键走 `flagsChanged` 配对检测（参考 `remote-mic-app/Sources/RemoteMic/ShortcutCaptureMonitor.swift`） |
| `ComposerFocus` | AX 遍历找可编辑输入框并设置焦点 | 安全边界见 5.2（参考 `remote-mic-app/Sources/RemoteMic/KeyboardInjector.swift:662` `focusFrontmostComposer`） |
| `VoiceSession` | 语音会话状态机 | 见第 7 节 |
| `AudioBridge` | 共享内存写入 + 排空 | 无锁 SPSC；积压时丢最旧样本并计数上报 |

**触发键的可选集合**：Fn、左 Command、右 Command、左 Option、右 Option。

限定为**单个修饰键**而非任意组合键，理由：

1. 长按语义要求明确的 down/up 配对，组合键会引入"哪个键是触发器"的歧义；
2. 会话期间用户仍在打字，组合键修饰位会污染正常输入；
3. 目标输入法的收音快捷键本身就是单键。

参考：`remote-mic-app/Sources/RemoteMic/VoiceKeyMode.swift:60` 明确排除了 Shift / Control。

**v1 的触发键集合固定为这 5 个**，不提供自定义按键或组合键。理由见上；目标输入法若只认其他按键，属于 v1 范围外的限制，需在文档中显式告知用户。

### 5.2 AX 聚焦的安全边界

`ComposerFocus` 只对**安全可编辑目标**执行聚焦，其余情况返回失败并保持现状。

允许的角色：`AXTextArea`、`AXTextField`、`AXComboBox`。
拒绝：`AXSecureTextField`（密码框）。
拒绝的语义文本关键词（中英双语，参考 `remote-mic-app/Sources/RemoteMic/VoiceInputDestinationCoordinator.swift:80`）：

```
password, passcode, secret, api key, apikey, token, credit card,
search, find, filter, address bar, settings, preferences, command palette,
密码, 口令, 密钥, 令牌, 信用卡, 搜索, 查找, 筛选, 设置, 偏好
```

遍历顺序：`AXChildrenInNavigationOrder` → `AXVisibleChildren` → `AXContents` → `AXChildren`。
遍历上限 5,000 个元素（防卡死）。重试 12 次 × 250ms（最多 3 秒），因为 Web 内容树在可编辑属性设置后需要 1–2 秒才稳定。

**聚焦是独立动作，不自动串进语音会话。** 语音会话开始前不自动聚焦（会引入最长 3 秒的不可控延迟，拖垮首个语音响应）。用户可在 Android 端单独点击"聚焦"按钮，或由 Mac 端在收到显式 `focusComposer` 指令时执行。

### 5.3 Driver（C++17，libASPL）

| 文件 | 职责 |
| --- | --- |
| `VoiceBridgeDriver.cpp` | libASPL 样板 + 设备属性 + I/O 回调 |
| `SharedRingBuffer.h` | SPSC 无锁环形缓冲（单生产者：Mac App；单消费者：驱动） |
| `PluginIPC.h` | 共享内存命名、布局、协议头定义 |

设备属性要求：

| 属性 | 值 | 理由 |
| --- | --- | --- |
| 设备名 | `VoiceBridge Mic` | 用户在语音 App 里能认出来 |
| 传输类型 | `kAudioDeviceTransportTypeUSB` | 部分第三方语音 App 会过滤非 USB 设备（豆包即如此，故 SayAll 的 `MiRemoteV 2ch` 需要此补丁） |
| 采样率 | 48000 Hz（与 Core Audio 设备默认一致） | 驱动侧不做重采样；重采样在 Mac App 侧完成 |
| 声道 | 2 | 与主流虚拟设备一致 |
| 格式 | 线性 PCM | — |

Mac App 侧写入 16 kHz 单声道，由 `AVAudioEngine` 完成 16k → 48k / 1ch → 2ch 转换。

**共享内存协议**（`protocol/shm_layout.h`）：

```
[8B  magic "VBSHM01\0"]
[4B  共享内存总字节数]
[4B  环形缓冲容量（帧）]
[4B  写指针（原子）]
[4B  读指针（原子）]
[4B  数据就绪标志]
[4B  累计写入样本数]
[4B  累计丢弃样本数]
[4B  会话序号]
[... 环形数据区：交错存储 left/right Int16]
```

写入方（Mac App）先写数据再更新写指针；读取方（驱动）先读数据再更新读指针。仅需内存屏障，不需要锁。

### 5.4 Android App（Kotlin，minSdk 26）

| 模块 | 单一职责 |
| --- | --- |
| `VoiceService` | 前台服务 + 常驻通知 + 保活 |
| `AudioCapture` | AudioRecord 16 kHz mono + 预录环形缓冲 |
| `WsClient` | OkHttp WebSocket，控制帧 + 二进制帧 |
| `RemoteUi` | Compose：大语音键 / 聚焦键 / 动作面板 / 状态灯 |
| `Discovery` | NsdManager 局域网发现 + 手动输入 IP 兜底 |

Android 端音频规格：

| 参数 | 值 | 理由 |
| --- | --- | --- |
| 采样率 | 16000 Hz | 语音足够；与 Mac 端写入格式一致，免重采样 |
| 声道 | 单声道 | |
| 格式 | Int16 PCM | |
| 音源 | `MediaRecorder.AudioSource.VOICE_RECOGNITION` | 系统自带 AGC + 降噪 |
| 缓冲 | 20 ms（320 样本） | 延迟与吞吐平衡 |
| 预录环缓冲 | 1 秒（16,000 样本） | 覆盖 Mac 端 prepare 阶段 |

**注意**：Android 同时只允许一个 App 录音。若用户正在别的 App 里录音，`AudioRecord` 会失败——需明确报错，不要静默重试。

---

## 6. 协议规范

传输：局域网内的一条 WebSocket 连接（`ws://`）。控制帧用文本帧，音频帧用二进制帧。

### 6.1 二进制音频帧

```
偏移  长度  内容
0     4     magic = "VB01"
4     1     type = 0x01 (PCM)
5     1     flags（保留，0）
6     4     sessionId（大端 uint32）
10    4     seq（大端 uint32，单调递增）
14    2     sampleCount（大端 uint16，本帧样本数）
16    N     Int16 PCM，小端
```

### 6.2 控制帧（JSON）

所有帧含 `"v"`（协议版本，当前 `1`）与 `"t"`（类型）。

```json
{"v":1,"t":"hello","device":"Pixel 8","fingerprint":"<sha256>"}
{"v":1,"t":"pairingChallenge","code":"42"}
{"v":1,"t":"pairingResult","approved":true}
{"v":1,"t":"voiceStart"}
{"v":1,"t":"voiceReady"}
{"v":1,"t":"voiceStop"}
{"v":1,"t":"button","id":"ok","phase":"press"}
{"v":1,"t":"focusComposer"}
{"v":1,"t":"ping"}
{"v":1,"t":"pong"}
{"v":1,"t":"error","code":"voice_output_unavailable","detail":""}
```

### 6.3 错误码

| code | 含义 | 触发方 |
| --- | --- | --- |
| `voice_output_unavailable` | 虚拟麦克风未安装 / 驱动未加载 / 共享内存未就绪 | Mac → Android |
| `voice_busy` | 已有语音会话在进行中 | Mac → Android |
| `voice_timeout` | `preparing` 阶段超时（5 秒）未就绪 | Mac → Android |
| `accessibility_denied` | 辅助功能权限缺失，无法注入触发键 | Mac → Android |
| `pairing_rejected` | 用户在 Mac 端拒绝了配对 | Mac → Android |
| `protocol_mismatch` | 协议版本不一致 | Mac → Android |

**协议版本协商**：`hello` 帧中的 `"v"` 必须与 Mac 端支持的版本相同，否则 Mac 回
`error{protocol_mismatch}` 并关闭连接，双方均不做降级猜测。版本号在 `protocol/` 中单点定义。

未知错误码的接收方必须按"未知错误"处理：UI 展示通用失败文案，不得崩溃。

### 6.4 语义按钮 id

Android 端只发送**语义 id**，不发送键码。映射关系由 Mac 端持有，改映射不需发版 Android。

固定集合（对齐参考项目 `remote-mic-app/Sources/PublicRemoteCompatibility/Core/RemoteCompatibility.swift:3`）：

```
power, up, left, ok, right, down, back, volumeUp, home, volumeDown, menu, tv
```

`phase` ∈ `press` | `release`。默认映射（参考 `remote-mic-app/TECHNICAL.md:87`）：

| 按钮 | 默认动作 |
| --- | --- |
| 方向 | 方向键 |
| ok | Return |
| back | Delete（退格） |
| home | 显示桌面（Fn-F11） |
| menu | macOS 上下文菜单键 |
| tv | Command-Tab |
| power | Escape |
| volumeUp / volumeDown | 系统音量增减 |

用户可改为：任意键盘快捷键、系统音量、播放控制、打开已安装 App、聚焦输入框、禁用。

### 6.5 配对流程

```
Android                                   Mac
──────                                    ───
hello{device,fingerprint}  ────────────►  比对已信任指纹
                                            ├ 已信任 → 直接进入已配对状态
                                            └ 未信任 → pairingChallenge{code:"42"}
pairingChallenge 回显 ◄───────────────────  （Mac 端弹窗：设备名 + 两位码）
                                            用户点"同意" → pairingResult{approved:true}
                                                        → 存入信任列表
                                                        → pairingResult{approved:false}
```

- 两位确认码由 Mac 端生成并显示，Android 端展示给用户比对（防止同网段其他设备冒充）。
- 信任列表以设备指纹（`fingerprint`）为键持久化。
- 指纹由 Android 端生成（Android ID + 设备型号的 SHA-256），不采集硬件序列号。

---

## 7. 语音会话时序与状态机

### 7.1 时序

```
Android                                     Mac
──────                                      ───
用户按住语音键
 ├─ AudioRecord.start()
 ├─ 本地预录环缓冲开始写入
 └─ ── voiceStart ────────────────────────►  收到
                                            ├ 预卷缓冲开始（不写驱动）
                                            ├ 确认共享内存 + 驱动就绪
                                            │   ├ 未就绪 → error{voice_output_unavailable}
                                            │   └ 就绪
                                            ├ ◄── voiceReady ─────  "可以说了"
                                            ├ 注入触发键 down
                                            └ 延迟 180ms（等目标 App 开麦）
 推流 PCM（含预录缓冲）◄──────────────────  开始写共享内存 → 驱动 → 虚拟麦克风
 （流式，边说边传）                          （目标语音 App 录音中）
 ...
用户松开语音键
 ├─ 继续推 300ms 尾音
 └─ ── voiceStop ─────────────────────────►  停止写入
                                            ├ 等驱动排空（drain）
                                            └ 注入触发键 up
```

### 7.2 首字丢失的三道防线

目标 App 收到触发键后需要 50–200ms 才打开麦克风，这段时间写入的音频会丢失。三道防线：

1. **Android 预录**：从用户按下的瞬间就录音（本地环缓冲 1 秒），推流时把预录一起带上。覆盖 Mac 端 prepare 阶段的全部延迟。
2. **Mac 端预卷**：`voiceReady` 之前收到的音频进预卷，不写驱动。
3. **`voiceReady` 握手**：Mac 端只有在共享内存与驱动都就绪后才回 `voiceReady`，Android 收到才开始推流。

残留风险：从 `voiceReady` 到目标 App 真正开麦之间的窗口。这段时间 Mac 端已开始写驱动，但目标 App 还没读——由**虚拟麦克风自身的 ring buffer** 吸收。驱动侧环形缓冲容量需 ≥ 512 帧（@48 kHz ≈ 10.7 ms）以覆盖该窗口；实现时按 4096 帧（约 85 ms）配置。

### 7.3 Mac 端状态机

```
idle ──voiceStart──► preparing ──ready──► streaming ──voiceStop──► draining ──drained──► idle
                        │                                                        │
                        └── timeout / cancel / disconnect ─────────────────────┘
                                              （释放触发键 + 清空预卷）
```

- `preparing`：预卷缓冲 + 等待驱动就绪 + 等待 `voiceReady` 已发出。
- `streaming`：实时写共享内存。
- `draining`：停止写入，等驱动排空（驱动侧上报已消费样本数），排空后释放触发键。
- 每次 `voiceStart` 递增 `generation`；所有异步回调携带 generation，迟到回调直接丢弃。
- **任何离开 `streaming` 的路径都必须幂等释放触发键**（`releaseVoiceKeyIfNeeded`），避免修饰键卡住导致用户键盘一直带 Fn。这是参考项目反复出现过的真实故障（`remote-mic-app/Testing/VoiceKeyModes.md` 用例 7）。
- 断连立即释放触发键、清空预卷、结束会话。

### 7.4 触发键的成对性

- 长按模式：voiceStart 时 down，drain 完成后 up。
- 点按模式（目标输入法只认点按）：voiceStart 时 down+up 一次，drain 完成后再 down+up 一次。
- 模式由用户在 Mac 端选择；**Android 端负责表达"按住"还是"按一次"**，两者独立。Android 端"按一次"时，发两次 voiceStart/voiceStop 序列。

参考：`remote-mic-app/Sources/RemoteMic/VoiceFnTapSessionController.swift` 的 `startDelay 0.15` / `tapDuration 0.12` / 预卷上限 80,000 样本。

---

## 8. 错误处理

| 场景 | 行为 |
| --- | --- |
| 辅助功能权限缺失 | 拒绝注入，返回明确错误提示给 UI；不静默吞掉 |
| 虚拟麦克风未安装 / 驱动未加载 | 拒绝开始语音会话，回 `error{voice_output_unavailable}` |
| 共享内存未就绪 | 同上 |
| 触发键注入失败 | 回滚 latch，不留按下状态 |
| 音频积压 | 丢弃最旧样本 + 计入 `droppedSamples`，不阻塞 |
| Android 断连 | 立即释放触发键 + 清会话；Android 侧指数退避重连（1s / 2s / 4s … 上限 30s） |
| 目标语音 App 无响应 | **不检测**。本项目不适配输入法，由 App 内"自检页"引导用户自查 |
| Android 麦克风被占用 | 明确报错，不静默重试 |
| macOS 休眠 / 唤醒 | 重新枚举虚拟麦克风；未就绪时拒绝开始会话 |

---

## 9. 权限

| 权限 | 平台 | 用途 | 缺失时行为 |
| --- | --- | --- | --- |
| Accessibility | macOS | 注入快捷键 + AX 聚焦 | 拒绝注入，UI 明确提示 |
| Local Network | macOS 15+ | 局域网连接 | 无法连接，UI 提示去系统设置 |
| Microphone | Android | AudioRecord | App 无法录音，UI 提示 |
| Foreground Service (microphone) | Android 14+ | 录音服务保活 | 服务无法启动 |
| 电池优化白名单 | Android | 防后台被杀 | 引导用户去系统设置加白名单（国产 ROM 必需） |

---

## 10. 配置与持久化

### Mac App

| 配置 | 存储 | 说明 |
| --- | --- | --- |
| 语音触发键 | UserDefaults | 单个修饰键 + 投递模式（长按 / 点按） |
| 快捷键映射 | UserDefaults（JSON） | 语义按钮 id → 动作 |
| 信任设备列表 | UserDefaults（JSON） | 指纹 + 设备名 + 首次信任时间 |
| 虚拟麦克风 UID | UserDefaults | 若存在则自动选中 |

配置支持导出 / 导入（JSON），导入时做字段级校验，旧配置缺字段走默认值。

### Android App

| 配置 | 存储 | 说明 |
| --- | --- | --- |
| Mac 端地址 | DataStore | 自动发现的结果 + 手动覆盖 |
| 设备指纹 | DataStore | 首次生成后固定 |

---

## 11. 平台与工具链

| 项 | 版本 |
| --- | --- |
| macOS | 14 Sonoma+，Apple Silicon 与 Intel 均可 |
| Swift | 6.x，Swift 5 语言模式 |
| Xcode | 16+ |
| C++ | C++17 |
| 构建（驱动） | CMake + Ninja |
| Android minSdk | 26 (Android 8.0) |
| Android targetSdk | 35 |
| Android compileSdk | 35 |
| Kotlin | 2.x |
| AGP | 8.x |
| JDK | 17 |
| Gradle | Wrapper（8.x） |
| 序列化 | 控制帧手写编解码（协议帧仅约 10 种消息，不引入序列化框架）；Android 端如使用 kotlinx.serialization，仅限本地配置存储层 |

---

## 12. 测试策略

### 12.1 Mac App（Swift Testing）

- `ProtocolCodec`：控制帧往返、音频帧解包边界（半包、超长包、错 magic、错 version）
- `ShortcutRegistry`：默认映射、导入导出往返、旧配置缺字段走默认
- `KeyInjector`：down/up 配对、合成事件标记、注入失败回滚
- `ShortcutCapture`：组合键 / 独立修饰键 / 忽略自身合成事件
- `ComposerFocus`：角色白名单、密码框拒绝、敏感关键词拒绝、遍历上限
- `VoiceSession`：状态机全部迁移、generation 隔离迟到回调、断连释放触发键、幂等释放

### 12.2 Driver

- C++ 单测：环形缓冲的并发正确性（生产者 / 消费者线程）、满/空判定
- 手工验收：Audio MIDI Setup 中可见设备、QuickTime 录制能拿到波形、transport 属性为 USB

### 12.3 Android

- JUnit：音频帧打包、预录缓冲、WS 断线重连退避
- Compose UI 测试：语音键按压状态机

### 12.4 跨端

- 协议一致性 golden test：两端的帧布局常量与 JSON 字段名共享 `protocol/` 中的单一事实源，CI 校验一致
- 端到端：手动验收清单（见 13.3）

---

## 13. 里程碑与验证门

### 13.0 实施分解

本规格覆盖完整系统，但不作为单次实施计划的目标。实施按里程碑推进，**每个里程碑独立出计划、独立验收**：

| 计划批次 | 覆盖里程碑 | 说明 |
| --- | --- | --- |
| 计划 1 | M0 + M1 | 验证门 + 协议骨架与配对打通。**这是第一批实施内容** |
| 计划 2 | M2 | 快捷键注入与 AX 聚焦 |
| 计划 3 | M3 | 虚拟麦克风驱动与共享内存 |
| 计划 4 | M4 | 语音链路端到端 |
| 计划 5 | M5 | 分发与自检页 |

M0 先行是硬性要求：G1 / G2 的结论会改变 M2 与 M3 的实现细节（触发键集合、驱动属性），在结论出来前实施 M2 / M3 属于返工。

### 13.1 验证门（**必须在 M1 之前完成**）

| 门 | 内容 | 方法 | 失败后果 |
| --- | --- | --- | --- |
| **G1** | 合成 Fn 能否唤醒目标语音 App | 改造参考仓库的 `remote-mic-app/Testing/VoiceKeyFnPanelProbe.swift`：改 2 行 target matcher 指向 ChatterFly，跑约 20 秒 | 触发键方案需改为 Command / Option；G1 判读规则：按下出现、松开消失 = 长按模式；按下出现、松开仍在、再按下消失 = 点按模式 |
| **G2** | 目标语音 App 能否枚举 transport=USB 的虚拟麦克风 | 先用现成 BlackHole 2ch 测 Chat；再在自研驱动出来后复测 | 若连 USB transport 都不认，需研究目标 App 的设备枚举规则 |
| **G3** | 端到端音频连通 | Android AudioRecord → WS → 共享内存 → 驱动 → Core Audio 设备 | 音频链路需重排 |

G1 与 G2 在没有自研驱动时即可执行（用现成 BlackHole + 改造探针），应作为独立的第一步。

### 13.2 里程碑

| 里程碑 | 内容 | 出口标准 |
| --- | --- | --- |
| **M0** | 执行 G1 / G2 / G3 前置验证 | 结论写入 `docs/`；G1 判定目标输入法的收音模式与快捷键 |
| **M1** | 协议 + 骨架：Mac WS 服务 + Android 客户端，能配对、能发指令 | 双端握手成功；手机点指令，Mac 收到 |
| **M2** | 快捷键注入 + AX 聚焦 + 映射配置 | 真机上按键生效；能聚焦到输入框 |
| **M3** | 虚拟麦克风驱动 + 共享内存 | Audio MIDI Setup 可见；QuickTime 能录到写入的 PCM |
| **M4** | 语音链路端到端（预录 / voiceReady / 排空 / 触发键成对） | 说一句，文字上屏；首字与尾音均不丢 |
| **M5** | 分发 + 自检页 | 干净机器上装完即用；自检页能显示"注入成功 + 有电平" |

### 13.3 端到端验收清单

1. 干净 Mac 安装 App + 驱动 → 音频 MIDI 设置中可见 `VoiceBridge Mic`
2. 安装 Android App → 局域网发现 Mac → 配对（两位码一致）→ 信任
3. 在目标语音 App 中把麦克风选为 `VoiceBridge Mic`
4. 聚焦一个文本框，按住手机语音键说"测试一下" → 松开 → 文字上屏
5. 快速连说三句短语音 → 三句均完整上屏
6. 说约 10 秒的长句 → 尾音不丢
7. 说话中途断开 Wi-Fi → 触发键被释放（键盘无残留 Fn 修饰位）
8. 说话中途退出 Mac App → 同上
9. 权限被撤销 → App 明确提示，不静默失败
10. 手机端点击"聚焦" → Mac 焦点切到输入框

---

## 14. 风险登记册

| # | 风险 | 等级 | 缓解 |
| --- | --- | --- | --- |
| R1 | 目标语音 App 不响应合成 Fn | 中 | G1 前置验证；备选 Command / Option 触发键 |
| R2 | 目标语音 App 不枚举虚拟麦克风 | 中高 | G2 前置验证；transport 报 USB；备选：把 ASR 移到 Android 端、Mac 只收文字并用 CGEvent 打字 |
| R3 | 首字 / 尾音丢失 | 中 | Android 预录 + Mac 预卷 + `voiceReady` + 驱动 ring buffer + drain 后再释放触发键 |
| R4 | 触发键修饰位卡住 | 中 | 所有离开 streaming 的路径幂等释放；断连 / 退出 / 权限撤销均触发清理 |
| R5 | Android 后台被杀 | 中 | 前台服务 + 电池优化白名单引导 + 断线重连；国产 ROM 需引导加自启动白名单 |
| R6 | 驱动安装需要管理员密码 | 低 | Installer PKG；卸载包同样提供 |
| R7 | libASPL 的 API 与文档版本不匹配 | 低 | 锁定 libASPL commit 版本；C++ 层接口极小 |
| R8 | 唤醒目标 App 造成系统 Fn 快捷键误触发 | 低 | 通用引导中提示用户在系统设置关闭对应的 Fn 功能；或改用 Command / Option 触发键 |
| R9 | 音频通道被第三方占用 | 低 | Android 端明确报错 |
| R10 | macOS 休眠后虚拟麦克风失效 | 低 | 会话开始前重新枚举；未就绪则拒绝开始 |

---

## 15. 参考资料

| 主题 | 位置 |
| --- | --- |
| 参考实现（SayAll，GPL-3.0，仅作行为参考） | `../remote-mic-app` |
| 合成 Fn 探针 | `../remote-mic-app/Testing/VoiceKeyFnPanelProbe.swift` |
| 手机语音键无响应的历史故障 | `../remote-mic-app/Bugs/2026-08-03-ios-phone-voice-button-no-response.md` |
| 语音键模式测试手册 | `../remote-mic-app/Testing/VoiceKeyModes.md` |
| CGEvent 注入实现 | `../remote-mic-app/Sources/RemoteMic/KeyboardInjector.swift:1878` |
| 快捷键录制实现 | `../remote-mic-app/Sources/RemoteMic/ShortcutCaptureMonitor.swift` |
| AX 聚焦实现 | `../remote-mic-app/Sources/RemoteMic/KeyboardInjector.swift:662` |
| AX 安全边界 | `../remote-mic-app/Sources/RemoteMic/VoiceInputDestinationCoordinator.swift:80` |
| 语义按钮枚举 | `../remote-mic-app/Sources/PublicRemoteCompatibility/Core/RemoteCompatibility.swift:3` |
| 配对批准 | `../remote-mic-app/Sources/RemoteMic/BridgeAppModel.swift:6277` |
| 语音会话点按控制器 | `../remote-mic-app/Sources/RemoteMic/VoiceFnTapSessionController.swift` |
| MIT 许可的虚拟麦克风参考实现 | `sheeshmohsin/noise-ai`（AudioServerPlugIn via libASPL，共享内存 IPC） |
| Apple libASPL | `ExistentialAudio/libASPL` |
| AudioDriverKit 限制说明 | Apple 文档 "Creating an audio device driver" 备注；Apple 开发者论坛 thread/775341（2025-03） |
| 黑盒驱动 USB transport 补丁（思路参考） | `../remote-mic-app/third_party/blackhole/blackhole-device-usb.patch` |
