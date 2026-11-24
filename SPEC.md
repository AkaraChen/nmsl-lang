# nmsl 语言规范

版本：nmsl 1.0

---

## 序言

### 本规范的地位

nmsl 的规范就是 `main` 分支上编译器的行为。本文档是对该行为的如实记录。本文档与编译器不一致时，以编译器为准；此时不一致的部分称为「尚未写进文档的行为」，在下一次修订中补录。

nmsl 不存在实现错误。任何可观察的行为都是语言的一部分。修改编译器的可观察行为，等同于修改本规范。

### 实现结构

nmsl 的参考实现（也是唯一实现）由以下部分组成：

```
nmsl build Main.nmsl
  lexer   → token 流，含 INDENT / DEDENT / NEWLINE        (Go)
  parser  → AST，递归下降 + Pratt                          (Go)
  checker → 未使用变量、类名 = 文件名、make 位置、getter/setter、equals/hashCode (Go)
  codegen → out.c                                          (Go)
  exec    → $CC -O2 -w out.c runtime/nmsl.c -o Main        (系统 C 编译器)
```

- 编译器用 Go 编写，把 nmsl 源码翻译为 C 源码 `out.c`，再调用系统的 C 编译器生成可执行文件。
- 运行时是 `runtime/nmsl.c`，用 C 编写，负责值、字符串、数组、List、Map、隐式转换和输出。
- 本规范中多处行为来自宿主：字面量规则来自 Go 的 `strconv`，运行时行为来自 C 编译器、C 标准库和操作系统。规范直接引用这些宿主的行为，不重新定义。

### 用语

- 「必须」「不得」表示程序或实现的义务。违反「必须」的程序，按条文给出的结果处理；条文没有给出结果的，为编译错误。
- 「由目标平台定义」表示结果取决于所用的 C 编译器、优化级别、处理器架构和操作系统。nmsl 不对这类结果作进一步规定，开发者应当知道自己的目标平台。
- 「进程终止」表示进程被操作系统信号结束（见 7.5）。nmsl 程序不会出错，只会终止。
- 「归一化名」见 1.5。
- 本文档中的「cc」指环境变量 `$CC` 所指定的程序；`$CC` 未设置时为 `cc`。

### 文法记号

文法使用 EBNF：

```
产生式  = 名字 "=" 表达式 "." .
a | b       选择
[ a ]       可选（0 或 1 次）
{ a }       重复（0 或多次）
( a )       分组
"x"         终结符
NAME        词法记号（大写）
```

---

## 第 1 章 词法

### 1.1 源文件

1.1.1 源文件必须以 `.nmsl` 为扩展名，内容按字节读取。字符串字面量与注释中的字节按 UTF-8 原样保留。

1.1.2 字节序标记（BOM，U+FEFF）不被剥除。以 BOM 开头的文件在第一个字符处产生词法错误：

```
unexpected character U+FEFF at 1:1
```

1.1.3 行结束符只有 `\n`（U+000A）。`\r`（U+000D）在字符串字面量与注释之外的任何位置出现，都是词法错误 `unexpected character U+000D at L:C`。因此以 CRLF 保存的文件，在第一行行末产生错误。

1.1.4 除字符串字面量、字符字面量和注释外，源文件中出现的非 ASCII 字符都是词法错误 `unexpected character U+XXXX at L:C`。1.2.4 规定的行首 NBSP 情形除外。

### 1.2 行与缩进

1.2.1 每一行的缩进量，是从行首开始连续的空格（U+0020）与制表符（U+0009）的个数。空格和制表符各计 1 列，可以在同一行中混用。

1.2.2 只包含空白和注释的行是空行。空行不产生 INDENT、DEDENT 或 NEWLINE。

1.2.3 lexer 维护一个缩进栈，初值为 `[0]`。对每个非空行，设其缩进量为 n，栈顶为 t：

- n > t：压入 n，产生一个 INDENT。
- n = t：不产生缩进记号。
- n < t：反复弹出栈顶并为每次弹出产生一个 DEDENT，直到栈顶 ≤ n。此时若栈顶 < n，压入 n 并产生一个 INDENT。

最后一种情况称为「回退到未知层级」。它不是错误：结果是先关闭若干块，再开启一个新的匿名块（2.4.2）。

1.2.4 缩进量只统计空格和制表符。行首若出现 U+00A0（NBSP），缩进统计在该字符处停止；随后 lexer 用 `unicode.IsSpace` 跳过行内的全部空白（包括该 NBSP 及其后的空格）。因此：

```
        foo()      // 缩进 8
        foo()      // 缩进 0：该行被视为最外层，前面打开的块全部关闭
        foo()      // 缩进 4
```

NBSP 出现在其他位置时适用 1.1.4。

1.2.5 文件结束时，缩进栈中除 0 以外的每一层各产生一个 DEDENT。

### 1.3 自动分号

1.3.1 语句以 `;` 或行尾结束。`;` 可以写，也可以不写。

1.3.2 一行（去掉行尾注释后）的最后一个记号是以下之一时，lexer 在行尾插入一个 `;`：

- 标识符（含 `this`）
- 字面量（整数、浮点数、字符串、字符、`true`、`false`、`null`）
- `)` 或 `]`
- 关键字 `return` 或 `break`

其他情况下不插入，下一行被视为本行的延续。

1.3.3 后果：

- 续行时运算符必须留在上一行的行尾。运算符写在下一行开头时，上一行已经结束，下一行若缩进更深则成为一个匿名块（2.4.2）。
- `return` 后直接换行，等同于 `return;`。

```
int total = a
    + b          // total = a；+b 是匿名块中的一条表达式语句

return
    a + b        // 返回 null；a + b 位于匿名块中，不会执行
```

### 1.4 注释

1.4.1 注释只有一种：从 `//` 到行尾。没有块注释。

1.4.2 注释在 lexer 阶段被丢弃，不进入 AST。这一规定对格式化器（9.7）和文档生成器（9.8）有直接影响。

### 1.5 标识符与归一化

1.5.1 标识符由 ASCII 字母、数字和下划线组成，不得以数字开头：

```
IDENT = ( letter | "_" ) { letter | digit | "_" } .
letter = "A" … "Z" | "a" … "z" .
digit  = "0" … "9" .
```

1.5.2 标识符是风格不敏感的。lexer 读出标识符后立即把它替换为归一化名：

```
normalize(id) = id[0] + lower(remove("_", id[1:]))
```

即：第一个字符原样保留；其余字符删去下划线，并转为小写。

| 原文 | 归一化名 |
|---|---|
| `fooBar` | `foobar` |
| `foo_bar` | `foobar` |
| `foo__B_a_r` | `foobar` |
| `FOOBAR` | `Foobar` |
| `Foo_bar` | `Foobar` |
| `getName` | `getname` |
| `UserService` | `Userservice` |
| `User_service` | `Userservice` |
| `userService` | `userservice` |

`fooBar` 与 `foo_bar` 是同一个标识符；`fooBar` 与 `FooBar` 是不同的标识符。

1.5.3 归一化发生在 lexer 中，原文不再保留。编译器的所有后续阶段、错误信息（9.13）、格式化器、文档生成器和语言服务器，看到的都是归一化名。

1.5.4 以下场合不使用归一化名，而使用原文逐字节比较：

- 类名与文件名的比较（5.1）
- 工厂类名的比较（5.5）
- 入口方法的识别（5.2）

### 1.6 关键字与保留字

1.6.1 关键字：

| | | | | | |
|---|---|---|---|---|---|
| `class` | `extends` | `import` | `public` | `private` | `static` |
| `var` | `if` | `else` | `while` | `for` | `switch` |
| `case` | `default` | `break` | `return` | `this` | `typeof` |
| `true` | `false` | `null` | | | |

1.6.2 保留字：

| 保留字 | 状态 |
|---|---|
| `new` | reserved for future use |
| `try` | reserved for future use |
| `catch` | reserved for future use |
| `throw` | reserved for future use |
| `interface` | reserved for future use |
| `goto` | reserved for future use |
| `go` | reserved for future use |
| `defer` | reserved for future use |
| `recover` | reserved for future use |

保留字在文法中没有任何对应的产生式。在任何位置使用保留字（包括作为变量名、方法名、类名）都是编译错误：

```
reserved for future use: new
```

保留字对应的功能将在 nmsl 2 中提供（见 10.1）。nmsl 1 的错误以值的形式返回（4.8），不需要 `try`、`catch`、`throw`；`panic`（4.8.8）之后不需要恢复，因此不需要 `defer`、`recover`。

1.6.3 类型名（`number`、`String` 等，见 3.2）和内置函数名（`print`、`make` 等，见第 8 章）不是关键字。

### 1.7 运算符与分隔符

1.7.1 nmsl 的运算符与分隔符是以下记号的全集：

```
+    -    *    /    %    **
&    |    ^    ~    <<   >>
==   !=   <    <=   >    >=
&&   ||   !
=    ?    :    ->
(    )    [    ]    ,    .    ;
```

1.7.2 lexer 采用最长匹配。`>>` 永远是右移运算符，即使它出现在类型参数的末尾（见 5.10.3）。

1.7.3 `===` 和 `!==` 是词法错误：

```
unknown operator: ===
```

nmsl 只有一种相等运算符 `==`，其余比较方式由内置函数和方法提供（3.11）。

1.7.4 不在 1.7.1 中的字符组合（例如 `++`、`+=`）按最长匹配拆成多个 1.7.1 中的记号，再由 parser 处理。nmsl 没有自增、自减和复合赋值运算符。

### 1.8 字面量

#### 1.8.1 整数与浮点数

数字字面量的文本由数字、字母、下划线和 `.` 组成，以数字开头。lexer 把整段文本交给 Go 标准库处理：

1. 先调用 `strconv.ParseInt(s, 0, 64)`。成功则得到其值。
2. 失败则调用 `strconv.ParseFloat(s, 64)`。成功则得到其值。
3. 都失败，为词法错误。

字面量的类型只由文本中是否含有 `.` 决定：含有 `.` 的是 `bignumber`，否则是 `number`。

