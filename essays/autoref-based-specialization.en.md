# Generalized Autoref-Based Specialization

Origin: https://lukaskalbertodt.github.io/2019/12/05/generalized-autoref-based-specialization.html

Dec 5, 2019

A few weeks ago, dtolnay [introduced the idea of autoref-based specialization](https://github.com/dtolnay/case-studies/tree/master/autoref-specialization), which makes it possible to use specialization-like behavior on stable Rust. While this approach has some fundamental limitations, some other limitations of the technique’s initial description *can* be overcome. This post describes an adopted version of autoref-based specialization – called auto*de*ref-based specialization – which, by introducing two key changes, is more general than the original one and can be used in a wider range of situation.

# 🔑 Key takeaways

- Use auto*de*ref instead of autoref to use arbitrarily many specialization levels.
- Use a simple wrapper type to avoid interference with existing blanket implementations for references.

#### Foreword

One thing might be worth clarifying up front: the adopted version described here does not solve *the* main limitation of autoref-based specialization, namely specializing in a generic context. For example, given `fn foo<T: Clone>()`, you cannot specialize for `T: Copy` in that function with autoref-based specialization. For these kinds of [parametricity](https://en.wikipedia.org/wiki/Parametricity)-destroying cases, “real” specialization is still required. As such, the whole autoref-based specialization technique is still mainly relevant for usage with macros.

I’d also like to thank dtolnay for coming up with and publicly describing this ingenious idea.



# Quick Recap: Method Resolution

“Method resolution” is the process in which the compiler tries to figure out certain details about a method call expression `receiver.method(args)`. This mainly includes two [interdependent](https://stackoverflow.com/q/58889717/2408867) pieces of information which are not specified explicitly:

- **Which method to call?** (An inherent method of a type? A method of a trait in scope?)
- **How to coerce the receiver type to match the `self` type of the method?**

Not requiring the programmer to explicitly write out these details make method calls more convenient to use. However, this rather complex method resolution sometimes results in [surprising behavior](https://dtolnay.github.io/rust-quiz/23) and [backwards-compatibility hazards](https://github.com/rust-lang/rust/pull/65819). Autoref-based specialization works by (ab)using the fact that method resolution prefers resolving to methods which require fewer type coercion of the receiver over methods that require more coercions. Specifically, the technique uses *autoref coercions*, hence the name.

One consequence worth emphasizing is that this way, we are not limited like the [specialization as proposed in RFC 1210](https://github.com/rust-lang/rfcs/blob/master/text/1210-impl-specialization.md) where one impl has to be strictly “more specific” than the other. As an example, an impl for `String` is strictly more specific than an impl for `T: Display`, whereas the two impls `T: Display` and `T: Debug` do not have this relationship in either direction (since neither is a super trait bound of the other). With generalized autoref-based specialization, we can simply define an ordered list of impls and the first one that applies to the receiver type is used. I will simply call one entry in this list a “specialization level”. Specialization levels do not have to have “strictly more specific” relations with one another at all!

# Using auto*de*ref for ≥ two specialization levels

All examples in dtolnay’s original post use two specialization levels, but sometimes it’s desirable to have three or more levels – especially since we are not limited by “strictly more specific”. Let’s naively try to use three levels with this technique. Instead of useful traits like `Display` we just use the traits `Ta`, `Tb` and `Tc` here. `Ta` has the highest priority, i.e. if a type implements `Ta` it should dispatch via that impl, ignoring the other two; `Tb` has the second highest priority, and so on. For our specialization trick we use this setup:

```
trait ViaA { fn foo(&self) { println!("A"); } }
impl<T: Ta> ViaA for T {}

trait ViaB { fn foo(&self) { println!("B"); } }
impl<T: Tb> ViaB for &T {}

trait ViaC { fn foo(&self) { println!("C"); } }
impl<T: Tc> ViaC for &&T {}
```

[If you try this](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=4ac63d20a3d19dc291150c28a91e2ba0), you will see that this fails. Assuming the type `struct Sc;` only implements `Tc`, the call `(&Sc).foo()` results in “no method named `foo` found for type `&Sc` in the current scope”.

This is simply because “autoref coercion” is only applied either zero or one time. The compiler does *not* add potentially infinitely many `&`s until the receiver type matches. Fortunately, there is a coercion that is automatically applied by method resolution, potentially multiple times: deref coercion. With autoderef, the levels are ordered the other way around: higher priority impls have *more* references in their `Self` type. In our example (`Ta` still having the highest priority):

```
impl<T: Ta> ViaA for &&T {}
impl<T: Tb> ViaB for &T {}
impl<T: Tc> ViaC for T {}
```

Another important difference is that our method call receiver has to have the same number of references as the `self` in the highest priority method. In this case, we would need to have the method call expression `(&&&Sc).foo()`. That is because `self` in `<&&T as ViaA>::foo` has the type `&&&T`. Method resolution actually just cares about the type of `self`, and *not* `Self`!

Having too few `&` in the method call leads to strange errors (e.g. first and second priority switched). This is due to method resolution trying receiver types in an unintuitive order. For very detailed information about this, checkout [this beast of a StackOverflow post](https://stackoverflow.com/q/28519997/2408867) (one of the answers is mine).

With these adjustments, [our example now works as we wanted](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=b95c7db29d933627b11fecfbe26c86d4). By using autoderef instead of autoref, we can use as many specialization levels as we want.

(Amusingly, before writing this section, my generalized technique was still more complicated than this. Specifically, I thought that due to the strangeness of method resolution, any two levels would be separated by *two* reference-layers; e.g. `ViaA for &&&&T` and `ViaB for &&T`. By writing this post and testing things again, I noticed this simpler and more straightforward way.)

# Avoid interference with other blanket impls

Returning to useful traits again, let’s try the new auto*de*ref-based trick with `String` and `Display`:

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

[Compiling this](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=bb5dfc1db8d316f3e7228ed108c9af0b) results in `error[E0034]: multiple applicable items in scope`. Quite unfortunate and surprising, given that the example with our dummy traits worked. Figuring out what went wrong here took me way longer than I’d like to admit. The `Display` trait has an interfering blanket impl:

```
impl<T> Display for &T    // <-- implemented for &T, overlaps with our impls
where
    T: Display + ?Sized,
{ ... }
```

And `Display` is by far not the only trait in `std` that has an impl like that. Luckily, a workaround is pretty simple: just use a wrapper type.

```
struct Wrap<T>(T);

impl ViaString for &Wrap<String> {}
impl<T: Display> ViaDisplay for Wrap<T> {}
```

This stops all interference with other impls as you are in full control over what traits `Wrap` implements.

Of course, by now, the method calls have become quite ugly: `(&&&Wrap(receiver)).foo()` is not how normal people want to call their methods. That’s why – as mentioned in the introduction – this approach is mostly useful in combination with macros, where the macro can easily emit these strange method calls.

# The Technique Summarized

Lets assume you have an ordered list of N sets of types (“specialization levels”), e.g. (0) `String`, (1) `T: Display`, (2) `T: Debug`. You want calls of your method `foo` to dispatch via the first (in the order of your list) set of types that contains the receiver type of the method call, e.g. `String` dispatches via (0), while `u32` dispatches via (1). To make this happen, you have to:

- Create the type `struct Wrap<T>(T)`.

- For each specialization level with index

   

  ```plaintext
  i
  ```

   

  in your list:

  - Create a trait `Via⟨desc⟩` where `⟨desc⟩` is a good description of that level (e.g. `ViaDisplay`). Add the method `foo` to this trait.
  - Implement that trait for `⟨refs⟩ Wrap<⟨set⟩>` where `⟨refs⟩` is simply `N - i - 1` times `&`, and `⟨set⟩` describes the set of types for this specialization level. E.g. `impl ViaString for &&Wrap<String>` or `impl<T: Display> ViaDisplay for &Wrap<T>`.

- For your method call:

  - Make sure all `Via*` traits are in scope.
  - Wrap your receiver type `( ⟨refs⟩ Wrap<⟨receiver⟩> ).method()` where `⟨refs⟩` is N times `&` and `⟨receiver⟩` is the original receiver. E.g. `(&&&Wrap(r)).method()`.

[**Full example**](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=f76739e2736a311530349f14477aa54c):

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

Of course, this exact recipe does not fit all situations. For example, you likely have to combine it with the [*type tag* technique described in dtolnay’s document](https://github.com/dtolnay/case-studies/tree/master/autoref-specialization#realistic-application). But this brief “algorithm” is a good start.

# Conclusion

Autoderef-based specialization as explained in this post is more versatile than the original description by dtolnay, but does not get around the fundamental limitations of the general approach. The technique can be immensely useful when used together with macros, but is too limited and verbose for general adoption beyond those use-cases. A proper “specialization solution” is certainly required in Rust.

I personally use autoderef-based specialization for the work-in-progress [`domsl` library](https://github.com/LukasKalbertodt/domsl), which is also the reason why I started toying with dtolnay’s idea in the first place.
