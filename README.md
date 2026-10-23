# VoiceBridge

用 Android 手机当 Mac 的遥控器，把语音输入法的收音快捷键"搬"到手机上。

按住手机上的语音键 → Mac 注入语音输入法的收音快捷键（如 Fn）→ 输入法用 **Mac 自带
的麦克风**收音、识别、文字上屏。手机上的其他按键可以触发 Mac 的键盘快捷键，并把焦点
切到当前 App 的输入框。

**不局限于语音**：手机上的每个按键都能绑定到 Mac 的**任意快捷键**、系统动作或已安装
的 App；绑定可以在**手机端录入**（选键名 + 修饰键），也可以在 Mac 端录制。键组合用规范
键规格表达（如 `cmd+shift+4`），按物理键位编码，与键盘布局无关。**触发键**（启动 / 停止
语音输入法的那个修饰键）也在手机端选，连同"长按收音 / 点按收音"的投递模式。

Mac 端**不适配任何具体输入法**——识别与上屏由用户自己的语音 App（ChatterFly /
豆包 / 微信输入法等）完成，本项目只负责按下触发键、执行快捷键、切换焦点。

**本项目不采集音频、不做虚拟麦克风。** 早期版本曾设计"手机当麦克风 + 自研 Core Audio
驱动"，因投入产出比不成立已废弃，决策溯源见设计规格 §0.3 / §1.4。

## 组成

| 目录 | 内容 |
| --- | --- |
| `mac-app/` | macOS App（Swift）：配对、快捷键注入、AX 聚焦、语音会话状态机 |
| `android-app/` | Android App（Kotlin/Compose）：遥控 UI、快捷键录入、WS 客户端、局域网发现 |
| `protocol/` | 跨端唯一事实源：帧格式、JSON schema、常量（codegen 生成） |
| `tools/probes/` | M0 前置验证探针（Swift Package）：`vb-g1a-trigger` / `vb-g1b-focus` |
| `docs/superpowers/specs/` | 设计规格 |
| `docs/superpowers/plans/` | 实施计划 |
| `docs/verification/` | 探针实测结论（`.md` 人读 + `.json` 机器可读） |

## 当前状态

设计阶段。规格见 [`docs/superpowers/specs/2026-10-23-voicebridge-design.md`](docs/superpowers/specs/2026-10-23-voicebridge-design.md)（v3.4）。

开工前必须先完成规格 §13.1 的两个验证门（M0）：

- **G1-a**：合成触发键能否唤醒目标语音输入法（ChatterFly），以及它认长按还是点按。
- **G1-b**：AX 聚焦耗时。**测的是目标 App 的输入框**（原生 TextEdit + Chromium/Electron
  各一次），不是 ChatterFly——后者是后台输入法，不参与聚焦。

两个探针的**设计与实施计划**：

- 探针设计：[`docs/superpowers/specs/2026-10-23-voicebridge-m0-probe-design.md`](docs/superpowers/specs/2026-10-23-voicebridge-m0-probe-design.md)（v1.5）
- 实施计划：[`G1-b 聚焦探针`](docs/superpowers/plans/2026-10-23-vb-g1b-focus-probe.md) ·
  [`G1-a 触发键探针`](docs/superpowers/plans/2026-10-23-vb-g1a-trigger-probe.md)

两个探针都**不依赖本仓库的产品代码**（纯 CGEvent 注入 + 窗口观察 + AX 查询），可以独立开跑；
结论写入 `docs/verification/` 后回填上游规格 §4.2 / §5.2 / §7.2 / §13.1 / §13.2。

## 平台要求

- macOS 14 (Sonoma) 及以上，Apple Silicon 与 Intel 均可
- Android 8.0 (API 26) 及以上
