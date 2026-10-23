# VoiceBridge M0 · G1 验证门探针设计

- **版本**：v1.5
- **日期**：2026-10-23
- **状态**：已评审（v1.5 为第五轮评审修复）
- **上游规格**：`docs/superpowers/specs/2026-10-23-voicebridge-design.md`（v3.4）
- **本文解决**：上游规格 §13.1 的 **G1-a / G1-b** 两道前置验证门，以及 §13.2 的 **M0** 里程碑出口
- **修订记录**：§10

---

## 0. 背景与目标

上游规格把 M0 定义为「执行 G1-a / G1-b 前置验证」，出口标准是「结论写入 `docs/`」。规格同时写明：**「G1-a / G1-b 在没有自研代码时即可执行（纯 CGEvent 注入 + 窗口观察 + AX 查询），应作为独立的第一步。」**

因此本文的设计对象**不是 VoiceBridge 产品代码**，而是两个一次性的黑盒探针：

| 门 | 要回答的问题 | 上游规格位置 |
| --- | --- | --- |
| **G1-a** | 合成的触发键能否唤醒目标语音输入法（ChatterFly）？它认**长按**还是**点按**？ | §13.1 |
| **G1-b** | **AX 聚焦耗时**：快路径在原生 / Chromium / Electron 上值不值得开自动聚焦？ | §13.1、§5.2、§14 R4 |

探针结论回填上游规格，替换当前留在规格里的「由 G1-b 给出依据」「G1-a 实测决定」等占位表述。

**为什么必须先过这两道门**（上游规格 §14 原文口径）：

- **R1（高）**——「整个产品的价值都压在『合成 Fn 能唤醒 ChatterFly』这一件事上」。R1 是头号风险。
- **R4（中）**——「聚焦耗时导致丢字」。若快路径在 Chromium/Electron 系无效，规格 §5.2 的「自动聚焦快速失败」规则不成立，应把自动聚焦整体降级为默认关闭的独立动作。

---

## 1. 现场勘查结论（已核实）

### 1.0 事实索引

| 小节 | 编号 | 内容 |
| --- | --- | --- |
| §1.1 | F-01 … F-15, **F-40** | Chatterfly 本体：安装位置、bundleId、快捷键管理器、计数器、配置可读性 |
| §1.2 | — | 参考探针（`../remote-mic-app`）的可复用 / 不可复用对照 |
| §1.3 | F-16 … F-24 | 判据可用性（本机实测）：音频输入属性、窗口、日志、权限 |
| §1.4 | F-25 … F-38, **F-41 … F-43** | 环境与 SDK：工具链、AX 角色与超时语义、Fn 相关常量、事件标记、App 启动 API |
| §1.5 | **F-39** | 上游规格 §5.2 的安全边界与全路径算法（约束探针行为） |

> 编号**按首次出现顺序递增**，但因多轮评审的增补，编号与出现顺序**不单调**（如 F-40 在 §1.1、F-39 在 §1.5）。**引用一律写 `F-xx`，不要按位置推断。**

### 1.1 目标输入法（ChatterFly）

| # | 事实 | 依据 |
| --- | --- | --- |
| F-01 | 安装位置是 **`/Library/Input Methods/Chatterfly.app`**（系统级），**不是** `~/Library/Input Methods/` | `mdfind -name Chatterfly` 命中 `/Library/Input Methods/Chatterfly.app`；`ls /Library/Input Methods/` 见该 bundle；`ls ~/Library/Input Methods/` 仅含 `.localized` |
| F-02 | bundleId = `com.tencent.inputmethod.chatterfly` | `plutil -p /Library/Input Methods/Chatterfly.app/Contents/Info.plist` → `CFBundleIdentifier` |
| F-03 | 版本 `1.0.8.13459`（build 13459） | 同上，`CFBundleShortVersionString` / `CFBundleVersion` |
| F-04 | `LSBackgroundOnly = true`；InputMethodKit 服务类 `YuanbaoInputController` | 同上，`LSBackgroundOnly` / `InputMethodServerControllerClass` |
| F-05 | **ChatterFly 是当前选中的输入法** | `defaults read com.apple.HIToolbox AppleSelectedInputSources` 含 `com.tencent.inputmethod.chatterfly` / `.pinyin`；TIS 枚举亦含 `com.tencent.inputmethod.chatterfly.pinyin`（`selectCapable = true`、`enabled = true`） |
| F-06 | 主进程常驻（本次勘查 pid 931），另有 `ChatterflyServices`(966)、`ChatterflyTaskManager`(1373) | `ps aux` |
| F-07 | 用户偏好含 `keyVoiceEnableGlobalVoice=1`、`keyBottomVoiceBar=0`、`keyShowInDynamicIsland=1` | `plutil -p ~/Library/Application Support/Chatterfly/InputMethod/ChatterflyPY/UserPreferences.plist` |
| F-08 | 语音快捷键由 `YBSpeechShortcutManager` 管理，功能槽为 `voice` 与 `voiceCommand`，每项含 `keys`（键数组）与 `type`（长按/点按） | `strings` 主二进制：`applyShortcut: func=%ld keys=%@ type=%ld`、`registered: func=%ld id=%@ keys=%@ type=%ld` |
| F-09 | 存在**双击 Fn** 触发路径 | 同上：`handleDoubleFnTriggered`、`registerDoubleFnShadow`、`showDoubleFnFeatureHint` |
| F-10 | 结束语音区分 `longPress` 与 `tap` 两种语义 | 同上：`SGSpeechBarEndVoiceHintLastShownTime_longPress` / `_tap`；且 `defaults read com.tencent.inputmethod.chatterfly` 确有这两个键 |
| F-11 | 存在**长按阈值**与**双击冲突窗**两个时长参数，值读不到 | 同上：`timing applied: longPress=%.3fs conflictWindow=%.3fs`（格式串存在，实际值未落盘） |
| F-12 | **语音快捷键的「绑定」读不到**——即读不出「当前配的是哪个键」。但**「结果」读得到**：IM 自报的计数器可读 | 计数器证据：`YBSpeechShortcutStartCount_voice=14`、`YBSpeechShortcutLongPressEndCount_voice=2`、`YBSpeechShortcutNeverUsedTap_voice=0`、`SGSpeechBarEndVoiceHintShownCount_tap=3` / `_longPress=1`。绑定证据缺失：`voiceShortcut` 字符串不在任何可读 plist；MMKV `roma.v2.control.kv` 为二进制内容；四个相关 `defaults` 域均无绑定键。**结论：只能读结果、读不到绑定 → 只能黑盒扫描** |
| F-13 | Chatterfly 自带语音 IPC 客户端，含 `querySpeechShortcutsWithCompletion:` / `setSpeechShortcutForFunction:keyNames:type:` | `strings` `ChatterflyPreference` 二进制。**属私有 IPC，本文方案不采用**（见 §3.1） |
| F-14 | `NSMicrophoneUsageDescription` 存在 | `Info.plist` |
| F-15 | **有「长按结束」与「点按结束」两套结束语义同时存在的痕迹**：14 次启动、2 次 longPress 结束、tap 提示出现 3 次 | 同 F-12 的计数器证据。**推论（待 G1-a 实测确认）：两种停止语义可能都成立**，产品要选的是「注哪种更稳」，而不是「IM 只认哪种」 |
| **F-40** | **S5 计数器只覆盖 `voice` 槽**：plist 里有 `YBSpeechShortcutStartCount_voice`，**没有** `YBSpeechShortcutStartCount_voiceCommand`（只有 `NeverUsedTap_voiceCommand`） | `defaults read com.tencent.inputmethod.chatterfly` 全量键清单。**推论**：若用户的绑定落在 `voiceCommand` 槽，S5 单独用会漏 → 必须同时读两个 key 取最大增量 |

### 1.2 参考探针（`../remote-mic-app`）

参考仓库为 **GPL-3.0**。本探针为**独立实现**，仅对齐可观察行为与所调用的系统 API，**不复制其代码**。

| 参考探针要素 | 处置 |
| --- | --- |
| `CGEventSource(stateID: .hidSystemState)` + `CGEvent(keyboardEventSource:virtualKey:keyDown:)` + `event.flags = [.maskSecondaryFn]` + `event.post(tap: .cghidEventTap)` | **行为对齐**：注入方式与产品 `KeyInjector` 同构，独立实现 |
| `CGEventSource.flagsState(.combinedSessionState).contains(.maskSecondaryFn)` | 行为对齐：用于清理校验（§6.2） |
| `postFnAsFlagsChanged`（flagsChanged 形态） | 行为对齐：本文用作事件形态 B（§4.1） |
| 目标进程匹配（bundleId 含 `doubao`） | **不可复用**：改为 `com.tencent.inputmethod.chatterfly` |
| 语音面板判据（进程 `layer == 3` 的 124×32 小窗） | **不可复用**：Chatterfly 无 layer==3 窗口（见 F-20） |
| 判读规则（按下出现、松开消失 = 长按；松开仍在、再按下消失 = 点按） | 行为对齐：本文 §4.3 改用「按命中手势回放 + 再按一次消歧」的协议 |
| 250ms 采样循环 | 行为对齐：本文收紧到 50ms（§4.2） |

### 1.3 判据可用性（本机实测）

| # | 事实 | 依据 |
| --- | --- | --- |
| F-16 | `kAudioProcessPropertyIsRunningInput` 对 Chatterfly 主进程**空闲时为 0**，且该属性**可读**（`noErr`） | 本机探针实测 |
| F-17 | **「收音时为真」尚未验证**——F-16 只验证了「空闲时为假」 | 因此必须设**正对照**（§4.3 阶段 -1），否则「全部未命中」无法区分「唤不醒」与「判据器坏了」 |
| F-18 | Chatterfly 空闲时**已在** CoreAudio 音频进程列表内（pid 931）→ 「进程进列表」**不可**用作起始判据 | 本机探针实测 |
| F-19 | 统一日志中**无**语音快捷键日志：`log show --last 7d --predicate 'process == "Chatterfly"'` 匹配 `ShortcutManager\|SpeechBar\|YBSpeech\|SGSpeech` = **0 条**（IM 的这些日志是 NSLog，不进统一日志） | 本机实测 |
| F-20 | Chatterfly 拥有 16 个窗口；**无 layer==3 窗口**。但**窗口集合随 IM 状态动态变化**：一次快照里看到 layer 2147483629 的 64×506 与 23×23（alpha=0.0），另一次快照里同 layer 变成 57×38 / 0×0，屏上最大的是 layer 5002 的 540×390。**结论：窗口只能按 owner+layer 全量记录供事后比对，不能作为静态分类判据** | `CGWindowListCopyWindowInfo(.optionAll)` 两次枚举 + 独立复核 |
| F-21 | 全局 layer 3 上有若干窗口，但**均不属于** Chatterfly | 同上 |
| F-22 | **AX 权限已授予**当前进程（`AXIsProcessTrusted()` = true） | 本机探针实测 |
| F-23 | `kAudioHardwarePropertyProcessObjectList` 可用（status 0） | 本机探针实测 |
| F-24 | 窗口枚举中 **`kCGWindowName` 全部为空串**，而 `kCGWindowOwnerName` / `kCGWindowBounds` / `kCGWindowAlpha` **齐全**——与「未授予屏幕录制权限」的表现一致。探针只依赖 owner/bounds/alpha，**故不需要屏幕录制权限** | 本机探针枚举输出（16 个窗口 name 皆空） |

### 1.4 环境与依赖

| # | 事实 | 依据 |
| --- | --- | --- |
| F-25 | 工具链：Swift 6.4 / clang 21 / `arm64-apple-macosx27.0.0`；`xcode-select -p` = `/Applications/Xcode.app/Contents/Developer` | `swiftc --version`、`clang --version` |
| F-26 | 实测系统为 **macOS 27.0.1（26A434）**；本文探针的目标下限写作 `macOS 14+` —— **14+ 未在本机验证**，仅依据 API 可用性 | `sw_vers` |
| F-27 | 目标 App 存在且 bundleId 确认：TextEdit `com.apple.TextEdit`、Google Chrome `com.google.Chrome`、Slack `com.tinyspeck.slackmacgap` | `defaults read <app>/Contents/Info CFBundleIdentifier` |
| F-28 | AX 角色常量在 `HIServices.framework/Headers/AXRoleConstants.h`：`kAXTextAreaRole`、`kAXTextFieldRole`、`kAXComboBoxRole`；**`AXSecureTextField` 是 subrole 而非独立 role** | 头文件 grep |
| F-29 | `AXUIElementSetMessagingTimeout` 语义（`AXUIElement.h:389-392` 逐字）：<br>「Pass the system-wide accessibility object ... if you want to set the timeout **globally for this process**.」<br>「Setting the timeout on another accessibility object sets it **only for that object**, not for other accessibility objects that are equal to it.」<br>「Setting `timeoutInSeconds` to **0** for the system-wide accessibility object **resets the global timeout to its default value**. Setting `timeoutInSeconds` to 0 for any other accessibility object makes that element use the current global timeout value.」<br>**推论：设 systemWide 即本进程全局超时，对本进程内所有 AX 调用生效；对其他 element 单独设置只作用于该对象本身。** | 头文件逐字核对 |
| F-30 | `kAXManualAccessibility` **没有 SDK 常量**（HIServices 头文件全文无命中），须用字面串 `"AXManualAccessibility"` | 头文件 grep |
| F-31 | `ev_keymap.h` 的 `NX_KEYTYPE_*` 共 **22 个唯一常量**（`grep -c` 会数成 33——那包含注释与用法行，**不要用**）：`BRIGHTNESS_UP/DOWN`、`CAPS_LOCK`、`CONTRAST_UP/DOWN`、`EJECT`、`FAST`、`HELP`、`ILLUMINATION_UP/DOWN/TOGGLE`、`LAUNCH_PANEL`、`MENU`、`MUTE`、`NEXT`、`NUM_LOCK`、`PLAY`、`PREVIOUS`、`REWIND`、`SOUND_UP/DOWN`、`VIDMIRROR`。**没有 Fn / FUNCTION / DICTATION** | `grep -oE NX_KEYTYPE_… \| sort -u \| wc -l` = 22 |
| F-32 | **SDK 里唯一的 Fn 常量**在**修饰键通道**而非 `NX_KEYTYPE_*`：`ev_keymap.h:113` → `NX_MODIFIERKEY_SECONDARYFN = 7`（同族还有 `SHIFT=1`、`CONTROL=2`、`ALTERNATE=3`、`COMMAND=4`） | 头文件 grep |
| F-33 | **Fn 相关的 HID 事件 subtype**：`IOLLEvent.h:186` → `NX_SUBTYPE_AUX_CONTROL_BUTTONS = 8`；`IOLLEvent.h:213` → `NX_SUBTYPE_STICKYKEYS_FN_DOWN = 114` | 头文件 grep |
| F-34 | `com.apple.HIToolbox` 下**不存在** `AppleFnUsageType` 键 → 无法读取「按 Fn 键来」的系统配置 | `defaults read com.apple.HIToolbox AppleFnUsageType` → `Could not find key` |
| F-35 | **TIS 可用于输入源的读取与切换**：`TISCopyCurrentKeyboardInputSource()`（`TextInputSources.h:799`）、`TISCopyCurrentASCIICapableKeyboardInputSource()`（:840）、`TISSelectInputSource()`（:952）、`TISCreateInputSourceList()` | 头文件 grep |
| F-36 | **TIS 实时枚举得 333 个输入源**。其中 `enabled = true` **且** `selectCapable = true` 的有 **8 个**：`com.apple.keylayout.ABC`、`com.apple.inputmethod.SCIM.ITABC`、`com.apple.CharacterPaletteIM`、`com.apple.PressAndHold`、**`com.tencent.inputmethod.chatterfly.pinyin`**、`com.apple.inputmethod.AinuIM.Ainu`、`com.apple.inputmethod.Kotoeri.KanaTyping.Japanese`、`com.apple.inputmethod.Kotoeri.RomajiTyping.Japanese`。注意 **`com.apple.inputmethod.SCIM` 与 `com.tencent.inputmethod.chatterfly` 本身是 `selectCapable = false` 的容器**（不可直接选用）。另：`AppleEnabledInputSources` 域里**不含** Chatterfly，而 `AppleSelectedInputSources` 域里含——**两个 defaults 域都不是权威枚举，应以 TIS 为准** | 本机 TIS 枚举实测（对**全部** 333 个源统计，`CFEqual(v, kCFBooleanTrue)` 判定） |
| F-37 | 上游规格 **§7.1 时序**（行 987-1026）含 `tapDuration = 120ms` 的**时序参数表**；**§7.2 触发键的成对性**（行 1027-1039）含**投递表**（`down → 120ms → up` / 停止拍 `down → 立即 up`）。两处都写 120ms | 上游规格行号核对 |
| F-38 | **按键状态常量**（`IOLLEvent.h:106-108`）：`NX_KEYDOWN = 10 (0x0A)`、`NX_KEYUP = 11 (0x0B)`、`NX_FLAGSCHANGED = 12`。**状态是 8 位值，必须左移 8 位放进 `data1` 的高半段** | 头文件逐字核对 |
| **F-41** | **事件来源标记**：`kCGEventSourceUserData = 42`（`CGEventTypes.h:356`）→ 可用 `CGEventSetIntegerValueField(event, .eventSourceUserData, <magic>)` 标记**自家注入的事件**，以便下游（物理输入监听、上游产品 `ShortcutCapture`）识别并忽略 | 头文件 grep |
| **F-42** | **可拿到 App 实例的启动 API**：`NSWorkspace.openApplicationAtURL:configuration:completionHandler:`（`NSWorkspace.h:46`，macOS 10.15+）直接返回 `NSRunningApplication`。**比 `open -n` 好**：能拿到该实例的 pid，便于校验 frontmost 与结束时关闭 | 头文件 grep |
| **F-43** | **用户按住修饰键产生的是 `flagsChanged` 而非 `keyDown`**（`NX_FLAGSCHANGED = 12`）→ 物理输入监听若只订阅 `.keyDown` 会漏检 | `IOLLEvent.h:108` |

