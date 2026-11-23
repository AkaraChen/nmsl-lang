# 工具链参考

nmsl 自带完整的工具链。别的语言需要你分别安装的编译器、运行器、测试框架、REPL、格式化器、文档生成器、包管理器、调试器和语言服务器，nmsl 全部内置在一个命令里。

```
nmsl <子命令> [选项] [参数]
```

| 子命令 | 作用 |
|---|---|
| [`build`](#nmsl-build) | 编译 |
| [`run`](#nmsl-run) | 编译并运行 |
| [`test`](#nmsl-test) | 运行测试 |
| [`repl`](#nmsl-repl) | 交互式环境 |
| [`fmt`](#nmsl-fmt) | 格式化 |
| [`doc`](#nmsl-doc) | 生成文档 |
| [`get`](#nmsl-get) | 获取依赖 |
| [`debug`](#nmsl-debug) | 调试 |
| [`lsp`](#nmsl-lsp) | 语言服务器 |
| [`version`](#nmsl-version) | 输出版本 |

## 通用约定

### 在项目根目录执行

导入路径相对于当前目录（SPEC 6.2.2），`nmsl get` 克隆到当前目录。所有子命令都请在项目根目录执行。

### 选项写在前面

命令行由 Go 标准库的 `flag` 包解析，遇到第一个非选项参数就停止。**选项必须写在文件名之前：**

```
$ nmsl build -o app Main.nmsl      # 正确
$ nmsl build Main.nmsl -o app      # -o 和 app 被当作源文件
error: open -o: no such file or directory
```

### C 编译器

nmsl 调用 `$CC`，未设置时调用 `cc`，参数固定为 `-O2 -w`：

- `-O2`：你的程序总是以优化模式编译。
- `-w`：C 编译器的警告全部关闭。生成的 C 代码由 nmsl 负责，你不需要为它操心。这也是 nmsl「零编译警告」的由来。

### 交叉编译

nmsl 没有 `--target` 选项，交叉编译就是换一个 C 编译器：

```
$ CC=x86_64-w64-mingw32-gcc nmsl build -o app.exe Main.nmsl
```

请注意，`number` 的宽度随目标平台变化：Windows x64 上它是 32 位（SPEC 3.4）。

### 错误信息

编译器在遇到第一个错误时停止，并输出该错误和完整的内部调用栈，退出码为 2：

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

阅读要点：

- 第一行是你要的信息：`文件:行号: 内容`。
- 只有行号，没有列号。与缩进有关的错误常常报告在下一行。
- 名字是归一化名：你写的 `userName`，错误中是 `username`（SPEC 1.5）。
- 一次一个错误。修好它，再编译一次。

C 编译器与链接器的输出原样透传。其中的行号是 `out.c` 的行号，名字是生成的 C 函数名，例如 `User_getname`。

---

## nmsl build

```
nmsl build [-o 输出名] 文件.nmsl [文件.nmsl ...]
```

编译源文件及其导入的所有目录，生成可执行文件。

```
$ nmsl build Main.nmsl
$ ls
Main  Main.nmsl  out.c
```

| 选项 | 作用 |
|---|---|
| `-o 名字` | 可执行文件名。默认为第一个源文件名去掉 `.nmsl` |

说明：

- 生成的 C 代码写在当前目录的 `out.c`，编译后保留。
- 请不要在同一目录中同时运行两个 `build`：它们共用同一个 `out.c`。
- 实际执行的命令是 `$CC -O2 -w out.c runtime/nmsl.c -o <输出名>`。

## nmsl run

```
nmsl run [-o 输出名] 文件.nmsl [文件.nmsl ...]
```

编译并运行。等价于 `nmsl build` 之后执行生成的程序。

```
$ nmsl run Main.nmsl
Hello, world
```

- `out.c` 和可执行文件保留在当前目录。
- `nmsl run` 在程序运行结束后**总是以 0 退出**，不论程序本身的退出码是什么，包括程序被信号终止的情况。编译失败时以 2 退出。

需要程序自身的退出码时，请先 `build`，再直接执行生成的文件：

```
$ nmsl build Main.nmsl && ./Main; echo $?
```

## nmsl test

```
nmsl test
```

运行当前目录中的测试。

### 编写测试

类名以 `Test` 结尾的类就是测试。每个测试类有自己的入口方法：

```
// UserTest.nmsl
class UserTest
    public static void main(String[] args)
        User u = UserFactory.create()
        u.setName("alice")
        assert(u.getName().equals("alice"))
```

### 运行

```
$ nmsl test
PASS UserTest
FAIL OrderTest (exit 134)
```

- 每个测试类单独编译、单独运行。退出码为 0 即通过。
- `assert(cond)` 就是 C 的 `assert`。条件为假时进程以 SIGABRT 终止，退出码 134。
- 测试报告只有一行。`assert` 的详细信息写在 stderr 上，由运行器丢弃；测试中先前 `print` 的内容在进程终止时随缓冲区一起丢失（SPEC 7.6.2）。请让每个测试类只验证一件事，这样 `FAIL` 后面的类名就是全部信息。

### 注意

- 没有 `assert` 的测试总是通过。
- `assert` 的条件为 `null` 时失败：`assert(x == 5)` 在 x 为 `null` 时失败。
- 写在 `spawn` 线程中的 `assert` 可能还没执行，`main` 就已经返回，此时测试通过。请把断言写在 `main` 中。

## nmsl repl

```
nmsl repl
```

交互式环境。每输入一行，repl 把它放进

```
class Main
    public static void main(String[] args)
        <你输入的这一行>
```

完整地编译并运行一次。

```
> print(7 / 2)
3
> print("12abc" * 1)
12
> print(typeof 1.5)
bignumber
```

- 每一行都经过完整的 C 编译，大约需要一秒。这保证 repl 中的行为与 `nmsl build` 完全一致。
- 行与行之间不共享状态。
- repl 中不能声明变量：`int x = 1` 中的 `x` 只出现一次，触发未使用变量检查（SPEC 4.3），报告 `declared and not used: x` 并附带完整堆栈。repl 适合探索表达式。

## nmsl fmt

```
nmsl fmt 文件.nmsl [文件.nmsl ...]
```

把源文件格式化为标准形式，直接写回文件。

fmt 把文件解析为语法树，再打印回去。因此格式化后的文件：

- 没有注释。
- 没有空行。
- 所有标识符使用归一化名：`getName` 变为 `getname`，`UserService` 变为 `Userservice`。

最后一条意味着类名改变了，而类名必须与文件名相同（SPEC 5.1）。**运行 `fmt` 之后，请手动重命名文件**：

```
$ nmsl fmt UserService.nmsl
$ mv UserService.nmsl Userservice.nmsl
```

同时请更新引用了旧文件名的地方。风格指南要求所有提交的代码都经过 `nmsl fmt`。

## nmsl doc

```
nmsl doc 文件.nmsl [文件.nmsl ...] > api.html
```

把所有类与方法的签名输出为 HTML，写到 stdout。

```
$ nmsl doc lib/model/User.nmsl
<h1>User</h1>
<ul>
<li>public String getname()</li>
<li>public void setname(String n)</li>
</ul>
```

- 输出中只有签名，名字是归一化名。
- 注释在编译的第一步就被丢弃，因此不会出现在文档中。

好的代码是自文档化的。

## nmsl get

```
nmsl get <仓库路径>
```

获取第三方依赖。

```
$ nmsl get github.com/someone/strutil
```

等价于：

```
$ git clone https://github.com/someone/strutil github.com/someone/strutil
```

依赖被克隆到当前目录下的同名路径中，之后可以直接导入：

```
import "github.com/someone/strutil"
```

- 没有版本号。`get` 总是获取默认分支的最新提交。
- 没有锁文件，没有清单文件，没有中央仓库。依赖就是一个目录。
- 目录已存在时，`git` 报告错误并退出。升级依赖：

  ```
  $ rm -rf github.com/someone/strutil
  $ nmsl get github.com/someone/strutil
  ```

- 两个依赖共同依赖同一个库时，该库会被导入两次，C 编译器将报告重复定义（SPEC 6.6.1）。

## nmsl debug

```
nmsl debug 可执行文件名
```

启动调试器。等价于 `exec gdb ./<可执行文件名>`。请先 `build`。

```
$ nmsl build Main.nmsl
$ nmsl debug Main
(gdb) break out.c:120
(gdb) run
(gdb) print v
$1 = {tag = 1 '\001', as = {n = 42, b = ..., s = 0x2a <error>, p = 0x2a}}
```

- 断点设在 `out.c` 上。生成的 C 代码中没有 `#line`，调试器显示的是 C 源码。
- 每个变量都是一个 `Value` 结构体。先看 `tag`（SPEC 3.1），再读 union 中对应的成员：tag 1 读 `n`，tag 2 读 `b`，tag 3 读 `s`。
- macOS 默认没有 `gdb`：

  ```
  exec: "gdb": executable file not found in $PATH
  ```

  请安装 `gdb`。

## nmsl lsp

```
nmsl lsp
```

在 stdio 上启动 Language Server Protocol 服务。编辑器插件会自动调用它。

| 功能 | 行为 |
|---|---|
| 诊断 | 一次一条，与编译器的第一行错误相同 |
| 补全 | 当前文件中出现过的所有标识符，以归一化名给出：输入 `get`，提示 `getname` |
| 跳转到定义 | 在所有文件中按归一化名搜索，跳到第一个匹配。多个类都有 `getName` 时，跳到文件名字母序最靠前的那个 |
| 语法高亮 | VS Code 插件使用 Java 的 TextMate 语法 |

语言服务器与编译器共用同一个前端。前端遇到错误时，语言服务器进程随之终止，由编辑器重新启动。VS Code 在语言服务器终止 5 次之后停止重启，并提示你；此时请执行「Developer: Reload Window」。

建议在输入时保持代码始终可以通过编译。

## nmsl version

```
$ nmsl version
nmsl 1.0
```

nmsl 的版本号是 1.0。见[《nmsl 1 兼容承诺》](../blog/2026-11-18-nmsl1-compatibility.md)。
