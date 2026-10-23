# mac-app

macOS App（Swift，macOS 14+）。职责见设计规格 §5.1。

计划中的模块：

| 模块 | 职责 |
| --- | --- |
| `PairingService` | WebSocket 服务、握手、两位码批准、长期信任 |
| `ProtocolCodec` | 控制帧 JSON 编解码 + 音频帧二进制解包 |
| `ShortcutRegistry` | 语义按钮 id → 快捷键 / 系统动作 的映射表 |
| `KeyInjector` | CGEvent 注入，成对 down / up |
| `ShortcutCapture` | event tap 录制任意快捷键 |
| `ComposerFocus` | AX 遍历找可编辑输入框并设置焦点 |
| `VoiceSession` | 语音会话状态机 |
| `AudioBridge` | 共享内存写入 + 排空 |

实现计划见 `docs/superpowers/specs/2026-10-23-voicebridge-design.md`。