### 1.5 上游规格 §5.2 的约束（F-39）

| # | 事实 | 依据 |
| --- | --- | --- |
| **F-39** | **上游规格 §5.2 的安全边界**（原文）：允许 role 为 `AXTextArea` / `AXTextField` / `AXComboBox`；拒绝 `AXSecureTextField`（**role 与 subrole 双重判断**）；并拒绝**语义关键词**——<br>`password, passcode, secret, api key, apikey, token, credit card, search, find, filter, address bar, settings, preferences, command palette, 密码, 口令, 密钥, 令牌, 信用卡, 搜索, 查找, 筛选, 设置, 偏好`<br>同节并明写：「**Chromium / Electron 系的 Web 内容壳在辅助客户端宣告自己之前，辅助功能树是空的**——此时 AX 遍历只会得到一个空壳，找不到任何可编辑元素」<br>同节并警告：若 `AXManualAccessibility` 返回 `attributeUnsupported` 而回退 `AXEnhancedUserInterface`，**部分 Electron 版本下它会扭曲窗口与动画**，所以只作回退 | 上游规格 §5.2 原文逐字 |

---

## 2. 范围

### 2.1 做

1. 两个黑盒探针，可重复运行，产出机器可读 + 人读两份结论。
2. 回填上游规格，消除「由 G1-a / G1-b 实测决定」类占位表述。

### 2.2 不做（明确排除）

| 不做 | 理由 |
| --- | --- |
| 实现任何 VoiceBridge 产品代码（协议 / 注册表 / 注入器 / UI） | 属 M1 / M2；M0 只出结论 |
| 逆向 Chatterfly 私有 IPC（`SGSpeechIPCClient`） | 成本高、协议不公开 |
| 重启 / kill Chatterfly 进程 | 「重启 IM 捕获 NSLog」方案已评估并排除（见 §3.1） |
| 修改 Chatterfly 任何配置或偏好 | 只读不改 |
| 测试语音**识别准确率** | G1-a 只问「有没有开始收音」 |
| 建 CI 基建 | 留到 M1 与 protocol codegen 一起做；本文保证构建/测试是**单条命令** |
| 写 VoiceBridge 的协议常量 / codegen | 属 M1（上游规格 §12.3） |

### 2.3 做（补）：人工抽检

命中组合**必须**做一次端到端人工抽检：对命中组合按一次，说一句话，看文字是否上屏。

- **这不是识别率测试**，是**判据器的端到端验收**——证明「判据器说收音了」与「确实在收音」是同一件事。
- 结果记入结论文档；若不通过，判据器判据需修正后重跑。

---

## 3. 方案选型

### 3.1 G1-a 的三种取证方案

| 方案 | 做法 | 优点 | 缺点 | 结论 |
| --- | --- | --- | --- | --- |
| **A** | 黑盒矩阵扫描 + 多路判据交叉验证 | 全自动、零逆向、结论是纯行为证据 | 矩阵较大；合成 Fn 可能误触系统功能 | **采用**（含「人工声明」作为快速预览，见 §4.0） |
| B | A + 重启 Chatterfly 捕获其 NSLog | 判据从「外部推断」升级为「IM 自报」 | 要重启用户正在用的输入法 | 排除（用户已确认） |
| C | 先读 Chatterfly 配置再定向验证 | 最快 | **读不到绑定**（F-12）：计数器只给「结果」不给「键位」，私有 IPC 不可用 | 排除 |

> **对评审意见的澄清**：计数器（`YBSpeechShortcutStartCount_voice` 等）让「结果」可读、可自动核对，因此它被用作**正对照与交叉验证**（§4.2 S5、§4.3 阶段 -1）；但它**不能**告诉探针「当前绑定的是哪个键」，所以方案 C 仍然排除。排除 C 的理由是**读不到绑定**，不是「只能人工/截图」。

### 3.2 为什么 A 足够

- G1-a 的问题本身就是行为问题（「合成键能否唤醒」），黑盒行为证据即为所需证据。
- 判据采用「主判据 + IM 自报计数器」双路，配合**正对照**（§4.3 阶段 -1）确保判据器本身可信。
- 窗口判据因窗口集合动态（F-20）降级为「只记录、事后比对」，不参与判定。

---

## 4. G1-a 探针设计

### 4.0 流程总览：快速预览 → 完整矩阵（**以矩阵为准**）

**用户本人知道自己的快捷键。** 因此 G1-a 分两段，**顺序固定**：

| 段 | 阶段 | 做法 | 耗时 | 地位 |
| --- | --- | --- | --- | --- |
| **快速预览** | 阶段 0.5 | 询问用户「语音快捷键是什么（**键 + 模式**）」→ 正对照 + **定向验证该声明** | ~1 分钟 | 矩阵的**前置子集**，**不是替代品** |
| **完整矩阵** | 阶段 1 | 50 组扫描 → 发现备选方案 | ~8–10 分钟 | **M0 必跑；最终结论以此为准** |

**M0 的收口定义**：**必须产出完整矩阵的结论**。快速预览只是它的第一步，且**两段结论都写进报告**。

> **为什么不能只跑预览**：上游 §13.1 的 M0 出口要的是「目标输入法的收音快捷键、长按/点按模式」这个**完整答案**，而不是「用户说的那个键能不能用」。

**声明的「模式」取值**：`longPress`（长按）/ `tap`（点按）/ `doubleTap`（双击）。三者都必须能被声明——F-09 显示 Chatterfly 存在**双击 Fn** 路径，只给「长按/点按」两档无法表达。

**`--declared-key` 与 `--declared-mode` 必须成对给出**：只给其一 → 启动即报错退出（不允许「只声明键」导致模式默认值被误用）。

### 4.1 候选矩阵

| 维度 | 取值 |
| --- | --- |
| **键（5）** | `fn`(63) / `cmdLeft`(55) / `cmdRight`(54) / `optLeft`(58) / `optRight`(61) |
| **事件形态（2 主 + 1 可选）** | `keyEvent`：`CGEvent(keyboardEventSource:virtualKey:keyDown:)` + 修饰 flags，post 到 `.cghidEventTap`（参考探针形态 A）<br>`flagsChanged`：`CGEvent` type = `.flagsChanged`，写 `.keyboardEventKeycode` + 修饰 flags（参考探针形态 B）<br>`systemDefined`：**可选 spike**，见下 |
| **手势（5）** | `tap120`：down → **120ms** → up<br>`hold1500`：down → 保持 1500ms → up<br>`hold3000`：down → 保持 3000ms → up<br>`double120`：两击，间隔 **120ms**<br>`double300`：两击，间隔 **300ms** |

**主矩阵规模：5 键 × 2 形态 × 5 手势 = 50 组。**

**手势参数定义（消除实现分叉）**

| 参数 | 值 | 定义 |
| --- | --- | --- |
| `tapDurationMs` | **120ms** | 单次按压的驻留时长（down → 120ms → up）。与上游规格 §7.1 的 `tapDuration = 120ms` **时序参数表**一致 |
| `doubleClickHoldMs` | **50ms** | 双击中**每一次**单击的驻留时长（down → 50ms → up）。取 50ms 是为了短于任何长按阈值 |
| `doubleClickGapMs` | **120ms / 300ms** | 两次单击之间的**间隔，定义为 up → down**（与系统双击惯例一致：间隔在两次点击之间测量，不是 down→down） |

**`tap120` 与产品实现必须用同一个常量来源**：上游规格 §7.1 的时序参数表写明 `tapDuration = 120ms`，并注「必须短于输入法判定『长按』的下限，否则会被当成按住不放」；§7.2 的投递表写明点按模式的开始拍是「down → 120ms → up」、停止拍是「down → 立即 up」。

产品实现要到 M2 才存在，因此 M0 阶段的落地方式是：探针内 `tapDurationMs` 是**唯一定义点**，M0 结论文档记录该值，M2 实现时对齐上游规格 §7.1 / §7.2。单测断言见 §8.1（**注意 `doubleClickGapMs = 120ms` 是另一处 120 字面量**，故断言范围必须区分「命名常量」与「无名字面量」）。

**`systemDefined` 降级为可选 spike**：`NX_KEYTYPE_*` 的 22 个常量里**没有 Fn / FUNCTION / DICTATION**（F-31）。但 SDK 里**有别的 Fn 依据**：

- `NX_MODIFIERKEY_SECONDARYFN = 7`（`ev_keymap.h:113`，F-32）——修饰键通道
- `NX_SUBTYPE_AUX_CONTROL_BUTTONS = 8`（`IOLLEvent.h:186`）、`NX_SUBTYPE_STICKYKEYS_FN_DOWN = 114`（`IOLLEvent.h:213`，F-33）
- `NX_KEYDOWN = 10 (0x0A)` / `NX_KEYUP = 11 (0x0B)`（`IOLLEvent.h:106-107`，F-38）——**按键状态是 8 位值**

因此 spike 的**首选构造路径**是修饰键通道：

```
data1 = (NX_MODIFIERKEY_SECONDARYFN << 16) | (NX_KEYDOWN << 8)   // 按下
data1 = (NX_MODIFIERKEY_SECONDARYFN << 16) | (NX_KEYUP   << 8)   // 松开
```

> ⚠️ **状态值是 `0x0A` / `0x0B`，不是 `0xA00` / `0xB00`**——后者大了 0x100 倍，会直接构造出垃圾事件，白白浪费一次 spike。

**它属于 modifier 通道，能否被 IM 的快捷键识别器接受，正是 spike 要回答的问题**——盲猜 `NX_KEYTYPE` 不靠谱。

spike 流程：
1. 判据：事件投递后 `CGEventSource.flagsState(.combinedSessionState)` 的 `.maskSecondaryFn` 是否置位
2. 构造成功 → 追加 **5 组**（仅 Fn × 5 手势），主矩阵变 55 组
3. 构造失败 → 该形态**从矩阵中移除**，记为 `unsupportedBySDK`（不是 `notApplicable`），并在结论文档说明

**`hold1500` 与 `hold3000` 都保留**：用于定位长按阈值落在哪一档。**「按住」轴一律取自命中手势自己的 `gDuring`**（§4.3 阶段 2），`hold1500` 只作为长按基线，**不参与模式归类**——否则 `hold3000` 命中（阈值 >1.5s）会被错归为「双击即启即停」。

**候选顺序**：`cmdLeft` / `cmdRight` / `optLeft` / `optRight` **先于** `fn`。理由：合成 Fn 可能误触系统级功能（上游规格 R2），且本机 enabled+selectable 的输入源有 8 个、含 ABC / SCIM ITABC / CharacterPalette / PressAndHold / Ainu / Kotoeri×2 / Chatterfly（F-36），Fn 默认行为很可能是切换输入源。

### 4.2 判据器

每 **50ms** 采样一次。

| 判据 | 采集方式 | 角色 |
| --- | --- | --- |
| **S1（主）** | 枚举 CoreAudio 音频进程列表，对 bundleId 含 `chatterfly` / `tencent` 的进程读 `kAudioProcessPropertyIsRunningInput` | 任一为 1 → 收音中。**空闲基线已实测为 0**（F-16）；**「收音时为真」由阶段 -1 正对照确认**（F-17） |
| **S4（辅）** | 出现「基线中不存在的、bundleId 含 `tencent` 的新音频进程」 | 也判为收音中，覆盖 ASR 跑在临时子进程的情况（F-18） |
| **S5（独立佐证）** | 进程内 `CFPreferencesCopyAppValue` 读 `YBSpeechShortcutStartCount_voice` **与** `YBSpeechShortcutStartCount_voiceCommand`（后者可能不存在，F-40），取**两者的最大增量**，并记录**命中哪个槽** | IM 自己写入的权威信号，用于交叉验证 S1。**不参与 `voiceState` 合成**，只作佐证 |
| **S2 / S3（只记录）** | `CGWindowListCopyWindowInfo` 里 Chatterfly 拥有的全部窗口：owner + layer + bounds + alpha | **不参与判定**，只进报告供事后人工比对（F-20） |

**合成规则**：`voiceState = S1 || S4`。S5 独立佐证。

**S5 必须覆盖两个槽**：若用户的绑定落在 `voiceCommand` 槽，只读 `voice` 槽的计数器会**永远不动**，正对照会误判失败（F-40）。因此 S5 取两个 key 的**最大增量**，并在报告里记 `s5HitSlot`（`voice` / `voiceCommand` / `none`）。

**S5 的采集方式**：**必须用进程内 `CFPreferencesCopyAppValue`**，不得用 `defaults read`——后者每 50ms 会 spawn 一个进程，污染采样与延迟测量。但要注意 **cfprefsd 缓存可能延迟落盘**，所以 S5 的可用性由阶段 -1 实测决定：

- **S5 的落盘延迟由阶段 -1 正对照实测**（见 §4.3 阶段 -1），**S5 是否可逐组归因只有一条判据**（v1.5 统一）：

> **S5 可逐组归因 = 正对照中 S5 通过（增量 ≥ 1） 且 落盘延迟 ≤ 1s**
>
> 两者缺一 → S5 **只能**用于阶段 -1 与整体佐证，**不逐组归因**——这一点必须写进报告，不能假装它可用。

- 落盘延迟的测量方式：从用户触发时刻起轮询计数器，直到增量出现，记录耗时

**去抖与短脉冲**：50ms 采样 + 「连续 2 次一致才翻转」会吞掉短脉冲（`tap120` 的收音窗口可能只有 ~120ms）。因此：

1. 报告**必须保留原始采样序列**（每条带时间戳与各判据的原始值）。
2. 去抖后的翻转是**主判定**。
3. 任何**单样本为真**（去抖前）→ 记为「**可疑命中，需复测**」，不直接判未命中；该组合在阶段 2 复测一次。

**延迟计算**：
- `startLatencyMs` = 状态由 0 变 1 的采样时刻 − 手势开始（keydown 投递）时刻
- `stopLatencyMs` = 状态由 1 变 0 的采样时刻 − 释放（keyup 投递）时刻 —— **上游规格 §7.1 的产品停止路径同样关心这个数**

### 4.3 流程

#### 阶段 -1 · 正对照（**必做，不通过则中止且不出结论**）

目的：证明「判据器能看见收音」。没有这一步，「全部未命中」无法区分「唤不醒」与「判据器坏了」。

**交互**：本阶段**等待用户操作**，最长 **60 秒**。探针打印提示并持续采样：

> 「请现在**用真实快捷键**触发一次 Chatterfly 的语音输入（**不要用菜单栏入口**），说一句话，然后按 Esc 或再次按快捷键结束。全程最长等待 60 秒。」

**为什么必须用快捷键**：菜单栏入口走的是 `SGSpeechIPCClient.startSpeech`，**不经过快捷键识别器**，S5 的启动计数（快捷键启动计数）可能不动——用它触发会让正对照误判失败。快捷键入口才能同时检验两条判据。

**判定**：

| 条件 | 结论 |
| --- | --- |
| S1 观测到 0→1 **且** 1→0 两次翻转，**且** S5 增量 ≥ 1 | 正对照**完全通过**；S1 可用于逐组归因；**S5 是否可逐组归因，还要看落盘延迟**（见 §4.2 的唯一判据） |
| **S1 与 S5 至少一路通过** | 正对照**通过**；**未通过的那一路**记为「本次运行不可用于归因」，**通过的那一路仍照常使用** |
| 两路都不通过 | **中止，不产出任何结论**（报告写明「正对照失败，结论无效」） |
| 60 秒内用户未操作 | **中止**，报告写明「正对照超时未触发」 |

同时测量 **S5 的落盘延迟**：从用户触发起轮询计数器直到增量出现，记录耗时。

> **S5 可逐组归因的判据只有一条**（v1.5 统一，避免两处规则互相打架）：
> **正对照中 S5 通过 且 落盘延迟 ≤ 1s**。
> 「正对照完全通过」**不等于**「S5 可逐组归因」——它还可能是落盘延迟过长导致的（cfprefsd 缓存未刷盘）。

