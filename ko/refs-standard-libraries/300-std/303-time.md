# 3.3. `time::`: 시간

시간은 필요한 것에 따라 두 모듈로 나뉩니다. `core::time`에는 `Duration`과 `Instant`가 있습니다. 단조 시계는 이미 `core`에서 돌고 있으므로(태스크 타이머, 2.18) 두 타입은 모든 타깃에서 동작하고 `use` 없이 씁니다(0.2). `std::time`에는 운영체제가 필요한 시계가 있습니다. 벽시계 `SystemTime`, 부팅 시계 `BootTime`, `UNIX_EPOCH`입니다. 시간대는 운영체제의 tz 데이터베이스에서 오므로 달력도 여기에 있습니다. `Date`, `Time`, `DateTime`, `Period`, `Offset`, `TimeZone`, `Zoned`입니다(3.3.6~3.3.8). 타입마다 경로는 하나입니다([0.1.6](../000-meta/001-tier-model.md#016-항목마다-경로-하나)). `std::time`은 core 타입을 다시 내보내지 않습니다. 단조 시계, 벽시계, 부팅 후 경과 시간을 명확히 나눠 두었습니다. 그래서 측정 목적에 맞는 시계를 고를 수 있습니다.

## 3.3.1. 타입 4개 모델

전체 구성은 다음과 같습니다.

| 타입         | 모듈              | 의미                        | 사용처                   |
|--------------|-------------------|-----------------------------|--------------------------|
| `Instant`    | `core::time` | 단조 시계 (monotonic)       | 경과 시간 측정 / 타임아웃 |
| `Duration`   | `core::time` | 시간 차이 (i64 나노초)      | 둘 사이 차 / sleep 인자  |
| `SystemTime` | `std::time`  | 벽시계 (UTC, NTP 가능)      | 파일 mtime / 사용자 표시 |
| `BootTime`   | `std::time`  | 부팅 후 시간 (절전 포함)    | 모바일 / 노트북 앱       |

`Duration`의 내부 표현은 `i64` ns입니다. Rust가 채택한 `u128`은 과하다고 판단했습니다. `i64`로도 약 ±292년을 표현할 수 있으므로 실용 범위로는 충분합니다.

**모든 시계에 같은 규칙.** 어느 시계든 두 값 사이의 간격은 부호 있는 정확한 `Duration`입니다. `b`가 더 나중이면 `a.duration_since(b)`는 음수입니다. 에러도, `none`도, 0도 아닙니다. `elapsed()`는 단조 시계로 재므로 벽시계가 바뀌어도 음수가 되지 않습니다. Go의 규칙(`time.Sub`, `time.Since`)과 같습니다.

## 3.3.2. `Instant`: 단조 시계

```nabi
// core::time
struct Instant { /* opaque (CLOCK_MONOTONIC / QueryPerformanceCounter) */ }

impl Instant {
    fn now() -> Instant;

    fn elapsed(self) -> Duration;
    const fn duration_since(self, earlier: Instant) -> Duration;   // self - earlier, 부호 있음

    // 산술: 실패 시 none 반환 (Rust 의 saturating-on-overflow silent 회피)
    const fn checked_add(self, dur: Duration) -> Instant | none;
    const fn checked_sub(self, dur: Duration) -> Instant | none;
}

// 일반 산술. overflow 시 overflow 검사가 켜져 있으면 panic, 꺼져 있으면 saturate (3.3.5)
impl Add<Duration> for Instant { type Output = Instant; ... }
impl Sub<Instant> for Instant { type Output = Duration; ... }
impl Sub<Duration> for Instant { type Output = Instant; ... }
```

`Instant`의 핵심 특성은 다음과 같습니다. 절대 뒤로 가지 않으므로 NTP 보정의 영향을 받지 않습니다. 기준이 프로세스 시작 시점이라 그 자체로는 절대 시간의 의미가 없습니다. 그래서 **직렬화할 수 없습니다**. 같은 이유로 프로세스 사이의 비교도 무의미합니다. 같은 프로세스 안에서만 의미가 있습니다.

`earlier`가 더 나중이면 `duration_since`는 음수 `Duration`을 돌려줍니다. 간격을 0 아래로 내려가지 않게 하려면 비교합니다. `d.max(Duration::ZERO)`.

### 사용 예

가장 흔한 패턴은 시작점을 잡아 두고 `elapsed`로 경과 시간을 얻는 방식입니다.

```nabi
let start = Instant::now();
heavy_computation();
let elapsed = start.elapsed();          // Duration
log(f"computation took {elapsed:?}");
```

## 3.3.3. `SystemTime`: 벽시계

```nabi
// std::time
struct SystemTime { /* opaque (CLOCK_REALTIME / GetSystemTimePreciseAsFileTime) */ }

impl SystemTime {
    fn now() -> SystemTime;

    const fn duration_since(self, earlier: SystemTime) -> Duration;   // self - earlier, 부호 있음
    fn elapsed(self) -> Duration;                                    // 단조 시계로 잼

    const fn checked_add(self, dur: Duration) -> SystemTime | none;
    const fn checked_sub(self, dur: Duration) -> SystemTime | none;
}

const UNIX_EPOCH: SystemTime;     // 1970-01-01 00:00:00 UTC

// SystemTime ↔ unix timestamp
impl SystemTime {
    const fn from_unix(secs: i64, nanos: u32) -> SystemTime;
    const fn unix_secs(self) -> i64;    // 내림: epoch 이전이면 음수
    const fn unix_nanos(self) -> i64;   // nabi 에 i128 이 없음. i64 ns 는 약 ±292 년
    const fn subsec_nanos(self) -> u32; // unix_secs() 뒤의 나노초
}

// 텍스트 (3.3.8)
impl SystemTime {
    const fn to_http_date(self) -> str;                              // "Sun, 06 Nov 1994 08:49:37 GMT"
    fn parse_http_date(text: str) throws (ParseError) -> SystemTime;
}
impl Display for SystemTime { ... }     // UTC 의 RFC 3339: "2023-11-14T22:13:20Z". Debug 도 같음
impl FromStr for SystemTime { ... }     // 오프셋은 무엇이든 RFC 3339

impl Add<Duration> for SystemTime { type Output = SystemTime; ... }
impl Sub<Duration> for SystemTime { type Output = SystemTime; ... }
```

`SystemTime`은 `UNIX_EPOCH` 기준 `i64` 나노초입니다. 그래서 표현 범위는 모든 타깃에서 약 ±292년입니다. 32비트 타깃도 64비트 시간 인터페이스(`time64`)로 시계를 읽으므로 2038년 한계가 없습니다.

`SystemTime`의 성격은 꽤 다릅니다. NTP 보정이나 사용자 변경으로 시간이 뒤로 갈 수 있습니다. UNIX_EPOCH 기준의 절대 시간이므로 **직렬화할 수 있습니다**. 그래서 프로세스 사이에서도 의미 있게 비교할 수 있습니다.

`duration_since`는 벽시계 값의 부호 있는 차입니다. `earlier`가 더 나중이면 에러가 아니라 음수 `Duration`이 나옵니다. `unix_secs`는 내림하므로 epoch 이전에도 `SystemTime::from_unix(t.unix_secs(), n)`이 초 아래 부분 `n`과 함께 `t`를 그대로 돌려줍니다.

**단조 시계 값.** `SystemTime::now()`는 단조 시계(`Instant`의 시계)도 함께 읽습니다. `elapsed()`는 이 값으로 잽니다. 그래서 `now()`와 `elapsed()` 사이에 벽시계가 바뀌어도 결과는 달라지지 않습니다. `Duration`과의 `+` / `-` / `checked_*`는 두 값을 함께 옮깁니다. `now()`에서 오지 않은 `SystemTime`(`from_unix`, `UNIX_EPOCH`, `Metadata::modified`)에는 단조 시계 값이 없고, 그 `elapsed()`는 `SystemTime::now().duration_since(self)`입니다. 비교, 같음, 해시는 벽시계 값만 봅니다.

### 사용 예

Unix 타임스탬프 변환과 파일 mtime 조회는 다음과 같습니다.

```nabi
use std::time::SystemTime;

let now = SystemTime::now();
log(f"unix timestamp: {now.unix_secs()}");

// 파일 mtime
let meta = try fs::metadata("data.txt");
let mtime: SystemTime = try meta.modified();
log(f"file age: {mtime.elapsed():?}");     // mtime 이 미래면 음수
```

## 3.3.4. `BootTime`: 부팅 후 시간(suspend 포함)

```nabi
// std::time
struct BootTime { /* CLOCK_BOOTTIME (Linux) / CLOCK_MONOTONIC = mach_continuous_time (macOS) */ }

impl BootTime {
    fn now() -> BootTime;
    fn elapsed(self) -> Duration;
    const fn duration_since(self, earlier: BootTime) -> Duration;   // self - earlier, 부호 있음
}
```

`Instant`처럼 `earlier`가 더 나중이면 `duration_since`는 음수입니다. `BootTime`과 `Instant`는 절전(suspend) 동안의 동작이 다릅니다. `Instant`는 Linux `CLOCK_MONOTONIC`처럼 절전 동안 멈춥니다. `BootTime`은 `CLOCK_BOOTTIME`을 따라 절전 중에도 계속 흐릅니다.

이 차이는 모바일이나 노트북 앱이 "실제 경과 시간(절전 포함)"을 측정해야 할 때 결정적입니다. Rust에는 2026년 현재도 이를 위한 안정 API가 없습니다. Nabi는 처음부터 노출합니다.

## 3.3.5. `Duration`

```nabi
// core::time
struct Duration { /* opaque: i64 nanosecond (약 ±292년) */ }

impl Duration {
    const ZERO: Duration;             // 0 ns
    const MAX: Duration;              // I64_MAX ns

    // 생성자: i64로 손실 없이 바뀌는 정수 타입이면 무엇이든
    // (i8 .. i64, u8 .. u32), u64 / usize는 try_into
    const fn ns<N>(n: N) -> Duration where i64: From<N>;
    const fn us<N>(n: N) -> Duration where i64: From<N>;        // microseconds
    const fn ms<N>(n: N) -> Duration where i64: From<N>;        // milliseconds
    const fn secs<N>(n: N) -> Duration where i64: From<N>;
    const fn mins<N>(n: N) -> Duration where i64: From<N>;
    const fn hours<N>(n: N) -> Duration where i64: From<N>;
    const fn days<N>(n: N) -> Duration where i64: From<N>;

    const fn secs_f64(s: f64) -> Duration;  // 부동소수 초, ns 단위로 버림

    // 추출
    const fn as_ns(self) -> i64;
    const fn as_us(self) -> i64;
    const fn as_ms(self) -> i64;
    const fn as_secs(self) -> i64;
    const fn as_secs_f64(self) -> f64;

    const fn subsec_ns(self) -> u32;          // 하위 nanosecond 부분
    const fn subsec_us(self) -> u32;
    const fn subsec_ms(self) -> u32;
}

// 산술
impl Add for Duration { ... }
impl Sub for Duration { ... }
impl Mul<i64> for Duration { ... }     // Duration × i64
impl Div<i64> for Duration { ... }
impl Mul<Duration> for f64 { ... }      // f64 × Duration

impl Display for Duration { ... }       // "1.234s" / "5ms" / "1.5µs" / "100ns"
impl Debug for Duration { ... }         // Display 와 같음
```

두 duration의 `a.min(b)`는 `Ord::min`입니다.

`Display`는 크기에 맞는 단위 하나로 씁니다. 절댓값이 1초 이상이면 `s`, 1 ms 이상이면 `ms`, 1 µs 이상이면 `µs`, 그 밖에는 `ns`입니다. 그 단위로 나눈 정수부 뒤에 나머지를 소수로 붙입니다. 끝의 0은 지우고, 나머지가 0이면 소수점도 쓰지 않습니다. 음수는 앞에 `-`를 붙입니다.

```nabi
Duration::ms(1234);      // "1.234s"
Duration::ms(5);         // "5ms"
Duration::ns(1500);      // "1.5µs"
Duration::ns(-20);       // "-20ns"
```

시간 타입은 모두 `Clone` / `Copy` / `PartialEq` / `Eq` / `PartialOrd` / `Ord` / `Hash` / `Debug`를 구현합니다. `Duration`은 `Default`(0)도 구현합니다.

### Duration 산술 정책

`Duration` / `SystemTime` / `Instant` / `BootTime`의 산술(`duration_since`, `from_unix` 포함)은 오버플로 검사 설정을 따릅니다(`-C overflow-checks`, [refs-lang 9.1.2](../../refs-lang/009-tools/901-nabic-cli.md#nabic-build-aot-빌드)). 검사가 켜져 있으면 오버플로에서 panic이 일어나고, 꺼져 있으면 포화합니다. `f64` 형태(`Duration::secs_f64`, `f64 * Duration`)도 같은 규칙을 따릅니다. 결과가 유한하지 않거나 범위를 벗어나면 검사가 켜져 있을 때 panic이 일어나고, 꺼져 있을 때 포화합니다(NaN은 0). 0으로 나누면 항상 panic이 일어납니다. `Duration`의 `checked_*` / `saturating_*` 변형으로 동작을 명시적으로 고를 수 있습니다.

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

## 3.3.6. 달력 날짜와 시각

달력 타입에는 시간대가 없습니다. 벽에 걸린 달력과 시계가 보여 주는 값입니다. 모두 `Copy` 값이고 시간 순으로 정렬되며 `Hash`를 구현합니다. `Display` / `FromStr`은 ISO 8601입니다(3.3.8).

| 타입       | 담는 것                                 | `Display`               |
|------------|-----------------------------------------|-------------------------|
| `Date`     | 연, 월, 일 (역산 그레고리력)            | `2024-02-29`            |
| `Time`     | 시, 분, 초, 나노초                      | `09:05:00.25`           |
| `DateTime` | `Date`와 `Time`                         | `2024-03-10T02:30:00`   |
| `Period`   | 연, 월, 일을 따로따로                   | `P1Y2M10D`              |
| `Weekday`  | 월요일 .. 일요일                        | `Monday`                |

```nabi
// std::time
const MIN_YEAR: i32;    // -999999
const MAX_YEAR: i32;    // 999999
const fn is_leap_year(year: i32) -> bool;
const fn days_in_month(year: i32, month: i32) -> i32;   // 1..=12 밖의 월은 0

enum Weekday { Monday, Tuesday, Wednesday, Thursday, Friday, Saturday, Sunday }
impl Weekday {
    const fn number(self) -> i32;                    // 1 = 월요일 .. 7 = 일요일 (ISO 8601)
    const fn from_number(number: i32) -> Weekday | none;
    const fn name(self) -> str;                      // "Monday"
    const fn next(self) -> Weekday;
    const fn prev(self) -> Weekday;
}

struct Date { /* 연, 월, 일 */ }
impl Date {
    const MIN: Date;    // -999999-01-01
    const MAX: Date;    // +999999-12-31

    const fn new(year: i32, month: i32, day: i32) throws (ValueError) -> Date;
    const fn from_ordinal(year: i32, ordinal: i32) throws (ValueError) -> Date;
    const fn from_iso_week(year: i32, week: i32, weekday: Weekday) throws (ValueError) -> Date;
    const fn from_unix_days(days: i64) -> Date;      // 1970-01-01 부터의 일수. MIN..=MAX 밖이면 panic
    const fn unix_days(self) -> i64;

    const fn year(self) -> i32;                      // 0 은 기원전 1 년
    const fn month(self) -> i32;                     // 1..=12
    const fn day(self) -> i32;
    const fn weekday(self) -> Weekday;
    const fn ordinal(self) -> i32;                   // 연중 일, 1..=366
    const fn iso_week(self) -> (i32, i32);           // (ISO 연도, 주 1..=53)
    const fn is_leap_year(self) -> bool;
    const fn days_in_month(self) -> i32;
    const fn days_in_year(self) -> i32;

    const fn with_day(self, day: i32) throws (ValueError) -> Date;
    const fn first_of_month(self) -> Date;
    const fn last_of_month(self) -> Date;

    const fn add_days(self, n: i64) -> Date;         // MIN..=MAX 밖이면 panic
    const fn add_months(self, n: i64) -> Date;       // 일을 맞춤
    const fn add_years(self, n: i64) -> Date;        // 2 월 29 일을 맞춤
    const fn checked_add_days(self, n: i64) -> Date | none;
    const fn checked_add_months(self, n: i64) -> Date | none;
    const fn days_since(self, earlier: Date) -> i64; // earlier 가 더 나중이면 음수
    const fn until(self, later: Date) -> Period;
    const fn and_time(self, time: Time) -> DateTime;
}
impl Add<Period> for Date { ... }
impl Sub<Period> for Date { ... }

struct Time { /* 시, 분, 초, 나노초 */ }
impl Time {
    const MIDNIGHT: Time;
    const fn new(hour: i32, minute: i32, second: i32) throws (ValueError) -> Time;
    const fn with_nanosecond(self, nano: i32) throws (ValueError) -> Time;
    const fn from_nanos_of_day(ns: i64) -> Time;     // 하루 밖이면 panic
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
    const fn from_unix(secs: i64, nano: u32) -> DateTime;   // unix 시각의 UTC 달력 시각
    const fn unix_secs(self) -> i64;                         // 그 반대, self 를 UTC 로 읽음
    const fn date(self) -> Date;
    const fn time(self) -> Time;
    // year() .. nanosecond() 는 Date, Time 과 같음
    const fn add_days(self, n: i64) -> DateTime;
    const fn add_months(self, n: i64) -> DateTime;
    const fn add_years(self, n: i64) -> DateTime;
    const fn duration_since(self, earlier: DateTime) -> Duration;
    fn in_zone(self, tz: TimeZone) -> Zoned;                                     // 3.3.7
    fn in_zone_with(self, tz: TimeZone, choice: Disambiguation) throws (ValueError) -> Zoned;
}
impl Add<Duration> for DateTime { ... }   // 정확한 시간: 하루는 늘 86400 초
impl Sub<Duration> for DateTime { ... }
impl Add<Period> for DateTime { ... }     // 날짜만 움직이고 시각은 그대로
impl Sub<Period> for DateTime { ... }

struct Period { /* 연, 월, 일 */ }
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
// Period + Period, Period - Period, -Period. Default 는 ZERO
```

**범위와 달력.** `Date`는 `MIN_YEAR`부터 `MAX_YEAR`까지 역산 그레고리력(proleptic Gregorian)의 하루입니다. ISO 8601처럼 0년은 기원전 1년입니다. 일수 계산은 Howard Hinnant의 civil 알고리즘을 쓰며 전체 범위에서 정확합니다. `unix_days`는 1970-01-01을 0일로 셉니다.

**잘못된 필드는 throw, 넘치는 산술은 panic.** 날짜나 시각이 아닌 필드를 받은 생성자(`Date::new(2023, 2, 29)`, `Time::new(24, 0, 0)`)는 그 필드와 범위를 밝히는 `ValueError`를 throw합니다. `day 29 is out of range for 2023-02 (1..=28)`처럼 나옵니다. 범위를 벗어나는 산술(`Date::MAX.add_days(1)`)은 `Duration`처럼 panic이 일어나고, `checked_*` 형태는 대신 `none`을 돌려줍니다.

**월은 맞춥니다.** `add_months`와 `add_years`는 새 월에 같은 날이 있으면 그날을, 없으면 그 달의 마지막 날을 씁니다. 2024-01-31에 한 달을 더하면 2024-02-29이고, 2024-02-29에 1년을 더하면 2025-02-28입니다. 호출마다 따로 맞추므로 1월 31일에서 `add_months(1)`을 두 번 하면 3월 29일이고, `add_months(2)`는 3월 31일입니다. `Date + Period`는 연과 월을 함께 더해 한 번 맞춘 뒤 일을 더합니다. java.time과 Temporal의 규칙과 같습니다.

**`until`**은 `self`부터 `later`까지의 달력 간격입니다. 온전한 연, 그다음 온전한 월, 그다음 일이며 모두 간격의 부호를 따릅니다. 어떤 두 날짜든 `d + d.until(e) == e`가 되도록 고릅니다. `days_since`는 그냥 일수 차입니다.

**ISO 주.** `iso_week`는 ISO 8601의 주 번호입니다. 주는 월요일에 시작하고 1주에는 그해 첫 목요일이 들어갑니다. 그래서 2021-01-03은 2020년 53주, 2024-12-30은 2025년 1주입니다.

### 사용 예

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

## 3.3.7. 오프셋, 시간대, `Zoned`

`Offset`은 UTC와의 고정된 차이입니다. `TimeZone`은 순간마다 오프셋을 알려 줍니다. `Zoned`는 한 순간과 그 순간을 보여 주는 시간대를 함께 담습니다.

```nabi
// std::time
struct Offset { /* UTC 동쪽으로 초, ±25:59:59 안 */ }
impl Offset {
    const UTC: Offset;
    const fn new(hours: i32, minutes: i32) throws (ValueError) -> Offset;   // (-3, -30) 은 -03:30
    const fn from_secs(secs: i32) throws (ValueError) -> Offset;
    const fn secs(self) -> i32;
    const fn is_utc(self) -> bool;
}

struct TimeZone { /* 이름과 규칙. 복사본은 표를 함께 씀 */ }
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

struct Zoned { /* 순간, 시간대, 그곳의 달력 시각과 오프셋 */ }
impl Zoned {
    fn now(tz: TimeZone) -> Zoned;
    fn from_system_time(t: SystemTime, tz: TimeZone) -> Zoned;
    fn from_unix(secs: i64, nano: u32, tz: TimeZone) -> Zoned;

    const fn unix_secs(self) -> i64;
    const fn unix_nanos(self) -> i64;            // 1678..2262 밖이면 panic
    const fn to_system_time(self) -> SystemTime; // 1678..2262 밖이면 panic
    const fn datetime(self) -> DateTime;
    const fn date(self) -> Date;
    const fn time(self) -> Time;
    const fn offset(self) -> Offset;
    const fn zone(self) -> TimeZone;
    const fn abbreviation(self) -> str;
    const fn is_dst(self) -> bool;
    // year() .. nanosecond(), weekday() 는 DateTime 과 같음

    fn in_zone(self, tz: TimeZone) -> Zoned;     // 같은 순간, 다른 시간대
    fn to_utc(self) -> Zoned;
    fn add_days(self, n: i64) -> Zoned;          // 달력의 날: 벽시계 시각은 그대로
    fn add_months(self, n: i64) -> Zoned;
    fn add_years(self, n: i64) -> Zoned;
    const fn duration_since(self, earlier: Zoned) -> Duration;
}
impl Add<Duration> for Zoned { ... }   // 정확한 시간: 전환 시점에서 벽시계 시각이 뛸 수 있음
impl Sub<Duration> for Zoned { ... }
impl Add<Period> for Zoned { ... }     // 달력 시간: add_months 다음 add_days
impl Sub<Period> for Zoned { ... }
```

**시간대를 어디서 읽나.** `TimeZone::get`은 tz 데이터베이스(IANA)의 시간대를 TZif 파일(RFC 8536, 버전 1~4)로 읽습니다. `$TZDIR`가 있으면 거기서, 없으면 `/usr/share/zoneinfo`, `/usr/lib/zoneinfo`, `/usr/share/lib/zoneinfo`, `/etc/zoneinfo` 가운데 먼저 있는 곳에서 읽습니다. 파일의 마지막 전환 뒤로는 파일 끝(footer)의 POSIX `TZ` 규칙이 오프셋을 정하므로 먼 미래의 날짜도 현재 규칙을 따릅니다. `UTC`, 다른 UTC 이름(`Etc/UTC`, `GMT`, `Zulu`, ...)과 오프셋(`+09:00`)은 데이터베이스 없이 동작합니다. 그 밖의 이름은 데이터베이스가 없으면 throw하고, 에러는 이 시스템에 데이터베이스가 없다고 알려 줍니다. Windows와 브라우저에는 tz 데이터베이스가 없고, WASI 프로그램은 호스트가 디렉터리를 열어 줄 때만 봅니다. `from_tzif`는 프로그램이 직접 가져온 TZif 파일(예: `include_bytes!`)을 읽으며 모든 타깃에서 동작합니다. `from_posix`는 POSIX `TZ` 규칙(`EST5EDT,M3.2.0,M11.1.0`)을 받습니다. 그 오프셋은 POSIX의 표기대로 서쪽이 양수입니다.

**지역 시간대.** `TimeZone::local()`은 먼저 `$TZ`를 읽습니다. 데이터베이스 이름(`Asia/Seoul`), TZif 파일 경로(`/usr/share/zoneinfo/Asia/Seoul`, 앞의 `:`는 뗌), POSIX 규칙 가운데 하나입니다. `$TZ`가 없으면 `/etc/localtime`을 읽습니다. 이름은 `/etc/timezone`이 시간대를 밝히면 그것, 아니면 `localtime`입니다. 아무것도 시간대를 주지 않으면 UTC입니다. 호출할 때마다 파일을 읽으므로 프로그램은 결과를 보관해 둡니다.

**건너뜀(gap)과 겹침(overlap).** 시계가 앞으로 가는 곳(gap)에서는 어떤 지역 시각이 아예 없고, 뒤로 가는 곳(overlap)에서는 두 번 있습니다. `in_zone_with`는 java.time, Temporal처럼 선택을 받습니다.

| `Disambiguation` | 겹침에서 (01:30이 두 번) | 건너뜀에서 (02:30을 건너뜀, 02:00에서 03:00으로) |
|------------------|--------------------------|---------------------------------------------------|
| `Compatible`     | 이른 순간                | 건너뛴 만큼 앞으로: 03:30                         |
| `Earlier`        | 이른 순간                | 건너뜀 뒤의 오프셋으로 읽음: 01:30                |
| `Later`          | 늦은 순간                | 건너뜀 앞의 오프셋으로 읽음: 03:30                |
| `Reject`         | `ValueError`를 throw     | `ValueError`를 throw                              |

`in_zone`은 `in_zone_with(.., Disambiguation::Compatible)`입니다. 대부분의 시스템이 이렇게 하며 실패하지 않습니다. `Reject`의 에러는 시간대, 두 오프셋, 대신 넘길 값을 밝힙니다. `2024-03-10T02:30:00 does not exist in America/New_York: the clocks skip it (-05:00 to -04:00); pass Disambiguation::Earlier or Disambiguation::Later`처럼 나옵니다. `Zoned`의 `add_days`, `add_months`, `add_years`, `+ Period`는 달력 날짜를 옮기고 시각은 그대로 두며 결과를 `Compatible`로 정합니다. `+ Duration`은 정확한 시간을 더합니다.

`is_dst_at`과 `Zoned::is_dst`는 시간대 데이터의 일광 절약 시간 표시를 알려 줍니다. 대부분의 시간대는 여름 시간을 표시하지만, 음수 DST를 쓰는 시간대는 겨울 시간을 표시합니다(`IST-1GMT0,M10.5.0,M3.5.0/1`, 대부분의 tz 빌드에서 Europe/Dublin의 규칙).

**같음.** 두 `Zoned`는 순간만으로 같음, 순서, 해시를 정합니다. `2024-01-01T09:00:00+09:00`과 `2024-01-01T00:00:00Z`는 같습니다. 두 `TimeZone`은 이름이 같으면 같습니다.

### 사용 예

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

## 3.3.8. 텍스트: ISO 8601, RFC 3339, 패턴

`Display`는 표준 형식으로 쓰고 `FromStr`(`str::parse`)은 읽습니다.

| 타입         | `Display`                               | `FromStr`이 더 읽는 것                        |
|--------------|-----------------------------------------|-----------------------------------------------|
| `Date`       | `2024-02-29`, `-000044-03-15`, `+012345-01-01` | |
| `Time`       | `09:05:00`, `09:05:00.250`              | `09:05`, 1~9자리 소수, `.` 대신 `,`           |
| `DateTime`   | `2024-03-10T02:30:00`                   | `T` 대신 `t`나 공백                           |
| `Offset`     | `+09:00`, `-03:30`, `+05:30:15`         | `Z`, `+0900`, `+09`                           |
| `Zoned`      | `2024-03-10T03:30:00-04:00`, `...Z`     | RFC 9557의 `[America/New_York]` 꼬리          |
| `SystemTime` | `2023-11-14T22:13:20Z` (UTC)            | 아무 오프셋                                   |
| `Period`     | `P1Y2M10D`, `P0D`, `P-1M-3D`            | (`FromStr` 없음)                              |

0..=9999 밖의 연도는 ISO 8601의 확장 형식처럼 부호와 여섯 자리로 씁니다. 소수 부분은 그 값을 담는 가장 짧은 3, 6, 9자리로 쓰고 0이면 뺍니다. 오프셋이 0인 `Zoned`는 `Z`로 씁니다. `Debug`는 괄호에 시간대 이름을 덧붙입니다(`2024-03-10T03:30:00-04:00[America/New_York]`). 괄호에 시간대가 있는 `Zoned`를 읽으면 그 시간대를 불러옵니다. 오프셋은 그 시간대와 맞아야 하며, 맞지 않으면 에러가 둘 다 밝힙니다(오프셋이 없으면 지역 시각을 `Compatible`로 정합니다). 괄호가 없으면 시간대는 그 고정 오프셋입니다. 오프셋이 붙은 텍스트를 `DateTime`으로 읽으면 `Zoned`나 `SystemTime`으로 읽으라는 에러가 납니다.

**에러는 자리를 밝힙니다.** 파싱 에러는 `ParseError`이고, 메시지는 바이트 위치와 그 자리에 와야 할 것을 밝힙니다. ``cannot parse "2024/02/03": expected `-` after the year at byte 4, found `/` ``, `cannot parse "2024-13-01": month 13 is out of range (1..=12) at byte 5`처럼 나옵니다. `position()`도 같은 위치입니다.

**패턴.** `Date`, `Time`, `DateTime`, `Zoned`의 `format(pattern)`은 strftime 형식의 지시자를 씁니다. `parse_with(text, pattern)`은 같은 지시자를 다시 읽습니다. 이름은 영어입니다(C 로캘).

| 지시자 | 뜻 | 지시자 | 뜻 |
|--------|----|--------|----|
| `%Y` | 연도, 4자리 이상 | `%H` | 시 00..23 |
| `%C` | 세기 (`%Y` / 100) | `%I` | 시 01..12 |
| `%y` | 연도 % 100, 2자리 | `%k` / `%l` | 공백으로 채운 `%H` / `%I` |
| `%m` | 월 01..12 | `%M` | 분 00..59 |
| `%d` | 일 01..31 | `%S` | 초 00..59 |
| `%e` | 일, 공백으로 채움 | `%f` | 나노초 9자리, `%3f` `%6f` `%9f` |
| `%j` | 연중 일 001..366 | `%p` / `%P` | `AM` / `PM`, `am` / `pm` |
| `%a` / `%A` | `Mon` / `Monday` | `%z` / `%:z` | 오프셋 `+0900` / `+09:00` |
| `%b` / `%h` / `%B` | `Jan` / `Jan` / `January` | `%Z` | 약어(`KST`), 없으면 오프셋 |
| `%u` / `%w` | 요일, 월요일부터 1..7 / 일요일부터 0..6 | `%s` | unix 초 |
| `%G` / `%g` / `%V` | ISO 주 연도 / 그 끝 2자리 / ISO 주 | `%F` | `%Y-%m-%d` |
| `%U` / `%W` | 일요일 / 월요일부터 센 연중 주 | `%T` | `%H:%M:%S` |
| `%D` / `%x` | `%m/%d/%y` | `%R` | `%H:%M` |
| `%c` | `%a %b %e %H:%M:%S %Y` | `%r` | `%I:%M:%S %p` |
| `%n` / `%t` | 줄바꿈 / 탭 | `%%` | `%` |

`%` 뒤의 플래그는 채움을 바꿉니다. `-`는 채우지 않고(`%-d`는 `5`), `_`는 공백, `0`은 0으로 채웁니다. 패턴은 프로그램 텍스트이므로 모르는 지시자나 값에 없는 지시자(`Date`의 `%H`, `DateTime`의 `%z`)는 무엇을 쓸지 알려 주는 메시지와 함께 panic이 일어납니다.

`parse_with`는 숫자를 지시자의 폭까지(`%Y`는 4자리, 부호 뒤에서는 6자리까지) 읽고, 이름은 대소문자와 상관없이 전체나 세 글자로 읽으며, 패턴의 공백 하나는 이어진 공백 전체와 맞습니다. 빠진 시각 필드는 0입니다. `Date`에는 연도(`%Y`, 또는 69..99를 19xx로 읽는 `%y`)와 월과 일, 또는 `%j`가 있어야 합니다. 날짜와 맞지 않는 요일은 에러입니다. `%Z`는 `UTC`, `GMT`, `UT`, `Z`나 `Europe/Paris` 같은 시간대 이름을 읽습니다. `EST` 같은 약어는 모호하므로 `%z`를 쓰라는 안내와 함께 거부합니다. `Zoned::parse_with`에는 오프셋, 시간대, `%s` 가운데 하나가 있어야 합니다. `%C`, `%g`, `%G`, `%V`, `%U`, `%W`는 쓰기 전용입니다.

**메일과 HTTP.** `Zoned::to_rfc2822`와 `Zoned::parse_rfc2822`는 RFC 5322 메일의 날짜입니다(`Sat, 14 Nov 2026 09:30:00 +0900`). 파싱은 요일과 초를 생략할 수 있게 받고 옛 시간대 이름(`GMT`, `EST`, `PDT`, ...)도 읽습니다. `SystemTime::to_http_date`는 RFC 9110의 IMF-fixdate(`Sun, 06 Nov 1994 08:49:37 GMT`)이고, `parse_http_date`는 서버가 받아야 하는 옛 형식 두 가지(`Sunday, 06-Nov-94 08:49:37 GMT`, `Sun Nov  6 08:49:37 1994`)도 읽습니다.

```nabi
impl Date {
    const fn format(self, pattern: str) -> str;
    fn parse_with(text: str, pattern: str) throws (ParseError) -> Date;
}
// Time, DateTime, Zoned 도 같음
impl Zoned {
    const fn to_rfc2822(self) -> str;
    fn parse_rfc2822(text: str) throws (ParseError) -> Zoned;
}
```

### 사용 예

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

## 3.3.9. 정책: TAI / 윤초 미지원

`SystemTime`은 엄밀히 **UTC**이고, 윤초(leap second)는 처리하지 않습니다. Linux 커널 자체가 이 부분에서 모호하다는 점도 고려했습니다. 달력 타입도 이를 따릅니다. 하루는 늘 86400초이고, `Time`에는 60초가 없으며, `23:59:60`을 파싱하면 그렇다고 알려 주는 에러가 납니다. TAI(원자시)나 정확한 윤초 처리가 필요하면 별도 패키지를 씁니다.

| 자리                       | 권장                      |
|----------------------------|---------------------------|
| 일반 앱 (타임스탬프 / 측정) | core::time, std::time |
| 시뮬레이션 / 정밀 천체     | 별도 `chrono`-like 패키지 |
| 금융 / 정확한 윤초         | 별도 패키지               |

## 3.3.10. 구현 / 타깃

플랫폼별 구현 매핑은 다음과 같습니다.

| 항목         | linux           | macos            | windows                 | wasi         |
|--------------|-----------------|------------------|-------------------------|--------------|
| `Instant`    | CLOCK_MONOTONIC | CLOCK_UPTIME_RAW | QueryPerformanceCounter | wasi clock   |
| `SystemTime` | CLOCK_REALTIME  | CLOCK_REALTIME   | GetSystemTimePreciseAsFileTime | wasi  |
| `BootTime`   | CLOCK_BOOTTIME  | CLOCK_MONOTONIC (mach_continuous_time, 절전 포함) | QueryPerformanceCounter (~ Instant) | (절전 없음) |
| `TimeZone::get` | `$TZDIR`, /usr/share/zoneinfo | /usr/share/zoneinfo | 없음: `UTC`와 오프셋만 | 호스트가 파일을 열어 주면 그 파일, 아니면 없음 |
| `TimeZone::local` | `$TZ`, /etc/localtime | `$TZ`, /etc/localtime | `$TZ`의 POSIX 규칙, 아니면 UTC | `$TZ`가 넘어오면 그것, 아니면 UTC |

## 3.3.11. 정책 요약

이 장의 핵심 결정은 다섯 가지입니다. 첫째, `Instant` / `SystemTime` / `Duration`을 나눠 단조 시간과 벽시계 시간을 의미상 구분합니다. `Duration`과 `Instant`는 `core::time`에, OS 시계는 `std::time`에 있습니다. 둘째, 모든 간격은 부호 있는 정확한 `Duration`이고 `elapsed()`는 단조 시계로 잽니다. 셋째, Rust의 std에 없는 `BootTime`을 따로 노출합니다. 그래서 절전을 포함한 부팅 후 경과 시간을 직접 읽을 수 있습니다. 넷째, TAI와 윤초는 이 모듈 밖의 별도 패키지로 나눕니다. 다섯째, Python, Go, Java처럼 달력을 std에 둡니다. 시간대가 없는 타입(`Date`, `Time`, `DateTime`)과 시간대 안의 한 순간(`Zoned`)을 나누고, 시간대가 건너뛰거나 두 번 보여 주는 지역 시각은 명시적인 `Disambiguation`으로 정하며, 텍스트는 표준 형식(`Display`의 RFC 3339, strftime 패턴, RFC 2822와 HTTP 날짜)을 씁니다.
