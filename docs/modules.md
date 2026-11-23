# 模块与导入

nmsl 的模块系统只有一个概念：**目录**。没有包声明，没有模块文件，没有可见性导出规则。你已经知道怎么用目录了，所以你已经知道怎么用 nmsl 的模块了。

## import

```
import "lib/model"
```

这一行把 `lib/model/` 目录下的所有 `.nmsl` 文件加入编译（SPEC 6.2）。它和 C 的 `#include` 一样直接：没有隐藏的解析算法，没有搜索路径，没有魔法。

- 导入不进入子目录。`lib/model/dto/` 需要单独导入。
- 标准库（`print`、`List`、`nulls`、`time`、`math` 等）不需要导入。

## 路径相对于当前目录

导入路径相对于**你执行 `nmsl` 命令时的当前目录**，而不是相对于写 `import` 的那个文件。

```
myapp/
├── app/
│   └── Main.nmsl          import "lib/model"
└── lib/
    └── model/
        └── User.nmsl
```

```
$ cd myapp
$ nmsl build app/Main.nmsl        # 正确

$ cd app
$ nmsl build Main.nmsl
error: open lib/model: no such file or directory
```

**请始终在项目根目录执行 nmsl。** 这条规则让整个项目里的每一条 import 都有唯一的含义，不论它写在哪个文件里。

## 一个命名空间

所有类都位于同一个全局命名空间。类名直接用，不需要前缀：

```
import "lib/model"

class Main
    public static void main(String[] args)
        User u = UserFactory.create()
        print(u.getName())
```

这意味着整个程序中类名必须唯一，包括归一化之后：`UserService` 和 `User_service` 是同一个名字（SPEC 1.5）。两个同名的类会生成两个同名的 C 函数，由 C 编译器报告：

```
multiple definition of `User_getname'
```

请注意报告中的名字是生成的 C 函数名。你也不能定义与标准库同名的类，例如 `List`、`Map`。

## 导入对整个程序生效

一个文件导入了某个目录，程序中所有文件都能使用该目录中的类（SPEC 6.5.1）。

建议每个文件都显式导入自己使用的目录，这样读者一眼就能看出依赖关系。但请同时遵守下面两条规则。

## 未使用的导入是编译错误

一个文件导入了某个目录，却没有使用其中任何一个类名：

```
imported and not used: "lib/model"
```

整洁的依赖列表从编译器开始。

## 每个目录只导入一次

nmsl 的导入不去重（SPEC 6.6.1）。同一个目录被导入两次，其中的类就被编译两次，C 编译器会报告重复定义。

```
app/Main.nmsl            import "lib/model"  import "lib/util"
lib/util/Strings.nmsl    import "lib/model"      ← 第二次
```

```
$ nmsl build app/Main.nmsl
out.c:(.text+0x1a2): multiple definition of `User_getname'
```

**在整个程序中，每个目录只能被导入一次。** 请由一个文件统一导入，通常是入口所在的文件。

### 与「未使用的导入」配合

把上面几条规则放在一起：`lib/util` 中的 `Strings` 需要使用 `User`，但 `lib/model` 已经由 `Main` 导入，`Strings` 不能再导入一次。`Strings` 直接使用 `User` 即可，因为导入对整个程序生效。

而 `Main` 导入了 `lib/model`，就必须在 `Main.nmsl` 里出现 `lib/model` 中的至少一个类名。如果 `Main` 本身用不到，请在 `Main` 里使用其中一个类，例如声明一个该类型的变量并使用它。

### 不要循环导入

A 导入 B、B 导入 A 时，编译器不会停止：

```
runtime: goroutine stack exceeds 1000000000-byte limit
...
```

随后是数千行堆栈。请保持依赖关系为一棵树。

## 编译列表与初始化顺序

编译器按以下方式构造编译列表（SPEC 6.3）：

1. 命令行上的源文件，按给出的顺序。
2. 每加入一个文件，立即按顺序处理它的 `import`：目录中的文件按**文件名字母序**加入，并立即递归处理这些文件的 `import`。

程序启动时，类按编译列表的顺序初始化：为静态字段赋初值，复制父类的方法表（SPEC 6.7）。

这有两个后果，请记住：

**继承依赖顺序。** 子类在初始化时复制父类的方法表。父类必须先初始化：

| 文件 | 字母序 | 继承 |
|---|---|---|
| `Animal.nmsl`、`Dog.nmsl` | Animal 在前 | 生效 |
| `Apple.nmsl`、`Fruit.nmsl` | Apple 在前 | 不生效：Apple 的继承方法调用结果为 `null` |

需要时，把父类放在一个更早被导入的目录中。

**静态字段依赖顺序。** 在一个类初始化之前读取它的静态字段，得到的是 `null`：

```
// lib/a/Config.nmsl
class Config
    static String name = Defaults.name    // Defaults 尚未初始化时，name 是 null
    ...
```

请让被依赖的类先进入编译列表。

## 第三方依赖

```
$ nmsl get github.com/someone/strutil
$ ls github.com/someone/
strutil
```

`nmsl get` 把仓库克隆到当前目录下同名的路径中。由于导入路径相对于当前目录，克隆完成后直接导入即可：

```
import "github.com/someone/strutil"
```

`nmsl get` 获取的是默认分支的最新提交。依赖的升级方式是删除目录后重新获取。详见[工具链参考](toolchain.md#nmsl-get)。

两个依赖共同依赖同一个库时，该库会被导入两次（见上文「每个目录只导入一次」）。请选择依赖较少的库。

## 检查清单

- [ ] 在项目根目录执行所有 nmsl 命令
- [ ] 全程序类名唯一（按归一化名）
- [ ] 每个目录只导入一次
- [ ] 每个导入都在本文件中被使用
- [ ] 父类与被依赖的类先进入编译列表
- [ ] 没有循环导入
