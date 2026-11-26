# Learning Objectives

Create a project using `esp-generate`. Since the `esp-rs/esp-hal` repository does not include any timer-related demos in its examples, this sample code was written by referring to the official documentation [esp_hal - Rust](https://docs.espressif.com/projects/rust/esp-hal/1.1.0/esp32s3/esp_hal/index.html), and implements timer interrupts and PWM output functionality.

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
    main,
    handler,
    ram,
    Blocking,
    clock::CpuClock,
    gpio::{Output, Level, OutputConfig, DriveMode},
    time::{Duration, Rate},
    timer::{PeriodicTimer, timg::TimerGroup},
    ledc::{self, Ledc, timer::{self, TimerIFace}, channel::{self, ChannelIFace}},
};

use esp_backtrace as _;
use log::info;

extern crate alloc;

// This creates a default app-descriptor required by the esp-idf bootloader.
// For more information see: <https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/app_image_format.html#application-description>
esp_bootloader_esp_idf::esp_app_desc!();

static LED: Mutex<RefCell<Option<Output>>> = Mutex::new(RefCell::new(None));
static TIMER: Mutex<RefCell<Option<PeriodicTimer<Blocking>>>> = Mutex::new(RefCell::new(None)); // Fill Blocking or Async inside PeriodicTimer; Blocking means blocking mode, Async means asynchronous mode

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

    // First obtain a timer group
    let timg0 = TimerGroup::new(peripherals.TIMG0);
    // Then instantiate a general-purpose timer of the timer group as a periodic timer
    let mut prd_timer = PeriodicTimer::new(timg0.timer0);
    // Set the interrupt handler to the timer_handler function
    prd_timer.set_interrupt_handler(timer_handler);

    // Initialize LED1 to control its blinking via the timer interrupt
    let mut led1 = Output::new(peripherals.GPIO4, Level::Low, OutputConfig::default());

    critical_section::with(|cs| {
        LED.borrow_ref_mut(cs)
            .replace(led1);

        // Listen for timer interrupts
        prd_timer.listen();
        // Set the timer period to 1000 ms and start the timer
        prd_timer.start(Duration::from_millis(1000));
        TIMER.borrow_ref_mut(cs)
            .replace(prd_timer);
    });

    // Create LED2 for PWM control of its brightness, using the LEDC peripheral
    let mut led2 = Ledc::new(peripherals.LEDC);
    // According to the official documentation, the clock source used by LEDC is APB 
    led2.set_global_slow_clock(ledc::LSGlobalClkSource::APBClk);

    // Create a low-speed timer instance for PWM control of LED2
    let mut pwm_timer = led2.timer::<ledc::LowSpeed>(ledc::timer::Number::Timer0);
    // The duty bit count and frequency only affect the breathing-light effect, not its implementation, so there is no need to obsess over the specific configuration parameters
    pwm_timer.configure(ledc::timer::config::Config {
            duty: ledc::timer::config::Duty::Duty8Bit,
            clock_source: ledc::timer::LSClockSource::APBClk,
            frequency: Rate::from_hz(1000),
        }).expect("Failed to configure PWM timer");

    // Configure LED2's GPIO6 pin as a PWM output channel
    let mut pwm_channel = led2.channel(channel::Number::Channel0, peripherals.GPIO6);
    pwm_channel.configure(channel::config::Config {
            timer: &pwm_timer,
            duty_pct: 0,
            drive_mode: DriveMode::PushPull,
        }).expect("Failed to configure PWM channel");

    info!("PROGRAM RUNNING...");
    loop {
        pwm_channel.start_duty_fade(0, 100, 1000).unwrap();
        while pwm_channel.is_duty_fade_running() {}

        pwm_channel.start_duty_fade(100, 0, 1000).unwrap();
        while pwm_channel.is_duty_fade_running() {}
    }

    // for inspiration have a look at the examples at https://github.com/esp-rs/esp-hal/tree/esp-hal-v1.1.0/examples
}

