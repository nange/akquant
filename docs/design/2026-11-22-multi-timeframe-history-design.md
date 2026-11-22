# 多周期历史：低频序列的容量解耦与用户注入

- 日期：2026-11-22
- 状态：草案（待评审）
- 关联：[多周期策略：三条路径](../zh/advanced/multi_timeframe_feed_api.md)；内部设计文档 `docs/superpowers/specs/2026-08-27-live-history-preload-design.md`、`docs/superpowers/specs/2026-10-21-auto-history-preload-design.md`（未随本仓库分发，仅作代码注释出处）

## 1. 问题

主数据周期为 1min 时，周/月周期与「会话开始之前的长历史」这两件事目前都无法表达：

1. `subscribe_bars` / `get_history(freq=)` 只接受 `Nmin` / `Nh` / `1d`，**周线、月线在入口就被拒绝**（`src/data/window.rs:44`、`python/akquant/strategy_window.py:24`）。
2. 高周期序列（`window_data`）虽然是独立缓冲，但**创建时的容量继承了基础周期深度**：`update_window` 用 `HistoryBuffer::default_capacity` 建序列（`src/history.rs:472`）。回测里它取 `max(warmup_period, history_depth)`（`python/akquant/backtest/engine.py:3089`），实盘取预热深度与 10000 的较大者（`python/akquant/live/_runner.py:1235`）。于是「1min 只留 100 根、日线留 2000 根」这种组合今天做不到——日线序列也只有 100 根。
3. 实盘会话启动时基础历史为空；即使开启自动预热，上限也是 10000 根**基础** bar（`python/akquant/live/_preload.py:265`）。1min × 240 根/天 ≈ **41 个交易日**，对应 41 根日线、8 根周线、1~2 根月线，周/月策略不可用。
4. 回测里日线能随输入数据累积，但受第 2 条约束：基础深度小的时候，日线序列同样被截断；周/月则根本不存在。
5. 前复权会改写除权日之前的全部价格；现有 `CorporateActionManager` 只处理持仓/现金，历史序列与 `self.I()` 都不会自动重算，复权日盘中指标必然错。

用户提出的方案是：「框架提供多周期数据注入接口；查询注入过的周期时不再从 1min 聚合，而是直接查注入数据；实盘每天在 `on_pre_open` 重灌」。

**这个方向是对的，但「查询时双源切换」的形态不建议采纳**：它会让同一个 `(symbol, freq)` 出现两个真相——`get_history` 读注入，而 `on_window_bar`、`current_window`、`self.I(freq=)` 仍读聚合。窗口指标只在 `on_window_bar_event` 里推进（`python/akquant/strategy_window.py:301`），注入不派发事件就不会更新，且没有任何报错。这比「数据少」更难排查。

### 1.1 与你提出的方案的差异

| 你的提案 | 本设计 | 原因 |
| --- | --- | --- |
| 框架提供多周期数据注入接口 | 保留（`inject_history` / `period_history`） | 需求成立 |
| 查询注入过的周期时不再从 1min 聚合，改查注入数据 | 注入写入**同一条** `(symbol, freq)` 序列；聚合继续追加；读取无分支 | 避免同周期两个真相；指标/回调/checkpoint 只有一条序列 |
| 实盘每天 `on_pre_open` 重灌 | 支持，但降级为「可选刷新」；启动 seed 是主路径 | 重启本来就会 seed；周/月新 bar 由 F1 聚合；每日重灌只解决长驻进程的 revision |
| （未涉及）周/月聚合 | 新增 `1w` / `1mon` | 周/月不是「只能靠注入」，原生聚合是零用户成本的基础能力 |
| （未涉及）容量按周期 | 新增 `set_history_depth(n, freq=)` | 「1min 留 100、日线留 2000」是本次最直接的痛点 |
| 复权后重新注入各周期历史（本次追加） | F4：`apply_adjustment_from_history`（从 qfq 数据推断比例，推荐）/ `apply_price_adjustment`（显式因子）/ `refresh_history`（数据重灌）+ **指标重建** | 只重灌历史不修 `self.I()` 状态，指标仍按旧基准算，且不报错 |

## 2. 目标与非目标

### 目标

- **G1**：原生支持 `1w` / `1M`（周/月，内部规范标签为小写 `1w` / `1mon`）窗口周期，闭合与标签口径与现有日线一致。
- **G2**：历史保留容量按周期独立配置，低频序列不再被基础周期深度截断。
- **G3**：提供「会话开始之前的长历史」的注入能力，回测与实盘同一套 API、同一套语义。
- **G4**：注入后 `get_history` / `get_history_multi` / `get_history_df` / `self.I(freq=)` / checkpoint 仍然只有一条真相。
- **G5**：提供复权刷新机制（因子重基 / 数据重灌），覆盖 base/tick/日/周/月，并在刷新后重建增量指标；回测提供点时复权模式以避免前视。

### 非目标

- 不支持秒级、季度、年度周期（本次不做；秒级建议继续走 `feed.resample`）。
- 不改变窗口 bar 不参与撮合的既有约束。
- 不把注入做成「旁路容器」：不新增第二套 `(symbol, freq)` 历史存储。
- 不解决 `extra` 字段参与窗口聚合的问题（注入路径可以携带 extra；聚合路径仍然丢弃，见 `src/data/window.rs:241`）。

## 3. 设计原则

1. **单一真相**：每个 `(symbol, freq)` 只有一条 `SymbolHistory`。注入是**写入**，不是查询时的分支。
2. **语义分离**：「保留多少」（容量）与「数据从哪来」（注入/聚合/预热）是两个独立问题，分别用独立 API 表达。
3. **口径统一**：日/周/月的标签一律是「该周期最后一个交易日的收盘戳」，与现有日线口径一致；不接受 pandas 的 00:00 / `W-FRI` 00:00 标签直接入库。
4. **默认安全**：注入时未来 bar 默认拒绝；重复时间戳默认拒绝；跨源 revision 默认告警。
5. **可退化**：不订阅、不注入时，现有行为逐字节不变。

## 4. 总体方案

分三层，可独立落地：

| 层 | 内容 | 解决 | 用户成本 |
| --- | --- | --- | --- |
| **F1** | 周/月原生聚合 + 低频序列容量独立（`set_history_depth(n, freq=)`） | 回测与实盘「往后」的周/月；小基础深度下的日线截断 | 零（只加 API） |
| **F2** | `ctx.inject_history`：写入同一条 `(symbol, freq)` 序列，`replace`/`upsert` | 实盘「会话开始之前」的长历史 | 需用户供数 |
| **F3** | `run_backtest(period_history=)` / `run_live(period_history=/history_providers=)` | 启动 seed 的入口与自动供数 | 可选 |
| **F4** | 复权刷新：`apply_price_adjustment` / `refresh_history` + 指标重建；回测 `adjust_mode="point_in_time"` | 复权日后历史与指标口径 | 手动或 provider 自动 |

推荐的实盘用法（F1+F2 组合）：

```python
class WeeklyStrategy(Strategy):
    def __init__(self) -> None:
        super().__init__()
        self.subscribe_bars("1w")          # F1: 让引擎继续聚合新的周线并派发回调
        self.weekly = self.I(aq.SMA(20), freq="1w", source="close")

    def on_pre_open(self, event) -> None:
        # F2: 每天把「截至昨日」的周线补进同一条序列；upsert 幂等
        self.inject_history(symbol="600000.SH", freq="1w", data=fetch_weekly(...), mode="upsert")
```

这里 `subscribe_bars("1w")` 与注入**不冲突**：聚合负责「新的、实时的周线」，注入负责「会话开始之前的旧周线」，二者写同一条序列，按时间戳 upsert。

## 5. 详细设计

### 5.1 F1a：周 / 月周期

#### 5.1.1 周期解析

`WindowFreq` 增加两个变体（`src/data/window.rs:36`）：

```rust
pub enum WindowFreq {
    Minutes(u32),
    Hours(u32),
    Day,
    Week,
    Month,
}
```

