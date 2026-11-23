# 我们都是成年人

2026-06-09 · nmsl 核心团队

nmsl 有 `public`，有 `private`。nmsl 不执行它们。

这篇文章解释为什么。

## 修饰符是写给人看的

nmsl 的 parser 会把 `public`、`private`、`static` 记进 AST。之后，没有任何一个阶段会去读可见性修饰符。运行时按名字查找方法和字段，不看修饰符。`private` 方法可以从任何地方调用。

我们收到过不少 issue，说「`private` 不起作用」。这些 issue 都被自动打上了 `Working As Intended` 标签。我们想借这篇文章把理由说清楚。

`private` 起作用了。它告诉读代码的人：这个方法是内部实现，请不要依赖它。这句话已经被说出来了，被读到了。至于读到之后你是否依然要调用它，那是你的决定。

我们都是成年人。

## Python 早就知道

这不是我们的发明。Python 社区很早就有一句话：we are all consenting adults here。Python 没有 `private` 关键字，只有约定：名字以下划线开头的，请当作内部实现。真想访问，没人拦你。

Python 也有双下划线的名字改写，让 `__x` 在类外变得难以访问。我们认为这是一处多余的让步。一门相信约定的语言，不应该再用名字改写去补一道锁。

nmsl 走得比 Python 更彻底。Python 用的是约定，nmsl 用的是关键字。我们给了你一个正式的、写进文法的 `private`，并且承诺它永远只是一个声明。你不需要猜测某个下划线是什么意思，`private` 就写在那里，清清楚楚。同时，它也不会妨碍你。

## getter 与 setter：强制，同时可以绕过

nmsl 的字段不能带可见性修饰符，一律视为私有。每个字段 `x` 都必须在同一个类中有 `getX` 与 `setX`；`boolean` 字段必须用 `isX` 代替 `getX`。缺任何一个，都是编译错误。

同时，`a.balance` 可以从任何地方直接读写。

有人觉得这两条放在一起很奇怪。我们觉得它们恰好互补。

```
// Account.nmsl
class Account
    number balance
    boolean frozen

    public number getBalance()
        return this.balance

    public void setBalance(number balance)
        this.balance = balance

    public boolean isFrozen()
        return this.frozen

    public void setFrozen(boolean frozen)
        this.frozen = frozen

    private void audit()
        print("audit: " + this.balance)
```

```
// AccountFactory.nmsl
class AccountFactory
    public static Account create()
        Account a = make(Account)
        a.setBalance(0)
        a.setFrozen(false)
        return a
```

```
// Main.nmsl
class Main
    public static void main(String[] args)
        Account a = AccountFactory.create()
        a.balance = 100            // 直接写字段
        print(a.getBalance())      // 100
        print(a.isFrozen())        // 0
        a.audit()                  // audit: 100
```

强制 getter/setter，保证每个类都有一套统一的访问接口，这是 JavaBeans 留给业界的宝贵遗产。允许直接访问字段，则保证你在需要的时候不被这套接口束缚。规则给了你结构，成年人的身份给了你自由。

## checker 只检查名字

checker 只检查 `getX`、`setX`、`isX` 这几个方法名是否存在，不检查参数、返回类型和方法体。`getName()` 可以返回任何东西，`setName()` 可以什么都不做。

这对只读字段尤其方便。只读字段同样必须有 setter，那就写一个什么都不做的 setter：

```
    public void setId(number id)
        _ = id
```

`_ = id` 是为了让参数 `id` 被提到第二次，未使用的参数是编译错误。这个 setter 什么都不做，于是 `id` 在语义上就是只读的。至于有人直接写 `x.id = 2`，参见上一节。

有人问，既然 checker 不看方法体，强制 getter/setter 的意义在哪里。意义在于名字。只要名字存在，你的类就有了一份统一的契约：任何人看到字段 `balance`，都知道可以调用 `getBalance` 和 `setBalance`。契约里写了什么，由你决定。我们相信开发者。

## 名字就是一切

nmsl 没有接口，`interface` 是保留字。方法按名字查找，名字对上就能调用。任何对象都「实现」了任何接口，只要方法名碰巧相同。由于标识符风格不敏感，`get_balance` 与 `getBalance` 也算相同。

这和可见性不执行是同一种哲学：nmsl 不在名字之外再加一层门禁。一个方法叫什么，它就是什么；它写着 `private`，它就是给你的一个提醒。

方法名拼错了怎么办？找不到的方法，调用结果是 `null`，然后 `null` 照常传染。程序不会因为一个拼写问题被打断。这一点我们在《Nulls are values》里已经讲过了。

## 结语

一门语言可以把开发者当成需要看管的孩子，在每个字段前面装一把锁。也可以把开发者当成成年人，告诉你哪里是内部实现，然后把钥匙交给你。

nmsl 选择后者。`public` 与 `private` 都在，它们都是真诚的。它们只是不执行。
