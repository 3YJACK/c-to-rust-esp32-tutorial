Enter the following command in the terminal to create the project:

```
esp-generate
```

Follow the terminal prompts to select the appropriate hardware platform and create the project template. The recommended project configuration is as follows:

**1. Core function configuration**

- **Enable unstable HAL features**: Enables experimental features in `esp-hal` that are not yet stable.

  - **Recommendation**: **Recommended to enable**. Many new features of the ESP32-S3 (such as Wi-Fi and BLE) depend on this. The API may change in the future, but this has little impact on personal projects.

- **Enable allocations via the esp-alloc crate**: Enables dynamic memory allocation (that is, using `Vec`, `String`, etc. on the heap).

  - **Recommendation**: **Be sure to enable**. It is the foundation for advanced features such as `wifi` and `ble`. Even if you do not use it immediately, it is recommended to enable it for future expansion.

- **Enable Wi-Fi via the esp-radio crate**: Enables Wi-Fi functionality.

  - **Recommendation**: **Enable as needed**. If the project needs network connectivity, enable it.

- **Enable BLE via the esp-radio crate (bleps)**: Enables BLE (Bluetooth Low Energy) functionality via the `bleps` stack.

  - **Recommendation**: **Enable as needed**. If you need BLE, you can start with `bleps`, which is relatively more mature.

- **Enable BLE via the esp-radio crate (embassy-trouble)**: Enables BLE via the `embassy-trouble` stack.

  - **Recommendation**: **Enable as needed**. If the project already uses the Embassy framework, choosing this option gives better integration.
  - **Note**: `wifi`, `ble-bleps`, and `ble-trouble` all depend on `alloc`. Wi-Fi and BLE may not be enableable at the same time at present.

- **Add embassy framework support**: Adds support for the Embassy asynchronous framework.

  - **Recommendation**: **Enable as needed; it can be left off during the early learning stage**. Embassy is the mainstream framework for asynchronous embedded programming in Rust. It lets you write clearer, more efficient concurrent code with `async/await`.

- **Enable stack smashing protection**: Enables stack overflow protection.

  - **Recommendation**: **Recommended to enable**. It can effectively improve system stability.
  - **Note**: This requires the **Nightly version** of Rust.

**2. Development and debugging tool configuration**

- **Use probe-rs to flash and monitor instead of espflash**: Uses `probe-rs` as the flashing and debugging tool.

  - **Recommendation**: **Choose as needed**. `probe-rs` supports more advanced debugging (such as breakpoints); `espflash` is simpler and more straightforward. Beginners can start with `espflash`.

- **Flashing, logging and debugging (probe-rs)**: Detailed configuration of `probe-rs`.

  - **`defmt`**: An efficient, lightweight logging framework, and a powerful tool for embedded debugging.

  - **`panic-rtt-target`**: Outputs `panic` information through RTT, making debugging easier.

  - **Recommendation**: If you selected `probe-rs`, **it is strongly recommended to enable `defmt` as well**.

- **Flashing, logging and debugging (espflash)**: Detailed configuration of `espflash`.

  - **`log`**: Uses the standard `log` library to print messages.

  - **`defmt`**: A low-overhead, efficient logging framework, normally used together with `probe-rs`; it can be chosen only as an alternative to `log`.

  - **`esp-backtrace`**: Provides detailed stack backtrace information.

  - **Recommendation**: If you selected `espflash`, it is recommended to enable **`esp-backtrace`** to make it easier to locate crash problems.

**3. Other configuration options**

- **Options**: Includes some auxiliary features.

  - **`wokwi`**: Supports running in the Wokwi simulator.

  - **`ci`**: Adds continuous integration configuration for GitHub Actions.

  - **Recommendation**: `wokwi` is suitable for rapid prototype verification; `ci` is suitable for team collaboration, and can be chosen as needed for personal projects.

- **Optional editor integration**: Generates configuration files for specific editors.

  - **Recommendation**: Just select the editor you use (such as VSCode).

- **Rust toolchain**: Specifies the Rust toolchain.

  - **Recommendation**: Unless there is a special reason, keep the default.

