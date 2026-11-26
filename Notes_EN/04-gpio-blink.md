# Learning objectives

Use `esp-generate` to create a project and refer to the `./example/interrupt/gpio` example in the `esp-rs/esp-hal` repository, then write code to implement GPIO blinking an LED.

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
    main,
    time::{Duration, Instant},
    gpio::*,   // Import all public types and functions from the gpio module
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
    // Initialize all peripherals uniformly
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

    // led initialization 
    let mut led = Output::new(peripherals.GPIO2, Level::Low, OutputConfig::default());
    let delay = esp_hal::delay::Delay::new();

    // led blink
    led.set_high();
    delay.delay_millis(1000);

    led.set_low();
    delay.delay_millis(1000);

    loop {
       led.toggle();
       info!("led level: {:?}", led.output_level());
       delay.delay_millis(1000);
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

**Expected result:**

After compilation and flashing succeed, the target pin outputs a high/low square wave, and the log output should look like the following. If an LED is connected, it should blink on and off periodically.

```
INFO - led level: Low
INFO - led level: High
INFO - led level: Low
INFO - led level: High
.......
```

# Code walkthrough

## GPIO

In Rust, the use of all peripherals must first go through a unified initialization, which takes ownership of all on-chip peripherals at once, and then manages them uniformly through a single object, allocating them for use as needed. 

```rust
let peripherals = esp_hal::init(config);
```

In the source code, after the unified initialization, `peripherals` obtains ownership of all peripherals; the required pin is taken from `peripherals` and configured for GPIO initialization:

```rust
let mut led = Output::new(peripherals.GPIO2, Level::Low, OutputConfig::default());
```

Here pin 2 is configured in output mode with an initial level of low, and the pin configuration uses the default configuration provided by the library.

Reading the `esp-hal` chapter in the official documentation [esp-rs Documentation and Resources](https://docs.espressif.com/projects/rust/), we can see that the members and default configuration of the output configuration struct are as follows:

```markdown
Output pin configuration

This struct is used to configure the drive mode, drive strength, and pull direction of an output pin.
 By default, the configuration is set to:

- Drive mode: `DriveMode::PushPull`
- Drive strength: `DriveStrength::_20mA`
- Pull direction: `Pull::None` (no pull resistors connected)
```

Therefore, the process and example for configuring the GPIO output/output mode yourself and initializing it are as follows:

**Create configuration → modify configuration struct fields → create `Output/Input` object**

```rust
let config = OutputConfig::default()
    .with_drive_mode(DriveMode::PushPull)
    .with_pull(Pull::None);
let mut led = Output::new(peripherals.GPIO2, Level::Low, config);
```

Then you can call the internal methods of `Output/Input`, i.e. the relevant GPIO driver functions; for details see the official documentation [Output in esp_hal::gpio - Rust](https://docs.espressif.com/projects/rust/esp-hal/1.1.0/esp32s3/esp_hal/gpio/struct.Output.html).
