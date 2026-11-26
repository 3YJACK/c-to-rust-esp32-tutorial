# Learning objectives

Based on the previous note `02 Project Creation`, use `esp-generate` to create a project template with the following configuration, then do a brief learning walkthrough and flash and run it.

```powershell
 ✅ Enable unstable HAL features. 
 ✅ Enable allocations via the esp-alloc crate. 
 ✅ Enable stack smashing protection.  
 Flashing, logging and debugging (espflash)
     ✅ Use the log crate to print messages. 
     ✅ Use esp-backtrace as the panic handler. 
 Optional editor integration 
     ✅ Add settings for Visual Studio Code        
```

# Complete source code

Below is the target source code of the project with explanatory comments; the corresponding source file is in the `./Code` folder:

```rust
// no_std and no_main indicate that this is an embedded program "without a standard library and without a default main entry point".
#![no_std]
#![no_main]

// The #![deny(...)] here means "treat certain unsafe/discouraged practices as errors".
// The benefit of this is that the compiler will warn you more strictly, helping you avoid writing code that is prone to problems in embedded programs.
//
// clippy::mem_forget: reminds you not to use mem::forget carelessly,
// because it may cause certain hardware objects containing buffers to be "forgotten to be released/cleaned up", leading to problems.
//
// clippy::large_stack_frames: reminds you not to put too much stack data in the main function, since stack space on embedded devices is usually very limited.
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
};

use log::info;

// Import the esp_backtrace crate, which can provide backtrace information when the program crashes, helping with debugging. Using "as _" means not binding a usable name, i.e. not directly using this crate in the code.
use esp_backtrace as _;

// Import the alloc crate for dynamic memory allocation
extern crate alloc;

// This creates a default app-descriptor required by the esp-idf bootloader.
// For more information see: <https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/app_image_format.html#application-description>
esp_bootloader_esp_idf::esp_app_desc!();

// For the main function, use #[allow] to permit large stack data so that compilation passes
#[allow(
    clippy::large_stack_frames,
    reason = "it's not unusual to allocate larger buffers etc. in main"
)]
//  #[main] is a procedural macro attribute provided by esp_hal, used to generate the real entry point for no_std/no_main programs.
#[main]
fn main() -> ! {    // ! indicates that this function never returns
    // generator version: 1.3.0
    // generator parameters: --chip esp32s3 -o esp32s3-wroom-1-octal-psram -o unstable-hal -o alloc -o stack-smashing-protection -o log -o esp-backtrace -o vscode

    // Initialize the log system as esp_println::logger 
    esp_println::logger::init_logger_from_env();

    // Initialize the chip configuration, setting the CPU clock to the maximum frequency
    let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
    let peripherals = esp_hal::init(config);

    // The following pins are used to bootstrap the chip. They are available
    // for use, but check the datasheet of the module for more information on them.
    // - GPIO0
    // - GPIO3
    // - GPIO45
    // - GPIO46
    // These GPIO pins are in use by some feature of the module and should not be used.
    // Drop the ownership of the unusable GPIO pins to avoid them being used later
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

    // Initialize the global heap allocator, allocating a block of heap memory of size 73744 bytes (placed in the RAM region specified by esp_hal::ram(reclaimed))
    esp_alloc::heap_allocator!(#[esp_hal::ram(reclaimed)] size: 73744);

    loop {
        // Use the log crate's info! macro to print log messages; println! cannot be used in a no_std context    
        info!("Hello world!");

        // Get the current time value into delay_start 
        let delay_start = Instant::now();
        // elapsed() returns the time interval from delay_start to now, and Duration::from_millis(500) creates a 500-millisecond duration object.
        while delay_start.elapsed() < Duration::from_millis(500) {}
    }

    // for inspiration have a look at the examples at https://github.com/esp-rs/esp-hal/tree/esp-hal-v1.1.0/examples
}
```

# Flashing and running

Use the following command to compile:

```powershell
cargo build 
```

Use the following command to flash and run:

```powershell
cargo espflash flash --monitor
```

`--monitor` automatically opens the serial monitor after flashing completes to view log output.

**Expected result:**

Seeing the following log output after running the compile command means compilation was successful:

```powershell
Finished `dev` profile [optimized + debuginfo] target(s) in 1m 32s 
```

Running the flash command should show the following log messages:

