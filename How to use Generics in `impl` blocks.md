# How to use Generic Parameters in `impl` block declaration statement

Finally I know where to put **Generic Parameters** in `impl<T, U> Trait<T> for Type<U> {}` block declaration, as well as what they mean.

## TLDR;
- After `impl` keyword we put all the generic parameters which are required for that particular `impl` block in `<…>`, may it be for trait or type.
- You can provide concrete types to trait and/or type in `impl` block declaration, in which case you don't need to write generic params like **impl<~~`T, U`~~> Trait\<u32\> for Type\<String\> {}**.
- method's generic param in an `impl` block is different from it's `impl<…>` declaration generics, as `impl`'s generic param are evaluated to be of same concrete type throughout the block, hence can be used in multiple functions depending on same generic type, whereas method generics are scoped to the particular function.

We'll use [Add Trait](https://doc.rust-lang.org/stable/std/ops/trait.Add.html) as our example. More about operator overloading [here](https://doc.rust-lang.org/stable/std/ops/index.html). Add trait takes a single generic parameter, which defaults to the type implementing trait, like following:
```rust
pub trait Add<Rhs = Self> {
  type Output;
  fn add(self, rhs: Rhs) -> Self::Output;
}
```
`Rhs` generic parameter represents what can be added to the type implementing this trait.
Let's try to implement `Add` for `Vec` type. Firstly wrapping `Vec` into a new type is required as `Vec` doesn't belong to our crate, and you can only implement a trait for a type, if at least one of two belongs to the current crate.
```rust
use std::ops::Add;

#[derive(Debug)]
struct MyVec<T>(Vec<T>);

impl<T> Add for MyVec<T> { // This is same as below comment
// impl<T> Add<MyVec<T>> for MyVec<T> {
  type Output = Self;
  fn add(mut self, rhs: Self) -> Self::Output {
    self.0.extend(rhs.0);
    MyVec(self.0)
  }
}

fn main() {
  let a = MyVec(vec![2, 3]);
  let b = MyVec(vec![4, 5]);
  let res = a + b;
  let _ = dbg!(res); // res = MyVec([2, 3, 4, 5])
}
```
In this `impl` declaration we take one generic param `T`, in form of `impl<T> ...`, and pass it to `MyVec`, in form of `... for MyVec<T> {`, and let generic param of `Add` default to `Self` by not passing it. This puts a restriction that we can only add `MyVec` to itself, but practically we should be able to add any homogeneous collection of `T` type items to it, e.g. `Vec<T>`, `&[T]`. It's time to use our `Rhs` generic parameter.
```rust
use std::ops::Add;

#[derive(Debug)]
struct MyVec<T>(Vec<T>);

impl<Rhs, T> Add<Rhs> for MyVec<T>
  where Rhs: std::iter::IntoIterator<Item = T>
{
  type Output = Self;
  fn add(mut self, rhs: Rhs) -> Self::Output {
    self.0.extend(rhs); // notice, we are NO more accessing "rhs.0"
    MyVec(self.0)
  }
}

fn main() {
  let a = MyVec(vec![2, 3]);
  let b = [4, 5];
  let c = MyVec(vec![6, 7]);
  let res = a + b + c.0; // notice, we have to access "c.0" as `MyVec` doesn't implement `IntoIterator`
  let _ = dbg!(res); // see for yourself
}
```
This time our `impl` block takes one more generic param `Rhs` which we pass to `Add` trait and `T` as before is passed to struct `MyVec`. We put a bound on our `Rhs` that it should be iterable to produce same items as the `MyVec`'s vector holds.

Let's see how do method generic parameters compare. In the following code we try to delegate two function calls from `MyVec` to underlying `Vec`.
```rust
impl<T> MyVec<T> {
    fn retain<F>(&mut self, f: F)
    where
        F: FnMut(&T) -> bool,
    {
        self.0.retain(f);
    }

    fn retain_mut<F>(&mut self, f: F)
    where
        F: FnMut(&mut T) -> bool,
    {
        self.0.retain_mut(f);
    }
}
```
In the above `impl` block we have created two methods, both generic over `F`. Type of closure (`F`) for both these methods is different as `FnMut` signature
for both differs, but even if both `F` generics in methods `retain` and `retain_mut` had the exact same trait bound, even then there concrete type
could have been different, assuming both of them satisfied their trait bounds.
At the same time, `&T` ( for `retain`'s closure ) and `&mut T` ( for `retain_mut`'s closure ), both will be references to the same concrete type T, which is derived at compile time.

Read [Summary (tldr) again](https://github.com/PreetamSing/rust_articles/blob/main/How%20to%20use%20Generics%20in%20%60impl%60%20blocks.md#tldr)

Lastly, maybe you can try implementing [std::iter::IntoIterator](https://doc.rust-lang.org/stable/std/iter/trait.IntoIterator.html) for `MyVec` such that we don't have to access it's 0th element to add it to another `MyVec`.
