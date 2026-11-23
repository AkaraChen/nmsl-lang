# 工厂与 getter/setter 指南

nmsl 的对象模型建立在两条经过业界二十年验证的实践之上：**对象由工厂创建**，**字段经由访问器读写**。别的语言把它们写在风格指南里，nmsl 把它们写进了编译器。

## 一个完整的类

一个类通常由两个文件组成：

```
// User.nmsl
class User
    String name
    number age
    boolean admin

    User()
        this.name = "匿名"
        this.age = 0
        this.admin = false

    public String getName()
        return this.name
    public void setName(String n)
        this.name = n

    public number getAge()
        return this.age
    public void setAge(number a)
        this.age = a

    public boolean isAdmin()
        return this.admin
    public void setAdmin(boolean a)
        this.admin = a
```

```
// UserFactory.nmsl
class UserFactory
    public static User create()
        return make(User)
```

使用：

```
User u = UserFactory.create()
u.setName("alice")
print(u.getName())       // alice
```

下面逐条说明。

## 第一部分：工厂

### 没有 new

nmsl 没有 `new` 表达式。`new` 是一个保留字，文法里没有对应的写法：

```
User u = new User()      // reserved for future use: new
```

### make

创建对象的唯一方式是内置函数 `make(X)`。它为 X 分配内存，调用 X 的构造函数，返回新对象（SPEC 5.5）。

### make 只能写在工厂里

`make(X)` 只能出现在名为 `XFactory` 的类中：

| `make` 写在 | 结果 |
|---|---|
| `UserFactory` 里的 `make(User)` | 合法 |
| `Main` 里的 `make(User)` | 编译错误 |
| `OrderFactory` 里的 `make(User)` | 编译错误 |

这保证了每个类的创建逻辑都有唯一的归属。代码审查时，你只需要打开一个文件就能知道某个类的对象是怎样诞生的。

### 标准写法

每个类配一个工厂，工厂里一个静态方法 `create`，方法体只有一行：

```
class UserFactory
    public static User create()
        return make(User)
```

需要带参数创建对象时，在工厂中设置字段：

```
class UserFactory
    public static User create()
        return make(User)
    public static User named(String n)
        User u = make(User)
        u.setName(n)
        return u
```

构造函数没有参数（SPEC 2.3.3）。参数化的创建逻辑属于工厂，这正是工厂存在的意义。

### 工厂类的名字请逐字照写

工厂类名按**原文**与 `XFactory` 比较（SPEC 5.5.5）。`UserFactory` 可以 `make(User)`；`User_factory` 不可以。

另一方面，类名在其他地方都按归一化名处理，`User_factory` 与 `UserFactory` 在运行时是同一个类。请统一使用 `XFactory` 的写法，同时请注意不要在一个程序里同时出现这两个类（SPEC 6.4.2）。

### 内置容器不需要工厂

```
var xs = List<int>()
var m = Map<String, int>()
```

`List` 与 `Map` 由标准库提供。标准库是可信代码。

## 第二部分：构造函数与字段

### 构造函数

构造函数是与类同名、没有返回类型、没有参数的方法。`make` 在分配内存后调用它。

### 为每个字段赋值

对象的内存在分配时不清零。构造函数中没有赋值的字段，内容是那块内存原有的内容，**不是** `null`（SPEC 5.5.6）：

```
class Point
    number x
    number y
    Point()
        this.x = 0
        // 忘记了 y
    ...
```

```
Point p = PointFactory.create()
print(p.getY())          // 堆上原有的内容
```

请在构造函数中为每一个字段赋值。nmsl 相信开发者。

## 第三部分：getter 与 setter

### 字段没有修饰符

字段声明不能写 `public` 或 `private`，字段一律视为私有：

```
class User
    String name          // 正确
    private String name  // 编译错误
```

### 每个字段都要有访问器

对每个字段 `x`，类中必须有 `getX` 和 `setX`。`boolean` 字段用 `isX` 代替 `getX`（SPEC 5.4）：

| 字段 | 必须有 |
|---|---|
| `String name` | `getName`、`setName` |
| `number age` | `getAge`、`setAge` |
| `boolean admin` | `isAdmin`、`setAdmin` |
| `static number count` | `getCount`、`setCount` |

缺少其中任何一个，或者为 `boolean` 字段写了 `getX`，都是编译错误。

### 只读字段

只读字段同样需要 setter。如果你不希望它被修改，就让 setter 不做任何事。参数没有用到会触发未使用检查，请这样写：

```
public void setId(number v)
    _ = v
```

### 访问器的内容由你决定

编译器只检查访问器的**名字**是否存在，不检查它的参数、返回类型和方法体。`getName` 可以做缓存、做计算、记日志。这是你的自由。

### 访问器之外

可见性不被执行（SPEC 5.3.2），字段同样可以直接读写：

```
u.name = "bob"
print(u.name)            // bob
```

我们都是成年人。访问器是你对团队的承诺，直接访问字段是你对自己的信任。两者 nmsl 都支持。

## 第四部分：equals 与 hashCode

一个类定义了 `equals`，就必须同时定义 `hashCode`（SPEC 5.9）：

```
class Point
    ...
    public boolean equals(Point o)
        return this.x == o.x && this.y == o.y
    public number hashCode()
        return this.x * 31 + this.y
```

编译器检查两个方法都存在。保持它们的一致性，是你的责任。

`Map` 比较键时使用 `==`，不调用这两个方法（SPEC 8.5.4）。`equals` 与 `hashCode` 是写给读代码的人看的契约。

## 第五部分：继承中的工厂

子类同样需要自己的工厂：

```
// Animal.nmsl
class Animal
    String sound
    Animal()
        this.sound = "..."
    public String getSound()
        return this.sound
    public void setSound(String s)
        this.sound = s
    public String speak()
        return this.sound
```

```
// Dog.nmsl
class Dog extends Animal
    Dog()
        this.sound = "汪"
```

```
// DogFactory.nmsl
class DogFactory
    public static Dog create()
        return make(Dog)
```

继承通过在启动时复制父类的方法表实现。请确保父类文件按文件名字母序排在子类之前，或者通过 `import` 顺序让父类先初始化（SPEC 5.6、6.7）。`Dog extends Animal` 符合这一条；`Apple extends Fruit` 不符合，此时请调整 import 顺序或文件布局。

## 检查清单

- [ ] 每个类都有一个 `XFactory`，名字逐字照写
- [ ] `make` 只出现在工厂中
- [ ] 构造函数为每个字段赋值
- [ ] 每个字段都有 `getX`/`isX` 与 `setX`
- [ ] 定义了 `equals` 就定义 `hashCode`
- [ ] 父类文件名排在子类之前

## 延伸阅读

- [《为什么没有 new》](../blog/2026-03-03-why-no-new.md)
- [《我们都是成年人》](../blog/2026-06-09-consenting-adults.md)