- 对外标签：**`1w`**、**`1M`**（大写 M 表示 month）；`1mon` 作为 `1M` 的等价小写别名。
- **内部规范标签一律小写**：`1w` / `1mon`。解析只在**边界**区分大小写：先看原始单位 token，`M` → `mon`，然后统一小写往下传。这样 `bar.freq`、历史桶键、订阅表、checkpoint 里的 `freq_label` 全是小写，现有那些 `to_ascii_lowercase()` / `.lower()` 调用**不需要逐个审计**——只有 `parse` 的边界不能先 `lower()` 再判断。
- 非法写法：`1m`、`1month`、`1mo`、`2w`、`3M` 一律报错；`1m` 的文案要明确「分钟请写 `1min`，月线请写 `1M`」。
- `total_minutes()` 仅用于闭合排序：`Week => 10080`、`Month => 43200`（近似值，不参与时间计算）。
- `session_windows`：周/月订阅传它直接报 `ValueError`（边界由交易日历决定，手写时段只对日内窗口有意义）；Rust `WindowSubscription::parse` 同样拒绝，不要让它落到 `place_daily` 的 `last_session` 分支。
- 新增 `fn kind(self) -> PeriodKind`，把现有 `period_seconds().is_none()` 的二分判断改为 `Intraday / Day / Week / Month` 四路；`place_daily` 的「无解析器时标签 = 当前 bar」逻辑只用于 `Day`。

Python 侧 `parse_window_freq`（`python/akquant/strategy_window.py:40`）同步：`_FREQ_RE` 改为 `^(\d+)(min|h|d|w|M|mon)$`（大小写敏感），匹配成功后归一化为 `1w` / `1mon`；不要在正则前 `lower()`。

**大小写边界清单**（实现时逐项确认，避免 `1M` 被某处 `.lower()` 变成 `1m`）：
- Rust：`WindowFreq::parse`、`WindowSubscription::parse` 的 `freq_label`、`is_forming`/`current` 的标签归一化、`context.rs::resolve_history_source` 的 `to_ascii_lowercase()`。
- Python：`strategy_window.parse_window_freq`、`strategy_window.subscribe_bars_impl` 的 `label`、`strategy_history._infer_freq` 的 `label = str(resolved).strip().lower()`、`Strategy.I()` 的 `freq_label`。

> 结论：内部只认小写 `1w`/`1mon`，边界只做一次「`M` → `mon`」的映射。这样比「内部保留大写 M」少改一整圈 `.lower()`，也不容易漏。

#### 5.1.2 分组与标签

沿用日线的两个既有概念：

- **分组键**：`WindowState.local_date` 复用为「本窗口内最后一个交易日」。`Day` 用交易日本身；`Week` 用 `NaiveDate::iso_week()`（返回 `IsoWeek { year, week }`，天然处理跨年 ISO 周）；`Month` 用 `(year, month)`。
- **标签**：每根基础 bar 归属的 `Placement.label` 仍是「当日收盘戳」（`DaySpan::label`，含收盘恰为次日 0 点时减 1ns 的规则，`src/data/window.rs:566`）。周/月窗口在 merge 时把 `label_ns` 更新为当前交易日的收盘戳，因此窗口闭合时标签天然是「最后一个交易日的收盘」。

`update()` 的换窗判断（`src/data/window.rs:756`）改为：

```rust
let changed = match freq.kind() {
    PeriodKind::Day => st.local_date != place.date,
    PeriodKind::Week => st.local_date.iso_week() != place.date.iso_week(),
    PeriodKind::Month => (st.local_date.year(), st.local_date.month())
        != (place.date.year(), place.date.month()),
    PeriodKind::Intraday(_) => st.label_ns != place.label || /* 现有 session_key 逻辑 */,
};
```

`interval_start`：周/月窗口只在**首根** bar 上取 `place.interval_start`（前一交易日收盘），merge 时不覆盖；日线保持现有的「每根更新」。

#### 5.1.3 闭合时机

- 即时闭合（`close_at`）只适用于日内周期；周/月 `close_at = None`。
- 日终走 `close_through_day(day, close_ts)`（`src/data/window.rs:836`）：`Day` 仍按 `trading_day <= day`；`Week`/`Month` 需要判断 `day` 是否为本周期最后一个交易日。
  - 新增 `SessionResolver::next_trading_day(symbol, asset_type, day)`（与现有 `previous_trading_day` 对称，`src/calendar/resolver.rs:906`），比较 `day` 与 `next` 的 ISO 周 / 年月；不同则闭合。
  - `next_trading_day` 越界（日历末尾）时视为周期结束并闭合，避免丢最后一根。
  - 不能用全局 `next_session_day` 代替：期货/自定义时段表的「最后交易日」可能因品种而异。
- 会话结束 `flush()` 仍会闭合尾部未完成窗口（`src/data/window.rs:811`）。**部分周/月的标签是该段最后一根 bar 的收盘**，与现有「尾部日线 flush」同口径；文档需明确标注「最后一段是部分周期」。
- 边界情况：本周/本月最后一个交易日**没有 bar**（数据缺失/停牌）时，日终不会为该日调用 `close_through_day`；窗口会等到下一根落入新周期的 bar 到达时通过 `changed` 闭合，标签取该周期内**最后一根实际 bar** 的收盘。不要用 `close_ts` 伪造一个不存在的收盘价。

#### 5.1.4 快照兼容

`WindowStateSnapshot` 已有 `local_date`，且 `freq` 从 `freq_label` 重新解析（`src/data/window.rs:908`），所以：
- 旧存档（只有 `1d`/`Nmin`）反序列化不变；
- 新存档里的 `1w`/`1mon` 能被新版本解析；
- 旧版本读到 `1w`/`1mon` 存档会 `unwrap_or(Minutes(0))` 退化（只影响 `flush()` 排序，不影响已恢复数据），可接受；如需更严格，加 `snapshot_version` 并在 `run_from_checkpoint` 时提示。

### 5.2 F1b：按周期容量

#### 5.2.1 数据模型

`SymbolHistory.capacity` 本来就是 per-series（`src/history.rs:8`），缺的是「谁来决定它」与「裁剪时是否保护」：

```rust
pub struct SymbolHistory {
    // ...
    /// 该序列的容量来源。基础序列与日内窗口跟随全局；日/周/月独立。
    pub retention: Retention,
}

pub enum Retention {
    /// 跟随 `HistoryBuffer::default_capacity`（现有行为）。
    Following,
    /// 固定容量，不随 `set_history_depth` / `configure_history_depth` 收缩。
    Fixed(usize),
    /// 用户注入的序列：至少保留 `len()` 行，且在 `max_capacity` 内只增不减。
    Pinned { max_capacity: usize },
}
```

- `HistoryBuffer::update_window` 建序列时按 `WindowFreq::kind()` 选择 `Retention::Fixed(freq.default_retention())`（日/周/月，取值见下表）或 `Following`（日内）。
- `trim_to_capacity`（`src/history.rs:343`）与 `set_capacity_preserve_existing`（`src/history.rs:399`）只裁剪 `Following`；`Fixed(n)` 保持不变；`Pinned` 至少保留 `len()`，仅在超过 `max_capacity` 时从最老端淘汰并告警一次。
- `ensure_capacity(capacity, freq)`：对 `Following` 跟随基础容量增长；对 `Fixed`/`Pinned` 取 `max(当前, 请求值)`（只增不减），不传 freq 时保持现有「全部增长」语义（向后兼容）。

**容量取值**（按行数，不预分配，见 5.2.3）：

| 周期 | 默认上限 | 约合 |
| --- | --- | --- |
| `1d` | 4000 | 16 年交易日 |
| `1w` | 1200 | 23 年 |
| `1mon` | 400 | 33 年 |
| 日内窗口 | 跟随基础容量 | 与现有行为一致 |

这些是**上限**，不是预分配；用户可用 `set_history_depth(n, freq=)` 调大。

#### 5.2.2 按周期的 depth API

现状：`strategy._history_depth` 是标量，`get_history` 用它做「是否启用」的闸门（`python/akquant/strategy_history.py:242`），`_ensure_history_depth` 用它算自动加长上限（`:132`）。

改为：

```python
# strategy_history.py
_history_depths: Dict[str, int]   # key: "bar" / "tick" / "1d" / "1w" / "1mon" / ...
```

- `set_history_depth(depth, freq=None)`：
  - `freq=None` → 写 `"bar"`，并且**仍然**调用现有全局 `ctx.ensure_history_capacity(depth)`；基础 `bar`/`tick` 序列是按 `default_capacity` **懒创建**的（`src/history.rs:472` 的窗口序列同理），只记 dict 不抬 `default_capacity`，之后新建的基础序列容量仍是旧值。
  - `freq` 为具体周期（`"1d"` / `"1w"` / `"1mon"`）→ 只对该序列 `ensure_history_capacity(depth, freq)`（只增不减）。
  - 无 ctx 时先记下，由引擎在 `on_start` 之后统一补齐。
