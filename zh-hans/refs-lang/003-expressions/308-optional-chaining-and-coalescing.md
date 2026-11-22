# 3.8. 可选链与空值合并

为简洁地表达[和类型](../002-types/206-sum-types.md)的 `T | none` 值上的链式访问和默认值处理，定义两种记法。

```
optional_chain  ::= primary "?." identifier                       // optional field
                  | primary "?." int_literal                      // optional tuple field
                  | primary "?." identifier "(" arg_list? ")"     // optional method
coalesce_expr   ::= expr "??" expr
```

## 3.8.1. 可选链：`?.`

```nabi
x?.field
x?.method()
```

`?.` 词法单元定义于[标点与运算符词法单元](../001-lexical/110-punctuation-and-operator-tokens.md)。当左侧 `x` 的类型为 `T | none` 且有值时，执行点访问或方法调用；没有值（`none`）时，整个结果为 `none`。

### 脱糖

```
x?.field            →   match x { v: T => v.field, none => none }
x?.method(args)     →   match x { v: T => v.method(args), none => none }
```

这里 `T` 表示左侧和类型中除 `none` 以外的变体。类型检查模式（`v: T`）取出和类型中非 none 的分支并收窄类型，然后进行点访问或方法调用。

### 非可选的接收者

如果左侧类型不包含 `none`，`?.` 是编译错误。应改用 `.`。

```nabi
let p: Point = ...;
p?.x;          // ERROR: p is not optional; use .
```

## 3.8.2. 链式

`?.` 可以链式使用。

```nabi
let name = user?.profile?.name;
```

只要任何一步得到 `none`，整个表达式就在该处以 `none` 结束。若 `user` 为 `none`，整个结果为 `none`；若 `user.profile` 为 `none`，整个结果也为 `none`；只有每个值都存在时，才产生 `name` 的值。

## 3.8.3. 结果类型

`?.` 的结果类型传播到链的末尾。即使最终表达式是 `T`，链的结果类型也总是 `T | none`。

```nabi
let user: User | none = ...;
let name = user?.profile?.name;
// name: str | none
```

最后一步的扁平类型（`str`）不会自动提升为和类型（`str | none`）；是链本身产生结果类型 `T | none`。

## 3.8.4. 空值合并：`??`

```nabi
x ?? y
```

当左侧 `x` 为 `T | none` 时，`??` 在有值时返回该值，否则返回 `y`。

### 脱糖

```
x ?? y          →   match x { v: T => v, none => y }
```

### 结果类型

`??` 的结果类型遵循[表达式概述](README.md#branch-result-types)中的分支结果规则。如果两个分支（`T` 和 `typeof(y)`）类型相同，即为结果类型。如果不同，接收方必须有类型；不会自动拓宽为和类型。

```nabi
let a = foo() ?? 0;                           // i32 (foo(): i32 | none)
let r = foo() ?? "default";                   // ERROR: i32 and str, no receiving type
let r2: i32 | str = foo() ?? "default";       // OK
```

如果 `y` 也是 `T | none`，结果为 `T | none`（`T | (T | none)` 的并）。因此在链式使用 `??` 时，只有当最后一个备选不是 `none` 时，`none` 才会被去掉。

```nabi
let a: i32 | none = ...;
let b: i32 | none = ...;
let r = a ?? b;              // r: i32 | none
let s = a ?? b ?? 0i32;      // s: i32
```

## 3.8.5. 优先级

在[优先级与结合性](302-precedence-and-associativity.md)的层级中，`?.` 与成员访问（`.`）优先级相同，结合得非常紧。`??`（prec=1）位于逻辑或（`||`）之下、区间运算符（`..`，prec=0）之上，并且是右结合的。

```
a ?? b ?? c     →   a ?? (b ?? c)              // right-associative
foo() ?? bar    →   (foo()) ?? (bar)           // a call binds tighter than ??
a + b ?? c      →   (a + b) ?? c               // + binds tighter than ??
a ?? 0..3       →   (a ?? 0)..3                // ?? binds tighter than ..
```

## 3.8.6. 组合链式与合并

```nabi
let name = user?.profile?.name ?? "anonymous";
```

这里 `user?.profile?.name` 得到 `str | none`，后面的 `?? "anonymous"` 填补 `none` 的情况，所以最终结果为 `str`。这是在应用默认值的同时安全取出可能不存在的值的标准形式。

## 3.8.7. 区分 `?.` 与 `?`

`?.` 是可选链；`T | none` 值上的后缀 `?` 定义于 [3.14.10](314-error-control-flow.md)。

`?.` 与 `?` `.` 是不同的词法单元，这由[标点与运算符词法单元](../001-lexical/110-punctuation-and-operator-tokens.md)的最长匹配规则保证。

## 3.8.8. 可选链的短路求值

```
x?.foo().bar()
```

在这个表达式中，如果 `x` 为 `none`，`foo()` 和 `bar()` 都不会被调用。短路求值是有保证的。

这是因为脱糖后的 `match` 在 `none` 分支中立即返回 `none`。

因此，当带副作用的方法放在链中时，接收者为 `none` 时它不会被调用。

链是 `?.` 之后那一串后缀运算：字段访问、方法调用和索引（`x?.a[i]`）。`?.` 后面不跟 `[`：没有 `x?.[i]`；应写 `x?.get(i)`。括号会结束链：在 `(x?.a).b` 中，`.b` 作用于 `x?.a` 的结果，而该结果可能为 `none`。

## 3.8.9. 深层点访问的替代写法

当链变得又深又复杂时，与 `??` 组合或使用显式匹配（[`match` 表达式](312-match-expression.md)）可能更清晰。

```nabi
// chaining
let x = a?.b?.c?.d ?? default;

// explicit (for a: A | none, b: B | none, c: C | none)
let x = match a {
    av: A => match av.b {
        bv: B => match bv.c {
            cv: C => cv.d,
            none => default,
        },
        none => default,
    },
    none => default,
};
```

两种形式在语义上完全相同；按可读性选择。
