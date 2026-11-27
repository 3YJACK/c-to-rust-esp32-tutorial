# Learning Objectives

Use `esp-generate` to create a project and refer to the `./example/interrupt/gpio` example in the `esp-rs/esp-hal` repository to write code and implement a simple button interrupt that controls an LED turning on and off.

**Prerequisites:**

It is recommended to study this content after mastering **<u>references</u>** in the **<u>data structures</u>** section and **<u>Option</u>** of the **<u>enum types</u>** in [Rust basics](./00-rust-basics.md).

# Complete Source Code

```rust
#![no_std]
#![no_main]
#![deny(
    clippy::mem_forget,
    reason = "mem::forget is generally not safe to do with esp_hal types, especially those \
    holding buffers for the duration of a data transfer."
)]
#![deny(clippy::large_stack_frames)]

use core::cell::RefCell;

use critical_section::Mutex;

use esp_hal::{
    clock::CpuClock,
    main,
    time::{Duration, Instant},
    gpio::*,
    handler,
    ram,
    delay::*,
};

use log::info;

use esp_backtrace as _;

extern crate alloc;

// This creates a default app-descriptor required by the esp-idf bootloader.
// For more information see: <https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/app_image_format.html#application-description>
esp_bootloader_esp_idf::esp_app_desc!();

// To make it accessible to the interrupt handler, define it as a global variable, then wrap it in a Mutex critical-section mutex and a RefCell interior mutability wrapper, and also use Option to wrap the Input type for lazy initialization
// Because a static variable has a fixed value at compile time, while the instantiation of an Input object can only be completed at runtime, first wrap it in Option and assign None as a placeholder, then replace it after instantiation is complete
static BUTTON: Mutex<RefCell<Option<Input>>> = Mutex::new(RefCell::new(None));

static LED: Mutex<RefCell<Option<Output>>> = Mutex::new(RefCell::new(None));

#[allow(
    clippy::large_stack_frames,
    reason = "it's not unusual to allocate larger buffers etc. in main"
)]
#[main]
fn main() -> ! {
    // generator version: 1.3.0
    // generator parameters: --chip esp32s3 -o esp32s3-wroom-1-octal-psram -o unstable-hal -o alloc -o stack-smashing-protection -o log -o esp-backtrace -o vscode

    esp_println::logger::init_logger_from_env();

    let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
    let peripherals = esp_hal::init(config);

    // The following pins are used to bootstrap the chip. They are available
    // for use, but check the datasheet of the module for more information on them.
    // - GPIO0
    // - GPIO3
    // - GPIO45
    // - GPIO46
    // These GPIO pins are in use by some feature of the module and should not be used.
    let _ = peripherals.GPIO27;
    let _ = peripherals.GPIO28;
    let _ = peripherals.GPIO29;
    let _ = peripherals.GPIO30;
    let _ = peripherals.GPIO31;
    let _ = peripherals.GPIO32;
    let _ = peripherals.GPIO33;
    let _ = peripherals.GPIO34;
    let _ = peripherals.GPIO35;
    let _ = peripherals.GPIO36;
    let _ = peripherals.GPIO37;

    esp_alloc::heap_allocator!(#[esp_hal::ram(reclaimed)] size: 73744);

    // Create an IO pin manager and set the interrupt handler to the handler function
    let mut io = Io::new(peripherals.IO_MUX);
    io.set_interrupt_handler(handler);

    let mut led = Output::new(peripherals.GPIO4, Level::Low, OutputConfig::default());

    let config = InputConfig::default().with_pull(Pull::Up);
    let mut button = Input::new(peripherals.GPIO6, config);

    // Critical section protection
    critical_section::with(|cs| {
        // Set the button pin to trigger an interrupt on the falling edge
        button.listen(Event::FallingEdge);
        // Acquire BUTTON's critical-section mutex and borrow a mutable reference
        BUTTON.borrow_ref_mut(cs)   
              .replace(button);   // Replace the BUTTON global variable with the instantiated button object so the interrupt handler can access the button pin

        LED.borrow_ref_mut(cs)
           .replace(led);   
    });

    let delay = Delay::new();

    loop {
        info!("Waiting for button press...");
        delay.delay_millis(5000);
    }

    // for inspiration have a look at the examples at https://github.com/esp-rs/esp-hal/tree/esp-hal-v1.1.0/examples
}

#[handler]
// The interrupt handler must use the #[ram] attribute to place it in RAM so that it can respond quickly when an interrupt occurs
#[ram]
fn handler() {
    critical_section::with(|cs| {
        // borrow_ref_mut returns the RefMut<Option<Input>> type, which is a smart pointer to an Option,
        // the as_mut() method extracts Option<&mut Input> from RefMut<Option<Input>>,
        // the expect() method unwraps the Option; if the Option is Some, it returns the contained value &mut Input, if it is None, it will panic. 
        let mut button_ref = BUTTON.borrow_ref_mut(cs); 
        let button = button_ref.as_mut()
                            .expect("Button not initialized");

        if button.is_interrupt_set() {
            LED.borrow_ref_mut(cs)
                .as_mut()
                .expect("LED not initialized")
                .toggle();

            info!("Button was the source of the interrupt");
        } 

        // Clear the interrupt flag
        button.clear_interrupt();
    });
}
```