- `get_history(count, freq=...)` 的闸门按解析后的 freq 取对应 depth；`"bar"`/`"tick"` 共用 `"bar"` 的 depth（与现有 tick/bar 共用一个 `_history_depth` 的行为一致）。
- 自动加长上限 `max(10000, 该 freq 的声明 depth)`；`ctx.ensure_history_capacity(count, freq)` 只长该序列（`freq` 为 `bar`/`tick` 时仍走全局，保证 `default_capacity` 被抬高）。
- `run_backtest(history_depth=...)` 接受 `int | Dict[str, int]`；`run_live` 的 `preload_bars` 仍只描述基础周期，窗口序列由 5.2.1 的固定上限兜底。

**行为变更提示**：`set_history_depth(100)` 之后，日线序列从「100 行」变成「独立上限 4000 行（实际受数据长度限制）」。这会让已有窗口策略的 `get_history(freq='1d')` 从「大量 NaN」变成「完整数据」，属于行为变更，需要 CHANGELOG 标注与 golden 基线重签。

#### 5.2.3 内存：停止对窗口序列预分配

`SymbolHistory::new(capacity)` 对 6 个 `VecDeque` 做 `with_capacity(capacity)`（`src/history.rs:127`）。若低频序列直接用 4000~100000，会对**每个标的**立即分配 6×cap×8B：

- cap=10000 → 480KB/标的/序列；5000 标的的日线序列 = 2.4GB（即使还没数据）。
- 所以**不能**简单把低频 capacity 设成 100000。

两种做法，建议同时采用：

1. 低频序列用**惰性分配**：新增 `SymbolHistory::new_lazy(capacity)`，`VecDeque::new()`，容量只作淘汰阈值；`extras` 新建列时也按 `min(capacity, len + 增量)` 预分配，而不是直接 `with_capacity(capacity)`（`src/history.rs:229` 附近）。
2. 低频上限取上表的 4000/1200/400，足够覆盖常见回测/实盘跨度，且单标的开销 < 200KB。

### 5.3 F2：注入

#### 5.3.1 数据模型：写同一条序列

注入**不新增容器**，直接写 `window_data[(symbol, freq)]`：

```rust
impl HistoryBuffer {
    pub fn inject_window(
        &mut self,
        symbol: &str,
        freq: &str,
        bars: Vec<Bar>,
        mode: InjectMode,          // Replace | Upsert
        cutoff_ns: Option<i64>,    // 拒绝 >= cutoff 的 bar
        max_capacity: usize,
    ) -> Result<InjectReport, InjectError>;
}
```

- 序列不存在则创建（`Retention::Pinned`）。
- 写完 `last_push = LastPush::None`：避免 `previous_view` 的「去掉最后一行」语义在批量写入后被误用（`src/history.rs:162`、`src/context.rs:1034`）。
- `capacity = min(max_capacity, max(capacity, len))`；超过 `max_capacity` 时从最老端淘汰并告警一次。
- 时间戳唯一且严格递增；`Upsert` 按 timestamp 合并（incoming 覆盖 existing），`Replace` 清空重建。

#### 5.3.2 输入规范化

复用 `_preload.py` / `normalize.py` 的列别名与时区处理，新增「标签规范化」：

| 输入标签 | 处理 |
| --- | --- |
| 日线 `00:00`（pandas `resample('1D')` 默认） | 映射到**同一交易日**的收盘戳（`DaySpan::label`） |
| 周线 `W-FRI` 00:00（pandas 默认右闭合） | 映射到该 ISO 周**最后一个交易日**的收盘戳 |
| 月线月末 00:00 | 映射到该自然月最后一个交易日的收盘戳 |
| 已是收盘戳（>= 当日收盘） | 保留 |
| 盘中时间戳（开盘~收盘之间） | `strict=True` 报错；`strict=False` 丢弃并告警（无法判断归属，宁可不要） |
| 未来 bar（>= cutoff） | `strict=True` 报错；`strict=False` 丢弃并告警 |

`normalize=True` 需要交易日历（`SessionResolver`）；无解析器（例如纯 `list[Bar]` 回测、无 `sessions`/`trading_calendar`）时无法可靠归一，改为**要求调用方给收盘戳**，否则报错并提示传 `trading_calendar=` 或 `normalize=False`。

时区：naive 时间按会话时区解释（与 `run_live(timezone=)` / `run_backtest(timezone=)` 一致）；带时区的时间先转换到会话时区再取本地日期。

#### 5.3.3 模式与 revision

- `Replace`（启动 seed 默认）：
  - 要求输入严格递增、无重复；
  - 如果输入的最后一行早于序列已有的最后一行，告警「将丢弃引擎已追加的 N 行」；
  - 只允许在**首个事件之前**（`on_start` 及之前）或该 freq 未订阅时使用；运行中 `Replace` 已订阅的序列报错，提示改用 `Upsert`。
- `Upsert`（每日刷新默认）：
  - 按 timestamp 合并；输入覆盖同 timestamp 的旧行；
  - 同 timestamp 的 OHLCV 与旧值相对差 > `1e-9` 时计一次 revision，汇总告警（每条序列每天最多一条日志，避免刷屏）；
  - 保留输入里没有、但序列里已有的行（例如引擎昨天 15:00 追加的日线）。
- 返回值 `InjectReport`：`written` / `replaced` / `kept` / `skipped_future` / `skipped_duplicate` / `revisions` / `first_ts` / `last_ts` / `series_len`。Python 侧记 INFO，并在 `revisions > 0` 时 WARNING。

#### 5.3.4 可见性与未来函数

- 注入的 cutoff 取 `ctx.current_time`（回调内）或会话启动时刻（启动前）；`on_pre_open` 阶段还要叠加 `_framework_history_cutoff_ns`（`python/akquant/strategy_framework_hooks.py:215`）。
- 注入**不**派发 `on_window_bar`、不触发用户回调（与 `preload_history` 一致），只把**新增的**行回放给该策略已注册的 `I(freq=...)` 增量指标：
  - `Upsert`：只回放 `timestamp > 上次指标 update 的 last_ts` 的行；
  - `Replace`：运行中禁止（见 5.3.3），启动 seed 由引擎在 `on_start` 之后统一回放（复用 `_preload_events` 的思路）。
- 读取入口的周期白名单要分开：
  - `get_history` / `get_history_multi` / `get_history_df` / `get_rolling_data` / `get_history_map`（`_infer_freq`）允许「**已订阅 ∪ 已注入**」；
  - `self.I(freq=)` 与 `current_window` 仍只认**已订阅**（指标由事件驱动、`current_window` 是「在形成窗口」，注入都不产生）。
- `current_window(symbol, freq)`：只注入、未订阅的周期没有「在形成窗口」，保持现有订阅校验并给出明确文案（「该周期只注入、未订阅，没有在形成窗口；请 subscribe_bars」），**不要返回 None**——返回 None 会与「已订阅但当前没有形成窗口」混淆。注入的是**已闭合** bar，不是 partial。

#### 5.3.5 指标与回调的边界

| 场景 | `get_history(freq=)` | `self.I(freq=)` | `on_window_bar` |
| --- | --- | --- | --- |
| 只注入、未订阅 | ✅ | ❌（报错，提示 `subscribe_bars`） | ❌ |
| 订阅 + 聚合 | ✅ | ✅（新闭合 bar 驱动） | ✅ |
| 订阅 + 注入 seed | ✅ | ✅（seed 回放 + 新 bar 驱动） | ✅（只对新 bar 触发） |
| 订阅 + 每次 upsert | ✅ | ✅（新增行回放） | ✅（只对新 bar 触发） |

这张表是本次设计的核心契约：**注入只影响历史与指标状态，不产生用户回调**；回调永远由引擎聚合的新 bar 驱动。

#### 5.3.6 checkpoint / 热启动

- `window_data` 已在 `HistoryBufferSnapshot` 里（`src/history.rs:553`），注入序列随存档持久化。
- `SymbolHistorySnapshot` 增加 `#[serde(default)] injected: bool`（或 `HistoryBufferSnapshot.injected_series: Vec<(String, String)>`），用途：
  - 恢复后仍按 `Pinned` 处理，不被后续 `configure_history_depth` 裁掉；
  - 日志/诊断：`run_from_checkpoint` 时可提示「已恢复 N 条注入序列，如需更新请重新 inject」；
  - 旧存档无该字段 → `false`，行为不变。