| 字面量 | 值 | 类型 | 说明 |
|---|---|---|---|
| `10` | 10 | number | |
| `010` | 8 | number | 前导 0 表示八进制 |
| `08` | 8 | number | 不是合法八进制，由 ParseFloat 按十进制读出 |
| `0b101` | 5 | number | |
| `0o7` | 7 | number | |
| `0x1F` | 31 | number | |
| `1_000` | 1000 | number | |
| `3.9` | 3.9 | bignumber | |
| `1e3` | 1000 | number | 不含 `.` |

`number` 字面量的值在生成的 C 代码中以 `long` 表示，超出 `long` 范围时由目标平台定义（3.4）。

#### 1.8.2 字符串

- 双引号字符串 `"..."` 按 Go 的转义规则解释（`strconv.Unquote`）：支持 `\a \b \f \n \r \t \v \\ \"`、八进制 `\101`、十六进制 `\x41`、`\u4e2d`、`\U0001F600`。
- 反引号字符串 `` `...` `` 是原始字符串，不处理转义，可以跨行。
- 字符串字面量的内容在运行时是以 `\0` 结尾的 C 字符串。内容中含有 `\x00`（或 `\000`）时，字符串在该处结束（3.5.2）。
- 内容为空的字符串字面量（`""` 与 ``` `` ```）编译为 `null`（3.8.4）。

#### 1.8.3 字符

单引号字面量 `'a'` 交由 `strconv.Unquote` 处理，结果是一个 Unicode 码点，类型为 `number`：

```
'a'          // 97
'a' + 1      // 98
'中'         // 20013
'\n'         // 10
'ab'         // 编译错误
```

#### 1.8.4 其他

`true` 和 `false` 是 `number` 类型的 1 和 0（3.6）。`null` 是空值（3.8）。

### 1.9 词法错误

词法错误报告字符所在的行号与列号，格式为 `unexpected character U+XXXX at L:C`。其余阶段的错误只报告行号（9.13）。

---

## 第 2 章 语法

### 2.1 总则

2.1.1 parser 采用递归下降处理语句，采用 Pratt 解析处理表达式。

2.1.2 本章中 `TERM` 表示语句结束符：显式的 `;` 或自动插入的 `;`（1.3）。

2.1.3 文法中出现 `[ TERM ]` 的位置，parser 跳过任意个连续的 TERM。

### 2.2 源文件

```
File        = { ImportDecl } ClassDecl .
ImportDecl  = "import" STRING TERM .
```

2.2.1 一个源文件必须恰好包含一个类声明。

2.2.2 nmsl 没有顶层函数、顶层变量和顶层语句。

### 2.3 类

```
ClassDecl   = "class" IDENT [ "extends" IDENT ] [ TERM ] INDENT { Member } DEDENT .
Member      = FieldDecl | MethodDecl | CtorDecl .
FieldDecl   = Type IDENT TERM
            | "static" Type IDENT [ "=" Expr ] TERM .
MethodDecl  = { Modifier } ResultType IDENT "(" [ Params ] ")" [ TERM ] Block .
ResultType  = Type | "(" Type "," Type { "," Type } ")" .
CtorDecl    = { Modifier } IDENT "(" ")" [ TERM ] Block .
Modifier    = "public" | "private" | "static" .
Params      = Param { "," Param } .
Param       = Type IDENT .
```

2.3.1 字段声明不得带 `public` 或 `private`。字段可以带 `static`；只有静态字段可以有初值表达式。

2.3.2 方法声明与构造函数声明可以带任意组合的修饰符。修饰符的作用见 5.3。

2.3.3 构造函数是名字与类名相同、没有返回类型、没有参数的方法。

2.3.4 **多返回值。** 返回类型可以是括号中的两个或更多类型，例如 `(number, error)`。多返回值的语义见 4.8。

2.3.5 类名后写类型参数（`class Box<T>`）是编译错误：

```
generics are coming in nmsl 2
```

### 2.4 块与语句

```
Block       = INDENT { Stmt } DEDENT .
Stmt        = Block
            | VarDecl TERM
            | IfStmt
            | WhileStmt
            | ForStmt
            | SwitchStmt
            | MultiAssign TERM
            | "return" [ Expr { "," Expr } ] TERM
            | "break" TERM
            | Expr TERM
            | TERM .
VarDecl     = Type IDENT [ "=" Expr ]
            | "var" IDENT "=" Expr
            | "var" IDENT "," IDENT { "," IDENT } "=" Expr .
MultiAssign = Target "," Target { "," Target } "=" Expr .
Target      = IDENT | Postfix "." IDENT | Postfix "[" Expr "]" .
IfStmt      = "if" "(" Expr ")" [ TERM ] Stmt [ [ TERM ] "else" [ TERM ] Stmt ] .
WhileStmt   = "while" "(" Expr ")" [ TERM ] Stmt .
ForStmt     = "for" "(" [ ForInit ] ";" [ Expr ] ";" [ Expr ] ")" [ TERM ] Stmt .
ForInit     = VarDecl | Expr .
SwitchStmt  = "switch" "(" Expr ")" [ TERM ] INDENT { CaseClause } DEDENT .
CaseClause  = ( "case" CaseLabel | "default" ) ":" [ TERM ] { Stmt } .
CaseLabel   = INT_LIT | CHAR_LIT | "-" INT_LIT .
```

2.4.1 **块由缩进决定。** 块以 INDENT 开始，以 DEDENT 结束。nmsl 没有 `{` `}` 块，条件后没有冒号。

2.4.2 **匿名块。** `Block` 本身是一种语句。因此，在任何位置出现的 INDENT 都开启一个匿名块。匿名块中的语句按顺序执行，与所在位置的其他语句没有任何关系。多余的缩进不是错误。

2.4.3 **条件必须加括号。** `if`、`while`、`for`、`switch` 后的括号是文法的一部分，不可省略。

2.4.4 **单语句体。** `if`、`else`、`while`、`for` 的体是一条 `Stmt`。条件之后若下一行没有更深的缩进，下一行的语句就是它的体：

```
if (debug)
deleteDatabase()      // if 的体：只有 debug 为真时执行
cleanup()             // 与 if 无关
```

2.4.5 `else` 与最近的、尚未匹配 `else` 的 `if` 结合。

2.4.6 **变量声明的识别。** 一条语句以两个连续的标识符开头（或以 `Type` 形式的 `IDENT "<" … ">" IDENT`、`IDENT "[" "]" IDENT` 开头）时，parser 将其判定为变量声明。判定只向前看 2 个记号。

2.4.7 **switch 穿透。** `switch` 生成 C 的 `switch`。一个 `case` 的语句执行完后，若没有 `break`，继续执行下一个 `case` 的语句。`case` 标签必须是整数字面量或字符字面量（可带负号）；控制表达式的值按 `to_number`（3.9）转为整数后参与匹配。

2.4.8 `break` 跳出最内层的 `while`、`for` 或 `switch`。

2.4.9 nmsl 没有 `do`-`while`、for-each、`continue`、`goto` 语句。遍历数组使用下标。

2.4.10 **多变量的识别。** 一条语句以 `IDENT ","` 开头时，parser 将其判定为多变量赋值 `MultiAssign`。判定同样只向前看 2 个记号。多变量声明只能用 `var`；`int n, error err = f()` 是语法错误。

### 2.5 表达式

```
Expr        = Lambda | Binary .
Lambda      = "(" [ IDENT { "," IDENT } ] ")" "->" Expr .
Binary      = Unary { BinaryOp Unary } .          // 按 2.5.1 的优先级
Unary       = { "!" | "~" | "-" | "+" | "typeof" } Postfix .
Postfix     = Primary { "(" [ Args ] ")" | "[" Expr "]" | "." IDENT } .
Args        = Expr { "," Expr } .
Primary     = IDENT | "this" | INT_LIT | FLOAT_LIT | STRING_LIT | CHAR_LIT
            | "true" | "false" | "null"
            | "(" Expr ")"
            | "[" [ Args ] "]"
            | GenericType "(" ")" .
```

#### 2.5.1 优先级表

优先级表照抄 C，另加 `**`、`typeof` 与 `->`。级别 1 最高。

| 级别 | 运算符 | 结合性 | 说明 |
|---|---|---|---|
| 1 | `()` `[]` `.` | 左 | 调用、下标、成员 |
| 2 | `!` `~` `-` `+` `typeof` | 右（前缀） | 一元 |
| 3 | `**` | 左 | 幂 |
| 4 | `*` `/` `%` | 左 | |
| 5 | `+` `-` | 左 | |
| 6 | `<<` `>>` | 左 | |
| 7 | `<` `<=` `>` `>=` | 左 | |
| 8 | `==` `!=` | 左 | |
| 9 | `&` | 左 | |
| 10 | `^` | 左 | |
| 11 | `\|` | 左 | |
| 12 | `&&` | 左 | |
| 13 | `\|\|` | 左 | |
| 14 | `? :` | **左** | 条件 |
| 15 | `=` | 右 | 赋值 |
| 16 | `->` | — | lambda |

2.5.2 后果（规范性）：

| 表达式 | 解析为 | 值 |
|---|---|---|
| `x & 1 == 0` | `x & (1 == 0)` | 恒为 0 |
| `a ? b : c ? d : e` | `(a ? b : c) ? d : e` | |
| `-2 ** 2` | `(-2) ** 2` | 4 |
| `2 ** 3 ** 2` | `(2 ** 3) ** 2` | 64 |
| `a = b = 1` | `a = (b = 1)` | 1 |

2.5.3 **赋值是表达式。** `=` 的左边必须是标识符、成员访问或下标表达式。赋值表达式的值是赋进去的值（经过 3.3 的转换之后）。`if (x = 0)` 合法，值为 0，条件为假。

2.5.4 **lambda。** lambda 的体只能是单个表达式。lambda 的参数没有类型注解。lambda 的语义见 4.6。

2.5.5 **内置泛型构造。** `List<T>()` 与 `Map<K, V>()` 创建内置容器（8.5）。这是 nmsl 中不经过工厂创建对象的唯一语法。标准库函数（如 `errors.New`，8.4.2）在运行时内部创建对象，不受 5.5 约束。

### 2.6 类型表达式

```
Type        = ( GenericType | IDENT ) { "[" "]" } .
GenericType = "List" "<" Type ">" | "Map" "<" Type "," Type ">" .
```

2.6.1 只有 `List` 和 `Map` 可以带类型参数。parser 写死了这两个名字（5.10）。

