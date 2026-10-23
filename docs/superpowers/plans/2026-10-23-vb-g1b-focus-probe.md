# 计划 1 / VoiceBridge G1-b 聚焦探针 Implementation Plan

> **计划编号写死：本文件是「计划 1」。** 计划 2 是 `2026-10-23-vb-g1a-trigger-probe.md`（G1-a 触发键探针）。文件名里的 `g1b` / `g1a` 是**探针代号**，不是计划序号——不要按文件名里的代号去读「计划 N」的引用。

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在本机实测「查 `AXFocusedUIElement`」在原生 / Chromium / Electron 三类 App 上的聚焦耗时与命中情况，产出机器可读 + 人读两份 G1-b 结论，回填上游规格 §5.2、§13.1、§13.2 的占位表述。

**Architecture:** 一个 Swift Package（`tools/probes/`），library target `VBProbeKit` 放与探针种类无关的底座（参数解析、确定性报告、环境采集、AX 权限、窗口检查、TIS 输入源、TextEdit 靶场），library target `VBG1BCore` 放 G1-b 特有的测量与判读逻辑，executable target `vb-g1b-focus` 只做装配。真实 AX / CGWindowList / NSWorkspace 调用全部封在 protocol 后面，单测用 stub 实现（规格 §8.2：真实 AX / 注入 / 采样调用不写单测，由实机实跑覆盖）。

**Tech Stack:** Swift 6 / SwiftPM / AppKit / ApplicationServices (AX) / CoreGraphics (CGWindowList) / Carbon-HIToolbox (TIS) / XCTest。**无第三方依赖**（规格 §8.5）。

**上游依据:** `docs/superpowers/specs/2026-10-23-voicebridge-m0-probe-design.md` v1.5 §5（本文称为「探针规格」）；`docs/superpowers/specs/2026-10-23-voicebridge-design.md` v3.4 §5.2 / §13.1 / §13.2（本文称为「上游规格」）。

**前置:** 无。G1-b 不需要先做 G1-a（两者互不依赖，见「与计划 2 的关系」）。

---

## Global Constraints

以下约束适用于本计划**每一个** Task，不再逐条重复：

- **平台**：macOS 14+（本机实测 27.0.1，规格 F-26）；Swift 6；**无第三方依赖**——只用 AppKit、ApplicationServices、CoreGraphics、CoreAudio、Carbon-HIToolbox。
- **权限**（规格 §8.5）：探针**必须**有 AX 权限并在启动时自检；**不申请**屏幕录制权限（只读窗口 owner / bounds / alpha，见 F-24）；**不申请** Automation 权限（**禁用 `osascript`**）；**不主动索要** Input Monitoring（只探测可用性，拿不到就记录限制）。
- **超时常量**（规格 §5.4）：快路径 0.3s；建树归因的后继查询 3.0s（测完**立即调回 0.3s**）；全路径 3.0s；结束复位为 `0`。
- **测量顺序是硬约束**（顺序错了会得出假结论）：
  1. 冷快路径必须是**整个运行的第一次 AX 调用**；
  2. **全路径必须跑在稳态快路径之前**——宣告 `AXManualAccessibility` 正是让 Chromium / Electron 建好 Web 内容树的动作，稳态必须在**树已建好**之后测；先测稳态会得出「快路径无效」→ **假 R4 降级**；
  3. 测完**恢复宣告属性** + **复位 AX 超时为 0**。
- **前台必须是目标 App**（D3）：准备焦点之后，探针**自己重新激活目标 App**（等 0.5s），再校验「前台 pid == 目标 pid」且「目标窗口在前台」；不匹配就**中止该 App 的测量**（记 `setupVerified = false` + 说明）。Slack 的人工准备流程以「回到终端按回车」结束，此时前台是**终端**——终端的文本区是 `AXTextArea`，会被可编辑过滤器放行 → **假的「Slack 快路径有效」**。
- **`--slack-setup=autoclick` 只能是粗粒度规则**（D11）：探针**刻意不碰 AX**（冷测量前）且**不申请屏幕录制权限**，拿不到控件级数据，所以规则只能是「点 Slack 主窗口**下 1/4**」且「**排除右侧 15%**」（发送/附件控件通常在那里）。这条规则**不能保证不误点**，因此失败一律回退人工点击，**默认仍是 `manual`**。
- **「不要碰键盘」横幅的适用范围**：那条横幅属于**计划 2 的键盘注入阶段**；G1-b 的 Slack 步骤是它自己的**阶段内显式指令**（点 composer → 回终端按回车），两者不冲突。
- **全路径算法**（上游规格 §5.2，规格 §5.3.2 第 8 步）：先设 `AXManualAccessibility`；若返回 `attributeUnsupported` → 回退设置 `AXEnhancedUserInterface`；遍历上限 **5,000 个元素**；重试 **12 次 × 250ms**（最多 3 秒）。
- **可编辑目标判定**（上游规格 §5.2，规格 §5.2）：**三重**——① role 白名单 `AXTextArea` / `AXTextField` / `AXComboBox`；② subrole 排除 `AXSecureTextField`（role 与 subrole **双重**判断）；③ 语义关键词拒绝（清单见下）。
  拒绝词（中英双语，逐字来自上游规格 §5.2）：
  ```
  password, passcode, secret, api key, apikey, token, credit card,
  search, find, filter, address bar, settings, preferences, command palette,
  密码, 口令, 密钥, 令牌, 信用卡, 搜索, 查找, 筛选, 设置, 偏好
  ```
- **确定性报告**（规格 §8.1）：报告中的时间戳与环境字段**必须可注入**（`--now <iso8601>` / `--env <json>`），否则「同一输入 → 同一份 JSON 字节」的断言不成立。
- **产物位置**：探针代码进 `tools/probes/`；结论进 `docs/verification/`。
- **`--out` 的默认值相对当前工作目录解析**：`docs/verification/` 是**相对 cwd** 的路径，**不是**相对仓库根。所以 `run-all.sh` **必须传显式路径**（`--out "$OUT_DIR"`），否则在不同 cwd 下会把报告写到别处。
- **提交策略**（规格 §8.5）：**不提交、不推送**——提交与否由用户决定。本计划的每个 Task 末尾保留 `git add`/`git commit` 步骤但**标注为可选**，默认跳过。
- **写代码的通用红线**（上游规格 §5.2 + 规格 §5.6）：判定必须能区分「测量无效」与「测量有效但快路径无效」。**冷快路径超时是预期行为，不构成无效测量**——只有「准备动作失败」或「冷查询与 3s 后继查询**都**没解析出元素」才判无效。

### 探针参数

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `--out <dir>` | `docs/verification/` | 报告输出目录。**默认值相对当前工作目录（cwd）解析**——`run-all.sh` 必须传显式路径 |
| `--dry-run` | off | 只打印测量计划，**不做任何 AX 调用** |
| `--apps <list>` | `textedit,chrome,slack` | **仅调试用**：只测子集时**不能**用来产出 M0 结论 |
| `--warm-n <n>` | 10 | 稳态快路径采样次数（判定要求 10/10，N 不等于 10 时结论不成立） |
| `--chrome-setup <mode>` | `autofocus` | `autofocus` / `tab`——**一次运行只试一种** |
| `--slack-setup <mode>` | `manual` | `manual`（人工点击，默认）/ `autoclick`（粗粒度规则，见 Global Constraints） |
| `--page-load-wait <ms>` | 2000 | 页面加载等待（Chrome），记录实际值 |
| `--include-toolbar-controls` | off | **仅调试用**：额外测地址栏 / 查找栏（标 `nativeToolbarControl`，**不参与判定**） |
| `--measure-full-cold` | off | **仅调试用**：测冷全路径（参考值，**不参与判定**） |
| `--relaunch-app` | off | 为 R4 二次确认重启目标 App；**必须同时给 `--confirm-relaunch`** |
| `--confirm-relaunch` | 无 | 显式确认允许重启目标 App；未给出时 `--relaunch-app` 报错退出 |
| `--now <iso8601>` | 无 | **仅测试用**：注入报告时间戳（供「同一输入 → 同一份 JSON 字节」的断言） |
| `--env <json>` | 无 | **仅测试用**：注入环境快照（供「同一输入 → 同一份 JSON 字节」的断言） |

> **「仅调试用 / 仅测试用」的参数不参与 M0 结论**：用它们跑出来的报告只能用来定位问题，不能回填上游规格。

## 与计划 2 的关系

- 本计划建 `VBProbeKit`（共享底座）+ `VBG1BCore` + `vb-g1b-focus` + 测试 target `VBProbeKitTests` / `VBG1BCoreTests`。
- 计划 2（`2026-10-23-vb-g1a-trigger-probe.md`）建 `VBG1ACore` + `vb-g1a-trigger` + 测试 target `VBG1ACoreTests`，**依赖本计划的 Task 1-3 已完成**（要用 `VBProbeKit` 的参数解析、报告编码、环境采集、窗口检查、TextEdit 靶场——含 `openTemporaryDocument()` 的**两步准备**与 `verifyDocumentFocus(_:)`——以及 frontmost 守卫）。
- 计划 2 的 **Task 0** 会把本计划 Task 1 建的 3-target `Package.swift` 改成 5-product / 8-target 的最终版——**不要**在本计划里提前加 G1-a 的 target。
- 两个计划**互不阻塞**：G1-b 不注入按键，可独立跑完。

---

## File Structure

```
tools/probes/
├── Package.swift
├── Sources/
│   ├── VBProbeKit/                     # 共享底座（两套探针共用）
│   │   ├── CLI/
│   │   │   ├── ArgumentScanner.swift   # 通用 --key value / --flag 扫描
│   │   │   └── ProbeArgumentSet.swift  # 参数集协议 + 通用校验原语
│   │   ├── Report/
│   │   │   ├── ReportDocument.swift    # 报告信封（泛型）
│   │   │   ├── ReportEncoder.swift     # 确定性 JSON 编码
│   │   │   └── ReportWriter.swift      # 落盘（JSON + Markdown）
│   │   ├── Environment/
│   │   │   ├── EnvironmentSnapshot.swift
│   │   │   ├── EnvironmentCollector.swift
│   │   │   └── EnvironmentParsing.swift # --now / --env 解析（两探针共用）
│   │   ├── Permissions/
│   │   │   └── AXPermission.swift
│   │   ├── InputSource/
│   │   │   ├── InputSourceDescriptor.swift
│   │   │   └── InputSourceService.swift
│   │   ├── Windows/
│   │   │   ├── WindowInfo.swift
│   │   │   └── WindowInspector.swift
│   │   ├── Harness/
│   │   │   ├── TextEditHarness.swift
│   │   │   └── FrontmostGuard.swift
│   │   └── Support/
│   │       └── Banner.swift            # 终端横幅
│   ├── VBG1BCore/                      # G1-b 专属
│   │   ├── Placeholder.swift           # Task 1 占位，Task 4 写入真实类型后删除
│   │   ├── Arguments/
│   │   │   └── G1BArguments.swift
│   │   ├── AX/
│   │   │   ├── AXMessaging.swift       # 全局超时管理
│   │   │   ├── AccessibilityDeclaration.swift  # 宣告属性读写 + 归一化
│   │   │   ├── ElementDescriptor.swift
│   │   │   ├── EditableClassifier.swift
│   │   │   ├── FastPathProbe.swift
│   │   │   └── FullPathProbe.swift
│   │   ├── Targets/
│   │   │   ├── G1BTargetApp.swift
│   │   │   ├── FocusTargetPreparation.swift
│   │   │   ├── TabInjector.swift        # Chrome tab 模式的单键注入
│   │   │   └── MouseClicker.swift       # Slack autoclick 的左键点击
│   │   ├── Measurement/
│   │   │   ├── G1BAppReport.swift
│   │   │   ├── G1BPayload.swift
│   │   │   ├── G1BCleanupProbe.swift    # 实测清理状态（不写死）
│   │   │   ├── WarmStatistics.swift     # median / p95（最近秩）
│   │   │   └── G1BJudge.swift
│   │   ├── Report/
│   │   │   └── G1BMarkdownRenderer.swift
│   │   └── Runner/
│   │       └── G1BRunner.swift
│   └── vb-g1b-focus/
│       ├── Placeholder.swift           # Task 1 占位，Task 10 删掉并换成 main.swift
│       └── main.swift
└── Tests/
    ├── VBProbeKitTests/
    │   ├── ArgumentScannerTests.swift
    │   ├── ReportEncoderTests.swift
    │   ├── EnvironmentSnapshotTests.swift
    │   └── FrontmostGuardTests.swift
    └── VBG1BCoreTests/
        ├── Placeholder.swift             # Task 1 占位，Task 4 写入真实测试后删除
        ├── AccessibilityDeclarationTests.swift
        ├── ChromePageFixtureTests.swift
        ├── EditableClassifierTests.swift
        ├── G1BArgumentsTests.swift
        ├── G1BJudgeTests.swift
        ├── G1BMarkdownRendererTests.swift
        └── WarmStatisticsTests.swift
```

**文件职责约束**：每个文件**只有一个**清晰职责。平台调用（AX / CGWindowList / NSWorkspace / TIS）封装在对应文件里并由 protocol 暴露，**不要**在业务逻辑文件里直接调平台 API——否则无法写单测。

---

### Task 1: SwiftPM 骨架 + 通用参数扫描 + 确定性报告底座

**Files:**
- Create: `tools/probes/Package.swift`
- Create: `tools/probes/Sources/VBG1BCore/Placeholder.swift`（Task 4 写入真实类型后删除）
- Create: `tools/probes/Sources/vb-g1b-focus/Placeholder.swift`（Task 10 会删掉它并创建真正的 `main.swift`）
- Create: `tools/probes/Tests/VBG1BCoreTests/Placeholder.swift`（Task 4 写入真实测试后删除）
- Create: `tools/probes/Sources/VBProbeKit/CLI/ArgumentScanner.swift`
- Create: `tools/probes/Sources/VBProbeKit/CLI/ProbeArgumentSet.swift`
- Create: `tools/probes/Sources/VBProbeKit/Environment/EnvironmentSnapshot.swift`（**必须在本 Task**——`ReportDocument` 引用它）
- Create: `tools/probes/Sources/VBProbeKit/Report/ReportDocument.swift`
- Create: `tools/probes/Sources/VBProbeKit/Report/ReportEncoder.swift`
- Create: `tools/probes/Sources/VBProbeKit/Report/ReportWriter.swift`
- Test: `tools/probes/Tests/VBProbeKitTests/ArgumentScannerTests.swift`
- Test: `tools/probes/Tests/VBProbeKitTests/ReportEncoderTests.swift`

**Interfaces:**
- Consumes: 无（本计划第一个 Task）。
- Produces:
  - `public enum OptionKind` — `.flag` / `.value`。
  - `public struct ArgumentScanner` — `init(_ argv: [String], schema: [String: OptionKind]) throws`；`string(_:) -> String?`；`flag(_:) -> Bool`；`requireString(_:) throws -> String`；`requirePair(_:_:) throws -> (String, String)`；`requireFlagWith(_:_:) throws`。
    - ⚠️ **必须传 `schema`**——扫描器据此判定每个 token 是 flag 还是带值选项。**不在 schema 里的参数直接报错退出**，这样 `--nope` 不会被静默忽略、`--out` 不会被误当成 flag。
    - 键名**不带 `--`**：schema 里写 `"out"`，取值用 `scanner.string("out")`。
  - `public enum ArgumentError: Error, CustomStringConvertible` — `unknownOption(String)` / `missingValue(String)` / `unexpectedValue(String)` / `duplicatedOption(String)` / `unpairedOptions(String, String)` / `missingGuard(String, String)`。
  - `public protocol ProbeArgumentSet { init(scanner: ArgumentScanner) throws; func validate() throws }`。
  - `public struct EnvironmentSnapshot: Codable, Equatable` — 字段见下（**本 Task 创建**——`ReportDocument` 引用它）。
  - `public struct ReportDocument<Payload: Encodable>: Encodable` — 字段 `probe: String`、`generatedAt: Date`、`environment: EnvironmentSnapshot`、`payload: Payload`。
  - `public enum ReportEncoder { static func encode<Payload: Encodable>(_ document: ReportDocument<Payload>) throws -> Data }`。
  - `public protocol ProbePayload: Encodable` — `var probeName: String { get }`、`var reportTimestamp: Date { get }`、`var reportEnvironment: EnvironmentSnapshot { get }`。
  - `public struct ReportWriter` — `init(outputDirectory: URL)`；`func writeJSON(_ payload: some ProbePayload, as fileName: String) throws -> URL`；`func writeMarkdown(_ text: String, as fileName: String) throws -> URL`。

- [ ] **Step 1: 写失败测试——参数扫描与成对校验**

创建 `tools/probes/Tests/VBProbeKitTests/ArgumentScannerTests.swift`：

```swift
import XCTest
@testable import VBProbeKit

final class ArgumentScannerTests: XCTestCase {
    private let schema: [String: OptionKind] = [
        "out": .value,
        "dry-run": .flag,
        "declared-key": .value,
        "declared-mode": .value,
        "relaunch-app": .flag,
        "confirm-relaunch": .flag,
    ]

    func testParsesValueOptions() throws {
        let s = try ArgumentScanner(["--out", "/tmp/x", "--dry-run"], schema: schema)
        XCTAssertEqual(s.string("out"), "/tmp/x")
        XCTAssertTrue(s.flag("dry-run"))
        XCTAssertNil(s.string("missing"))
    }

    func testUnknownOptionThrows() {
        XCTAssertThrowsError(try ArgumentScanner(["--nope"], schema: schema)) { error in
            guard case ArgumentError.unknownOption(let name)? = error as? ArgumentError else {
                return XCTFail("expected unknownOption, got \(error)")
            }
            XCTAssertEqual(name, "--nope")
        }
    }

    func testMissingValueThrows() {
        // 「值参数后面没有取值」必须报错——否则 `--out` 会被静默当成开关
        XCTAssertThrowsError(try ArgumentScanner(["--out"], schema: schema)) { error in
            guard case ArgumentError.missingValue(let name)? = error as? ArgumentError else {
                return XCTFail("expected missingValue, got \(error)")
            }
            XCTAssertEqual(name, "--out")
        }
    }

    func testValueOptionFollowedByAnotherOptionThrows() {
        XCTAssertThrowsError(try ArgumentScanner(["--out", "--dry-run"], schema: schema))
    }

    func testFlagFollowedByValueThrows() {
        XCTAssertThrowsError(try ArgumentScanner(["--dry-run", "stray"], schema: schema))
    }

    func testDuplicatedOptionThrows() {
        XCTAssertThrowsError(try ArgumentScanner(["--out", "a", "--out", "b"], schema: schema))
    }

    func testPositionalArgumentThrows() {
        XCTAssertThrowsError(try ArgumentScanner(["stray"], schema: schema))
    }

    func testRequirePairRejectsOnlyOneSide() {
        let s = try? ArgumentScanner(["--declared-key", "fn"], schema: schema)
        XCTAssertThrowsError(try s!.requirePair("--declared-key", "--declared-mode"))
        let s2 = try? ArgumentScanner(["--declared-key", "fn", "--declared-mode", "tap"], schema: schema)
        XCTAssertNoThrow(try s2!.requirePair("--declared-key", "--declared-mode"))
    }

    func testRequireFlagWithRejectsFlagWithoutGuard() {
        let s = try? ArgumentScanner(["--relaunch-app"], schema: schema)
        XCTAssertThrowsError(try s!.requireFlagWith("--relaunch-app", "--confirm-relaunch"))
        let s2 = try? ArgumentScanner(["--relaunch-app", "--confirm-relaunch"], schema: schema)
        XCTAssertNoThrow(try s2!.requireFlagWith("--relaunch-app", "--confirm-relaunch"))
    }

    func testKeyNamesAreStoredWithoutLeadingDashes() {
        let s = try! ArgumentScanner(["--out", "/tmp/x"], schema: schema)
        XCTAssertEqual(s.string("out"), "/tmp/x")
        // 传带 `--` 的名字也取得到——调用方不必记住「存的时候剥了前缀」这件事
        XCTAssertEqual(s.string("--out"), "/tmp/x")
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd tools/probes && swift test --filter ArgumentScannerTests`
Expected: FAIL（`Cannot find 'ArgumentScanner' in scope` / 包还不存在）

- [ ] **Step 3: 写最小实现**

创建 `tools/probes/Package.swift`：

```swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "voicebridge-probes",
    platforms: [.macOS(.v14)],
    products: [
        .library(name: "VBProbeKit", targets: ["VBProbeKit"]),
        .library(name: "VBG1BCore", targets: ["VBG1BCore"]),
        .executable(name: "vb-g1b-focus", targets: ["vb-g1b-focus"]),
    ],
    targets: [
        .target(name: "VBProbeKit"),
        .target(name: "VBG1BCore", dependencies: ["VBProbeKit"]),
        .executableTarget(name: "vb-g1b-focus", dependencies: ["VBG1BCore"]),
        .testTarget(name: "VBProbeKitTests", dependencies: ["VBProbeKit"]),
        .testTarget(name: "VBG1BCoreTests", dependencies: ["VBG1BCore"]),
    ]
)
```

> **计划 2 追加**：`VBG1ACore` library target、`vb-g1a-trigger` executable target、`VBG1ACoreTests` test target，并往 `products` 加对应条目。

创建 `tools/probes/Sources/VBG1BCore/Placeholder.swift`：

```swift
// Task 1 的占位文件——Task 4 写入 AXMessaging 等真实类型后删除。
```

创建 `tools/probes/Sources/vb-g1b-focus/Placeholder.swift`：

```swift
// Task 1 的占位文件——Task 10 会删掉它并创建真正的 `main.swift`。
// ⚠️ 不能叫 `main.swift`：Task 10 的真入口也叫这个名字，
// 同名会让「按路径抽取代码块落盘」这一步把两个版本搞混。
print("vb-g1b-focus 尚未实现")
```

创建 `tools/probes/Tests/VBG1BCoreTests/Placeholder.swift`：

```swift
import XCTest
```

创建 `tools/probes/Sources/VBProbeKit/CLI/ArgumentScanner.swift`：

