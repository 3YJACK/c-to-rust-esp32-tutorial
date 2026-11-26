**1. Rust Toolchain Installation**

Install `rustup` via the official website https://rustup.rs, then follow the instructions of the `rustup` program to install the Rust toolchain. After installation is complete, check in the terminal:

```powershell
rustc --version
cargo --version
```

**2. VS Code Extension Installation**

For Rust development in VS Code, it is recommended to install the following extensions:

| Extension name           | Function                                          |
| ---------------- | --------------------------------------------- |
| rust-analyzer    | Rust language server, improving code completion, go-to-definition, and error hints |
| Even Better TOML | Better syntax highlighting and completion for Cargo.toml        |
| Error Lens       | Display compilation error locations directly in the editor                |

**3. ESP32 Toolchain Installation**

ESP32 Rust development typically requires the following tools:

- espup: one-click installation of the ESP32 Rust development environment
- espflash: firmware flashing tool, supporting flashing and serial monitoring
- cargo-generate: quickly generate project templates

The installation commands are as follows:

```powershell
cargo install espup
espup install
cargo install espflash
cargo install cargo-espflash
cargo install cargo-generate
```

Check whether the installation was successful:

```powershell
espup --version
espflash --version
cargo generate --version
cargo espflash --version
```

For `no_std` mode projects, Espressif now recommends using `esp-generate` as the project generation tool. It can generate suitable project templates faster and reduce the amount of manual environment configuration work.

Installation command:

```powershell
cargo install esp-generate --locked
```

After installation is complete, check whether it is available:

```powershell
esp-generate 
```