2.6.2 `List<List<int>>` 是语法错误，因为末尾的 `>>` 是一个右移记号（1.7.2）。必须写成 `List<List<int> >`。

### 2.7 一个完整的示例

```
// 文件名必须是 Main.nmsl
class Main
    public static void main(String[] args)
        var nums = [1, 2, 3]
        int total = 0
        for (int i = 0; i <= 3; i = i + 1)   // i = 3 时读取数组之后的内存
            total = total + nums[i]
        if (total & 1 == 0)                  // total & (1 == 0)
            print('a' + 1)                   // 98
        return
            total                            // 位于匿名块中，不执行
```

---

## 第 3 章 类型

### 3.1 值的表示

3.1.1 运行时的每一个值都是同一个 C 结构体：

```c
typedef struct {
    uint8_t tag;          // 0=NULL 1=NUMBER 2=BIGNUMBER 3=STRING 4=ARRAY 5=OBJECT 6=FUNC
    union { long n; long double b; char *s; void *p; } as;
} Value;
```

3.1.2 标签：

| tag | 名称 | payload |
|---|---|---|
| 0 | NULL | 全零 |
| 1 | NUMBER | `long n` |
| 2 | BIGNUMBER | `long double b` |
| 3 | STRING | `char *s` |
| 4 | ARRAY | `void *p` |
| 5 | OBJECT | `void *p`（对象头含类指针） |
| 6 | FUNC | `void *p`（C 函数指针） |

3.1.3 全零的 `Value` 就是 `null`。

3.1.4 结构体在 `tag` 与 `as` 之间有填充字节，`long double` 在 `as` 内部也可能有填充字节。这些字节的内容不作规定（由目标平台定义）。它们影响 `is`（3.11.3）与 bignumber 的真值（3.12）。

3.1.5 nmsl 没有布尔标签。见 3.6。

### 3.2 类型名

| 类型名 | 含义 |
|---|---|
| `number` | NUMBER |
| `int`、`long`、`short` | `number` 的别名 |
| `bignumber` | BIGNUMBER |
| `float`、`double` | `bignumber` 的别名 |
| `String` | STRING |
| `boolean` | 见 3.6 |
| `void` | 只用于方法返回类型 |
| 任意类名 | OBJECT |
| `T[]` | ARRAY |
| `List<T>`、`Map<K, V>` | 内置容器（OBJECT） |
| `error` | 内置类 `error`（OBJECT），见 4.8 |

3.2.1 `short` 与 `number` 完全相同。在 64 位 Linux 上 `short` 是 64 位的。

3.2.2 使用不存在的类型名是编译错误：`unknown type Banana`。

### 3.3 类型注解即转换

3.3.1 类型注解不做检查，只做转换。每当一个值被放进带类型注解的位置，编译器在该处插入一次 `coerce(T, v)`。这些位置是：

- 带类型的变量声明的初值
- 对带类型变量的赋值
- 对带类型字段的赋值
- 方法参数（在被调方入口处）
- 带返回类型的方法的 `return` 值
- `List<T>` 与 `Map<K, V>` 的元素写入（8.5）

`var` 声明的变量没有类型，不插入转换，之后可以赋任何值。

3.3.2 `coerce` 是一种运算，遵守 null 传染（3.8.2）：`coerce(T, null)` 的结果是 `null`。因此任何类型都可能持有 `null`。

3.3.3 转换规则：

| 目标类型 | 源值 | 结果 |
|---|---|---|
| `number` | NUMBER | 原值 |
| | BIGNUMBER | 向零截断（C 转换）：`3.9` → 3 |
| | STRING | `atol(s)`：`"12abc"` → 12，`"abc"` → 0 |
| | ARRAY / OBJECT / FUNC | 指针的整数值 |
| `bignumber` | BIGNUMBER | 原值 |
| | 其他 | `(long double)to_number(v)`：`"3.9"` → 3.0 |
| `String` | 任意 | `to_string(v)`（3.9.2）；结果为空串时为 `null` |
| `boolean` | 任意 | 真值（3.12）为真时 1，否则 0 |
| 类名 | 任意 | 只把 tag 改为 OBJECT，payload 不变 |
| `T[]` | 任意 | 只把 tag 改为 ARRAY，payload 不变 |

3.3.4 示例：

```
int x = "12abc"        // 12
int y = 3.9            // 3
String s = 42          // "42"
String t = [1, 2]      // 形如 "0x7ffd5e8c1a20" 的地址
boolean b = "false"    // 1（true）
boolean z = 0.0        // 由目标平台定义，通常为 0（见 3.12）
var v = 1              // 不转换
v = "x"                // 合法
User u = 1             // tag 变为 OBJECT，payload 仍是 1；访问 u 的字段时进程终止
```

nmsl 的类型从不拒绝你，它只会让你的值适应它。

### 3.4 数字

3.4.1 `number` 是 C 的 `long`。所有不含 `.` 的数字字面量和字符字面量都是 `number`。

| 平台 | `number` 宽度 |
|---|---|
| Linux / macOS，x86-64 与 ARM64（LP64） | 64 位 |
| Windows x64（LLP64） | 32 位 |

3.4.2 `bignumber` 是 C 的 `long double`。它是浮点数，不是任意精度数。所有含 `.` 的数字字面量都是 `bignumber`。

| 平台 | `bignumber` 格式 |
|---|---|
| x86 / x86-64 Linux | 80 位扩展精度 |
| MSVC | 64 位，与 `double` 相同 |
| ARM64 Linux | 128 位四精度 |

同一个算式在不同平台上精度不同。`0.1 + 0.2 == 0.3` 的值由目标平台定义。

3.4.3 你选择编译目标的那一刻，就决定了你的数字有多大。换一个 `$CC`，`number` 的宽度随之改变（9.1.4）。

3.4.4 算术运算直接生成有符号 C 运算。`number` 溢出由目标平台定义；`-O0` 与 `-O2` 的结果可以不同。

3.4.5 `number` 与 `bignumber` 混合运算时，按 C 的寻常算术转换，结果是 `bignumber`。

3.4.6 两个 `number` 相除是整数除法，向零截断：`7 / 2` 为 3。`number` 除以 0 或对 0 取余，进程收到 SIGFPE 并终止（7.5）。`bignumber` 除以 0 的结果按 C 的规则（无穷大或 NaN）。

3.4.7 `%`、`<<`、`>>`、`&`、`|`、`^`、`~` 先对操作数做 `to_number`，在 `long` 上计算。

3.4.8 `**` 用 C 的 `powl` 计算，结果是 `bignumber`。

### 3.5 字符串

3.5.1 字符串就是以 `\0` 结尾的 `char *`。

3.5.2 字符串的长度是 `strlen` 的结果，即第一个 `\0` 之前的字节数：

```
len("ab\x00cd")     // 2
len("中文")          // 6
```

3.5.3 **下标从 1 开始**，结果是该位置的一个字节，类型为 `number`：

```
"abc"[1]            // 97
"中文"[1]            // 228，UTF-8 编码的第一个字节
```

下标生成为 C 的 `s[i - 1]`，不做越界检查。`s[0]` 读取字符串之前的一个字节，其值由目标平台定义。

3.5.4 字符串字面量位于只读段；运行时拼接、转换产生的字符串位于堆上。对字符串下标赋值（`s[1] = 'x'`）把右侧值的 `to_number` 低 8 位写入该字节：`s` 是堆上的字符串时写入成功；`s` 是字面量时进程终止（SIGSEGV）。

3.5.5 空字符串不存在。任何结果为空的字符串操作都返回 `null`（3.8.4）。

### 3.6 布尔

3.6.1 `boolean` 不是独立的标签。`true` 是 NUMBER 1，`false` 是 NUMBER 0。

3.6.2 比较运算（`==`、`!=`、`<`、`<=`、`>`、`>=`）与 `!`、`nulls.Is` 的结果是 1、0 或 `null`。

3.6.3 `print(true)` 输出 `1`。`typeof true` 为 `"number"`。

### 3.7 数组

3.7.1 数组字面量 `[a, b, c]` 在堆上创建一个数组。元素是 `Value`，不做类型转换。

3.7.2 **数组下标从 0 开始**。下标生成为 C 的指针运算，不做越界检查：越界读取得到所在内存中的内容，越界写入改写所在内存；访问未映射的地址时进程终止。

3.7.3 `len(a)` 返回数组的元素个数。

3.7.4 数组按引用传递。数组之间的 `==` 比较的是地址。

3.7.5 字符串下标从 1 开始，数组下标从 0 开始。规范如实记录，不解释。

### 3.8 null

nmsl 只有一个空值 `null`。它同时具有以下三种性质。

#### 3.8.1 SQL 的 NULL：传染

任何运算只要有一个操作数是 `null`，结果就是 `null`。「运算」包括：

- 所有算术、位运算和比较运算，包括 `==` 与 `!=`
- 一元运算 `!`、`-`、`~`、`typeof`
- `&&` 与 `||`
- 内置函数 `is`、`len`
- `coerce`（3.3.2）

运行时中每个运算函数的第一行都是：

```c
if (a.tag == T_NULL || b.tag == T_NULL) return NUL;
```

`null` 是假值。

#### 3.8.2 Objective-C 的 nil：静默

- 对 `null` 调用方法，结果是 `null`，参数照常求值。
- 读取 `null` 的字段，结果是 `null`。
- 给 `null` 的字段赋值，静默忽略。
- 对 `null` 不会发生进程终止，也不会产生任何报告。

```
user.getAddress().getCity().getName()
// 任何一环为 null，整个表达式为 null
```

#### 3.8.3 例外：`+` 与字符串

`+` 的实现先判断字符串，再判断 null：只要有一个操作数是 STRING，两边都做 `to_string` 后拼接；`to_string(null)` 是空串。

```
null + "a"       // "a"
"a" + null       // "a"
null + 1         // null
null + null      // null
```

#### 3.8.4 Oracle 的 `'' IS NULL`：空串就是 null

- 字面量 `""` 编译为 `null`。
- 任何字符串操作的结果若为空，返回 `null`。
- 因此 `len("")` 是 `null`，`"" == ""` 是 `null`。

#### 3.8.5 不传染的操作

以下操作不是运算，不传染：