```swift
import Foundation

public enum ArgumentError: Error, CustomStringConvertible {
    case unknownOption(String)
    case missingValue(String)
    case unexpectedValue(String)
    case duplicatedOption(String)
    case unpairedOptions(String, String)
    case missingGuard(String, String)

    public var description: String {
        switch self {
        case .unknownOption(let o): return "未知参数：\(o)"
        case .missingValue(let o): return "参数 \(o) 缺少取值"
        case .unexpectedValue(let o): return "参数 \(o) 不接受取值"
        case .duplicatedOption(let o): return "参数 \(o) 重复给出"
        case .unpairedOptions(let a, let b): return "参数 \(a) 与 \(b) 必须成对给出"
        case .missingGuard(let f, let g): return "参数 \(f) 必须与 \(g) 同时给出"
        }
    }
}

/// 参数的**形态**。探针必须声明自己的全部参数，
/// 否则「未知参数」无法被识别，「值参数缺值」也无法与「开关参数」区分开。
public enum OptionKind: Sendable {
    /// `--dry-run`：开关，无取值
    case flag
    /// `--out <dir>`：取值
    case value
}

/// 通用参数扫描器。
///
/// **键名一律不带前导 `--`**：`--out` 存进 `values["out"]`，用 `string("out")` 取。
/// 带前导 `--` 传给 `requirePair` / `requireFlagWith` 也会被自动归一化。
///
/// 未知参数**一律报错退出**——探针跑错参数会静默污染结论，所以宁可早退。
public struct ArgumentScanner {
    private var values: [String: String] = [:]
    private var flags: Set<String> = []

    /// - Parameters:
    ///   - argv: 原始参数（不含可执行文件名）
    ///   - schema: 该探针支持的**全部**参数与形态。**不在表里的参数直接报错**。
    public init(_ argv: [String], schema: [String: OptionKind]) throws {
        var index = 0
        while index < argv.count {
            let token = argv[index]
            guard token.hasPrefix("--") else {
                throw ArgumentError.unknownOption(token)
            }
            let name = String(token.dropFirst(2))
            guard let kind = schema[name] else {
                throw ArgumentError.unknownOption(token)
            }
            if values[name] != nil || flags.contains(name) {
                throw ArgumentError.duplicatedOption(token)
            }
            switch kind {
            case .value:
                guard index + 1 < argv.count, !argv[index + 1].hasPrefix("--") else {
                    throw ArgumentError.missingValue(token)
                }
                values[name] = argv[index + 1]
                index += 2
            case .flag:
                flags.insert(name)
                index += 1
                // 开关后面跟了取值 → 参数表与实际用法不一致，直接报错
                if index < argv.count, !argv[index].hasPrefix("--") {
                    throw ArgumentError.unexpectedValue(argv[index])
                }
            }
        }
    }

    public func string(_ name: String) -> String? { values[normalize(name)] }
    public func flag(_ name: String) -> Bool { flags.contains(normalize(name)) }

    public func requireString(_ name: String) throws -> String {
        let key = normalize(name)
        guard let value = values[key] else { throw ArgumentError.missingValue(name) }
        return value
    }

    /// 规格 §4.5 / §5.7 要求的「必须成对给出」：只给其一即报错。
    public func requirePair(_ first: String, _ second: String) throws -> (String, String) {
        let firstKey = normalize(first)
        let secondKey = normalize(second)
        let hasFirst = values[firstKey] != nil
        let hasSecond = values[secondKey] != nil
        guard hasFirst == hasSecond else {
            throw ArgumentError.unpairedOptions(first, second)
        }
        guard hasFirst else { return ("", "") }
        return (values[firstKey]!, values[secondKey]!)
    }

    /// 规格 §5.7 的 `--relaunch-app` 必须配 `--confirm-relaunch`。
    public func requireFlagWith(_ flagName: String, _ guardName: String) throws {
        if flag(flagName) && !flag(guardName) {
            throw ArgumentError.missingGuard(flagName, guardName)
        }
    }

    private func normalize(_ name: String) -> String {
        return name.hasPrefix("--") ? String(name.dropFirst(2)) : name
    }
}
```

创建 `tools/probes/Sources/VBProbeKit/CLI/ProbeArgumentSet.swift`：

```swift
import Foundation

/// 每套探针的参数集实现这个协议；`validate()` 负责跨参数的成对 / 互斥校验。
public protocol ProbeArgumentSet {
    init(scanner: ArgumentScanner) throws
    func validate() throws
}
```

> ⚠️ **这个文件必须在本 Task 创建**——`ReportDocument`（Task 1 的 Step 3）
> 引用了 `EnvironmentSnapshot`。它原本排在 Task 2，但那样 Task 1 的 Step 4/6
> 跑 `swift test` 时**整个 package 都要编译**，`ReportDocument` 会因为找不到
> `EnvironmentSnapshot` 而编译失败——Task 1 就卡住了。
> 所以把它**提前**到本 Task：它本身只依赖 Foundation，是最干净的底座类型。

创建 `tools/probes/Sources/VBProbeKit/Environment/EnvironmentSnapshot.swift`：

```swift
import Foundation

/// 探测环境快照。字段与 `--env <json>` 的 JSON 键**一一对应**，所以外部注入与实机采集走同一套结构。
public struct EnvironmentSnapshot: Codable, Equatable {
    public var hardwareModel: String
    public var macOSVersion: String
    public var swiftVersion: String
    public var chatterflyVersion: String?
    public var chatterflyPath: String?
    public var chatterflyIsSelectedInputSource: Bool
    public var axPermissionGranted: Bool
    public var physicalInputAbortAvailable: Bool?
    public var currentInputSourceID: String?

    public init(
        hardwareModel: String,
        macOSVersion: String,
        swiftVersion: String,
        chatterflyVersion: String?,
        chatterflyPath: String?,
        chatterflyIsSelectedInputSource: Bool,
        axPermissionGranted: Bool,
        physicalInputAbortAvailable: Bool?,
        currentInputSourceID: String?
    ) {
        self.hardwareModel = hardwareModel
        self.macOSVersion = macOSVersion
        self.swiftVersion = swiftVersion
        self.chatterflyVersion = chatterflyVersion
        self.chatterflyPath = chatterflyPath
        self.chatterflyIsSelectedInputSource = chatterflyIsSelectedInputSource
        self.axPermissionGranted = axPermissionGranted
        self.physicalInputAbortAvailable = physicalInputAbortAvailable
        self.currentInputSourceID = currentInputSourceID
    }
}
```

创建 `tools/probes/Sources/VBProbeKit/Report/ReportDocument.swift`：

```swift
import Foundation

/// 报告信封：探针名 + 生成时间 + 环境 + 载荷。
/// `generatedAt` 与 `environment` 都可被外部注入，保证「同一输入 → 同一份 JSON 字节」。
public struct ReportDocument<Payload: Encodable>: Encodable {
    public var probe: String
    public var generatedAt: Date
    public var environment: EnvironmentSnapshot
    public var payload: Payload

    public init(probe: String, generatedAt: Date, environment: EnvironmentSnapshot, payload: Payload) {
        self.probe = probe
        self.generatedAt = generatedAt
        self.environment = environment
        self.payload = payload
    }
}
```

创建 `tools/probes/Sources/VBProbeKit/Report/ReportEncoder.swift`：

```swift
import Foundation

public enum ReportEncoder {
    /// `.sortedKeys` + `.iso8601` + 无 fractional seconds ⇒ 同一输入必然同一份字节。
    public static func encode<Payload: Encodable>(_ document: ReportDocument<Payload>) throws -> Data {
        let encoder = JSONEncoder()
        encoder.outputFormatting = [.sortedKeys, .withoutEscapingSlashes]
        encoder.dateEncodingStrategy = .iso8601
        return try encoder.encode(document)
    }
}
```

创建 `tools/probes/Sources/VBProbeKit/Report/ReportWriter.swift`：

```swift
import Foundation

/// 报告载荷的**共同契约**：探针名、报告时间戳、环境快照。
/// 三者都**可被外部注入**（`--now` / `--env`），否则「同一输入 → 同一份 JSON 字节」的断言不成立。
public protocol ProbePayload: Encodable {
    var probeName: String { get }
    var reportTimestamp: Date { get }
    var reportEnvironment: EnvironmentSnapshot { get }
}

public struct ReportWriter {
    public let outputDirectory: URL

    public init(outputDirectory: URL) {
        self.outputDirectory = outputDirectory
    }

    public func writeJSON(_ payload: some ProbePayload, as fileName: String) throws -> URL {
        let document = ReportDocument(
            probe: payload.probeName,
            generatedAt: payload.reportTimestamp,
            environment: payload.reportEnvironment,
            payload: payload
        )
        let data = try ReportEncoder.encode(document)
        return try write(data, as: fileName)
    }

    public func writeMarkdown(_ text: String, as fileName: String) throws -> URL {
        return try write(Data(text.utf8), as: fileName)
    }

    private func write(_ data: Data, as fileName: String) throws -> URL {
        try FileManager.default.createDirectory(at: outputDirectory, withIntermediateDirectories: true)
        let url = outputDirectory.appendingPathComponent(fileName)
        try data.write(to: url, options: .atomic)
        return url
    }
}
```

> `G1BPayload` 在 Task 8 声明为 `ProbePayload` 的实现者，字段名与协议**逐字对齐**。

- [ ] **Step 4: 跑测试确认通过**

Run: `cd tools/probes && swift test --filter ArgumentScannerTests`
Expected: PASS（10 个用例全绿）

- [ ] **Step 5: 写失败测试——报告字节确定性**

创建 `tools/probes/Tests/VBProbeKitTests/ReportEncoderTests.swift`（本 Task 已有 `ReportDocument` / `ReportEncoder` / `EnvironmentSnapshot` 三个类型可编译，载荷用最小 stub）：

```swift
import XCTest
@testable import VBProbeKit

private struct StubPayload: Encodable {
    let probeName = "stub"
    let reportTimestamp = Date(timeIntervalSince1970: 1_700_000_000)
    let reportEnvironment = EnvironmentSnapshot(
        hardwareModel: "Mac15,7",
        macOSVersion: "27.0.1",
        swiftVersion: "6.4",
        chatterflyVersion: "1.0.8.13459",
        chatterflyPath: "/Library/Input Methods/Chatterfly.app",
        chatterflyIsSelectedInputSource: true,
        axPermissionGranted: true,
        physicalInputAbortAvailable: false,
        currentInputSourceID: "com.tencent.inputmethod.chatterfly.pinyin"
    )
    let value = 42
}

final class ReportEncoderTests: XCTestCase {
    func testSameInputProducesIdenticalBytes() throws {
        let document = ReportDocument(
            probe: "stub",
            generatedAt: Date(timeIntervalSince1970: 1_700_000_000),
            environment: StubPayload().reportEnvironment,
            payload: StubPayload()
        )
        let a = try ReportEncoder.encode(document)
        let b = try ReportEncoder.encode(document)
        XCTAssertEqual(a, b)
        XCTAssertTrue(String(data: a, encoding: .utf8)!.contains("\"probe\":\"stub\""))
    }

    func testKeysAreSortedDeterministically() throws {
        let document = ReportDocument(
            probe: "stub",
            generatedAt: Date(timeIntervalSince1970: 1_700_000_000),
            environment: StubPayload().reportEnvironment,
            payload: StubPayload()
        )
        let json = String(data: try ReportEncoder.encode(document), encoding: .utf8)!
        // `.sortedKeys` 按**字典序**排顶层键：
        // environment < generatedAt < payload < probe（不是声明顺序）
        let environmentIndex = json.range(of: "\"environment\"")!.lowerBound
        let generatedIndex = json.range(of: "\"generatedAt\"")!.lowerBound
        let payloadIndex = json.range(of: "\"payload\"")!.lowerBound
        let probeIndex = json.range(of: "\"probe\"")!.lowerBound
        XCTAssertTrue(environmentIndex < generatedIndex)
        XCTAssertTrue(generatedIndex < payloadIndex)
        XCTAssertTrue(payloadIndex < probeIndex)
    }

    func testEnvironmentSnapshotIsDefinedBeforeTheReportTypesUseIt() {
        // `ReportDocument` 依赖 `EnvironmentSnapshot`——两者**必须**在同一个 target 里，
        // 否则 Task 1（报告底座）单独编译不过。
        let snapshot = StubPayload().reportEnvironment
        XCTAssertEqual(snapshot.macOSVersion, "27.0.1")
    }
}
```

- [ ] **Step 6: 跑测试确认通过**

Run: `cd tools/probes && swift test --filter ReportEncoderTests`
Expected: PASS（3 个用例全绿）

- [ ] **Step 7: 提交（可选，默认跳过）**

```bash
cd tools/probes && git add Package.swift Sources Tests && git commit -m "feat(probes): SwiftPM 骨架与确定性报告底座"
```

> 规格 §8.5：**不提交、不推送**，提交与否由用户决定。若用户未明确要求，跳过本步。

---

### Task 2: 环境采集 + AX 权限自检 + TIS 输入源读写

**Files:**
- Create: `tools/probes/Sources/VBProbeKit/Environment/EnvironmentCollector.swift`
- Create: `tools/probes/Sources/VBProbeKit/Environment/EnvironmentParsing.swift`
- Create: `tools/probes/Sources/VBProbeKit/Permissions/AXPermission.swift`
- Create: `tools/probes/Sources/VBProbeKit/InputSource/InputSourceDescriptor.swift`
- Create: `tools/probes/Sources/VBProbeKit/InputSource/InputSourceService.swift`
- Create: `tools/probes/Sources/VBProbeKit/Support/Banner.swift`
- Test: `tools/probes/Tests/VBProbeKitTests/EnvironmentSnapshotTests.swift`

**Interfaces:**
- Consumes: `EnvironmentSnapshot`（**Task 1 已创建**——因为 `ReportDocument` 引用它，不能排到本 Task）。
- Produces:
  - `public struct EnvironmentSnapshot: Codable, Equatable` — 字段见下（**类型在 Task 1 定义**，本 Task 只测它的注入行为）。
  - `public enum EnvironmentCollector { static func collect() -> EnvironmentSnapshot }`。
  - `public enum EnvironmentParsing { static func parseISOD8601(_ raw: String?) throws -> Date?; static func parseEnvironment(_ raw: String?) throws -> EnvironmentSnapshot? }`。
  - `public enum AXPermission { public static func isTrusted() -> Bool }`。
  - `public struct InputSourceDescriptor: Equatable` — `id: String`、`isSelectCapable: Bool`、`isEnabled: Bool`。
  - `public protocol InputSourceProviding { func currentInputSourceID() -> String?; func selectInputSource(id: String) -> Bool }`。
  - `public struct InputSourceService: InputSourceProviding` — 另有 `func selectableInputSources() -> [InputSourceDescriptor]`。
  - `public enum Banner` — `printStart(estimatedMinutes:)` / `printNoPhysicalInputAbort()` / `printNoPositiveControl()` / `printAbortedByUserInput()`。

- [ ] **Step 1: 写失败测试——环境快照可从 JSON 注入**

> ⚠️ `EnvironmentSnapshot` **类型本身在 Task 1 就已创建**（`ReportDocument` 依赖它）。
> 所以这一步的失败点不是「类型不存在」，而是**解码行为还没被固定下来**。

创建 `tools/probes/Tests/VBProbeKitTests/EnvironmentSnapshotTests.swift`：

```swift
import XCTest
@testable import VBProbeKit

final class EnvironmentSnapshotTests: XCTestCase {
    func testDecodesFromInjectedJSON() throws {
        let json = """
        {
          "hardwareModel": "Mac15,7",
          "macOSVersion": "27.0.1",
          "swiftVersion": "6.4",
          "chatterflyVersion": "1.0.8.13459",
          "chatterflyPath": "/Library/Input Methods/Chatterfly.app",
          "chatterflyIsSelectedInputSource": true,
          "axPermissionGranted": true,
          "physicalInputAbortAvailable": false,
          "currentInputSourceID": "com.tencent.inputmethod.chatterfly.pinyin"
        }
        """.data(using: .utf8)!
        let snapshot = try JSONDecoder().decode(EnvironmentSnapshot.self, from: json)
        XCTAssertEqual(snapshot.hardwareModel, "Mac15,7")
        XCTAssertTrue(snapshot.chatterflyIsSelectedInputSource)
        XCTAssertTrue(snapshot.axPermissionGranted)
        XCTAssertEqual(snapshot.physicalInputAbortAvailable, false)
        XCTAssertEqual(snapshot.currentInputSourceID, "com.tencent.inputmethod.chatterfly.pinyin")
    }

    func testRoundTripIsStable() throws {
        let original = EnvironmentSnapshot(
            hardwareModel: "Mac15,7",
            macOSVersion: "27.0.1",
            swiftVersion: "6.4",
            chatterflyVersion: nil,
            chatterflyPath: nil,
            chatterflyIsSelectedInputSource: false,
            axPermissionGranted: true,
            physicalInputAbortAvailable: nil,
            currentInputSourceID: nil
        )
        let data = try JSONEncoder().encode(original)
        let decoded = try JSONDecoder().decode(EnvironmentSnapshot.self, from: data)
        XCTAssertEqual(original, decoded)
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd tools/probes && swift test --filter EnvironmentSnapshotTests`
Expected: FAIL（`EnvironmentSnapshot` 类型**已存在**（Task 1），所以失败点应是**解码行为不符**——比如 `testDecodesFromInjectedJSON` 报错或字段值不对）

- [ ] **Step 3: 写最小实现**


创建 `tools/probes/Sources/VBProbeKit/Environment/EnvironmentCollector.swift`：

```swift
import Foundation

public enum EnvironmentCollector {
    /// Chatterfly 的真实安装位置（规格 F-01：**系统级** `/Library/Input Methods/`，不是 `~/Library/Input Methods/`）。
    public static let chatterflyBundlePath = "/Library/Input Methods/Chatterfly.app"
    public static let chatterflyBundleIdentifier = "com.tencent.inputmethod.chatterfly"

    public static func collect() -> EnvironmentSnapshot {
        let chatterfly = readChatterflyInfo()
        let inputSource = InputSourceService().currentInputSourceID()
        return EnvironmentSnapshot(
            hardwareModel: hardwareModel(),
            macOSVersion: ProcessInfo.processInfo.operatingSystemVersionString,
            swiftVersion: swiftVersion(),
            chatterflyVersion: chatterfly?.version,
            chatterflyPath: chatterfly?.path,
            chatterflyIsSelectedInputSource: inputSource?.contains("chatterfly") == true,
            axPermissionGranted: AXPermission.isTrusted(),
            physicalInputAbortAvailable: nil,   // Task 8（物理输入中止探测）回填
            currentInputSourceID: inputSource
        )
    }

    private struct ChatterflyInfo { let version: String?; let path: String? }

    private static func readChatterflyInfo() -> ChatterflyInfo? {
        let plistURL = URL(fileURLWithPath: chatterflyBundlePath).appendingPathComponent("Contents/Info.plist")
        guard let data = try? Data(contentsOf: plistURL),
              let plist = try? PropertyListSerialization.propertyList(from: data, format: nil) as? [String: Any]
        else { return nil }
        let version = (plist["CFBundleShortVersionString"] as? String)
            .flatMap { "\($0)(\(plist["CFBundleVersion"] as? String ?? ""))" }
        return ChatterflyInfo(version: version, path: chatterflyBundlePath)
    }

    private static func hardwareModel() -> String {
        var size = 0
        sysctlbyname("hw.model", nil, &size, nil, 0)
        var buffer = [CChar](repeating: 0, count: size)
        sysctlbyname("hw.model", &buffer, &size, nil, 0)
        return String(cString: buffer)
    }

    private static func swiftVersion() -> String {
        #if swift(>=6.4)
        return "6.4+"
        #elseif swift(>=6.0)
        return "6.0"
        #else
        return "<6.0"
        #endif
    }
}
```

创建 `tools/probes/Sources/VBProbeKit/Permissions/AXPermission.swift`：

```swift
import ApplicationServices

public enum AXPermission {
    /// 规格 §4.3 阶段 0 第 3 步：不通过 → 直接退出并提示去「系统设置 › 隐私与安全性 › 辅助功能」。
    /// 授权归属取决于「谁启动探针」，所以**每次运行都要查**。
    public static func isTrusted() -> Bool {
        return AXIsProcessTrusted()
    }
}
```

创建 `tools/probes/Sources/VBProbeKit/InputSource/InputSourceDescriptor.swift`：

```swift
import Foundation

public struct InputSourceDescriptor: Equatable {
    public var id: String
    public var isSelectCapable: Bool
    public var isEnabled: Bool

    public init(id: String, isSelectCapable: Bool, isEnabled: Bool) {
        self.id = id
        self.isSelectCapable = isSelectCapable
        self.isEnabled = isEnabled
    }
}
```

创建 `tools/probes/Sources/VBProbeKit/InputSource/InputSourceService.swift`：

```swift
import Carbon.HIToolbox
import Foundation

public protocol InputSourceProviding {
    func currentInputSourceID() -> String?
    func selectInputSource(id: String) -> Bool
}

/// 输入源的读取与恢复**全部走 TIS**（规格 §6.3）：
/// `defaults write com.apple.HIToolbox AppleSelectedInputSources` **不会实时切换输入源**，不能用来恢复。
public struct InputSourceService: InputSourceProviding {
    public init() {}

    public func currentInputSourceID() -> String? {
        // `TISCopyCurrentKeyboardInputSource()` 返回 `Unmanaged<TISInputSource>!`
        guard let unmanaged = TISCopyCurrentKeyboardInputSource() else { return nil }
        return stringProperty(unmanaged.takeUnretainedValue(), key: kTISPropertyInputSourceID)
    }

    public func selectableInputSources() -> [InputSourceDescriptor] {
        // `TISCreateInputSourceList` 返回 `Unmanaged<CFArray>!`——**不是** `[TISInputSource]`
        guard let unmanaged = TISCreateInputSourceList(nil, true) else { return [] }
        guard let list = unmanaged.takeRetainedValue() as? [TISInputSource] else { return [] }
        return list.compactMap { source in
            guard let id = stringProperty(source, key: kTISPropertyInputSourceID) else { return nil }
            return InputSourceDescriptor(
                id: id,
                isSelectCapable: boolProperty(source, key: kTISPropertyInputSourceIsSelectCapable),
                isEnabled: boolProperty(source, key: kTISPropertyInputSourceIsEnabled)
            )
        }
    }

    public func selectInputSource(id: String) -> Bool {
        guard let unmanaged = TISCreateInputSourceList(nil, false) else { return false }
        guard let list = unmanaged.takeRetainedValue() as? [TISInputSource] else { return false }
        guard let target = list.first(where: { stringProperty($0, key: kTISPropertyInputSourceID) == id }) else {
            return false
        }
        return TISSelectInputSource(target) == noErr
    }

    private func stringProperty(_ source: TISInputSource, key: CFString) -> String? {
        // `TISGetInputSourceProperty` 在 Swift 里被导入成 `UnsafeMutableRawPointer!`
        guard let raw = TISGetInputSourceProperty(source, key) else { return nil }
        let object = Unmanaged<CFTypeRef>.fromOpaque(raw).takeUnretainedValue()
        guard CFGetTypeID(object) == CFStringGetTypeID() else { return nil }
        return (object as! CFString) as String
    }

    private func boolProperty(_ source: TISInputSource, key: CFString) -> Bool {
        guard let raw = TISGetInputSourceProperty(source, key) else { return false }
        let object = Unmanaged<CFTypeRef>.fromOpaque(raw).takeUnretainedValue()
        let typeID = CFGetTypeID(object)
        if typeID == CFBooleanGetTypeID() {
            return CFBooleanGetValue(object as! CFBoolean)
        }
        if typeID == CFNumberGetTypeID() {
            var number: Int32 = 0
            CFNumberGetValue(object as! CFNumber, .sInt8Type, &number)
            return number != 0
        }
        return false
    }
}
```

