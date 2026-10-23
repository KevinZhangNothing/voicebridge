# VoiceBridge 设计规格

- 日期：2026-10-23
- 版本：v3.4（v3 范围收敛为「遥控器」；v3.1 补齐 Android 端自定义快捷键；v3.2 / v3.3 / v3.4 评审修复）
- 状态：已评审
- 决策来源：参考仓库 `../remote-mic-app`（SayAll / 无线麦）调研 + 多轮设计澄清 + 多轮评审

---

## 0. 修订说明

### 0.1 v3.2 / v3.3 / v3.4：评审修复（共六轮）

规格经六轮逐条复核后修了 **24 + 18 + 6 + 18 + 15 + 9 条**：第一轮 8 个 P0（不改会导致实现分叉或做不出来）与 16 个 P1（返工 / 歧义），第二轮 3 个 P0（v3.1 的修复自身留下的矛盾与一个会锁死功能的缺陷）、1 个 P1、6 处引用同步、8 处小瑕疵，第三轮 1 个真缺口（导入 / 导出会破坏编号不复用规则）与 5 个小项，第四轮 4 个 P0（认证总则缺失、配对被超时杀掉、`preparing` 收到 `voiceStop` 会卡死键、一次点按执行两次）、8 个 P1 与 6 个 P2，第五轮 2 个 P0（**自动聚焦快速失败没同步下去**、**心跳与判死会误杀配对连接**）、3 个 P1、6 个 P2 残留与 4 项技术补强，**第六轮 8 个语义口子与测试 / 验收补齐**（不阻塞开工，但会让两端各写各的字符串、让文案掉进缝里）。

前四轮集中在 v3.1 新增的「注册表 + 手机录入」链路；**第五轮转向跨节一致性**——前两轮改动最大的地方（快速失败、配对窗口）自身留下了与正文冲突的残留；**第六轮是收口**：正文层面已无会卡实现的问题，剩下的是把机器码、分型文案、测试与验收补齐。

**第一轮评审（24 条）**

**P0（8 条）**

| # | 问题 | 结论 |
| --- | --- | --- |
| 1 | 配对码生成方与 `pairingRequest` 方向矛盾；`hello.tokenId` 必填但首次配对没有 | `pairingRequest` **去掉 `code`**（纯请求配对）；Mac 生成码后用**新帧 `pairingCode`** 推给两端显示；`hello.tokenId` **可选**，缺失即走配对（§6.1、§6.4） |
| 2 | `sid` 约定与帧定义不符（多数帧不带 `sid`） | **删除 `sid`**：单连接模型下"连接即身份"，迟到帧由 socket 关闭 + `generation` 机制兜底（§6.1、§7.3） |
| 3 | 媒体 / 音量键没有 virtual keycode，键规格模型做不出来 | **这 6 个名字从键名表删除**，只保留 `action` 一条路径；`action` 用 `NSSystemDefined` 事件实现（§6.3、§6.5.6） |
| 4 | `combo` 无主键时 press / release 语义未定义 | **主键恰好 1 个**：无主键的规格写入即 `invalid_binding`；单个修饰键只属于触发键（§6.5.2） |
| 5 | 自定义槽位删不掉；编号分配两端必然分叉 | `mappingSet` 允许 `binding:null` = 删除；**编号由 Mac 端 `customSeq` 分配**，`mappingListResult` 带回，删除后不复用（§6.3、§6.5.5） |
| 6 | 触发键配置仍然只能坐到 Mac 前 | **触发键并入注册表**（保留 id `__voice`，含触发键与投递模式），手机端可配，**不占 16 个自定义槽位**（§6.3、§6.5.5） |
| 7 | "触发键未配置"复用了 `accessibility_denied` | 新增 `trigger_key_not_configured`——"去系统设置授权"与"去 Mac 端配触发键"是两个补救动作，不能共用一个码（§6.2、§8） |
| 8 | 点按模式"停止拍失败"没有错误码 | 新增 `stop_tap_failed`（对齐参考实现的 `start_tap_failed` / `stop_tap_failed`）（§6.2、§7.3、§8） |

**P1（16 条）**

| # | 位置 | 结论 |
| --- | --- | --- |
| 9 | §6.4 | HMAC 输入串的**字节编码固定**：nonce 用原始 32B、tokenId 用 ASCII 文本，附拼接表（§6.4） |
| 10 | §6.1 | **端口定死**为 51880；手填 IP 兜底支持 `ip:port`（§6.1、§5.3） |
| 11 | §6.5.2 | 键名表**删掉 `backspace`**（与 `delete` 语义重复） |
| 12 | §6.5.2 | 修饰键上限改 **5**，比较规则统一为"解析后按固定顺序重排再比" |
| 13 | §0.6 / §5.2 | 聚焦默认值表述对齐：**默认自动聚焦，具体开 / 关由 G1 决定**（v3.3 进一步改为运行时探测，§5.2） |
| 14 | §6.5.5 | `app` 绑定的目标 App 事后被卸载 → **静默忽略 + 一条日志** |
| 15 | §8 | 连接状态补定义：**已认证连接唯一**（新连接驱逐旧连接）；**未认证连接最多 3 条**，10 秒未完成认证即断开 |
| 16 | §6.1 | **单帧 ≤ 8KB**，超出回 `protocol_violation` 并断开 |
| 17 | §9.1 | 本地网络隐私是 **macOS 15 (Sequoia) 起**才有，macOS 14 上没有这一层（TN3179 已核实） |
| 18 | §13.2 | 自检页归 **M2**；M3 只留分发 + 干净机器验收 |
| 19 | §12.2 | 删掉 v2 残留的测试项"指令键语义 id 映射"（v3 的映射在 Mac 端） |
| 20 | §3 | 术语"指令键"补上自定义槽位 `custom1`–`custom16` |
| 21 | §11.1 | 明确 **codegen 只产常量 / 键表 / 错误码**，帧的编解码手写 |
| 22 | §6.4 | 配对限流**以 IP 维度为准**，指纹仅用于展示与聚合 |
| 23 | §6.2 | `mic_unavailable` 补进错误码表（v3 不用，保留给未来） |
| 24 | §5.2 | 错字"前most App" → "frontmost App" |

**第二轮评审（18 条）**

| # | 问题 | 结论 |
| --- | --- | --- |
| A1 | §6.5.2 的**语法块没跟着规则改**（仍写"最多 4 个""主键 0 或 1 个"），与同节规则 2 / 3 矛盾 | 语法块改为"最多 5 个""恰好 1 个"（§6.5.2） |
| A2 | **新建自定义槽位没有协议动作**——编号由 Mac 分配，但 `mappingSet` 必须带 `id`，没有"请求新建"的帧 | 新增 **`mappingCreate{rev}`** 帧 → `mappingResult{ok:true, rev, id:"custom17"}`（`mappingResult` 增加可选 `id`）（§6.1、§6.5.5） |
| A3 | **编号上限 16 + `customSeq` 只增不减 ⇒ 累计建过 16 次后功能永久锁死** | 上限重新定义为**同时存在的自定义槽位数 ≤ 16**；**编号 N 无上限**（`custom17`+ 合法）；"清空注册表"**不重置 `customSeq`**（§6.3、§6.5.5、§10） |
| A4 | `__voice` 的"未配置"没有表示法 | **`key:null` 即未配置**；对 `__voice` 写 `{"type":"none"}` → `invalid_binding`（§6.3） |
| B1 | §5.1 `ShortcutRegistry` 仍写"四型"、漏 `__voice` | 改为五型（含 `trigger`）+ 含 `__voice`（§5.1） |
| B2 | §3 术语「绑定」仍是四类 | 补 `trigger`（§3） |
| B3 | §12.3 "生成两份常量**与帧定义**"与"codegen 只产常量"冲突 | 删"与帧定义"（§12.3） |
| B4 | §13.0 计划 3 仍写"分发 **+ 自检页**" | 自检页已归 M2，改为"分发"（§13.0） |
| B5 | R7 仍写"IP **+ 指纹**限流"，与 §6.4 冲突 | 改为"仅按 IP"（§14） |
| B6 | `auth_failed` 仍写"tokenId 未知"，与"未知 tokenId 走配对"冲突 | 删掉"tokenId 未知"（§6.2） |
| C1 | 字母段漏了 `0x0A = kVK_ISO_Section`（17 个字母占 18 个码位） | 字母段改写为分段标注（§6.5.3，已核实 `Events.h:319`） |
| C2 | 端口口径不一致（"可改" vs "固定"） | 统一为"**默认 51880，可改**"（§6.1、§11.1） |
| C3 | 本地网络被拒的表现写反了方向 | Mac 侧是**注册**（浏览在手机端），因此 Mac 注册失败、Android **发现不到**；Mac 设置页要能显示"本地网络被拒"（§9.1） |
| C4 | 触发键注入失败没有错误码 | `start_tap_failed` 覆盖两种"开始投递失败"（长按 / 点按）（§6.2、§8） |
| C5 | `trigger` 类型写到 `custom1` 上没定义 | → `invalid_binding`；`trigger` **仅 `__voice` 有效**（§6.3） |
| C6 | 触发键的 `key` 取值与键规格修饰键名是两套词汇，且不在 `keyspec.json` 里 | 明确为**独立枚举** `triggerKey`，纳入 `protocol/` 的 codegen 源（§6.3） |
| C7 | "槽位与固定 id 地位完全相同"与"固定 id 不可删"矛盾 | 改为"可绑、可改、**可删（仅槽位）**"（§6.3） |
| C8 | 改映射时若正处于 `held`，释放可能读到错误的键 | 释放**必须走已注入键登记里的实际键**，不得重读注册表（§6.5.5、§8） |

**第三轮评审（6 条）**

| # | 问题 | 结论 |
| --- | --- | --- |
| D1 | **导入 / 导出会破坏 `customSeq` 规则**：导入一份没有 `customSeq` 的旧配置 → 立刻复用已删掉的编号 | 导出**必含 `customSeq` 与 `rev`**；导入时 `customSeq = max(文件值, 本机最大编号 + 1)`（缺字段或低于下限时忽略文件值）；导入时校验自定义槽位 ≤ 16，超出**拒绝整份导入**；导入后 `rev` 不倒退（§10、§12.1） |
| D2 | 第二轮表标题写"12 条"，实际 18 行 | 改为 18（并同步 §0.1 引言） |
| D3 | `mappingSet` 打不存在的编号被判 `protocol_violation` → 断开连接，代价过大 | **未知 id 分两类**：格式非法 → `protocol_violation`（结构问题，断开）；格式合法但从未创建 → `invalid_binding`（语义问题，**不断连**）。`protocol_violation` 收敛为"只管结构问题"（§6.1、§6.2、§6.3） |
| D4 | `custom<N>` 的 N 合法性没定义 | 正则 **`^custom[1-9][0-9]{0,9}$`**：N ≥ 1、最多 10 位；`custom0` / `custom-1` / 13 位数都非法（§6.3） |
| D5 | §12.3 的一致性检查没覆盖 `triggerKey` | 键名清单条目补"（含 `triggerKey` 枚举）"（§12.3） |
| D6 | 两处图文不一致 | §7.1 时序图补上**长按模式**的 `start_tap_failed` 分支；§6.1 要点补上 `mappingCreate` 也只接受已认证连接（§6.1、§7.1） |

**第四轮评审（18 条）**

| # | 问题 | 结论 |
| --- | --- | --- |
| P0-1 | **没有"哪些帧需要已认证连接"的总则**——按字面实现，同网段任意设备连上就能注入按键 | §6.1 加**认证总则**：认证前只接受 `hello` / `helloAuth` / `pairingRequest`（及 Mac→Android 的握手帧），其余一律 `error{auth_failed}` + 断开；§8 补一行（§6.1、§8） |
| P0-2 | **"未认证连接 10 秒未握手即断开"会杀掉配对流程**（人要走到 Mac 前点同意） | 已发 `pairingRequest`、等待审批的连接走 **120 秒配对窗口**，不套用 10 秒（§6.1、§8、§6.4） |
| P0-3 | **`preparing` 收到 `voiceStop` 未定义**（快按必现）→ 触发键 down 之后没有 up 路径，Fn 卡死 | `preparing` + `voiceStop` = **取消**（不投递；已投递则立即释放）；补 `preparing` 总超时（当时写作 3.5s，**v3.3 改为 0.8–1 秒**，见第五轮 P0-1）→ `focus_failed`（§7.1、§7.3） |
| P0-4 | **`action` / `app`"忽略 `phase` = 一次点按执行两次**（音量 +2、App 开两次） | 改为**仅 `phase:"press"` 执行，`release` 忽略**；§12.1 加"`release` 不触发"用例（§6.3、§12.1） |
| P1-5 | Mac **不认识 tokenId** 时手机收不到任何"去配对"的信号 | 新增 **`helloAck{paired:false}`** → 手机清 secret 与 tokenId 并发 `pairingRequest`（§6.1、§6.4） |
| P1-6 | `pairingResult{approved:true}` 之后当前连接算不算已认证没写 | 明确：**配对成功即把当前连接转为已认证**（无需重连、无需再走 HMAC），同时**驱逐既有已认证连接**（§6.4） |
| P1-7 | **`held` 没有最长持续时间**（Android 被冻结但 socket 还在回 ping → 触发键一直按下） | `held` 加 **120 秒**上限 → 走释放路径 + `error{voice_timeout}`；§13.3 加验收项（§7.3、§6.2、§8） |
| P1-8 | **G1 把 AX 聚焦耗时测在 ChatterFly 上是错的**——聚焦对象是 frontmost App 的输入框 | G1 拆成 **G1-a**（触发键与投递模式）/ **G1-b**（AX 聚焦耗时，测**原生 TextEdit + Chromium/Electron Chrome 或 Slack**）；§5.2 决策表改成"按目标 App 类别"，判据用**运行时快路径探测**而非 App 名单（§5.2、§13.1） |
| P1-9 | `stale_mapping` 只写 `mappingSet`，与"两者都必须带 rev"冲突 | 三处统一为 `mappingCreate` / `mappingSet`（§6.2、§6.5.5） |
| P1-10 | 导入下限漏了"**文件内最大编号 + 1**"；"导入是替换还是合并"没定义 | `customSeq = max(文件值, 本机最大编号+1, 文件内最大编号+1)`；**导入 = 整份替换**（不合并）（§10） |
| P1-11 | §5.3 "点按模式要改手势"与 §7.2 "点按是 Mac 端投递细节"矛盾 | 删掉 §5.3 那句，Android **永远按住 / 松开**（§5.3、§7.2） |
| P1-12 | 验收 #11 同 P1-11 的语义错误 | 改为"点按模式下**按住说话 → 松开触发停止拍**，文字上屏"（§13.3） |
| P2-1 | **"fn 不在 Android 的 keyCode / metaState 里"不成立** | **更正**：Android 有 `KeyEvent.KEYCODE_FUNCTION`（119）与 `META_FUNCTION_ON`（0x8）（已核实 android-37 SDK）；真正的限制是"多数键盘的 Fn 在固件 / HAL 层被消费、不送到前台应用，且不区分左右"（§5.3、§6.5.4、§14 R12） |
| P2-2 | Android 侧"Keystore 放不可导出的 HMAC key"意味着导入外部生成密钥，各机型支持不一 | 改为：**Keystore 内生成 AES-GCM 密钥加密 `secret`、密文存 DataStore**，不导入外部密钥（§10） |
| P2-3 | 缺"状态 × 帧"的处理表（`idle` 收 `voiceStop`、`held` 收 `button`…都没定义） | §7.3 加**状态 × 帧 处理表**，并区分"会话状态层面丢弃并记日志"与"语义错误回 `invalid_binding`"（§7.3） |
| P2-4 | `voice_busy` 没写哪些状态算 busy；`binding:null` 打到固定 id / `__voice` 的错误码没指定；`rev` / `customSeq` 初值没写 | 忙 = `preparing` / `held` / `releasing`；`binding:null` 打到不可删除的 id → `invalid_binding`；初值 `rev = 1` / `customSeq = 1`（§6.2、§6.5.5） |
| P2-5 | §6.4 "（2 位两位数容易看错成 8/6）"不通顺 | 改成"6 位数字比 2 位更不容易被相近数字混淆"（§6.4） |
| P2-6 | §11.2 没定义 Mac App 形态；`action: focus_composer` 是否回 `focusResult` 没定 | 形态定为**菜单栏常驻（`LSUIElement = true`）+ 首次启动一次性设置窗口**；`action: focus_composer` **不回 `focusResult`**（同步回执只属 `focusComposer` 帧）（§11.2） |

**第五轮评审（15 条）**

