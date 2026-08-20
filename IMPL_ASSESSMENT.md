# TinyCal 只读技术实施评估：4 项产品口径 + SQLite 影响

| 项 | 值 |
|---|---|
| 文档类型 | 只读技术实施评估（涉及模块/调用链/改动点/兼容性风险/验证方式） |
| 状态 | 草稿（待发布，未关联 PingCode） |
| 仓库 | `TinyCal`，分支 `refactor/code-quality-polish`，HEAD `769bc4970943c903a295166c739be3eae4826ba0` |
| 评估人 | 测试总监（只读，不修改生产代码、不运行构建） |
| 评审日期 | 2026-08-20 |
| 基线确认 | HEAD `769bc49` 未变（已 `git rev-parse` 核对）；`Info.plist` 仍 `NSAllowsArbitraryLoads=true`；`HolidayViewModel.swift:20` 仍 `http://timor.tech`；`TinyCalApp.onTapBar:103` 仍每次重建根视图。**基线无变化，可继续评估。** |

> 说明：本评估为只读性质，列出若工程师实施 4 项口径需触及的模块、调用链、改动点与风险，供规划师/工程师决策排期；**测试总监不实施这些生产改动**，仅据此设计验收用例（见 [TEST_PLAN_DRAFT.md v2](TEST_PLAN_DRAFT.md)）。

---

## C1：缓存自然年失效 + 离线复用

### 涉及模块与调用链
- `HolidayViewModel.populateStocks(year:)`（`View Models/HolidayViewModel.swift`）
  - 调用链：`AppDelegate.applicationDidFinishLaunching`（`TinyCalApp.swift:97-99`）启动时按当年加载；`CalendarView.nextMonth/preMonth`（`CalendarView.swift:46-48,63-65`）跨年翻页时按相邻年加载；`CalendarView.isHoliday/isHolidayWork`（`CalendarView.swift:82-101`）渲染时查表。
  - 当前流程：① 内存 `stocks[year]` 命中即 return；② 否则 `File.readField` 命中即解码入内存 return；③ 否则网络拉取并 `File.createFile` 写缓存。
- 缓存存储：`File`（`Utils/File.swift`），沙盒 Documents 下以年份字符串为文件名。

### 改动点（交工程师）
1. **自然年失效语义**：当前缓存一旦写入**永久命中、永不刷新**（违反 C1"每年首次请求强制刷新"）。需新增失效判定，候选实现：缓存文件内/旁路记录"自然年"或"写入年份"，当 `Date().getYear() > 缓存所属年` 或跨自然年首请求时强制走网络分支；或引入 TTL=至当年末。具体"年内首次 vs 会话首次"需工程口径（影响 TC-20）。
2. **离线/失败复用语义**：当前 cache-first 使"有缓存则不发网络"，间接满足"失败时复用缓存"，但**缺少"先尝试刷新、失败再回退缓存"的语义**。C1 期望"每年首次强制刷新 + 失败回退缓存"意味着：首请求应先网络、失败回退；非首请求可继续用缓存。需调整 `populateStocks` 分支顺序：判断是否"需刷新"→ 尝试网络 → 失败回退 `readField` → 都失败则内存为空。
3. **坏缓存不永久阻塞（G9）**：`readField` 解码失败当前仅 `print` 不删文件，下次仍命中坏缓存永不回源。需在解码失败时删除坏缓存文件或绕过，使其可回源（C1 隐含要求）。

### 兼容性风险
- 中。改 `populateStocks` 分支顺序影响所有节假日加载路径，需保证：刷新失败不清空已有内存缓存（C1"不清空"）；并发首请求去重（G11，避免刷新风暴触发 timor 限流）。
- 自然年判定依赖 `Date().getYear()`，本身用 `Calendar.current`（系统时区，与 C4 一致，无冲突）。

### 验证方式（测试侧）
- TC-19（自然年失效强制刷新）、TC-21（失败复用缓存）、TC-22（无缓存离线显纯公历）、TC-18（坏缓存回源）。mock `URLProtocol` + `File` 协议注入。修复前为失败用例（暴露缺陷），修复后通过。

