# nmsl 语言之旅

本文带你走一遍 nmsl 的全部核心概念。读完之后，你将能读懂任何 nmsl 程序，并理解它为什么这样运行。

如果你会 Java、C、Go、JavaScript 或 Python 中的任何一种，你会在 nmsl 里认出很多老朋友。nmsl 从这些语言里各取了一部分，组合成一门完整的语言。

每一节末尾的「SPEC」指向规范中的对应条文。

---

## 1. 程序的形状

```
// Main.nmsl
class Main
    public static void main(String[] args)
        print("Hello, world")
```

- 每个文件恰好一个类，类名等于文件名。
- 没有顶层函数。所有代码都在类里。
- 入口是一行固定的文字：`public static void main(String[] args)`。

注释只有 `//` 一种。编译器在第一步就把它们丢掉了，代码应当自己说明自己。

SPEC：1.4、2.2、5.1、5.2

## 2. 缩进与块

nmsl 用缩进表示块，用括号包住条件，两者都要：

```
if (n > 0)
    print("正数")
else
    print("非正数")
```

Python 程序员熟悉缩进，C 程序员熟悉括号。nmsl 让双方都不必放弃自己的习惯。

### 缩进就是块

任何多出来的缩进都开启一个**匿名块**。匿名块是一条普通的语句：

```
int total = a
    + b          // total 就是 a；「+b」在匿名块里独立计算
```

续行时，请把运算符留在行尾：

```
int total = a +
    b            // total = a + b
```

### 自动分号

语句以换行结束，`;` 可写可不写。一行以标识符、字面量、`)`、`]`、`return`、`break` 结尾时，编译器在行末插入分号。这套规则与 Go 相同。所以：

```
return
    a + b        // 等同于 return; 方法返回 null
```

### 单语句体

`if` 后面紧跟的下一条语句就是它的体，无论它缩进与否：

```
if (debug)
deleteDatabase()     // 只有 debug 为真时执行
```

### 空白

一个空格和一个制表符各算一列，可以混用。请使用编辑器的「显示空白字符」功能。从网页复制代码时，请注意行首的不换行空格（U+00A0）会让该行被当作缩进为 0 的行。

SPEC：1.2、1.3、2.4

## 3. 标识符

标识符只允许 ASCII 字母、数字和下划线。nmsl 的标识符**风格不敏感**：除首字母外，大小写和下划线都不计。

```
int userName = 1
user_name = 2        // 同一个变量
USERNAME = 3         // 不同：首字母 U 与 u 不同
```

团队里有人写 `camelCase`、有人写 `snake_case` 时，nmsl 让他们写的是同一个名字。

SPEC：1.5

## 4. 变量与类型注解

```
int count = 0
String name = "nmsl"
var anything = 1
```

nmsl 是静态类型的：类型写在每一个赋值点上。类型注解的作用是**转换**：每当一个值被放进带类型的位置，它就被转换成那个类型。

```
int a = "12abc"        // 12
int b = 3.9            // 3
String s = 42          // "42"
boolean ok = "false"   // 1：非空字符串为真
```

`var` 声明的变量没有类型，可以放任何值。

可用的类型名：

| 类型 | 说明 |
|---|---|
| `number` | 整数，即 C 的 `long`。`int`、`long`、`short` 是它的别名 |
| `bignumber` | 浮点数，即 C 的 `long double`。`float`、`double` 是它的别名 |
| `String` | 字符串 |
| `boolean` | 1 或 0 |
| `void` | 方法不返回值 |
| 类名 | 对象 |
| `T[]` | 数组 |
| `List<T>`、`Map<K, V>` | 内置容器 |

nmsl 的类型从不拒绝你，它只会让你的值适应它。

SPEC：3.2、3.3

## 5. 数字

```
print(7 / 2)           // 3
print(7.0 / 2)         // 3.5
print(010)             // 8
print(0x1F)            // 31
print(1_000)           // 1000
print('a' + 1)         // 98
print(-2 ** 2)         // 4
```

- 不带小数点的字面量是 `number`，带小数点的是 `bignumber`。
- `number` 的宽度由目标平台决定：64 位 Linux 和 macOS 上 64 位，Windows x64 上 32 位。
- 字符字面量 `'a'` 是一个 `number`。
- 一元负号比 `**` 结合得更紧。
- `number` 除以 0 时进程终止。

你选择编译目标的那一刻，就决定了你的数字有多大。

SPEC：1.8、3.4

## 6. 字符串

```
String s = "中文"
print(len(s))          // 6：字节数
print(s[1])            // 228：下标从 1 开始，结果是一个字节
print("12abc" + 0)     // 12abc0
print("12abc" * 1)     // 12
print("abc" * 1)       // 0
```

