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

### The WASM Compilation Pipeline

The build produces three distinct WASM artifacts. Each one is a transformation of the previous, and understanding all three is essential to understanding how code written in Rust ends up executing on a microcontroller through a WebAssembly runtime.

#### Artifact 1: Core WASM Module (`.wasm`)

**What it is:** A standard WebAssembly module compiled from the guest Rust source code.

**Where it lives:** `wasm-app/target/wasm32-unknown-unknown/release/wasm_app.wasm`

**How it is produced:** `build.rs` spawns `cargo build --release --target wasm32-unknown-unknown` inside the `wasm-app/` directory. The Rust compiler (`rustc`) compiles `wasm-app/src/lib.rs` into a core WASM module. Because the guest uses `wit-bindgen`, the compiler also embeds Component Model type metadata as a custom section inside this `.wasm` file. This metadata describes the WIT imports and exports the guest expects.

**What is inside:** Standard WASM sections — types, functions, tables, memories, globals, exports, and code. Plus custom sections injected by `wit-bindgen` that describe the component type signature. At this stage, the file is a plain core module — it has no Component Model envelope yet.

**How to inspect it:**
```bash
# Install wasm-tools (one-time)
cargo install wasm-tools

# Print the WAT (WebAssembly Text Format) — human-readable disassembly
wasm-tools print wasm-app/target/wasm32-unknown-unknown/release/wasm_app.wasm

# Validate the module
wasm-tools validate wasm-app/target/wasm32-unknown-unknown/release/wasm_app.wasm

# Dump raw section headers and sizes
wasm-tools dump wasm-app/target/wasm32-unknown-unknown/release/wasm_app.wasm

# Show the component type metadata embedded by wit-bindgen
wasm-tools metadata show wasm-app/target/wasm32-unknown-unknown/release/wasm_app.wasm
```

#### Artifact 2: Component-Encoded WASM (in-memory only)

**What it is:** A WASM Component Model component that wraps the core module with typed imports and exports matching the WIT definition.

**Where it lives:** Only in memory — inside `build.rs` as the `component_bytes` variable in `compile_wasm_to_pulley()`. It is never written to disk because it is immediately passed to the Cranelift compiler.

**How it is produced:** `build.rs` reads the core `.wasm` file (Artifact 1) and feeds it to `wit_component::ComponentEncoder`. The encoder reads the custom sections that `wit-bindgen` embedded, validates them against the WIT definition, and wraps the core module in a Component Model envelope. The result is a fully-typed component with:
- **Typed imports** — `embedded:platform/gpio.set-high(pin: u32)`, `embedded:platform/timing.delay-ms(ms: u32)`, etc.
- **Typed exports** — `run: func()`
- **Canonical ABI adapters** — shim functions that translate between the Component Model's typed interface and the core module's raw integer-based calling convention

The `ComponentEncoder` is the library version of `wasm-tools component new`. Using it as a library avoids a subprocess call and keeps the entire pipeline in-process.

**What is inside:** A Component Model binary — it contains the original core module nested inside, plus component-level type sections, import/export declarations, and canonical lifting/lowering adapters.

**How to inspect it (if you write it to disk for debugging):**
```bash
# You can temporarily add this line to build.rs to dump it:
# std::fs::write(out.join("component.wasm"), &component_bytes).unwrap();

# Then inspect the component
wasm-tools print component.wasm
wasm-tools component wit component.wasm    # extract the WIT back out
wasm-tools validate --features component-model component.wasm
```

#### Artifact 3: Precompiled Pulley Bytecode (`.cwasm`)

**What it is:** A serialized, AOT-compiled blob of Pulley interpreter bytecode. This is the final artifact — the one that actually runs on the RP2350.

**Where it lives:** `$OUT_DIR/blinky.cwasm` (e.g., `target/thumbv8m.main-none-eabihf/release/build/embedded-wasm-blinky-<hash>/out/blinky.cwasm`). It is embedded directly into the firmware ELF binary via `include_bytes!` in `main.rs`:
```rust
const WASM_BINARY: &[u8] = include_bytes!(concat!(env!("OUT_DIR"), "/blinky.cwasm"));
```

**How it is produced:** `build.rs` creates a Wasmtime `Engine` configured to target `pulley32` (the 32-bit Pulley interpreter). It then calls `engine.precompile_component(&component_bytes)` which invokes Cranelift to compile every function in the component (Artifact 2) into Pulley bytecode. The result is serialized into a platform-independent blob.

The engine configuration is critical — it must exactly match the runtime engine on the MCU:
- `target("pulley32")` — compile for the 32-bit Pulley interpreter, not native ARM
- `signals_based_traps(false)` — bare-metal has no OS signal handlers for traps
- `memory_init_cow(false)` — no virtual memory, no copy-on-write page tricks
- `memory_reservation(0)` — no virtual address space to reserve (embedded target)
- `memory_guard_size(0)` — no guard pages (no MMU/MPU-based trap detection)
- `guard_before_linear_memory(false)` — no pre-guard pages
- `max_wasm_stack(16384)` — 16 KiB WASM stack (constrained by available RAM)

If the build-time and run-time engine configurations do not match, deserialization will fail at boot with a version/config mismatch error.

**What is inside:** Serialized Wasmtime metadata plus compiled Pulley bytecode for every function. This is not a standard WASM format — it is a Wasmtime-internal serialization format. You cannot inspect it with `wasm-tools`.

**How to inspect it:**
```bash
# .cwasm files are Wasmtime-internal format, not standard WASM
# You can check the file size to verify it was produced
ls -lh $OUT_DIR/blinky.cwasm

# To verify it loads correctly, the firmware itself is the test —
# Engine::deserialize() at boot will reject any corrupt or mismatched blob
```