**IO Connection Reference Table:**

| Peripheral | Corresponding IO |
| ---- | ----- |
| Button | GPIO6 |
| LED | GPIO4 |

# Flashing and Running

Use the following command to build:

```powershell
cargo build 
```

Use the following command to flash and run:

```powershell
cargo espflash flash --monitor
```

**Expected result:**

After a successful flash, the log message *"Waiting for button press..."* is output every 5 seconds. After pressing the button, the LED's level state toggles, and at the same time the log message *"Button was the source of the interrupt"* is output.

# Code Explanation

## RefCell and Interior Mutability

In Rust's borrowing rules, **a value can have either multiple immutable references `&` or one mutable reference `&mut`, but the two kinds of references cannot coexist**.

But in actual development, a piece of data is often held and operated on by multiple modules. As in the button interrupt example in this article, the button's value is both created and modified in the main function and continuously read in the interrupt service function. Because an interrupt can occur at any time, from the compiler's perspective the two may coexist, which violates Rust's borrowing rules and cannot pass compilation. Therefore we need to use `RefCell` to work around this rule restriction.

```rust
use core::cell::RefCell;
```

`RefCell` changes this rule from **compile time** to **runtime** (only for values of this type, not globally). Since compile-time static checks cannot handle this kind of asynchronous concurrent scenario, read/write synchronization of the data is maintained at runtime; if the rules are violated at runtime, the program will directly `panic` and crash.

Its working mechanism is as follows:

`RefCell` provides two main methods:

- **`.borrow()`**: requests "read permission". Multiple may exist at the same time (because they only read and do not write).

- **`.borrow_mut()`**: requests "write permission". Only one may exist.

When you call these two methods, `RefCell` internally maintains a **counter**:

- Records how many reads (`borrow`) are currently active.

- Or whether there is one write (`borrow_mut`) active.

**Runtime rules:**

When requesting `borrow_mut()`, if the counter shows that a `borrow` or `borrow_mut` is already occupying, `RefCell` will immediately trigger a `panic!`.

`RefCell` **hides mutable permission inside the data type**, allowing a value declared immutable from the outside to be modified internally; therefore this feature is also called **interior mutability**.

Here is a simple example:

```rust
use core::cell::RefCell;

fn main() {
    let data = RefCell::new(5); // data is immutable from the outside but mutable internally

    // 1. Multiple simultaneous reads of data (allowed)
    let r1 = data.borrow();
    let r2 = data.borrow();
    println!("Read values: {} {}", r1, r2);
    // r1, r2 go out of scope, releasing the read state

    // 2. Request write permission (allowed, data is neither being read nor has a unique write permission granted)
    let mut w = data.borrow_mut();
    *w += 1; // modify the internal value
    println!("After modification: {}", w);
    // w goes out of scope, releasing write permission (in real projects it is recommended to mark the scope explicitly with {})

    // 3. Dangerous operation (runtime crash!)
    let r3 = data.borrow();     // in read state
    let w2 = data.borrow_mut(); // attempting to request write permission
    // the program crashes directly when it reaches here
}
```

## Mutex - Mutual Exclusion Lock

For our button interrupt example, RefCell only solves the syntactic restriction at static compile time, rather than truly avoiding read/write conflicts in the program, because an interrupt may break the atomicity of read/write operations. Therefore we also need to use the mutex from the critical-section module:

```rust
use critical_section::Mutex;
```

The mutex masks global interrupts within the critical section, thereby avoiding concurrent preemption of the same data by the main function and the interrupt.

**How to use a critical section:**

Simply place the code block that needs critical-section protection inside `critical_section::with()`.

```rust
critical_section::with(|cs| {
    // the code should be protected by critical_section 
});
```

Here `|cs|` is the critical-section token, used to guarantee that data access occurs while in the critical-section state. Therefore all methods that attempt to "access" or "modify" internal data protected by the critical section need to be passed `|cs|`.

## Option and Lazy Initialization

Besides handling possible null values, `Option` can also be used for lazy initialization.

In Rust, global variables are declared with `static` and require their values to be determined at compile time. Therefore, for global variables whose actual value can only be determined at runtime, you can use `Option`'s null value as a placeholder to pass compilation, then use the `replace` method at runtime to replace it, passing in the value to replace with, as shown in this article's example:

```rust
static BUTTON: Mutex<RefCell<Option<Input>>> = Mutex::new(RefCell::new(None));
static LED: Mutex<RefCell<Option<Output>>> = Mutex::new(RefCell::new(None));

fn main() -> ! {
    critical_section::with(|cs| {
        BUTTON.replace(button);
        LED.replace(led);  
    });
}
```

## interrupt - Interrupts

The interrupt development process is as follows:

**Register the interrupt → configure the interrupt → write the interrupt service function**

1. **Register the interrupt**

First create an IO manager. Because of the ESP32's hardware characteristic — all GPIOs share a single interrupt source. Therefore, in `esp-hal`, pin interrupts are designed to be managed centrally by the IO manager; the interrupt service function must be set through the IO manager, and inside the function `is_interrupt_set()` is used as the interrupt flag to determine and dispatch the specific handling logic.

```rust
  // Create an IO pin manager and set the interrupt handler to the handler function
  let mut io = Io::new(peripherals.IO_MUX);
  io.set_interrupt_handler(handler);
```

2. **Interrupt configuration**

Set the trigger condition of the interrupt, complete the lazy initialization replacement of the global variables that the interrupt needs to access, and so on, as shown in this article's example code:

```rust
   critical_section::with(|cs| {
        // Set the button pin to trigger an interrupt on the falling edge
        button.listen(Event::FallingEdge);
        // Acquire BUTTON's critical-section mutex and borrow a mutable reference
        BUTTON.borrow_ref_mut(cs)
        // Replace the BUTTON global variable with the instantiated button object so the interrupt handler can access the button pin   
              .replace(button);   

        LED.borrow_ref_mut(cs)
           .replace(led);   
    });
```

3. **Interrupt service function**

In Rust, interrupt service functions are usually marked with the `#[interrupt]` attribute, but in the `esp-hal` library this is wrapped into a more integrated and more powerful `#[handler]` attribute; for example, you can directly specify the interrupt priority in the attribute #`[handler(priority = esp_hal::interrupt::Priority::Priority2)]`.

In addition, `esp-hal` also provides the `#[ram]` attribute, used to store the interrupt service function's code in faster-to-read RAM, whereas by default code is stored in Flash, which may cause serious problems for time-sensitive tasks such as interrupt service functions.

