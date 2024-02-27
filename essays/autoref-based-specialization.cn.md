# 广义的基于自动引用的特化

来源：https://lukaskalbertodt.github.io/2019/12/05/generalized-autoref-based-specialization.html

2019年12月5日

几周前，dtolnay [提出了基于自动引用的特化的想法](https://github.com/dtolnay/case-studies/tree/master/autoref-specialization)，这使得在稳定的Rust中可以使用类似特化的行为。虽然这种方法有一些根本性的限制，但是技术的初始描述的一些其他限制是可以克服的。本文介绍了一种采用的基于自动引用的特化版本——称为自动*解*引用的特化——通过引入两个关键变化，比原始版本更通用，并且可以在更广泛的情况下使用。

# 🔑 主要观点

- 使用自动*解*引用而不是自动引用，以使用任意多个特化级别。
- 使用简单的包装类型，避免与现有的引用的全局实现冲突。

#### 前言

首先值得澄清的一件事是，这里描述的采用的版本并没有解决自动引用的特化的*主要*限制，即在泛型上下文中进行特化。例如，给定 `fn foo<T: Clone>()`，你不能使用自动引用的特化在该函数中为 `T: Copy` 进行特化。对于这种破坏[参数性](https://en.wikipedia.org/wiki/Parametricity)的情况，“真正”的特化仍然是必需的。因此，整个自动引用的特化技术仍然主要适用于与宏一起使用的情况。

我还要感谢 dtolnay 提出并公开描述这个巧妙的想法。

# 快速回顾：方法解析

“方法解析”是编译器试图确定方法调用表达式 `receiver.method(args)` 的某些细节的过程。这主要包括两个[相互依赖](https://stackoverflow.com/q/58889717/2408867)的信息，这些信息没有明确指定：

- **调用哪个方法？**（类型的内在方法？作用域中的 trait 的方法？）
- **如何将接收器类型强制转换为与方法的 `self` 类型匹配？**

不要求程序员明确地写出这些细节使得方法调用更方便使用。然而，这种相当复杂的方法解析有时会导致[令人惊讶的行为](https://dtolnay.github.io/rust-quiz/23)和[向后兼容性风险](https://github.com/rust-lang/rust/pull/65819)。自动引用的特化通过（滥用）方法解析更倾向于需要更少的接收器类型强制转换的方法而不是需要更多强制转换的方法来工作。具体来说，该技术使用*自动引用强制转换*，因此得名。

值得强调的一个结果是，这种方式不像[RFC 1210中提出的特化](https://github.com/rust-lang/rfcs/blob/master/text/1210-impl-specialization.md)那样受到限制，其中一个 impl 必须严格地“比另一个更具体”。例如，对于 `String` 的 impl 严格比 `T: Display` 的 impl 更具体，而 `T: Display` 和 `T: Debug` 的两个 impl 在任何方向上都没有这种关系（因为它们都不是对方的超 trait 约束）。使用广义的自动引用的特化，我们可以简单地定义一个有序的 impl 列表，然后使用适用于接收器类型的第一个 impl。我将简单地将这个列表中的一个条目称为“特化级别”。特化级别之间不必在任何方面具有“严格更具体”的关系！

# 使用自动*解*引用实现 ≥ 两个特化级别

dtolnay 原始文章中的所有示例都使用了两个特化级别，但有时希望有三个或更多级别——特别是因为我们不受“严格更具体”的限制。让我们尝试使用这个技术来实现三个级别。在这里，我们只使用 `Ta`、`Tb` 和 `Tc` 这样的虚拟 trait，而不是有用的 trait，其中 `Ta` 优先级最高，即如果一个类型实现了 `Ta`，它应该通过该 impl 进行分发，忽略其他两个；`Tb` 优先级第二高，依此类推。我们使用以下设置来实现我们的特化技巧：

```
trait ViaA { fn foo(&self) { println!("A"); } }
impl<T: Ta> ViaA for T {}

trait ViaB { fn foo(&self) { println!("B"); } }
impl<T: Tb> ViaB for &T {}

trait ViaC { fn foo(&self) { println!("C"); } }
impl<T: Tc> ViaC for &&T {}
```

[如果你尝试这样做](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=4ac63d20a3d19dc291150c28a91e2ba0)，你会发现它失败了。假设类型 `struct Sc;` 只实现了 `Tc`，调用 `(&Sc).foo()` 会导致“在当前作用域中找不到名为 `foo` 的类型为 `&Sc` 的方法”。

这只是因为“自动引用强制转换”只应用零次或一次。编译器不会添加可能无限多个 `&` 直到接收器类型匹配。幸运的是，方法解析自动应用的一个强制转换可能会多次应用：解引用强制转换。使用自动解引用，级别的顺序相反：优先级更高的 impl 在其 `Self` 类型中有*更多*的引用。在我们的示例中（`Ta` 仍然具有最高优先级）：

```
impl<T: Ta> ViaA for &&T {}
impl<T: Tb> ViaB for &T {}
impl<T: Tc> ViaC for T {}
```

另一个重要的区别是我们的方法调用接收器必须具有与最高优先级方法中的 `self` 相同数量的引用。在这种情况下，我们需要方法调用表达式 `(&&&Sc).foo()`。这是因为 `<&&T as ViaA>::foo` 中的 `self` 的类型为 `&&&T`。方法解析实际上只关心 `self` 的类型，而不关心 `Self`！

方法调用中引用数量过少会导致奇怪的错误（例如，第一和第二优先级交换）。这是因为方法解析以一种不直观的顺序尝试接收器类型。有关此的非常详细的信息，请查看[这个庞然大物般的 StackOverflow 帖子](https://stackoverflow.com/q/28519997/2408867)（其中一个答案是我的）。

通过这些调整，[我们的示例现在按照我们的期望工作](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=b95c7db29d933627b11fecfbe26c86d4)。通过使用自动解引用而不是自动引用，我们可以使用任意多个特化级别。

（有趣的是，在撰写本节之前，我的广义技术仍然比这个更复杂。具体来说，我认为由于方法解析的奇怪性质，任何两个级别之间将被*两个*引用层分隔；例如，`ViaA for &&&&T` 和 `ViaB for &&T`。通过撰写本文并重新测试，我注意到了这种更简单和更直接的方式。）

# 避免与其他全局 impl 冲突

再次回到有用的 trait，让我们尝试使用新的自动*解*引用的技巧来处理 `String` 和 `Display`：

```
trait ViaString {
    fn foo(&self) { println!("String"); }
}
impl ViaString for &String {}

trait ViaDisplay {
    fn foo(&self) { println!("Display"); }
}
impl<T: Display> ViaDisplay for T {}


(&&String::from("hi")).foo();
```

[编译这个](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=bb5dfc1db8d316f3e7228ed108c9af0b)会导致 `error[E0034]: multiple applicable items in scope`。非常不幸和令人惊讶，考虑到我们的虚拟 trait 的示例是可以工作的。弄清楚这里出了什么问题花费了我比我愿意承认的时间。`Display` trait 有一个干扰的全局 impl：

```
impl<T> Display for &T    // <-- 实现了 &T，与我们的 impl 重叠
where
    T: Display + ?Sized,
{ ... }
```

而 `Display` 绝对不是 `std` 中唯一一个具有这种 impl 的 trait。幸运的是，解决方法非常简单：只需使用一个包装类型。

```
struct Wrap<T>(T);

impl ViaString for &Wrap<String> {}
impl<T: Display> ViaDisplay for Wrap<T> {}
```

这样就可以避免与其他 impl 的冲突，因为你完全控制 `Wrap` 实现了哪些 trait。

当然，到目前为止，方法调用变得非常丑陋：`(&&&Wrap(receiver)).foo()` 不是正常人想要调用方法的方式。这就是为什么——如介绍中所提到的——这种方法在大多数情况下与宏结合使用时最有用，因为宏可以轻松地生成这些奇怪的方法调用。

# 技术总结

假设你有一个有序的 N 个类型集合的列表（“特化级别”），例如（0）`String`，（1）`T: Display`，（2）`T: Debug`。你希望你的方法 `foo` 的调用通过列表中的第一个（按顺序）包含方法调用的接收器类型的类型集合进行分发，例如 `String` 通过（0）进行分发，而 `u32` 通过（1）进行分发。为了实现这一点，你需要：

- 创建类型 `struct Wrap<T>(T)`。

- 对于列表中索引为 `i` 的每个特化级别：

  - 创建一个 `Via⟨desc⟩` trait，其中 `⟨desc⟩` 是该级别的一个好描述（例如 `ViaDisplay`）。在这个 trait 中添加 `foo` 方法。
  - 为 `⟨refs⟩ Wrap<⟨set⟩>` 实现该 trait，其中 `⟨refs⟩` 简单地是 `N - i - 1` 次 `&`，`⟨set⟩` 描述了该特化级别的类型集合。例如 `impl ViaString for &&Wrap<String>` 或 `impl<T: Display> ViaDisplay for &Wrap<T>`。

- 对于你的方法调用：

  - 确保所有的 `Via*` trait 在作用域中。
  - 使用包装类型将你的接收器类型包装起来 `( ⟨refs⟩ Wrap<⟨receiver⟩> ).method()`，其中 `⟨refs⟩` 是 N 次 `&`，`⟨receiver⟩` 是原始的接收器。例如 `(&&&Wrap(r)).method()`。

[**完整示例**](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=f76739e2736a311530349f14477aa54c)：

```
struct Wrap<T>(T);

trait ViaString { fn foo(&self); }
impl ViaString for &&Wrap<String> {
    fn foo(&self) { println!("String: {}", self.0); }
}

trait ViaDisplay { fn foo(&self); }
impl<T: Display> ViaDisplay for &Wrap<T> {
    fn foo(&self) { println!("Display: {}", self.0); }
}

trait ViaDebug { fn foo(&self); }
impl<T: Debug> ViaDebug for Wrap<T> {
    fn foo(&self) { println!("Debug: {:?}", self.0); }
}

// 测试方法调用
(&&&Wrap(String::from("hi"))).foo();
(&&&Wrap(3)).foo();
(&&&Wrap(['a', 'b'])).foo();
```

当然，这个确切的步骤并不适用于所有情况。例如，你可能需要将其与 dtolnay 文档中描述的[*类型标签*技术](https://github.com/dtolnay/case-studies/tree/master/autoref-specialization#realistic-application)结合使用。但是，这个简短的“算法”是一个很好的起点。

# 结论

如本文所述的自动解引用的特化比 dtolnay 的原始描述更灵活，但无法摆脱一般方法的根本限制。当与宏一起使用时，这种技术非常有用，但对于除了这些用例之外的一般采用来说，它过于受限且冗长。Rust 中肯定需要一个适当的“特化解决方案”。

我个人在进行中的 [`domsl` 库](https://github.com/LukasKalbertodt/domsl) 中使用了自动解引用的特化，这也是我最初开始尝试 dtolnay 的想法的原因。