| # | 问题 | 结论 |
| --- | --- | --- |
| P0-1 | **"自动聚焦快速失败"没同步下去**：§7.1 / §7.3 / §8 / §12.1 五处仍写"快路径未命中转全路径 ≤3s""总超时 3.5s"，与 §5.2 冲突——照 §7 实现，第四轮的 P1-8 就白修了 | **自动路径只探快路径**（≤300ms）：§7.1 时序图、§7.3 `preparing` 与总超时（**0.8–1 秒**）、§8 拆成"自动聚焦超时 / 手动聚焦超时"两行、§12.1 `ComposerFocus` 用例（§5.2、§7.1、§7.3、§8、§12.1） |
| P0-2 | **`ping`/`pong` 与 6 秒判死没划界**，会把 120 秒配对窗口再杀一次（认证前白名单不含 `pong`） | **`ping`/`pong` 与 6 秒判死只作用于已认证连接**；未认证 / 待审批连接不发 `ping`、不判死，只受 10s / 120s 两个计时约束；`pong` 任何时候都允许（§6.1、§8） |
| P1-3 | `focusComposer` 被丢弃但没有回执 → Android 一直等 | 丢弃时回 **`focusResult{ok:false, detail:"session_busy"}`**（§7.3） |
| P1-4 | "Android 给出短提示或震动反馈"——`button` **没有回执通道**，实现不出来 | 改为**Android 本地禁用**：会话期间把指令键面板置灰（它自己知道发起了 `voiceStart`）；Mac 端丢弃只作兜底并记日志，**不新增回执帧**（§7.3） |
| P1-5 | "`mappingList` 是只读，**可放宽**"与认证总则直接矛盾，还等于让未配对设备读走快捷键表 | 删掉"可放宽"，**三个映射帧都要求已认证连接**（§6.1） |
| P2-1 | §8 / §12.1 仍只写"`mappingSet` 的 `rev` 冲突"（P1-9 漏改的两处） | 统一为 `mappingCreate` / `mappingSet`（§8、§12.1） |
| P2-2 | §15 参考资料表仍写着"`fn` 不在 keyCode / metaState 中"——错误结论留在"已核实"来源清单里 | 改为 `KEYCODE_FUNCTION` = 119 / `META_FUNCTION_ON` = 0x8（已核实 android-37 SDK），并写明真正的限制是固件 / HAL 消费（§15） |
| P2-3 | §14 R4 仍把 **ChatterFly 当聚焦对象**；R1 未标 G1-a；R14 缺 120 秒例外 | R4 改为"自动聚焦只探快路径 + **G1-b** 标定"；R1 → **G1-a**；R14 补配对窗口例外（§14） |
| P2-4 | 裸 `G1` 引用（§0.6 / §4.2 / §7.2 / §5.2 / §13.1）与已拆成 G1-a / G1-b 的事实不符 | 全部标成 G1-a / G1-b；§0.6 的"默认值由 G1 实测决定"改为指向 §5.2 的运行时规则（§0.6、§4.2、§5.2、§7.2、§7.3、§13.1） |
| P2-5 | §5.1 `ComposerFocus` 仍写"**必须先做** AX 前置宣告"——快路径刻意不宣告 | 改为"**全路径才做** AX 前置宣告"（§5.1） |
| P2-6 | 版本没升：line 4 仍是 v3.2、§0.1 标题仍是"v3.2 本轮变更"、引言末句没覆盖第四轮 | 升 **v3.3**；§0.1 改为"v3.2 / v3.3：评审修复（共五轮）"；引言按轮次重写（§0） |
| 补强-1 | "探测 ≤300ms"要能兑现，必须用 **`AXUIElementSetMessagingTimeout(0.3)`**（`AXUIElement.h:402`，`CF_AVAILABLE_MAC(10_4)`）——否则 AX 调用在 Chromium 上同步阻塞 1–2 秒，快速失败退化成慢速失败 | §5.2 写进实现要求（**探测后恢复默认超时**，否则全路径遍历也被截断）；§12.1 加用例（§5.2、§12.1） |
| 补强-2 | 快速失败的提示不能只有一句："焦点不在可编辑元素"（原生 App）与"深度壳"（Chromium 系）要用不同话术 | §5.2 加**两行分型表**（§5.2） |
| 补强-3 | §6.4 流程图把 `paired:false` 画在 `helloAuth` 之后，tokenId 未知其实收到 `hello` 就能判定 | 流程图改成 **hello → 判定**（省一次往返，也避免手机拿不存在的 secret 算 HMAC）（§6.4） |
| 补强-4 | `pairingResult{approved:false}` 之后连接是否关闭没写 | 明确：**保持连接**，手机可再发 `pairingRequest`（限流照旧）；不转为已认证、不占已认证坑位；不再重试则按 10s / 120s 规则断开（§6.4） |

**第六轮评审（9 条）**

| # | 问题 | 结论 |
| --- | --- | --- |
| 1 | **`focusResult.detail` 承担机器码**（`session_busy` / `focus_timeout`），但 `detail` 是自由文本、与 `error.detail` 同名 | `focusResult` 增加 **`code` 枚举**（`""` / `session_busy` / `focus_timeout` / `no_editable_target` / `secure_target`），`detail` 放人话；**未知 code 按通用失败展示**（§6.1、§7.3、§8） |
| 2 | **快速失败分型表漏了第三类**（命中元素、role 也在白名单，但被**安全边界拒绝**）——按第一行文案会给密码框提示"先点一下输入框"，用户照做还是失败 | 分型表加**"安全目标"**一行（"当前输入框不支持语音输入，请切换到普通输入框"）；§8 的提示分型同步（§5.2、§8） |
| 3 | `focus_failed` 描述仍含"宣告后仍无树"——那是手动全路径的失败，回的是 `focusResult` | 改成"**自动路径**的失败（快路径无可编辑目标 / 被安全边界拒绝）"，并注明手动路径走 `focusResult`（§6.2） |
| 4 | §12.2 没覆盖第五轮新增的客户端行为 | 加：`helloAck{paired:false}` → 清 secret 重配对（纯 JVM）、**会话期间指令键面板置灰**（Compose UI）、`voice_timeout` / `session_busy` 的 UI 处理（Compose UI）（§12.2） |
| 5 | §13.3 验收缺三项 | 补 **#23–#26**：手动聚焦超时 → `focusResult{ok:false,code:"focus_timeout"}`；会话中按指令键 → 面板是灰的；配对被拒后连接保持可重试；密码框提示分型（§13.3） |
| 6 | §12.1 WebSocket 用例仍写"6 秒无帧判死"，没加"仅已认证连接"限定 | 补限定，与 §8 一致；并补"`pong` 在认证前也允许回"（§12.1） |
| 7 | §5.2 "手动聚焦**总是**走全路径"与同节表格"手动也先走快路径"字面打架 | 改为"手动聚焦先走快路径，**快路径未命中时一定继续**全路径"（§5.2） |
| 8 | **未登记过的 `button` release** 未定义（press 被丢弃、release 落在 idle）→ 按字面会补发一次 up | **未登记的 release 一律忽略、不补发 up**——那个键从没被按下过；`KeyInjector` 侧只需一行判断（§7.3） |
| 附 | 全路径设过 `AXManualAccessibility` 后，Chromium 的树通常保持，后续会话的快路径会命中 | 写进 §5.2，免得用户以为"每个 App 每次都要先点聚焦键"（§5.2） |

### 0.2 v3.1 本轮变更：Android 端也能自定义快捷键

**遥控器的价值不止于语音，Android 端要成为快捷键的"录入面"。**

v3 里"用户可改为任意快捷键…"只有一句话（§6.3），而且改映射的入口全在 Mac 端（`ShortcutCapture` 录制，需要 Input Monitoring 权限）。这与产品形态是矛盾的：用户手里只有手机，配置快捷键却必须坐到 Mac 前。

v3.1 补上这条链路，**不改变核心架构**（仍然是 Android 发指令、Mac 注入键）：

| 变更 | 内容 |
| --- | --- |
| 指令键 id 扩展 | 12 个固定 id 之外，增加**用户自定义槽位** `custom<N>`（同时存在 ≤ 16 个，编号不复用）（§6.3） |
| 绑定类型 | 每个 id 可绑 `combo`（任意键组合）/ `action`（系统动作）/ `app`（打开已安装 App）/ `none`（禁用）（§6.3） |
| 键规格 | 跨端统一的规范化键规格字符串（如 `cmd+shift+4`），用 ANSI 物理键位编码，与键盘布局无关（§6.5） |
| 映射读写 | 新增 `mappingList` / `mappingSet` 两帧，Android 可读写 Mac 端注册表（§6.1、§6.5） |
| 录入 UI | Android 端新增"选键名 + 修饰键"录入，以及（可选）硬件键盘捕获（§5.3、§6.5） |
| 权限收益 | Mac 端 Input Monitoring 从"必需"降为"仅在 Mac 上录制时需要"（§9.1） |

**没变的**：Mac 端仍是唯一注入方、注册表权威副本仍在 Mac 端、触发键与语音会话时序完全不动。

### 0.3 v3 的核心决策：范围收敛

**语音输入完全交给 Mac 上已有的语音输入法（ChatterFly），Android 只做遥控器。**

v1 / v2 的设计是"Android 当麦克风"：Android 采集音频 → 推流到 Mac → 写入虚拟麦克风设备 → 语音输入法从虚拟设备收音。v3 废弃该方案，改为：

```
用户按住手机上的语音键
  → Mac 注入 ChatterFly 的收音快捷键（如 Fn）
  → ChatterFly 用 Mac 自带麦克风收音、识别、上屏
```

**理由**（按权重排序）：

1. **价值密度**：遥控器是确定能用的功能；"手机当麦克风"依赖目标 App 是否接受第三方虚拟设备，是一个未验证的假设（v1 的 G2 门至今悬而未决）。
2. **复杂度断层**：虚拟麦克风方案的全部硬问题都集中在驱动侧——HAL 插件开发、`transport = USB` 兼容性、Developer ID 签名与公证、PKG 安装与 coreaudiod 重启、跨进程音频 IPC、缓冲预算与首尾音时序。砍掉驱动，这整片风险归零。
3. **首发延迟**：遥控器形态没有"文字晚 0–5 秒出现"的问题（v2 的固有时延，见 §7.2）。
4. **平台约束**：不再需要 Android 麦克风权限与麦克风类型前台服务，Android 端退化为一个纯遥控 App。

### 0.4 v3 删除的内容

| 删除项 | 原位置 | 原因 |
| --- | --- | --- |
| 虚拟麦克风驱动（HAL 插件 / libASPL / 环回） | v2 §5.3 | 语音改走 Mac 自带麦克风 |
| 二进制音频帧协议 | v2 §6.1 | 不再传输音频 |
| 预录 / 预卷 / 三道防线 / 排空缓冲预算 | v2 §7.2 | 无音频链路 |
| Android `AudioRecord` / 前台录音服务 / 麦克风权限 / 通知权限 | v2 §5.4 | 不采集音频 |
| 驱动 `.pkg` 安装链路 / 驱动签名公证 / `transport=USB` | v2 §11.3 | 无驱动可分发 |
| 验证门 G2（目标 App 是否接受 USB transport） | v2 §13.1 | 无虚拟设备 |
| 验证门 G3（端到端音频连通） | v2 §13.1.1 | 无音频链路 |
| 风险 R2（目标 App 不枚举虚拟麦克风）、R3（首尾音丢失）、R6（驱动安装） | v2 §14 | 前提已消失 |

### 0.5 v3 保留并继续深化的部分

| 保留项 | 说明 |
| --- | --- |
| 配对 + 设备令牌 + HMAC 挑战应答 | **权重上升**：现在唯一的输出能力是键盘注入，冒充已信任设备 = 能在用户机器上敲任意键 |
| AX 聚焦（含前置宣告） | 保留全部细节，见 §5.2 |
| 触发键注入与快捷键录制 | 核心功能 |
| 连接生命周期管理（双向判死、断连清理、连接驱逐） | **权重上升**：没有音频兜底，键卡住的代价更直接 |
| 指令键语义映射 | 保留并扩展（v3.1：自定义槽位 + 四类绑定，见 §6.3、§6.5） |

### 0.6 早前已拍板且不受本轮影响的三项

| 决策项 | 结论 |
| --- | --- |
| 信任模型 | 配对下发 32B 设备令牌，之后每连接 HMAC-SHA256 挑战应答 |
| 聚焦归属 | 默认自动聚焦（同步、阻塞），进入语音触发的关键路径；**具体行为由 §5.2 的规则决定**——自动聚焦只探快路径（≤300ms），未命中即快速失败；手动聚焦键走全路径 |
| 分发方式 | 不上架 Mac App Store；Developer ID 直分发，.dmg 分发 .app；不启用 App Sandbox |

---

## 1. 背景与目标

### 1.1 要解决的问题

用户希望用 Android 手机**代替小米蓝牙遥控器**，充当 Mac 的语音键盘：

1. 手机上的语音键要能启动 / 停止 Mac 上语音输入法的收音；
2. 手机上的其他按键要能触发 Mac 的键盘快捷键，并且**能在手机上自定义要触发什么**；
3. 按下语音键时，要把焦点切到当前 App 的输入框；
4. Mac 端需要人工批准手机的连接请求。

**语音的采集、识别、上屏全部由 Mac 上已有的语音输入法完成**（本机为 ChatterFly，装在 `~/Library/Input Methods/Chatterfly.app`）。VoiceBridge 不碰音频。

### 1.2 核心设计取舍

**Mac App 不认识、不适配任何具体输入法。**

参考项目 SayAll 为此维护了一张"遥控器 × 输入法"能力矩阵（5 种输入法 × 3 种遥控器 × 2 种收音模式 × 快捷键侧别），并为此反复返工：ChatterFly 在 SayAll 1.9.21 被以"不再支持"从首次设置中移除（`remote-mic-app/Resources/zh-Hans.lproj/ReleaseHistory.md:17`），历史配置需迁移为未选中（`Sources/RemoteMic/AppSettings.swift:1032`）。

VoiceBridge 把这一层完全交给用户：

| 事项 | 谁负责 |
| --- | --- |
| 用哪个输入法 | 用户 |
| 输入法的收音快捷键是什么 | 用户（Mac 端可配置） |
| 长按收音 / 按一次收音 | 用户（Mac 端配置） |
| 在哪里说话 | 用户（对着 Mac 的麦克风） |
| 按下用户配置的触发键 | VoiceBridge |
| 执行用户配置的快捷键 | VoiceBridge |
| 切换输入框 | VoiceBridge（默认自动） |
| 批准/拒绝手机连接 | VoiceBridge |

### 1.3 成功标准

用户在目标 App 里，按住手机上的语音键**对着 Mac 麦克风说话**，松开后文字出现在输入框里——全程不需要碰 Mac。

### 1.4 已废弃的方案（决策溯源，留档）

v1 / v2 曾设计"Android 当麦克风 + 自研虚拟麦克风驱动"。该方案技术上可行（参考实现 SayAll 用 BlackHole 派生驱动在生产中跑通了），但在本项目上**投入产出比不成立**，故在 v3 废弃。废弃时已查明的关键技术事实，留档备查：

- **跨进程共享内存在本机不成立**：HAL 插件被 coreaudiod 以 `_coreaudiod`（uid 202）身份加载，App 是普通用户（uid 501）；coreaudiod 的 sandbox profile（`/System/Library/Sandbox/Profiles/com.apple.audio.coreaudiod.sb`）的 `file-write*` 白名单只有 `TMPDIR`、`/Library/Preferences(/Audio)`、`/dev/dtracehelper`，**不含 `/private/tmp`**；且两端 `TMPDIR` 不同。唯一可行的跨进程方案是 Mach/XPC。
- **可行的替代通道是 Core Audio 客户端 API**：参考实现让 App 用 `AVAudioEngine` + `AVAudioPlayerNode` 把 PCM 写进虚拟设备的输出流（`AudioOutput.swift:989`），设备做 output→input 环回，**完全不需要自定义 IPC**。若日后重启该方案，从这里接手即可。
- **环回设备没有回放缓冲**：读端开流之前写入的音频直接丢失（参考实现注释原话），这是任何"手机当麦克风"方案都必须先解决的首字丢失问题。

---

## 2. 范围

### 2.1 包含

- macOS App（Swift）：局域网服务、配对批准、快捷键录制与注入、AX 聚焦输入框、触发键投递
- Android App（Kotlin/Compose）：遥控 UI（语音键 / 指令键面板 / 聚焦键 / 状态灯）、WebSocket 客户端、Bonjour 发现
- 跨端协议：控制帧（JSON over WebSocket）
- **快捷键自定义**：在 Android 端录入键组合 → 写入 Mac 端注册表 → Mac 执行（§6.5）

### 2.2 明确不包含

| 不做 | 原因 |
| --- | --- |
| **音频采集与传输** | 语音走 Mac 自带麦克风与已有语音输入法（§0.3） |
| **虚拟麦克风驱动** | 同上 |
| 适配任何具体输入法 | 见 §1.2，用户自行配置 |
| 输入法能力矩阵 / 工具识别 / 截图指引 | 上述决定的直接结果 |
| 网页版 / Apple Watch 版 / iOS 版 | 非本次范围 |
| 多客户端并发 | 只允许一个手机连接 |
| 跨互联网连接 | 局域网足够，且避免服务器运维 |
| 指令键手势（双击 / 长按） | v1 不做。注意：语音键的"长按 / 点按"是投递模式，不是手势，两者不冲突 |

---

## 3. 术语

| 术语 | 含义 |
| --- | --- |
| 触发键 | 用于启动/停止语音输入法收音的单个修饰键（Fn / 左Command / 右Command / 左Option / 右Option）。**只有触发键有长按 / 点按之分**；配置在注册表的保留条目 `__voice` 里（§6.3） |
| 指令键 | 手机上除语音键外的语义按键，映射到 Mac 快捷键或系统动作（§6.3 的 12 个固定 id + 用户自定义槽位 `custom<N>`，同时存在 ≤ 16 个） |
| 语音会话 | 一次"按下 → 准备 → 收音 → 释放" 的完整生命周期 |
| 设备令牌 | 配对成功时下发的 32B 随机密钥，Android 存 Keystore、Mac 存 Keychain，用于后续连接的 HMAC 校验 |
| 目标就绪 | AX 聚焦完成、目标 App 的输入框已就位（对应参考实现的 `destinationReadiness`） |
| 键规格（KeySpec） | 跨端统一的规范化键组合字符串，如 `cmd+shift+4`；用 ANSI 物理键位编码，与键盘布局无关（§6.5.2） |
| 绑定（binding） | 一个指令键 id 指向的目标：`combo`（键组合）/ `action`（系统动作）/ `app`（打开已安装 App）/ `trigger`（触发键，仅 `__voice`）/ `none`（禁用）（§6.3） |
| 自定义槽位 | 用户在 Android 端新增的指令键，id 形如 `custom<N>`；**同时存在 ≤ 16 个**，编号 N 无上限且不复用（§6.3、§6.5.5） |

---

## 4. 系统架构

```
┌──────────────────┐   WebSocket（局域网）    ┌────────────────────────────────┐
│  Android App     │ ───────────────────────► │  Mac App (Swift)               │
│  Kotlin/Compose  │       控制帧 JSON        │  ├ PairingService  配对+令牌校验 │
│                  │                         │  ├ ProtocolCodec   编解码      │
│  遥控 UI         │ ◄── Bonjour 服务注册 ── │  ├ KeyInjector     CGEvent 注入│
│  ├ 大语音键       │                         │  ├ ShortcutRegistry 语义映射    │
│  ├ 指令键面板     │                         │  ├ KeySpecParser  键规格解析   │
│  │  └ 自定义槽位  │                         │  ├ SystemActionRunner 系统动作 │
│  ├ 聚焦键         │                         │  ├ ShortcutCapture 快捷键录制  │
│  ├ 快捷键录入     │                         │  ├ ComposerFocus   AX 聚焦     │
│  Discovery(NSD)  │                         │  └ VoiceSession    语音会话    │
└──────────────────┘                         └───────────────┬────────────────┘
                                                             │ CGEvent 注入
                                                             ▼
                                              用户的语音 App（ChatterFly / 豆包…）
                                              ↑ 用 Mac 自带麦克风收音、识别、上屏
```

### 4.1 仓库结构

```
voicebridge/
├── mac-app/                 Xcode 工程（.xcodeproj），产出 VoiceBridge.app
├── android-app/             Gradle 工程（Kotlin + Compose）
├── protocol/                协议定义（JSON schema + 帧格式 + codegen）
└── docs/superpowers/specs/  设计规格
```