**本阶段可以按键**：正对照需要用户主动操作。「请勿触碰键盘」的提示**从阶段 0 起生效**（见 §4.3 阶段 0 与 §6.1）。

**倒计时的位置**：3 秒倒计时在**正对照之后**（§4.3 阶段 0），因为正对照需要用户主动操作，不能「倒计时后自行接管」。

#### 阶段 0 · 前置

1. 打印醒目横幅，内容含：**「探测期间请勿触碰键盘，约 8–10 分钟」**，并注明「**该提示从此刻起生效**（正对照阶段已结束）」与「输入法可能被切换，探测结束后**尽力恢复**，失败会提示」（§4.1 矩阵规模；快速预览约 1 分钟）
2. **参数校验**：`--declared-key` 与 `--declared-mode` 必须成对；只给其一 → **报错退出**
3. **AX 权限自检**：`AXIsProcessTrusted()` 不通过 → **直接退出**并提示去「系统设置 › 隐私与安全性 › 辅助功能」授权运行探针的终端。授权归属取决于「谁启动探针」，所以每次运行都要查
4. **物理输入监听能力探测**（§6.1）：尝试创建键盘 `CGEventTap`。成功 → 安装「物理输入即中止」监听；失败 → **在横幅与报告中显著标注「本次运行无法中止物理输入」**（监听键盘事件需要 **Input Monitoring** 权限，AX 权限**不覆盖**它）
5. **残留自检**：读 `CGEventSource.flagsState(.combinedSessionState)`——若修饰键异常置位**持续 ≥ 1 秒**才判定为「上次运行残留」（避免用户正按着修饰键时误报），并在报告标注
6. 记录**输入源基线**：`TISCopyCurrentKeyboardInputSource()` 的 `kTISPropertyInputSourceID`（F-35/F-36）
7. 准备 TextEdit 落地靶场：
   - `mktemp -t vb-probe-target` 生成临时文件（**不用固定路径**，避免多会话冲突）
   - 用 **`NSWorkspace.openApplicationAtURL:configuration:completionHandler:`**（F-42）启动 TextEdit 并**拿到该实例的 `NSRunningApplication`**（含 pid）。配置 `createsNewApplicationInstance = true` 以避免与用户已有窗口混淆
   - ⚠️ **不要用 `osascript` 派发 Cmd+N**——那需要 **Automation 权限**，而 `AXIsProcessTrusted()` **不覆盖**它（首次运行会弹「终端想要控制 System Events」）
   - `activate` 该实例，等 0.5s
   - **校验 frontmost 的 pid == 该实例 pid**（§6.1），不匹配 → 立即中止
   - **校验窗口唯一**：用 `CGWindowListCopyWindowInfo` 找该 pid 的窗口号（**不碰 AX**），后续所有注入都以「该窗口在前台」为条件——只校验进程不校验窗口，多文档窗口时仍可能注错地方
   - 用 AX 把焦点设到文档正文（`kAXFocusedAttribute` = true）
8. 采集**基线**样本 0.5s：`voiceState` 应为 false、S1 全 0。若为 true → **中止并报告**（已有语音会话在进行）

**结束时**：关闭步骤 7 启动的 TextEdit 实例（**只关自己开的那一个**，用 pid 校验），否则每跑一次留一个 TextEdit 进程。

#### 阶段 0.5 · 快速预览（**矩阵的前置子集**）

仅当用户给出了 `--declared-key` + `--declared-mode` 时执行：

1. 用声明的键 + 该模式对应的手势，跑**一组**（事件形态用 `keyEvent`）
2. 记录命中与否、模式判定结果
3. **此段结论只是预览**，若与阶段 1 的矩阵结果矛盾 → **以矩阵为准**，并在报告里写明矛盾

> **顺序固定**：阶段 0.5 → 阶段 1（完整矩阵）。**两段都跑**，不允许「预览通过就跳过矩阵」。

#### 阶段 1 · 筛选（50 组 × ~8–12s ≈ 8–10 分钟）

**观察窗 3 秒**（不是 1.5 秒）——`tapOnRelease` 叠加 IM 冷启动可能超过 1.5 秒，观察窗太短会把命中记成未命中。

每组：
1. **校验 frontmost == 目标 TextEdit 实例且目标窗口在前台**，不匹配 → 立即中止（§6.1）
2. 记录输入源（TIS）
3. 采样基线 0.5s（**必须为 false**，否则先执行停止程序，见 §4.3「停止程序」）
4. 执行手势
5. 观察 **3s**，采样
6. 判定为**三态**（v1.5 修订）：**命中**（出现 `voiceState` 0→1 翻转）/ **可疑命中**（去抖前出现过单样本为真，但未构成翻转）/ **未命中**。**可疑命中不得直接判为未命中**——它必须进入阶段 2 复测一次（§4.2）。同时记录 S5 增量与命中槽（若 S5 可逐组归因，见 §4.2）
7. **执行停止程序**（§4.3「停止程序」），确认 `voiceState == false`
8. **再次记录输入源**：与第 2 步不一致 → **立即中止整轮探测并用 TIS 恢复**（F-36：Fn 可能切走输入源，后续所有组都会跑在错误的输入法上）

#### 阶段 2 · 判读模式（**按命中的手势回放**）

> **关键**：阶段 1 的命中可能来自任意手势。判读**必须在同一手势上完成**，否则会与阶段 1 的记录直接矛盾。**不得**固定跑一组手势就下结论。

> **阶段 2 的输入集合 = 阶段 1 的「命中」∪「可疑命中」**（v1.5 修订）。
> 原先只把「命中」送进阶段 2，导致 §4.2 写下的「可疑命中，需复测」**永远不会被复测**——
> 这些组合最终被当作未命中处理，正好违反「不直接判未命中」的规定。

**「按住」轴一律取自命中手势自己的 `gDuring`**。`hold1500` 只作**长按基线**（用来确认长按语义存在），**不参与模式归类**。

> **`gDuring` 的精确定义**（v1.5 补）：`gDuring` = 该手势中**单次按压的最长驻留**期间是否在收音。
> **单次按压驻留 ≥ 200ms 才记录**（否则记 `n/a`）。按 §4.1 的手势参数：
>
> | 手势 | 单次按压驻留 | `gDuring` |
> | --- | --- | --- |
> | `tap120` | 120ms | `n/a` |
> | `double120` / `double300` | **50ms**（`doubleClickHoldMs`） | `n/a` |
> | `hold1500` | 1500ms | true / false |
> | `hold3000` | 3000ms | true / false |
>
> **为什么必须这样定义**：若把双击的「按压窗口」读成**整个手势跨度**
> （`double120` = 50 + 120 + 50 = **220ms ≥ 200ms**），则 `double120` 命中且松开即停会落进决策程序分支 2a
> → 被判成【长按模式】，与专为双击手势准备的【双击即启即停】标签**互相矛盾**。
> 两种读法给出不同的模式结论，所以必须钉死成「单次按压驻留」。

**每一步之间必须清零**：用**停止程序**（下节）把 `voiceState` 压回 false（最多等 5s，超时记 `stateResetFailed`）再进入下一步——否则上一步的残留会污染下一步。

对每个命中手势 **G**：

| 步骤 | 动作 | 记录 |
| --- | --- | --- |
| **A** | `hold1500`（**长按基线**，与 G 无关） | `holdDuring` / `holdAfter`——只用于判断「长按语义是否存在」，不用于归类 |
| **B** | **回放命中手势 G** | `gStarted`（G 是否启动）/ `gDuring`（按上表的「单次按压驻留 ≥ 200ms」规则，否则 `n/a`）/ `gAfter`（松开后是否仍收音） |
| **C** | **当 `gAfter` 为 true 时**执行：再按一次 G → 观察 3s | `secondPressStops`。<br>⚠️ 若 G 是双击手势，「再按一次」应理解为「**再双击一次**」（即完整的双击序列），不是再单击一次 |

**归类决策程序（按序取第一个满足的分支）**

```
1. gStarted == false
   → 阶段 1 假阳性：复测该组合一次；两次都 false 才记「不触发」

2. gAfter == false（松开后已经停了）
   2a. gDuring == true
       → 【长按模式】按住期间收音，松开即停
   2b. gDuring == false 或 n/a
       → 【自启自停类】启动发生在松开之后、且已自行停止
         · 若 G ∈ {double120, double300} → 更具体的标签【双击即启即停】
         · 若 G ∉ {double*} → 标签【松开后自启自停】
         （这类必须在上游 §7.2 投递表单列，不能并入「长按模式」）

3. gAfter == true（松开后还在收音）
   3a. secondPressStops == true
       → 【点按模式】松开不停，再按一次才停
   3b. secondPressStops == false
       → 【tapOnRelease 变体】第二次按也停不了，需要另一种停止动作
   3c. secondPressStops 未测 / 未定论
       → 【未定论】必须复测一次，不得归入任何模式
```

**为什么不能用固定的 `hold1500` 当「按住」轴**：`hold3000` 存在的唯一理由就是「长按阈值可能 >1.5s」。若阈值 = 2s，则 `hold1500` 不启动（`holdDuring=false`）而 `hold3000` 启动后松开即停（`gStarted=true, gAfter=false`）——用 `holdDuring` 归类会**误标成「双击即启即停」**。同理 `tap120` 在阈值很短的长按型 IM 上也会自启自停。

**「两种都支持」分支**：若不同手势分别落在「长按模式」与「点按模式」两个分支，说明**两种停止语义都成立**。此时结论写成「**IM 两种都支持**」，并按 §4.4 的选键规则取一种（这是产品决策，不是 IM 的限制）。

#### 阶段 3 · 人工抽检（v1.5 新增：给 §2.3 / §8.3 第 6 条一个执行落点）

**有命中组合时必须做**。目的是**判据器的端到端验收**——证明「判据器说收音了」与「确实在收音」是同一件事。

```
阶段 3(人工抽检):
  1. 用选中的组合（键 + 手势）**由探针注入一次**
  2. 提示用户：「请对着 Mac 的麦克风说一句话，文字会直接出现在输入框里。
     说完后回终端按回车继续确认。」（观察窗不设上限，等用户确认）
  3. 记录 manualCheck = { passed: 文字是否上屏, note: 用户原话描述 }
  4. 接着执行停止程序，确认 voiceState == false
```

- **用户只说话，不碰键盘**——所以不与「请勿触碰键盘」的横幅冲突。
- **由探针注入而不是用户手动按真实快捷键**：手动按验的是 **IM**，不是**探针的注入路径**，
  与 §2.3 自称的「判据器的端到端验收」不是一回事。
- 结果记入结论文档 §1；**不通过则判据器判据需修正后重跑**（不是「重测识别率」）。

#### 停止程序（**按模式分叉**，阶段 1 / 阶段 2 / 阶段 3 / 退出共用）

> **为什么不能只「补发 up」**：Chatterfly 的点按语义是「再按一次才停」（F-10/F-15），**up 不会停止收音**。若只补发 up，第一组命中（很可能就是点按模式）就会留下一个仍在收音的会话，下一组的基线 `voiceState == true` → **整轮中止**。

```
停止程序(stopProcedure):
  1. 补发 up（幂等；长按模式下这本来就够）
  2. 等 1s
     若 voiceState 仍为 true
       → 补一次「停止拍」：down → **立即 up**（驻留 0ms）
         ⚠️ 停止拍的驻留**不是** `tapDurationMs`——上游规格 §7.2 用驻留时长区分
            「开始拍」(down → 120ms → up) 与「停止拍」(down → 立即 up)。
            停止拍若也驻留 120ms，IM 极可能把它当成一次「开始拍」→ 残留会话 → 整轮中止。
         ⚠️ 只在「仍在收音时」才发 —— 长按模式下多发这一次会短暂开一个新会话
  3. 再等 1.5s
     若 voiceState 仍为 true
       → 记 stopFailed：
         · 阶段 1 / 阶段 2 → 【中止整轮】并在报告与终端显著提示「请在 Mac 上手动停止录音」
         · 退出路径 → 写入报告的「清理状态」栏
```

停止程序在每次发出「停止拍」之后**必须再验证一次**（第 3 步），不能假设它生效。

**长按模式不发停止拍**（v1.5 修订）：长按模式的 `up` **本来就该停**；若 `up` 停不下来说明状态异常，
此时再补一次 `down → 立即 up` 也只会**开一个新会话**，把一个异常变成两个异常。
因此停止程序**按模式分叉**：长按模式下 `up` 后仍为 true → 直接记 `stopFailed`（并中止整轮），
**不发停止拍**；点按 / 未定论模式下才补停止拍。

### 4.4 判读规则与选键规则

**命中判定**

| 观察 | 结论 |
|---|---|
| 存在组合使 `voiceState` 0→1 | 该组合**能唤醒** Chatterfly |
| 全部**可执行组**（50 组；若 `systemDefined` spike 成功则为 55 组）均无翻转（**且阶段 -1 正对照已通过**） | **G1-a 失败**：合成键唤不醒 → 触发上游规格 R1，触发键方案需改（备选 Command / Option） |
| 能唤醒，但仅**双击**形态命中 | 触发键需要支持**双击**投递 → 上游规格 §4.2 的触发键集合要扩展投递模式 |
| 仅 Chatterfly 自带的**双击 Fn 特性路径**命中（与 F-09 的 `handleDoubleFnTriggered` 行为一致） | **单独记一类**：这可能是 Chatterfly 的独立特性路径，**不等于**「触发键必须支持双击投递」。两条结论分开写 |

**选键规则（多个候选同时命中时，产品注哪一个）**

按序执行（v1.5 修订了顺序——见下方「为什么第 3 步要挪到第 4 步前面」）：

1. **非 Fn 优先**——避免系统 Fn 功能误触与输入源切换（F-34/F-36、上游 R2）
2. **单键、无系统级副作用优先**——Cmd/Option 类比 Fn 安全
3. **停止语义定档**（同为可用时）：**优先长按模式**——单次 down/up，没有「停止拍失败」这一失败面（上游规格 §7.2 的 `stop_tap_failed`）；**例外**：长按模式的 `startLatencyMs` 明显更差时（阈值定死：**> 2× 点按模式**）才改选点按模式
4. **同档内 `startLatencyMs` 最低者优先**——直接对应「聚焦耗时窗口内用户说的话不会被录到」的丢字风险（上游 R4）

> **为什么第 3 步要挪到第 4 步前面**（v1.5 修订）：原顺序把「延迟最低者优先」放在「停止语义」之前，
> 于是延迟规则已经选出了唯一赢家，**「长按 > 2× 点按」的例外分支永远不可能被命中**——
> 对选出的赢家而言它就是最低的，2× 关系恒不成立。改成「先定停止语义档，再在档内比延迟」之后，
> 例外分支才可达，§8.1 的「2× 阈值边界用例」才测得到真东西。

> 没有这条规则，结论就无法直接变成上游规格 §7.2 的实现。

### 4.5 探针参数

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `--out <dir>` | `docs/verification/` | 报告输出目录。**路径相对于当前工作目录**（v1.5 补） |
| `--dry-run` | off | 只打印将要执行的矩阵，不注入任何事件 |
| `--sample-ms` | 50 | 采样周期。它同时决定**去抖窗口**（连续 2 个样本 = 2 × 采样周期）与**延迟计算的分辨率**（§4.2）——改它就改这两者 |
| `--only <key>` | 无 | 只跑指定候选（**仅调试用**；M0 结论必须跑完整矩阵） |
| `--declared-key <key>` | 无 | 用户声明的键（`fn` / `cmdLeft` / `cmdRight` / `optLeft` / `optRight`） |
| `--declared-mode <mode>` | 无 | 用户声明的模式：`longPress` / `tap` / `doubleTap`。**必须与 `--declared-key` 成对** |
| `--matrix-only` | off | 跳过快速预览，直接跑完整矩阵（**仅调试用**；M0 必须两段都跑，§4.0） |
| `--max-groups <n>` | 无限制 | 最多跑多少组（**仅调试用**；限制组数会让「全部未命中」的结论失去意义） |
| `--abort-on-voice-stuck` | on | 停止程序失败即中止整轮。**实现说明**（v1.5）：本探针实现为**默认不中止**，用 `--no-abort-on-voice-stuck` 显式关闭——因为中止整轮会丢掉已完成组的数据 |
| `--skip-positive-control` | off | **仅调试用**；跳过阶段 -1 时报告必须标注「无正对照，结论不可信」 |
| **`--now <iso8601>`** | 无 | **仅测试用**：注入报告时间戳（§8.1 的确定性断言依赖它） |
| **`--env <json>`** | 无 | **仅测试用**：注入环境快照（§8.1 的确定性断言依赖它） |

---

## 5. G1-b 探针设计

### 5.1 目标 App

| App | bundleId | 类别 | **测量目标**（必须是产品会用的可编辑目标） |
| --- | --- | --- | --- |
| TextEdit | `com.apple.TextEdit` | 原生 | 文档正文（`AXTextArea`） |
| Google Chrome | `com.google.Chrome` | Chromium | **页面内的 `<textarea>`**（web 内容树内的真实输入框） |
| Slack | `com.tinyspeck.slackmacgap` | Electron | **消息 composer**（消息输入框） |

