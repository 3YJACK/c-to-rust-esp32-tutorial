# Learning Objectives

Use `esp-generate` to create the project (remember to enable the embassy async framework), and refer to the `./example/async/embassy_multicore` example in the `esp-rs/esp-hal` repository and the official embassy synchronization communication documentation [embassy_sync - Rust](https://docs.rs/embassy-sync/latest/embassy_sync/), write the code and complete the communication example of one-to-one synchronization using `Signal` with LED0 and one-to-many synchronization using `Watch` with LED1 and serial printing.

# Complete Source Code

First, you need to add the corresponding dependency in the terminal with the following command before you can import embassy's synchronization communication module:

```powershell
cargo add embassy-sync
```

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
    interrupt::{software::SoftwareInterruptControl},
    timer::timg::TimerGroup,
    gpio::{Output, Level, OutputConfig, Input, InputConfig, Pull},
};

use embassy_executor::Spawner;
use embassy_time::{Duration, Timer};
use embassy_sync::{     // Import embassy synchronization communication module
    blocking_mutex::raw::CriticalSectionRawMutex, 
    signal::Signal, 
    watch::{self, Watch},
};

use static_cell::StaticCell;

use log::info;

use esp_backtrace as _;

extern crate alloc;

// This creates a default app-descriptor required by the esp-idf bootloader.
// For more information see: <https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/app_image_format.html#application-description>
esp_bootloader_esp_idf::esp_app_desc!();

const WATCH_RCV_NUM: usize = 2;

#[embassy_executor::task]   // Create the button monitoring task, as the sender of watch; sends a signal when the button is pressed
async fn button_monitor(
    button: Input<'static>,
    watch_send: watch::Sender<'static, CriticalSectionRawMutex, bool, WATCH_RCV_NUM>,
) {
    loop {
        if button.is_low() {
            watch_send.send(true);
        }
        Timer::after(Duration::from_millis(100)).await;
    }
}

#[embassy_executor::task]   // Create the watch receiver task, prints information when the button is pressed
async fn rcv0_print(
    mut watch_rev0: watch::Receiver<'static, CriticalSectionRawMutex, bool, WATCH_RCV_NUM>,
) {
    loop {
        if watch_rev0.changed().await { 
            info!("button pressed!");
        }
    }
}

#[embassy_executor::task]   // Create the watch receiver task, toggles the LED level when the button is pressed
async fn rcv1_led(
    mut led1: Output<'static>,
    mut watch_rev1: watch::Receiver<'static, CriticalSectionRawMutex, bool, WATCH_RCV_NUM>
){
    loop {
        if watch_rev1.changed().await {
            led1.toggle();
        }
    }
}


#[embassy_executor::task]   // Create the signal receiver task, waits for a signal to control the LED on/off
async fn led_control(
    mut led: Output<'static>,
    ctrl_signal: &'static Signal<CriticalSectionRawMutex, bool>,
) {
    loop {
        // Wait for the signal until a task sends one
        if ctrl_signal.wait().await {
            led.set_high();
        }
        else {
            led.set_low();
        }
    }
}

#[allow(
    clippy::large_stack_frames,
    reason = "it's not unusual to allocate larger buffers etc. in main"
)]
#[esp_rtos::main]
async fn main(spawner: Spawner){
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

    let timg0 = TimerGroup::new(peripherals.TIMG0);
    let sw_interrupt = SoftwareInterruptControl::new(peripherals.SW_INTERRUPT);

    esp_rtos::start(timg0.timer0, sw_interrupt.software_interrupt0);

    info!("Embassy initialized!");

    let led0 = Output::new(peripherals.GPIO4, Level::Low, OutputConfig::default());

    // Create a signal for controlling the LED on/off
    static LED_CTRL_SIGNAL: StaticCell<Signal<CriticalSectionRawMutex, bool>> = StaticCell::new();
    let ctrl_signal = LED_CTRL_SIGNAL.init(Signal::new());

    spawner.spawn(led_control(led0, ctrl_signal).expect("Failed to spawn led_control task"));

    let config = InputConfig::default().with_pull(Pull::Up);
    let button = Input::new(peripherals.GPIO6, config);

    let led1 = Output::new(peripherals.GPIO7, Level::Low, OutputConfig::default());

    // Create a watch to monitor the button state
    static WATCH: Watch<CriticalSectionRawMutex, bool, WATCH_RCV_NUM> = Watch::new();

    let  watch_send = WATCH.sender();
    let  watch_rev0 = WATCH.receiver().expect("Failed to create watch receiver 0");
    let  watch_rev1 = WATCH.receiver().expect("Failed to create watch receiver 1");

    spawner.spawn(button_monitor(button, watch_send).expect("Failed to spawn button_monitor task"));
    spawner.spawn(rcv0_print(watch_rev0).expect("Failed to spawn rcv0_print task"));
    spawner.spawn(rcv1_led(led1, watch_rev1).expect("Failed to spawn rcv1_led task"));

    loop {
        // Send a signal every 1 second to control the LED on/off
        ctrl_signal.signal(true);
        Timer::after(Duration::from_millis(1000)).await;
        ctrl_signal.signal(false);
        Timer::after(Duration::from_millis(1000)).await;
    }

    // for inspiration have a look at the examples at https://github.com/esp-rs/esp-hal/tree/esp-hal-v1.1.0/examples
}
```

**Pin Connection Reference Table:**

| Peripheral | Corresponding Pin |
| ---- | ----- |
| LED0 | GPIO4 |
| Button | GPIO6 |
| LED1 | GPIO7 |

# Flashing and Running

Use the following command to compile:

```powershell
cargo build 
```

Use the following command to flash and run:

```powershell
cargo espflash flash --monitor
```

**Expected Result:**

LED0 toggles its level once per second, and the main loop task sends a `Signal` once per second to control LED0 to periodically turn on and off; when the button is pressed, the serial port prints log information and LED1 toggles its level. The `button_monitor` task sends a `Watch` to the serial print task and the LED1 control task, performing serial printing and LED1 on/off control respectively.

# Code Explanation

## Signal - Signal

`Signal` is a synchronization communication method used for **one-to-one transfer of a single data item**. When sending, the new value overwrites the old value, making it suitable for state synchronization tasks where the receiver only cares about the latest value.

The definition of `Signal` is as follows:

```rust
pub struct Signal<M, T>
where
    M: RawMutex,
{ /* private fields */ }
```

- **`T`: This is the data type carried by `Signal`**. It can carry only one data item and **only keeps the latest value**.

- **`M`: The "lock" that implements synchronization**: This is a type parameter that must implement the `RawMutex` trait. This "lock" **does not protect your data**; what it protects is the **state machine** inside `Signal` (whether it is `None`, `Waiting`, or `Signaled`). All operations inside `Signal`, such as sending a signal or waiting for a signal, need to first go through this "lock" to safely modify the internal state.

When choosing a lock, passing in `CriticalSectionRawMutex` is the safest, most reliable, and least error-prone choice. It guarantees the atomicity of operations by masking interrupts, so it is suitable for any scenario.

Below is a simple example of creating and using `Signal`:

```rust
static SIGNAL: Signal<CriticalSectionRawMutex, u32> = Signal::new();