**`mac-app/` 必须是 Xcode 工程，不能只用 Swift Package**：Swift Package 的可执行 target 产不出带 `Info.plist` 的 `.app` bundle，而 `.app` 是签名、公证、Info.plist 权限键（§9）与拖拽安装的前提。`mac-app/` 用 Xcode 工程 + 一个可被单测引用的 Swift Package target 承载核心逻辑（`VoiceBridgeCore`），使 §12.1 的单测不必依赖 App 壳。

`protocol/` 下的协议定义是**跨端唯一事实源**：Mac 端、Android 端都从这里取常量与契约（帧格式、JSON 字段名、触发键集合、错误码），避免两处各写一遍导致漂移。**必须以 codegen 方式落地**（见 §12.3），不能靠两份手抄。

### 4.2 触发键集合

**v1 固定为这 5 个**，不提供自定义按键或组合键：Fn、左 Command、右 Command、左 Option、右 Option。

限定为**单个修饰键**而非任意组合键，理由：

1. 长按语义要求明确的 down/up 配对，组合键会引入"哪个键是触发器"的歧义；
2. 会话期间用户仍在打字，组合键修饰位会污染正常输入；
3. 目标输入法的收音快捷键本身就是单键。

参考：`remote-mic-app/Sources/RemoteMic/VoiceKeyMode.swift:60` 明确排除了 Shift / Control。

目标输入法若只认其他按键，属于 v1 范围外的限制，需在文档中显式告知用户。**具体目标输入法认哪个键、认长按还是点按，由验证门 G1-a 实测确定**（§13.1）。

---

## 5. 组件设计

### 5.1 Mac App（Swift，macOS 14+）

| 模块 | 单一职责 | 关键约束 / 参考 |
| --- | --- | --- |
| `PairingService` | WebSocket 服务、握手、HMAC 校验、配对批准、长期信任 | 只允许一个活动客户端；配对流程参考 `remote-mic-app/Sources/RemoteMic/BridgeAppModel.swift:6277` `requestPhoneApproval`；令牌与信任列表存 Keychain（§10） |
| `ProtocolCodec` | 控制帧 JSON 编解码 | 纯函数，无 I/O，可单测 |
| `ShortcutRegistry` | 指令键语义 id → **绑定** 的映射表 | 12 个固定 id + `__voice`（触发键）+ `custom<N>` 槽位；绑定**五型** `combo` / `action` / `app` / `trigger` / `none`（§6.3）；**Mac 端持有权威副本**，带单调递增的 `rev` 与 `customSeq`；纯数据，可导入导出；**不随配置导出信任信息** |
| `KeySpecParser` | 规范化键规格 → (修饰键 flags, 主键 virtual keycode) | 纯函数、无 I/O、可单测；键名不在规范表内 → `invalid_binding`；主键用 **ANSI 物理键位**编码（`kVK_ANSI_*`），与键盘布局无关（§6.5.2） |
| `KeyInjector` | CGEvent 注入，成对 down / up | 注入事件必须写 `eventSourceUserData` 标记，避免被 `ShortcutCapture` 当成用户输入捕获回去（参考 `remote-mic-app/Sources/RemoteMic/KeyboardInjector.swift:1878` `postKey` 写 `eventSourceUserData`，公开入口 `postKeyState` 在 1891）；**按连接登记已注入的键**，连接结束时统一释放（§8） |
| `SystemActionRunner` | 执行 `action` / `app` 类绑定 | 音量与播放控制用 **`NSSystemDefined` 系统键事件**（它们没有 virtual keycode，§6.5.6）；打开已安装 App 走 `NSWorkspace.openApplication`；`focus_composer` 复用 `ComposerFocus`；`none` 直接忽略 |
| `ShortcutCapture` | event tap 录制任意快捷键 | 组合键走 `keyDown`；独立修饰键走 `flagsChanged` 配对检测（参考 `remote-mic-app/Sources/RemoteMic/ShortcutCaptureMonitor.swift`）；**可选路径**——Android 端录入（§6.5）已覆盖大部分需求，Mac 端录制仍保留；**需要 Input Monitoring 权限**（§9.1） |
| `ComposerFocus` | AX 遍历找可编辑输入框并设置焦点 | 安全边界见 §5.2；**全路径才做 AX 前置宣告**（快路径刻意不宣告，见 §5.2） |
| `VoiceSession` | 语音会话状态机 | 见 §7 |

### 5.2 AX 聚焦的安全边界

`ComposerFocus` 只对**安全可编辑目标**执行聚焦，其余情况返回失败并保持现状。

允许的角色：`AXTextArea`、`AXTextField`、`AXComboBox`。
拒绝：`AXSecureTextField`（密码框）——**按 role 和 subrole 双重判断**：只查 role 会漏掉那些 role 为 `AXTextField` 但 subrole 为 `AXSecureTextField` 的元素，参考实现两者都查。
拒绝的语义文本关键词（中英双语，参考 `remote-mic-app/Sources/RemoteMic/VoiceInputDestinationCoordinator.swift:80`）：

```
password, passcode, secret, api key, apikey, token, credit card,
search, find, filter, address bar, settings, preferences, command palette,
密码, 口令, 密钥, 令牌, 信用卡, 搜索, 查找, 筛选, 设置, 偏好
```

**AX 前置动作（缺了会在 Chrome / VS Code / Slack 里永远找不到输入框）**

Chromium / Electron 系的 Web 内容壳在**辅助客户端宣告自己之前，辅助功能树是空的**——此时 AX 遍历只会得到一个空壳，找不到任何可编辑元素。参考实现的顺序是：

1. 对目标进程设置 `AXManualAccessibility`（Chromium / Electron 约定，优先尝试）；
2. 若目标 App 对该属性返回 `attributeUnsupported`，回退设置 `AXEnhancedUserInterface`（VoiceOver 用的标准属性；部分 Electron 版本下它会扭曲窗口与动画，所以只作回退）；
3. 设置完成后再做 AX 遍历与聚焦。

参考实现常量（`remote-mic-app/Sources/RemoteMic/KeyboardInjector.swift:175-187`）：

```swift
static let manualAccessibilityAttribute = "AXManualAccessibility"
static let enhancedUserInterfaceAttribute = "AXEnhancedUserInterface"
static let composerFocusMaximumAttempts = 12
static let composerFocusRetryMilliseconds = 250
```

**重试的真实原因**：不是"等 1–2 秒让树稳定"，而是**宣告之后 Chromium / Electron 仍需 1–2 秒才建好 Web 内容树**。因此遍历上限 5,000 个元素（防卡死），重试 12 次 × 250ms（**最多 3 秒**）。

**快路径（v3 新增，缓解聚焦耗时）**

聚焦在语音触发的关键路径上（§7.1），而**聚焦期间用户说的话会丢**——v2 靠 Android 预录兜住，v3 没有音频链路，这个兜底消失了。原生 App 的 AX 树是现成的（聚焦 ~50ms），只有 Chromium / Electron 系才需要 3 秒。

因此聚焦分两段：

| 段 | 做什么 | 时限 | 谁走这条路 |
| --- | --- | --- | --- |
| **快路径** | 直接查 frontmost App 的 `AXFocusedUIElement`，命中可编辑元素就用它 | **≤ 300ms** | **自动聚焦**（语音触发）与**手动聚焦**都先走它 |
| **全路径** | AX 前置宣告 → 完整遍历 + 重试 | ≤ 3 秒 | **只有手动 `focusComposer` 帧走**（idle 态的"聚焦键"）。**自动聚焦不走** |

快路径的价值在于把常见情况的延迟压到 100ms 以内；**但"查 `AXFocusedUIElement` 本身会不会触发 Chromium 建树"需要实测**，G1-b 要测出这个数（§13.1）。

**一个好消息：Chromium 的树一旦建好就一直有效。** 全路径设过 `AXManualAccessibility` 之后，Chromium / Electron 的 Web 内容树通常**保持可用**，所以**后续语音会话的快路径会命中**（只是首次要等树建好那 1–2 秒）。因此用户**不需要为每个 App 每次都先点"聚焦键"**——只有"这个 App 还没被全路径宣告过"时才会落到快速失败。

**自动聚焦只探快路径**：未命中就快速失败（下面的决策表），**不进入 3 秒全路径**——否则"快速失败"就变成了"慢速失败"，还白白丢掉等待窗口里用户说的话。**手动聚焦**（点手机上的"聚焦键"）**总是走全路径**：用户是主动要求聚焦的，愿意等这 1–3 秒。

**快路径的 300ms 必须真的只花 300ms**（v3.3 补）：`AXUIElementCopyAttributeValue` 是**同步阻塞**调用，不设超时时它会在 Chromium / Electron 系上**阻塞 1–2 秒**（等 Web 内容建树）——"快速失败"就退化成了"慢速失败"，P1-8 的规则等于白写。因此**探测前必须调用 `AXUIElementSetMessagingTimeout(systemWide, 0.3)`**（`AXUIElement.h:402`，`CF_AVAILABLE_MAC(10_4)`，对 system-wide 元素设置 AX 消息超时），让 AX 调用在 300ms 内返回。**探测完必须把超时恢复成默认值**——否则后续的全路径遍历（本来要 3 秒）也会被截到 300ms。

**快速失败的提示要分型**（v3.3 补，v3.4 加第三类）：

| 情况 | 判定 | 给用户的提示 |
| --- | --- | --- |
| **焦点不在可编辑元素**（原生 App：焦点在按钮 / 列表 / 桌面） | 快路径**拿到了元素**，但角色不在白名单 | "先点一下输入框再说话" |
| **安全目标**（命中了可编辑元素，但**被安全边界拒绝**：`AXSecureTextField` / 敏感关键词，见本节上文） | role 在白名单内，但**命中拒绝规则** | "当前输入框不支持语音输入，请切换到普通输入框" |
| **深度壳**（Chromium / Electron 系） | 快路径**没拿到任何可编辑元素** | "这个 App 需要先手动聚焦（点手机上的聚焦键）" |

**两条路径各用各的回执，但分型是同一套**：自动路径回 `error{focus_failed}`（§7.1），手动路径回 `focusResult{ok:false, code:"no_editable_target" | "secure_target"}`（§6.1）。

**三类不能用同一句话**：原生 App 里如果也喊"这个 App 需要手动聚焦"，用户会莫名其妙（他明明就在 TextEdit 里）；**"安全目标"那一类尤其要分开**——按第一行的文案提示"先点一下输入框"，用户点了**还是失败**（密码框永远是密码框），只会以为 App 坏了。

**默认值的决策规则（按目标 App 类别，G1-b 给出依据）**（v3.2 修正：G1-b 要测的是**目标 App 的输入框**，不是 ChatterFly——ChatterFly 是后台输入法，不参与聚焦）

| 目标 App 类别 | 自动聚焦 | 深度聚焦（全路径） | 为什么 |
| --- | --- | --- | --- |
| **原生 App**（TextEdit / Xcode / Word / 备忘录…） | **默认开启** | 默认关闭 | 快路径 ≤300ms 命中，聚焦代价可忽略 |
| **Chromium / Electron 系**（Chrome / Slack / VS Code / Electron 应用…） | 自动聚焦会**快速失败**（见上） | **默认关闭** | 深度聚焦要 1–3 秒，等待窗口内用户说的话会丢（§7.1） |

**判据不靠 App 名单，靠运行时探测**：语音会话开始时先探一次快路径（≤300ms，代价可接受）——

- **命中可编辑元素** → 原生系，聚焦完成，立即投递触发键；
- **未命中** → 判定为"深度壳"，**自动聚焦到此为止**（不进入 3 秒全路径），回 `error{focus_failed}`，提示按上面的分型给（§5.2 的"快速失败的提示要分型"）。

**为什么不自动等 3 秒**：等待窗口内用户说的话会丢（§7.1），而用户这会儿可能刚开始说话——用 3 秒沉默换一个"字打进错误的框"不划算。快速失败 + 明确提示，用户点一下手机上的"聚焦键"（这条路径**知道**自己在等），体验更可控。

> **手动"聚焦键"先走快路径，快路径未命中时**一定**继续走全路径**（最多 3 秒），不受上面两个默认值影响——用户是主动要求聚焦的，愿意等。

若 G1-b 实测发现"查焦点元素本身"在 Chromium 系上同样要等 1–2 秒（快路径失去意义），则把**自动聚焦整体降级为默认关闭的独立动作**（§14 R4）。

**聚焦失败**：回 `error{focus_failed}`，会话终止，**不降级为"不聚焦直接收音"**——降级会把文字打进错误的输入框，比失败更糟。

### 5.3 Android App（Kotlin，minSdk 26）

| 模块 | 单一职责 |
| --- | --- |
| `RemoteUi` | Compose：大语音键（按住 / 滑动释放）/ 指令键面板（含自定义槽位）/ 聚焦键 / 状态灯 |
| `ShortcutEditor` | 绑定编辑：选键名 + 修饰键、硬件键盘捕获、拉取 / 写回 Mac 端注册表（§6.5） |
| `WsClient` | OkHttp WebSocket，控制帧 |
| `Discovery` | NsdManager 局域网发现 + **手动输入 `ip:port`** 兜底（只填 IP 时用默认端口 51880，§6.1） |
| `Settings` | DataStore：Mac 端地址 |

**Android 端是一个纯遥控 App，没有麦克风、没有前台服务、没有常驻通知。** 这带来的直接简化：

| 省掉的东西 | 说明 |
| --- | --- |
| `RECORD_AUDIO` 权限 | 不采集音频 |
| `FOREGROUND_SERVICE` / `FOREGROUND_SERVICE_MICROPHONE` | 没有前台服务 |
| `POST_NOTIFICATIONS` | 没有常驻通知 |
| 电池优化白名单引导 | 没有后台保活需求——遥控 App 不常驻，用户打开才连 |
| 麦克风类型前台服务的 while-in-use / `BOOT_COMPLETED` 限制 | 没有麦克风服务，这条平台约束整体消失 |

Android 端只需要 `INTERNET` + Wi-Fi 状态相关权限（见 §9.2）。

**手势**：语音键**恒为"按住生效、松开结束"**。即使 G1-a 实测目标输入法只认点按模式，**手势也不变**——点按是 **Mac 端的投递细节**（§7.2）：Mac 收到 `voiceStart` 注入开始拍，收到 `voiceStop` 注入停止拍。Android 永远只发"按住 / 松开"。

> v3.1 曾写过"点按模式下语音键手势要改成按一下开始、再按一下停止"——**那是错的**：一次按住会被拆成"开始拍 + 立即停止拍"，用户说的话还没开始就结束了。v3.2 删掉这句。

**自定义快捷键的录入（v3.1）**

遥控器的用户形态是"人不在 Mac 前"，因此**录入面放在手机**（§6.5）：

| 方式 | 操作 | 依赖 | 限制 |
| --- | --- | --- | --- |
| **选键名 + 修饰键**（主路径） | 指令键面板里长按某个 id（或新建自定义槽位）→ 选主键 + 勾选修饰键 | 无 | 键名较多，需要分类与搜索 |
| **硬件键盘捕获**（增强） | 手机接蓝牙 / USB 键盘，直接按组合键，手机 App 捕获后填入 | 手机接了键盘 | 只能捕获 App 在前台且获焦时的按键；**`fn` 未必能被捕获**——Android 的按键模型里虽然有 `KeyEvent.KEYCODE_FUNCTION`（119）与 `META_FUNCTION_ON`（0x8），但多数键盘的 Fn 在**固件 / HAL 层就被消费**（用来切换 F1–F12），不会作为按键事件送到前台应用，且 Android **不区分左右 Fn**；需要 fn 时只能手选 |
| **改已有绑定** | `mappingList` 拉回当前注册表，在手机端改完写回 | 无 | 只是编辑已有项 |

**Android 无法凭空"按下"一个只存在于 Mac 上的组合键**——录入是"声明"而不是"试按"。要验证绑定对不对，用 Mac 端 App 的**试按一次**（§13.2 M2 的自检页）。

**触发键也在手机端配**（v3.2）：触发键是注册表的保留条目 `__voice`，在手机端选一个修饰键 + 选投递模式（长按 / 点按）即可，不必坐到 Mac 前（§6.3）。

Android 端**不需要任何新权限**：捕获硬件按键走 Activity 的 `dispatchKeyEvent`，前台 App 自身即可接收，不涉及系统权限（§9.2）。

---

## 6. 协议规范

传输：局域网内的一条 WebSocket 连接（`ws://`）。控制帧用 JSON 文本帧。

**信道安全性**：v1 使用明文 `ws://`。连接级认证由 §6.4 的设备令牌 + HMAC 提供；**帧级完整性与机密性不在 v1 范围**，残余风险见 §14 R3。

### 6.1 控制帧

所有帧含 `"v"`（协议版本，当前 `1`）与 `"t"`（类型）。

```
—— 握手 ——
{"v":1,"t":"hello","device":"Pixel 8","fingerprint":"<sha256>","tokenId":"<hex16>","clientNonce":"<base64 32B>"}
{"v":1,"t":"authChallenge","serverNonce":"<base64 32B>"}
{"v":1,"t":"helloAuth","mac":"<base64 32B HMAC-SHA256>"}
{"v":1,"t":"helloAck","paired":true}
{"v":1,"t":"helloAck","paired":false}

—— 配对 ——
{"v":1,"t":"pairingRequest"}
{"v":1,"t":"pairingCode","code":"042815"}
{"v":1,"t":"pairingResult","approved":true,"tokenId":"<hex16>","secret":"<base64 32B>"}
{"v":1,"t":"pairingResult","approved":false,"reason":"user_denied"}

—— 语音会话 ——
{"v":1,"t":"voiceStart"}
{"v":1,"t":"voiceStop"}

—— 指令 ——
{"v":1,"t":"button","id":"ok","phase":"press"}
{"v":1,"t":"button","id":"custom3","phase":"press"}
{"v":1,"t":"focusComposer"}
{"v":1,"t":"focusResult","ok":true,"code":"","detail":""}
{"v":1,"t":"focusResult","ok":false,"code":"session_busy","detail":"语音会话进行中"}

—— 快捷键映射（v3.1）——
{"v":1,"t":"mappingList"}
{"v":1,"t":"mappingListResult","rev":7,"customSeq":3,"items":[{"id":"__voice","type":"trigger","key":null,"mode":"hold"},{"id":"ok","type":"combo","spec":"enter"},{"id":"custom1","type":"action","action":"volume_up"},{"id":"custom2","type":"none"}]}
{"v":1,"t":"mappingCreate","rev":7}
{"v":1,"t":"mappingSet","rev":7,"id":"custom1","binding":{"type":"combo","spec":"cmd+shift+4"}}
{"v":1,"t":"mappingSet","rev":7,"id":"custom2","binding":null}
{"v":1,"t":"mappingSet","rev":7,"id":"__voice","binding":{"type":"trigger","key":"fn","mode":"tap"}}
{"v":1,"t":"mappingResult","ok":true,"rev":8}
{"v":1,"t":"mappingResult","ok":true,"rev":8,"id":"custom17"}
{"v":1,"t":"mappingResult","ok":false,"code":"stale_mapping","rev":9}

—— 保活 ——
{"v":1,"t":"ping"}
{"v":1,"t":"pong"}

—— 错误（双向） ——
{"v":1,"t":"error","code":"focus_failed","detail":""}
```

