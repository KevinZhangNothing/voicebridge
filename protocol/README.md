# protocol

跨端协议唯一事实源。Mac 端、Android 端、驱动端都从这里取常量与布局，
避免三处各写一遍导致漂移。

| 文件 | 内容 |
| --- | --- |
| `frames.md` | 控制帧 JSON schema 与二进制音频帧布局 |
| `shm_layout.h` | 共享内存头部与环形缓冲布局 |
| `constants.kt` / `constants.swift` | 由 `shm_layout.h` 与 `frames.md` 派生 |

协议细节见设计规格 §6（协议规范）与 §5.3（共享内存协议）。

实现计划产出的协议定义必须与 `docs/superpowers/specs/2026-10-23-voicebridge-design.md`
保持一致；任何改动先改规格，再改本目录。
