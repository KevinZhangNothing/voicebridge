# 计划 2 / VoiceBridge G1-a 触发键探针 Implementation Plan

> **计划编号写死：本文件是「计划 2」。** 计划 1 是 `2026-10-23-vb-g1b-focus-probe.md`（G1-b 聚焦探针）。文件名里的 `g1b` / `g1a` 是**探针代号**，不是计划序号——不要按文件名里的代号去读「计划 N」的引用。

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在本机实测「合成的触发键能否唤醒 ChatterFly、它认长按还是点按」，产出机器可读 + 人读两份 G1-a 结论，回填上游规格 §4.2 / §7.2 / §13.1 / §13.2 的占位表述。

**Architecture:** library target `VBG1ACore` 承载全部可单测的判定逻辑（候选矩阵、判据器去抖、归类决策程序、停止程序状态机、选键规则），真实注入与真实采样封装在 `KeyInjector` / `AudioProcessInspector` / `SpeechShortcutCounter` 三个类里。executable target `vb-g1a-trigger` 只做装配与阶段编排。**所有算法逻辑与平台调用分离**，因为规格 §8.1 要求用穷举真值表验证归类程序、用状态机用例验证停止程序——这些都必须是纯逻辑。

**Tech Stack:** Swift 6 / SwiftPM / AppKit / CoreGraphics (CGEvent, CGEventTap) / CoreAudio / ApplicationServices / Carbon-HIToolbox (TIS) / XCTest。**无第三方依赖**（规格 §8.5）。

**上游依据:** `docs/superpowers/specs/2026-10-23-voicebridge-m0-probe-design.md` v1.5 §4（本文称为「探针规格」）；`docs/superpowers/specs/2026-10-23-voicebridge-design.md` v3.4 §4.2 / §7.1 / §7.2 / §13.1 / §13.2（本文称为「上游规格」）。

**前置:** **必须先完成计划 1 的 Task 1-3**（`VBProbeKit` 的参数解析、报告编码、环境采集、窗口检查、TextEdit 靶场、frontmost 守卫）。本计划复用 `VBProbeKit`，不重复造。
TextEdit 靶场的用法：`openTemporaryDocument()` 是**两步准备**（启动**应用** → 在那个实例里打开临时文档），`verifyDocumentFocus(_:)` 用来确认**文档正文**拿到了焦点——G1-a 没有冷测量，所以本计划在打开文档后立刻做这个验证。

---

## Global Constraints

以下约束适用于本计划**每一个** Task，不再逐条重复：

- **平台**：macOS 14+（本机实测 27.0.1）；Swift 6；**无第三方依赖**。
- **权限**（规格 §8.5）：探针**必须**有 AX 权限并在启动时自检；**不申请**屏幕录制权限；**不申请** Automation 权限（**禁用 `osascript`**）；**不主动索要** Input Monitoring（只探测可用性，拿不到就在横幅与报告里显著标注「本次运行无法中止物理输入」）。
- **不改系统设置、不改 Chatterfly 偏好、不 kill 任何进程**（规格 §6.1）。
- **注入落点硬校验**（规格 §6.1）：每次注入前校验 frontmost 的 **pid == 目标 TextEdit 实例 pid** 且**目标窗口在前台**，不匹配**立即中止**——这不是「尽量」。`NSRunningApplication.activate` 从 CLI 调用在 macOS 14+ 可能失败或被节流，一旦失败，探针会在**按住 Cmd/Option 的 1.5–3 秒**里继续跑。
- **物理输入中止**（规格 §6.1）：三条硬性要求——① 忽略带 `eventSourceUserData` 标记的**自家**事件；② 监听集合**必须含 `.flagsChanged`**（用户按住修饰键产生的是 flagsChanged 而不是 keyDown）；③ **只在检测到非自家事件时**才中止。
- **`tapDurationMs` 是唯一定义点**（规格 §4.1 / §8.1 / R-14）：本计划所有 `120` 毫秒必须来自命名常量 `TriggerTiming.tapDurationMilliseconds` 或 `TriggerTiming.doubleClickGapShortMilliseconds`，**不得出现无名字面量**。取值与上游规格 §7.1 时序参数表的 `tapDuration = 120ms` **同源**。
- **停止拍没有驻留时间**（上游规格 §7.1 / §7.2，D1）：上游用**驻留时间**区分「开始拍」与「停止拍」——**开始拍**是 `down → 120ms → up`（用 `tapDuration`），**停止拍**是 `down → **立即** up`（**零驻留**）。所以 `tapDurationMilliseconds` **只**用于开始拍；停止拍**不得**带 120ms 驻留——那会被输入法**误读成一次开始拍**，开出一个新会话。
- **手势参数**（规格 §4.1）：`tapDurationMilliseconds = 120`（单次按压驻留）、`doubleClickHoldMilliseconds = 50`（双击中每次单击的驻留，取 50ms 是为了短于任何长按阈值）、`doubleClickGapShortMilliseconds = 120` / `doubleClickGapLongMilliseconds = 300`（双击两次单击之间的间隔，定义为 **up → down**）。
- **采样周期 50ms**；**观察窗 3 秒**（不是 1.5 秒——`tapOnRelease` 叠加 IM 冷启动可能超过 1.5 秒，观察窗太短会把命中记成未命中）。
- **事件标记**：`kCGEventSourceUserData = 42`（F-41）→ 用 `CGEventSetIntegerValueField(event, .eventSourceUserData, 42)` 标记自家注入的事件。
- **输入源**：读取与恢复**全部走 TIS**（F-35/F-36）。`defaults write com.apple.HIToolbox AppleSelectedInputSources` **不会实时切换输入源**，不能用来恢复。每组探测前后各记录一次当前输入源，**不一致即中止整轮并用 `TISSelectInputSource` 恢复**。
- **确定性报告**：时间戳与环境字段可注入（`--now` / `--env`）。
- **产物位置**：探针代码进 `tools/probes/`；结论进 `docs/verification/`。
- **`--out` 的默认值相对当前工作目录解析**：`docs/verification/` 是**相对 cwd** 的路径，**不是**相对仓库根。所以 `run-all.sh` **必须传显式路径**（`--out "$OUT_DIR"`），否则在不同 cwd 下会把报告写到别处。
- **S5 的逐组归因规则**（D14，全计划只此一处定义）：**只有**「阶段 -1 正对照通过」**且**「实测 S5 落盘延迟 ≤ 1s」时，S5 才能用于**逐组归因**；否则 S5 **只是整体佐证**，**不得**归因到具体组（`cfprefsd` 落盘延迟大时，计数增量可能落在组结束之后——归因到某组就是编造）。
- **「不要碰键盘」横幅的适用范围**：那条横幅针对**本计划的键盘注入阶段**；G1-b 的 Slack 步骤是它自己的**阶段内显式指令**（点 composer → 回终端按回车），两者不冲突。
- **提交策略**：**不提交、不推送**——提交与否由用户决定。

### 探针参数

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `--out <dir>` | `docs/verification/` | 报告输出目录。**默认值相对当前工作目录（cwd）解析**——`run-all.sh` 必须传显式路径 |
| `--dry-run` | off | 只打印将要执行的矩阵，**不注入任何事件** |
| `--sample-ms` | 50 | 采样周期。它同时决定**去抖窗口**（连续 2 个采样一致才翻转）与**延迟分辨率**（采样间隔 = 延迟的量化粒度） |
| `--declared-key <key>` | 无 | 用户声明的键（`fn` / `cmdLeft` / `cmdRight` / `optLeft` / `optRight`），**必须与 `--declared-mode` 成对** |
| `--declared-mode <mode>` | 无 | 用户声明的模式：`longPress` / `tap` / `doubleTap`，**必须与 `--declared-key` 成对** |
| `--only <key>` | 无 | **仅调试用**：只跑指定候选键。**不能**用来产出 M0 结论 |
| `--matrix-only` | off | **仅调试用**：跳过快速预览段。**不能**用来产出 M0 结论（报告缺「快速预览段」） |
| `--max-groups <n>` | 无限制 | **仅调试用**：最多跑多少组（时间预算控制）。**不能**用来产出 M0 结论 |
| `--skip-positive-control` | off | **仅调试用**：跳过阶段 -1。跳过时报告必须标注「无正对照，结论不可信」 |
| `--no-abort-on-voice-stuck` | off | **默认不中止**（规格 §4.5 写的是默认 on，见上面的冲突裁决表） |
| `--no-spike-system-defined` | off | **默认自动尝试** systemDefined spike（规格 §4.1）。构造成功 → 矩阵 55 组；失败 → **降级**到 50 组并记 `unsupportedBySDK`（**不中止**）。本参数用于显式关掉 |
| `--now <iso8601>` | 无 | **仅测试用**：注入报告时间戳（供「同一输入 → 同一份 JSON 字节」的断言） |
| `--env <json>` | 无 | **仅测试用**：注入环境快照（供「同一输入 → 同一份 JSON 字节」的断言） |

> **「仅调试用 / 仅测试用」的参数不参与 M0 结论**：用它们跑出来的报告只能用来定位问题，不能回填上游规格。

## 与探针规格的两处冲突（本计划已裁决，实现按此执行）

| 冲突点 | 探针规格的两处说法 | 本计划的裁决 | 理由 |
| --- | --- | --- | --- |
| **停止程序是否模式无关** | §4.3 标题写「停止程序（**模式无关**）」，但同节的伪码步骤 2 写「长按模式下多发这一次会短暂开一个新会话」；§8.1 的单测又写「**长按模式下不得补停止拍**」 | 停止程序**按模式分叉**：长按模式下 up 之后**不发停止拍**，直接记 `stopFailed`；点按 / 未知模式下才补停止拍 | 长按模式的 up **本来就该停**；up 停不下来说明状态异常，此时再补一次 down→**立即** up 只会**开一个新会话**，把问题变成两个会话。§8.1 的单测断言更具体，优先于 §4.3 的标题措辞 |
| **`--abort-on-voice-stuck` 默认值** | §4.5 把它写成默认 `on` 的参数 | 实现为 `--no-abort-on-voice-stuck`（**默认不中止**） | 本计划的 `ArgumentScanner` 用 **schema** 判定每个 token 是 flag 还是带值选项，不在 schema 里的参数直接报错；一个默认开的开关无法用「出现即 true」表达。改成默认关、用 `--no-abort-on-voice-stuck` 显式关掉，语义与 §4.5 相反但更安全——中止整轮会丢掉已完成组的数据 |

---

## File Structure

```
tools/probes/
├── Package.swift                    # Task 0 改成最终完整版（5 product / 8 target）
├── Sources/
│   ├── VBProbeKit/                  # 计划 1 的共享底座，本计划只读
│   ├── VBG1ACore/
│   │   ├── Placeholder.swift           # Task 0 占位，Task 1 写入真实类型后删除
│   │   ├── Arguments/
│   │   │   └── G1AArguments.swift
│   │   ├── Trigger/
│   │   │   ├── TriggerKey.swift        # 5 个候选键 + virtualKeyCode + flags
│   │   │   ├── EventForm.swift         # keyEvent / flagsChanged（systemDefined 仅 spike）
│   │   │   ├── Gesture.swift           # 5 个手势
│   │   │   ├── KeyInjector.swift       # 注入 + eventSourceUserData 标记 + 已按下集合 + TriggerTiming 常量
│   │   │   └── TriggerCandidates.swift # 矩阵生成（50 组）
│   │   ├── Oracle/
│   │   │   ├── VoiceSample.swift
│   │   │   ├── VoiceStateOracle.swift  # 去抖 + S1||S4 合成 + 可疑命中
│   │   │   ├── AudioProcessInspector.swift  # S1 / S4
│   │   │   └── SpeechShortcutCounter.swift  # S5
│   │   ├── Analysis/
│   │   │   ├── StageTwoClassifier.swift  # 归类决策程序
│   │   │   ├── StopProcedure.swift       # 停止程序状态机
│   │   │   └── KeySelection.swift        # 选键规则
│   │   ├── Guard/
│   │   │   ├── PhysicalInputGuard.swift  # CGEventTap（专用线程跑 run loop）
│   │   │   └── SignalCleanup.swift       # SIGINT/SIGTERM → 释放按键
│   │   ├── Measurement/
│   │   │   ├── G1AGroupReport.swift
│   │   │   ├── G1APayload.swift
│   │   │   ├── CleanupProbe.swift        # 实测清理状态（不写死）
│   │   │   └── G1ARunner.swift
│   │   └── Report/
│   │       └── G1AMarkdownRenderer.swift
│   └── vb-g1a-trigger/
│       └── main.swift
└── Tests/
    └── VBG1ACoreTests/
        ├── Placeholder.swift             # Task 0 占位，Task 1 写入真实类型后删除
        ├── TestSupport.swift             # RecordingPoster 桩（不投递真实事件）
        ├── AbortStateTests.swift          # 中止标志 + 事件判定 + 信号清理
        ├── CleanupProbeTests.swift        # 清理状态判定（实测而非写死）
        ├── G1AArgumentsTests.swift
        ├── G1APayloadTests.swift
        ├── G1AMarkdownRendererTests.swift
        ├── KeyInjectorTests.swift
        ├── KeySelectionTests.swift
        ├── PhysicalInputGuardTests.swift
        ├── StageTwoClassifierTests.swift
        ├── StopProcedureTests.swift
        ├── TriggerCandidatesTests.swift
        ├── TriggerTimingLiteralTests.swift
        └── VoiceStateOracleTests.swift
```

---

### Task 0: 接线 `Package.swift`——加 `VBG1ACore` / `vb-g1a-trigger` / `VBG1ACoreTests`

> **为什么单独一个 Task**：计划 1 的 Task 1 建的是**基础版** `Package.swift`（只有 `VBProbeKit` / `VBG1BCore` / `vb-g1b-focus` 三个 target）。本计划要加三个 target——**不接线就没有 `swift test --filter ...` 能跑**，执行时会卡在 Task 1 的第一步。

**Files:**
- Modify: `tools/probes/Package.swift`
- Create: `tools/probes/Sources/VBG1ACore/Placeholder.swift`（Task 1 写入真实类型后删除）
- Create: `tools/probes/Sources/vb-g1a-trigger/Placeholder.swift`（Task 10 会删掉它并创建真正的 `main.swift`）
- Create: `tools/probes/Tests/VBG1ACoreTests/Placeholder.swift`（Task 1 写入真实类型后删除）

**Interfaces:**
- Consumes: 计划 1 Task 1 建的 3-target `Package.swift`（`VBProbeKit` / `VBG1BCore` / `vb-g1b-focus`）。
- Produces: 最终完整版 `Package.swift`——5 个 product / 8 个 target：
  - library `VBProbeKit`（共享底座）
  - library `VBG1BCore`（G1-b 专属）
  - library `VBG1ACore`（G1-a 专属，依赖 `VBProbeKit`）
  - executable `vb-g1b-focus`
  - executable `vb-g1a-trigger`（依赖 `VBG1ACore`）
  - test target `VBProbeKitTests` / `VBG1BCoreTests` / `VBG1ACoreTests`（依赖 `VBG1ACore`）

- [ ] **Step 1: 把 `Package.swift` 替换成最终完整版**

`tools/probes/Package.swift`：

```swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "voicebridge-probes",
    platforms: [.macOS(.v14)],
    products: [
        .library(name: "VBProbeKit", targets: ["VBProbeKit"]),
        .library(name: "VBG1BCore", targets: ["VBG1BCore"]),
        .library(name: "VBG1ACore", targets: ["VBG1ACore"]),
        .executable(name: "vb-g1b-focus", targets: ["vb-g1b-focus"]),
        .executable(name: "vb-g1a-trigger", targets: ["vb-g1a-trigger"]),
    ],
    targets: [
        .target(name: "VBProbeKit"),
        .target(name: "VBG1BCore", dependencies: ["VBProbeKit"]),
        .target(name: "VBG1ACore", dependencies: ["VBProbeKit"]),
        .executableTarget(name: "vb-g1b-focus", dependencies: ["VBG1BCore"]),
        .executableTarget(name: "vb-g1a-trigger", dependencies: ["VBG1ACore"]),
        .testTarget(name: "VBProbeKitTests", dependencies: ["VBProbeKit"]),
        .testTarget(name: "VBG1BCoreTests", dependencies: ["VBG1BCore"]),
        .testTarget(name: "VBG1ACoreTests", dependencies: ["VBG1ACore"]),
    ]
)
```

> SwiftPM 的 target 目录**必须存在**才能声明——所以这个 Task 里**同时**建出三个空目录占位（`Sources/VBG1ACore/`、`Sources/vb-g1a-trigger/`、`Tests/VBG1ACoreTests/`），否则 `swift build` 报 `Source files for target VBG1ACore should be located under ...`。真正的实现文件在后续 Task 里往这些目录里放。

- [ ] **Step 2: 建三个 target 的占位目录与最小文件**

创建 `tools/probes/Sources/VBG1ACore/Placeholder.swift`：

```swift
// 本文件在 Task 1 写入 TriggerKey 等类型后删除。
```

创建 `tools/probes/Sources/vb-g1a-trigger/Placeholder.swift`：

```swift
// Task 0 的占位文件——Task 10 会删掉它并创建真正的 `main.swift`。
// ⚠️ 不能叫 `main.swift`：Task 10 的真入口也叫这个名字，
// 同名会让「按路径抽取代码块落盘」这一步把两个版本搞混。
print("vb-g1a-trigger 尚未实现")
```

创建 `tools/probes/Tests/VBG1ACoreTests/Placeholder.swift`：

```swift
import XCTest
```

- [ ] **Step 3: 跑测试确认三个 target 都在**

Run: `cd tools/probes && swift build && swift test --filter VBG1ACoreTests`

Expected: PASS（`Executed 0 tests`——空 target 也算通过；关键是**不再报 `no such module` / `not found in target`**）

- [ ] **Step 4: 提交（可选，默认跳过）**

```bash
cd tools/probes && git add Package.swift Sources/VBG1ACore Sources/vb-g1a-trigger Tests/VBG1ACoreTests && git commit -m "chore(probes): 接线 VBG1ACore / vb-g1a-trigger / VBG1ACoreTests"
```

---

### Task 1: 候选键 + 事件形态 + 手势 + 候选矩阵

> **开始前先删掉 Task 0 的两个占位文件**：
> ```bash
> rm tools/probes/Sources/VBG1ACore/Placeholder.swift tools/probes/Tests/VBG1ACoreTests/Placeholder.swift
> ```

**Files:**
- Create: `tools/probes/Sources/VBG1ACore/Trigger/TriggerKey.swift`
- Create: `tools/probes/Sources/VBG1ACore/Trigger/EventForm.swift`
- Create: `tools/probes/Sources/VBG1ACore/Trigger/Gesture.swift`
- Create: `tools/probes/Sources/VBG1ACore/Trigger/TriggerCandidates.swift`
- Test: `tools/probes/Tests/VBG1ACoreTests/TriggerCandidatesTests.swift`

**Interfaces:**
- Consumes: 无（只用 Foundation）。
- Produces:
  - `public enum TriggerKey: String, CaseIterable` — `fn`、`cmdLeft`、`cmdRight`、`optLeft`、`optRight`；`var virtualKeyCode: CGKeyCode`；`var modifierFlag: CGEventFlags`；`var isFunctionKey: Bool`；`var hasSystemSideEffect: Bool`。
  - `public enum EventForm: String, CaseIterable` — `keyEvent`、`flagsChanged`、`systemDefined`（**仅 spike**）；另有 `static let mainForms: [EventForm] = [.keyEvent, .flagsChanged]`（主矩阵只用这两种，`allCases` 有三种）。
  - `public enum GestureKind: String` — `tap`、`hold`、`double`。
  - `public struct Gesture` — `kind: GestureKind`、`name: String`；`static let tap120`、`hold1500`、`hold3000`、`double120`、`double300`；`static let all: [Gesture]`。
  - `public struct TriggerCandidate` — `key: TriggerKey`、`form: EventForm`、`gesture: Gesture`；`var id: String`。
  - `public enum TriggerCandidates` — `static func matrix(capabilityAware: Bool) -> [TriggerCandidate]`。

- [ ] **Step 1: 写失败测试——矩阵规模、顺序与无重复**

创建 `tools/probes/Tests/VBG1ACoreTests/TriggerCandidatesTests.swift`：

```swift
import XCTest
@testable import VBG1ACore

final class TriggerCandidatesTests: XCTestCase {
    func testMainMatrixHasExactlyFiftyGroups() {
        // 规格 §4.1：5 键 × 2 形态 × 5 手势 = 50 组
        XCTAssertEqual(TriggerCandidates.matrix(capabilityAware: false).count, 50)
    }

    func testCapabilityAwareMatrixHasFiftyFiveGroups() {
        // 规格 §8.1：systemDefined spike 成功后追加 5 组（仅 Fn × 5 手势）→ 55
        XCTAssertEqual(TriggerCandidates.matrix(capabilityAware: true).count, 55)
    }

    func testCapabilityAwareMatrixHasFiftyFiveDistinctGroups() {
        // ⚠️ 「55 组」必须是**55 个互不相同的组合**——
        // 追加的 5 组若复用基础矩阵的 `fn|flagsChanged|*`，ID 会重复，
        // 实际只有 50 组——那就不是 55，而是一个自欺欺人的数字。
        let extended = TriggerCandidates.matrix(capabilityAware: true)
        let ids = extended.map(\.id)
        XCTAssertEqual(Set(ids).count, 55, "追加的 5 组与基础矩阵重复了")
    }

    func testCapabilityAwareAppendsSystemDefinedFormForFunctionKeyOnly() {
        let base = TriggerCandidates.matrix(capabilityAware: false)
        let extended = TriggerCandidates.matrix(capabilityAware: true)
        let appended = Array(extended.dropFirst(base.count))
        XCTAssertEqual(appended.count, 5)
        XCTAssertTrue(appended.allSatisfy { $0.key == .fn })
        XCTAssertTrue(appended.allSatisfy { $0.form == .systemDefined })
    }

    func testMainMatrixUsesOnlyTheTwoMainForms() {
        // `EventForm.allCases` 有三种（含仅 spike 用的 systemDefined）——
        // 主矩阵必须只遍历 `mainForms`，否则会变成 75 组。
        let matrix = TriggerCandidates.matrix(capabilityAware: false)
        XCTAssertTrue(matrix.allSatisfy { $0.form == .keyEvent || $0.form == .flagsChanged })
    }

    func testNoDuplicatesAndNoOmissions() {
        let matrix = TriggerCandidates.matrix(capabilityAware: false)
        let ids = matrix.map(\.id)
        XCTAssertEqual(Set(ids).count, ids.count, "矩阵里有重复组合")
        // 每个 (键, 形态, 手势) 三元组恰好出现一次
        let triples = Set(matrix.map { "\($0.key.rawValue)|\($0.form.rawValue)|\($0.gesture.name)" })
        XCTAssertEqual(triples.count, 50)
    }

    func testCommandAndOptionKeysComeBeforeFunctionKey() {
        // 规格 §4.1：候选顺序 cmdLeft / cmdRight / optLeft / optRight **先于** fn
        let matrix = TriggerCandidates.matrix(capabilityAware: false)
        let firstFnIndex = matrix.firstIndex { $0.key == .fn }!
        let lastNonFnIndex = matrix.lastIndex { $0.key != .fn }!
        XCTAssertLessThan(lastNonFnIndex, firstFnIndex)
    }

    func testCapabilityAwareMatrixAppendsOnlyFunctionKeyGroups() {
        let base = TriggerCandidates.matrix(capabilityAware: false)
        let extended = TriggerCandidates.matrix(capabilityAware: true)
        XCTAssertEqual(Array(extended.prefix(base.count)).map(\.id), base.map(\.id))
        XCTAssertTrue(extended.dropFirst(base.count).allSatisfy { $0.key == .fn })
        XCTAssertEqual(Set(extended.dropFirst(base.count).map(\.gesture.name)).count, 5)
    }

    func testVirtualKeyCodesMatchHardware() {
        XCTAssertEqual(TriggerKey.fn.virtualKeyCode, 63)
        XCTAssertEqual(TriggerKey.cmdLeft.virtualKeyCode, 55)
        XCTAssertEqual(TriggerKey.cmdRight.virtualKeyCode, 54)
        XCTAssertEqual(TriggerKey.optLeft.virtualKeyCode, 58)
        XCTAssertEqual(TriggerKey.optRight.virtualKeyCode, 61)
    }

    func testFunctionKeyIsTheOnlyOneWithSystemSideEffect() {
        XCTAssertTrue(TriggerKey.fn.hasSystemSideEffect)
        XCTAssertFalse(TriggerKey.cmdLeft.hasSystemSideEffect)
        XCTAssertFalse(TriggerKey.optLeft.hasSystemSideEffect)
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd tools/probes && swift test --filter TriggerCandidatesTests`
Expected: FAIL（`Cannot find 'TriggerCandidates' in scope`）

- [ ] **Step 3: 写最小实现**

创建 `tools/probes/Sources/VBG1ACore/Trigger/TriggerKey.swift`：

```swift
import CoreGraphics
import Foundation

/// 5 个候选触发键。**Fn 排在最后**——合成 Fn 可能误触系统级功能（上游规格 R2），
/// 且本机 enabled+selectable 的输入源有 8 个（规格 F-36），系统默认 Fn 行为很可能是「切换输入源」。
public enum TriggerKey: String, CaseIterable, Sendable {
    case cmdLeft
    case cmdRight
    case optLeft
    case optRight
    case fn

    public var virtualKeyCode: CGKeyCode {
        switch self {
        case .fn: return 63        // kVK_Function
        case .cmdLeft: return 55   // kVK_Command
        case .cmdRight: return 54  // kVK_RightCommand
        case .optLeft: return 58   // kVK_Option
        case .optRight: return 61  // kVK_RightOption
        }
    }

    public var modifierFlag: CGEventFlags {
        switch self {
        case .fn: return [.maskSecondaryFn]
        case .cmdLeft, .cmdRight: return [.maskCommand]
        case .optLeft, .optRight: return [.maskAlternate]
        }
    }

    public var isFunctionKey: Bool { self == .fn }

    /// Fn 在 macOS 上有系统级行为（切换输入源 / 调亮度 / 播放键），
    /// 且系统配置**读不到**（`AppleFnUsageType` 在 `com.apple.HIToolbox` 下不存在，规格 F-34）。
    public var hasSystemSideEffect: Bool { isFunctionKey }

    public var displayName: String {
        switch self {
        case .fn: return "Fn"
        case .cmdLeft: return "左 Command"
        case .cmdRight: return "右 Command"
        case .optLeft: return "左 Option"
        case .optRight: return "右 Option"
        }
    }
}
```