---

## C2：HTTPS 迁移 + ATS 收紧

### 涉及模块与调用链
- `Webservice.getStocks(url:)`（`Services/Webservice.swift`）：`URLSession.shared.data(from:)` + 200 校验 + `JSONDecoder` 解码。
- URL 构造点：`HolidayViewModel.populateStocks`（`HolidayViewModel.swift:20`）`URL(string: "http://timor.tech/...")`。
- ATS 配置：`Info.plist`（`NSAllowsArbitraryLoads=true`，全局放行明文）。
- 唯一数据源：timor.tech（公历法定节假日 + 调休）。

### 改动点（交工程师）
1. URL scheme `http://` → `https://`（`HolidayViewModel.swift:20`）。需先确认 timor.tech 是否提供 HTTPS 端点（只读评估无法联网验证；工程师落地前需 `curl https://timor.tech/api/holiday/year/2026` 确认可达且证书有效）。
2. `Info.plist` 移除 `NSAllowsArbitraryLoads` 或改为 `NSExceptionDomains` 仅对必要域放行（若 timor 证书有问题才需例外）。C2 期望"不再全局放行"。
3. 失败态（与 G12/C1 一致）：HTTPS 证书异常/超时需可观察失败态，非静默空白。

### 兼容性风险
- 低-中。若 timor.tech 无有效 HTTPS 证书，迁移后节假日功能直接失效——**这是落地前必须确认的前置项**（只读评估无法确认，需工程师联网验证）。
- 移除全局 ATS 后，若有其他明文请求会一并被阻断；当前仅 timor 一处网络调用，影响面可控。
- 旧缓存文件内容不依赖 scheme，无需迁移。

### 验证方式（测试侧）
- TC-31（URL 为 https）、TC-32（无 http 回退）、TC-33（ATS 收紧，静态查 `Info.plist`）、TC-34（证书异常失败态）。mock `URLProtocol` 注入 HTTPS 失败。

---

## C3：菜单栏重置当前月

### 涉及模块与调用链
- `AppDelegate.onTapBar`（`TinyCalApp.swift:102-105`）：每次点击重建 `popover.contentViewController = NSHostingController(rootView: ContentView(vm:))`。
- `ContentView`（`ContentView.swift`）→ `CalendarView`（`CalendarView.swift:12-14`）：`@State year = Date().getYear()`、`month = Date().getMonth()`、`allDays = monthDates(...)` 初始化即取当前年月。
- 首启：`applicationDidFinishLaunching`（`TinyCalApp.swift:77`）同样构造初始 ContentView。

### 改动点（交工程师）
- **无需改动**。当前代码每次 `onTapBar` 重建根视图，`@State` 随之重置为今天，**已满足 C3 期望态**。
- 注意：`HolidayViewModel`（`stockListVM`）是单例注入、跨次打开保留（缓存不丢），仅 UI 状态重置——符合 C3"重置月份"但不丢节假日缓存，与 C1 不冲突。

### 兼容性风险
- 无。属"现状即期望"。

### 验证方式（测试侧）
- TC-35（重开重置当前月）、TC-36（跨天后重开）。E2E 冒烟 + 可注入固定时间的单元断言（待可测性改造）。

---

## C4：今天按系统时区 / 节假日与农历按固定公历日

### 涉及模块与调用链
- "今天"判定：`Day.isToday`（`Models.swift:31-33`）`toDate(format:"yyyy-MM-dd") == Date().toDate(...)`；`Date.isToday`/`isCurrentMouth`（`Date+Extensions.swift:37-45`）；`CalendarView.reflush`（`CalendarView.swift:26-28`）`today.getYear()/getMonth()/getDay()`。均经 `Calendar.current`/`DateFormatter`（默认系统时区）→ **"今天"已跟随系统时区**。
- 节假日查表：`CalendarView.isHoliday/isHolidayWork`（`CalendarView.swift:82-101`）以 `d.toDate(format:"yyyy")` / `"MM-dd"` 为键查 `vm.stocks`。键是公历日期字符串 → **节假日按固定公历日，不随时区换算**（满足 C4）。
- 农历：`Date.getChineseDay`（`Date+Extensions.swift:93-122`）用 `Calendar(identifier:.chinese)` + `DateFormatter`，**默认时区为系统时区**——农历由 Date 瞬时计算，跨时区边界处可能与"按固定公历日"产生偏差（C4 期望农历不做时区换算）。