### The Build-to-Boot Sequence

Here is the complete sequence from source code to blinking LED, showing exactly which crate does what and when:

**Phase 1: WIT Definition (developer writes once)**

The developer defines the hardware interface contract in `wit/world.wit`:
```wit
package embedded:platform;

interface gpio {
    set-high: func(pin: u32);
    set-low: func(pin: u32);
}

interface timing {
    delay-ms: func(ms: u32);
}

world blinky {
    import gpio;
    import timing;
    export run: func();
}
```

This single file drives code generation on both sides — the guest uses `wit-bindgen` to generate Rust import stubs, and the host uses Wasmtime's `bindgen!` to generate Rust trait implementations.

**Phase 2: Build Script (`cargo build` triggers `build.rs`)**

When you run `cargo build`, Cargo executes `build.rs` on the host machine before compiling the firmware. The build script runs four steps in sequence:

1. **Write linker script** — copies `rp2350.x` (the RP2350 memory layout: Flash at `0x10000000`, RAM at `0x20000000`) into `$OUT_DIR/memory.x` so `cortex-m-rt` knows where to place code and data

2. **Compile WASM guest** — spawns `cargo build --release --target wasm32-unknown-unknown` inside `wasm-app/`. Rust compiles the guest code + `wit-bindgen` stubs into a core WASM module (Artifact 1)

3. **Encode as component** — `ComponentEncoder` from `wit-component` wraps the core module into a Component Model component (Artifact 2, in-memory only)

4. **AOT-compile to Pulley** — Wasmtime's `Engine` (with Cranelift backend) compiles the component into Pulley bytecode and writes `blinky.cwasm` to `$OUT_DIR/` (Artifact 3)

**Phase 3: Firmware Compilation (`rustc` compiles `src/main.rs`)**

After `build.rs` finishes, Cargo compiles the firmware source. The `include_bytes!` macro bakes the `.cwasm` blob directly into the firmware's `.rodata` section. The result is a single ELF binary that contains both the ARM firmware code and the precompiled WASM bytecode.

**Phase 4: Flashing**

`probe-rs download --chip RP235x firmware.elf` writes the ELF to the RP2350's flash memory at `0x10000000`.

**Phase 5: Boot (`cortex-m-rt` starts the Cortex-M33)**

On power-up, the RP2350's Boot ROM reads the image definition from the `.start_block` section, verifies it, and jumps to the reset vector. `cortex-m-rt` takes over:
1. Initializes `.bss` (zero-fills) and `.data` (copies from Flash to RAM)
2. Calls the function marked `#[entry]` — which is `main()`

**Phase 6: Hardware Initialization (`main()` in `src/main.rs`)**

`main()` brings up the hardware in order:
1. `init_heap()` — carves 256 KiB from RAM and registers `embedded-alloc` as `#[global_allocator]`
2. `init_clocks()` — configures the RP2350's PLLs and clock tree to 150 MHz via `rp235x-hal`
3. `init_hardware()` — initializes UART0 (GPIO0/GPIO1 at 115200 baud) and GPIO25 (the onboard LED) via `rp235x-hal` + `embedded-hal` traits
4. Calls `run_wasm()` to enter the WASM execution loop

**Phase 7: WASM Execution (`Wasmtime` runs the guest)**

`run_wasm()` sets up and executes the WASM component:
1. `create_engine()` — creates a Wasmtime `Engine` with the same `pulley32` configuration used at build time
2. `Engine::deserialize()` — loads the precompiled `.cwasm` bytes (the `WASM_BINARY` constant) back into a `Component`. No compilation happens here — the bytecode is ready to execute
3. `build_linker()` — creates a `Linker` and registers host implementations for every WIT import (`gpio.set-high`, `gpio.set-low`, `timing.delay-ms`)
4. `execute_wasm()` — creates a `Store` (which holds the host state), instantiates the component through the linker, and calls the guest's exported `run()` function

**Phase 8: Guest Execution (Pulley interpreter runs the guest code)**

The guest's `run()` function executes inside the Pulley interpreter. It enters an infinite loop:
1. Calls `gpio.set-high(25)` -> crosses the WASM boundary -> host drives GPIO25 high via `rp235x-hal`
2. Calls `timing.delay-ms(500)` -> crosses the WASM boundary -> host spins for 500 ms
3. Calls `gpio.set-low(25)` -> crosses the WASM boundary -> host drives GPIO25 low
4. Calls `timing.delay-ms(500)` -> crosses the WASM boundary -> host spins for 500 ms
5. Back to step 1

Each "crosses the WASM boundary" means the Pulley interpreter hits a host-call instruction, suspends guest execution, calls the host function registered in the `Linker`, and resumes the guest when it returns.

### Inspecting WASM Artifacts

Install the [`wasm-tools`](https://github.com/bytecodealliance/wasm-tools) CLI to inspect and debug the WASM artifacts produced by the build:

```bash
cargo install wasm-tools
```

| Command                                                | What it does                                                                |
| ------------------------------------------------------ | --------------------------------------------------------------------------- |
| `wasm-tools print <file>.wasm`                         | Disassemble to WAT (human-readable text format)                             |
| `wasm-tools validate <file>.wasm`                      | Validate a core module or component                                         |
| `wasm-tools dump <file>.wasm`                          | Dump raw binary sections and offsets                                        |
| `wasm-tools component wit <file>.wasm`                 | Extract the WIT interface from a component                                  |
| `wasm-tools metadata show <file>.wasm`                 | Show embedded metadata (producers, component type)                          |
| `wasm-tools component new core.wasm -o component.wasm` | Manually encode a core module as a component (what `ComponentEncoder` does) |
| `wasm-tools stats <file>.wasm`                         | Show size breakdown by section                                              |

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