创建 `tools/probes/Sources/VBProbeKit/Environment/EnvironmentParsing.swift`：

```swift
import Foundation

/// ISO8601 时间与环境 JSON 的解析——**两个探针共用**。
/// 放在 `VBProbeKit` 里是为了避免「两份实现悄悄漂移」。
public enum EnvironmentParsing {
    public static func parseISOD8601(_ raw: String?) throws -> Date? {
        guard let raw, !raw.isEmpty else { return nil }
        let formatter = ISO8601DateFormatter()
        formatter.formatOptions = [.withInternetDateTime]
        guard let date = formatter.date(from: raw) else {
            throw ArgumentError.unknownOption("--now=\(raw)")
        }
        return date
    }

    public static func parseEnvironment(_ raw: String?) throws -> EnvironmentSnapshot? {
        guard let raw, !raw.isEmpty else { return nil }
        guard let data = raw.data(using: .utf8) else {
            throw ArgumentError.unknownOption("--env 不是合法的 UTF-8 文本")
        }
        do {
            return try JSONDecoder().decode(EnvironmentSnapshot.self, from: data)
        } catch {
            throw ArgumentError.unknownOption("--env 的 JSON 结构与 EnvironmentSnapshot 不符：\(error)")
        }
    }
}
```

创建 `tools/probes/Sources/VBProbeKit/Support/Banner.swift`：

```swift
import Foundation

/// 终端横幅与提示（规格 §6.1）。
///
/// 横幅文案是**必含项**——它同时承担「不要碰键盘」与「会改输入源」两个告知。
public enum Banner {
    private static let rule = String(repeating: "=", count: 64)

    public static func printStart(estimatedMinutes: String = "8–10") {
        print("""
        \(rule)
         VoiceBridge G1 探测即将开始
         约 \(estimatedMinutes) 分钟（快速预览约 1 分钟）
         从此刻起请勿触碰键盘 —— 探针会按住 Cmd/Option 键
         输入法可能被切换；探测结束后会尽力恢复，失败会提示
        \(rule)
        """)
    }

    public static func printNoPhysicalInputAbort() {
        print("""
        ⚠️ 本次运行**无法中止物理输入**（拿不到 Input Monitoring 权限）。
           探测期间请**格外**不要碰键盘——探针会按住 Cmd/Option 键，
           你此时敲 Q 就是系统级 Cmd+Q，敲 W 就是关闭窗口。
        """)
    }

    public static func printNoPositiveControl() {
        print("""
        ⚠️ 本次运行**跳过了正对照**（--skip-positive-control）。
           所有结论**不可信**——「全部未命中」无法区分「唤不醒」与「判据器坏了」。
        """)
    }

    public static func printAbortedByUserInput() {
        print("""
        ⚠️ 检测到物理键盘输入，已中止本轮探测（避免与探针的按键叠加）。
        """)
    }
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd tools/probes && swift test --filter EnvironmentSnapshotTests`
Expected: PASS（2 个用例全绿）

- [ ] **Step 5: 提交（可选，默认跳过）**

```bash
cd tools/probes && git add Sources/VBProbeKit/Environment Sources/VBProbeKit/Permissions Sources/VBProbeKit/InputSource Tests && git commit -m "feat(probes): 环境采集、AX 权限自检与 TIS 输入源服务"
```

---

### Task 3: 窗口检查器 + TextEdit 靶场 + frontmost 守卫

**Files:**
- Create: `tools/probes/Sources/VBProbeKit/Windows/WindowInfo.swift`
- Create: `tools/probes/Sources/VBProbeKit/Windows/WindowInspector.swift`
- Create: `tools/probes/Sources/VBProbeKit/Harness/TextEditHarness.swift`
- Create: `tools/probes/Sources/VBProbeKit/Harness/FrontmostGuard.swift`
- Test: `tools/probes/Tests/VBProbeKitTests/FrontmostGuardTests.swift`

**Interfaces:**
- Consumes: `VBProbeKit`（Task 1、2）。
- Produces:
  - `public struct WindowInfo: Equatable` — `ownerPID: Int`、`ownerName: String`、`layer: Int`、`bounds: CGRect`、`alpha: Double`、`windowNumber: Int`。
  - `public protocol WindowInspecting { func allWindows() -> [WindowInfo]; func windows(ownerPID: Int) -> [WindowInfo] }`。
  - `public struct WindowInspector: WindowInspecting`。
  - `public struct TextEditHarness` — `init(inspector:)`；`func openTemporaryDocument() throws -> TextEditInstance`（**两步**：先启动**应用**、再在那个实例里打开文档）；`func verifyDocumentFocus(_ instance: TextEditInstance) throws -> Bool`；`static func makeTemporaryDocument() throws -> URL`；`static let textEditBundleURL: URL`。
  - `public struct TextEditInstance` — `processIdentifier: pid_t`、`temporaryFileURL: URL`、`windowNumber: Int?`、`func terminate() -> Bool`、`func findWindowNumber() -> Int?`、`func resolveWindowNumber(timeoutSeconds: Double = 5.0) -> Int?`。
    - ⚠️ `resolveWindowNumber` 是**轮询**——窗口在 `NSWorkspace.open` 之后才出现。规格 §6.1 的「目标窗口在前台」靠它；拿不到就必须**显式标注**「只比对了进程 pid」，不能悄悄退化。
  - `public struct FrontmostGuard` — `init(windowInspector: WindowInspecting, frontmostPIDProvider: @escaping () -> pid_t)`；`func isFrontmost(targetPID: pid_t, windowNumber: Int?) -> FrontmostCheck`。
  - `public struct WindowInspector`（扩展方法）— `func frontmostRegularWindowNumber(ownerPID: Int) -> Int?`。
  - `public struct FrontmostCheck` — `processMatches: Bool`、`windowIsFrontmost: Bool`、`var isValid: Bool { processMatches && windowIsFrontmost }`。

- [ ] **Step 1: 写失败测试——守卫的窗口级判定**

创建 `tools/probes/Tests/VBProbeKitTests/FrontmostGuardTests.swift`：

```swift
import XCTest
@testable import VBProbeKit

/// stub 的 `onScreenWindowsFrontToBack()` 返回**前到后**的顺序——
/// 与真实 `CGWindowListCopyWindowInfo(.optionOnScreenOnly)` 的语义一致。
private struct StubInspector: WindowInspecting {
    let ordered: [WindowInfo]
    func allWindows() -> [WindowInfo] { ordered }
    func onScreenWindowsFrontToBack() -> [WindowInfo] { ordered }
    func windows(ownerPID: Int) -> [WindowInfo] { ordered.filter { $0.ownerPID == ownerPID } }
}

private func info(pid: Int, layer: Int = 0, number: Int, alpha: Double = 1.0) -> WindowInfo {
    WindowInfo(ownerPID: pid, ownerName: "app", layer: layer, bounds: .zero, alpha: alpha, windowNumber: number)
}

final class FrontmostGuardTests: XCTestCase {
    func testAcceptsWhenTargetWindowIsTheFrontmostRegularWindow() {
        let inspector = StubInspector(ordered: [info(pid: 42, number: 1), info(pid: 99, number: 2)])
        let guard_ = FrontmostGuard(windowInspector: inspector, frontmostPIDProvider: { 42 })
        let check = guard_.isFrontmost(targetPID: 42, windowNumber: 1)
        XCTAssertTrue(check.processMatches)
        XCTAssertTrue(check.windowIsFrontmost)
        XCTAssertTrue(check.isValid)
    }

    func testRejectsWhenAnotherAppsWindowIsInFront() {
        let inspector = StubInspector(ordered: [info(pid: 99, number: 2), info(pid: 42, number: 1)])
        let guard_ = FrontmostGuard(windowInspector: inspector, frontmostPIDProvider: { 42 })
        let check = guard_.isFrontmost(targetPID: 42, windowNumber: 1)
        XCTAssertTrue(check.processMatches, "进程确实在前台")
        XCTAssertFalse(check.windowIsFrontmost, "但前面是别的 App 的窗口")
        XCTAssertFalse(check.isValid)
    }

    func testRejectsWhenAnotherAppIsFrontmost() {
        let inspector = StubInspector(ordered: [info(pid: 99, number: 2), info(pid: 42, number: 1)])
        let guard_ = FrontmostGuard(windowInspector: inspector, frontmostPIDProvider: { 99 })
        let check = guard_.isFrontmost(targetPID: 42, windowNumber: 1)
        XCTAssertFalse(check.processMatches)
        XCTAssertFalse(check.isValid)
    }

    func testRejectsWhenTheSameAppHasAnotherWindowInFront() {
        // 规格 §6.1：多文档窗口时，只校验进程会注错地方
        let inspector = StubInspector(ordered: [info(pid: 42, number: 7), info(pid: 42, number: 1)])
        let guard_ = FrontmostGuard(windowInspector: inspector, frontmostPIDProvider: { 42 })
        let check = guard_.isFrontmost(targetPID: 42, windowNumber: 1)
        XCTAssertTrue(check.processMatches)
        XCTAssertFalse(check.windowIsFrontmost)
        XCTAssertFalse(check.isValid)
    }

    func testIgnoresNonRegularLayersAndTransparentWindows() {
        let inspector = StubInspector(ordered: [
            info(pid: 99, layer: 3, number: 9),
            info(pid: 99, number: 8, alpha: 0),
            info(pid: 42, number: 1),
        ])
        let guard_ = FrontmostGuard(windowInspector: inspector, frontmostPIDProvider: { 42 })
        XCTAssertTrue(guard_.isFrontmost(targetPID: 42, windowNumber: 1).isValid)
    }

    func testWithoutWindowNumberFallsBackToOwnerPID() {
        let inspector = StubInspector(ordered: [info(pid: 42, number: 1), info(pid: 99, number: 2)])
        let guard_ = FrontmostGuard(windowInspector: inspector, frontmostPIDProvider: { 42 })
        XCTAssertTrue(guard_.isFrontmost(targetPID: 42, windowNumber: nil).isValid)
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd tools/probes && swift test --filter FrontmostGuardTests`
Expected: FAIL（`Cannot find 'WindowInfo' in scope`）

- [ ] **Step 3: 写最小实现**

创建 `tools/probes/Sources/VBProbeKit/Windows/WindowInfo.swift`：

```swift
import CoreGraphics
import Foundation

public struct WindowInfo: Equatable {
    public var ownerPID: Int
    public var ownerName: String
    public var layer: Int
    public var bounds: CGRect
    public var alpha: Double
    public var windowNumber: Int

    public init(ownerPID: Int, ownerName: String, layer: Int, bounds: CGRect, alpha: Double, windowNumber: Int) {
        self.ownerPID = ownerPID
        self.ownerName = ownerName
        self.layer = layer
        self.bounds = bounds
        self.alpha = alpha
        self.windowNumber = windowNumber
    }
}
```

创建 `tools/probes/Sources/VBProbeKit/Windows/WindowInspector.swift`：

```swift
import CoreGraphics
import Foundation

public protocol WindowInspecting {
    func allWindows() -> [WindowInfo]
    /// **前到后**（z-order）的屏幕内窗口列表。
    /// `CGWindowListCopyWindowInfo(.optionOnScreenOnly)` **本身就是前到后**排的——
    /// 它**不能**按 windowNumber 排序（windowNumber 是 ID，不是 z-order）。
    func onScreenWindowsFrontToBack() -> [WindowInfo]
    func windows(ownerPID: Int) -> [WindowInfo]
}

/// 只读 owner / bounds / alpha / layer——**不读 `kCGWindowName`**（它需要屏幕录制权限，规格 F-24）。
public struct WindowInspector: WindowInspecting {
    public init() {}

    public func allWindows() -> [WindowInfo] {
        return read(.optionAll)
    }

    public func onScreenWindowsFrontToBack() -> [WindowInfo] {
        // `.optionOnScreenOnly` 的返回顺序即**前到后**——保持原序，不做任何排序
        return read(.optionOnScreenOnly)
    }

    public func windows(ownerPID: Int) -> [WindowInfo] {
        return allWindows().filter { $0.ownerPID == ownerPID }
    }

    /// 该 pid 的**最前面**常规窗口号（layer == 0 且不透明）。
    ///
    /// 规格 §6.1 要求校验「**目标窗口**在前台」——只比进程 pid 不够：
    /// 同一个 App 有多个窗口时，前台的可能是**另一个**窗口。
    ///
    /// ⚠️ 用 `onScreenWindowsFrontToBack()`（**前到后**）而不是 `windows(ownerPID:)`——
    /// 后者的顺序来自 `.optionAll`，**不保证**是 z-order。
    public func frontmostRegularWindowNumber(ownerPID: Int) -> Int? {
        return onScreenWindowsFrontToBack()
            .first { $0.ownerPID == ownerPID && $0.layer == 0 && $0.alpha > 0 }?
            .windowNumber
    }

    private func read(_ option: CGWindowListOption) -> [WindowInfo] {
        guard let raw = CGWindowListCopyWindowInfo(option, kCGNullWindowID) as? [[String: Any]] else {
            return []
        }
        return raw.compactMap { entry in
            guard let ownerPID = entry[kCGWindowOwnerPID as String] as? Int else { return nil }
            let layer = entry[kCGWindowLayer as String] as? Int ?? 0
            let alpha = entry[kCGWindowAlpha as String] as? Double ?? 0
            let number = entry[kCGWindowNumber as String] as? Int ?? 0
            let ownerName = entry[kCGWindowOwnerName as String] as? String ?? ""
            let bounds = (entry[kCGWindowBounds as String] as? [String: Any])
                .map { CGRect(x: $0["X"] as? Double ?? 0, y: $0["Y"] as? Double ?? 0,
                              width: $0["Width"] as? Double ?? 0, height: $0["Height"] as? Double ?? 0) }
                ?? .zero
            return WindowInfo(ownerPID: ownerPID, ownerName: ownerName, layer: layer,
                              bounds: bounds, alpha: alpha, windowNumber: number)
        }
    }
}
```

创建 `tools/probes/Sources/VBProbeKit/Harness/FrontmostGuard.swift`：

```swift
import Foundation

public struct FrontmostCheck {
    public var processMatches: Bool
    public var windowIsFrontmost: Bool

    public var isValid: Bool { processMatches && windowIsFrontmost }
}

/// 规格 §6.1 的**硬校验**：只校验进程不够（多文档窗口时会注错地方），
/// 必须同时校验「**目标窗口**在前台」。不满足 → **立即中止**，不是「尽量」。
///
/// 判定方式：
/// - 进程维度：`NSWorkspace.shared.frontmostApplication` 的 pid == 目标 pid
/// - 窗口维度：`.optionOnScreenOnly` 的窗口列表**前到后**，
///   过滤掉非常规层（layer != 0）与透明窗口后，**最前面那个**必须就是目标窗口
///   （给了 `windowNumber` 就比窗口号，没给就比 owner pid）
///
/// ⚠️ **不能用 `as? WindowInspector` 做类型判断**——那会让 stub 落到 fallback 分支，
/// 而 fallback 一律判「目标在前台」，正好把要防的情况放过去。
public struct FrontmostGuard {
    private let windowInspector: WindowInspecting
    private let frontmostPIDProvider: () -> pid_t

    public init(windowInspector: WindowInspecting, frontmostPIDProvider: @escaping () -> pid_t) {
        self.windowInspector = windowInspector
        self.frontmostPIDProvider = frontmostPIDProvider
    }

    public func isFrontmost(targetPID: pid_t, windowNumber: Int?) -> FrontmostCheck {
        let processMatches = frontmostPIDProvider() == targetPID

        let ordered = windowInspector.onScreenWindowsFrontToBack()
            .filter { $0.layer == 0 && $0.alpha > 0 }
        let frontmostRegular = ordered.first

        let windowIsFrontmost: Bool
        if let windowNumber {
            // 指名道姓：最前面的常规窗口必须**就是**那个窗口号
            windowIsFrontmost = (frontmostRegular?.windowNumber == windowNumber)
        } else {
            windowIsFrontmost = (frontmostRegular?.ownerPID == Int(targetPID))
        }

        return FrontmostCheck(processMatches: processMatches, windowIsFrontmost: windowIsFrontmost)
    }
}
```

> `windowNumber` 保留在签名里是为了给后续「按窗口号精确匹配」留位置；当前判定用 owner + layer 已经能区分「目标 App 的常规窗口是不是最前的那个」。**不要**在没想清楚前把它删掉——计划 2 的注入落点校验也会用到。

创建 `tools/probes/Sources/VBProbeKit/Harness/TextEditHarness.swift`：

```swift
import AppKit
import ApplicationServices
import Foundation

public struct TextEditInstance {
    public var processIdentifier: pid_t
    public var temporaryFileURL: URL
    public var windowNumber: Int?
    private let inspector: WindowInspecting

    init(processIdentifier: pid_t, temporaryFileURL: URL, windowNumber: Int?, inspector: WindowInspecting) {
        self.processIdentifier = processIdentifier
        self.temporaryFileURL = temporaryFileURL
        self.windowNumber = windowNumber
        self.inspector = inspector
    }

    /// 只关**自己开的那一个**（按 pid 校验），否则每跑一次留一个 TextEdit 进程（规格 §4.3 阶段 0 末尾）。
    public func terminate() -> Bool {
        guard let app = NSRunningApplication(processIdentifier: processIdentifier) else { return false }
        return app.terminate()
    }

    public func findWindowNumber() -> Int? {
        return inspector.windows(ownerPID: Int(processIdentifier))
            .filter { $0.layer == 0 && $0.alpha > 0 }
            .first?.windowNumber
    }

    /// **轮询**解析出这个实例的窗口号。
    ///
    /// 规格 §6.1 要求校验「**目标窗口**在前台」——只比进程 pid 不够：
    /// 同一个 App 有多个窗口时，前台的可能是**另一个**窗口，
    /// 于是按键会落到错误的位置（这次按住 Cmd/Option 1.5–3 秒就全按错了地方）。
    ///
    /// 窗口在 `NSWorkspace.open` 之后才出现，所以必须**轮询**等它出现。
    /// 解析不到时返回 `nil`——调用方**必须**把它当作「窗口级校验不可用」显式记录，
    /// 不能悄悄退化成只比 pid。
    public func resolveWindowNumber(timeoutSeconds: Double = 5.0) -> Int? {
        let deadline = Date().addingTimeInterval(timeoutSeconds)
        while true {
            if let number = findWindowNumber() { return number }
            guard Date() < deadline else { return nil }
            Thread.sleep(forTimeInterval: 0.1)
        }
    }
}

/// 规格 §4.3 阶段 0 第 7 步：用 `NSWorkspace.openApplication(at:)`（F-42）启动 **新实例** 并拿到 pid。
/// 不用 `open -n`：那个拿不到新实例的 pid（F-42 / 修订 #78）。
///
/// ⚠️ **`openApplication(at:)` 收的是「应用包 URL」，不是文档 URL**（D10）。
/// 把临时文档路径传给它会报「找不到应用包」。
/// 所以准备动作是**两步**：
/// 1. `openApplication(at: TextEdit.app 的包 URL)` → 拿到 `NSRunningApplication`（pid）
/// 2. `NSWorkspace.shared.open(tempFileURL)` → 在**那个实例**里打开临时文档
public struct TextEditHarness {
    /// TextEdit 的**应用包**（macOS 14+ 位于 `/System/Applications/`）。
    public static let textEditBundleURL = URL(fileURLWithPath: "/System/Applications/TextEdit.app")

    /// 文档正文获得焦点时，焦点元素的 role 只能是这些之一。
    public static let focusedDocumentRoles: Set<String> = ["AXTextArea", "AXTextField", "AXComboBox"]

    private let inspector: WindowInspecting

    public init(inspector: WindowInspecting = WindowInspector()) {
        self.inspector = inspector
    }

    /// `mktemp -t vb-probe-target` 的等价物：**不用固定路径**，避免多会话冲突（修订 #42）。
    public static func makeTemporaryDocument() throws -> URL {
        let url = URL(fileURLWithPath: NSTemporaryDirectory())
            .appendingPathComponent("vb-probe-target-\(UUID().uuidString).txt")
        try "".write(to: url, atomically: true, encoding: .utf8)
        return url
    }

    public func openTemporaryDocument() throws -> TextEditInstance {
        let fileURL = try Self.makeTemporaryDocument()

        // ① 启动**应用**（包 URL）
        let configuration = NSWorkspace.OpenConfiguration()
        configuration.createsNewApplicationInstance = true
        configuration.activates = true

        var application: NSRunningApplication?
        var openError: Error?
        let semaphore = DispatchSemaphore(value: 0)
        NSWorkspace.shared.openApplication(at: Self.textEditBundleURL, configuration: configuration) { app, error in
            application = app
            openError = error
            semaphore.signal()
        }
        semaphore.wait()

        if let openError { throw openError }
        guard let application else {
            throw NSError(domain: "TextEditHarness", code: 1,
                          userInfo: [NSLocalizedDescriptionKey: "TextEdit 未启动"])
        }

        // ② 在**那个实例**里打开临时文档
        NSWorkspace.shared.open(fileURL)
        application.activate()
        return TextEditInstance(
            processIdentifier: application.processIdentifier,
            temporaryFileURL: fileURL,
            windowNumber: nil,
            inspector: inspector
        )
    }

    /// 焦点验证：确认**文档正文**拿到了焦点。
    ///
    /// ⚠️ 这是**一次 AX 调用**——必须放在**冷测量窗口结束之后**（D10），
    /// 否则它会污染冷值。G1-b 在冷测量之后调用它；G1-a 没有冷测量，
    /// 可以在打开文档后立刻调用（见计划 2 的编排器）。
    public func verifyDocumentFocus(_ instance: TextEditInstance) throws -> Bool {
        let appElement = AXUIElementCreateApplication(instance.processIdentifier)
        var value: CFTypeRef?
        let error = AXUIElementCopyAttributeValue(appElement, kAXFocusedUIElementAttribute as CFString, &value)
        // AXUIElement 是 CoreFoundation 类型——条件 downcast 恒成功，必须用强制转换
        guard error == .success, let value else { return false }
        let element = (value as! AXUIElement)

        var roleValue: CFTypeRef?
        guard AXUIElementCopyAttributeValue(element, kAXRoleAttribute as CFString, &roleValue) == .success,
              let role = roleValue as? String else { return false }
        return Self.focusedDocumentRoles.contains(role)
    }
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd tools/probes && swift test --filter FrontmostGuardTests`
Expected: PASS（6 个用例全绿）

- [ ] **Step 5: 提交（可选，默认跳过）**