要点：

| 要点 | 说明 |
| --- | --- |
| **没有 `sid`** | 单连接模型下"连接即身份"：一条 WebSocket 连接对应一个客户端，Mac 不需要额外的连接标识。v3.1 的 `sid` 只出现在部分帧里（协议与正文互相矛盾），v3.2 直接删除。**迟到帧的防护靠两层**：新连接到达即驱逐旧连接并关闭其 socket（迟到帧随 socket 一起消失）+ 语音会话的 `generation` 机制丢弃迟到的异步回调（§7.3） |
| **端口** | **默认 51880，可改**。Bonjour 注册 `_voicebridge._tcp` 时携带该端口；Android 自动发现走 Bonjour（拿到 IP + 端口），**手填 IP 兜底支持 `ip:port` 形式**（只填 IP 时用默认端口）。改了端口后 Bonjour 发现依然有效（手填兜底的用户需填新端口） |
| `hello.tokenId` | **可选**：首次配对的设备还没有令牌，**省略该字段**即表示"请求配对"；带上则表示"我有令牌，请校验" |
| `helloAck` | 握手结果。**`paired:true`** = 令牌有效，连接**立即转为已认证**；**`paired:false`** = **Mac 不认识这个 tokenId**（被移除信任 / 换机 / 重装），手机应**清掉本地 secret 与 tokenId** 并发 `pairingRequest` 重新配对（§6.4）。**没有这一帧手机就收不到"去配对"的信号** |
| `pairingRequest` / `pairingCode` | 配对是**两个阶段**：`pairingRequest`（手机 → Mac）**只表示"我要配对"，不带码**；Mac 生成 6 位码后用 `pairingCode`（Mac → Android）推给手机，两端各自显示给人眼比对。**手机从不提交这个码**（§6.4） |
| `id`（`button`） | 指令键 id：12 个固定 id（§6.3）或用户自定义槽位 `custom<N>`（编号 N 无上限，但**同时存在的槽位 ≤ 16**，§6.3）。Mac 收到**不存在的 id** → `error{invalid_binding}`（语义错误，**不断连**）；**只有帧结构问题**才用 `protocol_violation` 并断开。**不静默忽略**（静默忽略会让用户以为按了没反应） |
| `voiceStart` / `voiceStop` | Android 端表达"按住 / 松开"，**不表达目标输入法的长按还是点按**——那是 Mac 端的配置（§7.2） |
| `focusResult` | `focusComposer` 的回执，没有它验收清单里的用例无法判定。**`code` 是机器可读的枚举**（两端各写各的字符串会让 UI 只能模糊匹配），**`detail` 放给人看的话**：`code` ∈ `""`（成功）/ `session_busy`（会话进行中）/ `focus_timeout`（全路径 3 秒超时）/ `no_editable_target`（没有可编辑目标）/ `secure_target`（被安全边界拒绝，如密码框）。**收到未知 `code` 一律按"通用失败"展示**，不得崩溃（§6.2） |
| `mappingList` / `mappingCreate` / `mappingSet` | 快捷键注册表的读 / **新建** / 写（v3.1，§6.5.5）。`mappingCreate` = "请给我建一个自定义槽位"，**不带 id**——编号由 Mac 分配，回执 `mappingResult{ok:true, rev, id:"custom17"}` 里的 `id` 才是新槽位的编号。**三个帧都要求已认证连接**——`mappingList` 虽然是只读的，但读到的**就是用户整套快捷键表**，未配对设备不该看得到。**`binding:null` = 删除该槽位**；`{"type":"none"}` 是"禁用但保留"，两者语义不同 |
| `customSeq` | Mac 端持有的**下一个可分配的自定义槽位编号**（单调递增，只增不减）。自定义槽位的编号由 Mac 端分配，Android 不本地另存编号状态（否则两端必然分叉，§6.3） |
| `__voice` | **保留 id**，存放触发键配置（`type:"trigger"`，`key` 为 `null` 表示**未配置**）。不占 16 个自定义槽位，且不可删除（§6.3） |
| **单帧长度** | **≤ 8KB**（JSON 编码后的完整帧）。超出 → `error{protocol_violation}` 并断开连接 |
| `error` | **双向**。Mac 可报 `focus_failed` / `accessibility_denied` / `trigger_key_not_configured`；Android 可报 `mic_unavailable`（v3 已无麦克风，该码保留给未来） |
| `ping` / `pong` | **只对已认证连接生效**：Mac 每 2 秒发 `ping`，**6 秒内没收到任何帧即判死**（§8）。**未认证 / 待审批连接不发 `ping`、也不做 6 秒判死**——它们只受"未认证 10 秒"与"配对窗口 120 秒"两个计时约束（否则配对等待中会被自己的心跳打断，或被静默判死）。`pong` 任何时候都允许回 |

**认证总则：哪些帧需要已认证连接（v3.2 补）**

| 阶段 | 允许的帧 |
| --- | --- |
| **认证前**（socket 刚建连） | Android → Mac：`hello`、`helloAuth`、`pairingRequest`、`pong`；Mac → Android：`authChallenge`、`pairingCode`、`pairingResult`、`helloAck`、`error`（**Mac 不对未认证连接发 `ping`**） |
| **认证后** | 全部帧 |

**认证前收到上表以外的帧**（`voiceStart` / `voiceStop` / `button` / `focusComposer` / `mappingList` / `mappingCreate` / `mappingSet` / `ping` …）→ 回 `error{auth_failed}` 并断开连接。

**这条总则是安全底线，不是形式条款**：如果实现成"没认证也能发 `button`"，那么同网段任意设备连上 51880 端口就能在用户机器上敲任意键——这与 §6.4 威胁模型里"未授权设备冒充已信任设备"是同一件事的另一个入口。**配对流程本身是唯一的例外通道**，而它必须有人工闸门（Mac 上点「同意」）。

**配对审批窗口**：已发出 `pairingRequest`、正在等待用户在 Mac 上点「同意」的连接，**不走"10 秒未认证即断开"**（§8），单独走 **120 秒**的配对窗口（弹窗期间计时照走，但到 120 秒仍未审批则断开并撤销弹窗）。配对要人走到 Mac 前，10 秒的统一超时会把配对成功率直接吃掉。

### 6.2 错误码

| code | 含义 | 触发方 |
| --- | --- | --- |
| `focus_failed` | **自动路径**的 AX 聚焦失败（快路径无可编辑目标 / 被安全边界拒绝）。**手动全路径的失败不走这个码**——它回 `focusResult{ok:false, code:...}`（§6.1、§7.3） | Mac → Android |
| `accessibility_denied` | 辅助功能权限缺失，无法注入按键 | Mac → Android |
| `trigger_key_not_configured` | **触发键未配置**（用户还没选收音快捷键）。与 `accessibility_denied` 分开：前者的补救动作是"去 Mac 端 / 手机端选触发键"，后者是"去系统设置授权" | Mac → Android |
| `start_tap_failed` | **开始投递触发键失败**（长按模式的 down 注入失败，或点按模式的开始拍失败，§7.2）。对齐参考实现 `VoiceFnTapSessionController.swift:30` 的 `start_tap_failed` | Mac → Android |
| `stop_tap_failed` | 点按模式的**停止拍**注入失败——**输入法可能仍在持续收音**，用户必须看到（§7.3）。对齐参考实现 `VoiceFnTapSessionController.swift:31` 的 `stop_tap_failed` | Mac → Android |
| `auth_failed` | 设备令牌校验失败（HMAC 不匹配）。**tokenId 未知不走这里**——那是"没有令牌"，走配对流程（§6.4） | Mac → Android |
| `pairing_rate_limited` | 配对请求过于频繁，已限流（§6.4） | Mac → Android |
| `protocol_mismatch` | 协议版本不一致 | Mac → Android |
| `protocol_violation` | **帧结构违规**（缺字段、未知帧类型、单帧超长、id 格式非法）——**一律断开连接**。语义错误（如引用了不存在的槽位）**不走这个码**，见 `invalid_binding` | 双向 |
| `voice_busy` | 已有语音会话在进行中。**"忙" = 状态处于 `preparing` / `held` / `releasing`**（这三个状态下不接受新的 `voiceStart`） | Mac → Android |
| `voice_timeout` | **语音会话超时**：会话处于 `held` 超过 **120 秒**仍未收到 `voiceStop`（Android 进程被冻结但 socket 还在回 ping 的反例，§7.3） | Mac → Android |
| `invalid_binding` | 绑定非法：键规格无法解析 / 键名不在规范表内 / `app` 的 bundleId 未安装 / **`binding:null` 打到了固定 id 或 `__voice`**（这两类不可删除） | Mac → Android |
| `mapping_limit` | 超出映射操作上限（**同时存在的自定义槽位数 > 16**，或 `mappingSet` / `mappingCreate` 过于频繁） | Mac → Android |
| `stale_mapping` | **`mappingCreate` / `mappingSet`** 携带的 `rev` 与服务端当前版本不符（两处同时改过） | Mac → Android |
| `mic_unavailable` | **v3 不用**：项目已不采集音频，该码保留给将来可能恢复的场景 | Android → Mac |

> 用户拒绝配对**不走错误码**：直接回 `pairingResult{approved:false, reason:"user_denied"}`，与"批准"共用一个帧。错误码只用于异常。

> `invalid_binding` / `mapping_limit` / `stale_mapping` 三个码在**有专用回执的帧**（`mappingCreate` / `mappingSet`）上通过 **`mappingResult{ok:false, code}`** 交付（§6.5.5）；在没有回执的帧上（例如 `button` 引用了不存在的槽位）走通用 `error` 帧。**错误码词表只有一套**，交付通道按帧而定。

**协议版本协商**：`hello` 帧中的 `"v"` 必须与 Mac 端支持的版本相同，否则 Mac 回 `error{protocol_mismatch}` 并关闭连接，双方均不做降级猜测。版本号在 `protocol/` 中单点定义。

未知错误码的接收方必须按"未知错误"处理：UI 展示通用失败文案，不得崩溃。

### 6.3 指令键语义 id

Android 端只发送**语义 id**，不发送键码。映射关系由 Mac 端持有（**权威副本**），**Android 端可读可写它**（v3.1，§6.5）——改映射不需要发版任何一端。

固定集合（对齐参考项目 `remote-mic-app/Sources/PublicRemoteCompatibility/Core/RemoteCompatibility.swift:3`）：

```
power, up, left, ok, right, down, back, volume_up, home, volume_down, menu, tv
```

**wire 值**：参考实现里枚举**名**是 `volumeUp` / `volumeDown`，但 `rawValue`（也就是真正上线的协议值）是 `volume_up` / `volume_down`（`case volumeUp = "volume_up"`）。协议里用的是 rawValue。

`phase` ∈ `press` | `release`。默认映射（参考 `remote-mic-app/TECHNICAL.md:87`），第二列给出**规范写法**（§6.5.2 的键规格 / 系统动作）：

| 按钮 | 默认动作 | 规范写法 |
| --- | --- | --- |
| 方向（up/down/left/right） | 方向键 | `up` / `down` / `left` / `right` |
| ok | Return | `enter` |
| back | Delete（退格） | `delete` |
| home | 显示桌面（Fn-F11） | `fn+f11` |
| menu | macOS 上下文菜单键 | `menu` |
| tv | Command-Tab | `cmd+tab` |
| power | Escape | `esc` |
| volume_up / volume_down | 系统音量增减 | `action: volume_up` / `action: volume_down` |

**绑定类型**：每个 id 的绑定是下列类型之一

| type | 含义 | 触发时的行为 |
| --- | --- | --- |
| `combo` | 任意键组合 | 注入键组合：`press` → 修饰键 down + 主键 down；`release` → 主键 up + 修饰键 up。**主键恰好 1 个**——无主键的规格写入即 `invalid_binding`（§6.5.2） |
| `action` | 系统动作 | **仅在 `phase:"press"` 时执行一次**，`release` 忽略（无状态动作）。音量与播放控制用 `NSSystemDefined` 系统键事件实现（§6.5.6） |
| `app` | 打开已安装 App | **仅在 `phase:"press"` 时执行**，`release` 忽略。用 `NSWorkspace` 启动该 `bundleId` 的 App |
| `trigger` | **触发键**（**仅 `__voice` 有效**） | 不直接注入；决定语音会话投递哪个修饰键、用长按还是点按模式（§7.2）。写到 `custom1` 等其它 id 上 → `invalid_binding` |
| `none` | 禁用 | 什么都不做（槽位保留但不可用）。**对 `__voice` 不合法** → `invalid_binding`（触发键只有"未配置"和"已配置"两种状态） |

`action` 的取值：`volume_up`、`volume_down`、`mute`、`media_next`、`media_prev`、`media_play_pause`、`focus_composer`。
`app` 的 `bundleId` 必须指向本机已安装的 App，否则 `invalid_binding`（§6.5.5）。

**`action` / `app` 只认 `press`**（v3.2 修正）：Android 端每个指令键都发 **press + release 两帧**，若两相都执行，**一次点按会触发两次**——`volume_up` 音量 +2、绑定的 App 被打开两次。这两类绑定是无状态的，没有"按住"语义，所以 `release` 直接忽略。只有 `combo` 绑定才用 press/release 配对表达"按住不放"（方向键连发等）。

**保留 id `__voice`：触发键**

触发键（启动 / 停止语音输入法收音的那个修饰键）**也是注册表的一项**，这样手机端就能配它——否则 v3.1 "录入面在手机上" 的立论，恰好在最该在手机上配的那一个键上破了（§6.5.5）。

| 字段 | 取值 |
| --- | --- |
| `id` | `__voice`（双下划线前缀 = 保留 id，非用户槽位） |
| `type` | `trigger` |
| `key` | `null` = **未配置**（首次安装的初始状态）；否则为 §4.2 的 5 个触发键之一 |
| `mode` | `hold`（长按模式）/ `tap`（点按模式） |

**"未配置"的表示法**（v3.2 补）：首次安装时注册表里 `__voice` 就是

```json
{"id":"__voice","type":"trigger","key":null,"mode":"hold"}
```

`key:null` 表示"用户还没选触发键"，此时按语音键 → `error{trigger_key_not_configured}`（§8）。`mode` 在 `key` 为 `null` 时无意义，保留默认值 `hold` 即可。

它**不占 16 个自定义槽位**，也**不可删除**；`mappingSet` 对它的写入按普通条目处理，但**不能写成 `{"type":"none"}`**——触发键没有"禁用"这个状态，只有"未配置"和"已配置"。

**`key` 的取值是独立枚举**：`fn` / `cmd_l` / `cmd_r` / `opt_l` / `opt_r` 与键规格里的修饰键名（`cmd` / `ctrl` / `opt` / `shift` / `fn`）**是两套不同的词汇**——键规格的修饰键没有左右之分，触发键有。**它同样纳入 `protocol/` 的 codegen 源**（与 `keyspec.json` 同批生成），不手写两份。

**用户自定义槽位（v3.1）**

12 个固定 id 不够表达"我自己的一组快捷键"，因此用户在 Android 端可以新增槽位：

- id 形如 `custom<N>`，**N 无上限**（`custom17`、`custom45` 都合法）；**N 的合法性由正则定义**：`^custom[1-9][0-9]{0,9}$`——N ≥ 1、最多 10 位，因此 `custom0`、`custom-1`、`custom1234567890123` 都是非法 id；
- **同时存在的自定义槽位数 ≤ 16**——这是**数量**上限，不是编号上限（v3.2 修正：v3.1 写成"N ∈ 1..16" + `customSeq` 只增不减，累计建过 16 次后即使全删光也再也建不出新槽位，功能会永久锁死）；
- 槽位**可绑、可改、可删**；固定 id **可绑、可改、不可删**（删固定 id 等于抹掉功能）；
- **新建走 `mappingCreate{rev}`**（不带 id），Mac 分配编号并在回执里给出 `id`（§6.5.5）；
- **编号由 Mac 端分配**：Mac 在注册表里维护 `customSeq`（下一个可分配编号），随 `mappingListResult` 下发。Android **不在本地另存编号状态**——两端各自维护"已用编号"必然分叉；
- **删除后不复用该编号**：`customSeq` 只增不减，删掉的编号成为空洞（避免历史配置错位）；
- 删除的写法是 `mappingSet` 带 `binding:null`；`{"type":"none"}` 是"禁用但保留"，两者语义不同；
- **未知 id 分两类**（v3.2 拆分，边界要清楚）：

  | 情况 | 处理 | 为什么 |
  | --- | --- | --- |
  | **格式非法**（不满足 `^custom[1-9][0-9]{0,9}$`，如 `custom0`、`customx`） | `error{protocol_violation}` → **断开连接** | 帧结构违规 |
  | **格式合法但从未创建**（如 `custom99`） | `mappingResult{ok:false, code:"invalid_binding"}` → **不断连** | 语义错误（UI 竞态 / 手误），为这个踢掉整条连接代价过大 |

用户可改为：任意键盘快捷键、系统音量、播放控制、打开已安装 App、聚焦输入框、禁用。

### 6.4 配对流程

**信任模型**：设备令牌（32B）+ HMAC-SHA256 挑战应答。

- 早期版本用 Android 在 `hello` 里**明文自报**的 `fingerprint` 作信任键，信道又是明文 `ws://`。同网段嗅到一次 `hello` 就能复制指纹、冒充已信任设备，直接拿到键盘注入权限——**而拿到键盘注入权限就等于能在用户机器上敲任意键**。v3 中 `fingerprint` 只用于**界面展示**（"Pixel 8"），不再作为信任凭据。

**令牌分发（首次配对）**