> **Chrome 与 Slack 分开判定，不合并成「Chromium/Electron 一类」**——两者的快路径表现可能不同（Chrome 是纯 Chromium，Slack 是 Electron），合并判会在「一个快一个慢」时得出自相矛盾的规则。

### 5.2 可编辑目标判定（**与上游规格 §5.2 对齐**）

上游规格 §5.2（F-39）规定：`ComposerFocus` 只对**安全可编辑目标**执行聚焦。判定必须是**三重**：

1. **role 白名单**：`AXTextArea` / `AXTextField` / `AXComboBox`
2. **subrole 排除**：`AXSecureTextField`（密码框）——**role 与 subrole 双重判断**
3. **语义关键词拒绝**：`AXDescription` / `AXTitle` / `AXHelp` / placeholder 中出现下列任一词 → **拒绝**：

```
password, passcode, secret, api key, apikey, token, credit card,
search, find, filter, address bar, settings, preferences, command palette,
密码, 口令, 密钥, 令牌, 信用卡, 搜索, 查找, 筛选, 设置, 偏好
```

> **关键**：Chrome 地址栏（`cmd+l`）、查找栏（`cmd+f`）、Slack 快速切换器（`cmd+k`）**全部命中拒绝词**，而且它们是**原生工具条控件**——AX 可用性**不依赖 web 内容树**，拿它们测冷路径必然得到「很快」的假阳性，**直接误导 R4 决策**。
>
> 因此 `AXSearchField` subrole **不再无条件计入可编辑**：它只在「语义关键词检查通过」时才计入。

### 5.3 测量协议（每个 App）—— **两个前置条件，顺序是关键**

> **前置条件 1（可编辑焦点）**：冷测量要求「当前焦点就在**产品会用的**可编辑元素上」。否则 `AXFocusedUIElement` 返回窗口 / `AXWebArea`，那是**测量装置的问题**，不是快路径的能力问题，绝不能据此触发 R4 降级。
>
> **前置条件 2（无 AX 触碰）**：冷测量必须发生在**任何 AX 调用之前**。Chromium / Electron 的树一旦被 AX 触碰就会建起来，之后再测「首次」已不是冷值。

#### 5.3.1 可编辑焦点的准备（按 App 分别设计，**全程不碰 AX**）

| App | 目标 | 准备方式 | 备注 |
| --- | --- | --- | --- |
| **TextEdit** | 文档正文 `AXTextArea` | **两步**（v1.5 修订）：① `mktemp -t vb-probe-target` 生成临时文件；② **`NSWorkspace.openApplicationAtURL:configuration:completionHandler:`**（F-42，`createsNewApplicationInstance = true`）传入的是 **TextEdit.app 的 bundle URL**（**不是**文档 URL）→ 拿到 `NSRunningApplication`（含 pid）；③ **再**用 `NSWorkspace.open(临时文件 URL)` 把文档打开到该实例 | 结束时关闭该实例（pid 校验）。**为什么分两步**：该 API 的第一个参数是**应用 URL**，直接传文档 URL 会失败或行为未定义 |
| **Chrome** | **页面内 `<textarea>`** | 生成 `probe.html`：`<textarea autofocus>` 放在 body 首位；用 **`open -a "Google Chrome" "file://<tmp>/probe.html"`** 打开 | **两种方式二选一，见下** |
| **Slack** | 消息 composer | **提示用户自己点进 composer**，然后回车继续；**探针随后会自行重新激活 Slack**（§5.3.2 第 4 步） | 见下 |

**Chrome 的两种准备方式必须显式二选一**（`--chrome-setup=autofocus|tab`）：

| 方式 | 做法 | 限制 |
| --- | --- | --- |
| `autofocus`（默认） | 页面 `<textarea autofocus>`，打开即聚焦 | 若未生效 → 记 `noEditableFocus`，**本轮不重试** |
| `tab` | 页面**不含** `autofocus`，打开后按一次 **Tab** | 单键、非破坏性、不建树 |

> **为什么不能「autofocus 没生效就按 Tab」**：判断 autofocus 有没有生效必须查 AX，那就污染冷测量了；而 autofocus **已生效**时多按一次 Tab 反而把焦点移走。**一次运行只试一种**，失败就走 `noEditableFocus` 重测，不在同一次运行里二次猜测。

**Slack 改为人工点击（默认）**：

```
请切换到 Slack，用鼠标点一下消息输入框（composer），然后回到终端按回车继续。
（探针随后会自行把 Slack 重新激活到前台，再做冷测量 —— 否则量到的是终端的聚焦元素。）
```

- **零 AX、零误点**——固定坐标点窗口底边中央可能点中发送按钮 / 附件 / 消息，是真实副作用风险。
- **「回到终端按回车」与 G1-a 的「请勿触碰键盘」不冲突**：G1-b 的这一步是**独立的在场提示**，
  发生在 G1-b 自己的流程里；G1-a 的「请勿触碰键盘」横幅只管 G1-a 的键盘注入阶段（§6.5）。
- 自动点击作为**可选**（`--slack-setup=autoclick`）：用 `CGWindowListCopyWindowInfo` 取 Slack 主窗口 bounds（不碰 AX），
  点击前做**坐标合法性校验**——v1.5 把它降级为**可实现的粗规则**：
  只允许落在**窗口下部 1/4** 且**排除最右侧 15% 水平区域**（发送 / 附件控件通常在那儿）。
  - **为什么不能校验「不落在任何子按钮区域」**：那需要控件级信息，
    而探针**不申请屏幕录制权限**（窗口名全为空，F-24）且**冷测量前不碰 AX**（前置条件 2）——数据拿不到。
  - **必须如实声明**：粗规则**无法保证**不误点按钮。失败一律**回退到人工点击**，默认仍是 `manual`。

**对照数据点（不参与判定）**：Chrome 地址栏（`cmd+l`）与查找栏（`cmd+f`）、Slack 快速切换器（`cmd+k`）**可以额外测一次**，但结果标记为 **`nativeToolbarControl`**——它们是原生工具条控件，**明确不参与 R4 判定**，只作为「原生控件 AX 天然可用」的对照。

> **为什么不用 `cmd+l` 作为 Chrome 的主目标**：它落在上游拒绝词表内（F-39），而且**测不到 web 内容树**——上游 §5.2 明写「Chromium / Electron 系在宣告之前辅助功能树是空的」，只有 web 内容里的输入框才能检验「未宣告时 `AXFocusedUIElement` 能不能拿到」。

**`fastPathSetupVerified` 的语义修正**：

- 它表示「**准备动作本身成功**」（例如 `Tab` 按过、鼠标点过、TextEdit 已打开），**按构造声明**，**不做前置 AX 校验**——因为校验本身就要调 `AXFocusedUIElement`，而那**正是冷快路径那一次调用**，会直接污染冷值（前置条件 2）。
- 真正的可编辑性复核放在**冷调用之后**：用冷调用解析出的元素读 role / subrole / description / title / help / placeholder，按 §5.2 的三重判定 → `coldFastPathElementEditable`。
- 因此 `fastPathSetupVerified = false` 只表示「准备动作失败 / 冷调用什么都没解析出来」，**不是**「解析出了但不可编辑」——后者由 `coldFastPathElementEditable = false` 表达。

**准备失败** → 记 `noEditableFocus` → 该 App 判为「**无效测量**」（§5.6），**不触发 R4**。

#### 5.3.2 测量顺序

1. `NSRunningApplication.activate` 激活目标 App → 等 0.5s
2. **等待页面加载**：`--page-load-wait`（默认 **2s**，记录实际值）。`open` 返回后页面可能还没解析完，焦点停在 body → 会被误判成「快路径无效」
3. 按 §5.3.1 准备可编辑焦点 → 记录 `fastPathSetupVerified` / `fastPathSetupMethod`（`fileOpen` / `autofocus` / `tab` / `manualClick` / `autoclick` / `none`）。**此步不碰 AX**
4. **重新激活目标 App → 等 0.5s → 校验 frontmost**（v1.5 新增）
   - 校验「frontmost 的 pid == 目标 App 的 pid」**且**「目标窗口在前台」；不通过 → 该 App 记 `noEditableFocus`（**准备失败**类）→ **本次测量无效**，**不触发任何降级结论**
   - **为什么必须重新激活**：Slack 的人工点击流程以「回到终端按回车继续」结束，此时 frontmost 是**终端**。终端的聚焦元素也是 `AXTextArea` → 会通过 §5.2 的 role 白名单 → 探针会把**终端**的聚焦耗时与可编辑性记在 Slack 名下 → **假阳性**。这正是 v1.3 #44 声称已经消除的错误类型
   - **为什么不用 AX 校验 frontmost**：查 frontmost 用 `NSWorkspace` 即可，**不产生 AX 树触碰**，不污染冷值
5. **不做任何 AX 调用**，直接测 **冷快路径**：`AXUIElementCopyAttributeValue(systemWide, kAXFocusedUIElementAttribute)`
   - 超时已按 §5.4 设为 **0.3s**（与生产一致）
   - 记录耗时、是否解析出元素、是否在超时处返回空
   - **不得在此之前做全路径宣告**（否则树已建，冷值失效）
6. **复核可编辑性**（用上一步解析出的元素，不产生新的 AX 树触碰）：读 role / subrole / description / title / help / placeholder → 按 §5.2 的三重判定 → `coldFastPathElementEditable`
7. **树建起归因**（`coldFastPathTriggeredTreeBuild`）——**非循环论证**的判据：
   - 冷调用若**在 300ms 内解析出元素** → 树此前已可用 → `coldFastPathTriggeredTreeBuild = notApplicable`
   - 冷调用若**超时**：**不做宣告**，改用 **3 秒超时**再查一次，记录这次查询的耗时 `postColdQueryMs` 与是否解析出元素 `postColdQueryResolved`：
     - `postColdQueryMs` **快**（< ~300ms）→ 树在冷调用期间就已建好 → **`coldFastPathTriggeredTreeBuild = true`**（冷调用触发了建树，只是没等到）
     - `postColdQueryMs` **慢**（~1–2s）→ 是这次查询自己建的树 → **`coldFastPathTriggeredTreeBuild = false`**
   - **为什么旧判据是循环论证**：旧写法是「调用后若能取到元素就算建树」——而**取到元素本身就可能是这次查询建树的结果**，无法区分。改用「后一次查询的耗时快慢」才能区分。
   - **测完必须把超时调回 0.3s**（第 8 步之前），否则后续测量会带上 3s 的超时
8. **全路径**（**每个 App 都跑一次**，即使快路径已命中——避免「未运行 ⇒ 指标缺失 ⇒ 被当成 false 计入未定论」的歧义）：
   - **必须与上游规格 §5.2 的全路径算法一致**，否则数字不可比：
     - **先把超时抬到 3.0s**（§5.4 阶梯）——不抬的话每次 AX 调用都被截到 300ms，既让 `fullPathHit` 假阴性，又让 `fullPathWarmMs` 失真
     - 先设 `AXManualAccessibility`；若返回 `attributeUnsupported` → **回退设置 `AXEnhancedUserInterface`**
     - 遍历上限 **5,000 个元素**（防卡死）
     - 重试 **12 次 × 250ms**（最多 **3 秒**）
   - 指标命名为 **`fullPathWarmMs`**（树已建条件下的全路径耗时）
   - **本步同时承担「保证树已建」的职责**——所以稳态快路径（第 9 步）必须排在它**之后**（v1.5 修订）
   - ⚠️ 若走了 `AXEnhancedUserInterface` 回退：**恢复后必须验证窗口几何**（用 `CGWindowListCopyWindowInfo` 比对 bounds）——上游 §5.2 警告「部分 Electron 版本下它会扭曲窗口与动画」（F-39）。若几何被改动 → 在报告里提示**重启 Slack**
9. **稳态快路径**：连续 **N = 10** 次，每次记录：查询耗时、**属性读取耗时**、二者之和 `fastPathTotalMs`、是否解析出元素、是否可编辑、是否超时
   - **前置条件：树已建**——由第 8 步的全路径宣告保证 → 记录 `warmMeasuredAfterDeclaration = true`
   - **超时已按第 7 步调回 0.3s**
   - **为什么必须排在全路径之后**（v1.5 修订）：Chromium / Electron 的 Web 内容树**只有被宣告过才有内容**（上游 §5.2）。在空树上测稳态，解析率会接近 0/10 → 探针得出「快路径无效」→ **触发 R4 降级**——而上游的真实情况是「首次宣告一次后快路径就会命中」，**结论方向完全相反**。上游 §5.2「树一旦建好就一直有效」这句话，**也只有在宣告之后测才谈得上验证**
10. **恢复**：`AXManualAccessibility` / `AXEnhancedUserInterface` 改回原值（**归一化后比较**，见 §5.5）；`AXUIElementSetMessagingTimeout(systemWide, 0)` 复位全局超时（F-29）
11. **冷全路径**（`fullPathColdMs`）只能靠**新进程**测 → `--measure-full-cold`

**冷快路径的两种结果都要写进报告**

| 冷调用结果 | 含义 |
|---|---|
| 在 0.3s 内**解析出元素** | 该 App 的快路径**开箱即用**，自动聚焦可直接开启 |
| 在 0.3s 处**超时返回空** | 这是**预期行为**（正是生产里的「快速失败」），意味着该 App **首次**需要一次全路径宣告才能把树建起来 |

因此：**快路径是否有效，由稳态指标判定**；冷指标只回答「首次要不要先宣告一次」。

**冷测量的新鲜度要求**：Chromium 的树一旦被**任何**辅助客户端（VoiceOver、其它工具、上一次探针运行）碰过就保持建好 → 那时首次测量也不是冷的。因此：

- 报告必须记录 `appStartedAt` 与 `coldMeasuredAt`，并计算 `coldWithinFreshWindow`（**冷测量须在 App 新启动后 60 秒内完成**，否则标 `coldStale`）
- `coldStale = true` 时，冷指标**不参与任何判定**，只在报告里标注
- 记录 `mayHaveBeenDeclaredBefore`（本次运行前该 App 是否可能已被宣告）——无法完全确知，只能按「该 App 上次运行后是否重启过」推断并如实标注

**可选 `--measure-full-cold`（默认关闭）**：退出并重启目标 App，在**无 AX 触碰**的全新进程上测一次，得到 `fullPathColdMs`（真实建树 + 宣告 + 遍历 + 重试的总成本）。**仅作参考值，不参与任何判定。** 限制：**只允许对无未保存状态的 App 执行**（TextEdit 用临时文件保证无未保存内容；Chrome / Slack 有未保存内容时必须拒绝执行并在报告里说明）。

### 5.4 超时设置

按 F-29（`AXUIElement.h:389-392` 逐字）：

| 动作 | 调用 | 效果 |
| --- | --- | --- |
| 设定 | `AXUIElementSetMessagingTimeout(AXUIElementCreateSystemWide(), 0.3)` | **本进程全局超时**——对本进程内所有 AX 调用生效 |
| 设定（可选） | `AXUIElementSetMessagingTimeout(appElement, 0.3)` | 只作用于该对象本身；对**其他**对象（含它返回的子元素）无影响 |
| 复位 | `AXUIElementSetMessagingTimeout(systemWide, 0)` | **复位全局超时为默认值** |

> **只设 systemWide 即全局**，它已覆盖对目标 App 的调用。对其他 element 单独设置只作用于该对象本身——这一点写清楚，是为了避免误以为「给子元素设了超时就会继承」。

**超时阶梯**

| 阶段 | 超时 | 理由 |
| --- | --- | --- |
| 快路径（冷 + 稳态） | 0.3s | 对应上游规格 §5.2 的快路径预算 |
| 树建起归因的后一次查询（§5.3.2 第 7 步） | 3.0s | 需要给「自己建树」的慢路径留出完成时间；**测完立即调回 0.3s** |
| 全路径 | 3.0s | 对应上游规格 §5.2 的全路径预算（≤3 秒） |
| 结束 | `0`（复位全局默认） | F-29 的复位语义；**必须复位**，否则后续全路径遍历也会被截到 300ms（上游规格 §5.2:418） |

### 5.5 指标