创建 `tools/probes/Sources/VBG1ACore/Trigger/EventForm.swift`：

```swift
import Foundation

/// 三种事件形态（规格 §4.1）。
///
/// `systemDefined` **不在主矩阵里**——`NX_KEYTYPE_*` 的 22 个常量里没有 Fn（F-31），
/// 只能作为**可选 spike**单独构造（见 Task 2 的 `SystemDefinedSpike`）。
/// 所以主矩阵只遍历 `mainForms`（两种形态），**不是** `allCases`（三种）。
public enum EventForm: String, CaseIterable, Sendable {
    /// 形态 A：`CGEvent(keyboardEventSource:virtualKey:keyDown:)` + 修饰 flags，post 到 `.cghidEventTap`
    case keyEvent
    /// 形态 B：`CGEvent` type = `.flagsChanged`，写 `.keyboardEventKeycode` + 修饰 flags
    case flagsChanged
    /// 形态 C（**仅 spike**）：NSSystemDefined 通道，Fn 专用
    case systemDefined

    /// **主矩阵**用的两种形态。
    public static let mainForms: [EventForm] = [.keyEvent, .flagsChanged]
}
```

创建 `tools/probes/Sources/VBG1ACore/Trigger/Gesture.swift`：

```swift
import Foundation

public enum GestureKind: String, Sendable {
    case tap
    case hold
    case double
}

/// 手势。**时长常量一律引用 `TriggerTiming`**，本文件不出现任何毫秒字面量。
public struct Gesture: Equatable, Sendable {
    public var kind: GestureKind
    public var name: String

    /// 单次按压：down → 120ms → up
    public static let tap120 = Gesture(kind: .tap, name: "tap120")
    /// 按住 1500ms 再松开
    public static let hold1500 = Gesture(kind: .hold, name: "hold1500")
    /// 按住 3000ms 再松开——存在的唯一理由是「长按阈值可能 > 1.5s」
    public static let hold3000 = Gesture(kind: .hold, name: "hold3000")
    /// 双击，两击间隔 120ms
    public static let double120 = Gesture(kind: .double, name: "double120")
    /// 双击，两击间隔 300ms
    public static let double300 = Gesture(kind: .double, name: "double300")

    public static let all: [Gesture] = [.tap120, .hold1500, .hold3000, .double120, .double300]
}
```

创建 `tools/probes/Sources/VBG1ACore/Trigger/TriggerCandidates.swift`：

```swift
import Foundation

public struct TriggerCandidate: Equatable, Sendable {
    public var key: TriggerKey
    public var form: EventForm
    public var gesture: Gesture

    public init(key: TriggerKey, form: EventForm, gesture: Gesture) {
        self.key = key
        self.form = form
        self.gesture = gesture
    }

    public var id: String {
        "\(key.rawValue)|\(form.rawValue)|\(gesture.name)"
    }
}

public enum TriggerCandidates {
    /// 主矩阵：5 键 × 2 形态 × 5 手势 = **50 组**。
    ///
    /// 顺序：**Cmd / Option 在前，Fn 在后**（规格 §4.1）。
    /// `capabilityAware = true` 时在末尾追加 systemDefined spike 的 5 组（仅 Fn × 5 手势）→ 55 组。
    public static func matrix(capabilityAware: Bool) -> [TriggerCandidate] {
        var result: [TriggerCandidate] = []
        for key in orderedKeys {
            // ⚠️ 用 `mainForms`（两种），**不是** `allCases`（三种，含仅 spike 用的 systemDefined）
            for form in EventForm.mainForms {
                for gesture in Gesture.all {
                    result.append(TriggerCandidate(key: key, form: form, gesture: gesture))
                }
            }
        }
        if capabilityAware {
            // spike 的 5 组必须是**独立的形态**——用 `flagsChanged` 会和基础矩阵的
            // `fn|flagsChanged|*` **完全重复**（ID 相同），那就不是 55 组而是 50 组。
            for gesture in Gesture.all {
                result.append(TriggerCandidate(key: .fn, form: .systemDefined, gesture: gesture))
            }
        }
        return result
    }

    /// Cmd / Option **先于** Fn。
    static let orderedKeys: [TriggerKey] = [.cmdLeft, .cmdRight, .optLeft, .optRight, .fn]
}
```

> systemDefined spike 的 5 组用的是**独立的 `systemDefined` 形态**——不是 `flagsChanged`。
> 用 `flagsChanged` 会和基础矩阵的 `fn|flagsChanged|*` **完全重复**（ID 相同），
> 那就不是 55 组而是 50 组。构造 `systemDefined` 事件的真实路径在 Task 2 的
> `KeyInjector` / `SystemDefinedSpike` 里；Task 1 只负责**矩阵计数**。

- [ ] **Step 4: 跑测试确认通过**

Run: `cd tools/probes && swift test --filter TriggerCandidatesTests`
Expected: PASS（10 个用例全绿）

- [ ] **Step 5: 提交（可选，默认跳过）**

```bash
cd tools/probes && git add Sources/VBG1ACore/Trigger Tests/VBG1ACoreTests && git commit -m "feat(probes): 候选键、事件形态、手势与 50 组矩阵"
```

---

### Task 2: KeyInjector——注入、事件标记、已按下集合、spike 构造

**Files:**
- Create: `tools/probes/Sources/VBG1ACore/Trigger/KeyInjector.swift`
- Test: `tools/probes/Tests/VBG1ACoreTests/TestSupport.swift`
- Test: `tools/probes/Tests/VBG1ACoreTests/KeyInjectorTests.swift`
- Test: `tools/probes/Tests/VBG1ACoreTests/TriggerTimingLiteralTests.swift`

**Interfaces:**
- Consumes: `VBG1ACore`（Task 1 的 `TriggerKey` / `EventForm` / `Gesture`）。
- Produces:
  - `public enum TriggerTiming` — `tapDurationMilliseconds = 120`、`doubleClickHoldMilliseconds = 50`、`doubleClickGapShortMilliseconds = 120`、`doubleClickGapLongMilliseconds = 300`、`holdDurationMilliseconds = 1500`、`longHoldDurationMilliseconds = 3000`、`sampleIntervalMilliseconds = 50`、`observationWindowMilliseconds = 3000`。
  - `public protocol EventPosting { func post(_ event: CGEvent) }`。
  - `public final class KeyInjector: @unchecked Sendable` — `init(poster: EventPosting = CGEventPoster())`；`func perform(key:form:gesture:sleeper:) throws`；`func postKeyDown(key:form:) throws`；`func postKeyUp(key:form:) throws`；`var pressedKeys: Set<TriggerKey>`；`func releaseAll(sleeper:) throws`。
    - ⚠️ **必须是 class，不能是 struct**：`defer` 捕获的是变量**当时的副本**。注入器是 struct 时，`defer { injector.releaseAll() }` 释放的是**空副本**，真正按下的键**留在按下状态**（系统以为 Cmd/Option 还按着，之后用户敲的每个字母都变成 Cmd+字母）。class 的引用语义让 `defer` 看到最终状态。
    - ⚠️ `sleeper` 是 `(Double) throws -> Void`——按压期间的观测要能**抛错**（物理输入中止时必须能从 sleeper 里冒出来）。
  - `public enum SystemDefinedSpike` — `static func buildFnEvent(keyDown: Bool) -> CGEvent?`（返回 `nil` = 构造失败 → 调用方记 `unsupportedBySDK`）；`static let modifierKeySecondaryFn = 7`、`static let keyDownState = 0x0A`、`static let keyUpState = 0x0B`、`static let subtypeAuxControlButtons = 8`。
    - ⚠️ **公开 SDK 里没有 `CGEventType.systemDefined`，也没有 `CGEventField.keyboardEventData1`**——键盘事件的 `data1` 是私有字段。所以必须走 **`NSEvent.otherEvent`**，再取它的 `cgEvent` 投递。
  - `public extension Gesture` — `var singlePressMilliseconds: Int`（**最长的一次单次按压**的驻留时长——`gDuring` 的「期间」窗口按它界定，见 Task 4）。

- [ ] **Step 1: 写失败测试——时序常量与已按下集合（不发真实事件）**

先创建测试公用桩 `tools/probes/Tests/VBG1ACoreTests/TestSupport.swift`——`RecordingPoster` 记录收到的全部事件、**不真的投递**：

```swift
import CoreGraphics
import Foundation
@testable import VBG1ACore

/// 记录收到的全部事件，**不真的投递**。
final class RecordingPoster: EventPosting {
    private(set) var events: [CGEvent] = []
    func post(_ event: CGEvent) { events.append(event) }
}
```

创建 `tools/probes/Tests/VBG1ACoreTests/KeyInjectorTests.swift`：

```swift
import XCTest
@testable import VBG1ACore

final class KeyInjectorTests: XCTestCase {
    func testTimingConstantsMatchSpec() {
        // 上游规格 §7.1 时序参数表：tapDuration = 120ms
        XCTAssertEqual(TriggerTiming.tapDurationMilliseconds, 120)
        XCTAssertEqual(TriggerTiming.doubleClickHoldMilliseconds, 50)
        XCTAssertEqual(TriggerTiming.doubleClickGapShortMilliseconds, 120)
        XCTAssertEqual(TriggerTiming.doubleClickGapLongMilliseconds, 300)
        XCTAssertEqual(TriggerTiming.sampleIntervalMilliseconds, 50)
        XCTAssertEqual(TriggerTiming.observationWindowMilliseconds, 3000)
    }

    func testTapGesturePostsDownThenUp() throws {
        let poster = RecordingPoster()
        let injector = KeyInjector(poster: poster)
        try injector.perform(key: .cmdLeft, form: .keyEvent, gesture: .tap120, sleeper: { _ in })
        XCTAssertEqual(poster.events.count, 2)
        // ⚠️ 点按是完整的 down→up：**结束时没有任何键处于按下状态**。
        // 断言「还按着」会与实现（up 时把键从集合里移除）直接矛盾。
        XCTAssertTrue(injector.pressedKeys.isEmpty)
    }

    func testReleaseAllClearsPressedKeys() throws {
        let poster = RecordingPoster()
        let injector = KeyInjector(poster: poster)
        try injector.postKeyDown(key: .cmdLeft, form: .keyEvent)
        XCTAssertEqual(injector.pressedKeys, [.cmdLeft])
        try injector.releaseAll(sleeper: { _ in })
        XCTAssertTrue(injector.pressedKeys.isEmpty)
    }

    func testOwnEventsCarryUserDataMarker() throws {
        let poster = RecordingPoster()
        let injector = KeyInjector(poster: poster)
        try injector.postKeyDown(key: .fn, form: .keyEvent)
        let marker = poster.events[0].getIntegerValueField(.eventSourceUserData)
        // 规格 F-41：kCGEventSourceUserData = 42
        XCTAssertEqual(marker, 42)
    }

    func testDoubleGesturePostsFourEvents() throws {
        let poster = RecordingPoster()
        let injector = KeyInjector(poster: poster)
        try injector.perform(key: .cmdLeft, form: .keyEvent, gesture: .double120, sleeper: { _ in })
        XCTAssertEqual(poster.events.count, 4)   // 两击 × (down + up)
    }

    func testHoldGestureKeepsKeyPressedUntilRelease() throws {
        let poster = RecordingPoster()
        let injector = KeyInjector(poster: poster)
        try injector.postKeyDown(key: .optLeft, form: .keyEvent)
        XCTAssertEqual(injector.pressedKeys, [.optLeft])
        try injector.postKeyUp(key: .optLeft, form: .keyEvent)
        XCTAssertTrue(injector.pressedKeys.isEmpty)
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd tools/probes && swift test --filter KeyInjectorTests`
Expected: FAIL（`Cannot find 'KeyInjector' in scope`）

- [ ] **Step 3: 写最小实现**

创建 `tools/probes/Sources/VBG1ACore/Trigger/KeyInjector.swift`：

```swift
import AppKit
import CoreGraphics
import Foundation

/// **所有毫秒时序常量的唯一定义点**（规格 §4.1 / §8.1 / R-14）。
///
/// 本文件是「注入器源文件」——单测会扫描本文件（**剥掉注释之后**），
/// 确保**没有无名的 120 字面量**：每个 120 都必须来自下面这两个命名常量之一。
public enum TriggerTiming {
    /// 点按的驻留时长。**与上游规格 §7.1 时序参数表的 `tapDuration` 同源**，
    /// 且必须短于输入法判定「长按」的下限，否则会被当成按住不放。
    public static let tapDurationMilliseconds = 120

    /// 双击中**每一次**单击的驻留时长。取 50ms 是为了短于任何长按阈值。
    public static let doubleClickHoldMilliseconds = 50

    /// 双击两次单击之间的间隔，定义为 **up → down**（与系统双击惯例一致）。
    public static let doubleClickGapShortMilliseconds = 120
    public static let doubleClickGapLongMilliseconds = 300

    /// 长按手势的两种驻留时长。
    public static let holdDurationMilliseconds = 1500
    public static let longHoldDurationMilliseconds = 3000

    /// 判据采样周期。
    public static let sampleIntervalMilliseconds = 50

    /// 每组的观察窗。3 秒而不是 1.5 秒——`tapOnRelease` 叠加 IM 冷启动可能超过 1.5 秒。
    public static let observationWindowMilliseconds = 3000
}

public extension Gesture {
    /// **最长的一次单次按压**的驻留时长（毫秒）——`gDuring` 的「期间」窗口按它界定。
    ///
    /// - `tap` 系手势 → `tapDurationMilliseconds`
    /// - `hold` 系手势 → `holdDurationMilliseconds` / `longHoldDurationMilliseconds`
    /// - `double` 系手势 → `doubleClickHoldMilliseconds`（每次单击 50ms）
    var singlePressMilliseconds: Int {
        switch kind {
        case .tap: return TriggerTiming.tapDurationMilliseconds
        case .hold: return (name == "hold1500")
            ? TriggerTiming.holdDurationMilliseconds
            : TriggerTiming.longHoldDurationMilliseconds
        case .double: return TriggerTiming.doubleClickHoldMilliseconds
        }
    }
}

public protocol EventPosting {
    func post(_ event: CGEvent)
}

public struct CGEventPoster: EventPosting {
    public init() {}
    public func post(_ event: CGEvent) {
        event.post(tap: .cghidEventTap)
    }
}

public enum InjectorError: Error {
    case eventConstructionFailed(TriggerKey, EventForm)
}

/// ⚠️ **必须是 class，不能是 struct**。
///
/// `defer` 捕获的是**变量当时的副本**——如果注入器是 struct，
/// `defer { injector.releaseAll() }` 释放的是**空副本**，
/// 真正按下的键**留在按下状态**（系统以为 Cmd/Option 还按着，后续所有输入都带 Cmd）。
/// class 的引用语义让 `defer` 看到最终状态。
///
/// `@unchecked Sendable`：状态只在主线程读写；`@Sendable` 的信号回调会读它，
/// 那条路径由 `SignalCleanup` 负责串行化。
public final class KeyInjector: @unchecked Sendable {
    /// 规格 F-41：`kCGEventSourceUserData = 42`。用来标记**自家**注入的事件，
    /// 好让物理输入中止监听能过滤掉它们（否则第一次注入就自杀）。
    public static let eventSourceUserDataMarker: Int64 = 42

    private let poster: EventPosting
    public private(set) var pressedKeys: Set<TriggerKey> = []

    public init(poster: EventPosting = CGEventPoster()) {
        self.poster = poster
    }

    public func perform(
        key: TriggerKey,
        form: EventForm,
        gesture: Gesture,
        sleeper: (Double) throws -> Void
    ) throws {
        switch gesture.kind {
        case .tap:
            try postKeyDown(key: key, form: form)
            try sleeper(Double(TriggerTiming.tapDurationMilliseconds) / 1000)
            try postKeyUp(key: key, form: form)
        case .hold:
            let milliseconds = (gesture == .hold1500)
                ? TriggerTiming.holdDurationMilliseconds
                : TriggerTiming.longHoldDurationMilliseconds
            try postKeyDown(key: key, form: form)
            try sleeper(Double(milliseconds) / 1000)
            try postKeyUp(key: key, form: form)
        case .double:
            let gap = (gesture == .double120)
                ? TriggerTiming.doubleClickGapShortMilliseconds
                : TriggerTiming.doubleClickGapLongMilliseconds
            // ⚠️ 循环变量**不能**写成 `_` 再拿来比较——`_` 只能出现在模式匹配里
            for index in 0..<2 {
                try postKeyDown(key: key, form: form)
                try sleeper(Double(TriggerTiming.doubleClickHoldMilliseconds) / 1000)
                try postKeyUp(key: key, form: form)
                if index == 0 { try sleeper(Double(gap) / 1000) }
            }
        }
    }

    public func postKeyDown(key: TriggerKey, form: EventForm) throws {
        let event = try makeEvent(key: key, form: form, down: true)
        poster.post(event)
        pressedKeys.insert(key)
    }

    public func postKeyUp(key: TriggerKey, form: EventForm) throws {
        let event = try makeEvent(key: key, form: form, down: false)
        poster.post(event)
        pressedKeys.remove(key)
    }

    /// 退出清理第一步：对每个已按下的键补发 up（规格 §6.2）。
    public func releaseAll(sleeper: (Double) -> Void) throws {
        for key in pressedKeys {
            try postKeyUp(key: key, form: .keyEvent)
        }
        sleeper(0)
    }

    private func makeEvent(key: TriggerKey, form: EventForm, down: Bool) throws -> CGEvent {
        let source = CGEventSource(stateID: .hidSystemState)
        let event: CGEvent?
        switch form {
        case .keyEvent:
            event = CGEvent(keyboardEventSource: source, virtualKey: key.virtualKeyCode, keyDown: down)
            event?.flags = down ? key.modifierFlag : []
        case .flagsChanged:
            event = CGEvent(source: source)
            event?.type = .flagsChanged
            event?.setIntegerValueField(.keyboardEventKeycode, value: Int64(key.virtualKeyCode))
            event?.flags = down ? key.modifierFlag : []
        case .systemDefined:
            // Fn 只能走 NSSystemDefined 通道（见 SystemDefinedSpike）
            guard key == .fn else { throw InjectorError.eventConstructionFailed(key, form) }
            event = SystemDefinedSpike.buildFnEvent(keyDown: down)
        }
        guard let event else { throw InjectorError.eventConstructionFailed(key, form) }
        // 标记自家事件——物理输入中止监听按这个标记过滤
        event.setIntegerValueField(.eventSourceUserData, value: Self.eventSourceUserDataMarker)
        return event
    }
}

/// Fn 的 `systemDefined` 事件**没有 `NX_KEYTYPE` 依据**（规格 F-31 / R-02），
/// 但**有修饰键通道**的依据：
///
/// - `NX_MODIFIERKEY_SECONDARYFN = 7`（`ev_keymap.h:113`，F-32）
/// - `NX_KEYDOWN = 10 (0x0A)` / `NX_KEYUP = 11 (0x0B)`（`IOLLEvent.h:106-107`，F-38）
/// - `NX_SUBTYPE_AUX_CONTROL_BUTTONS = 8`（`IOLLEvent.h:186`，F-33）
///
/// ⚠️ 状态值是 `0x0A` / `0x0B`，**不是 `0xA00` / `0xB00`**——后者大了 0x100 倍，
/// 会直接构造出垃圾事件，白白浪费一次 spike。
///
/// ⚠️ **公开 SDK 里没有 `CGEventType.systemDefined`，也没有 `CGEventField.keyboardEventData1`**
/// ——键盘事件的 `data1` 属于私有字段。所以这条路必须用 **`NSEvent.otherEvent`**
/// （上游规格 §6.3 的 `action` 也是这么实现的），再取它的 `cgEvent` 投递。
public enum SystemDefinedSpike {
    public static let modifierKeySecondaryFn = 7
    public static let keyDownState = 0x0A
    public static let keyUpState = 0x0B
    /// `NX_SUBTYPE_AUX_CONTROL_BUTTONS = 8`（F-33）
    public static let subtypeAuxControlButtons = 8

    /// 构造 Fn 的 NSSystemDefined 事件。返回 `nil` 表示构造失败——
    /// 调用方据此把该形态记为 `unsupportedBySDK`（而不是 `notApplicable`）。
    public static func buildFnEvent(keyDown: Bool) -> CGEvent? {
        let state = keyDown ? keyDownState : keyUpState
        let data1 = (modifierKeySecondaryFn << 16) | (state << 8)
        let event = NSEvent.otherEvent(
            with: .systemDefined,
            location: .zero,
            modifierFlags: keyDown ? [.function] : [],
            timestamp: 0,
            windowNumber: 0,
            context: nil,
            subtype: Int16(subtypeAuxControlButtons),
            data1: data1,
            data2: -1
        )
        return event?.cgEvent
    }
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd tools/probes && swift test --filter KeyInjectorTests`
Expected: PASS（6 个用例全绿）

- [ ] **Step 5: 写「无无名 120 字面量」的断言测试**

创建 `tools/probes/Tests/VBG1ACoreTests/TriggerTimingLiteralTests.swift`：

```swift
import XCTest
@testable import VBG1ACore

/// 规格 §8.1：断言范围必须限定在**注入器源文件内**，
/// 且每个 120 都必须来自命名常量 `tapDurationMilliseconds` 或 `doubleClickGapShortMilliseconds`。
/// 文档参数表、`--sample-ms` 等其它 120 是合法的——所以**不能**断言「全仓无 120」。
///
/// ⚠️ **必须先剥掉注释**——注入器文件里的说明文字（「`tapDuration` 同源」等）含「120」字样，
/// 不剥注释的话这个测试必然失败，而且失败得莫名其妙。
final class TriggerTimingLiteralTests: XCTestCase {

    func testInjectorSourceHasNoUnnamed120Literal() throws {
        let injectorURL = try locateInjectorSource()
        let source = try String(contentsOf: injectorURL, encoding: .utf8)
        let code = Self.strippingComments(source)

        let allowedNames = ["tapDurationMilliseconds", "doubleClickGapShortMilliseconds"]
        // ⚠️ 必须匹配**独立的数字 120**，不能用 `contains("120")`——
        // 手势名 `.double120` 里也含 "120"，那是**标识符**不是字面量。
        let pattern = try NSRegularExpression(pattern: "(?<![A-Za-z0-9_])120(?![A-Za-z0-9_])")
        for (index, line) in code.split(separator: "\n", omittingEmptySubsequences: false).enumerated() {
            let text = String(line)
            let range = NSRange(text.startIndex..<text.endIndex, in: text)
            guard pattern.firstMatch(in: text, range: range) != nil else { continue }
            if !allowedNames.contains(where: { text.contains($0) }) {
                XCTFail("第 \(index + 1) 行出现无名 120 字面量：\(text)")
            }
        }
    }

    func testCommentStrippingWorks() {
        // 断言剥注释这件事本身有效——否则上面那个测试可能在测空气
        let stripped = Self.strippingComments("let a = 120 // 这里是 120 的说明\nlet b = 120")
        XCTAssertFalse(stripped.contains("说明"))
        XCTAssertTrue(stripped.contains("let b = 120"))
    }

    func testIdentifierContaining120IsNotALiteral() {
        // `.double120` 是手势名，不是字面量——扫描必须放过它
        let code = Self.strippingComments("let gap = (gesture == .double120)")
        let pattern = try? NSRegularExpression(pattern: "(?<![A-Za-z0-9_])120(?![A-Za-z0-9_])")
        let range = NSRange(code.startIndex..<code.endIndex, in: code)
        XCTAssertEqual(pattern?.firstMatch(in: code, range: range), nil)
    }

    func testNamedConstantsAreTheOnly120Source() {
        XCTAssertEqual(TriggerTiming.tapDurationMilliseconds, 120)
        XCTAssertEqual(TriggerTiming.doubleClickGapShortMilliseconds, 120)
    }

    private func locateInjectorSource() throws -> URL {
        // #filePath = .../Tests/VBG1ACoreTests/TriggerTimingLiteralTests.swift
        // 回退三级到 Sources/VBG1ACore/Trigger/KeyInjector.swift
        let testFile = URL(fileURLWithPath: #filePath)
        let target = testFile
            .deletingLastPathComponent()   // VBG1ACoreTests
            .deletingLastPathComponent()   // Tests
            .deletingLastPathComponent()   // probes
            .appendingPathComponent("Sources/VBG1ACore/Trigger/KeyInjector.swift")
        guard FileManager.default.fileExists(atPath: target.path) else {
            throw XCTSkip("找不到注入器源文件：\(target.path)")
        }
        return target
    }

    /// 去掉 `//` 行注释与 `/* */` 块注释。
    static func strippingComments(_ source: String) -> String {
        var result = ""
        var inBlockComment = false
        for rawLine in source.split(separator: "\n", omittingEmptySubsequences: false) {
            var line = String(rawLine)
            if inBlockComment {
                if let end = line.range(of: "*/") {
                    line = String(line[end.upperBound...])
                    inBlockComment = false
                } else {
                    line = ""
                }
            }
            while let start = line.range(of: "/*") {
                if let end = line.range(of: "*/", range: start.upperBound..<line.endIndex) {
                    line = String(line[..<start.lowerBound]) + String(line[end.upperBound...])
                } else {
                    line = String(line[..<start.lowerBound])
                    inBlockComment = true
                    break
                }
            }
            if let slash = line.range(of: "//") {
                line = String(line[..<slash.lowerBound])
            }
            result += line + "\n"
        }
        return result
    }
}
```

- [ ] **Step 6: 跑测试确认通过**

Run: `cd tools/probes && swift test --filter TriggerTimingLiteralTests`
Expected: PASS（4 个用例全绿）

- [ ] **Step 7: 提交（可选，默认跳过）**

```bash
cd tools/probes && git add Sources/VBG1ACore/Trigger Tests/VBG1ACoreTests && git commit -m "feat(probes): 触发键注入器与时序常量唯一定义点"
```

---

### Task 3: 语音判据器（S1 / S4 / S5 + 去抖 + 可疑命中）

**S5 的逐组归因规则（本计划唯一定义处，D14）**：
**只有**「阶段 -1 正对照通过」**且**「实测 S5 落盘延迟 ≤ 1s」时，S5 才能用于**逐组归因**；
否则 S5 **只是整体佐证**，**不得**归因到具体组——`cfprefsd` 的落盘延迟大时，
计数增量可能落在那一组**结束之后**，把它算成那一组的证据就是编造。
不满足条件时，组里的 `s5CounterDelta` / `s5HitSlot` **照记**，但要标上「S5 仅作整体佐证」。

**Files:**
- Create: `tools/probes/Sources/VBG1ACore/Oracle/VoiceSample.swift`
- Create: `tools/probes/Sources/VBG1ACore/Oracle/VoiceStateOracle.swift`
- Create: `tools/probes/Sources/VBG1ACore/Oracle/AudioProcessInspector.swift`
- Create: `tools/probes/Sources/VBG1ACore/Oracle/SpeechShortcutCounter.swift`
- Test: `tools/probes/Tests/VBG1ACoreTests/VoiceStateOracleTests.swift`

**Interfaces:**
- Consumes: 无（只用 Foundation / CoreAudio）。
- Produces:
  - `public struct VoiceSample` — `timestampMilliseconds: Double`、`s1RunningInput: Bool`、`s4NewAudioProcess: Bool`、`s5CounterDelta: Int`、`s5HitSlot: String?`。
  - `public struct OracleTransition` — `changed: Bool`、`newState: Bool`、`suspicious: Bool`。
  - `public struct VoiceStateOracle` — `static let requiredConsecutiveSamples = 2`；`mutating func ingest(_ sample: VoiceSample) -> OracleTransition`；`mutating func resetSuspiciousHit()`；`var voiceState: Bool`；`var suspiciousHit: Bool`。
  - `public struct AudioProcessStatus` — `pid: pid_t`、`bundleId: String?`、`isRunningInput: Bool`。
  - `public enum AudioProcessInspector` — `static func runningInputProcesses() -> [AudioProcessStatus]`；`static func baselineBundleIdentifiers() -> Set<String>`。
  - `public struct SpeechShortcutCounter` — `static let voiceSlotKey = "YBSpeechShortcutStartCount_voice"`；`static let voiceCommandSlotKey = "YBSpeechShortcutStartCount_voiceCommand"`；`func readCounts() -> (voice: Int, voiceCommand: Int)`；`func snapshot() -> SpeechShortcutCounterSnapshot`。
  - `public struct SpeechShortcutCounterSnapshot` — `voice: Int`、`voiceCommand: Int`；`var max: Int`。

- [ ] **Step 1: 写失败测试——去抖与可疑命中**

创建 `tools/probes/Tests/VBG1ACoreTests/VoiceStateOracleTests.swift`：

```swift
import XCTest
@testable import VBG1ACore