```
Android                                    Mac
──────                                    ───
hello{device,fingerprint,clientNonce} ───►  没有 tokenId → 走配对流程
pairingRequest{} ───────────────────────►  生成 6 位配对码（配对窗口 120 秒）
pairingCode{code} ◄─────────────────────  ├ Mac 弹窗：设备名 + 同一个码
                                            │ Android 端显示同一个码，供用户比对
                                            │ 用户点「同意」
pairingResult{approved:true,               存 Keychain：
               tokenId, secret} ◄────────  ├ tokenId（公开标识）
              （secret 由 Mac 生成）         └ secret（32B，Keychain）
                                            └ 当前连接**立即转为已认证**
```

**配对成功后当前连接就算已认证**（v3.2 明确）：`pairingResult{approved:true}` 之后，**这条连接直接转为已认证**，**不需要断开重连、不需要再走一遍 HMAC**——令牌就是这条连接刚刚建立的信任。同时它**驱逐既有已认证连接**（"已认证连接唯一"，§8）：用户拿新手机配对时，老手机的连接被踢掉是预期行为，不是异常。

**配对码是 Mac 生成、单向推给两端的**：Android 的 `pairingRequest` **不带 `code`**（§6.1）——手机没有资格决定"配对码是多少"，它只能请求配对；码由 Mac 生成后用 `pairingCode` 帧推给手机，两端各自显示给人看。

**令牌校验（后续连接）**

```
hello{device,fingerprint,tokenId?,clientNonce} ──►  Mac
                                                 ├ 无 tokenId（首次）→ 走配对流程（见上）
                                                 ├ tokenId 未知 → **立刻** helloAck{paired:false}
                                                 │                → 手机清 secret 并发 pairingRequest
                                                 └ tokenId 已知 → 生成 serverNonce（32B 随机，每连接唯一）
authChallenge{serverNonce}                     ◄──
helloAuth{mac}  ──────────────────────────────►  mac = HMAC-SHA256(...)（见下表）
                                                 ├ 比对成功 → helloAck{paired:true}
                                                 │              → 当前连接转为已认证
                                                 └ 比对失败（mac 不匹配）→ error{auth_failed} 并关闭连接
```

**`tokenId` 未知在收到 `hello` 时就能判定**（v3.3 补）：不必等手机算完 HMAC 才回话——省一次往返，也避免手机拿一个**根本不存在的 secret** 去算 HMAC。

**`paired:false` 的含义**：Mac 的信任列表里**没有**这个 tokenId（用户在 Mac 上移除了信任 / 换了 Mac / 重装了 Mac App / 换了手机）。此时**不是攻击**（mac 都没算出来就没得比），而是"这个令牌作废了"——所以回 `paired:false` 而不是 `auth_failed`：手机据此**清掉本地 secret 与 tokenId**，改发 `pairingRequest` 重新配对。**没有这一帧，手机就收不到"去配对"的信号**，只能一直重连同一个无效令牌。

**配对被拒绝后连接不关闭**：回 `pairingResult{approved:false, reason:"user_denied"}` 后**保持连接**，手机可以再发一次 `pairingRequest`（限流规则仍然生效：同一 IP 10 分钟 3 次弹窗）。该连接**不转为已认证**，也不占"已认证连接唯一"的位置；手机不再重试时，按未认证连接的 10 秒 / 120 秒规则断开。

**HMAC 的字节编码必须固定死**（v3.2 修复：原文只写了符号连接，没写各部分是原始字节还是文本，两端会各自实现出不同的结果）：

| 部分 | 参与计算的字节 | 说明 |
| --- | --- | --- |
| HMAC key | `secret` 的 **原始 32B** | base64 只是传输编码，计算前先解码 |
| 域分隔前缀 | ASCII `"vb1-auth"`（8 字节） | 防止同一密钥被跨协议复用 |
| `clientNonce` | **原始 32B** | 同上 |
| `serverNonce` | **原始 32B** | 同上 |
| `tokenId` | **ASCII 文本**（16 个小写 hex 字符 = 16 字节） | `tokenId` 是 hex **字符串**，不按 8 字节二进制参与计算 |

```
mac = HMAC-SHA256(
        key  = secret(32B 原始),
        msg  = "vb1-auth"(ASCII 8B) ‖ clientNonce(32B 原始) ‖ serverNonce(32B 原始) ‖ tokenId(ASCII 16B))
```

**`tokenId` 大小写**：Mac 端把收到的 `tokenId` **先规范化为小写 hex**再参与计算，避免大小写差异导致校验失败（手机端也应统一输出小写）。
**`serverNonce` 每连接唯一**：32B 随机、每条连接一个新值，作用是防重放（旧的 `helloAuth` 重放到新连接上必然算不出同样的 mac）。

**配对码是人工核对的显示码，不是凭据。**

协议里没有任何一端把码提交给另一端做校验——两端都从 Mac 拿码、只显示给人看。因此"尝试上限 / 有效期 / 防在线爆破"这套语义**没有落点**，不能假装它成立。v3 的处理：

| 项 | 结论 |
| --- | --- |
| 码长 | **6 位**，纯粹为了人眼比对不易错——6 位数字比 2 位更不容易被相近数字混淆；长度不承担安全职责 |
| 有效期 / 尝试上限 | **删除**——码不参与校验，没有"过期"与"尝试"的概念 |
| 码的价值 | 把"用户手里这台手机"与"Mac 上这个弹窗"绑定起来：用户比对两端显示的码，确认发起配对的就是自己手里的设备 |

**真正的防护是这三条**：

1. **用户必须在 Mac 上点「同意」**——不点就没有令牌；
2. **配对弹窗限流**（防同网段设备反复弹窗骚扰）：

   | 规则 | 值 | 理由 |
   | --- | --- | --- |
   | 同一来源 IP | **10 分钟内最多 3 次配对弹窗** | 超出回 `error{pairing_rate_limited}` |
   | 计数维度 | **以 IP 维度为准** | 设备指纹是手机**自报**的，可随意伪造，**不作为限流维度**（否则限流可被伪造指纹绕过）；指纹只用于界面展示与日志聚合 |

3. **配对之后所有连接都要 HMAC 校验**——令牌此后不再上线，只上线 HMAC。

**威胁模型（写进文档，不含糊）**

| 攻击 | 是否挡住 | 说明 |
| --- | --- | --- |
| 未授权设备自报 fingerprint 冒充已信任设备 | ✅ | 必须出示 HMAC，fingerprint 不再是凭据 |
| 配对码在线爆破 | — | **不适用**：码不参与任何校验，无从爆破 |
| 配对弹窗骚扰 | ✅ | 按来源 IP 限流（§6.4） |
| 事后嗅探者（配对完成后才上钩） | ✅ | 令牌此后不再上线，只上线 HMAC |
| **配对握手期间的被动嗅探者** | ❌ | 明文 `ws://` 下 secret 会经过网络一次 |
| 主动中间人 | ❌ | 同上；且能通过配对码比对——中间人若忠实转发，码在两端都显示为同一个，用户看不出问题 |

后两项是明文信道的固有代价。**升级路径**：v1.1 引入 `wss://` + Mac 端自签证书 + Android 侧 pin 公钥，可同时关闭后两项；v3 把它列为备选而不做（§14 R3）。

### 6.5 自定义快捷键（Android 端录入 → Mac 端注册表）

#### 6.5.1 为什么录在手机上

v3 里改映射必须坐到 Mac 前：`ShortcutCapture` 在 Mac 上录键，需要 Input Monitoring 权限。遥控器的使用形态是"人不在 Mac 前"，因此录入面必须在手机上。

v3.1 的分工：

| 环节 | 在哪 | 说明 |
| --- | --- | --- |
| 录入（人按什么键） | **Android** | 选键名 + 修饰键；或用接在手机上的硬件键盘捕获 |
| 校验与解析 | **Mac** | `KeySpecParser` 把规范化键规格解析成 CGEvent 配方；非法则拒绝 |
| 存储（权威副本） | **Mac** | `ShortcutRegistry`（UserDefaults JSON），Android 侧只是副本 |
| 执行（真按下去） | **Mac** | `KeyInjector` / `SystemActionRunner` |

Android 端**不能**凭空"按下"一个只存在于 Mac 上的组合键——所以录入是"声明"，不是"试按"。要验证绑定对不对，用 Mac 端 App 的**试按一次**（§13.2 M2 的自检页）。

#### 6.5.2 键规格（KeySpec）

绑定写成**规范化 ASCII 串**，例如 `cmd+shift+4`、`ctrl+alt+space`、`f5`、`fn+f11`。

```
键规格 = 修饰键("+")* 主键
修饰键 ∈ cmd | ctrl | opt | shift | fn          （0..5 个，不重复）
主键   ∈ <规范键名表>                            （恰好 1 个）
```

规则：

1. 全小写 ASCII，`+` 连接；**修饰键在前、主键在后**。
2. **主键恰好 1 个**。无主键的规格（例如只有 `cmd`）**写入即 `invalid_binding`**——单个修饰键只属于触发键（`__voice`），不属于普通指令键。
3. 修饰键取自 `cmd` `ctrl` `opt` `shift` `fn`，**不重复**；上限 **5 个**（就是全部 5 个，没有额外保留位）。
4. 主键名必须取自规范键名表；不在表内 → `invalid_binding`（写入时经 `mappingResult` 返回，§6.5.5）。
5. **比较前先归一**：解析后按 `cmd` `ctrl` `opt` `shift` `fn` 的固定顺序重排修饰键再拼接。因此 `shift+cmd+a` 与 `cmd+shift+a` **等价**——比较发生在解析之后，不是原始字符串直接比。

规范键名表：

| 类别 | 规范名 |
| --- | --- |
| 字母 | `a`–`z` |
| 数字 | `0`–`9` |
| 功能键 | `f1`–`f20` |
| 符号 | `` ` `` `-` `=` `[` `]` `\` `;` `'` `,` `.` `/` |
| 命名键 | `space` `tab` `enter` `esc` `delete` `forward_delete` `home` `end` `pageup` `pagedown` `up` `down` `left` `right` `menu` |
| 小键盘 | `keypad0`–`keypad9` `keypad_plus` `keypad_minus` `keypad_multiply` `keypad_divide` `keypad_enter` `keypad_decimal` `keypad_equal` |

`delete` = 退格（对齐 Android `KEYCODE_DEL` 与 macOS `kVK_Delete`），`forward_delete` = 前向删除（`KEYCODE_FORWARD_DEL` / `kVK_ForwardDelete`）。**命名统一按 Android 习惯**，避免"退格到底叫 delete 还是 backspace"的两端歧义——因此表里**不保留 `backspace`**（它和 `delete` 是同一个键，留着只会让两端各选一个）。

**媒体 / 音量键不在键名表里**：它们在 macOS 上**没有 virtual keycode**，只能用 `NSSystemDefined` 系统键事件实现（§6.5.6），所以走 `action` 类型，不走 `combo`。把它们塞进键名表会让 `KeySpecParser` 无法表达（v3.1 的漏洞，v3.2 修掉）。

`menu` = macOS 的上下文菜单键，对应 **`kVK_ContextualMenu`（0x6E）**——SDK 里**没有** `kVK_Menu` 这个常量，写代码时别想当然（已核实：本机 SDK 的 `HIToolbox/Headers/Events.h`）。

**物理键位，不是字符**：`a` 表示 ANSI 布局上 A 键**所在的那个物理位置**（macOS `kVK_ANSI_A`），不是"当前布局下打出 a 的那个键"。因此法语 / 德语布局的用户按物理 A 键也能命中 `a`。这让 Android 侧只需要知道 keycode，不需要知道 Mac 的键盘布局。

#### 6.5.3 两张表的单一事实源

| 表 | 方向 | 产物 |
| --- | --- | --- |
| Android `KeyEvent.KEYCODE_*` → 规范名 | 录入时把手机上的按键翻译成规范名 | `protocol/keyspec.json`（规范键名清单）codegen → `KeySpecCatalog.kt` |
| 规范名 → macOS virtual keycode | 注入时把规范名翻成 CGEvent 用的键码 | 同一份 `keyspec.json` codegen → `KeySpecCatalog.swift` |

`protocol/keyspec.json` 是**唯一的规范键名清单**，两张表都由它生成；CI 跑生成器并校验 `git diff` 为空（§12.3）。因此"两端认识的键名集合永远相同"由机制保证，不靠人肉同步。

**映射表必须逐条列举，不能用公式推**（已核实：本机 SDK `Events.h`）：

| 段 | 事实 |
| --- | --- |
| 字母 | **顺序是 QWERTY 而不是字母表**：`A S D F H G Z X C V` 对应 `0x00`–`0x09`，**`0x0A` 是 `kVK_ISO_Section`（ISO § 键，不是字母）**，`B Q W E R Y T` 对应 `0x0B`–`0x11`——**17 个字母占了 18 个码位，中间空着一个 § 键**。按字母序推断必然错位（v3.1 的表述有误导，v3.2 修正，已核实 `Events.h:319`） |
| 数字 | 同样是乱序：`1 2 3 4 6 5 = 9 7 - 8 0` 对应 `0x12`–`0x1D` |
| 功能键 | **不连续**：`kVK_F1 = 0x7A`、`kVK_F10 = 0x6D`、`kVK_F11 = 0x67`、`kVK_F12 = 0x6F`、`kVK_F13 = 0x69`、`kVK_F14 = 0x6B`、`kVK_F15 = 0x71`、`kVK_F16 = 0x6A`、`kVK_F17 = 0x40`、`kVK_F18 = 0x4F`、`kVK_F19 = 0x50`、`kVK_F20 = 0x5A` |

因此 `keyspec.json` 里存的是**显式对照表**，生成器只做搬运，不做算术。

#### 6.5.4 录入方式（Android 端）

三种方式的对照表见 §5.3。要点：

- 主路径是**选键名 + 修饰键**；硬件键盘捕获是增强路径，且 **`fn` 未必捕获得到**（Android 的按键模型里有 `KeyEvent.KEYCODE_FUNCTION` = 119 / `META_FUNCTION_ON` = 0x8，**已核实 android-37 SDK**——但多数键盘的 Fn 在固件 / HAL 层就被消费，且 Android 不区分左右 Fn，§5.3）；
- 捕获只能发生在 App 前台且获焦时；
- Android **不新增任何权限**。

#### 6.5.5 协议与约束

```
Android                                     Mac
──────                                      ───
mappingList ─────────────────────────────►  读注册表
mappingListResult{rev, customSeq, items[]} ◄─  rev = 注册表版本号（单调递增）
mappingCreate{rev} ───────────────────────►  新建一个自定义槽位（不带 id，编号由 Mac 分配）
mappingResult{ok:true, rev, id} ◄──────────  id = 新建的槽位编号（仅创建成功时带）
mappingSet{rev, id, binding} ───────────►  写注册表（binding:null = 删除槽位）
mappingResult{ok:true, rev} ◄─────────────
mappingResult{ok:false, code, rev} ◄──────  stale_mapping / invalid_binding / mapping_limit
```

`mappingResult` 是 `mappingCreate` / `mappingSet` 的**专用回执**，其 `code` 取自 §6.2 的错误码表，**不另造一套**。用专用回执而不是通用 `error` 帧，是因为写入失败时客户端需要拿到最新的 `rev` 才能继续（§8 的三条对应处理）。

`binding` 是五选一：

```json
{"type":"combo","spec":"cmd+shift+4"}
{"type":"action","action":"volume_up"}
{"type":"app","bundleId":"com.apple.Safari"}
{"type":"trigger","key":"fn","mode":"hold"}
{"type":"none"}
```

`binding:null` 表示**删除该槽位**——只对 `custom<N>` 合法；`__voice` 与 12 个固定 id 不可删除（删固定 id 等于抹掉功能）。`{"type":"trigger"}` 只对 `__voice` 合法；`{"type":"none"}` 对 `__voice` **不合法**（§6.3）。

`mappingListResult.items[]` 的每一项与 `binding` 同构，另带 `id`：

```json
{"id":"__voice","type":"trigger","key":"fn","mode":"hold"}
{"id":"ok","type":"combo","spec":"enter"}
{"id":"custom1","type":"action","action":"volume_up"}
{"id":"custom2","type":"none"}
```

**自定义槽位的编号由 Mac 端分配**：`mappingListResult.customSeq` 是下一个可分配编号。Android 要新建槽位时发 **`mappingCreate{rev}`（不带 id）**，Mac 分配编号、建好槽位（初始绑定 `{"type":"none"}`），并在 `mappingResult{ok:true, rev, id:"custom17"}` 里把 `id` 给回来。Android **不在本地另存"已用编号"列表**——两端各自维护必然分叉（§6.3）。

**编号与上限的关系**（v3.2 修正）：

| 约束 | 值 | 说明 |
| --- | --- | --- |
| 编号 N | **无上限** | `custom17`、`custom45` 都合法——N 只是计数器，不是配额 |
| 同时存在的自定义槽位数 | **≤ 16** | 这是**数量**上限。建到第 17 个 → `mapping_limit`；删掉一个又回到 16，可以再建 |
| `customSeq` | **只增不减** | 删除的编号成为空洞，永不复用（避免历史配置错位） |
| "清空注册表" | **不重置 `customSeq`** | 否则编号会被复用，与上一条冲突 |

> v3.1 把上限写成"N ∈ 1..16" + `customSeq` 只增不减，累计建过 16 次之后**即使全删光也建不出新槽位**（没有合法的 N）——功能会永久锁死。v3.2 把上限改成"同时存在的数量"，编号与配额解耦。

**并发与覆盖**：注册表带单调递增的 `rev`。`mappingCreate` / `mappingSet` 必须带客户端读到的 `rev`；不匹配 → `mappingResult{ok:false, code:"stale_mapping"}`（附最新 `rev`），客户端重新拉表再改。**不做自动合并**——两处同时改同一个 id 时，"告诉用户冲突了"比定义一套合并语义更容易解释。

**初值**：空注册表的 `rev = 1`、`customSeq = 1`（第一个创建的槽位是 `custom1`）。`rev` 每次成功写入 **+1**；`customSeq` 每次 `mappingCreate` 成功 **+1**。

**`app` 绑定的目标被卸载**：写入时校验 bundleId 已安装；**之后**用户把该 App 卸载了，按下这个槽位时**静默忽略 + 记一条日志**（不弹错误、不回执——按下是即时动作，回执已经来不及，而"App 被卸载"不是需要立刻打断用户的错误）。

**安全边界**：