```bash
cd tools/probes && git add Sources/VBProbeKit/Windows Sources/VBProbeKit/Harness Tests && git commit -m "feat(probes): 窗口检查器、TextEdit 靶场与 frontmost 硬校验"
```

---

### Task 4: AX 超时管理 + 可编辑目标三重判定

> **开始前先删掉 Task 1 的两个占位文件**：
> ```bash
> rm tools/probes/Sources/VBG1BCore/Placeholder.swift tools/probes/Tests/VBG1BCoreTests/Placeholder.swift
> ```
> 留着它们不会编译失败（空文件合法），但会让「这个 target 到底有哪些实现文件」变得不清楚。

**Files:**
- Create: `tools/probes/Sources/VBG1BCore/AX/AXMessaging.swift`
- Create: `tools/probes/Sources/VBG1BCore/AX/ElementDescriptor.swift`
- Create: `tools/probes/Sources/VBG1BCore/AX/EditableClassifier.swift`
- Test: `tools/probes/Tests/VBG1BCoreTests/EditableClassifierTests.swift`

**Interfaces:**
- Consumes: `VBProbeKit`（Task 1-3）。
- Produces:
  - `public enum AXMessaging { public static func setGlobalTimeout(_ seconds: Float); public static func resetGlobalTimeout() }`。
  - `public struct ElementDescriptor: Equatable` — `role: String?`、`subrole: String?`、`description: String?`、`title: String?`、`help: String?`、`placeholder: String?`。
  - `public enum EditableClassifier` — `public struct Verdict: Equatable { public var isEditable: Bool; public var rejectionReason: String? }`；`public static func classify(_ element: ElementDescriptor) -> Verdict`；`public static let allowedRoles: Set<String>`；`public static let rejectedKeywords: [String]`。

- [ ] **Step 1: 写失败测试——三重判定**

创建 `tools/probes/Tests/VBG1BCoreTests/EditableClassifierTests.swift`：

```swift
import XCTest
@testable import VBG1BCore

final class EditableClassifierTests: XCTestCase {
    private func element(role: String?, subrole: String? = nil, description: String? = nil,
                         title: String? = nil, help: String? = nil, placeholder: String? = nil) -> ElementDescriptor {
        ElementDescriptor(role: role, subrole: subrole, description: description,
                          title: title, help: help, placeholder: placeholder)
    }

    func testTextAreaIsEditable() {
        XCTAssertTrue(EditableClassifier.classify(element(role: "AXTextArea")).isEditable)
    }

    func testTextFieldIsEditable() {
        XCTAssertTrue(EditableClassifier.classify(element(role: "AXTextField")).isEditable)
    }

    func testComboBoxIsEditable() {
        XCTAssertTrue(EditableClassifier.classify(element(role: "AXComboBox")).isEditable)
    }

    func testUnknownRoleIsRejected() {
        let verdict = EditableClassifier.classify(element(role: "AXButton"))
        XCTAssertFalse(verdict.isEditable)
        XCTAssertEqual(verdict.rejectionReason, "notWhitelistedRole")
    }

    func testSecureTextFieldIsRejectedByRole() {
        let verdict = EditableClassifier.classify(element(role: "AXSecureTextField"))
        XCTAssertFalse(verdict.isEditable)
        XCTAssertEqual(verdict.rejectionReason, "secureTextField")
    }

    func testSecureTextFieldIsRejectedBySubroleEvenWhenRoleIsTextField() {
        // 上游规格 §5.2：只查 role 会漏掉 role 为 AXTextField、subrole 为 AXSecureTextField 的元素。
        let verdict = EditableClassifier.classify(element(role: "AXTextField", subrole: "AXSecureTextField"))
        XCTAssertFalse(verdict.isEditable)
        XCTAssertEqual(verdict.rejectionReason, "secureTextField")
    }

    func testSearchFieldSubrolePassesOnlyWhenNoKeywordHits() {
        // 规格 §5.2：AXSearchField 不再无条件计入可编辑——只在语义关键词检查通过时计入。
        let withoutKeyword = EditableClassifier.classify(element(role: "AXTextField", subrole: "AXSearchField"))
        XCTAssertTrue(withoutKeyword.isEditable)

        let withKeyword = EditableClassifier.classify(
            element(role: "AXTextField", subrole: "AXSearchField", description: "Search this page")
        )
        XCTAssertFalse(withKeyword.isEditable)
        XCTAssertEqual(withKeyword.rejectionReason, "semanticKeyword")
    }

    func testRejectsEnglishKeywordsAcrossAllTextAttributes() {
        for text in ["Password", "api key", "Address Bar", "command palette"] {
            let verdict = EditableClassifier.classify(element(role: "AXTextArea", description: text))
            XCTAssertFalse(verdict.isEditable, "应拒绝：\(text)")
            XCTAssertEqual(verdict.rejectionReason, "semanticKeyword")
        }
    }

    func testRejectsChineseKeywords() {
        for text in ["密码", "搜索", "筛选", "偏好"] {
            let verdict = EditableClassifier.classify(element(role: "AXTextArea", title: text))
            XCTAssertFalse(verdict.isEditable, "应拒绝：\(text)")
        }
    }

    func testRejectsKeywordFromHelpAndPlaceholder() {
        XCTAssertFalse(EditableClassifier.classify(element(role: "AXTextArea", help: "token here")).isEditable)
        XCTAssertFalse(EditableClassifier.classify(element(role: "AXTextArea", placeholder: "信用卡号")).isEditable)
    }

    func testKeywordMatchIsCaseInsensitive() {
        XCTAssertFalse(EditableClassifier.classify(element(role: "AXTextArea", description: "PASSWORD")).isEditable)
    }

    func testMissingRoleIsRejected() {
        let verdict = EditableClassifier.classify(element(role: nil))
        XCTAssertFalse(verdict.isEditable)
        XCTAssertEqual(verdict.rejectionReason, "missingRole")
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd tools/probes && swift test --filter EditableClassifierTests`
Expected: FAIL（`Cannot find 'EditableClassifier' in scope`）

- [ ] **Step 3: 写最小实现**

创建 `tools/probes/Sources/VBG1BCore/AX/AXMessaging.swift`：

```swift
import ApplicationServices

/// 规格 §5.4 超时阶梯。
///
/// 依据 F-29（`AXUIElement.h:389-392` 逐字）：对 **system-wide** 元素设置超时是
/// **本进程全局**超时，对本进程内所有 AX 调用生效；对其他 element 单独设置**只作用于该对象**。
/// 因此**只设 systemWide 即全局**。
public enum AXMessaging {
    public static let fastPathTimeout: Float = 0.3
    public static let slowQueryTimeout: Float = 3.0
    public static let fullPathTimeout: Float = 3.0

    public static func setGlobalTimeout(_ seconds: Float) {
        AXUIElementSetMessagingTimeout(AXUIElementCreateSystemWide(), seconds)
    }

    /// 传 `0` 表示**复位全局超时为默认值**（F-29）。测完**必须**复位，
    /// 否则后续全路径遍历也会被截到 300ms（上游规格 §5.2）。
    public static func resetGlobalTimeout() {
        AXUIElementSetMessagingTimeout(AXUIElementCreateSystemWide(), 0)
    }
}
```

创建 `tools/probes/Sources/VBG1BCore/AX/ElementDescriptor.swift`：

```swift
import Foundation

/// 一次 AX 解析出来的元素的**文本与角色快照**。
/// 四个文本属性（description / title / help / placeholder）**都要读**——上游规格 §5.2 的语义拒绝要用到它们。
public struct ElementDescriptor: Equatable {
    public var role: String?
    public var subrole: String?
    public var description: String?
    public var title: String?
    public var help: String?
    public var placeholder: String?

    public init(role: String?, subrole: String?, description: String?,
                title: String?, help: String?, placeholder: String?) {
        self.role = role
        self.subrole = subrole
        self.description = description
        self.title = title
        self.help = help
        self.placeholder = placeholder
    }
}
```

创建 `tools/probes/Sources/VBG1BCore/AX/EditableClassifier.swift`：

```swift
import Foundation

/// 上游规格 §5.2 的**三重**判定：role 白名单 → subrole 排除 → 语义关键词拒绝。
///
/// 产品要的是**可编辑目标**，不是「解析出某个东西」——
/// 一个永远解析出 `AXWebArea` 的 App 解析率可以是 10/10，但产品拿到的是不可编辑的元素。
public enum EditableClassifier {
    public static let allowedRoles: Set<String> = ["AXTextArea", "AXTextField", "AXComboBox"]

    /// 逐字来自上游规格 §5.2（中英双语）。
    public static let rejectedKeywords: [String] = [
        "password", "passcode", "secret", "api key", "apikey", "token", "credit card",
        "search", "find", "filter", "address bar", "settings", "preferences", "command palette",
        "密码", "口令", "密钥", "令牌", "信用卡", "搜索", "查找", "筛选", "设置", "偏好",
    ]

    public struct Verdict: Equatable {
        public var isEditable: Bool
        public var rejectionReason: String?

        public init(isEditable: Bool, rejectionReason: String? = nil) {
            self.isEditable = isEditable
            self.rejectionReason = rejectionReason
        }
    }

    public static func classify(_ element: ElementDescriptor) -> Verdict {
        guard let role = element.role, !role.isEmpty else {
            return Verdict(isEditable: false, rejectionReason: "missingRole")
        }
        // ② subrole 排除——role 与 subrole **双重**判断
        if role == "AXSecureTextField" || element.subrole == "AXSecureTextField" {
            return Verdict(isEditable: false, rejectionReason: "secureTextField")
        }
        // ① role 白名单
        guard allowedRoles.contains(role) else {
            return Verdict(isEditable: false, rejectionReason: "notWhitelistedRole")
        }
        // ③ 语义关键词拒绝
        let haystack = [element.description, element.title, element.help, element.placeholder]
            .compactMap { $0?.lowercased() }
            .joined(separator: "\n")
        for keyword in rejectedKeywords {
            if haystack.contains(keyword.lowercased()) {
                return Verdict(isEditable: false, rejectionReason: "semanticKeyword")
            }
        }
        return Verdict(isEditable: true)
    }
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd tools/probes && swift test --filter EditableClassifierTests`
Expected: PASS（12 个用例全绿）

- [ ] **Step 5: 提交（可选，默认跳过）**

```bash
cd tools/probes && git add Sources/VBG1BCore/AX Tests/VBG1BCoreTests && git commit -m "feat(probes): AX 超时管理与可编辑目标三重判定"
```

---

### Task 5: 快路径测量 + 建树归因的时序判据

**Files:**
- Create: `tools/probes/Sources/VBG1BCore/AX/FastPathProbe.swift`
- Create: `tools/probes/Sources/VBG1BCore/Measurement/WarmStatistics.swift`
- Test: `tools/probes/Tests/VBG1BCoreTests/WarmStatisticsTests.swift`

**Interfaces:**
- Consumes: `VBG1BCore`（Task 4 的 `ElementDescriptor` / `EditableClassifier`）。
- Produces:
  - `public struct FastPathMeasurement` — `queryMilliseconds: Double`、`attributeReadMilliseconds: Double`、`totalMilliseconds: Double`、`resolved: Bool`、`timedOut: Bool`、`errorCode: Int?`、`descriptor: ElementDescriptor?`。
  - `public enum FastPathProbe` — `static func measure(timeout: Float) -> FastPathMeasurement`；`static func copyFocusedElement(timeout: Float) -> (element: AXUIElement?, elapsedMilliseconds: Double, timedOut: Bool, errorCode: Int?)`。
  - `public struct WarmStatistics` — `init(samples: [Double])`；`median: Double?`、`p95: Double?`、`min: Double?`、`max: Double?`。

- [ ] **Step 1: 写失败测试——统计量**

创建 `tools/probes/Tests/VBG1BCoreTests/WarmStatisticsTests.swift`：

```swift
import XCTest
@testable import VBG1BCore

final class WarmStatisticsTests: XCTestCase {
    func testMedianOfEvenCount() {
        let stats = WarmStatistics(samples: [10, 20, 30, 40])
        XCTAssertEqual(stats.median, 25)
    }

    func testMedianOfOddCount() {
        let stats = WarmStatistics(samples: [10, 20, 30])
        XCTAssertEqual(stats.median, 20)
    }

    func testMinMax() {
        let stats = WarmStatistics(samples: [5, 9, 7])
        XCTAssertEqual(stats.min, 5)
        XCTAssertEqual(stats.max, 9)
    }

    func testEmptySampleHasNoStatistics() {
        let stats = WarmStatistics(samples: [])
        XCTAssertNil(stats.median)
        XCTAssertNil(stats.p95)
        XCTAssertNil(stats.min)
        XCTAssertNil(stats.max)
    }

    func testP95IsTheMaximumForTenSamples() {
        // N = 10 时 p95 落在最大值——判读用的是「零超时 + p95 阈值」三元组
        let stats = WarmStatistics(samples: [1, 2, 3, 4, 5, 6, 7, 8, 9, 100])
        XCTAssertEqual(stats.p95, 100)
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd tools/probes && swift test --filter WarmStatisticsTests`
Expected: FAIL（`Cannot find 'WarmStatistics' in scope`）

- [ ] **Step 3: 写最小实现**

创建 `tools/probes/Sources/VBG1BCore/Measurement/WarmStatistics.swift`：

```swift
import Foundation

public struct WarmStatistics {
    private let samples: [Double]

    public init(samples: [Double]) {
        self.samples = samples.sorted()
    }

    public var min: Double? { samples.first }
    public var max: Double? { samples.last }

    public var median: Double? {
        guard !samples.isEmpty else { return nil }
        let mid = samples.count / 2
        if samples.count % 2 == 1 { return samples[mid] }
        return (samples[mid - 1] + samples[mid]) / 2
    }

    /// **最近秩（nearest-rank）**分位数：index = ceil(p/100 × N) - 1。
    ///
    /// ⚠️ **不能用线性插值**：N = 10 时线性插值的 p95 会落在第 9 与第 10 个样本**之间**
    /// （例如 [1…9, 100] → 59.05），**最大那个慢样本被稀释掉了**——
    /// 规格 §5.6 的「零超时 + p95 阈值」三元组就失去了尾部敏感性。
    /// 最近秩让 N = 10 的 p95 **等于最大值**，正好对上这个判读口径。
    public var p95: Double? {
        guard !samples.isEmpty else { return nil }
        let rank = Int((0.95 * Double(samples.count)).rounded(.up))
        let index = Swift.max(0, Swift.min(rank - 1, samples.count - 1))
        return samples[index]
    }
}
```

创建 `tools/probes/Sources/VBG1BCore/AX/FastPathProbe.swift`：

```swift
import ApplicationServices
import Foundation

public struct FastPathMeasurement {
    public var queryMilliseconds: Double
    public var attributeReadMilliseconds: Double
    public var totalMilliseconds: Double
    public var resolved: Bool
    public var timedOut: Bool
    public var errorCode: Int?
    public var descriptor: ElementDescriptor?

    public init(queryMilliseconds: Double, attributeReadMilliseconds: Double, totalMilliseconds: Double,
                resolved: Bool, timedOut: Bool, errorCode: Int?, descriptor: ElementDescriptor?) {
        self.queryMilliseconds = queryMilliseconds
        self.attributeReadMilliseconds = attributeReadMilliseconds
        self.totalMilliseconds = totalMilliseconds
        self.resolved = resolved
        self.timedOut = timedOut
        self.errorCode = errorCode
        self.descriptor = descriptor
    }
}

/// 快路径 = **查 `AXFocusedUIElement` + 读安全过滤所需的四个文本属性**。
///
/// 规格 §5.6 明确：判定必须用**总耗时**（`totalMilliseconds`）而不是纯查询耗时——
/// 只测查询会**低估**真实聚焦成本，对着 300ms 预算判定会偏松。
public enum FastPathProbe {
    public static func copyFocusedElement(timeout: Float = AXMessaging.fastPathTimeout)
        -> (element: AXUIElement?, elapsedMilliseconds: Double, timedOut: Bool, errorCode: Int?) {
        AXMessaging.setGlobalTimeout(timeout)
        let systemWide = AXUIElementCreateSystemWide()
        var value: CFTypeRef?
        let start = CFAbsoluteTimeGetCurrent()
        let error = AXUIElementCopyAttributeValue(
            systemWide,
            kAXFocusedUIElementAttribute as CFString,
            &value
        )
        let elapsed = (CFAbsoluteTimeGetCurrent() - start) * 1000
        // AXUIElement 是 CoreFoundation 类型——**条件 downcast 恒成功**，必须用强制转换
        var element: AXUIElement?
        if error == .success, let value {
            element = (value as! AXUIElement)
        }
        // 「超时」与「本来就没有焦点元素」必须区分开：
        // 超时的特征是耗时贴着超时上限；用 90% 作为判据是保守近似。
        // 原始 AXError 码一并记录，便于事后判读。
        let timedOut = (element == nil) && (elapsed >= Double(timeout) * 1000 * 0.9)
        return (element, elapsed, timedOut, error == .success ? nil : Int(error.rawValue))
    }

    public static func measure(timeout: Float = AXMessaging.fastPathTimeout) -> FastPathMeasurement {
        let (element, queryMs, timedOut, errorCode) = copyFocusedElement(timeout: timeout)
        guard let element else {
            return FastPathMeasurement(queryMilliseconds: queryMs, attributeReadMilliseconds: 0,
                                       totalMilliseconds: queryMs, resolved: false, timedOut: timedOut,
                                       errorCode: errorCode, descriptor: nil)
        }
        let attributeStart = CFAbsoluteTimeGetCurrent()
        let descriptor = readDescriptor(element)
        let attributeMs = (CFAbsoluteTimeGetCurrent() - attributeStart) * 1000
        return FastPathMeasurement(queryMilliseconds: queryMs, attributeReadMilliseconds: attributeMs,
                                   totalMilliseconds: queryMs + attributeMs, resolved: true,
                                   timedOut: false, errorCode: nil, descriptor: descriptor)
    }

    /// 读 role / subrole / description / title / help / placeholder——
    /// **四个文本属性都要读**（规格 §5.5），供人工复核拒绝词。
    public static func readDescriptor(_ element: AXUIElement) -> ElementDescriptor {
        ElementDescriptor(
            role: stringAttribute(element, kAXRoleAttribute),
            subrole: stringAttribute(element, kAXSubroleAttribute),
            description: stringAttribute(element, kAXDescriptionAttribute),
            title: stringAttribute(element, kAXTitleAttribute),
            help: stringAttribute(element, kAXHelpAttribute),
            placeholder: stringAttribute(element, kAXPlaceholderValueAttribute)
        )
    }

    private static func stringAttribute(_ element: AXUIElement, _ attribute: String) -> String? {
        var value: CFTypeRef?
        let error = AXUIElementCopyAttributeValue(element, attribute as CFString, &value)
        guard error == .success else { return nil }
        return value as? String
    }
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd tools/probes && swift test --filter WarmStatisticsTests`
Expected: PASS（5 个用例全绿）

- [ ] **Step 5: 提交（可选，默认跳过）**

```bash
cd tools/probes && git add Sources/VBG1BCore/AX Sources/VBG1BCore/Measurement Tests && git commit -m "feat(probes): 快路径测量与稳态统计量"
```

---

### Task 6: 全路径（宣告 + 遍历 + 重试）+ 宣告属性归一化

**Files:**
- Create: `tools/probes/Sources/VBG1BCore/AX/FullPathProbe.swift`
- Create: `tools/probes/Sources/VBG1BCore/AX/AccessibilityDeclaration.swift`
- Test: `tools/probes/Tests/VBG1BCoreTests/AccessibilityDeclarationTests.swift`

**Interfaces:**
- Consumes: `VBG1BCore`（Task 4、5）。
- Produces:
  - `public enum AccessibilityDeclaration` — `public enum State: String { case declared, notDeclared }`；`static func normalize(_ raw: Bool?) -> State`；`static func read(appElement: AXUIElement) -> Bool?`；`static func read(appElement: AXUIElement, attribute: String) -> Bool?`；`static func restore(appElement: AXUIElement, declaredAttribute: String?, originalValue: Bool?) -> Bool`；`static let manualAttribute: String`；`static let enhancedAttribute: String`。
  - `public struct FullPathResult` — `hit: Bool`、`durationMilliseconds: Double`、`descriptor: ElementDescriptor?`、`usedEnhancedFallback: Bool`、`attempts: Int`、`declaredAttribute: String?`。
  - `public enum FullPathProbe` — `static let maximumElements = 5000`、`static let maximumAttempts = 12`、`static let retryIntervalMilliseconds = 250`；`static func run(appElement: AXUIElement, timeout: Float) -> FullPathResult`。

- [ ] **Step 1: 写失败测试——宣告状态归一化**

创建 `tools/probes/Tests/VBG1BCoreTests/AccessibilityDeclarationTests.swift`：

```swift
import XCTest
@testable import VBG1BCore

final class AccessibilityDeclarationTests: XCTestCase {
    func testNilNormalizesToNotDeclared() {
        XCTAssertEqual(AccessibilityDeclaration.normalize(nil), .notDeclared)
    }

    func testFalseNormalizesToNotDeclared() {
        XCTAssertEqual(AccessibilityDeclaration.normalize(false), .notDeclared)
    }

    func testTrueNormalizesToDeclared() {
        XCTAssertEqual(AccessibilityDeclaration.normalize(true), .declared)
    }

    func testNilAndFalseAreEquivalentForRecoveryCheck() {
        // 规格 §5.5：nil（属性不存在/读回 error）与 false 视为等价，都记 notDeclared；
        // 只有 declared vs notDeclared 不一致才算恢复失败。
        XCTAssertEqual(AccessibilityDeclaration.normalize(nil), AccessibilityDeclaration.normalize(false))
        XCTAssertNotEqual(AccessibilityDeclaration.normalize(true), AccessibilityDeclaration.normalize(nil))
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd tools/probes && swift test --filter AccessibilityDeclarationTests`
Expected: FAIL（`Cannot find 'AccessibilityDeclaration' in scope`）

- [ ] **Step 3: 写最小实现**

创建 `tools/probes/Sources/VBG1BCore/AX/AccessibilityDeclaration.swift`：

