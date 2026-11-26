# Learning Objectives

Use `esp-generate` to create a project and refer to the `./example/interrupt/uart` example in the `esp-rs/esp-hal` repository to write code and implement simple UART serial communication functionality. 

**Prerequisites:**

It is recommended to study this content after mastering **<u>Result</u>** of the **<u>enum types</u>** in [Rust basics](./00-rust-basics.md).

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

use esp_hal::{
    clock::CpuClock,
    main,
    time::{Duration, Instant},
    uart::*, // includes Uart Module 
};

use log::info;

use esp_backtrace as _;

extern crate alloc;

// This creates a default app-descriptor required by the esp-idf bootloader.
// For more information see: <https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/app_image_format.html#application-description>
esp_bootloader_esp_idf::esp_app_desc!();

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

    // uart initialization
    let uart_config = Config::default()
        .with_baudrate(115200)
        .with_data_bits(DataBits::_8)
        .with_parity(Parity::None)
        .with_stop_bits(StopBits::_1);

    let mut uart = Uart::new(peripherals.UART1, uart_config)
        .expect("Failed to initialize UART")    
        .with_tx(peripherals.GPIO43)
        .with_rx(peripherals.GPIO44);   

    let message = b"Hello, UART!\n";

    // delay initialization
    let delay = esp_hal::delay::Delay::new();

    loop {
        uart.write(message);
        delay.delay_millis(1000);
    }

    // for inspiration have a look at the examples at https://github.com/esp-rs/esp-hal/tree/esp-hal-v1.1.0/examples
}
```

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

After successfully flashing and running the program, the log output should print *"Hello, UART!"* once per second.

# Code Explanation

## UART

The initialization process of UART is exactly the same as that of the GPIO in the previous article:

**Create default configuration → modify configuration struct fields → create the serial object and bind pins**

```rust
    // uart initialization
    let uart_config = Config::default()
        .with_baudrate(115200)
        .with_data_bits(DataBits::_8)
        .with_parity(Parity::None)
        .with_stop_bits(StopBits::_1);

    let mut uart = Uart::new(peripherals.UART1, uart_config)
        .expect("Failed to initialize UART")    
        .with_tx(peripherals.GPIO43)
        .with_rx(peripherals.GPIO44);   
```

Once initialization is complete, you can operate UART to send and receive data. Related function methods, data types, and so on can all be consulted in the official documentation [esp_hal::uart - Rust](https://docs.espressif.com/projects/rust/esp-hal/1.1.0/esp32s3/esp_hal/uart/index.html).

## Result

In the UART initialization code snippet shown above, `Uart::new()` returns a `Result` type, which contains the result of the UART initialization. If the initialization succeeds, the corresponding UART instance can be extracted from the `Result`; otherwise, what is extracted is the UART error code. For details of the error code, see `ConfigError` in the `uart` module of the official documentation.

```rust
    let mut uart = Uart::new(peripherals.UART1, uart_config)
        .expect("Failed to initialize UART")    
```

The example uses the `.expect()` method to unwrap the result; if the result is an error, it will also print the message *"Failed to initialize UART"*.

## Byte Slices

```rust
let message = b"Hello, UART!\n";
```

Here, adding a `b` in front of `Hello UART!` converts the string slice `&str` into a byte slice `&[u8]`, which is equivalent to the code shown below:

```rust
let msg:&str = "Hello";
let bytes: &[u8] = msg.as_bytes(); // yields [72, 101, 108, 108, 111]
```

The difference between the two is:

- **`&str` (string slice)**: **must** be valid **UTF-8** encoding. It can only point to legal character sequences that conform to the UTF-8 standard.

- **`&[u8]` (byte slice)**: **can be any data**. It is simply a collection of bytes in a segment of memory and does not care what those bytes represent. It can be UTF-8 text, the binary data of an image, raw sensor readings, or even garbled data.

Serial communication requires the use of the byte slice `&[u8]`, and the data it transmits is itself a byte stream.