```powershell
Chip type:         esp32s3 (revision v0.2)
Crystal frequency: 40 MHz
Flash size:        16MB
Features:          WiFi, BLE, Embedded Flash
MAC address:       ac:a7:04:ee:14:b0
App/part. size:    107,744/16,384,000 bytes, 0.66%
[00:00:00] [========================================]       1/1       0x0      Verifying... OK!                                                                                 
[00:00:00] [========================================]       1/1       0x8000   Verifying... OK!                                                                                 
[00:00:01] [========================================]       3/3       0x10000  Verifying... OK!                                                                                 [2026-08-03T15:03:44Z INFO ] Flashing has completed!
Commands:
    CTRL+R    Reset chip
    CTRL+C    Exit

ESP-ROM:esp32s3-20210327
Build:Mar 27 2021
rst:0x15 (USB_UART_CHIP_RESET),boot:0x8 (SPI_FAST_FLASH_BOOT)
Saved PC:0x40378eb1
SPIWP:0xee
mode:DIO, clock div:2
load:0x3fce2820,len:0x158c
load:0x403c8700,len:0xd24
load:0x403cb700,len:0x2f34
entry 0x403c8924
I (27) boot: ESP-IDF v5.5.1-838-gd66ebb86d2e 2nd stage bootloader
I (27) boot: compile time Nov 26 2025 12:27:56
I (27) boot: Multicore bootloader
I (29) boot: chip revision: v0.2
I (31) boot: efuse block revision: v1.3
I (35) boot.esp32s3: Boot SPI Speed : 40MHz
I (39) boot.esp32s3: SPI Mode       : DIO
I (43) boot.esp32s3: SPI Flash Size : 16MB
I (47) boot: Enabling RNG early entropy source...
I (51) boot: Partition Table:
I (54) boot: ## Label            Usage          Type ST Offset   Length
I (60) boot:  0 nvs              WiFi data        01 02 00009000 00006000
I (66) boot:  1 phy_init         RF data          01 01 0000f000 00001000
I (73) boot:  2 factory          factory app      00 00 00010000 00fa0000
I (80) boot: End of partition table
I (83) esp_image: segment 0: paddr=00010020 vaddr=3c000020 size=07ff4h ( 32756) map
I (98) esp_image: segment 1: paddr=0001801c vaddr=3fc896bc size=00a88h (  2696) load
I (99) esp_image: segment 2: paddr=00018aac vaddr=40378000 size=016bch (  5820) load
I (107) esp_image: segment 3: paddr=0001a170 vaddr=00000000 size=05ea8h ( 24232) 
I (118) esp_image: segment 4: paddr=00020020 vaddr=42010020 size=0a490h ( 42128) map
I (131) boot: Loaded app from partition at offset 0x10000
I (131) boot: Disabling RNG early entropy source...
INFO - Hello world!
INFO - Hello world!
INFO - Hello world!
INFO - Hello world!
INFO - Hello world!
INFO - Hello world!
INFO - Hello world!
```

You can see the Hello world! message printed normally, and the template program runs successfully!

# Code walkthrough

## Attributes

At the top of the template code, you can see several lines of code enclosed in `#![]`. In Rust, `#[]` and `#![]` are collectively called **attributes**.

You can think of them as **"special instructions" or "metadata" written for the compiler**, used to tell the compiler to perform specific processing on the code, such as conditional compilation, automatic code generation, setting the entry point, and so on.

The difference between the two lies in their **scope of application**:

- **`#[]`**: **outer attribute** — placed outside the region it modifies, modifying the code block immediately below it (a function, struct, variable, etc.).

- **`#![]`**: **inner attribute** — placed outside the region it modifies, modifying the entire file, module, or crate (software package) in which it resides.

The inner attributes at the beginning of the source code apply to the whole file, declaring a bare-metal embedded environment and forbidding the arbitrary use of large stack data and `mem::forget` (`mem::forget` makes Rust skip the destructor (Drop) of that object, i.e. "forgetting" to release the resources occupied by the object).

```rust
// no_std and no_main indicate that this is an embedded program "without a standard library and without a default main entry point".
#![no_std]
#![no_main]

// The #![deny(...)] here means "treat certain unsafe/discouraged practices as errors".
// The benefit of this is that the compiler will warn you more strictly, helping you avoid writing code that is prone to problems in embedded programs.
//
// clippy::mem_forget: reminds you not to use mem::forget carelessly,
// because it may cause certain hardware objects containing buffers to be "forgotten to be released/cleaned up", leading to problems.
//
// clippy::large_stack_frames: reminds you not to put too much stack data in the main function, since stack space on embedded devices is usually very limited.
#![deny(
    clippy::mem_forget,
    reason = "mem::forget is generally not safe to do with esp_hal types, especially those \
    holding buffers for the duration of a data transfer."
)]
#![deny(clippy::large_stack_frames)]
```

In front of the `main` function, an outer attribute is used again, that is, a local permission to use large stack data that applies only to the `main` function, without triggering compiler warnings. The reason is also given in the code:

> it's not unusual to allocate larger buffers etc. in main
> 
> It is quite common to allocate larger buffers etc. in the main function.

```rust
// For the main function, use #[allow] to permit large stack data so that compilation passes
#[allow(
    clippy::large_stack_frames,
    reason = "it's not unusual to allocate larger buffers etc. in main"
)]
//  #[main] is a procedural macro attribute provided by esp_hal, used to generate the real entry point for no_std/no_main programs.
#[main]
fn main() -> ! {
}
```

## Log printing

The `log` crate is one of the widely used logging tools in the Rust community.

In the source code, `esp_println` provides the concrete implementation of the logging tool: it initializes the logging system and makes the ESP logger the backend of the `log` system, so that log messages printed with macros such as `info!` are output through the ESP logger (usually to the serial port).

```rust
use log::info;

// Initialize the log system as esp_println::logger 
esp_println::logger::init_logger_from_env();
```

## Macros

Calls like `info!` with a `!` in the source code are actually how macros are denoted in Rust.

To define your own macro, use `macro_rules!` to declare it, as shown in the following example:

```rust
macro_rules! add_one { 
    // $x:expr means accepting an expression as an argument
    ($x:expr) => { $x + 1 };
} 

let result = add_one!( 5 ); // expands to 5 + 1 
println! ( "{}" , result); // prints 6

// Macro with no arguments
macro_rules! say_hello { 
     // () means this macro needs no arguments 
     () => { 
         info! ( "Hello from macro!" );
      };
}
```

Note that when defining a macro, parameters in `macro_rules!` must be declared and referenced with `$`.