| 指标 | 含义 |
| --- | --- |
| `fastPathSetupVerified` | **准备动作本身**是否成功；**按构造声明，不做前置 AX 校验** |
| `fastPathSetupMethod` | `fileOpen` / `autofocus` / `tab` / `manualClick` / `autoclick` / `none` |
| `targetKind` | `editableComposer`（产品会用的目标）/ `nativeToolbarControl`（对照点，不参与判定） |
| `appStartedAt` / `coldMeasuredAt` / `coldWithinFreshWindow` | 冷测量的新鲜度（§5.3.2 末） |
| `mayHaveBeenDeclaredBefore` | 本次运行前该 App 是否可能已被宣告（**推断值，如实标注**） |
| `coldFastPathQueryMs` | 冷快路径**纯查询**耗时（0.3s 超时生效下） |
| `coldFastPathTotalMs` | 冷快路径**总耗时** = 查询 + 读取 role/subrole/description/title/help/placeholder 做安全过滤 |
| `coldFastPathResolved` | 冷快路径是否在 300ms 内**解析出元素**（元素是否可编辑不影响此项） |
| `coldFastPathElementEditable` | 冷快路径解析出的元素按 §5.2 三重判定是否**可编辑** |
| `coldFastPathTimedOut` | 冷快路径是否在 0.3s 处超时返回空（= 首次需要先全路径宣告） |
| `coldFastPathTriggeredTreeBuild` | 这次快路径调用本身是否触发了 Chromium 建树（**时序判据**）——直接回答上游规格 §5.2:412 |
| `postColdQueryMs` / `postColdQueryResolved` | 冷调用超时后的 3s 查询耗时与是否解析出元素（**§5.6 的有效性判定用它**） |
| `warmFastPathQueryMedianMs` / `warmFastPathQueryP95` / `warmFastPathQueryMin` / `warmFastPathQueryMax` | 稳态**纯查询**耗时 |
| `warmFastPathTotalMedianMs` / **`warmFastPathTotalP95`** | 稳态**总耗时**（含属性读取）——**对着 300ms 预算判定用 `warmFastPathTotalP95`**（v1.5 修订，见 §5.6 的口径说明） |
| `warmMeasuredAfterDeclaration` | 稳态是否测在**全路径宣告之后**（即树上已有内容）。**必须为 true** 才是有效测量（§5.3.2 第 9 步） |
| `warmFastPathResolvedRate` | 稳态**解析率**（如 10/10，指「解析出元素」） |
| `warmFastPathEditableCount` | 稳态中**可编辑**的样本数——**以样本数为分母**（不是以解析数为分母）：判定要求「解析 10/10 **且** 可编辑 10/10」 |
| `warmFastPathTimeoutCount` | 稳态超时**次数**（N=10） |
| `fastPathRole` / `fastPathSubrole` | 命中元素的角色 / 子角色 |
| `fastPathDescription` / `fastPathTitle` / `fastPathHelp` / `fastPathPlaceholder` | 命中元素的这四个属性（§5.2 三重判定要用，**四个都要记**供人工复核拒绝词） |
| `fullPathWarmMs` / `fullPathHit` | 全路径耗时（3.0s 超时，树已建条件）与命中——**每个 App 都会跑，不存在缺失** |
| `fullPathColdMs` | 冷全路径耗时（**仅 `--measure-full-cold` 可得**，参考值） |
| `manualAccessibilityBefore` / `After` | 宣告属性恢复验证。**归一化规则**：三态 `nil`（属性不存在/读回 error）/ `false`（未宣告）视为**等价**，都记作 `notDeclared`；`true` 记作 `declared`。**只有 `declared` vs `notDeclared` 不一致才算恢复失败** |
| `windowGeometryChanged` | 若走了 `AXEnhancedUserInterface` 回退：恢复后窗口几何是否被改动（§5.3.2 第 8 步） |
| `cleanup` | 退出清理结果（无卡键 / 未在收音 / 输入源已恢复），见 §6.2 |

### 5.6 判读（对应上游规格 §5.2 决策表）

**Chrome 与 Slack 各自独立判定。**

> **v1.5 修订**：判定阈值统一用 **`warmFastPathTotalP95`**（原表用 median，与紧邻的「为什么不用 median」自相矛盾）；
> 补上**原生类不达标**的判定行（上游 §13.1 问题①需要一个否定答案）；补上 **`coldStale`** 规则
> （冷测量失效时判读改用稳态指标，且**不得**据此触发 R4）。

| 观察 | 结论 |
|---|---|
| `fastPathSetupVerified` = false | **`noEditableFocus`：准备动作失败**，本次测量无效 → 不触发任何降级结论，列入「需重测」 |
| **`coldWithinFreshWindow` = false（`coldStale`）** | **本次冷测量作废** → 冷指标**不参与任何判定**，改用**稳态指标**判定；若稳态也不达标 → 记 **`noEditableFocus`（需重测）**，**不触发 R4**。理由：冷测量不新鲜时，任何结论都不可靠；R4 降级会砍掉一个功能，宁可标成需重测 |
| `coldFastPathResolved` = false **且** `postColdQueryResolved` = false（且冷测量新鲜） | **`noEditableFocus`：连 3s 查询都没解析出元素** → 本次测量无效，列入「需重测」 |
| `coldFastPathResolved` = false **但** `postColdQueryResolved` = true（且冷测量新鲜） | **正常测量**（冷调用超时是预期行为）→ **继续走稳态判定**，不判无效 |
| `targetKind` = `nativeToolbarControl` | **对照数据点**：不参与判定，只在报告里列出 |
| 原生类：`warmFastPathResolvedRate` = 10/10 **且** `warmFastPathEditableCount` = 10/10 **且** `warmFastPathTimeoutCount` = 0 **且** `warmFastPathTotalP95` ≤ **150ms** | 快路径有效 → 自动聚焦保持默认开启 |
| Chrome（或 Slack）单独：同上四条件，且 `warmFastPathTotalP95` ≤ **300ms** | 该类快路径有效 → 自动聚焦保持默认开启 |
| Chrome（或 Slack）单独：`coldFastPathTimedOut` = true **但**稳态达标 | 「首次贵、后续快」成立 → 上游规格 §5.2「树会保持」得到实测支持；该类 App **首次**需要一次全路径宣告（这正是「自动聚焦只探快路径 + 快速失败」的预期行为，**不构成降级理由**） |
| Chrome（或 Slack）单独：稳态解析率 < 10/10 **或** 可编辑数 < 10/10 **或** 超时数 > 0 **或** `warmFastPathTotalP95` > 300ms | 该类快路径**无效** → **触发上游规格 §14 R4 降级：自动聚焦整体降级为默认关闭的独立动作** |
| **原生类：同上四条件但 `warmFastPathTotalP95` > 150ms** | **原生类快路径不达标** → 上游 §13.1 问题①的答案是「**不值得开自动聚焦**」→ **原生类的自动聚焦默认值同样翻转为「默认关闭」**（v1.5 补行：否则上游问题①没有否定答案，M0 回填会出现「原生类未结论」的空洞） |
| 任一类 `fullPathHit` = false | 该类无法用全路径聚焦 → 记入结论文档的「未定论事项」 |

> **150ms 门槛的来源**（v1.5 补）：上游规格 §5.2 预期原生 App 快路径 **~50ms**，且预算为 **≤300ms**。探针把原生类阈值**收紧到 150ms**——因为原生 App 的树是现成的，300ms 的预算对它是「不达标也算合格」的宽松线；收紧后原生类的结论才有区分度。**300ms 的预算本身仍对 Chromium / Electron 类适用**。

> **为什么必须同时要求「解析率」与「可编辑数」**：一个「永远解析出 `AXWebArea`」的 App，解析率可以是 10/10，但产品拿到的是**不可编辑的元素**——上游规格 §5.2 的 `ComposerFocus` 会拒绝它。**产品要的是可编辑目标，不是「解析出某个东西」。** 两者**都以样本数 10 为分母**（v1.5 统一口径）。

> **为什么用「总耗时」而不是「纯查询耗时」**：生产快路径 = 查询 + 读 role/subrole/description/title/help/placeholder 做安全过滤（§5.2 三重判定）。只测查询会**低估**真实聚焦成本，对着 300ms 预算判定会偏松。

> **为什么用 p95 而不是 median**（v1.5 统一）：超时设 0.3s 时，**任何超时样本都恰好落在 ~300ms**，「median ≤ 300」会把「一半超时」判成通过。所以用「**命中率 + 零超时 + p95 阈值**」三元组。median 仍然记录，但**不参与判定**。

> **稳态必须在宣告之后测**（v1.5 补）：见 §5.3.2 第 9 步。稳态判定的前提是**树已建**，由全路径宣告保证；`warmMeasuredAfterDeclaration` 必须为 true 才是有效测量。

> **避免单次抖动砍掉功能**：任何触发 **R4 降级**的结论必须**经过两次独立运行**都成立，才写入回填结论。**两次之间必须重启目标 App**（`--relaunch-app`）——因为 `AXManualAccessibility` 恢复成 false **不会拆掉 Chromium 已建好的树**，不重启的第二轮是 warm 的，测不出冷值。若不重启，第二轮必须标为 `warmOnly`，且**只有两轮都触发降级**才采信。

### 5.7 探针参数

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `--out <dir>` | `docs/verification/` | 报告输出目录。**路径相对于当前工作目录**（v1.5 补）——§8.4 的运行入口在 `tools/probes` 下执行，默认会落到 `tools/probes/docs/verification/`，所以 `run-all.sh` **必须传绝对或仓库根相对路径** |
| `--apps <list>` | `textedit,chrome,slack` | 要测的 App 子集（**调试用**；M0 结论必须跑全三个） |
| `--warm-n <n>` | 10 | 稳态快路径采样次数 |
| `--chrome-setup <mode>` | `autofocus` | `autofocus` / `tab`——**一次运行只试一种**（§5.3.1） |
| `--slack-setup <mode>` | `manual` | `manual`（人工点击，默认）/ `autoclick`（可选，粗规则坐标校验，失败回退人工） |
| `--page-load-wait <ms>` | 2000 | 页面加载等待，记录实际值 |
| `--include-toolbar-controls` | off | 额外测地址栏 / 查找栏 / 快速切换器（标 `nativeToolbarControl`，**不参与判定**） |
| `--measure-full-cold` | off | 重启目标 App 测 `fullPathColdMs`（参考值，不参与判定；仅允许无未保存状态的 App） |
| `--relaunch-app` | off | **为 R4 二次确认重启目标 App**（§5.6）。**必须同时给 `--confirm-relaunch`**——重启 Chrome 可能丢用户未保存的表单 |
| `--confirm-relaunch` | 无 | 显式确认允许重启目标 App；未给出时 `--relaunch-app` 报错退出 |
| `--dry-run` | off | 只打印将要执行的测量计划，不做任何 AX 调用 |
| **`--now <iso8601>`** | 无 | **仅测试用**：注入报告时间戳，让「同一输入 → 同一份 JSON 字节」的断言成立（§8.1） |
| **`--env <json>`** | 无 | **仅测试用**：注入环境快照（与报告的环境字段同构），同上 |

> `--measure-full-cold` 与 `--relaunch-app` **是两个不同的语义**，不共用一个参数名：前者是「测冷全路径」，后者是「为二次确认重启 App」。
>
> **哪些参数是「仅调试用」**（v1.5 补）：`--apps` 子集、`--include-toolbar-controls`、`--dry-run`、`--now` / `--env`。
> **M0 结论的正式运行必须使用默认值**，且 Chrome 与 Slack **各自独立判定**（§5.6）。

---

## 6. 副作用控制与退出清理

### 6.1 注入落点与**强制校验**

| 探针 | 副作用范围 | 强制校验 |
| --- | --- | --- |
| G1-a | 探针自己启动 TextEdit（`NSWorkspace.openApplication`，F-42）→ AX 设焦点到正文；**全部按键注入只落在该 TextEdit 实例** | **阶段 0 与每组注入前**校验 frontmost 的 **pid == 目标实例 pid** 且**目标窗口在前台**，不匹配立即中止 |
| G1-b | 注入**仅限白名单**：`Tab`（Chrome）、鼠标点击（Slack）、`cmd+l`/`cmd+f`/`cmd+k`（**仅对照点，默认关闭**）。**不注入 Option、不做长按修饰键** | 每次注入前后同样校验 frontmost |

> **为什么这是硬校验而不是「尽量」**：`NSRunningApplication.activate` 从 CLI 调用在 macOS 14+ **可能失败或被节流**。一旦失败，探针会在**按住 Cmd/Option 的 1.5–3 秒**里继续跑——此刻用户敲 Q 就是系统级 Cmd+Q，敲 W 就关窗口。50 组 × 数秒 = 数分钟的暴露面。**这不是理论风险。**
>
> **只校验进程不够**：TextEdit 可能已有多份文档窗口，用户在另一个窗口里打字时，探针的 Cmd/Option 修饰位仍会叠加上去。必须**同时校验目标窗口在前台**。

**物理输入中止（`CGEventTap`）——三条硬性要求**：

| # | 要求 | 理由 |
| --- | --- | --- |
| 1 | **忽略带 `eventSourceUserData` 标记的自家事件** | 探针 post 的事件**会**出现在 event tap 里。若不忽略，**第一次注入就自杀**。用 `kCGEventSourceUserData = 42`（F-41）打标记，tap 侧按标记过滤 |
| 2 | **监听集合必须含 `.flagsChanged`** | 用户按住 Shift / Cmd / Option 产生的是 **flagsChanged** 而不是 keyDown（F-43）。只订阅 `.keyDown` 会**漏检**最危险的「用户正按着修饰键」场景 |
| 3 | **只在检测到「非自家」事件时中止** | 与 1 配套：过滤掉自家标记后再判定，否则会把探针自己的注入当成用户输入 |

- 监听键盘事件需要 **Input Monitoring** 权限，**AX 权限不覆盖**它。
- 取得权限 → 安装上述监听。
- 取得失败 → **不弹窗索要**（避免打断用户），改为在**横幅与报告的显著位置**写明「**本次运行无法中止物理输入**」，并把「探测期间请勿触碰键盘」写得更醒目。

**横幅文案（必含）**：

```
================================================================
 VoiceBridge G1 探测即将开始
 约 8–10 分钟（快速预览约 1 分钟）
 从此刻起请勿触碰键盘 —— 探针会按住 Cmd/Option 键
 输入法可能被切换；探测结束后会尽力恢复，失败会提示
================================================================
```

不改系统设置、不改 Chatterfly 偏好、不 kill 任何进程。

### 6.2 修饰键与退出清理（防卡键）

`KeyInjector` 维护「已按下键集合」。

**能覆盖的退出路径**：正常结束、阶段切换、`SIGINT`、`SIGTERM`、`SIGABRT`、`atexit`、Swift 错误退出前的 `defer`。

**不能覆盖的**：`SIGKILL`、进程 panic / 崩溃、断电——**这些没有清理路径**，属客观限制，文档如实写明。

清理流程（**复用 §4.3 的停止程序**）：

1. 对已按下的每个键补发 up
2. 读 `CGEventSource.flagsState(.combinedSessionState)`——若 `.maskSecondaryFn` / `.maskCommand` / `.maskAlternate` 仍置位 → 判定有键卡住 → 强制补一次带修饰位的 up 并清标志
3. 执行**停止程序**（§4.3），确认 `voiceState == false`；若仍为 true → **写入报告的「清理状态」栏**并提示用户手动停止

**启动自检**：每次启动时读取 `CGEventSource.flagsState(.combinedSessionState)`——若修饰键异常置位**持续 ≥ 1 秒**才判定为「上次运行异常终止留下了卡键」（用户启动探针时若正按着修饰键，不应误报），并在报告里显著标注并提示用户手动释放。

> **原则：清理失败必须出现在报告里**，不能只打日志了事。

### 6.3 系统 Fn 误触与输入源切换

**不能靠读配置判断**：`AppleFnUsageType` 在 `com.apple.HIToolbox` 下**不存在**（F-34），无法读取「按 Fn 键来」的系统设置。

**读取与恢复都必须走 TIS**（F-35/F-36）：

| 用途 | API |
| --- | --- |
| 读当前输入源 | `TISCopyCurrentKeyboardInputSource()` → `kTISPropertyInputSourceID` |
| 枚举可选用源 | `TISCreateInputSourceList(nil, true)` → `kTISPropertyInputSourceID` + `kTISPropertyInputSourceIsSelectCapable` |
| **切换/恢复** | `TISSelectInputSource(source)` |

> ⚠️ **`defaults write com.apple.HIToolbox AppleSelectedInputSources` 不会实时切换输入源**，所以「恢复基线」不能用 `defaults`，必须用 `TISSelectInputSource`。

**行为检测**：

1. 每组探测**前后**各记录一次当前输入源 id（TIS）
2. 一旦与基线不一致 → **立即中止整轮探测**并用 `TISSelectInputSource` 恢复回基线
3. 恢复失败 → 写入报告的「清理状态」栏，并**显著提示**用户手动切回

**为什么必须这样**：本机 enabled+selectable 的输入源有 **8 个**（ABC、SCIM.ITABC、CharacterPaletteIM、PressAndHold、chatterfly.pinyin、AinuIM.Ainu、Kotoeri×2，F-36），系统默认的 Fn 行为很可能是「切换输入源」。一旦切走，**后续所有组都跑在错误的输入法上**，整轮结论作废。

**结论文档附提示**：若探测期间出现亮度 / 播放键异常，检查 系统设置 › 键盘 ›「按 🌐 键来」。

### 6.4 麦克风

收音期间 Chatterfly 占麦（菜单栏出现橙色指示）。每组执行停止程序后确认已停止；全部结束后再等 3s，确认 `kAudioProcessPropertyIsRunningInput` 回到 0，否则在报告标注。

