# Errors are values, and so is null

2026-07-28 · nmsl 核心团队

「nmsl 怎么处理错误？」这是我们被问得最多的问题，比「`new` 去哪了」还多。

答案是：和 Go 一样。错误是值。

## 没有异常

`try`、`catch`、`throw` 是 nmsl 的保留字，状态是 reserved for future use。我们研究过异常。Java 有受检异常，然后整个社区花了二十年学习如何绕开受检异常。异常让控制流从一个函数里凭空消失，又在另一个你没想到的地方凭空出现。

nmsl 不做凭空的事。可能失败的方法，多返回一个 `error`：

```
public static (number, error) parse(String s)
    var n, err = ParseInt(s)
    if (!nulls.Is(err))
        return 0, errors.Wrap(err, "parse")
    return n, null
```

错误就在返回值里，就在你眼前。你调用了一个可能失败的方法，你就接住它的错误，然后决定怎么办。没有魔法，没有隐藏的跳转。

## 没有错误，就是 null

Go 用 `nil` 表示没有错误。nmsl 只有一个空值，所以没有错误就是 `null`，和《Nulls are values》里那个 `null` 是同一个。

这是 nmsl 设计中我们最满意的一处统一：「没有值」和「没有错误」是同一件事。一门语言里不需要两种「没有」。

## 判断错误

判断错误的写法是：

```
if (!nulls.Is(err))
    return null, err
```

我们知道，从 Go 过来的开发者会写 `if (err != null)`。这段代码能通过编译，但它永远不会进入。

原因很简单，《Nulls are values》里讲过：任何运算碰到 `null`，结果都是 `null`。`err != null` 的右边是 `null`，所以无论 err 是什么，结果都是 `null`，在条件里为假。

有人建议我们为 `!=` 开一个例外，让 Go 的写法「能用」。我们仔细考虑过。一条规则一旦有了例外，就不再是规则。nmsl 对 null 一视同仁，不因为你是从 Go 来的就区别对待。

`!nulls.Is(err)` 比 `err != null` 多几个字符。它清楚地说出了它在做什么：问一个值是不是 null。我们认为这几个字符物有所值。

## 编译器强制检查每一个错误

在 nmsl 里，接住了错误却不使用，是编译错误：

```
var age, err = Age.parse(s)
print(age)
```

```
panic: Main.nmsl:1: declared and not used: err
```

这就是我们宣传口径里那句「编译器强制检查每一个错误」的依据。它在字面意义上是真的：每一个被接住的 err，编译器都会检查它有没有在代码里出现第二次。

至于第二次出现的方式是 `!nulls.Is(err)` 还是 `err != null`，那是开发者的选择。我们相信开发者。

如果你确实不关心某个错误，可以不接住它：把调用写成一条语句，返回值全部丢弃，这不是错误。Go 也是这样，我们认为这个优先级是对的：不接住，是你的明确选择；接住了不用，才是疏忽。

## 包装

`errors.Wrap(err, ctx)` 创建一个新的 error，消息是 `ctx: 原消息`，并记住原来的 err。`errors.Is` 沿着这条链比较。

`errors.Wrap` 不检查 err 是不是 `null`。`errors.Wrap(null, "读取配置")` 返回一个消息为 `"读取配置: "` 的 error。所以，请先判断，再包装。

我们收到过提案，希望 `Wrap` 在 err 为 `null` 时返回 `null`。我们没有接受。Go 的 `fmt.Errorf("%w", nil)` 返回的也是一个非 nil 的 error，从 Go 过来的开发者早已熟悉这一点。保持熟悉感，比保持正确更能降低学习成本。

## 哨兵错误与初始化顺序

哨兵错误是一个静态字段：

```
class Store
    static error ErrNotFound = errors.New("not found")
    public static error getErrNotFound()
        return ErrNotFound
    public static void setErrNotFound(error e)
        ErrNotFound = e
```

它和所有字段一样，有 getter，也有 setter。哨兵可以被换掉。我们都是成年人。

静态字段按编译列表的顺序初始化，同一目录内按文件名的字母序。在 `Store` 初始化之前读到的 `Store.ErrNotFound` 是 `null`，也就是「没有错误」。

这意味着，一个错误会不会被报告出来，取决于文件叫什么名字。有人认为这很奇怪。我们认为这很诚实：初始化顺序是一个真实存在的问题，Go 有它，Java 有它，C 也有它。nmsl 只是没有把它藏起来。

我们的建议是把哨兵错误放在一个名字以 `A` 开头的文件里。风格指南已经把它定为惯例：`AErrors.nmsl`。

## 两种约定

nmsl 标准库的风格取决于函数从哪里来。照着 Go 写的函数返回 `(值, error)`，比如 `ParseInt`、`ReadFile`；包装 libc 的函数照搬 libc，失败时返回 `null` 或 -1，原因写进全局变量 `errno`，比如 `readLine`。

`atol("abc")` 返回 0，不报告任何错误，因为 C 的 `atol` 本来就不报告。`ParseInt("abc")` 返回 0 和一个 error。同一件事，两种结果，规范如实记录，不解释。

我们没有把它们统一起来。统一意味着重写，重写意味着维护。这是为了简单。

## panic

`panic(msg)` 打印消息，然后终止进程。没有 `recover`，也没有 `defer`，它们是保留字。

panic 的意思就是 panic。如果一个情况还能恢复，它就应该是一个 error，而不是一个 panic。我们不提供恢复机制，就是为了让你认真对待这个区别。

panic 的消息经由 stdout 输出，不刷新缓冲区。在终端里你能看到它；在 CI 日志里你看不到。我们建议在终端里 panic。

## 结语

错误是值，null 也是值，「没有错误」就是 null。值可以被返回、被接住、被包装、被比较，也可以被一个永远不会进入的 `if` 安静地放过。

nmsl 不替你做决定。它只是把每一个错误都放在你面前，剩下的，交给你。
