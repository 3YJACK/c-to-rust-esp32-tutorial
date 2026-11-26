# c-to-rust-esp32-tutorial

[中文](README.md) | **English**

A personal learning record of Rust `no_std` embedded development, written by a C embedded developer — and also a beginner-friendly tutorial for developers with a similar background.

Using the ESP32-S3 as the hardware platform, it covers everything from Rust syntax basics and toolchain setup to peripheral drivers and Embassy-based async concurrency. Every chapter has notes and a runnable source project.

## Repository Structure

```
.
├── Notes/                          # Learning notes (the main tutorial)
│   ├── 00语法基础.md                # Minimum Rust syntax you need
│   ├── 01环境搭建.md                # Installing Rust + ESP32 toolchains
│   ├── 02工程创建.md                # Creating a project with esp-generate
│   ├── 03template.md               # Explaining the project template
│   ├── 04GPIO点灯.md                # GPIO output, blinking an LED
│   ├── 05uart串口.md                # UART serial communication
│   ├── 06interrupt引脚中断.md        # External interrupts
│   ├── 07timer定时器.md             # Hardware timers
│   ├── 08i2c-oled.md               # Driving an OLED display over I2C
│   ├── 09async异步.md               # Async programming concepts
│   ├── 10async-blink.md            # Async blinking with Embassy
│   ├── 11async-priority.md         # Async task priorities
│   ├── 12async-signal&watch.md     # Signal and Watch communication
│   ├── 13async-channel.md          # Channel communication
│   └── 14async-mutex.md            # Mutex for shared data
│
├── Notes_EN/                        # Full English translations of the notes
│   ├── 00-rust-basics.md           # Minimum Rust syntax you need
│   ├── 01-toolchain-setup.md       # Installing Rust + ESP32 toolchains
│   ├── 02-project-creation.md      # Creating a project with esp-generate
│   ├── 03-template.md              # Explaining the project template
│   ├── 04-gpio-blink.md            # GPIO output, blinking an LED
│   ├── 05-uart.md                  # UART serial communication
│   ├── 06-interrupts.md            # External interrupts
│   ├── 07-timers.md                # Hardware timers
│   ├── 08-i2c-oled.md              # Driving an OLED display over I2C
│   ├── 09-async-concepts.md        # Async programming concepts
│   ├── 10-async-blink.md           # Async blinking with Embassy
│   ├── 11-async-priority.md        # Async task priorities
│   ├── 12-signal-watch.md          # Signal and Watch communication
│   ├── 13-async-channel.md         # Channel communication
│   └── 14-async-mutex.md           # Mutex for shared data
│
├── Code/                           # Runnable projects matching the notes
│   ├── 03template/                 
│   ├── 04GPIO-blink/               
│   ├── 05uart/                     
│   ├── 06interrupt/               
│   ├── 07timer/                    
│   ├── 08i2c-oled/                 
│   ├── 10async-blink/              
│   ├── 11async-priority/           
│   ├── 12async-signal&watch/       
│   ├── 13async-channel/            
│   └── 14async-mutex/              
│
├── Cargo.toml                     
├── README.md                      
└── README_EN.md                    
```

## Learning Path

The Notes column links to the English translation. The Chinese original can be found under `Notes/` with the filename shown in the directory tree above.

| Chapter | Notes                                               | Project                 | Topics                                                                         |
| ------- | --------------------------------------------------- | ----------------------- | ------------------------------------------------------------------------------ |
| 00      | [Rust basics](Notes_EN/00-rust-basics.md)           | —                       | Minimal Rust syntax: variables, functions, ownership, modules, generics, enums |
| 01      | [Toolchain setup](Notes_EN/01-toolchain-setup.md)   | —                       | Installing rustup / espup / espflash / esp-generate                            |
| 02      | [Project creation](Notes_EN/02-project-creation.md) | —                       | Creating a `no_std` project with esp-generate                                  |
| 03      | [Project template](Notes_EN/03-template.md)         | `03template/`           | Template layout, Cargo.toml, build.rs                                          |
| 04      | [GPIO blink](Notes_EN/04-gpio-blink.md)             | `04GPIO-blink/`         | GPIO output, blinking an LED                                                   |
| 05      | [UART](Notes_EN/05-uart.md)                         | `05uart/`               | UART communication, log output                                                 |
| 06      | [Interrupts](Notes_EN/06-interrupts.md)             | `06interrupt/`          | External interrupts and interrupt handlers                                     |
| 07      | [Timers](Notes_EN/07-timers.md)                     | `07timer/`              | Hardware timers                                                                |
| 08      | [I2C OLED](Notes_EN/08-i2c-oled.md)                 | `08i2c-oled/`           | I2C bus, SSD1306 OLED display                                                  |
| 09      | [Async concepts](Notes_EN/09-async-concepts.md)     | —                       | Async concepts: Future / Waker / Executor                                      |
| 10      | [Async blink](Notes_EN/10-async-blink.md)           | `10async-blink/`        | Embassy framework, async LED blink                                             |
| 11      | [Async priority](Notes_EN/11-async-priority.md)     | `11async-priority/`     | Async task priorities                                                          |
| 12      | [Signal & Watch](Notes_EN/12-signal-watch.md)       | `12async-signal&watch/` | Signal and Watch communication                                                 |
| 13      | [Channel](Notes_EN/13-async-channel.md)             | `13async-channel/`      | Channel communication                                                          |
| 14      | [Mutex](Notes_EN/14-async-mutex.md)                 | `14async-mutex/`        | Async mutex and safe shared data                                               |

> **Start with Chapter 00 and go in order.** If you already know Rust, you can skip Chapter 00 and start at Chapter 01.

## Prerequisites

### Hardware

- An ESP32-S3 development board (other ESP32 chips with Rust support also work — see the [esp_hal documentation](https://docs.espressif.com/projects/rust/esp-hal/1.2.0/index.html) for supported models)
- Basic peripherals such as LEDs, buttons, and an OLED display

### Software

- **Rust toolchain**: install via [rustup](https://rustup.rs); each project's `rust-toolchain.toml` pins the version
- **ESP32 toolchain**: `espup` (Xtensa Rust compiler), `espflash` (flashing tool), and `esp-generate` (project generator)
- **VS Code** (recommended) with the rust-analyzer extension

See [toolchain setup](Notes_EN/01-toolchain-setup.md) for detailed installation steps .

### Background Knowledge

- C programming basics (required)
- Embedded development experience and familiarity with GPIO / UART / I2C / interrupts / timers (required)
- Rust basics (optional — Chapter 00 covers the minimum, but reading [The Rust Programming Language](https://doc.rust-lang.org/stable/book/) alongside is recommended)

## About the Notes

The notes are the core of this tutorial. Full English translations are in `Notes_EN/`; Chinese originals are in `Notes/`. Each chapter typically follows this structure:

1. **Learning goals** — what this chapter does and what you will learn
2. **Concepts** — "what it is" and "how it compares with C"
3. **Complete source code** — matching a runnable project under `./Code/`
4. **Key takeaways** — important APIs and common pitfalls

The code examples in the notes have full versions in the corresponding `./Code/` projects. Read the notes alongside the source, and don't hesitate to modify the code and run it when something is unclear.

## Building and Flashing

Each project under `Code/` is a standalone Cargo package. From the project directory:

```bash
cargo build                 # compile
cargo espflash flash --monitor   # flash and open the serial monitor
```

`cargo run` also works: each project's `.cargo/config.toml` configures `espflash flash --monitor --chip esp32s3` as the runner.