```swift
import ApplicationServices
import Foundation

public enum AccessibilityDeclaration {
    /// 主宣告属性（上游规格 §5.2）
    public static let manualAttribute = "AXManualAccessibility"
    /// 回退宣告属性（主属性返回 `attributeUnsupported` 时用）
    public static let enhancedAttribute = "AXEnhancedUserInterface"

    public enum State: String {
        case declared
        case notDeclared
    }

    /// 规格 §5.5 的归一化规则：`nil`（属性不存在 / 读回 error）与 `false` **视为等价**。
    public static func normalize(_ raw: Bool?) -> State {
        return (raw == true) ? .declared : .notDeclared
    }

    public static func read(appElement: AXUIElement) -> Bool? {
        return read(appElement: appElement, attribute: manualAttribute)
    }

    public static func read(appElement: AXUIElement, attribute: String) -> Bool? {
        var value: CFTypeRef?
        let error = AXUIElementCopyAttributeValue(appElement, attribute as CFString, &value)
        guard error == .success else { return nil }
        if let boolean = value as? Bool { return boolean }
        if let number = value as? NSNumber { return number.boolValue }
        return nil
    }

    /// 把宣告属性恢复到**测量前的原值**。
    ///
    /// 探针宣告过就必须恢复——被测 App 的可访问性状态不能被探针改掉。
    /// 原值已经是 `true` 时**不动**：探针没有改变它，写回 false 反而破坏了用户的状态。
    @discardableResult
    public static func restore(appElement: AXUIElement, declaredAttribute: String?, originalValue: Bool?) -> Bool {
        guard let declaredAttribute, originalValue != true else { return false }
        let error = AXUIElementSetAttributeValue(appElement, declaredAttribute as CFString, kCFBooleanFalse)
        return error == .success
    }
}
```

创建 `tools/probes/Sources/VBG1BCore/AX/FullPathProbe.swift`：

```swift
import ApplicationServices
import Foundation

public struct FullPathResult {
    public var hit: Bool
    public var durationMilliseconds: Double
    public var descriptor: ElementDescriptor?
    public var usedEnhancedFallback: Bool
    public var attempts: Int
    /// 本次**实际**使用的宣告属性名——恢复时按它写回原值（`nil` = 宣告失败）
    public var declaredAttribute: String?

    public init(hit: Bool, durationMilliseconds: Double, descriptor: ElementDescriptor?,
                usedEnhancedFallback: Bool, attempts: Int, declaredAttribute: String?) {
        self.hit = hit
        self.durationMilliseconds = durationMilliseconds
        self.descriptor = descriptor
        self.usedEnhancedFallback = usedEnhancedFallback
        self.attempts = attempts
        self.declaredAttribute = declaredAttribute
    }
}

/// 全路径：**必须与上游规格 §5.2 的全路径算法完全一致**，否则数字不可比。
///
/// 1. 先设 `AXManualAccessibility`；若返回 `attributeUnsupported` → 回退设置 `AXEnhancedUserInterface`；
/// 2. 遍历上限 5,000 个元素（防卡死）；
/// 3. 重试 12 次 × 250ms（最多 3 秒）——重试的真实原因是「宣告之后 Chromium / Electron
///    仍需 1–2 秒才建好 Web 内容树」，不是「等树稳定」。
public enum FullPathProbe {
    public static let maximumElements = 5000
    public static let maximumAttempts = 12
    public static let retryIntervalMilliseconds = 250

    public static func run(appElement: AXUIElement, timeout: Float = AXMessaging.fullPathTimeout) -> FullPathResult {
        AXMessaging.setGlobalTimeout(timeout)
        let declaredAttribute = declare(appElement: appElement)

        let start = CFAbsoluteTimeGetCurrent()
        var attempts = 0
        var hitDescriptor: ElementDescriptor?

        while attempts < maximumAttempts {
            attempts += 1
            if let found = findEditableElement(root: appElement) {
                hitDescriptor = found
                break
            }
            Thread.sleep(forTimeInterval: Double(retryIntervalMilliseconds) / 1000)
        }

        let elapsed = (CFAbsoluteTimeGetCurrent() - start) * 1000
        return FullPathResult(
            hit: hitDescriptor != nil,
            durationMilliseconds: elapsed,
            descriptor: hitDescriptor,
            usedEnhancedFallback: (declaredAttribute == AccessibilityDeclaration.enhancedAttribute),
            attempts: attempts,
            declaredAttribute: declaredAttribute
        )
    }

    /// 返回本次**实际**使用的宣告属性名；`nil` = 宣告失败（两个属性都写不进去）。
    @discardableResult
    public static func declare(appElement: AXUIElement) -> String? {
        let manualError = AXUIElementSetAttributeValue(appElement,
                                                        AccessibilityDeclaration.manualAttribute as CFString,
                                                        kCFBooleanTrue)
        if manualError == .success { return AccessibilityDeclaration.manualAttribute }
        // attributeUnsupported → 回退（上游规格 §5.2）
        let enhancedError = AXUIElementSetAttributeValue(appElement,
                                                         AccessibilityDeclaration.enhancedAttribute as CFString,
                                                         kCFBooleanTrue)
        if enhancedError == .success { return AccessibilityDeclaration.enhancedAttribute }
        return nil
    }

    /// 广度优先遍历，上限 5,000 个元素。
    public static func findEditableElement(root: AXUIElement) -> ElementDescriptor? {
        var queue: [AXUIElement] = [root]
        var visited = 0
        while !queue.isEmpty {
            let element = queue.removeFirst()
            visited += 1
            if visited > maximumElements { return nil }

            var roleValue: CFTypeRef?
            if AXUIElementCopyAttributeValue(element, kAXRoleAttribute as CFString, &roleValue) == .success,
               let role = roleValue as? String {
                let descriptor = FastPathProbe.readDescriptor(element)
                if role == descriptor.role, EditableClassifier.classify(descriptor).isEditable {
                    return descriptor
                }
            }

            var childrenValue: CFTypeRef?
            guard AXUIElementCopyAttributeValue(element, kAXChildrenAttribute as CFString, &childrenValue) == .success,
                  let children = childrenValue as? [AXUIElement] else { continue }
            queue.append(contentsOf: children)
        }
        return nil
    }
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd tools/probes && swift test --filter AccessibilityDeclarationTests`
Expected: PASS（4 个用例全绿）

- [ ] **Step 5: 提交（可选，默认跳过）**

```bash
cd tools/probes && git add Sources/VBG1BCore/AX Tests && git commit -m "feat(probes): 全路径宣告与遍历、宣告状态归一化"
```

---

### Task 7: 三类 App 的焦点准备（TextEdit / Chrome / Slack）

**Files:**
- Create: `tools/probes/Sources/VBG1BCore/Targets/G1BTargetApp.swift`
- Create: `tools/probes/Sources/VBG1BCore/Targets/FocusTargetPreparation.swift`
- Create: `tools/probes/Sources/VBG1BCore/Targets/TabInjector.swift`
- Create: `tools/probes/Sources/VBG1BCore/Targets/MouseClicker.swift`
- Test: `tools/probes/Tests/VBG1BCoreTests/ChromePageFixtureTests.swift`

**Interfaces:**
- Consumes: `VBProbeKit`（Task 3 的 `TextEditHarness` / `WindowInspector`）。
- Produces:
  - `public enum G1BTargetApp: String, CaseIterable` — `textedit`、`chrome`、`slack`；`var bundleIdentifier: String`；`var displayName: String`；`var category: String`。
  - `public enum FocusPreparationMethod: String` — `fileOpen`、`autofocus`、`tab`、`manualClick`、`autoclick`、`none`。
  - `public struct FocusPreparationOutcome` — `setupVerified: Bool`、`method: FocusPreparationMethod`、`processIdentifier: pid_t?`、`detail: String?`。
  - `public protocol FocusTargetPreparing` — `func prepare(app: G1BTargetApp, options: PreparationOptions) throws -> FocusPreparationOutcome`；`func verifyFocus(app: G1BTargetApp, outcome: FocusPreparationOutcome) throws -> Bool`（**焦点验证是一次 AX 调用，必须在冷测量窗口之后**）。
  - `public struct PreparationOptions` — `chromeSetup: ChromeSetupMode`、`slackSetup: SlackSetupMode`、`pageLoadWaitMilliseconds: Int`。
  - `tools/probes/Sources/VBG1BCore/Targets/MouseClicker.swift` — `enum MouseClicker`；`static func click(at point: CGPoint)`。

- [ ] **Step 1: 写失败测试——Chrome 页面的 HTML 必须真的含 textarea**

创建 `tools/probes/Tests/VBG1BCoreTests/ChromePageFixtureTests.swift`：

```swift
import XCTest
@testable import VBG1BCore

final class ChromePageFixtureTests: XCTestCase {
    func testAutofocusFixtureContainsTextareaWithAutofocus() {
        let html = ChromePageFixture.html(mode: .autofocus)
        XCTAssertTrue(html.contains("<textarea autofocus"))
    }

    func testTabFixtureHasNoAutofocus() {
        // 规格 §5.3.1：tab 模式的页面**不含** autofocus，否则多按一次 Tab 会把焦点移走。
        let html = ChromePageFixture.html(mode: .tab)
        XCTAssertFalse(html.contains("autofocus"))
        XCTAssertTrue(html.contains("<textarea"))
    }

    func testFixtureWritesToTemporaryDirectory() throws {
        let url = try ChromePageFixture.write(mode: .autofocus)
        defer { try? FileManager.default.removeItem(at: url) }
        let contents = try String(contentsOf: url, encoding: .utf8)
        XCTAssertTrue(contents.contains("<textarea"))
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd tools/probes && swift test --filter ChromePageFixtureTests`
Expected: FAIL（`Cannot find 'ChromePageFixture' in scope`）

- [ ] **Step 3: 写最小实现**

创建 `tools/probes/Sources/VBG1BCore/Targets/G1BTargetApp.swift`：

```swift
import Foundation

public enum G1BTargetApp: String, CaseIterable {
    case textedit
    case chrome
    case slack

    public var bundleIdentifier: String {
        switch self {
        case .textedit: return "com.apple.TextEdit"
        case .chrome: return "com.google.Chrome"
        case .slack: return "com.tinyspeck.slackmacgap"
        }
    }

    public var displayName: String {
        switch self {
        case .textedit: return "TextEdit"
        case .chrome: return "Google Chrome"
        case .slack: return "Slack"
        }
    }

    /// Chrome 与 Slack **分开判定**，不合并成「Chromium/Electron 一类」——
    /// 两者的快路径表现可能不同（规格 §5.1）。
    public var category: String {
        switch self {
        case .textedit: return "native"
        case .chrome: return "chromium"
        case .slack: return "electron"
        }
    }
}
```

创建 `tools/probes/Sources/VBG1BCore/Targets/FocusTargetPreparation.swift`：

```swift
import AppKit
import Foundation
import VBProbeKit

public enum ChromeSetupMode: String {
    case autofocus
    case tab
}

public enum SlackSetupMode: String {
    case manual
    case autoclick
}

public struct PreparationOptions {
    public var chromeSetup: ChromeSetupMode = .autofocus
    public var slackSetup: SlackSetupMode = .manual
    public var pageLoadWaitMilliseconds: Int = 2000

    public init(chromeSetup: ChromeSetupMode = .autofocus, slackSetup: SlackSetupMode = .manual,
                pageLoadWaitMilliseconds: Int = 2000) {
        self.chromeSetup = chromeSetup
        self.slackSetup = slackSetup
        self.pageLoadWaitMilliseconds = pageLoadWaitMilliseconds
    }
}

public enum FocusPreparationMethod: String {
    case fileOpen
    case autofocus
    case tab
    case manualClick
    case autoclick
    case none
}

public struct FocusPreparationOutcome {
    public var setupVerified: Bool
    public var method: FocusPreparationMethod
    public var processIdentifier: pid_t?
    /// **目标窗口号**——规格 §6.1 的「目标窗口在前台」要用它。
    /// `nil` 表示**拿不到**：调用方必须显式标注「只比对了进程 pid」，
    /// 不能悄悄退化成进程级校验。
    public var windowNumber: Int?
    public var detail: String?

    public init(setupVerified: Bool, method: FocusPreparationMethod, processIdentifier: pid_t? = nil,
                windowNumber: Int? = nil, detail: String? = nil) {
        self.setupVerified = setupVerified
        self.method = method
        self.processIdentifier = processIdentifier
        self.windowNumber = windowNumber
        self.detail = detail
    }
}

public protocol FocusTargetPreparing {
    func prepare(app: G1BTargetApp, options: PreparationOptions) throws -> FocusPreparationOutcome

    /// 焦点验证——**一次 AX 调用**，所以必须放在**冷测量窗口结束之后**（D10）。
    /// 冷测量本身只需要前台校验，不需要这一步。
    func verifyFocus(app: G1BTargetApp, outcome: FocusPreparationOutcome) throws -> Bool
}

/// Chrome 的测试页面。**textarea 必须在 body 首位**且带 `autofocus`——
/// 上游规格 §5.2 的核心命题就是「Chromium 的 Web 内容树在宣告之前是空的」，
/// 只有页面**内**的输入框才能检验这一点（地址栏是原生控件，会给出假阳性）。
public enum ChromePageFixture {
    public static func html(mode: ChromeSetupMode) -> String {
        let autofocusAttribute = (mode == .autofocus) ? " autofocus" : ""
        return """
        <!DOCTYPE html>
        <html lang="zh">
        <head><meta charset="utf-8"><title>VoiceBridge Probe</title></head>
        <body>
        <textarea\(autofocusAttribute) rows="6" cols="60"></textarea>
        </body>
        </html>
        """
    }

    public static func write(mode: ChromeSetupMode) throws -> URL {
        let directory = URL(fileURLWithPath: NSTemporaryDirectory())
            .appendingPathComponent("vb-probe-\(UUID().uuidString)", isDirectory: true)
        try FileManager.default.createDirectory(at: directory, withIntermediateDirectories: true)
        let url = directory.appendingPathComponent("probe.html")
        try html(mode: mode).write(to: url, atomically: true, encoding: .utf8)
        return url
    }
}

/// 三类 App 的焦点准备，**准备阶段全程不碰 AX**（规格 §5.3.1）——
/// 判断 autofocus 有没有生效必须查 AX，而那正是冷快路径那一次调用，会污染冷值。
///
/// ⚠️ **准备阶段结束后，探针会自己重新激活目标 App 并校验前台**（D3）——
/// 准备流程（尤其是 Slack 的人工点击）结束时前台往往是**终端**，不重新激活就会测到终端的焦点元素。
public struct FocusTargetPreparation: FocusTargetPreparing {
    private let textEditHarness: TextEditHarness
    private let windowInspector: WindowInspecting
    /// 准备阶段打开的 TextEdit 实例——供**冷测量之后**的焦点验证使用。
    private let openedTextEdit = InstanceHolder()

    /// 线程/引用语义的可变盒子：准备阶段记下实例，验证阶段取用。
    final class InstanceHolder {
        var instance: TextEditInstance?
    }

    public init(textEditHarness: TextEditHarness = TextEditHarness(),
                windowInspector: WindowInspecting = WindowInspector()) {
        self.textEditHarness = textEditHarness
        self.windowInspector = windowInspector
    }

    public func prepare(app: G1BTargetApp, options: PreparationOptions) throws -> FocusPreparationOutcome {
        switch app {
        case .textedit:
            let instance = try textEditHarness.openTemporaryDocument()
            openedTextEdit.instance = instance
            // ⚠️ **必须解析窗口号**——规格 §6.1 的「目标窗口在前台」靠它。
            // 拿不到就传 nil（编排器会标注「只比对了进程 pid」）。
            return FocusPreparationOutcome(setupVerified: true, method: .fileOpen,
                                           processIdentifier: instance.processIdentifier,
                                           windowNumber: instance.resolveWindowNumber(),
                                           detail: instance.temporaryFileURL.path)
        case .chrome:
            return try prepareChrome(options: options)
        case .slack:
            return try prepareSlack(options: options)
        }
    }

    /// 焦点验证（**一次 AX 调用**——由编排器放在冷测量窗口之后调用）。
    ///
    /// - TextEdit：查**文档正文**是否拿到焦点（`AXTextArea` / `AXTextField` / `AXComboBox`）。
    /// - Chrome / Slack：焦点由准备阶段保证（autofocus 属性 / 人工点击），
    ///   这里**不再发 AX 调用**——多余的 AX 调用会无谓地触碰 AX 树。
    public func verifyFocus(app: G1BTargetApp, outcome: FocusPreparationOutcome) throws -> Bool {
        switch app {
        case .textedit:
            guard let instance = openedTextEdit.instance else { return false }
            return try textEditHarness.verifyDocumentFocus(instance)
        case .chrome, .slack:
            return outcome.setupVerified
        }
    }

    private func prepareChrome(options: PreparationOptions) throws -> FocusPreparationOutcome {
        let pageURL = try ChromePageFixture.write(mode: options.chromeSetup)
        let task = Process()
        task.executableURL = URL(fileURLWithPath: "/usr/bin/open")
        task.arguments = ["-a", "Google Chrome", pageURL.absoluteString]
        try task.run()
        task.waitUntilExit()

        // 等页面加载——`open` 返回后页面可能还没解析完，焦点停在 body → 被误判成「快路径无效」
        Thread.sleep(forTimeInterval: Double(options.pageLoadWaitMilliseconds) / 1000)

        if options.chromeSetup == .tab {
            // 单键、非破坏性、不建树
            TabInjector.press()
            let pid = ChromePidLocator.locate()
            return FocusPreparationOutcome(setupVerified: true, method: .tab,
                                           processIdentifier: pid,
                                           windowNumber: pid.flatMap { WindowInspector().frontmostRegularWindowNumber(ownerPID: Int($0)) },
                                           detail: "page-load-wait=\(options.pageLoadWaitMilliseconds)ms")
        }
        let pid = ChromePidLocator.locate()
        return FocusPreparationOutcome(setupVerified: true, method: .autofocus,
                                       processIdentifier: pid,
                                       windowNumber: pid.flatMap { WindowInspector().frontmostRegularWindowNumber(ownerPID: Int($0)) },
                                       detail: "page-load-wait=\(options.pageLoadWaitMilliseconds)ms")
    }

    /// Slack 的准备。**两种模式**：
    ///
    /// - `manual`（默认）：用户自己点 composer——**零 AX、零误点**。
    /// - `autoclick`：**粗粒度**几何规则（窗口下 1/4 且排除右侧 15%）。
    ///   探针**刻意不碰 AX**（冷测量前）且**不申请屏幕录制权限**，拿不到控件级数据，
    ///   所以这条规则**不能保证不误点**——失败一律回退人工点击。
    private func prepareSlack(options: PreparationOptions) throws -> FocusPreparationOutcome {
        if options.slackSetup == .autoclick {
            if let outcome = autoclickSlackComposer() { return outcome }
            print("autoclick 模式未能在窗口内找到可点的 composer 位置，回退到人工点击。")
        }
        print("""
        请切换到 Slack，用鼠标点一下消息输入框（composer），然后回到终端按回车继续。
        回车之后探针会**自己重新激活 Slack** 并校验前台归属，所以此刻 Slack 不在前台也没有关系。
        （人工点击是零 AX、零误点；自动点击有误触「发送按钮」的真实副作用风险。）
        """)
        _ = readLine()
        let pid = SlackPidLocator.locate()
        return FocusPreparationOutcome(setupVerified: true, method: .manualClick,
                                       processIdentifier: pid,
                                       windowNumber: pid.flatMap { WindowInspector().frontmostRegularWindowNumber(ownerPID: Int($0)) })
    }

    /// autoclick 的**粗粒度**规则（规格 §5.3.1 的「不落在任何子按钮区域」不可实现——D11）：
    ///
    /// 1. 点击点必须落在 Slack 主窗口 bounds 的**下 1/4** 内；
    /// 2. 且**排除右侧 15%**（发送 / 附件控件通常在那里）。
    ///
    /// ⚠️ 这**不能保证不误点**——探针没有控件级数据。拿不到窗口时返回 `nil` → 回退人工点击。
    private func autoclickSlackComposer() -> FocusPreparationOutcome? {
        guard let pid = SlackPidLocator.locate() else { return nil }
        NSRunningApplication(processIdentifier: pid)?.activate()
        Thread.sleep(forTimeInterval: 0.5)

        let windows = windowInspector.windows(ownerPID: Int(pid))
            .filter { $0.layer == 0 && $0.alpha > 0 && $0.bounds.width > 0 && $0.bounds.height > 0 }
        guard let main = windows.max(by: { $0.bounds.width * $0.bounds.height < $1.bounds.width * $1.bounds.height })
        else { return nil }

        let bounds = main.bounds
        // 水平：左侧 85% 区域（排除右侧 15%）
        let x = bounds.minX + bounds.width * 0.50
        // 垂直：下 1/4 的中心（macOS 屏幕坐标原点在左下，所以「下方」= 更小的 y）
        let y = bounds.minY + bounds.height * 0.125
        guard x <= bounds.minX + bounds.width * 0.85,
              y >= bounds.minY, y <= bounds.minY + bounds.height * 0.25 else { return nil }

        MouseClicker.click(at: CGPoint(x: x, y: y))
        return FocusPreparationOutcome(setupVerified: true, method: .autoclick,
                                       processIdentifier: pid,
                                       windowNumber: main.windowNumber,
                                       detail: "autoclick=(\(Int(x)),\(Int(y)))，window=\(main.windowNumber)")
    }
}

/// 按 bundleId 找运行中的实例 pid。
enum ChromePidLocator {
    static func locate() -> pid_t? { BundleIdLocator.locate("com.google.Chrome") }
}

enum SlackPidLocator {
    static func locate() -> pid_t? { BundleIdLocator.locate("com.tinyspeck.slackmacgap") }
}

enum BundleIdLocator {
    static func locate(_ bundleIdentifier: String) -> pid_t? {
        NSRunningApplication.runningApplications(withBundleIdentifier: bundleIdentifier).first?.processIdentifier
    }
}
```

创建 `tools/probes/Sources/VBG1BCore/Targets/TabInjector.swift`：

```swift
import CoreGraphics
import Foundation

/// 只用于 Chrome 的 `tab` 模式——**单键、非破坏性**。
/// 不带 `eventSourceUserData` 标记也无妨：这是 G1-b，G1-b 不做物理输入中止。
enum TabInjector {
    static func press() {
        let source = CGEventSource(stateID: .hidSystemState)
        let keyCode: CGKeyCode = 48   // kVK_Tab
        let down = CGEvent(keyboardEventSource: source, virtualKey: keyCode, keyDown: true)
        down?.post(tap: .cghidEventTap)
        let up = CGEvent(keyboardEventSource: source, virtualKey: keyCode, keyDown: false)
        up?.post(tap: .cghidEventTap)
    }
}
```

创建 `tools/probes/Sources/VBG1BCore/Targets/MouseClicker.swift`：

```swift
import CoreGraphics
import Foundation

/// 左键点击——只用于 Slack 的 `autoclick` 模式。
/// 不带 `eventSourceUserData` 标记也无妨：这是 G1-b，G1-b 不做物理输入中止。
enum MouseClicker {
    static func click(at point: CGPoint) {
        let source = CGEventSource(stateID: .hidSystemState)

        let move = CGEvent(mouseEventSource: source,
                           mouseType: .mouseMoved,
                           mouseCursorPosition: point,
                           mouseButton: .left)
        move?.post(tap: .cghidEventTap)

        let down = CGEvent(mouseEventSource: source,
                           mouseType: .leftMouseDown,
                           mouseCursorPosition: point,
                           mouseButton: .left)
        down?.post(tap: .cghidEventTap)

        let up = CGEvent(mouseEventSource: source,
                         mouseType: .leftMouseUp,
                         mouseCursorPosition: point,
                         mouseButton: .left)
        up?.post(tap: .cghidEventTap)
    }
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd tools/probes && swift test --filter ChromePageFixtureTests`
Expected: PASS（3 个用例全绿）

