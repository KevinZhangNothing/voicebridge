# android-app

Android App（Kotlin + Compose，minSdk 26）。职责见设计规格 §5.3。

计划中的模块：

| 模块 | 职责 |
| --- | --- |
| `RemoteUi` | Compose：大语音键（按住生效）/ 指令键面板（含自定义槽位）/ 聚焦键 / 状态灯 |
| `ShortcutEditor` | 绑定编辑：选键名 + 修饰键、硬件键盘捕获、拉取 / 写回 Mac 端注册表 |
| `WsClient` | OkHttp WebSocket，控制帧 |
| `Discovery` | NsdManager 局域网发现 + 手动输入 IP 兜底 |
| `Settings` | DataStore：Mac 端地址 |

**快捷键录入在手机上**（规格 §6.5）：主路径是"选键名 + 勾修饰键"，生成规范键规格
（如 `cmd+shift+4`）；接了蓝牙 / USB 键盘时可以直接按组合键捕获。**`fn` 未必捕获得到**
——Android 的按键模型里有 `KEYCODE_FUNCTION`（119）/ `META_FUNCTION_ON`（0x8），但多数
键盘的 Fn 在固件 / HAL 层就被消费，不送到前台应用，且不区分左右；需要时手选。
录入**不需要任何新权限**。

**触发键也在手机端选**（规格 §6.3）：触发键是注册表的保留条目 `__voice`，手机端选
修饰键 + 选投递模式（长按 / 点按），写回 Mac 端注册表即可。

自定义槽位的**编号由 Mac 端分配**：新建走 `mappingCreate{rev}`（不带 id），Mac 在回执里
给出新槽位的 `id`（`customSeq` 单调递增、删除后不复用，**同时存在的槽位 ≤ 16**）。删除
槽位用 `mappingSet` 带 `binding:null`（`none` 只是禁用、保留槽位）。

**没有麦克风、没有前台服务、没有常驻通知**——这是一个纯遥控 App。权限只需
`INTERNET` + Wi-Fi 状态相关权限（见规格 §9.2）。

实现计划见 `docs/superpowers/specs/2026-10-23-voicebridge-design.md`。