### 6.5 在场提示

- **阶段 -1（正对照）**：等待用户操作，最长 60 秒，**这里不能用倒计时自行接管**（§4.3）；**本阶段允许用户按键**
- **阶段 0 起**：3 秒倒计时 + 横幅（含「请勿触碰键盘」），然后自行接管
- **阶段 3（人工抽检）**：提示用户「说一句话，文字会出现在输入框里」，**用户只说话、不碰键盘**（§4.3）
- **G1-b 侧**：Slack 的人工点击步骤需要用户参与一次（§5.3.1）——**那是 G1-b 自己的在场提示**，
  不是 G1-a 的阶段；「回到终端按回车」与 G1-a 的「请勿触碰键盘」不冲突（v1.5 修正：原把这一步错列在 G1-a 的在场提示里）
- 支持 `--dry-run`（只打印矩阵，不注入）

---

## 7. 输出与规格回填

### 7.1 产物

| 文件 | 内容 |
| --- | --- |
| `docs/verification/2026-10-23-g1-results.md` | 人读结论（结构见 §7.2） |
| `docs/verification/2026-10-23-g1-results.json` | 原始采样 + 摘要（机器可读） |

### 7.2 结论文档结构

| 节 | 内容 |
| --- | --- |
| §0 探测环境 | 机型、**macOS 实测版本**、Swift 版本、Chatterfly 版本与**真实安装路径**、AX 权限自检结果、**物理输入中止能力（可用/不可用）**、Chatterfly 是否为当前选中输入法、**正对照结果**（S1/S5 各自是否通过 + S5 落盘延迟 + 命中槽） |
| §1 G1-a | **快速预览段 + 完整矩阵段两段结论**（都写）· 50（或 55）组候选总表（命中 / 未命中 / **可疑命中** / `startLatencyMs` / `stopLatencyMs`）+ 归类决策程序的逐组结果 + **选键结论** + **阶段 3 人工抽检结果**（`manualCheck`：是否上屏 + 用户描述） |
| §2 G1-b | 三 App × §5.5 全部指标（**Chrome / Slack 分列**；含 `nativeToolbarControl` 对照点）+ 判读结论 + **R4 降级结论是否经两次运行确认** |
| §3 对规格的影响 | 逐条列出「哪一节 → 改成什么」 |
| §4 未定论事项 | 探针**没能**回答的问题，显式列出 |
| **§5 清理状态**（v1.5 补） | 无卡住修饰键 / 未在收音 / 输入源已用 TIS 恢复到基线 / 探针自开的 TextEdit 实例已关闭 + 失败项。**§6.2、§6.3 与停止程序都往这里写**——原文档多处要求「写入报告的清理状态栏」，但 §7.2 的结构里没有这一栏 |

### 7.3 上游规格回填清单

| 规格位置 | 回填内容 |
| --- | --- |
| §0.1 | 新增 **v3.5** 变更记录 + 一张「M0 实测回填」追溯表（沿用前六轮惯例） |
| §1.1 | **修正路径**：`~/Library/Input Methods/Chatterfly.app` → `/Library/Input Methods/Chatterfly.app`（F-01） |
| **§13.2 M0 出口** | 填入实测结论：收音快捷键、长按/点按模式、三类 App 的 AX 聚焦耗时（v1.5 修正：原写「§13.1 M0 出口」，但 M0 出口标准在 **§13.2**，§13.1 是验证门） |
| **§5.2（建树问题）** | **上游 §5.2:412 明写「查 `AXFocusedUIElement` 本身会不会触发 Chromium 建树，需要实测」→ 用 `coldFastPathTriggeredTreeBuild` 回填** |
| §4.2 触发键集合 | 若实测需双击 → 明确「触发键需要双击投递模式」；若某候选无效 → 标注 |
| §5.2 决策表 | 把「由 G1-b 给出依据」的占位换成实测数字与结论 |
| §7.2 投递表 | 若判定为点按模式 → 写明 Mac 端「开始拍 / 停止拍」的具体投递要求；若测出「松开后自启自停」或 `tapOnRelease` 变体 → **新增投递分支** |
| §14 R1 / R4 | 更新风险状态（已验证 / 仍开放 / 已触发） |
| §15 参考资料 | 补「已核实」条目：本机实测结论 |

### 7.4 回填原则

- 只回填**实测到的**。探针没测到的（例如「识别出的文字是否真的上屏」）显式标为「**未验证**」，**不写成结论**。
- **触发 R4 降级的结论必须两次独立运行都成立**才写入（§5.6）。
- 回填后上游规格版本升 **v3.5**。

---

## 8. 测试与验收

### 8.1 单测（`swift test`，纯逻辑、不依赖实机状态）

| 测试目标 | 断言 |
| --- | --- |
| `TriggerCandidates` | 主矩阵恰为 5×2×5 = **50 组**，无重复无遗漏。**测试带 capability 参数**（`systemDefined` spike 成功后为 55 组）→ 断言数量 = `capabilityAware ? 55 : 50`；Cmd/Option 排在 Fn 之前 |
| `tapDurationMs` | 注入器与报告**共用同一常量**；断言 **注入器源文件内**不存在**无名**的 `120` 字面量。<br>⚠️ **范围必须限定**：文档参数表、`doubleClickGapMs = 120ms`、§5.6 的 `150ms` 门槛等都与注入器无关，故**不能**断言「全仓无 120」。正确形态是「所有 120 必须来自命名常量 `tapDurationMs` 或 `doubleClickGapMs`」 |
| **停止拍的驻留时长**（v1.5 新增） | 停止拍必须是 `down → 立即 up`（驻留 **0ms**），**与 `tapDurationMs` 分离**——断言「停止拍路径里不出现 `tapDurationMs`」。上游 §7.2 用驻留时长区分开始拍 / 停止拍，停止拍驻留 120ms 会被 IM 当成开始拍 |
| `VoiceStateOracle` | 连续 2 次一致才翻转（去抖）；`S1 \|\| S4` 合成；**单样本为真 → 标记「可疑命中」而非直接判否**；抖动序列不误判 |
| **归类决策程序** | **用穷举真值表验证「每种 `(gStarted, gDuring, gAfter, secondPressStops)` 组合都落到恰好一个分支」**（含 `gDuring = n/a`），**无死分支、无表外落点**（不依赖实机） |
| **`gDuring` 的手势映射**（v1.5 新增） | `tap120` → `n/a`；`double120` / `double300` → `n/a`（单次按压驻留 50ms < 200ms）；`hold1500` / `hold3000` → true / false。断言**双击手势永不被归类为长按模式** |
| **停止程序** | 状态机用例：up 即停 / up 无效需补停止拍 / 补停止拍后仍不停 → `stopFailed`；**长按模式下不得补停止拍**（否则会开新会话） |
| `Report` | **同一输入 → 同一份 JSON 字节**：报告中的时间戳与环境字段必须**可注入**（`--now <iso8601>` / `--env <json>`），否则该断言不成立 |
| 延迟计算 | 样本时间戳 → 翻转时刻的毫秒差；`startLatencyMs` / `stopLatencyMs` |
| `KeyInjector` 状态机 | 抽象接口下「已按下集合」的释放路径全覆盖（**测试不发真实事件**）；自家事件**必须**带 `eventSourceUserData` 标记（F-41） |
| 选键规则 | 多候选命中时按 §4.4 的四级规则选出唯一结果；**「2× 延迟」阈值有边界用例**（`> 2×` 而非 `≥ 2×`）——且该例外分支**必须可达**（选键顺序：先定停止语义档，再在档内比延迟，§4.4） |
| **G1-b 判读**（v1.5 新增） | 阈值用 **`warmFastPathTotalP95`**（不是 median）：原生 150ms / Chromium-Electron 300ms；边界值（恰好等于阈值）**通过**，超出**不通过**；「解析 10/10 **且** 可编辑 10/10」双条件缺一即不通过；**`coldStale` 为真时冷指标不参与判定**，且稳态也不达标时结论是「无效测量」而**不是** R4 降级 |
| 参数校验 | `--declared-key` 与 `--declared-mode` **只给其一必须报错**；`--relaunch-app` 无 `--confirm-relaunch` 必须报错 |

### 8.2 不做单测的部分

真实 CGEvent 注入、真实 AX 调用、真实窗口采样、真实计数器读取——依赖实机状态，由探针实跑产出报告覆盖。

**原始采样序列必须保留在 JSON 报告里**（供事后复核去抖是否吞掉短脉冲）。

### 8.3 M0 完成定义

1. `swift build -c release` 通过
2. `swift test` 全绿
3. 阶段 -1 正对照**通过**（否则后续结论无效，M0 不得宣称完成）
4. `vb-g1a-trigger` 跑完：**快速预览段与完整矩阵段两段结论都产出**；矩阵段产出 50（或 55）行结论（**「全部未命中」在正对照通过的前提下也是有效结论**，同样能触发 R1）。**正式运行必须用默认参数**——`--only` / `--matrix-only` / `--max-groups` 都会让「全部未命中」的结论失去意义（§4.5）
5. `vb-g1b-focus` 跑完，三类 App 各有指标**或**明确的跳过原因；**稳态测量必须在全路径宣告之后**（`warmMeasuredAfterDeclaration = true`）
6. 人工抽检完成（§2.3 / §4.3 阶段 3）：**由探针注入**命中组合，用户说一句话，文字上屏，结果记入报告
7. `docs/verification/` 下 `.md` + `.json` 均存在
8. 上游规格完成回填并升版 v3.5，§0.1 有追溯表
9. **探测后系统状态干净**：无卡住修饰键、Chatterfly 未在收音、**输入源已用 TIS 恢复基线**、探针自己开的 TextEdit 实例已关闭

### 8.4 运行入口

```bash
cd tools/probes
swift build -c release
.build/release/vb-g1a-trigger --out ../../docs/verification/
.build/release/vb-g1b-focus  --out ../../docs/verification/
```

另配 `tools/probes/run-all.sh` 依次跑两个探针并汇总。

### 8.5 工程约束

- 目标平台：`macOS 14+`（**14+ 未在本机验证**，实测环境为 macOS 27.0.1 —— 见 F-26），Swift 6，**无第三方依赖**（仅 AppKit / ApplicationServices / CoreAudio / CoreGraphics / Carbon-HIToolbox 的 TIS）
- **权限**：探针**必须**有 AX 权限（自检）；**不申请**屏幕录制权限（只读窗口 owner/bounds/alpha —— 见 F-24）；**不申请** Automation 权限（不用 `osascript` —— 见 §4.3 阶段 0）；**不主动索要** Input Monitoring（探测其可用性，拿不到就记录限制 —— 见 §6.1）
- 探针代码进仓库 `tools/probes/`，结论进 `docs/verification/`
- **不提交、不推送**——按项目现行约定，提交与否由用户决定

---

## 9. 未定论与风险

| # | 项 | 说明 | 处置 |
| --- | --- | --- | --- |
| R-01 | 合成事件是否被 Chatterfly 的快捷键识别器接受 | 若它用 Carbon `RegisterEventHotKey`，合成 CGEvents 可能不被识别 | 两种事件形态全覆盖（§4.1） |
| R-02 | **Fn 的 `systemDefined` 事件无 `NX_KEYTYPE` 依据** | 22 个 `NX_KEYTYPE_*` 常量里没有 Fn（F-31） | **先做 30 分钟 spike**，首选**有 SDK 依据的修饰键通道**：`data1 = (NX_MODIFIERKEY_SECONDARYFN << 16) \| (NX_KEYDOWN << 8)`（F-32/F-33/F-38）；构造不出则记 `unsupportedBySDK` |
| R-03 | **合成 Fn 可能切换输入源** | `AppleFnUsageType` 读不到（F-34），本机 enabled+selectable 输入源有 **8 个**（F-36） | 每组前后用 **TIS** 记录当前输入源，变化即**中止并用 `TISSelectInputSource` 恢复**（§6.3）；Fn 排在候选最后 |
| R-04 | **判据器可能整体失效** | S1 只验证过「空闲 = 假」（F-17） | **阶段 -1 正对照**（§4.3）；不通过则不出结论 |
| R-05 | **S5 计数器的落盘延迟未知** | `cfprefsd` 可能缓存未刷盘 | 正对照时**实测落盘延迟**；延迟过大则 S5 降级为整体佐证，不逐组归因 |
| R-06 | 语音输入可能要求当前有可编辑焦点 | 未满足时 IM 可能不启动，探针会误判「唤不醒」 | 阶段 0 强制 AX 设焦点 + 校验基线 `voiceState == false` |
| R-07 | 「按下 → 收音」延迟的绝对值 | 依赖 IM 内部实现与机器负载 | 只作**相对比较**（选键规则的第 3 级），不设绝对阈值判定 |
| R-08 | Chrome / Slack 的可编辑目标定位 | 不同版本 AX 树结构可能不同 | 页面内 `<textarea>`（Chrome）+ 人工点击 composer（Slack）；都失败则记 `noEditableFocus`，**不触发 R4** |
| R-09 | **探针抢焦点 / 抢按键** | 用户选了「全自动」，但探针会 activate App、注入按键 | 阶段 0 + 每组注入前**强制校验 frontmost pid + 目标窗口**；横幅写明「请勿触碰键盘」；§6.1 |
| R-10 | **物理输入中止会自杀** | `CGEventTap` 会看到探针自己 post 的事件 | ① 用 `eventSourceUserData`（F-41）标记并过滤自家事件；② 监听集合含 `.flagsChanged`（F-43）；③ 只在检测到非自家事件时中止（§6.1） |
| R-11 | 崩溃 / `SIGKILL` 后的残留卡键 | 无清理路径 | 文档如实写明（§6.2）+ **启动自检**（持续 ≥1s 才判定）检测残留并标注 |
| R-12 | **G1-b 的冷测量被前序 AX 调用污染** | Chromium 树一旦被 AX 触碰即建起 | **测量顺序重排**（§5.3.2）：冷测量在任何 AX 调用之前；焦点准备用**不碰 AX** 的方式（文件打开 / autofocus / Tab / 人工点击） |
| R-13 | **G1-b 的快路径被误判为无效** | 焦点不在可编辑元素 → 判定失真 | 焦点前置 + `fastPathSetupVerified` + `noEditableFocus` 类（§5.6）；**降级结论需两次运行确认**（§7.4） |
| R-14 | 探针与产品实现之间的常量漂移 | 产品实现要到 M2 才存在 | 探针内 `tapDurationMs` 为**唯一定义点**，单测断言注入器内无第二处硬编码；M0 结论文档记录该值，M2 对齐（上游 §7.1 / §7.2） |
| R-15 | **测量目标与产品真实场景不符** | 地址栏 / 查找栏 / 快速切换器是原生控件且命中上游拒绝词 → 假阳性 | Chrome 改用**页面内 `<textarea>`**、Slack 改用**人工点击 composer**；原生控件只作 `nativeToolbarControl` 对照点（§5.2 / §5.3.1） |
| R-16 | **第二轮 R4 确认不是冷测量** | `AXManualAccessibility` 恢复成 false **不拆树** | 两次之间用 `--relaunch-app`（需 `--confirm-relaunch`）重启；否则第二轮标 `warmOnly`，且两轮都触发才采信（§5.6） |
| R-17 | **S5 可能只覆盖 `voice` 槽** | plist 里无 `StartCount_voiceCommand`（F-40） | S5 同时读两个 key 取**最大增量**，记录命中槽（§4.2） |
| R-18 | **停止程序不完整导致整轮作废** | 点按语义下「补发 up」停不下来 → 下一组基线为 true → 整轮中止 | **模式无关的停止程序**：up → 等 1s → 仍在收音则补一次停止拍 → 再等 1.5s → 仍不停则 `stopFailed` 并中止（§4.3） |
| R-19 | **「按住」轴取错导致误分类** | 固定用 `hold1500` 会把 `hold3000` 命中误判成「双击即启即停」 | 「按住」轴一律取自命中手势自己的 `gDuring`；`hold1500` 只作长按基线；归类用**穷尽决策程序**（§4.3） |
| R-20 | **冷快路径超时被误判为无效测量** | 会让 Chromium 永远得不出结论，R4 永不触发 | 无效条件加严：必须是「准备失败」**或**「冷查询与 3s 后继查询**都**没解析出元素」才算无效（§5.6） |
| R-21 | **冷测量可能被其它 AX 客户端预热** | VoiceOver / 上次探针运行碰过 Chromium，树就保持建好 | 记录 `appStartedAt` / `coldMeasuredAt` / `coldWithinFreshWindow`（60s 窗口）与 `mayHaveBeenDeclaredBefore`；`coldStale` 时冷指标不参与判定（§5.3.2） |
| R-22 | **`AXEnhancedUserInterface` 回退可能扭曲 Electron 窗口** | 上游 §5.2 已有此警告 | 恢复后用 `CGWindowListCopyWindowInfo` 比对窗口几何 → `windowGeometryChanged`；被改动则提示重启 Slack（§5.3.2 第 8 步） |
| R-23 | **v1.5 新增**：**重新激活目标 App 可能改变其内部焦点状态**——Slack 被重新激活后，用户点过的 composer 是否仍是该 App 的聚焦元素，未经验证 | 若焦点丢了，冷快路径会量到 App 的窗口 / 列表而不是输入框 → 误判「不可编辑」 | 重新激活后**等 0.5s** 再量；用**冷调用解析出的元素**复核可编辑性（§5.3.2 第 6 步）；复核不通过 → 记 `coldFastPathElementEditable = false`，**不改判冷值本身**（§5.3.2） |
| R-24 | **v1.5 新增**：**宣告本身也是一次 AX 触碰，也可能建树**——`coldFastPathTriggeredTreeBuild` 只区分「冷调用建了树」与「后继查询建了树」，**分不出「宣告建了树」** | 建树归因可能把宣告的功劳算到冷调用头上 | 归因只在**冷调用超时**的场景下做（此时尚未宣告）；报告里写明归因的边界；**稳态判定不依赖归因结果**（§5.3.2 第 7 步 / §5.6） |
| R-25 | **v1.5 新增**：**「创建键盘 `CGEventTap` 成功」不等于「拿到了 Input Monitoring 权限」**——macOS 上 tap 可能创建成功但收不到事件 | §4.3 阶段 0 第 4 步会误报「物理输入中止可用」，实际形同虚设 | 阶段 0 的能力探测**只报告 tap 是否创建成功**，并在报告里**如实标注这是「tap 创建成功」而非「已确认能收到事件」**；横幅照写「请勿触碰键盘」（§4.3 / §6.1） |

