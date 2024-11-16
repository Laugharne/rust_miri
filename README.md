# Exploring Rust's Miri: Unleashing the Ultimate Bug Buster in Your Rust Arsenal

> **Source:** [Exploring Rust's Miri: Unleashing the Ultimate Bug Buster in Your Rust Arsenal | by Byte Blog - Freedium](https://freedium.cfd/https://byteblog.medium.com/exploring-rusts-miri-unleashing-the-ultimate-bug-buster-in-your-rust-arsenal-00f6dfd97504)

When it comes to safe systems programming, Rust has a pretty high opinion of itself. And, well… it kind of has the chops to back it up. The ownership model, lifetimes, strict typing — Rust is an ecosystem that does **not** let undefined behavior (UB) waltz around unnoticed. But what if you really want to dig into the gnarly edge cases and guarantee there's no sneaky, lurking UB? That's where Miri comes in.

## Unsafe code

In Rust, `unsafe` allows you to bypass certain safety checks enforced by the compiler. Rust's core promise is memory safety, achieved through strict ownership, borrowing, and lifetime rules. However, some operations, like working directly with raw pointers or calling foreign functions (e.g., C code), require finer control over memory or system-level tasks that can't be managed within Rust's safe abstractions.

Using `unsafe` means telling the compiler, "I've checked this myself, and I believe it's safe." Here are the primary scenarios where `unsafe` is used in Rust:

1. **Dereferencing raw pointers**: Unlike regular references, raw pointers don't have ownership rules, so they can lead to segmentation faults if not handled carefully.
2. **Calling unsafe functions or foreign functions (FFI)**: Functions marked as `unsafe` must be called within an `unsafe` block since they could break Rust's safety guarantees.
3. **Accessing/modifying mutable static variables**: Since static variables are shared across the entire program, they need special handling to prevent data races.
4. **Implementing unsafe traits**: Some traits in Rust have special requirements that the compiler cannot enforce, so they're marked as `unsafe`.
5. **Manual memory management**: Allocating or deallocating memory directly or modifying memory layouts can be done in `unsafe`.

Although `unsafe` lets you do more, it's also a responsibility. Rust won't protect you from undefined behavior in `unsafe` blocks, so it's best to use it minimally and thoroughly verify the safety of your code…. which is where tools like Miri come in!

## Miri: Rust's Code Inspector Extraordinaire

Think of Miri as Rust's secret weapon for catching every possible bug before it has a chance to mess with your perfectly crafted code. It's an interpreter for Rust, tailored specifically to detect and prevent undefined behaviour in your programs. Miri dives into memory, checks your lifetimes, and uncovers all the lurking bugs that might only pop up when the moon is full and Mars is in retrograde.

**What Miri Can Do** - Flag any usage of uninitialised memory - Detect data races in concurrent code - Check for alignment violations (critical for low-level programming) - Keep a sharp eye on borrowing violations, especially when we're using Rust's `unsafe` superpowers

Ready to unleash Miri? Here's how.

## 1. Setting Up Miri

Installing Miri is as easy as adding a component in Rust:

`rustup component add miri`

To run your code under Miri's hawkish inspection, you can use:

`cargo miri run`

Miri will then interpret your Rust code line-by-line, looking for issues. While it's slower than compiled code, it's thorough — a true bug-busting powerhouse.

## 2. Undefined Behavior: The Dark Side of Unsafe Code

In Rust, undefined behavior is the result of violating the language's strict rules on memory safety and thread safety. It's not "just a bug" — it's a lurking menace that can lead to crashes, corrupted data, or unexpected behavior. The Rust compiler does its best to prevent UB in safe code, but with `unsafe` code or complex lifetimes, a few UB risks can still slip through.

Here are the most common UB types that Miri catches:

- **Uninitialized Memory**: Using memory that's never been initialised.
- **Data Races**: Concurrent threads accessing the same memory without proper synchronization.
- **Dangling Pointers**: Pointers to memory that's already been freed or moved.
- **Misaligned Memory Access**: Accessing memory at an offset that doesn't match its required alignment.
- **Violations of Rust's Borrowing Rules**: Especially likely to pop up in `unsafe`code.

## 3. Practical Examples: Miri in Action

The best way to understand Miri is to see it in action. Let's walk through a few scenarios where Miri shines.

**Example 1: Catching Uninitialised Memory**

Uninitialised memory is like leaving random trash on your desk and then telling your friend to guess what's inside it. Let's see what happens when we try to use an uninitialised variable in `unsafe`code:

```rust
fn main() {
    unsafe {
        let x: i32;
        println!("The value of x is: {}", x);
    }
}
```

This code will print garbage in release mode, but with Miri:

Copy`cargo miri run`

Miri's output:

```
error: attempted to read undefined bytes
 → src/main.rs:4:40
  |
4 | println!("The value of x is: {}", x);
  | ^ use of uninitialized memory
```

**Explanation** Miri catches the bug right away, letting us know `x`was never initialized. It saves us from what could have been a mysterious bug at runtime.

**Example 2: Data Race Detection**

Data races are sneaky bugs that can occur in concurrent code. Rust's ownership model prevents data races in safe code, but once we venture into `unsafe`, things can get chaotic.

```rust
use std::thread;

fn main() {
    let mut data = 0;
    let data_ptr = &mut data as *mut i32;
    let handle = thread::spawn(move || {
        unsafe {
            *data_ptr += 1;
        }
    });

    unsafe {
        *data_ptr += 1;
    }

      handle.join().unwrap();
}
```

In Rust's safe world, this would be flagged immediately, but since we're in `unsafe`, Rust assumes we know what we're doing. Miri, however, won't let us off the hook:

`cargo miri run`

Miri's output:

```
error: Data race detected between (2) Read on (i32) and (1) Write on (i32)
  → src/main.rs:14:9
   |
14 | *data_ptr += 1;
   | ^^^^^^^^^^^^^^^
```

**Explanation** Miri points out that we're causing a data race between the main thread and the spawned thread, as they're both accessing the same memory without synchronisation.

**Example 3: Alignment Misstep**

Certain types require memory to be aligned just right. Miri spots these misalignments like an overly precise carpenter with a ruler.

```rust
fn main() {
    let data = [1u16, 2, 3, 4];
    let ptr = data.as_ptr();
    unsafe {
        let misaligned_ptr = ptr as *const u8;
        let _value = *(misaligned_ptr as *const u16);
    }
}
```

Running it under Miri:

`cargo miri run`

Miri's output:

```
error: memory access at offset does not fulfill alignment requirements
 → src/main.rs:7:20
  |
7 | let _value = *(misaligned_ptr as *const u16);
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ memory access with alignment of 1 required for type `u16`, but pointer was 0x… (unaligned)
```

**Explanation** Miri's pointing out that accessing memory at `misaligned_ptr` doesn't meet the required alignment for `u16`.

**Example 4: Borrow Checker's Wild Side**

Sometimes, even Rust's beloved borrow checker needs a boost. Here's an example that should never compile but is worth testing in complex cases:

```rust
fn main() {
    let mut data = 10;
    let r1 = &data;
    let r2 = &mut data;
    println!("r1: {}, r2: {}", r1, r2);
}
```

Rust will throw a compilation error, but if you have trickier borrowing scenarios, Miri helps ensure no sneaky violations slip through.

Miri output:

```
error: borrow stack violated
 → src/main.rs:4:14
  |
4 | let r2 = &mut data;
  | ^^^^^^^^^ mutable borrow conflicts with previous immutable borrow

```

**Explanation** Miri tells us that `r2`'s mutable borrow conflicts with the immutable `r1`.

## 4. Miri Tips and Tricks

1. **Only Use in Debug Mode**: Miri is a strict interpreter, so it's slower than compiled code. 2. **Focus on Unsafe Code**: Since Rust already prevents UB in safe code, Miri shines brightest in unsafe or complex scenarios. 3. **Add Miri to CI**: Miri can add a safety layer to CI for projects that involve low-level or unsafe code, helping to catch UB at every step.

**5. Wrapping Up: Using Miri to Create Bulletproof Rust**

Miri takes Rust's memory safety to new heights, checking each line for anything Rust's compiler missed. It's like a detective that ensures there's no UB hiding in the shadows, waiting to cause chaos. Give it a try in your Rust projects, especially if you're venturing into unsafe code — Rust and Miri will have your back every step of the way.