- [ ] **Step 5: 提交（可选，默认跳过）**

```bash
cd tools/probes && git add Sources/VBG1BCore/Targets Tests && git commit -m "feat(probes): 三类 App 的焦点准备（不碰 AX）"
```

---

### Task 8: 指标模型 + 判读逻辑

**Files:**
- Create: `tools/probes/Sources/VBG1BCore/Measurement/G1BAppReport.swift`
- Create: `tools/probes/Sources/VBG1BCore/Measurement/G1BPayload.swift`
- Create: `tools/probes/Sources/VBG1BCore/Measurement/G1BCleanupProbe.swift`
- Create: `tools/probes/Sources/VBG1BCore/Measurement/G1BJudge.swift`
- Test: `tools/probes/Tests/VBG1BCoreTests/G1BJudgeTests.swift`

**Interfaces:**
- Consumes: `VBG1BCore`（Task 4、5、6）。
- Produces:
  - `public struct G1BAppReport: Codable` — 字段逐条对应规格 §5.5（见下）。
  - `public struct G1BPayload: Codable` — `probeName`、`reportTimestamp`、`reportEnvironment`、`apps: [G1BAppReport]`、`cleanup: CleanupStatus`。
  - `public struct CleanupStatus: Codable` — `noStuckModifier: Bool`、`notStillRecording: Bool`、`inputSourceRestored: Bool`、`notes: [String]`。
  - `public enum G1BCleanupProbe` — `static func collect(baselineInputSource: String?) -> CleanupStatus`（**实机采集**）；`static func judge(stuckModifier:baselineInputSource:currentInputSource:) -> CleanupStatus`（**纯逻辑**，可单测）。
    - ⚠️ 清理状态必须**实测**，不能写死 `true`。G1-b 不注入按键，所以「未在收音」对它不适用——只测「无卡键」与「输入源已恢复」。
  - `public enum G1BJudge` — `public enum Verdict: String`；`public struct Input`；`static func judge(_ input: Input) -> Verdict`；`static func budgetMilliseconds(forCategory:) -> Double`。

- [ ] **Step 1: 写失败测试——判读规则**

创建 `tools/probes/Tests/VBG1BCoreTests/G1BJudgeTests.swift`：

```swift
import XCTest
@testable import VBG1BCore

final class G1BJudgeTests: XCTestCase {
    /// ⚠️ `coldResolved` 默认 **true**——「冷调用在 0.3s 内解析出元素」是**常态**。
    /// 默认 false 会让所有「快路径有效」的用例被判成「首次贵、后续快」，
    /// 测试与实现就对不上了。冷超时的用例**必须显式**传 `coldResolved: false`。
    private func input(category: String = "native",
                       setupVerified: Bool = true,
                       coldResolved: Bool = true,
                       coldStale: Bool = false,
                       postColdResolved: Bool = true,
                       resolvedRate: Int = 10,
                       editableRate: Int = 10,
                       timeoutCount: Int = 0,
                       totalP95Ms: Double = 50,
                       fullPathHit: Bool = true) -> G1BJudge.Input {
        G1BJudge.Input(
            category: category,
            setupVerified: setupVerified,
            coldResolved: coldResolved,
            coldStale: coldStale,
            postColdResolved: postColdResolved,
            warmResolvedCount: resolvedRate,
            warmEditableCount: editableRate,
            warmSampleCount: 10,
            warmTimeoutCount: timeoutCount,
            warmTotalP95Ms: totalP95Ms,
            fullPathHit: fullPathHit
        )
    }

    func testSetupFailureMeansInvalidMeasurement() {
        // 规格 §5.6：准备动作失败 → 无效测量，**不触发任何降级结论**
        XCTAssertEqual(G1BJudge.judge(input(setupVerified: false)), .noEditableFocus)
    }

    func testNeitherColdQueryResolvedMeansInvalidMeasurement() {
        XCTAssertEqual(G1BJudge.judge(input(coldResolved: false, postColdResolved: false)), .noEditableFocus)
    }

    func testColdTimeoutButPostColdResolvedIsNormalMeasurement() {
        // 规格 §5.6 / R-20：冷超时是**预期行为**，不构成无效测量
        XCTAssertNotEqual(G1BJudge.judge(input(coldResolved: false, postColdResolved: true)), .noEditableFocus)
    }

    // MARK: - 冷指标失效（coldStale）时的回退

    func testStaleColdDoesNotTriggerColdExpensiveVerdict() {
        // 冷测量超出 60s 新鲜度窗口 → 冷指标**不参与任何判定**（规格 §5.3.2 末 / R-21）
        XCTAssertEqual(G1BJudge.judge(input(category: "chromium", coldResolved: false, coldStale: true,
                                            postColdResolved: true, totalP95Ms: 100)),
                       .fastPathEffective)
    }

    func testStaleColdWithHealthyWarmIsEffective() {
        // D5：冷指标失效时，**有效性完全回退到稳态指标**——稳态达标就是有效
        XCTAssertEqual(G1BJudge.judge(input(category: "chromium", coldStale: true, totalP95Ms: 280)),
                       .fastPathEffective)
    }

    func testStaleColdWithFailingWarmIsInvalidMeasurementNotIneffective() {
        // D5：冷指标失效 + 稳态也不达标 → 判「**测量无效 / 需重测**」，
        // **不是** R4 降级——降级会砍掉一个功能，宁可标成需重测。
        let verdict = G1BJudge.judge(input(category: "chromium", coldStale: true, postColdResolved: false,
                                           resolvedRate: 0, editableRate: 0, timeoutCount: 10,
                                           totalP95Ms: 300))
        XCTAssertEqual(verdict, .noEditableFocus)
        XCTAssertNotEqual(verdict, .fastPathIneffective)
    }

    // MARK: - 阈值判定用 p95（不是中位数）

    func testNativeFastPathEffective() {
        XCTAssertEqual(G1BJudge.judge(input(category: "native", totalP95Ms: 120)), .fastPathEffective)
    }

    func testNativeFastPathTooSlowIsIneffectiveNative() {
        // D6：原生类不达标 → **原生类的自动聚焦也降级为默认关闭**（独立结论）
        XCTAssertEqual(G1BJudge.judge(input(category: "native", totalP95Ms: 400)), .fastPathIneffectiveNative)
    }

    func testChromiumWithinBudgetIsEffective() {
        XCTAssertEqual(G1BJudge.judge(input(category: "chromium", totalP95Ms: 280)), .fastPathEffective)
    }

    func testChromiumExactlyAtBudgetIsEffective() {
        // 边界：p95 == 预算（300ms）→ **有效**（判据是「≤ 预算」）
        XCTAssertEqual(G1BJudge.judge(input(category: "chromium", totalP95Ms: 300)), .fastPathEffective)
    }

    func testChromiumJustOverBudgetIsIneffective() {
        // 边界：p95 = 301ms > 300ms → **无效**
        XCTAssertEqual(G1BJudge.judge(input(category: "chromium", totalP95Ms: 301)), .fastPathIneffective)
    }

    func testNativeExactlyAtBudgetIsEffective() {
        // 边界：原生类 p95 == 150ms → 有效
        XCTAssertEqual(G1BJudge.judge(input(category: "native", totalP95Ms: 150)), .fastPathEffective)
    }

    func testChromiumColdTimeoutButWarmFastIsColdExpensiveWarmEffective() {
        let verdict = G1BJudge.judge(input(category: "chromium", coldResolved: false,
                                            postColdResolved: true, totalP95Ms: 100))
        XCTAssertEqual(verdict, .coldExpensiveWarmEffective)
    }

    func testChromiumTimeoutCountAboveZeroIsIneffective() {
        XCTAssertEqual(G1BJudge.judge(input(category: "chromium", timeoutCount: 1, totalP95Ms: 100)),
                       .fastPathIneffective)
    }

    func testChromiumEditableRateBelowTenIsIneffective() {
        // D16：可编辑数按**样本数**（10）计，不是按解析数计
        XCTAssertEqual(G1BJudge.judge(input(category: "chromium", editableRate: 9, totalP95Ms: 100)),
                       .fastPathIneffective)
    }

    func testResolvedRateBelowTenIsIneffectiveEvenWhenEditableCountIsTen() {
        // D16：解析率与可编辑数**都**要 10/10——解析率不达标就已经不合格
        XCTAssertEqual(G1BJudge.judge(input(category: "chromium", resolvedRate: 9, editableRate: 10,
                                            totalP95Ms: 100)),
                       .fastPathIneffective)
    }

    func testFullPathMissIsReportedSeparately() {
        XCTAssertEqual(G1BJudge.judge(input(fullPathHit: false)), .fastPathEffectiveWithFullPathMiss)
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd tools/probes && swift test --filter G1BJudgeTests`
Expected: FAIL（`Cannot find 'G1BJudge' in scope`）

- [ ] **Step 3: 写最小实现**

创建 `tools/probes/Sources/VBG1BCore/Measurement/G1BAppReport.swift`：

```swift
import Foundation

/// 一个 App 的全部指标，**逐条对应规格 §5.5**。
public struct G1BAppReport: Codable {
    public var app: String
    public var category: String
    public var bundleIdentifier: String
    public var targetKind: String              // editableComposer / nativeToolbarControl
    public var setupVerified: Bool
    /// 焦点验证结果（**一次 AX 调用**，在冷测量窗口之后做的）
    public var focusVerified: Bool
    public var setupMethod: String
    public var appStartedAt: Date?
    public var coldMeasuredAt: Date?
    public var coldWithinFreshWindow: Bool
    public var mayHaveBeenDeclaredBefore: Bool
    public var coldFastPathQueryMs: Double?
    public var coldFastPathTotalMs: Double?
    public var coldFastPathResolved: Bool
    public var coldFastPathElementEditable: Bool
    public var coldFastPathTimedOut: Bool
    public var coldFastPathTriggeredTreeBuild: String?      // true / false / notApplicable
    public var postColdQueryMs: Double?
    public var postColdQueryResolved: Bool?
    public var warmFastPathQueryMedianMs: Double?
    public var warmFastPathQueryP95: Double?
    public var warmFastPathQueryMin: Double?
    public var warmFastPathQueryMax: Double?
    /// 稳态总耗时**中位数**——只作**参考观测值**留档，**不参与判定**
    public var warmFastPathTotalMedianMs: Double?
    /// 稳态总耗时 **p95**——**判定用的是这个**（规格 §5.6 的「命中率 + 零超时 + p95 阈值」三元组）
    public var warmFastPathTotalP95: Double?
    /// 稳态解析率（分子 = 解析出元素的次数，分母 = **样本数**）
    public var warmFastPathResolvedRate: String?
    /// 稳态**可编辑样本数**（解析出**且可编辑**的次数）——判定要求「解析 10/10 **且** 可编辑 10/10」，
    /// 两者都以**样本数**为分母（规格 §5.6 v1.5 统一口径：不再用「以解析数为分母的比例」）
    public var warmFastPathEditableCount: Int?
    /// 稳态采样次数（N）——**可编辑数的分母**必须记下来，否则报告里的比例无法复核
    public var warmSampleCount: Int
    public var warmFastPathTimeoutCount: Int
    /// 稳态是否在**宣告之后**测得（D2：必须在树建好之后测，否则会误判「快路径无效」）
    public var warmMeasuredAfterDeclaration: Bool
    public var fastPathRole: String?
    public var fastPathSubrole: String?
    public var fastPathDescription: String?
    public var fastPathTitle: String?
    public var fastPathHelp: String?
    public var fastPathPlaceholder: String?
    public var fullPathWarmMs: Double?
    public var fullPathHit: Bool?
    public var fullPathColdMs: Double?
    public var manualAccessibilityBefore: String?
    public var manualAccessibilityAfter: String?
    public var windowGeometryChanged: Bool?
    public var verdict: String
    public var notes: [String]
}

extension G1BAppReport {
    /// **前置校验失败**（找不到实例 / 前台校验不过）时的「无效测量」记录。
    /// 按规格 §5.6：无效测量**不触发任何降级结论**，只进「需重测」。
    static func invalid(app: G1BTargetApp, bundleIdentifier: String, appStartedAt: Date,
                        mayHaveBeenDeclaredBefore: Bool, note: String) -> G1BAppReport {
        G1BAppReport(
            app: app.rawValue,
            category: app.category,
            bundleIdentifier: bundleIdentifier,
            targetKind: "editableComposer",
            setupVerified: false,
            focusVerified: false,
            setupMethod: "none",
            appStartedAt: appStartedAt,
            coldMeasuredAt: nil,
            coldWithinFreshWindow: false,
            mayHaveBeenDeclaredBefore: mayHaveBeenDeclaredBefore,
            coldFastPathQueryMs: nil,
            coldFastPathTotalMs: nil,
            coldFastPathResolved: false,
            coldFastPathElementEditable: false,
            coldFastPathTimedOut: false,
            coldFastPathTriggeredTreeBuild: "notApplicable",
            postColdQueryMs: nil,
            postColdQueryResolved: nil,
            warmFastPathQueryMedianMs: nil,
            warmFastPathQueryP95: nil,
            warmFastPathQueryMin: nil,
            warmFastPathQueryMax: nil,
            warmFastPathTotalMedianMs: nil,
            warmFastPathTotalP95: nil,
            warmFastPathResolvedRate: nil,
            warmFastPathEditableCount: nil,
            warmSampleCount: 0,
            warmFastPathTimeoutCount: 0,
            warmMeasuredAfterDeclaration: false,
            fastPathRole: nil,
            fastPathSubrole: nil,
            fastPathDescription: nil,
            fastPathTitle: nil,
            fastPathHelp: nil,
            fastPathPlaceholder: nil,
            fullPathWarmMs: nil,
            fullPathHit: nil,
            fullPathColdMs: nil,
            manualAccessibilityBefore: nil,
            manualAccessibilityAfter: nil,
            windowGeometryChanged: nil,
            verdict: G1BJudge.Verdict.noEditableFocus.rawValue,
            notes: [note]
        )
    }
}
```

创建 `tools/probes/Sources/VBG1BCore/Measurement/G1BPayload.swift`：

```swift
import Foundation
import VBProbeKit

public struct CleanupStatus: Codable {
    public var noStuckModifier: Bool
    public var notStillRecording: Bool
    public var inputSourceRestored: Bool
    public var notes: [String]
}

/// 报告载荷必须自带 `probeName` / `reportTimestamp` / `reportEnvironment`——
/// `ReportWriter.writeJSON` 用它们装配报告信封。
public struct G1BPayload: Codable, ProbePayload {
    public var probeName: String = "vb-g1b-focus"
    public var reportTimestamp: Date
    public var reportEnvironment: EnvironmentSnapshot
    public var apps: [G1BAppReport]
    public var cleanup: CleanupStatus

    public init(reportTimestamp: Date, reportEnvironment: EnvironmentSnapshot,
                apps: [G1BAppReport], cleanup: CleanupStatus) {
        self.reportTimestamp = reportTimestamp
        self.reportEnvironment = reportEnvironment
        self.apps = apps
        self.cleanup = cleanup
    }
}
```

创建 `tools/probes/Sources/VBG1BCore/Measurement/G1BCleanupProbe.swift`：

```swift
import CoreGraphics
import Foundation
import VBProbeKit

/// **实测**的清理状态（规格 §6.2）。
///
/// ⚠️ 报告里的「清理状态」必须来自**实测**，不能写死 `true`——
/// 写死等于「探针自己声称系统干净」。
///
/// G1-b **不注入按键**，所以「未在收音」对它不适用（它不测语音）——
/// 这里只实测「无卡住的修饰键」与「输入源已恢复」。
public enum G1BCleanupProbe {
    public static func collect(baselineInputSource: String?) -> CleanupStatus {
        let flags = CGEventSource.flagsState(.combinedSessionState)
        let stuckModifier = flags.contains(.maskSecondaryFn)
            || flags.contains(.maskCommand)
            || flags.contains(.maskAlternate)
        let current = InputSourceService().currentInputSourceID()
        return judge(stuckModifier: stuckModifier,
                     baselineInputSource: baselineInputSource,
                     currentInputSource: current)
    }

    /// **纯逻辑**的判定——采集与判定分开，才能单测。
    public static func judge(stuckModifier: Bool, baselineInputSource: String?,
                             currentInputSource: String?) -> CleanupStatus {
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
        if !restored {
            notes.append("输入源未回到基线：\(baselineInputSource ?? "?") → \(currentInputSource ?? "?")")
        }
        if notes.isEmpty {
            notes.append("清理检查通过：无卡键、输入源已回到基线")
        }
        notes.append("G1-b 不注入按键，故不测「未在收音」")

        return CleanupStatus(noStuckModifier: !stuckModifier,
                             notStillRecording: true,
                             inputSourceRestored: restored,
                             notes: notes)
    }
}
```

创建 `tools/probes/Sources/VBG1BCore/Measurement/G1BJudge.swift`：

```swift
import Foundation

/// 规格 §5.6 的判读表。**Chrome 与 Slack 各自独立判定**。
public enum G1BJudge {
    public enum Verdict: String {
        /// 准备动作失败 / 冷查询与 3s 后继查询都没解析出元素 → 本次测量**无效**，不触发任何降级结论
        case noEditableFocus
        /// 快路径有效 → 自动聚焦保持默认开启
        case fastPathEffective
        /// 「首次贵、后续快」→ 首次需要一次全路径宣告，**不构成降级理由**
        case coldExpensiveWarmEffective
        /// 快路径无效（Chromium / Electron 类）→ **触发上游规格 §14 R4 降级：自动聚焦整体降级为默认关闭的独立动作**
        case fastPathIneffective
        /// 原生类快路径不达标 → **原生类的自动聚焦也降级为默认关闭**（与 Chromium / Electron 的降级分开记）
        case fastPathIneffectiveNative
        /// 快路径有效，但全路径也没命中 → 记入「未定论事项」
        case fastPathEffectiveWithFullPathMiss
        /// 对照数据点（`nativeToolbarControl`），不参与判定
        case controlPointOnly
    }

    public struct Input {
        public var category: String
        /// 准备动作**与**焦点验证**都**成功才算 true
        public var setupVerified: Bool
        public var coldResolved: Bool
        /// 冷测量是否超出 60 秒新鲜度窗口。**为 true 时冷指标不参与任何判定**（规格 §5.3.2 末 / R-21）。
        public var coldStale: Bool
        public var postColdResolved: Bool
        public var warmResolvedCount: Int
        /// 解析出**且可编辑**的次数——按**样本数**计，不是按解析数计
        public var warmEditableCount: Int
        public var warmSampleCount: Int
        public var warmTimeoutCount: Int
        /// 稳态总耗时 **p95**（毫秒）——**判定用的就是这个值**。
        /// 不用中位数：超时设 0.3s 时超时样本恰好落在 ~300ms，「median ≤ 300」会把「一半超时」判成通过。
        public var warmTotalP95Ms: Double
        public var fullPathHit: Bool

        public init(category: String, setupVerified: Bool, coldResolved: Bool, coldStale: Bool,
                    postColdResolved: Bool, warmResolvedCount: Int, warmEditableCount: Int,
                    warmSampleCount: Int, warmTimeoutCount: Int, warmTotalP95Ms: Double,
                    fullPathHit: Bool) {
            self.category = category
            self.setupVerified = setupVerified
            self.coldResolved = coldResolved
            self.coldStale = coldStale
            self.postColdResolved = postColdResolved
            self.warmResolvedCount = warmResolvedCount
            self.warmEditableCount = warmEditableCount
            self.warmSampleCount = warmSampleCount
            self.warmTimeoutCount = warmTimeoutCount
            self.warmTotalP95Ms = warmTotalP95Ms
            self.fullPathHit = fullPathHit
        }
    }

    /// 原生类预算 150ms，Chromium / Electron 类预算 300ms（规格 §5.6）。
    ///
    /// **为什么原生类收紧到 150ms**：上游规格 §5.2 期望原生 App 的聚焦 **~50ms**
    /// （原生 AX 树是现成的），而它给出的总体预算是 **≤300ms**。
    /// 探针在原生类上**收紧到 150ms**——相对 ~50ms 仍然宽松，但如果原生类连 150ms 都做不到，
    /// 那上游 §13.1 的第 ① 问「原生 App 上快路径值不值得开自动聚焦」就得到**否定答案**：
    /// 原生类的自动聚焦**也不该默认开启**。
    public static func budgetMilliseconds(forCategory category: String) -> Double {
        return (category == "native") ? 150 : 300
    }

    public static func judge(_ input: Input) -> Verdict {
        // ① 无效测量——**不触发任何降级结论**
        if !input.setupVerified { return .noEditableFocus }
        if !input.coldStale && !input.coldResolved && !input.postColdResolved { return .noEditableFocus }

        let budget = budgetMilliseconds(forCategory: input.category)
        // 「解析率 10/10」**且**「可编辑数 10/10」——可编辑数按**样本数**计，
        // 不是按解析数计：产品要的是**可编辑目标**，不是「解析出某个东西」（规格 §5.6）。
        let fullyResolved = input.warmResolvedCount == input.warmSampleCount
            && input.warmEditableCount == input.warmSampleCount
            && input.warmTimeoutCount == 0
            && input.warmTotalP95Ms <= budget

        guard fullyResolved else {
            // 冷指标不可信时，稳态失败**不能**判定为「快路径无效」——
            // R4 降级会砍掉一个功能，宁可标成需重测（保守读法，R-13 / R-21 / D5）。
            if input.coldStale { return .noEditableFocus }
            // 原生类给**独立**结论：它的降级动作与 Chromium / Electron 不同
            if input.category == "native" { return .fastPathIneffectiveNative }
            return .fastPathIneffective
        }

        if !input.coldStale && !input.coldResolved {
            // 「首次贵、后续快」成立
            return .coldExpensiveWarmEffective
        }
        return input.fullPathHit ? .fastPathEffective : .fastPathEffectiveWithFullPathMiss
    }
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd tools/probes && swift test --filter G1BJudgeTests`
Expected: PASS（17 个用例全绿）

- [ ] **Step 5: 提交（可选，默认跳过）**

```bash
cd tools/probes && git add Sources/VBG1BCore/Measurement Tests && git commit -m "feat(probes): G1-b 指标模型与判读逻辑"
```

---

### Task 9: 参数集 + 编排器（冷测量在前、全路径在中、稳态在后、宣告恢复）

**Files:**
- Create: `tools/probes/Sources/VBG1BCore/Arguments/G1BArguments.swift`
- Create: `tools/probes/Sources/VBG1BCore/Runner/G1BRunner.swift`
- Test: `tools/probes/Tests/VBG1BCoreTests/G1BArgumentsTests.swift`