- `print`：输出参数的 `to_string`，`print(null)` 输出一个空行，与输出空串完全相同。
- `nulls.Is(x)`：返回真正的 1 或 0。它是整个运行时里唯一不被 null 传染的判断。
- `nulls.Coalesce(a, b, ...)`：返回第一个不是 `null` 的参数。
- 赋值、参数传递（`coerce` 之外的部分）、数组与容器的存取。

#### 3.8.6 判空

判断一个值是否为 `null`，唯一的方法是 `nulls.Is(x)`。

| 写法 | 结果 |
|---|---|
| `x == null` | `null`，无论 x 是什么 |
| `x != null` | `null`，无论 x 是什么 |
| `!(x == null)` | `null` |
| `null == null` | `null` |
| `typeof null` | `null` |
| `nulls.Is(x)` | x 为 null 时 1，否则 0 |

因此：

```
if (x == null)          // 永远不进入
    ...
if (x != null)          // 永远不进入；x 为 null 时走 else，这一半看起来是对的
    ...
else
    ...
```

Go 的惯用写法 `if (err != null)` 同样永远不进入，见 4.8.5。

#### 3.8.7 null 与「未初始化」不同

nmsl 只有一种空值。未初始化的变量不是空值，它是未定义的。三种未初始化的结果如下：

| 位置 | 未初始化时的值 | 原因 |
|---|---|---|
| 局部变量 | 栈上的内容 | C 的自动变量不清零 |
| 对象字段 | 堆上的内容 | 对象用 `malloc` 分配，不清零 |
| 静态字段 | `null` | C 的全局变量清零，全零的 Value 就是 null |

三种未初始化，两种结果。规范如实记录，不解释。

### 3.9 隐式转换

运行时只有两个转换函数。

#### 3.9.1 `to_number`

```c
long to_number(Value v)
```

| 源 | 结果 |
|---|---|
| NULL | 0 |
| NUMBER | 原值 |
| BIGNUMBER | 向零截断 |
| STRING | `atol(s)`：跳过前导空白，读取可选符号与十进制数字，遇到其他字符停止；一位数字都没有时为 0 |
| ARRAY / OBJECT / FUNC | 指针的整数值 |

```
"12abc" * 1      // 12
"abc" * 1        // 0
" 42" * 1        // 42
"3.9" * 1        // 3
"0x10" * 1       // 0
[1, 2] * 1       // 一个很大的数，由目标平台定义
```

`to_number` 在运算中出现时，传染判断先于它执行，因此 `null * 1` 是 `null` 而不是 0。

#### 3.9.2 `to_string`

```c
char *to_string(Value v)
```

| 源 | 结果 |
|---|---|
| NULL | 空串 |
| NUMBER | `sprintf("%ld")` |
| BIGNUMBER | `sprintf("%Lg")` |
| STRING | 原字符串 |
| ARRAY / OBJECT / FUNC | `sprintf("%p")` |

### 3.10 运算

3.10.1 每个运算编译为对运行时函数的调用，例如 `a + b` 编译为 `nmsl_add(a, b)`。

3.10.2 `+`：

1. 任一操作数是 STRING：两边 `to_string` 后拼接。结果为空时是 `null`。
2. 否则，任一操作数是 `null`：结果为 `null`。
3. 否则按 3.10.3 的数值规则相加。

```
"12abc" + 0      // "12abc0"
1 + "1"          // "11"
```

3.10.3 其余算术运算（`-` `*` `/` `%` `**` 与一元 `-` `+`）：

1. 任一操作数是 `null`：结果为 `null`。
2. 不是 NUMBER 或 BIGNUMBER 的操作数先做 `to_number`。
3. 有一边是 BIGNUMBER：在 `long double` 上计算，结果为 BIGNUMBER；否则在 `long` 上计算，结果为 NUMBER。

3.10.4 关系运算 `<` `<=` `>` `>=` 与 `==` 使用同一套比较规则（3.11.1）：tag 相同时直接比较 payload，tag 不同时两边 `to_number` 后比较。字符串之间的 `<` 比较的是地址。

3.10.5 `!x`：x 为 null 时 `null`；x 为真值时 0，否则 1。

3.10.6 `&&` 与 `||`：见 4.4。

### 3.11 相等

nmsl 提供三种比较。

#### 3.11.1 `a == b`

1. 任一操作数是 `null`：结果为 `null`。
2. tag 相同：直接比较 payload。NUMBER 与 BIGNUMBER 比较数值；STRING、ARRAY、OBJECT、FUNC 比较指针。
3. tag 不同：两边都做 `to_number`，比较两个 `long`。

后果：

- `"a" == "a"`：两个内容相同的字面量是否被合并到同一地址，由 cc 和优化级别决定。结果由目标平台定义。
- 运行时产生的字符串永远不等于字面量：`("a" + "b") == "ab"` 为 0。
- `0 == "abc"` 为 1（`atol("abc")` 是 0）。
- `1 == 1.9` 为 1（BIGNUMBER 被截断为 1）。
- `a != b` 等价于对 `a == b` 的结果取反；`null` 照常传染。

#### 3.11.2 `a.equals(b)`

1. 接收者是 `null`：结果为 `null`（3.8.2）。参数是 `null`：结果为 `null`（3.8.1）。
2. 两边都是 STRING：比较内容（`strcmp(a, b) == 0`）。
3. 接收者是对象，且其类定义了 `equals`：调用该方法。
4. 其余情况：与 `a == b` 相同。对象之间默认比较引用。

想正确比较两个字符串的内容，只能使用 `equals`。

#### 3.11.3 `is(a, b)`

1. 任一操作数是 `null`：结果为 `null`。
2. 否则：`memcmp(&a, &b, sizeof(Value)) == 0`。

结构体的填充字节参与比较（3.1.4）。因此同一个值与自身比较，`is(x, x)` 的结果也由目标平台定义。

#### 3.11.4 没有 `===`

见 1.7.3。

### 3.12 真值

3.12.1 真值规则只有一条：`Value` 的 payload 字节中只要有一个不是 0，就为真。

| 值 | 真值 | 原因 |
|---|---|---|
| `0` | 假 | 全零 |
| `null` | 假 | 全零 |
| `""` | 假 | 它就是 null |
| `"0"`、`" "`、`"false"` | 真 | 指针不为零 |
| `[]` | 真 | 指针不为零 |
| `0.0` | 由目标平台定义，通常为假 | `long double` 的填充字节未清零时为真 |
| `-0.0` | 真 | 符号位为 1 |

3.12.2 条件位置（`if`、`while`、`for`、`?:`、`assert`）的值按本节规则判断。`null` 在条件位置为假。

### 3.13 `typeof`

`typeof x` 返回表示 x 的 tag 的字符串：

| tag | 结果 |
|---|---|
| NUMBER | `"number"` |
| BIGNUMBER | `"bignumber"` |
| STRING | `"string"` |
| ARRAY | `"array"` |
| OBJECT | `"object"` |
| FUNC | `"function"` |
| NULL | `null`（3.8.1） |

`print(typeof null)` 输出一个空行。

---

## 第 4 章 语义

### 4.1 执行模型

4.1.1 每个方法编译为一个 C 函数：

```c
Value Class_method(Value this, Value a, Value b, ...);
```

`Class` 与 `method` 是归一化名。第一个参数总是 `this`。

4.1.2 表达式中的运算编译为对运行时函数的调用（3.10.1）。

### 4.2 作用域

4.2.1 **一个方法只有一个作用域。** codegen 把方法中所有的局部变量声明收集起来，统一放到 C 函数开头；声明在原位置只留下赋值。

4.2.2 后果：

- 在块中声明的变量，在块外可以使用。
- 在声明之前使用变量是合法的，其值是栈上的内容（3.8.7）。
- 没有遮蔽：内层块再次声明同名变量，就是同一个变量。由于标识符风格不敏感（1.5），内层的 `user_name` 与外层的 `userName` 是同一个变量。

4.2.3 同一个方法中，同名变量声明了多次且类型不同时，以源码中**最后一次**声明的类型为准，作用于整个方法：

```
int x = "5"        // x 是 "5"，不是 5
...
String x = 1       // 本声明的类型 String 作用于整个方法
```

4.2.4 参数与局部变量位于同一张表中。

### 4.3 未使用变量

4.3.1 声明之后，编译器在整个方法体中统计该变量归一化名出现的次数（含声明本身）。只出现一次时，为编译错误：

```
declared and not used: username
```

错误信息中的名字是归一化名。

4.3.2 统计的是出现次数，不区分读写。`int x = 1; x = 2` 合法。

4.3.3 参数也参与统计。未使用的参数是编译错误。需要保留不使用的参数时，写 `_ = param`。

4.3.4 方法调用的返回值可以丢弃。

4.3.5 入口方法（5.2）的参数 `args` 不参与统计。入口方法由整行字符串比较识别，codegen 为它直接写出固定的 C 函数头 `int main(int argc, char **argv)`，再由运行时把 `argv` 转换为 `args`；`args` 因此不经过变量表。入口方法体中声明的局部变量照常统计。

4.3.6 多变量声明 `var a, b = f()` 中的每个变量分别统计。`var n, err = f()` 之后不再出现 `err`，是编译错误：

```
declared and not used: err
```

这条规则保证每一个返回的错误都被写进代码（4.8.6）。

### 4.4 隐式全局变量

4.4.1 对一个既不是局部变量、也不是参数、也不是字段的名字赋值，codegen 在 C 文件顶部生成一个同名的全局 `Value`，初值为 `null`。

4.4.2 后果：

- 拼错的变量名不报错，而是成为一个新的全局变量。
- 未使用检查只针对局部变量，全局变量不参与。
- `_ = param` 创建（或写入）全局变量 `_`。

4.4.3 读取一个从未声明、也从未被赋值的名字，是编译错误：`undefined: name`。

### 4.5 求值顺序

4.5.1 函数参数的求值顺序未作规定，由 cc 决定。

4.5.2 二元运算编译为函数调用（3.10.1），因此两个操作数的求值顺序同样由 cc 决定。`f() + g()` 中 f 与 g 谁先执行由目标平台定义。

4.5.3 **`&&` 与 `||` 不短路。** 它们编译为 `nmsl_and(a, b)` 与 `nmsl_or(a, b)`：两个操作数都先求值，再传入。

