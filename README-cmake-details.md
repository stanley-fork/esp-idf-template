# Integrating a Rust Component into an ESP-IDF Project

ESP-IDF, the official development framework for the ESP32 Series SoCs, supports integration of components written in C/C++ and Rust which is gaining traction for embedded development due to its safety features. This article outlines the project layout generated by using the `esp-idf-template` utility and its support for using Rust in CMake-based ESP-IDF projects.

Note that this article assumes prior knolwedge in the ESP-IDF build system (CMake).

## Prerequisites

Follow all the steps in the [CMake tutorial](../README-cmake.md). With the steps outlined there, generate a project named `test`. I.e.

```sh
cargo generate --vcs none --git https://github.com/esp-rs/esp-idf-template cmake --name test
```

## Generated Project Structure

Here's how your project directory might look after following the guide:

```
test/
|-- CMakeLists.txt
|-- main/
|   |-- CMakeLists.txt
|   |-- main.c
|-- sdkconfig
|-- components/
|   |-- rust-test/
|       |-- CMakeLists.txt
|       |-- placeholder.c
|       |-- build.rs
|       |-- Cargo.toml
|       |-- rust-toolchain.toml
|       |-- src/
|           |-- lib.rs
```

### Key Elements

- `test` contains the main C code like any other ESP-IDF application.
- An ESP-IDF component, with name `rust-test` is stored in a subdirectory named `components`.
- The Rust code is stored directly as part of the `rust-test` ESP-IDF component, i.e. the `rust-test` ESP-IDF component is simultaneously an ESP-IDF component as well as a Rust library crate.

