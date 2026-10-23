# android-app

Android App（Kotlin + Compose，minSdk 26）。职责见设计规格 §5.4。

计划中的模块：

| 模块 | 职责 |
| --- | --- |
| `VoiceService` | 前台服务 + 常驻通知 + 保活 |
| `AudioCapture` | AudioRecord 16 kHz mono + 预录环形缓冲 |
| `WsClient` | OkHttp WebSocket，控制帧 + 二进制帧 |
| `RemoteUi` | Compose：大语音键 / 聚焦键 / 动作面板 / 状态灯 |
| `Discovery` | NsdManager 局域网发现 + 手动输入 IP 兜底 |

实现计划见 `docs/superpowers/specs/2026-10-23-voicebridge-design.md`。