#[handler]
#[ram]
fn timer_handler() {
    critical_section::with(|cs| {
        LED.borrow_ref_mut(cs)
            .as_mut()
            .expect("LED not initialized")
            .toggle();

        TIMER.borrow_ref_mut(cs)
            .as_mut()
            .expect("Timer not initialized")
            .clear_interrupt();
    });
    info!("Timer interrupt triggered, LED toggled.");
}
```

**Pin Connection Reference Table:**

| Peripheral                            | Corresponding Pin |
| ------------------------------------- | ----------------- |
| LED1 (controlled by timer interrupt)  | GPIO4             |
| LED2 (controlled by PWM)              | GPIO6             |

# Flashing and Running

Compile using the following command:

```powershell
cargo build 
```

Flash and run using the following command:

```powershell
cargo espflash flash --monitor
```

**Expected Result:**

According to the pin connection reference table, connect the two LEDs to the corresponding GPIOs, then flash and run the code. You should see one LED toggle its level once per second, while the log prints *"Timer interrupt triggered, LED toggled."*, and the other LED shows a breathing-light effect, alternating between bright and dim.

# Code Explanation

## Timers and Their Interrupts

There are two types of timers in `esp-hal` — `OneShotTimer` and `PeriodicTimer`, namely one-shot timers and periodic timers. Timers are managed by the timer group `timg`. For example, in the `ESP32S3`, one timer group consists of two general-purpose timers and one watchdog timer. Therefore, the process of creating a timer is as follows:

```rust
    // First obtain a timer group
    let timg0 = TimerGroup::new(peripherals.TIMG0);
    // Then instantiate a general-purpose timer of the timer group as a periodic timer
    let mut prd_timer = PeriodicTimer::new(timg0.timer0);
```

 Then it is just a matter of configuring it and writing the timer interrupt function:

```rust
    // Set the interrupt handler to the timer_handler function
    prd_timer.set_interrupt_handler(timer_handler);    
    // Listen for timer interrupts
    prd_timer.listen();
    // Set the timer period to 1000 ms and start the timer
    prd_timer.start(Duration::from_millis(1000));

    #[handler]
    #[ram]
    fn timer_handler() {
    // ......
    }
```

## LEDC and MCPWM

There are two main PWM peripherals in `esp-hal` — `LEDC` and `MCPWM`. The former is a general-purpose PWM module designed for simple PWM applications (such as LED control), while the latter is a powerful, more complex dedicated PWM module designed for motor control and power applications. (Note: not all ESP chips support `MCPWM`; you need to confirm this with the chip datasheet and the official documentation.)

This example only demonstrates `LEDC`; for `MCPWM`, you can refer to the official documentation and implement it yourself.

The implementation flow of LEDC is: **create → configure the timer and PWM output channel → apply**. As shown below, the code is fairly simple, so there is not much to explain here. Note that the methods related to `LEDC`'s `channel` are defined in `ChannelIFace` of its `Trait` module.

```rust
    // Create LED2 for PWM control of its brightness, using the LEDC peripheral
    let mut led2 = Ledc::new(peripherals.LEDC);
    // According to the official documentation, the clock source used by LEDC is APB 
    led2.set_global_slow_clock(ledc::LSGlobalClkSource::APBClk);

    // Create a low-speed timer instance for PWM control of LED2
    let mut pwm_timer = led2.timer::<ledc::LowSpeed>(ledc::timer::Number::Timer0);
    // The duty bit count and frequency only affect the breathing-light effect, not its implementation, so there is no need to obsess over the specific configuration parameters
    pwm_timer.configure(ledc::timer::config::Config {
            duty: ledc::timer::config::Duty::Duty8Bit,
            clock_source: ledc::timer::LSClockSource::APBClk,
            frequency: Rate::from_hz(1000),
        }).expect("Failed to configure PWM timer");

    // Configure LED2's GPIO6 pin as a PWM output channel
    let mut pwm_channel = led2.channel(channel::Number::Channel0, peripherals.GPIO6);
    pwm_channel.configure(channel::config::Config {
            timer: &pwm_timer,
            duty_pct: 0,
            drive_mode: DriveMode::PushPull,
        }).expect("Failed to configure PWM channel");

    loop {
        pwm_channel.start_duty_fade(0, 100, 1000).unwrap();
        while pwm_channel.is_duty_fade_running() {}

        pwm_channel.start_duty_fade(100, 0, 1000).unwrap();
        while pwm_channel.is_duty_fade_running() {}
    }    