- `run_from_checkpoint(..., period_history=...)`：先恢复，再按 `Upsert` 应用新数据（幂等）；顺序写入文档。
- `preload_history`（基础周期）与 `inject_window`（低频）互不依赖；`preload_history` 的前置条件（必须先 `configure_history_depth`，`src/engine/python.rs:1012`）同样适用于注入——文档与实现都要在 `default_capacity == 0` 时给出明确报错，而不是静默丢弃。

#### 5.3.7 多策略 / slot

历史缓冲区是共享的（`Arc<RwLock<HistoryBuffer>>`）。注入对**所有策略与 slot 可见**：

- 文档化该语义；
- 指标回放按策略各自注册的 `I(freq=)` 分别进行（不共享指标状态）；
- 不提供按策略隔离的注入（如未来需要，可加 `owner` 标记，但不在本次范围）。

#### 5.3.8 错误处理

- 启动路径（`run_backtest(period_history=)` / `run_live(period_history=)`）：fail-fast，抛 `ValueError`。
- 运行中（`on_pre_open`）：`strict=True` 抛错，交给 `runtime_config.error_mode`；文档建议实盘用 `strict=True` + `error_mode="continue"`，但必须保证「失败后策略知道自己在用旧数据」——至少 WARNING，且 `InjectReport` 可查。
- 未订阅但注入成功：`get_history` 可用；若策略随后调用 `self.I(freq=)`，在 `I()` 处报错并提示 `subscribe_bars`（现有行为，`python/akquant/strategy.py:1733`）。

### 5.4 F3：入口 API

#### 5.4.1 回测

```python
run_backtest(
    data=...,                          # 基础周期数据（不变）
    period_history={                   # 新增：低频序列 seed
        "1d": {"600000.SH": df_daily},
        "1w": {"600000.SH": df_weekly},
    },
    history_depth=100,                  # int | Dict[str, int]
)
```

顺序（实现必须遵守）：

1. 建 `Engine`；
2. `configure_history_depth(base)`（**不是** `set_history_depth`，后者会 `clear()`）；
3. `add_data`；
4. `configure_window_subscriptions`；
5. `inject_window`（每个 freq）；
6. `engine.run()`。

#### 5.4.2 实盘

```python
run_live(
    ...,
    preload_history=df_1min,           # 现有：基础周期预热（不变）
    period_history={"1d": df_daily, "1w": df_weekly},   # 新增：低频 seed
    history_providers={"1w": fetch_weekly},             # 可选：自动供数
)
```

- **不要**重载 `preload_history` 的 `dict` 形态：它现在是 `{symbol: DataFrame}`，再解释成 `{freq: ...}` 会产生歧义。低频一律走新参数 `period_history` / `history_providers`。
- `history_providers` 签名：`callable(symbols: list[str], now_ns: int) -> dict[str, pd.DataFrame]`；框架负责归一化、cutoff、upsert、指标回放。provider 抛错时，启动阶段 fail-fast；自动模式下可降级为「本次不 seed + WARNING」（与现有 `history_provider` 自动模式一致）。
- 会话启动后，`period_history` 的 seed 只做一次；长驻进程需要刷新时用 `self.inject_history(..., mode="upsert")`。

#### 5.4.3 运行期

```python
report = self.inject_history(
    symbol="600000.SH",
    freq="1w",
    data=df_or_bars,
    mode="upsert",          # "replace" | "upsert"
    normalize=True,
    strict=True,
)  # 返回 InjectReport
```

- 必须走 `ctx`（`StrategyContext.inject_history`），不能走 `Engine`：`engine.run()` 期间再入 Engine pymethod 会 `Already borrowed`（见 `python/akquant/strategy_window.py:288` 的既有注释）。
- 可在 `on_start` / `on_pre_open` / `on_before_trading` / 行情回调内调用；`__init__` 无 ctx，报错。
- `freq` 必须是已解析的周期标签；`freq` 为基础周期（`bar`/`tick`/`1min`）时报错，提示用 `preload_history`。
- 基础周期的整段替换（含复权刷新）走 F4 的 `refresh_history`（见 5.5.3），不要用 `inject_history`：两者都会重建指标，但语义（追加 seed vs 替换整段）不同。

### 5.5 F4：复权刷新（Corporate Actions）

#### 5.5.1 问题

前复权是「以最新价为锚」的调整：任何一次除权除息都会把**除权日之前**的全部价格按比例改写。框架里受影响的有三层：

1. `get_history` 读到的历史序列（base bar / tick / 1d / 1w / 1mon）是旧基准；
2. `self.I()` 增量指标的状态是旧基准算出来的——**重灌历史不会自动修正它**；
3. 用户自己缓存的 numpy/DataFrame 不会被框架修正。

现有 `CorporateActionManager`（`src/market/corporate_action.rs:1`）只处理**持仓数量/现金**（split 改数量、dividend 发现金，`src/analysis/tracker.rs:346`），完全不碰历史价格；`adj_close` 只是用户自己放进 `extra` 的一列（`examples/16_adj_returns_signal.py`），复权日之后框架不会自动重算。所以「复权后指标错」是必然的。

#### 5.5.2 信号价与成交价必须分离

- **历史缓冲只服务信号与指标**：`get_history` / `self.I()` / `get_rolling_data`。
- **撮合、持仓成本、权益、风控、TradeTracker/MAE-MFE**读 feed 的 raw bar 与成交价，绝不复权。现有 split 已经通过 `Portfolio` / `TradeTracker` 单独处理数量与成本，不能被价格重基再动一次。
- 前复权的锚点是「最新价」，所以重基后历史序列的最后一个值仍等于 raw 最新价，`on_bar` 收到的 raw bar 与历史序列在接缝处天然一致。这也是选前复权（而不是后复权）做实时信号的原因。
- **缓冲里的不变量**：历史序列存的是「当前锚点的前复权价」。首次复权前锚点就是现在，所以它等于 raw；每次重基把锚点推进到新的 ex-date。任何时刻，序列最后一行都应等于 raw 最新价。

#### 5.5.3 三条刷新路径

| 路径 | API | 适用 | 成本 |
| --- | --- | --- | --- |
| A1. 显式因子重基 | `apply_price_adjustment(symbol, ex_date, factor, volume_factor=1.0, freqs="all")` | 手里有因子/券商事件 | Rust 单遍 O(rows)，无数据搬运 |
| A2. 从新旧前复权数据推断（推荐） | `apply_adjustment_from_history(symbol, data=df_qfq_new, reference_freq="1d", ...)` | 只有 qfq OHLC、没有 `adj_factor`（akshare 等） | 同 A1，多一次重叠比对 |
| B. 数据重灌 | `refresh_history(symbol, freq, data, mode="replace")` | 每个周期都有整段 qfq 数据；或供应商口径不是纯比例 | 传输 + 解析，分钟级 10000 行/标的 |

推荐：**默认 A2，能拿到因子时 A1，只有 B 的数据/口径不匹配时 B**。三条路径都必须以「重建指标」收尾（见 5.5.6）。

**关键点：框架只需要「比例」，不需要用户知道绝对因子。** A2 的输入就是用户每天本来就会重新拉的前复权序列；框架拿它和历史缓冲里同一 timestamp 的旧值做比值，反推出这次要乘的系数。**日线 qfq 足够推出比例，再把这个比例应用到分钟/tick/周/月序列——不需要拉分钟级的前复权数据。**

**A2 与 B 的分工**：A2 只买「比例」，然后把所有序列一起重基（含 `WindowAggregator` 的在形成窗口），因此不存在 B 的「只重灌 1min、日/周/月仍是旧基准」问题；B 用于供应商返回的不是纯比例缩放（例如同时改了历史成交量、或换了复权算法）的场景。

**分钟级也支持 B**：`refresh_history(symbol, freq="1min"/"bar", data=...)` 直接替换 base 序列（10000 行/标的），但盘中传输/解析不便宜，所以优先 A2/A1。`refresh_history` 与 F2 的 `inject_history` 分工：前者是**替换任意序列**（含 base）并重建指标，用于复权；后者是**追加派生周期 seed**，不接受 base。

