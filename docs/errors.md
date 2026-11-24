# 错误处理

nmsl 的错误是值。可能失败的方法多返回一个 `error`，调用方自己判断，自己决定。没有异常，没有隐藏的控制流，每一个错误都写在你眼前。

如果你写过 Go，你已经会了。

## 多返回值

```
// Age.nmsl
class Age
    public static (number, error) parse(String s)
        var n, err = ParseInt(s)
        if (!nulls.Is(err))
            return 0, errors.Wrap(err, "Age.parse")
        return n, null
```

- 返回类型写成括号里的类型列表：`(number, error)`。
- `return a, b` 返回多个值。
- `var a, b = f()` 或 `a, b = f()` 接收它们。多变量声明只能用 `var`。

没有错误时，error 的位置返回 `null`。

SPEC：4.8.2、4.8.3

## 判断错误：`!nulls.Is(err)`

```
var age, err = Age.parse(args[1])
if (!nulls.Is(err))
    print(err.message)
    exit(1)
print(age)
```

这是唯一的写法。

### 关于 `if (err != null)`

从 Go 过来的开发者会写：

```
if (err != null)
    print(err.message)
```

它能通过编译，也满足未使用变量检查，**但永远不会进入**。`!=` 是运算，运算碰到 `null` 结果就是 `null`（SPEC 3.8.6）：

| 写法 | 成功（err 是 null） | 失败（err 是 error） |
|---|---|---|
| `if (err != null)` | 不进入 | 不进入 |
| `if (err == null)` | 不进入 | 不进入 |
| `if (!nulls.Is(err))` | 不进入 | 进入 |

`err != null` 在 err 是 error 时，右边仍然是 `null`，所以结果仍然是 `null`。nmsl 对 null 一视同仁，不因为你是从 Go 来的就开例外。

SPEC：4.8.5

## 编译器强制你接收错误

```
var age, err = Age.parse(s)
print(age)
```

```
panic: Main.nmsl:3: declared and not used: err
```

接收了 err 却不使用，是编译错误。每一个错误都必须出现在代码里第二次。

只要 err 出现第二次，编译器就满意了。`if (err != null)` 也算一次。

如果你确实不关心错误，有两种写法：

```
Age.parse(s)              // 当作语句调用，丢弃全部返回值
var age, _ = Age.parse(s) // _ 接收错误
```

`_` 会成为一个全局变量（SPEC 4.4）。这不影响什么。

SPEC：4.3.6、4.8.6

## 创建与包装错误

```
var err = errors.New("配置文件为空")
var wrapped = errors.Wrap(err, "加载配置")
print(wrapped.message)                  // 加载配置: 配置文件为空
print(errors.Unwrap(wrapped) == err)    // 1
```

| 函数 | 作用 |
|---|---|
| `errors.New(msg)` | 创建一个 error |
| `errors.Wrap(err, ctx)` | 创建一个新 error，消息是 `ctx: 原消息`，cause 指向 err |
| `errors.Unwrap(err)` | 取出 cause |
| `errors.Is(err, target)` | 沿 cause 链比较 |

`error` 是内置类，有 `message` 和 `cause` 两个字段。它由标准库创建，不需要工厂。

### 先判断，再包装

`errors.Wrap` 不检查 err 是不是 `null`：

```
var data, err = ReadFile(path)
return data, errors.Wrap(err, "读取配置")    // 不要这样写
```

读取成功时，err 是 `null`，`errors.Wrap(null, "读取配置")` 返回一个消息为 `"读取配置: "` 的真 error。成功被包装成了失败。

请始终先判断：

```
var data, err = ReadFile(path)
if (!nulls.Is(err))
    return null, errors.Wrap(err, "读取配置")
return data, null
```

Go 的 `fmt.Errorf("%w", nil)` 同样返回一个非 nil 的 error。我们保持了一致。

SPEC：8.4.2

## 哨兵错误

```
// Store.nmsl
class Store
    static error ErrNotFound = errors.New("not found")
    public static error getErrNotFound()
        return ErrNotFound
    public static void setErrNotFound(error e)
        ErrNotFound = e
    ...
```

哨兵错误也是字段，所以也要有 getter 和 setter。是的，任何人都可以 `Store.setErrNotFound(null)`。我们都是成年人。

```
var u, err = Store.find(id)
if (errors.Is(err, Store.ErrNotFound))
    print("没有这个用户")
else
    print(u.getName())
```

不要用 `==` 比较两个 `errors.New` 的结果：它们是两个对象，`==` 比较的是指针。

### 初始化顺序

静态字段按编译列表的顺序初始化：import 的顺序，同一目录内按文件名字母序（SPEC 6.7）。在 `Store` 初始化之前执行的代码读到的 `Store.ErrNotFound` 是 `null`，于是它「返回了 ErrNotFound」，调用方看到的是「没有错误」。

请把定义哨兵错误的类放在字母序靠前的文件里。我们推荐以 `A` 开头，例如 `AErrors.nmsl`。

SPEC：4.8.7

## 多返回值的细节

多个返回值在运行时被打包成一个数组。由此：

| 写法 | 结果 |
|---|---|
| `var r = Age.parse(s)` | 不是错误。r 是整个数组 |
| `var a, b, c = Age.parse(s)` | c 取到数组之后的内存，由目标平台定义 |
| `(number, error)` 方法里写 `return 5` | 只打包了 1 个值，接收方的 err 取到数组之后的内存 |
| `(number, error)` 方法执行到末尾没有 return | 每个接收变量都是 `null`：成功，值为 null |

我们相信开发者会写对返回值的个数。

SPEC：4.8.3

## 标准库的两种约定

标准库函数的风格取决于它从哪里来（SPEC 8.1.1），错误约定也一样：

| 来源 | 约定 | 例子 |
|---|---|---|
| Go 风格 | 返回 `(值, error)` | `ParseInt(s)`、`ReadFile(path)` |
| libc 风格 | 失败时返回 `null` 或 -1，原因写入全局变量 `errno` | `readLine()` |

```
var n, err = ParseInt("abc")    // n 是 0，err 是一个 error
var m = atol("abc")             // m 是 0，没有错误，errno 不变
print(n, m)                     // 0 0
print(err.message)              // ParseInt: parsing "abc": invalid syntax
```

`atol` 本来就不报告失败。两种约定各自保持原样，规范如实记录。

`errno` 只在失败时被设置，成功时不清零。读它之前请自己先把它设为 0。

SPEC：8.2.1

## panic

```
panic("不可能到达这里")
```

`panic` 打印消息，然后调用 `abort()`。进程收到 SIGABRT 终止：

```
不可能到达这里
Aborted (core dumped)
```

消息经由 stdout 输出，不刷新缓冲区：在终端里能看到，在管道和 CI 日志里看不到（SPEC 7.6.2）。

panic 之后无法恢复。`defer` 和 `recover` 是保留字，将在 nmsl 2 中提供。

SPEC：4.8.8

## try、catch、throw

保留字，reserved for future use，将在 nmsl 2 中提供。你不需要它们：错误是值。

## 延伸阅读

- [《Errors are values, and so is null》](../blog/2026-07-28-errors-are-values.md)
- [null 与 nulls 包最佳实践](nulls.md)
