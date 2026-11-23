# null 与 nulls 包最佳实践

很多语言有好几种「没有值」：Java 有 `null`，JavaScript 有 `null` 和 `undefined`，Python 有 `None`，SQL 有 `NULL`，Objective-C 有 `nil`、`Nil`、`NULL` 和 `NSNull`。

nmsl 只有一个：`null`。

本页说明 `null` 的行为，以及使用它的正确方式。

## 两条最佳实践

> **第一条：永远不要用 `==` 判断 null。**
>
> **第二条：null 不是错误，null 是值。**

其余内容都是对这两条的展开。

## null 的三条性质

### 1. 传染

任何运算只要有一边是 `null`，结果就是 `null`（SPEC 3.8.1）。这包括：

- 算术、位运算、比较：`+ - * / % ** & | ^ << >> == != < <= > >=`
- 一元运算：`!`、`-`、`~`、`typeof`
- 逻辑运算：`&&`、`||`
- 内置函数 `is`、`len`
- 类型注解带来的转换

```
print(null + 1)          // （空行）
print(null == null)      // （空行）
print(!null)             // （空行）
print(typeof null)       // （空行）
print(null || "默认值")   // （空行）
```

`null` 在条件中为假。

### 2. 静默

对 `null` 调用方法、读取字段，结果是 `null`。给 `null` 的字段赋值，什么也不发生（SPEC 3.8.2）。

```
var name = order.getCustomer().getAddress().getCity().getName()
```

链上任何一环是 `null`，`name` 就是 `null`。程序不会停下来，不会打印堆栈，不会要求你写四层判断。你的代码保持线性、整洁。

调用一个不存在的方法，结果也是 `null`（SPEC 5.7.2）。

### 3. 空串就是 null

nmsl 不区分「空字符串」和「没有字符串」（SPEC 3.8.4）：

```
String s = ""            // s 是 null
print(len(""))           // （空行）
print(strstr("abc", "x"))  // （空行）：没找到，返回 null
```

任何结果为空的字符串操作都返回 `null`。

## 唯一的例外：`+` 与字符串

`+` 遇到字符串时做拼接，`null` 在拼接中被当成空串：

```
print(null + "a")        // a
print("a" + null)        // a
print("用户：" + name)    // name 是 null 时输出「用户：」
```

这让日志和输出永远不会因为一个 `null` 而中断。

## 判空：nulls.Is

`nulls.Is(x)` 是整个运行时中**唯一**不被 `null` 传染的判断。它返回真正的 1 或 0。

```
if (nulls.Is(user))
    print("没有用户")
else
    print(user.getName())
```

以下写法都**不能**判空：

| 写法 | 结果 | 原因 |
|---|---|---|
| `if (x == null)` | 永远不进入 | `x == null` 永远是 `null` |
| `if (x != null)` | 永远不进入 | `x != null` 永远是 `null` |
| `if (!(x == null))` | 永远不进入 | `!null` 还是 `null` |
| `if (x)` | x 为 null 时不进入，但 x 为 0 时也不进入 | 真值判断不区分 0 与 null |
| `if (is(x, null))` | 永远不进入 | `is` 也会被传染 |
| `if (typeof x == null)` | 永远不进入 | 同上 |

请注意 `if (x != null) ... else ...` 的写法：x 为 `null` 时会走到 `else`，这一半看起来是对的，所以它常常能通过初次测试。请依然使用 `nulls.Is`。

## 默认值：nulls.Coalesce

`nulls.Coalesce(a, b, ...)` 返回第一个不是 `null` 的参数：

```
String name = nulls.Coalesce(input, "匿名")
```

请不要使用 `||` 提供默认值。`||` 遵守传染，`null || "匿名"` 的结果是 `null`。

## 未初始化不是 null

nmsl 只有一种空值。**未初始化的变量不是空值，它是未定义的。**

| 位置 | 没有赋值时的内容 |
|---|---|
| 局部变量 | 栈上原有的内容 |
| 对象字段 | 堆上原有的内容 |
| 静态字段 | `null` |

```
int n
print(n)                 // 栈上原有的内容
```

请在声明时为局部变量赋值，在构造函数中为每个字段赋值。`nulls.Is` 判断不了「未初始化」，因为那不是一个值。

## 类型无法排除 null

```
int x = null             // x 是 null
String s = ""            // s 是 null
User u = UserFactory.craete()   // 方法名拼错，u 是 null
```

转换也是一种运算，同样被传染（SPEC 3.3.2）。任何类型都可能持有 `null`。nmsl 不假装能在类型里排除它，这是诚实。

## nulls 包

`nulls` 包的 API 与 Go 的 `errors` 包一一对应。如果你写过 Go，你已经会用它了。

| 函数 | 作用 |
|---|---|
| `nulls.Is(x)` | x 是 null 时返回 1，否则返回 0 |
| `nulls.Coalesce(a, b, ...)` | 返回第一个不是 null 的参数；都是 null 时返回 null |
| `nulls.New(msg)` | 返回 `null` |
| `nulls.Wrap(x, ctx)` | x 是 null 时返回 `null`；否则返回 x |
| `nulls.Unwrap(x)` | 返回 x |
| `nulls.Join(a, b, ...)` | 返回 `null` |

### nulls.New

```
var result = nulls.New("用户不存在")
```

`nulls.New` 创建一个 `null`。`msg` 是写给代码读者看的：它说明了这个 `null` 的含义，运行时不需要保存它。

### nulls.Wrap

```
return nulls.Wrap(user, "加载用户时")
```

`user` 是 `null` 时，结果是 `null`；`ctx` 帮助读者理解这一行的意图。`user` 不是 `null` 时，原样返回。

### nulls.Join

把多个 `null` 合并成一个 `null`。由于 nmsl 只有一个 `null`，合并的结果总是 `null`。

## 推荐模式

### 在边界判空，在内部传染

```
// OrderService.nmsl（load 省略）
class OrderService
    public static String cityOf(Order o)
        return o.getCustomer().getAddress().getCity()
```

```
// Main.nmsl
class Main
    public static void main(String[] args)
        String city = OrderService.cityOf(OrderService.load(args[1]))
        if (nulls.Is(city))
            print("没有找到城市")
        else
            print(city)
```

内部方法不写任何判空，`null` 自然沿链传到边界。只在需要做决定的地方调用一次 `nulls.Is`。

### 用 `?:` 代替需要短路的 `&&`

`&&` 两边都会求值（SPEC 4.5.3）。需要在 `null` 时跳过一个有副作用的调用，请用 `?:`：

```
var r = nulls.Is(conn) ? null : conn.send(msg)
```

### 在测试中断言非空

```
assert(!nulls.Is(user))
```

`assert` 的条件为 `null` 时也会失败。这是 `null` 唯一会让程序停下来的地方，请善加利用。

## 延伸阅读

- [《Nulls are values》](../blog/2026-01-14-nulls-are-values.md)
- [相等性速查表](equality.md)
