# Learning Objectives

1. Add the `ssd1306` driver, a third-party dependency package for the OLED display, via the official package manager `Cargo`

2. Create a project using `esp-generate`, and write this example by referring to the official documentation [esp_hal - Rust](https://docs.espressif.com/projects/rust/esp-hal/1.1.0/esp32s3/esp_hal/index.html) and the ssd1306 driver documentation [ssd1306 - Rust](https://docs.rs/ssd1306/0.10.0/ssd1306/index.html), implementing the driving of an OLED display via I2C communication

# Adding Dependencies

The process of adding a third-party dependency package can be summarized in three steps: **find the crate**, **add it**, **use it**.

1. **Find the crate**

Rust's official crate repository is [**crates.io**](https://crates.io/), where you can search for the packages you need. Each package comes with detailed documentation and a GitHub source code link, and many packages state the addition method and instructions right in their documentation.

2. **Add it**

According to the documentation, add the dependency package with the following command:

```powershell
cargo add ssd1306
```

This command automatically adds the latest version of `ssd1306` to the `[dependencies]` section of `Cargo.toml`

Alternatively, you can also add it manually directly in the Cargo.toml file:

```powershell
ssd1306 = "0.10.0"
```

3. **Use it**

After the dependency is added and saved, Cargo will automatically download and compile it at the next build. Then you can import the `ssd1306` driver library in your code and use it according to the documentation.

```rust
use ssd1306::{......};
```

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
    i2c::master::{Config, I2c},
};

use ssd1306::{I2CDisplayInterface, Ssd1306, command::AddrMode, rotation::*, size::*};
use embedded_graphics::{ 
    prelude::*,
    mono_font::{ascii::FONT_6X10, MonoTextStyleBuilder},
    pixelcolor::BinaryColor,
    text::{Baseline, Text},
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

    // Initialize I2C by default, details as follows:
    // frequency：100kHz, timeout：disabled, software_timeout：disabled
    // scl_st_timeout: 16, scl_main_st_timeout: 16
    let i2c_config = Config::default(); 
    let i2c = I2c::new(peripherals.I2C0, i2c_config)
            .expect("Failed to initialize I2C")
            .with_scl(peripherals.GPIO42)
            .with_sda(peripherals.GPIO41);

    let interface = I2CDisplayInterface::new(i2c);

    let mut display = Ssd1306::new(interface, DisplaySize128x64, DisplayRotation::Rotate0)
        .into_buffered_graphics_mode();
    display.init_with_addr_mode(AddrMode::Horizontal)
        .expect("Failed to initialize oled display");

    let text_style = MonoTextStyleBuilder::new()
        .font(&FONT_6X10)
        .text_color(BinaryColor::On)
        .build();

    display.clear_buffer();

    Text::with_baseline("Hello world!", Point::zero(), text_style, Baseline::Top)
    .draw(&mut display)
    .expect("Failed to draw text 1");

    Text::with_baseline("Hello Rust!", Point::new(0, 16), text_style, Baseline::Top)
    .draw(&mut display)
    .expect("Failed to draw text 2");

    display.flush().expect("Failed to flush display");

    info!("program running");
    loop {

    }

    // for inspiration have a look at the examples at https://github.com/esp-rs/esp-hal/tree/esp-hal-v1.1.0/examples
}
```

**Pin Connection Reference Table:**

| Peripheral       | Corresponding Pin   |
| ---------------- | ------------------- |
| OLED_SCL         | GPIO42              |
| OLED_SDA         | GPIO41              |

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

After connecting the OLED display driven by the `SSD1306` driver and flashing and running the program, you should see information displayed in two lines on the OLED display. The first line shows *"Hello world!"*, and the second line shows *"Hello Rust!"*.

# Code Explanation

## I2C

The configuration and creation example of the I2C peripheral is as follows:

```rust
    let i2c_config = Config::default(); 
    let i2c = I2c::new(peripherals.I2C0, i2c_config)
            .expect("Failed to initialize I2C")
            .with_scl(peripherals.GPIO42)
            .with_sda(peripherals.GPIO41);
```

If you want to modify the configuration, just use the `with_XXX()` method on the configuration struct. For the specific API, see the official documentation. Once creation is complete, you can call the relevant I2C read and write methods; details can also be found in the official documentation.

When driving peripherals via I2C, implementation often relies on third-party driver libraries. Generally, you follow the driver's documentation or example instructions, inject the created I2C instance into the driver library, and then directly call the driver library's API to perform driving and communication.

```rust
    let interface = I2CDisplayInterface::new(i2c);
    let mut display = Ssd1306::new(interface, DisplaySize128x64, 
                                    DisplayRotation::Rotate0);
    // ......