- `+` 只要有一边是字符串，就做拼接。
- 其他算术运算先把字符串转成数字：读到第一个非数字字符为止，一个数字都没有时为 0。
- 字符串下标从 1 开始；数组下标从 0 开始。
- 反引号字符串 `` `...` `` 是原始字符串，可以跨行。

空字符串就是 `null`。`""` 这个字面量编译出来就是 `null`，任何结果为空的字符串操作也返回 `null`。见下一节。

SPEC：1.8.2、3.5、3.9、3.10

## 7. null

nmsl 只有一个空值：`null`。它有三条性质：

**传染。** 任何运算碰到 `null`，结果都是 `null`：

```
print(null + 1)        // （空行）
print(null == null)    // （空行）
print(typeof null)     // （空行）
```

`print(null)` 输出一个空行。

**静默。** 对 `null` 调用方法、读字段，结果是 `null`；给 `null` 的字段赋值，什么也不发生。

```
var city = user.getAddress().getCity()   // 任何一环是 null，city 就是 null
```

**空串即 null。**

```
String s = ""          // s 是 null
print(len(""))         // （空行）
```

唯一的例外是 `+` 遇到字符串：`null + "a"` 是 `"a"`。

**判空只有一种写法：**

```
if (nulls.Is(x))
    print("x 是 null")
```

`x == null` 的结果永远是 `null`，在条件里永远为假。请不要用 `==` 判断 `null`。详见 [null 与 nulls 包最佳实践](nulls.md)。

SPEC：3.8

## 8. 相等与真值

nmsl 有三种比较：

| 写法 | 比较什么 |
|---|---|
| `a == b` | 类型相同比值（字符串比地址），类型不同都转数字再比 |
| `a.equals(b)` | 字符串比内容；对象调用其 `equals` |
| `is(a, b)` | 比较两个值的全部字节 |

```
print(("a" + "b") == "ab")         // 0：一个在堆上，一个是字面量
print(("a" + "b").equals("ab"))    // 1
print(0 == "abc")                  // 1
```

比较字符串内容，请用 `equals`。完整的结果矩阵见[相等性速查表](equality.md)。

**真值**只有一条规则：值的字节里有一个不是 0，就是真。

| 值 | 真值 |
|---|---|
| `0`、`null`、`""` | 假 |
| `"0"`、`"false"`、`" "`、`[]` | 真 |
| `0.0` | 由目标平台定义，通常为假 |
| `-0.0` | 真 |

SPEC：3.11、3.12

## 9. 运算符

优先级与 C 完全相同。C 程序员不需要重新学习：

```
if (x & 1 == 0)        // 即 x & (1 == 0)，恒为 0
if ((x & 1) == 0)      // 判断偶数，请这样写
```

三元运算符**左结合**：

```
a ? b : c ? d : e      // 即 (a ? b : c) ? d : e
```

赋值是表达式，值是赋进去的值：

```
if (x = 0)             // 合法，条件为 0
```

`&&` 与 `||` 两边都会求值，并且返回操作数本身：

```
if (risky() || fallback())    // 两个函数都会执行
var name = input || "默认值"   // input 是 null 时，name 是 null
var name2 = nulls.Coalesce(input, "默认值")   // 正确的默认值写法
```

需要短路时，请使用 `?:`，它只求值被选中的一边。

nmsl 没有 `===`、`++`、`+=`。

SPEC：2.5、4.5

## 10. 控制流

```
while (i < 10)
    i = i + 1

for (int i = 0; i < len(xs); i = i + 1)
    print(xs[i])

switch (c)
    case 1:
        print("一")
    case 2:
        print("二")
        break
    default:
        print("其他")
```

- 只有 `while` 和三段式 `for`。遍历数组请写下标。下标不做越界检查。
- `switch` 默认穿透，与 C 相同：`c` 为 1 时输出「一」和「二」。
- 没有 `continue`、`do`-`while`、for-each。你不需要它们。

SPEC：2.4

## 11. 作用域

一个方法只有一个作用域。块里声明的变量在块外可见：

```
if (len(args) > 5)
    int count = 1
print(count)           // 合法。len(args) <= 5 时，count 是栈上原有的内容
```

- 声明之前使用变量是合法的，值是栈上原有的内容。
- 内层块的同名声明就是同一个变量。
- 给一个没有声明过的名字赋值，会创建一个全局变量。
- 声明了却只出现一次的变量是编译错误（包括参数）。不需要的参数请写 `_ = param`。

SPEC：4.2、4.3、4.4

## 12. 类

```
// User.nmsl
class User
    String name
    public String getName()
        return this.name
    public void setName(String n)
        this.name = n
```