### 改动点（交工程师，视验证结果）
- "今天"与节假日：**无需改动**，当前已满足 C4。
- 农历（待验证）：若 TC-39 验证边界处偏差，需将 `getChineseDay` 的农历日历时区固定（如 `lunarCalendar.timeZone = TimeZone(identifier:"Asia/Shanghai")` 或以公历日为基准换算），以符合 C4"农历以公历日期为准"。

### 兼容性风险
- 低。"今天"随系统时区是既有行为，固定后无回归。农历若改时区固定，需确认不破坏现有农历显示。

### 验证方式（测试侧）
- TC-37（今天随系统时区，当前已满足）、TC-38（节假日固定公历日，当前已满足）、TC-39（农历边界，待验证）、TC-40（跨年边界"今天"不漂移）。

---

## SQLite / `db.swift` 清理与否对方案的影响

### 事实确认
- `db.swift`（`Services/db.swift`）`import FileKit`/`import SQLite`，定义 `InitDB()`/`TODOList`；`project.pbxproj` 引用计数=0 → **未入编译 Sources，运行时不执行**。
- `FileKit.xcframework`/`SQLite.xcframework` 在 `project.pbxproj` 作 PBXFileReference 与 group 成员存在，但 `db.swift` 不编译，二者实际未被任何 Sources/Libraries 阶段引用（待工程师最终核实链接阶段）。

### 对测试方案的影响
- **不阻塞**。`db.swift` 死代码不影响运行时行为，无功能测试责任，不进用例集。
- 仅追加 TC-51：**若决策清理**，清理后做一次构建/静态回归，确认无 `db.swift`/`InitDB`/FileKit/SQLite 残留引用。
- **若保留**：需入编译并接入 UI，属未来迭代，本批不覆盖；测试侧不据此设阻塞项。

### 决策依赖
- SQLite 去留等待**用户**对"近期是否规划待办/日程管理"表态（产品规划问题，非技术判断）。在确认前，测试方案不推进 TC-51、不固化其相关断言。

---

## 汇总：实施依赖与风险一览

| 口径 | 当前是否满足 | 需工程师改动 | 兼容性风险 | 测试解锁 |
|---|---|---|---|---|
| C1 缓存自然年失效/离线复用 | 否 | `populateStocks` 分支顺序 + 失效判定 + 坏缓存清理 | 中（刷新风暴/限流、不清空缓存） | TC-18/19/20/21/22 |
| C2 HTTPS/ATS | 否 | URL scheme + `Info.plist` ATS + 失败态 | 低-中（**需先确认 timor HTTPS 可达**） | TC-31/32/33/34 |
| C3 菜单栏重置当前月 | 是 | 无 | 无 | TC-35/36 |
| C4 今天/节假日时区 | 是（农历待验证） | 农历 TZ 固定（视 TC-39 结果） | 低 | TC-37/38/39/40 |
| SQLite 清理 | —（死代码） | 删 `db.swift` + 依赖（若决策清理） | 低 | TC-51（仅清理分支） |

**前置确认项（交工程师/规划师）**：
1. timor.tech 是否提供有效 HTTPS 端点（C2 落地前提，需联网验证，超出只读评估范围）。
2. C1 自然年失效的工程实现口径（年内首次 vs 会话首次/版本号/TTL）。
3. 农历时区边界（TC-39）是否实际偏差，需运行期/单元验证。
4. SQLite 去留（用户确认）。

**结论**：C3/C4（今天/节假日部分）当前代码已满足，仅需回归固化；C1/C2 需工程师修复后通过，C2 有一个外部前置确认（timor HTTPS 可达）。当前无测试侧 P0，方案不因 SQLite 决策而阻塞。