// Send a signal carrying data (value: u32)
SIGNAL.signal(value);
// Wait for the signal and return the value carried by the signal
SIGNAL.wait().await;
```

The `.wait()` method is an asynchronous operation that requires `.await`. As we said earlier, asynchronous operations cannot be used in interrupt service routines, so if you want to perform signal synchronization in an interrupt, you should use its synchronous version method `.try_take()`, whose return value is `Option<T>`.

This is a non-blocking method, suitable for interrupts where blocking is not possible and other scenarios where you don't want to block. When `.try_take()` takes the value from `Signal`, `Signal`'s cache is cleared, and a subsequent `.try_take()` will directly return `None`, so you can use it together with the `.signaled()` method to avoid taking an empty value.

```rust
if SIGNAL.signaled(){ // Returns true if the Signal is triggered and the value is updated, false otherwise
    // Try to take the value from Signal; if there is no value in Signal, return None immediately
    SIGNAL.try_take();
}
```

The main methods of `Signal` are as shown above; for more details, you can consult the official documentation.

**Comparison with FreeRTOS:**

Although Embassy's `Signal` is called a signal, it is essentially completely different from FreeRTOS's semaphore `Semaphore`. The essence of a signal is a **latest-data cache, concerned with the specific value of the data**, while the essence of a semaphore is a **resource counter, concerned with the presence or absence of a resource rather than its specific content**. Functionally, a signal is closer to FreeRTOS's task notifications `Task Notifications`.

| Feature Dimension | **Embassy Signal** | **FreeRTOS Task Notifications** |
| --------- | --------------------- | ------------------------------------ |
| **Core Mechanism** | Latest state cache, only keeps the latest value. | A built-in notification array per task, containing state and value. |
| **Data Payload** | Can carry data of **any type** `T`. | Carries a **32-bit unsigned integer** value. |
| **Behavior Semantics** | **Overwrite**, new data unconditionally overwrites old data. | **Flexible and configurable**: supports overwrite, no-overwrite, set-bit, increment and various other update methods. |
| **Number of Consumers** | **Single consumer**, designed for a specific task. | **Single consumer**, the notification is sent directly to the specified task. |
| **Use Cases** | Transferring "latest state" such as sensor readings, state machine states, etc. | Can be used as lightweight binary/counting semaphores, data transfer, and other task synchronization. |

## Watch

Simply put, `Watch` is an upgraded version of `Signal`. It also transfers a single latest value, but `Watch` **supports one-to-many communication**. Its definition is as follows:

```rust
pub struct Watch<M: RawMutex, T: Clone, const N: usize> 
```

The meanings of `M` and `T` are the same as in `Signal`, and `N` refers to the maximum number of data receivers that can be created.

An example of creating `Watch` and its sender and receivers is as follows:

```rust
static WATCH: Watch<CriticalSectionRawMutex, bool, 2> = Watch::new();

let  watch_send = WATCH.sender();
let  watch_rev0 = WATCH.receiver().expect("Failed to create watch receiver 0");
let  watch_rev1 = WATCH.receiver().expect("Failed to create watch receiver 1");
```

The main usage of `Watch` is as follows:

```rust
// Send the watch carrying data
watch_send.send(true);
// Wait for the watch to update and get the latest value
watch_rev0.changed().await;
// Get the current value directly without waiting for a watch update
watch_rev1.get().await; 
// Synchronous non-blocking version of .changed()
watch_rev0.try_changed();
// Synchronous non-blocking version of .get()
watch_rev1.try_get();
```

For more details about `Watch`, you can consult the official documentation.