---

## 附录 A：上游规格中的「待实测」占位（本文负责回填）

| 占位表述 | 位置 | 由谁回填 |
| --- | --- | --- |
| 「具体目标输入法认哪个键、认长按还是点按，由验证门 **G1-a** 实测确定」 | §4.2 | 本文 §4.3 / §4.4 |
| 「**G1-a 实测决定用哪种模式**」 | §7.2 | 本文 §4.3（归类决策程序）+ §4.4（选键规则） |
| 「默认值的决策规则（按目标 App 类别，**G1-b 给出依据**）」 | §5.2 | 本文 §5.6 |
| 「**『查 `AXFocusedUIElement` 本身会不会触发 Chromium 建树』需要实测**」 | §5.2（行 412） | 本文 §5.3.2 第 7 步 / §5.5（`coldFastPathTriggeredTreeBuild`） |
| 「**G1-b 实测标定**（原生 App 与 Chromium/Electron 系各一次）」 | §14 R4 | 本文 §5.6（Chrome / Slack 分列判定） |
| 「**M0** \| 执行 G1-a / G1-b 前置验证 \| 结论写入 `docs/`」 | §13.2 | 本文 §7 |
| 「装在 `~/Library/Input Methods/Chatterfly.app`」 | §1.1 | 本文 §7.3（路径修正） |

---

## 10. 修订记录

### v1.0 → v1.1

| # | 级别 | 问题 | 处置 |
| --- | --- | --- | --- |
| 1 | **P0** | 判据器只有「空闲 = 假」的验证，**没有正对照** → 「全部未命中」无法判定 | 新增**阶段 -1 正对照**；不通过即**中止且不出结论**（§4.3） |
| 2 | **P0** | 手势矩阵**漏了产品真正会发的 120ms 点按**；`single`(1.5s) 与 `longPress`(3s) 语义重叠 | 手势改为 `tap120` / `hold1500` / `hold3000` / `double120` / `double300`；`tap120` 与上游 §7.1 的 `tapDuration` **同源**（§4.1） |
| 3 | **P0** | G1-b 的**冷测量被前序 AX 遍历污染**，`fullPathMs` 系统性偏小 | 测量顺序重排：冷测量在任何 AX 调用之前；指标改名 `fullPathWarmMs`。**（v1.2 #27 已改为「每个 App 都跑全路径」）** |
| 4 | **P0** | S2/S3 的窗口判据与 F-20 自相矛盾，且窗口集合动态 | 窗口判据**降级为「只记录、不参与判定」**；主判据改为 S1 + S5（§1.3 / §4.2） |
| 5 | P1 | 合成 Fn 可能**静默切换输入源** | 每组前后记录输入源，变化即**中止并恢复**（§6.3、R-03） |
| 6 | P1 | `systemDefined` 对 Fn **无 SDK 依据** | 从主矩阵移出，改为**可选 spike**（§4.1、R-02） |
| 7 | P1 | N=10 求 p95 无统计意义，且**超时样本恰好落在 ~300ms** | 改用「命中率 + 零超时 + p95 阈值」三元组；**Chrome 与 Slack 分开判定**（§5.6） |
| 8 | P1 | §5.4「只设 systemWide 是不够的」**与头文件相反** | 改正为「**只设 systemWide 即全局**」（F-29） |
| 9 | P1 | 全路径缺上游要求的 `AXEnhancedUserInterface` 回退、5,000 元素上限、12×250ms 重试 | 全路径算法与上游 §5.2 **完全对齐**（§5.3.2） |
| 10 | P1 | `--cold-unbounded` 会重启用户正在用的 App | 改名 `--fresh-launch`，只允许无未保存状态的 App（§5.3） |
| 11 | P1 | `osascript` 需要 **Automation 权限**，AX 权限不覆盖 | 改用 `open` 启动文件（零额外权限）（§4.3） |
| 12 | P1 | 「崩溃也补发 up」做不到 | 改为「信号处理器 + atexit 覆盖，`SIGKILL`/panic/断电不可捕获」+ **启动残留自检**（§6.2） |
| 13 | P1 | 多候选命中时**没有选键规则** | 新增四级选键规则（§4.4） |
| 14 | P1 | 回填清单漏了上游 §5.2:412 的**建树问题** | 新增 `coldFastPathTriggeredTreeBuild` 指标与回填项（§5.5 / §7.3） |
| 15 | P2 | 时间预算低估（未计清理等待） | 改为「50 组 ≈ 5–8 分钟」，加 `--max-groups` / `--abort-on-voice-stuck` |
| 16 | P2 | 去抖会吞短脉冲 | 报告保留**原始采样序列**；单样本为真记「**可疑命中，需复测**」（§4.2 / §8.2） |
| 17 | P2 | 缺「结束延迟」测量 | 新增 `stopLatencyMs`（§4.2） |
| 18 | P2 | 「同一输入 → 同一份 JSON 字节」与时间戳字段冲突 | 报告时间戳与环境字段**可注入**（§8.1） |
| 19 | P2 | 缺判据器的端到端验收 | 新增**人工抽检**环节（§2.3、§8.3 第 6 条） |
| 20 | P2 | 环境章应区分实测与目标平台 | 补 F-26（macOS 27.0.1 实测 / 14+ 未验证） |
| 21 | P2 | `kCGWindowName` 需屏幕录制权限 | 补 F-24；探针**不申请屏幕录制**（§8.5） |
| 22 | P2 | 方案 C 的排除理由表述不准 | 排除理由改为「**读不到绑定**」（§3.1） |

### v1.1 → v1.2

| # | 级别 | 问题 | 处置 |
| --- | --- | --- | --- |
| 23 | **P0** | **阶段 2 的 2×2 与「阶段 1 命中的手势」脱钩** | 阶段 2 改为**按命中手势回放**；「按住期间」轴由 `hold1500` 基线测；`tap120` 的「期间」轴记 `n/a`；「否/是」格先做再按一次测试；新增「阶段 1 假阳性 → 复测」分支（§4.3） |
| 24 | **P0** | **「全部注入只落 TextEdit」无强制校验** | 阶段 0 + 每组注入前校验 frontmost；横幅写明「请勿触碰键盘」；键盘 `CGEventTap` 物理输入中止；§8.5 权限声明相应限定（§6.1） |
| 25 | **P0** | **冷快路径缺「焦点在可编辑元素」前置** | 焦点准备提为显式前置；记录 `fastPathSetupVerified`；§5.6 新增 `noEditableFocus` 类（§5.3.1 / §5.6） |
| 26 | P1 | `coldFastPathTriggeredTreeBuild` 判据**循环论证** | 改为**时序判据**（§5.3.2） |
| 27 | P1 | `fullPath` 未运行时**指标缺失语义未定义** | **每个 App 都跑一次全路径**，`fullPathWarmMs` 始终可得（§5.3.2） |
| 28 | P1 | **S5 用 `defaults read` 每 50ms spawn 进程**；`defaults write` **不会实时切换输入源** | S5 改用**进程内 `CFPreferencesCopyAppValue`**；输入源的读取与恢复**全部改用 TIS**（§4.2 / §6.3） |
| 29 | P1 | 正对照**判定过严**且**交互流程缺失** | 必须用真实快捷键触发；判定改为「**S1 或 S5 至少一路通过 = 通过**」；正对照**等待用户最长 60 秒**（§4.3） |
| 30 | P1 | 「全仓只此一处 120 字面量」**不可实现** | 改为「注入器与报告共用同一常量；断言 Swift 源码内无第二处硬编码」（§8.1） |
| 31 | P1 | **双击的单击驻留时长未定义**、间隔方向未定义 | `doubleClickHoldMs = 50ms`；`doubleClickGapMs` **定义为 up → down**（§4.1） |
| 32 | P1 | 选键规则第 4 级「产品侧更稳」**无判据** | 优先**长按模式**，例外：长按 `startLatencyMs` > **2×** 点按（§4.4） |
| 33 | P1 | 阶段 1 观察窗 **1.5s 偏短** | 放宽到 **3s**；预算更新为 50 组 ≈ **7–10 分钟**（§4.3） |
| 34 | P2 | 引用错位：参数表在**上游 §7.1**，§7.2 是投递表 | 全文改为「§7.1 时序参数表 / §7.2 投递表」（F-37） |
| 35 | P2 | F-33「3 个输入源」与实测域不一致 | 改用 **TIS 实时枚举**为权威来源（F-36） |
| 36 | P2 | F-24 缺正向证据 | 补：`kCGWindowName` **全为空串**而 owner/bounds/alpha 齐全（F-24） |
| 37 | P2 | F-31 缺建设性事实 | 补 **F-32** / **F-33**；spike 首选修饰键通道（§4.1 / R-02） |
| 38 | P2 | 启动自检会因用户正按着修饰键而误报 | 要求异常置位**持续 ≥ 1 秒**才判定（§4.3 / §6.2） |
| 39 | P2 | 单次采样就决定「整体降级自动聚焦」 | R4 降级结论必须**两次独立运行都成立**（§5.6 / §7.4） |
| 40 | P2 | §8.1 断言「恰为 50 组」与「55 组」冲突 | 测试**带 capability 参数**（§8.1） |
| 41 | P2 | §4.2 把 S5 标为「主」但合成式不含它 | S5 统一标为「**独立佐证**」（§4.2） |
| 42 | P2 | 固定临时路径会多会话冲突 | 改用 `mktemp -t vb-probe-target`（§4.3） |
| 43 | P2 | 全扫描对「用户本就知道绑定」过重 | 新增**快速路径**（§4.0） |

### v1.2 → v1.3

| # | 级别 | 问题 | 处置 |
| --- | --- | --- | --- |
| 44 | **P0** | **G1-b 的测量对象选错了**——`cmd+l`/`cmd+f`/`cmd+k` **全部命中上游 §5.2 的拒绝词表**，且是**原生工具条控件**，AX 可用性**不依赖 web 内容树** → 冷测量必然快 → **假阳性误导 R4** | Chrome 改用**页面内 `<textarea>`**；Slack 改用**鼠标点击 composer**；地址栏 / 查找栏 / 快速切换器降为 **`nativeToolbarControl` 对照点**；§5.2 补上**语义关键词拒绝**（F-39），删除与上游冲突的表述（§5.2 / §5.3.1 / §5.5） |
| 45 | **P0** | **`fastPathSetupVerified` 的前置校验会毁掉冷测量** | 改为**按构造声明**，**不做前置 AX 校验**；可编辑性复核移到**冷调用之后** → `coldFastPathElementEditable`（§5.3.1 / §5.3.2 / §5.5） |
| 46 | P1 | 正对照判定表第 2 行**逻辑写反** | 改为「**未通过的那一路**不可用于归因，**通过的那一路仍照常使用**」（§4.3） |
| 47 | P1 | 阶段 2 判定表**缺格 + 有死行** | 重写为**六行穷尽表** + 「`secondPressStops` 未测 → 未定论」兜底；单测用穷举真值表验证无死行（§4.3 / §8.1） |
| 48 | P1 | §5.6 **用了错的指标**（解析率而非可编辑率） | 判定改为**同时要求** `warmFastPathResolvedRate` 与 `warmFastPathEditableRate` = 10/10（§5.6） |
| 49 | P1 | **`AXManualAccessibility` 恢复不会拆树** → 第二轮不是冷测量 | 两次之间**必须重启目标 App**；不重启则第二轮标 `warmOnly`（§5.6 / R-16） |
| 50 | P1 | **spike 公式数值错**（`0xA00` vs `0x0A`） | 改为 `(NX_MODIFIERKEY_SECONDARYFN << 16) \| (NX_KEYDOWN << 8)`；新增 **F-38**（§4.1） |
| 51 | P1 | **S5 只覆盖 `voice` 槽** | S5 同时读两个 key 取**最大增量**并记录命中槽；新增 **F-40**（§4.2 / R-17） |
| 52 | P1 | **声明空间缺「双击」** | 声明模式扩为 `longPress` / `tap` / `doubleTap`；新增 `--declared-mode`（§4.0 / §4.5） |
| 53 | P1 | §8.1 单测与 §4.1 **自相矛盾**（`doubleClickGapMs = 120ms` 是第二处 120） | 断言改为「所有 120 必须来自命名常量」，并**限定在注入器源文件内**（§8.1） |
| 54 | P1 | 阶段 2 步骤 C 的**触发条件与判定表不一致** | C 的门槛改为「`holdAfter` 或 `gAfter` 任一为 true」（§4.3） |
| 55 | P1 | §4.0 与 §8.3 / §7.2 **收口冲突** | 明确 **M0 必跑完整矩阵**；快速预览改为**矩阵的前置子集**（§4.0 / §7.2 / §8.3） |
| 56 | P2 | F-36 事实错：`SCIM` 实测 `selectCapable = false` | 改写为 333 个源中 **enabled + selectable 的有 8 个**（**v1.4 已更正为 8**）（F-36） |
| 57 | P2 | §5.3.1 **缺 TextEdit 的准备行** | 补：`mktemp` + 打开临时文件，正文默认获得焦点（§5.3.1） |
| 58 | P2 | 阶段 -1 在前、「请勿触碰键盘」在后 | 明确「该提示从阶段 0 起生效」；正对照阶段允许按键（§4.3 / §6.5） |
| 59 | P2 | 阶段 2 的 A/B/C 之间**没有状态清零要求** | 每步之间强制停止并等到 `voiceState == false`（§4.3） |
| 60 | P2 | §10 v1.1 条目 #3 与现行 §5.3.2 相反 | 追加交叉引用（§10） |
| 61 | P2 | 横幅承诺「会自动恢复」但恢复可能失败 | 改为「**尽力恢复，失败会提示**」（§6.1） |
| 62 | P2 | TextEdit 多文档窗口，frontmost 只校验进程 | 用新实例启动，并**同时校验目标窗口在前台**（§4.3 / §6.1） |
| 63 | P2 | 「仅双击命中」与 F-09 的 `handleDoubleFnTriggered` **应分开记** | 判定表拆为两行（§4.4） |

### v1.3 → v1.4