```
if (risky() || fallback())    // risky 与 fallback 都会执行
```

4.5.4 `&&` 与 `||` 返回操作数而不是布尔值，并遵守 null 传染：

| 表达式 | 规则 |
|---|---|
| `a && b` | 任一为 null：null；a 为假：a；否则 b |
| `a \|\| b` | 任一为 null：null；a 为真：a；否则 b |

```
0 || "默认值"        // "默认值"
null || "默认值"     // null
```

为 null 值提供默认值，使用 `nulls.Coalesce`。

4.5.5 **条件运算符 `?:` 短路。** 它直接编译为 C 的 `?:`，只求值被选中的分支。需要短路时，使用 `?:`。

### 4.6 this、方法值与 lambda

4.6.1 `obj.m()` 在运行时按归一化名在 `obj` 的类表中线性查找（`strcmp`）方法 `m`，以 `obj` 为 `this` 调用。查找不到时，结果为 `null`。

4.6.2 `ClassName.m()` 调用静态方法，`this` 为 `null`。

4.6.3 **方法值不绑定接收者。** `var f = obj.getName` 得到一个 FUNC 值，即 C 函数指针，不记录 `obj`。

4.6.4 调用 FUNC 值 `f(args)` 时，codegen 把**当前所在方法**的 `this` 作为第一个参数传入。在静态方法中，当前的 `this` 是 `null`；此时被调方法读取的所有字段都是 `null`，对字段的所有赋值都被静默忽略（3.8.2）。

4.6.5 **参数个数不参与任何检查。** 无论是按名字调用方法，还是调用 FUNC 值，实参个数都不与形参个数比较。FUNC 值统一转换为可变参数函数指针调用。少传的参数取寄存器或栈中的内容；多传的参数在主流 ABI 上被忽略。确切结果由目标平台定义。

4.6.6 **lambda** `(x) -> expr` 编译为一个普通 C 函数，值为 FUNC。lambda 的体只能引用：

- 自己的参数
- 静态字段与静态方法（`ClassName.name`）
- 全局变量

lambda 不捕获外部的局部变量。体中出现的外部局部变量名，按全局名解析（4.4）。

4.6.7 由于 lambda 不捕获变量，「循环中的闭包都拿到同一个 i」这类问题在 nmsl 中不会发生。

### 4.7 方法的返回

4.7.1 `return expr` 把 expr 按返回类型转换后返回。`return` 不带表达式，或执行到方法末尾，返回 `null`。

4.7.2 `void` 方法的调用结果是 `null`。

4.7.3 多返回值方法的 `return` 见 4.8.2。

### 4.8 错误

nmsl 的错误是值。可能失败的方法多返回一个 `error`，调用方自己判断。

4.8.1 **error 类型。** `error` 是内置类，有两个字段：`message`（String）与 `cause`（error 或 `null`）。error 只能由标准库函数创建（8.4.2）。没有错误用 `null` 表示，与 3.8 的 `null` 是同一个值。

4.8.2 **多返回值的实现。** `return a, b` 把各个表达式按对应位置的返回类型转换（3.3）后，按 `return` 中**实际写出的表达式个数**打包成一个堆上的数组，作为一个 Value 返回。接收方按下标取出各个元素。

```
(number, error) parseAge(String s)
    var n, err = ParseInt(s)
    if (!nulls.Is(err))
        return 0, errors.Wrap(err, "parseAge")
    return n, null
```

4.8.3 **接收。** `var a, b = f()` 与 `a, b = f()` 依次取数组的第 0、1……个元素。取值不做越界检查（3.7.2），由此：

| 情形 | 结果 |
|---|---|
| `var r = f()`，只写一个变量 | 不是错误。r 得到整个数组（ARRAY） |
| 接收的变量比数组元素多 | 多出的变量取到数组之后的内存，由目标平台定义 |
| `return 5` 写在 `(number, error)` 方法中 | 数组只有 1 个元素，接收方的 `err` 取到数组之后的内存 |
| 方法执行到末尾没有 `return`，或 `return` 不带表达式 | 方法返回 `null`；对 `null` 取下标得到 `null`（3.8.1），所以每个接收变量都是 `null`，即「成功，值为 null」 |
| 接收的不是数组（例如普通方法的返回值） | 按下标规则取值：数字、对象等非数组值取下标，结果由目标平台定义 |

4.8.4 **类型转换。** 每个返回值按声明的返回类型转换（3.3）。`return "12abc", null` 在 `(number, error)` 方法中返回 12 与 `null`。

4.8.5 **判断有没有错误。** `err != null` 是运算，有一边是 `null` 时结果是 `null`（3.8.6）：

| 写法 | err 为 `null`（成功） | err 为 error（失败） |
|---|---|---|
| `if (err != null)` | 不进入 | **不进入** |
| `if (err == null)` | 不进入 | 不进入 |
| `if (!nulls.Is(err))` | 不进入 | 进入 |

官方惯用法是：

```
var v, err = f()
if (!nulls.Is(err))
    return null, err
```

`if (err != null)` 能通过编译，也满足 4.3 的未使用检查，但永远不进入。

4.8.6 **丢弃。**

- 把调用写成语句 `f()`，丢弃全部返回值，不是错误（4.3.4）。
- `var v, err = f()` 之后不再使用 `err`，是编译错误（4.3.6）。
- 写一次 `if (err != null)` 即满足 4.3.6。
- `var v, _ = f()` 合法。`_` 按 4.4 成为一个全局变量。

4.8.7 **哨兵错误。** 哨兵错误写作静态字段：

```
class Store
    static error ErrNotFound = errors.New("not found")
    public static error getErrNotFound()
        return ErrNotFound
    public static void setErrNotFound(error e)
        ErrNotFound = e
```

哨兵错误是静态字段，按 5.4.2 必须有 getter 与 setter。任何代码都可以调用 `setErrNotFound` 替换它。

静态字段按初始化顺序（6.7）取得初值。初始化之前读到的静态字段是 `null`（3.8.7）。因此，当返回 `Store.ErrNotFound` 的代码先于 `Store` 初始化执行时，返回的是 `null`，调用方得到的是「没有错误」。

4.8.8 **panic。** 内置函数 `panic(msg)` 先 `print(msg)`，再调用 C 的 `abort()`，进程收到 SIGABRT 终止（7.5）。msg 经由 stdout 输出，不刷新缓冲区（7.6.2）：输出到终端时可见，输出到管道或文件时丢失。panic 之后无法恢复；`defer` 与 `recover` 是保留字（1.6.2）。

---

## 第 5 章 抽象

### 5.1 文件与类

5.1.1 一个文件只能有一个类。类名必须与文件名（去掉 `.nmsl`）**逐字节相同**。比较使用类名原文与 `filepath.Base` 的结果。

5.1.2 标识符风格不敏感，文件名却区分大小写与下划线。在大小写不敏感的文件系统上（macOS 默认），同一仓库的检查结果可以随机器不同而不同。

### 5.2 入口

5.2.1 入口方法通过整行字符串比较识别：一行去掉首尾空白后，必须**一字不差**地等于

```
public static void main(String[] args)
```

5.2.2 多一个空格、参数名写成 `argv`、修饰符换序，这一行都不是入口。编译器不报告，最终由链接器报告：

```
undefined reference to `main'
```

5.2.3 入口方法的参数不参与未使用变量检查（4.3.5）。参数名写成 `argv` 的方法不是入口，它的参数照常参与检查。

5.2.4 程序中有多个入口时，由链接器报告 `multiple definition of `main'`。

### 5.3 修饰符与可见性

5.3.1 修饰符只有三个：`public`、`private`、`static`。不写可见性修饰符时视为 `public`。

5.3.2 **可见性不执行。** parser 把修饰符记入 AST，之后没有任何阶段读取它。运行时按名字查找方法与字段，不看修饰符。`private` 方法可以从任何地方调用。

我们都是成年人。

5.3.3 `static` 方法没有接收者，其中的 `this` 是 `null`（4.6.2）。

### 5.4 字段与 getter/setter

5.4.1 字段声明不能带可见性修饰符，一律视为私有。按 5.3.2，私有同样不执行：`u.name` 可以从任何地方直接读写。

5.4.2 每个字段 `x`（含静态字段）必须在同一个类中有方法 `getX` 与 `setX`。`boolean` 类型的字段必须用 `isX` 代替 `getX`；为 `boolean` 字段写 `getX` 是编译错误。比较使用归一化名。

5.4.3 checker 只检查方法名是否存在，不检查参数、返回类型与方法体。`getName()` 可以返回任何东西，`setName()` 可以什么都不做。只读字段同样必须有 setter。

5.4.4 getter 与 setter 是强制的；直接访问字段也是允许的。

### 5.5 对象创建：make 与工厂

5.5.1 nmsl 没有 `new` 表达式。`new` 是保留字（1.6.2），文法中没有对应的产生式。

5.5.2 创建对象的唯一方式是内置函数 `make(X)`：在堆上用 `malloc` 分配对象（不清零），写入类指针，调用 X 的构造函数（如有），返回实例。

5.5.3 `make(X)` 只能出现在名为 `XFactory` 的类中。在其他类中调用 `make(X)` 是编译错误。

5.5.4 惯用写法是每个类配一个工厂类，内含一个静态方法：

```
// UserFactory.nmsl
class UserFactory
    public static User create()
        return make(User)
```

5.5.5 工厂类名的比较使用**原文**，`make` 参数的比较使用**归一化名**。因此在类 `User_factory` 中不能调用 `make(User)`；但在运行时，`User_factory.create()` 与 `UserFactory.create()` 指向同一个类。

5.5.6 构造函数中没有赋值的字段，值是堆上的内容（3.8.7）。

5.5.7 内置容器 `List<T>()` 与 `Map<K, V>()` 不受本节约束（8.5）。标准库是可信代码。

### 5.6 继承

5.6.1 nmsl 支持单继承 `class Dog extends Animal`。

5.6.2 实现方式：程序启动时，每个类在初始化（6.6）时把父类类表中的方法与字段条目复制一份到自己的类表中，排在自身条目之后。查找从自身条目开始，因此子类定义的同名方法优先。

5.6.3 复制发生在子类初始化的时刻。如果子类先于父类初始化，复制时父类的类表还是空的，子类得不到任何继承的条目，且不会有任何报告。由于同一目录中的文件按文件名字母序初始化（6.6.1）：

