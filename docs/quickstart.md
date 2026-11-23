# 快速开始

本页用五分钟带你写出、编译并运行第一个 nmsl 程序。

## 你需要什么

- `nmsl` 编译器
- 一个 C 编译器。nmsl 调用环境变量 `$CC` 指定的程序，没有设置时调用 `cc`。在大多数现代系统上，`cc` 就是 clang，这也是我们说 nmsl **Powered by LLVM** 的原因。
- 仓库中的 `runtime/nmsl.c`。

确认安装：

```
$ nmsl version
nmsl 1.0
```

nmsl 的版本号是 1.0。它将一直是 1.0（见[《nmsl 1 兼容承诺》](../blog/2026-11-18-nmsl1-compatibility.md)）。

## Hello, world

新建文件 `Main.nmsl`：

```
class Main
    public static void main(String[] args)
        print("Hello, world")
```

三件事需要一次记住，之后就再也不用想了：

1. **文件名必须和类名逐字节相同。** 类 `Main` 必须放在 `Main.nmsl` 里，不能是 `main.nmsl`（SPEC 5.1）。
2. **入口方法那一行必须一字不差。** 编译器用整行比较来识别入口：`public static void main(String[] args)`。多一个空格、把参数叫成 `argv`，它就不是入口（SPEC 5.2）。
3. **块由缩进决定。** 没有花括号，也没有冒号。

编辑器请保存为 UTF-8、**不带 BOM**、行尾为 **LF**。nmsl 只认 `\n`；它不处理你的编辑器替你做的决定（SPEC 1.1）。

## 编译

请在**项目根目录**下执行所有命令：

```
$ nmsl build Main.nmsl
$ ls
Main  Main.nmsl  out.c
```

`build` 生成了两样东西：

- `Main`：原生可执行文件。
- `out.c`：生成的 C 源码。它会保留在当前目录，方便你欣赏。

运行：

```
$ ./Main
Hello, world
```

### 指定输出名

选项必须写在文件名**之前**：

```
$ nmsl build -o hello Main.nmsl
```

写成 `nmsl build Main.nmsl -o hello` 时，`-o` 和 `hello` 会被当成两个源文件：

```
error: open -o: no such file or directory
```

这是 Go 标准 `flag` 包的约定，nmsl 完整继承。

## 编译并运行

```
$ nmsl run Main.nmsl
Hello, world
$ echo $?
0
```

`nmsl run` 先 `build`，再执行生成的程序。`nmsl run` 自身总是以 0 退出（SPEC 9.4），你的 CI 将因此保持稳定。

## 读取参数

```
class Main
    public static void main(String[] args)
        print("程序：" + args[0])
        print("第一个参数：" + args[1])
```

```
$ ./Main hello
程序：./Main
第一个参数：hello
```

`args` 与 C 的 `argv` 一致：`args[0]` 是程序自身。签名是 Java 的，语义是 C 的，两边的程序员都会感到熟悉。

## 一个稍长的例子

```
class Main
    public static void main(String[] args)
        int total = 0
        for (int i = 1; i < len(args); i = i + 1)
            int n = args[i]
            total = total + n
        print("合计：" + total)
```

```
$ ./Main 10 20 12abc
合计：42
```

`int n` 是一个带类型注解的变量。往里放值时，值会被转换为 `number`：`"12abc"` 变成 12（SPEC 3.3）。类型从不拒绝你。

注意这里先把参数放进 `int n`，再做加法。直接写 `total + args[i]` 时，`+` 看到一边是字符串，会做字符串拼接（SPEC 3.10.2）。这是 `+` 的定义，不是意外。

## 编译出错时

```
$ nmsl build Main.nmsl
panic: Main.nmsl:4: declared and not used: count

goroutine 1 [running]:
main.(*Checker).checkUnused(0xc000132000, ...)
        /src/nmsl/checker.go:88 +0x1f4
...
```

第一行就是你要的信息，其余部分是编译器的完整调用栈，供你深入了解编译器。nmsl 一次报告一个错误，让你专注于眼前的问题。

## 下一步

- [语言之旅](tour.md)
- [工具链参考](toolchain.md)
