# 3.3. `time::`：時間

時間依其需求分在兩個模組。`core::time` 存放 `Duration` 和 `Instant`：單調時鐘在 `core` 中就已經在執行（任務計時器，2.18），因此兩者在每個目標上都能使用，也不需要 `use`（0.2）。`std::time` 存放需要作業系統的時鐘：牆上時鐘 `SystemTime`、開機時鐘 `BootTime` 和 `UNIX_EPOCH`。它也存放民用曆法：`Date`、`Time`、`DateTime`、`Period`、`Offset`、`TimeZone` 和 `Zoned`（3.3.6 至 3.3.8），因為時區來自作業系統的 tz 資料庫。每個型別只有一個路徑（[0.1.6](../000-meta/001-tier-model.md#016-每個項目只有一條路徑)）；`std::time` 不會重新匯出 core 的型別。單調時鐘、牆上時鐘與自開機以來經過的時間明確分開，因此可以選擇適合該測量的時鐘。

## 3.3.1. 四型別模型

整體概況：

| 型別         | 模組            | 意義                     | 用途                 |
|--------------|-------------------|-----------------------------|--------------------------|
| `Instant`    | `core::time` | 單調時鐘             | 時間長度測量 / 逾時 |
| `Duration`   | `core::time` | 時間差（i64 奈秒） | 兩個時間之差 / sleep 的引數 |
| `SystemTime` | `std::time`  | 牆上時鐘（UTC，可由 NTP 調整） | 檔案 mtime / 顯示給使用者 |
| `BootTime`   | `std::time`  | 自開機以來的時間（含休眠） | 行動裝置 / 筆電應用程式 |

`Duration` 的內部表示是 `i64` 奈秒。Rust 採用的 `u128` 被認為過大；`i64` 已涵蓋約 ±292 年，對實務用途綽綽有餘。

**所有時鐘適用同一條規則**。任何時鐘兩次讀數之間的間隔，都是精確的有號 `Duration`：`b` 較晚時 `a.duration_since(b)` 為負值，絕不會是錯誤、`none` 或零。`elapsed()` 在單調時鐘上測量，因此牆上時鐘的跳動不會讓它變成負值。這是 Go 的規則（`time.Sub`、`time.Since`）。

## 3.3.2. `Instant`：單調時鐘

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

`Instant` 的關鍵性質：它永不倒退，因此不受 NTP 調整影響。它本身沒有絕對時間的意義（其基準是行程啟動），因此**無法序列化**。基於同樣的理由，跨行程比較沒有意義；它只在同一個行程內有意義。

`earlier` 較晚時，`duration_since` 回傳負的 `Duration`。要夾限一段間隔，就比較它：`d.max(Duration::ZERO)`。

### 範例

最常見的模式是取一個起點，再用 `elapsed` 取得經過的時間。

```nabi
let start = Instant::now();
heavy_computation();
let elapsed = start.elapsed();          // Duration
log(f"computation took {elapsed:?}");
```

## 3.3.3. `SystemTime`：牆上時鐘

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

`SystemTime` 是相對於 `UNIX_EPOCH` 的 `i64` 奈秒，因此在每個目標上的範圍都約為 ±292 年。32 位元目標透過 64 位元時間介面（`time64`）讀取時鐘，因此沒有 2038 年的限制。

`SystemTime` 的行為相當不同。時間可能因 NTP 調整或使用者變更而倒退，而且由於它是相對於 UNIX_EPOCH 的絕對時間，因此**可以序列化**。所以跨行程比較它也有意義。

`duration_since` 是牆上讀數的有號差：較晚的 `earlier` 會得到負的 `Duration`，而不是錯誤。`unix_secs` 向下捨入，因此對於秒以下部分為 `n` 的 `t`，`SystemTime::from_unix(t.unix_secs(), n)` 會還原出 `t`，在紀元之前也是如此。

**單調讀數**。`SystemTime::now()` 也會讀取單調時鐘（`Instant` 的時鐘）。`elapsed()` 在該讀數上測量，因此 `now()` 與 `elapsed()` 之間牆上時鐘的跳動不會改變結果。與 `Duration` 進行的 `+` / `-` / `checked_*` 會同時移動兩個讀數。不是來自 `now()` 的 `SystemTime`（`from_unix`、`UNIX_EPOCH`、`Metadata::modified`）沒有單調讀數，其 `elapsed()` 是 `SystemTime::now().duration_since(self)`。比較、相等性與雜湊只使用牆上讀數。

### 範例

轉換為 unix 時間戳記，以及讀取檔案的 mtime：

```nabi
use std::time::SystemTime;

let now = SystemTime::now();
log(f"unix timestamp: {now.unix_secs()}");

// file mtime
let meta = try fs::metadata("data.txt");
let mtime: SystemTime = try meta.modified();
log(f"file age: {mtime.elapsed():?}");     // negative if the mtime is in the future
```

## 3.3.4. `BootTime`：自開機以來的時間（含休眠）

```nabi
// std::time
struct BootTime { /* CLOCK_BOOTTIME (Linux) / CLOCK_MONOTONIC = mach_continuous_time (macOS) */ }

impl BootTime {
    fn now() -> BootTime;
    fn elapsed(self) -> Duration;
    const fn duration_since(self, earlier: BootTime) -> Duration;   // self - earlier, signed
}
```

與 `Instant` 相同，`earlier` 較晚時 `duration_since` 為負值。`BootTime` 和 `Instant` 的差別在於休眠期間的行為。`Instant` 與 Linux 的 `CLOCK_MONOTONIC` 一樣在休眠期間停止，而 `BootTime` 依循 `CLOCK_BOOTTIME`，在休眠期間持續前進。

當行動裝置或筆電應用程式必須測量「實際經過的時間（含休眠）」時，這個差別具有決定性。截至 2026 年，Rust 仍沒有這方面的穩定 API；Nabi 及早將它直接公開。

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

兩個時間長度的 `a.min(b)` 是 `Ord::min`。

`Display` 寫出一個符合數量級的單位：絕對值至少 1 秒時用 `s`，至少 1 ms 時用 `ms`，至少 1 µs 時用 `µs`，否則用 `ns`。先寫出除以該單位所得的整數部分，再以小數寫出餘數；尾端的零會移除，餘數為 0 時不寫小數點。負值會加上前綴 `-`。

```nabi
Duration::ms(1234);      // "1.234s"
Duration::ms(5);         // "5ms"
Duration::ns(1500);      // "1.5µs"
Duration::ns(-20);       // "-20ns"
```

所有時間型別都實作 `Clone` / `Copy` / `PartialEq` / `Eq` / `PartialOrd` / `Ord` / `Hash` / `Debug`。`Duration` 還實作 `Default`（零）。

### 時間長度的算術策略

`Duration` / `SystemTime` / `Instant` / `BootTime` 的算術（包括 `duration_since` 和 `from_unix`）在溢位檢查開啟時遇到溢位會觸發 panic，關閉時則飽和（`-C overflow-checks`，[refs-lang 9.1.2](../../refs-lang/009-tools/901-nabic-cli.md#nabic-buildaot-建置)）。`f64` 形式（`Duration::secs_f64`、`f64 * Duration`）遵循相同的規則：非有限或超出範圍的結果在檢查開啟時觸發 panic，關閉時則飽和（NaN 變為零）。除以零一律觸發 panic。`Duration` 的 `checked_*` / `saturating_*` 變體則明確地做出選擇。

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

## 3.3.6. 曆法日期與時間

曆法型別沒有時區：它們就是牆上的月曆與時鐘所顯示的內容。全部都是 `Copy` 值，依時間排序，並具有 `Hash`；`Display` / `FromStr` 採用 ISO 8601（3.3.8）。

| 型別       | 內容                                   | `Display`               |
|------------|-----------------------------------------|-------------------------|
| `Date`     | 年、月、日（外推格里曆）  | `2024-02-29`            |
| `Time`     | 時、分、秒、奈秒        | `09:05:00.25`           |
| `DateTime` | 一個 `Date` 和一個 `Time`                   | `2024-03-10T02:30:00`   |
| `Period`   | 年、月、日，各自分開保存    | `P1Y2M10D`              |
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

**範圍與曆法**。`Date` 是 `MIN_YEAR` 到 `MAX_YEAR` 之間外推格里曆的一天；與 ISO 8601 的計法相同，第 0 年是西元前 1 年。日數計算使用 Howard Hinnant 的民用曆演算法，在整個範圍內都是精確的。`unix_days` 從 1970-01-01 起算，該日為第 0 天。

**錯誤的欄位會擲回，溢位的算術會觸發 panic**。建構子收到不構成日期或時間的欄位時（`Date::new(2023, 2, 29)`、`Time::new(24, 0, 0)`），會擲回一個指出該欄位與其範圍的 `ValueError`：`day 29 is out of range for 2023-02 (1..=28)`。超出範圍的算術（`Date::MAX.add_days(1)`）與 `Duration` 一樣會觸發 panic；`checked_*` 形式則改為回傳 `none`。

**月份會夾限**。新月份有相同日期時，`add_months` 和 `add_years` 保留該月中的日，否則取新月份的最後一天：2024-01-31 加一個月是 2024-02-29，2024-02-29 加一年是 2025-02-28。每次呼叫各自夾限，因此從 1 月 31 日起連續兩次 `add_months(1)` 會落在 3 月 29 日，而 `add_months(2)` 落在 3 月 31 日。`Date + Period` 先將年與月一起相加（只夾限一次），再加日數。這是 java.time 與 Temporal 的規則。

**`until`** 是從 `self` 到 `later` 的曆法間隔：先是整年，再是整月，最後是日數，全部帶有該間隔的正負號，其選法使得對任意兩個日期都有 `d + d.until(e) == e`。`days_since` 則是單純的日數差。

**ISO 週**。`iso_week` 是 ISO 8601 的週編號：每週從星期一開始，而第 1 週包含該年的第一個星期四，因此 2021-01-03 屬於 2020 年第 53 週，2024-12-30 屬於 2025 年第 1 週。

### 範例

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

## 3.3.7. 時差、時區與 `Zoned`

`Offset` 是與 UTC 之間的固定差。`TimeZone` 給出每個時刻的時差。`Zoned` 是一個時刻加上用來顯示它的時區。

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

**時區從哪裡來**。`TimeZone::get` 以 TZif 檔案（RFC 8536，版本 1 至 4）的形式讀取 tz 資料庫（IANA）中的時區：設定了 `$TZDIR` 時從該處讀取，否則從 `/usr/share/zoneinfo`、`/usr/lib/zoneinfo`、`/usr/share/lib/zoneinfo` 和 `/etc/zoneinfo` 中第一個存在的目錄讀取。超過檔案中最後一次轉換之後，由其結尾的 POSIX `TZ` 規則給出時差，因此遙遠未來的日期會遵循現行規則。`UTC`、其他 UTC 名稱（`Etc/UTC`、`GMT`、`Zulu`、...）和時差（`+09:00`）不需要資料庫即可使用。其他任何名稱在沒有資料庫時都會擲回，錯誤會說明系統沒有資料庫：Windows 與瀏覽器沒有 tz 資料庫，而 WASI 程式只有在主機授予該目錄時才看得到。`from_tzif` 讀取程式自行帶入的 TZif 檔案（例如用 `include_bytes!`），可在每個目標上使用。`from_posix` 接受 POSIX `TZ` 規則（`EST5EDT,M3.2.0,M11.1.0`；其時差以西為正，與 POSIX 的寫法相同）。

**本地時區**。`TimeZone::local()` 先讀取 `$TZ`：資料庫名稱（`Asia/Seoul`）、TZif 檔案的路徑（`/usr/share/zoneinfo/Asia/Seoul`，開頭的 `:` 會去掉）或 POSIX 規則。沒有 `$TZ` 時，它讀取 `/etc/localtime`；若 `/etc/timezone` 指名了某個時區，就以該名稱命名，否則命名為 `localtime`。這些都無法給出時區時，就是 UTC。它每次呼叫都會讀取檔案，因此程式應當保留結果。

**間隙與重疊**。時鐘向前撥（間隙）時，有些本地時間從未發生；向後撥（重疊）時，有些本地時間發生兩次。`in_zone_with` 與 java.time 和 Temporal 一樣接受一個選擇：

| `Disambiguation` | 重疊時（01:30 顯示兩次） | 間隙時（跳過 02:30，02:00 至 03:00） |
|------------------|-----------------------------------|-------------------------------------------|
| `Compatible`     | 較早的時刻               | 向前移動間隙的長度：03:30           |
| `Earlier`        | 較早的時刻               | 以間隙之後的時差讀取：01:30 |
| `Later`          | 較晚的時刻                 | 以間隙之前的時差讀取：03:30 |
| `Reject`         | 擲回 `ValueError`               | 擲回 `ValueError`                       |

`in_zone` 是 `in_zone_with(.., Disambiguation::Compatible)`，這是大多數系統的做法，而且不會失敗。`Reject` 的錯誤會指出時區、兩個時差，以及應改傳的值：`2024-03-10T02:30:00 does not exist in America/New_York: the clocks skip it (-05:00 to -04:00); pass Disambiguation::Earlier or Disambiguation::Later`。`Zoned` 上的 `add_days`、`add_months`、`add_years` 和 `+ Period` 移動民用日期、保留一天中的時間，並以 `Compatible` 解析結果；`+ Duration` 則加上精確的時間。

`is_dst_at` 和 `Zoned::is_dst` 回報時區資料中的日光節約時間旗標。大多數時區標記的是夏令時間；負 DST 的時區則改為標記冬令時間（`IST-1GMT0,M10.5.0,M3.5.0/1`，多數 tz 建置中 Europe/Dublin 的規則）。

**相等性**。兩個 `Zoned` 值只依其時刻判斷相等、排序與雜湊：`2024-01-01T09:00:00+09:00` 等於 `2024-01-01T00:00:00Z`。兩個 `TimeZone` 名稱相同時即相等。

### 範例

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

## 3.3.8. 文字：ISO 8601、RFC 3339 與樣式

`Display` 寫出、`FromStr`（`str::parse`）讀取標準形式：

| 型別         | `Display`                               | `FromStr` 另外可讀取                          |
|--------------|-----------------------------------------|-----------------------------------------------|
| `Date`       | `2024-02-29`、`-000044-03-15`、`+012345-01-01` | |
| `Time`       | `09:05:00`、`09:05:00.250`              | `09:05`、1 至 9 位數的小數、以 `,` 代替 `.` |
| `DateTime`   | `2024-03-10T02:30:00`                   | 以 `t` 或空格代替 `T`                        |
| `Offset`     | `+09:00`、`-03:30`、`+05:30:15`         | `Z`、`+0900`、`+09`                           |
| `Zoned`      | `2024-03-10T03:30:00-04:00`、`...Z`     | RFC 9557 的 `[America/New_York]` 後綴        |
| `SystemTime` | `2023-11-14T22:13:20Z`（UTC）            | 任何時差                                    |
| `Period`     | `P1Y2M10D`、`P0D`、`P-1M-3D`            | （沒有 `FromStr`）                                |

0..=9999 之外的年份與 ISO 8601 的擴充形式一樣，帶有正負號與六位數。小數以 3、6 或 9 位數寫出，取能容納它的最少位數，為零時則省略。時差為零的 `Zoned` 寫出 `Z`；其 `Debug` 會在方括號中加上時區名稱（`2024-03-10T03:30:00-04:00[America/New_York]`）。讀取帶有方括號時區的 `Zoned` 時會載入該時區；時差必須與之一致，否則錯誤會指出兩者（沒有時差時，本地時間以 `Compatible` 解析）。沒有方括號時，時區就是該固定時差。帶有時差的 `DateTime` 文字是錯誤，錯誤會說明應剖析為 `Zoned` 或 `SystemTime`。

**錯誤會指出位置**。剖析錯誤是一個 `ParseError`，其訊息指出位元組偏移量以及該處應有的內容：``cannot parse "2024/02/03": expected `-` after the year at byte 4, found `/` ``、`cannot parse "2024-13-01": month 13 is out of range (1..=12) at byte 5`。`position()` 是同一個偏移量。

**樣式**。`Date`、`Time`、`DateTime` 和 `Zoned` 上的 `format(pattern)` 寫出 strftime 風格的指示詞；`parse_with(text, pattern)` 讀回相同的指示詞。名稱為英文（C 地區設定）。

| 指示詞 | 意義 | 指示詞 | 意義 |
|-----------|---------|-----------|---------|
| `%Y` | 年，至少 4 位數 | `%H` | 時 00..23 |
| `%C` | 世紀（`%Y` / 100） | `%I` | 時 01..12 |
| `%y` | 年 % 100，2 位數 | `%k` / `%l` | 以空格填補的 `%H` / `%I` |
| `%m` | 月 01..12 | `%M` | 分 00..59 |
| `%d` | 日 01..31 | `%S` | 秒 00..59 |
| `%e` | 日，以空格填補 | `%f` | 奈秒，9 位數；`%3f` `%6f` `%9f` |
| `%j` | 一年中的第幾天 001..366 | `%p` / `%P` | `AM` / `PM`、`am` / `pm` |
| `%a` / `%A` | `Mon` / `Monday` | `%z` / `%:z` | 時差 `+0900` / `+09:00` |
| `%b` / `%h` / `%B` | `Jan` / `Jan` / `January` | `%Z` | 縮寫（`KST`），或時差 |
| `%u` / `%w` | 從星期一起算 1..7 / 從星期日起算 0..6 的星期 | `%s` | unix 秒數 |
| `%G` / `%g` / `%V` | ISO 週年 / 其末 2 位數 / ISO 週 | `%F` | `%Y-%m-%d` |
| `%U` / `%W` | 從星期日 / 星期一起算的年中週數 | `%T` | `%H:%M:%S` |
| `%D` / `%x` | `%m/%d/%y` | `%R` | `%H:%M` |
| `%c` | `%a %b %e %H:%M:%S %Y` | `%r` | `%I:%M:%S %p` |
| `%n` / `%t` | 換行 / 定位字元 | `%%` | `%` |

`%` 之後的旗標會改變填補方式：`-` 不填補（`%-d` 為 `5`），`_` 以空格填補，`0` 以零填補。樣式是程式文字，因此未知的指示詞，或值所缺少的指示詞（`Date` 上的 `%H`、`DateTime` 上的 `%z`），會觸發 panic，訊息會說明應寫什麼。

`parse_with` 讀取最多到指示詞寬度的數字（`%Y` 最多 4 位數，帶正負號時為 6 位數），名稱不分大小寫、可為全名或三個字母，並讓樣式中的一個空格比對任意長度的連續空白字元。缺少的時間欄位為零；`Date` 需要年（`%Y`，或將 69..99 視為 19xx 的 `%y`），以及月與日或 `%j`。與日期不符的星期是錯誤。`%Z` 讀取 `UTC`、`GMT`、`UT`、`Z` 或像 `Europe/Paris` 這樣的時區名稱；像 `EST` 這樣的縮寫有歧義，會被拒絕，並提示改用 `%z`。`Zoned::parse_with` 需要時差、時區或 `%s`。`%C`、`%g`、`%G`、`%V`、`%U` 和 `%W` 只能寫出。

**郵件與 HTTP**。`Zoned::to_rfc2822` 和 `Zoned::parse_rfc2822` 處理 RFC 5322 郵件的日期（`Sat, 14 Nov 2026 09:30:00 +0900`）；剖析時星期與秒數為選擇性，並接受過時的時區名稱（`GMT`、`EST`、`PDT`、...）。`SystemTime::to_http_date` 是 RFC 9110 的 IMF-fixdate（`Sun, 06 Nov 1994 08:49:37 GMT`），而 `parse_http_date` 也讀取伺服器必須接受的兩種過時形式（`Sunday, 06-Nov-94 08:49:37 GMT`、`Sun Nov  6 08:49:37 1994`）。

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

### 範例

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

## 3.3.9. 策略：不支援 TAI / 閏秒

`SystemTime` 嚴格採用 **UTC**，且不處理閏秒（考量到 Linux 核心本身在這方面就不明確）。曆法型別也依循此規則：每天都有 86400 秒，`Time` 沒有第 60 秒，剖析 `23:59:60` 是錯誤，錯誤會如此說明。需要 TAI（原子時）或精確閏秒處理的情況，屬於獨立的套件。

| 用途                        | 建議            |
|----------------------------|---------------------------|
| 一般應用程式（時間戳記 / 測量） | core::time、std::time |
| 模擬 / 精密天文學 | 獨立的類 `chrono` 套件 |
| 金融 / 精確閏秒 | 獨立的套件      |

## 3.3.10. impl / target

各平台的實作對應：

| 項目         | linux           | macos            | windows                 | wasi         |
|--------------|-----------------|------------------|-------------------------|--------------|
| `Instant`    | CLOCK_MONOTONIC | CLOCK_UPTIME_RAW | QueryPerformanceCounter | wasi clock   |
| `SystemTime` | CLOCK_REALTIME  | CLOCK_REALTIME   | GetSystemTimePreciseAsFileTime | wasi  |
| `BootTime`   | CLOCK_BOOTTIME  | CLOCK_MONOTONIC（mach_continuous_time，含睡眠） | QueryPerformanceCounter（~ Instant） | （無休眠） |
| `TimeZone::get` | `$TZDIR`、/usr/share/zoneinfo | /usr/share/zoneinfo | 無：只有 `UTC` 和時差 | 主機授予時為主機的檔案，否則無 |
| `TimeZone::local` | `$TZ`、/etc/localtime | `$TZ`、/etc/localtime | `$TZ` 中的 POSIX 規則，否則 UTC | 有傳入時為 `$TZ`，否則 UTC |

## 3.3.11. 策略摘要

本章的關鍵決定有五項。第一，`Instant` / `SystemTime` / `Duration` 彼此分開，在語意上區分單調時間與牆上時鐘時間；`Duration` 和 `Instant` 位於 `core::time`，作業系統時鐘位於 `std::time`。第二，每段間隔都是精確的有號 `Duration`，而 `elapsed()` 在單調時鐘上執行。第三，Rust 的 std 未涵蓋的 `BootTime` 另外公開，因此可以直接讀取自開機以來（含休眠）經過的時間。第四，TAI 與閏秒分離到本模組之外的獨立套件。第五，曆法與 Python、Go 和 Java 一樣位於 std：沒有時區的民用型別（`Date`、`Time`、`DateTime`）與時區中的時刻（`Zoned`）分開，時區跳過或重複的本地時間以明確的 `Disambiguation` 解析，文字形式則採用標準形式（`Display` 用 RFC 3339、strftime 樣式、RFC 2822 與 HTTP 日期）。
