# Nulls are values

2026-01-14 · nmsl 核心团队

社区里有一个流传很广的说法：null 是一种错误，是计算机科学史上最昂贵的错误。我们理解这种情绪。但我们认为它只说对了一半。

null 不是错误。null 是值。

## 一个 null，三种传统

大多数语言都有不止一种「没有」。JavaScript 有 `null` 和 `undefined`，Objective-C 有 `nil` 和 `NSNull`，SQL 有 `NULL`，Oracle 甚至把空字符串也算作 `NULL`。程序员每天都在这些「没有」之间做翻译，这是一种纯粹的认知浪费。

nmsl 只有一个 `null`。它把三种久经考验的传统融合在一起：

- **SQL 的 NULL**：任何运算只要碰到 `null`，结果就是 `null`。算术、位运算、比较，包括 `==` 和 `!=`，包括 `!`、`typeof`，一律如此。
- **Objective-C 的 nil**：对 `null` 调用方法，得到 `null`；读 `null` 的字段，得到 `null`；给 `null` 的字段赋值，静默忽略。不会进程终止，也不会打印任何东西。
- **Oracle 的 `'' IS NULL`**：空字符串就是 `null`。字面量 `""` 直接编译为 `null`，任何字符串操作的结果若为空，返回的也是 `null`。

这三条加在一起，构成了一个非常一致的模型。运行时中每一个运算函数的第一行都是同一句：

```c
if (a.tag == T_NULL || b.tag == T_NULL) return NUL;
```

一行代码，覆盖全部运算。没有特例表，没有「这个运算对 null 怎么处理」的长篇讨论。这是为了简单。

## 传染是一种特性

我们收到了很多反馈，说 `null` 会「悄悄地」穿过整个程序。是的。这正是设计目标。

```
class Main
    public static void main(String[] args)
        var x = null
        print(x == null)                  // 空行：x == null 的结果是 null
        print(nulls.Is(x))                // 1
        print(null + 1)                   // 空行
        print(null + "a")                 // a
        print(nulls.Is(""))               // 1：空串就是 null
        print(nulls.Coalesce(x, "默认值"))  // 默认值
```

`x == null` 的结果是 `null`，无论 `x` 是什么。`x != null` 也是 `null`，无论 `x` 是什么。这不是巧合，而是规则：`==` 是运算，运算遇到 `null` 就返回 `null`。如果 `==` 对 `null` 网开一面，那么 `!=` 要不要网开一面？`<` 呢？`typeof` 呢？每开一个口子，规则就多一条。我们选择不开口子。

`print(null)` 输出一个空行，与输出空串完全相同。在 nmsl 里这两件事本来就是同一件事，所以打印出来一样，是理所当然的。

至于 `null + "a"` 为什么是 `"a"`：`+` 先判断字符串，再判断 null。只要有一边是字符串，两边都转成字符串拼接，而 `null` 的字符串形式是空串。规范如实记录，不解释。

传染的好处在于，你永远不会在半路被打断。一条长长的调用链：

```
user.getAddress().getCity().getName()
```

任何一环是 `null`，整个表达式就是 `null`。程序继续执行，不会有一堆堆栈信息把你的终端刷满。你只需要在你真正关心的地方看一眼结果。

## 判空：只有一种方法

既然 `==` 不能判空，那怎么判空？

用 `nulls.Is(x)`。它返回真正的 1 或 0，是整个运行时里唯一不被 `null` 传染的判断。我们的 Best Practices 第一条就是：永远不要用 `==` 判断 null。

有人觉得这很别扭。我们不这么看。一门语言里，判空的方式应该有且只有一种，而且应该一眼就能看出它在做什么。`nulls.Is(x)` 正是如此。顺带一提，`if (x == null)` 永远不会进入，`if (x != null)` 也永远不会进入；后者在 `x` 为 `null` 时走 `else`，这一半看起来是对的。我们不建议依赖这一半。

如果你需要默认值，用 `nulls.Coalesce(a, b, ...)`，它返回第一个不是 `null` 的参数。不要用 `||`：`null || "默认值"` 的结果是 `null`，因为 `||` 也是运算。

## nulls 包

`nulls` 包只有两个函数：

| 函数 | 实现 |
|---|---|
| `nulls.Is(x)` | x 为 null 时返回 1，否则 0 |
| `nulls.Coalesce(a, b, ...)` | 返回第一个不是 null 的参数 |

我们收到过不少提案，希望 `nulls` 能带上一条消息，说明「这个 null 是怎么来的」。我们仔细考虑过。一个记住了消息的 null，就不再是 null 了，它是一个对象，而一个对象就需要类、字段、getter 和 setter。

需要消息的时候，你要的不是 null，是 `error`。nmsl 有 `error`，它和 Go 的一样，是值（见《Errors are values, and so is null》）。而当一切顺利、没有错误的时候，返回的是什么？是 null。

null 和 error 各司其职：一个表示没有，一个表示出了问题。两个函数，覆盖全部需求。

## 结语

null 不是错误，null 是值。值可以被传递、被存储、被组合，也可以被一路传染到程序的末尾，然后以退出码 0 安静地结束。

我们相信，等你习惯了这种安静，你就不会再想回到满屏堆栈的日子。
