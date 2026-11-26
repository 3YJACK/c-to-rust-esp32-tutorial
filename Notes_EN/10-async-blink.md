# Learning objectives

Use `esp-generate` to create a project (be sure to enable the `embassy` async framework during creation) and refer to the `./example/async/multicore` example in the `esp-rs/esp-hal` repository. Write code that implements concurrent serial log output and an LED toggling its level once per second.

# Complete source code

```rust
#![no_std]
#![no_main]
#![deny(
    clippy::mem_forget,
    reason = "mem::forget is generally not safe to do with esp_hal types, especially those \
    holding buffers for the duration of a data transfer."
)]
#![deny(clippy::large_stack_frames)]

use esp_hal::{
    clock::CpuClock,
    interrupt::software::SoftwareInterruptControl,
    timer::timg::TimerGroup,
    gpio::{Output, Level, OutputConfig},
};

use embassy_executor::Spawner;
use embassy_time::{Duration, Timer};

use log::info;

use esp_backtrace as _;

extern crate alloc;

// This creates a default app-descriptor required by the esp-idf bootloader.
// For more information see: <https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/app_image_format.html#application-description>
esp_bootloader_esp_idf::esp_app_desc!();

// Declare an async task, marked with the embassy_executor::task attribute
#[embassy_executor::task]
async fn blink(mut led: Output<'static>) {
    loop {
        led.toggle();
        Timer::after(Duration::from_millis(1000)).await;
    }
}

#[allow(
    clippy::large_stack_frames,
    reason = "it's not unusual to allocate larger buffers etc. in main"
)]
#[esp_rtos::main]
async fn main(spawner: Spawner) -> ! {
    // generator version: 1.3.0
    // generator parameters: --chip esp32s3 -o esp32s3-wroom-1-octal-psram -o unstable-hal -o alloc -o embassy -o stack-smashing-protection -o log -o esp-backtrace -o vscode

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

    let led = Output::new(peripherals.GPIO4, Level::Low, OutputConfig::default());

    // Timer 0 serves as Embassy's time source; a software interrupt drives task scheduling
    let timg0 = TimerGroup::new(peripherals.TIMG0);
    let sw_interrupt = SoftwareInterruptControl::new(peripherals.SW_INTERRUPT);
    // Start async scheduling
    esp_rtos::start(timg0.timer0, sw_interrupt.software_interrupt0);

    info!("Embassy initialized!");

    // Create the async task blink and add it to the executor's scheduling queue
    spawner.spawn(blink(led).expect("Failed to spawn blink task"));

    loop {
        info!("Hello world!");
        Timer::after(Duration::from_secs(1)).await;
    }

    // for inspiration have a look at the examples at https://github.com/esp-rs/esp-hal/tree/esp-hal-v1.1.0/examples
}
```

**Pin connection reference table:**

| Peripheral | Corresponding pin |
| ---- | ----- |
| LED        | GPIO4             |

# Flashing and running

Compile with the following command:

```powershell
cargo build 
```

Flash and run with the following command:

```powershell
cargo espflash flash --monitor
```

**Expected result:**

After flashing and running the program, the serial log outputs *"Hello world!"* once per second, while the LED toggles its level once per second, blinking with a one-second period. The two tasks do not block each other and run asynchronously.

# Code walkthrough

## esp_rtos

The `esp_rtos` appearing in the example source code does not refer to a real-time operating system like FreeRTOS; rather, it is a **task scheduler** specially providing runtime support for `esp-hal` under the async framework. It is an integration component that enables the `embassy` framework to run on the ESP platform, and it does not itself provide traditional RTOS APIs such as task creation.

## embassy

The initialization of async scheduling is as follows; it requires a timer as the time source and a software interrupt for task scheduling.

```rust
    // Timer 0 serves as Embassy's time source; a software interrupt drives task scheduling
    let timg0 = TimerGroup::new(peripherals.TIMG0);
    let sw_interrupt = SoftwareInterruptControl::new(peripherals.SW_INTERRUPT);
    // Start async scheduling
    esp_rtos::start(timg0.timer0, sw_interrupt.software_interrupt0);
```

Creating an async task is as follows: declare the async task with the `#[embassy_executor::task]` attribute, then call `spawner` to add the task to the executor for scheduling.

```rust
#[embassy_executor::task]
async fn blink(mut led: Output<'static>) {
    // .......
}

#[esp_rtos::main]
async fn main(spawner: Spawner) -> ! {
    // ......

    // Create the async task blink and add it to the executor's scheduling queue
    spawner.spawn(blink(led).expect("Failed to spawn blink task"));

    // ......
}
```

In this example, the execution flow of the two async tasks is as follows: after each execution, a task suspends for one second, is awakened only when the timer expires, then triggers a software interrupt and is scheduled by the executor to run again. The two tasks run independently and do not block each other.

```
main:
    Output Hello world!
    Wait 1 second, suspend temporarily

blink:
    Toggle GPIO4
    Wait 1 second, suspend temporarily

After 1 second:
    main is awakened and outputs the log
    blink is awakened and toggles the LED
    Both enter waiting again
```

## The 'static lifetime

In the async task `blink`, the parameter passed in is `mut led: Output<'static>`. If it were changed to `mut led: Output`, compilation would fail. Here, `'static` denotes a lifetime, indicating that `led` lives for the entire lifetime of the program.

Requiring task arguments to have the `'static` lifetime is not a mandate of the Rust language itself; it is **a core and necessary characteristic of `Embassy`'s executor design**.

At program startup, the `main` function creates tasks and adds them to scheduling. However, **the `main` function itself returns**, and its local variables become invalid as a result. Yet the spawned tasks may continue running independently in the background. This means **a task's lifetime is completely independent of the context that spawned it**.

If a task holds a reference to a local variable in `main`, then after `main` returns that variable will be destroyed while the task still lives — creating a **dangling reference**, a memory-safety problem strictly forbidden in Rust.

In the case of a semantic move, this problem does not occur. But in the case of borrowing, there is a risk of a dangling reference.

Therefore `embassy` requires all task functions to have the `'static` lifetime. This ensures **that any data owned by a task will not be destroyed during its execution.**

## The "binding" nature of mutability

Some readers may wonder: since `led` is defined as immutable, why can it be passed to a function that requires a mutable value as its parameter?

This is a concept that is easy to confuse when learning Rust. The key is that **mutability is an external property of a binding, not an intrinsic property of the value itself**.

When `led` is defined in `main`, it is bound with an immutable attribute, so no mutable operations can be performed on it inside `main`. But when it undergoes a semantic move into the `blink` task, the old immutable attribute is "unbound", and it can be redeclared as mutable inside `blink`. **Whenever a value undergoes a semantic move, its mutability can be redeclared.** In the case of borrowing, however, strict rule restrictions apply.