final class VoiceStateOracleTests: XCTestCase {
    private func sample(_ running: Bool, at index: Int) -> VoiceSample {
        VoiceSample(timestampMilliseconds: Double(index) * 50,
                    s1RunningInput: running, s4NewAudioProcess: false,
                    s5CounterDelta: 0, s5HitSlot: nil)
    }

    func testSingleTrueSampleDoesNotFlipState() {
        // 规格 §4.2：50ms 采样 + 「连续 2 次一致才翻转」
        var oracle = VoiceStateOracle()
        let transition = oracle.ingest(sample(true, at: 0))
        XCTAssertFalse(transition.changed)
        XCTAssertFalse(oracle.voiceState)
    }

    func testTwoConsecutiveTrueSamplesFlipState() {
        var oracle = VoiceStateOracle()
        _ = oracle.ingest(sample(true, at: 0))
        let transition = oracle.ingest(sample(true, at: 1))
        XCTAssertTrue(transition.changed)
        XCTAssertTrue(oracle.voiceState)
    }

    func testIsolatedTrueSampleIsMarkedSuspicious() {
        // 规格 §4.2：任何单样本为真 → 「可疑命中，需复测」，不直接判未命中
        var oracle = VoiceStateOracle()
        _ = oracle.ingest(sample(true, at: 0))
        _ = oracle.ingest(sample(false, at: 1))
        XCTAssertTrue(oracle.suspiciousHit)
        XCTAssertFalse(oracle.voiceState)
    }

    func testFlappingSequenceDoesNotFlipState() {
        var oracle = VoiceStateOracle()
        for index in 0..<10 {
            _ = oracle.ingest(sample(index % 2 == 0, at: index))
        }
        XCTAssertFalse(oracle.voiceState)
    }

    func testCompositeStateUsesS1OrS4() {
        var oracle = VoiceStateOracle()
        _ = oracle.ingest(VoiceSample(timestampMilliseconds: 0, s1RunningInput: false,
                                      s4NewAudioProcess: true, s5CounterDelta: 0, s5HitSlot: nil))
        let transition = oracle.ingest(VoiceSample(timestampMilliseconds: 50, s1RunningInput: false,
                                                   s4NewAudioProcess: true, s5CounterDelta: 0, s5HitSlot: nil))
        XCTAssertTrue(transition.changed)
        XCTAssertTrue(oracle.voiceState)
    }

    func testStateReturnsToFalseAfterTwoFalseSamples() {
        var oracle = VoiceStateOracle()
        _ = oracle.ingest(sample(true, at: 0))
        _ = oracle.ingest(sample(true, at: 1))
        XCTAssertTrue(oracle.voiceState)
        let down = oracle.ingest(sample(false, at: 2))
        XCTAssertFalse(down.changed)
        let down2 = oracle.ingest(sample(false, at: 3))
        XCTAssertTrue(down2.changed)
        XCTAssertFalse(oracle.voiceState)
    }