**Interfaces:**
- Consumes: `VBG1BCore`（Task 4-8）、`VBProbeKit`（Task 1-3）。
- Produces:
  - `public struct G1BArguments: ProbeArgumentSet` — 字段 `outDirectory: URL`、`apps: [G1BTargetApp]`、`warmSampleCount: Int`、`chromeSetup: ChromeSetupMode`、`slackSetup: SlackSetupMode`、`pageLoadWaitMilliseconds: Int`、`includeToolbarControls: Bool`、`measureFullCold: Bool`、`relaunchApp: Bool`、`confirmRelaunch: Bool`、`dryRun: Bool`、`nowOverride: Date?`、`environmentOverride: EnvironmentSnapshot?`。
  - `public struct G1BRunner` — `init(arguments:preparation:windowInspector:)`；`func run() throws -> G1BPayload`。

- [ ] **Step 1: 写失败测试——参数校验**

创建 `tools/probes/Tests/VBG1BCoreTests/G1BArgumentsTests.swift`：

```swift
import XCTest
import VBProbeKit
@testable import VBG1BCore

final class G1BArgumentsTests: XCTestCase {
    private func scan(_ argv: [String]) throws -> ArgumentScanner {
        return try ArgumentScanner(argv, schema: G1BArguments.optionSchema)
    }

    func testRelaunchWithoutConfirmIsRejected() throws {
        // 规格 §5.7：--relaunch-app 必须同时给 --confirm-relaunch
        let scanner = try scan(["--relaunch-app"])
        XCTAssertThrowsError(try G1BArguments(scanner: scanner))
    }

    func testRelaunchWithConfirmIsAccepted() throws {
        let scanner = try scan(["--relaunch-app", "--confirm-relaunch"])
        XCTAssertNoThrow(try G1BArguments(scanner: scanner))
    }

    func testUnknownOptionIsRejected() throws {
        // 拼错的参数必须报错，而不是被静默忽略——**扫描阶段**就拦下来
        XCTAssertThrowsError(try scan(["--app", "textedit"])) { error in
            guard case ArgumentError.unknownOption(let name)? = error as? ArgumentError else {
                return XCTFail("expected unknownOption, got \(error)")
            }
            XCTAssertEqual(name, "--app")
        }
    }

    func testDefaultsMatchSpec() throws {
        let arguments = try G1BArguments(scanner: try scan([]))
        XCTAssertEqual(arguments.warmSampleCount, 10)
        XCTAssertEqual(arguments.pageLoadWaitMilliseconds, 2000)
        XCTAssertEqual(arguments.chromeSetup, .autofocus)
        XCTAssertEqual(arguments.slackSetup, .manual)
        XCTAssertFalse(arguments.includeToolbarControls)
        XCTAssertFalse(arguments.measureFullCold)
        XCTAssertEqual(arguments.apps.map(\.rawValue), ["textedit", "chrome", "slack"])
    }

    func testDefaultOutputDirectoryIsRelativeToWorkingDirectory() throws {
        // `URL(fileURLWithPath:)` 返回**绝对路径**（相对 cwd）——断言必须按后缀比，
        // 不能拿它跟相对的 "docs/verification" 直接比。
        let arguments = try G1BArguments(scanner: try scan([]))
        XCTAssertTrue(arguments.outDirectory.path.hasSuffix("docs/verification"),
                      "实际路径：\(arguments.outDirectory.path)")
        XCTAssertTrue(arguments.outDirectory.path.hasPrefix("/"),
                      "实际路径：\(arguments.outDirectory.path)")
    }

    func testAppsSubsetIsParsed() throws {
        let scanner = try scan(["--apps", "textedit,chrome"])
        let arguments = try G1BArguments(scanner: scanner)
        XCTAssertEqual(arguments.apps.map(\.rawValue), ["textedit", "chrome"])
    }

    func testUnknownAppIsRejected() throws {
        let scanner = try scan(["--apps", "textedit,nope"])
        XCTAssertThrowsError(try G1BArguments(scanner: scanner))
    }

    func testUnknownChromeSetupModeIsRejected() throws {
        let scanner = try scan(["--chrome-setup", "autofocus-or-tab"])
        XCTAssertThrowsError(try G1BArguments(scanner: scanner))
    }

    func testInvalidWarmSampleCountIsRejected() throws {
        let scanner = try scan(["--warm-n", "0"])
        XCTAssertThrowsError(try G1BArguments(scanner: scanner))
    }

    func testNowOverrideIsParsed() throws {
        let scanner = try scan(["--now", "2026-10-23T08:00:00Z"])
        let arguments = try G1BArguments(scanner: scanner)
        XCTAssertNotNil(arguments.nowOverride)
    }

    func testInvalidNowIsRejected() throws {
        let scanner = try scan(["--now", "not-a-date"])
        XCTAssertThrowsError(try G1BArguments(scanner: scanner))
    }

    func testEnvironmentOverrideIsParsed() throws {
        let json = """
        {"hardwareModel":"Mac15,7","macOSVersion":"27.0.1","swiftVersion":"6.4",
         "chatterflyIsSelectedInputSource":true,"axPermissionGranted":true}
        """
        let scanner = try scan(["--env", json])
        let arguments = try G1BArguments(scanner: scanner)
        XCTAssertEqual(arguments.environmentOverride?.hardwareModel, "Mac15,7")
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd tools/probes && swift test --filter G1BArgumentsTests`
Expected: FAIL（`Cannot find 'G1BArguments' in scope`）

- [ ] **Step 3: 写最小实现——参数集**

创建 `tools/probes/Sources/VBG1BCore/Arguments/G1BArguments.swift`：

```swift
import Foundation
import VBProbeKit

public struct G1BArguments: ProbeArgumentSet {
    /// 该探针支持的**全部**参数。**不在表里的参数直接报错退出**——
    /// 探针跑错参数会静默污染结论（比如 `--app` 拼错成 `--apps` 就少测一个 App）。
    public static let optionSchema: [String: OptionKind] = [
        "out": .value,
        "apps": .value,
        "warm-n": .value,
        "chrome-setup": .value,
        "slack-setup": .value,
        "page-load-wait": .value,
        "include-toolbar-controls": .flag,
        "measure-full-cold": .flag,
        "relaunch-app": .flag,
        "confirm-relaunch": .flag,
        "dry-run": .flag,
        "now": .value,
        "env": .value,
    ]

    public var outDirectory: URL
    public var apps: [G1BTargetApp]
    public var warmSampleCount: Int
    public var chromeSetup: ChromeSetupMode
    public var slackSetup: SlackSetupMode
    public var pageLoadWaitMilliseconds: Int
    public var includeToolbarControls: Bool
    public var measureFullCold: Bool
    public var relaunchApp: Bool
    public var confirmRelaunch: Bool
    public var dryRun: Bool
    public var nowOverride: Date?
    public var environmentOverride: EnvironmentSnapshot?

    public init(scanner: ArgumentScanner) throws {
        // ⚠️ `--out` 的默认值**相对当前工作目录（cwd）解析**，不是相对仓库根（D15）——
        // 所以 `run-all.sh` 必须传显式路径。
        outDirectory = URL(fileURLWithPath: scanner.string("out") ?? "docs/verification", isDirectory: true)
        apps = try Self.parseApps(scanner.string("apps"))
        warmSampleCount = try Self.parsePositiveInt(scanner.string("warm-n"), name: "--warm-n", fallback: 10)
        chromeSetup = try Self.parseSetup(scanner.string("chrome-setup"), name: "--chrome-setup", fallback: .autofocus)
        slackSetup = try Self.parseSetup(scanner.string("slack-setup"), name: "--slack-setup", fallback: .manual)
        pageLoadWaitMilliseconds = try Self.parseNonNegativeInt(scanner.string("page-load-wait"),
                                                               name: "--page-load-wait", fallback: 2000)
        includeToolbarControls = scanner.flag("include-toolbar-controls")
        measureFullCold = scanner.flag("measure-full-cold")
        relaunchApp = scanner.flag("relaunch-app")
        confirmRelaunch = scanner.flag("confirm-relaunch")
        dryRun = scanner.flag("dry-run")
        nowOverride = try Self.parseISOD8601(scanner.string("now"))
        environmentOverride = try Self.parseEnvironment(scanner.string("env"))
        // ⚠️ **校验在 init 里做**——只靠 `validate()` 的话，测试与调用方只要忘了调就静默通过
        try validate()
    }

    public func validate() throws {
        // 规格 §5.7：--relaunch-app 必须同时给 --confirm-relaunch（重启 Chrome 可能丢未保存表单）
        if relaunchApp && !confirmRelaunch {
            throw ArgumentError.missingGuard("--relaunch-app", "--confirm-relaunch")
        }
    }

    static func parseApps(_ raw: String?) throws -> [G1BTargetApp] {
        guard let raw else { return G1BTargetApp.allCases }
        let names = raw.split(separator: ",").map { $0.trimmingCharacters(in: .whitespaces) }
        var apps: [G1BTargetApp] = []
        for name in names {
            guard let app = G1BTargetApp(rawValue: name) else {
                throw ArgumentError.unknownOption("--apps=\(name)")
            }
            apps.append(app)
        }
        return apps
    }

    static func parsePositiveInt(_ raw: String?, name: String, fallback: Int) throws -> Int {
        guard let raw else { return fallback }
        guard let value = Int(raw), value > 0 else {
            throw ArgumentError.unknownOption("\(name)=\(raw)")
        }
        return value
    }

    static func parseNonNegativeInt(_ raw: String?, name: String, fallback: Int) throws -> Int {
        guard let raw else { return fallback }
        guard let value = Int(raw), value >= 0 else {
            throw ArgumentError.unknownOption("\(name)=\(raw)")
        }
        return value
    }

    static func parseSetup<T: RawRepresentable>(_ raw: String?, name: String,
                                              fallback: T) throws -> T where T.RawValue == String {
        guard let raw else { return fallback }
        guard let value = T(rawValue: raw) else {
            throw ArgumentError.unknownOption("\(name)=\(raw)")
        }
        return value
    }

    /// 时间与环境 JSON 的解析走**共享实现**（`VBProbeKit.EnvironmentParsing`）——
    /// 两份实现迟早会漂移。
    static func parseISOD8601(_ raw: String?) throws -> Date? {
        return try EnvironmentParsing.parseISOD8601(raw)
    }

    static func parseEnvironment(_ raw: String?) throws -> EnvironmentSnapshot? {
        return try EnvironmentParsing.parseEnvironment(raw)
    }
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cd tools/probes && swift test --filter G1BArgumentsTests`
Expected: PASS（12 个用例全绿）

- [ ] **Step 5: 写编排器**

创建 `tools/probes/Sources/VBG1BCore/Runner/G1BRunner.swift`：

```swift
import AppKit
import Foundation
import VBProbeKit

/// G1-b 的测量编排。**测量顺序是本探针的核心**（规格 §5.3.2）：
///
/// 1. 激活目标 App → 等 0.5s
/// 2. 等页面加载（`--page-load-wait`，默认 2s，记录实际值）
/// 3. 准备可编辑焦点（**此步不碰 AX**）
/// 4. **重新激活目标 App** → 等 0.5s → 校验「前台 pid == 目标 pid」**且**「目标窗口在前台」→ **不匹配就中止该 App 的测量**
/// 5. 冷快路径（0.3s，**此前没有任何 AX 调用**）
/// 6. 用冷调用解析出的元素复核可编辑性
/// 7. 建树归因（**时序判据**，不是循环论证）
/// 8. **把超时调回 0.3s**
/// 9. **全路径**（自己把超时抬到 3.0s、宣告 `AXManualAccessibility`、遍历）→ **保证树已建好**
/// 10. **稳态快路径**（N=10，0.3s）——**在树已建好之后**测
/// 11. 恢复宣告属性 + 复位全局超时为 0
///
/// ⚠️ **第 9 步和第 10 步不能换**（D2）：宣告 `AXManualAccessibility` 正是让 Chromium / Electron
/// 建好 Web 内容树的动作。先测稳态就是在**空树**上测，会得出「快路径无效」→ **假 R4 降级**。
///
/// ⚠️ **第 4 步不能省**（D3）：Slack 的人工准备流程以「回终端按回车」结束，此时前台是**终端**——
/// 终端的文本区是 `AXTextArea`，会被可编辑过滤器放行 → **假的「Slack 快路径有效」**。
public struct G1BRunner {
    private let arguments: G1BArguments
    private let preparation: FocusTargetPreparing
    private let windowInspector: WindowInspecting
    private let frontmostGuard: FrontmostGuard

    public init(arguments: G1BArguments,
                preparation: FocusTargetPreparing = FocusTargetPreparation(),
                windowInspector: WindowInspecting = WindowInspector()) {
        self.arguments = arguments
        self.preparation = preparation
        self.windowInspector = windowInspector
        self.frontmostGuard = FrontmostGuard(
            windowInspector: windowInspector,
            frontmostPIDProvider: { NSWorkspace.shared.frontmostApplication?.processIdentifier ?? -1 }
        )
    }

    public func run() throws -> G1BPayload {
        let environment = arguments.environmentOverride ?? EnvironmentCollector.collect()
        guard environment.axPermissionGranted else {
            throw NSError(domain: "vb-g1b-focus", code: 10, userInfo: [
                NSLocalizedDescriptionKey: "未获得辅助功能（AX）权限。请到「系统设置 › 隐私与安全性 › 辅助功能」授权运行本探针的终端。"
            ])
        }

        if arguments.dryRun {
            printPlan()
            return G1BPayload(reportTimestamp: arguments.nowOverride ?? Date(),
                              reportEnvironment: environment,
                              apps: [],
                              cleanup: G1BCleanupProbe.collect(baselineInputSource: nil))
        }

        // 基线输入源——用于实测「输入源已恢复」
        let baselineInputSource = InputSourceService().currentInputSourceID()

        var reports: [G1BAppReport] = []
        for app in arguments.apps {
            reports.append(try measure(app: app, environment: environment))
        }
        AXMessaging.resetGlobalTimeout()

        return G1BPayload(reportTimestamp: arguments.nowOverride ?? Date(),
                          reportEnvironment: environment,
                          apps: reports,
                          cleanup: G1BCleanupProbe.collect(baselineInputSource: baselineInputSource))
    }

    private func measure(app: G1BTargetApp, environment: EnvironmentSnapshot) throws -> G1BAppReport {
        let probeStartedAt = Date()
        let running = NSRunningApplication.runningApplications(withBundleIdentifier: app.bundleIdentifier).first
        running?.activate()
        Thread.sleep(forTimeInterval: 0.5)                       // ① 激活目标 App → 等 0.5s

        // ③ 准备可编辑焦点（**此步不碰 AX**）
        let outcome = try preparation.prepare(
            app: app,
            options: PreparationOptions(chromeSetup: arguments.chromeSetup,
                                        slackSetup: arguments.slackSetup,
                                        pageLoadWaitMilliseconds: arguments.pageLoadWaitMilliseconds)
        )
        // 冷测量的新鲜度基准 = **App 的启动时刻**，不是探针启动时刻（规格 §5.3.2 末 / R-21）：
        // Chromium 的树一旦被任何辅助客户端碰过就保持建好，
        // 所以「App 已经跑了很久」意味着冷测量大概率已经不冷了。
        let appStartedAt = running?.launchDate ?? probeStartedAt
        // 本次运行前该 App 是否**可能**已被宣告过——无法完全确知，按「启动时刻早于探针启动」如实推断。
        let mayHaveBeenDeclaredBefore = appStartedAt < probeStartedAt
        var notes: [String] = []

        // ④ **重新激活目标 App** → 校验前台 → 不匹配就中止该 App 的测量（D3）。
        // 用**准备阶段拿到的 pid**（TextEdit 是新实例，用 bundleId 找到的可能是另一个实例）。
        guard let targetPID = outcome.processIdentifier ?? running?.processIdentifier else {
            return G1BAppReport.invalid(app: app, bundleIdentifier: app.bundleIdentifier,
                                        appStartedAt: appStartedAt,
                                        mayHaveBeenDeclaredBefore: mayHaveBeenDeclaredBefore,
                                        note: "找不到 \(app.displayName) 的运行实例，测量中止")
        }
        NSRunningApplication(processIdentifier: targetPID)?.activate()
        Thread.sleep(forTimeInterval: 0.5)
        // ⚠️ **传实际窗口号**——规格 §6.1 的「目标窗口在前台」靠它。
        // 传 nil 会退化成只比进程 pid（多窗口时前台的可能不是目标窗口）。
        let frontmost = frontmostGuard.isFrontmost(targetPID: targetPID, windowNumber: outcome.windowNumber)
        guard frontmost.isValid else {
            return G1BAppReport.invalid(
                app: app, bundleIdentifier: app.bundleIdentifier, appStartedAt: appStartedAt,
                mayHaveBeenDeclaredBefore: mayHaveBeenDeclaredBefore,
                note: "前置前台校验失败（进程匹配=\(frontmost.processMatches)，目标窗口在前台=\(frontmost.windowIsFrontmost)，窗口级校验=\(outcome.windowNumber == nil ? "不可用（没拿到窗口号）" : "已比对")），测量中止")
        }

        // ⑤ 冷快路径——**整个运行的第一次 AX 调用**
        let cold = FastPathProbe.measure(timeout: AXMessaging.fastPathTimeout)
        let coldMeasuredAt = Date()

        // ⑥ 用冷调用解析出的元素复核可编辑性（不产生新的 AX 树触碰）
        let editable = cold.descriptor.map { EditableClassifier.classify($0).isEditable } ?? false

        // ⑦ 建树归因——**时序判据**
        var triggeredTreeBuild = "notApplicable"
        var postColdMs: Double?
        var postColdResolved: Bool?
        if !cold.resolved {
            AXMessaging.setGlobalTimeout(AXMessaging.slowQueryTimeout)
            let post = FastPathProbe.measure(timeout: AXMessaging.slowQueryTimeout)
            postColdMs = post.queryMilliseconds
            postColdResolved = post.resolved
            triggeredTreeBuild = (post.queryMilliseconds < 300) ? "true" : "false"
            notes.append("冷调用超时后以 3s 超时复查：postColdQueryMs=\(Int(post.queryMilliseconds))ms")
        }

        // ⑧ **必须调回 0.3s**，否则稳态测量会带上 3s 的超时
        AXMessaging.setGlobalTimeout(AXMessaging.fastPathTimeout)

        // ⑧′ 焦点验证——**一次 AX 调用**，所以必须放在冷测量窗口**之后**（D10）。
        // 冷测量本身只需要前台校验，不需要这一步。
        let focusVerified = (try? preparation.verifyFocus(app: app, outcome: outcome)) ?? false
        if !focusVerified {
            notes.append("焦点验证未通过（可编辑目标未持有焦点）")
        }

        // ⑨ 全路径——**必须跑在稳态测量之前**（D2）。AX 应用元素必须用**该 App 真实 pid**构造。
        let appElement = AXUIElementCreateApplication(targetPID)
        let before = AccessibilityDeclaration.read(appElement: appElement)
        let fullPath = FullPathProbe.run(appElement: appElement, timeout: AXMessaging.fullPathTimeout)
        let after = AccessibilityDeclaration.read(appElement: appElement)
        // 全路径把全局超时抬到了 3.0s → 稳态测量前**必须调回 0.3s**
        AXMessaging.setGlobalTimeout(AXMessaging.fastPathTimeout)

        // ⑩ 稳态快路径——**现在树已经建好了**
        var warmTotals: [Double] = []
        var warmQueries: [Double] = []
        var resolvedCount = 0
        var editableCount = 0
        var timeoutCount = 0
        var lastDescriptor: ElementDescriptor?
        for _ in 0..<arguments.warmSampleCount {
            let sample = FastPathProbe.measure(timeout: AXMessaging.fastPathTimeout)
            warmQueries.append(sample.queryMilliseconds)
            warmTotals.append(sample.totalMilliseconds)
            if sample.resolved {
                resolvedCount += 1
                if let descriptor = sample.descriptor {
                    lastDescriptor = descriptor
                    if EditableClassifier.classify(descriptor).isEditable { editableCount += 1 }
                }
            } else {
                timeoutCount += 1
            }
        }

        // ⑪ 恢复宣告属性 + 复位全局超时
        _ = AccessibilityDeclaration.restore(appElement: appElement,
                                            declaredAttribute: fullPath.declaredAttribute,
                                            originalValue: before)
        AXMessaging.resetGlobalTimeout()

        let totalStats = WarmStatistics(samples: warmTotals)
        let queryStats = WarmStatistics(samples: warmQueries)
        let coldWithinFreshWindow = coldMeasuredAt.timeIntervalSince(appStartedAt) <= 60
        let judgeInput = G1BJudge.Input(
            category: app.category,
            setupVerified: outcome.setupVerified && focusVerified,
            coldResolved: cold.resolved,
            coldStale: !coldWithinFreshWindow,
            postColdResolved: postColdResolved ?? cold.resolved,
            warmResolvedCount: resolvedCount,
            warmEditableCount: editableCount,
            warmSampleCount: arguments.warmSampleCount,
            warmTimeoutCount: timeoutCount,
            warmTotalP95Ms: totalStats.p95 ?? .infinity,
            fullPathHit: fullPath.hit
        )
        let verdict = G1BJudge.judge(judgeInput)

        return G1BAppReport(
            app: app.rawValue,
            category: app.category,
            bundleIdentifier: app.bundleIdentifier,
            targetKind: "editableComposer",
            setupVerified: outcome.setupVerified,
            focusVerified: focusVerified,
            setupMethod: outcome.method.rawValue,
            appStartedAt: appStartedAt,
            coldMeasuredAt: coldMeasuredAt,
            coldWithinFreshWindow: coldWithinFreshWindow,
            mayHaveBeenDeclaredBefore: mayHaveBeenDeclaredBefore,
            coldFastPathQueryMs: cold.queryMilliseconds,
            coldFastPathTotalMs: cold.totalMilliseconds,
            coldFastPathResolved: cold.resolved,
            coldFastPathElementEditable: editable,
            coldFastPathTimedOut: cold.timedOut,
            coldFastPathTriggeredTreeBuild: triggeredTreeBuild,
            postColdQueryMs: postColdMs,
            postColdQueryResolved: postColdResolved,
            warmFastPathQueryMedianMs: queryStats.median,
            warmFastPathQueryP95: queryStats.p95,
            warmFastPathQueryMin: queryStats.min,
            warmFastPathQueryMax: queryStats.max,
            warmFastPathTotalMedianMs: totalStats.median,
            warmFastPathTotalP95: totalStats.p95,
            warmFastPathResolvedRate: "\(resolvedCount)/\(arguments.warmSampleCount)",
            warmFastPathEditableCount: editableCount,
            warmSampleCount: arguments.warmSampleCount,
            warmFastPathTimeoutCount: timeoutCount,
            warmMeasuredAfterDeclaration: true,
            fastPathRole: lastDescriptor?.role,
            fastPathSubrole: lastDescriptor?.subrole,
            fastPathDescription: lastDescriptor?.description,
            fastPathTitle: lastDescriptor?.title,
            fastPathHelp: lastDescriptor?.help,
            fastPathPlaceholder: lastDescriptor?.placeholder,
            fullPathWarmMs: fullPath.durationMilliseconds,
            fullPathHit: fullPath.hit,
            fullPathColdMs: nil,
            manualAccessibilityBefore: AccessibilityDeclaration.normalize(before).rawValue,
            manualAccessibilityAfter: AccessibilityDeclaration.normalize(after).rawValue,
            windowGeometryChanged: nil,
            verdict: verdict.rawValue,
            notes: notes
        )
    }

    private func printPlan() {
        print("G1-b 测量计划（dry-run，不做任何 AX 调用）")
        for app in arguments.apps {
            print("  - \(app.displayName)：准备方式=\(app == .chrome ? arguments.chromeSetup.rawValue : "—")，稳态样本=\(arguments.warmSampleCount)")
        }
        print("  超时阶梯：快路径 \(AXMessaging.fastPathTimeout)s / 后继查询 \(AXMessaging.slowQueryTimeout)s / 全路径 \(AXMessaging.fullPathTimeout)s")
    }
}
```

