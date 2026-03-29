# Embedded Wasm 

> See also: [Embedded Hacking](https://github.com/mytechnotalent/Embedded-Hacking) — A FREE comprehensive step-by-step embedded hacking course covering Embedded Software Development to Reverse Engineering.

A collection of repos that runs a WebAssembly Component Model runtime (wasmtime + Pulley interpreter) directly on bare-metal w/ HW capabilities exposed through WIT.

<br>

## How It All Fits Together

Every repo in this collection shares the same architecture: a bare-metal Rust firmware that boots an RP2350 microcontroller, compiles a WebAssembly component ahead of time, and executes it through the Wasmtime runtime's Pulley interpreter. The WASM guest calls back into the host for hardware access (GPIO, UART, timers) through interfaces defined in WIT.

The dependency stack has three layers — **hardware**, **WASM runtime**, and **glue** — and each crate plays a specific role.

### Hardware Layer

These crates give the firmware direct access to the RP2350's Cortex-M33 core and peripherals.

- **[cortex-m](https://docs.rs/cortex-m)** — Low-level access to ARM Cortex-M registers, interrupts, and intrinsics. The firmware uses it to disable/enable interrupts and access core peripherals like the SysTick timer.

- **[cortex-m-rt](https://docs.rs/cortex-m-rt)** — The Cortex-M startup runtime. It provides the reset vector, initializes `.bss` and `.data` sections, and calls `main()`. The `#[entry]` attribute macro marks the firmware's entry point.

- **[rp235x-hal](https://docs.rs/rp235x-hal)** — The RP2350 Hardware Abstraction Layer. It wraps the chip's raw registers into safe Rust types for clocks, GPIO pins, UART peripherals, and more. Every hardware operation — toggling an LED, reading a button, sending a UART byte — goes through this crate.

- **[embedded-hal](https://docs.rs/embedded-hal)** — A set of traits that define portable hardware interfaces (GPIO, UART, SPI, I2C). The firmware codes against these traits so the same logic works across different HALs. `rp235x-hal` implements these traits for the RP2350.

- **[fugit](https://docs.rs/fugit)** — Type-safe time units for embedded systems. Baud rates (`115_200.Hz()`) and clock frequencies (`150.MHz()`) are expressed as `fugit` types, catching unit mismatches at compile time.

- **[critical-section](https://docs.rs/critical-section)** — Cross-platform interrupt-safe mutual exclusion. The firmware uses it to protect shared global state (UART handle, GPIO pins) from interrupt races. `rp235x-hal` provides the platform-specific critical section implementation.

### WASM Runtime Layer

These crates compile, encode, and execute WebAssembly components on the microcontroller.

- **[Wasmtime](https://docs.wasmtime.dev)** ([API docs](https://docs.rs/wasmtime)) — The WebAssembly runtime. At build time, Wasmtime's Cranelift backend compiles the guest `.wasm` into Pulley bytecode. At run time, Wasmtime's engine deserializes this precompiled component and executes it through the Pulley interpreter. The `Engine`, `Store`, `Component`, and `Linker` types orchestrate the entire lifecycle.

- **[Cranelift](https://cranelift.dev)** ([API docs](https://docs.rs/cranelift-codegen)) — The compiler backend used during `build.rs` to AOT-compile WASM into Pulley bytecode. It runs on the host machine (not the MCU) and produces a serialized artifact that gets embedded into the firmware binary.

- **[Pulley](https://docs.rs/pulley-interpreter)** — Wasmtime's portable interpreter bytecode format. Instead of compiling WASM to native ARM instructions, the pipeline compiles to Pulley bytecode, which Wasmtime's built-in interpreter executes on any architecture — including the Cortex-M33. This is what makes WASM on a microcontroller possible without a native code generator for Thumb-2.

- **[WIT / Component Model](https://component-model.bytecodealliance.org)** ([wit-bindgen docs](https://docs.rs/wit-bindgen)) — The interface definition language that describes what the WASM guest can import from the host (e.g., `gpio.set-high`, `button.is-pressed`, `timing.delay-ms`). `wit-bindgen` generates guest-side Rust bindings, and Wasmtime's `bindgen!` macro generates host-side bindings. The `wit-component` crate encodes a core WASM module into a Component Model component during the build.

### Glue Layer

- **[embedded-alloc](https://docs.rs/embedded-alloc)** — A heap allocator for `no_std` environments. Wasmtime needs heap allocation for its `Store`, `Engine`, and component instantiation. The firmware carves out a fixed-size heap from RAM and registers this allocator as the `#[global_allocator]`.

### The Build-to-Boot Pipeline

The crates come together in this order:

1. **`wit/world.wit`** defines the hardware interfaces using the Component Model IDL
2. **`build.rs`** compiles the WASM guest app, encodes it as a component (`wit-component`), and AOT-compiles it to Pulley bytecode (`Cranelift` via `Wasmtime`)
3. **`cortex-m-rt`** boots the Cortex-M33 core and calls `main()`
4. **`main()`** initializes clocks (`rp235x-hal`), UART (`embedded-hal` traits), GPIO pins, and the heap (`embedded-alloc`)
5. **`Wasmtime`** deserializes the precompiled Pulley bytecode and instantiates the component
6. The WASM guest's `run()` function executes, calling back into the host through WIT-defined imports
7. Each host call drives real hardware through `rp235x-hal`, protected by `critical-section`

<br>

## Repos

## embedded-wasm-uart-rp2350
A pure Embedded Rust UART echo project that runs a WebAssembly Component Model runtime (Wasmtime + Pulley interpreter) directly on the RP2350 bare-metal w/ HW capabilities exposed through WIT.

-> Click [HERE](https://github.com/mytechnotalent/embedded-wasm-uart-rp2350) to access the repo.

## embedded-wasm-blinky-rp2350
A pure Embedded Rust blinky project that runs a WebAssembly Component Model runtime (Wasmtime + Pulley interpreter) directly on the RP2350 bare-metal w/ HW capabilities exposed through WIT.

-> Click [HERE](https://github.com/mytechnotalent/embedded-wasm-blinky-rp2350) to access the repo.

## embedded-wasm-button-rp2350
A pure Embedded Rust button project that runs a WebAssembly Component Model runtime (Wasmtime + Pulley interpreter) directly on the RP2350 bare-metal with hardware capabilities exposed through WIT.

-> Click [HERE](https://github.com/mytechnotalent/embedded-wasm-button-rp2350) to access the repo.

<br>

## License

- [MIT License](LICENSE)
