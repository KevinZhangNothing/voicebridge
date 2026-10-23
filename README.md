# VoiceBridge

用 Android 手机充当 Mac 的语音键盘与麦克风。

Android 手机的麦克风声音会成为 Mac 上一个虚拟麦克风设备（`VoiceBridge Mic`），
手机上的按键可以触发 Mac 的键盘快捷键，并把焦点切到当前 App 的输入框。

Mac 端**不适配任何具体输入法**——识别与上屏由用户自己的语音 App（ChatterFly /
豆包 / 微信输入法等）完成，本项目只负责提供麦克风、按下触发键、执行快捷键。

## 组成

| 目录 | 内容 |
| --- | --- |
| `mac-app/` | macOS App（Swift）：配对、快捷键注入、AX 聚焦、语音会话、虚拟麦写入 |
| `android-app/` | Android App（Kotlin/Compose）：前台录音、遥控 UI、客户端 |
| `driver/` | 虚拟麦克风驱动（C++17 + libASPL，AudioServerPlugIn） |
| `protocol/` | 跨端协议唯一事实源：帧格式、JSON schema、共享内存布局 |
| `docs/superpowers/specs/` | 设计规格 |

## 当前状态

设计阶段。规格见 [`docs/superpowers/specs/2026-10-23-voicebridge-design.md`](docs/superpowers/specs/2026-10-23-voicebridge-design.md)。

开工前必须先完成规格 13.1 的三个验证门（G1 / G2 / G3），其中 G1、G2 可用现成
BlackHole 与改造版 `VoiceKeyFnPanelProbe` 完成，不依赖本仓库代码。

## 平台要求

- macOS 14 (Sonoma) 及以上，Apple Silicon 与 Intel 均可
- Android 8.0 (API 26) 及以上