```rust
#[handler]
// The interrupt handler must use the #[ram] attribute to place it in RAM so that it can respond quickly when an interrupt occurs
#[ram]
fn handler() {
    critical_section::with(|cs| {
        // borrow_ref_mut returns the RefMut<Option<Input>> type, which is a smart pointer to an Option,
        // the as_mut() method extracts Option<&mut Input> from RefMut<Option<Input>>,
        // the expect() method unwraps the Option; if the Option is Some, it returns the contained value &mut Input, if it is None, it will panic. 
        let mut button_ref = BUTTON.borrow_ref_mut(cs); 
        let button = button_ref.as_mut()
                            .expect("Button not initialized");

        if button.is_interrupt_set() {
            LED.borrow_ref_mut(cs)
                .as_mut()
                .expect("LED not initialized")
                .toggle();

            info!("Button was the source of the interrupt");
        } 

        // Clear the interrupt flag
        button.clear_interrupt();
    });
}
```

In the interrupt service function, the `.is_interrupt_set()` method is used to determine whether the interrupt on the corresponding pin has been triggered. After executing the interrupt service, remember to use the `.clear_interrupt()` method to clear the interrupt flag before exiting the function, to avoid repeated interrupts.

## RefMut Guard

In the interrupt service function code, LED is manipulated only once, so it can be chained and discarded after use, whereas BUTTON needs both to query the interrupt flag and to clear the flag. If written as a call chain, it requires borrowing multiple times, which is somewhat cumbersome:

```rust
if BUTTON.borrow_ref_mut(cs)
            .as_mut()   
            .unwrap()
            .is_interrupt_set()
{
   // ......             
}

    BUTTON.borrow_ref_mut(cs)
        .as_mut()   
        .unwrap()
        .clear_interrupt()            
```

If you want to borrow once and operate multiple times, you need to save the mutable reference to `BUTTON` after borrowing by assigning it to a variable.

At this point you may wonder why the source code assigns values separately twice instead of assigning to `button` once as in the code shown below?

```rust
let button = BUTTON.borrow_ref_mut(cs)
                    .as_mut()
                    .expect("Button not initialized");

if button.is_interrupt_set() {
        LED.borrow_ref_mut(cs)
            .as_mut()
            .expect("LED not initialized")
            .toggle();

        info!("Button was the source of the interrupt");
} 

    // Clear the interrupt flag
    button.clear_interrupt();
```

This is because `BUTTON.borrow_ref_mut()` returns data of type `RefMut`, and `RefMut` is a **guard** that internally contains a reference to the `BUTTON` data. What `.as_mut()` obtains is the reference to the data inside the `RefMut` guard, which is the data we want to use.

Although `button` in the example above retains this reference, **the lifetime of this reference is bound to the lifetime of the guard**. The guard is released immediately after the call chain ends, and the reference held by `button` becomes invalid along with it. But `button.clear_interrupt()` is called afterward, so the compiler reports an error "*creates a temporary value which is freed while still in use*", which roughly means that the already-freed temporary value `button` was used.

Therefore, you first need an intermediate value to save the guard, extending its lifetime from the current statement to the current code block, and then extract its internal mutable reference and assign it to an object. In this way, both the guard and the reference to its internal data have sufficient lifetimes, naturally enabling a single borrow of the internal data with multiple operations.

Earlier we said that `RefCell` defers borrow checking from compile time to runtime, and `RefMut` is the guard responsible for this runtime check. Internally it not only contains an interior mutable reference to the data, but also maintains a borrow counter, through which references to the internal data are guaranteed to comply with the borrowing rules. Therefore the data reference must be bound to the guard's lifetime. **While the guard exists, no other place may borrow this data; when the guard goes out of scope, it is destroyed, the borrow counter is reset to zero, and the reference should become invalid along with it.** Otherwise, if the guard disappeared but the reference remained, a mutable reference might be requested elsewhere, and then there would be two mutable references, which clearly violates Rust's borrowing rules.