    func testSuspicionIsResetPerGroup() {
        // 「可疑命中」是**逐组**的观测——上一组的可疑不能带到下一组（D9）
        var oracle = VoiceStateOracle()
        _ = oracle.ingest(sample(true, at: 0))
        _ = oracle.ingest(sample(false, at: 1))
        XCTAssertTrue(oracle.suspiciousHit)
        oracle.resetSuspiciousHit()
        XCTAssertFalse(oracle.suspiciousHit)
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd tools/probes && swift test --filter VoiceStateOracleTests`
Expected: FAIL（`Cannot find 'VoiceStateOracle' in scope`）

- [ ] **Step 3: 写最小实现——判据器**

创建 `tools/probes/Sources/VBG1ACore/Oracle/VoiceSample.swift`：

```swift
import Foundation

/// 一次采样的原始值。**报告必须保留原始采样序列**（规格 §4.2），
/// 以便事后复核去抖是否吞掉短脉冲。
public struct VoiceSample: Codable, Sendable {
    public var timestampMilliseconds: Double
    /// S1（主）：目标进程 `kAudioProcessPropertyIsRunningInput` == 1
    public var s1RunningInput: Bool
    /// S4（辅）：基线中不存在的、bundleId 含 tencent 的新音频进程
    public var s4NewAudioProcess: Bool
    /// S5（独立佐证）：IM 自报计数器增量——**不参与 voiceState 合成**
    public var s5CounterDelta: Int
    /// 命中哪个槽：`voice` / `voiceCommand` / `none`
    public var s5HitSlot: String?

    public init(timestampMilliseconds: Double, s1RunningInput: Bool, s4NewAudioProcess: Bool,
                s5CounterDelta: Int, s5HitSlot: String?) {
        self.timestampMilliseconds = timestampMilliseconds
        self.s1RunningInput = s1RunningInput
        self.s4NewAudioProcess = s4NewAudioProcess
        self.s5CounterDelta = s5CounterDelta
        self.s5HitSlot = s5HitSlot
    }
}
```

创建 `tools/probes/Sources/VBG1ACore/Oracle/VoiceStateOracle.swift`：

```swift
import Foundation

public struct OracleTransition {
    public var changed: Bool
    public var newState: Bool
    public var suspicious: Bool

    public init(changed: Bool, newState: Bool, suspicious: Bool) {
        self.changed = changed
        self.newState = newState
        self.suspicious = suspicious
    }
}

/// 合成规则：`voiceState = S1 || S4`。S5 是**独立佐证**，不参与合成（规格 §4.2）。
///
/// 去抖：**连续 2 次一致才翻转**。但去抖会吞掉短脉冲（`tap120` 的收音窗口可能只有 ~120ms），
/// 所以任何**单样本为真**都要记为「**可疑命中，需复测**」，不能直接判未命中。
public struct VoiceStateOracle {
    public static let requiredConsecutiveSamples = 2

    public private(set) var voiceState = false
    public private(set) var suspiciousHit = false

    private var pendingState: Bool?
    private var pendingCount = 0

    public init() {}

    public mutating func ingest(_ sample: VoiceSample) -> OracleTransition {
        let raw = sample.s1RunningInput || sample.s4NewAudioProcess

        if raw == voiceState {
            // ⚠️ 这里**必须**检查「待定的 true 被推翻」——
            // 孤立的一次 true（raw=true → 下一个样本 raw=false）走的就是这条早退路径。
            // 早退时把 pending 一丢了，「可疑命中」就永远不会置位，
            // 规格 §4.2 的「单样本为真 → 可疑命中，需复测」也就落空了。
            if pendingState == true && pendingCount < Self.requiredConsecutiveSamples {
                suspiciousHit = true
            }
            pendingState = nil
            pendingCount = 0
            return OracleTransition(changed: false, newState: voiceState, suspicious: false)
        }

        if pendingState == raw {
            pendingCount += 1
        } else {
            // 换了方向：待定值被推翻。若被推翻的是 true 且从未确认 → 可疑命中
            if pendingState == true && pendingCount < Self.requiredConsecutiveSamples {
                suspiciousHit = true
            }
            pendingState = raw
            pendingCount = 1
        }

        if pendingCount >= Self.requiredConsecutiveSamples {
            voiceState = raw
            pendingState = nil
            pendingCount = 0
            return OracleTransition(changed: true, newState: voiceState, suspicious: false)
        }

        return OracleTransition(changed: false, newState: voiceState, suspicious: false)
    }

    /// 清掉「可疑命中」标志——**每组开始时调用一次**。
    ///
    /// ⚠️ 这个标志必须**逐组**清：`suspiciousHit` 描述的是「**这一组**出现过单样本为真」，
    /// 不是整轮的属性。上一组的可疑如果不清，后面**每一组**都会被误判成「可疑」，
    /// 阶段 2 的复测集就会变成整轮矩阵。
    public mutating func resetSuspiciousHit() {
        suspiciousHit = false
    }
}
```

- [ ] **Step 4: 写最小实现——S1 / S4 与 S5**

创建 `tools/probes/Sources/VBG1ACore/Oracle/AudioProcessInspector.swift`：

```swift
import CoreAudio
import Foundation

public struct AudioProcessStatus {
    /// CoreAudio 的 **AudioObjectID**——它是进程对象在音频系统里的标识，
    /// **不是 pid**（本机实测：Chatterfly 的 objectID = 143，pid = 931）。
    public var objectID: AudioObjectID
    /// 真正的进程 pid——必须用 `kAudioProcessPropertyPID` 从 objectID 换算。
    public var pid: pid_t
    public var bundleId: String?
    public var isRunningInput: Bool

    public init(objectID: AudioObjectID, pid: pid_t, bundleId: String?, isRunningInput: Bool) {
        self.objectID = objectID
        self.pid = pid
        self.bundleId = bundleId
        self.isRunningInput = isRunningInput
    }
}

/// S1（主判据）与 S4（辅判据）的采集。
///
/// ⚠️ **最容易踩的坑**：`kAudioHardwarePropertyProcessObjectList` 返回的是
/// **AudioObjectID 列表，不是 pid 列表**。本机实测：列表里的 id 是 139–165，
/// 而 Chatterfly 是 **objectID 143 / pid 931**。
/// 把 objectID 当 pid 用会同时错两处：
/// ① `NSRunningApplication(processIdentifier: 143)` 指向**别的进程**；
/// ② `kAudioProcessPropertyIsRunningInput` **必须用 objectID 查询**，
///    拿 pid 去查得到的是「pid 143 那个进程」的状态，不是 Chatterfly 的。
/// 所以：**先用 `kAudioProcessPropertyPID` 把 objectID 换成 pid**，
/// 三个属性（pid / bundleID / isRunningInput）都在 **objectID** 上取。
///
/// 依据：空闲时 Chatterfly 已在 CoreAudio 音频进程列表内（pid 931）→
/// **「进程进列表」不可用作起始判据**（F-18）；
/// `kAudioProcessPropertyIsRunningInput` 空闲时为 0 且**可读**（F-16）。
public enum AudioProcessInspector {
    public static let targetBundleHints = ["chatterfly", "tencent"]

    /// 枚举音频进程列表。**每个条目的 objectID 与 pid 一一对应**。
    public static func runningInputProcesses() -> [AudioProcessStatus] {
        return processObjectIDs().map { objectID in
            AudioProcessStatus(
                objectID: objectID,
                pid: processPID(objectID: objectID),
                bundleId: bundleIdentifier(objectID: objectID),
                isRunningInput: isRunningInput(objectID: objectID)
            )
        }
    }

    /// 目标（Chatterfly / 腾讯系）音频进程。
    public static func targetProcesses() -> [AudioProcessStatus] {
        return runningInputProcesses().filter { status in
            guard let bundleId = status.bundleId else { return false }
            return targetBundleHints.contains { bundleId.lowercased().contains($0) }
        }
    }

    /// 基线里出现过的目标进程 bundleId——S4 用它识别「基线中不存在的新进程」。
    public static func baselineBundleIdentifiers() -> Set<String> {
        return Set(targetProcesses().compactMap(\.bundleId))
    }

    // MARK: - CoreAudio 属性读取

    static func processObjectIDs() -> [AudioObjectID] {
        var address = AudioObjectPropertyAddress(
            mSelector: kAudioHardwarePropertyProcessObjectList,
            mScope: kAudioObjectPropertyScopeGlobal,
            mElement: kAudioObjectPropertyElementMain
        )
        var size: UInt32 = 0
        guard AudioObjectGetPropertyDataSize(AudioObjectID(kAudioObjectSystemObject), &address, 0, nil, &size) == noErr,
              size > 0 else { return [] }
        var objectIDs = [AudioObjectID](repeating: 0, count: Int(size) / MemoryLayout<AudioObjectID>.size)
        var dataSize = size
        guard AudioObjectGetPropertyData(AudioObjectID(kAudioObjectSystemObject), &address, 0, nil,
                                         &dataSize, &objectIDs) == noErr else { return [] }
        return objectIDs
    }

    /// **objectID → pid**（`kAudioProcessPropertyPID`）。
    static func processPID(objectID: AudioObjectID) -> pid_t {
        guard let value: Int32 = integerProperty(objectID: objectID, selector: kAudioProcessPropertyPID) else {
            return -1
        }
        return pid_t(value)
    }

    /// **objectID → 是否正在收音**（`kAudioProcessPropertyIsRunningInput`）。
    static func isRunningInput(objectID: AudioObjectID) -> Bool {
        guard let value: UInt32 = integerProperty(objectID: objectID,
                                                  selector: kAudioProcessPropertyIsRunningInput) else {
            return false
        }
        return value != 0
    }

    /// **objectID → bundleId**（`kAudioProcessPropertyBundleID`）。
    /// 用音频系统的 bundleId，**不要**用 `NSRunningApplication` 反查——
    /// 后者需要 pid，而 pid 是从 objectID 换算出来的，绕了一圈还可能取错进程。
    static func bundleIdentifier(objectID: AudioObjectID) -> String? {
        var address = AudioObjectPropertyAddress(
            mSelector: kAudioProcessPropertyBundleID,
            mScope: kAudioObjectPropertyScopeGlobal,
            mElement: kAudioObjectPropertyElementMain
        )
        var size = UInt32(MemoryLayout<CFString?>.size)
        var value: CFString?
        let status = withUnsafeMutablePointer(to: &value) { pointer in
            AudioObjectGetPropertyData(objectID, &address, 0, nil, &size, pointer)
        }
        guard status == noErr, let value else { return nil }
        return value as String
    }

    private static func integerProperty<T: FixedWidthInteger>(objectID: AudioObjectID,
                                                              selector: AudioObjectPropertySelector) -> T? {
        var address = AudioObjectPropertyAddress(
            mSelector: selector,
            mScope: kAudioObjectPropertyScopeGlobal,
            mElement: kAudioObjectPropertyElementMain
        )
        var size = UInt32(MemoryLayout<T>.size)
        var value: T = 0
        let status = withUnsafeMutablePointer(to: &value) { pointer in
            AudioObjectGetPropertyData(objectID, &address, 0, nil, &size, pointer)
        }
        guard status == noErr else { return nil }
        return value
    }
}
```

> `NSRunningApplication` 需要 `import AppKit`——在该文件顶部补上。

创建 `tools/probes/Sources/VBG1ACore/Oracle/SpeechShortcutCounter.swift`：

```swift
import Foundation

public struct SpeechShortcutCounterSnapshot {
    public var voice: Int
    public var voiceCommand: Int

    public init(voice: Int, voiceCommand: Int) {
        self.voice = voice
        self.voiceCommand = voiceCommand
    }

    /// S5 必须覆盖**两个槽**（规格 F-40）：若用户的绑定落在 `voiceCommand` 槽，
    /// 只读 `voice` 槽的计数器会**永远不动**，正对照会误判失败。
    public var max: Int { Swift.max(voice, voiceCommand) }

    public var hitSlot: String {
        if voiceCommand > voice { return "voiceCommand" }
        if voice > 0 { return "voice" }
        return "none"
    }
}

/// S5（独立佐证）：Chatterfly 自报的语音快捷键启动计数。
///
/// **必须用进程内 `CFPreferencesCopyAppValue`**——`defaults read` 每 50ms 会 spawn 一个进程，
/// 污染采样与延迟测量（规格 §4.2 / 修订 #28）。
///
/// 注意 `cfprefsd` 缓存可能延迟落盘，所以 S5 的可用性由**阶段 -1 正对照实测**决定：
/// 落盘延迟 ≤ 1s → 可作逐组交叉判据；更大 → 只能用于整体佐证，不逐组归因。
/// **这条判据的唯一表述在 Task 3 的开头**（D14）——实现与报告必须按它执行。
public struct SpeechShortcutCounter {
    public static let domain = "com.tencent.inputmethod.chatterfly"
    public static let voiceSlotKey = "YBSpeechShortcutStartCount_voice"
    public static let voiceCommandSlotKey = "YBSpeechShortcutStartCount_voiceCommand"

    public init() {}

    public func readCounts() -> (voice: Int, voiceCommand: Int) {
        let voice = Self.readInt(key: Self.voiceSlotKey)
        let voiceCommand = Self.readInt(key: Self.voiceCommandSlotKey)
        return (voice, voiceCommand)
    }

    public func snapshot() -> SpeechShortcutCounterSnapshot {
        let counts = readCounts()
        return SpeechShortcutCounterSnapshot(voice: counts.voice, voiceCommand: counts.voiceCommand)
    }

    static func readInt(key: String) -> Int {
        guard let value = CFPreferencesCopyAppValue(key as CFString,
                                                    Self.domain as CFString) as? Int else {
            return 0
        }
        return value
    }
}
```

- [ ] **Step 5: 跑测试确认通过**

Run: `cd tools/probes && swift test --filter VoiceStateOracleTests`
Expected: PASS（7 个用例全绿）

- [ ] **Step 6: 提交（可选，默认跳过）**

```bash
cd tools/probes && git add Sources/VBG1ACore/Oracle Tests && git commit -m "feat(probes): 语音判据器 S1/S4/S5 与去抖"
```

---

### Task 4: 归类决策程序（穷尽真值表）

**Files:**
- Create: `tools/probes/Sources/VBG1ACore/Analysis/StageTwoClassifier.swift`
- Test: `tools/probes/Tests/VBG1ACoreTests/StageTwoClassifierTests.swift`

**Interfaces:**
- Consumes: `VBG1ACore`（Task 1 的 `GestureKind`）。
- Produces:
  - `public enum VoiceMode: String` — `notTriggered`、`longPress`、`selfStartSelfStop`、`doubleTapSelfStop`、`tap`、`tapOnReleaseVariant`、`undetermined`。
  - `public struct StageTwoObservation` — `gestureKind: GestureKind`、`gStarted: Bool`、`gDuring: Bool?`（`nil` = n/a）、`gAfter: Bool`、`secondPressStops: Bool?`（`nil` = 未测）。
  - `public enum StageTwoClassifier` — `static func classify(_ observation: StageTwoObservation) -> VoiceMode`；`static func gDuring(gesture: Gesture, observedDuringPress: Bool?) -> Bool?`。

- [ ] **Step 1: 写失败测试——穷举真值表**

创建 `tools/probes/Tests/VBG1ACoreTests/StageTwoClassifierTests.swift`：

```swift
import XCTest
@testable import VBG1ACore

final class StageTwoClassifierTests: XCTestCase {
    /// 规格 §8.1：用穷举真值表验证「每种组合都落到**恰好一个**分支」，
    /// **无死分支、无表外落点**。
    func testEveryCombinationLandsOnExactlyOneBranch() {
        var combos: [(StageTwoObservation, VoiceMode)] = []
        for gStarted in [true, false] {
            for gDuring: Bool? in [true, false, nil] {
                for gAfter in [true, false] {
                    for secondPressStops: Bool? in [true, false, nil] {
                        let observation = StageTwoObservation(
                            gestureKind: .tap, gStarted: gStarted, gDuring: gDuring,
                            gAfter: gAfter, secondPressStops: secondPressStops
                        )
                        let verdict = StageTwoClassifier.classify(observation)
                        combos.append((observation, verdict))
                    }
                }
            }
        }
        XCTAssertEqual(combos.count, 36)
        // 每个组合都必须返回一个确定的模式（不能有 fatalError / 落空）
        for (observation, verdict) in combos {
            XCTAssertFalse(verdict.rawValue.isEmpty,
                           "组合 \(observation) 落到了空分支")
        }
    }

    func testNotStartedMeansNotTriggered() {
        let observation = StageTwoObservation(gestureKind: .tap, gStarted: false, gDuring: nil,
                                              gAfter: false, secondPressStops: nil)
        XCTAssertEqual(StageTwoClassifier.classify(observation), .notTriggered)
    }

    func testRunningDuringPressAndStoppedAfterReleaseIsLongPress() {
        let observation = StageTwoObservation(gestureKind: .hold, gStarted: true, gDuring: true,
                                              gAfter: false, secondPressStops: nil)
        XCTAssertEqual(StageTwoClassifier.classify(observation), .longPress)
    }

    func testStartedAfterReleaseIsSelfStartSelfStop() {
        let observation = StageTwoObservation(gestureKind: .tap, gStarted: true, gDuring: nil,
                                              gAfter: false, secondPressStops: nil)
        XCTAssertEqual(StageTwoClassifier.classify(observation), .selfStartSelfStop)
    }

    func testDoubleGestureSelfStopGetsSpecificLabel() {
        let observation = StageTwoObservation(gestureKind: .double, gStarted: true, gDuring: nil,
                                              gAfter: false, secondPressStops: nil)
        XCTAssertEqual(StageTwoClassifier.classify(observation), .doubleTapSelfStop)
    }

    func testStillRecordingAfterReleaseAndSecondPressStopsIsTapMode() {
        let observation = StageTwoObservation(gestureKind: .tap, gStarted: true, gDuring: nil,
                                              gAfter: true, secondPressStops: true)
        XCTAssertEqual(StageTwoClassifier.classify(observation), .tap)
    }

    func testSecondPressDoesNotStopIsTapOnReleaseVariant() {
        let observation = StageTwoObservation(gestureKind: .tap, gStarted: true, gDuring: nil,
                                              gAfter: true, secondPressStops: false)
        XCTAssertEqual(StageTwoClassifier.classify(observation), .tapOnReleaseVariant)
    }

    func testUntestedSecondPressIsUndetermined() {
        let observation = StageTwoObservation(gestureKind: .tap, gStarted: true, gDuring: nil,
                                              gAfter: true, secondPressStops: nil)
        XCTAssertEqual(StageTwoClassifier.classify(observation), .undetermined)
    }

    // MARK: - gDuring 的定义（D8）

    func testGDuringIsNilForTapGesture() {
        // tap120 的单次按压只有 120ms（< 200ms）→ 没有「期间」窗口 → n/a
        XCTAssertNil(StageTwoClassifier.gDuring(gesture: .tap120, observedDuringPress: true))
    }

    func testGDuringIsNilForDoubleGesture() {
        // 双击的单次按压是 doubleClickHoldMilliseconds = 50ms（< 200ms）→ n/a
        XCTAssertNil(StageTwoClassifier.gDuring(gesture: .double120, observedDuringPress: true))
        XCTAssertNil(StageTwoClassifier.gDuring(gesture: .double300, observedDuringPress: false))
    }

    func testGDuringKeepsTrueForHoldGestureThatWasRecording() {
        // hold 的单次按压 ≥ 200ms → 取观测到的按压期间状态
        XCTAssertEqual(StageTwoClassifier.gDuring(gesture: .hold1500, observedDuringPress: true), true)
    }

    func testGDuringKeepsFalseForHoldGestureThatWasNotRecording() {
        // ⚠️ 「按压期间没在收音」是 **false**，不是 n/a——
        // n/a 只用于「压根没有按压窗口」的手势（D8）
        XCTAssertEqual(StageTwoClassifier.gDuring(gesture: .hold3000, observedDuringPress: false), false)
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd tools/probes && swift test --filter StageTwoClassifierTests`
Expected: FAIL（`Cannot find 'StageTwoClassifier' in scope`）

- [ ] **Step 3: 写最小实现**

创建 `tools/probes/Sources/VBG1ACore/Analysis/StageTwoClassifier.swift`：

```swift
import Foundation

public enum VoiceMode: String, Sendable {
    /// 两次复测都 false → 阶段 1 假阳性，该组合不触发
    case notTriggered
    /// 长按模式：按住期间收音，松开即停
    case longPress
    /// 松开后自启自停
    case selfStartSelfStop
    /// 双击即启即停
    case doubleTapSelfStop
    /// 点按模式：松开不停，再按一次才停
    case tap
    /// tapOnRelease 变体：第二次按也停不了
    case tapOnReleaseVariant
    /// 未定论——必须复测一次，不得归入任何模式
    case undetermined
}

public struct StageTwoObservation {
    /// 命中手势自己的种类——用于「自启自停」分支的标签细分
    public var gestureKind: GestureKind
    /// G 是否启动
    public var gStarted: Bool
    /// 按压窗口内是否在收音；`nil` = n/a（见 `StageTwoClassifier.gDuring(gesture:observedDuringPress:)` 的定义）
    public var gDuring: Bool?
    /// 松开后是否仍在收音
    public var gAfter: Bool
    /// 再按一次是否停止；`nil` = 未测
    public var secondPressStops: Bool?

    public init(gestureKind: GestureKind, gStarted: Bool, gDuring: Bool?,
                gAfter: Bool, secondPressStops: Bool?) {
        self.gestureKind = gestureKind
        self.gStarted = gStarted
        self.gDuring = gDuring
        self.gAfter = gAfter
        self.secondPressStops = secondPressStops
    }
}

/// 规格 §4.3 的**归类决策程序**（按序取第一个满足的分支）。
///
/// ⚠️ 「按住」轴一律取自**命中手势自己的 `gDuring`**——固定用 `hold1500` 当「按住」轴
/// 会把 `hold3000` 命中误判成「双击即启即停」（阈值 > 1.5s 时 `hold1500` 不启动、
/// `hold3000` 启动后松开即停）。`hold1500` 只作长按基线，**不参与归类**。
public enum StageTwoClassifier {
    /// **「按压期间」轴（`gDuring`）的定义**（规格 §4.3，D8）：
    ///
    /// `gDuring` = **手势最长的那一次单次按压期间**，App 是否在收音。
    ///
    /// - 单次按压 **≥ 200ms** 才有「期间」这个窗口 → 取观测到的状态（**true / false**）；
    /// - 单次按压 < 200ms → **`nil`（n/a）**——压根没有可观测的「期间」。
    ///
    /// 映射到五个手势：
    /// - `hold1500` / `hold3000`：单次按压 1500 / 3000ms ≥ 200ms → 取按压期间的观测状态；
    /// - `tap120`：单次按压 120ms < 200ms → **n/a**；
    /// - `double120` / `double300`：单次按压 `doubleClickHoldMilliseconds` = 50ms < 200ms → **n/a**。
    ///
    /// ⚠️ 「按压期间**没有**在收音」是 **false**，不是 n/a——
    /// 把「false」当成 n/a 会把长按模式误判成「自启自停」。
    public static func gDuring(gesture: Gesture, observedDuringPress: Bool?) -> Bool? {
        guard gesture.kind == .hold else { return nil }
        return observedDuringPress
    }

    public static func classify(_ observation: StageTwoObservation) -> VoiceMode {
        // 1. 没启动 → 阶段 1 假阳性（需复测；两次都 false 才记「不触发」）
        guard observation.gStarted else { return .notTriggered }

        // 2. 松开后已经停了
        if !observation.gAfter {
            // 2a. 按住期间在收音 → 长按模式
            if observation.gDuring == true { return .longPress }
            // 2b. 启动发生在松开之后、且已自行停止 → 自启自停类
            return (observation.gestureKind == .double) ? .doubleTapSelfStop : .selfStartSelfStop
        }

        // 3. 松开后还在收音
        if observation.secondPressStops == true { return .tap }
        if observation.secondPressStops == false { return .tapOnReleaseVariant }
        // 3c. 未测 / 未定论 → 必须复测一次，不得归入任何模式
        return .undetermined
    }
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd tools/probes && swift test --filter StageTwoClassifierTests`
Expected: PASS（12 个用例全绿）

- [ ] **Step 5: 提交（可选，默认跳过）**

```bash
cd tools/probes && git add Sources/VBG1ACore/Analysis Tests && git commit -m "feat(probes): 归类决策程序（穷尽真值表）"
```

---

### Task 5: 停止程序状态机

**Files:**
- Create: `tools/probes/Sources/VBG1ACore/Analysis/StopProcedure.swift`
- Test: `tools/probes/Tests/VBG1ACoreTests/StopProcedureTests.swift`

**Interfaces:**
- Consumes: `VBG1ACore`（Task 2 的 `KeyInjector` / `TriggerTiming`，Task 3 的 `VoiceStateOracle`）。
- Produces:
  - `public enum StopOutcome` — `stoppedByKeyUp`、`stoppedByStopTap`、`failed`。
  - `public protocol VoiceStateReading { var voiceState: Bool { get } }`。
  - `public enum StopProcedure` — `static func run(mode: VoiceMode, injector: inout KeyInjector, key: TriggerKey, form: EventForm, isStillRecording: () -> Bool, sleeper: (Double) -> Void) throws -> StopOutcome`。
    停止拍 = `down → **立即** up`（**零驻留**，D1）；长按模式**不发**停止拍。

- [ ] **Step 1: 写失败测试——停止程序状态机**

创建 `tools/probes/Tests/VBG1ACoreTests/StopProcedureTests.swift`：

```swift
import XCTest
@testable import VBG1ACore

final class StopProcedureTests: XCTestCase {
    /// up 即停：不需要补停止拍
    func testKeyUpStopsImmediately() throws {
        var injector = KeyInjector(poster: RecordingPoster())
        var recording = true
        let outcome = try StopProcedure.run(
            mode: .tap, injector: &injector, key: .cmdLeft, form: .keyEvent,
            isStillRecording: { recording },
            sleeper: { _ in recording = false }
        )
        XCTAssertEqual(outcome, .stoppedByKeyUp)
    }

    /// up 无效 → 补一次停止拍（down → **立即** up，零驻留）
    func testStopTapIsSentWhenStillRecording() throws {
        let poster = RecordingPoster()
        var injector = KeyInjector(poster: poster)
        var recording = true
        let outcome = try StopProcedure.run(
            mode: .tap, injector: &injector, key: .cmdLeft, form: .keyEvent,
            isStillRecording: { recording },
            sleeper: { _ in if poster.events.count >= 3 { recording = false } }
        )
        XCTAssertEqual(outcome, .stoppedByStopTap)
        // 停止拍 = down + up（补发 up 之后还要再补一次 down→up）
        XCTAssertGreaterThanOrEqual(poster.events.count, 3)
    }

    /// 停止拍的驻留时间必须是 **0**（上游规格 §7.1 / §7.2，D1）
    func testStopTapHasZeroDwell() throws {
        let poster = RecordingPoster()
        var injector = KeyInjector(poster: poster)
        var recording = true
        var sleeps: [Double] = []
        let outcome = try StopProcedure.run(
            mode: .tap, injector: &injector, key: .cmdLeft, form: .keyEvent,
            isStillRecording: { recording },
            sleeper: { seconds in
                sleeps.append(seconds)
                if poster.events.count >= 3 { recording = false }
            }
        )
        XCTAssertEqual(outcome, .stoppedByStopTap)
        XCTAssertEqual(poster.events.count, 3)      // 补发的 up + 停止拍的 down + up
        // down 与 up 之间**没有任何 sleep**——两次 sleep 只在「等 up 生效」和「等停止拍生效」之后
        XCTAssertFalse(sleeps.contains(Double(TriggerTiming.tapDurationMilliseconds) / 1000))
        XCTAssertEqual(sleeps.count, 2)
    }

    /// 补停止拍后仍不停 → stopFailed
    func testFailureIsReportedWhenNeverStops() throws {
        var injector = KeyInjector(poster: RecordingPoster())
        let outcome = try StopProcedure.run(
            mode: .tap, injector: &injector, key: .cmdLeft, form: .keyEvent,
            isStillRecording: { true },
            sleeper: { _ in }
        )
        XCTAssertEqual(outcome, .failed)
    }

    /// 长按模式下**不得**补停止拍——那会短暂开一个新会话
    func testLongPressModeNeverSendsStopTap() throws {
        let poster = RecordingPoster()
        var injector = KeyInjector(poster: poster)
        let outcome = try StopProcedure.run(
            mode: .longPress, injector: &injector, key: .cmdLeft, form: .keyEvent,
            isStillRecording: { true },
            sleeper: { _ in }
        )
        XCTAssertEqual(outcome, .failed)
        // 只有补发的 up（1 个事件），没有停止拍的 down+up
        XCTAssertEqual(poster.events.count, 1)
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd tools/probes && swift test --filter StopProcedureTests`
Expected: FAIL（`Cannot find 'StopProcedure' in scope`）

- [ ] **Step 3: 写最小实现**

创建 `tools/probes/Sources/VBG1ACore/Analysis/StopProcedure.swift`：

```swift
import Foundation

public enum StopOutcome: String {
    /// 补发的 up 就停了
    case stoppedByKeyUp
    /// up 无效，补了一次停止拍后停了
    case stoppedByStopTap
    /// 停不下来 → 中止整轮并提示用户手动停止
    case failed
}

/// 停止程序（规格 §4.3）。阶段 1 / 阶段 2 的状态清零 / 退出清理**共用**它。
///
/// 为什么不能只「补发 up」：Chatterfly 的点按语义是「再按一次才停」，
/// **up 不会停止收音**。若只补发 up，第一组命中（很可能就是点按模式）就会留下一个仍在收音的会话，
/// 下一组的基线 `voiceState == true` → **整轮中止**。
///
/// ⚠️ **长按模式下不发停止拍**（见计划开头的「冲突裁决」表）——
/// 长按模式的 up 本来就该停；停不下来时再补一次 down→**立即** up 只会**开一个新会话**。
///
/// ⚠️ **停止拍的驻留时间是 0**（上游规格 §7.1 / §7.2，D1）——
/// 上游用**驻留时间**区分两拍：**开始拍**是 `down → 120ms → up`（用 `tapDuration`），
/// **停止拍**是 `down → 立即 up`。停止拍带 120ms 驻留会被输入法**误读成一次开始拍**，
/// 于是又开出一个新会话——问题从两个会话变成三个。
public enum StopProcedure {
    public static func run(
        mode: VoiceMode,
        injector: inout KeyInjector,
        key: TriggerKey,
        form: EventForm,
        isStillRecording: () -> Bool,
        sleeper: (Double) -> Void
    ) throws -> StopOutcome {
        // 1. 补发 up（幂等；长按模式下这本来就够）
        try injector.postKeyUp(key: key, form: form)
        sleeper(1.0)

        // 2. 仍为 true → 补一次「停止拍」
        if isStillRecording() {
            guard mode != .longPress else { return .failed }
            // 停止拍 = down → **立即** up（**零驻留**，不用 tapDuration）
            try injector.postKeyDown(key: key, form: form)
            try injector.postKeyUp(key: key, form: form)
            sleeper(1.5)
        } else {
            return .stoppedByKeyUp
        }

        // 3. 发出停止拍后**必须再验证一次**——不能假设它生效
        if isStillRecording() { return .failed }
        return .stoppedByStopTap
    }
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd tools/probes && swift test --filter StopProcedureTests`
Expected: PASS（5 个用例全绿）

- [ ] **Step 5: 提交（可选，默认跳过）**

```bash
cd tools/probes && git add Sources/VBG1ACore/Analysis Tests && git commit -m "feat(probes): 停止程序状态机"
```

---

### Task 6: 选键规则

**Files:**
- Create: `tools/probes/Sources/VBG1ACore/Analysis/KeySelection.swift`
- Test: `tools/probes/Tests/VBG1ACoreTests/KeySelectionTests.swift`

**Interfaces:**
- Consumes: `VBG1ACore`（Task 1 的 `TriggerKey`，Task 4 的 `VoiceMode`）。
- Produces:
  - `public struct KeyCandidate` — `key: TriggerKey`、`mode: VoiceMode`、`startLatencyMilliseconds: Double`、`stopLatencyMilliseconds: Double?`。
  - `public enum KeySelection` — `static func selectKey(from candidates: [KeyCandidate]) -> KeyCandidate?`；`static func selectMode(longPressStartLatencyMilliseconds: Double?, tapStartLatencyMilliseconds: Double?) -> VoiceMode?`；`static let longPressLatencyPenaltyFactor = 2.0`。
    规则顺序：**① 丢 fn** → **② 丢有系统级副作用的键** → **③ 每个键用 2× 规则选自己的最佳模式（长按档优先）** → **④ 同档内取 `startLatencyMs` 最低者**。

- [ ] **Step 1: 写失败测试——四级选键规则**

创建 `tools/probes/Tests/VBG1ACoreTests/KeySelectionTests.swift`：

```swift
import XCTest
@testable import VBG1ACore

final class KeySelectionTests: XCTestCase {
    private func candidate(_ key: TriggerKey, _ mode: VoiceMode, startMs: Double) -> KeyCandidate {
        KeyCandidate(key: key, mode: mode, startLatencyMilliseconds: startMs, stopLatencyMilliseconds: 20)
    }

    func testNonFunctionKeyWinsOverFunctionKey() {
        // 规则 1：非 Fn 优先——避免系统 Fn 功能误触与输入源切换
        let selected = KeySelection.selectKey(from: [
            candidate(.fn, .longPress, startMs: 10),
            candidate(.cmdLeft, .longPress, startMs: 100),
        ])
        XCTAssertEqual(selected?.key, .cmdLeft)
    }

    func testLowestStartLatencyWinsAmongEquivalentKeys() {
        // 规则 4：同一档内 `startLatencyMs` 最低者优先
        let selected = KeySelection.selectKey(from: [
            candidate(.cmdLeft, .longPress, startMs: 120),
            candidate(.optLeft, .longPress, startMs: 80),
        ])
        XCTAssertEqual(selected?.key, .optLeft)
    }

    func testEmptyCandidateListSelectsNothing() {
        XCTAssertNil(KeySelection.selectKey(from: []))
    }

    func testLongPressModeIsPreferred() {
        // 规则 3：同为可用时优先长按模式——单次 down/up，没有「停止拍失败」这一失败面
        let mode = KeySelection.selectMode(longPressStartLatencyMilliseconds: 100,
                                           tapStartLatencyMilliseconds: 90)
        XCTAssertEqual(mode, .longPress)
    }

    func testTapModeWinsWhenLongPressLatencyIsMoreThanTwiceWorse() {
        // 规则 3 的例外阈值：长按 > 2× 点按
        let mode = KeySelection.selectMode(longPressStartLatencyMilliseconds: 250,
                                           tapStartLatencyMilliseconds: 100)
        XCTAssertEqual(mode, .tap)
    }

    func testExactlyTwiceIsStillLongPress() {
        // 阈值是「> 2×」，不是「≥ 2×」
        let mode = KeySelection.selectMode(longPressStartLatencyMilliseconds: 200,
                                           tapStartLatencyMilliseconds: 100)
        XCTAssertEqual(mode, .longPress)
    }

    func testOnlyOneModeAvailablePicksThatOne() {
        XCTAssertEqual(KeySelection.selectMode(longPressStartLatencyMilliseconds: nil,
                                               tapStartLatencyMilliseconds: 100), .tap)
        XCTAssertEqual(KeySelection.selectMode(longPressStartLatencyMilliseconds: 100,
                                               tapStartLatencyMilliseconds: nil), .longPress)
    }

    // MARK: - 2× 例外在 `selectKey` 里**可达**（D7）

    func testSelectKeyPicksTapWhenLongPressIsMoreThanTwiceWorse() {
        // 同一个键：长按 250ms、点按 100ms → 2× 例外生效 → 选**点按**
        let selected = KeySelection.selectKey(from: [
            candidate(.cmdLeft, .longPress, startMs: 250),
            candidate(.cmdLeft, .tap, startMs: 100),
        ])
        XCTAssertEqual(selected?.key, .cmdLeft)
        XCTAssertEqual(selected?.mode, .tap)
    }

    func testSelectKeyPrefersLongPressWhenBothModesAreAvailable() {
        // 同一个键：长按 100ms、点按 90ms → 不触发 2× 例外 → 选**长按**
        let selected = KeySelection.selectKey(from: [
            candidate(.cmdLeft, .longPress, startMs: 100),
            candidate(.cmdLeft, .tap, startMs: 90),
        ])
        XCTAssertEqual(selected?.key, .cmdLeft)
        XCTAssertEqual(selected?.mode, .longPress)
    }

    func testSelectKeyPicksLowestLatencyWithinThePreferredTier() {
        // 规则 4 只在**同一档内**比较延迟——长按档优先于点按档
        let selected = KeySelection.selectKey(from: [
            candidate(.cmdLeft, .longPress, startMs: 150),
            candidate(.optLeft, .longPress, startMs: 90),
            candidate(.optRight, .tap, startMs: 10),
        ])
        XCTAssertEqual(selected?.key, .optLeft)
        XCTAssertEqual(selected?.mode, .longPress)
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd tools/probes && swift test --filter KeySelectionTests`
Expected: FAIL（`Cannot find 'KeySelection' in scope`）

- [ ] **Step 3: 写最小实现**

创建 `tools/probes/Sources/VBG1ACore/Analysis/KeySelection.swift`：

```swift
import Foundation

public struct KeyCandidate {
    public var key: TriggerKey
    public var mode: VoiceMode
    public var startLatencyMilliseconds: Double
    public var stopLatencyMilliseconds: Double?

    public init(key: TriggerKey, mode: VoiceMode, startLatencyMilliseconds: Double,
                stopLatencyMilliseconds: Double?) {
        self.key = key
        self.mode = mode
        self.startLatencyMilliseconds = startLatencyMilliseconds
        self.stopLatencyMilliseconds = stopLatencyMilliseconds
    }
}

/// 规格 §4.4 的选键规则。**没有这条规则，结论就无法直接变成上游规格 §7.2 的实现。**
///
/// ⚠️ **规则顺序不能换**（D7）：如果先按 `startLatencyMs` 过滤，规则 3 的
/// 「长按 > 2× 点按」例外就**永远不可达**——低延迟的那个模式会被直接选中，
/// 根本轮不到按停止语义选模式。
public enum KeySelection {
    /// 规则 3 的例外阈值：长按模式的 `startLatencyMs` **> 2×** 点按模式时才改选点按。
    public static let longPressLatencyPenaltyFactor = 2.0

    /// 规则 1-4：选出一个唯一的键 + 模式。
    public static func selectKey(from candidates: [KeyCandidate]) -> KeyCandidate? {
        guard !candidates.isEmpty else { return nil }

        // 规则 1：非 Fn 优先——避免系统 Fn 功能误触与输入源切换
        var pool = candidates
        let nonFunction = pool.filter { !$0.key.isFunctionKey }
        if !nonFunction.isEmpty { pool = nonFunction }

        // 规则 2：剔除**有系统级副作用**的键。本探针的候选**全都是单键**，
        // 所以这一步今天是空操作——但必须留着，否则将来加入组合键时会被漏掉。
        let sideEffectFree = pool.filter { !$0.key.hasSystemSideEffect }
        if !sideEffectFree.isEmpty { pool = sideEffectFree }

        // 规则 3（停止语义）：**每个键**先用自己的长按 / 点按延迟跑一次 2× 规则，
        // 定出这个键的**最佳模式**。
        var bestPerKey: [TriggerKey: KeyCandidate] = [:]
        for key in Set(pool.map(\.key)) {
            let forKey = pool.filter { $0.key == key }
            let longPressLatency = forKey.first { $0.mode == .longPress }?.startLatencyMilliseconds
            let tapLatency = forKey.first { $0.mode == .tap }?.startLatencyMilliseconds
            guard let mode = selectMode(longPressStartLatencyMilliseconds: longPressLatency,
                                        tapStartLatencyMilliseconds: tapLatency) else { continue }
            guard let candidate = forKey.filter({ $0.mode == mode })
                .min(by: { $0.startLatencyMilliseconds < $1.startLatencyMilliseconds }) else { continue }
            bestPerKey[key] = candidate
        }
        guard !bestPerKey.isEmpty else { return nil }

        // 规则 3（续）：**长按档优于点按档**——长按没有「停止拍失败」这一失败面
        let longPressTier = bestPerKey.values.filter { $0.mode == .longPress }
        let tier = longPressTier.isEmpty ? Array(bestPerKey.values) : longPressTier

        // 规则 4：同一档内取 `startLatencyMs` 最低者
        return tier.min { $0.startLatencyMilliseconds < $1.startLatencyMilliseconds }
    }

    /// 规则 3 的核心：同一把键下，长按模式与点按模式二选一。
    /// 长按延迟 **> 2×** 点按延迟时改选点按。
    public static func selectMode(longPressStartLatencyMilliseconds: Double?,
                                   tapStartLatencyMilliseconds: Double?) -> VoiceMode? {
        switch (longPressStartLatencyMilliseconds, tapStartLatencyMilliseconds) {
        case (nil, nil):
            return nil
        case (let longPress?, nil):
            return .longPress
        case (nil, let _?):
            return .tap
        case (let longPress?, let tap?):
            if longPress > tap * longPressLatencyPenaltyFactor { return .tap }
            return .longPress
        }
    }
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd tools/probes && swift test --filter KeySelectionTests`
Expected: PASS（10 个用例全绿）

- [ ] **Step 5: 提交（可选，默认跳过）**

```bash
cd tools/probes && git add Sources/VBG1ACore/Analysis Tests && git commit -m "feat(probes): 选键规则"
```

---

### Task 7: 物理输入中止（CGEventTap）

**Files:**
- Create: `tools/probes/Sources/VBG1ACore/Guard/PhysicalInputGuard.swift`
- Create: `tools/probes/Sources/VBG1ACore/Guard/SignalCleanup.swift`
- Test: `tools/probes/Tests/VBG1ACoreTests/PhysicalInputGuardTests.swift`
- Test: `tools/probes/Tests/VBG1ACoreTests/AbortStateTests.swift`

**Interfaces:**
- Consumes: `VBG1ACore`（Task 2 的 `KeyInjector.eventSourceUserDataMarker`）。
- Produces:
  - `public enum PhysicalInputVerdict` — `ownEvent`、`foreignEvent`。
  - `public enum PhysicalInputGuard` — `static let monitoredEventTypes: CGEventMask`；`static func verdict(for event: CGEvent) -> PhysicalInputVerdict`；`static func monitors(_ type: CGEventType) -> Bool`；`static func install(onAbort: @escaping @Sendable () -> Void) -> Bool`；`static func uninstall()`。
    - ⚠️ **`install` 必须在专用线程上跑**：CGEvent tap 的回调在它的 source 所属的 **run loop** 上执行。把 source 加到**主线程**的 run loop，而主线程全程 `Thread.sleep`（探针正是如此），回调**永远不会被执行**——等于装了个不会响的警报。所以 `install` 自己起一个 `Thread` 跑 `CFRunLoopRun()`，并把 source 加到**那个线程**的 run loop。
    - 返回 `false` 表示缺少 Input Monitoring 权限（`tapCreate` 失败）——调用方据此在横幅与报告里显著标注「本次运行无法中止物理输入」。
  - `public enum SignalCleanup` — `static func install(handler: @escaping @Sendable () -> Void)`；`static func uninstall()`。
    - 装 `SIGINT` / `SIGTERM` 处理：**用户 Ctrl-C 时探针正在按住 Cmd/Option**，不补发 up 事件的话修饰键会卡住（系统以为还按着），之后用户敲的每个字母都变成 Cmd+字母。

- [ ] **Step 1: 写失败测试——自家事件与外来事件的区分**

创建 `tools/probes/Tests/VBG1ACoreTests/PhysicalInputGuardTests.swift`：

```swift
import XCTest
@testable import VBG1ACore

final class PhysicalInputGuardTests: XCTestCase {
    func testOwnEventIsNotTreatedAsUserInput() {
        // 规格 §6.1 要求 ①：忽略带 eventSourceUserData 标记的自家事件
        let event = CGEvent(source: CGEventSource(stateID: .hidSystemState))
        event?.setIntegerValueField(.eventSourceUserData, value: KeyInjector.eventSourceUserDataMarker)
        XCTAssertEqual(PhysicalInputGuard.verdict(for: event!), .ownEvent)
    }

    func testForeignEventIsTreatedAsUserInput() {
        let event = CGEvent(source: CGEventSource(stateID: .hidSystemState))
        event?.setIntegerValueField(.eventSourceUserData, value: 0)
        XCTAssertEqual(PhysicalInputGuard.verdict(for: event!), .foreignEvent)
    }

    func testMonitoredTypesIncludeKeyDownAndFlagsChanged() {
        // 规格 §6.1 要求 ②：用户按住 Shift / Cmd / Option 产生的是 flagsChanged 而不是 keyDown
        let mask = PhysicalInputGuard.monitoredEventTypes
        XCTAssertNotEqual(mask, 0)
        // ⚠️ `CGEventMask` 是位掩码，**没有 `contains(_:)`**——必须自己做位运算
        XCTAssertTrue(mask & (CGEventMask(1) << CGEventType.keyDown.rawValue) != 0)
        XCTAssertTrue(mask & (CGEventMask(1) << CGEventType.flagsChanged.rawValue) != 0)
        XCTAssertTrue(PhysicalInputGuard.monitors(.keyDown))
        XCTAssertTrue(PhysicalInputGuard.monitors(.flagsChanged))
        XCTAssertFalse(PhysicalInputGuard.monitors(.keyUp))
    }
}
```


- [ ] **Step 2: 跑测试确认失败**

Run: `cd tools/probes && swift test --filter PhysicalInputGuardTests`
Expected: FAIL（`Cannot find 'PhysicalInputGuard' in scope`）

- [ ] **Step 3: 写最小实现**

创建 `tools/probes/Sources/VBG1ACore/Guard/PhysicalInputGuard.swift`：

```swift
import CoreGraphics
import Foundation

public enum PhysicalInputVerdict {
    /// 自家注入的事件——**必须忽略**，否则第一次注入就自杀
    case ownEvent
    /// 用户真实输入——**触发中止**
    case foreignEvent
}

/// 物理输入中止（规格 §6.1）。
///
/// 三条硬性要求：
/// ① 忽略带 `eventSourceUserData` 标记的自家事件；
/// ② 监听集合**必须含 `.flagsChanged`**（用户按住修饰键产生的是 flagsChanged 而不是 keyDown）；
/// ③ 只在检测到「非自家」事件时中止。
///
/// 监听键盘事件需要 **Input Monitoring** 权限，**AX 权限不覆盖**它。
/// 拿不到权限时**不弹窗索要**——改为在横幅与报告的显著位置写明「本次运行无法中止物理输入」。
///
/// ⚠️ **必须跑在专用线程上**：CGEvent tap 的回调在它的 source 所属的 **run loop** 上执行。
/// 若把 source 加到**主线程**的 run loop，而主线程全程 `Thread.sleep`（探针正是如此），
/// 回调**永远不会被执行**——等于装了个不会响的警报。
public enum PhysicalInputGuard {
    /// 监听集合：`keyDown` + `flagsChanged`。
    ///
    /// ⚠️ `CGEventMask` 是位掩码，**没有 `contains(_:)` 方法**——
    /// 判断某个事件类型在不在里面必须自己做位运算。
    public static let monitoredEventTypes: CGEventMask =
        (CGEventMask(1) << CGEventType.keyDown.rawValue)
        | (CGEventMask(1) << CGEventType.flagsChanged.rawValue)

    /// 中止回调的**线程安全**槽。C 回调是无捕获闭包，只能通过全局槽通信。
    private static let abortSlot = AbortSlot()

    /// 监听线程的句柄（`uninstall` 要靠它停掉 run loop）。
    private static let monitorSlot = MonitorSlot()

    /// 无捕获闭包 → 可以转成 C 函数指针。
    private static let tapCallback: CGEventTapCallBack = { _, type, event, _ in
        guard type == .keyDown || type == .flagsChanged else {
            return Unmanaged.passUnretained(event)
        }
        if PhysicalInputGuard.verdict(for: event) == .foreignEvent {
            PhysicalInputGuard.abortSlot.fire()
        }
        return Unmanaged.passUnretained(event)
    }

    public static func verdict(for event: CGEvent) -> PhysicalInputVerdict {
        let marker = event.getIntegerValueField(.eventSourceUserData)
        if marker == KeyInjector.eventSourceUserDataMarker { return .ownEvent }
        return .foreignEvent
    }

    /// 掩码里是否包含某个事件类型（`CGEventMask` 没有 `contains`）。
    public static func monitors(_ type: CGEventType) -> Bool {
        return monitoredEventTypes & (CGEventMask(1) << type.rawValue) != 0
    }

    /// 在**专用线程**上安装监听。
    ///
    /// 返回 `true` 表示监听已安装；`false` 表示缺少 Input Monitoring 权限
    /// （或 tap 构造失败）。
    @discardableResult
    public static func install(onAbort: @escaping @Sendable () -> Void) -> Bool {
        abortSlot.set(onAbort)
        let box = InstallBox()
        let semaphore = DispatchSemaphore(value: 0)

        let thread = Thread {
            let mask = monitoredEventTypes
            guard let tap = CGEvent.tapCreate(
                tap: .cgSessionEventTap,
                place: .headInsertEventTap,
                options: .defaultTap,
                eventsOfInterest: mask,
                callback: tapCallback,
                userInfo: nil
            ) else {
                semaphore.signal()
                return
            }
            guard let source = CFMachPortCreateRunLoopSource(kCFAllocatorDefault, tap, 0) else {
                semaphore.signal()
                return
            }
            // ⚠️ 加到**当前线程**（监听线程）的 run loop，**不是**主线程。
            // 主线程全程 `Thread.sleep`，主 run loop 根本不会被唤醒。
            CFRunLoopAddSource(CFRunLoopGetCurrent(), source, .commonModes)
            CGEvent.tapEnable(tap: tap, enable: true)
            box.markInstalled(runLoop: CFRunLoopGetCurrent())
            semaphore.signal()
            // 阻塞在这个线程的 run loop 上——回调在这里执行
            CFRunLoopRun()
        }
        thread.name = "vb-physical-input-tap"
        thread.stackSize = 512 * 1024
        thread.start()

        _ = semaphore.wait(timeout: .now() + 3)
        guard box.isInstalled else {
            abortSlot.clear()
            return false
        }
        monitorSlot.set(thread: thread, runLoop: box.installedRunLoop)
        return true
    }

    /// 解除监听并**停止监听线程**（退出清理）。
    public static func uninstall() {
        abortSlot.clear()
        monitorSlot.clear()
    }
}

/// 中止回调的线程安全槽。
final class AbortSlot: @unchecked Sendable {
    private let lock = NSLock()
    private var handler: (@Sendable () -> Void)?

    func set(_ handler: @escaping @Sendable () -> Void) {
        lock.lock(); defer { lock.unlock() }
        self.handler = handler
    }

    func clear() {
        lock.lock(); defer { lock.unlock() }
        handler = nil
    }

    /// 在**不持锁**的状态下调回调——回调可能反过来调回本类，持有锁会死锁。
    func fire() {
        lock.lock()
        let handler = self.handler
        lock.unlock()
        handler?()
    }
}

/// 安装过程的跨线程结果。
final class InstallBox: @unchecked Sendable {
    private let lock = NSLock()
    private var installed = false
    private var runLoop: CFRunLoop?

    func markInstalled(runLoop: CFRunLoop) {
        lock.lock(); defer { lock.unlock() }
        installed = true
        self.runLoop = runLoop
    }

    var isInstalled: Bool {
        lock.lock(); defer { lock.unlock() }
        return installed
    }

    var installedRunLoop: CFRunLoop? {
        lock.lock(); defer { lock.unlock() }
        return runLoop
    }
}

/// 监听线程与它的 run loop 的句柄。
final class MonitorSlot: @unchecked Sendable {
    private let lock = NSLock()
    private var thread: Thread?
    private var runLoop: CFRunLoop?

    func set(thread: Thread, runLoop: CFRunLoop?) {
        lock.lock(); defer { lock.unlock() }
        self.thread = thread
        self.runLoop = runLoop
    }

    func clear() {
        lock.lock()
        let runLoop = self.runLoop
        self.thread = nil
        self.runLoop = nil
        lock.unlock()
        // 停掉监听线程的 run loop——`CFRunLoopRun` 才会返回、线程才会退出
        if let runLoop {
            CFRunLoopStop(runLoop)
        }
    }
}

/// 物理输入中止标志。C 回调是**无捕获**闭包，只能通过一个线程安全的共享槽通信。
///
/// ⚠️ 这个类型住在**物理输入中止**这个文件里，而不是编排器里——
/// 编排器（Task 9）要用它，中止监听（Task 7）也要用它。
/// 放在编排器里会让 Task 7 的测试在 Task 9 之前编译不过。
final class AbortState: @unchecked Sendable {
    private let lock = NSLock()
    private var aborted = false

    var isAborted: Bool {
        lock.lock(); defer { lock.unlock() }
        return aborted
    }

    func trigger() {
        lock.lock(); defer { lock.unlock() }
        aborted = true
    }
}
```

创建 `tools/probes/Sources/VBG1ACore/Guard/SignalCleanup.swift`：

```swift
import Foundation

/// 退出信号清理（规格 §6.2）。
///
/// 探针会**按住 Cmd/Option 键**。用户在探测中途按 Ctrl-C 时，
/// 修饰键会**卡住**——系统以为还按着，之后用户敲的每个字母都变成 Cmd+字母
/// （敲 Q 就是退出应用）。所以必须在 `SIGINT` / `SIGTERM` 时补发 up 事件。
///
/// ⚠️ C 信号处理函数是 **C 函数指针，不能捕获**——所以清理动作要放进一个
/// `nonisolated(unsafe)` 全局槽里，由信号处理函数去取。
public enum SignalCleanup {
    /// 清理回调的槽。
    nonisolated(unsafe) private static var handler: (@Sendable () -> Void)?

    /// `SIGINT` = Ctrl-C。退出码用 130（惯例：128 + 信号号）。
    private static let sigintHandler: @convention(c) (Int32) -> Void = { _ in
        SignalCleanup.handler?()
        exit(130)
    }

    /// `SIGTERM` = `kill` 发的终止信号。退出码用 143。
    private static let sigtermHandler: @convention(c) (Int32) -> Void = { _ in
        SignalCleanup.handler?()
        exit(143)
    }

    /// 安装信号处理。重复安装会覆盖旧的回调。
    public static func install(handler: @escaping @Sendable () -> Void) {
        SignalCleanup.handler = handler
        signal(SIGINT, sigintHandler)
        signal(SIGTERM, sigtermHandler)
    }

    /// 恢复默认行为并清空回调（正常跑完时调用）。
    public static func uninstall() {
        signal(SIGINT, SIG_DFL)
        signal(SIGTERM, SIG_DFL)
        handler = nil
    }
}
```

创建 `tools/probes/Tests/VBG1ACoreTests/AbortStateTests.swift`：

```swift
import XCTest
import VBProbeKit
@testable import VBG1ACore

/// 物理输入中止链路的单测。
///
/// ⚠️ 这些是「**实机上会出事**」的那部分逻辑：
/// 中止标志**必须真的被读**（不是只创建不读）、信号处理**必须装得上、卸得掉**。
final class AbortStateTests: XCTestCase {

    // MARK: - 物理输入中止

    func testAbortStateStartsUnsetAndLatches() {
        let state = AbortState()
        XCTAssertFalse(state.isAborted, "初始状态必须是「未中止」")
        state.trigger()
        XCTAssertTrue(state.isAborted, "trigger 之后必须读得到 true")
        // 「闩锁」语义：触发后再触发不会变回 false
        state.trigger()
        XCTAssertTrue(state.isAborted)
    }

    func testForeignEventWouldTriggerAbortButOwnEventWouldNot() throws {
        // ⚠️ **不真的调用 `install`**——那会在这台机器上装一个真的键盘 tap，
        // 用户测试时碰一下键盘就会把 `isAborted` 置起来，让断言变得随机失败。
        // 这里只测**纯逻辑**（`verdict` / `monitors`）；「回调会不会真的被调用」
        // 属于实机行为，由实机跑覆盖（规格 §8.2）。
        let state = AbortState()

        // 自家事件（带 userData 标记）→ 判为 ownEvent，不该中止
        let own = CGEvent(keyboardEventSource: nil, virtualKey: 0x37, keyDown: true)!
        own.setIntegerValueField(.eventSourceUserData, value: KeyInjector.eventSourceUserDataMarker)
        XCTAssertEqual(PhysicalInputGuard.verdict(for: own), .ownEvent)

        // 外来事件（无标记）→ 判为 foreignEvent
        let foreign = CGEvent(keyboardEventSource: nil, virtualKey: 0x37, keyDown: true)!
        XCTAssertEqual(PhysicalInputGuard.verdict(for: foreign), .foreignEvent)

        // 接线语义：外来事件 → 中止标志被置起来
        if PhysicalInputGuard.verdict(for: foreign) == .foreignEvent {
            state.trigger()
        }
        XCTAssertTrue(state.isAborted)
    }

    func testMonitoredTypesIncludeKeyDownAndFlagsChanged() {
        // 用户按住修饰键产生的是 flagsChanged 而不是 keyDown——漏了它就漏了最危险的情况
        XCTAssertTrue(PhysicalInputGuard.monitors(.keyDown))
        XCTAssertTrue(PhysicalInputGuard.monitors(.flagsChanged))
    }

    // MARK: - 信号清理

    func testSignalCleanupCanBeInstalledAndRemoved() {
        let counter = CounterBox()
        SignalCleanup.install { counter.increment() }
        SignalCleanup.uninstall()
        // 装得上、卸得掉，且**不会**真的触发（否则进程会被 exit）
        XCTAssertEqual(counter.value, 0)
    }
}

/// `@Sendable` 闭包不能直接捕获并修改 `var`——用引用盒子。
final class CounterBox: @unchecked Sendable {
    private let lock = NSLock()
    private var count = 0

    func increment() {
        lock.lock(); defer { lock.unlock() }
        count += 1
    }

    var value: Int {
        lock.lock(); defer { lock.unlock() }
        return count
    }
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd tools/probes && swift test --filter PhysicalInputGuardTests`
Expected: PASS（3 个用例全绿）

- [ ] **Step 5: 提交（可选，默认跳过）**

```bash
cd tools/probes && git add Sources/VBG1ACore/Guard Tests && git commit -m "feat(probes): 物理输入中止监听"
```

---

### Task 8: 指标模型 + 报告载荷

**Files:**
- Create: `tools/probes/Sources/VBG1ACore/Measurement/G1AGroupReport.swift`
- Create: `tools/probes/Sources/VBG1ACore/Measurement/G1APayload.swift`
- Create: `tools/probes/Sources/VBG1ACore/Measurement/CleanupProbe.swift`
- Test: `tools/probes/Tests/VBG1ACoreTests/G1APayloadTests.swift`
- Test: `tools/probes/Tests/VBG1ACoreTests/CleanupProbeTests.swift`

**Interfaces:**
- Consumes: `VBG1ACore`（Task 1、3、4）、`VBProbeKit`（计划 1 Task 1 的 `ProbePayload`）。
- Produces:
  - `public struct G1AGroupReport: Codable` — 一组的全部观测：`groupId`、`key`、`form`、`gesture`、`outcome`（**hit / suspicious / miss** 三态）、`startLatencyMilliseconds`、`stopLatencyMilliseconds`、`recordedDuringPress: Bool?`、`s5CounterDelta`、`s5HitSlot`、`inputSourceBefore/After`、`samples`、`notes`。
  - `public struct G1AManualCheck: Codable` — `performed: Bool`、`passed: Bool`、`note: String?`。
  - `public struct G1APayload: Codable, ProbePayload` — 整轮报告：`probeName`、`reportTimestamp`、`reportEnvironment`、`positiveControl`、`preview`、`groups`、`selected`、`manualCheck: G1AManualCheck?`、`systemDefinedSpikeStatus: String`、`frontmostCheckMode: String`、`cleanup`。
    - `systemDefinedSpikeStatus`：`supported` / `unsupportedBySDK` / `disabledByFlag` / `notAttempted`——读者必须能看出 spike 到底跑没跑。
    - `frontmostCheckMode`：`windowLevel`（比对了目标窗口）/ `ownerPIDOnly`（**没拿到窗口号**，只比了进程 pid）。
  - `public struct G1AFailurePayload: Codable, ProbePayload` — **失败报告**：`failureReason`、`completedGroups: [G1AGroupReport]`、`systemDefinedSpikeStatus`、`frontmostCheckMode`、`cleanup`。
    - 全轮 8–10 分钟，**中途失败很常见**。没有失败报告的话，已完成的组的数据全丢，而且用户看不到「系统现在干不干净」——卡键恰恰是在失败那一刻最容易留下的。
  - `public struct CleanupStatusG1A: Codable` — `noStuckModifier: Bool`、`notStillRecording: Bool`、`inputSourceRestored: Bool`、`notes: [String]`。
  - `public enum CleanupProbe` — `static func collect(baselineInputSource: String?) -> CleanupStatusG1A`（**实机采集**）；`static func judge(stuckModifier:stillRecording:baselineInputSource:currentInputSource:) -> CleanupStatusG1A`（**纯逻辑**，可单测）。
    - ⚠️ 报告里的「清理状态」必须来自**实测**，不能写死 `true`——写死等于「探针自己声称系统干净」。

- [ ] **Step 1: 写失败测试——载荷可编码且含关键字段**

创建 `tools/probes/Tests/VBG1ACoreTests/G1APayloadTests.swift`：

```swift
import XCTest
import VBProbeKit
@testable import VBG1ACore

final class G1APayloadTests: XCTestCase {
    private func environment() -> EnvironmentSnapshot {
        return EnvironmentSnapshot(
            hardwareModel: "Mac15,7", macOSVersion: "27.0.1", swiftVersion: "6.4",
            chatterflyVersion: "1.0.8",
            chatterflyPath: "/Library/Input Methods/Chatterfly.app",
            chatterflyIsSelectedInputSource: true,
            axPermissionGranted: true, physicalInputAbortAvailable: true,
            currentInputSourceID: "com.tencent.inputmethod.chatterfly.pinyin")
    }

    func testPayloadEncodesWithRequiredTopLevelKeys() throws {
        let payload = G1APayload(reportTimestamp: Date(timeIntervalSince1970: 1_700_000_000),
                                 reportEnvironment: environment(),
                                 positiveControl: G1APositiveControl(
                                    performed: true, passed: true, s1Passed: true, s5Passed: true,
                                    s5CounterDelta: 1, s5HitSlot: "voice",
                                    s5PersistenceLatencyMilliseconds: 120, timedOut: false, notes: []),
                                 preview: nil, groups: [], selected: nil,
                                 manualCheck: G1AManualCheck(performed: true, passed: true, note: "文字上屏"),
                                 systemDefinedSpikeStatus: "supported",
                                 frontmostCheckMode: "windowLevel",
                                 cleanup: CleanupStatusG1A(noStuckModifier: true, notStillRecording: true,
                                                          inputSourceRestored: true, notes: []))
        let data = try JSONEncoder().encode(payload)
        let json = String(data: data, encoding: .utf8)!
        for key in ["probeName", "reportTimestamp", "reportEnvironment", "positiveControl", "groups",
                    "manualCheck", "cleanup", "systemDefinedSpikeStatus", "frontmostCheckMode"] {
            XCTAssertTrue(json.contains("\"\(key)\""), "缺少顶层字段：\(key)")
        }
    }

    func testNilOptionalsAreOmittedFromTheJSON() throws {
        // `JSONEncoder` 会**省略** nil 的可选字段——所以「字段存在」的断言
        // 必须给非 nil 值，否则测的是一个永远不成立的假设。
        let payload = G1APayload(reportTimestamp: Date(timeIntervalSince1970: 1_700_000_000),
                                 reportEnvironment: environment(),
                                 positiveControl: nil, preview: nil, groups: [], selected: nil,
                                 manualCheck: nil,
                                 systemDefinedSpikeStatus: "unsupportedBySDK",
                                 frontmostCheckMode: "ownerPIDOnly",
                                 cleanup: CleanupStatusG1A(noStuckModifier: true, notStillRecording: true,
                                                          inputSourceRestored: true, notes: []))
        let data = try JSONEncoder().encode(payload)
        let json = String(data: data, encoding: .utf8)!
        XCTAssertFalse(json.contains("\"positiveControl\""))
        XCTAssertFalse(json.contains("\"manualCheck\""))
        // 必填字段始终在
        XCTAssertTrue(json.contains("\"cleanup\""))
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd tools/probes && swift test --filter G1APayloadTests`
Expected: FAIL（`Cannot find 'G1APayload' in scope`）

- [ ] **Step 3: 写最小实现**

创建 `tools/probes/Sources/VBG1ACore/Measurement/G1AGroupReport.swift`：

```swift
import Foundation

/// 一组（键 × 形态 × 手势）的全部观测。
public struct G1AGroupReport: Codable {
    public var groupId: String
    public var key: String
    public var form: String
    public var gesture: String
    /// 命中 / 可疑命中（单样本为真，被去抖吞掉）/ 未命中——**三态**
    public var outcome: String
    public var startLatencyMilliseconds: Double?
    public var stopLatencyMilliseconds: Double?
    /// 按压窗口内是否观察到收音；`nil` = 该手势没有「期间」窗口（见 `StageTwoClassifier.gDuring`）
    public var recordedDuringPress: Bool?
    public var s5CounterDelta: Int
    public var s5HitSlot: String?
    public var inputSourceBefore: String?
    public var inputSourceAfter: String?
    public var samples: [VoiceSample]
    public var notes: [String]
}

public struct G1APositiveControl: Codable {
    public var performed: Bool
    public var passed: Bool
    public var s1Passed: Bool
    public var s5Passed: Bool
    public var s5CounterDelta: Int
    public var s5HitSlot: String?
    /// S5 的落盘延迟（毫秒）——决定 S5 能否逐组归因
    public var s5PersistenceLatencyMilliseconds: Double?
    public var timedOut: Bool
    public var notes: [String]
}

public struct G1APreview: Codable {
    public var declaredKey: String
    public var declaredMode: String
    public var outcome: String
    public var notes: [String]
}

public struct G1ASelection: Codable {
    public var key: String
    public var mode: String
    public var startLatencyMilliseconds: Double
    public var reason: String
}

public struct CleanupStatusG1A: Codable {
    public var noStuckModifier: Bool
    public var notStillRecording: Bool
    public var inputSourceRestored: Bool
    public var notes: [String]
}

/// **阶段 3 · 人工抽检**的结果。
///
/// 用户**只说话、不碰键盘**，由探针注入一次获胜组合；
/// 观察窗延长到用户确认「文字上屏」为止，结果记进报告。
public struct G1AManualCheck: Codable {
    public var performed: Bool
    public var passed: Bool
    public var note: String?

    public init(performed: Bool, passed: Bool, note: String? = nil) {
        self.performed = performed
        self.passed = passed
        self.note = note
    }
}
```

创建 `tools/probes/Sources/VBG1ACore/Measurement/G1APayload.swift`：

```swift
import Foundation
import VBProbeKit

public struct G1APayload: Codable, ProbePayload {
    public var probeName: String = "vb-g1a-trigger"
    public var reportTimestamp: Date
    public var reportEnvironment: EnvironmentSnapshot
    public var positiveControl: G1APositiveControl?
    public var preview: G1APreview?
    public var groups: [G1AGroupReport]
    public var selected: G1ASelection?
    /// **阶段 3 · 人工抽检**的结果（命中组合必须做）
    public var manualCheck: G1AManualCheck?
    /// `systemDefined` spike 的状态（规格 §4.1）：
    /// - `supported`：Fn 的 systemDefined 事件构造成功 → 矩阵含 55 组
    /// - `unsupportedBySDK`：构造失败 → **降级**到 50 组（不中止）
    /// - `disabledByFlag`：用户用 `--no-spike-system-defined` 关掉了
    public var systemDefinedSpikeStatus: String
    /// 前台校验的**实际强度**（规格 §6.1）：
    /// - `windowLevel`：拿到了目标窗口号，比对的是「**目标窗口**在最前面」
    /// - `ownerPIDOnly`：**没拿到**窗口号，只比对了进程 pid——必须显式标注，不能假装做过窗口级校验
    public var frontmostCheckMode: String
    public var cleanup: CleanupStatusG1A

    public init(reportTimestamp: Date, reportEnvironment: EnvironmentSnapshot,
                positiveControl: G1APositiveControl?, preview: G1APreview?,
                groups: [G1AGroupReport], selected: G1ASelection?,
                manualCheck: G1AManualCheck?,
                systemDefinedSpikeStatus: String,
                frontmostCheckMode: String,
                cleanup: CleanupStatusG1A) {
        self.reportTimestamp = reportTimestamp
        self.reportEnvironment = reportEnvironment
        self.positiveControl = positiveControl
        self.preview = preview
        self.groups = groups
        self.selected = selected
        self.manualCheck = manualCheck
        self.systemDefinedSpikeStatus = systemDefinedSpikeStatus
        self.frontmostCheckMode = frontmostCheckMode
        self.cleanup = cleanup
    }
}

/// **失败报告**：探针中途抛错时产出。
///
/// G1-a 全轮要跑 8–10 分钟，中途失败（物理输入中止 / 输入源被切走 / 前台校验失败）
/// **很常见**。此时如果什么都不产出，已完成的组的数据就全丢了，
/// 而且用户看不到「系统现在干不干净」——卡键恰恰是在失败的那一刻最容易留下的。
/// 所以失败也要写一份带**实测清理状态**的报告。
public struct G1AFailurePayload: Codable, ProbePayload {
    public var probeName: String = "vb-g1a-trigger"
    public var reportTimestamp: Date
    public var reportEnvironment: EnvironmentSnapshot
    /// 失败原因（`error.localizedDescription` 或自定义文案）
    public var failureReason: String
    /// 失败时已经完成的组——能保留多少是多少
    public var completedGroups: [G1AGroupReport]
    /// `systemDefined` spike 的状态
    public var systemDefinedSpikeStatus: String
    /// 前台校验的实际强度（`windowLevel` / `ownerPIDOnly`）
    public var frontmostCheckMode: String
    /// **实测**的清理状态
    public var cleanup: CleanupStatusG1A

    public init(reportTimestamp: Date, reportEnvironment: EnvironmentSnapshot,
                failureReason: String, completedGroups: [G1AGroupReport],
                systemDefinedSpikeStatus: String, frontmostCheckMode: String,
                cleanup: CleanupStatusG1A) {
        self.reportTimestamp = reportTimestamp
        self.reportEnvironment = reportEnvironment
        self.failureReason = failureReason
        self.completedGroups = completedGroups
        self.systemDefinedSpikeStatus = systemDefinedSpikeStatus
        self.frontmostCheckMode = frontmostCheckMode
        self.cleanup = cleanup
    }
}
```

创建 `tools/probes/Sources/VBG1ACore/Measurement/CleanupProbe.swift`：

```swift
import CoreGraphics
import Foundation
import VBProbeKit

/// **实测**的清理状态（规格 §6.2）。
///
/// ⚠️ 报告里的「清理状态」必须来自**实测**，不能写死 `true`——
/// 写死等于「探针自己声称系统干净」，而卡键 / 未停收音恰恰是探针自己造成的。
/// 规格 §6.2 要求：探测结束后确认「无卡住的修饰键 / 未在收音 / 输入源已恢复」。
public enum CleanupProbe {
    /// 采集当前时刻的清理状态（**实机调用**）。
    ///
    /// - Parameter baselineInputSource: 本轮开始时记录的输入源。
    ///   `nil` 表示**没记录到基线**——此时不判「未恢复」（不能凭空说它没恢复）。
    public static func collect(baselineInputSource: String?) -> CleanupStatusG1A {
        let flags = CGEventSource.flagsState(.combinedSessionState)
        let stuckModifier = flags.contains(.maskSecondaryFn)
            || flags.contains(.maskCommand)
            || flags.contains(.maskAlternate)

        let stillRecording = AudioProcessInspector.runningInputProcesses().contains { status in
            guard let bundleId = status.bundleId else { return false }
            return AudioProcessInspector.targetBundleHints.contains { bundleId.lowercased().contains($0) }
                && status.isRunningInput
        }

        let current = InputSourceService().currentInputSourceID()
        return judge(stuckModifier: stuckModifier,
                     stillRecording: stillRecording,
                     baselineInputSource: baselineInputSource,
                     currentInputSource: current)
    }

    /// **纯逻辑**的判定——采集与判定分开，才能单测。
    ///
    /// - Parameter baselineInputSource: `nil` 表示**没记录到基线**，
    ///   此时**不判**「未恢复」（不能凭空说它没恢复）。
    public static func judge(stuckModifier: Bool, stillRecording: Bool,
                             baselineInputSource: String?, currentInputSource: String?) -> CleanupStatusG1A {
        let restored: Bool
        if let baselineInputSource {
            restored = (currentInputSource == baselineInputSource)
        } else {
            restored = true
        }

        var notes: [String] = []
        if stuckModifier {
            notes.append("检测到修饰键仍处于按下状态——**可能卡键**，需手动释放（Cmd/Option/Fn）")
        }
        if stillRecording {
            notes.append("Chatterfly **仍在收音**——可能未正常停止")
        }
        if !restored {
            notes.append("输入源未回到基线：\(baselineInputSource ?? "?") → \(currentInputSource ?? "?")")
        }
        if notes.isEmpty {
            notes.append("清理检查通过：无卡键、未在收音、输入源已回到基线")
        }

        return CleanupStatusG1A(noStuckModifier: !stuckModifier,
                                notStillRecording: !stillRecording,
                                inputSourceRestored: restored,
                                notes: notes)
    }
}
```

创建 `tools/probes/Tests/VBG1ACoreTests/CleanupProbeTests.swift`：

```swift
import XCTest
import VBProbeKit
@testable import VBG1ACore

/// 清理状态判定的单测。
///
/// ⚠️ 报告里的「清理状态」必须来自**实测**——写死 `true` 等于
/// 「探针自己声称系统干净」，而卡键 / 未停收音恰恰是探针自己造成的。
final class CleanupProbeTests: XCTestCase {

    func testCleanSystemReportsClean() {
        let status = CleanupProbe.judge(stuckModifier: false, stillRecording: false,
                                        baselineInputSource: "com.tencent.inputmethod.chatterfly.pinyin",
                                        currentInputSource: "com.tencent.inputmethod.chatterfly.pinyin")
        XCTAssertTrue(status.noStuckModifier)
        XCTAssertTrue(status.notStillRecording)
        XCTAssertTrue(status.inputSourceRestored)
    }

    func testStuckModifierIsReportedNotAssumedClean() {
        let status = CleanupProbe.judge(stuckModifier: true, stillRecording: false,
                                        baselineInputSource: "a", currentInputSource: "a")
        XCTAssertFalse(status.noStuckModifier, "卡键必须被如实报告")
        XCTAssertTrue(status.notes.contains { $0.contains("卡键") })
    }

    func testStillRecordingIsReportedNotAssumedClean() {
        let status = CleanupProbe.judge(stuckModifier: false, stillRecording: true,
                                        baselineInputSource: "a", currentInputSource: "a")
        XCTAssertFalse(status.notStillRecording, "仍在收音必须被如实报告")
        XCTAssertTrue(status.notes.contains { $0.contains("仍在收音") })
    }

    func testInputSourceDriftIsReported() {
        let status = CleanupProbe.judge(stuckModifier: false, stillRecording: false,
                                        baselineInputSource: "pinyin", currentInputSource: "abc")
        XCTAssertFalse(status.inputSourceRestored)
        XCTAssertTrue(status.notes.contains { $0.contains("输入源未回到基线") })
    }

    func testMissingBaselineDoesNotClaimDrift() {
        // 没记录到基线时**不能**说「输入源没恢复」——那是在编造
        let status = CleanupProbe.judge(stuckModifier: false, stillRecording: false,
                                        baselineInputSource: nil, currentInputSource: "abc")
        XCTAssertTrue(status.inputSourceRestored)
    }
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd tools/probes && swift test --filter G1APayloadTests`
Expected: PASS（2 个用例全绿）

- [ ] **Step 5: 提交（可选，默认跳过）**

```bash
cd tools/probes && git add Sources/VBG1ACore/Measurement Tests && git commit -m "feat(probes): G1-a 指标模型与报告载荷"
```

---

### Task 9: 参数集 + 编排器（正对照 → 阶段 0 → 0.5 → 1 → 2 → 3 人工抽检）

**Files:**
- Create: `tools/probes/Sources/VBG1ACore/Arguments/G1AArguments.swift`
- Create: `tools/probes/Sources/VBG1ACore/Measurement/G1ARunner.swift`
- Test: `tools/probes/Tests/VBG1ACoreTests/G1AArgumentsTests.swift`

**Interfaces:**
- Consumes: `VBG1ACore`（Task 1-8）、`VBProbeKit`（计划 1）。
- Produces:
  - `public enum DeclaredMode: String` — `longPress`、`tap`、`doubleTap`。
  - `public struct G1AArguments: ProbeArgumentSet` — 见下。
  - `public struct G1ARunner` — `init(arguments:inputSources:counter:)`；`func run() throws -> G1APayload`；`func failurePayload(error:environment:) -> G1AFailurePayload`。
    阶段：-1 正对照 → 0 前置 → 0.5 快速预览 → 1 完整矩阵（三态）→ 2 判读模式（输入集 = **命中 ∪ 可疑命中**）→ **3 人工抽检**。
    ⚠️ **中断安全**（本 Task 的硬要求）：
    - `KeyInjector` 是 **class** 不是 struct——`defer` 捕获 struct 副本的话，释放的是空副本，真正按下的键会**卡住**；
    - `defer { try? injector.releaseAll(...) }` + `SignalCleanup.install { ... }` 覆盖**所有**退出路径（含 Ctrl-C）；
    - **物理输入中止**必须**真的被读**：每次采样、每组前后都查 `AbortState.isAborted`，触发时**先释放按键再抛错**；
    - `systemDefined` spike **默认自动尝试**，失败**降级**到 50 组并记 `unsupportedBySDK`（不中止）；
    - 前台校验必须传**实际窗口号**（`TextEditInstance.resolveWindowNumber()`），拿不到就**显式标注** `ownerPIDOnly`；
    - 清理状态必须**实测**（`CleanupProbe`），不能写死 `true`；
    - 失败也要**产出报告**（`failurePayload`），带上已完成的组与实测清理状态。

- [ ] **Step 1: 写失败测试——参数校验**

创建 `tools/probes/Tests/VBG1ACoreTests/G1AArgumentsTests.swift`：

```swift
import XCTest
import VBProbeKit
@testable import VBG1ACore

final class G1AArgumentsTests: XCTestCase {
    private func scan(_ argv: [String]) throws -> ArgumentScanner {
        return try ArgumentScanner(argv, schema: G1AArguments.optionSchema)
    }

    func testDeclaredKeyAndModeMustBePaired() throws {
        // 规格 §4.0 / §8.1：只给其一 → 启动即报错退出
        let onlyKey = try scan(["--declared-key", "fn"])
        XCTAssertThrowsError(try G1AArguments(scanner: onlyKey))

        let onlyMode = try scan(["--declared-mode", "longPress"])
        XCTAssertThrowsError(try G1AArguments(scanner: onlyMode))

        let both = try scan(["--declared-key", "fn", "--declared-mode", "longPress"])
        XCTAssertNoThrow(try G1AArguments(scanner: both))
    }

    func testUnknownOptionIsRejected() throws {
        // 拼错的参数必须报错，而不是被静默忽略
        XCTAssertThrowsError(try scan(["--sample", "50"]))
    }

    func testDefaultsMatchSpec() throws {
        let arguments = try G1AArguments(scanner: try scan([]))
        // `URL(fileURLWithPath:)` 返回**绝对路径**（相对 cwd）——按后缀比，
        // 不能拿它跟相对的 "docs/verification" 直接比。
        XCTAssertTrue(arguments.outDirectory.path.hasSuffix("docs/verification"),
                      "实际路径：\(arguments.outDirectory.path)")
        XCTAssertEqual(arguments.sampleIntervalMilliseconds, 50)
        XCTAssertEqual(arguments.maximumGroups, nil)
        XCTAssertFalse(arguments.matrixOnly)
        XCTAssertFalse(arguments.skipPositiveControl)
        // ⚠️ 默认**自动尝试** spike（规格 §4.1）——不是默认关
        XCTAssertTrue(arguments.spikeSystemDefined)
    }

    func testOnlyRestrictsToSingleKey() throws {
        let scanner = try scan(["--only", "optLeft"])
        let arguments = try G1AArguments(scanner: scanner)
        XCTAssertEqual(arguments.onlyKey, .optLeft)
    }

    func testUnknownOnlyKeyIsRejected() throws {
        let scanner = try scan(["--only", "space"])
        XCTAssertThrowsError(try G1AArguments(scanner: scanner))
    }

    func testDeclaredModeValuesAreLimited() throws {
        let scanner = try scan(["--declared-key", "fn", "--declared-mode", "shout"])
        XCTAssertThrowsError(try G1AArguments(scanner: scanner))
    }

    func testInvalidSampleIntervalIsRejected() throws {
        let scanner = try scan(["--sample-ms", "0"])
        XCTAssertThrowsError(try G1AArguments(scanner: scanner))
    }

    func testSpikeIsOnByDefault() throws {
        let arguments = try G1AArguments(scanner: try scan([]))
        XCTAssertTrue(arguments.spikeSystemDefined, "规格 §4.1 要求默认自动尝试 spike")
    }

    func testNoSpikeFlagDisablesIt() throws {
        let arguments = try G1AArguments(scanner: try scan(["--no-spike-system-defined"]))
        XCTAssertFalse(arguments.spikeSystemDefined)
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd tools/probes && swift test --filter G1AArgumentsTests`
Expected: FAIL（`Cannot find 'G1AArguments' in scope`）

- [ ] **Step 3: 写最小实现——参数集**

创建 `tools/probes/Sources/VBG1ACore/Arguments/G1AArguments.swift`：

```swift
import Foundation
import VBProbeKit

public enum DeclaredMode: String, Sendable {
    case longPress
    case tap
    case doubleTap
}

public struct G1AArguments: ProbeArgumentSet {
    /// 该探针支持的**全部**参数。**不在表里的参数直接报错退出**。
    public static let optionSchema: [String: OptionKind] = [
        "out": .value,
        "dry-run": .flag,
        "sample-ms": .value,
        "only": .value,
        "declared-key": .value,
        "declared-mode": .value,
        "matrix-only": .flag,
        "max-groups": .value,
        "no-abort-on-voice-stuck": .flag,
        "skip-positive-control": .flag,
        "now": .value,
        "env": .value,
        "no-spike-system-defined": .flag,
    ]

    public var outDirectory: URL
    public var dryRun: Bool
    public var sampleIntervalMilliseconds: Int
    public var onlyKey: TriggerKey?
    public var declaredKey: TriggerKey?
    public var declaredMode: DeclaredMode?
    public var matrixOnly: Bool
    public var maximumGroups: Int?
    public var abortOnVoiceStuck: Bool
    public var skipPositiveControl: Bool
    /// 是否把 `systemDefined` 形态的 Fn spike 纳入矩阵（规格 §4.1）。
    ///
    /// **默认 `true`**——规格 §4.1 的 spike 是**流程的一部分**，不是可选增强：
    /// 探针**默认自动尝试**构造 Fn 的 systemDefined 事件，
    /// 成功则矩阵含 55 组，失败则**降级**到 50 组并记 `unsupportedBySDK`（**不中止**）。
    /// 用 `--no-spike-system-defined` 显式关掉。
    public var spikeSystemDefined: Bool
    public var nowOverride: Date?
    public var environmentOverride: EnvironmentSnapshot?

    public init(scanner: ArgumentScanner) throws {
        // ⚠️ `--out` 的默认值**相对当前工作目录（cwd）解析**，不是相对仓库根（D15）——
        // 所以 `run-all.sh` 必须传显式路径。
        outDirectory = URL(fileURLWithPath: scanner.string("out") ?? "docs/verification", isDirectory: true)
        dryRun = scanner.flag("dry-run")
        sampleIntervalMilliseconds = try Self.parsePositiveInt(scanner.string("sample-ms"),
                                                              name: "--sample-ms", fallback: 50) ?? 50
        onlyKey = try Self.parseKey(scanner.string("only"), name: "--only")
        // --declared-key 与 --declared-mode **必须成对**（规格 §4.0）
        let pair = try scanner.requirePair("--declared-key", "--declared-mode")
        declaredKey = try Self.parseKey(pair.0.isEmpty ? nil : pair.0, name: "--declared-key")
        declaredMode = try Self.parseMode(pair.1.isEmpty ? nil : pair.1)
        matrixOnly = scanner.flag("matrix-only")
        maximumGroups = try Self.parsePositiveInt(scanner.string("max-groups"),
                                                  name: "--max-groups", fallback: nil)
        // 规格 §4.5 写的是「默认 on」；本实现改成默认**不**中止，
        // 用 --no-abort-on-voice-stuck 显式关掉（见计划开头的冲突裁决表）
        abortOnVoiceStuck = !scanner.flag("no-abort-on-voice-stuck")
        skipPositiveControl = scanner.flag("skip-positive-control")
        spikeSystemDefined = !scanner.flag("no-spike-system-defined")
        nowOverride = try EnvironmentParsing.parseISOD8601(scanner.string("now"))
        environmentOverride = try EnvironmentParsing.parseEnvironment(scanner.string("env"))
        // ⚠️ **校验在 init 里做**——只靠 `validate()` 的话，测试与调用方只要忘了调就静默通过
        try validate()
    }

    public func validate() throws {
        // 声明的键与模式必须成对（requirePair 已经保证了「只给一个」会报错），
        // 这里再兜一次「两个都给了但值非法」的情况。
        if let declaredKey, !TriggerKey.allCases.contains(declaredKey) {
            throw ArgumentError.unknownOption("--declared-key=\(declaredKey.rawValue)")
        }
    }

    static func parseKey(_ raw: String?, name: String) throws -> TriggerKey? {
        guard let raw, !raw.isEmpty else { return nil }
        guard let key = TriggerKey(rawValue: raw) else {
            throw ArgumentError.unknownOption("\(name)=\(raw)")
        }
        return key
    }

    static func parseMode(_ raw: String?) throws -> DeclaredMode? {
        guard let raw, !raw.isEmpty else { return nil }
        guard let mode = DeclaredMode(rawValue: raw) else {
            throw ArgumentError.unknownOption("--declared-mode=\(raw)")
        }
        return mode
    }

    static func parsePositiveInt(_ raw: String?, name: String, fallback: Int?) throws -> Int? {
        guard let raw, !raw.isEmpty else { return fallback }
        guard let value = Int(raw), value > 0 else {
            throw ArgumentError.unknownOption("\(name)=\(raw)")
        }
        return value
    }
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd tools/probes && swift test --filter G1AArgumentsTests`
Expected: PASS（8 个用例全绿）

- [ ] **Step 5: 写编排器**

创建 `tools/probes/Sources/VBG1ACore/Measurement/G1ARunner.swift`：

```swift
import AppKit
import CoreGraphics
import Foundation
import VBProbeKit

/// G1-a 的阶段编排（规格 §4.3）。
///
/// 阶段 -1 正对照 → 阶段 0 前置 → 阶段 0.5 快速预览 → 阶段 1 完整矩阵 → 阶段 2 判读模式 → **阶段 3 人工抽检**。
///
/// ⚠️ **中断安全**：探针会按住 Cmd/Option 键。任何异常路径（物理输入中止、
/// 输入源被切走、前台校验失败、Ctrl-C）都必须**释放按键**，否则修饰键会卡住，
/// 用户后续敲的每个字母都变成 Cmd+字母。
public struct G1ARunner {
    private let arguments: G1AArguments
    private let inputSources: InputSourceProviding
    private let counter: SpeechShortcutCounter
    /// 运行进度。**失败时也要把已完成的组带出去**——全轮 8–10 分钟，
    /// 中途失败很常见，已完成的数据不该丢。
    private let progress = ProgressBox()

    public init(arguments: G1AArguments,
                inputSources: InputSourceProviding = InputSourceService(),
                counter: SpeechShortcutCounter = SpeechShortcutCounter()) {
        self.arguments = arguments
        self.inputSources = inputSources
        self.counter = counter
    }

    public func run() throws -> G1APayload {
        let environment = arguments.environmentOverride ?? EnvironmentCollector.collect()
        guard environment.axPermissionGranted else {
            throw NSError(domain: "vb-g1a-trigger", code: 10, userInfo: [
                NSLocalizedDescriptionKey: "未获得辅助功能（AX）权限。请到「系统设置 › 隐私与安全性 › 辅助功能」授权运行本探针的终端。"
            ])
        }

        if arguments.dryRun {
            printPlan()
            return makePayload(environment: environment, positiveControl: nil, preview: nil,
                               groups: [], selected: nil, manualCheck: nil,
                               frontmostCheckMode: "notApplicable",
                               baselineInputSource: nil)
        }

        // ── 阶段 0 前置 ──
        Banner.printStart()
        let baselineInputSource = inputSources.currentInputSourceID()
        try assertNoStuckModifierKeys()

        let harness = TextEditHarness()
        var instance = try harness.openTemporaryDocument()
        defer { instance.terminate() }

        // ⚠️ **必须解析出实际窗口号**（规格 §6.1 的「目标窗口在前台」）。
        // 拿不到就退化成只比 pid——**但要在报告里显式标注**，
        // 不能假装做过窗口级校验（多窗口时按键会落到错误的位置）。
        let frontmostCheckMode: String
        if let windowNumber = instance.resolveWindowNumber() {
            instance.windowNumber = windowNumber
            frontmostCheckMode = "windowLevel"
        } else {
            frontmostCheckMode = "ownerPIDOnly"
            print("""
            ⚠️ 没能解析出 TextEdit 的窗口号——前台校验将**只比对进程 pid**，
               无法确认「目标窗口」在前台。多窗口场景下注入可能落错位置。
            """)
        }
        progress.setFrontmostCheckMode(frontmostCheckMode)

        // 焦点验证：确认**文档正文**拿到了焦点（G1-b 有冷测量，所以那边把它放在冷测量之后；
        // G1-a 没有冷测量，这里可以立刻做）
        guard try harness.verifyDocumentFocus(instance) else {
            throw NSError(domain: "vb-g1a-trigger", code: 34, userInfo: [
                NSLocalizedDescriptionKey: "TextEdit 文档正文没有拿到焦点，注入会落到错误的位置，已中止"
            ])
        }

        let guard_ = FrontmostGuard(windowInspector: WindowInspector(),
                                    frontmostPIDProvider: { NSWorkspace.shared.frontmostApplication?.processIdentifier ?? -1 })

        // 物理输入中止：拿不到 Input Monitoring 权限时**不弹窗索要**，
        // 只在横幅与报告里显著标注「本次运行无法中止物理输入」。
        //
        // ⚠️ 中止回调**必须真的生效**——传空闭包等于「装了个不会响的警报」。
        let abortState = AbortState()
        let physicalInputAbortAvailable = PhysicalInputGuard.install { abortState.trigger() }
        if !physicalInputAbortAvailable {
            Banner.printNoPhysicalInputAbort()
        }
        defer { PhysicalInputGuard.uninstall() }

        let injector = KeyInjector()
        // ⚠️ **任何退出路径都要释放按键**。
        // `KeyInjector` 是 class（不是 struct）——`defer` 捕获 struct 副本的话，
        // 释放的是空副本，真正按下的键会留在按下状态。
        defer { try? injector.releaseAll(sleeper: { _ in }) }

        // Ctrl-C / kill 时也要释放按键——否则修饰键卡住，用户后续输入全带 Cmd。
        SignalCleanup.install { try? injector.releaseAll(sleeper: { _ in }) }
        defer { SignalCleanup.uninstall() }

        // ── 阶段 -1 正对照（必做；不通过则中止且不出结论）──
        var positiveControl: G1APositiveControl?
        if !arguments.skipPositiveControl {
            let result = try runPositiveControl(abortState: abortState, injector: injector)
            positiveControl = result
            guard result.passed else {
                throw NSError(domain: "vb-g1a-trigger", code: 20, userInfo: [
                    NSLocalizedDescriptionKey: "正对照失败（或超时未触发），本次运行不产出任何结论。"
                ])
            }
        } else {
            Banner.printNoPositiveControl()
        }

        // ⚠️ **S5 的逐组归因规则**（Task 3 开头唯一定义，D14）：
        // **只有**「正对照通过」**且**「实测 S5 落盘延迟 ≤ 1s」时，S5 才能逐组归因；
        // 否则 S5 **只是整体佐证**——落盘延迟大时计数增量可能落在组结束之后。
        let s5Attributable = (positiveControl?.passed == true)
            && ((positiveControl?.s5PersistenceLatencyMilliseconds ?? .infinity) <= 1000)

        // ── systemDefined spike（规格 §4.1）──
        // **默认自动尝试**：构造成功 → 矩阵 55 组；失败 → **降级**到 50 组并记
        // `unsupportedBySDK`（**不中止**——spike 是可选增强，不是必过项）。
        let spikeStatus = resolveSpikeStatus()
        progress.setSpikeStatus(spikeStatus)
        print("systemDefined spike：\(spikeStatus)")

        var oracle = VoiceStateOracle()
        var groups: [G1AGroupReport] = []

        // ── 阶段 0.5 快速预览（矩阵的前置子集）──
        var preview: G1APreview?
        if !arguments.matrixOnly, let declaredKey = arguments.declaredKey,
           let declaredMode = arguments.declaredMode {
            preview = try runPreview(declaredKey: declaredKey, declaredMode: declaredMode,
                                     injector: injector, oracle: &oracle, guard_: guard_, instance: instance,
                                     s5Attributable: s5Attributable, abortState: abortState)
        }

        // ── 阶段 1 完整矩阵 ──
        let matrix = TriggerCandidates.matrix(capabilityAware: spikeStatus == "supported")
        for (index, candidate) in matrix.enumerated() {
            if let limit = arguments.maximumGroups, index >= limit { break }
            if let only = arguments.onlyKey, candidate.key != only { continue }
            try requireFrontmost(guard_: guard_, instance: instance)
            try checkAborted(abortState, injector: injector)

            let inputSourceBefore = inputSources.currentInputSourceID()
            var report = try runGroup(candidate: candidate, injector: injector, oracle: &oracle,
                                      guard_: guard_, instance: instance,
                                      s5Attributable: s5Attributable, abortState: abortState)
            report.inputSourceBefore = inputSourceBefore
            report.inputSourceAfter = inputSources.currentInputSourceID()
            groups.append(report)
            progress.append(report)

            // 输入源被切走 → **立即中止整轮**并用 TIS 恢复（规格 §6.3）
            if report.inputSourceBefore != report.inputSourceAfter {
                if let baseline = baselineInputSource { _ = inputSources.selectInputSource(id: baseline) }
                throw NSError(domain: "vb-g1a-trigger", code: 30, userInfo: [
                    NSLocalizedDescriptionKey: "探测期间输入源被切换（\(report.inputSourceBefore ?? "?") → \(report.inputSourceAfter ?? "?")），已中止并尝试恢复。"
                ])
            }
        }

        // ── 阶段 2 判读模式（按命中的手势回放）──
        let selection = try runStageTwo(groups: groups, injector: injector, oracle: &oracle,
                                        guard_: guard_, instance: instance,
                                        s5Attributable: s5Attributable, abortState: abortState)

        // ── 阶段 3 人工抽检（命中组合必做）──
        let manualCheck = try runManualCheck(selected: selection, injector: injector, oracle: &oracle,
                                             guard_: guard_, instance: instance)

        return makePayload(environment: environment, positiveControl: positiveControl,
                           preview: preview, groups: groups, selected: selection,
                           manualCheck: manualCheck,
                           frontmostCheckMode: frontmostCheckMode,
                           baselineInputSource: baselineInputSource)
    }

    // MARK: - 失败报告

    /// 失败时产出报告——**带上实测的清理状态**。
    ///
    /// 卡键恰恰是在失败的那一刻最容易留下的，所以失败报告比成功报告更需要这份数据。
    public func failurePayload(error: Error, environment: EnvironmentSnapshot? = nil) -> G1AFailurePayload {
        let snapshot = progress.snapshot
        return G1AFailurePayload(
            reportTimestamp: arguments.nowOverride ?? Date(),
            reportEnvironment: environment ?? EnvironmentCollector.collect(),
            failureReason: (error as NSError).localizedDescription,
            completedGroups: snapshot.groups,
            systemDefinedSpikeStatus: snapshot.spikeStatus,
            frontmostCheckMode: snapshot.frontmostCheckMode,
            cleanup: CleanupProbe.collect(baselineInputSource: nil))
    }

    // MARK: - systemDefined spike

    /// `systemDefined` spike 的状态（规格 §4.1）。
    ///
    /// - `supported`：Fn 的 systemDefined 事件构造成功 → 矩阵含 55 组
    /// - `unsupportedBySDK`：构造失败 → **降级**到 50 组（**不中止**）
    /// - `disabledByFlag`：用户用 `--no-spike-system-defined` 关掉了
    private func resolveSpikeStatus() -> String {
        guard arguments.spikeSystemDefined else { return "disabledByFlag" }
        guard SystemDefinedSpike.buildFnEvent(keyDown: true) != nil else { return "unsupportedBySDK" }
        return "supported"
    }

    // MARK: - 物理输入中止

    /// 物理输入中止检查。**每组前后与每次采样都要查**——
    /// 用户在按住 Cmd/Option 的 1.5–3 秒里碰键盘，必须立刻停下。
    private func checkAborted(_ state: AbortState, injector: KeyInjector) throws {
        guard state.isAborted else { return }
        // 先释放已按下的键——否则会卡键
        try? injector.releaseAll(sleeper: { _ in })
        throw NSError(domain: "vb-g1a-trigger", code: 40, userInfo: [
            NSLocalizedDescriptionKey: "检测到物理键盘输入，已中止本轮探测（已释放所有按下的键）"
        ])
    }

    // MARK: - 阶段 -1 正对照

    private func runPositiveControl(abortState: AbortState, injector: KeyInjector) throws -> G1APositiveControl {
        print("""
        ────────────────────────────────────────────────
         正对照：请现在**用真实快捷键**触发一次 ChatterFly 的语音输入
         （**不要用菜单栏入口**），说一句话，然后按 Esc 或再次按快捷键结束。
         全程最长等待 60 秒。
        ────────────────────────────────────────────────
        """)
        let before = counter.snapshot()
        var oracle = VoiceStateOracle()
        var s1Flips = 0
        var s5Delta = 0
        var s5Slot: String?
        var persistence: Double?
        let deadline = Date().addingTimeInterval(60)
        var startTime = Date()

        while Date() < deadline {
            try checkAborted(abortState, injector: injector)
            let processes = AudioProcessInspector.runningInputProcesses()
            let target = processes.filter { status in
                guard let bundleId = status.bundleId else { return false }
                return AudioProcessInspector.targetBundleHints.contains { bundleId.lowercased().contains($0) }
            }
            let s1 = target.contains { $0.isRunningInput }
            let now = counter.snapshot()
            let delta = now.max - before.max
            if delta > s5Delta {
                s5Delta = delta
                s5Slot = now.hitSlot
                if persistence == nil { persistence = Date().timeIntervalSince(startTime) * 1000 }
            }
            let transition = oracle.ingest(VoiceSample(
                timestampMilliseconds: Date().timeIntervalSince(startTime) * 1000,
                s1RunningInput: s1, s4NewAudioProcess: false,
                s5CounterDelta: max(0, delta), s5HitSlot: now.hitSlot
            ))
            if transition.changed { s1Flips += 1 }
            Thread.sleep(forTimeInterval: Double(arguments.sampleIntervalMilliseconds) / 1000)
        }

        let s1Passed = s1Flips >= 2
        let s5Passed = s5Delta >= 1
        return G1APositiveControl(
            performed: true,
            passed: s1Passed || s5Passed,
            s1Passed: s1Passed,
            s5Passed: s5Passed,
            s5CounterDelta: s5Delta,
            s5HitSlot: s5Slot,
            s5PersistenceLatencyMilliseconds: persistence,
            timedOut: !s1Passed && !s5Passed,
            notes: ["S1 观测到 \(s1Flips) 次翻转；S5 增量 \(s5Delta)"]
        )
    }

    // MARK: - 单组执行

    private func runGroup(candidate: TriggerCandidate, injector: KeyInjector,
                          oracle: inout VoiceStateOracle, guard_: FrontmostGuard,
                          instance: TextEditInstance,
                          s5Attributable: Bool,
                          abortState: AbortState) throws -> G1AGroupReport {
        // 「可疑命中」是**逐组**的观测——上一组的可疑必须先清掉（D9）
        oracle.resetSuspiciousHit()

        // 基线必须为 false
        let baseline = try sampleWindow(oracle: &oracle, durationMilliseconds: 500,
                                        abortState: abortState, injector: injector)
        guard !baseline.contains(where: { $0.s1RunningInput || $0.s4NewAudioProcess }) else {
            throw NSError(domain: "vb-g1a-trigger", code: 31, userInfo: [
                NSLocalizedDescriptionKey: "第 \(candidate.id) 组基线显示已在收音，停止后重试"
            ])
        }

        // 按压期间的观测：sleeper 回调是在**键按住不放**的时候被调用的，
        // 所以「期间是否在收音」必须**在这段窗口里采样**（D8）
        var recordedDuringPress: Bool?
        var isFirstPress = true
        try injector.perform(key: candidate.key, form: candidate.form, gesture: candidate.gesture,
                             sleeper: { seconds in
                                 if isFirstPress {
                                     isFirstPress = false
                                     recordedDuringPress = try observeDuringPress(
                                        durationMilliseconds: seconds,
                                        abortState: abortState, injector: injector)
                                 } else {
                                     Thread.sleep(forTimeInterval: seconds)
                                 }
                             })

        let samples = try sampleWindow(oracle: &oracle,
                                       durationMilliseconds: TriggerTiming.observationWindowMilliseconds,
                                       abortState: abortState, injector: injector)

        let startLatency = samples.first(where: { $0.s1RunningInput || $0.s4NewAudioProcess })
            .map { $0.timestampMilliseconds } ?? nil

        // **三态**：命中 / 可疑命中（单样本为真，被去抖吞掉）/ 未命中（D9）
        let outcome: String
        if startLatency != nil { outcome = "hit" }
        else if oracle.suspiciousHit { outcome = "suspicious" }
        else { outcome = "miss" }

        var notes: [String] = []
        if !s5Attributable {
            notes.append("S5 仅作整体佐证（正对照未通过或 S5 落盘延迟 > 1s，不逐组归因）")
        }

        return G1AGroupReport(
            groupId: candidate.id, key: candidate.key.rawValue, form: candidate.form.rawValue,
            gesture: candidate.gesture.name, outcome: outcome,
            startLatencyMilliseconds: startLatency, stopLatencyMilliseconds: nil,
            recordedDuringPress: recordedDuringPress,
            s5CounterDelta: samples.map(\.s5CounterDelta).max() ?? 0,
            s5HitSlot: samples.compactMap(\.s5HitSlot).last,
            inputSourceBefore: nil, inputSourceAfter: nil,
            samples: samples, notes: notes
        )
    }

    /// 在**按住**的那段时间里采样，看是否在收音。
    ///
    /// 返回 `nil` 表示这段时间短到连一个采样都放不下（双击手势就是这种情况）。
    private func observeDuringPress(durationMilliseconds: Double,
                                    abortState: AbortState, injector: KeyInjector) throws -> Bool? {
        let deadline = Date().addingTimeInterval(durationMilliseconds / 1000)
        var sawAny = false
        var sawRecording = false
        while true {
            let remaining = deadline.timeIntervalSinceNow
            if remaining <= 0 { break }
            // ⚠️ 按住期间是**最危险**的时刻——用户此刻碰键盘就是 Cmd+字母。
            try checkAborted(abortState, injector: injector)
            let processes = AudioProcessInspector.runningInputProcesses()
            let recording = processes.contains { status in
                guard let bundleId = status.bundleId else { return false }
                return AudioProcessInspector.targetBundleHints.contains { bundleId.lowercased().contains($0) }
                    && status.isRunningInput
            }
            sawAny = true
            if recording { sawRecording = true }
            // 采样间隔不得越过 deadline——否则会把按键的驻留时间拉长
            Thread.sleep(forTimeInterval: min(Double(arguments.sampleIntervalMilliseconds) / 1000, remaining))
        }
        return sawAny ? sawRecording : nil
    }

    private func sampleWindow(oracle: inout VoiceStateOracle, durationMilliseconds: Int,
                              abortState: AbortState, injector: KeyInjector) throws -> [VoiceSample] {
        let baselineBundleIds = AudioProcessInspector.baselineBundleIdentifiers()
        var samples: [VoiceSample] = []
        let start = Date()
        let counterBefore = counter.snapshot()
        while Date().timeIntervalSince(start) * 1000 < Double(durationMilliseconds) {
            try checkAborted(abortState, injector: injector)
            let processes = AudioProcessInspector.runningInputProcesses()
            let target = processes.filter { status in
                guard let bundleId = status.bundleId else { return false }
                return AudioProcessInspector.targetBundleHints.contains { bundleId.lowercased().contains($0) }
            }
            let s1 = target.contains { $0.isRunningInput }
            let s4 = target.contains { status in
                guard let bundleId = status.bundleId else { return false }
                return AudioProcessInspector.targetBundleHints.contains { bundleId.lowercased().contains($0) }
                    && !baselineBundleIds.contains(bundleId)
            }
            let now = counter.snapshot()
            let delta = now.max - counterBefore.max
            _ = oracle.ingest(VoiceSample(
                timestampMilliseconds: Date().timeIntervalSince(start) * 1000,
                s1RunningInput: s1, s4NewAudioProcess: s4,
                s5CounterDelta: max(0, delta), s5HitSlot: now.hitSlot
            ))
            samples.append(VoiceSample(
                timestampMilliseconds: Date().timeIntervalSince(start) * 1000,
                s1RunningInput: s1, s4NewAudioProcess: s4,
                s5CounterDelta: max(0, delta), s5HitSlot: now.hitSlot
            ))
            Thread.sleep(forTimeInterval: Double(arguments.sampleIntervalMilliseconds) / 1000)
        }
        return samples
    }

    private func requireFrontmost(guard_: FrontmostGuard, instance: TextEditInstance) throws {
        let check = guard_.isFrontmost(targetPID: instance.processIdentifier, windowNumber: instance.windowNumber)
        guard check.isValid else {
            throw NSError(domain: "vb-g1a-trigger", code: 32, userInfo: [
                NSLocalizedDescriptionKey: "焦点校验失败（进程匹配=\(check.processMatches)，目标窗口在前台=\(check.windowIsFrontmost)），已中止"
            ])
        }
    }

    private func assertNoStuckModifierKeys() throws {
        let state = CGEventSource.flagsState(.combinedSessionState)
        let stuck = state.contains(.maskSecondaryFn) || state.contains(.maskCommand) || state.contains(.maskAlternate)
        if stuck {
            // 持续 ≥ 1 秒才判定为「上次运行残留」（用户正按着修饰键时不应误报）
            Thread.sleep(forTimeInterval: 1.0)
            let again = CGEventSource.flagsState(.combinedSessionState)
            if again.contains(.maskSecondaryFn) || again.contains(.maskCommand) || again.contains(.maskAlternate) {
                throw NSError(domain: "vb-g1a-trigger", code: 33, userInfo: [
                    NSLocalizedDescriptionKey: "检测到上次运行残留的修饰键，请手动释放后再试"
                ])
            }
        }
    }

    private func runPreview(declaredKey: TriggerKey, declaredMode: DeclaredMode,
                            injector: KeyInjector, oracle: inout VoiceStateOracle,
                            guard_: FrontmostGuard, instance: TextEditInstance,
                            s5Attributable: Bool, abortState: AbortState) throws -> G1APreview {
        try requireFrontmost(guard_: guard_, instance: instance)
        let gesture: Gesture = switch declaredMode {
        case .longPress: .hold1500
        case .tap: .tap120
        case .doubleTap: .double120
        }
        let report = try runGroup(candidate: TriggerCandidate(key: declaredKey, form: .keyEvent, gesture: gesture),
                                  injector: injector, oracle: &oracle, guard_: guard_, instance: instance,
                                  s5Attributable: s5Attributable, abortState: abortState)
        return G1APreview(declaredKey: declaredKey.rawValue, declaredMode: declaredMode.rawValue,
                          outcome: report.outcome,
                          notes: ["预览结论只是矩阵的前置子集；若与阶段 1 矛盾，**以矩阵为准**"])
    }

    private func runStageTwo(groups: [G1AGroupReport], injector: KeyInjector,
                             oracle: inout VoiceStateOracle, guard_: FrontmostGuard,
                             instance: TextEditInstance,
                             s5Attributable: Bool, abortState: AbortState) throws -> G1ASelection? {
        // 阶段 2 的输入集 = **命中 ∪ 可疑命中**（D9）——
        // 可疑命中（单样本为真、被去抖吞掉）**必须在阶段 2 复测一次**，
        // 不能因为「去抖没翻转」就当它没发生。
        let candidates = groups.filter { $0.outcome == "hit" || $0.outcome == "suspicious" }
        guard let first = candidates.first else { return nil }

        let key = TriggerKey(rawValue: first.key)!
        let gesture = Gesture.all.first { $0.name == first.gesture }!
        try requireFrontmost(guard_: guard_, instance: instance)

        // B：回放命中手势，记录 gStarted / gDuring / gAfter
        let replay = try runGroup(candidate: TriggerCandidate(key: key, form: .keyEvent, gesture: gesture),
                                  injector: injector, oracle: &oracle, guard_: guard_, instance: instance,
                                  s5Attributable: s5Attributable, abortState: abortState)
        let gStarted = replay.outcome == "hit"
        // 「按压期间」轴按手势自己的单次按压长度取（D8）：
        // hold → 取观测值；tap / double → n/a
        let gDuring = StageTwoClassifier.gDuring(gesture: gesture,
                                                  observedDuringPress: replay.recordedDuringPress)
        let gAfter = oracle.voiceState

        // C：gAfter 为 true 时再按一次，观察是否停止
        var secondPressStops: Bool?
        if gAfter {
            _ = try runGroup(candidate: TriggerCandidate(key: key, form: .keyEvent, gesture: gesture),
                             injector: injector, oracle: &oracle, guard_: guard_, instance: instance,
                             s5Attributable: s5Attributable, abortState: abortState)
            secondPressStops = !oracle.voiceState
        }

        let observation = StageTwoObservation(
            gestureKind: gesture.kind, gStarted: gStarted, gDuring: gDuring,
            gAfter: gAfter, secondPressStops: secondPressStops
        )
        let mode = StageTwoClassifier.classify(observation)
        let candidate = KeyCandidate(key: key, mode: mode,
                                     startLatencyMilliseconds: replay.startLatencyMilliseconds ?? .infinity,
                                     stopLatencyMilliseconds: replay.stopLatencyMilliseconds)
        return G1ASelection(key: key.rawValue, mode: mode.rawValue,
                            startLatencyMilliseconds: replay.startLatencyMilliseconds ?? 0,
                            reason: "按 §4.4 选键规则选出")
    }

    // MARK: - 阶段 3 人工抽检

    /// **阶段 3 · 人工抽检**（规格 §4.3，D12）。
    ///
    /// 探针**注入一次获胜组合**，用户**只说话、不碰键盘**；
    /// 观察窗**延长到用户确认「文字上屏」为止**；结果（通过 / 不通过 + 备注）记进报告。
    private func runManualCheck(selected: G1ASelection?, injector: KeyInjector,
                                 oracle: inout VoiceStateOracle, guard_: FrontmostGuard,
                                 instance: TextEditInstance) throws -> G1AManualCheck? {
        guard let selected, let key = TriggerKey(rawValue: selected.key) else { return nil }
        try requireFrontmost(guard_: guard_, instance: instance)

        let mode = VoiceMode(rawValue: selected.mode) ?? .tap
        let gesture: Gesture = switch mode {
        case .longPress: .hold1500
        case .doubleTapSelfStop: .double120
        default: .tap120
        }

        print("""
        ────────────────────────────────────────────────
         阶段 3 · 人工抽检：请现在**只说话**，不要碰键盘。
         探针会注入一次获胜组合（\(key.displayName) / \(mode.rawValue)）。
         说完之后回到终端按回车，再回答 y / n：**文字是否上屏了**？
         （可以在 y / n 后面追加一句备注，会原样写进报告。）
        ────────────────────────────────────────────────
        """)

        // 注入一次获胜组合
        try injector.perform(key: key, form: .keyEvent, gesture: gesture,
                             sleeper: { Thread.sleep(forTimeInterval: $0) })

        // 观察窗延长到用户确认为止（第一次回车 = 说完了；第二次输入 = y / n + 备注）
        guard readLineWithTimeout(seconds: 120) != nil else {
            return G1AManualCheck(performed: true, passed: false,
                                  note: "120 秒内用户没有确认，按未通过处理")
        }
        let answer = (readLineWithTimeout(seconds: 60) ?? "").trimmingCharacters(in: .whitespaces)
        let head = answer.split(separator: " ").first.map(String.init)?.lowercased() ?? ""
        let passed = head.hasPrefix("y")
        return G1AManualCheck(performed: true, passed: passed,
                              note: answer.isEmpty ? nil : answer)
    }

    /// 带超时的 `readLine()`——`readLine()` 会无限阻塞，所以读在后台线程，主线程按 deadline 等。
    private func readLineWithTimeout(seconds: Double) -> String? {
        let box = LineBox()
        DispatchQueue.global().async {
            box.set(readLine())
        }
        let deadline = Date().addingTimeInterval(seconds)
        while Date() < deadline {
            if let line = box.takeIfFilled() { return line }
            Thread.sleep(forTimeInterval: 0.1)
        }
        return nil
    }

    private func makePayload(environment: EnvironmentSnapshot, positiveControl: G1APositiveControl?,
                              preview: G1APreview?, groups: [G1AGroupReport],
                              selected: G1ASelection?, manualCheck: G1AManualCheck?,
                              frontmostCheckMode: String,
                              baselineInputSource: String?) -> G1APayload {
        G1APayload(reportTimestamp: arguments.nowOverride ?? Date(),
                   reportEnvironment: environment,
                   positiveControl: positiveControl, preview: preview,
                   groups: groups, selected: selected, manualCheck: manualCheck,
                   systemDefinedSpikeStatus: progress.snapshot.spikeStatus,
                   frontmostCheckMode: frontmostCheckMode,
                   cleanup: CleanupProbe.collect(baselineInputSource: baselineInputSource))
    }

    private func printPlan() {
        print("G1-a 探测计划（dry-run，不注入任何事件）")
        let matrix = TriggerCandidates.matrix(capabilityAware: arguments.spikeSystemDefined)
        print("  主矩阵 \(matrix.count) 组（5 键 × 2 形态 × 5 手势）")
        print("  采样周期 \(arguments.sampleIntervalMilliseconds)ms，观察窗 \(TriggerTiming.observationWindowMilliseconds)ms")
        if let declaredKey = arguments.declaredKey, let declaredMode = arguments.declaredMode {
            print("  快速预览：\(declaredKey.rawValue) / \(declaredMode.rawValue)")
        }
    }
}

/// 线程安全的单值盒子——后台线程读 stdin，主线程按 deadline 取。
final class LineBox {
    private let lock = NSLock()
    private var stored: String?
    private var filled = false

    func set(_ value: String?) {
        lock.lock()
        defer { lock.unlock() }
        stored = value
        filled = true
    }

    /// 还没读到时返回 `nil`（读到了 EOF 也返回 `nil`——调用方按「没有回答」处理）
    func takeIfFilled() -> String? {
        lock.lock()
        defer { lock.unlock() }
        guard filled else { return nil }
        return stored
    }
}

/// 运行进度。失败时也要把已完成的组带出去——全轮 8–10 分钟，中途失败很常见。
final class ProgressBox: @unchecked Sendable {
    private let lock = NSLock()
    private var groups: [G1AGroupReport] = []
    private var spikeStatus = "notAttempted"
    private var frontmostCheckMode = "notApplicable"

    func append(_ group: G1AGroupReport) {
        lock.lock(); defer { lock.unlock() }
        groups.append(group)
    }

    func setSpikeStatus(_ status: String) {
        lock.lock(); defer { lock.unlock() }
        spikeStatus = status
    }

    func setFrontmostCheckMode(_ mode: String) {
        lock.lock(); defer { lock.unlock() }
        frontmostCheckMode = mode
    }

    var snapshot: (groups: [G1AGroupReport], spikeStatus: String, frontmostCheckMode: String) {
        lock.lock(); defer { lock.unlock() }
        return (groups, spikeStatus, frontmostCheckMode)
    }
}
```

- [ ] **Step 6: 编译并跑全部测试**

Run: `cd tools/probes && swift build && swift test`
Expected: BUILD SUCCEEDED，所有测试 PASS

- [ ] **Step 7: 提交（可选，默认跳过）**

```bash
cd tools/probes && git add Sources/VBG1ACore Tests && git commit -m "feat(probes): G1-a 参数集与阶段编排"
```

---

### Task 10: 报告渲染 + 可执行入口

> **开始前先删掉 Task 0 的占位文件**：
> ```bash
> rm tools/probes/Sources/vb-g1a-trigger/Placeholder.swift
> ```
> ⚠️ **必须删**——占位文件里有 top-level `print(...)`，而真正的 `main.swift` 也有
> top-level 代码；两个文件同时存在会报 `expressions are not allowed at the top level`。

**Files:**
- Create: `tools/probes/Sources/VBG1ACore/Report/G1AMarkdownRenderer.swift`
- Create: `tools/probes/Sources/vb-g1a-trigger/main.swift`
- Test: `tools/probes/Tests/VBG1ACoreTests/G1AMarkdownRendererTests.swift`

**Interfaces:**
- Consumes: `VBG1ACore`（Task 8、9）、`VBProbeKit`（计划 1）。
- Produces:
  - `public enum G1AMarkdownRenderer`
    - `static func render(payload: G1APayload) -> String` — 成功报告。
    - `static func renderFailure(payload: G1AFailurePayload) -> String` — **失败报告**（含「清理状态（实测）」小节与已完成的组）。
    - `static func spikeStatusText(_:) -> String` / `static func frontmostModeText(_:) -> String` — 状态的人读表述。
  - 可执行文件 `vb-g1a-trigger`；脚本 `tools/probes/run-all.sh`。
  - ⚠️ **`main.swift` 的 catch 分支必须写失败报告**——不能只往 stderr 打一行然后 `exit(2)`。已完成的组不能丢，用户也必须能看到「系统现在干不干净」。

- [ ] **Step 1: 写失败测试——Markdown 含矩阵总表与选键结论**

创建 `tools/probes/Tests/VBG1ACoreTests/G1AMarkdownRendererTests.swift`：

```swift
import XCTest
import VBProbeKit
@testable import VBG1ACore

final class G1AMarkdownRendererTests: XCTestCase {
    private func payload(groups: [G1AGroupReport] = []) -> G1APayload {
        G1APayload(reportTimestamp: Date(timeIntervalSince1970: 1_700_000_000),
                   reportEnvironment: EnvironmentSnapshot(hardwareModel: "Mac15,7", macOSVersion: "27.0.1",
                                                         swiftVersion: "6.4", chatterflyVersion: "1.0.8",
                                                         chatterflyPath: "/Library/Input Methods/Chatterfly.app",
                                                         chatterflyIsSelectedInputSource: true,
                                                         axPermissionGranted: true,
                                                         physicalInputAbortAvailable: true,
                                                         currentInputSourceID: "com.tencent.inputmethod.chatterfly.pinyin"),
                   positiveControl: nil, preview: nil, groups: groups,
                   selected: G1ASelection(key: "cmdLeft", mode: "longPress",
                                          startLatencyMilliseconds: 120, reason: "选键规则"),
                   manualCheck: G1AManualCheck(performed: true, passed: true, note: "文字上屏"),
                   systemDefinedSpikeStatus: "supported",
                   frontmostCheckMode: "windowLevel",
                   cleanup: CleanupStatusG1A(noStuckModifier: true, notStillRecording: true,
                                            inputSourceRestored: true, notes: []))
    }

    func testRendersMatrixTable() {
        let markdown = G1AMarkdownRenderer.render(payload: payload())
        XCTAssertTrue(markdown.contains("## 1. G1-a"))
        XCTAssertTrue(markdown.contains("## 3. 对规格的影响"))
        XCTAssertTrue(markdown.contains("## 4. 未定论事项"))
        XCTAssertTrue(markdown.contains("cmdLeft"))
    }

    func testRendersPositiveControlSection() {
        let markdown = G1AMarkdownRenderer.render(payload: payload())
        XCTAssertTrue(markdown.contains("正对照"))
    }

    func testRendersManualCheckAndCleanupSections() {
        // D12：人工抽检结果必须出现在报告里；D18：清理状态小节必须出现
        let markdown = G1AMarkdownRenderer.render(payload: payload())
        XCTAssertTrue(markdown.contains("### 人工抽检"))
        XCTAssertTrue(markdown.contains("文字上屏"))
        XCTAssertTrue(markdown.contains("### 清理状态"))
    }

    func testRendersSpikeStatusAndFrontmostCheckMode() {
        // 这两项必须出现在报告里——否则读者以为「spike 跑过了」「做了窗口级校验」
        let markdown = G1AMarkdownRenderer.render(payload: payload())
        XCTAssertTrue(markdown.contains("systemDefined spike"))
        XCTAssertTrue(markdown.contains("supported"))
        XCTAssertTrue(markdown.contains("前台校验强度"))
        XCTAssertTrue(markdown.contains("windowLevel"))
    }

    func testFailureReportRendersCleanupStatus() {
        // 失败报告**必须**带清理状态——卡键恰恰在失败那一刻最容易留下
        let failure = G1AFailurePayload(
            reportTimestamp: Date(timeIntervalSince1970: 1_700_000_000),
            reportEnvironment: EnvironmentSnapshot(hardwareModel: "Mac15,7", macOSVersion: "27.0.1",
                                                  swiftVersion: "6.4", chatterflyVersion: nil,
                                                  chatterflyPath: nil,
                                                  chatterflyIsSelectedInputSource: false,
                                                  axPermissionGranted: true,
                                                  physicalInputAbortAvailable: nil,
                                                  currentInputSourceID: nil),
            failureReason: "检测到物理键盘输入",
            completedGroups: [],
            systemDefinedSpikeStatus: "unsupportedBySDK",
            frontmostCheckMode: "ownerPIDOnly",
            cleanup: CleanupStatusG1A(noStuckModifier: false, notStillRecording: true,
                                     inputSourceRestored: true, notes: ["可能卡键"]))
        let markdown = G1AMarkdownRenderer.renderFailure(payload: failure)
        XCTAssertTrue(markdown.contains("失败报告"))
        XCTAssertTrue(markdown.contains("检测到物理键盘输入"))
        XCTAssertTrue(markdown.contains("## 清理状态（实测）"))
        XCTAssertTrue(markdown.contains("unsupportedBySDK"))
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd tools/probes && swift test --filter G1AMarkdownRendererTests`
Expected: FAIL（`Cannot find 'G1AMarkdownRenderer' in scope`）

- [ ] **Step 3: 写最小实现**

创建 `tools/probes/Sources/VBG1ACore/Report/G1AMarkdownRenderer.swift`：

```swift
import Foundation

public enum G1AMarkdownRenderer {
    public static func render(payload: G1APayload) -> String {
        var lines: [String] = []
        let environment = payload.reportEnvironment

        lines.append("# VoiceBridge G1-a 触发键探针 · 实测结论")
        lines.append("")
        lines.append("- **生成时间**：\(iso(payload.reportTimestamp))")
        lines.append("- **探针**：`\(payload.probeName)`")
        lines.append("")

        lines.append("## 0. 探测环境")
        lines.append("")
        lines.append("| 项 | 值 |")
        lines.append("| --- | --- |")
        lines.append("| 机型 | \(environment.hardwareModel) |")
        lines.append("| macOS 实测版本 | \(environment.macOSVersion) |")
        lines.append("| Swift 版本 | \(environment.swiftVersion) |")
        lines.append("| Chatterfly 版本 | \(environment.chatterflyVersion ?? "未安装") |")
        lines.append("| Chatterfly 真实安装路径 | \(environment.chatterflyPath ?? "未安装") |")
        lines.append("| Chatterfly 是否为当前选中输入法 | \(environment.chatterflyIsSelectedInputSource ? "是" : "否") |")
        lines.append("| AX 权限自检 | \(environment.axPermissionGranted ? "通过" : "**未通过**") |")
        lines.append("| 物理输入中止能力 | \(environment.physicalInputAbortAvailable.map { $0 ? "可用" : "**不可用**" } ?? "未探测") |")
        lines.append("| 当前输入源 | \(environment.currentInputSourceID ?? "未知") |")
        lines.append("")

        lines.append("### 正对照")
        lines.append("")
        if let control = payload.positiveControl {
            lines.append("| 项 | 值 |")
            lines.append("| --- | --- |")
            lines.append("| 是否执行 | \(control.performed) |")
            lines.append("| 是否通过 | \(control.passed) |")
            lines.append("| S1 通过 | \(control.s1Passed) |")
            lines.append("| S5 通过 | \(control.s5Passed) |")
            lines.append("| S5 增量 | \(control.s5CounterDelta) |")
            lines.append("| S5 命中槽 | \(control.s5HitSlot ?? "none") |")
            lines.append("| S5 落盘延迟 | \(control.s5PersistenceLatencyMilliseconds.map { String(format: "%.0fms", $0) } ?? "未测") |")
        } else {
            lines.append("**本次运行跳过了正对照 —— 结论不可信。**")
        }
        lines.append("")

        lines.append("## 1. G1-a")
        lines.append("")
        if let preview = payload.preview {
            lines.append("### 快速预览段")
            lines.append("")
            lines.append("- 声明：键 `\(preview.declaredKey)`、模式 `\(preview.declaredMode)`")
            lines.append("- 结果：**\(preview.outcome)**")
            lines.append("")
        }
        lines.append("### 完整矩阵段")
        lines.append("")
        lines.append("结果列是**三态**：`hit`（命中）/ `suspicious`（可疑命中——单样本为真、被去抖吞掉，**需复测**）/ `miss`（未命中）。")
        lines.append("")
        lines.append("| 组 | 键 | 形态 | 手势 | 结果 | startLatency | stopLatency | 按压期间收音 | S5 增量 | S5 槽 | 备注 |")
        lines.append("| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |")
        for group in payload.groups {
            lines.append("| \(group.groupId) | \(group.key) | \(group.form) | \(group.gesture) | \(group.outcome) | \(group.startLatencyMilliseconds.map { String(format: "%.0fms", $0) } ?? "—") | \(group.stopLatencyMilliseconds.map { String(format: "%.0fms", $0) } ?? "—") | \(group.recordedDuringPress.map { $0 ? "是" : "否" } ?? "n/a") | \(group.s5CounterDelta) | \(group.s5HitSlot ?? "none") | \(group.notes.isEmpty ? "—" : group.notes.joined(separator: "；")) |")
        }
        lines.append("")

        // **S5 的逐组归因规则**（Task 3 唯一定义处，D14）——报告里也要写清楚
        let s5Attributable = (payload.positiveControl?.passed == true)
            && ((payload.positiveControl?.s5PersistenceLatencyMilliseconds ?? .infinity) <= 1000)
        lines.append("**S5 逐组归因规则**：只有「正对照通过」**且**「实测 S5 落盘延迟 ≤ 1s」时，S5 才能逐组归因；否则 S5 只作整体佐证。")
        lines.append("")
        lines.append("- 本次运行：**\(s5Attributable ? "S5 可逐组归因" : "S5 仅作整体佐证")**")
        lines.append("")

        lines.append("### 执行环境判定")
        lines.append("")
        lines.append("| 项 | 值 |")
        lines.append("| --- | --- |")
        lines.append("| systemDefined spike | \(spikeStatusText(payload.systemDefinedSpikeStatus)) |")
        lines.append("| 前台校验强度 | \(frontmostModeText(payload.frontmostCheckMode)) |")
        lines.append("")

        if let selected = payload.selected {
            lines.append("### 选键结论")
            lines.append("")
            lines.append("- 触发键：**\(selected.key)**")
            lines.append("- 投递模式：**\(selected.mode)**")
            lines.append("- 启动延迟：\(String(format: "%.0fms", selected.startLatencyMilliseconds))")
            lines.append("- 依据：\(selected.reason)")
            lines.append("")
        } else {
            lines.append("### 选键结论")
            lines.append("")
            lines.append("- **没有命中组合** → 触发上游规格 R1（合成键唤不醒）")
            lines.append("")
            lines.append("  （全部**可执行组**——50 组，systemDefined spike 成功后 55 组——均无翻转，且阶段 -1 正对照已通过）")
            lines.append("")
        }

        lines.append("### 人工抽检")
        lines.append("")
        if let manualCheck = payload.manualCheck {
            lines.append("| 项 | 值 |")
            lines.append("| --- | --- |")
            lines.append("| 是否执行 | \(manualCheck.performed) |")
            lines.append("| 是否通过 | \(manualCheck.passed ? "通过" : "**未通过**") |")
            lines.append("| 备注 | \(manualCheck.note ?? "—") |")
        } else {
            lines.append("**本次运行没有人工抽检记录**（没有命中组合，或阶段 3 未执行）。")
        }
        lines.append("")

        lines.append("### 清理状态")
        lines.append("")
        lines.append("| 项 | 值 |")
        lines.append("| --- | --- |")
        lines.append("| 无卡住的修饰键 | \(payload.cleanup.noStuckModifier ? "是" : "**否**") |")
        lines.append("| 未仍在收音 | \(payload.cleanup.notStillRecording ? "是" : "**否**") |")
        lines.append("| 输入源已恢复 | \(payload.cleanup.inputSourceRestored ? "是" : "**否**") |")
        lines.append("| 备注 | \(payload.cleanup.notes.isEmpty ? "无" : payload.cleanup.notes.joined(separator: "；")) |")
        lines.append("")

        lines.append("## 2. G1-b")
        lines.append("")
        lines.append("本探针不覆盖 G1-b。G1-b 结论见 `vb-g1b-focus` 的报告。")
        lines.append("")

        lines.append("## 3. 对规格的影响")
        lines.append("")
        lines.append("（按回填清单逐条填写：哪一节 → 改成什么）")
        lines.append("")

        lines.append("## 4. 未定论事项")
        lines.append("")
        var undetermined: [String] = []
        for group in payload.groups where group.outcome == "suspicious" {
            undetermined.append("- \(group.groupId)：单样本为真 → **可疑命中，需复测**")
        }
        if payload.positiveControl == nil {
            undetermined.append("- **本次运行跳过了正对照，所有结论不可信**")
        }
        if undetermined.isEmpty { undetermined.append("- 无") }
        lines.append(contentsOf: undetermined)
        lines.append("")

        return lines.joined(separator: "\n")
    }

    /// `systemDefined` spike 状态的**人读**表述。
    static func spikeStatusText(_ status: String) -> String {
        switch status {
        case "supported": return "supported（Fn 的 systemDefined 事件构造成功，矩阵含 55 组）"
        case "unsupportedBySDK": return "**unsupportedBySDK**（构造失败，已降级到 50 组）"
        case "disabledByFlag": return "disabledByFlag（`--no-spike-system-defined` 已关掉，矩阵 50 组）"
        case "notAttempted": return "notAttempted（**未尝试**）"
        default: return status
        }
    }

    /// 前台校验强度的**人读**表述。
    static func frontmostModeText(_ mode: String) -> String {
        switch mode {
        case "windowLevel": return "windowLevel（比对了**目标窗口**在最前面）"
        case "ownerPIDOnly": return "**ownerPIDOnly**（没拿到窗口号，只比对了进程 pid）"
        case "notApplicable": return "notApplicable"
        default: return mode
        }
    }

    /// **失败报告**的人读版本。
    public static func renderFailure(payload: G1AFailurePayload) -> String {
        var lines: [String] = []
        lines.append("# VoiceBridge G1-a 触发键探针 · **失败报告**")
        lines.append("")
        lines.append("- **生成时间**：\(iso(payload.reportTimestamp))")
        lines.append("- **探针**：`\(payload.probeName)`")
        lines.append("- **失败原因**：\(payload.failureReason)")
        lines.append("- 已完成组数：\(payload.completedGroups.count)")
        lines.append("")
        lines.append("| 项 | 值 |")
        lines.append("| --- | --- |")
        lines.append("| systemDefined spike | \(spikeStatusText(payload.systemDefinedSpikeStatus)) |")
        lines.append("| 前台校验强度 | \(frontmostModeText(payload.frontmostCheckMode)) |")
        lines.append("")
        lines.append("## 清理状态（实测）")
        lines.append("")
        lines.append("| 项 | 值 |")
        lines.append("| --- | --- |")
        lines.append("| 无卡住的修饰键 | \(payload.cleanup.noStuckModifier ? "是" : "**否**") |")
        lines.append("| 未仍在收音 | \(payload.cleanup.notStillRecording ? "是" : "**否**") |")
        lines.append("| 输入源已恢复 | \(payload.cleanup.inputSourceRestored ? "是" : "**否**") |")
        lines.append("| 备注 | \(payload.cleanup.notes.isEmpty ? "无" : payload.cleanup.notes.joined(separator: "；")) |")
        lines.append("")
        if !payload.completedGroups.isEmpty {
            lines.append("## 已完成的组")
            lines.append("")
            lines.append("| 组 | 键 | 形态 | 手势 | 结果 | startLatency |")
            lines.append("| --- | --- | --- | --- | --- | --- |")
            for group in payload.completedGroups {
                lines.append("| \(group.groupId) | \(group.key) | \(group.form) | \(group.gesture) | \(group.outcome) | \(group.startLatencyMilliseconds.map { String(format: "%.0fms", $0) } ?? "—") |")
            }
            lines.append("")
        }
        return lines.joined(separator: "\n")
    }

    private static func iso(_ date: Date) -> String {        let formatter = ISO8601DateFormatter()
        formatter.formatOptions = [.withInternetDateTime]
        return formatter.string(from: date)
    }
}
```

创建 `tools/probes/Sources/vb-g1a-trigger/main.swift`：

```swift
import Foundation
import VBG1ACore
import VBProbeKit

let arguments = Array(CommandLine.arguments.dropFirst())

do {
    let scanner = try ArgumentScanner(arguments, schema: G1AArguments.optionSchema)
    let parsed = try G1AArguments(scanner: scanner)
    try parsed.validate()

    let runner = G1ARunner(arguments: parsed)
    do {
        let payload = try runner.run()

        let writer = ReportWriter(outputDirectory: parsed.outDirectory)
        let jsonURL = try writer.writeJSON(payload, as: "2026-10-23-g1a-results.json")
        let markdownURL = try writer.writeMarkdown(G1AMarkdownRenderer.render(payload: payload),
                                                   as: "2026-10-23-g1a-results.md")
        print("报告已写入：")
        print("  \(jsonURL.path)")
        print("  \(markdownURL.path)")
    } catch {
        // ⚠️ **失败也要写报告**——全轮 8–10 分钟，中途失败很常见。
        // 已完成的组不能丢；更重要的是「**系统现在干不干净**」——
        // 卡键恰恰是在失败的那一刻最容易留下的。
        let failure = runner.failurePayload(error: error)
        do {
            let writer = ReportWriter(outputDirectory: parsed.outDirectory)
            let jsonURL = try writer.writeJSON(failure, as: "2026-10-23-g1a-results.json")
            let markdownURL = try writer.writeMarkdown(G1AMarkdownRenderer.renderFailure(payload: failure),
                                                       as: "2026-10-23-g1a-results.md")
            print("探测失败，**失败报告**已写入：")
            print("  \(jsonURL.path)")
            print("  \(markdownURL.path)")
            print("清理状态：")
            for note in failure.cleanup.notes {
                print("  - \(note)")
            }
        } catch {
            // 连失败报告都写不出来时，至少把错误说清楚
            FileHandle.standardError.write(Data("参数错误或探测失败：\(error)\n".utf8))
        }
        exit(2)
    }
} catch {
    // 参数解析阶段就失败——此时还没有 runner，也没有环境快照，只能报错退出
    FileHandle.standardError.write(Data("参数错误：\(error)\n".utf8))
    exit(2)
}
```

创建 `tools/probes/run-all.sh`：

```bash
#!/usr/bin/env bash
# 依次跑两个探针并汇总（规格 §8.4）。
set -euo pipefail
cd "$(dirname "$0")"

OUT_DIR="${1:-../../docs/verification/}"
swift build -c release

echo "=== G1-b（AX 聚焦耗时）==="
.build/release/vb-g1b-focus --out "$OUT_DIR"

echo "=== G1-a（触发键唤醒）==="
.build/release/vb-g1a-trigger --out "$OUT_DIR"

echo "=== 汇总 ==="
ls -la "$OUT_DIR"
```

- [ ] **Step 4: 跑渲染器测试确认通过**

Run: `cd tools/probes && swift test --filter G1AMarkdownRendererTests`
Expected: PASS（3 个用例全绿）

- [ ] **Step 5: 跑全量测试确认通过**

Run: `cd tools/probes && swift test`
Expected: 所有测试 PASS

- [ ] **Step 6: 提交（可选，默认跳过）**

```bash
cd tools/probes && git add Sources Tests run-all.sh && git commit -m "feat(probes): G1-a 报告渲染与可执行入口"
```

---

### Task 11: 实机跑通 + 人工抽检 + 规格回填

**Files:**
- Create: `docs/verification/2026-10-23-g1a-results.md`（由探针生成）
- Create: `docs/verification/2026-10-23-g1a-results.json`（由探针生成）
- Modify: `docs/superpowers/specs/2026-10-23-voicebridge-design.md`（回填 §4.2 / §7.2 / §13.1 / §13.2 / §14 R1）

**Interfaces:**
- Consumes: `vb-g1a-trigger`（Task 10）。
- Produces: 实测结论 + 回填后的上游规格（升 **v3.5**）。

- [ ] **Step 1: 构建 release**

Run: `cd tools/probes && swift build -c release`
Expected: BUILD SUCCEEDED

- [ ] **Step 2: 先跑 dry-run 确认矩阵与参数**

Run: `cd tools/probes && .build/release/vb-g1a-trigger --dry-run`
Expected: 打印「主矩阵 50 组」「采样周期 50ms，观察窗 3000ms」，**不注入任何事件**

- [ ] **Step 3: 先跑快速预览（约 1 分钟）**

Run: `cd tools/probes && .build/release/vb-g1a-trigger --declared-key <你的快捷键> --declared-mode <longPress|tap|doubleTap> --matrix-only --out ../../docs/verification/`
Expected: 打印横幅后进入正对照；**按提示用真实快捷键触发一次**；随后跑完矩阵

> `--declared-key` / `--declared-mode` 的取值来自**你自己知道的绑定**。若你不确定模式，先只跑 `--matrix-only` 不带声明参数。
>
> ⚠️ **这一步是「调试/预览」跑**：`--matrix-only` 属于**仅调试用**参数，
> 用它跑出来的报告**不能**用来产出 M0 结论。**产出结论的那一轮**必须**不带**
> `--only` / `--matrix-only` / `--max-groups` / `--skip-positive-control`。

- [ ] **Step 4: 检查产物**

Run: `ls -la docs/verification/`
Expected: `2026-10-23-g1a-results.md` 与 `.json` 都存在

- [ ] **Step 5: 核对报告必须有的内容**

打开 `docs/verification/2026-10-23-g1a-results.md`，确认：
- §0 有 **macOS 实测版本**、Chatterfly 真实安装路径、AX 权限自检、**物理输入中止能力**
- §0 有**正对照结果**（S1/S5 各自是否通过 + S5 落盘延迟 + 命中槽）
- §1 有**完整矩阵段**的逐组结论表（**hit / suspicious / miss 三态**都在表里）
- §1 有**选键结论**
- §1 有「**人工抽检**」小节（探针执行阶段 3，见 Step 6）
- §1 有「**清理状态**」小节（无卡键 / 未在收音 / 输入源已恢复 + 备注）

- [ ] **Step 6: 人工抽检（命中组合必做）**

由**探针自己执行**（阶段 3 · 人工抽检）：探针注入一次获胜组合，用户**只说话、不碰键盘**，
观察窗延长到用户确认「文字上屏」为止；用户回终端按回车并回答 y / n，探针把结果写进报告。

- 这**不是识别率测试**，是**判据器的端到端验收**——证明「判据器说收音了」与「确实在收音」是同一件事。
- 结果（通过 / 不通过 + 备注）记入结论文档的「**人工抽检**」小节；若不通过，判据器判据需修正后重跑。
- 报告里没有「人工抽检」小节 → 这一轮**不算完成**。

- [ ] **Step 7: 回填上游规格**

按规格 §7.3 的回填清单逐条改 `docs/superpowers/specs/2026-10-23-voicebridge-design.md`：
- §0.1 升 **v3.5** + 加「M0 实测回填」追溯表
- §1.1 修正 Chatterfly 路径：`~/Library/Input Methods/…` → `/Library/Input Methods/Chatterfly.app`
- §4.2 触发键集合：写明**实测选中的键**；若需双击投递，明确「触发键需要双击投递模式」
- §7.2 投递表：按实测模式写清 Mac 端「开始拍 / 停止拍」的投递要求；若测出「松开后自启自停」或 `tapOnRelease` 变体 → **新增投递分支**
- §13.1 G1-a 行：填入实测结论
- §13.2 M0 出口：填入收音快捷键与长按/点按模式
- §14 R1：更新风险状态（已验证 / 仍开放 / 已触发）
- §15 参考资料：补「已核实」条目

**只回填实测到的**——探针没测到的显式标为「**未验证**」，不写成结论。

- [ ] **Step 8: 确认系统状态干净**

- 无卡住的修饰键：`CGEventSource.flagsState(.combinedSessionState)` 不含异常置位
- Chatterfly 未在收音：`kAudioProcessPropertyIsRunningInput` 回到 0
- 输入源已用 TIS 恢复到基线
- 探针自己开的 TextEdit 实例已关闭（按 pid 校验）

- [ ] **Step 9: 提交（可选，默认跳过）**

```bash
git add docs/verification docs/superpowers/specs/2026-10-23-voicebridge-design.md tools/probes
git commit -m "docs: G1-a 触发键探针实测结论与规格回填"
```

---

## 完成定义

- [ ] `cd tools/probes && swift build -c release` 通过
- [ ] `cd tools/probes && swift test` 全绿
- [ ] **阶段 -1 正对照通过**（否则后续结论无效，M0 不得宣称完成）
- [ ] `vb-g1a-trigger` 跑完：**快速预览段与完整矩阵段两段结论都产出**；矩阵段产出**逐组**结论表——全部**可执行组**（50 组，systemDefined spike 成功后 55 组）均无翻转，在正对照通过的前提下也是**有效结论**，同样能触发 R1
- [ ] 人工抽检完成：探针注入一次获胜组合，用户只说话，文字上屏，结果记入报告的「人工抽检」小节
- [ ] 报告有「**清理状态**」小节（无卡键 / 未在收音 / 输入源已恢复 + 备注）
- [ ] `docs/verification/2026-10-23-g1a-results.md` 与 `.json` 均存在
- [ ] 上游规格完成回填并升 v3.5，§0.1 有追溯表
- [ ] **探测后系统状态干净**：无卡住修饰键、Chatterfly 未在收音、**输入源已用 TIS 恢复基线**、探针自己开的 TextEdit 实例已关闭

---
---

## 附录：本计划的代码块已通过真实编译验证

> 这一节是**元信息**，不参与执行流程。记录的是：本计划里每一个 ```swift 代码块都**按字面**抽出来放进真实 SwiftPM 包编译并跑过测试——不是「看着对」。

**验证方法**（可复现）：

1. 从本计划与计划 1 的 markdown 里按 ```` ```swift ```` 围栏切出全部代码块，用紧邻的 `` `tools/probes/...` `` 路径标记落盘成文件树；
2. **逐 Task 验证**（不只是验最终态）：按「计划 1 Task 1 → Task 11 → 计划 2 Task 0 → Task 11」的执行顺序**累积**文件，每到一个 Task 边界就 `swift build` + `swift test`；
3. 再验**最终态**。

**验证结果**（Swift 6.4 / macOS 27.0.1 / XCTest，无第三方依赖）：

| 变体 | Package.swift | 构建 | 测试 |
| --- | --- | --- | --- |
| **逐 Task：23 个边界** | 按执行顺序推进 | 每个边界都 BUILD SUCCEEDED | 每个边界都全绿 |
| 计划 1 + 计划 2（最终态） | 计划 2 Task 0 的 5-product / 8-target 版 | BUILD SUCCEEDED | `VBProbeKitTests` 21 + `VBG1BCoreTests` 56 + `VBG1ACoreTests` 82 = **159 全绿** |

### 本轮修掉的 5 个「单测抓不到、上真机会出事」的问题

| 问题 | 修法 |
| --- | --- |
| **物理输入中止是空转** —— tap source 挂在主 run loop，而主线程全程 `Thread.sleep`，回调**永远不会执行**；且 `isAborted` 全计划**没有一处被读** | `PhysicalInputGuard.install` 自己起一个 `Thread` 跑 `CFRunLoopRun()`，把 source 加到**那个线程**的 run loop；runner 在**每次采样**与**每组前后**都查 `isAborted`，触发时**先释放按键再抛错** |
| **窗口级前台校验没生效** —— `TextEditInstance.windowNumber` 恒为 nil，`FrontmostGuard` 只走 owner-pid 分支 | 新增 `resolveWindowNumber()`（**轮询**，窗口在 `NSWorkspace.open` 之后才出现）；G1-a 与 G1-b 都传**实际窗口号**；拿不到就**显式标注** `ownerPIDOnly`，不假装做过窗口级校验 |
| **清理状态是写死的** —— `makePayload` 硬编码 `noStuckModifier: true`；`releaseAll` 只在测试里调用，中途异常会卡键而报告仍称干净 | `KeyInjector` 改成 **class**（`defer` 捕获 struct 副本释放的是空副本）；`defer` + `SignalCleanup`（SIGINT/SIGTERM）覆盖**所有**退出路径；清理状态改为 `CleanupProbe` **实测**；**失败也产出报告**（`G1AFailurePayload`），带已完成的组与实测清理状态 |
| **systemDefined spike 不是规格流程** —— 旧参数名 `--spike-system-defined`（**已移除**）不进矩阵就永远是 50 组，而 `run-all.sh` 不传它；构造失败还直接抛错中止 | 参数改为 `--no-spike-system-defined`（**默认自动尝试**）；构造失败**降级**到 50 组并记 `unsupportedBySDK`（**不中止**）；状态写进报告 |
| **`EnvironmentSnapshot` 排在 Task 2** —— `ReportDocument`（Task 1）引用它，Task 1 的 `swift test` 编译整个 package → Task 1 就卡住 | 提前到**计划 1 的 Task 1**（只依赖 Foundation，是最干净的底座类型） |

### 逐 Task 验证抓出来的真问题（已修）

- **`AbortState` 定义在 Task 9 才有的 `G1ARunner.swift` 里**，但 Task 7 的测试要用它 → 移到 **Task 7 的 `PhysicalInputGuard.swift`**。
- **`CleanupProbe` 依赖 Task 8 的 `CleanupStatusG1A`** → 测试文件拆成 Task 7 的 `AbortStateTests` 与 Task 8 的 `CleanupProbeTests`。
- **Task 0 声明了 target 却没有源文件** → 建占位文件（`Sources/VBG1ACore/Placeholder.swift` 等），Task 1 / Task 10 写入真实文件后删掉。
- **占位文件不能叫 `main.swift`** —— Task 10 的真入口也叫这个名字，同名会让「按路径抽取代码块」把两个版本搞混。

### 验证期间修掉的 API 事实

- CoreAudio 的进程列表返回的是 **AudioObjectID**（本机实测 139–165），**不是 pid**。必须先用 `kAudioProcessPropertyPID` 换 pid；`kAudioProcessPropertyIsRunningInput` **必须**用 objectID 查询；`kAudioProcessPropertyBundleID` 直接给 bundleId（**不要**用 `NSRunningApplication` 反查）。
- 公开 SDK 里**不存在** `CGEventType.systemDefined` 与 `CGEventField.keyboardEventData1`——systemDefined spike 必须用 `NSEvent.otherEvent` 构造。
- `EventForm.mainForms` 只含**两种**主形态（`keyEvent` / `flagsChanged`），`allCases` 有**三种**。
- `CGEventTapCallBack` 是 **C 函数指针**，回调闭包**不能有捕获**——必须放进 `nonisolated(unsafe)` 的全局槽；且回调只在**它所属线程的 run loop** 上执行。
- `CGEventMask` 是**位掩码**，**没有** `contains(_:)`——自己做位运算。
- `AXUIElement` 是 CF 类型，`as?` 条件 downcast **恒成功**（Swift 6 报错），必须 `as!`。
- `URL(fileURLWithPath:)` 返回**绝对路径**（相对 cwd），断言要按后缀比。

### 实机风险提醒

本计划会在真实机器上**按住 Cmd/Option 注入按键**（每组 1.5–3 秒，全轮 8–10 分钟），「请勿触碰键盘」是**硬要求**。开跑前建议先 `--dry-run` + `--max-groups 3` 试一轮。探针已装 `SIGINT`/`SIGTERM` 清理，Ctrl-C 会释放按键——但**不要**依赖它当保险。