The component can be uploaded later to [Component Manager](https://components.espressif.com/).

### How the Build Process Works

The CMake build for your `rust-test` component would proceed as usual and will compile all C code in the component (which is not much - just an empty `placeholder.c` file which needs to be there for the approach to work).

More importantly, `components/rust-test/CMakeLists.txt` is setup by the generator in such a way, so as to **also** call the Rust standard build utility - [`cargo`](https://doc.rust-lang.org/cargo/index.html) - which would take care of compiling the Rust source code in your ESP-IDF component.

The Rust source code (or as it is named in the Rust world - the Rust *crate*) is setup - in `components/rust-test/Cargo.toml` - to have the shape of a *static* library - just like any other ESP-IDF component. When the CMake build calls `cargo` to compile your `rust-test` Rust crate, you'll essentially end up with a static library named `librust_test.a`, which is then linked to the rest of the ESP-IDF project by CMake.

## Project Structure Details

We would be skipping over the project root (the `test/` top-level directory), as it is pretty standard and there is nothing Rust-specific in it. All the interesting stuff happens in the `components/rust-test` sub-folder. Before going into the details of each file, let's do the following separation:

```
|-- components/
|   |-- rust-test/
|       |-- CMakeLists.txt
|       |-- placeholder.c
|       ^^^ These two files are standard ESP-IDF CMake component build files, with CMakeLists.txt being slightly more complicated than usual.
|       |
|       |-- build.rs
|       |-- Cargo.toml
|       |-- rust-toolchain.toml
|       |-- src/
|           |-- lib.rs
|       ^^^ All these other files are what typically constitutes the layout of a Rust library crate and they are NOT specifc to ESP-IDF.
|           You can google any Rust source on the internet as to their purpose and meaning.
```

### ESP-IDF Component Portion

#### `components/rust-test/CMakeLists.txt`:

This file builds the `/components/rust-test` ESP-IDF component. It is however more complex than usual, as it needs to setup quite a few things so that calling into Rust `cargo` is running smoothly. The content of this file is the main boilerplate that the `esp-idf-template` automates.

You can of course modify this file post-generation. The various variables and overall behavior of the file is documented with inline comments.

#### `components/rust-test/placeholder.c`:

As mentioned previously, this is an empty C file which is only there so as the CMake->Cargo integration magic to work.

### Rust Library Crate Portion

#### `components/rust-test/build.rs`:

This file is a standard Rust [build scriptlet](https://doc.rust-lang.org/cargo/reference/build-scripts.html). It would be empty if you have not opted into the "HAL" option (note that by default "HAL" is enabled).

If you have selected the "HAL" option, the `build.rs` file will contain a call into the `embuild` library that nakes sure that your component can properly link against - and in general - can "talk" to the [Safe Rust bindings for ESP-IDF](https://github.com/esp-rs/esp-idf-svc) (the "HAL").

#### `components/rust-test/Cargo.toml`:

This is a pretty standard and mostly empty `cargo` [build manifest file](https://doc.rust-lang.org/cargo/reference/manifest.html), which sets up your Rust crate to be built as a Rust static libray.

If you have selected the "HAL" option when generating the project, the manifest would also list:
* a dependency to `esp-idf-svc` (i.e. [the safe Rust wrappers for ESP-IDF](https://github.com/esp-rs/esp-idf-svc));
* a built-time dependency to [`embuild`](https://github.com/esp-rs/embuild) which helps the build integration between ESP-IDF and Rust to run smoothly);
* the popular Rust [`log`](https://github.com/rust-lang/log) crate.

Post-generation, you can add/remove additional dependencies on Rust crates here as well as modify how Rust compiles your project (as in optimization level and others).

#### `components/rust-test/rust-toolchain.toml`:

This file instructs `cargo` [what Rust compiler toolchain to use](https://rust-lang.github.io/rustup/overrides.html#the-toolchain-file) when building your component. It would list either the `esp` toolchain (a Rust toolchain provided by Espressif that has extra support for the esp32 xtensa MCUs), or the standard Rust `nightly` toolchain. The latter can only be used with Espressif RiscV MCUs.

#### `components/rust-test/src/lib.rs`:

The actual source code of your Rust library. You can extend here at will.

## Troubleshooting

- If you encounter linker errors, you may need to update your Rust flags. For example, you might need to add the `-Zbuild-std-features=compiler-builtins-weak-intrinsics` flag to `CARGO_BUILD_FLAGS` in your `CMakeLists.txt`.

## Simulation

### Simulation with Wokwi in VS Code

[Wokwi for Visual Studio Code](https://docs.wokwi.com/vscode/getting-started) provides a simulation solution for embedded and IoT system engineers. The extension integrates with your existing development environment, allowing you to simulate your projects directly from your code editor.

- [Install VS Code](https://code.visualstudio.com/download)
- [Install the Wokwi plugin](https://docs.wokwi.com/vscode/getting-started#installation)
- Activate Wokwi plugin - command palette, search for `Wokwi: Start Simulator`, select and activate the plugin using web browser

### Add files for Wokwi simulator

Create [`wokwi.toml`](./wokwi.toml) in the root of the project. The file contains references to BIN and ELF previously built by `idf.py`.

```toml
[wokwi]
version = 1
elf = "build/test.elf"
firmware = "build/test.bin"
```

Create [`diagram.json`](./diagram.json). The file contains board selected for the simulation. [Here](https://github.com/wokwi/wokwi-boards/tree/main/boards)'s the list of available boards.

```json
{
  "version": 1,
  "author": "Espressif Systems",
  "editor": "wokwi",
  "parts": [ { "type": "wokwi-esp32-devkit-v1", "id": "esp", "top": 0, "left": 0, "attrs": {} } ],
  "connections": [ [ "esp:TX0", "$serialMonitor:RX", "", [] ], [ "esp:RX0", "$serialMonitor:TX", "", [] ] ],
  "dependencies": {}
}
```

Open VS Code, open command palette (CMD/Ctrl+Shift+P), search for `Wokwi: Start Simulator`, select the option to start simulation.

Use Pause button to display state of pins.

The plugin auto-reload application if the binary was updated.

## Adding GitHub Action Tests

If we want to add CI/CD to our project, we can leverage the [Wokwi CI](https://docs.wokwi.com/wokwi-ci/getting-started).
For example, the following YAML file will build and check that the generated project runs properly:

```yaml
name: CI
on:
  push:
  pull_request:
  workflow_dispatch:

env:
  CARGO_TERM_COLOR: always
  GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  ESP_TARGET: esp32
  ESP_IDF_VERSION: v5.3.2


jobs:
  build-check:
    name: Checks
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup | Rust (RISC-V)
        if: env.ESP_TARGET != 'esp32' && env.ESP_TARGET != 'esp32s2' && env.ESP_TARGET != 'esp32s3'
        uses: dtolnay/rust-toolchain@v1
        with:
          target: riscv32imc-unknown-none-elf
          toolchain: nightly
          components: rust-src
      - name: Setup | Rust (Xtensa)
        if: env.ESP_TARGET == 'esp32' && env.ESP_TARGET == 'esp32s2' && env.ESP_TARGET == 'esp32s3'
        uses: esp-rs/xtensa-toolchain@v1.6
        with:
          default: true
          buildtargets: {{ env.ESP_TARGET }}
          ldproxy: false
      - uses: Swatinem/rust-cache@v2
      - name: Setup | ESP-IDF
        shell: bash
        run: |
          git clone -b {{ env.ESP_IDF_VERSION }} --shallow-submodules --single-branch --recursive https://github.com/espressif/esp-idf.git /home/runner/work/esp-idf
          /home/runner/work/esp-idf/install.sh  {{ env.ESP_TARGET }}
      - name: Build project
        shell: bash
        run: |
          . /home/runner/work/esp-idf/export.sh
          idf.py set-target  {{ env.ESP_TARGET }}
          idf.py build
      - name: Wokwi CI check
        uses: wokwi/wokwi-ci-action@v1
        with:
          token: ${{ secrets.WOKWI_CLI_TOKEN }}
          timeout: 10000
          expect_text: 'Hello ESP-RS. https://github.com/esp-rs'
          fail_text: 'Error'
```

Its important to note that we need to set the `WOKWI_CLI_TOKEN` secret:
* [Create a Wokwi CI token](https://wokwi.com/dashboard/ci);
* [Add it as a secret in your GitHub repository](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions).

Also, the CI file needs to be modified if used for other targets.