```

## Traits and Platform-Independent Drivers

In traditional embedded development, a device's driver library often depends on specific vendors' HAL library functions; once the hardware platform changes, the driver must be rewritten as well. But in this example, when we use the `SSD1306` driver library, we use it almost directly, without caring whether its underlying implementation supports the ESP32 platform.

If you open the source code of this driver library, you will find that **no specific hardware platform is mentioned at all** in the source code. It recognizes neither ESP32 nor STM32, yet it can run directly on these hardware platforms.

This is a powerful core design philosophy of Rust embedded development — **platform-independent drivers**.

To achieve platform-independent drivers, Rust designed a library called [**`embedded-hal`**](https://crates.io/crates/embedded-hal). Its core idea is very simple: **define a set of standard interfaces (traits), drivers depend only on this set of interfaces, and chip vendors provide concrete implementations for this set of interfaces**.

A `trait` is the core mechanism in Rust for defining **shared behavior**. **A trait defines a set of methods, and any type that implements this trait can call these methods.** It specifies what a type can do, but does not care what the type actually is, similar to "polymorphism" in other programming languages.

Simply put, the `embedded-hal` `trait` is a set of "**interface protocols**", **but only interface definitions**, without any concrete implementation. The concrete implementation is the responsibility of the chip vendors, and developers only need to focus on the driver logic on the basis of this interface protocol.

Therefore, in the `embedded-hal` ecosystem, the roles of device drivers and chip vendors are very clearly divided:

| Role                 | What it does                                                 | Example                           |
| -------------------- | ------------------------------------------------------------ | --------------------------------- |
| **Chip HAL library** | Implements `embedded-hal` traits for a specific chip         | `esp-hal`, `stm32f4xx-hal`        |
| **Device driver**    | Depends only on `embedded-hal` traits, does not care about the chip | `ssd1306`, `bmp180`, `mpu6050`    |

**Chip HAL library**: such as `esp-hal`, is responsible for wrapping ESP32 hardware register operations into an implementation of the `embedded_hal::i2c::I2c` `trait`.

**Device driver**: such as `ssd1306`, recognizes only the `embedded_hal::i2c::I2c` `trait`, calls the corresponding method interfaces, and **completely does not care whether the underlying implementation** is ESP32 or STM32.

And when using it, we only need to inject the I2C instance created by the chip HAL library into the device driver. This is the so-called "**dependency injection**".

Analyzing with the code, in the source code of this driver library, the I2C entry looks like this:

```rust
pub fn new<I>(i2c: I) -> I2CInterface<I>
where
    I: embedded_hal::i2c::I2c,
{
    Self::new_custom_address(i2c, 0x3C)
}
```

It only requires: give me a type that implements `embedded_hal::i2c::I2c`. The `I2c` of `esp-hal` implements this `trait`, and the HALs of STM32, nRF, and RP2040 also implement it, so **the same driver can be used on different chips**.

When you inject an ESP32 instance implementing the `embedded_hal::i2c::I2c` type into the driver:

```rust
let i2c = esp_hal::i2c::master::I2c::new(...);

let interface = I2CDisplayInterface::new(i2c);

let mut display = Ssd1306::new(interface,......）;
```

The compiler replaces the generic `I` with the concrete type:

```rust
Ssd1306<esp_hal::i2c::master::I2c<'_, Blocking>>
```

Then it calls the concrete implementation of esp_hal, and further down are the ESP register operations. The same applies when switching to other hardware platforms.

**Comparison with Linux — same idea, different implementation:**

Driving devices through **standard interfaces, isolating the underlying implementation**? Readers familiar with Linux should, like me, feel that this mechanism is very similar to that of Linux.

Indeed, the two are consistent in design philosophy, but the concrete implementations are different. Simply put, **Linux is runtime hardware abstraction, while Rust is compile-time runtime abstraction**.

- **Linux's approach**: `read(fd, buffer, size)`, the VFS (Virtual File System) in the kernel forwards it to the concrete ext4 or FAT32 driver, or to the concrete NVMe or SD card driver. The application-layer code **does not care** whether the disk is SATA or USB.

- **Rust `embedded-hal`'s approach**: `i2c.write(address, data)`, this trait method is replaced at compile time with the concrete implementation of `esp_hal::i2c::I2C` or `stm32_hal::i2c::I2C`. The driver code **does not care** whether the hardware platform uses ESP32 or STM32.

|          | **Linux abstraction**                | **Rust abstraction**                                    |
| -------- | ----------------------------------- | ------------------------------------------------------ |
| **When it occurs** | **Runtime**                     | **Compile time**                                        |
| **Implementation mechanism** | Function pointers, structure virtual tables (Vtable), dynamic device tree matching. | Generic monomorphization (Monomorphization), static dispatch (Static Dispatch). |
| **Performance overhead** | Has runtime overhead (function pointer jumps, instruction cache pollution).      | **Zero overhead**, code generated after compilation.                              |
| **Flexibility**  | Driver modules can be loaded/unloaded without restarting the system.      | Drivers are completely fixed at compile time. Switching to a different chip requires recompiling the entire firmware.                 |