1. `mappingCreate` / `mappingSet` 只接受**已通过 HMAC 校验**的连接；未认证连接直接拒绝，不回 `mappingResult`。
2. 单连接 **10 分钟内最多 20 次**写操作（`mappingCreate` + `mappingSet` 合计），超出 → `mappingResult{ok:false, code:"mapping_limit"}`。
3. 每次写入记一条日志（时间 + 设备名 + id + 绑定），便于事后回溯"是谁改的"。
4. 这**不是新的信任等级**：已认证设备本来就能在 Mac 上敲任意键，改一条映射不扩大能力面；真正的门槛是配对时的用户点同意（§6.4）。

**自定义槽位上限 16**：注册表存在 UserDefaults（JSON），且 `mappingList` 每次整表拉取；16 个自定义槽位足够覆盖"我自己的一组快捷键"，同时把注册表体量与同步开销钉在有界范围。

**与语音触发互不影响**：改映射不需要断开连接，写入后下一条 `button` 帧即按新绑定执行。

**但正在 `held` 时改 `__voice` 有个坑**：会话已经开始、触发键已经按下，此时把 `__voice` 改成别的键，**释放时必须释放"已注入键登记"里记录的那个实际键**，**不得重读注册表**——否则会释放错键（漏释放 → 修饰键卡死；或释放了没按下的键）。登记与释放的规则见 §8。

#### 6.5.6 `action` 怎么执行：系统键事件

音量与播放控制**没有 virtual keycode**，走不了 `KeyInjector` 的常规 `CGEvent` 路径，必须发 `NSSystemDefined` 事件。

参考实现的做法（`../remote-mic-app/Sources/RemoteMic/KeyboardInjector.swift:2065-2087`）：

```swift
private static func postSystemKey(type: Int32, isDown: Bool) {
    let keyState = isDown ? 0xA : 0xB
    let data1 = Int((type << 16) | Int32(keyState << 8))
    guard let event = NSEvent.otherEvent(
        with: .systemDefined, location: .zero, modifierFlags: [],
        timestamp: ProcessInfo.processInfo.systemUptime,
        windowNumber: 0, context: nil,
        subtype: 8, data1: data1, data2: -1
    ) else { return }
    guard let cgEvent = event.cgEvent else { return }
    cgEvent.setIntegerValueField(.eventSourceUserData, value: syntheticEventMarker)
    cgEvent.post(tap: CGEventTapLocation.cghidEventTap)
}
```

`type` 是 NX_KEYTYPE 取值，参考实现里的对应关系（`KeyboardInjector.swift:389-400`）：

| `action` | NX_KEYTYPE | 参考实现调用 |
| --- | --- | --- |
| `volume_up` | 0 | `postSystemKey(type: 0)` |
| `volume_down` | 1 | `postSystemKey(type: 1)` |
| `mute` | 7 | `postSystemKey(type: 7)` |
| `media_play_pause` | 16 | `systemKeyPoster(16)` |
| `media_next` | 17 | `systemKeyPoster(17)` |
| `media_prev` | 18 | `systemKeyPoster(18)` |

`systemKeyPoster` 在参考实现里是 `postSystemKey` 的注入点别名（`KeyboardInjector.swift:291`）。

**注入的事件同样要写 `eventSourceUserData` 合成标记**，否则会被自己的 `KeyboardEventSuppressor`（`KeyboardEventSuppressor.swift:21` 监听 `systemDefined` 事件类型 14）当成真实输入吞掉。

**这条路径的已知限制**：合成系统键在部分场景下可能被系统忽略（系统正显示音量 HUD、媒体键被别的 App 抢焦点等）。注入后**没有"按键是否生效"的回执**——验收以屏幕上的音量 HUD 是否变化为准（CoreAudio 可以读当前音量，但那是另一套 API，v3 不引入）。

---

## 7. 语音触发时序与状态机

### 7.1 时序

```
Android                                     Mac
──────                                      ───
用户按住语音键
 └─ ── voiceStart ─────────────────────────►  进入 preparing
                                            ├ 触发键是否可用（用户已配置）
                                            │   └ 未配置 → error{trigger_key_not_configured}
                                            ├ AX 聚焦（**自动路径：只探快路径 ≤300ms**）
                                            │   ├ 命中可编辑元素 → 继续
                                            │   └ 未命中 → error{focus_failed}
                                            │        （**不进入 3 秒全路径**，§5.2）
                                            │   └ preparing 总超时 0.8–1s → error{focus_failed}
                                            ├ 等待 startDelay 150ms
                                            ├ 长按模式：注入触发键 down（保持按下）
                                            │   └ 注入失败 → error{start_tap_failed}（回滚 latch）
                                            └ 点按模式：开始拍（down → 120ms → up）
                                                └ 注入失败 → error{start_tap_failed}
 （用户对着 Mac 麦克风说话，ChatterFly 收音、识别、上屏）
用户松开语音键
 └─ ── voiceStop ──────────────────────────►  注入触发键 up（长按模式）
                                            └ 点按模式：此处补一次停止拍（down → 立即 up）
                                               └ 注入失败 → error{stop_tap_failed}
                                                 （**输入法可能仍在持续收音**，必须让用户看到，§8）
 → idle
```

**快按（<300ms）怎么办**：`voiceStop` 会在 AX 聚焦期间就到达（`preparing` 还没进 `held`）。此时**必须取消**：不投递触发键；万一已经投递（极少见），立即走释放路径。**否则触发键被按下却没有对应的 up 路径 → Fn 卡死**（§7.3、§8）。

**三个时序参数的来源**（全部对齐参考实现 `remote-mic-app/Sources/RemoteMic/VoiceFnTapSessionController.swift`）：

| 参数 | 值 | 作用 |
| --- | --- | --- |
| `startDelay` | **150ms** | 目标就绪后再等 150ms 才投递触发键，给目标 App 一个"准备收音"的缓冲 |
| `tapDuration` | **120ms** | 点按模式下触发键按住的时长（必须短于输入法判定"长按"的下限，否则会被当成按住不放） |
| 停止拍 | **立即** | 点按模式的第二次触发（结束录音）用 down → 立即 up，不用 `tapDuration` |

**聚焦期间用户说的话会丢。** 顺序必须是"先聚焦、再投递触发键"——反过来的话，语音输入法会把文字插进聚焦之前那个 App 的输入框。聚焦耗时（原生 App ~50ms，Chromium/Electron 最坏 3 秒）就是这段丢字的窗口。缓解手段见 §5.2 的快路径与 §14 R4。

### 7.2 触发键的成对性

**投递模式由用户在注册表的 `__voice` 条目里选择（v3.2 起可在手机端配，§6.3），Android 端不表达。** Android 只发一次 `voiceStart` / `voiceStop`，表达"按住 / 松开"。

| 模式 | voiceStart | voiceStop |
| --- | --- | --- |
| **长按模式**（输入法认"按住 Fn 说话"） | 触发键 down，保持按下 | 触发键 up |
| **点按模式**（输入法认"按一次开始、再按一次停止"） | 触发键 down → 120ms → up（**开始拍**） | 触发键 down → 立即 up（**停止拍**） |

> 早期版本让 Android 在"按一次"时发两次 `voiceStart`/`voiceStop`，是错的：那会撞上 `voice_busy`，并额外产生一个空会话。点按模式是 Mac 端在同一个语音会话内的投递细节，与 Android 无关。

**G1-a 实测决定用哪种模式**：按下出现、松开消失 = 长按模式；按下出现、松开仍在、再按下消失 = 点按模式。

### 7.3 Mac 端状态机

```
idle ──voiceStart──► preparing ──ready──► held ──voiceStop──► releasing ──released──► idle
                        │                                                        │
                        └── timeout / cancel / disconnect ─────────────────────┘
                                              （释放触发键 + 释放本连接所有已注入键）
```

- `preparing`：确认触发键已配置 → **AX 聚焦（自动路径 = 只探快路径 ≤300ms，不走全路径）** → 等 `startDelay` 150ms → 注入触发键 down。任一失败 → `error` 并回到 `idle`。
- **`preparing` 收到 `voiceStop` = 取消**：快按（<300ms）时 `voiceStop` 会在 AX 聚焦期间到达。此时**不得继续投递触发键**——否则触发键被按下却没有对应的 up 路径，Fn 卡死（"断连兜底"救不了：那次 down 根本还没发生）。取消 = 立即回 `idle`，不发任何注入；万一已经投递（极少见），先走释放路径。**这是状态图里 `cancel` 分支的主要触发源**（其余触发源是 timeout / disconnect）。
- **`preparing` 有总超时**：快路径探测 300ms + `startDelay` 150ms + 注入余量 ≈ **0.8–1 秒**（**不含全路径的 3 秒——自动聚焦不走全路径**，§5.2）。超时 → `error{focus_failed}` 并回 `idle`（此刻尚未注入任何键，无需释放）。
- `held`：触发键保持按下，等待 `voiceStop`。此期间**不注入任何其他键**（否则会污染用户的正常输入）。
- **`held` 有最长持续时间 120 秒**：Android 进程被冻结、socket 却还在回 `ping` 的反例下，"断连兜底"不会触发，触发键会一直按下。超时 → 走释放路径（释放触发键 + 本连接所有已注入键）→ 回 `error{voice_timeout}` → 回 `idle`。**Android 端也要有对应的客户端超时**（双保险）。
- `releasing`：长按模式 → 触发键 up；点按模式 → 停止拍（down → 立即 up）。
- **点按模式两拍的失败要分开报错**：开始拍失败 → `error{start_tap_failed}`（会话终止，还没开始收音，用户重试即可）；停止拍失败 → `error{stop_tap_failed}`（**输入法的麦克风可能还在收音**，UI 必须明确提示"请在 Mac 上手动停止录音"，不能静默回 idle 让用户以为已经停了）。对齐参考实现的 `VoiceFnTapFailure`（`VoiceFnTapSessionController.swift:29-32`）。
- 每次 `voiceStart` 递增 `generation`；所有异步回调携带 generation，迟到回调直接丢弃。
- **任何离开 `held` 的路径都必须幂等释放触发键**（`releaseVoiceKeyIfNeeded`），避免修饰键卡住导致用户键盘一直带 Fn。这是参考项目反复出现过的真实故障（`remote-mic-app/Testing/VoiceKeyModes.md` 用例 7）。
- 断连立即释放触发键、结束会话，并释放该连接上所有已注入的指令键（§8）。

**状态 × 帧 处理表（v3.2 补：哪些帧在什么状态下被接受）**

| 帧 | `idle` | `preparing` | `held` | `releasing` |
| --- | --- | --- | --- | --- |
| `voiceStart` | 正常开始 → `preparing` | 回 `error{voice_busy}` | 回 `error{voice_busy}` | 回 `error{voice_busy}` |
| `voiceStop` | **忽略**（没有会话可停） | **取消** → 回 `idle`，不注入 | 正常 → `releasing` | **忽略**（已在释放中） |
| `button` | 正常执行 | **丢弃并记日志**（会话准备中，注入会污染聚焦） | **丢弃并记日志**（会话进行中，§7.3 "不注入其他键"） | 丢弃并记日志 |
| `focusComposer` | 正常执行（**走全路径**） | **丢弃并回 `focusResult{ok:false, code:"session_busy"}`**（避免与自动聚焦抢 AX） | 丢弃并回同一回执 | 丢弃并回同一回执 |
| `mappingList` / `mappingCreate` / `mappingSet` | 正常 | 正常 | 正常 | 正常 |
| `ping` | 正常 | 正常 | 正常 | 正常 |

**未登记的 `button` release 一律忽略**（v3.4 补）：`press` 落在会话里被丢弃、`release` 落在 `idle` 的情况（快按、或 press 被会话状态拒绝）——Mac 端**不能补发 up**：那个键**从来没被按下过**，凭空补一个 key-up 反而会破坏用户真实的按键状态。`KeyInjector` 侧只需一行判断：**release 只与"本连接登记在册的 press"配对**，登记里没有就丢弃。

**三类"被拒绝"的区别**（别混成一句"丢弃"）：

| 类别 | 触发条件 | 有回执吗 | 谁负责让用户看见 |
| --- | --- | --- | --- |
| **会话状态拒绝**（`button`） | 会话进行中，注入会污染会话 | **没有**——`button` 帧无回执通道 | **Android 本地禁用**：它自己知道发起了 `voiceStart`，会话期间把**指令键面板置灰**（不发帧）；Mac 端丢弃只作**兜底**（防竞态 / 旧版本），丢弃后记日志 |
| **会话状态拒绝**（`focusComposer`） | 同上 | **有**——回 `focusResult{ok:false, code:"session_busy"}` | Mac 回执，Android 自己决定怎么提示 |
| **语义错误**（未知 id、非法绑定） | 帧本身有问题 | **有**——`error{invalid_binding}`（§6.3、§6.5.5） | Mac 回执 |

**不要指望回执告诉用户"你的按键被丢了"**：`button` 没有回执通道，Android 发出去就无从知道结果——所以"会话期间禁用指令键面板"必须由 **Android 本地**实现，Mac 端的丢弃只是兜底。

---

## 8. 错误处理

| 场景 | 行为 |
| --- | --- |
| 辅助功能权限缺失 | 拒绝注入，返回明确错误提示给 UI；不静默吞掉 |
| Input Monitoring 权限缺失 | `ShortcutCapture` 无法在 Mac 上录制快捷键；**Android 端录入与注入仍全部可用**（§6.5） |
| 触发键未配置 | 拒绝开始语音会话，回 `error{trigger_key_not_configured}`，并提示用户**先选一个触发键**（可在手机端或 Mac 端配，§6.3）。**不要复用 `accessibility_denied`**——那个码的补救动作是"去系统设置授权"，两者不同 |
| **触发键注入失败**（长按模式的 down、点按模式的开始拍） | 回滚 latch，不留按下状态；回 `error{start_tap_failed}`——两种"开始投递失败"共用一个码（§6.2） |
| **绑定非法**（键规格无法解析 / 键名不在规范表内 / `app` 的 bundleId 未安装） | 回 `mappingResult{ok:false, code:"invalid_binding"}`，注册表保持不变；Android 端把编辑框回填为修改前的状态 |
| **注册表版本过期**（`mappingCreate` / `mappingSet` 的 `rev` 与服务端不符） | 回 `mappingResult{ok:false, code:"stale_mapping", rev:<最新>}`；Android 端重新拉表，并提示"配置已在其他端被修改" |
| **自定义槽位超限 / 写操作过于频繁** | 回 `mappingResult{ok:false, code:"mapping_limit"}`——**超限指"同时存在的自定义槽位 > 16"**（编号本身无上限），或 10 分钟内写操作 > 20 次（§6.5.5） |
| **未认证连接发 `mappingCreate` / `mappingSet`** | 直接拒绝并关闭连接，不进入映射逻辑 |
| AX 聚焦失败 | 回 `error{focus_failed}`，终止会话；**不降级为不聚焦直接收音** |
| **自动聚焦超时**（快路径 300ms 探测后未命中可编辑元素） | 回 `focus_failed`，会话终止；提示按**三型**分（"先点一下输入框" / "当前输入框不支持语音输入" / "这个 App 需要手动聚焦"，§5.2）。**不进入 3 秒全路径** |
| **手动聚焦超时**（idle 态的 `focusComposer` 帧：全路径 3 秒之后仍无结果） | 回 `focusResult{ok:false, code:"focus_timeout"}` |
| **`preparing` 总超时**（探测 300ms + `startDelay` 150ms + 余量 ≈ **0.8–1 秒**仍未就绪） | 回 `focus_failed`，回 `idle`；此刻尚未注入任何键，无需释放（§7.3） |
| **连接死亡（Mac 侧：6 秒无任何帧）**——**仅已认证连接** | 触发与断连相同的清理：释放触发键 + 释放本连接所有已注入键 + 清会话。**未认证 / 待审批连接不做 6 秒判死**（只受"未认证 10 秒"与"配对窗口 120 秒"约束） |
| **连接死亡（Android 侧：6 秒无任何帧）**——**仅已认证连接** | 对称检测：Mac 崩溃 / 断电时手机不会一直以为连着。Android 停止会话、UI 转"已断开"，进入退避重连 |
| **Android 断连** | 同上；Android 侧指数退避重连（1s / 2s / 4s … 上限 30s） |
| **新连接到达而旧连接仍在** | **已认证的新连接立即驱逐旧连接**：关闭旧连接并执行完整清理（释放旧连接上所有已注入键）。**不靠"6 秒无帧判死"来让位**——Android 的退避重连是 1/2/4 秒，远短于 6 秒判死窗口，新连接会先到并被拒 |
| 目标语音 App 无响应 | **不检测**。本项目不适配输入法，由 App 内"自检页"引导用户自查 |
| **认证前收到非握手帧**（`voiceStart` / `button` / `focusComposer` / `mapping*` 等） | 回 `error{auth_failed}` 并断开连接（§6.1 认证总则）。**这条是安全底线**：否则同网段任意设备连上端口就能在用户机器上敲任意键 |
| **未认证连接过多** | **已认证连接唯一**（新连接驱逐旧连接，§8 下一行）；**未认证连接最多 3 条并行**，超过则拒绝新的；单条未认证连接 **10 秒**内没完成 HMAC 握手即断开（防止半开连接堆积占位）。**例外：已发出 `pairingRequest`、正在等待审批的连接走 120 秒配对窗口**——配对要人走到 Mac 前点「同意」，10 秒的统一超时会直接吃掉配对成功率（§6.1） |
| **语音会话超时**（`held` 超过 120 秒） | 走释放路径（释放触发键 + 本连接所有已注入键）→ 回 `error{voice_timeout}`（§7.3） |
| **单帧超过 8KB** | 回 `error{protocol_violation}` 并断开连接（§6.1） |
| **点按模式停止拍失败** | 回 `error{stop_tap_failed}`，UI 明确提示"**输入法的麦克风可能还在收音，请到 Mac 上手动停止**"。**不能静默回 idle**——用户会以为已经停了，结果麦一直在收（§7.3） |
| **`app` 绑定的目标 App 已被卸载** | 按下时**静默忽略 + 记一条日志**，不弹错误、不回执（§6.5.5） |
| macOS 休眠 / 唤醒 | 重新确认辅助功能与触发键可用性；不可用时拒绝开始会话 |

**断连时必须释放"所有已注入的键"，不只是触发键**：用户按住方向键时断网，方向键的按下态会卡住（表现为 Mac 上一直往下滑）。因此 `KeyInjector` 按**连接**登记已注入的键（含触发键与指令键的 press 态），连接关闭时统一补发 up 事件。**v3 中这比 v2 更重要**——没有音频链路兜底，键卡住就是唯一的故障表现。