**B 的覆盖完整性**：只重灌 `1min` 不会修正 `1d` / `1w` / `1mon`——它们是从旧基准聚合出来的。`refresh_history` 要么要求调用方把所有用到的周期都给全，要么接受 `factor=` 并在重灌 base 的同时自动重基派生周期；后者更省事，建议默认。
- 还要处理 `WindowAggregator` 的**在形成周/月窗口**：它持有的是旧基准的 OHLCV，闭合时会产出一根混基准的窗口 bar。重灌 base 时必须用同一因子重基这些状态（或从新 base 重建），否则周/月序列会悄悄错一根。

#### 5.5.4 因子/比例从哪来

**先澄清：重基只需要一个比例 `r`，不需要知道复权因子的绝对值。** 对「除权日之前的所有行」乘以同一个 `r`，就从前一锚点切到新锚点；`r` 可以由下列任一来源给出或推断。

0. **从新旧前复权数据推断（推荐，用户零额外知识）**：
   - 选除权日之前的重叠 bar（至少 1 根，建议 3~5 根），逐根算 `r_i = new_qfq_close(ts_i) / stored_close(ts_i)`；
   - 取中位数作为 `r`；若 `max(r_i)/min(r_i)` 超出容差（默认 `1e-3`，qfq 常见 2 位小数舍入；低价股可放宽到 `1e-2`，更稳的做法是按 `tick_size` 判定残差），说明重叠区跨了多个新除权日或供应商改了复权口径 → 按时间分段，或 `strict=True` 报错、退回 B；
   - 若只有 1 根重叠 → 接受 + WARNING；无重叠 → 报错，要求显式 factor 或 B；
   - **是否「无复权」要分两种**：显式传了 `ex_date`/事件时以事件为准，`r` 即使接近 1 也应用（用 float epsilon 判断供应商是否还没更新，未更新则 `strict=True` 报错）；未传 `ex_date` 时才用「整段比值无显著变化」判定跳过，避免每天空重建；
   - **分辨力限制**：两位小数 qfq 对低价股的小额分红分辨力有限（3 元股 0.01 元 ≈ 3.3e-3，与舍入同量级）。这种场景优先用显式因子/事件，或接受 `r` 的估计误差（指标上表现为一个小跳变）；
   - 为什么对：缓冲里存的是「上一锚点的前复权价」，新 qfq 是「新锚点的前复权价」；同一 timestamp 的比值就是两锚点之间的缩放系数，对**所有早于最新除权日的行**都成立（更早的历史除权日在两个锚点里都已被包含，比值里自然抵消）。若缓冲还是 raw（从未复权过），比值就是累计前复权因子，同样直接可用；
   - `data` 与 `reference_freq` 可以**不同周期**：按交易日对齐即可（例如用户只有 qfq 日线、策略主周期是 1min，就取「该交易日最后一根 1min close」与 qfq 日线 close 比较；不需要先 subscribe `1d`）。对齐后仍是同一个「比例」问题。
   - `ex_date` 可显式传；省略时取「比值由 `r` 变为 1 的第一个 timestamp」的前一根；盘前场景所有存量行都早于今天，整段乘 `r` 即可。
   - API：
     ```python
     report = self.apply_adjustment_from_history(
         symbol="600000.SH",
         data=df_qfq_new,        # 今天重新拉的前复权数据，日线即可
         reference_freq="1d",    # 比对序列；可与 data 不同周期（按交易日对齐），没有 1d 序列时用 "1min"
         ex_date="2024-06-14",   # 可省略，按比值变化推断
         old_data=None,          # 可选：缓冲里没有参考序列时，显式给旧快照
         apply_to="all",         # base + tick + 所有窗口 + 在形成窗口
         strict=True,
     )  # 返回 AdjustReport: ratio / overlap_rows / applied_rows / rebuilt_indicators
     ```
   - 例：缓冲里 06-13 收盘 10.00，新 qfq 日线同日收盘 9.88 → `r=0.988`；06-14 除权，则 06-14 之前的所有 1min/日/周价格乘 0.988，06-14 的 raw bar 不动。
1. **显式因子**（最稳）：`factor` = 除权日之前所有价格要乘的系数。适合券商/数据商直接给因子、或用户自己算得出来的场景。
2. **由 `CorporateAction` 推导**（便利）：
   - Split：`1 / ratio`（现有 `CorporateAction.value` 是拆股比例，`src/model/corporate_action.rs:69`）；
   - 现金分红：`(prev_close - dividend_per_share) / prev_close`，`prev_close` 取该标的历史序列最后一行（重基前它仍锚定 raw 前收）；
   - 送转/配股/复杂组合：**必须显式给 factor**，框架不猜。
3. **回测从 `adj_factor` 列推导**：若输入含累计复权因子 `adj_factor`，相邻因子变化 `f_prev -> f_new` 对应前复权重基比例 `f_prev / f_new`；在该 ex-date 第一根 bar 之前应用（见 5.5.7）。
4. **实盘 `corporate_action_provider`**：每个交易日盘前返回 `[{symbol, ex_date, factor}]`，由 runner 统一应用。

**用户数据可得性对照**：

| 用户手里有什么 | 推荐 | 需要知道因子？ |
| --- | --- | --- |
| 只有 qfq 日线（akshare `adjust="qfq"`） | A2：日线 qfq 推断 + 全序列重基 | 否 |
| 有 qfq 分钟线、没有日线 | A2 用 `reference_freq="1min"` | 否 |
| 有 `adj_factor` | A1 直接用，或回测 `adjust_mode="point_in_time"` | 不需要手算（数据里已有） |
| 有除权除息事件（简单拆分/分红） | C：框架按公式推导 | 否 |
| 只有 raw、没有任何复权数据 | 无法凭空推断；先补 qfq/`adj_factor`/事件，或接受不复权 | — |

#### 5.5.5 时机、顺序与幂等

- **唯一安全窗口是 ex-date 第一根 bar 之前**（盘前）。此时历史序列与所有在形成窗口（周/月窗口可能跨日）都只含 ex-date 之前的数据，可以整段按同一因子缩放；ex-date 的集合竞价/首根 bar 是 raw，不能被缩放。
- 推荐顺序（实盘）：`corporate_action_provider` → `apply_adjustment_from_history`（推荐）/ `apply_price_adjustment`（有显式因子）/ `refresh_history`（只有整段数据）→ 重建指标 → `on_corporate_action`（可选）→ `on_pre_open`。
- 与现有 `CorporateActionManager` 的关系：它在每日 `open_trading_day`（`src/pipeline/stages/data.rs:990`）处理持仓/现金，时机在第一根 bar 之前、`on_pre_open` 之后；价格重基在盘前，两者互不冲突。**不要**让同一 split 既在价格序列重基、又在 `TradeTracker` 里对历史价格做第二次缩放——tracker 只读 raw 成交/持仓，不读历史缓冲。
- **幂等**：维护 `applied_adjustments: HashSet<(symbol, ex_date)>`（随 checkpoint 落盘）。同一 `(symbol, ex_date)` 只应用一次；自动 provider 与用户手动刷新同时存在时，第二次只做指标重建/校验，不重复缩放。
- **自动与手动不要同时开**：如果配了 `corporate_action_provider` 又想在 `on_pre_open` 手动 `refresh_history`，两者会对同一 ex-date 各做一次。手动路径请传 `ex_date=` 登记 ledger，或直接不配 provider；不要让框架去猜「这次重灌是不是已经包含复权」。
- **盘中发现**：`strict=True` 拒绝并 CRITICAL 告警（「请在下一盘前刷新」）；`strict=False` 延迟到下一盘前。绝不允许带内部分 ex-date 数据的状态去缩放，否则会把今天的价格也乘进去。

#### 5.5.6 指标重建（本方案最硬的部分）

**为什么必须做**：`get_history` 修好只解决「手动取数」；`self.I()` 是有状态增量指标（`python/akquant/indicator_declaration.py:166`），它的内部均值/EMA/Wilder 状态还是旧基准，不重建就继续算错，而且不报错。

**机制**：

