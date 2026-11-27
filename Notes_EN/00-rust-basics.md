## Before We Begin

This chapter is positioned as "**the minimum necessary knowledge**"; its job is to quickly get the reader started with learning and practicing this tutorial.

Therefore, we will try to cover only the basic syntax needed for subsequent learning, rather than being exhaustive like an encyclopedia, so as to avoid introducing unnecessary cognitive load. When new syntax knowledge is encountered in later chapters, it will be explained together in the accompanying notes.

If you want to learn more comprehensive syntax knowledge, you can read [The Rust Programming Language](https://doc.rust-lang.org/stable/book/).

At the same time, when studying, you do not need to finish this chapter completely; **once you have reached the "module" section in this chapter, you can start directly on the following chapters**. When you encounter unlearned topics in later chapters, come back to this chapter to make up for them. The tutorial will also include "prerequisite knowledge" reminders to ensure that, once you have learned the prerequisites, you can continue with the corresponding chapters.

## Variables

In Rust, data types are represented as follows:

| Type name         | Representation                 |
| ----------------- | ------------------------------ |
| Signed integers   | i8, i16, i32, i64, i128, isize |
| Unsigned integers | u8, u16, u32, u64, u128, usize |
| Floating point    | f32, f64                       |
| Unicode character | char                           |
| Boolean           | bool                           |

### Variable Definition

In Rust, the keyword **`let`** is used to define variables and assign values to them.

**1. Direct assignment**

```rust
let x = 1;  // Rust allows the type to be omitted; the compiler can infer the data type from context
```

**2. Explicitly specifying the type**

```rust
let x:u32 = 1;  
```

**3. Specifying the type via a suffix**

```rust
let x = 1u32;  
```

### static and const

`let` is generally used to define **local variables at runtime**, while **global static variables and constants are defined using `static` and `const`**.

`const`: a compile-time constant

```rust
const MAX: u32 = 100;
const PI: f64 = 3.14159;
```

`static`: a global static variable

```rust
static GREETING: &str = "Hello";
static mut COUNTER: u32 = 0; // access requires unsafe

fn main() {
    println!("{}", GREETING);
    unsafe {
        COUNTER += 1;
    }
}
```

- Initialization must be a constant expression.

- Immutable by default; the keyword `static mut` can make it mutable, but access still requires adding `unsafe`, indicating that you are aware an unsafe operation is being performed (which can easily cause data races).

However, Rust does not recommend using `unsafe`, and it should be avoided as much as possible in practice. In later study, you will encounter safer ways to work with global variables.

### Mutable Variables

In Rust, variables are immutable by default; the following code will produce a compile-time error:

```rust
let x = 1;
x = 2;    // × compile error 
```

For variables that can be modified, Rust requires the keyword **`mut`** to be declared explicitly:

```rust
let mut x = 1;
x = 2;    // √ correct form
```

### Variable Shadowing

Rust allows reusing variable names; this feature is called **shadowing**.

As shown in the following code, when a variable with the same name appears, the new variable shadows the old one until the scope of the new variable ends, at which point the shadowed variable becomes usable again.

```rust
    let a = 1;
    {
        let a = 2;
        println!("{a}");    // prints 2        
    }
    println!("{a}");    // prints 1

    let a = 3;
    println!("{a}");    // prints 3
```

The essence of shadowing is creating a new variable without changing the variable name, so its type or value can be changed.

```rust
    let a = 1;
    println!("{a}");    // prints 1

    let a = "Hello, world!";    
    println!("{a}");    // prints Hello, world!
```

## Functions

All functions in Rust begin with the keyword **`fn`**, and the default entry point of an executable is `main()`.

Function parameters must have their types declared explicitly, and a return value must be indicated with `->` followed by the data type, as shown in the following code:

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b   // equivalent to return a + b;
}