**释放必须以"登记"为准，不能重读注册表**：会话进行中（`held`）用户可能改了 `__voice`（把触发键从 Fn 改成 Command）。此时释放要放的是**当初实际按下的那个键**，不是配置里现在写的那个——否则要么漏释放（修饰键卡死，用户键盘一直带 Fn），要么去释放一个从未按下的键。**已注入键登记表是唯一事实源**，注册表只决定"下一次会话用什么键"。

---

## 9. 权限

### 9.1 macOS

| 权限 | 用途 | 缺失时行为 | 是否必需 |
| --- | --- | --- | --- |
| Accessibility | 注入快捷键 + AX 聚焦 | 拒绝注入，UI 明确提示 | **必需** |
| Input Monitoring | `ShortcutCapture` 的 CGEventTap 录制（**可选路径**：Android 端录入已可覆盖大部分需求，§6.5） | Mac 端录制不可用；**Android 端录入与注入仍全部可用** | **可选**（仅 Mac 端录制需要） |
| Local Network | **Bonjour 注册 / 浏览** | Android 端发现不到 Mac，需手填 IP | 见下 |
| 麦克风 | **不需要** | VoiceBridge 不采集音频 | — |

**Local Network 权限的定性**

Apple TN3179《Understanding local network privacy》明确：

> Listening for and accepting incoming TCP connections — **no**（不需要本地网络访问）
> Making an outgoing TCP connection — yes（需要）

VoiceBridge 的 Mac 端是**服务端**（监听端口、接受入站 TCP），**因此监听本身不需要 Local Network 权限**。真正需要的是：

1. Mac 端用 Bonjour **注册** `_voicebridge._tcp` 服务，供 Android 浏览 → Bonjour 操作**全部**需要本地网络访问权限；
2. 因此 Info.plist 需要 `NSLocalNetworkUsageDescription`；
3. 使用 Bonjour 服务类型时，Info.plist 需要 `NSBonjourServices`（列出 `_voicebridge._tcp`）；
4. macOS 上 multicast entitlement **不需要**（TN3179 明确 "The multicast entitlement isn't required on macOS"）；
5. **macOS 15 (Sequoia) 起**为每个用户账户分别维护本地网络隐私状态，需要在 System Settings > Privacy & Security > Local Network 中放行。**macOS 14 (Sonoma) 上没有这一层**——TN3179 的平台表明确写着 "macOS / yes / **Introduced macOS 15**"（并注明该特性由 WWDC 2024 Session 10123 引入）。

> **版本差异必须处理**：本项目的最低目标是 macOS 14，而 Local Network 隐私是 macOS 15 才有的。因此：**macOS 14 上没有弹窗、Bonjour 直接可用；macOS 15+ 上可能被拒**。被拒时的表现要分清方向：**Mac 端做的是 Bonjour「注册」**（浏览在手机端），所以失败发生在 Mac 侧——注册报 `kDNSServiceErr_PolicyDenied`（-65570，`dns_sd.h:801`），而 Android 端只能看到"**发现不到设备**"。因此 **Mac 端的设置页必须能显示"本地网络被拒"**，否则用户只会莫名其妙地找不到设备。两条路径都要能跑：15+ 被拒时用户去系统设置放行，或者改用**手填 `ip:port`**（§6.1）——手填路径不依赖 Bonjour，因此不受本地网络隐私影响。

**兜底**：用户拒绝或系统不弹窗时，Android 端支持**手动输入 `ip:port`**（§5.3 `Discovery`，只填 IP 时用默认端口 51880），此时完全不需要 Bonjour。

### 9.2 Android

| 权限 | 用途 | 缺失时行为 |
| --- | --- | --- |
| `INTERNET` | WebSocket 连接 | **必需** |
| `ACCESS_WIFI_STATE` | NSD 发现（读取 Wi-Fi 状态） | 发现不可用，需手填 IP |
| `CHANGE_WIFI_MULTICAST_STATE` | NSD 发现（组播锁） | 同上 |

**明文流量**：`ws://` 需要 `res/xml/network_security_config.xml` 放行局域网地址段（§5.3 的手填 IP 兜底要求支持**任意**内网地址，无法用 `<domain-config>` 预先枚举），因此用 **`<base-config cleartextTrafficPermitted="true">`** 全局放开明文。明文本身的风险由 §6.4 的 HMAC 承担——伪造连接拿不到令牌。

**v3 不再需要的权限**：`RECORD_AUDIO`、`FOREGROUND_SERVICE`、`FOREGROUND_SERVICE_MICROPHONE`、`POST_NOTIFICATIONS`、电池优化白名单（无麦克风、无前台服务、无常驻通知）。

**v3.1 也不新增权限**：快捷键录入（选键名 / 硬件键盘捕获）只用 Activity 的 `dispatchKeyEvent`——前台 App 自身即可收到按键，不涉及系统权限；映射的读写复用已有的 WebSocket 连接。

---

## 10. 配置与持久化

### Mac App

| 配置 | 存储 | 说明 |
| --- | --- | --- |
| **快捷键注册表**（含触发键） | UserDefaults（JSON） | 一个注册表装下：12 个固定 id + `__voice`（触发键：`key` + `mode`）+ `custom<N>` 槽位（**同时存在 ≤ 16 个**，编号不复用）→ 绑定（`combo` / `action` / `app` / `trigger` / `none`）；带单调递增的 `rev` 与 `customSeq`；**Android 端可读可写**（§6.5.5） |
| **信任设备列表 + 设备令牌** | **Keychain**（`kSecClassGenericPassword`，`kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`） | 设备名 + tokenId + secret + 首次信任时间 |

**触发键为什么并进注册表**（v3.2 修复，§0.1 P0-6）：v3.1 把触发键单独放在 UserDefaults 里，意味着**最该在手机上配的那个键，反而只能坐到 Mac 前配**——与 v3.1 "录入面在手机上" 的立论自相矛盾。现在触发键是注册表的保留条目 `__voice`，手机端改 `mappingSet` 即可，Mac 端读的仍是同一份权威副本。

**"清空注册表"不重置 `customSeq`**：清空只是删掉各条目，**`customSeq` 必须保留**（§6.5.5）——否则编号会被复用，与"删除后不复用编号"直接冲突。

**信任列表必须存 Keychain，不能存 UserDefaults**：

1. UserDefaults 是明文 plist，同机任何进程或备份都能读到；
2. 配置的"导出 / 导入"会把信任列表一起带走——**导入别人的配置等于授权对方的设备在你的 Mac 上注入键盘**。现在：**信任列表与设备令牌禁止随配置导出 / 导入**，导出格式中显式排除这两项，导入时若出现相关字段直接拒绝并提示；
3. `AfterFirstUnlockThisDeviceOnly` 表示：设备首次解锁后可用、且**不随 iCloud / 备份迁移到其他设备**——换机后需要重新配对，这是想要的行为。

**必须提供撤回信任的入口**：设置界面列出已信任设备，每项提供"移除信任"，移除即删除 Keychain 条目并让该设备下次连接重新走配对。

配置支持导出 / 导入（JSON），导入时做字段级校验，旧配置缺字段走默认值。

**注册表的导入 / 导出有三条特殊规则**（v3.2 补：否则导入会直接破坏"编号永不复用"这条规矩）：

1. **导出必含 `customSeq` 与 `rev`**。导出的 JSON 里必须带这两个字段——少了 `customSeq`，那份配置在别的 Mac 上导入时会立刻复用已删掉的编号。
2. **导入时 `customSeq` 取下限**：

   ```
   customSeq = max(文件里的值, 本机现有注册表的最大自定义编号 + 1, 文件内最大自定义编号 + 1)
   ```

   三个来源缺一不可：文件里**缺 `customSeq` 字段**，或它的值**低于下限**，一律**忽略文件里的值**，用下限。**第三项容易被漏**：文件缺 `customSeq` 但含 `custom50` 时，下限必须是 51——否则之后依次分配迟早撞上 50，把一个"已删除"的编号重新发出去。

   **编号只增不减**这条规矩在任何导入路径下都成立。
3. **导入时校验自定义槽位数量 ≤ 16**：超出直接**拒绝整份导入**（不做部分导入），并提示"该配置的自定义快捷键超过 16 个"。

**导入的语义是"整份替换"，不是合并**：导入 = 用文件里的注册表**整体替换**本机注册表（含 12 个固定 id 与 `__voice` 的绑定）；`customSeq` 是唯一的例外——它按上面的下限取值，不取文件里的原值。

导入完成后 `rev` 取本机当前值 **+ 1**（**不倒退**）。

**注册表可被手机改写**（v3.1）：`mappingSet` 来自已认证连接（§6.5.5）。这不是新的信任等级——已认证设备本来就能在 Mac 上敲任意键；但注册表是**持久化配置**，改动在重启后依然生效，因此写入必须留日志（§6.5.5 第 3 条），且用户随时能在 Mac 端改回或清空。

### Android App

| 配置 | 存储 | 说明 |
| --- | --- | --- |
| Mac 端地址 | DataStore | 自动发现的结果 + 手动覆盖 |
| **设备令牌 secret** | **Android Keystore 中的 AES-GCM 密钥加密后的密文**（密文存 DataStore） | 配对成功后写入。**不导入外部生成的 HMAC key**——各机型对"把外部密钥导入 Keystore"的支持不一致；minSdk 26 上更稳妥的做法是**在 Keystore 内生成一把不可导出的 AES-GCM 密钥**，用它加密 `secret`，密文存 DataStore。Keystore 密钥卸载应用即失效 → 需重新配对 |
| tokenId | DataStore | 公开标识，随 `hello` 上行 |

---

## 11. 平台与工具链

### 11.1 版本

| 项 | 版本 |
| --- | --- |
| macOS | 14 Sonoma+，Apple Silicon 与 Intel 均可 |
| Swift | 6.x，Swift 5 语言模式 |
| Xcode | 16+ |
| JDK | 17 |
| Android minSdk | 26 (Android 8.0) |
| Android targetSdk | 35 |
| Android compileSdk | 35 |
| Kotlin | 2.x（**必须 ≥ 2.0**） |
| AGP | **≥ 8.6.0**（compileSdk 35 的下限；AGP 8.6 要求 Gradle ≥ 8.7） |
| Gradle | Wrapper **8.7+** |
| Compose 编译器 | Kotlin 2.0 起 Compose 编译器已并入 Kotlin，**必须显式应用 `org.jetbrains.kotlin.plugin.compose` 插件**（不再有独立的 `composeOptions.kotlinCompilerExtensionVersion`）；漏掉会直接编译失败 |
| 序列化 | 协议帧**手写编解码**（帧类型约 20 种，不引入序列化框架）；**codegen 只产出常量 / 键表 / 错误码**（§12.3），帧的编解码本身手写；Android 端如使用 kotlinx.serialization，仅限本地配置存储层 |
| 端口 | **默认 51880，可改**（§6.1） |

> C++ / CMake / libASPL 相关行已随虚拟麦克风方案一并移除（§0.4）。

### 11.2 macOS 工程形态

- `mac-app/` 是 **Xcode 工程**，产出 `VoiceBridge.app`；核心逻辑放在可被单测引用的 Swift Package target 中（§4.1）。
- Info.plist 必须包含：`NSLocalNetworkUsageDescription`、`NSBonjourServices`（`_voicebridge._tcp`）。
- **不启用 App Sandbox**（见 §11.3）。

**App 形态：菜单栏常驻（`LSUIElement = true`）+ 首次启动的一次性设置窗口。**

| 决定 | 理由 |
| --- | --- |
| **菜单栏常驻**（`LSUIElement = true`，不占 Dock） | 遥控器的服务端必须**一直在跑**——用户没开 Mac App 时手机按不动；常驻也免掉"忘了打开"这个失败模式 |
| 菜单栏图标显示状态 | 已连接 / 配对待批准 / 权限缺失（辅助功能、Input Monitoring）/ 触发键未配置——这几种状态用户都得看得见 |
| **首次启动弹一次设置窗口** | 做引导：申请辅助功能权限、选触发键、显示配对码。设完即关，之后只留菜单栏 |

**`action: focus_composer` 不回 `focusResult`**：`focusComposer` 是**同步帧**——等结果再回 `focusResult`（验收清单第 7 条依赖它）；而 `action` 是**无状态即触发**（§6.3），`focus_composer` 只是"顺手聚焦一下"，失败**只记日志**、不回执、不打断用户。需要"聚焦并知道结果"时用独立的 `focusComposer` 帧。

### 11.3 分发形态

**不上架 Mac App Store。**

原因：Mac App Store 强制 App Sandbox，与本项目的核心机制冲突——**CGEvent 注入**（`CGEvent.post`）在 Sandbox 下受限，无法全局注入键盘事件。（虚拟麦克风方案下还有"HAL 驱动要写 `/Library/Audio/Plug-Ins/HAL`"这一条冲突，v3 已无驱动。）

**形态：Developer ID 直分发，.dmg。**

| 组件 | 分发形式 | 签名 |
| --- | --- | --- |
| `VoiceBridge.app` | **.dmg**（用户拖到 `/Applications`） | Developer ID Application 签名 + 公证 + staple（票据打在 .app 上） |

`.app` 作为被 Gatekeeper 检查的分发单元，**必须签名 + 公证**，否则在别人机器上直接被拦（"无法验证开发者"）。v3 没有 `.pkg`，因此不再有驱动安装、coreaudiod 重启、universal binary 这些链路。

**若只自用**：Developer ID 签名即可（公证与 staple 仅在对外分发、避免"无法验证开发者"提示时才必需）。

---

## 12. 测试策略

### 12.1 Mac App（Swift Testing）

- `ProtocolCodec`：控制帧往返、帧结构违规（缺字段、未知类型、超长）、**单帧 > 8KB → `protocol_violation`**、**认证前收到非握手帧 → `auth_failed`**
- `ShortcutRegistry`：默认映射、导入导出往返、旧配置缺字段走默认、**导出结果不含信任信息**、**导出必含 `customSeq` 与 `rev`**、**导入时 `customSeq` 取下限（`max(文件值, 本机最大编号+1, 文件内最大编号+1)`，缺字段或低于下限时忽略文件值）→ 编号永不倒退**、**导入超 16 个自定义槽位 → 拒绝整份导入**、**导入 = 整份替换（不合并）**、**导入后 `rev` 不倒退**、**自定义槽位增删**、**`mappingCreate` 分配编号并回填 id**、**同时存在的槽位数 ≤ 16（建到第 17 个 → `mapping_limit`，删一个后可再建）**、**`binding:null` 删除槽位而 `none` 只禁用**、**`binding:null` 打到固定 id / `__voice` → `invalid_binding`**、**`__voice` 不可删除**、**`trigger` 只对 `__voice` 合法（写到 `custom1` → `invalid_binding`）**、**`none` 对 `__voice` 非法**、**`key:null` 表示未配置**、**初值 `rev = 1` / `customSeq = 1`**、**`customSeq` 只增不减（删除后编号成为空洞）**、**"清空注册表"不重置 `customSeq`**、**`rev` 单调递增**、**`mappingCreate` / `mappingSet` 的 `rev` 冲突检测**、**未知 id 分流（格式非法 → `protocol_violation` 断开；格式合法但未创建 → `invalid_binding` 不断开）**
- `KeySpecParser`：规范名 → (修饰键 flags, 主键 keycode) **全表逐条核对**、未知键名 → `invalid_binding`、**修饰键顺序归一**（`shift+cmd+a` 与 `cmd+shift+a` 等价）、**主键恰好 1 个（无主键 → `invalid_binding`）**、**修饰键重复 → 拒绝**、**5 个修饰键合法**、**主键用 ANSI 物理键位**
- `SystemActionRunner`：`action` / `app` / `none` 三类绑定的执行、**`phase` 只认 `press`（`release` 不触发——否则一次点按执行两次）**、未安装的 bundleId → `invalid_binding`、**`NSSystemDefined` 事件的 `data1` 编码**（`(NX_KEYTYPE << 16) | (state << 8)`，down=0xA / up=0xB）、**写入合成事件标记**
- `KeyInjector`：down/up 配对、合成事件标记、注入失败回滚、**按连接登记与统一释放**
- `ShortcutCapture`：组合键 / 独立修饰键 / 忽略自身合成事件
- `ComposerFocus`：角色白名单、**role 与 subrole 双重判定**、敏感关键词拒绝、遍历上限、**AX 前置宣告顺序（Manual 优先、Enhanced 回退）——但只有全路径才宣告，快路径刻意不宣告**、**快路径命中与超时降级**、**`AXUIElementSetMessagingTimeout(0.3)` 让探测真的只花 300ms，且探测后恢复默认超时**、**自动路径只探快路径，全路径只属手动 `focusComposer` 帧**、**快速失败的两种提示分类**、宣告后重试
- `VoiceSession`：状态机全部迁移、generation 隔离迟到回调、断连释放触发键、幂等释放、**长按模式 down/up 配对**、**点按模式开始拍与停止拍的时序**、**开始投递失败 → `start_tap_failed`（长按 / 点按两种）**、**停止拍失败 → `stop_tap_failed` 且不静默回 idle**、**`preparing` 收到 `voiceStop` = 取消（不注入触发键，不留 Fn）**、**`preparing` 总超时 0.8–1s → `focus_failed`**（自动路径只探快路径，§5.2）、**`held` 超过 120s → 释放并回 `voice_timeout`**、**状态 × 帧表的逐格行为**（`idle` 收 `voiceStop` 忽略 / `held` 收 `button` 丢弃并记日志 / `releasing` 收 `voiceStop` 忽略）、**`held` 期间改 `__voice` 后释放的是登记里的实际键（不是新配置）**
- `PairingService`：**HMAC 字节编码固定向量**（§6.4 的拼接表，tokenId 用 ASCII、nonce 用原始 32B）、HMAC 计算与比对一致 / 不一致、**`serverNonce` 每连接唯一**、**`tokenId` 大小写归一后仍能校验通过**、tokenId 未知 → `helloAck{paired:false}`（不是 `auth_failed`）、**配对成功即把当前连接转为已认证（无需重连）并驱逐既有已认证连接**、**配对码由 Mac 生成并经 `pairingCode` 推给手机（手机不提交码）**、**配对弹窗限流（同一来源 IP 10 分钟 3 次，指纹不参与限流）**、**配对审批窗口 120 秒（不套用"未认证 10 秒"）**、**导出配置不含令牌**
- `WebSocket` 层：帧路由、`ping` / `pong` 周期、**6 秒无帧判死并触发清理（仅已认证连接）**、**新连接驱逐旧连接并释放其已注入键**、**已认证连接唯一**、**未认证连接最多 3 条且 10 秒未握手即断开（配对等待中的连接走 120 秒窗口）**、**认证前收到非握手帧 → `auth_failed` 并断开**、**`pong` 在认证前也允许回**

