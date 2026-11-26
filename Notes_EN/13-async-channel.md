# Learning Objectives

Use `esp-generate` to create the project (remember to enable the embassy async framework), and refer to the official embassy synchronization communication module documentation [embassy_sync - Rust](https://docs.rs/embassy-sync/latest/embassy_sync/), write the code and use `Channel` to complete an application example of many-to-one synchronization communication.

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
    timer::timg::TimerGroup,
    gpio::{Input, InputConfig, Pull},
};

use embassy_executor::Spawner;
use embassy_time::{Duration, Timer};
use embassy_sync::{     // Import embassy synchronization communication module
    blocking_mutex::raw::CriticalSectionRawMutex, 
    channel::{Channel, Receiver, Sender},
};

use log::info;

use esp_backtrace as _;

extern crate alloc;


// This creates a default app-descriptor required by the esp-idf bootloader.
// For more information see: <https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/app_image_format.html#application-description>
esp_bootloader_esp_idf::esp_app_desc!();

const CHANNEL_CAPACITY: usize = 10;

#[embassy_executor::task]
async fn print_task(
    channel_receiver: Receiver<'static, CriticalSectionRawMutex, &'static str, CHANNEL_CAPACITY>,
) {
    loop {
        let msg = channel_receiver.receive().await;
        info!("{}", msg);
    }
}

#[embassy_executor::task]
async fn button_sender(
    button: Input<'static>,
    channel_sender: Sender<'static,CriticalSectionRawMutex, &'static str, CHANNEL_CAPACITY>,
) {
    loop {
        if button.is_low() {
            channel_sender.send("message from sender1").await;
        }
        Timer::after(Duration::from_millis(100)).await;
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

    let timg0 = TimerGroup::new(peripherals.TIMG0);
    let sw_interrupt =
    esp_hal::interrupt::software::SoftwareInterruptControl::new(peripherals.SW_INTERRUPT);
    esp_rtos::start(timg0.timer0, sw_interrupt.software_interrupt0);

    info!("Embassy initialized!");

    static CHANNEL: Channel<CriticalSectionRawMutex, &'static str, CHANNEL_CAPACITY> = Channel::new();

    let channel_sender0 = CHANNEL.sender();
    let channel_sender1 = CHANNEL.sender();
    let channel_receiver = CHANNEL.receiver();

    let config = InputConfig::default().with_pull(Pull::Up);
    let button = Input::new(peripherals.GPIO6, config);

    spawner.spawn(print_task(channel_receiver).expect("Failed to spawn print task"));
    spawner.spawn(button_sender(button, channel_sender1).expect("Failed to spawn button sender"));

    loop {
        channel_sender0.send("message from sender 0").await;
        Timer::after(Duration::from_millis(2000)).await;
    }

    // for inspiration have a look at the examples at https://github.com/esp-rs/esp-hal/tree/esp-hal-v1.1.0/examples
}
```

**Pin Connection Reference Table:**

| Peripheral | Corresponding Pin |
| --- | ----- |
| Button | GPIO6 |

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

The main loop task sends a message to the channel every two seconds. When the button is pressed, the `button_sender` task also sends a message to the channel. The print task continuously receives messages from the channel and prints them out, ultimately achieving many-send-one-receive log printing synchronization communication.

# Code Explanation

## Channel

The `channel` in the embassy async framework is equivalent to the `queue` in FreeRTOS; both are queues used to transfer multiple sets of data between asynchronous tasks, and their design goals and core mechanisms are highly similar. Its definition is as follows:

```rust
pub struct Channel<M, T, const N: usize>
where
    M: RawMutex,
{ /* private fields */ }
```

Here `M` refers to the lock type, `T` refers to the message data type, and `N` refers to the length of the `channel`, i.e., the amount of data that can be transferred.

Below is a simple example of creating and using a `channel`:

```rust
static CHANNEL: Channel<CriticalSectionRawMutex, u32, 8> = Channel::new();

let channel_sender = CHANNEL.sender();
let channel_receiver = CHANNEL.receiver();

// Async send; if the channel is full, it will suspend until space is available 
channel_sender.send(42).await;
if !channel_sender.is_full(){
    // Synchronous non-blocking send; if the channel is full, it returns immediately
    channel_sender.try_send(42);
}
// Async receive; if the channel is empty, it will suspend until data is available 
channel_receiver.receive().await;
// Synchronous non-blocking receive; if there is no data in the channel, it returns immediately
 if !channel_sender.is_empty(){
     channel_receiver.try_receive();
}
```

Like FreeRTOS's `queue`, `channel` follows a **first-in, first-out** data output order and also **supports many-to-many communication with multiple producers and multiple consumers**. When data in the channel queue is taken by one consumer, other consumers can no longer obtain that data.

If you need **high-priority messages to leave the channel queue first, you can use `PriorityChannel`**; if you want to implement a **one-send-many-receive channel queue, you should use `PubSubChannel`**. The message data of a publish-subscribe channel can be obtained by all receivers; it holds an internal counter, and **the data is removed from the channel queue only after all receivers have received it**. The usage of priority channels and publish-subscribe channels is similar to that of the ordinary `channel`; for details you can also consult the official documentation for further understanding, so we won't go into more detail here.
