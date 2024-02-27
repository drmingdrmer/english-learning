# Generalized Autoref-Based Specialization

Source: [https://lukaskalbertodt.github.io/2019/12/05/generalized-autoref-based-specialization.html](https://lukaskalbertodt.github.io/2019/12/05/generalized-autoref-based-specialization.html)

December 5, 2019

A few weeks ago, dtolnay [proposed the idea of autoref-based specialization](https://github.com/dtolnay/case-studies/tree/master/autoref-specialization), which allows for specialization-like behavior in stable Rust. While this approach has some fundamental limitations, some of the other limitations of the initial description of the technique can be overcome. This article presents a version of autoref-based specialization that uses automatic *deref* instead of automatic reference, called autoderef-based specialization, which is more general and can be used in a wider range of situations.

# 🔑 Key Points

- Use automatic *deref* instead of automatic reference to allow for an arbitrary number of specialization levels.
- Use simple wrapper types to avoid conflicts with existing global implementations of references.

#### Introduction

First of all, it is worth clarifying that the version described here does not address the *main* limitation of autoref-based specialization, which is specialization in a generic context. For example, given `fn foo<T: Clone>()`, you cannot specialize `T: Copy` using autoref-based specialization in that function. "True" specialization is still necessary for cases that break [parametricity](https://en.wikipedia.org/wiki/Parametricity). Therefore, the whole technique of autoref-based specialization is still primarily applicable to cases where it is used with macros.

I would also like to thank dtolnay for proposing and publicly describing this clever idea.

# Quick Review: Method Resolution

"Method resolution" is the process by which the compiler tries to determine certain details of a method call expression `receiver.method(args)`. This mainly includes two pieces of information [that depend on each other](https://stackoverflow.com/q/58889717/2408867) and are not explicitly specified:

- **Which method is being called?** (The inherent method of the type? The method of a trait in scope?)
- **How is the receiver type coerced to match the `self` type of the method?**

Programmers are not required to explicitly write out these details to make method calls more convenient to use. However, this rather complex method resolution process can sometimes lead to [surprising behavior](https://dtolnay.github.io/rust-quiz/23) and [backwards compatibility risks](https://github.com/rust-lang/rust/pull/65819). Autoref-based specialization works by (abusing) method resolution to prefer methods that require fewer receiver type coercions over methods that require more coercions. Specifically, the technique uses *automatic reference coercion*, hence the name.

One important consequence is that this approach is not limited like the specialization proposed in [RFC 1210](https://github.com/rust-lang/rfcs/blob/master/text/1210-impl-specialization.md), where one impl must strictly be "more specific" than another. For example, an impl for `String` is strictly more specific than an impl for `T: Display`, but two impls for `T: Display` and `T: Debug` do not have this relationship in either direction (because they are not supertrait constraints of each other). With generalized autoref-based specialization, we can simply define an ordered list of impls and use the first one that applies to the receiver type. I will simply refer to an entry in this list as a "specialization level". Specialization levels do not have to have a "strictly more specific" relationship in any way!

# Implementing ≥ Two Specialization Levels with Autoderef

All the examples in dtolnay's original article used two specialization levels, but sometimes it is desirable to have three or more levels - especially since we are not bound by the "strictly more specific" limitation. Let's try to use this technique to implement three levels. Here, we use virtual traits like `Ta`, `Tb`, and `Tc` instead of useful traits, where `Ta` has the highest priority, meaning if a type implements `Ta`, it should be dispatched through that impl, ignoring the other two; `Tb` has the second highest priority, and so on. We use the following setup to implement our specialization trick:

```
trait ViaA { fn foo(&self) { println!("A"); } }
impl<T: Ta> ViaA for T {}

trait ViaB { fn foo(&self) { println!("B"); } }
impl<T: Tb> ViaB for &T {}

trait ViaC { fn foo(&self) { println!("C"); } }
impl<T: Tc> ViaC for &&T {}
```

[If you try to do this](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=4ac63d20a3d19dc291150c28a91e2ba0), you will find that it fails. Assuming the type `struct Sc;` only implements `Tc`, calling `(&Sc).foo()` will result in "no method named `foo` found for type `&Sc` in the current scope".

This is simply because "automatic reference coercion" is only applied zero or one times. The compiler does not add potentially infinite `&`s until the receiver type matches. Fortunately, a coercion that is automatically applied during method resolution can be applied multiple times: dereference coercion. With autoderef, the order of the levels is reversed: the impl with higher priority has *more* references in its `Self` type. In our example (`Ta` still has the highest priority):

```
impl<T: Ta> ViaA for &&T {}
impl<T: Tb> ViaB for &T {}
impl<T: Tc> ViaC for T {}
```

Another important difference is that our method call receiver must have the same number of references as the `self` in the highest priority method. In this case, we need the method call expression `(&&&Sc).foo()`. This is because the type of `self` in `<&&T as ViaA>::foo` is `&&&T`. Method resolution actually only cares about the type of `self`, not `Self`!

Having too few references in the method call leads to strange errors (e.g., the first and second priorities are swapped). This is because method resolution tries receiver types in a non-intuitive order. For very detailed information about this, see [this monstrous StackOverflow post](https://stackoverflow.com/q/28519997/2408867) (one of the answers is mine).

With these adjustments, [our example now works as expected](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=b95c7db29d933627b11fecfbe26c86d4). By using autoderef instead of autoref, we can use an arbitrary number of specialization levels.

(It is interesting to note that my generalized technique was actually more complex than this before writing this section. Specifically, I thought that any two levels would be separated by *two* reference layers due to the weird nature of method resolution, e.g., `ViaA for &&&&T` and `ViaB for &&T`. By writing this article and retesting, I noticed this simpler and more direct way.)

# Avoiding Conflicts with Other Global Impls

Returning to useful traits, let's try to use the new autoderef-based trick to handle `String` and `Display`:

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

[Compiling this](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=bb5dfc1db8d316f3e7228ed108c9af0b) will result in `error[E0034]: multiple applicable items in scope`. Very unfortunately and surprisingly, considering that our example with virtual traits worked. It took me more time than I'm willing to admit to figure out what went wrong here. The `Display` trait has a conflicting global impl:

```
impl<T> Display for &T    // <-- Overlaps with our impl
where
    T: Display + ?Sized,
{ ... }
```

And `Display` is definitely not the only trait in `std` that has such an impl. Fortunately, the solution is very simple: just use a wrapper type.

```
struct Wrap<T>(T);

impl ViaString for &Wrap<String> {}
impl<T: Display> ViaDisplay for Wrap<T> {}
```

This avoids conflicts with other impls because you have full control over which traits `Wrap` implements.

Of course, the method call becomes very ugly in the process: `(&&&Wrap(receiver)).foo()` is not how normal people want to call methods. That's why - as mentioned in the introduction - this technique is most useful when combined with macros, as macros can easily generate these weird method calls.

# Technical Summary

Suppose you have an ordered list of N sets of types (the "specialization levels"), e.g., (0) `String`, (1) `T: Display`, (2) `T: Debug`. You want calls to your method `foo` to be dispatched through the set of types in the first (in order) specialization level that contains the method call's receiver type, e.g., `String` through (0), `u32` through (1). To achieve this, you need to:

- Create the type `struct Wrap<T>(T)`.

- For each specialization level at index `i` in the list:

  - Create a `Via⟨desc⟩` trait, where `⟨desc⟩` is a good description of that level (e.g., `ViaDisplay`). Add a `foo` method to this trait.
  - Implement this trait for `⟨refs⟩ Wrap<⟨set⟩>`, where `⟨refs⟩` is simply `N - i - 1` times `&`, and `⟨set⟩` describes the set of types for that specialization level. For example, `impl ViaString for &&Wrap<String>` or `impl<T: Display> ViaDisplay for &Wrap<T>`.

- For your method call:

  - Make sure all the `Via*` traits are in scope.
  - Wrap your receiver type with the wrapper type `( ⟨refs⟩ Wrap<⟨receiver⟩> ).method()`, where `⟨refs⟩` is N times `&`, and `⟨receiver⟩` is the original receiver. For example, `(&&&Wrap(r)).method()`.

[**Complete example**](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=f76739e2736a311530349f14477aa54c):

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

// Test method calls
(&&&Wrap(String::from("hi"))).foo();
(&&&Wrap(3)).foo();
(&&&Wrap(['a', 'b'])).foo();
```

Of course, these exact steps do not apply to all situations. For example, you might need to combine it with the [*type tag* technique](https://github.com/dtolnay/case-studies/tree/master/autoref-specialization#realistic-application) described in dtolnay's documentation. However, this short "algorithm" is a good starting point.

# Conclusion

Autoderef-based specialization, as described in this article, is more flexible than dtolnay's original description, but it cannot overcome the fundamental limitations of the general method. This technique is very useful when used with macros, but it is too limited and verbose for general adoption beyond these use cases. Rust definitely needs a proper "specialization solution".

I personally use autoderef-based specialization in my ongoing [`domsl` library](https://github.com/LukasKalbertodt/domsl), which is why I initially started experimenting with dtolnay's idea.