1. `I()` 声明时捕获一份 **pristine 副本**：内置 Rust 指标用 `clone_indicator`（`src/indicators.rs:74`），Python 自定义指标用 `copy.deepcopy`（与 `peek_indicator` 同一套兜底，`python/akquant/indicator_declaration.py:40`）。克隆失败不阻塞声明，只把该指标标记为 `rebuild_supported=False`。**热启动恢复路径也要捕获**（`I()` 的 `is_restored` early-return 分支），否则复权后没有可用的 pristine。有 `factory=` 时直接再调一次 `factory()`，不需要 pristine。
2. 重建时：`instances[symbol] = clone(pristine)`（或 `factory()`），清空该 symbol 的 `history`（`ind[n]` 回溯缓冲）、`provisional` 与 `provisional_label`（intrabar 临时状态），按周期从**重基后的历史序列**回放：
   - `freq=None`：base bar 序列；纯 tick 会话回放 tick 序列；
   - `freq="1d"` / `"1w"` / `"1mon"`：对应窗口序列。
   - 输入口径复用 `_build_incremental_indicator_args`（`python/akquant/strategy.py:2437` 的 bootstrap 同款）。
3. **不触发用户回调、不重置 warmup 计数**（`_symbol_bar_counts` 保持），与 `preload_history` 的回放语义一致。
4. 回放长度：默认整个保留序列（`history_depth`），实现简单、数值稳定；为控制成本也可只回放最后 `max(item.warmup_bars, 指标推断预热, lookback)` 行。若保留序列不足 `max(...)`，WARNING 点名「重建后指标可能仍有偏差」，并提示调大 `set_history_depth`。
5. 重基的 Rust 侧细节：对所有周期的在形成窗口状态按同一因子缩放（盘前无当日 bar，可整段缩放）；把被改序列的 `last_push` 置空，避免 `previous_view` 返回「半重基」的视图。
6. 预计算指标（`PrecomputedIndicator`）：实盘本来不支持；回测 PIT 模式下先直接报错，要求改用增量指标（按 ex-date 重算整段太贵，且与「点时」语义纠缠）。
7. 不可克隆指标：盘前重建时 fail-fast（`StrategyConfigurationError`），文案提示「该指标无法克隆，请用 `I(factory=...)` 声明」；`strict=False` 只告警会让策略带着错误状态交易，不建议。
8. 已上报的绘图点不重写：指标线在复权日会有一次跳变，这是前复权的固有现象，文档要说明。

**为什么不「按比例缩放指标状态」**：只有线性滤波器（SMA/EMA/BOLL/ATR 等）的状态值可按因子缩放；RSI（比值不变）、成交量类、路径依赖/非线性指标、以及 `ind[n]` 回溯缓冲都不能用一个统一系数修正。逐个指标实现 `scale()` 的表面成本远高于「克隆 + 回放」，而且容易漏。重建是唯一通用且可验证的做法。

**测试要求**：遍历所有内置指标类型，断言 `clone_indicator` 可克隆（防止新指标漏登记白名单）；对 SMA/EMA/RSI/MACD/ATR 做「重基后重建 == 直接用重基数据从头算」的等价性测试。

#### 5.5.7 回测：点时复权，避免未来函数

- 现状：用户传前复权日线/分钟线，过去价格用了未来才发生的复权因子——绝对价位类信号（价格阈值、固定止损价、BOLL 上下轨）存在前视偏差；收益率/交叉类信号通常不受影响。
- 新增 `adjust_mode`：
  - `"pre_adjusted"`（默认，保持现状）：输入即最终前复权，框架不重基。文档标注前视风险。
  - `"point_in_time"`：输入为 raw + `adj_factor`（或显式 `corporate_actions=`），引擎在 ex-date 第一根 bar 前按 `f_prev / f_new` 重基历史与在形成窗口、重建指标。没有未来因子，回测与实盘同一条代码路径。
- PIT 模式要求 `adj_factor` 覆盖全部 bar；缺失时 fail-fast（不要静默按 1 处理，否则等于没复权）。
- PIT 与预计算指标互斥（见 5.5.6）。
- 空头/持仓成本：PIT 重基只动历史缓冲，不动 `Portfolio`；split 的数量/成本仍由现有 `CorporateActionManager` 处理。

#### 5.5.8 checkpoint / 热启动

- 快照新增 `applied_adjustments: Vec<(symbol, ex_date, factor)>`（serde default；旧存档视为空）。
- 恢复后：若 provider 报出的因子比存档新，按新因子重基并重建指标；否则沿用存档值。
- 指标状态随 checkpoint 保存，但**重基后必须重建**，不能只恢复旧状态；重建在恢复完成、`on_start` 之后、首根 bar 之前进行。
- 旧存档没有 `applied_adjustments`：按空处理，允许首次 provider 应用；不会有双重复权。

#### 5.5.9 复杂度评估

| 子项 | 内容 | 预估 |
| --- | --- | --- |
| R1 | `HistoryBuffer::rebase_prices` + 在形成窗口重基 + tick 序列 | 2~3 天 |
| R2 | 指标 pristine 克隆 + 重建回放 + 回溯/临时值清理 + 不可克隆处理 | 4~6 天 |
| R3 | `apply_price_adjustment` / `apply_adjustment_from_history` / `refresh_history(base)` / 幂等 ledger | 2~3 天 |
| R4 | 从新旧 qfq 推断比例：重叠比对、容差、分段/回退、无变化跳过 | 1~2 天 |
| B | 自动 provider + 盘前调度 + `on_corporate_action` + 多 slot | 3~4 天 |
| C | 回测 PIT：`adj_factor` 通路 + 因子变化检测 + `adjust_mode` | 3~5 天 |
| D | 文档/示例/测试/CHANGELOG | 2~3 天 |

- **最小可用（R1+R2+R3+R4，手动盘前调用，含 qfq 推断）**：约 2 周。
- **完整（R1~R4+B+C+D）**：约 3~4 周。
- 风险排序：指标重建（最高，取决于指标克隆覆盖）> 因子/比例口径（中，分红/送转公式与 qfq 舍入因数据源而异）> PIT 数据通路（中）> 调度/幂等（低）。
- 若只做「重灌历史、不重建指标」，工作量减半，但**不解决问题**——`self.I()` 仍是旧基准，不建议作为交付形态。

#### 5.5.10 限制与开放问题

- 复杂送转/配股/换股：必须外部给因子，框架不推导。
- 推断比例的容差：低价股/两位小数 qfq 的相对误差可能到 `1e-2`；建议按 `tick_size` 判定残差，并允许 `tolerance=` 覆盖。
- 推断的边界：重叠区跨多个新除权日时，单比例失效；需要分段或 `strict=True` 退回 B。
- 不可克隆的指标：需 `factory=` 或改用可克隆指标。
- tick 驱动指标：tick 缓冲只有有限笔数，重建深度可能不足；建议 bar 驱动。
- 盘中复权：禁止（只能盘前）；盘中发现延迟到下一盘前。
- 与 `adj_close` extra 的关系：重基时是否也缩放 `adj_close` / `adj_factor` extra？建议 `factor` 重基时把 `adj_factor` 归一为 1、`adj_close` 按同为价格列缩放（可配 `price_extras=["adj_close"]`）；否则用户两套口径会打架。
- 迁移：已有用 `field="adj_close"` 做信号的策略，启用 F4 后应改用 `field="close"`（缓冲已是当前锚点前复权），或显式把 `adj_close` 列入 `price_extras`；否则两套口径会各算各的。
- 成交量：split 建议 `volume_factor=ratio`（保持成交额一致），dividend 为 1；是否默认缩放成交量需要决策。
- 多策略/slot：历史上共享一份缓冲；重基是全局的，指标重建按策略各自进行。若某策略不允许重建，整个盘前刷新 fail-fast，而不是只让一个策略带错误状态。
- 是否默认开启自动 provider：建议默认关闭，显式 `adjust_mode="point_in_time"` / `corporate_action_provider=` 才启用，避免改变既有回测。

## 6. API 一览

