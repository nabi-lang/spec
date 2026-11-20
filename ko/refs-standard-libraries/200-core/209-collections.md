# 2.9. 컬렉션: `List` / `Dict` / `Set` / `Tuple` / `Slice`

`core::collections` 모듈은 범용 컨테이너를 정의합니다. 모든 컬렉션은 GC 힙에 할당되고, `List`, `Dict`, `Set`은 `#[literal(...)]`로 묶은 리터럴 문법으로 바로 만들 수 있습니다.

이 장은 다음 컬렉션과 `Deque<T>`, `BinaryHeap<T>`의 시그니처를 정의합니다.

| 컬렉션              | 리터럴 문법                | 의미                          |
|---------------------|----------------------------|-------------------------------|
| `List<T>`           | `[1, 2, 3]`                | 동적 배열(삽입 순서)          |
| `Dict<K, V, S>`     | `{"a": 1, "b": 2}`         | 해시 맵(삽입 순서)            |
| `Set<T, S>`         | `{1, 2, 3}`                | 해시 집합(삽입 순서)          |
| `Tuple<...>`        | `(1, "a", true)`           | 고정 크기, 원소 타입이 제각각 |
| `Slice<T>`          | `xs[1:3]`                  | 복사 없는 컨테이너 뷰         |
| `SliceMut<T>`       | `SliceMut`을 기대하는 자리의 `xs[1:3]` | 리스트에 쓰는 뷰 |
| `FrozenList<T>`, `FrozenDict<K, V, S>`, `FrozenSet<T, S>` | `xs.freeze()` | 읽기 전용, 스레드 사이에서 공유 |

## 2.9.1. 리터럴 문법: `#[literal(query)]` 바인딩

리터럴 문법은 [refs-lang 6.5 `#[literal]`](../../refs-lang/006-meta-system/605-literal-magic.md)의 **query DSL**로 다음과 같이 정의합니다.

```nabi
#[literal("[" {expr} % "," * "]")]
fn list_literal<T>(items: ...T) -> List<T> { ... }

#[literal("{" {expr ":" expr} % "," * "}")]
fn dict_literal<K: Hash, V>(pairs: ...(K, V)) -> Dict<K, V> { ... }

#[literal("{" {expr} % "," + "}")]
fn set_literal<T: Hash>(items: ...T) -> Set<T> { ... }
```

각 핸들러는 `core::collections`의 패키지 비공개 함수입니다. 튜플 `(1, "a", true)`와 고정 배열은 핸들러 없이 컴파일러가 만듭니다([refs-lang 6.5](../../refs-lang/006-meta-system/605-literal-magic.md)).

리터럴 query 문법의 뜻은 다음과 같습니다. `{...} % sep *`는 sep으로 구분한 0번 이상의 반복입니다. `{...} % sep +`는 1번 이상의 반복입니다. 원소가 하나인 Set은 뒤에 콤마를 붙여 `{1,}`로 씁니다. 그래야 블록 `{1}`과 구분됩니다. 토큰열을 매칭할 때는 렉서가 먼저 토큰을 직접 구분하고, 그다음 파서가 query를 매칭합니다.

### Dict / Set 모호성 해결