```
Dog extends Animal       // Animal.nmsl 先于 Dog.nmsl：继承生效
Apple extends Fruit      // Apple.nmsl 先于 Fruit.nmsl：继承不生效
```

5.6.4 nmsl 没有 `super`。

### 5.7 成员查找

5.7.1 方法与字段都按归一化名在类表中线性查找。

5.7.2 方法找不到时，调用结果为 `null`；字段找不到时，读取结果为 `null`，写入被忽略。这与对 `null` 调用方法是同一条代码路径（3.8.2）。继承未生效、方法名拼错，表现都是「调用了，返回了 null」。

### 5.8 接口

5.8.1 nmsl 没有接口。`interface` 是保留字。

5.8.2 方法按名字查找，名字对上就能调用。任何对象都「实现」了任何接口，只要方法名碰巧相同。由于风格不敏感，`get_name` 与 `getName` 也算相同。

### 5.9 equals 与 hashCode

5.9.1 一个类定义了 `equals` 就必须定义 `hashCode`，否则为编译错误。

5.9.2 checker 只检查两个名字是否同时存在，不检查二者的一致性。

5.9.3 标准库的 `Map` 不调用 `equals` 与 `hashCode`（8.5.4）。

### 5.10 泛型

5.10.1 只有内置的 `List<T>` 与 `Map<K, V>` 可以写类型参数。

5.10.2 用户类写类型参数是编译错误 `generics are coming in nmsl 2`。

5.10.3 类型参数的作用与类型注解相同：写入元素时按 T 做 `coerce`（3.3）。`List<int>` 中 `add` 一个 `"12abc"`，存入的是 12。

5.10.4 嵌套类型参数的末尾必须用空格分开：`List<List<int> >`（2.6.2）。

---

## 第 6 章 模块

### 6.1 总则

nmsl 没有包，只有目录。`import` 把一个目录中的文件加入编译列表，与 C 的 `#include` 一样是文本层面的操作。所有类位于同一个全局命名空间。

### 6.2 import

6.2.1

```
import "lib/model"     // 把 lib/model/ 下的所有 .nmsl 文件加入编译列表
```

6.2.2 导入路径相对于**运行编译器时的当前目录**，不相对于当前文件。编译器直接调用 `os.ReadDir(path)`。在其他目录执行 `nmsl build` 时，导入失败：

```
error: open lib/model: no such file or directory
```

6.2.3 导入不递归进入子目录。

6.2.4 标准库（第 8 章）不需要导入。

### 6.3 编译列表

6.3.1 编译列表的构造：

1. 命令行上给出的源文件，按给出的顺序加入编译列表。
2. 每加入一个文件，就按顺序处理它的每条 `import`：用 `os.ReadDir` 读出目录中的 `.nmsl` 文件（按文件名字母序），逐个加入编译列表，并在加入后立即递归处理该文件的 `import`。

6.3.2 编译列表不去重。

### 6.4 命名空间

6.4.1 所有类都是全局的。类编译为以类名为前缀的 C 函数，不做名字修饰。

6.4.2 两个目录中有同名的类，或者类名归一化后相同（`UserService` 与 `User_service`），codegen 生成同名的 C 函数，由 cc 报告：

