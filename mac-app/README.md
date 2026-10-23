# mac-app

macOS App（Swift，macOS 14+）。职责见设计规格 §5.1。

计划中的模块：

| 模块 | 职责 |
| --- | --- |
| `PairingService` | WebSocket 服务、握手、令牌校验、配对批准、长期信任 |
| `ProtocolCodec` | 控制帧 JSON 编解码 |
| `ShortcutRegistry` | 语义按钮 id → 绑定 的映射表（12 固定 id + `__voice` 触发键 + `custom<N>` 槽位），Mac 端持权威副本，带 `rev` 与 `customSeq` |
| `KeySpecParser` | 规范键规格 → (修饰键 flags, 主键 virtual keycode)；按 ANSI 物理键位编码；主键恰好 1 个 |
| `KeyInjector` | CGEvent 注入，成对 down / up；按连接登记已注入的键 |
| `SystemActionRunner` | 执行 `action` / `app` 类绑定；音量与播放控制走 `NSSystemDefined` 系统键事件（无 virtual keycode） |
| `ShortcutCapture` | event tap 录制任意快捷键（**可选路径**，需 Input Monitoring 权限） |
| `ComposerFocus` | AX 前置宣告 + 遍历找可编辑输入框并设置焦点 |
| `VoiceSession` | 语音会话状态机（准备 / 保持 / 释放） |

工程形态：**Xcode 工程**（产出带 `Info.plist` 的 `.app`），核心逻辑放在可被单测引用的
Swift Package target 中。**菜单栏常驻**（`LSUIElement = true`，不占 Dock）+ 首次启动的
一次性设置窗口——服务端必须一直在跑，否则用户没开 App 时手机按不动。不启用 App
Sandbox，Developer ID 签名 + 公证后用 .dmg 分发。

实现计划见 `docs/superpowers/specs/2026-10-23-voicebridge-design.md`。
