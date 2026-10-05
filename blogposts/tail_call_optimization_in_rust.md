# tail call optimizations, Rust and why you can't have both (yet)
tl;dr: Rust does not optimize recursive calls. Almost at all.

Recently I've been learning Gleam programming language. In Gleam tail call optimization (TCO)
is kinda guaranteed, and it got me curious about TCO state in Rust.

#### What is TCO?
Basically, TCO nukes the stack of the current function call as long as the recursive call is
the last function in the body. Stops the stack from growing on recursion and makes the whole
thing work faster.

#### Why TCO at all, why not loops?
Because functional programming does not have loops.
Then there are some other cases where TCO improves performance, but eh, who cares.

#### Why doesn't Rust have TCO?
First reason is RAII. When Rust desugars drop, calls to drop happen at the end of the block,
which normally happens to be a function block. To illustrate this, let's look at this function:
```rust
fn main() {
    println!("{}", not_tco_fib(0, 1, 3));
}
fn not_tco_fib(previous: u64, current: u64, n: u64) -> u64 {
    let _dropme = DropMe(n, "not_tco_fib");
    match n {
        0 => previous,
        1 => current,
        n => not_tco_fib(current, previous + current, n - 1),
    }
}
struct DropMe(u64, &'static str);
impl Drop for DropMe {
    fn drop(&mut self) {
        println!("dropme from {} with n={}", self.1, self.0);
    }
}
```
Running that program outputs this:
```
$ cargo run
dropme from not_tco_fib with n=1
dropme from not_tco_fib with n=2
dropme from not_tco_fib with n=3
2
```
As output shows, `not_tco_fib` drops `_dropme` **after** recursing.
That means TCO cannot be applied, as applying it would break the semantics of the code.

Second reason is pointers to values on the stack.
It's probably not Rust-only issue, but it is quite easy to do in Rust:
```rust
fn rec(i: &i32) {
    let x = *i;
    rec(&x)
}
```
Remember how TCO is nuking the stack of the function before recursion?
What happens when TCO nukes `x` and then calls a function with a reference `&x`?

Third reason… Actually, before discussing the third reason let me check:
if I leave only `Copy` values on the stack, will compiler optimize the thing for me?
```rust
fn main() {
    println!("{}", kinda_tco_fib(0, 1, 3));
}
fn kinda_tco_fib(previous: u64, current: u64, n: u64) -> u64 {
    {
        println!("{}", psm::stack_pointer() as usize);
    }
    match n {
        0 => previous,
        1 => current,
        n => kinda_tco_fib(current, previous + current, n - 1),
    }
}
```
Okay, close my eyes and pray…
```
$ cargo run --release
140726103980496
140726103980432
140726103980368
2
```
Yep, third reason: TCO is rarely happening without LLVM instructions.
And rustc does not emit those for any recursion ever.
Don't know what happens with gcc or cranelift backends and I'm too lazy to check.

#### Okay, no TCO, now what?
If you want your tailcalls to be optimized on stable, there are some crates out there.
For example: [tailcall](https://lib.rs/crates/tailcall).

But that is lame. Cool kids go to nightly and use all the cool keywords and features!
More specifically `become` keyword.

Now, let's see what happens with previous examples when using `become`:
```rust
#![feature(explicit_tail_calls)]
fn main() {
    println!("{}", not_tco_fib(0, 1, 3));
}
fn not_tco_fib(previous: u64, current: u64, n: u64) -> u64 {
    let _dropme = DropMe(n, "not_tco_fib");
    match n {
        0 => previous,
        1 => current,
        n => become not_tco_fib(current, previous + current, n - 1),
    }
}
```
The output changes very much compared to previous one:
```
# cargo run --release
dropme from not_tco_fib with n=3
dropme from not_tco_fib with n=2
dropme from not_tco_fib with n=1
2
```

`become` drops all local values before recursing. Interesting!
Now, what happens with the stack size?
```rust
#![feature(explicit_tail_calls)]
fn main() {
    println!("{}", kinda_tco_fib(0, 1, 3));
}
fn kinda_tco_fib(previous: u64, current: u64, n: u64) -> u64 {
    println!("{}", psm::stack_pointer() as usize);
    match n {
        0 => previous,
        1 => current,
        n => become kinda_tco_fib(current, previous + current, n - 1),
    }
}
```
Output:
```
$ cargo run --release
140733392630000
140733392630000
140733392630000
2
```

Stack doesn't grow! Yay!
Okay, what about the reference to local value?
```rust
fn rec(i: &i32) {
    let x = *i - 1;
    become rec(&x)
}
```
It does not compile!
```
error[E0597]: `x` does not live long enough
  --> src/main.rs:43:16
   |
42 |     let x = *i - 1;
   |         - binding `x` declared here
43 |     become rec(&x)
   |                ^^- `x` dropped here while still borrowed
   |                |
   |                borrowed value does not live long enough
```

#### Bonus: async
I lied. There is no bonus. Async functions are not functions, they are finite state machines.
No stack to nuke, no TCO.

Okay, fine, there is your bonus:
```rust
async fn foo(arg) {
    become foo(arg + arg)
}
```
I feel like `become` should be doing enough of a cleanup for this to simply return state machine
to the initial state with new arguments.

The problems begin when I'm trying to think what could possibly happen in the next code.
How would it be working with state machines? Which state could possibly represent this `become`?
```rust
async fn foo() {
    some_code().await;
    become bar()
}
async fn bar() {
    std::futures::ready(()).await
}
```

