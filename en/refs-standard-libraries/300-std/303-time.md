# 3.3. `time::` - time

Time is split across two modules by what it needs. `core::time` holds `Duration` and `Instant`: the monotonic clock already runs in `core` (task timers, 2.18), so both work on every target and need no `use` (0.2). `std::time` holds the clocks that need the operating system: the wall clock `SystemTime`, the boot clock `BootTime` and `UNIX_EPOCH`. It also holds the civil calendar: `Date`, `Time`, `DateTime`, `Period`, `Offset`, `TimeZone` and `Zoned` (3.3.6 to 3.3.8), since time zones come from the operating system's tz database. Each type has one path ([0.1.6](../000-meta/001-tier-model.md#016-one-path-per-item)); `std::time` does not re-export the core types. The monotonic clock, the wall clock and time elapsed since boot are kept clearly separate, so the clock that fits the measurement can be chosen.

## 3.3.1. The four-type model

The overall picture:

| Type         | Module            | Meaning                     | Used for                 |
|--------------|-------------------|-----------------------------|--------------------------|
| `Instant`    | `core::time` | Monotonic clock             | Duration measurement / timeouts |
| `Duration`   | `core::time` | Time difference (i64 nanoseconds) | Difference between two times / sleep argument |
| `SystemTime` | `std::time`  | Wall clock (UTC, NTP-adjustable) | File mtime / display to users |
| `BootTime`   | `std::time`  | Time since boot (including suspend) | Mobile / laptop apps |

The internal representation of `Duration` is `i64` ns. The `u128` that Rust adopted was judged excessive; `i64` already covers about ±292 years, which is ample for practical use.

**One rule for every clock.** The span between two readings of any clock is the exact signed `Duration`: `a.duration_since(b)` is negative when `b` is later, never an error, `none` or zero. `elapsed()` measures on a monotonic clock, so a step of the wall clock cannot make it negative. This is Go's rule (`time.Sub`, `time.Since`).

## 3.3.2. `Instant` - monotonic clock

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

The key properties of `Instant`: it never goes backward, so it is unaffected by NTP adjustment. It has no absolute time meaning of its own (its reference is process start), so it **cannot be serialized**. For the same reason, comparisons across processes are meaningless; it is meaningful only within the same process.

`duration_since` returns a negative `Duration` when `earlier` is later. To clamp a span, compare it: `d.max(Duration::ZERO)`.

### Example

The most common pattern is to take a starting point and get the elapsed time with `elapsed`.

```nabi
let start = Instant::now();
heavy_computation();
let elapsed = start.elapsed();          // Duration
log(f"computation took {elapsed:?}");
```

## 3.3.3. `SystemTime` - wall clock

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

`SystemTime` is `i64` nanoseconds relative to `UNIX_EPOCH`, so its range is about ±292 years on every target. 32-bit targets read the clock through 64-bit time interfaces (`time64`), so there is no 2038 limit.

`SystemTime` behaves quite differently. Time may go backward through NTP adjustment or user changes, and since it is an absolute time relative to UNIX_EPOCH it **can be serialized**. It can therefore be compared meaningfully across processes.

`duration_since` is the signed difference of the wall readings: a later `earlier` gives a negative `Duration`, not an error. `unix_secs` rounds down, so `SystemTime::from_unix(t.unix_secs(), n)` gives `t` back for its sub-second part `n`, also before the epoch.

