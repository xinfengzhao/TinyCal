# TinyCal 假期拼假助手技术方案（草稿 v1.2）

| 项 | 值 |
|---|---|
| 状态 | 草稿，待产品口径确认后实施 |
| 仓库基线 | `refactor/code-quality-polish` / `7eac58bdfc57c2620397a78713c58cc1edd2a426` |
| PRD | [TinyCal 假期拼假助手](https://docs.xiaohongshu.com/doc/a732bef7c55b4eb4c3f95eabbbc9cc41)，读取 hash `dbe50e62d63b89a7f442d134c3a72793` |
| 测试方案 | `TEST_PLAN_HOLIDAY_BRIDGE.md`，HB-1～51 + TC-G1 |
| 编写日期 | 2026-08-24 |
| 实施约束 | 本文只设计方案，不修改生产代码 |

## 修订记录

| 版本 | 日期 | 说明 |
|---|---|---|
| v1 | 2026-08-24 | 初版技术方案 |
| v1.1 | 2026-08-24 | 按质量评审修正 G1 索引公式、MR 测试映射，并补充候选过滤、支配去重、unknown、业务 code、缓存兼容、任务代次和 UI 状态迁移验证 |
| v1.2 | 2026-08-24 | 补充 timor.tech HTTP/HTTPS 联网证据、响应状态映射及发布阻塞结论 |

## 1. 目标与非目标

### 目标

- 在未来 12 个月范围内，基于中国大陆法定节假日、调休上班日和周末生成可解释的拼假方案。
- 排序遵循“需请假的法定工作日最少 → 连续休息自然日最多 → 起始日期最近”。
- 支持固定 Top 5 推荐、按目标节假日或期望连休天数筛选、方案详情及月历联动高亮。
- 展示数据年份、来源和更新时间，支持强制刷新以及旧缓存降级。
- 首期无账户、不接入 EventKit、不申请通知权限、不持久化选中方案。
- 建立最小 XCTest Target；算法为纯函数，网络、文件、时间均可注入。

### 非目标

- 年假余额管理、企业自定义排班、历史规律估算、方案收藏与菜单栏倒数。
- 多国家/地区节假日、账号同步、审批系统、系统日历写入和通知提醒。
- 借本功能重构无关菜单栏、设置或历史 SQLite 待办草稿。

## 2. 已确认现状与前置缺陷

- `CalendarView` 通过 `MyCalendar.getMonthPart` 生成月历；`HolidayViewModel.stocks` 使用 `stocks[yyyy][MM-dd]` 两级键查询假期。
- `HolidayViewModel.populateStocks` 依次读取内存、年份 JSON 文件和网络；没有元数据、TTL、强制刷新、请求去重或可观察错误状态。
- 当前接口为明文 HTTP，`Info.plist` 使用 `NSAllowsArbitraryLoads=true`。
- 工程没有 XCTest/UI Test Target。
- G2“周一表头与 weekday 错位”为误报：月首补位正确。
- G1 为真实缺陷：月份最后一天为周日时，尾部循环仍追加 7 天，产生幽灵周。

G1 修复采用列索引公式，避免继续维护分支：

```swift
let trailingCount = (6 - endDate.mondayBasedWeekdayIndex) % 7
```

其中明确使用 0 基索引 `Mon=0 ... Sun=6`，因此补齐到周日所需天数为 `(6 - index) % 7`：周一月末补 6 天，周六月末补 1 天，周日月末补 0 天。仅在 `trailingCount > 0` 时追加 `1...trailingCount`。TC-G1 必须参数化覆盖全部 7 种月末星期，而不只覆盖周日，以防修复幽灵周时破坏其余六种边界。

## 3. 产品决策与默认值

| 事项 | 方案建议 | 实现影响 |
|---|---|---|
| 年假余额 | 首期不支持，仅在查询模型预留可选约束 | 非阻塞，按“不支持”设计 |
| Top N | 固定 Top 5，不做设置项 | 非阻塞，常量实现 |
| 跨年 | 支持从“今天”开始未来 12 个月内跨自然年 | 阻塞实现，需产品写回 PRD |
| 次年未公布 | 不估算；缺失年份不参与确定性计算，并明确提示结果不完整 | 阻塞实现，需产品写回 PRD |
| XCTest | 首期硬门禁 | 非阻塞，按必须实施设计 |
| HTTPS/ATS | 首期安全门禁；先验证同域 HTTPS，成功后移除全局放行 | 若服务不支持 HTTPS，阻塞发布并需选新数据源 |
| 推荐候选资格 | 默认要求方案包含至少一个法定节假日且至少一个建议请假日 | **新增阻塞项**；否则普通周末的 0 请假方案会占满 Top 5 |
| 期望天数上限 | UI 输入 `1...30` | 非阻塞，沿用 PRD建议 |

除“候选资格”外，本文按表中建议值完成结构设计；未确认前不得固化相应产品行为。

## 4. 领域模型

建议新增 `TinyCal/Models/HolidayBridgeModels.swift`：

```swift
enum WorkdayKind: Equatable {
    case regularWorkday
    case weekend
    case statutoryHoliday(name: String)
    case adjustedWorkday(name: String)
    case unknown
}

struct CalendarDay: Equatable {
    let date: DateOnly
    let kind: WorkdayKind
}

struct BridgeQuery: Equatable {
    let range: DateInterval
    let minimumRestDays: Int?
    let targetHolidayName: String?
    let limit: Int
}

struct BridgePlan: Identifiable, Equatable {
    let id: String
    let startDate: DateOnly
    let endDate: DateOnly
    let leaveDates: [DateOnly]
    let holidayDates: [DateOnly]
    let adjustedWorkDates: [DateOnly]
    let holidayNames: [String]

    var leaveDayCount: Int { leaveDates.count }
    var restDayCount: Int { endDate.daysSince(startDate) + 1 }
}

enum HolidayDataFreshness: Equatable {
    case networkFetched(Date)
    case cached(Date)
    case stale(Date)
    case unavailable
    case unpublished
}
```

`DateOnly` 应封装固定 Gregorian 日历下的 `year/month/day`，算法内部不以 `DateFormatter` 字符串比较日期。与 UI/网络边界转换时显式传入 `Calendar(identifier: .gregorian)` 和系统时区，避免夏令时与时间分量污染“自然日”计算。

领域判定优先级：

1. 当天存在接口记录且 `holiday == false`：调休工作日。
2. 当天存在接口记录且 `holiday == true`：法定休息日。
3. 无接口记录但为周六/周日：普通周末。
4. 其他：普通工作日。
5. 目标年份数据缺失时标记 `unknown`，不得默认当作普通工作日或休息日。

## 5. 拼假算法

新增 `TinyCal/Services/HolidayBridgePlanner.swift`，对外只暴露同步、无副作用的纯函数：

```swift
protocol HolidayBridgePlanning {
    func plans(days: [CalendarDay], query: BridgeQuery) -> [BridgePlan]
}
```

### 5.1 输入准备

- `today` 由注入的时间源提供，按本地自然日归一化。
- 查询区间为 `[today, calendar.date(byAdding: .month, value: 12, to: today))`，末端不包含。
- 数据层先加载区间涉及的所有年份，再一次性生成连续、无重复的 `[CalendarDay]`。
- 任一年份为 `unavailable/unpublished` 时，将该年份日期标为 `unknown`；算法不得生成跨越未知日期的确定性方案。

### 5.2 候选生成

在最多约 366 天的数据上枚举连续区间：

1. 每个区间内的普通工作日和调休工作日计为 `leaveDates`；周末及法定休息日自然休息。
2. 区间必须连续、不得包含 `unknown`。
3. 默认推荐候选需包含法定节假日，并按待确认默认值要求至少 1 个请假日。
4. 指定目标节假日时，候选必须覆盖该节假日的至少一天。
5. 指定期望连休天数时，`restDayCount >= minimumRestDays`。
6. 纯周末候选不进入推荐；完全无需请假的法定长假只进入节假日信息区，不进入拼假 Top 5。对应的 0 请假场景仍由算法分类测试覆盖，但推荐结果必须过滤。
7. 先按规范化的 `[startDate,endDate,leaveDates]` 去除完全重复项；再在同一查询及同一目标节假日分组内做支配去重。若方案 A 的请假天数不多于 B、连续休息天数不少于 B，且至少一项严格更优，则 A 支配 B，B 不进入推荐。不同目标节假日之间不互相支配，避免 Top 5 丢失节日多样性。

区间数量约为 `366 × 367 / 2 ≈ 67k`，每个区间若增量维护工作日计数、假期标记和名称，时间复杂度 O(n²)、空间复杂度 O(k)，无需并行或复杂缓存。计算在后台 Task 执行，目标基准为 Release 构建单次小于 100ms；UI 只接收最终值。

### 5.3 排序和截断

稳定排序键依次为：

1. `leaveDayCount` 升序；
2. `restDayCount` 降序；
3. `startDate` 升序；
4. `endDate` 升序，作为完全稳定的最终键。

排序后截取固定 Top 5。算法层接收 `limit` 便于测试和未来扩展，但首期 UI 固定传 5。

### 5.4 无方案参考值

空列表时额外返回在当前完整数据范围内可达到的最大连续休息天数，用于空态提示。若数据不完整，则只声明“当前已公布数据内未找到”，不提供可能误导的全局最大值。

## 6. 数据层、缓存与刷新

### 6.1 可注入边界

```swift
protocol HolidayRemoteDataSource {
    func fetch(year: Int) async throws -> HolidayYearPayload
}

protocol HolidayCacheStore {
    func load(year: Int) throws -> CachedHolidayYear?
    func save(_ value: CachedHolidayYear, year: Int) throws
    func remove(year: Int) throws
}

protocol DateProviding {
    var now: Date { get }
}
```

`HolidayViewModel` 通过初始化器接收这些协议的生产实现，默认参数保持现有调用方兼容。网络实现基于可注入 `URLSession`；测试使用自定义 `URLProtocol`。文件实现允许测试注入临时目录或内存 Store。

### 6.2 缓存格式与兼容

- 新缓存记录包含 `schemaVersion`、`year`、`fetchedAt`、`sourceURL` 和原始 `Response`。
- 读取时兼容现有仅含 `Response` 的年份 JSON；以文件修改时间补充 `fetchedAt`，下次成功刷新后迁移为新格式。
- 坏缓存应记录失败、移除或隔离后尝试网络回源，不能每次解码失败后直接结束。
- 首期不设置自动 TTL；以 `freshness` 明示数据时间，并提供手动刷新。这与 PRD当前策略一致。

### 6.3 加载与强制刷新

- 普通加载：内存 → 文件 → 网络，同一年份使用 in-flight Task 表去重。
- 强制刷新：绕过内存和文件读取，直接请求网络；成功后原子覆盖文件和内存，再触发重新计算。
- 强刷失败且有旧值：保留旧值和现有方案，状态变为 `stale`，显示非阻塞警告。
- 强刷失败且无旧值：年份状态为 `unavailable`；基础月历继续工作，拼假计算不可用或仅对完整年份的区间给出明确受限结果。
- 接口 HTTP 200 但业务 `code` 非成功值、空 `holiday`、年份不匹配均不得覆盖有效缓存。空数据结合请求年份和接口语义映射为 `unpublished` 或 `invalidResponse`，映射规则需在联调后固化 fixture。
- 文件写入采用临时文件后原子替换，避免进程中断产生截断缓存。

补充验证口径：

- `Response.code` 非成功时即使 HTTP 为 200，也必须返回业务错误；有旧缓存时继续使用旧值，无旧缓存时进入不可用态，且不得写文件或覆盖内存。
- 旧格式年份缓存（直接编码 `Response`）必须可读；读取后以内存兼容，不要求立即写盘迁移。只有网络刷新成功时才原子写入带 `schemaVersion/fetchedAt` 的新格式。
- 坏缓存与“次年未公布”必须区分：前者尝试回源并报告缓存损坏，后者是合法数据状态，不反复重试制造请求风暴。

响应到状态的首期映射定义：

| 响应 | 状态 | 缓存行为 |
|---|---|---|
| 传输失败、超时、TLS 失败或 HTTP 非 2xx（包括 403） | `unavailable` | 保留并降级使用旧缓存；不得解释为“未公布” |
| HTTP 2xx 但非 JSON或字段无法解码 | `invalidResponse` | 保留旧缓存，不写盘 |
| HTTP 2xx、可解码，但 `Response.code` 不是经 fixture 确认的成功码 | `invalidResponse` | 保留旧缓存，不写盘；除非供应商文档明确给出独立“未公布”业务码，否则不得猜测 |
| 成功码、`holiday` 为空、请求年份晚于当前自然年 | `unpublished` | 不写空缓存，展示“该年度安排尚未公布” |
| 成功码、`holiday` 为空、请求年份为当前或历史年份 | `invalidResponse` | 保留旧缓存并允许重试 |
| payload 中可验证年份与请求年份不一致 | `invalidResponse` | 不写盘、不参与计算 |
| 成功码、非空且日期均属于请求年份 | 可用数据 | 原子写入并替换内存 |

由于现有 `Response` 没有顶层年份字段，“年份不匹配”通过逐条校验 `Holiday.date` 的年份实现；若接口日期字段实际不含年份，则必须在 Remote DTO 层补充可验证信息或取消该项校验，不能用字典键 `MM-dd` 推断年份。

## 7. HTTPS 与 ATS

实施前进行只读联调确认 `https://timor.tech/api/holiday/year/{year}` 的证书、重定向、响应结构和可用性：

### 7.1 2026-08-24 联网核对结果

- `http://timor.tech/api/holiday/year/{year}` 对 2026、2027 均返回 HTTP 301，重定向到同路径 HTTPS。
- HTTPS TLS 握手及 HTTP/2 可达，但对 2026、2027、2099 均返回 HTTP 403、`cf-mitigated: challenge` 和 HTML Challenge 页面，不是接口 JSON。
- 因所有年份都在 Cloudflare 层被拦截，本次无法观察成功 `Response.code`、空 `holiday` 或次年未公布的真实业务响应；403 只能映射为 `unavailable`，不能映射为 `unpublished`。
- 当前结果表明“支持 HTTPS”不等于“原生 macOS 客户端可稳定调用”。进入 MR-2 前必须以应用使用的 `URLSession` User-Agent 在目标网络环境复测，或取得供应商 API 客户端放行说明。

- HTTPS 同域可用：生产 URL 改为 HTTPS，删除 `NSAllowsArbitraryLoads`；保留沙盒 `network.client` 权限。
- HTTPS 不可用、持续触发 Challenge 或稳定性不足：不得静默保留全局 ATS 放行作为首发方案；HTTP 当前也只会重定向到 HTTPS，有限 ATS 例外不能解决 Cloudflare 403。应更换可供客户端直接调用的数据源、增加受控服务端代理，或由供应商放行；这将阻塞发布而非阻塞算法开发。
- 测试覆盖 TLS/网络失败、非 200、业务错误、坏 JSON、空数据和超时；不记录用户查询参数或年假信息。

## 8. UI 状态与调用链

新增 `HolidayBridgeViewModel` 作为拼假功能单一状态源：

```swift
enum HolidayBridgeViewState {
    case idle
    case loading
    case list([BridgePlan], DataSourceSummary)
    case detail(BridgePlan, DataSourceSummary)
    case empty(maximumRestDays: Int?)
    case stale([BridgePlan], warning: String)
    case unavailable(message: String)
    case unpublished(years: [Int])
}
```

调用链：

```text
用户打开拼假助手
  → HolidayBridgeViewModel.load(now, 12 months)
  → HolidayViewModel/HolidayRepository 加载涉及年份
  → HolidayCalendarBuilder 生成 CalendarDay 序列
  → HolidayBridgePlanner 纯函数计算并排序 Top 5
  → HolidayBridgePanel 展示列表/空态/异常态
  → 用户选中 BridgePlan
  → ContentView 共享 selectedPlan
  → CalendarView 跳转 startDate 月份并叠加 leaveDates 高亮
```

### UI 组织

- `ContentView` 持有 `HolidayBridgeViewModel`、当前面板模式和 `selectedPlan`，避免放在单个日期 cell 内。
- `CalendarView` 新增只读 `highlightedLeaveDates: Set<DateOnly>`；原“休/班/农历”文字逻辑不变，建议请假使用背景描边或角标作为叠加层。
- Popover 当前宽度约 300、内容宽 260；首期采用同一 Popover 内的月历/助手模式切换，避免同时塞入完整列表导致溢出。
- 点击方案后回到月历并跳转方案开始月份；跨月方案随用户翻页继续高亮。
- Popover 关闭时清空选中方案；列表是否保留仅限当前 `HolidayBridgeViewModel` 生命周期。

### 状态行为

- 加载：显示进度并禁止重复刷新。
- 旧缓存：列表可用，顶部持续显示“可能非最新”与更新时间。
- 无缓存且失败：仅助手不可用，保留重试按钮，不影响基础月历。
- 次年未公布：不生成跨未知区间方案；展示缺失年份和“官方数据尚未公布，公布后请刷新”。
- 无方案：展示调整目标天数建议和可用的最大连休参考。
- 刷新失败：不清空已有列表或高亮。

## 9. 并发、线程与性能

- 文件 I/O、JSON 编解码、网络和 O(n²) 计算均不在 MainActor 上执行；只在发布 UI 状态时切回 MainActor。
- 每年只允许一个 in-flight 请求；强刷与普通加载对同一年份串行化，采用“后发强刷覆盖普通加载”的明确策略。
- ViewModel 保存计算 generation ID；过期 Task 完成时不得覆盖较新的查询结果。
- `Set<DateOnly>` 提供 O(1) 高亮查询，避免每个 cell 对方案数组线性扫描。
- 12 个月约 366 天，算法性能目标 Release <100ms、无明显 Popover 卡顿；实际基准随 MR 提交。

任务代次必须有确定性测试：先启动慢查询 A，再启动快查询 B；即使 A 最后完成，最终状态仍必须属于 B。关闭 Popover 或切换查询时取消旧 Task；无法及时取消的依赖结果也由 generation ID 丢弃。

## 10. 异常、兼容与回滚

- macOS 最低版本保持工程现状 12.0，验证 Intel 与 Apple Silicon 架构构建。
- 保持 UserDefaults key、现有年份缓存可读和基础月历入口不变。
- 无新系统权限、无账户、无数据库迁移。
- 所有拼假 UI 位于独立入口；出现严重问题可移除入口并回滚新模块，G1 修复与数据层兼容改造可独立保留。
- 不使用历史 `db.swift`；该文件不在工程 Target 中，与本功能无关。
- Cron 更新 AppKit 主线程问题不属于本期功能范围；若测试确认失败，单独缺陷处理，不夹带进本功能 MR。

## 11. 文件级改动清单

### 修改

- `TinyCal/Utils/Date.swift`：修复 G1，提取可测试的周一基准索引/尾部补位计算。
- `TinyCal/Models/Models.swift`：保持接口 DTO；如需最小兼容改动，仅增加校验辅助，不承载拼假状态。
- `TinyCal/View Models/HolidayViewModel.swift`：依赖注入、加载状态、按年元数据、强制刷新及 in-flight 去重。
- `TinyCal/Services/Webservice.swift`：注入 URLSession、业务响应校验、HTTPS URL。
- `TinyCal/Utils/File.swift`：实现 `HolidayCacheStore`、缓存元数据、原子写入和坏缓存回源。
- `TinyCal/ContentView.swift`：持有助手状态、入口模式和选中方案。
- `TinyCal/Views/CalendarView.swift`：接收建议请假日集合、跳月及叠加高亮。
- `TinyCal/Info.plist`：HTTPS 验证通过后移除 `NSAllowsArbitraryLoads`。
- `TinyCal.xcodeproj/project.pbxproj`：加入新源码和最小 XCTest Target。

### 新增

- `TinyCal/Models/HolidayBridgeModels.swift`：DateOnly、查询、方案、日期分类和数据新鲜度。
- `TinyCal/Services/HolidayBridgePlanner.swift`：纯函数候选生成、去重、排序和 Top N。
- `TinyCal/Services/HolidayRepository.swift`：跨年数据聚合、缓存/网络协调与状态映射。
- `TinyCal/View Models/HolidayBridgeViewModel.swift`：UI 状态机、异步加载、刷新和任务代次控制。
- `TinyCal/Views/HolidayBridgeView.swift`：入口、推荐/自定义、列表、详情及各类状态视图。
- `TinyCalTests/DateGridTests.swift`：TC-G1 与各 weekday 月末回归。
- `TinyCalTests/HolidayBridgePlannerTests.swift`：HB-1～15。
- `TinyCalTests/HolidayRepositoryTests.swift`：HB-16～28、35～39。
- `TinyCalTests/HolidayBridgeViewModelTests.swift`：加载、刷新、选择、高亮状态转换。
- `TinyCalTests/Fixtures/`：合法、坏缓存、空数据、跨年、闰年和调休 JSON。

实际实施时可按工程 Target membership 调整文件归属，但不得复用未接入的 SQLite 草稿。

## 12. 分阶段实现与 MR 验证计划

### MR-1：地基、G1 与算法

- 建立 XCTest Target 和 fixture 基础设施。
- 引入 DateOnly/可控时间源、修复 G1。
- 实现纯函数 Planner、排序、去重和性能基准。
- 验证：TC-G1（全部 7 种月末星期）、HB-1～15、HB-47、HB-50、HB-51；执行 Debug/Release 构建。HB-48/49 不属于 MR-1。

### MR-2：数据、缓存、刷新与安全

- 引入 Remote/Cache 协议、Repository、缓存元数据和坏缓存回源。
- 实现跨年加载、未公布状态、强刷保旧、请求去重。
- 验证 HTTPS 后移除全局 ATS 放行。
- 验证：HB-16～28、HB-35～39、HB-48、HB-49；补充业务 `Response.code`、旧缓存兼容、unknown 跨界和坏缓存区分用例；检查 `Info.plist` 不含全局 arbitrary loads。

### MR-3：UI 联动与回归

- 实现助手状态机、列表/详情/空异常态和月历高亮。
- 验证：HB-29～34、HB-40～46；补充 ViewModel generation ID 和 `idle → loading → list/detail/empty/stale/unavailable/unpublished` 合法迁移测试；手工覆盖深浅色、Popover 收起、跨月翻页、离线和刷新失败。

每个 MR 必须附：基线/目标 commit、变更范围、实际执行命令、测试通过数、失败或豁免、截图/录屏（UI MR）、风险和回滚方式。未执行的验证不得标记通过。

## 13. 质量门禁

- 产品必须确认跨年、次年不估算、候选资格和 HTTPS处理方式。
- TC-G1 必须通过，不允许幽灵周进入首发。
- HB-1～15 与可测性 HB-47～51 全通过。
- 候选过滤测试必须证明纯周末及零请假法定长假不进入 Top 5，同时节假日信息仍可展示。
- 支配去重必须覆盖完全重复、同节日被支配和不同节日不互相支配三类场景。
- 网络/缓存失败不得清空有效旧方案，坏缓存不得永久阻塞回源。
- `unknown` 日期不得被跨越；缺失年份不得生成看似确定的跨年方案。
- HTTP 200 但 `Response.code` 非成功不得覆盖缓存；旧格式缓存必须保持可读。
- 过期 ViewModel Task 不得覆盖新查询状态；UI 状态迁移必须可重复测试。
- 无新增 EventKit/通知权限；无用户查询数据上传。
- Debug/Release 在 macOS 12+ 目标成功构建，相关单测全部通过。
- UI 不破坏月历导航、农历/节假日标记、设置和菜单栏刷新。

## 14. 待确认事项

### 阻塞实现

1. 默认推荐是否明确排除“0 天请假普通周末”，以及是否要求候选至少包含一个法定节假日和一个建议请假日。
2. 首期是否正式支持未来 12 个月内跨自然年方案。
3. 次年未公布是否确认“不估算、不生成跨未知日期的确定性方案”。
4. HTTPS 同域不可用时，是更换数据源还是允许经过安全评审的有限域名例外。

### 建议默认值，可先开发公共结构

1. 年假余额首期不支持，仅预留查询约束。
2. Top N 固定为 5，不提供用户配置。
3. 并列最终按起始日期、结束日期升序，保证稳定输出。
4. XCTest 是首期硬门禁，算法及日期网格无自动化测试不得合并。
5. 最低系统保持 macOS 12.0，同时验证 Intel/Apple Silicon。

## 15. 结论

方案采用“可注入数据仓库 + 纯函数 Planner + MainActor UI 状态机”三层结构，以 MR-1/2/3 分离日期/算法、数据安全和 UI 风险。当前可以先实现不依赖产品口径的基础设施；推荐候选资格、跨年降级及 HTTPS 不可用时的安全策略确认前，不应完成或发布首期功能。
