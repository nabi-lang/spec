# 3.3. `time::`：时间

时间按需求分布在两个模块中。`core::time` 存放 `Duration` 和 `Instant`：单调时钟本来就在 `core` 中运行（任务计时器，2.18），所以这两者在所有目标上都可用，也不需要 `use`（0.2）。`std::time` 存放需要操作系统的时钟：挂钟 `SystemTime`、启动时钟 `BootTime` 和 `UNIX_EPOCH`。它还存放民用日历：`Date`、`Time`、`DateTime`、`Period`、`Offset`、`TimeZone` 和 `Zoned`（3.3.6 至 3.3.8），因为时区来自操作系统的 tz 数据库。每个类型只有一条路径（[0.1.6](../000-meta/001-tier-model.md#016-每个项只有一条路径)）；`std::time` 不重新导出 core 的类型。单调时钟、挂钟和自启动以来经过的时间被明确区分，因此可以选择适合所做测量的时钟。

## 3.3.1. 四类型模型

整体情况如下：

| 类型         | 模块              | 含义                        | 用途                     |
|--------------|-------------------|-----------------------------|--------------------------|
| `Instant`    | `core::time` | 单调时钟                    | 测量时长 / 超时          |
| `Duration`   | `core::time` | 时间差（i64 纳秒）          | 两个时间之差 / sleep 参数 |
| `SystemTime` | `std::time`  | 挂钟（UTC，可被 NTP 调整）  | 文件 mtime / 向用户显示  |
| `BootTime`   | `std::time`  | 自启动以来的时间（含挂起）  | 移动 / 笔记本应用        |

`Duration` 的内部表示是 `i64` 纳秒。Rust 采用的 `u128` 被认为过度；`i64` 已经覆盖约 ±292 年，对实际使用绰绰有余。

**所有时钟共用一条规则。** 任何时钟的两次读数之间的跨度都是精确的有符号 `Duration`：`b` 较晚时 `a.duration_since(b)` 为负，永远不会是错误、`none` 或零。`elapsed()` 在单调时钟上测量，因此挂钟的跳变不会让它变成负数。这是 Go 的规则（`time.Sub`、`time.Since`）。

## 3.3.2. `Instant`：单调时钟

```nabi
// core::time
struct Instant { /* opaque (CLOCK_MONOTONIC / QueryPerformanceCounter) */ }

impl Instant {
    fn now() -> Instant;

    fn elapsed(self) -> Duration;
    const fn duration_since(self, earlier: Instant) -> Duration;   // self - earlier, signed

    // arithmetic: returns none on failure (avoids Rust's silent saturating-on-overflow)
    const fn checked_add(self, dur: Duration) -> Instant | none;
    const fn checked_sub(self, dur: Duration) -> Instant | none;
}

// ordinary arithmetic; on overflow, panic with overflow checks on, saturate with them off (3.3.5)
impl Add<Duration> for Instant { type Output = Instant; ... }
impl Sub<Instant> for Instant { type Output = Duration; ... }
impl Sub<Duration> for Instant { type Output = Instant; ... }
```

`Instant` 的关键性质：它永不倒退，因此不受 NTP 调整的影响。它本身没有绝对时间的含义（参考点是进程启动），因此**不能序列化**。出于同样的原因，跨进程比较没有意义；它只在同一进程内有意义。

当 `earlier` 较晚时，`duration_since` 返回负的 `Duration`。要把跨度限制为非负，就进行比较：`d.max(Duration::ZERO)`。

### 示例

最常见的模式是取一个起点，再用 `elapsed` 得到经过的时间。

```nabi
let start = Instant::now();
heavy_computation();
let elapsed = start.elapsed();          // Duration
log(f"computation took {elapsed:?}");
```

## 3.3.3. `SystemTime`：挂钟

```nabi
// std::time
struct SystemTime { /* opaque (CLOCK_REALTIME / GetSystemTimePreciseAsFileTime) */ }

impl SystemTime {
    fn now() -> SystemTime;

    const fn duration_since(self, earlier: SystemTime) -> Duration;   // self - earlier, signed
    fn elapsed(self) -> Duration;                                    // on the monotonic clock

    const fn checked_add(self, dur: Duration) -> SystemTime | none;
    const fn checked_sub(self, dur: Duration) -> SystemTime | none;
}

const UNIX_EPOCH: SystemTime;     // 1970-01-01 00:00:00 UTC

// SystemTime ↔ unix timestamp
impl SystemTime {
    const fn from_unix(secs: i64, nanos: u32) -> SystemTime;
    const fn unix_secs(self) -> i64;    // rounded down: negative before the epoch
    const fn unix_nanos(self) -> i64;   // nabi has no i128; i64 ns covers about ±292 years
    const fn subsec_nanos(self) -> u32; // the nanoseconds past unix_secs()
}

// text (3.3.8)
impl SystemTime {
    const fn to_http_date(self) -> str;                              // "Sun, 06 Nov 1994 08:49:37 GMT"
    fn parse_http_date(text: str) throws (ParseError) -> SystemTime;
}
impl Display for SystemTime { ... }     // RFC 3339 in UTC: "2023-11-14T22:13:20Z"; Debug is the same
impl FromStr for SystemTime { ... }     // RFC 3339 with any offset

impl Add<Duration> for SystemTime { type Output = SystemTime; ... }
impl Sub<Duration> for SystemTime { type Output = SystemTime; ... }
```

`SystemTime` 是相对于 `UNIX_EPOCH` 的 `i64` 纳秒，所以它在所有目标上的范围都约为 ±292 年。32 位目标通过 64 位时间接口（`time64`）读取时钟，因此不存在 2038 年的限制。

`SystemTime` 的行为截然不同。时间可能因 NTP 调整或用户修改而倒退，又因为它是相对于 UNIX_EPOCH 的绝对时间，所以它**可以序列化**。因此它可以在进程之间进行有意义的比较。

`duration_since` 是挂钟读数之间的有符号差：较晚的 `earlier` 得到负的 `Duration`，而不是错误。`unix_secs` 向下取整，因此对于亚秒部分为 `n` 的 `t`，`SystemTime::from_unix(t.unix_secs(), n)` 会还原出 `t`，在纪元之前也是如此。

**单调读数。** `SystemTime::now()` 同时也读取单调时钟（即 `Instant` 的时钟）。`elapsed()` 基于这个读数测量，因此在 `now()` 与 `elapsed()` 之间挂钟的跳变不会改变结果。与 `Duration` 进行的 `+` / `-` / `checked_*` 会同时移动两个读数。不是来自 `now()` 的 `SystemTime`（`from_unix`、`UNIX_EPOCH`、`Metadata::modified`）没有单调读数，它的 `elapsed()` 是 `SystemTime::now().duration_since(self)`。比较、相等和哈希只使用挂钟读数。

### 示例

转换为 unix 时间戳，以及读取文件的 mtime：

```nabi
use std::time::SystemTime;

let now = SystemTime::now();
log(f"unix timestamp: {now.unix_secs()}");

// file mtime
let meta = try fs::metadata("data.txt");
let mtime: SystemTime = try meta.modified();
log(f"file age: {mtime.elapsed():?}");     // negative if the mtime is in the future
```

## 3.3.4. `BootTime`：自启动以来的时间（含挂起）

```nabi
// std::time
struct BootTime { /* CLOCK_BOOTTIME (Linux) / CLOCK_MONOTONIC = mach_continuous_time (macOS) */ }

impl BootTime {
    fn now() -> BootTime;
    fn elapsed(self) -> Duration;
    const fn duration_since(self, earlier: BootTime) -> Duration;   // self - earlier, signed
}
```

与 `Instant` 一样，当 `earlier` 较晚时 `duration_since` 为负。`BootTime` 与 `Instant` 的区别在于跨越挂起时的行为。`Instant` 与 Linux 的 `CLOCK_MONOTONIC` 一样在挂起期间停止，而 `BootTime` 沿用 `CLOCK_BOOTTIME`，在挂起期间继续前进。

当移动或笔记本应用必须测量“真实经过的时间（含挂起）”时，这一区别是决定性的。截至 2026 年，Rust 仍没有用于此目的的稳定 API；Nabi 早早就把它放到了表面上。

## 3.3.5. `Duration`

```nabi
// core::time
struct Duration { /* opaque: i64 nanoseconds (about ±292 years) */ }

impl Duration {
    const ZERO: Duration;             // 0 ns
    const MAX: Duration;              // I64_MAX ns

    // constructors: any integer type that converts to i64 without loss
    // (i8 .. i64, u8 .. u32); u64 / usize need try_into
    const fn ns<N>(n: N) -> Duration where i64: From<N>;
    const fn us<N>(n: N) -> Duration where i64: From<N>;        // microseconds
    const fn ms<N>(n: N) -> Duration where i64: From<N>;        // milliseconds
    const fn secs<N>(n: N) -> Duration where i64: From<N>;
    const fn mins<N>(n: N) -> Duration where i64: From<N>;
    const fn hours<N>(n: N) -> Duration where i64: From<N>;
    const fn days<N>(n: N) -> Duration where i64: From<N>;

    const fn secs_f64(s: f64) -> Duration;  // floating point seconds, truncated to ns

    // extraction
    const fn as_ns(self) -> i64;
    const fn as_us(self) -> i64;
    const fn as_ms(self) -> i64;
    const fn as_secs(self) -> i64;
    const fn as_secs_f64(self) -> f64;

    const fn subsec_ns(self) -> u32;          // sub-second nanosecond part
    const fn subsec_us(self) -> u32;
    const fn subsec_ms(self) -> u32;
}

// arithmetic
impl Add for Duration { ... }
impl Sub for Duration { ... }
impl Mul<i64> for Duration { ... }     // Duration × i64
impl Div<i64> for Duration { ... }
impl Mul<Duration> for f64 { ... }      // f64 × Duration

impl Display for Duration { ... }       // "1.234s" / "5ms" / "1.5µs" / "100ns"
impl Debug for Duration { ... }         // same as Display
```

对两个时长调用 `a.min(b)` 就是 `Ord::min`。

`Display` 写出一个与数量级相称的单位：绝对值至少为 1 秒时用 `s`，至少为 1 ms 时用 `ms`，至少为 1 µs 时用 `µs`，否则用 `ns`。先写出除以该单位后的整数部分，再把余数写成小数部分；去掉末尾的零，余数为 0 时不写小数点。负值前加 `-`。

```nabi
Duration::ms(1234);      // "1.234s"
Duration::ms(5);         // "5ms"
Duration::ns(1500);      // "1.5µs"
Duration::ns(-20);       // "-20ns"
```

所有时间类型都实现了 `Clone` / `Copy` / `PartialEq` / `Eq` / `PartialOrd` / `Ord` / `Hash` / `Debug`。`Duration` 还实现了 `Default`（零）。

### 时长运算策略

`Duration` / `SystemTime` / `Instant` / `BootTime` 上的运算（包括 `duration_since` 和 `from_unix`）在溢出检查开启时溢出会触发 panic，关闭时则饱和（`-C overflow-checks`，[refs-lang 9.1.2](../../refs-lang/009-tools/901-nabic-cli.md#nabic-build-aot-build)）。`f64` 形式（`Duration::secs_f64`、`f64 * Duration`）遵循同样的规则：非有限值或超出范围的结果在检查开启时触发 panic，关闭时饱和（NaN 变为零）。除以零始终触发 panic。`Duration` 的 `checked_*` / `saturating_*` 变体用于显式选择行为。

```nabi
impl Duration {
    const fn checked_add(self, rhs: Duration) -> Duration | none;
    const fn checked_sub(self, rhs: Duration) -> Duration | none;
    const fn checked_mul(self, factor: i64) -> Duration | none;
    const fn checked_div(self, divisor: i64) -> Duration | none;

    const fn saturating_add(self, rhs: Duration) -> Duration;
    const fn saturating_sub(self, rhs: Duration) -> Duration;
    const fn saturating_mul(self, factor: i64) -> Duration;
}
```

## 3.3.6. 日历日期和时间

日历类型没有时区：它们就是墙上的日历和挂钟所显示的内容。它们都是 `Copy` 值，按时间排序，带有 `Hash`；`Display` / `FromStr` 采用 ISO 8601（3.3.8）。

| 类型       | 保存的内容                              | `Display`               |
|------------|-----------------------------------------|-------------------------|
| `Date`     | 年、月、日（外推格里历）                | `2024-02-29`            |
| `Time`     | 时、分、秒、纳秒                        | `09:05:00.25`           |
| `DateTime` | 一个 `Date` 和一个 `Time`               | `2024-03-10T02:30:00`   |
| `Period`   | 年、月、日，各自分开保存                | `P1Y2M10D`              |
| `Weekday`  | Monday .. Sunday                        | `Monday`                |

```nabi
// std::time
const MIN_YEAR: i32;    // -999999
const MAX_YEAR: i32;    // 999999
const fn is_leap_year(year: i32) -> bool;
const fn days_in_month(year: i32, month: i32) -> i32;   // 0 for a month outside 1..=12

enum Weekday { Monday, Tuesday, Wednesday, Thursday, Friday, Saturday, Sunday }
impl Weekday {
    const fn number(self) -> i32;                    // 1 = Monday .. 7 = Sunday (ISO 8601)
    const fn from_number(number: i32) -> Weekday | none;
    const fn name(self) -> str;                      // "Monday"
    const fn next(self) -> Weekday;
    const fn prev(self) -> Weekday;
}

struct Date { /* year, month, day */ }
impl Date {
    const MIN: Date;    // -999999-01-01
    const MAX: Date;    // +999999-12-31

    const fn new(year: i32, month: i32, day: i32) throws (ValueError) -> Date;
    const fn from_ordinal(year: i32, ordinal: i32) throws (ValueError) -> Date;
    const fn from_iso_week(year: i32, week: i32, weekday: Weekday) throws (ValueError) -> Date;
    const fn from_unix_days(days: i64) -> Date;      // days from 1970-01-01; panics outside MIN..=MAX
    const fn unix_days(self) -> i64;

    const fn year(self) -> i32;                      // 0 is 1 BC
    const fn month(self) -> i32;                     // 1..=12
    const fn day(self) -> i32;
    const fn weekday(self) -> Weekday;
    const fn ordinal(self) -> i32;                   // day of the year, 1..=366
    const fn iso_week(self) -> (i32, i32);           // (ISO year, week 1..=53)
    const fn is_leap_year(self) -> bool;
    const fn days_in_month(self) -> i32;
    const fn days_in_year(self) -> i32;

    const fn with_day(self, day: i32) throws (ValueError) -> Date;
    const fn first_of_month(self) -> Date;
    const fn last_of_month(self) -> Date;

    const fn add_days(self, n: i64) -> Date;         // panics outside MIN..=MAX
    const fn add_months(self, n: i64) -> Date;       // clamps the day
    const fn add_years(self, n: i64) -> Date;        // clamps February 29
    const fn checked_add_days(self, n: i64) -> Date | none;
    const fn checked_add_months(self, n: i64) -> Date | none;
    const fn days_since(self, earlier: Date) -> i64; // negative when earlier is later
    const fn until(self, later: Date) -> Period;
    const fn and_time(self, time: Time) -> DateTime;
}
impl Add<Period> for Date { ... }
impl Sub<Period> for Date { ... }

struct Time { /* hour, minute, second, nanosecond */ }
impl Time {
    const MIDNIGHT: Time;
    const fn new(hour: i32, minute: i32, second: i32) throws (ValueError) -> Time;
    const fn with_nanosecond(self, nano: i32) throws (ValueError) -> Time;
    const fn from_nanos_of_day(ns: i64) -> Time;     // panics outside one day
    const fn hour(self) -> i32;
    const fn minute(self) -> i32;
    const fn second(self) -> i32;
    const fn nanosecond(self) -> i32;
    const fn secs_of_day(self) -> i64;
    const fn nanos_of_day(self) -> i64;
}

struct DateTime { /* date, time */ }
impl DateTime {
    const fn new(year: i32, month: i32, day: i32, hour: i32, minute: i32, second: i32) throws (ValueError) -> DateTime;
    const fn of(date: Date, time: Time) -> DateTime;
    const fn from_unix(secs: i64, nano: u32) -> DateTime;   // the civil time of a unix time, as UTC
    const fn unix_secs(self) -> i64;                         // the reverse, reading self as UTC
    const fn date(self) -> Date;
    const fn time(self) -> Time;
    // year() .. nanosecond() as on Date and Time
    const fn add_days(self, n: i64) -> DateTime;
    const fn add_months(self, n: i64) -> DateTime;
    const fn add_years(self, n: i64) -> DateTime;
    const fn duration_since(self, earlier: DateTime) -> Duration;
    fn in_zone(self, tz: TimeZone) -> Zoned;                                     // 3.3.7
    fn in_zone_with(self, tz: TimeZone, choice: Disambiguation) throws (ValueError) -> Zoned;
}
impl Add<Duration> for DateTime { ... }   // exact time: every day is 86400 s
impl Sub<Duration> for DateTime { ... }
impl Add<Period> for DateTime { ... }     // the date moves, the time stays
impl Sub<Period> for DateTime { ... }

struct Period { /* years, months, days */ }
impl Period {
    const ZERO: Period;
    const fn new(years: i32, months: i32, days: i32) -> Period;
    const fn of_days(n: i32) -> Period;
    const fn of_months(n: i32) -> Period;
    const fn of_years(n: i32) -> Period;
    const fn years(self) -> i32;
    const fn months(self) -> i32;
    const fn days(self) -> i32;
    const fn total_months(self) -> i64;
    const fn is_zero(self) -> bool;
}
// Period + Period, Period - Period, -Period; Default is ZERO
```

**范围与历法。** `Date` 是外推格里历（proleptic Gregorian calendar）中 `MIN_YEAR` 与 `MAX_YEAR` 之间的某一天；与 ISO 8601 的计法一样，第 0 年是公元前 1 年。天数计算使用 Howard Hinnant 的民用历算法，在整个范围内都是精确的。`unix_days` 从 1970-01-01 起算，该日为第 0 天。

**字段错误会抛出，运算溢出会触发 panic。** 构造函数收到不构成日期或时间的字段时（`Date::new(2023, 2, 29)`、`Time::new(24, 0, 0)`），会抛出指出该字段及其范围的 `ValueError`：`day 29 is out of range for 2023-02 (1..=28)`。超出范围的运算（`Date::MAX.add_days(1)`）会像 `Duration` 一样触发 panic；`checked_*` 形式则返回 `none`。

**月份会被钳制。** 新的月份有原来那一天时，`add_months` 和 `add_years` 保留该日，否则取新月份的最后一天：2024-01-31 加一个月是 2024-02-29，2024-02-29 加一年是 2025-02-28。每次调用各自钳制，所以从 1 月 31 日起两次 `add_months(1)` 落在 3 月 29 日，而 `add_months(2)` 落在 3 月 31 日。`Date + Period` 先把年和月一起加上（钳制一次），再加天数。这是 java.time 和 Temporal 的规则。

**`until`** 是从 `self` 到 `later` 的日历跨度：先取整年，再取整月，然后是天，都带有跨度的符号，其选取方式保证对任意两个日期都有 `d + d.until(e) == e`。`days_since` 是单纯的天数差。

**ISO 周。** `iso_week` 是 ISO 8601 的周编号：每周从星期一开始，第 1 周包含当年的第一个星期四，因此 2021-01-03 属于 2020 年的第 53 周，2024-12-30 属于 2025 年的第 1 周。

### 示例

```nabi
use std::time::{Date, DateTime, Period, Time};

let d = try Date::new(2024, 1, 31);
println(f"{d} {d.weekday()} day {d.ordinal()}");
println(f"{d.add_months(1)} {d.add_months(1).add_months(1)} {d.add_months(2)}");
println(f"{try Date::new(2024, 2, 29).add_years(1)}");

let end = try Date::new(2025, 3, 1);
let p = d.until(end);
println(f"{p} {d + p} {end.days_since(d)} days");
println(f"{try Date::new(2021, 1, 3).iso_week():?}");

let t = try (try Time::new(9, 5, 0)).with_nanosecond(250000000);
let dt = d.and_time(t);
println(f"{dt} {dt + Duration::hours(15)} {dt + Period::of_days(1)}");
try { println(f"{try Date::new(2023, 2, 29)}"); } catch e { println(f"{e}"); }
```

```output
2024-01-31 Wednesday day 31
2024-02-29 2024-03-29 2024-03-31
2025-02-28
P1Y1M1D 2025-03-01 395 days
(2020, 53)
2024-01-31T09:05:00.250 2024-02-01T00:05:00.250 2024-02-01T09:05:00.250
day 29 is out of range for 2023-02 (1..=28)
```

## 3.3.7. 偏移、时区与 `Zoned`

`Offset` 是与 UTC 之间的固定差值。`TimeZone` 给出每个时刻的偏移。`Zoned` 是一个时刻连同显示它所用的时区。

```nabi
// std::time
struct Offset { /* seconds east of UTC, within ±25:59:59 */ }
impl Offset {
    const UTC: Offset;
    const fn new(hours: i32, minutes: i32) throws (ValueError) -> Offset;   // (-3, -30) is -03:30
    const fn from_secs(secs: i32) throws (ValueError) -> Offset;
    const fn secs(self) -> i32;
    const fn is_utc(self) -> bool;
}

struct TimeZone { /* a name and its rules; copies share the table */ }
impl TimeZone {
    const fn utc() -> TimeZone;
    const fn fixed(offset: Offset) -> TimeZone;
    fn get(name: str) throws (ValueError) -> TimeZone;                   // "Asia/Seoul", "UTC", "+09:00"
    fn local() -> TimeZone;
    fn from_tzif<B: AsBytes>(name: str, data: B) throws (ValueError) -> TimeZone;
    fn from_posix(rule: str) throws (ValueError) -> TimeZone;           // "CET-1CEST,M3.5.0,M10.5.0/3"

    const fn name(self) -> str;
    const fn is_fixed(self) -> bool;
    fn offset_at(self, t: SystemTime) -> Offset;
    fn abbreviation_at(self, t: SystemTime) -> str;                      // "KST", "EDT"
    fn is_dst_at(self, t: SystemTime) -> bool;
}

enum Disambiguation { Compatible, Earlier, Later, Reject }

struct Zoned { /* an instant, its zone, and the civil time and offset there */ }
impl Zoned {
    fn now(tz: TimeZone) -> Zoned;
    fn from_system_time(t: SystemTime, tz: TimeZone) -> Zoned;
    fn from_unix(secs: i64, nano: u32, tz: TimeZone) -> Zoned;

    const fn unix_secs(self) -> i64;
    const fn unix_nanos(self) -> i64;            // panics outside 1678..2262
    const fn to_system_time(self) -> SystemTime; // panics outside 1678..2262
    const fn datetime(self) -> DateTime;
    const fn date(self) -> Date;
    const fn time(self) -> Time;
    const fn offset(self) -> Offset;
    const fn zone(self) -> TimeZone;
    const fn abbreviation(self) -> str;
    const fn is_dst(self) -> bool;
    // year() .. nanosecond(), weekday() as on DateTime

    fn in_zone(self, tz: TimeZone) -> Zoned;     // the same instant elsewhere
    fn to_utc(self) -> Zoned;
    fn add_days(self, n: i64) -> Zoned;          // calendar days: the wall time stays
    fn add_months(self, n: i64) -> Zoned;
    fn add_years(self, n: i64) -> Zoned;
    const fn duration_since(self, earlier: Zoned) -> Duration;
}
impl Add<Duration> for Zoned { ... }   // exact time: the wall time may jump at a transition
impl Sub<Duration> for Zoned { ... }
impl Add<Period> for Zoned { ... }     // calendar time, as add_months then add_days
impl Sub<Period> for Zoned { ... }
```

**时区从哪里来。** `TimeZone::get` 以 TZif 文件（RFC 8536，版本 1 至 4）的形式读取 tz 数据库（IANA）中的时区：设置了 `$TZDIR` 时从中读取，否则从 `/usr/share/zoneinfo`、`/usr/lib/zoneinfo`、`/usr/share/lib/zoneinfo` 和 `/etc/zoneinfo` 中第一个存在的目录读取。越过文件中最后一次转换之后，由文件尾部的 POSIX `TZ` 规则给出偏移，因此遥远未来的日期遵循现行规则。`UTC`、其他 UTC 名称（`Etc/UTC`、`GMT`、`Zulu` 等）和偏移（`+09:00`）无需数据库即可使用。没有数据库时，其他任何名称都会抛出错误，错误会说明系统没有 tz 数据库：Windows 和浏览器没有 tz 数据库，WASI 程序只有在主机授予该目录时才能看到它。`from_tzif` 读取程序自己携带的 TZif 文件（例如用 `include_bytes!`），在所有目标上都可用。`from_posix` 接受 POSIX `TZ` 规则（`EST5EDT,M3.2.0,M11.1.0`；其偏移按 POSIX 的写法以西为正）。

**本地时区。** `TimeZone::local()` 先读取 `$TZ`：数据库名称（`Asia/Seoul`）、TZif 文件的路径（`/usr/share/zoneinfo/Asia/Seoul`，开头的 `:` 会被去掉）或 POSIX 规则。没有 `$TZ` 时，它读取 `/etc/localtime`；若 `/etc/timezone` 中写有时区名，就用该名称命名，否则命名为 `localtime`。这些都给不出时区时，使用 UTC。它每次调用都会读取文件，因此程序应当保存其结果。

**间隙与重叠。** 在时钟拨快的地方（间隙），某些本地时间从未出现；在时钟拨回的地方（重叠），某些本地时间出现两次。`in_zone_with` 接受一个选择，与 java.time 和 Temporal 的做法相同：

| `Disambiguation` | 在重叠中（01:30 出现两次）        | 在间隙中（跳过 02:30，02:00 直接到 03:00） |
|------------------|-----------------------------------|-------------------------------------------|
| `Compatible`     | 较早的时刻                        | 按间隙长度向后推：03:30                   |
| `Earlier`        | 较早的时刻                        | 用间隙之后的偏移解读：01:30               |
| `Later`          | 较晚的时刻                        | 用间隙之前的偏移解读：03:30               |
| `Reject`         | 抛出 `ValueError`                 | 抛出 `ValueError`                         |

`in_zone` 就是 `in_zone_with(.., Disambiguation::Compatible)`，这是大多数系统的做法，并且不会失败。`Reject` 的错误会指出时区、两个偏移以及应当改传什么：`2024-03-10T02:30:00 does not exist in America/New_York: the clocks skip it (-05:00 to -04:00); pass Disambiguation::Earlier or Disambiguation::Later`。`Zoned` 上的 `add_days`、`add_months`、`add_years` 和 `+ Period` 移动民用日期、保留一天中的时间，并按 `Compatible` 解析结果；`+ Duration` 加上精确的时间。

`is_dst_at` 和 `Zoned::is_dst` 报告时区数据中的夏令时标志。大多数时区标记的是夏季时间；采用负夏令时的时区则标记其冬季时间（`IST-1GMT0,M10.5.0,M3.5.0/1`，大多数 tz 构建中 Europe/Dublin 的规则）。

**相等。** 两个 `Zoned` 值只按其时刻判断相等、排序和哈希：`2024-01-01T09:00:00+09:00` 等于 `2024-01-01T00:00:00Z`。两个 `TimeZone` 名称相同时相等。

### 示例

```nabi
use std::time::{DateTime, Disambiguation, Offset, TimeZone};

let ny = try TimeZone::get("America/New_York");
let dt = try DateTime::new(2024, 3, 10, 2, 30, 0);
println(f"{dt.in_zone(ny)}");
println(f"{try dt.in_zone_with(ny, Disambiguation::Earlier)}");
try { println(f"{try dt.in_zone_with(ny, Disambiguation::Reject)}"); } catch e { println(f"{e}"); }

let before = (try DateTime::new(2024, 3, 9, 12, 0, 0)).in_zone(ny);
println(f"{before.add_days(1)} {before + Duration::hours(24)}");
println(f"{before.abbreviation()} {before.add_days(1).abbreviation()}");

let seoul = before.in_zone(TimeZone::fixed(try Offset::new(9, 0)));
println(f"{seoul} {seoul == before}");
```

```output
2024-03-10T03:30:00-04:00
2024-03-10T01:30:00-05:00
2024-03-10T02:30:00 does not exist in America/New_York: the clocks skip it (-05:00 to -04:00); pass Disambiguation::Earlier or Disambiguation::Later
2024-03-10T12:00:00-04:00 2024-03-10T13:00:00-04:00
EST EDT
2024-03-10T02:00:00+09:00 true
```

## 3.3.8. 文本：ISO 8601、RFC 3339 与模式

`Display` 写出、`FromStr`（`str::parse`）读取以下标准形式：

| 类型         | `Display`                               | `FromStr` 还能读取                            |
|--------------|-----------------------------------------|-----------------------------------------------|
| `Date`       | `2024-02-29`、`-000044-03-15`、`+012345-01-01` | |
| `Time`       | `09:05:00`、`09:05:00.250`              | `09:05`、1 至 9 位的小数部分、以 `,` 代替 `.` |
| `DateTime`   | `2024-03-10T02:30:00`                   | 以 `t` 或空格代替 `T`                         |
| `Offset`     | `+09:00`、`-03:30`、`+05:30:15`         | `Z`、`+0900`、`+09`                           |
| `Zoned`      | `2024-03-10T03:30:00-04:00`、`...Z`     | RFC 9557 的 `[America/New_York]` 后缀         |
| `SystemTime` | `2023-11-14T22:13:20Z`（UTC）           | 任意偏移                                      |
| `Period`     | `P1Y2M10D`、`P0D`、`P-1M-3D`            | （没有 `FromStr`）                            |

0..=9999 以外的年份带符号并写成六位数字，即 ISO 8601 的扩展形式。小数部分写成 3、6 或 9 位，取能容纳它的最少位数，为零时省略。偏移为零的 `Zoned` 写作 `Z`；它的 `Debug` 会在方括号中加上时区名（`2024-03-10T03:30:00-04:00[America/New_York]`）。读取带方括号时区的 `Zoned` 时会加载该时区；偏移必须与之一致，否则错误会同时指出两者（没有偏移时，本地时间按 `Compatible` 解析）。没有方括号时，时区就是该固定偏移。带偏移的 `DateTime` 文本是错误，错误会提示改为解析 `Zoned` 或 `SystemTime`。

**错误指出位置。** 解析错误是一个 `ParseError`，其消息指出字节偏移以及那里应有的内容：``cannot parse "2024/02/03": expected `-` after the year at byte 4, found `/` ``、`cannot parse "2024-13-01": month 13 is out of range (1..=12) at byte 5`。`position()` 就是这个偏移。

**模式。** `Date`、`Time`、`DateTime` 和 `Zoned` 上的 `format(pattern)` 写出 strftime 风格的指令；`parse_with(text, pattern)` 按同样的指令读回。名称使用英语（C locale）。

| 指令 | 含义 | 指令 | 含义 |
|-----------|---------|-----------|---------|
| `%Y` | 年，至少 4 位 | `%H` | 时 00..23 |
| `%C` | 世纪（`%Y` / 100） | `%I` | 时 01..12 |
| `%y` | 年 % 100，2 位 | `%k` / `%l` | 用空格填充的 `%H` / `%I` |
| `%m` | 月 01..12 | `%M` | 分 00..59 |
| `%d` | 日 01..31 | `%S` | 秒 00..59 |
| `%e` | 日，用空格填充 | `%f` | 纳秒，9 位；`%3f` `%6f` `%9f` |
| `%j` | 一年中的第几天 001..366 | `%p` / `%P` | `AM` / `PM`，`am` / `pm` |
| `%a` / `%A` | `Mon` / `Monday` | `%z` / `%:z` | 偏移 `+0900` / `+09:00` |
| `%b` / `%h` / `%B` | `Jan` / `Jan` / `January` | `%Z` | 缩写（`KST`），或偏移 |
| `%u` / `%w` | 星期几，从星期一起 1..7 / 从星期日起 0..6 | `%s` | unix 秒数 |
| `%G` / `%g` / `%V` | ISO 周年 / 其末 2 位 / ISO 周 | `%F` | `%Y-%m-%d` |
| `%U` / `%W` | 一年中的第几周，从星期日 / 星期一起算 | `%T` | `%H:%M:%S` |
| `%D` / `%x` | `%m/%d/%y` | `%R` | `%H:%M` |
| `%c` | `%a %b %e %H:%M:%S %Y` | `%r` | `%I:%M:%S %p` |
| `%n` / `%t` | 换行 / 制表符 | `%%` | `%` |

`%` 之后的标志改变填充方式：`-` 不填充（`%-d` 为 `5`），`_` 用空格，`0` 用零。模式是程序文本，因此未知的指令，或者值所缺少的指令（`Date` 上的 `%H`、`DateTime` 上的 `%z`），会触发 panic，消息会说明应当写什么。

`parse_with` 读取不超过指令宽度的数字（`%Y` 最多 4 位，带符号时最多 6 位），名称不区分大小写，可以是全称或三个字母的缩写，模式中的一个空格可以匹配任意长度的空白。缺少的时间字段为零；`Date` 需要年份（`%Y`，或 `%y`，其中 69..99 视为 19xx），以及月和日或 `%j`。与日期不符的星期几是错误。`%Z` 读取 `UTC`、`GMT`、`UT`、`Z` 或 `Europe/Paris` 这样的时区名；`EST` 之类的缩写有歧义，会被拒绝，并提示改用 `%z`。`Zoned::parse_with` 需要偏移、时区或 `%s`。`%C`、`%g`、`%G`、`%V`、`%U` 和 `%W` 只能用于写出。

**邮件与 HTTP。** `Zoned::to_rfc2822` 和 `Zoned::parse_rfc2822` 处理 RFC 5322 邮件中的日期（`Sat, 14 Nov 2026 09:30:00 +0900`）；解析时星期几和秒是可选的，并接受过时的时区名（`GMT`、`EST`、`PDT` 等）。`SystemTime::to_http_date` 是 RFC 9110 的 IMF-fixdate（`Sun, 06 Nov 1994 08:49:37 GMT`），`parse_http_date` 还能读取服务器必须接受的两种过时形式（`Sunday, 06-Nov-94 08:49:37 GMT`、`Sun Nov  6 08:49:37 1994`）。

```nabi
impl Date {
    const fn format(self, pattern: str) -> str;
    fn parse_with(text: str, pattern: str) throws (ParseError) -> Date;
}
// the same on Time, DateTime and Zoned
impl Zoned {
    const fn to_rfc2822(self) -> str;
    fn parse_rfc2822(text: str) throws (ParseError) -> Zoned;
}
```

### 示例

```nabi
use std::time::{Date, DateTime, SystemTime, Zoned};

let t = SystemTime::from_unix(1700000000, 250000000);
println(f"{t} {t.to_http_date()}");
let z = try "2023-11-15T07:13:20+09:00".parse::<Zoned>();
println(f"{z} {z.to_utc()} {z.format("%A %-d %B %Y, %I:%M %p %:z")}");

let dt = try DateTime::parse_with("10/03/2024 2:05 PM", "%d/%m/%Y %I:%M %p");
println(f"{dt} {dt.format("%F %T")} {dt.date().format("%G-W%V-%u")}");
try { println(f"{try "2024-02-30".parse::<Date>()}"); } catch e { println(f"{e}"); }
println(f"{try Zoned::parse_rfc2822("Sat, 14 Nov 2026 09:30:00 +0900")}");
```

```output
2023-11-14T22:13:20.250Z Tue, 14 Nov 2023 22:13:20 GMT
2023-11-15T07:13:20+09:00 2023-11-14T22:13:20Z Wednesday 15 November 2023, 07:13 AM +09:00
2024-03-10T14:05:00 2024-03-10 14:05:00 2024-W10-7
cannot parse "2024-02-30": day 30 is out of range for 2024-02 (1..=29) at byte 8
2026-11-14T09:30:00+09:00
```

## 3.3.9. 策略：不支持 TAI / 闰秒

`SystemTime` 严格采用 **UTC**，不处理闰秒（考虑到 Linux 内核本身在这方面就不明确）。日历类型也遵循这一点：每天都是 86400 秒，`Time` 没有第 60 秒，解析 `23:59:60` 是错误，错误会说明这一点。需要 TAI（原子时）或精确处理闰秒的场景属于独立的包。

| 用途                       | 建议                      |
|----------------------------|---------------------------|
| 普通应用（时间戳 / 测量）  | core::time、std::time     |
| 仿真 / 精密天文学          | 类似 `chrono` 的独立包    |
| 金融 / 精确闰秒            | 独立的包                  |

## 3.3.10. impl / 目标

各平台的实现映射：

| 项           | linux           | macos            | windows                 | wasi         |
|--------------|-----------------|------------------|-------------------------|--------------|
| `Instant`    | CLOCK_MONOTONIC | CLOCK_UPTIME_RAW | QueryPerformanceCounter | wasi 时钟    |
| `SystemTime` | CLOCK_REALTIME  | CLOCK_REALTIME   | GetSystemTimePreciseAsFileTime | wasi  |
| `BootTime`   | CLOCK_BOOTTIME  | CLOCK_MONOTONIC（mach_continuous_time，含睡眠） | QueryPerformanceCounter（≈ Instant） | （无挂起） |
| `TimeZone::get` | `$TZDIR`、/usr/share/zoneinfo | /usr/share/zoneinfo | 无：只有 `UTC` 和偏移 | 主机授予时使用主机的文件，否则无 |
| `TimeZone::local` | `$TZ`、/etc/localtime | `$TZ`、/etc/localtime | `$TZ` 中的 POSIX 规则，否则 UTC | 传入 `$TZ` 时使用它，否则 UTC |

## 3.3.11. 策略总结

本章的关键决定有五项。第一，`Instant` / `SystemTime` / `Duration` 彼此分开，在语义上区分单调时间和挂钟时间；`Duration` 和 `Instant` 位于 `core::time`，操作系统时钟位于 `std::time`。第二，所有跨度都是精确的有符号 `Duration`，`elapsed()` 在单调时钟上运行。第三，单独提供 Rust 的 std 没有涵盖的 `BootTime`，以便直接读取自启动以来经过的时间（含挂起）。第四，TAI 和闰秒被拆分到本模块之外的独立包中。第五，与 Python、Go 和 Java 一样，日历位于 std 中：没有时区的民用类型（`Date`、`Time`、`DateTime`）与带时区的时刻（`Zoned`）分开，时区跳过或重复的本地时间通过显式的 `Disambiguation` 解析，文本形式采用标准格式（`Display` 用 RFC 3339、strftime 模式、RFC 2822 和 HTTP 日期）。
