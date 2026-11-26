# Learning objectives

Use `esp-generate` to create a project (be sure to enable the embassy async framework) and refer to the `./example/async/embassy_multiprio` example in the `esp-rs/esp-hal` repository. Write the code and complete the priority preemption example.

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
    interrupt::{software::SoftwareInterruptControl, Priority},
    timer::timg::TimerGroup,
    delay::Delay,
};

use embassy_executor::Spawner;
use embassy_time::{Duration, Timer};
use esp_rtos::embassy::InterruptExecutor;
use static_cell::StaticCell;

use log::info;

use esp_backtrace as _;

extern crate alloc;

// This creates a default app-descriptor required by the esp-idf bootloader.
// For more information see: <https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/app_image_format.html#application-description>
esp_bootloader_esp_idf::esp_app_desc!();

// Declare a high-priority async task, marked with the embassy_executor::task attribute
#[embassy_executor::task]
async fn high_prio_task() {
    loop {
        info!("High priority task running!");
        Timer::after(Duration::from_millis(1000)).await;
    }
}

// Declare a low-priority async task, simulating its execution with a blocking wait
#[embassy_executor::task]
async fn low_prio_task() {
    let delay = Delay::new();
    loop {
        info!("Low priority task running!");
        delay.delay_millis(3000);
    }
}

#[allow(
    clippy::large_stack_frames,
    reason = "it's not unusual to allocate larger buffers etc. in main"
)]
#[esp_rtos::main]
async fn main(low_prio_spawner: Spawner){
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

    // Timer 0 serves as Embassy's time source; a software interrupt drives task scheduling
    let timg0 = TimerGroup::new(peripherals.TIMG0);
    let sw_interrupt = SoftwareInterruptControl::new(peripherals.SW_INTERRUPT);
    // Start async scheduling
    esp_rtos::start(timg0.timer0, sw_interrupt.software_interrupt0);

    info!("Embassy initialized!");

    // Create an interrupt executor for handling high-priority tasks; StaticCell is used for static initialization
    static EXECUTOR: StaticCell<InterruptExecutor<2>> = StaticCell::new();
    let executor = InterruptExecutor::new(sw_interrupt.software_interrupt2);
    let executor = EXECUTOR.init(executor);

    // Set the priority and start task scheduling
    let high_prio_spawner = executor.start(Priority::Priority3);
    
    // Attach the high-priority task to the high-priority executor's scheduling queue
    high_prio_spawner.spawn(high_prio_task().expect("Failed to spawn high priority task"));

    // Attach the low-priority task to the low-priority executor's scheduling queue
    low_prio_spawner.spawn(low_prio_task().expect("Failed to spawn low priority task"));

    // for inspiration have a look at the examples at https://github.com/esp-rs/esp-hal/tree/esp-hal-v1.1.0/examples
}
```

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

`low_prio_task` outputs *"Low priority task running!"* every three seconds, and `high_prio_task` outputs *"High priority task running!"* every second. `low_prio_task` simulates being busy by blocking for three seconds; because `high_prio_task` has a higher priority, it can interrupt `low_prio_task`'s blocking and keep running normally.

# Code walkthrough

## Task priority

Rust's async frameworks also have task priorities, but the implementation differs from that of a traditional RTOS. Embassy's priority mechanism essentially **works by creating multiple executor (Executor) instances with different priorities**.

- **One executor, one priority**: each executor instance is assigned a fixed priority.

- **High priority preempts low priority**: a higher-priority executor can **preempt** a lower-priority executor that is running.

- **Within one executor, tasks take fair turns**: inside the same executor, all tasks are still scheduled **cooperatively**, taking turns to run, so no single task monopolizes the CPU.

To implement this preemption, Embassy provides two main executor types:

- **`Executor` (thread mode)**: runs at the lowest priority and typically handles regular tasks in the background.

- **`InterruptExecutor` (interrupt mode)**: runs at a higher interrupt priority and can preempt `Executor`. You can even create multiple `InterruptExecutor`s with different interrupt priorities for finer-grained priority control.

In Embassy's priority design, **the default executor created by `#[esp_rtos::main]` is `Executor`, which has the lowest priority.**

Therefore, high-priority task scheduling is implemented as follows:

**Create an interrupt executor → set the priority and attach the corresponding scheduler → attach tasks of the corresponding priority to join scheduling**

```rust
async fn main(low_prio_spawner: Spawner){
    // Create an interrupt executor for handling high-priority tasks; StaticCell is used for static initialization
    static EXECUTOR: StaticCell<InterruptExecutor<2>> = StaticCell::new();
    let executor = InterruptExecutor::new(sw_interrupt.software_interrupt2);
    let executor = EXECUTOR.init(executor);

    // Set the priority and start task scheduling
    let high_prio_spawner = executor.start(Priority::Priority3);
    
    // Attach the high-priority task to the high-priority executor's scheduling queue
    high_prio_spawner.spawn(high_prio_task().expect("Failed to spawn high priority task"));

    // Attach the low-priority task to the low-priority executor's scheduling queue
    low_prio_spawner.spawn(low_prio_task().expect("Failed to spawn low priority task"));
}
```

 Note that **Embassy does not design a priority inheritance mechanism to avoid priority inversion**, so during development you must proactively avoid this problem through code design.

## StaticCell

In the example source code, creating the interrupt executor uses `StaticCell` as a container for the lazy initialization of a static variable.

**`StaticCell` is a container designed specifically for static initialization.** It occupies only memory space at compile time and is filled with a concrete value at runtime; once initialized, it cannot be initialized again, or the system will crash (this applies only to the `StaticCell` container itself — the data inside the container can still be modified).

This sounds similar to the lazy-initialization approach we learned earlier using `Option` as a placeholder and then replacing it with a concrete value, but the difference is that this method is not specifically meant for static initialization. It is more targeted at scenarios involving shared resources that **may still need to be replaced or modified after the program starts**.

**How to choose?**

- **`StaticCell`**: directly gives you a `&mut T` with no runtime checking overhead when modifying; this reference is unique throughout the program and the resource is exclusively owned. It is only suitable for single-threaded/single-task contexts.

- **`Mutex<RefCell<Option<T>>>`**: to gain modification access you must go through **runtime checks** (`.borrow_mut()` checks whether it is already borrowed, `.lock()` checks whether it has been preempted by another task). The resource can be shared, making it suitable for multi-task concurrent contexts.