fn main() {
    let result = add(5, 3);
    println!("The sum is: {}", result);
}
```

The last line inside a function, when it does not end with a semicolon, is treated as an expression to be returned, equivalent to `return`.

## Ownership

The ownership feature is designed by Rust to avoid memory-safety problems caused by manual memory management by the programmer. Each piece of data can be held by only one object; this is the ownership of the data.

When data is assigned, passed into a function, or returned, ownership is also transferred (move semantics), and that data becomes invalid and can no longer be used later.

This design is intended to avoid memory-safety problems such as two pointers pointing to the same memory or double freeing. Obviously, it is designed specifically for heap data and pointer data (although the ownership rules apply to all data). Therefore, for scalar data on the stack such as integers, floating-point numbers, and booleans, the change in ownership when they are used is **copy semantics rather than move semantics**, which is equivalent to creating a new variable. Thus, even after scalar data has been used, it can still be used afterward.

Take the following code as an example:

```rust
fn main() {
    let x = 42;          
    let y = x;           // copy semantics, x can still be used
    println!("{} {}", x, y); // √ compiles normally

    let s1 = String::from("hello"); // String is pointer data
    let s2 = s1;         // move semantics, ownership of s1 moves to s2, s1 becomes invalid and cannot be used
    println!("{}", s1);  // × compile error! Using the invalidated s1
}
```

## Modules and Crates

A module (Module): every `.rs` file is itself an implicit module, and within that module file you can use the keyword **`mod`** to define and nest modules.

Modules organize functionally related functions, methods, and data types together, similar to C language header files, and can use the keyword **`pub`** (public) to control their external visibility. By default, code inside a module is private and not visible from the outside.

### Module Definition

An example of module definition is as follows:

```rust
mod math {
    // externally visible function methods
    pub fn add(a: i32, b: i32) -> i32 {
        a + b
    }
}

fn main() {
    let sum = math::add(2, 3);        // call via path
    println!("{}", sum);
}
```

The example above is the definition and use of an internal module within the same `.rs` file. An example of declaring and using an external module is as follows:

```rust
// The project structure has two .rs files
src/
├── main.rs
└── mods.rs

// mods.rs    Just write the module content directly; the .rs file itself is an implicit module
pub fn greet() {
    println!("Hello!");
}

// main.rs
mod mods;    // declare the external module mods

fn main()
{
      mods::greet();     
}
```

### Module Calls

Before using a module, it must be declared with `mod`. When calling its functions, the path must be written clearly. Suppose the mods module also defines a module `mod1`, and `greet()` is a function inside `mod1`; then the call path is `mods::mod1::greet()`.

Obviously, writing the full path every time you make a call is tedious, so Rust also provides the keyword **`use`** to simplify paths.

```rust
// Use mod to declare: tells the compiler to load mods.rs; this step is mandatory, otherwise Rust has no idea that a module named mods exists.
mod mods;

// Use use to bring the module content into the current scope; this step is optional, and you can still call it with the full path without it.
use mods::mod1::greet;

// use can be combined with as to give the module content an alias
use mods::mod1::greet as SAYHELLO;

fn main() {
    // 1. Without use, call with the full path
    mods::mod1::greet();     // works, but the full path must be written every time

    // 2. With use, call directly
    greet();     // simplified path, equivalent to the above

    // 3. With use as, call the alias directly
    SAYHELLO();        // equivalent to the above         
} 
```

### Crate

**A crate is Rust's compilation unit.** **A crate consists of a root file (such as `lib.rs` / `main.rs`) and the several child module files it declares via `mod`.** A crate can be understood as a "dependency package" or a "library in C". But unlike C libraries, which must be downloaded locally before being referenced, in Rust crates are mainly managed and distributed by the official community. To use one, you only need to declare the dependency in `Cargo.toml`, and Cargo will automatically fetch it from the official repository [crates.io](https://crates.io/), resolve the version, compile, and link it.

`mod` is used to **declare and call modules you write locally**, while **external crates do not need to be declared with `mod`**, because they are brought in through the `[dependencies]` section of `Cargo.toml` and can be used directly via `use`.

In embedded development, driver libraries for many common peripheral modules can be found on [crates.io](https://crates.io/) or referenced by declaring them as dependencies, as we will learn in later chapters.

## Control Structures

### Conditional Branches

In Rust, the `if/else` structure is actually an expression, so it can have a return value, while also being usable as a statement in behavior.

In regular usage it is similar to C, but note that the condition must be of type `bool`; you cannot use 0 and 1 as in C.

As for using the return value, see the following code:

```rust
let value = if condition { 1 } else { 2 };    // if the condition holds, return 1 and assign to value; otherwise return 2
```

All branches must evaluate to the same type, otherwise a compile error occurs. Note that `if/else` is itself an expression: the semicolon in `let value = if ... { ... } else { ... };` terminates the `let` statement, not the `if` expression. Without the `let` binding, the same `if` expression can be used as a statement without a trailing semicolon.

### Loops

The loop structures in Rust are `while`, `loop`, and `for`.

Among them, `while` loops are used similarly to C, so no further explanation is needed; `for` is also similar in usage to C, with a traversal range expressed as `A..B`. An example is as follows:

```rust
fn main() {
    for x in 1..10 {
        println!("{}", x); // prints 1-9 in order; 1..10 does not include the last number; to include 10, write the range as 1..=10
    } 
}
```

The `loop` loop, like in most languages, uses `continue` to skip the rest of the current iteration and start the next one directly; it exits upon encountering `break`. However, with `break` you can return an expression as the value of `loop`, as shown below:

```rust
fn main() {
    let mut counter = 0;
    let result = loop {
        counter += 1;
        if counter == 10 {
            break counter * 2; // break out of the loop and carry out the value of counter*2 (20)
        }
    };
    println!("{}", result); // prints 20
}
```

In addition, when dealing with nested `loop` loops, there are loop labels. In multi-level loops, `break` and `continue` apply by default only to the nearest level. If you want to break out of an outer loop, you can use loop labels.

You label a loop with `'label: loop`, and then use `break 'label` or `continue 'label` to precisely control which loop level to break out of, as shown in the following code:

```rust
fn main() {
    let mut x = 0;
    'outer: loop {    // this is the outer loop, labeled with 'outer 
        let mut y = 0;
        loop {    // this is the inner loop
            if y == 5 {
                break 'outer; // break out of the outer loop directly, terminating all loops
            }
            y += 1;
        }
        x += 1;
    }
    println!("Broke out of all loops!");
}
```

## Data Structures

**1. Arrays**

Arrays are defined as shown below. Like variables, they are immutable by default unless `mut` is used. In addition, the `len()` method can be used to obtain the length of an array.

```rust
    let arr : [u8; 3] = [0; 3];        // all array members are 0 and immutable
    let mut arr : [i32; 3] = [1, 2, 3];        
    arr[0] = 0;                
    println!("{}", arr[0]);        // prints 0
    let arr_len = arr.len();
    println!("{}", arr_len);    // prints 3    
```

**2. Tuples**

Tuples are similar to arrays, but can be composed of different data types, as shown below:

```rust
    let person : (String, i32, bool) = ("Alice".to_string(), 30, true);
    let name = person.0; // access by index
    let (name, age, active) = person; // destructuring
```

**3. References (Pointers)**

References in Rust are similar to pointers in C and are denoted by the symbol `&`. As shown below:

```rust
fn main()
{
    let a = 42;
    let b = &a;
    let c = b;

    print!("{} {}", b, c);      // what is passed is a reference (i.e., an address), automatically dereferenced at compile time, prints 42 42  
    print!("{} {}", *b, *c);    // manual dereferencing, prints 42 42
}
```

In Rust, the feature of the `&` reference is that it allows read-only access to data without taking its ownership; this is also called **borrowing**.

References can be divided into **immutable references `&`** and **mutable references `&mut`**. In Rust's borrowing rules, **an immutable reference `&` and a mutable reference `&mut` to a value cannot coexist**. The reason is obvious: to prevent data races caused by reading and writing a value at the same time. This rule makes Rust eliminate this possibility at the syntax level.

For an immutable value, only immutable references are allowed; for a mutable value, one mutable reference or multiple immutable references are allowed.

For a mutable value, taking a mutable reference is essentially the same as a pointer in C. But when taking an immutable reference, the mutable value cannot be modified while the immutable reference exists; it can only be modified after the immutable reference has been used, i.e., after its lifetime ends. As shown in the following code:

```rust
let mut x = 5;
let y = &x;      // y borrows x

// x = 100;     // × compile error: x cannot be modified here because y is still alive

println!("{}", y); // prints 5

// y is not used in the following code; y's lifetime ends, and now x can be modified
x = 100;

// take a new reference y2
let y2 = &x;

println!("{}", y2); // prints 100
```

**Note**: Non-lexical lifetimes (NLL) have been enabled since Rust 1.31 and apply to both the 2015 and 2018 editions. A reference's lifetime ends at its last use, not necessarily at the closing brace of its scope, which is why `x` can be modified after the final `println!("{}", y)` in the example above.

