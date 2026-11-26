# Learning Objectives

Use `esp-generate` to create the project (remember to enable the embassy async framework), and refer to the official embassy synchronization communication module documentation [embassy_sync - Rust](https://docs.rs/embassy-sync/latest/embassy_sync/), write the code and complete a synchronization communication example that uses `Mutex` to read and write shared data.

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
    interrupt::software::SoftwareInterruptControl,
};

use embassy_executor::Spawner;
use embassy_time::{Duration, Timer};
use embassy_sync::{     // Import embassy synchronization communication module
    blocking_mutex::raw::CriticalSectionRawMutex, 
    mutex::Mutex,
};

use log::info;

use esp_backtrace as _;

extern crate alloc;

// This creates a default app-descriptor required by the esp-idf bootloader.
// For more information see: <https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/app_image_format.html#application-description>
esp_bootloader_esp_idf::esp_app_desc!();

#[embassy_executor::task]
async fn write_task_a(
    shared_data: &'static Mutex<CriticalSectionRawMutex, u32>
) {
    loop {
        Timer::after(Duration::from_millis(1000)).await;
        {
            let mut guard = shared_data.lock().await;
            *guard =  *guard + 1;
            info!("data: {}, written by Task A", *guard);
        }
    }
}

#[embassy_executor::task]
async fn write_task_b(
    shared_data: &'static Mutex<CriticalSectionRawMutex, u32>
) {
    loop {
        Timer::after(Duration::from_millis(1500)).await;
        {
            let mut guard = shared_data.lock().await;
            *guard =  *guard + 1;
            info!("data: {}, written by Task B", *guard);
        }
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
    let sw_interrupt = SoftwareInterruptControl::new(peripherals.SW_INTERRUPT);
    esp_rtos::start(timg0.timer0, sw_interrupt.software_interrupt0);

    info!("Embassy initialized!");

    static MUTEX: Mutex<CriticalSectionRawMutex, u32> = Mutex::new(0);

    spawner.spawn(write_task_a(&MUTEX).expect("Failed to spawn write_task_a"));
    spawner.spawn(write_task_b(&MUTEX).expect("Failed to spawn write_task_b"));

    loop {
        Timer::after(Duration::from_millis(5000)).await;
    }
    // for inspiration have a look at the examples at https://github.com/esp-rs/esp-hal/tree/esp-hal-v1.1.0/examples
}
```

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

Through the mutex, task_a and task_b alternately access and modify the shared data, while printing out the value of the data.

# Code Explanation

## Mutex

A synchronization communication method used to **protect shared resources and avoid resource contention**. Its definition is as follows:

```rust
pub struct Mutex<M, T>
```

Here `M` is the lock type, and `T` is the type of the protected data.

The essence of `Mutex<T>` is a container that holds the data `T` inside it, but it **does not itself provide direct access to the data; instead, it uses the `.lock()` method to "request access to the data"**, which is `MutexGuard<'a, T>` — the guard returned after calling `.lock()`. It is a smart pointer through which you can operate on the mutex's internally protected data just like operating on a `&mut T`.

The guard is the core mechanism of the Rust language in the safety design of `mutex`. Unlike C, which separates the lock from the data, **the guard binds "holding the lock" and "data access permission" together, thereby guaranteeing the operation order of "lock the data first, then access it". At the same time, the lifetimes of the two are bound together as well; when the guard is destroyed upon leaving its scope, the lock is automatically released along with it, thus avoiding the classic concurrency problems in C of "forgetting to release the lock" or "one of the lock and the data having become invalid while the other is still in use"**.

A simple example of creating and using a mutex is as follows:

```rust
static MUTEX: Mutex<CriticalSectionRawMutex, u32> = Mutex::new();

// Acquire the lock asynchronously; if it is already occupied, suspend until the lock is released
let mut guard = MUTEX.lock().await;
*guard += 1;
// The lock is automatically released when guard goes out of scope

// Synchronous non-blocking attempt to acquire the lock; if it is already occupied, return None immediately
if let Some(mut guard) = MUTEX.try_lock() {
    *guard += 1;
}
// The lock is automatically released when guard goes out of scope
```

In addition, Mutex has two methods that are more suitable for scenarios where a lock is not needed:

```rust
static MUTEX: Mutex<CriticalSectionRawMutex, u32> = Mutex::new(42);

// Destroy the lock and directly take out its internal value; suitable when the lock is no longer needed
let value = MUTEX.into_inner();  // mutex is consumed, value = 42
// The mutex is invalid here and can no longer be used

// Get a mutable reference directly without locking; suitable for phases with no concurrency, avoiding lock overhead
// If concurrency safety can be guaranteed at compile time and pass the compiler's checks, there is no need to lock at runtime
let value = MUTEX.get_mut();
*value += 1;  

// When concurrent access is needed, then add the lock
let guard = MUTEX.lock();
```

**Difference between the `Mutex` of `critical_section` and `embassy_sync`:**

In our earlier study of interrupts, we also encountered `Mutex`, but that was `critical_section`'s `Mutex`. What we are learning now is `embassy_sync`'s `Mutex`. The differences between the two can be seen in the table below:

|           | `critical_section::Mutex`         | `embassy_sync::mutex::Mutex`                          |
| --------- | --------------------------------- | ----------------------------------------------------- |
| **Core Mechanism** | Guarantees atomicity by **disabling interrupts** (critical section). | Internally uses a **`RawMutex`** to protect its "locked state" flag. |
| **Sync/Async** | **Synchronous**; acquiring the lock **busy-waits** until the lock is available. | **Asynchronous**. When the lock is occupied, `lock().await` **suspends the current task**, yielding the CPU, rather than busy-waiting. |
| **Use Cases** | Synchronization **between tasks and interrupts**, or shared data protection in **single-core** systems. | Synchronization **between asynchronous tasks**, especially within the Embassy executor. |
| **Use in Interrupts** | ✅ **Yes**. It is designed for use in interrupt contexts. | ❌ **No**. `.await` cannot be used in interrupts. |
| **Performance Overhead** | Acquiring and releasing the lock **briefly disables global interrupts**, which may affect system real-time performance. | `lock().await` involves task scheduling and context switching, with greater overhead than a critical section, but it does not block other tasks. |
| **Lock Type** | It is itself a lock. | It is a **composite** lock whose behavior is determined by the passed-in `RawMutex` type parameter. |