| # | 级别 | 问题 | 处置 |
| --- | --- | --- | --- |
| 64 | **P0** | **「按住」轴取自固定 `hold1500`，把 `hold3000` 命中误判成双击**——`hold3000` 存在的唯一理由就是「长按阈值可能 >1.5s」；阈值 = 2s 时 `hold1500` 不启动而 `hold3000` 启动后松开即停 → 落到「双击即启即停」行 | 「按住」轴一律取自**命中手势自己的 `gDuring`**（按压窗口 ≥200ms 时有效，`tap120` 记 `n/a`）；`hold1500` **只作长按基线，不参与归类**；归类改写为**穷尽决策程序**（§4.3），单测用穷举真值表验证（§8.1） |
| 65 | **P0** | **点按模式下「补发 up」停不下来，第一组命中后整轮就废**——Chatterfly 点按语义是「再按一次才停」，up 不停；残留会话使下一组基线为 true → 整轮中止 | 新增**模式无关的停止程序**（§4.3）：up → 等 1s → 仍在收音则补一次停止拍（down → 120ms → up，**只在仍在收音时才发**）→ 再等 1.5s → 仍不停则 `stopFailed` 并中止整轮 + 提示人工停止。阶段 1 / 阶段 2 状态清零 / 退出清理**共用**该程序；发出停止拍后**必须再验证一次** |
| 66 | **P0** | **§5.6 第 1 行把「冷快路径超时」判成无效测量** → Chromium 永远无结论，R4 永不触发 | 无效条件加严：`fastPathSetupVerified = false` **或**（`coldFastPathResolved = false` **且** `postColdQueryResolved = false`）。冷超时但 3s 查询能解析 → **正常测量**，继续走稳态判定（§5.6） |
| 67 | P1 | **快路径总耗时没测**——生产快路径 = 查询 + 读 role/subrole/description/title/help/placeholder 做安全过滤，只测查询会**低估**真实成本 | 新增 `coldFastPathTotalMs` / `warmFastPathTotalMedianMs` / `warmFastPathTotalP95`，**对着 300ms 预算用总耗时判定**（§5.3.2 / §5.5 / §5.6） |
| 68 | P1 | **物理输入中止监听会把自己注入的事件当成用户输入** → 第一次注入即自杀；且只订阅 `.keyDown` 会漏检用户按住修饰键 | 三条硬性要求：① 忽略带 `eventSourceUserData` 标记的自家事件（新增 **F-41**：`kCGEventSourceUserData = 42`）；② 监听集合含 `.flagsChanged`（新增 **F-43**：`NX_FLAGSCHANGED = 12`）；③ 只在检测到非自家事件时中止（§6.1） |
| 69 | P1 | **Chrome 的「autofocus 未生效则按 Tab」不可观测**——判断要查 AX（污染冷测量），而 autofocus 已生效时多按 Tab 反而移走焦点 | `--chrome-setup=autofocus|tab` **显式二选一，一次运行只试一种**；失败走 `noEditableFocus` 重测，不在同一次运行里二次猜测（§5.3.1 / §5.7） |
| 70 | P1 | **Chrome 页面没有加载等待**——`open` 返回后页面可能还没解析完，焦点在 body → 被当成「快路径无效」 | 新增 `--page-load-wait`（默认 2s，记录实际值），放在激活之后、准备焦点之前（§5.3.2 / §5.7） |
| 71 | P1 | **Slack 固定坐标点底边中央有副作用风险**——可能点中发送按钮 / 附件 / 消息 | 改为**默认人工点击**：提示用户点进 composer 后回车继续（零 AX、零误点）；自动点击降为可选 `--slack-setup=autoclick`，需坐标合法性校验与失败回退（§5.3.1 / §5.7） |
| 72 | P1 | **`--fresh-launch` 语义重载**（测冷全路径 vs 为二次确认重启 App） | 拆成 `--measure-full-cold` 与 `--relaunch-app`；后者还需 `--confirm-relaunch` 显式确认（重启 Chrome 可能丢未保存表单）（§5.6 / §5.7） |
| 73 | P1 | **`manualAccessibilityBefore/After 必须相等` 缺归一化**——TextEdit 上该属性 unsupported，Chrome 上初始可能 false 或缺失 | 归一化规则：`nil`（属性不存在/读回 error）与 `false`（未宣告）**视为等价**，都记 `notDeclared`；只有 `declared` vs `notDeclared` 才算恢复失败（§5.5） |
| 74 | P1 | 判定表第 2 行需限定手势；步骤 C 对双击手势应是「再双击一次」 | 归类决策程序内「自启自停」分支按 `G ∈ {double120, double300}` 细分标签；步骤 C 注明双击手势应「再双击一次」（§4.3） |
| 75 | P1 | **冷测量可能被其它 AX 客户端预热**——Chrome 的树被 VoiceOver / 上次探针运行碰过就保持建好 | 记录 `appStartedAt` / `coldMeasuredAt` / `coldWithinFreshWindow`（60s 窗口）与 `mayHaveBeenDeclaredBefore`；`coldStale` 时冷指标不参与判定（§5.3.2 / §5.5 / R-21） |
| 76 | P2 | **F-36 数量仍错**——实测 `enabled && selectCapable` 是 **8 个**，不是 5 个 | 改为 8 个并列出全部；§6.3 与 R-03 同步（**本轮实测复核确认：8 个，含 AinuIM.Ainu 与两种 Kotoeri**） |
| 77 | P2 | F 表顺序乱了（F-40 在 §1.1、F-38/F-39 在 §1.4） | 新增 **§1.0 事实索引表**（按小节给出编号范围与摘要），并注明「引用一律写 `F-xx`」 |
| 78 | P2 | `open -n -a TextEdit` 拿不到新实例的 pid | 改用 **`NSWorkspace.openApplicationAtURL:configuration:completionHandler:`**（新增 **F-42**）直接拿 `NSRunningApplication`；结束时**按 pid 关闭自己开的实例** |
| 79 | P2 | §5.5 只列了 `fastPathDescription`，但三重判定要查四个属性 | 补齐 `fastPathTitle` / `fastPathHelp` / `fastPathPlaceholder`（§5.5） |
| 80 | P2 | §5.3.2 用完 3s 超时后进入 warm 前要**显式调回 0.3s** | 超时阶梯增加一行说明「测完立即调回 0.3s」，并在测量顺序里显式写明（§5.3.2 / §5.4） |
| 81 | P2 | `AXEnhancedUserInterface` 在 Electron 上会扭曲窗口 | 恢复后用 `CGWindowListCopyWindowInfo` 比对窗口几何 → 新增指标 `windowGeometryChanged`；被改动则提示重启 Slack（§5.3.2 / §5.5 / R-22） |
| 82 | P2 | 快速预览在 §4.3 流程里没有落点 | 新增 **阶段 0.5 · 快速预览**，明确顺序「预览 → 矩阵 → **以矩阵为准**」，并规定两段都跑（§4.0 / §4.3） |
| 83 | P2 | `--declared-key` 与 `--declared-mode` 只给一个时的行为未定义 | **必须成对**，只给其一 → 启动即报错退出（§4.3 阶段 0 / §4.5 / §8.1） |

### v1.4 → v1.5

> **本轮来源**：对 v1.4 做了一次**独立一致性审计**（3 个 P0 / 14 个 P1 / 19 个 P2，判定 FAIL）。
> 三个 P0 **都会直接改变探针产出**，其中两个会让 G1-b 对 R4 给出**方向相反**的结论。

| # | 级别 | 问题 | 处置 |
| --- | --- | --- | --- |
| 84 | **P0** | **停止拍的驻留时长与上游 §7.2 冲突**——本文 §4.1 自己写着「停止拍是 down → **立即 up**」，但 §4.3 停止程序第 2 步写的是 `down → tapDurationMs(120ms) → up`。上游用**驻留时长**区分开始拍与停止拍；停止拍若也驻留 120ms，IM 极可能当成一次**开始拍** → 残留会话 → `voiceState` 不回 false → `stopFailed` → **整轮中止，探针产不出任何结论** | 停止拍改为 **`down → 立即 up`（驻留 0ms）**；`tapDurationMs` 只用于**开始拍**（`tap120` 手势）。§8.1 增「停止拍的驻留时长」断言；同步 §10 #65（§4.3） |
| 85 | **P0** | **稳态快路径测在「树还没建」的时候**——稳态排在步骤 7，而唯一会建立 Chromium/Electron Web 内容树的动作（全路径宣告）在步骤 8。Chromium 的树**只有被宣告过才有内容**（上游 §5.2），在空树上测稳态 → 解析率 0/10 → 探针判「快路径无效」→ **触发 R4 降级**，而上游的真实情况是「首次宣告一次后快路径就会命中」——**结论方向完全相反** | **重排测量顺序**：全路径（步骤 8）提到稳态（步骤 9）之前；新增指标 `warmMeasuredAfterDeclaration`；§5.6 增「稳态判定的前提是树已建」（§5.3.2） |
| 86 | **P0** | **Slack 的冷快路径量到的是终端**——Slack 的人工点击流程以「回到终端按回车」结束，而 `activate` 发生在准备动作**之前**，于是冷测量时 frontmost 是**终端**。终端的聚焦元素也是 `AXTextArea` → 通过 §5.2 白名单 → 探针把**终端**的耗时与可编辑性记在 Slack 名下 → **假阳性**，正是 v1.3 #44 声称已消除的错误类型 | §5.3.2 新增**第 4 步**：准备焦点后**重新激活目标 App → 等 0.5s → 校验 frontmost 的 pid 与目标窗口**；不通过 → 记 `noEditableFocus`（准备失败类）。校验用 `NSWorkspace`，**不产生 AX 触碰**，不污染冷值（§5.3.2） |
| 87 | P1 | §5.6 判读表用 median，而同节紧邻的理由写明「**不能用 median**（超时样本恰好落在 ~300ms，会把一半超时判成通过）」→ 判定口径自相矛盾 | 判定统一改用 **`warmFastPathTotalP95`**（原生 150ms / Chromium-Electron 300ms）；median 仍记录但不参与判定；§5.5 补全 p95 指标名（§5.6） |
| 88 | P1 | **`coldStale` 与有效性判定冲突**——冷指标被判 stale 就「不参与任何判定」，但 §5.6 的有效性判定又依赖冷指标 → 落入未定义 | §5.6 增 stale 规则：冷测量失效 → **改用稳态指标**判定；稳态也不达标 → 记 **`noEditableFocus`（需重测）**，**不触发 R4**（降级会砍功能，宁可标需重测） |
| 89 | P1 | **原生类不达标时没有判定行**——上游 §13.1 问题①（原生 App 上快路径值不值得开自动聚焦）没有否定答案；且 150ms 门槛比上游 300ms 预算更严却没给依据 | §5.6 增「**原生类不达标 → 原生类自动聚焦默认值同样翻转为关闭**」；补 150ms 门槛的来源（上游预期原生 ~50ms、预算 ≤300ms，收紧是为了有区分度） |
| 90 | P1 | **选键规则第 3 级使第 4 级的「2× 延迟」例外不可达**——规则按序执行时，延迟规则已选出唯一赢家，赢家就是最低的，2× 关系恒不成立 | **重排**：第 3 步改为「**停止语义定档**（优先长按，例外：长按 > 2× 点按才改选点按）」，第 4 步改为「**同档内延迟最低者优先**」；例外分支恢复可达，§8.1 的边界用例测得到真东西（§4.4） |
| 91 | P1 | **`gDuring` 对双击手势没有定义**——「按压窗口」可读成单次驻留（50ms → n/a）或整个手势跨度（`double120` = 220ms ≥ 200ms → true）。按跨度读，`double120` 命中且松开即停会被判成【长按模式】，与专为双击准备的【双击即启即停】标签矛盾 | 钉死为「**单次按压的最长驻留**」，并给出四类手势的映射表（`tap120` / `double*` → n/a；`hold*` → true/false）；§8.1 增「双击手势永不被归类为长按模式」的断言（§4.3） |
| 92 | P1 | **「可疑命中，需复测」没有落点**——阶段 1 的判定只有「翻转 / 没翻转」两态，而阶段 2 只处理「命中手势」→ 可疑组合**永远不会被复测**，最终被当作未命中，违反「不直接判未命中」 | 阶段 1 改为**三态**（命中 / 可疑命中 / 未命中）；**阶段 2 的输入集合 = 命中 ∪ 可疑命中**（§4.3） |
| 93 | P1 | **停止程序自称「模式无关」，但 §8.1 要求「长按模式下不得补停止拍」**——模式无关就拿不到模式输入，单测断言没有可判定依据 | 停止程序**按模式分叉**：长按模式下 `up` 后仍为 true → **不发停止拍**，直接 `stopFailed`（并中止整轮）；点按 / 未定论模式才补停止拍。§4.3 标题与伪码同步（§4.3） |
| 94 | P1 | **全路径前没有把 AX 超时抬回 3.0s**——步骤 6 结束时超时是 0.3s，步骤 8 却按 3.0s 预算执行，但步骤列表里没有抬升这一步 → 每次 AX 调用被截到 300ms → `fullPathHit` 假阴性 + `fullPathWarmMs` 失真 | 步骤 8 开头显式写「**先把超时抬到 3.0s**」（§5.3.2 / §5.4） |
| 95 | P1 | **S5 能否逐组归因有两套互斥判据**（§4.2 用「落盘延迟 ≤ 1s」，§4.3 阶段 -1 用「S5 通过」）→ 同一个决策由两处不同规则决定 | 统一为一条：**S5 可逐组归因 = 正对照中 S5 通过 且 落盘延迟 ≤ 1s**；「正对照完全通过」**不等于**「S5 可逐组归因」（§4.2 / §4.3） |
| 96 | P1 | **`--now` / `--env` 不在参数表里**——它们是「同一输入 → 同一份 JSON 字节」断言的前提，不进表就没人知道它们存在 | 补进 §4.5 与 §5.7 参数表，标「**仅测试用**」 |
| 97 | P1 | **6 个参数在正文流程中没有落点**；`--abort-on-voice-stuck` 的 off 行为未定义；`--sample-ms` 改了采样周期却没说它连带影响去抖窗口与延迟分辨率 | §4.5 / §5.7 补落点：`--only` / `--matrix-only` / `--max-groups` / `--skip-positive-control` 标「**仅调试用**」；`--sample-ms` 注明它同时决定去抖窗口与延迟分辨率；`--abort-on-voice-stuck` 注明实现口径（默认不中止，用 `--no-abort-on-voice-stuck` 显式关） |
| 98 | P1 | **`openApplicationAtURL:` 的参数没写清**——它收的是**应用 bundle URL**，不是文档 URL；且没有任何一步真正「打开临时文件」 | §5.3.1 改为**三步**：① 生成临时文件；② 用 **TextEdit.app 的 bundle URL** 启动新实例拿 pid；③ **`NSWorkspace.open(临时文件 URL)`** 打开文档 |
| 99 | P1 | **`--slack-setup=autoclick` 的坐标校验不可实现**——「不落在任何子按钮区域」需要控件级数据，而探针不申请屏幕录制权限（窗口名全空，F-24）且冷测量前不碰 AX | 降级为**可实现的粗规则**：只允许落在窗口下部 1/4 且排除最右侧 15% 水平区域；**如实声明无法保证不误点**，失败一律回退人工点击，默认仍为 `manual`（§5.3.1） |
| 100 | P1 | **人工抽检（§2.3 / §8.3 第 6 条）没有执行落点**——阶段序列里没有这一步；且「按一次」是探针注入还是用户手动按键没定义 | 新增 **阶段 3 · 人工抽检**：**由探针注入**命中组合（用户只说话、不碰键盘），观察窗等用户确认上屏；结果记入 §7.2 §1（§4.3） |
| 101 | P2 | §7.2 的结论文档结构**没有「清理状态」栏**，但 §4.3 / §6.2 / §6.3 多处要求「写入报告的清理状态栏」 | §7.2 增 **§5 清理状态**节（无卡键 / 未在收音 / 输入源已恢复 / 自开实例已关闭 + 失败项） |
| 102 | P2 | §5.5 缺 `coldStale` 指标（它被反复用作判定字段）、缺 `warmMeasuredAfterDeclaration` | §5.5 补 `warmMeasuredAfterDeclaration`；`coldStale` 由 `coldWithinFreshWindow` 表达（`coldStale = !coldWithinFreshWindow`），不另设字段 |
| 103 | P2 | `warmFastPathEditableRate` 的分母口径与判读表「= 10/10」冲突 | 改为 `warmFastPathEditableCount`，**以样本数为分母**；判读要求「解析 10/10 **且** 可编辑 10/10」 |
| 104 | P2 | §7.3 回填清单写「§13.1 M0 出口」，但 M0 出口标准在 **§13.2** | 改为 §13.2 |
| 105 | P2 | `--out` 默认值是仓库根相对路径，而 §8.4 的运行入口在 `tools/probes` 下执行 → 默认会落到 `tools/probes/docs/verification/` | §4.5 / §5.7 注明**路径相对于当前工作目录**，`run-all.sh` 必须传显式路径 |
| 106 | P2 | §8.1 把 `--sample-ms` 举为「都是合法的 120」的例子，但它的默认值是 **50** | 改举 `doubleClickGapMs = 120ms` 为例（§8.1） |
| 107 | P2 | 时间预算口径不一：§4.0 / §4.3 写「≈ 7–10 分钟」，§6.1 横幅写「约 8–10 分钟」 | 统一为 **8–10 分钟**（§4.0 / §4.3 / §6.1） |
| 108 | P2 | §6.5 把「Slack 的人工点击步骤」列在 G1-a 的在场提示里（该步骤属 G1-b）；且「回终端按回车」与 G1-a 的「请勿触碰键盘」冲突 | §6.5 改为：该步骤属 **G1-b**，是独立在场提示；G1-a 的「请勿触碰键盘」只管 G1-a 的键盘注入阶段（§6.5 / §5.3.1） |
| 109 | P2 | §4.4 写「全部 **50** 组可执行组」，与 §4.1 / §8.3 的「50（**或 55**）」冲突 | 改为「全部**可执行组**（50 组；spike 成功后为 55 组）」（§4.4） |
| 110 | P2 | §1.0 索引说明举的例子不能说明自己的论断；且写「因**三轮**评审」，实际已过四轮 | 改写索引说明（只讲「编号与出现顺序不单调」，引用一律写 `F-xx`）；改为「四轮」（§1.0） |