### 12.2 Android

**纯 JVM 单测（JUnit）**：控制帧打包与解析、WS 断线重连退避、语音键按压状态机、路由/地址配置的持久化、**键规格字符串的生成与归一化**（修饰键顺序、主键缺失拒绝）、**新建槽位走 `mappingCreate` 并用回执里的 `id`**（Android 不自行分配编号）、**触发键（`__voice`）配置的读写与"未配置"状态**、**收到 `helloAck{paired:false}` → 清掉本地 secret 与 tokenId → 转入配对流程**、**`focusResult` 的 `code` 分派（未知 code 按通用失败处理）**

**仪器化测试 / Robolectric**：NSD 发现与解析依赖系统 Bonjour 栈，JVM 测不出来——需 Robolectric 或 androidTest

**Compose UI 测试**：语音键按压状态机、指令键面板、状态灯、**快捷键录入界面**（选键名 + 修饰键 → 生成规范键规格）、**触发键选择界面**（选键 + 选投递模式）、**会话期间指令键面板置灰**（§7.3 的 P1-4：Android 本地禁用，不靠回执）、**`voice_timeout` / `session_busy` 的 UI 处理**（提示文案 + 状态复位）

### 12.3 跨端

- **协议常量 codegen**：`protocol/` 放一份机读描述（JSON / YAML），由脚本生成 Swift / Kotlin 两份**常量与错误码**。**CI 执行生成器并检查 `git diff` 为空**——这就是"CI 校验一致"的机制。手抄两份必然漂移。
- **协议一致性 golden test**：用同一组黄金 JSON 喂两端解码器，断言输出一致。
- **HMAC golden 向量**：固定一组 secret / nonce / tokenId，两端算出的 mac 必须逐字节相同（§6.4 的字节编码表就是为这条测试准备的）。
- **键名清单一致性**：`protocol/keyspec.json`（**含 `triggerKey` 枚举**）→ `KeySpecCatalog.swift` / `KeySpecCatalog.kt`，CI 校验生成后 `git diff` 为空——两端认识的键名集合与触发键枚举因此永远相同（§6.5.3、§6.3）。
- 端到端：手动验收清单（见 §13.3）。

---

## 13. 里程碑与验证门

### 13.0 实施分解

| 计划批次 | 覆盖里程碑 | 说明 |
| --- | --- | --- |
| 计划 1 | M0 + M1 | 验证门 + 协议骨架与配对打通。**这是第一批实施内容** |
| 计划 2 | M2 | 触发键注入 + AX 聚焦 + 端到端 |
| 计划 3 | M3 | 分发。**自检页已归 M2** |

### 13.1 前置验证门（**必须在 M2 之前完成**）

| 门 | 内容 | 方法 | 失败后果 |
| --- | --- | --- | --- |
| **G1-a** | 合成触发键能否唤醒目标语音输入法，以及它认长按还是点按 | **针对 ChatterFly 重写探针**：参考仓库的 `../remote-mic-app/Testing/VoiceKeyFnPanelProbe.swift` 硬编码了**豆包**的 bundleId（`contains("doubao")`）与它的语音面板窗口特征（主进程 `layer == 3` 的 124×32 小窗）。改测 ChatterFly **不是"改 2 行"**：bundleId matcher 与窗口识别逻辑都要重写，且 ChatterFly 的语音面板是否同样是独立窗口尚不确定 | 触发键方案需改为 Command / Option；投递模式需相应调整 |
| **G1-b** | **AX 聚焦耗时**（快路径是否有效）——**测的是目标 App 的输入框，不是 ChatterFly** | 在两类 App 上各测一次：①**原生 App**（TextEdit，新建一个文本文档）；②**Chromium / Electron 系**（Chrome 或 Slack）。量三件事：快路径（查 `AXFocusedUIElement`）耗时、快路径是否命中可编辑元素、全路径耗时 | 快路径无效 → 自动聚焦降级为默认关闭的独立动作（§5.2） |

**G1-a 判读规则**：按下出现、松开消失 = **长按模式**；按下出现、松开仍在、再按下消失 = **点按模式**。

**G1-b 的测量对象为什么要改**（v3.2 修正）：聚焦的目标是 **frontmost App 的输入框**（TextEdit / Chrome / Slack…），**ChatterFly 是后台输入法，根本不参与聚焦**。v3.1 把 AX 聚焦耗时测在 ChatterFly 上，是测错了对象，测出来的数跟真实聚焦耗时毫无关系。

G1-b 只需要回答两个问题：

1. **原生 App 上快路径值不值得开自动聚焦**（期望：≤300ms 命中可编辑元素）；
2. **Chromium / Electron 系上快路径命不命中**（决定"自动聚焦快速失败"这条规则是否必要，§5.2）。

G1-a / G1-b 在没有自研代码时即可执行（纯 CGEvent 注入 + 窗口观察 + AX 查询），应作为独立的第一步。

### 13.2 里程碑

| 里程碑 | 内容 | 出口标准 |
| --- | --- | --- |
| **M0** | 执行 G1-a / G1-b 前置验证 | 结论写入 `docs/`：目标输入法的收音快捷键、长按/点按模式（G1-a）；**原生 App 与 Chromium/Electron App 上的 AX 聚焦耗时**（G1-b） |
| **M1** | 协议 + 骨架：Mac WS 服务 + Android 客户端，能配对、能发指令 | 双端握手成功（含 HMAC 校验）；手机点指令，Mac 收到；**`mappingList` / `mappingCreate` / `mappingSet` 往返打通**（新建槽位能拿到 Mac 分配的 id） |
| **M2** | 触发键注入 + AX 聚焦 + 映射配置 + 端到端 | 真机上按住手机语音键 → ChatterFly 收音 → **文字上屏**；松开即停止；指令键生效；**在 Android 端新建自定义槽位并绑 `cmd+shift+4`，Mac 端按该槽位即生效**；Mac 自检页能"试按一次" |
| **M3** | 分发 | 干净机器上装完即用（.dmg）：安装、启动、发现、配对、首条指令生效。**自检页已在 M2 验完，M3 只验分发** |

### 13.3 端到端验收清单

1. 安装 Android App → 局域网发现 Mac → 配对（两端配对码一致）→ 信任
2. 在 ChatterFly 中把麦克风设为 Mac 自带麦克风（默认即可）
3. 聚焦一个文本框 → 按住手机语音键说"测试一下" → 松开 → 文字上屏
4. 快速连说三句短语音 → 三句均完整上屏
5. 说话中途退出 Mac App → 触发键被释放（键盘无残留 Fn 修饰位）
6. 权限被撤销（Accessibility / Input Monitoring）→ App 明确提示，不静默失败
7. 手机端点击"聚焦" → Mac 焦点切到输入框，Android 端收到 `focusResult{ok:true}`
8. 在手机端按住方向键的同时断网 → 方向键的按下态被释放，不卡键
9. 拔掉网线（TCP 半开，**不触发 FIN**，socket 不会立刻报错）→ 6 秒内 Mac 判定连接死亡并释放触发键；Mac 崩溃时手机侧同样在 6 秒内停止会话并显示"已断开"
10. 手机断网后恢复 → 重连成功时**旧连接被驱逐**，且旧连接上按住的键全部释放
11. 目标输入法为点按模式时：**按住说话 → 松开触发停止拍**，文字同样上屏（**Android 手势不变**，§7.2）
12. 在手机端**新建**一个自定义槽位（`mappingCreate`，编号由 Mac 分配）→ 绑 `cmd+shift+4` → Mac 端按该槽位 → 目标 App 收到该组合键
13. 手机与 Mac 同时改同一个映射 → 后提交的一方收到"配置已在其他端被修改"提示，**不静默覆盖**
14. 手机端把某个槽位改成"禁用"（`none`）→ 按下无任何效果，且 Mac 端注册表里确实是禁用状态
15. 手机端**删除**一个自定义槽位（`binding:null`）→ Mac 端注册表里该槽位消失；再新建槽位时拿到的编号**比之前大**（`customSeq` 不回退）
16. **建满 16 个槽位后再删光、再建** → 仍能建出新的槽位（回归测试：v3.1 的"N ∈ 1..16 + 只增不减"会把功能锁死）
17. **手机端改触发键**（选一个修饰键 + 选投递模式）→ Mac 端语音触发随之改变，无需坐到 Mac 前
18. 触发键从未配置时按语音键 → Mac 回 `trigger_key_not_configured`，手机端提示"先选触发键"（**不是**"请授予辅助功能权限"）
19. **快按**（按下 <300ms 就松开）→ Mac **不注入触发键**，键盘上**没有残留 Fn**
20. 按住语音键不放、让手机进程被冻结 / 断网，但 socket 仍在回 ping → **120 秒后** Mac 自动释放触发键并回 `voice_timeout`
21. 用一台**未配对**的设备连上 Mac 的端口后直接发 `button` → Mac 回 `auth_failed` 并断开，**不执行任何注入**
22. 在 Mac 上移除对某台手机的信任后，该手机重连 → 收到 `helloAck{paired:false}` → 自动转入配对流程
23. **手动聚焦超时**（在 Chromium 系 App 上点"聚焦键"）→ Android 收到 `focusResult{ok:false, code:"focus_timeout"}`，提示"聚焦超时"
24. **语音会话进行中点击指令键** → 手机端**指令键面板是灰的**（本地禁用，不发帧）；Mac 端即使收到也丢弃（§7.3）
25. 在 Mac 上点配对弹窗的**「拒绝」** → 手机收到 `pairingResult{approved:false}`，**连接保持**、可以再发起一次配对（限流照旧，§6.4）
26. 焦点在**密码框**上按语音键 → 提示"当前输入框不支持语音输入"，**不是**"先点一下输入框"（§5.2 的三型分型）

---

## 14. 风险登记册

| # | 风险 | 等级 | 缓解 |
| --- | --- | --- | --- |
| R1 | **目标语音输入法不响应合成的触发键** | **高** | **G1-a** 前置验证；备选 Command / Option 触发键（用户可在手机端或 Mac 端配置）。v3 中这是**头号风险**——整个产品的价值都压在"合成 Fn 能唤醒 ChatterFly"这一件事上 |
| R2 | 合成 Fn 触发 macOS 系统级 Fn 快捷键（调亮度 / 播放键等） | 中 | 通用引导中提示用户在系统设置关闭对应的 Fn 功能；或改用 Command / Option 触发键 |
| R3 | **明文信道下的残余嗅探风险** | 中 | 令牌 + HMAC 已挡住"自报 fingerprint 冒充"与"事后嗅探者"；但**配对握手期间的被动嗅探者**可捕获一次 secret，主动中间人也可劫持会话。升级路径：v1.1 上 `wss://` + 自签证书 pin（§6.4 威胁模型表） |
| R4 | **聚焦耗时导致丢字** | 中 | 聚焦在触发键投递之前，耗时窗口内用户说的话不会被 ChatterFly 录到。缓解：**自动聚焦只探快路径**（≤300ms，§5.2）——未命中就快速失败，不等 3 秒；**G1-b 实测标定**（原生 App 与 Chromium/Electron 系各一次）。若 G1-b 显示快路径在 Chromium/Electron 系上也无效，**应把自动聚焦整体降级为默认关闭的独立动作** |
| R5 | 触发键修饰位卡住 | 中 | 所有离开 `held` 的路径幂等释放；断连 / 退出 / 权限撤销均触发清理；`KeyInjector` 按连接登记全部已注入键 |
| R6 | 用户在别的 App 录音 / 系统音频被占用 | 低 | 不适用——VoiceBridge 不采集音频，Mac 麦克风由 ChatterFly 独占，与本项目无关 |
| R7 | 配对被同网段设备骚扰 | 低 | 配对弹窗**仅按来源 IP 限流**（指纹是自报可伪造的，不参与限流，§6.4） |
| R8 | macOS 休眠后辅助功能权限失效 | 低 | 会话开始前复核辅助功能状态；不可用则拒绝开始 |
| R9 | 分发被 Gatekeeper 拦 | 低 | Developer ID 签名 + 公证 + staple（§11.3） |
| R10 | **Android 端录入的键规格与用户心里想的键不一致** | 低 | 用 ANSI 物理键位编码，键规格与键盘布局无关（§6.5.2）；Mac 端提供"试按一次"，绑错当场就能发现；键名表由 codegen 保证两端一致（§6.5.3） |
| R11 | **注册表被手机改写后用户不知情**（绑定是持久化配置，重启后仍生效） | 低 | 写入留日志（§6.5.5）；Mac 端随时可改回 / 清空；`rev` 冲突检测避免静默覆盖；绑定即时生效，误绑当场可察觉 |
| R12 | Android 侧**捕获不到 `fn`**（硬件键盘的 Fn 在固件 / HAL 层被消费），用户无法用手机录入带 fn 的组合 | 低 | `fn` 在规范键名表里，可**手选**；硬件键盘捕获路径对 fn 的不可靠已在 UI 提示（§5.3、§6.5.4） |
| R13 | **合成系统键（音量 / 播放控制）被系统忽略** | 低 | 这些键没有 virtual keycode，只能发 `NSSystemDefined` 事件（§6.5.6），在系统正显示音量 HUD、媒体键被别的 App 抢焦点时可能无效；注入后没有回执，验收以屏幕上的音量 HUD 是否变化为准 |
| R14 | **未认证连接堆积**（半开连接占位，挤掉正常连接） | 低 | 已认证连接唯一；**未认证连接最多 3 条**，单条 10 秒未握手即断开（**已发 `pairingRequest` 等审批的走 120 秒配对窗口**，§8） |

---

## 15. 参考资料

| 主题 | 位置 |
| --- | --- |
| 参考实现（SayAll，GPL-3.0，仅作行为参考） | `../remote-mic-app` |
| 合成 Fn 探针（**当前硬编码豆包**，改测 ChatterFly 需重写窗口识别） | `../remote-mic-app/Testing/VoiceKeyFnPanelProbe.swift` |
| 手机语音键无响应的历史故障 | `../remote-mic-app/Bugs/2026-08-03-ios-phone-voice-button-no-response.md` |
| 语音键模式测试手册 | `../remote-mic-app/Testing/VoiceKeyModes.md` |
| CGEvent 注入实现（`eventSourceUserData` 合成标记在 `postKey`，公开入口 `postKeyState`） | `../remote-mic-app/Sources/RemoteMic/KeyboardInjector.swift:1878`（`postKeyState` 在 1891） |
| 快捷键录制实现 | `../remote-mic-app/Sources/RemoteMic/ShortcutCaptureMonitor.swift` |
| AX 聚焦实现 | `../remote-mic-app/Sources/RemoteMic/KeyboardInjector.swift:662` |
| AX 前置宣告与重试常量 | `../remote-mic-app/Sources/RemoteMic/KeyboardInjector.swift:175-187`（`AXManualAccessibility` 182 / `AXEnhancedUserInterface` 183 / 最大尝试 12 次 186 / 重试间隔 250ms 187） |
| AX 安全边界 | `../remote-mic-app/Sources/RemoteMic/VoiceInputDestinationCoordinator.swift:80` |
| 触发键集合（排除 Shift / Control） | `../remote-mic-app/Sources/RemoteMic/VoiceKeyMode.swift:60` |
| 指令键语义枚举（wire 值 = `rawValue`） | `../remote-mic-app/Sources/PublicRemoteCompatibility/Core/RemoteCompatibility.swift:3` |
| 配对批准 | `../remote-mic-app/Sources/RemoteMic/BridgeAppModel.swift:6277` |
| **触发键时序参数**（`startDelay 0.15` / `tapDuration 0.12` / 停止拍立即 / 目标就绪等待） | `../remote-mic-app/Sources/RemoteMic/VoiceFnTapSessionController.swift` |
| **点按模式两拍的失败码**（`start_tap_failed` / `stop_tap_failed`） | `../remote-mic-app/Sources/RemoteMic/VoiceFnTapSessionController.swift:29-32`（`VoiceFnTapFailure` 枚举） |
| **系统键事件注入**（`NSSystemDefined` / subtype 8 / `data1 = (NX_KEYTYPE << 16) \| (state << 8)`） | `../remote-mic-app/Sources/RemoteMic/KeyboardInjector.swift:2065-2087`（`postSystemKey`），NX_KEYTYPE 取值见同文件 `:389-400`（音量 0 / 1、静音 7、播放 16、下一首 17、上一首 18），注入点别名 `systemKeyPoster` 在 `:291` |
| **合成事件标记与系统键抑制** | `../remote-mic-app/Sources/RemoteMic/KeyboardEventSuppressor.swift:21`（监听 `systemDefined` 事件类型 14，注入必须带 `eventSourceUserData` 标记） |
| 环回设备无回放缓冲的记录（虚拟麦克风方案的留档依据） | `../remote-mic-app/Sources/RemoteMic/BridgeAppModel.swift:8865` |
| **本地网络隐私（TN3179）** | https://developer.apple.com/documentation/technotes/tn3179-understanding-local-network-privacy —— 监听入站 TCP **不需要**权限；Bonjour 注册 / 浏览**需要**；**macOS 15 起才有这一层**（macOS 14 无） |
| **Android 明文流量** | https://developer.android.com/privacy-and-security/risks/cleartext-communications —— API 28 起 OkHttp 等默认强制 HTTPS |
| **Apple Developer Program 会员权益** | https://developer.apple.com/support/compare-memberships/ —— 99 USD/年，含 Developer ID 与公证 |
| **coreaudiod sandbox profile**（虚拟麦克风方案的留档依据） | `/System/Library/Sandbox/Profiles/com.apple.audio.coreaudiod.sb` —— `file-write*` 不含 `/private/tmp` |
| **macOS 虚拟键码常量 `kVK_*`**（本机 SDK 已核实） | `$(xcrun --show-sdk-path)/System/Library/Frameworks/Carbon.framework/Frameworks/HIToolbox.framework/Headers/Events.h` —— 上下文菜单键是 `kVK_ContextualMenu`（0x6E），**没有** `kVK_Menu`；F 键码不连续（§6.5.2、§6.5.3） |
| **Android `KeyEvent`**（keyCode / metaState） | https://developer.android.com/reference/android/view/KeyEvent —— `KEYCODE_DEL` = 退格、`KEYCODE_FORWARD_DEL` = 前向删除；**`KEYCODE_FUNCTION` = 119、`META_FUNCTION_ON` = 0x8（已核实 android-37 SDK）——Fn 在 Android 的按键模型里是存在的**，真正的限制是多数键盘在固件 / HAL 层就消费了它、且不区分左右（§5.3） |