- [ ] **Step 6: 编译并跑全部测试**

Run: `cd tools/probes && swift build && swift test`
Expected: BUILD SUCCEEDED，所有测试 PASS

- [ ] **Step 7: 提交（可选，默认跳过）**

```bash
cd tools/probes && git add Sources/VBG1BCore Tests && git commit -m "feat(probes): G1-b 参数集与测量编排"
```

---

### Task 10: 报告渲染（Markdown）+ 可执行入口

> **开始前先删掉 Task 1 的占位文件**：
> ```bash
> rm tools/probes/Sources/vb-g1b-focus/Placeholder.swift
> ```
> ⚠️ **必须删**——占位文件里有 top-level `print(...)`，而真正的 `main.swift` 也有
> top-level 代码；两个文件同时存在会报 `expressions are not allowed at the top level`。

**Files:**
- Create: `tools/probes/Sources/VBG1BCore/Report/G1BMarkdownRenderer.swift`
- Create: `tools/probes/Sources/vb-g1b-focus/main.swift`
- Test: `tools/probes/Tests/VBG1BCoreTests/G1BMarkdownRendererTests.swift`

**Interfaces:**
- Consumes: `VBG1BCore`（Task 8、9）、`VBProbeKit`（Task 1）。
- Produces: `public enum G1BMarkdownRenderer { static func render(payload: G1BPayload) -> String }`；可执行文件 `vb-g1b-focus`。

- [ ] **Step 1: 写失败测试——Markdown 必须含规格 §7.2 要求的四节**

创建 `tools/probes/Tests/VBG1BCoreTests/G1BMarkdownRendererTests.swift`：

```swift
import XCTest
import VBProbeKit
@testable import VBG1BCore

final class G1BMarkdownRendererTests: XCTestCase {
    func testRendersRequiredSections() {
        let payload = G1BPayload(
            reportTimestamp: Date(timeIntervalSince1970: 1_700_000_000),
            reportEnvironment: EnvironmentSnapshot(hardwareModel: "Mac15,7", macOSVersion: "27.0.1",
                                                  swiftVersion: "6.4", chatterflyVersion: "1.0.8",
                                                  chatterflyPath: "/Library/Input Methods/Chatterfly.app",
                                                  chatterflyIsSelectedInputSource: true,
                                                  axPermissionGranted: true,
                                                  physicalInputAbortAvailable: false,
                                                  currentInputSourceID: "com.tencent.inputmethod.chatterfly.pinyin"),
            apps: [],
            cleanup: CleanupStatus(noStuckModifier: true, notStillRecording: true,
                                  inputSourceRestored: true, notes: [])
        )
        let markdown = G1BMarkdownRenderer.render(payload: payload)
        for heading in ["## 0. 探测环境", "## 1. G1-a", "## 2. G1-b", "## 3. 对规格的影响", "## 4. 未定论事项",
                        "### 清理状态"] {
            XCTAssertTrue(markdown.contains(heading), "缺少章节：\(heading)")
        }
    }

    func testRendersWarmP95AndDeclarationOrderRows() {
        // 判定用的是 p95，稳态必须在宣告之后测——两行都要出现在报告里（D2 / D4）
        let app = G1BAppReport.invalid(app: .textedit,
                                       bundleIdentifier: "com.apple.TextEdit",
                                       appStartedAt: Date(timeIntervalSince1970: 1_700_000_000),
                                       mayHaveBeenDeclaredBefore: false,
                                       note: "单测用")
        let payload = G1BPayload(
            reportTimestamp: Date(timeIntervalSince1970: 1_700_000_000),
            reportEnvironment: EnvironmentSnapshot(hardwareModel: "Mac15,7", macOSVersion: "27.0.1",
                                                  swiftVersion: "6.4", chatterflyVersion: "1.0.8",
                                                  chatterflyPath: "/Library/Input Methods/Chatterfly.app",
                                                  chatterflyIsSelectedInputSource: true,
                                                  axPermissionGranted: true,
                                                  physicalInputAbortAvailable: false,
                                                  currentInputSourceID: nil),
            apps: [app],
            cleanup: CleanupStatus(noStuckModifier: true, notStillRecording: true,
                                  inputSourceRestored: true, notes: [])
        )
        let markdown = G1BMarkdownRenderer.render(payload: payload)
        XCTAssertTrue(markdown.contains("稳态总耗时 **p95（判定值）**"))
        XCTAssertTrue(markdown.contains("稳态是否在宣告之后测量"))
        XCTAssertTrue(markdown.contains("焦点验证"))
    }

    func testIncludesMacOSVersionAndChatterflyPath() {
        let payload = G1BPayload(
            reportTimestamp: Date(timeIntervalSince1970: 1_700_000_000),
            reportEnvironment: EnvironmentSnapshot(hardwareModel: "Mac15,7", macOSVersion: "27.0.1",
                                                  swiftVersion: "6.4", chatterflyVersion: "1.0.8",
                                                  chatterflyPath: "/Library/Input Methods/Chatterfly.app",
                                                  chatterflyIsSelectedInputSource: true,
                                                  axPermissionGranted: true,
                                                  physicalInputAbortAvailable: false,
                                                  currentInputSourceID: nil),
            apps: [],
            cleanup: CleanupStatus(noStuckModifier: true, notStillRecording: true,
                                  inputSourceRestored: true, notes: [])
        )
        let markdown = G1BMarkdownRenderer.render(payload: payload)
        XCTAssertTrue(markdown.contains("27.0.1"))
        XCTAssertTrue(markdown.contains("/Library/Input Methods/Chatterfly.app"))
    }
}
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cd tools/probes && swift test --filter G1BMarkdownRendererTests`
Expected: FAIL（`Cannot find 'G1BMarkdownRenderer' in scope`）

- [ ] **Step 3: 写最小实现**

创建 `tools/probes/Sources/VBG1BCore/Report/G1BMarkdownRenderer.swift`：

```swift
import Foundation

/// 规格 §7.2 的结论文档结构。**§1 G1-a 一节留空占位**——
/// G1-b 单独跑时该节写「未在本探针覆盖范围内，见 G1-a 报告」；
/// 合并报告时由计划 2 的产物填入。
public enum G1BMarkdownRenderer {
    public static func render(payload: G1BPayload) -> String {
        var lines: [String] = []
        let environment = payload.reportEnvironment

        lines.append("# VoiceBridge G1-b 聚焦探针 · 实测结论")
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
        lines.append("| 物理输入中止能力 | \(environment.physicalInputAbortAvailable.map { $0 ? "可用" : "不可用" } ?? "未探测") |")
        lines.append("| 当前输入源 | \(environment.currentInputSourceID ?? "未知") |")
        lines.append("")

        lines.append("## 1. G1-a")
        lines.append("")
        lines.append("本探针不覆盖 G1-a（触发键）。G1-a 结论见 `vb-g1a-trigger` 的报告。")
        lines.append("")

        lines.append("## 2. G1-b")
        lines.append("")
        if payload.apps.isEmpty {
            lines.append("本次运行未测量任何 App。")
        } else {
            for app in payload.apps {
                lines.append("### \(app.app)（\(app.category)）")
                lines.append("")
                lines.append("| 指标 | 值 |")
                lines.append("| --- | --- |")
                lines.append("| 准备动作 | \(app.setupMethod)（verified=\(app.setupVerified)） |")
                lines.append("| 焦点验证 | \(app.focusVerified ? "通过" : "**未通过**") |")
                lines.append("| 冷快路径是否解析出元素 | \(app.coldFastPathResolved) |")
                lines.append("| 冷快路径是否超时 | \(app.coldFastPathTimedOut) |")
                lines.append("| 冷快路径总耗时 | \(app.coldFastPathTotalMs.map { String(format: "%.1fms", $0) } ?? "—") |")
                lines.append("| 冷调用元素可编辑 | \(app.coldFastPathElementEditable) |")
                lines.append("| 冷调用是否触发建树 | \(app.coldFastPathTriggeredTreeBuild ?? "—") |")
                lines.append("| 冷测量是否在新鲜度窗口内 | \(app.coldWithinFreshWindow) |")
                lines.append("| 稳态解析率 | \(app.warmFastPathResolvedRate ?? "—") |")
                lines.append("| 稳态可编辑数（按样本数计） | \(app.warmFastPathEditableCount.map { "\($0)/\(app.warmSampleCount)" } ?? "—") |")
                lines.append("| 稳态超时次数 | \(app.warmFastPathTimeoutCount) |")
                lines.append("| 稳态总耗时 **p95（判定值）** | \(app.warmFastPathTotalP95.map { String(format: "%.1fms", $0) } ?? "—") |")
                lines.append("| 稳态总耗时中位数（仅参考） | \(app.warmFastPathTotalMedianMs.map { String(format: "%.1fms", $0) } ?? "—") |")
                lines.append("| 稳态纯查询 p95（仅参考） | \(app.warmFastPathQueryP95.map { String(format: "%.1fms", $0) } ?? "—") |")
                lines.append("| 稳态是否在宣告之后测量 | \(app.warmMeasuredAfterDeclaration) |")
                lines.append("| 全路径耗时 | \(app.fullPathWarmMs.map { String(format: "%.1fms", $0) } ?? "—") |")
                lines.append("| 全路径命中 | \(app.fullPathHit.map { $0 ? "是" : "否" } ?? "—") |")
                lines.append("| 宣告属性恢复 | \(app.manualAccessibilityBefore ?? "—") → \(app.manualAccessibilityAfter ?? "—") |")
                lines.append("| **判读结论** | **\(app.verdict)** |")
                lines.append("")
            }
        }

        lines.append("### 清理状态")
        lines.append("")
        lines.append("| 项 | 值 |")
        lines.append("| --- | --- |")
        lines.append("| 无卡住的修饰键 | \(payload.cleanup.noStuckModifier ? "是" : "**否**") |")
        lines.append("| 未仍在收音 | \(payload.cleanup.notStillRecording ? "是" : "**否**") |")
        lines.append("| 输入源已恢复 | \(payload.cleanup.inputSourceRestored ? "是" : "**否**") |")
        lines.append("| 备注 | \(payload.cleanup.notes.isEmpty ? "无" : payload.cleanup.notes.joined(separator: "；")) |")
        lines.append("")

        lines.append("## 3. 对规格的影响")
        lines.append("")
        lines.append("（按回填清单逐条填写：哪一节 → 改成什么）")
        lines.append("")

        lines.append("## 4. 未定论事项")
        lines.append("")
        var undetermined: [String] = []
        for app in payload.apps {
            if app.verdict == G1BJudge.Verdict.noEditableFocus.rawValue {
                undetermined.append("- \(app.app)：准备失败或冷/后继查询都未解析出元素 → **需重测**")
            }
            if app.fullPathHit == false {
                undetermined.append("- \(app.app)：全路径未命中 → 无法用全路径聚焦")
            }
        }
        if undetermined.isEmpty { undetermined.append("- 无") }
        lines.append(contentsOf: undetermined)
        lines.append("")

        return lines.joined(separator: "\n")
    }

    private static func iso(_ date: Date) -> String {
        let formatter = ISO8601DateFormatter()
        formatter.formatOptions = [.withInternetDateTime]
        return formatter.string(from: date)
    }
}
```

创建 `tools/probes/Sources/vb-g1b-focus/main.swift`：

```swift
import Foundation
import VBG1BCore
import VBProbeKit

let arguments = Array(CommandLine.arguments.dropFirst())

do {
    let scanner = try ArgumentScanner(arguments, schema: G1BArguments.optionSchema)
    let parsed = try G1BArguments(scanner: scanner)
    try parsed.validate()

    let runner = G1BRunner(arguments: parsed)
    let payload = try runner.run()

    let writer = ReportWriter(outputDirectory: parsed.outDirectory)
    let jsonURL = try writer.writeJSON(payload, as: "2026-10-23-g1b-results.json")
    let markdownURL = try writer.writeMarkdown(G1BMarkdownRenderer.render(payload: payload),
                                               as: "2026-10-23-g1b-results.md")
    print("报告已写入：")
    print("  \(jsonURL.path)")
    print("  \(markdownURL.path)")
} catch {
    FileHandle.standardError.write(Data("参数错误或探测失败：\(error)\n".utf8))
    exit(2)
}
```

> `ReportWriter.writeJSON` 需要载荷提供 `probeName` / `reportTimestamp` / `reportEnvironment`——`G1BPayload` 已提供（Task 8）。

- [ ] **Step 4: 跑渲染器测试确认通过**

Run: `cd tools/probes && swift test --filter G1BMarkdownRendererTests`
Expected: PASS（3 个用例全绿）

- [ ] **Step 5: 跑全量测试确认通过**

Run: `cd tools/probes && swift test`
Expected: 所有测试 PASS

- [ ] **Step 6: 提交（可选，默认跳过）**

```bash
cd tools/probes && git add Sources Tests && git commit -m "feat(probes): G1-b 报告渲染与可执行入口"
```

---

### Task 11: 实机跑通并产出 `docs/verification/` 结论

**Files:**
- Create: `docs/verification/2026-10-23-g1b-results.md`（由探针生成）
- Create: `docs/verification/2026-10-23-g1b-results.json`（由探针生成）
- Modify: `docs/superpowers/specs/2026-10-23-voicebridge-design.md`（回填 §5.2 / §13.1 / §13.2）

**Interfaces:**
- Consumes: `vb-g1b-focus`（Task 10）。
- Produces: 实测结论文件 + 回填后的上游规格（升 **v3.5**）。

- [ ] **Step 1: 构建 release**

Run: `cd tools/probes && swift build -c release`
Expected: BUILD SUCCEEDED

- [ ] **Step 2: 先跑 dry-run 确认参数解析与计划打印正常**

Run: `cd tools/probes && .build/release/vb-g1b-focus --dry-run --out ../../docs/verification/`
Expected: 打印三个 App 的测量计划，**不产生任何 AX 调用**，不生成报告文件

- [ ] **Step 3: 第一轮实机测量**

Run: `cd tools/probes && .build/release/vb-g1b-focus --out ../../docs/verification/`
Expected: 依次测量 TextEdit / Chrome / Slack；Slack 阶段终端会提示「请切换到 Slack，用鼠标点一下消息输入框（composer），然后回到终端按回车继续」——回车后**探针自己重新激活 Slack** 再开始测量

- [ ] **Step 4: 检查产物**

Run: `ls -la docs/verification/`
Expected: `2026-10-23-g1b-results.md` 与 `2026-10-23-g1b-results.json` 都存在

- [ ] **Step 5: 核对报告里必须出现的项**

打开 `docs/verification/2026-10-23-g1b-results.md`，确认：
- §0 含 **macOS 实测版本**（应为 27.0.1）、Chatterfly 真实安装路径（应为 `/Library/Input Methods/Chatterfly.app`）、AX 权限自检结果
- §2 中 Chrome 与 Slack **分列**，各有 `coldFastPathTriggeredTreeBuild` 与 **`warmFastPathTotalP95`**（判定值）、`warmMeasuredAfterDeclaration`（应为 `true`）
- §2 有「**清理状态**」小节（无卡键 / 未在收音 / 输入源已恢复 + 备注）
- 判读结论不是 `noEditableFocus`（若全是 `noEditableFocus`，说明焦点准备失败，**按 §5.6 重测**而不是下结论）

- [ ] **Step 6: 若任一类触发 R4 降级，做第二次独立确认**

**必须重启目标 App**（`AXManualAccessibility` 恢复成 false **不会拆掉** Chromium 已建好的树，不重启的第二轮是 warm 的）：

Run: `cd tools/probes && .build/release/vb-g1b-focus --relaunch-app --confirm-relaunch --out ../../docs/verification/`
Expected: 第二次运行；报告中两轮都触发降级才可采信，否则标 `warmOnly` 且不写入回填结论

- [ ] **Step 7: 回填上游规格**

按规格 §7.3 的回填清单逐条改 `docs/superpowers/specs/2026-10-23-voicebridge-design.md`：
- §0.1 升 **v3.5** + 加一张「M0 实测回填」追溯表
- §5.2 决策表：把「**G1-b 给出依据**」的占位换成实测数字与结论
- §5.2 建树问题（行 412）：用 `coldFastPathTriggeredTreeBuild` 回填
- §13.1 G1-b 行：填入实测结论
- §13.2 M0 出口：填入三类 App 的 AX 聚焦耗时
- §14 R4：更新风险状态
- §15 参考资料：补「已核实」条目

**只回填实测到的**——探针没测到的（例如「识别出的文字是否真的上屏」）显式标为「**未验证**」，不写成结论。

- [ ] **Step 8: 提交（可选，默认跳过）**

```bash
git add docs/verification docs/superpowers/specs/2026-10-23-voicebridge-design.md tools/probes
git commit -m "docs: G1-b 聚焦探针实测结论与规格回填"
```

---

## 完成定义

- [ ] `cd tools/probes && swift build -c release` 通过
- [ ] `cd tools/probes && swift test` 全绿
- [ ] `docs/verification/2026-10-23-g1b-results.md` 与 `.json` 都存在
- [ ] 报告中 Chrome 与 Slack **分列**，且每个 App 的判读结论都有明确取值
- [ ] 每个 App 的 `warmMeasuredAfterDeclaration` 为 `true`（稳态在**宣告之后**测），判定值是 **`warmFastPathTotalP95`**
- [ ] 报告含「**清理状态**」小节（无卡键 / 未在收音 / 输入源已恢复 + 备注）
- [ ] 若触发 R4 降级：两次独立运行（第二次带 `--relaunch-app --confirm-relaunch`）都成立
- [ ] 上游规格已回填并升 v3.5，§0.1 有追溯表
- [ ] 探测后系统状态干净：无卡住修饰键、输入源未被切走、探针自己开的 TextEdit 实例已关闭

---
---

## 附录：本计划的代码块已通过真实编译验证

> 这一节是**元信息**，不参与执行流程。记录的是：本计划里每一个 ```swift 代码块都**按字面**抽出来放进真实 SwiftPM 包编译并跑过测试——不是「看着对」。

**验证方法**（可复现）：

1. 从本计划与计划 2 的 markdown 里按 ```` ```swift ```` 围栏切出全部代码块，用紧邻的 `` `tools/probes/...` `` 路径标记落盘成文件树；
2. **逐 Task 验证**（不只是验最终态）：按「计划 1 Task 1 → Task 11 → 计划 2 Task 0 → Task 11」的执行顺序**累积**文件，每到一个 Task 边界就 `swift build` + `swift test`；
3. 再验**最终态**的两个变体（计划 1 单独 / 计划 1 + 计划 2）。

**验证结果**（Swift 6.4 / macOS 27.0.1 / XCTest，无第三方依赖）：

| 变体 | Package.swift | 构建 | 测试 |
| --- | --- | --- | --- |
| **逐 Task：23 个边界** | 按执行顺序推进 | 每个边界都 BUILD SUCCEEDED | 每个边界都全绿 |
| 计划 1 单独执行 | 计划 1 的 3-target 版 | BUILD SUCCEEDED | `VBProbeKitTests` 21 + `VBG1BCoreTests` 56 = **77 全绿** |
| 计划 1 + 计划 2 | 计划 2 Task 0 的 5-product / 8-target 版 | BUILD SUCCEEDED | 再加 `VBG1ACoreTests` 82 = **159 全绿** |

**逐 Task 验证抓出来的真问题**（都已修进本计划）：

1. **Task 1 声明了 target 却没有源文件** → SwiftPM 报 `Source files for target VBG1BCore should be located under ...`，Task 1 直接 build 不过。修法：Task 1 同时建三个**占位文件**（`Sources/VBG1BCore/Placeholder.swift`、`Sources/vb-g1b-focus/Placeholder.swift`、`Tests/VBG1BCoreTests/Placeholder.swift`），Task 4 / Task 10 写入真实文件后**删掉**。
2. **`EnvironmentSnapshot` 原本排在 Task 2** —— 但 `ReportDocument`（Task 1）引用它，`swift test` 会编译**整个 package**，Task 1 的 Step 4/6 直接编译失败。修法：把它**提前**到 Task 1（它只依赖 Foundation，是最干净的底座类型）。
3. **同名文件在两个 Task 出现**（`Package.swift` 在计划 1 是 3-target、计划 2 是 5-target）→ 按路径抽取代码块时会被搞混。修法：`Package.swift` **不进回写流程**，两个版本各自手工维护；占位文件**不能叫 `main.swift`**（Task 10 的真入口也叫这个名字）。

**验证期间修掉的 API 事实**（这些在公开 SDK 里与直觉不同，照抄即可）：

- `TISCopyCurrentKeyboardInputSource()` → `Unmanaged<TISInputSource>`；`TISCreateInputSourceList` → `Unmanaged<CFArray>`；`TISGetInputSourceProperty` → `UnsafeMutableRawPointer`（用 `Unmanaged<CFTypeRef>.fromOpaque(raw).takeUnretainedValue()` 取）。
- `CGEventMask` 是**位掩码**，**没有** `contains(_:)`——自己做位运算。
- `AXUIElement` 是 CF 类型，`as?` 条件 downcast **恒成功**（Swift 6 报错），必须 `as!`。
- `CGWindowListCopyWindowInfo(.optionOnScreenOnly)` 的返回顺序**就是前到后**（不能按 `windowNumber` 排序——`windowNumber` 是 ID 不是 z-order）。
- `URL(fileURLWithPath:)` 返回**绝对路径**（相对 cwd），断言要按后缀比。
- SwiftPM 的 **target 目录必须存在**才能在 `Package.swift` 里声明——空目录不行，至少要有一个 `.swift` 文件。