**4. Strings**

The two most commonly used string types in Rust are `&str` and `String`. The former is a string literal/string slice, and the latter is an owned string/dynamic string.

A `String` string is heap-allocated, growable, and owns its data; it requires a memory allocator and can be freely manipulated (adding, deleting, replacing). It is dynamically sized and can grow at runtime.

The `&str` type is obviously a `&` reference; it borrows a string without owning it, so the data is read-only and cannot be manipulated. Its size is fixed and determined at compile time, and it is placed directly in the read-only data segment of the binary, requiring no heap allocation. For embedded `no_std` scenarios, `&str` literals are the absolute mainstay.

```rust
let msg = "message";    // a &str type string
msg.push_str("!");        // × compile error; &str is read-only and cannot be manipulated


let msg = "message".to_string();    // a String type string
let msg = String::from("message");    // equivalent to the previous line
msg.push_str("!");                    // √ compiles normally; a String type string can be modified
```

**5. Structs**

An example of struct code is as follows:

```rust
    struct Person {
        name: String,
        age: u32,
    }

    let a = Person {
        name: String::from("3YJACK"),
        age: 30,
    };
```

In addition to regular structs, there are tuple structs, whose members have no names and which are often used to wrap a single type, as shown below:

```rust
    struct Color(i32, i32, i32); // RGB
    let black = Color(0, 0, 0);
```

Besides data members, structs can also have associated functions, similar to the abstraction in C++, where objects can have properties and methods.

Association of functions is achieved through **`impl` + struct + code block**. Associated functions include methods (with `self` in the parameters) and ordinary associated functions (without `self` in the parameters). As shown in the following code:

```rust
    struct Person {
        name: String,
        age: u32,
    }

impl Person {
    // associated function, does not use self, called via Person::new()
    fn new(name: &str, age: u32) -> Self {
        Person {
            name: String::from(name),
            age,
        }
    }

    // method, uses self, called via person.greet(); here &self is used, i.e., borrowing, data is read-only and ownership does not move
    fn greet(&self) {
        println!("Hello, my name is {} and I am {} years old.", self.name, self.age);
    }

    // method, uses self, called via person.have_birthday(); here &mut self is used, i.e., borrowing, data is modifiable and ownership does not move
    fn have_birthday(&mut self) {
        self.age += 1;
        println!("Happy Birthday! Now I am {} years old.", self.age);
    }

    // method, uses self, called via person.passaway(); here self is used, i.e., a move, data is modifiable and ownership moves to the passaway method; after self leaves the method scope it is automatically freed and can no longer be used
    fn passaway(self) {
        println!("{} has passed away.", self.name);
        }
}
```

**6. Enums**

Enums in Rust are similar to structs, allowing members of different types, and members can carry data.

Enums are often used together with pattern matching `match`, in a way similar to `switch`, as shown in the following code:

```rust
fn main() {
    enum Numbers {
        Zero,
        SmallNumber(u8),
        BiggerNumber(u32),
        EvenBiggerNumber(u64),
    }

    let b = Numbers::SmallNumber(42);
    match b {
        Numbers::Zero => println!("Zero"),
        Numbers::SmallNumber(value) => println!("Small number {value}"),
        Numbers::BiggerNumber(_) | Numbers::EvenBiggerNumber(_) => println!("Some BiggerNumber or EvenBiggerNumber"),
    }

    if matches!(b, Numbers::Zero | Numbers::SmallNumber(_)) {
        println!("Matched Zero or small number");
    }
}
```

## Generics

**Generics are a mechanism of "not specifying a concrete type when writing code, and filling in the concrete type only when it is used."** Its goal is: **one piece of code adapting to multiple types, while maintaining type safety and with no runtime overhead.**

Take a function that compares two values as an example. In C, you write one function per type:

```c
int max_i32(int a, int b) { return a > b ? a : b; }
float max_f32(float a, float b) { return a > b ? a : b; }
unsigned char max_u8(unsigned char a, unsigned char b) { return a > b ? a : b; }
/* ...and again for int16_t, uint32_t, int64_t, double, ... */
```

Code reuse is poor: every type needs its own function. C11 `_Generic` can select one at compile time, but you still have to write and maintain each variant, and a macro-based version loses the type checking that Rust gives you here.

