# protocol

跨端协议唯一事实源。Mac 端、Android 端都从这里取常量与帧定义，避免两处各写一遍
导致漂移。

| 文件 | 内容 |
| --- | --- |
| `frames.md` | 控制帧 JSON schema 与错误码表（含 `mappingList` / `mappingCreate` / `mappingSet` 与 `trigger_key_not_configured` 等新码） |
| `keyspec.json` | **规范键名清单**（单一事实源）：规范名 ↔ Android keycode ↔ macOS virtual keycode 的显式对照表；另含触发键枚举 `triggerKey`（`fn` / `cmd_l` / `cmd_r` / `opt_l` / `opt_r`） |
| `constants.kt` / `constants.swift` | 由 `frames.md` **codegen 生成**，不手写 |
| `KeySpecCatalog.kt` / `KeySpecCatalog.swift` | 由 `keyspec.json` **codegen 生成**，不手写 |

**codegen 只产出常量 / 键表 / 错误码**；帧的编解码在两端**手写**（帧类型约 20 种，不引
入序列化框架）。

**必须以 codegen 方式落地**：手抄两份必然漂移。CI 执行生成器并检查 `git diff` 为空，
这就是"跨端协议一致性"的校验机制。

协议细节见设计规格 §6（协议规范）。

实现计划产出的协议定义必须与 `docs/superpowers/specs/2026-10-23-voicebridge-design.md`
保持一致；任何改动先改规格，再改本目录。