| API | 位置 | 语义 |
| --- | --- | --- |
| `subscribe_bars("1w")` / `("1M")`（内部 `1mon`） | `Strategy` | 新增周期；聚合 + 回调 + 指标 |
| `get_history(..., freq="1w")` | `Strategy` | 读注入/聚合后的同一条序列 |
| `set_history_depth(n, freq=None)` | `Strategy` | 按周期设置保留容量；`None` = 基础 |
| `run_backtest(period_history=, history_depth=int\|dict)` | 入口 | 启动 seed + 容量 |
| `run_live(period_history=, history_providers=)` | 入口 | 启动 seed / 自动供数 |
| `ctx.inject_history(...)` / `Strategy.inject_history(...)` | 运行期 | `replace`/`upsert` 写同一条序列 |
| `ctx.apply_price_adjustment(...)` / `Strategy.apply_price_adjustment(...)` | 运行期 | 按因子重基历史 + 重建指标 |
| `ctx.apply_adjustment_from_history(...)` / `Strategy.apply_adjustment_from_history(...)` | 运行期 | 从新旧 qfq 数据推断比例 → 重基 + 重建指标 |
| `ctx.refresh_history(..., freq="bar"/"1d"/...)` / `Strategy.refresh_history(...)` | 运行期 | 用整段数据替换序列 + 重建指标 |
| `run_live(corporate_action_provider=)` / `run_backtest(adjust_mode=)` | 入口 | 自动复权刷新 / 点时复权 |
| `on_corporate_action(event)` | 回调 | 重基与重建之后、`on_pre_open` 之前 |
| `current_window(symbol, "1w")` | 运行期 | 已订阅 → 在形成窗口；只注入未订阅 → `ValueError`（无在形成窗口） |
| `InjectReport` / `RefreshReport` | 返回 | written/replaced/kept/skipped/revisions/first/last/series_len |

## 7. 失败模式与处理

| 场景 | 行为 |
| --- | --- |
| 注入未来 bar | `strict=True` 抛 `ValueError`；`False` 丢弃 + 告警 |
| 同一批内重复 timestamp | 抛 `ValueError`（数据错误，不猜） |
| 时间戳非递增 | 排序 + 一次性 WARNING（可配置为报错） |
| 缺 OHLCV 列 | `ValueError`，点名缺失列与别名 |
| `freq` 不可解析 / 不高于基础周期 | `ValueError` |
| 用 `inject_history` 注入基础周期 | `ValueError`，提示启动用 `preload_history`、复权用 F4 的 `refresh_history` |
| 运行中 `Replace` 已订阅序列 | `ValueError`，提示 `Upsert` |
| `default_capacity == 0` 时注入 | `ValueError`（先 `set_history_depth` / `configure_history_depth`） |
| 注入后被 `set_history_depth(小值)` | 注入序列 pinned，不被裁剪 |
| `replace` 丢掉引擎已追加行 | WARNING，报告被丢弃行数 |
| 同 timestamp 值变化 | revision 计数 + 汇总 WARNING |
| 未订阅却用 `self.I(freq=)` | 现有报错，提示 `subscribe_bars` |
| checkpoint 恢复后旧数据 | 提示「已恢复注入序列；如需更新请重新 inject」 |
| qfq 推断：无重叠 bar | `ValueError`，要求显式 factor 或 `refresh_history` |
| qfq 推断：重叠区比值不一致 | 按时间分段；`strict=True` 报错并退回 B |
| qfq 推断：`r ≈ 1`（无复权） | 跳过重基与指标重建，不产生每日空重建 |
| 复权日盘中才刷新 | `strict=True` 拒绝；`False` 延迟到下一盘前 + CRITICAL |
| 同一 `(symbol, ex_date)` 重复刷新 | 幂等 ledger，跳过缩放，只重建/校验 |
| 指标不可克隆 | `StrategyConfigurationError`，提示 `factory=` |
| PIT 缺 `adj_factor` | fail-fast，不静默按 1 处理 |
| PIT + 预计算指标 | 报错，提示改用增量指标 |
| 重建历史不足 | WARNING，点名指标与所需长度 |

## 8. 兼容性与迁移

- **不订阅、不注入**：行为逐字节不变（新增分支全部在 opt-in 路径上）。
- **已有窗口订阅**：日线序列容量从「基础深度」变为「独立上限」，`get_history(freq='1d')` 可能从 NaN 变为有值——行为变更，需 CHANGELOG（行为变更段）+ golden 基线重签 + 文档「保留长度」一节更新。
- `1M` / `1mon` 之前是非法输入，现在是合法周期；无破坏；`bar.freq` 对外呈现为小写 `1mon`。
- `set_history_depth` 新增 `freq=` 关键字参数，默认 `None`，向后兼容。
- `run_backtest(history_depth=)` 从 `int` 扩为 `int | dict`，向后兼容。
- `period_history` / `history_providers` 为新增参数，默认 `None`。
- 仓库若仍使用 `__engine_rule_version__` 标记行为变更，本次容量变更需要相应 bump；以落地时的仓库口径为准（当前代码树中未搜到该属性定义）。
- `adjust_mode` 默认 `"pre_adjusted"`，不传 `corporate_action_provider` / `refresh_history` 时行为完全不变。
- 指标声明新增 pristine 克隆：每条声明多一份指标副本（启动时一次性），行为不变；克隆失败的指标标记 `rebuild_supported=False`，只在启用复权刷新时才报错。
- checkpoint 新增 `applied_adjustments`（serde default），旧存档按空处理。
- 启用复权刷新前需确认所有用到的内置指标都在 `clone_indicator` 白名单里（见测试计划）。

## 9. 测试计划

### Rust

- `WindowFreq::parse`：`1w`/`1M`/`1mon` 通过且后两者归一化为 `1mon`；`1m`/`2w`/`3M`/`1month` 报错；边界只做一次 `M→mon`。
- 周线标签：跨年 ISO 周（2024-12-30 属 2025-W01）、长假缩短周、月末最后交易日。
- `close_through_day`：周五收盘闭合周线；周四为最后交易日时周四闭合；日历末尾不丢。
- 部分周/月 `flush()`：标签 = 最后一根 bar 的收盘。
- 容量：日/周/月序列不被 `set_capacity_preserve_existing(100)` 裁；`ensure_capacity(n, freq)` 只长目标序列。
- 注入：`Replace`/`Upsert`、去重、revision、cutoff、`last_push = None`、pinned 容量、快照往返（含 `injected` 标记）。
- 内存 smoke：5000 标的 × 日线序列不预分配（可用 `VecDeque::capacity()` 断言）。
- 复权重基：只缩 `< ex_date` 的行；在形成周/月窗口整体缩放；tick 序列缩放；`volume_factor` 生效；同一 `(symbol, ex_date)` 幂等。
- qfq 推断：单次除权、2 位小数舍入、低价股、`r≈1` 无变化、多个除权日分段、无重叠报错、`old_data=` 显式快照。
- qfq 推断 + 指标重建：用日线 qfq 推出比例后，分钟/日/周序列与指标同时切到新基准。
- `WindowAggregator` 在形成窗口重基后与「直接用重基数据重算」逐值一致。

### Python

- `subscribe_bars("1w")` + `get_history(freq="1w")` 回测端到端。
- lookahead：周五 `on_pre_open` 注入标签为 `2024-01-05 00:00` 的周线，`on_bar` 在 `09:31` 不应看到它（`get_history` 返回上一周）。
- 幂等：同一份数据连续 `upsert` 3 次，序列长度与值不变。
- 容量：注入 5000 根周线后 `set_history_depth(100)`，数据仍在；`get_history(2000, freq="1w")` 只长周线序列。
- 指标连续：`I(SMA(20), freq="1w")` 在注入后与下一根聚合周线上连续，不跳变、不停更。
- checkpoint：save → load → 注入序列仍在；再 `upsert` 不重复。
- 多策略/slot：A 注入，B 的 `get_history` 可见。
- 未订阅注入：`get_history` 可用；`I(freq=)` 报错文案指向 `subscribe_bars`。
- 旧存档（无 `injected` 字段）反序列化 + 运行。
- 指标重建等价性：SMA/EMA/RSI/MACD/ATR 在「重基后重建」与「用重基数据从头算」下逐值一致。
- 指标克隆覆盖：遍历全部内置指标类型，`clone_indicator` 不得抛 `TypeError`。
- 复权 lookahead：合成一次拆股，PIT 模式在 ex-date 前不得出现未来的复权因子；pre_adjusted 模式保持现状。
- 自动 provider + 手动刷新同时存在时只应用一次（ledger）。
- checkpoint 恢复后 provider 报新因子 → 重基 + 重建，不与旧状态叠加。

### Golden / 回归

- 现有无窗口订阅的基线不变。
- 新增周/月基线（含节假日、跨年）。
- `cargo clippy` / `cargo test`；Python `pytest tests/test_window_*.py` 与全量。

## 10. 分阶段落地