For Rust, when generics can be used:

```rust
fn max<T: PartialOrd>(a: T, b: T) -> T {
    if a > b { a } else { b }
}

// when used
max(3, 5);         // T = i32
max(3.14, 2.71);   // T = f64
max('a', 'z');     // T = char
```

Here `T` is a **type placeholder**, and `<T: PartialOrd>` means "`T` can be any type that implements `PartialOrd` (comparable)".

With generics, not only is code reusable — one set of code can be used for a variety of types — but it is also type-safe and less error-prone.

## Enum Types

Rust also provides some common and practical enum types when writing code. Here we mainly introduce the two most commonly used ones that you will encounter in later study — `Result` and `Option`.

### Result

The `Result` type is a layer of type wrapping around a returned result, defined as follows:

```rust
enum Result<T, E> {
   Ok(T),
   Err(E),
}
```

where `T` is the data type and `E` is the error type.

The `Result` type reflects Rust's strict requirements for code safety. It is commonly used for error handling; its result may be success (Ok) or failure (Err). If successful, it returns a data value of the corresponding type; if failed, it returns an error code of the corresponding type.

It explicitly separates the correct result from the error, and regardless of which result it is, it requires the developer to handle it. Unlike C, where you can continue using a result whether it is correct or erroneous, making it easy to overlook or confuse error information.

The commonly used error-handling methods for the `Result` type are as follows:

1. **`.unwrap()`**

Using `.unwrap()` on a `Result` type is a simple and crude way to handle errors. If the result is a success, the data is taken out; if the result is an error, it **directly crashes the system to terminate the program** and prints the error code information.

2. **`.expect()`**

The `.expect("msg")` method behaves the same as `.unwrap()`, with the difference that when it triggers a system crash, it additionally **prints the specified "msg" message**.

```rust
let value = result.expect("Initialization failed");  // the corresponding log message is printed when the system crashes
```

3. **match matching**

The two results are handled explicitly via `match`.

```rust
match result {
    Ok(v) => { /* ... */ }
    Err(e) => { /* ... */ }
}
```

4. **`?` propagation upward**

`?` is Rust's **error propagation syntax**, appended after an expression that returns `Result` / `Option`.

An expression with `?`, when an error occurs at runtime, returns the error value to the caller of that expression. The error information from the `?` operator can also bubble upward along the call chain until the error information is handled somewhere in the propagation or finally reaches the `main` function, at which point the program terminates and prints the returned error message.

Take the following code as an example:

```rust
fn function_a() -> Result<(), E> {
    ...
    // If function_b() returns an error, function_a() also returns immediately, passing the error result upward
    function_b()?; 
    ...
}

fn main() -> Result<(), E> {
    ...
    // If function_a() returns an error, main() also returns, the program exits and prints the corresponding error result
    function_a()?;
    ...
}
```

If the result returned by `function_b()` is `Err(e)`, then `function_a()` passes that error result upward to its caller, namely `main()`. After `main()` receives the error result, it returns `Err(e)`, and the **Rust runtime** (which can be understood as the "underlying system" responsible for program startup and shutdown) terminates the program and prints the error result returned by `main()`.

`?` propagates errors upward when they occur, but you can also intercept and handle errors during propagation. Note that `?` can only be used inside **functions that return `Result` or `Option`** on **expressions that return `Result` / `Option`**.

### Option

The `Option` type is similar to the `Result` type in that it is also a layer of type wrapping. But while `Result` expresses the success or failure of a result, `Option` expresses **the presence or absence of a value**. It is defined as follows:

```rust
enum Option<T> {
    Some(T),
    None,
}
```

`Option` can be a value of some type, or a null value (null). It is often used in combination with pattern matching to check whether a value exists and take appropriate action. The unwrapping methods learned earlier for `Result` can also be used on `Option`.

When attempting to extract the inner value of an `Option`, the compiler mandates handling the case where the inner value is `None`; otherwise a compile error occurs.

This enum type is used to replace null values in other programming languages, such as the null pointer in C. Compared with C, where null pointers must be handled with great care, Rust's `Option` lets programmers use it safely and with confidence. By making the null value part of the system's data types and requiring explicit handling of the null case, it greatly avoids the problems that null pointers can cause in C.
