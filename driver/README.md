# driver

虚拟麦克风驱动：C++17 + Apple 开源 libASPL，Core Audio AudioServerPlugIn。

设备名 `VoiceBridge Mic`，transport 报 `kAudioDeviceTransportTypeUSB`。

| 文件 | 职责 |
| --- | --- |
| `VoiceBridgeDriver.cpp` | libASPL 样板 + 设备属性 + I/O 回调 |
| `SharedRingBuffer.h` | SPSC 无锁环形缓冲 |
| `PluginIPC.h` | 共享内存命名、布局、协议头定义 |

共享内存布局定义在 `protocol/shm_layout.h`（跨端唯一事实源）。

设计依据与参考实现见设计规格 §4.1、§5.3。