| 阶段 | 内容 | 主要文件 | 预估 |
| --- | --- | --- | --- |
| **P0** | 周/月解析、分组、闭合、快照兼容；按周期容量；`set_history_depth(freq=)`；惰性分配 | `src/data/window.rs`、`src/history.rs`、`src/calendar/resolver.rs`、`python/akquant/strategy_window.py`、`python/akquant/strategy_history.py` | 2~3 天 |
| **P1** | `inject_window` + 标签归一化 + upsert/revision + pinned + 指标回放 + checkpoint provenance | `src/history.rs`、`src/context.rs`、`src/engine/python.rs`、`python/akquant/strategy_history.py`、`python/akquant/strategy.py` | 3~4 天 |
| **P2** | `period_history` / `history_providers` 入口；文档（`multi_timeframe_feed_api.md`、`data.md`、`warm_start.md`）；示例；CHANGELOG | `python/akquant/backtest/engine.py`、`python/akquant/live/_runner.py`、`docs/zh|en/...`、`examples/` | 2~3 天 |
| **P3** | 复权刷新：`rebase_prices` + 指标 pristine/重建 + `apply_adjustment_from_history`（qfq 推断）/ `apply_price_adjustment` / `refresh_history(base)`；自动 provider 与回测 PIT 可拆成 P3a/P3b | `src/history.rs`、`src/data/window.rs`、`src/pipeline/stages/data.rs`、`python/akquant/indicator_declaration.py`、`python/akquant/strategy.py` | 最小约 2 周；完整 3~4 周 |

P0 单独就能解决回测与实盘「往后」的周/月能力，且不引入双源问题；建议先合 P0。复权（P3）依赖 P1 的注入/指标基础设施，建议排在 P1 之后。

## 11. 完成标准（DoD）

- 回测与 replay/live 都能 `subscribe_bars("1w")` / `("1M")` 并端到端运行，`get_history(freq="1w")` 标签 = 最后一个交易日收盘。
- 基础深度 100 + 日线 2000 根可同时成立（`set_history_depth(100)` + `set_history_depth(2000, freq="1d")`）。
- 同一份周线连续 `inject_history(mode="upsert")` 3 次，序列长度与值不变。
- 周五 `on_pre_open` 注入标签为「周五 00:00」的周线，周五 `09:31` 的 `get_history` 不包含当周。
- 注入 5000 根周线后调用 `set_history_depth(100)`，注入序列仍在。
- checkpoint save/load 后注入序列仍在，重复注入不产生重复行；指标在注入后与下一根聚合周线连续。
- 只注入未订阅时 `get_history` 可用、`self.I(freq=)` 报错文案指向 `subscribe_bars`。
- 文档（`multi_timeframe_feed_api.md` / `data.md` / `warm_start.md` / API 参考）+ CHANGELOG 行为变更段 + 至少一个可运行示例。
- 复权刷新后 `get_history` 与 `self.I()` 都切到新基准；重建结果与「直接用重基数据从头算」一致。
- 只用 akshare 的 qfq 日线（无 `adj_factor`）也能完成分钟级复权刷新：不需要用户知道因子，不需要拉分钟级 qfq 数据。
- 同一 ex-date 重复刷新幂等；盘中刷新被拒绝并给出下一盘前的指引。
- 回测 PIT 模式与实盘复权路径复用同一套 rebase/rebuild 代码。

## 12. 开放问题（需决策）

1. **标签拼写**：建议对外接受 `1M` 与 `1mon`，内部统一 `1mon`（见 5.1.1）。如果你更希望内部也保留 `1M`，需要额外审计全仓的 `.lower()` 调用，成本更高。
2. **未订阅的注入周期是否允许 `get_history(freq=)`**？建议允许（这是注入的核心价值），但 `I(freq=)` 仍要求订阅。
3. **注入是否派发 `on_window_bar`**？建议不派发，只回放指标（避免启动时批量触发交易）。
4. **`current_window` 对只注入未订阅的周期**：建议抛错并提示 `subscribe_bars`（返回 `None` 会与「已订阅但当前无形成窗口」混淆）。
5. **provenance 存哪**：每序列 `injected: bool` vs 快照级 `Vec<(symbol, freq)>`。建议每序列字段（裁剪逻辑就地可用）。
6. **周/月口径**：ISO 周 vs 交易所周；自然月 vs 交易日历月。建议 ISO 周 + 自然月，与大多数数据源一致；期货若不一致，用 `session_windows` 无法表达周/月，需要单独约定。
7. **是否支持 `1q`/`1y`**：建议本次不做，留扩展点。
8. **`replace` 运行期是否完全禁止**：建议对已订阅序列禁止；未订阅序列允许，便于「一次性换源」。
9. **多策略隔离**：建议共享 + 文档化；如需隔离，后续加 `owner`。
10. **provider 签名**：建议 `(symbols, now_ns) -> dict[str, DataFrame]`；是否需要 `(start, end)` 由 provider 自行决定（框架只保证传入当前时刻）。
11. **复权相关**：因子口径（现金分红是否用税前/税后、送转/配股是否推导）、成交量是否默认随 split 缩放、`adj_close` extra 是否一起重基、默认是否启用自动 provider —— 详见 5.5.10。
12. **qfq 推断的默认容差**：按固定相对误差（`1e-3`）还是按 `tick_size` 残差？低价股/两位小数数据差异较大，建议后者，但需要 `tick_size` 可得；同时要接受「低价股小额分红分辨不出」这一数据限制，并在文档里给出改走显式因子/事件的建议。

## 附录 A：关键代码位置

| 主题 | 位置 |
| --- | --- |
| 周期解析 | `src/data/window.rs:44`、`python/akquant/strategy_window.py:24`、`:40` |
| 窗口聚合 | `src/data/window.rs:430`（configure）、`:717`（update）、`:836`（close_through_day） |
| 窗口写历史 | `src/pipeline/stages/data.rs:717` |
| 历史缓冲 | `src/history.rs:8`（SymbolHistory）、`:127`（new）、`:343`（trim）、`:370`（set_capacity）、`:383`（ensure）、`:399`（preserve）、`:472`（update_window） |
| 历史读取 / cutoff | `src/context.rs:760`（resolve）、`:1034`（history）、`:1115`（history_multi） |
| Python 历史入口 | `python/akquant/strategy_history.py:14`（AUTO_GROW_LIMIT）、`:42`（_infer_freq，校验在 `:79`）、`:117`（_ensure_history_depth）、`:242`（get_history） |
| 指标驱动 | `python/akquant/strategy.py:2037`、`python/akquant/strategy_window.py:301` |
| 生命周期 / cutoff | `python/akquant/strategy_framework_hooks.py:215`、`:398` |
| 预热 | `src/engine/python.rs:1012`、`python/akquant/live/_preload.py:265`、`python/akquant/live/_runner.py:1235` |
| 深度解析 | `python/akquant/backtest/engine.py:3089`、`:5028`、`:5808` |
| 公司行为（持仓/现金） | `src/market/corporate_action.rs:1`、`src/pipeline/stages/data.rs:990`、`src/analysis/tracker.rs:346` |
| 公司行为模型 / 注册 | `src/model/corporate_action.rs:69`、`src/engine/python.rs:1350` |
| 指标克隆 / 声明 | `src/indicators.rs:74`、`python/akquant/indicator_declaration.py:40`、`:166` |
| 指标 bootstrap | `python/akquant/strategy.py:2437` |

## 附录 B：容量与内存估算

- 每标的每序列每行 ≈ 6 × 8B = 48B（不含 extras）。
- 1min、A 股 240 根/天：10000 根 ≈ 41.7 个交易日。
- 10000 根基础 1min = 480KB/标的；5000 标的 = 2.4GB。
- 日线 4000 行 = 192KB/标的；周线 1200 = 57.6KB；月线 400 = 19.2KB。
- 关键：`SymbolHistory::new` 会按 capacity 预分配（`src/history.rs:127`），所以低频上限必须配合惰性分配，否则「上限」直接变成「常驻内存」。

## 附录 C：与现有三条多周期路径的关系

| 路径 | 本设计后 |
| --- | --- |
| `subscribe_bars` | 新增 `1w`/`1mon`（对外可写 `1M`）；仍是回调与实时聚合的推荐路径 |
| `BarGenerator` | 不变；不接历史/指标（既有取舍） |
| `feed.resample` / `feed.replay` | 仍是离线多 feed 编排；注入取代「伪标的拼历史」的常见用法，但不取代自定义时间戳场景 |
| `preload_history` | 仍只管基础周期；低频走 `period_history` + `inject_history` |