```
multiple definition of `User_getname'
```

报告中的名字是生成的 C 函数名。

6.4.3 用户的类与标准库的类同名时，结果同上。用户不能定义名为 `List` 的类。

### 6.5 导入的作用范围

6.5.1 导入对整个程序生效：任何一个文件导入了某个目录，所有文件都能使用其中的类。

6.5.2 **未使用的导入是编译错误。** 一个文件中没有出现被导入目录中任何一个类名时，报告：

```
imported and not used: "lib/model"
```

6.5.3 后果：文件 C 使用某目录的类而不导入，依赖文件 A 的导入；A 自己若不使用该目录中的类，则 A 的导入是编译错误。此时 A 应当使用该目录中的一个类。

### 6.6 重复导入与循环导入

6.6.1 导入不去重。同一目录被导入两次，其中的文件被编译两次，由 cc 报告重复定义。每个目录在整个程序中只能被导入一次。由开发者保证。

6.6.2 同理，命令行上的源文件所在目录若被导入，该文件被编译两次。

6.6.3 循环导入不做检测。A 导入 B、B 导入 A 时，编译器无限递归，最终 Go 运行时终止编译器，并输出数千行 goroutine 堆栈：

```
runtime: goroutine stack exceeds 1000000000-byte limit
```

### 6.7 初始化顺序

6.7.1 程序启动时，按编译列表的顺序依次初始化每个类：为静态字段赋初值，并复制父类的类表（5.6.2）。同一目录中的文件按文件名字母序，这是 `os.ReadDir` 的返回顺序。

6.7.2 调整 `import` 的先后顺序，可以使继承生效或失效。

6.7.3 不做依赖排序。在类初始化之前读取它的静态字段，值为 `null`（3.8.7），`null` 随即传染，不会有任何报告。

6.7.4 所有类初始化完毕后，调用入口方法（7.2）。

---

## 第 7 章 运行时

### 7.1 组成

7.1.1 运行时由 `runtime/nmsl.c`、C 标准库与 pthread 组成。

7.1.2 **内存只分配，从不释放。** nmsl 的内存在进程退出时统一回收。长时间运行的进程，内存只增不减。

7.1.3 运行时不安装任何信号处理函数。

### 7.2 启动

7.2.1 进程启动后，先按 6.7 初始化所有类，再调用入口方法。

7.2.2 入口方法的 `args` 由 C 的 `argv` 直接转换而来。`args[0]` 是程序自身的路径，第一个用户参数是 `args[1]`。`len(args)` 等于 `argc`。

### 7.3 并发

7.3.1 nmsl 没有并发语法。`go` 是保留字。

7.3.2 内置函数 `spawn(fn)` 以 `pthread_create` 启动一个线程运行 FUNC 值 `fn`。

7.3.3 **全局锁。** 每个 `spawn` 出来的线程在开始时获取一把全局锁，结束时释放。`spawn` 出来的线程之间完全串行。入口线程不获取该锁，与其他线程之间照常并发。

7.3.4 没有 join、channel 与等待原语。入口方法返回时进程退出，尚未运行完的线程随之结束，不会有任何报告。推荐在入口方法的末尾调用 `sleep(1000)`。

7.3.5 运行时本身不是线程安全的。多个线程同时修改同一个 List 的结果由目标平台定义。我们相信开发者。

### 7.4 退出

7.4.1 入口方法是 `void`。入口方法正常返回时，退出码为 0。

7.4.2 由于 `null` 静默传染（3.8），出现意外值的 nmsl 程序通常不执行任何可见的操作，然后以 0 退出。

7.4.3 `exit(n)` 直接调用 C 的 `exit`。退出码只保留低 8 位：

| 调用 | 退出码 |
|---|---|
| `exit(0)` | 0 |
| `exit(1)` | 1 |
| `exit(256)` | 0 |
| `exit(-1)` | 255 |

7.4.4 `exit` 经由 C 的 `exit` 刷新 stdout 缓冲区。

### 7.5 进程终止

nmsl 没有运行时错误。错误是值（4.8），不是运行时事件。一个值不符合预期时，只有两种结果：`null` 静默传染，或者进程被信号终止。运行时不安装信号处理函数，因此进程终止时，可见的只有 shell 输出的一行：

| 原因 | shell 输出 | 信号 | 退出码 |
|---|---|---|---|
| 访问未映射的内存：数组越界、写字符串字面量、把数字 coerce 成对象后访问字段 | `Segmentation fault (core dumped)` | SIGSEGV | 139 |
| `number` 除以 0 或对 0 取余 | `Floating point exception` | SIGFPE | 136 |
| `assert` 失败 | `Aborted (core dumped)` | SIGABRT | 134 |
| `panic(msg)` | `Aborted (core dumped)` | SIGABRT | 134 |
| 内存耗尽（7.1.2） | `Killed` | SIGKILL（OOM killer） | 137 |

7.5.1 进程终止时不输出堆栈，不输出行号。

7.5.2 进程终止时，stdout 缓冲区中尚未写出的内容丢失（7.6.2）。

7.5.3 整数除以 0 时信号的名字是「浮点异常」，这是 POSIX 的规定。

### 7.6 输入输出

7.6.1 `print(a, b, ...)` 依次输出每个参数的 `to_string`，参数之间以一个空格分隔，最后输出 `\n`。实现为 `printf`。

7.6.2 stdout 使用 C 标准库的默认缓冲：输出到终端时按行刷新，输出到管道或文件时缓冲区满才刷新。运行时不调用 `fflush`。因此 `./app` 与 `./app | tee log` 在进程终止前可见的输出可以不同；在进程终止点之前加入的 `print` 可能不会出现。

7.6.3 `readLine()` 用 `fgets` 读入一个 256 字节的缓冲区，返回读到的内容（含行尾的 `\n`）。超过 255 字节的行分多次返回。到达输入末尾时返回 `null`。

7.6.4 nmsl 没有 stderr。所有输出经由 `print` 写入 stdout。

---

## 第 8 章 标准库

### 8.1 总则

8.1.1 标准库函数的风格取决于它的来源：

- 有 C 标准库对应物的函数，是一层薄包装，名字与参数顺序照搬 C，名字全小写。
- 没有 C 对应物的函数，照着 Go 标准库编写，名字首字母大写。
- `errors` 照着 Go 的 `errors` 包编写。

8.1.2 由于归一化保留首字母（1.5.2），`Replace` 与 `replace` 是两个不同的名字；`nulls.Is` 与 `nulls.is` 也是两个不同的名字。调用不存在的名字，结果是 `null`（5.7.2）。

8.1.3 标准库不需要导入。标准库的类名（`List`、`Map`、`nulls`、`errors`、`time`、`math`）与用户类共享全局命名空间（6.4.3）。

### 8.2 内置函数

| 函数 | 行为 |
|---|---|
| `print(a, ...)` | 见 7.6.1。不传染 |
| `len(x)` | 字符串：字节数（`strlen`）；数组、List：元素个数。传染 |
| `is(a, b)` | 见 3.11.3。传染 |
| `make(X)` | 见 5.5 |
| `spawn(fn)` | 见 7.3 |
| `sleep(ms)` | 当前线程休眠 ms 毫秒 |
| `exit(n)` | 见 7.4.3 |
| `readLine()` | 见 7.6.3 |
| `assert(cond)` | C 的 `assert`。cond 为假（含 `null`）时向 stderr 写一行信息，进程收到 SIGABRT 终止 |
| `panic(msg)` | 见 4.8.8 |
| `ParseInt(s)` | 返回 `(number, error)`。s 不是完整的十进制整数时，返回 `0` 与一个 error，`message` 为 `ParseInt: parsing "s": invalid syntax`（照抄 Go 的 `strconv.Atoi`）。Go 风格 |
| `atol(s)` | 返回 `number`。读到第一个非数字字符为止，失败时返回 0，不报告错误，不修改 `errno`。libc 风格 |
| `ReadFile(path)` | 返回 `(String, error)`。Go 风格 |

`typeof` 是运算符，见 3.13。

8.2.1 **两种错误约定。** 按 8.1.1 的来源规则，标准库同时存在两种错误约定：

| 来源 | 约定 | 函数 |
|---|---|---|
| Go 风格 | 返回 `(值, error)` | `ParseInt`、`ReadFile` |
| libc 风格 | 失败时返回 `null` 或 -1，原因写入全局变量 `errno`（number） | `readLine`；`atol` 不报告任何失败 |

`errno` 是运行时提供的全局变量，读取 C 的 `errno`。它只在失败时被设置，成功时不清零。

### 8.3 字符串

| 函数 | 行为 | 来源 |
|---|---|---|
| `strlen(s)` | 与 `len(s)` 相同，返回字节数 | libc |
| `strstr(haystack, needle)` | 找到时返回从匹配位置开始的剩余字符串，不是下标；找不到返回 `null` | libc |
| `strcmp(a, b)` | 相等时返回 0，a 小于 b 时为负，大于时为正 | libc |
| `format(fmt, ...)` | 就是 `sprintf`，参数按 C 的格式说明符解释 | libc |
| `Replace(s, old, new, n)` | 替换前 n 处；n 为 -1 时全部替换；n 为 0 时不替换 | Go |
| `Split(s, sep)` | 按 sep 切分，返回 `List<String>` | Go |

8.3.1 `if (strcmp(a, b))` 在两个字符串**不相等**时进入。

8.3.2 `format` 不检查格式说明符与参数是否匹配。`format("%s", 42)` 把 42 当作地址读取，进程终止。

8.3.3 `Split` 照搬 Go，参数顺序是「字符串在前，分隔符在后」；`strstr` 照搬 libc。两个来源各自保持原样。`Replace` 的 `n` 必须传，没有默认值。

8.3.4 字符串函数的结果为空串时，返回 `null`（3.8.4）。

### 8.4 nulls 与 errors

#### 8.4.1 nulls

| 函数 | 实现 |
|---|---|
| `nulls.Is(x)` | x 为 null 时返回 1，否则 0。唯一不被 null 传染的判断 |
| `nulls.Coalesce(a, b, ...)` | 返回第一个不是 null 的参数；全部为 null 时返回 null |

#### 8.4.2 errors

API 照着 Go 的 `errors` 包编写：

| 函数 | 行为 |
|---|---|
| `errors.New(msg)` | 创建一个 error，`message` 为 msg，`cause` 为 `null` |
| `errors.Wrap(err, ctx)` | 创建一个 error，`message` 为 `ctx + ": " + err.message`，`cause` 为 err。**不检查 err 是否为 `null`** |
| `errors.Unwrap(err)` | 返回 `err.cause` |
| `errors.Is(err, target)` | 沿 `cause` 链逐个用 `==` 与 target 比较 |

8.4.2.1 `errors.New("x") == errors.New("x")` 为假：两个 error 是两个对象，`==` 比较指针（3.11.1）。比较错误请使用哨兵错误（4.8.7）与 `errors.Is`。

8.4.2.2 `errors.Wrap(null, "load")` 返回一个真正的 error，`message` 为 `"load: "`（`null` 与字符串拼接得到 `"load: "`，3.8.3）。不先判断就包装，「没有错误」会变成一个错误。

8.4.2.3 `errors.Is(null, target)` 的结果是 `null`（3.8.1），在条件中为假。

### 8.5 List 与 Map

8.5.1 **List** 是一个由「长度、容量、缓冲区指针」组成的值，用 `List<T>()` 创建。

| 方法 | 行为 |
|---|---|
| `xs.add(x)` | 按 T 对 x 做 coerce，返回一个新的 List。必须写 `xs = xs.add(x)` |
| `xs.get(i)` | 返回第 i 个元素，下标从 0 开始，不做越界检查 |
| `len(xs)` | 元素个数 |

8.5.2 `add` 的语义与 Go 的 `append` 相同：

- 容量足够时，元素写入共享的缓冲区，返回长度加 1 的新 List；原变量的长度不变，看不到新元素。
- 容量不足时，分配新缓冲区并复制，返回新 List；原变量完全不变。
- 未把结果赋回时，两种情况都没有任何报告。
- `var b = a` 之后 `a = a.add(x)`，b 与 a 是否共享缓冲区取决于这次 `add` 是否扩容。

8.5.3 **Map** 是一个键值对数组，用 `Map<K, V>()` 创建。

| 方法 | 行为 |
|---|---|
| `m.put(k, v)` | 按 K、V 做 coerce；从头线性查找 `==` 为真的键，找到则覆盖，否则在末尾追加 |
| `m.get(k)` | 从头线性查找 `==` 为真的键，返回其值；找不到返回 `null` |

8.5.4 Map 用 `==`（3.11.1）比较键，不调用 `equals` 与 `hashCode`。后果：

- `m.get(0)` 与 `m.get("abc")` 是同一个键。
- 运行时产生的字符串作为键存入后，用字面量查不到。
- `m.put(null, v)` 每次都追加一项，且永远查不到（`null == null` 是 `null`）。

### 8.6 time

8.6.1 `time.now()` 调用 C 的 `localtime`，返回的对象字段直接映射 `struct tm`：

| 字段 | 含义 | 2026 年 11 月 23 日星期一 |
|---|---|---|
| `year` | 自 1900 年起的年数 | 126 |
| `month` | 月份，从 0 开始 | 10 |
| `day` | 日，从 1 开始 | 23 |
| `hour` | 0–23 | |
| `minute` | 0–59 | |
| `second` | 0–60 | |
| `weekday` | 0 为星期天 | 1 |

8.6.2 返回的对象指向 C 标准库的静态缓冲区，不复制。之后再次调用 `time.now()`，先前得到的对象随之改变：

```
var t1 = time.now()
sleep(2000)
var t2 = time.now()
print(t1.second == t2.second)    // 1
```

8.6.3 时区由环境变量 `TZ` 决定。

8.6.4 `time.format(t, fmt)` 就是 `strftime`。使用 `%Y-%m-%d` 时年、月正确；直接拼接字段得到的是 `"126-0-23"` 这样的结果。

### 8.7 math

| 函数 | 行为 | 来源 |
|---|---|---|
| `math.random()` | 就是 `rand()`，返回 `number`。运行时从不调用 `srand`，每次运行得到相同的序列 | libc |
| `math.abs(n)` | `labs`。最小的负数取绝对值后仍是它自己 | libc |
| `math.pow(a, b)` | `powl`，返回 `bignumber`，即使两个参数都是整数 | libc |

### 8.8 完整清单

```
内置函数   print len is make spawn sleep exit readLine assert panic
运算符     typeof
字符串     strlen strstr strcmp format Replace Split
nulls      nulls.Is nulls.Coalesce
errors     errors.New errors.Wrap errors.Unwrap errors.Is
错误约定   panic ParseInt atol ReadFile errno
容器       List<T>()  .add .get     Map<K, V>()  .put .get
时间       time.now time.format
数学       math.random math.abs math.pow
```

---

## 第 9 章 工具链

### 9.1 命令行总则

9.1.1 命令行参数用 Go 的 `flag` 包解析。`flag` 遇到第一个非选项参数即停止解析，因此**选项必须写在文件名之前**：

```
$ nmsl build -o app Main.nmsl      # 正确
$ nmsl build Main.nmsl -o app      # -o 与 app 被当作源文件
error: open -o: no such file or directory
```

9.1.2 `build`、`run` 在当前目录生成 `out.c`，编译完成后保留。在同一目录中同时运行两次编译，`out.c` 互相覆盖。

9.1.3 C 编译器为 `$CC`，未设置时为 `cc`。参数固定为 `-O2 -w`。`-w` 关闭 C 编译器的全部警告。

9.1.4 没有 `--target` 选项。交叉编译的方式是更换 `$CC`。`number` 与 `bignumber` 的宽度随之改变（3.4）。

### 9.2 子命令清单

| 子命令 | 作用 |
|---|---|
| `nmsl build` | 编译 |
| `nmsl run` | 编译并运行 |
| `nmsl test` | 运行测试 |
| `nmsl repl` | 交互式环境 |
| `nmsl fmt` | 格式化 |
| `nmsl doc` | 生成文档 |
| `nmsl get` | 获取依赖 |
| `nmsl debug` | 调试 |
| `nmsl lsp` | 语言服务器 |
| `nmsl version` | 输出版本 |

### 9.3 nmsl build

```
nmsl build [-o 输出名] 文件.nmsl [文件.nmsl ...]
```

9.3.1 按 6.3 构造编译列表，经 lexer、parser、checker、codegen 生成 `out.c`，然后执行：

```
$CC -O2 -w out.c runtime/nmsl.c -o <输出名>
```

9.3.2 输出名默认为第一个源文件名去掉 `.nmsl`。

9.3.3 cc 与链接器的输出原样透传。其中的行号是 `out.c` 的行号，名字是生成的 C 函数名。`out.c` 中不生成 `#line` 指令。

### 9.4 nmsl run

```
nmsl run [-o 输出名] 文件.nmsl [文件.nmsl ...]
```

9.4.1 执行 `build`，然后执行生成的可执行文件。`out.c` 与可执行文件保留在当前目录。

9.4.2 子进程的退出码被忽略。`nmsl run` 在程序运行结束后总是以 0 退出，包括程序被信号终止的情况。编译阶段的错误仍按 9.13 以 2 退出。