```nabi
{"a": 1}      // Dict: `:` 패턴
{1, 2}        // Set: 값만
{1,}          // Set (단일 원소, 후행 콤마)
{1}           // 항상 block. 슬롯이 Set 이어도 마찬가지
{ x = 1 }     // block expression: `{` 다음 토큰이 statement-like
```

파서가 내용만 보고 정합니다. `expr ":"` 형태이면 Dict, 콤마가 있으면 Set, 그 밖에는 블록입니다. 받는 슬롯의 타입은 판정에 쓰지 않습니다.

### `[...]`의 타입

`[1, 2, 3]`은 기대 타입이 없으면 `List<T>`입니다. 고정 크기 배열 `[T; N]`은 기대 타입을 `[T; N]`으로 적었을 때만 만들어집니다. 예를 들면 `let a: [i32; 3] = [1, 2, 3];`입니다. 고정 배열은 컴파일러가 핸들러를 거치지 않고 직접 만듭니다.

기대하는 원소 타입이 없으면 원소들은 분기와 같은 규칙으로 맞춥니다([refs-lang 3](../../refs-lang/003-expressions/README.md)). 모든 원소의 타입이 같거나, 한 원소의 합 타입이 다른 모든 원소의 타입을 담으면 어느 원소가 먼저 오든 그 타입입니다(`x: f64 | none`일 때 `[none, x]`와 `[x, none]`은 `List<f64 | none>`). 타입이 다른 원소들로 합 타입을 새로 만들지는 않습니다. `[0.5, none]`은 오류(E0040)이고, 도움말은 `let xs: List<f64 | none> = [0.5, none];`처럼 리스트의 타입을 적으라고 안내합니다.

반복 리터럴 `[v; N]`은 언제나 고정 배열입니다. `v`를 한 번 평가하고 원소마다 따로 복사본을 줍니다. `Copy`이거나 평범한 값인 원소는 비트 복사, 컬렉션을 가진 원소는 `v.clone()`이므로 `[List::<i32>::new(); 3]`은 서로 다른 리스트 세 개입니다([refs-lang 2.11.6](../../refs-lang/002-types/211-const-generics.md#2116-매개변수-추론)). `[T; N]`의 `Default`도 원소마다 `T::default()`를 한 번씩 호출합니다.

원소가 `Clone` / `Copy` / `Default` / `Debug` / `PartialEq` / `Eq` / `PartialOrd` / `Ord` / `Hash`를 구현하면 고정 배열도 그 트레잇을 구현합니다. 순서 비교는 사전식이므로 비교 연산자(`==`, `<` 등)를 바로 쓸 수 있습니다. 메서드는 다음과 같습니다.

```nabi
impl<T, const N: usize> [T; N] {
    fn len(self) -> usize;                       // N
    fn is_empty(self) -> bool;
    fn iter(self) -> SliceIter<T>;               // 복사본 위에서 (배열은 값)
    fn first(self) -> T | none;
    fn last(self) -> T | none;
    fn map<U>(self, f: fn(T) -> U) -> [U; N];    // 새 배열
    fn fill(ref self, x: T);                     // 제자리
}

impl<T: PartialEq, const N: usize> [T; N] {
    fn contains(self, x: T) -> bool;
}
```

`array::from_fn::<T, N>(f: fn(usize) -> T) -> [T; N]`은 원소 `i`가 `f(i)`인 고정 배열을 만들며, `f`는 순서대로 부릅니다. `N`은 기대 타입이나 타입 인자에서 옵니다. 원소마다 따로 만든 값이므로 `[v; N]`이 복사할 수 없는 원소, 곧 `Clone`이 아닌 타입이나 소유자가 하나인 자원으로도 배열을 만들 수 있습니다.

```nabi
let bags = array::from_fn::<Bag, 3>(|i| Bag { xs: [i] });
let files: [File; 2] = array::from_fn(|i| try! File::open(paths[i]));
```

```nabi
let a: [i32; 3] = [1, 2, 3];
a.len();                 //→ 3
a == [1, 2, 3];          //→ true
a < [1, 2, 4];           //→ true (사전식)
```

### 참조 핸들

`List`, `Dict`, `Set` 값은 GC 힙 객체를 가리키는 참조 핸들입니다. `let ys = xs;`나 인자 전달은 핸들을 복사하므로 같은 컬렉션을 공유합니다. 그래서 컬렉션을 바꾸는 메서드도 `ref self`가 아니라 `self`를 받습니다. 독립된 사본이 필요하면 `clone()`을 부릅니다. struct 원소는 값입니다. `xs[i]`와 `d[k]`는 사본을 돌려주므로 바꾼 값은 `xs[i] = v`로 저장합니다([refs-lang 2.3.8](../../refs-lang/002-types/203-struct.md#238-값-의미)). `iter` / `keys` / `values`는 컬렉션을 복사하지 않는 지연 반복자를 돌려줍니다.

## 2.9.2. `List<T>`: 동적 배열

```nabi
struct List<T> {
    // 내부: ptr + len + capacity
}
```

내부는 연속된 메모리 버퍼와 `len` / `capacity`로 이루어집니다. push와 pop은 분할 상환 O(1), 임의 접근은 O(1)입니다.

### 핵심 메서드

```nabi
impl<T> List<T> {
    fn new() -> List<T>;
    fn with_capacity(cap: usize) -> List<T>;
    fn filled(n: usize, x: T) -> List<T>;        // x 를 n 개

    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn capacity(self) -> usize;

    fn push(self, x: T);
    fn pop(self) -> T | none;
    fn insert(self, idx: usize, x: T);
    fn remove(self, idx: usize) -> T;
    fn clear(self);
    fn truncate(self, len: usize);              // 앞의 len 개만 남김; 더 적으면 그대로
    fn drain(self, r: Range<usize>) -> List<T>; // r 의 원소를 순서대로 빼서 돌려줌; r 이 거꾸로이거나 len() 을 넘어 끝나면 panic

    fn get(self, idx: usize) -> T | none;       // safe: out-of-bounds → none
    fn first(self) -> T | none;
    fn last(self) -> T | none;
    fn swap(self, i: usize, j: usize);          // 둘 중 하나라도 범위 밖이면 panic
    fn fill(self, x: T);                         // 모든 원소가 x 가 됨

    fn extend<I: IntoIterator<Item = T>>(self, iter: I);
    fn retain(self, keep: fn(T) -> bool);        // 제자리, 순서 유지
    fn sort(self) where T: Ord;                  // 오름차순, stable
    fn sort_by(self, cmp: fn(T, T) -> Ordering); // stable
    fn sort_by_key<K: Ord>(self, key: fn(T) -> K); // 키 기준 오름차순, stable
    fn reserve(self, additional: usize);
    fn shrink_to_fit(self);
    unsafe fn set_len(self, n: usize);           // n > capacity() 면 panic
    fn reverse(self);                            // 제자리

    fn sort_unstable_by(self, cmp: fn(T, T) -> Ordering);   // 안정 정렬 아님, 제자리, 추가 메모리 없음, O(n log n)
    fn sort_unstable_by_key<K: Ord>(self, key: fn(T) -> K);
    fn binary_search_by(self, f: fn(T) -> Ordering) -> usize | none;          // f(x)는 x가 목표의 어느 쪽인지 말함
    fn binary_search_by_key<K: Ord>(self, k: K, key: fn(T) -> K) -> usize | none;
    fn rposition(self, pred: fn(T) -> bool) -> usize | none;  // pred가 받아들이는 마지막 인덱스
    fn rotate_left(self, k: usize);              // 앞의 k개가 뒤로 감. k > len 이면 panic
    fn rotate_right(self, k: usize);             // 뒤의 k개가 앞으로 옴
    fn split_off(self, at: usize) -> List<T>;    // [at, len)을 빼서 돌려줌. at > len 이면 panic
    fn dedup_by(self, same: fn(T, T) -> bool);   // same(남긴 것, x)이면 x를 버림
    fn dedup_by_key<K: PartialEq>(self, key: fn(T) -> K);

    fn split_at(self, mid: usize) -> (Slice<T>, Slice<T>);   // [0, mid) 와 [mid, len) 의 view; mid > len 이면 panic
    fn windows(self, size: usize) -> slice::Windows<T>;   // self[:].windows(size)
    fn chunks(self, size: usize) -> slice::Chunks<T>;     // self[:].chunks(size)
    fn split_at_mut(self, mid: usize) -> (SliceMut<T>, SliceMut<T>);   // [0, mid) 와 [mid, len) 의 쓰기 가능한 뷰
    fn chunks_mut(self, size: usize) -> slice::ChunksMut<T>;   // size 개씩의 쓰기 가능한 뷰, 마지막은 더 짧음
    fn partition_point(self, pred: fn(T) -> bool) -> usize;   // self[:].partition_point(pred)

    fn iter(self) -> ListIter<T>;
    fn into_iter(self) -> ListIntoIter<T>;
}

impl<T: PartialEq> List<T> {
    fn contains(self, x: T) -> bool;
    fn dedup(self);                              // 바로 앞 원소와 같은 원소를 버림
}

impl<T: Ord> List<T> {
    fn binary_search(self, x: T) -> usize | none;   // 오름차순 리스트에서 x 의 첫 인덱스, O(log n)
    fn sort_unstable(self);                          // 오름차순, 안정 정렬 아님
}

impl<T> List<List<T>> {
    fn concat(self) -> List<T>;                      // 모든 리스트의 원소를 차례로 담은 새 리스트
}

impl<T: Send + Sync> List<T> {
    fn freeze(self) -> FrozenList<T>;                // 읽기 전용이고 Sync, O(1); 리스트는 넘겨짐(2.9.7b)
}
```

`sort_unstable`은 introsort입니다. 세 값의 중앙값을 피벗으로 하는 퀵 정렬에, 원소 20개 미만은 삽입 정렬, 깊이가 2 log2(n)을 넘으면 힙 정렬을 씁니다. 그래서 어떤 경우에도 O(n log n)이고 메모리를 할당하지 않으며, 같은 원소의 순서는 바뀔 수 있습니다. `sort`(병합 정렬)는 안정 정렬이며 n칸을 더 씁니다. `_by` 검색은 `sort_by`와 같은 `Ordering` 약속을 씁니다. `binary_search_by(|x| x.cmp(target))`는 `f`가 오름차순으로 보는 리스트에서 `f`가 `Equal`이라 하는 첫 원소를 찾습니다.

위치로 읽는 연산에는 `T | none` 형태가 있습니다(`get`, `first`, `last`, `pop`). 그 밖의 위치 연산은 범위를 벗어나면 panic이 일어납니다(`xs[i]`, `xs[i] = v`, `insert`, `remove`, `swap`, `drain`, `split_at`). `set`은 없습니다. 쓰기는 `xs[i] = v`이고, 범위 밖 인덱스를 건너뛰려면 먼저 검사합니다(`if i < xs.len() { xs[i] = v }`).

`set_len(n)`은 원소를 쓰지 않고 길이만 정합니다. 제자리에서 채우는 버퍼(뷰의 `raw_parts`, 런타임 호출)를 위한 것입니다. 옛 길이 너머의 원소는 저장소에 있던 값(쓴 적 없거나 비워진 칸은 0)을 그대로 가지며, 이 값이 올바른 `T`라는 보장은 없으므로 `unsafe`입니다. 호출자는 읽기 전에 그 원소를 써야 합니다. 이 방식으로 줄이면 잘린 원소는 버퍼에 그대로 남습니다. `n`이 용량을 넘으면 패닉하므로 먼저 `reserve`를 부릅니다.

`partition_point(pred)`는 원소가 `pred`를 처음으로 만족하지 않는 인덱스입니다. `pred`는 리스트의 앞부분에서 참이고 나머지에서 거짓이어야 하며, O(log n)입니다. 오름차순 리스트에서 `x`가 들어갈 자리는 `xs.partition_point(|e| e < x)`이고, 키 기준이면 `xs.partition_point(|e| key(e) < k)`입니다. `binary_search(x)`는 `x`와 같은 원소 가운데 첫 인덱스(하한)를 돌려주고, 없으면 `none`입니다.

```nabi
impl List<str> {
    fn join(self, sep: str) -> str;              // 원소를 sep 로 이어 붙임
}

impl List<u8> {
    fn find<B: AsBytes>(self, needle: B) -> usize | none;   // self[:].find(needle)
}
```

### 사용 예

```nabi
let xs: List<i32> = [];
xs.push(1);
xs.push(2);
xs.push(3);

let ys: List<i32> = [10, 20, 30];
ys[0];                    //→ 10  (Index trait)
ys[100];                  //→ panic: index 100 out of bounds for length 3
ys.get(100);              //→ none

for x in ys {
    process(x);               //→ 10, 20, 30
}

let total: i32 = ys.into_iter().sum();   //→ 60

let zs: List<i32> = [3, 1, 1, 2, 2, 4];
zs.dedup();                                  //→ [3, 1, 2, 4]
zs.retain(|x| x != 3);                       //→ [1, 2, 4]
zs.binary_search(4);                         //→ 2
zs.partition_point(|x| x < 3);               //→ 2 (3이 들어갈 자리)
let (a, b) = zs.split_at(1);                 //→ [1] 과 [2, 4] 의 view
```

### 트레잇 구현

| 트레잇                | 구현 조건           | 의미                            |
|-----------------------|---------------------|---------------------------------|
| `IntoIterator`        | 항상                | `for x in xs`                   |
| `FromIterator<T>`     | 항상                | `(0..5).collect::<List<i32>>()` |
| `Index<usize>`        | 항상                | `xs[i]`                         |
| `IndexSet<usize, T>`  | 항상                | `xs[i] = v`                     |
| `AddAssign<T>`        | 항상                | `xs += x`는 `x`를 뒤에 넣음     |
| `AddAssign<List<T>>`  | 항상                | `xs += ys`는 `ys`의 모든 원소를 덧붙임 |
| `Sliceable<Range<usize>>` | 항상            | `xs[1:3]`                       |
| `Display`             | **아님**            | **수동 출력 권장**              |
| `Debug`               | `T: Debug`          | `f"{xs:?}"` → `[1, 2, 3]`       |
| `Default`             | 항상                | `List::new()`                   |
| `Clone`               | `T: Clone`          | 깊은 복제                       |
| `PartialEq`           | `T: PartialEq`      | 원소별                          |
| `PartialOrd`          | `T: PartialOrd`     | 사전식, 접두사가 먼저           |
| `Eq`/`Ord`/`Hash`     | T가 같은 트레잇을 구현할 때 | 사전식                  |

### 비교 / 해시 / Debug: 공통 규칙

- `List<T>`는 `PartialEq` / `Eq` / `Hash`를 순서대로 원소별로 구현합니다.
- `Dict<K, V, S>`와 `Set<T, S>`는 순서와 무관하게 비교하고 해시합니다. 든 원소(키와 값)가 같으면 같습니다. 원소마다 프로세스 키로 따로 해시해 결과를 더하므로([2.5.6](205-cmp.md#256-hash-해시eq-일관성-약속)) 해시도 `S`와 무관합니다.
- 세 컬렉션 모두 원소가 그 트레잇을 구현하면 `Debug`를 구현합니다. 출력은 순회 순서를 따릅니다.
- 따라서 원소가 `Eq + Hash`인 컬렉션은 `Set` 원소나 `Dict` 키가 될 수 있습니다(`Set<List<i32>>`).

### 용량과 제거: 공통 규칙

- 범위 밖 인덱스는 panic을 일으킵니다. 메시지에는 인덱스와 길이가 들어갑니다(`index 100 out of bounds for length 3`). 메서드는 메시지 앞에 자기 이름을 붙이고(`List::remove: index 5 out of bounds for length 3`, `List::drain: range 2..4 out of bounds for length 3`), panic 위치는 호출한 곳을 가리킵니다.
- `reserve`, `with_capacity`, `repeat`, 확장에서 크기가 넘치면 "capacity overflow" panic이 일어납니다. 최적화 수준과 관계없이 같습니다.
- 원소를 빼면(`pop`, `remove`, `clear`, `truncate`, `drain` 등) 그 참조도 놓습니다. 그 뒤로 GC는 그 컬렉션을 거쳐 그 원소에 닿지 않습니다.

## 2.9.3. `Dict<K, V, S>`: 해시 맵

```nabi
struct Dict<K: Hash + Eq, V, S = RandomState> {
    // 내부: 삽입 순서로 놓인 원소 배열과 선형 탐사 슬롯의 인덱스
    // (작은 표는 인덱스 없음)
}
```

조회, 삽입, 제거는 평균 O(1)입니다. 순회는 삽입 순서를 따릅니다. 이미 있는 키를 다시 삽입하면 자리를 지키고, 제거한 뒤 다시 삽입한 키는 맨 뒤로 갑니다.

`S`가 해셔를 만듭니다([2.5.11](205-cmp.md#2511-corehash-해셔)). 기본값 `RandomState`는 프로세스마다 한 번 뽑은 키를 쓰므로 키가 표의 어느 칸에 놓이는지는 실행마다 바뀝니다. 순회 순서는 바뀌지 않습니다. 상수 평가 중에 만든 dict는 런타임에 처음 쓸 때 프로세스 키로 다시 해시합니다. `Dict<K, V>`는 `Dict<K, V, RandomState>`이고, `{k: v}` 리터럴도 모두 그렇습니다.

### 핵심 메서드

```nabi
impl<K: Hash + Eq, V> Dict<K, V> {
    fn new() -> Dict<K, V>;
    fn with_capacity(cap: usize) -> Dict<K, V>;
}

impl<K: Hash + Eq, V, S: BuildHasher> Dict<K, V, S> {
    fn with_hasher(s: S) -> Dict<K, V, S>;
    fn with_capacity_and_hasher(cap: usize, s: S) -> Dict<K, V, S>;
    fn hasher(self) -> S;

    fn len(self) -> usize;
    fn is_empty(self) -> bool;

    fn insert(self, k: K, v: V) -> V | none;       // 기존 V 반환 / 없으면 none
    fn remove(self, k: K) -> V | none;
    fn remove_entry(self, k: K) -> (K, V) | none;  // 저장된 키도 함께
    fn pop_last(self) -> (K, V) | none;            // 마지막에 넣은 항목을 뺌. Python d.popitem()
    fn pop_first(self) -> (K, V) | none;           // 처음에 넣은 항목을 뺌
    fn drain(self) -> List<(K, V)>;                // 모든 항목을 빼서 삽입 순서로 돌려줌
    fn get(self, k: K) -> V | none;
    fn get_or(self, k: K, default: V) -> V;        // 없으면 `default`, 넣지 않음
    fn get_or_insert(self, k: K, default: V) -> V; // 없으면 `default`를 넣음
    fn contains_key(self, k: K) -> bool;
    fn clear(self);
    fn retain(self, keep: fn(K, V) -> bool);      // 제자리
    fn extend<I: IntoIterator<Item = (K, V)>>(self, pairs: I);   // 뒤의 쌍의 값이 이김

    fn iter(self) -> DictIter<K, V, S>;              // (K, V) 쌍
    fn keys(self) -> Keys<K, V, S>;
    fn values(self) -> Values<K, V, S>;

    fn entry(self, k: K) -> Entry<K, V, S>;       // 그 키의 자리 (있든 없든)
}

struct Entry<K, V, S = RandomState> { /* dict + key */ }

impl<K: Hash + Eq, V, S: BuildHasher> Entry<K, V, S> {
    fn key(self) -> K;
    fn or_insert(self, v: V) -> V;                   // 없으면 v 를 넣고, 그 값
    fn or_insert_with(self, f: fn() -> V) -> V;      // 없으면 f() 를 넣고, 그 값
    fn and_modify(self, f: fn(V) -> V) -> Entry<K, V, S>;   // 있으면 f(값) 으로 바꿈
}

impl<K: Hash + Eq, V: Default, S: BuildHasher> Entry<K, V, S> {
    fn or_default(self) -> V;                         // 없으면 V::default() 를 넣고, 그 값
}
```

`d.freeze()`(`K`, `V`, `S`가 `Send + Sync`)는 딕셔너리를 `FrozenDict<K, V, S>`로 넘깁니다. 스레드들이 락 없이 함께 읽습니다([2.9.7b](#297b-frozenlistt--frozendictk-v-s--frozensett-s-읽기-전용-컬렉션)).

`Dict`는 `IntoIterator`, `Index<K>`, `IndexSet<K, V>`, `AddAssign<Dict<K, V, S>>`, `PartialEq` / `Eq` / `Hash`(`V` 기준), `Debug`(`K: Debug, V: Debug`), `Clone`(`K: Clone, V: Clone`, 키와 값을 복제하고 해셔는 같음), `Default`, `FromIterator<(K, V)>`, `From<List<(K, V)>>`를 구현합니다. `Default`, `FromIterator`, `From<List<(K, V)>>`는 `S: BuildHasher + Default`가 필요합니다. 나머지 구현은 모든 `S: BuildHasher`를 받습니다.

```nabi
use core::hash::BuildHasherDefault;

let fixed: Dict<str, i32, BuildHasherDefault<Fnv>> = Dict::with_hasher(BuildHasherDefault::new());
```

```nabi
let counts: Dict<str, i32> = Dict::new();   // `{}`는 빈 Dict가 아니라 빈 블록
for w in words {
    counts.entry(w).and_modify(|n| n + 1).or_insert(1);
}
```

### 사용 예

```nabi
let d: Dict<str, i32> = {"a": 1, "b": 2};
d["a"];                   //→ 1
d["c"];                   //→ panic: key "c" not found; use `d.get(k)` to get `i32 | none`
d.get("c");               //→ none

d.insert("c", 3);
d.contains_key("c");      //→ true

for (k, v) in d.iter() {
    log("{k} = {v}");
}
```

### `Index<K>`

```nabi
impl<K: Hash + Eq, V> Index<K> for Dict<K, V> {
    type Output = V;
    fn index(self, k: K) -> V {
        self.get(k) ?? panic(f"key {k:?} not found; use `d.get(k)` to get `V | none`")
    }
}
```

`d[k]` 표기는 키가 있다고 가정합니다. 안전하게 접근하려면 `.get(k)`를 쓰세요. panic 메시지는 키를 `{k:?}`처럼 보여 줍니다. `K`에 `Debug`가 없으면 타입 이름 `<K>`로 보여 줍니다. `V` 자리에는 값 타입의 이름이 들어갑니다.

`d += e`(`AddAssign<Dict<K, V>>`)는 `e`의 모든 항목을 `d`에 제자리에서 삽입합니다. 키가 겹치면 `e`의 값이 남습니다.

`Dict::from(pairs)`(`From<List<(K, V)>>`)는 쌍의 리스트로 dict를 만듭니다. 키가 겹치면 뒤의 값이 남습니다. `K`와 `V`는 인자에서 정해지므로 `let d = Dict::from([("a", 1)]);`에는 타입 표기가 필요 없습니다.

### lazy view

`iter` / `keys` / `values`는 사본이 아닙니다. 현재 테이블을 그대로 보는 지연 뷰입니다. 순회 도중 dict를 바꾸면 결과는 사양이 정하지 않습니다. 원소를 빠뜨리거나 두 번 볼 수 있습니다. 그래도 메모리 안전성은 지켜집니다.

## 2.9.4. `Set<T, S>`: 해시 집합

```nabi
struct Set<T: Hash + Eq, S = RandomState> {
    // 내부: Dict<T, bool, S>
}
```

`Dict`처럼 삽입 순서로 순회합니다. 연산은 평균 O(1)입니다. `Dict`처럼 `S`가 해셔를 만듭니다. `s.freeze()`(`T`와 `S`가 `Send + Sync`)는 집합을 `FrozenSet<T, S>`로 넘깁니다([2.9.7b](#297b-frozenlistt--frozendictk-v-s--frozensett-s-읽기-전용-컬렉션)).

```nabi
impl<T: Hash + Eq> Set<T> {
    fn new() -> Set<T>;
}

impl<T: Hash + Eq, S: BuildHasher> Set<T, S> {
    fn with_hasher(s: S) -> Set<T, S>;
    fn with_capacity_and_hasher(cap: usize, s: S) -> Set<T, S>;
    fn hasher(self) -> S;
    fn len(self) -> usize;
    fn is_empty(self) -> bool;

    fn insert(self, x: T) -> bool;        // true if 새로 들어감
    fn remove(self, x: T) -> bool;
    fn pop(self) -> T | none;             // 마지막에 넣은 원소를 뺌
    fn drain(self) -> List<T>;            // 모든 원소를 빼서 삽입 순서로 돌려줌
    fn contains(self, x: T) -> bool;
    fn clear(self);
    fn retain(self, keep: fn(T) -> bool);         // 제자리
    fn extend<I: IntoIterator<Item = T>>(self, items: I);

    fn union(self, other: Set<T, S>) -> Set<T, S>;
    fn intersection(self, other: Set<T, S>) -> Set<T, S>;
    fn difference(self, other: Set<T, S>) -> Set<T, S>;
    fn symmetric_difference(self, other: Set<T, S>) -> Set<T, S>;

    fn is_subset(self, other: Set<T, S>) -> bool;
    fn is_superset(self, other: Set<T, S>) -> bool;
    fn is_disjoint(self, other: Set<T, S>) -> bool;

    fn iter(self) -> SetIter<T, S>;
    fn to_list(self) -> List<T>;
}
```

`Set`은 `IntoIterator`, `AddAssign<T>`(삽입), `PartialEq` / `Eq` / `Hash`, `Debug`(`T: Debug`), `Clone`(`T: Clone`, 원소를 복제하고 해셔는 같음), `Default`, `FromIterator<T>`, `From<List<T>>`를 구현합니다. 마지막 세 가지는 `S: BuildHasher + Default`가 필요합니다.

```nabi
let s: Set<i32> = {1, 2, 3};
s.contains(2);            //→ true
s.insert(4);

let a: Set<i32> = {1, 2, 3};
let b: Set<i32> = {2, 3, 4};
let u = a.union(b);       //→ {1, 2, 3, 4}
let i = a.intersection(b);//→ {2, 3}
```

`Set::from(xs)`(`From<List<T>>`)는 리스트로 set을 만들고 중복은 버립니다. `T`는 인자에서 정해집니다(`let s = Set::from([1, 2, 2]);`). `s += x`(`AddAssign<T>`)는 `x`를 삽입합니다.

## 2.9.5. 삽입 순서

`Dict`와 `Set`은 Python 3.7 이상의 `dict`나 Java의 `LinkedHashMap`처럼 삽입 순서로 순회합니다. 그래서 순서를 지키는 타입을 따로 두지 않습니다. `iter`, `keys`, `values`, `to_list`, `Debug`, `for`가 모두 이 순서를 따릅니다.

- 이미 있는 키를 다시 넣으면 자리는 그대로이고 값만 바뀝니다.
- 지웠다가 다시 넣은 키는 맨 뒤로 갑니다.
- `retain`은 남는 항목의 순서를 지키고, `extend`와 `d += e`는 순서대로 삽입합니다(새 키는 맨 뒤로).
- `pop_last` / `Set::pop`은 가장 새 항목을, `pop_first`는 가장 오래된 항목을 빼며, 각각 평균 O(1)입니다(남긴 빈자리는 항목보다 많아지면 한꺼번에 치움).
- 집합 연산의 결과는 왼쪽 피연산자의 순서를 먼저 따릅니다.

같음 비교와 해시는 순서를 보지 않습니다([2.9.2](#292-listt-동적-배열)). 그래서 `{"a": 1, "b": 2} == {"b": 2, "a": 1}`입니다.

## 2.9.5a. `Deque<T>`: 양끝 큐

링 버퍼 위에 만든 양끝 큐입니다.

```nabi
impl<T> Deque<T> {
    fn new() -> Deque<T>;
    fn with_capacity(cap: usize) -> Deque<T>;
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn capacity(self) -> usize;
    fn push_back(self, x: T);
    fn push_front(self, x: T);
    fn pop_front(self) -> T | none;
    fn pop_back(self) -> T | none;
    fn get(self, i: usize) -> T | none;
    fn front(self) -> T | none;
    fn back(self) -> T | none;
    fn clear(self);
    fn iter(self) -> DequeIter<T>;             // 앞에서 뒤로
}
```

`set`은 없습니다. 쓰기는 `d[i] = v`입니다. `capacity`는 링 버퍼가 커지기 전까지 담을 수 있는 원소 수입니다. 이 타입은 `d[i]` / `d[i] = v`를 구현합니다. 범위 밖이면 `List`처럼 panic이 일어납니다(`index 5 out of bounds for length 3`). 그 밖에 `IntoIterator`, `FromIterator`(앞에서 뒤로), `Default`, `Clone`(`T: Clone`), `PartialEq` / `Eq`(앞에서 뒤로 원소별), `Debug`를 구현합니다. `d += x`(`AddAssign<T>`)는 `x`를 뒤에 넣습니다.

## 2.9.5b. `BinaryHeap<T>`: 우선순위 큐

```nabi
impl<T: Ord> BinaryHeap<T> {
    fn new() -> BinaryHeap<T>;                 // max-heap: 가장 큰 것이 먼저
}
impl<T> BinaryHeap<T> {
    fn by_ordering(cmp: fn(T, T) -> Ordering + Send) -> BinaryHeap<T>; // cmp 기준 가장 큰 것이 먼저, new()가 Ord::cmp 기준인 것처럼
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn peek(self) -> T | none;
    fn push(self, x: T);
    fn pop(self) -> T | none;
    fn clear(self);
    fn retain(self, keep: fn(T) -> bool);
    fn iter(self) -> HeapIter<T>;              // 순서 없음
}
```

`retain`은 제자리에서 걸러 낸 뒤 힙 순서를 되살리며 O(n)입니다. `HeapIter<T>`는 `ExactSizeIterator`입니다. 힙은 `IntoIterator`(`iter`처럼 순서 없음), `Debug`(저장 순서, 꼭대기가 먼저), `Clone`(`T: Clone`, 사본도 힙의 순서를 유지)을 구현합니다. `T: Ord`이면 `Default`와 `FromIterator`가 `new`처럼 최대 힙을 만듭니다.

비교 함수는 `List::sort_by`의 것처럼 `Ordering`을 반환합니다. `by_ordering(|a, b| b.cmp(a))`는 가장 작은 것을 먼저 꺼냅니다.

비교 함수는 `fn .. + Send`이므로, `T`가 `Send`이면 힙도 `Send`입니다. 채널로 보내거나 `Mutex`에 넣을 수 있습니다. 비교 함수가 `Send + Sync`가 아닌 값(컬렉션 등)을 캡처하면 컴파일 에러이고, help는 캡처를 없애라고 안내합니다. 값을 클로저 안에서 계산하거나 배열을 캡처합니다.

## 2.9.5c. `SortedDict<K, V>` / `SortedSet<T>`: 정렬 맵과 집합

`core::collections::{SortedDict, SortedSet}`은 항목을 키 순서(`K: Ord`)로 B-트리에 둡니다. 조회, 삽입, 제거는 O(log n)이고, 순회, `range`, `first` / `last`, `floor` / `ceil`은 이 순서를 따릅니다. prelude에는 없습니다: `use core::collections::SortedDict;`.

```nabi
impl<K: Ord, V> SortedDict<K, V> {
    fn new() -> SortedDict<K, V>;
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn insert(self, k: K, v: V) -> V | none;       // 이전 값 / 새 키면 none
    fn remove(self, k: K) -> V | none;
    fn get(self, k: K) -> V | none;
    fn get_or(self, k: K, default: V) -> V;
    fn contains_key(self, k: K) -> bool;
    fn clear(self);

    fn first(self) -> (K, V) | none;               // 가장 작은 키의 항목
    fn last(self) -> (K, V) | none;                // 가장 큰 키의 항목
    fn pop_first(self) -> (K, V) | none;
    fn pop_last(self) -> (K, V) | none;
    fn floor(self, k: K) -> (K, V) | none;         // k 이하인 가장 큰 키
    fn ceil(self, k: K) -> (K, V) | none;          // k 이상인 가장 작은 키
    fn lower(self, k: K) -> (K, V) | none;         // k 미만인 가장 큰 키
    fn higher(self, k: K) -> (K, V) | none;        // k 초과인 가장 작은 키

    fn iter(self) -> SortedIter<K, V>;             // 키 순서의 (K, V)
    fn iter_rev(self) -> SortedIter<K, V>;         // 내림차순
    fn keys(self) -> SortedKeys<K, V>;
    fn values(self) -> SortedValues<K, V>;
    fn range<R: KeyRange<K>>(self, r: R) -> SortedIter<K, V>;      // r 안의 키, 오름차순
    fn range_rev<R: KeyRange<K>>(self, r: R) -> SortedIter<K, V>;  // r 안의 키, 내림차순
}

pub trait KeyRange<K> {                            // a..b, a..=b, a.., ..b, ..=b, ..
    fn lower(self) -> (K, bool) | none;            // 경계와 포함 여부. 열려 있으면 none
    fn upper(self) -> (K, bool) | none;
}
```

`SortedSet<T>`(안은 `SortedDict<T, bool>`)에는 `new`, `len`, `is_empty`, `insert(x) -> bool`(새로 들어가면 true), `remove(x) -> bool`, `contains`, `clear`, `first`, `last`, `pop_first`, `pop_last`, `floor`, `ceil`, `lower`, `higher`(각각 `T | none`), `iter`, `iter_rev`, `range`, `range_rev`, `to_list`와 `Set`의 집합 연산(`union`, `intersection`, `difference`, `symmetric_difference`, `is_subset`, `is_superset`, `is_disjoint`)이 있으며, 결과는 정렬 집합입니다.

`range`는 모든 범위 형태를 받습니다([refs-lang 3.7](../../refs-lang/003-expressions/307-range-expressions.md)). `..`로 시작하는 형태는 인수로 쓸 때 괄호로 감쌉니다(`d.range((..10))`). 첫 항목을 찾는 데 O(log n), 그다음 항목마다 평균 O(1)입니다. 비었거나 거꾸로 된 범위는 아무것도 주지 않습니다.

둘 다 `IntoIterator`(키 순서), `FromIterator`, `From<List<..>>`(나중 쌍의 값이 이김), `Default`, `Clone`(`K: Clone, V: Clone`, 항목을 복제), `PartialEq` / `Eq`(같은 항목. 순서는 키 순서), `Hash`(길이, 이어서 차례대로 각 항목), `Debug`(차례대로 `{k: v, ..}` / `{a, b, ..}`)를 구현합니다. `SortedDict`는 `d[k]`(없으면 패닉: `key 7 not found; use `d.get(k)` to get `V | none``)와 `d[k] = v`를, `SortedSet`은 `s += x`를 구현합니다. 복사본은 컬렉션을 공유하고, 둘 다 `Dict`처럼 `!Sync`입니다.

반복자가 도는 동안 정렬 사전이나 집합을 바꾸면 반복자의 다음 걸음에서 키를 먼저 모으라고 알리며 패닉합니다(`SortedDict changed while it was walked; ..`). 이미 있는 키의 값을 바꾸는 것은 변경이 아닙니다(`for (k, v) in d { d[k] = v + 1; }`는 괜찮음).

```nabi
use core::collections::SortedDict;

let events: SortedDict<i64, str> = SortedDict::new();
events[1700] = "boot";
events[1650] = "power";
events[1720] = "login";
for (t, what) in events.range(1600..1710) {   // 시간 순서로 1650, 1700
    println(f"{t} {what}");
}
let before = events.floor(1710);              //→ (1700, "boot")
let latest = events.last();                   //→ (1720, "login")
```

**구현.** 최소 차수는 16입니다. 노드 하나는 키 15개에서 31개를 지니고(루트는 더 적을 수 있음), 노드는 번호로 찾는 평평한 리스트에 삽니다(키, 값, 자식 번호가 각각 리스트 하나). 그래서 조회는 단계마다 리스트 칸 하나를 봅니다. 삽입은 내려가는 길에 가득 찬 노드를 나누고, 제거(CLRS)는 내려가기 전에 키가 16개 미만인 자식을 형제에게서 채우거나 둘을 합칩니다. 빈 칸은 0으로 채우므로 제거한 항목은 아무것도 붙들지 않습니다.

## 2.9.6. `Tuple<...>`: 고정 길이 이종 묶음

```nabi
struct Tuple { /* heterogeneous, 컴파일 타임 결정 */ }
```

튜플은 언어에 내장되어 있어 리터럴 핸들러로 등록할 수 없습니다([refs-lang 6.5.7](../../refs-lang/006-meta-system/605-literal-magic.md#657-사용자-정의-리터럴)). 튜플 식에는 콤마가 하나 이상 있어야 합니다. `(e)`는 그룹 식이고, 항이 하나인 튜플은 `(e,)`입니다.

위치마다 타입이 다를 수 있습니다. 길이는 컴파일 타임에 고정됩니다.

```nabi
let t: Tuple<i32, str, bool> = (42, "hello", true);
let n = t.0;              //→ 42
let s = t.1;              //→ "hello"
let b = t.2;              //→ true

// 패턴 분해
let (n, s, b) = t;
```

### 트레잇 구현

| 트레잇       | 조건                       |
|--------------|----------------------------|
| `Clone`      | 모든 컴포넌트 `Clone`      |
| `Copy`       | 모든 컴포넌트 `Copy`       |
| `Debug`      | 모든 컴포넌트 `Debug`      |
| `PartialEq`  | 모든 컴포넌트 `PartialEq`  |
| `Eq`         | 모든 컴포넌트 `Eq`         |
| `PartialOrd` | 모든 컴포넌트 `PartialOrd` |
| `Ord`        | 모든 컴포넌트 `Ord`        |
| `Hash`       | 모든 컴포넌트 `Hash`       |

항이 1~12개인 튜플에는 `core::tuple`이 표의 트레잇을 구현합니다. `PartialOrd` / `Ord`는 사전식입니다. 첫 항부터 비교하고 처음으로 다른 항이 결과를 정합니다. 그래서 `<` / `<=` / `>` / `>=`를 튜플에 바로 쓸 수 있습니다(`(1, "b") < (1, "c")`).

`Display`는 자동으로 파생되지 않습니다. 필요한 곳에서 직접 구현하기를 권장합니다.

### 함수 다중 반환

[refs-lang 4.3.5 다중 반환](../../refs-lang/004-functions/403-return.md#435-다중-반환-튜플로-묶기)의 다중 반환은 Tuple을 반환하는 것으로 처리합니다.

```nabi
fn divmod(a: i32, b: i32) -> (i32, i32) {
    (a / b, a % b)
}

let (q, r) = divmod(17, 5);     //→ q = 3, r = 2
```

## 2.9.7. `Slice<T>`: 뷰

```nabi
struct Slice<T> {
    // GC 버퍼 + 시작 위치 + 길이 + stride (1 = 연속, k = step, 음수 = 역순)
}
```

`xs[1:3]` / `xs[::2]` / `xs[1:]` 같은 식의 결과입니다([2.3.4 `Sliceable`](203-ops.md#234-인덱싱-트레잇)). 뷰는 원소를 복사하지 않고 원본의 GC 버퍼를 가리킵니다. 뷰가 살아 있는 동안 그 버퍼도 살아 있습니다.

| 원본       | 뷰가 보는 것                                                                    |
|------------|---------------------------------------------------------------------------------|
| `List<T>`  | 리스트의 현재 버퍼. 원소 쓰기는 보이고, 리스트가 자라 버퍼를 옮긴 뒤의 변경은 보이지 않음 |
| `Slice<T>` | 같은 버퍼(뷰의 뷰)                                                              |
| `[T; N]`   | 배열의 사본(배열은 값)                                                          |

### 핵심 메서드

```nabi
impl<T> Slice<T> {
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn get(self, i: usize) -> T | none;
    fn first(self) -> T | none;
    fn last(self) -> T | none;
    fn step(self, k: isize) -> Slice<T>;     // k 번째마다, k < 0 이면 끝에서부터. k == 0 은 panic
    fn split_at(self, mid: usize) -> (Slice<T>, Slice<T>);   // [0, mid) 와 [mid, len); mid > len 이면 panic
    fn windows(self, size: usize) -> slice::Windows<T>;   // size 개씩 겹치는 view; size == 0 은 panic
    fn chunks(self, size: usize) -> slice::Chunks<T>;     // size 개씩의 view, 마지막은 더 짧음; size == 0 은 panic
    fn iter(self) -> SliceIter<T>;
    fn to_list(self) -> List<T>;             // 새 List 할당
    unsafe fn raw_parts(self) -> (GcBuf<T>, usize, usize);   // (버퍼, 시작, 길이); 보폭 있는 뷰는 먼저 복사
    unsafe fn from_raw_parts(buf: GcBuf<T>, start: usize, len: usize) -> Slice<T>;   // buf 의 start 부터 len 개
}

impl<T> Slice<T> {
    fn partition_point(self, pred: fn(T) -> bool) -> usize;   // 원소가 pred 를 처음 만족하지 않는 인덱스, O(log n)
}

impl<T: PartialEq> Slice<T> {
    fn contains(self, x: T) -> bool;
}

impl<T: Ord> Slice<T> {
    fn binary_search(self, x: T) -> usize | none;   // 오름차순 뷰에서 x 의 첫 인덱스, O(log n)
}

impl Slice<u8> {
    fn find<B: AsBytes>(self, needle: B) -> usize | none;   // 처음 나타나는 offset; 빈 needle 은 0
}

impl<T> Index<usize> for Slice<T> { ... }               // s[i]: 범위 밖이면 panic (`slice index 5 out of bounds for length 3`)
impl<T> Sliceable<Range<usize>> for Slice<T> { ... }    // s[a:b]: view 의 view, 범위는 길이에 맞춰 잘라 맞춤
impl<T> IntoIterator for Slice<T> { ... }
impl<T: PartialEq> PartialEq for Slice<T> { ... }
impl<T: Debug> Debug for Slice<T> { ... }
```

`raw_parts`는 뷰의 원소를 버퍼의 연속된 한 구간으로 돌려줍니다. 버퍼 영역을 다루는 런타임 호출용입니다. 보폭 있는 뷰(`xs[::2]`)는 먼저 새 버퍼로 복사하므로, 그 구간에 쓴 값은 뷰나 그 리스트에 닿지 않습니다. `from_raw_parts(buf, start, len)`은 그 반대입니다. 둘 다 `unsafe`입니다. `start + len`이 `buf` 안에 있는지, 원소가 초기화되었는지 아무것도 검사하지 않습니다.

`slice::Windows<T>`와 `slice::Chunks<T>`는 `Item = Slice<T>`인 `ExactSizeIterator`입니다. 각 창(window)과 덩어리(chunk)는 복사본이 아니라 같은 버퍼의 뷰입니다. `Iterator::windows` / `chunks`([2.7.1](207-iter.md#271-iterator-단계-진행))는 버퍼가 없는 반복자에서 동작하므로 대신 리스트로 모읍니다.

`find`의 needle은 `AsBytes`([2.11.13](211-conversion.md#21113-asbytes-트레잇-바이트-뷰)) 타입이면 무엇이든 됩니다.

```nabi
trait AsBytes {
    fn byte_slice(self) -> Slice<u8>;      // 바이트를 복사 없이 view 로
}
```

`str`(UTF-8 바이트), `List<u8>`, `Slice<u8>`, `[u8; N]`이 이 트레잇을 구현합니다. 그래서 바이트 문자열도 needle로 넘길 수 있습니다(바이트 문자열의 `Slice<u8>` 규칙은 [refs-lang 1.7.5](../../refs-lang/001-lexical/107-string-literals.md#175-바이트-문자열)).

`Slice`는 읽기 전용입니다(`IndexSet` 없음). 리스트에 쓰는 뷰는 `SliceMut<T>`입니다([2.9.7a](#297a-slicemutt-쓰기-가능한-뷰)). 음수 인덱스는 끝에서부터 셉니다([refs-lang 3.6.1](../../refs-lang/003-expressions/306-indexing-and-slicing.md#361-인덱싱-arri)).

### 사용 예

```nabi
let xs: List<i32> = [1, 2, 3, 4, 5];
let s1: Slice<i32> = xs[1:4];           //→ view of [2, 3, 4]
let s2: Slice<i32> = xs[::2];           //→ view of [1, 3, 5]
let s3: Slice<i32> = xs[..];            //→ view of all

s1.len();                               //→ 3
let t: i32 = s1.iter().sum();           //→ 9
xs[2] = 30;
s1[1];                                  //→ 30 (리스트의 원소 쓰기가 보임)

let owned: List<i32> = s1.to_list();    //→ [2, 30, 4] (새 List)
```

### 가변 인자와 통합

`fn f(xs: ...T)`의 `xs`는 `Slice<T>` 뷰입니다([refs-lang 4.2.5](../../refs-lang/004-functions/402-parameters.md#425-variadic-t)). 인자를 하나씩 넘겨도, 리스트를 펼쳐(`f(..ys)`) 넘겨도 같은 모양으로 받습니다.

```nabi
fn sum(xs: ...i32) -> i32 {
    let total = 0;
    for x in xs {            // xs: Slice<i32>
        total = total + x;   // 재할당은 `let` 없이
    }
    total
}

sum(1, 2, 3);                //→ 6
sum(..[4, 5]);               //→ 9 (리스트 spread)
```

## 2.9.7a. `SliceMut<T>`: 쓰기 가능한 뷰

```nabi
struct SliceMut<T> {
    // 리스트의 버퍼 + 시작 위치 + 길이, 그리고 뷰를 만들 때의 리스트와 그 버전
}
```

`SliceMut<T>`는 보고 있는 리스트에 쓰는 연속 뷰입니다. Rust의 `&mut [T]`에 해당합니다. 리스트의 일부를 채우거나 순서를 바꾸는 함수가 이 타입을 받습니다(`fn zero(buf: SliceMut<u8>)`). `SliceMut<T>`를 기대하는 자리(인자, 타입을 적은 `let`, 반환값)에서 만들어지고, `split_at_mut`과 `chunks_mut`도 만듭니다. prelude에 있습니다.

| 그 자리에 적은 것               | 뷰                                                                       |
|---------------------------------|--------------------------------------------------------------------------|
| `xs`(`List<T>`)                 | 리스트 전체([refs-lang 2.14.6](../../refs-lang/002-types/214-cast-and-conversion.md#2146-문맥-기반-자동-변환)) |
| `xs[a:b]`, `xs[a..b]`, `xs[..]` | 리스트의 그 범위. 경계는 `Slice`처럼 잘라 맞춤([refs-lang 3.6.2](../../refs-lang/003-expressions/306-indexing-and-slicing.md#362-슬라이싱-arrstartend)) |
| `s[a:b]`(`SliceMut<T>`)         | 뷰의 쓰기 가능한 뷰                                                      |

다른 자리에서 `xs[a:b]`는 그대로 읽기 전용 `Slice<T>`입니다. 간격 있는 범위(`xs[a:b:k]`)는 연속이 아니므로 `SliceMut`이 되지 않습니다.

### 핵심 메서드

```nabi
impl<T> SliceMut<T> {
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn get(self, i: usize) -> T | none;
    fn first(self) -> T | none;
    fn last(self) -> T | none;
    fn fill(self, x: T);
    fn swap(self, i: usize, j: usize);                 // 범위 밖이면 panic
    fn reverse(self);
    fn copy_from_slice(self, src: Slice<T>);           // src.len() != len 이면 panic; src 가 뷰와 겹쳐도 됨
    fn split_at(self, mid: usize) -> (SliceMut<T>, SliceMut<T>);   // [0, mid) 와 [mid, len), 둘 다 쓰기 가능; mid > len 이면 panic
    fn chunks_mut(self, size: usize) -> ChunksMut<T>;  // size 개씩의 쓰기 가능한 뷰, 마지막은 더 짧음; size 가 0 이면 panic
    fn sort_by(self, cmp: fn(T, T) -> Ordering);       // 안정 정렬
    fn sort_by_key<K: Ord>(self, key: fn(T) -> K);     // 안정 정렬
    fn as_slice(self) -> Slice<T>;                     // 같은 원소의 읽기 전용 뷰
    fn iter(self) -> SliceMutIter<T>;
    fn to_list(self) -> List<T>;                       // 새 List 할당
    unsafe fn raw_parts(self) -> (GcBuf<T>, usize, usize);   // (버퍼, 시작, 길이), 영역을 채우는 런타임 호출용
}

impl<T: Ord> SliceMut<T> {
    fn sort(self);                                     // 오름차순, 안정 정렬
}

impl<T: PartialEq> SliceMut<T> {
    fn contains(self, x: T) -> bool;
}

impl<T> Index<usize> for SliceMut<T> { ... }              // s[i]: 범위 밖이면 panic
impl<T> IndexSet<usize, T> for SliceMut<T> { ... }           // s[i] = x 는 리스트의 i 번째 원소에 씀
impl<T> Sliceable<Range<usize>> for SliceMut<T> { ... }   // s[a:b]: 뷰의 쓰기 가능한 뷰, 범위는 잘라 맞춤
impl<T> IntoIterator for SliceMut<T> { ... }
impl<T: PartialEq> PartialEq for SliceMut<T> { ... }
impl<T: Debug> Debug for SliceMut<T> { ... }
```

`SliceMutIter<T>`는 `Item = T`인 `DoubleEndedIterator`이자 `ExactSizeIterator`이고, `ChunksMut<T>`는 `Item = SliceMut<T>`인 `ExactSizeIterator`입니다. `SliceMut`은 `List`처럼 `Sync`가 아닙니다([2.13.9](213-marker.md#2139-send--sync-스레드-경계-마커)). 넘기기(hand-off)는 리스트를 복사하고, 뷰는 그 사본을 봅니다. `chunks_mut`과 `split_at_mut`의 뷰는 서로 겹치지 않으므로, `par::for_each`가 각 뷰를 서로 다른 워커에게 주고 워커는 리스트의 자기 부분에 그 자리에서 씁니다([2.19.1](219-par.md#2191-핵심-api)). 음수 인덱스는 리스트와 같이 끝에서부터 셉니다.

### 뷰가 살아 있는 동안 리스트 크기 바꾸기

크기를 바꾸면 리스트의 원소가 새 버퍼로 옮겨질 수 있습니다. 그래서 리스트의 크기를 바꾼 뒤에는 그 전에 만든 뷰를 쓰지 않습니다. 길이나 버퍼를 바꾸는 `List` 메서드는 모두 크기 바꾸기입니다. `push`, `pop`, `insert`, `remove`, `clear`, `truncate`, `drain`, `set_len`, `reserve`, `shrink_to_fit`, `extend`, `retain`, `dedup`, `freeze`, `+=`가 그렇습니다. 원소 쓰기(`xs[i] = x`, `fill`, `swap`, `sort`, `reverse`)는 크기 바꾸기가 아닙니다.

- 한 함수 안에서 둘 다 보이면, 리스트의 크기를 바꾼 뒤 뷰를 쓰는 것은 컴파일 에러 `E0137`입니다([refs-lang 9.9.4](../../refs-lang/009-tools/909-debug-tools.md#994-진단--에러-메시지)). 컴파일러는 리스트의 복사(`let ys = xs`), 뷰의 뷰(`s[1:]`, `split_at`), 뷰를 도는 `for` 루프, 분기, 루프의 다음 회차까지 뷰를 따라갑니다. 뷰를 다시 대입하면(`s = xs[..]`) 다시 쓸 수 있고, 빠져나가는 경로(`return`, `break`)의 크기 바꾸기는 그 뒤의 코드에 닿지 않습니다.
- 나머지는 실행 시간에 검사합니다. 호출한 함수 안의 크기 바꾸기, 구조체에 넣어 둔 뷰, 호출자가 크기를 바꾸는 동안 기다리는 `async fn`이 그런 경우입니다. 리스트는 크기를 바꿀 때마다 올라가는 버전을 들고 있고, 그 전에 만든 뷰의 메서드는 panic합니다(`SliceMut used after its list was resized (push, insert, clear, ...): make the view again after the resize`). 그 뷰를 도는 `for` / `for ref` 루프의 각 단계, 뷰의 넘기기, 그 뷰를 항목에 넣은 `par` 호출도 마찬가지입니다. 원소 읽기와 쓰기(`s[i]`, `s[i] = x`, `get`, `first`, `last`)는 아무것도 검사하지 않으므로, 뷰를 도는 루프는 리스트를 도는 루프만큼 빠릅니다. 낡은 뷰에서는 이 접근이 뷰를 만들 때의 저장 공간에 닿습니다. 저장 공간을 그대로 둔 크기 바꾸기 뒤라면 그 공간은 여전히 리스트의 것이고, 아니라면 더는 리스트의 것이 아닙니다. 어느 경우든 다른 스레드는 그 공간에 닿지 않습니다(`freeze`는 뷰가 있었던 리스트를 복사합니다).

```nabi
let xs: List<i32> = [1, 2, 3];
let s: SliceMut<i32> = xs[1:];
xs.push(4);
s[0] = 9;          // error[E0137]: `s` views `xs`, which `xs.push(..)` resized at line 3
```

실행 시간 검사가 있으므로 `SliceMut`은 `async fn`의 인자가 될 수 있고 `await`을 넘어 살아 있을 수 있습니다.

### 사용 예

```nabi
fn zero(buf: SliceMut<u8>) {
    buf.fill(0);
}

let xs: List<i32> = [5, 4, 3, 2, 1];
let mid: SliceMut<i32> = xs[1:4];
mid.sort();                             // xs 는 [5, 2, 3, 4, 1]
mid[0] = 20;                            // xs 는 [5, 20, 3, 4, 1]
let (a, b) = mid.split_at(1);
b.fill(0);                              // xs 는 [5, 20, 0, 0, 1]

let bytes: List<u8> = [1, 2, 3];
zero(bytes[1:]);                        // [1, 0, 0]
zero(bytes);                            // 리스트 전체: [0, 0, 0]
```

## 2.9.7b. `FrozenList<T>` / `FrozenDict<K, V, S>` / `FrozenSet<T, S>`: 읽기 전용 컬렉션

한 번 만든 뒤 읽기만 하는 컬렉션(조회 테이블, 시스템이 읽는 컴포넌트 배열, 서버가 응답에 쓰는 맵)은 얼립니다. `xs.freeze()`는 `List`, `Dict`, `Set`을 아무도 바꿀 수 없는 컬렉션으로 만들고, 모든 스레드가 락도 복사도 없이 이를 읽습니다.

```nabi
struct FrozenList<T> { /* 리스트의 저장 공간, 바뀌지 않음 */ }
struct FrozenDict<K, V, S = RandomState> { /* ... */ }
struct FrozenSet<T, S = RandomState> { /* ... */ }

impl<T> FrozenList<T> {
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn get(self, i: usize) -> T | none;
    fn first(self) -> T | none;
    fn last(self) -> T | none;
    fn iter(self) -> ListIter<T>;
    fn split_at(self, mid: usize) -> (Slice<T>, Slice<T>);
    fn windows(self, size: usize) -> slice::Windows<T>;
    fn chunks(self, size: usize) -> slice::Chunks<T>;
    fn to_list(self) -> List<T>;                       // 원소를 담은 새 리스트, 바꿀 수 있음
}
impl<T: PartialEq> FrozenList<T> { fn contains(self, x: T) -> bool; }
impl<T: Ord> FrozenList<T> { fn binary_search(self, x: T) -> usize | none; }
impl<T: Send + Sync> FrozenList<T> { fn new() -> FrozenList<T>; }   // 비어 있음

impl<K: Hash + Eq, V, S: BuildHasher> FrozenDict<K, V, S> {
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn get(self, k: K) -> V | none;
    fn get_or(self, k: K, default: V) -> V;
    fn contains_key(self, k: K) -> bool;
    fn keys(self) -> Keys<K, V, S>;
    fn values(self) -> Values<K, V, S>;
    fn iter(self) -> DictIter<K, V, S>;
    fn hasher(self) -> S;
    fn to_dict(self) -> Dict<K, V, S>;                 // 새 딕셔너리, 바꿀 수 있음
}

impl<T: Hash + Eq, S: BuildHasher> FrozenSet<T, S> {
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn contains(self, x: T) -> bool;
    fn iter(self) -> Keys<T, bool, S>;
    fn hasher(self) -> S;
    fn to_set(self) -> Set<T, S>;                      // 새 집합, 바꿀 수 있음
}
```

`fr[i]`(범위 밖이면 panic)와 `fr[a:b]`(복사 없는 읽기 전용 `Slice<T>`)는 리스트에서와 같고, `fd[k]`는 딕셔너리에서와 같습니다. `IndexSet`도, 무언가를 바꾸는 메서드도 없습니다. 세 타입 모두 `IntoIterator`(얼리기 전 컬렉션의 순서), `Clone`(같은 컬렉션: 바뀌지 않으므로 공유가 복사와 다를 바 없음), `PartialEq`, `Debug`(얼리기 전 컬렉션과 같은 모양), `Default`(빈 컬렉션)를 구현하고, `FrozenList`는 `Eq`와 `Hash`도 구현합니다.

**얼리기.** `freeze`는 원소(딕셔너리는 키, 값, 해셔)가 `Send + Sync`일 때만 있습니다. 스레드들이 읽는 것은 그 안쪽도 바뀌면 안 되기 때문입니다. 리스트의 리스트를 얼리면 모든 스레드가 바뀔 수 있는 같은 안쪽 리스트를 받게 됩니다(`List<List<i32>>::freeze`는 에러 `E0069`이고, 도움말은 원소를 먼저 얼리거나 컬렉션을 `Mutex`로 공유하라고 알려 줍니다). `freeze`는 컬렉션을 넘깁니다([2.13.9](213-marker.md#2139-send--sync-스레드-경계-마커)). 그 뒤 `xs`를 쓰면 컴파일 에러 `E0105`이고, 다른 넘기기처럼 복사, 필드, 호출을 따라갑니다. 얼린 값을 읽거나, `xs`를 그대로 두려면 `clone()`을 얼립니다. 실행 시간에는 얼린 컬렉션이 저장 공간을 O(1)로 넘겨받고 `xs`는 비워 둡니다. 그래서 검사기가 보지 못하는 핸들(`unsafe` 안의 코드)이 바꾸는 것은 새 빈 컬렉션이지, 읽는 쪽이 읽는 것이 아닙니다. `SliceMut`이 있었던 리스트는 대신 원소의 사본을 줍니다(O(n)). 낡은 뷰가 아직 옛 저장 공간에 쓸 수 있기 때문입니다. 상수 평가 중에 만든 딕셔너리는 얼릴 때 다시 해시하므로 조회가 아무것도 쓰지 않습니다. 얼린 컬렉션을 바꾸려면 바꿀 수 있는 사본(`to_list()`, `to_dict()`, `to_set()`)을 만들어 바꾼 뒤 다시 얼립니다.

**공유.** 얼린 타입은 `Sync`이므로 컬렉션이 갈 수 없는 곳에 갑니다. 여러 워커가 실행하는 클로저(`par::map(0..n, |i| table[i])`), 스레드의 클로저, `static`(`static PRIMES: FrozenList<i64> = [2, 3, 5, 7].freeze();`), 여럿이 읽는 `RwLock`, 그리하여 `Sync`가 되는 구조체의 필드가 그렇습니다. 넘기기(`send`, `Mutex::new`, `[move x]`, 태스크의 결과)는 얼린 컬렉션을 복사하지 않습니다. 원소도 `Sync`이므로 `FrozenList<Atomic<i64>>`는 여러 스레드가 더하는 카운터 배열입니다.

```nabi
let table = load_table().freeze();                  // FrozenList<Entry>
let hits = par::map(queries, |q| table[q.slot]);    // 모든 워커가 한 테이블을 읽음

static NAMES: FrozenDict<str, i32> = {"two": 2, "seven": 7}.freeze();

let hist = List::from_fn(16, |_| Atomic::new(0i64)).freeze();
par::for_each(samples, |x| { hist[bucket(x)].fetch_add(1, Relaxed); });
```

## 2.9.8. `Index` / `IndexSet` / `Sliceable` 트레잇 매핑

[2.3 ops](203-ops.md)의 트레잇이 컨테이너의 `[]`, `[i] = v`, `[a:b]`의 의미를 정의합니다.

| 컨테이너     | `Index<I>`                     | `IndexSet<I, V>`          | `Sliceable<R>`                          |
|--------------|--------------------------------|---------------------------|-----------------------------------------|
| `List<T>`    | `Index<usize>` → T             | `IndexSet<usize, T>` | `Sliceable<Range<usize>>` → `Slice<T>`  |
| `Dict<K, V>` | `Index<K>` → V                 | `IndexSet<K, V>` | **없음**                                |
| `Set<T>`     | **없음**                       | **없음**                  | **없음**                                |
| `Tuple<...>` | **컴파일러 직접** (`.0`/.1)    | **컴파일러 직접**         | **없음**                                |
| `[T; N]`     | `Index<usize>` → T             | `IndexSet<usize, T>` | `Sliceable<Range<usize>>` → `Slice<T>` (사본) |
| `Slice<T>`   | `Index<usize>` → T             | **없음**(읽기 전용 뷰)    | `Sliceable<Range<usize>>` → `Slice<T>`  |
| `SliceMut<T>` | `Index<usize>` → T            | `IndexSet<usize, T>` | `Sliceable<Range<usize>>` → `SliceMut<T>` |
| `FrozenList<T>` | `Index<usize>` → T          | **없음**(읽기 전용)       | `Sliceable<Range<usize>>` → `Slice<T>`  |
| `FrozenDict<K, V>` | `Index<K>` → V           | **없음**(읽기 전용)       | **없음**                                |
| `str`        | **없음**(바이트 직접 접근 거부) | **없음**                 | `Sliceable<Range<usize>>` → `str` (사본, UTF-8 경계 검사) |
| `Array<T, N>` | 축 튜플(2.9.14) → `T` 또는 뷰 | 축 튜플, `V` = `T` 또는 뷰 모양의 `Array` | `Index<SliceArg>`로 (refs-lang 3.6.3) |

`str`의 `s[10]`은 UTF-8 멀티바이트 문자를 지키려고 일부러 두지 않았습니다. 문자 단위로 접근하려면 `s.chars()`를 쓰세요. 일부를 잘라 내려면 `s[10..15]`를 쓰세요. 슬라이스이므로 멀티바이트 경계를 검사합니다.

## 2.9.9. `IntoIterator` / `FromIterator`

모든 컨테이너는 `IntoIterator`를 구현합니다.

| 컨테이너            | `Item`            | 순서                              |
|---------------------|-------------------|-----------------------------------|
| `List<T>`           | `T`               | 첫 → 마지막                       |
| `Dict<K, V>`        | `(K, V)`          | 삽입 순서                         |
| `Set<T>`            | `T`               | 삽입 순서                         |
| `Deque<T>`          | `T`               | 앞 → 뒤                           |
| `BinaryHeap<T>`     | `T`               | **순서 없음**(pop 순서 아님)      |
| `Tuple<T...>`       | **구현하지 않음** | (원소 타입이 제각각이라 반복자로 볼 수 없음) |
| `Slice<T>`          | `T`               | 뷰 순서(stride 적용)              |
| `SliceMut<T>`       | `T`               | 뷰 순서                           |
| `FrozenList<T>`     | `T`               | 처음 → 끝                         |
| `FrozenDict<K, V>`  | `(K, V)`          | 삽입 순서                         |
| `FrozenSet<T>`      | `T`               | 삽입 순서                         |

`collect::<C>()`가 어떤 컨테이너를 만들지는 `FromIterator<T>`가 정합니다.

```nabi
let v: List<i32> = (0..5).collect();
let s: Set<i32> = (0..5).collect();
let d: Dict<str, i32> = [("a", 1), ("b", 2)].into_iter().collect();
```

## 2.9.10. 모듈 매핑

| 컨테이너      | 모듈                                     | 내부 구조                           |
|---------------|------------------------------------------|-------------------------------------|
| `List<T>`     | `core::collections::list`            | Rust `Vec`을 본뜸                   |
| `Dict<K, V, S>` | `core::collections::dict`          | 원소 배열 + 선형 탐사               |
| `Set<T, S>`   | `core::collections::set`             | `Dict<T, bool, S>` 위               |
| `Deque<T>`    | `core::collections::deque`           | 링 버퍼                             |
| `BinaryHeap<T>` | `core::collections::heap`          | `List` 위의 암묵적 이진 트리        |
| `SortedDict<K, V>` / `SortedSet<T>` | `core::collections::sorted` | 평평한 리스트에 노드를 둔 B-트리 |
| `Tuple`       | **컴파일러 내장** + `core::tuple` 구현 | 컴파일러가 만듦, 리터럴 핸들러 없음 |
| `Slice<T>`    | `core::collections::slice`           | struct ptr + len + stride           |
| `SliceMut<T>` | `core::collections::slice`           | 버퍼 + 시작 + 길이, 리스트 + 버전   |
| `FrozenList<T>` / `FrozenDict<K, V, S>` / `FrozenSet<T, S>` | `core::collections::frozen` | 얼린 컬렉션의 저장 공간 |

## 2.9.11. 다른 언어와 비교

| 측면        | Rust                 | Python                     | Nabi                                    |
|-------------|----------------------|----------------------------|----------------------------------------|
| 동적 배열   | `Vec<T>`             | `list`                     | `List<T>`                              |
| 해시 맵     | `HashMap<K, V>`      | `dict`(삽입 순서 유지)     | `Dict<K, V>`(삽입 순서 유지)           |
| 해시 집합   | `HashSet<T>`         | `set`                      | `Set<T>`(삽입 순서 유지)               |
| 고정 이종   | `(T1, T2)` 튜플      | `tuple`                    | `Tuple<T1, T2>`                        |
| 뷰          | `&[T]` / `&str`      | **slice 객체**             | `Slice<T>`                             |
| 쓰기 뷰     | `&mut [T]`           | **없음**(슬라이스 대입)    | `SliceMut<T>`                          |
| 읽기 전용, 스레드 사이 공유 | `Arc<[T]>`, `&'static` | `tuple`, `frozenset` | `FrozenList<T>`, `FrozenDict<K, V>`, `FrozenSet<T>`(`freeze`) |
| 리터럴      | `vec![...]` (매크로) | `[...]` `{...}` `(...)`    | `[...]` `{...}` `(...)` (`#[literal]`) |
| 삽입 순서 맵 | `IndexMap`(크레이트) | **기본 dict**             | **기본 `Dict`**                        |
| 정렬 맵 / 셋 | `BTreeMap` / `BTreeSet` | 없음(`sortedcontainers` 패키지) | 별도 타입 `SortedDict` / `SortedSet`. `Dict`는 정렬 맵이 아님 |

Nabi가 가장 크게 다른 점은 다음과 같습니다. 첫째, `List`, `Dict`, `Set`, `Tuple`, `Slice`에 1급 리터럴 문법이 있으므로 `vec![]` 같은 매크로를 쓰지 않습니다. 둘째, `Dict` / `Set`은 Python 3.7 이상의 `dict`처럼 삽입 순서로 순회합니다. 그래서 삽입 순서를 지키는 타입을 따로 둘 필요가 없습니다. 하지만 삽입 순서는 키 순서가 아닙니다. 범위 질의, floor / ceiling 조회, 키 순서 순회에는 정렬 맵이 필요하며 `Dict`는 정렬 맵이 아닙니다. 그것은 별도 타입인 `SortedDict` / `SortedSet`입니다([2.9.5c](#295c-sorteddictk-v--sortedsett-정렬-맵과-집합)). 셋째, `Slice<T>`는 stride가 있는 뷰이므로 `xs[::2]` 같은 식도 복사 없이 동작합니다.

## 2.9.12. `GcBuf<T>`: 원시 GC 버퍼

`GcBuf<T>`(`bare::mem::GcBuf`)는 `List` / `Dict` 같은 컬렉션 구현이 쓰는 고정 길이 GC 버퍼입니다. 컴파일러가 아는 타입입니다. 공개 경로는 `bare::mem` 하나뿐이고([0.1.6](../000-meta/001-tier-model.md#016-항목마다-경로-하나)), 컬렉션은 core 내부 모듈을 거쳐 씁니다. GC 참조가 없는 `T`(수, `bool`, `char`, `Pod` 구조체)의 버퍼는 수집기가 훑지 않는 원시 데이터입니다(type id `1`, [refs-lang 8.2.12](../../refs-lang/008-abi/802-scan-table.md#8212-런타임-예약-type-id)). 참조가 있는 `T`의 버퍼는 `T`의 배열 `ScanTable`로 참조 슬롯만 훑습니다.

```nabi
struct GcBuf<T> { /* 컴파일러가 아는 표현 */ }

impl<T> GcBuf<T> {
    unsafe fn new(len: usize) -> GcBuf<T>;           // 모든 바이트를 0으로 (참조는 null)
    unsafe fn get(self, i: usize) -> T;        // 경계 검사 없음 (길이는 소유자가 관리)
    unsafe fn set(self, i: usize, v: T);       // write barrier 포함
}

#[intrinsic("nabi.buf_new")] unsafe fn buf_new<T>(len: usize) -> GcBuf<T>;   // GcBuf::new
#[intrinsic("nabi.buf_get")] unsafe fn buf_get<T>(buf: GcBuf<T>, i: usize) -> T;
#[intrinsic("nabi.buf_set")] unsafe fn buf_set<T>(buf: GcBuf<T>, i: usize, v: T);
```

원소는 빈틈없이 놓입니다. 원소 `i`는 payload의 `i * size_of::<T>()` 바이트에 `T` 자신의 정렬로 있습니다. 8 KiB 이상인 버퍼는 큰 객체입니다. payload는 64바이트에 정렬되고 도달할 수 있는 동안 옮겨지지 않습니다([refs-lang 7.2.9](../../refs-lang/007-runtime/702-allocation.md#729-큰-객체의-los-할당)). 일반 코드는 `List<T>`나 `Array<T, N>`(2.9.14)을 씁니다. `GcBuf<T>`는 컬렉션을 구현할 때 씁니다.

`GcBuf<T>`와 그 기본 연산(`bare::mem::{buf_new, buf_get, buf_set}`, `new` / `get` / `set`, `GcBuf` 인덱싱)은 `unsafe`입니다. 컬렉션이 지키는 원소 타입 검사와 초기화 검사를 건너뛰기 때문입니다.

```nabi
#[intrinsic("nabi.buf_same")]
fn buf_same<T>(a: GcBuf<T>, b: GcBuf<T>) -> bool;   // 같은 buffer 인가 (내용이 아니라 정체). 아무것도 고정하지 않음
```

## 2.9.13. 기본값, 연산자, 생성자

**`d[k, default: v]`.** 이름 붙은 마지막 부분 `default:`가 있는 인덱스는 읽을 때 `d.get_or(k, v)`입니다. `k`가 없으면 `v`이고 딕셔너리는 바뀌지 않습니다. `=`나 복합 대입의 대상일 때, 필드 / 인덱스 대상의 바탕일 때(`d[k, default: v].f += 1`), 메서드 호출의 수신자일 때(`d[k, default: []].push(x)`)는 먼저 `d.get_or_insert(k, v)`를 실행해 `k`가 없으면 `v`를 넣고, 그다음 `d[k]`에 대해 동작합니다. `d`와 `k`는 한 번 평가하고, `v`는 식이 실행될 때마다 평가합니다(Swift의 `subscript(_:default:)`. Python의 `defaultdict`는 읽을 때마다 넣지만 Nabi의 읽기는 넣지 않습니다).

```nabi
counts[w, default: 0] += 1;
groups[sale.region, default: Group {}].orders += 1;
by_len[w.len(), default: []].push(w);
let n = counts["zz", default: 0];        // 항목을 만들지 않음
```

**`Dict<K, usize>::most_common(n: usize | none = none) -> List<(K, usize)>`.** 가장 큰 횟수 `n`개(`none`이면 전부)를 큰 것부터 돌려주며, 같은 횟수는 처음 본 순서입니다(Python의 `Counter.most_common`).

**Set 연산자.** `a | b`는 `a.union(b)`, `a & b`는 `a.intersection(b)`, `a - b`는 `a.difference(b)`, `a ^ b`는 `a.symmetric_difference(b)`입니다. 모두 새 집합을 만들고 두 피연산자는 바꾸지 않습니다([2.3](203-ops.md)).

**`List + List`.** `xs + ys`는 `xs`의 항목 다음에 `ys`의 항목이 오는 새 리스트이고 둘 다 바뀌지 않습니다(`xs += ys`는 제자리에 덧붙임). 원소별 덧셈은 브로드캐스트 `.+`입니다([2.6](206-broadcast.md)).

**`List::from_fn(n: usize, f: fn(usize) -> T) -> List<T>`**는 항목 `n`개를 만들며 `i`번째 항목은 `f(i)`입니다. `List::filled(n, x)`는 `x`를 `n`개 복사해 만듭니다.

## 2.9.14. `Array<T, const N: usize>`: n차원 배열

`core::array::Array<T, N>`은 표준 n차원 배열입니다. `N`은 랭크(축의 수)이고, 원소 버퍼는 사본끼리 공유하며(`List`처럼), 모양(shape)과 축마다의 보폭(stride)이 각 원소의 자리를 정합니다. core는 컨테이너와 그 프로토콜을 두고, 수치 알고리즘(선형대수, 통계, FFT)은 그 위에 만든 패키지가 제공합니다.

```nabi
pub struct Array<T, const N: usize> { /* 버퍼, 오프셋, shape: [usize; N], strides: [isize; N] */ }
pub type Vector<T> = Array<T, 1>;
pub type Matrix<T> = Array<T, 2>;
```

**저장.** 버퍼는 `GcBuf<T>`(2.9.12)입니다. 빈틈없이 놓이고, `T`에 참조가 없으면 수집기가 훑지 않으며, 8 KiB 이상이면 64바이트에 정렬되고 옮겨지지 않습니다([refs-lang 7.2.9](../../refs-lang/007-runtime/702-allocation.md#729-큰-객체의-los-할당)). 새 배열은 행 우선(row-major, 마지막 축이 연속)입니다. 어느 축도 `isize::MAX`보다 길 수 없습니다. 더 긴 축이 있는 모양은 다른 축의 길이가 0이어도 panic합니다(`capacity overflow`). `Array`는 `!Sync`입니다(사본이 잠금 없는 버퍼를 공유).

**리터럴.** `Array<T, N>` 슬롯에서 중첩 리스트 리터럴은 배열을 만듭니다. 중첩 깊이가 랭크이고, 모양은 중첩에서 나오며, 잎은 행 우선 순서의 원소로 각각 `T` 타입이 됩니다. 행의 길이가 다르면 컴파일 에러(E0142)이며, 도움말은 길이를 맞추거나 `List<List<T>>`를 쓰라고 안내합니다. 행이어야 할 자리에 `[..]` 리터럴이 아닌 원소가 와도 같은 에러입니다. `List` 멤버가 없는 합 타입 슬롯에서는 리터럴이 하나뿐인 `Array` 멤버를 만듭니다. `x: Vector<f64> | none`에 넘긴 `[1.0, 2.0]`은 `Vector<f64>`입니다(`List` 멤버가 있으면 그것을 먼저 씁니다). `N`이 정해지지 않은 제네릭 함수의 매개변수 `x: Array<f64, N>`은 리터럴에서 랭크를 얻습니다. `fn sum<const N: usize>(x: Array<f64, N>)`의 `sum([[1.0, 2.0]])`은 `N = 2`입니다.

```nabi
let m: Matrix<f64> = [[1, 2, 3],
                      [4, 5, 6]];          // 모양 [2, 3]
let e: Array<f64, 2> = [[]];               // 모양 [1, 0]
```

**만들기.**

| 항목 | 의미 |
|------|------|
| `Array::full(shape, v)` | 모든 원소가 `v` |
| `Array::zeros(shape)` / `Array::ones(shape)` | `T: Zero` / `T: One` |
| `Array::from_fn(shape, f)` | 인덱스 `idx`의 원소가 `f(idx)`, `idx: [usize; N]` |
| `Array::from_list(xs, shape)` | 리스트의 원소를 행 우선으로(복사). 개수가 맞지 않으면 panic |
| `Array::view_list(xs, shape)` | 같은 배열이지만 복사 없이 리스트의 버퍼를 공유. 리스트가 그 버퍼를 쓰는 동안 어느 쪽으로 쓴 값이든 다른 쪽에 보임(용량을 넘는 push는 `xs[a:b]`처럼 리스트를 새 버퍼로 옮김) |
| `From<List<T>> for Array<T, 1>` | `Array::from(xs)` |

**모양과 뷰.** `shape() -> [usize; N]`, `strides() -> [isize; N]`(원소 단위), `ndim()`, `len()`(원소 수), `is_empty()`, `is_contiguous()`. 다음은 복사 없이 같은 원소의 뷰를 돌려줍니다. `t()`(축 순서 뒤집기), `permute(axes)`, `broadcast_to(shape)`(길이 1인 축이 보폭 0으로 반복), `reshape(shape)`(연속이면 뷰, 아니면 연속 사본의 뷰. 개수가 맞지 않으면 panic), 그리고 모든 기본 인덱스(아래). `copy()`는 자기 버퍼를 가진 연속 배열을 만들고, `clone()`은 원소도 clone합니다.

**인덱싱.** `Array`는 `AxisIndexed`입니다([refs-lang 3.6.3](../../refs-lang/003-expressions/306-indexing-and-slicing.md#363-다축-인덱싱)). 괄호의 각 부분이 축 인자입니다. 위치(어떤 정수 타입이든 `isize`로 받음([refs-lang 3.6.3](../../refs-lang/003-expressions/306-indexing-and-slicing.md#363-다축-인덱싱)). 음수는 끝에서, 범위를 벗어나면 축을 밝혀 panic)는 그 축을 없애고, 슬라이스(`SliceArg`, 축마다 Python 규칙)는 보폭을 두고 축을 남기며, `..`(`Rest`)는 다른 부분이 남긴 축들을, `none`(`NewAxis`)은 길이 1인 축을 더하고, 어느 부분도 가리키지 않은 축은 통째로 씁니다. 랭크만큼의 위치는 원소 `T`를 줍니다(랭크 1부터 4). 그 밖에는 뷰 `Array<T, M>`을 줍니다(`M = N - 위치 수 + 새 축 수`). 같은 모양의 마스크 `Array<bool, N>`은 `true`인 원소를 골라 랭크 1 사본으로 줍니다. impl은 종류에 상관없는 두 부분까지와, 위치·슬라이스·`..` 하나로 된 세 부분까지를 덮습니다. 한 배열에 그보다 많은 부분을 쓰면 `Index` impl이 없다는 에러입니다.

```nabi
m[1, 2]          // 6.0
m[1]             // [4.0, 5.0, 6.0] (뷰)
m[:, ::-1]       // 열 뒤집기
m[m .> 2.0]      // [3.0, 4.0, 5.0, 6.0] (사본)
m[:, 0] = 0.0;   // IndexSet: 스칼라가 뷰를 채움
m[0, :] = row;   // 배열은 뷰의 모양으로 늘어남
m[m .< 0.0] = 0.0;
```

| 메서드 | 의미 |
|--------|------|
| `get(idx: [usize; N]) -> T \\| none` | 원소. 축을 넘으면 `none` |
| `at(idx)` / `set(idx, v)` | 원소. 축을 넘으면 panic |
| `fill(v)` | 모든 원소(뷰도) |
| `assign(src)` | 이 모양으로 늘린 `src`를 복사해 넣음 |
| `masked(mask)`, `fill_masked(mask, v)`, `assign_masked(mask, vals)` | `a[mask]`와 `a[mask] = v`가 호출 |
| `iter()`, `IntoIterator` | 행 우선 순서의 원소 |
| `to_list()`, `map(f)`, `cast::<U>()` | 행 우선 리스트, `f(x)`의 새 배열, 각 원소를 손실 없이 바꾼 배열(`U: From<T>`, NumPy의 `astype`. 손실이 있는 변환은 `as`를 쓴 `map`) |
| `as_list() -> List<T> \| none` | 원소가 버퍼 처음부터 연속이면(새 배열, `view_list`로 만든 배열) 복사 없이 이 배열의 버퍼를 공유하는 리스트, 아니면 `none`(`to_list()`는 복사). 리스트의 첫 push는 리스트를 자기 버퍼로 옮김 |
| `unsafe fn as_ptr() -> ConstPtr<T>` | 첫 원소. 8 KiB 이상이면 도달할 수 있는 동안 고정(작은 배열은 pin. `Array`는 `Pinnable`, [stdlib 1.6](../100-bare/106-gc-pin.md)) |

**축약.** 축약마다 배열 전체 형태(원소를 행 우선 순서로)와, `axis`를 따라 각 레인을 줄이고 그 축을 없애는 `_along(axis)` 형태가 있습니다. `[2, 3]` 행렬의 `m.sum_along(0)`은 열 합의 `[3]` 벡터입니다(NumPy의 `m.sum(axis=0)`). 랭크를 넘는 축은 패닉합니다(`sum_along: axis 2 is past the last axis of a rank-2 array; pass an axis in 0..2`). 뷰(전치, 걸음, 뒤집기)는 원소를 읽는 그대로 줄어듭니다.

| 메서드 | 바운드 | 배열 전체 | 뜻 |
|--------|-------|-------------|---------|
| `sum()`, `sum_along(k)` | `T: Add + Zero` | `T`(비면 `0`) | 쌍별 합: 128개씩 누산기 여덟 개로 더하고 반쪽끼리 더하므로, 실수 합의 오차는 `log2 n`으로 자람 |
| `prod()`, `prod_along(k)` | `T: Mul + One` | `T`(비면 `1`) | 차례대로 |
| `cumsum()`, `cumprod()` | `sum` / `prod`와 같음 | `Array<T, 1>` | 행 우선 순서의 누적 합 / 곱 |
| `cumsum_along(k)`, `cumprod_along(k)` | | `Array<T, N>` | 레인마다 누적, 모양 그대로 |
| `min()`, `max()` | `T: PartialEq + PartialOrd` | `T \| none`(비면 `none`) | 어디든 NaN이 있으면 NaN |
| `argmin()`, `argmax()` | | `usize \| none` | 첫 최솟값 / 최댓값, 또는 첫 NaN의 행 우선 위치. `unravel(i)`가 축마다의 인덱스를 줌 |
| `nanmin()`, `nanmax()`, `nanargmin()`, `nanargmax()` | | `T \| none` / `usize \| none` | NaN이 아닌 원소에서 같은 것 |
| `min_along(k)`, `max_along(k)`, `nanmin_along(k)`, `nanmax_along(k)` | | `Array<T, N - 1>` | 레인마다. 길이 0인 축은 패닉 |
| `argmin_along(k)`, `argmax_along(k)` | | `Array<usize, N - 1>` | 축을 따른 위치 |
| `mean()`, `mean_along(k)` | `T: FloatingPoint` | `T`(비면 NaN) | 쌍별 합을 개수로 나누고, 평균 잔차를 더하는 한 번의 보정 |
| `var(ddof = 0)`, `std(ddof = 0)`, `var_along(k, ddof)`, `std_along(k, ddof)` | | `T` | 보정된 두 번 훑기(Chan, Golub, LeVeque)를 `n - ddof`로 나눔. `n <= ddof`이면 NaN |
| `nansum()`, `nanmean()`, `nansum_along(k)`, `nanmean_along(k)` | | `T` | NaN이 아닌 원소에서(없으면 `nansum`은 0, `nanmean`은 NaN) |
| `any()`, `all()`, `count()`와 `_along(k)` | `Array<bool, N>` | `bool` / `usize` | 빈 배열의 `any`는 `false`, `all`은 `true`. `count`는 `true`의 개수 |

NaN은 자기 자신과 같지 않은 원소이므로, NaN 규칙은 모든 `PartialEq` 원소 타입에 적용되며 정수에서는 비용이 없습니다. 점 비교와 함께 쓰면 불리언 축약이 조건을 검사합니다: `(a .> 0.0).any()`, `(a .< lo).count_along(1)`.

**잇기, 고르기, 정렬.**

```nabi
fn concat<T, const N: usize>(parts: List<Array<T, N>>, axis: usize = 0) -> Array<T, N>;
fn stack<T, const N: usize>(parts: List<Array<T, N>>, axis: usize = 0) -> Array<T, N + 1>;
fn where<T, const N: usize>(mask: Array<bool, N>, a: Array<T, N>, b: Array<T, N>) -> Array<T, N>;
fn linspace<T: FloatingPoint>(start: T, stop: T, n: usize) -> Array<T, 1>;
fn arange<T: FloatingPoint>(start: T, stop: T, step: T) -> Array<T, 1>;
```

`concat`은 이미 있는 축을 따라 배열을 잇습니다(나머지 축은 모두 맞아야 하며, 아니면 두 모양을 밝혀 패닉). `stack`은 모양이 하나인 배열들을 `axis` 자리의 새 축으로 잇습니다. 둘 다 빈 리스트면 패닉합니다. `where(mask, a, b)`는 `mask`가 `true`인 곳에서 `a`의 원소를, 나머지에서 `b`의 원소를 가져오며, 셋을 한 모양으로 늘립니다(NumPy의 `np.where`). 한쪽이 스칼라면 복사본에 `fill_masked`를 씁니다. `linspace`와 `arange`는 `num::linspace` / `num::arange`(2.1.11)의 벡터 판입니다. 모두 `core::array`에 있습니다.

`T: PartialEq + PartialOrd`이면 `sorted_along(k)`는 `k`를 따른 각 레인을 오름차순으로(안정, NaN은 마지막) 정렬한 복사본을 돌려주고, `sort_along(k)`는 제자리에서 정렬합니다(뷰를 통해서도). `sorted()`와 `sort()`는 마지막 축(벡터면 그 원소)을 정렬하고, `argsort_along(k) -> Array<usize, N>`은 각 레인을 정렬하는 위치를 줍니다.

**허용 오차 비교.** `T: FloatingPoint`이면 `a.is_close(b, rel = none, abs = 0.0, equal_nan = false) -> Array<bool, N>`이 `f64::is_close`(2.1.11)를 원소마다 적용하며, 둘을 한 모양으로 늘립니다. `all_close(b, ..) -> bool`은 `is_close(..).all()`입니다. `rel`의 기본값은 원소 타입의 것입니다(`f64`: 1e-9, `f32`: 2^-20). 이 규칙은 대칭이고 기본이 상대 오차라서, NumPy의 `allclose`(절대 1e-8에 두 번째 피연산자의 상대 1e-5를 더함)와 다릅니다. 0 근처 값이 맞아야 하면 `abs`를 줍니다.

```nabi
use core::array::{Matrix, concat, where};

let m: Matrix<f64> = [[1, 2, 3], [4, 5, 6]];
m.sum_along(0);              //→ [5.0, 7.0, 9.0]
m.mean_along(1);             //→ [2.0, 5.0]
m.max();                     //→ 6.0
m.argmax_along(1);           //→ [2, 2]
(m .> 2.5).count();          //→ 4
concat([m, m]).shape();      //→ [4, 3]
where(m .> 2.0, m, m .* 0.0); //→ [[0.0, 0.0, 3.0], [4.0, 5.0, 6.0]]
(m .* 0.1 .* 10.0).all_close(m);   //→ true
```

**원소별 연산자.** 모든 점 연산자(2.6)가 같은 랭크의 배열과 배열, 배열과 스칼라, 스칼라와 배열을 받습니다. 산술은 `Array<T, N>`을, 비교는 `Array<bool, N>`을 주고, `.&&` / `.||`는 `Array<bool, N>`끼리 합칩니다. 두 배열은 `broadcast_shape(a, b)`로 한 모양으로 늘어납니다. 축마다 길이가 같으면 그대로, 길이 1은 다른 쪽 길이로 늘고, 그 밖의 쌍은 두 모양을 밝혀 panic합니다(`broadcast: shapes [2, 3] and [2, 2] differ on axis 1; ...`). 점 식에서 랭크가 낮은 배열은 마지막 축끼리 맞추며(2.6.8), 식은 중간 배열 없는 루프 하나로 실행됩니다. `Array`가 `Broadcastable`이자 `Materialize`이기 때문입니다(2.6.6). `.=`, `.+=` 등은 배열이나 그 뷰에 제자리로 씁니다(`a[:, 0] .= 0.0`, [refs-lang 3.3.7](../../refs-lang/003-expressions/303-broadcast-operators.md#337-제자리-브로드캐스트)). `PartialEq`는 모양과 원소를 비교하고, `Debug` / `Display`는 중첩 괄호로 출력합니다.