```

## About Imports

Unlike the previous examples, which simply and mindlessly imported all members of a module via the wildcard `*`, this example explicitly and clearly imports the modules used and their members. This is a more standard-compliant way of importing.

```rust
use esp_hal::{
    main,
    handler,
    ram,
    Blocking,
    clock::CpuClock,
    gpio::{Output, Level, OutputConfig, DriveMode},
    time::{Duration, Rate},
    timer::{PeriodicTimer, timg::TimerGroup},
    ledc::{self, Ledc, timer::{self, TimerIFace}, channel::{self, ChannelIFace}},
};
```

It should also be pointed out that there is a common misconception about wildcard imports — mistakenly believing that wildcard imports are recursive. This is also a mistake I made when writing this sample tutorial.

In the early stages of building this example, for the import style I still used wildcard imports as before, as shown in the code below. However, an unexpected error occurred during compilation: the compiler reported that the struct `TimerGroup` could not be found.

```rust
use  esp_hal::{
    // ......
    timer::*,
};

// Compilation error
fn main(){
    let timg0 = TimerGroup::new(peripherals.TIMG0);
}
```

By reading the official documentation or the source code of `esp_hal`, we can see that the directory structure of the `timer` module is as follows:

```
src/
└── timer/
    ├── mod.rs      // module entry
    └── timg.rs     // submodule, defines TimerGroup
    └── ....        // other submodules
```

`TimerGroup` is indeed an internal member of the `timer` module, except that it is inside the `timg` module. So why does the compiler report an error?

After changing the import style to the code shown below, compilation passed smoothly.

```rust
use  esp_hal::{
    // ......
    timer::{*, timg::*},
};   

fn main(){
    // Compilation successful
    let timg0 = TimerGroup::new(peripherals.TIMG0);
}
```

So it can be inferred that the cause lies in an incorrect understanding of wildcard imports. I had assumed wildcards were recursive by default, but in reality, wildcard imports may import all public members of the `timer` module and the public submodules it imports, but do not recursively import the public members inside the submodule `timg`. Therefore, the `TimerGroup` struct cannot be used directly. Based on this inference, it can also be deduced that the code shown below should also compile:

```rust
use  esp_hal::{
    // ......
    timer::*,
};

fn main() {
    // Compilation successful
    let timg0 = timg::TimerGroup::new(peripherals.TIMG0);
}
```

After trying to compile, it turned out to be exactly the case. Combined with information found online, we can confirm that the inference is correct — **Rust's wildcard imports are non-recursive**.

Many people learning Rust may, like me, mistakenly believe that wildcard imports are recursive, after all, in some other programming languages import behavior is indeed recursive.

But in Rust, **wildcard imports only import the direct public members of the current module and do not recursively expand the internal members of its submodules.**

This is a deliberate design rule of the Rust language, intended to avoid dumping all the submodule members inside a library wholesale into the current namespace. Because this would cause two problems: first, the **probability of naming conflicts increases greatly**, and you have no idea what exactly `*` has brought in; second, the **readability and maintainability of the code decrease**, since readers may not be able to tell at a glance which submodule a name comes from just by looking at it.

Therefore, in Rust coding conventions, one should **try to avoid using simplistic, mindless import styles like wildcard imports**, and instead **explicitly and clearly write out the full import paths of the members used**, just as in the import example finally adopted in this article.

By the way, let's also talk about the differences between importing the module itself (`self`), module members, and wildcards, taking the `ledc` module as an example:

| Feature               | **Module itself (`self`)**                                 | **Specific member (`Item`)**                                    | **Wildcard (`*`)**                                        |
| --------------------- | ------------------------------------------------- | ---------------------------------------------------- | ---------------------------------------------------- |
| **Syntax example**     | `ledc::{self}`                                    | `ledc::Ledc`                                         | ` ledc::*`                                           |
| **Content brought into scope** | The module itself                                              | A specific member within the module                                             | All public members within the module                                           |
| **How to call in code**  | Must be called via the module name<br/>`ledc::Ledc::xxx`✅<br/>`Ledc::xxx`❌ | Can be called via the name of the imported member within the module<br/>`Ledc::xxx`✅<br/> `timer::xxx`❌ | Can be called directly via any public member name within the module<br/>`Ledc::xxx`✅<br/>`timer::xxx`✅ |
| **Risk of naming conflicts**   | ✅ **Very low** (must include the module prefix, path is clear)                            | ⚠️ **Medium** (conflicts occur if types with the same name are imported, e.g., importing the `timer` module into itself would cause a conflict)        | ❌ **Very high** (very easy to pollute the namespace)                                   |
| **Code readability**    | ✅ **High** (readers can immediately see the reference path)                              | ⚠️ **Medium** (need to look at the `use` at the top to determine the source)                         | ❌ **Low** (difficult to tell which module it is referenced from)                                 |

In actual practice, you should choose the appropriate import style according to the actual scenario!