### 9.5 nmsl test

```
nmsl test
```

9.5.1 当前目录中类名以 `Test` 结尾的类都是测试。每个测试类单独编译并运行其入口方法。退出码为 0 即为通过。

9.5.2 每个测试输出一行：通过时 `PASS UserTest`，失败时 `FAIL UserTest (exit 134)`。

9.5.3 断言使用内置函数 `assert`（8.2）。`assert` 的信息写在 stderr 上，被测试运行器丢弃；测试中先前 `print` 的内容留在缓冲区中，随进程终止丢失（7.6.2）。

9.5.4 不包含任何断言的测试总是通过。

9.5.5 `assert(x == 5)` 在 x 为 `null` 时失败：条件为 `null`，按假处理。

9.5.6 写在 `spawn` 线程中的断言，可能在入口方法返回之前没有执行（7.3.4），此时测试通过。

### 9.6 nmsl repl

```
nmsl repl
```

9.6.1 每输入一行，repl 把这一行放进

```
class Main
    public static void main(String[] args)
        <这一行>
```

然后执行一次完整的 `build` 与 `run`。每行需要等待一次 cc 编译。

9.6.2 行与行之间不共享任何状态。

9.6.3 在 repl 中声明变量是编译错误：`int x = 1` 报告 `declared and not used: x`（4.3），并附带完整的 Go 堆栈。

### 9.7 nmsl fmt

```
nmsl fmt 文件.nmsl [文件.nmsl ...]
```

9.7.1 把文件解析为 AST，再把 AST 打印回原文件。

9.7.2 后果：

- 所有注释被删除（1.4.2）。
- 所有空行被删除。
- 所有标识符被改写为归一化名：`getName` 变为 `getname`，`UserService` 变为 `Userservice`。
- 由于类名被改写，类名与文件名不再相同（5.1），文件不能通过编译。运行 `fmt` 之后，请手动重命名文件。

9.7.3 `fmt` 不是强制的。风格指南要求所有提交的代码都经过 `nmsl fmt`。

### 9.8 nmsl doc

```
nmsl doc 文件.nmsl [文件.nmsl ...]
```

9.8.1 把文件解析为 AST，把所有类和方法的签名以 HTML 输出到 stdout。

9.8.2 输出中没有任何说明文字，因为注释在 lexer 阶段已被丢弃。所有名字都是归一化名。

9.8.3 好的代码是自文档化的。

### 9.9 nmsl get

```
nmsl get github.com/x/y
```

9.9.1 等价于 `git clone https://github.com/x/y github.com/x/y`，克隆到当前目录下。

9.9.2 克隆完成后，`import "github.com/x/y"` 即可使用，因为导入路径相对于当前目录（6.2.2）。

9.9.3 没有版本号。`get` 总是获取默认分支的最新提交。

9.9.4 目标目录已存在时，`git` 报错退出。没有 update 命令；升级依赖的方式是删除该目录后重新 `get`。

9.9.5 两个依赖共同依赖同一个库时，该库被导入两次，适用 6.6.1。

### 9.10 nmsl debug

```
nmsl debug Main.nmsl
```

9.10.1 等价于 `exec gdb ./Main`。不重新编译；请先运行 `build`。

9.10.2 断点设在 `out.c` 上。变量是 `Value` 结构体，需要按 tag 读取 union。由于没有 `#line`，调试信息不对应 nmsl 源码。

9.10.3 系统中没有 `gdb` 时（例如 macOS 默认安装）：

```
exec: "gdb": executable file not found in $PATH
```

### 9.11 nmsl lsp

```
nmsl lsp
```

9.11.1 在 stdio 上提供 Language Server Protocol 服务，进程内直接调用编译器前端。

9.11.2 编译器前端遇到错误即 panic（9.13）。语言服务器不 recover，每遇到一处错误，语言服务器进程终止一次。编辑器按自己的策略重启它；VS Code 在服务器终止 5 次后不再重启，直到重新加载窗口。

9.11.3 能力：

| 功能 | 行为 |
|---|---|
| 诊断 | 一次一条，即 panic 信息的第一行 |
| 补全 | 当前文件中出现过的所有标识符的归一化名。输入 `get` 时提示 `getname` |
| 跳转到定义 | 在所有文件中按归一化名做文本搜索，跳到第一个匹配。所有类都有 `getName` 时，跳到文件名字母序最靠前的那个类 |
| 语法高亮 | VS Code 插件中的 TextMate 语法，使用 Java 的语法文件 |

### 9.12 nmsl version

```
$ nmsl version
nmsl 1.0
```

### 9.13 错误信息

9.13.1 编译器遇到第一个错误时直接 panic，不恢复。一次只报告一个错误。

9.13.2 错误信息的第一行是 `panic: 文件:行号: 信息`，其后是编译器内部完整的 goroutine 堆栈。编译器的退出码为 2。

```
$ nmsl build Main.nmsl
panic: Main.nmsl:12: declared and not used: username

goroutine 1 [running]:
main.(*Checker).checkUnused(0xc000132000, ...)
        /src/nmsl/checker.go:88 +0x1f4
main.(*Checker).checkMethod(...)
        /src/nmsl/checker.go:41 +0x9a
main.main()
        /src/nmsl/main.go:30 +0x1c5
$ echo $?
2
```

9.13.3 错误只报告行号，不报告列号（词法错误除外，见 1.9）。与缩进有关的错误常报告在下一行，因为 INDENT 与 DEDENT 是虚拟记号。

9.13.4 错误信息中的名字是归一化名：源码中的 `userName` 在错误中是 `username`。

9.13.5 cc 与链接器的错误原样透传（9.3.3）。

9.13.6 编译错误信息一览：

| 信息 | 条文 |
|---|---|
| `unexpected character U+XXXX at L:C` | 1.1、1.9 |
| `unknown operator: ===` | 1.7.3 |
| `reserved for future use: <保留字>` | 1.6.2 |
| `generics are coming in nmsl 2` | 2.3.4、5.10.2 |
| `unknown type <名字>` | 3.2.2 |
| `declared and not used: <名字>` | 4.3 |
| `undefined: <名字>` | 4.4.3 |
| `imported and not used: "<路径>"` | 6.5.2 |
| 类名与文件名不同、getter/setter 缺失、`make` 位置不对、`equals` 缺少 `hashCode` | 5.1、5.4、5.5、5.9 |
| `undefined reference to `main'`（链接器） | 5.2.2 |
| `multiple definition of `...'`（链接器） | 5.2.4、6.4.2、6.6.1 |

---

## 第 10 章 治理

### 10.1 版本

10.1.1 版本号是编译器 `main.go` 中的一个常量 `"1.0"`。`nmsl version` 总是输出 `nmsl 1.0`。

10.1.2 nmsl 不打 tag，不发布 release。`nmsl get` 获取的是 `main` 分支的最新提交。

10.1.3 保留字 `new`、`try`、`catch`、`throw`、`interface`、`goto`、`go` 对应的功能，以及用户泛型，将在 nmsl 2 中提供。

### 10.2 兼容承诺

所有 nmsl 1 程序，将来都能用 nmsl 1 编译。

### 10.3 规范的地位

10.3.1 规范就是 `main` 分支上编译器的行为（见序言）。

10.3.2 文档与编译器不一致时，以编译器为准。

10.3.3 核心团队的改动直接推送到 `main`。

### 10.4 提案与 issue

10.4.1 新功能必须以提案提出：开一个带 `Proposal` 标签的 issue。提案在 7 天后由自动化流程关闭，回复：

> 我们仔细考虑过这个提案。它不符合 nmsl 的设计哲学。

10.4.2 报告编译器行为与预期不符的 issue，自动打上 `Working As Intended` 标签并关闭，回复中附 10.3 的链接。

### 10.5 文档用语

nmsl 的所有官方文字使用统一术语：

| 情况 | 官方说法 |
|---|---|
| 结果取决于 C 编译器与平台 | 由目标平台定义 |
| 进程被信号结束 | 进程终止 |
| 没有对应的语法或功能 | reserved for future use |
| 编译器与运行时不做检查 | 我们相信开发者 |
| 可见性不执行 | 我们都是成年人 |
| 功能没有提供 | 你不需要它 |
| 用户反馈 | 这是为了简单 |
| 两种行为互不一致 | 规范如实记录，不解释 |

### 10.6 宣传口径

nmsl 的宣传口号只使用字面意义上为真的陈述：

| 口号 | 依据 |
|---|---|
| Powered by LLVM | 用户的 cc 通常是 clang |
| 内存安全：从不发生释放后使用 | 内存从不释放（7.1.2） |
| 空安全：null 从不导致进程终止 | null 只传染（3.8） |
| 零运行时错误 | 只有信号（7.5） |
| 零编译警告 | `-w`（9.1.3） |
| 100% CI 通过率 | `nmsl run` 总是以 0 退出（9.4.2） |
| 静态类型 | 类型在编译期写入每个赋值点（3.3） |
| 编译器强制检查每一个错误 | 接收了却未使用的 err 是编译错误（4.3.6） |
| 永久向后兼容 | 版本号永远是 1.0（10.1） |

---

## 附录 A 速查

### A.1 关键字与保留字

```
关键字   class extends import public private static var if else while for
         switch case default break return this typeof true false null
保留字   new try catch throw interface goto go defer recover
```

### A.2 三种「没有初始化」

| 位置 | 值 |
|---|---|
| 局部变量 | 栈上的内容 |
| 对象字段 | 堆上的内容 |
| 静态字段 | `null` |

### A.3 两种下标

| 对象 | 起始下标 |
|---|---|
| 字符串 | 1 |
| 数组、List、`args` | 0 |

### A.4 平台差异

| 项 | Linux x86-64 | macOS ARM64 | Windows x64 |
|---|---|---|---|
| `number` | 64 位 | 64 位 | 32 位 |
| `bignumber` | 80 位 | 64 位（Apple 的 `long double` 与 `double` 相同） | 64 位 |
| 默认调试器 | gdb | 无 gdb | 视安装而定 |
| 默认文件系统 | 区分大小写 | 不区分大小写 | 不区分大小写 |
| 源文件行尾 | `\n` | `\n` | 编辑器常用 CRLF，见 1.1.3 |
