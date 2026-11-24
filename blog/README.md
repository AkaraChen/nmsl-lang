# nmsl 官方博客

nmsl 核心团队关于语言设计的文章，按发布日期倒序排列。

| 日期 | 文章 | 摘要 |
|---|---|---|
| 2026-11-18 | [nmsl 1 兼容承诺](2026-11-18-nmsl1-compatibility.md) | 所有 nmsl 1 程序，将来都能用 nmsl 1 编译；版本号、规范、nmsl 2 与提案流程。 |
| 2026-09-15 | [我们的安全承诺：Powered by LLVM](2026-09-15-safety-promises.md) | 逐条说明 nmsl 的八句宣传语为什么在字面意义上都是真的。 |
| 2026-07-28 | [Errors are values, and so is null](2026-07-28-errors-are-values.md) | 错误是值，没有错误就是 null；为什么 `if (err != null)` 永远不进入，以及编译器如何强制检查每一个错误。 |
| 2026-06-09 | [我们都是成年人](2026-06-09-consenting-adults.md) | `public` 与 `private` 都在，它们只是不执行；getter/setter 强制，字段也可以直接访问。 |
| 2026-04-21 | [我们相信开发者](2026-04-21-trust-the-developer.md) | `number` 就是 C 的 `long`，宽度属于平台；换一个 `$CC` 就是交叉编译。 |
| 2026-03-03 | [为什么 nmsl 没有 new](2026-03-03-why-no-new.md) | `new` 是保留字，对象由 `XFactory` 里的 `make(X)` 创建。 |
| 2026-01-14 | [Nulls are values](2026-01-14-nulls-are-values.md) | 一个 null 融合三种传统；判空只有 `nulls.Is` 一种方法。 |