**Monotonic reading.** `SystemTime::now()` also reads the monotonic clock (`Instant`'s). `elapsed()` measures on that reading, so a step of the wall clock between `now()` and `elapsed()` does not change the result. `+` / `-` / `checked_*` with a `Duration` move both readings. A `SystemTime` that did not come from `now()` (`from_unix`, `UNIX_EPOCH`, `Metadata::modified`) has none, and its `elapsed()` is `SystemTime::now().duration_since(self)`. Comparison, equality and hashing use the wall reading only.

### Example

Converting to a unix timestamp and reading a file's mtime:

```nabi
use std::time::SystemTime;

let now = SystemTime::now();
log(f"unix timestamp: {now.unix_secs()}");

// file mtime
let meta = try fs::metadata("data.txt");
let mtime: SystemTime = try meta.modified();
log(f"file age: {mtime.elapsed():?}");     // negative if the mtime is in the future
```

## 3.3.4. `BootTime` - time since boot (including suspend)

```nabi
// std::time
struct BootTime { /* CLOCK_BOOTTIME (Linux) / CLOCK_MONOTONIC = mach_continuous_time (macOS) */ }

impl BootTime {
    fn now() -> BootTime;
    fn elapsed(self) -> Duration;
    const fn duration_since(self, earlier: BootTime) -> Duration;   // self - earlier, signed
}
```

As with `Instant`, `duration_since` is negative when `earlier` is later. `BootTime` and `Instant` differ in their behavior across suspend. `Instant`, like Linux `CLOCK_MONOTONIC`, stops during suspend, whereas `BootTime` follows `CLOCK_BOOTTIME` and keeps advancing during suspend.

This difference is decisive when a mobile or laptop app must measure "real elapsed time (including suspend)". Rust still has no stable API for this as of 2026; Nabi exposes it on the surface early.

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

`a.min(b)` on two durations is `Ord::min`.

`Display` writes a single unit fitting the magnitude: `s` if the absolute value is at least 1 second, `ms` if at least 1 ms, `µs` if at least 1 µs, and `ns` otherwise. The integer part divided by that unit is followed by the remainder as a fraction; trailing zeros are removed, and if the remainder is 0 no decimal point is written. Negative values are prefixed with `-`.

```nabi
Duration::ms(1234);      // "1.234s"
Duration::ms(5);         // "5ms"
Duration::ns(1500);      // "1.5µs"
Duration::ns(-20);       // "-20ns"
```

All time types implement `Clone` / `Copy` / `PartialEq` / `Eq` / `PartialOrd` / `Ord` / `Hash` / `Debug`. `Duration` also implements `Default` (zero).

### Duration arithmetic policy

Arithmetic on `Duration` / `SystemTime` / `Instant` / `BootTime` (including `duration_since` and `from_unix`) panics on overflow when overflow checks are on and saturates when they are off (`-C overflow-checks`, [refs-lang 9.1.2](../../refs-lang/009-tools/901-nabic-cli.md#nabic-build-aot-build)). The `f64` forms (`Duration::secs_f64`, `f64 * Duration`) follow the same rule: a non-finite or out-of-range result panics with checks on and saturates (NaN to zero) with them off. Division by zero always panics. The `checked_*` / `saturating_*` variants of `Duration` choose explicitly.

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

## 3.3.6. Calendar dates and times

The calendar types have no time zone: they are what a wall calendar and a wall clock show. All are `Copy` values, ordered by time, with `Hash`; `Display` / `FromStr` are ISO 8601 (3.3.8).

| Type       | Holds                                   | `Display`               |
|------------|-----------------------------------------|-------------------------|
| `Date`     | year, month, day (proleptic Gregorian)  | `2024-02-29`            |
| `Time`     | hour, minute, second, nanosecond        | `09:05:00.25`           |
| `DateTime` | a `Date` and a `Time`                   | `2024-03-10T02:30:00`   |
| `Period`   | years, months, days, each kept apart    | `P1Y2M10D`              |
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

**Range and calendar.** A `Date` is a day of the proleptic Gregorian calendar between `MIN_YEAR` and `MAX_YEAR`; year 0 is 1 BC, as ISO 8601 counts. Day counts use Howard Hinnant's civil algorithms, exact over the whole range. `unix_days` counts from 1970-01-01, which is day 0.

**Wrong fields throw, overflowing arithmetic panics.** A constructor that gets fields which are not a date or time (`Date::new(2023, 2, 29)`, `Time::new(24, 0, 0)`) throws a `ValueError` naming the field and its range: `day 29 is out of range for 2023-02 (1..=28)`. Arithmetic that leaves the range (`Date::MAX.add_days(1)`) panics, as `Duration` does; the `checked_*` forms return `none` instead.

**Months clamp.** `add_months` and `add_years` keep the day of the month when the new month has it, and otherwise take the new month's last day: 2024-01-31 plus one month is 2024-02-29, and 2024-02-29 plus one year is 2025-02-28. Each call clamps on its own, so two `add_months(1)` from January 31 end on March 29, while `add_months(2)` ends on March 31. `Date + Period` adds years and months together (clamping once), then the days. This is the rule of java.time and Temporal.

**`until`** is the calendar span from `self` to `later`: whole years, then whole months, then days, all with the sign of the span, chosen so that `d + d.until(e) == e` for any two dates. `days_since` is the plain difference in days.

**ISO weeks.** `iso_week` is ISO 8601's week numbering: weeks start on Monday, and week 1 holds the year's first Thursday, so 2021-01-03 is in week 53 of 2020 and 2024-12-30 in week 1 of 2025.

### Example

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

## 3.3.7. Offsets, time zones and `Zoned`

An `Offset` is a fixed difference from UTC. A `TimeZone` gives the offset at each instant. A `Zoned` is one instant together with the zone it is shown in.

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

**Where zones come from.** `TimeZone::get` reads a zone of the tz database (IANA) as a TZif file (RFC 8536, versions 1 to 4) from `$TZDIR` when it is set, else from the first of `/usr/share/zoneinfo`, `/usr/lib/zoneinfo`, `/usr/share/lib/zoneinfo` and `/etc/zoneinfo` that exists. Past the last transition in the file, the POSIX `TZ` rule in its footer gives the offsets, so dates far in the future follow the current rules. `UTC`, the other UTC names (`Etc/UTC`, `GMT`, `Zulu`, ...) and offsets (`+09:00`) work with no database. Any other name without a database throws, and the error says that the system has none: Windows and the browser have no tz database, and WASI programs see one only when the host grants the directory. `from_tzif` reads a TZif file the program brings itself (for example with `include_bytes!`), which works on every target. `from_posix` takes a POSIX `TZ` rule (`EST5EDT,M3.2.0,M11.1.0`; its offsets are west positive, as POSIX writes them).

**The local zone.** `TimeZone::local()` reads `$TZ` first: a database name (`Asia/Seoul`), a path to a TZif file (`/usr/share/zoneinfo/Asia/Seoul`, a leading `:` is dropped) or a POSIX rule. Without `$TZ` it reads `/etc/localtime`, named after `/etc/timezone` when that file names a zone, else `localtime`. When none of these gives a zone, it is UTC. It reads files on each call, so a program keeps the result.

**Gaps and overlaps.** Where the clocks move forward (a gap), some local times never happen; where they move back (an overlap), some happen twice. `in_zone_with` takes the choice, as java.time and Temporal do:

| `Disambiguation` | In an overlap (01:30 shown twice) | In a gap (02:30 skipped, 02:00 to 03:00) |
|------------------|-----------------------------------|-------------------------------------------|
| `Compatible`     | the earlier instant               | moved forward by the gap: 03:30           |
| `Earlier`        | the earlier instant               | read with the offset after the gap: 01:30 |
| `Later`          | the later instant                 | read with the offset before the gap: 03:30 |
| `Reject`         | throws `ValueError`               | throws `ValueError`                       |

`in_zone` is `in_zone_with(.., Disambiguation::Compatible)`, which is what most systems do and cannot fail. The `Reject` error names the zone, the two offsets and what to pass instead: `2024-03-10T02:30:00 does not exist in America/New_York: the clocks skip it (-05:00 to -04:00); pass Disambiguation::Earlier or Disambiguation::Later`. `add_days`, `add_months`, `add_years` and `+ Period` on a `Zoned` move the civil date, keep the time of day, and resolve the result as `Compatible`; `+ Duration` adds exact time.

`is_dst_at` and `Zoned::is_dst` report the zone data's daylight saving flag. Most zones mark summer time; a zone with negative DST marks its winter time instead (`IST-1GMT0,M10.5.0,M3.5.0/1`, the rule of Europe/Dublin in most tz builds).

**Equality.** Two `Zoned` values are equal, ordered and hashed by their instant only: `2024-01-01T09:00:00+09:00` equals `2024-01-01T00:00:00Z`. Two `TimeZone`s are equal when they have the same name.

### Example

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

## 3.3.8. Text: ISO 8601, RFC 3339 and patterns

`Display` writes, and `FromStr` (`str::parse`) reads, the standard forms:

| Type         | `Display`                               | `FromStr` also reads                          |
|--------------|-----------------------------------------|-----------------------------------------------|
| `Date`       | `2024-02-29`, `-000044-03-15`, `+012345-01-01` | |
| `Time`       | `09:05:00`, `09:05:00.250`              | `09:05`, a fraction of 1 to 9 digits, `,` for `.` |
| `DateTime`   | `2024-03-10T02:30:00`                   | `t` or a space for `T`                        |
| `Offset`     | `+09:00`, `-03:30`, `+05:30:15`         | `Z`, `+0900`, `+09`                           |
| `Zoned`      | `2024-03-10T03:30:00-04:00`, `...Z`     | RFC 9557's `[America/New_York]` suffix        |
| `SystemTime` | `2023-11-14T22:13:20Z` (UTC)            | any offset                                    |
| `Period`     | `P1Y2M10D`, `P0D`, `P-1M-3D`            | (no `FromStr`)                                |

A year outside 0..=9999 has a sign and six digits, as ISO 8601's expanded form. The fraction is written in 3, 6 or 9 digits, the fewest that hold it, and left out when it is zero. A `Zoned` with a zero offset writes `Z`; its `Debug` adds the zone name in brackets (`2024-03-10T03:30:00-04:00[America/New_York]`). Reading a `Zoned` with a bracketed zone loads that zone; the offset must agree with it, or the error names both (with no offset, the local time resolves as `Compatible`). Without a bracket, the zone is the fixed offset. A `DateTime` text with an offset is an error that says to parse a `Zoned` or a `SystemTime`.

**Errors name the place.** A parse error is a `ParseError` whose message names the byte offset and what belongs there: ``cannot parse "2024/02/03": expected `-` after the year at byte 4, found `/` ``, `cannot parse "2024-13-01": month 13 is out of range (1..=12) at byte 5`. `position()` is the same offset.

**Patterns.** `format(pattern)` on `Date`, `Time`, `DateTime` and `Zoned` writes strftime-style directives; `parse_with(text, pattern)` reads the same directives back. Names are English (the C locale).

| Directive | Meaning | Directive | Meaning |
|-----------|---------|-----------|---------|
| `%Y` | year, at least 4 digits | `%H` | hour 00..23 |
| `%C` | century (`%Y` / 100) | `%I` | hour 01..12 |
| `%y` | year % 100, 2 digits | `%k` / `%l` | `%H` / `%I` padded with a space |
| `%m` | month 01..12 | `%M` | minute 00..59 |
| `%d` | day 01..31 | `%S` | second 00..59 |
| `%e` | day, padded with a space | `%f` | nanoseconds, 9 digits; `%3f` `%6f` `%9f` |
| `%j` | day of the year 001..366 | `%p` / `%P` | `AM` / `PM`, `am` / `pm` |
| `%a` / `%A` | `Mon` / `Monday` | `%z` / `%:z` | offset `+0900` / `+09:00` |
| `%b` / `%h` / `%B` | `Jan` / `Jan` / `January` | `%Z` | abbreviation (`KST`), or the offset |
| `%u` / `%w` | weekday 1..7 from Monday / 0..6 from Sunday | `%s` | unix seconds |
| `%G` / `%g` / `%V` | ISO week year / its last 2 digits / ISO week | `%F` | `%Y-%m-%d` |
| `%U` / `%W` | week of the year from Sunday / Monday | `%T` | `%H:%M:%S` |
| `%D` / `%x` | `%m/%d/%y` | `%R` | `%H:%M` |
| `%c` | `%a %b %e %H:%M:%S %Y` | `%r` | `%I:%M:%S %p` |
| `%n` / `%t` | newline / tab | `%%` | `%` |

A flag after `%` changes the padding: `-` none (`%-d` is `5`), `_` spaces, `0` zeros. A pattern is program text, so an unknown directive, or one the value lacks (`%H` on a `Date`, `%z` on a `DateTime`), panics with a message that says what to write.

`parse_with` reads numbers of up to the directive's width (`%Y` up to 4 digits, or 6 after a sign), names in any case and in full or three letters, and lets one space in the pattern match any run of whitespace. Missing time fields are zero; a `Date` needs a year (`%Y`, or `%y` with 69..99 as 19xx), and a month and day or a `%j`. A weekday that does not match the date is an error. `%Z` reads `UTC`, `GMT`, `UT`, `Z` or a zone name such as `Europe/Paris`; abbreviations such as `EST` are ambiguous and rejected with a pointer to `%z`. `Zoned::parse_with` needs an offset, a zone or `%s`. `%C`, `%g`, `%G`, `%V`, `%U` and `%W` are write-only.

**Mail and HTTP.** `Zoned::to_rfc2822` and `Zoned::parse_rfc2822` are the date of RFC 5322 mail (`Sat, 14 Nov 2026 09:30:00 +0900`); parsing takes the weekday and the seconds as optional and the obsolete zone names (`GMT`, `EST`, `PDT`, ...). `SystemTime::to_http_date` is RFC 9110's IMF-fixdate (`Sun, 06 Nov 1994 08:49:37 GMT`), and `parse_http_date` also reads the two obsolete forms a server must accept (`Sunday, 06-Nov-94 08:49:37 GMT`, `Sun Nov  6 08:49:37 1994`).

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

### Example

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

## 3.3.9. Policy - no TAI / leap second support

`SystemTime` is strictly **UTC** and does not handle leap seconds (taking into account that the Linux kernel itself is ambiguous here). The calendar types follow it: every day has 86400 seconds, `Time` has no second 60, and parsing `23:59:60` is an error that says so. Cases that need TAI (atomic time) or exact leap second handling belong to separate packages.

| Use                        | Recommendation            |
|----------------------------|---------------------------|
| Ordinary apps (timestamps / measurement) | core::time, std::time |
| Simulation / precise astronomy | A separate `chrono`-like package |
| Finance / exact leap seconds | A separate package      |

## 3.3.10. impl / target

Per-platform implementation mapping:

| Item         | linux           | macos            | windows                 | wasi         |
|--------------|-----------------|------------------|-------------------------|--------------|
| `Instant`    | CLOCK_MONOTONIC | CLOCK_UPTIME_RAW | QueryPerformanceCounter | wasi clock   |
| `SystemTime` | CLOCK_REALTIME  | CLOCK_REALTIME   | GetSystemTimePreciseAsFileTime | wasi  |
| `BootTime`   | CLOCK_BOOTTIME  | CLOCK_MONOTONIC (mach_continuous_time, includes sleep) | QueryPerformanceCounter (~ Instant) | (no suspend) |
| `TimeZone::get` | `$TZDIR`, /usr/share/zoneinfo | /usr/share/zoneinfo | none: `UTC` and offsets only | the host's files when it grants them, else none |
| `TimeZone::local` | `$TZ`, /etc/localtime | `$TZ`, /etc/localtime | a POSIX rule in `$TZ`, else UTC | `$TZ` when passed in, else UTC |

## 3.3.11. Policy summary

The chapter's key decisions are five. First, `Instant` / `SystemTime` / `Duration` are separated, distinguishing monotonic time from wall-clock time semantically; `Duration` and `Instant` live in `core::time`, the OS clocks in `std::time`. Second, every span is the exact signed `Duration`, and `elapsed()` runs on a monotonic clock. Third, `BootTime`, which Rust's std does not cover, is exposed separately so that elapsed time since boot, including suspend, can be read directly. Fourth, TAI and leap seconds are split out into separate packages outside this module. Fifth, the calendar is in std, as in Python, Go and Java: civil types with no zone (`Date`, `Time`, `DateTime`) are kept apart from an instant in a zone (`Zoned`), a local time that a zone skips or repeats is resolved by an explicit `Disambiguation`, and the text forms are the standard ones (RFC 3339 for `Display`, strftime patterns, RFC 2822 and HTTP dates).