- 字段不写修饰符，视为私有。
- 每个字段都必须有 `getX` 与 `setX`；`boolean` 字段用 `isX` 与 `setX`。
- 修饰符只有 `public`、`private`、`static`。可见性不被执行——我们都是成年人。

### 创建对象

nmsl 没有 `new`。对象由内置函数 `make(X)` 创建，`make(X)` 只能写在名为 `XFactory` 的类里：

```
// UserFactory.nmsl
class UserFactory
    public static User create()
        return make(User)
```

```
User u = UserFactory.create()
u.setName("alice")
print(u.getName())     // alice
```

对象的字段在构造函数赋值之前，是堆上原有的内容。请在构造函数里为每个字段赋值。

详见[工厂与 getter/setter 指南](factories-and-accessors.md)。

### 继承

```
class Dog extends Animal
```

单继承。子类在程序启动时复制父类的方法表。类按编译列表的顺序初始化，同一目录中按文件名字母序：父类文件名排在子类之前时，继承生效。

### 没有接口

方法按名字查找。名字对得上，就能调用。

### 方法找不到时

调用不存在的方法，结果是 `null`。这与对 `null` 调用方法是同一条规则。

SPEC：第 5 章

## 13. 方法值与 lambda

```
var f = (x) -> x * 2
print(f(21))           // 42
```

- lambda 的体是单个表达式。
- lambda 不捕获局部变量，只能使用参数、静态字段和全局变量。循环里的闭包共享同一个变量这类问题，在 nmsl 中不存在。
- 把方法取出来 `var g = u.getName` 时，得到的是一个不绑定 `u` 的函数。调用 `g()` 时，`this` 是当前方法的 `this`。
- 调用时不检查参数个数。

SPEC：4.6

## 14. 容器

```
var xs = List<int>()
xs = xs.add(1)
xs = xs.add("12abc")   // 存入 12
print(len(xs))         // 2
print(xs.get(1))       // 12

var m = Map<String, int>()
String key = "a"
m.put(key, 1)
print(m.get(key))      // 1
print(m.get(0))        // 1：0 == "a"，因为 "a" 转成数字是 0
```

- `add` 返回新的 List，请赋值回去：`xs = xs.add(x)`。这与 Go 的 `append` 相同。
- `Map` 用 `==` 比较键，字符串比较的是地址。存和取请使用同一个字符串值。两处分别写的字面量 `"a"` 是否是同一个地址，由目标平台定义。
- `List<List<int> >` 末尾的两个 `>` 之间要有空格。
- 内置容器不需要工厂，标准库是可信代码。

SPEC：8.5

## 15. 模块

```
import "lib/model"
```

`import` 把一个目录下的所有 `.nmsl` 文件加入编译。路径相对于你执行 `nmsl build` 的目录，所以请始终在项目根目录执行。所有类位于同一个命名空间。

详见[模块与导入](modules.md)。

## 16. 错误

错误是值。可能失败的方法多返回一个 `error`：

```
public static (number, error) parse(String s)
    var n, err = ParseInt(s)
    if (!nulls.Is(err))
        return 0, errors.Wrap(err, "parse")
    return n, null
```

没有错误时返回 `null`。判断错误只能写 `!nulls.Is(err)`；`if (err != null)` 永远不进入，因为它是一个碰到了 `null` 的运算。接收了 err 却不使用，是编译错误。

`panic(msg)` 打印消息后终止进程，无法恢复。`try`、`catch`、`throw`、`defer`、`recover` 是保留字。详见[错误处理](errors.md)。

SPEC：4.8

## 17. 并发

```
spawn(() -> Worker.run())
sleep(1000)
```

`spawn` 启动一个线程。所有 `spawn` 出来的线程共享一把全局锁，依次执行。`main` 返回时进程退出，请在 `main` 的末尾调用 `sleep`。

SPEC：7.3

## 18. 程序的结束

- `main` 正常结束，退出码为 0。
- `exit(n)` 以 `n` 的低 8 位退出：`exit(256)` 的退出码是 0。
- nmsl 没有运行时错误。错误是值，由方法返回；值不符合预期时，`null` 会安静地传下去；访问了不该访问的内存时，进程终止。

| 情形 | 你会看到 |
|---|---|
| 数组越界、写字符串字面量 | `Segmentation fault (core dumped)` |
| 整数除以 0 | `Floating point exception` |
| `panic(msg)` | `Aborted (core dumped)` |
| 内存耗尽 | `Killed` |

nmsl 程序不会崩溃，只会终止。

SPEC：第 7 章

---

恭喜，你已经了解了 nmsl 的全部。接下来请阅读[风格指南](style.md)。
