# 🦀 rustymicrobit

> **Learning bare-metal Rust on the BBC micro:bit v2, one abstraction layer at a time.** 🔬➡️🧩➡️🛠️➡️🎁

This repo is a hands-on learning journal. The goal is simple: **blink one LED** 💡. It is done **four times**, each time climbing one level of abstraction, so you can *see* what every layer of the embedded Rust ecosystem does for you:

```text
 🔬 Raw registers  ──►  🧩 PAC  ──►  🛠️ HAL  ──►  🎁 BSP
  (unsafe pointers)    (generated    (safe, typed   (the board
                        registers)    pins + traits) knows its pins)
```

Each step is its own commit and pull request, so you can check out any lesson and flash it.

## 📺 Video series followed

This repo follows the **embedded Rust video series** by [**The Rusty Bits**](https://www.youtube.com/@therustybits) on YouTube 🙌. Watch the videos, then come back here to see the code for each step:

| Episode | Video | Lessons in this repo |
| --- | --- | --- |
| 1️⃣ | 🎬 [Embedded Rust setup explained](https://www.youtube.com/watch?v=TOAynddiu5M) | 🧰 [Setup](#setup), 🗂️ [project files](#project-files), 🐞 [Lesson 0: debugger](#lesson-0) |
| 2️⃣ | 🎬 [Blinking an LED: Embedded Rust ecosystem explored](https://www.youtube.com/watch?v=A9wvA_S6m7Y) | 🔬 [Raw](#lesson-1-raw) → 🧩 [PAC](#lesson-2-pac) → 🛠️ [HAL](#lesson-3-hal) → 🎁 [BSP](#lesson-4-bsp) |

> 🙏 All credit for the teaching goes to The Rusty Bits. This repo is my own hands-on notes, plus the [problems I hit along the way](#-troubleshooting-errors-i-actually-hit).

---

## 📚 Table of contents

- [📺 Video series followed](#-video-series-followed)
- [🧰 Hardware](#-hardware)
- [🏷️ Toolchain versions used](#versions)
- [⚙️ Setup from zero](#setup)
- [🗂️ Project files explained](#project-files)
- [🚀 Build, flash and debug](#-build-flash-and-debug)
- [🐞 Debugging with GDB](#-debugging-with-gdb)
- [🎓 The lessons](#-the-lessons)
  - [0️⃣ Lesson 0: hello debugger](#lesson-0)
  - [1️⃣ Lesson 1: raw registers](#lesson-1-raw)
  - [2️⃣ Lesson 2: PAC](#lesson-2-pac)
  - [3️⃣ Lesson 3: HAL](#lesson-3-hal)
  - [4️⃣ Lesson 4: BSP](#lesson-4-bsp)
- [📊 Side-by-side comparison](#-side-by-side-comparison)
- [🧹 Dependency hygiene with cargo-machete](#-dependency-hygiene-with-cargo-machete)
- [🩺 Troubleshooting (errors I actually hit)](#-troubleshooting-errors-i-actually-hit)
- [🔗 Resources](#-resources)
- [📄 License](#-license)

---

## 🧰 Hardware

| | |
| --- | --- |
| 🟩 Board | **BBC micro:bit v2** |
| 🧠 MCU | Nordic **nRF52833**, ARM **Cortex-M4F** @ 64 MHz (with hardware floating point) |
| 💾 Memory | **512 KB** flash, **128 KB** RAM |
| 🔌 Debug probe | On-board (CMSIS-DAP over USB), no extra hardware needed |
| 💡 LED used | Top-left LED of the 5×5 matrix: **ROW1 = P0.21**, **COL1 = P0.28** |

💡 **How the LED matrix works:** the LEDs aren't wired to a single pin each. Every LED sits between a *row* pin and a *column* pin. An LED lights up when its **row is HIGH ⬆️** and its **column is LOW ⬇️**. All four lessons keep COL1 low and toggle ROW1.

> ⚠️ This repo targets the **v2** board only. The v1 board uses a different chip (nRF51822, Cortex-M0) with a different target triple, memory layout and crates.

---

<a id="versions"></a>

## 🏷️ Toolchain versions used

Everything in this repo was built and flashed with these exact versions:

| Tool | Version |
| --- | --- |
| 🦀 `rustc` | `1.99.1 (58cf8f9c3 2026-10-13)` |
| 📦 `cargo` | `1.99.1 (5f94df478 2026-08-27)` |
| 🔧 `rustup` | `1.29.2 (d8c92c676 2026-10-21)` |
| 📝 Rust edition | `2024` |
| 🎯 Target | `thumbv7em-none-eabihf` |
| 🔌 `probe-rs` / `cargo-embed` / `cargo-flash` | `0.32.0` |
| 🐞 `arm-none-eabi-gdb` | `GNU gdb 18.1` (Homebrew) |
| 🧹 `cargo-machete` | `0.9.2` |
| 🧩 VS Code: rust-analyzer | `0.3.3089` |
| 🧩 VS Code: Even Better TOML | `0.21.2` |
| 💻 Host | macOS 27 on Apple Silicon (`aarch64-apple-darwin`) |

Check yours with:

```sh
rustc --version && cargo --version && rustup --version && probe-rs --version
```

---

<a id="setup"></a>

## ⚙️ Setup from zero

### 1. 🦀 Install Rust with rustup

```sh
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

> 🍺 **Homebrew users:** if you *also* have the Homebrew `rust` formula installed, remove it (`brew uninstall rust`). Otherwise its `cargo`/`rustc` shadow rustup's, and you'll get `can't find crate for core`. See [Troubleshooting](#-troubleshooting-errors-i-actually-hit).

### 2. 🎯 Add the Cortex-M4F target

The nRF52833 is a Cortex-M4 **with** a floating-point unit, so we use the `eabihf` (hard-float) variant:

```sh
rustup target add thumbv7em-none-eabihf
```

### 3. 🔌 Install the probe-rs tools (flash + debug)

```sh
curl --proto '=https' --tlsv1.2 -LsSf https://github.com/probe-rs/probe-rs/releases/latest/download/probe-rs-tools-installer.sh | sh
```

This gives you `probe-rs`, `cargo embed` and `cargo flash`.

> ⚠️ Don't run `cargo install cargo-embed`. That crate is now an empty placeholder that fails on purpose and points you to probe-rs.

Check that the board is detected (plug it in via USB first 🔌):

```sh
probe-rs list
```

### 4. 🐞 Install a debugger (optional, see [Debugging with GDB](#-debugging-with-gdb))

```sh
brew install arm-none-eabi-gdb
```

📏 Optionally, `cargo-binutils` gives you `cargo size`, `cargo objdump`, `cargo nm`… to inspect the firmware:

```sh
cargo install cargo-binutils
rustup component add llvm-tools
```

### 5. 🧑‍💻 VS Code

Extensions:

- **rust-analyzer** (`rust-lang.rust-analyzer`): completion, types, errors
- **Even Better TOML** (`tamasfe.even-better-toml`): `Cargo.toml` / `Embed.toml` support

The repo ships a [`.vscode/settings.json`](.vscode/settings.json):

```json
{
  "rust-analyzer.check.allTargets": false,
  "rust-analyzer.cargo.target": "thumbv7em-none-eabihf",
  "cSpell.words": [
    "microbit",
    "PINCNF",
    "rprintln"
  ]
}
```

- `cargo.target`: makes rust-analyzer check the code **for the microcontroller**, not for your Mac. Without it, you'd see bogus errors about `std` and `#![no_std]`.
- `check.allTargets = false`: stops it from trying to build tests and benches, which don't exist in a `no_std` binary.
- `cSpell.words`: just tells the spell-checker extension that these embedded words are fine. ✍️

### 6. 📦 Dependencies (`cargo add`)

These are the commands used across the lessons. Each lesson below says which ones it needs:

```sh
# 🧱 Base (every lesson)
cargo add cortex-m --features critical-section-single-core
cargo add cortex-m-rt
cargo add panic-halt
cargo add rtt-target                # added early, removed in Lesson 3

# 🧩 Lesson 2: PAC
cargo add nrf52833-pac

# 🛠️ Lesson 3: HAL
cargo add nrf52833-hal
cargo add embedded-hal

# 🎁 Lesson 4: BSP
cargo add microbit-v2
```

| Crate | Why it's here |
| --- | --- |
| `cortex-m` | Low-level access to the Cortex-M core (e.g. `asm::nop()`). The `critical-section-single-core` feature provides a critical-section implementation for single-core chips. |
| `cortex-m-rt` | The **runtime**: vector table, reset handler, RAM init, and the `#[entry]` macro. It also provides `link.x`, the linker script. |
| `panic-halt` | `no_std` programs must define what happens on `panic!`. This one just halts in an infinite loop. |
| `nrf52833-pac` | Typed access to every register of the nRF52833. |
| `nrf52833-hal` | Safe drivers (GPIO, timers…) built on top of the PAC. |
| `embedded-hal` | Standard **traits** (`OutputPin`, `DelayNs`…) that drivers and HALs share across all chips. |
| `microbit-v2` | Board support: knows which pin is which on the micro:bit v2. ⚠️ Its library name is **`microbit`**, so you write `use microbit::…`. |

---

<a id="project-files"></a>

## 🗂️ Project files explained

```text
rustymicrobit/
├── .cargo/config.toml     🎯 default target + linker script
├── .vscode/settings.json  🧑‍💻 rust-analyzer → embedded target
├── Cargo.toml             📦 dependencies
├── Embed.toml             🔌 cargo-embed config (chip, gdb, rtt)
├── memory.x               🧠 where flash and RAM live
└── src/main.rs            🦀 the program
```

### 🎯 `.cargo/config.toml`

```toml
[build]
target = "thumbv7em-none-eabihf"

[target.thumbv7em-none-eabihf]
rustflags = ["-C", "link-arg=-Tlink.x"]
```

- `target`: plain `cargo build` builds for the micro:bit; no `--target` flag needed.
- `-Tlink.x`: tells the linker to use `cortex-m-rt`'s linker script, which in turn includes our `memory.x`.

### 🧠 `memory.x`

```text
MEMORY
{
  FLASH : ORIGIN = 0x00000000, LENGTH = 512K
  RAM : ORIGIN = 0x20000000, LENGTH = 128K
}
```

This tells the linker the nRF52833's memory map: code goes to flash at `0x0000_0000`, variables to RAM at `0x2000_0000`.

> ⚠️ Keep the **space before the colon** (`FLASH : ORIGIN`). `FLASH:` is read as one name and the linker fails. See [Troubleshooting](#-troubleshooting-errors-i-actually-hit).

### 🔌 `Embed.toml`

```toml
[default.general]
chip = "nRF52833_xxAA"

[default.rtt]
enabled = false

[default.gdb]
enabled = true

[default.reset]
halt_afterwards = true
```

- `chip`: the exact chip name probe-rs uses (find it with `probe-rs chip list | grep -i nrf52833`).
- `gdb.enabled`: `cargo embed` starts a GDB server on **`127.0.0.1:1337`**.
- `reset.halt_afterwards = true`: after flashing, the CPU **stops** and waits for the debugger.

> ⚠️ **Heads-up:** because of `halt_afterwards = true` (set up for Lesson 0), the LED **won't blink** after `cargo embed` until you attach GDB and `continue`. To just run the program, use `cargo flash` (below) or set `halt_afterwards = false`.

---

## 🚀 Build, flash and debug

```sh
cargo build                                          # 🔨 compile
cargo size                                           # 📏 how big is it? (needs cargo-binutils)
cargo flash --chip nRF52833_xxAA                     # ⚡ flash and run
cargo embed                                          # 🔌 flash + start the GDB server (uses Embed.toml)
```

- ⚡ **Just want to see it run?** Use `cargo flash`.
- 🐞 **Want to step through the code?** Use `cargo embed` + GDB (next section).

---

## 🐞 Debugging with GDB

This project is already set up for on-chip debugging. The micro:bit's built-in probe lets you pause the CPU, step line by line, read variables and even peek at hardware registers, all over the same USB cable. 🔍

### 🧩 How the pieces fit

```text
┌──────────────────┐   GDB protocol    ┌────────────────────┐   USB (CMSIS-DAP)   ┌──────────────┐
│ arm-none-eabi-gdb│ ◄──────────────►  │    cargo embed     │ ◄─────────────────► │  micro:bit   │
│   (terminal 2)   │  127.0.0.1:1337   │ (GDB server, term 1)│                     │  nRF52833 🧠 │
└──────────────────┘                   └────────────────────┘                     └──────────────┘
```

The three settings in [`Embed.toml`](Embed.toml) that make this work:

| Setting | What it does |
| --- | --- |
| `[default.gdb] enabled = true` | `cargo embed` starts a **GDB server** on `127.0.0.1:1337` |
| `[default.reset] halt_afterwards = true` | after flashing, the CPU is **paused at reset**, so you don't miss the first lines |
| `[default.rtt] enabled = false` | no RTT log window (we don't print anything yet) |

### ▶️ Step by step

**1. Terminal 1:** build, flash, and start the GDB server. Leave it running. 🏃

```sh
cargo embed
```

**2. Terminal 2:** start GDB with the **debug** ELF (it has the debug symbols that map machine code back to your Rust lines):

```sh
arm-none-eabi-gdb -q target/thumbv7em-none-eabihf/debug/rustymicrobit
```

**3. Inside GDB:**

```gdb
(gdb) target remote :1337     # 🔌 connect to cargo embed
(gdb) monitor reset halt      # 🔄 restart the chip and keep it paused
(gdb) break main              # 🛑 breakpoint at the entry point
(gdb) continue                # ▶️ run until we hit it
(gdb) step                    # 👣 enter our real main (see note below)
(gdb) next                    # ⏭️ run one line
(gdb) info locals             # 📋 show local variables (x, is_on, …)
(gdb) continue                # ▶️ let it run: the LED blinks 💡
                              #    press Ctrl+C to pause it again ⏸️
```

> 💡 **Why `step` after `break main`?** `#[entry]` renames your function to `__cortex_m_rt_main` and generates a tiny `main` that just calls it. `break main` stops in that wrapper, and `step` takes you into your code. You can also break on your code directly with `break src/main.rs:12` (file:line).

### 🧰 Handy GDB commands

| Command | What it does |
| --- | --- |
| `break src/main.rs:20` · `info breakpoints` · `delete 1` | 🛑 set / list / remove breakpoints |
| `continue` · `Ctrl+C` | ▶️ run · ⏸️ pause |
| `next` · `step` · `finish` | ⏭️ next line · 👣 into a call · ⤴️ out of a function |
| `print is_on` · `set var is_on = true` | 🔍 read · ✏️ change a variable |
| `info locals` · `backtrace` | 📋 locals · 🧵 call stack |
| `info registers` | 🧠 CPU registers (`pc`, `sp`, `r0`…) |
| `x/1xw 0x50000504` | 🔬 read GPIO **`OUT`**: watch bit 21 flip as the LED blinks! |
| `x/1xw 0x50000754` | 🔬 read **`PIN_CNF[21]`**: bit 0 = `1` means output (see note below 👇) |
| `layout src` | 🖥️ split screen with the source code (`tui disable` to exit) |
| `monitor reset` · `monitor reset halt` | 🔄 restart the chip (and keep it paused) |
| `detach` · `quit` | 👋 let the chip run freely · leave GDB |

🧪 **Fun experiment:** in **any** lesson, set a breakpoint inside the `loop`, then run `x/1xw 0x50000504` after each `continue`. You'll see bit 21 (`0x00200000`) toggle: the same register Lesson 1 writes by hand, whichever layer (PAC, HAL, BSP) you use. 🤯

🔎 **Bonus puzzle:** `x/1xw 0x50000754` (`PIN_CNF[21]`) reads **`0x1`** in Lessons 1, 3 and 4 but **`0x3`** in Lesson 2. Why? Bit 1 is `INPUT` (1 = input buffer *disconnected*), and its **reset value is 1** (the register resets to `0x2`). The PAC's `.write()` starts from that reset value and only sets `dir`, so you get `0x2 | 0x1 = 0x3`. Lesson 1 writes a plain `1`, and the HAL explicitly sets `input().connect()`, so both give `0x1`. Same LED, different bits: that's why it pays to know what each layer does. 🧠

> ⚡ **Tip:** skip typing the connect command each time:
>
> ```sh
> arm-none-eabi-gdb -q -ex "target remote :1337" target/thumbv7em-none-eabihf/debug/rustymicrobit
> ```
>
> 🐢 **Debug vs release:** stick to `cargo build` (debug) for stepping. In `--release`, the optimizer reorders and removes code, so lines jump around and variables show as `<optimized out>`.

---

## 🎓 The lessons

📺 Lessons 1–4 follow [Episode 2: Blinking an LED: Embedded Rust ecosystem explored](https://www.youtube.com/watch?v=A9wvA_S6m7Y).

Every lesson is **one pull request** 🔀 merged into `main`, in order. Open a PR to see *exactly* what changed from the previous layer: the diff **is** the lesson. 📖

| # | Lesson | Pull request 🔀 | Branch 🌿 | Commit |
| --- | --- | --- | --- | --- |
| 0️⃣ | 🐞 Hello debugger | (initial commit) | n/a | `5d2d577` |
| 1️⃣ | 🔬 How to blink with **raw** registers | [#1](https://github.com/christiangda/rustymicrobit/pull/1) | `feat/blink-a-led-raw` | `a72baf0` |
| 2️⃣ | 🧩 How to blink with the **PAC** | [#2](https://github.com/christiangda/rustymicrobit/pull/2) | `feat/blink-a-led-pac` | `c1d887f` |
| 3️⃣ | 🛠️ How to blink with the **HAL** | [#3](https://github.com/christiangda/rustymicrobit/pull/3) | `feat/blink-a-led-hal` | `ebd5b88` |
| 4️⃣ | 🎁 How to blink with the **BSP** | [#4](https://github.com/christiangda/rustymicrobit/pull/4) | `feat/blink-a-led-bsp` | `231f648` |

▶️ **To run any lesson on your board:**

```sh
git checkout <commit>              # e.g. git checkout a72baf0  (🔬 raw)
cargo flash --chip nRF52833_xxAA   # ⚡ flash it and watch the LED
git checkout main                  # 🔙 come back to the latest lesson
```

🔍 **To compare two layers** (e.g. what the HAL changed compared to the PAC):

```sh
git diff c1d887f ebd5b88 -- src/main.rs Cargo.toml
```

---

<a id="lesson-0"></a>

### 0️⃣ Lesson 0: hello debugger

> 🎯 **Goal:** get a `no_std` program onto the chip and step through it with GDB.
>
> 📺 **Video:** [Episode 1: Embedded Rust setup explained](https://www.youtube.com/watch?v=TOAynddiu5M)
>
> 📌 **Commit:** `5d2d577` (the initial commit, before the PRs) · ▶️ `git checkout 5d2d577 && cargo embed`, then follow [🐞 Debugging with GDB](#-debugging-with-gdb)

```rust
#![no_std]
#![no_main]

use cortex_m::asm::nop;
use cortex_m_rt::entry;
use panic_halt as _;

#[entry]
fn main() -> ! {
    let mut x: usize = 0;
    loop {
        x += 1;
        for _ in 0..x {
            nop();
        }
    }
}
```

🧠 **What to notice**

- `#![no_std]`: no standard library. There's no OS, so no files, threads or heap, only `core`.
- `#![no_main]`: there's no normal `main` called by an OS. `cortex-m-rt`'s `#[entry]` marks the function the reset handler jumps to.
- `fn main() -> !`: the `!` ("never") return type means **this function never returns**. On a microcontroller there's nothing to return *to*.
- `use panic_halt as _;`: links a panic handler without naming it.
- Flash it with `cargo embed`, connect GDB, and watch `x` grow with `info locals`. 🔍

---

<a id="lesson-1-raw"></a>

### 1️⃣ Lesson 1: raw registers (unsafe + pointers)

> 🎯 **Goal:** blink the LED by writing **directly to hardware memory addresses**, no helper crates.
>
> 🔀 **PR:** [#1 feat(led): blink the led using unsafe and raw calls directly](https://github.com/christiangda/rustymicrobit/pull/1) · 🌿 `feat/blink-a-led-raw` · 📌 `a72baf0`
>
> ▶️ **Try it:** `git checkout a72baf0 && cargo flash --chip nRF52833_xxAA`
>
> 📦 **Dependencies:** none new.

Peripherals on a microcontroller are controlled through **memory-mapped registers**: special addresses where reading and writing talks to hardware instead of RAM. The nRF52833's GPIO port 0 (`P0`) starts at **`0x5000_0000`**. From the datasheet:

| Register | Offset | Address | Purpose |
| --- | --- | --- | --- |
| `OUT` | `0x504` | `0x5000_0504` | One bit per pin: 1 = HIGH, 0 = LOW |
| `PIN_CNF[n]` | `0x700 + 4·n` | `PIN_CNF[21]` = `0x5000_0754`<br>`PIN_CNF[28]` = `0x5000_0770` | Configure pin *n* (bit 0 = direction: 1 = output) |

```rust
#![no_std]
#![no_main]

use core::ptr::write_volatile;

use cortex_m::asm::nop;
use cortex_m_rt::entry;
use panic_halt as _;

#[entry]
fn main() -> ! {
    const GPIO0_PINCNF21_ROW1_ADDR: *mut u32 = 0x5000_0754 as *mut u32;
    const GPIO0_PINCNF28_COL1_ADDR: *mut u32 = 0x5000_0770 as *mut u32;
    const DIR_OUTPUT_POS: u32 = 0;
    const PINCNF_DRIVE_LED: u32 = 1 << DIR_OUTPUT_POS;

    unsafe {
        write_volatile(GPIO0_PINCNF21_ROW1_ADDR, PINCNF_DRIVE_LED);
        write_volatile(GPIO0_PINCNF28_COL1_ADDR, PINCNF_DRIVE_LED);
    }

    const GPIO0_OUT_ADDR: *mut u32 = 0x5000_0504 as *mut u32;
    const GPIO0_OUT_ROW1_POS: u32 = 21;

    let mut is_on: bool = false;

    loop {
        unsafe {
            write_volatile(GPIO0_OUT_ADDR, (is_on as u32) << GPIO0_OUT_ROW1_POS);
        }

        for _ in 0..400_000 {
            nop();
        }

        is_on = !is_on;
    }
}
```

🧠 **What to notice**

- 🔧 **Two steps:** first make P0.21 and P0.28 **outputs** (`PIN_CNF`), then toggle bit 21 of `OUT` in a loop. Writing the whole `OUT` register also keeps bit 28 (COL1) at **0 = LOW**, so the LED can light.
- ⚠️ **`unsafe`:** writing to an arbitrary address can break anything, so Rust makes *you* vouch for it. The compiler can't check the address is right.
- 🌀 **`write_volatile`:** a normal write to memory that's "never read" could be optimized away. Volatile tells the compiler: *this write has side effects, always do it, in this order.*
- ⏳ **`nop()` busy-loop:** a crude delay. The speed depends on the CPU clock **and** on debug vs `--release` builds.
- 😬 **The pain:** magic numbers copied from a PDF, bit shifting by hand, no type checking. One wrong digit and you configure the wrong peripheral.

---

<a id="lesson-2-pac"></a>

### 2️⃣ Lesson 2: PAC (Peripheral Access Crate)

> 🎯 **Goal:** same blink, but registers have **names and types** instead of addresses.
>
> 🔀 **PR:** [#2 feat(led): blink the led using PAC](https://github.com/christiangda/rustymicrobit/pull/2) · 🌿 `feat/blink-a-led-pac` · 📌 `c1d887f`
>
> ▶️ **Try it:** `git checkout c1d887f && cargo flash --chip nRF52833_xxAA`
>
> 📦 **Dependencies:** `cargo add nrf52833-pac`

A **PAC** is generated automatically (with `svd2rust`) from the vendor's **SVD file**, an XML description of every peripheral, register and bit field on the chip. No hand-written magic numbers anymore.

```rust
#![no_std]
#![no_main]

use cortex_m::asm::nop;
use cortex_m_rt::entry;
use nrf52833_pac::Peripherals;
use panic_halt as _;

#[entry]
fn main() -> ! {
    let p: Peripherals = Peripherals::take().unwrap();
    p.P0.pin_cnf[21].write(|w| w.dir().output());
    p.P0.pin_cnf[28].write(|w| w.dir().output());

    let mut is_on: bool = false;

    loop {
        p.P0.out.write(|w| w.pin21().bit(is_on));
        for _ in 0..200_000 {
            nop();
        }

        is_on = !is_on;
    }
}
```

🧠 **What to notice**

- ✅ **No `unsafe`** in our code. 🎉
- 🔐 **`Peripherals::take()`** returns `Some` only the **first** time it's called (a *singleton*). Exactly one owner of the hardware can exist, so two parts of the program can't fight over the same register.
- ✍️ **Closure-based writes:** `.write(|w| w.dir().output())` gives you a writer `w`. You set **named fields** (`dir`, `pin21`) to **named values** (`output()`), and the PAC does the address math and bit shifting.
- 🔁 `write` starts from the register's **reset value**, so every bit you don't set goes back to its default. There's also `.modify(|r, w| …)` for read-modify-write and `.read()` for reading.
- 😐 **Still low-level:** you still think in *registers* and *pin numbers*. The PAC doesn't know what a "pin" or an "LED" is.

---

<a id="lesson-3-hal"></a>

### 3️⃣ Lesson 3: HAL (Hardware Abstraction Layer)

> 🎯 **Goal:** use **safe, typed pins** instead of registers.
>
> 🔀 **PR:** [#3 feat(led): blink the led using HAL](https://github.com/christiangda/rustymicrobit/pull/3) · 🌿 `feat/blink-a-led-hal` · 📌 `ebd5b88`
>
> ▶️ **Try it:** `git checkout ebd5b88 && cargo flash --chip nRF52833_xxAA`
>
> 📦 **Dependencies:** `cargo add nrf52833-hal` · `cargo add embedded-hal` (and `nrf52833-pac` / `rtt-target` removed: the HAL re-exports the PAC as `hal::pac`)

```rust
#![no_std]
#![no_main]

use cortex_m::asm::nop;
use cortex_m_rt::entry;
use embedded_hal::digital::{OutputPin, PinState};
use hal::{gpio::Level, pac};
use nrf52833_hal as hal;
use panic_halt as _;

#[entry]
fn main() -> ! {
    let p = pac::Peripherals::take().unwrap();
    let port0 = hal::gpio::p0::Parts::new(p.P0);
    let _col1 = port0.p0_28.into_push_pull_output(Level::Low);
    let mut row1 = port0.p0_21.into_push_pull_output(Level::Low);

    let mut is_on: bool = false;

    loop {
        let _ = row1.set_state(PinState::from(is_on));

        for _ in 0..100_000 {
            nop();
        }

        is_on = !is_on;
    }
}
```

🧠 **What to notice**

- 🧩 **`Parts::new(p.P0)`** *consumes* the raw `P0` peripheral and splits it into **32 individual pin objects** (`p0_00` … `p0_31`) that you can hand out separately.
- 🏷️ **Typestate pins:** `port0.p0_21` starts life as a *disconnected* pin. `.into_push_pull_output(Level::Low)` **converts its type** into an output pin. Calling `set_high()` on an *input* pin is a **compile error**, not a runtime bug. 🛡️
- 🔌 **`embedded-hal` traits:** `set_state`, `set_high` and `set_low` come from the `OutputPin` **trait**, so the trait must be imported (`use embedded_hal::digital::OutputPin`). Any driver written against `OutputPin` works on **any** chip whose HAL implements it, not just nRF.
- 🤷 `let _ = …`: these methods return a `Result`. For nRF GPIO the error type is `Infallible` (it can't fail), so it's safe to ignore.
- 😐 **Still:** *you* must know that ROW1 is P0.21 and COL1 is P0.28, and the delay is still a `nop` loop.

---

<a id="lesson-4-bsp"></a>

### 4️⃣ Lesson 4: BSP (Board Support Package)

> 🎯 **Goal:** talk about the **board** ("row 1 of the display"), not the chip ("pin 21"), and use a **real timer** for delays.
>
> 🔀 **PR:** [#4 feat(led): blink the led using Board Support Package (bsp)](https://github.com/christiangda/rustymicrobit/pull/4) · 🌿 `feat/blink-a-led-bsp` · 📌 `231f648` (this is what's on `main` today)
>
> ▶️ **Try it:** `git checkout main && cargo flash --chip nRF52833_xxAA`
>
> 📦 **Dependencies:** `cargo add microbit-v2` (and `cortex-m` removed: no more `nop` loops)

```rust
#![no_std]
#![no_main]

use cortex_m_rt::entry;
use embedded_hal::{delay::DelayNs, digital::OutputPin};
use microbit::{board::Board, hal::timer::Timer};
use panic_halt as _;

#[entry]
fn main() -> ! {
    let mut board = Board::take().unwrap();
    let mut timer = Timer::new(board.TIMER0);
    let _ = board.display_pins.col1.set_low();
    let mut row1 = board.display_pins.row1;

    loop {
        let _ = row1.set_low();
        timer.delay_ms(100);
        let _ = row1.set_high();
        timer.delay_ms(1000);
    }
}
```

🧠 **What to notice**

- 🎁 **`Board::take()`** gives you every peripheral **already configured for the micro:bit**, with names from the board's schematic: `display_pins.row1`, `display_pins.col1`, buttons, etc. The BSP already set `row1`/`col1` up as push-pull outputs on P0.21/P0.28. No pin numbers in your code!
- ⏱️ **Hardware timer:** `Timer::new(board.TIMER0)` + the `DelayNs` trait (`delay_ms`) give **real milliseconds**, independent of debug/release builds and compiler optimizations. Goodbye `nop` loops 👋
- 💡 **Blink pattern:** `set_high` = row HIGH = **LED on** for 1000 ms, `set_low` = **off** for 100 ms.
- 🧱 **Layers stack:** the BSP uses the **HAL**, which uses the **PAC**, which writes the same **registers** from Lesson 1. You can still reach every lower layer (`microbit::hal`, `microbit::pac`) when you need it.

---

## 📊 Side-by-side comparison

| | 🔬 Raw | 🧩 PAC | 🛠️ HAL | 🎁 BSP |
| --- | --- | --- | --- | --- |
| `unsafe` in your code | ⚠️ Yes | ✅ No | ✅ No | ✅ No |
| You need register addresses | ⚠️ Yes | ✅ No | ✅ No | ✅ No |
| You need pin numbers | ⚠️ Yes | ⚠️ Yes | ⚠️ Yes | ✅ No (named pins) |
| Compile-time pin-mode checks | ❌ | ❌ | ✅ typestate | ✅ typestate |
| Portable code (`embedded-hal`) | ❌ | ❌ | ✅ | ✅ |
| Delay | `nop` loop | `nop` loop | `nop` loop | ⏱️ `TIMER0` |
| Crate you add | none | `nrf52833-pac` | `nrf52833-hal` + `embedded-hal` | `microbit-v2` |
| You learn… | how hardware *really* works | how registers are organized | safe ownership & traits | productive app code |

🧭 **Rule of thumb:** write apps at the **BSP/HAL** level, and drop down to **PAC** (or raw) only when the higher layer doesn't expose what you need. Knowing all four makes debugging much easier. 🧠

---

## 🧹 Dependency hygiene with cargo-machete

Cargo has no built-in `go mod tidy`. [`cargo-machete`](https://github.com/bnjbvr/cargo-machete) finds unused dependencies:

```sh
cargo install cargo-machete
cargo machete          # 🔍 report unused deps
cargo machete --fix    # ✂️ remove them from Cargo.toml
```

> ⚠️ **Don't run `--fix` blindly.** `cargo-machete` searches your source for each crate's name. On `main` today it reports:
>
> - `microbit-v2`: ❌ **false positive.** The crate's library name is `microbit`, so `use microbit::…` doesn't match. Tell machete to ignore it:
>
>   ```toml
>   [package.metadata.cargo-machete]
>   ignored = ["microbit-v2"]
>   ```
>
> - `nrf52833-hal`: ✅ **genuinely unused** since Lesson 4 (the BSP re-exports it as `microbit::hal`). Safe to remove with `cargo remove nrf52833-hal`.

For an exact (compiler-based) check, there's also `cargo +nightly udeps` ([cargo-udeps](https://github.com/est31/cargo-udeps)).

---

## 🩺 Troubleshooting (errors I actually hit)

<details>
<summary>❌ <code>error[E0463]: can't find crate for `core`</code> even after <code>rustup target add</code></summary>

**Cause:** two Rust installs. Homebrew's `rust` formula (`/opt/homebrew/bin/cargo`) came first in `PATH`, so it shadowed rustup's. Homebrew's Rust only has the standard library for your Mac, and `rustup target add` can't add targets to it.

**Fix:**

```sh
brew uninstall rust     # keep the `rustup` formula if you installed rustup via brew
hash -r                 # or open a new terminal
which cargo rustc       # should point to ~/.cargo/bin/
```

</details>

<details>
<summary>❌ <code>rust-lld: error: memory.x:3: : expected, but got ORIGIN</code></summary>

**Cause:** `FLASH: ORIGIN = …`. In linker scripts a `:` can be part of a name, so `FLASH:` is read as one word.

**Fix:** put a space before the colon: `FLASH : ORIGIN = …`.

</details>

<details>
<summary>❌ rust-analyzer: <code>proc macro server error … rustc version mismatch</code></summary>

**Cause:** rust-analyzer was still using the old (Homebrew) toolchain while `cargo` now uses rustup's.

**Fix:** VS Code → `Cmd+Shift+P` → **Developer: Reload Window**. If it persists: `cargo clean` and reload again.

</details>

<details>
<summary>❌ <code>cargo install cargo-embed</code> panics: "cargo-embed is now a part of probe-rs"</summary>

**Cause:** the `cargo-embed` crate is now an empty placeholder that fails on purpose.

**Fix:** install the probe-rs tools with the installer from [Setup step 3](#3--install-the-probe-rs-tools-flash--debug).

</details>

<details>
<summary>❌ <code>cargo embed --list-chips</code>: unexpected argument</summary>

**Cause:** chip listing moved to the `probe-rs` command.

**Fix:**

```sh
probe-rs chip list | grep -i nrf52833
probe-rs chip info nRF52833_xxAA
```

</details>

<details>
<summary>❌ <code>cannot find type `W` in this scope</code></summary>

**Cause:** closures written as `|w: &mut W| …`. That's usually a rust-analyzer *inlay hint* that got inserted into the code by double-clicking it. `W` lives in each register's module (e.g. `nrf52833_pac::p0::out::W`) and isn't imported.

**Fix:** drop the annotation and let Rust infer it: `.write(|w| w.dir().output())`.

</details>

<details>
<summary>❌ <code>no method named `set_state` found for struct `P0_21&lt;MODE&gt;`</code></summary>

**Cause:** `set_state` is a method of the `OutputPin` **trait**, and trait methods only work when the trait is in scope.

**Fix:**

```sh
cargo add embedded-hal
```

```rust
use embedded_hal::digital::{OutputPin, PinState};   // ✅ 1.0 traits, not `digital::v2`
```

</details>

<details>
<summary>🤔 I ran <code>cargo embed</code> but the LED doesn't blink</summary>

**Cause:** `Embed.toml` has `halt_afterwards = true`, so the CPU stops after flashing and waits for GDB.

**Fix:** attach GDB and `continue`, or use `cargo flash --chip nRF52833_xxAA`, or set `halt_afterwards = false`.

</details>

---

## 🔗 Resources

- 📺 **The Rusty Bits** video series ([channel](https://www.youtube.com/@therustybits)):
  - 🎬 Episode 1: [Embedded Rust setup explained](https://www.youtube.com/watch?v=TOAynddiu5M)
  - 🎬 Episode 2: [Blinking an LED: Embedded Rust ecosystem explored](https://www.youtube.com/watch?v=A9wvA_S6m7Y)
- 📘 [The Embedded Rust Book](https://docs.rust-embedded.org/book/)
- 🧭 [Discovery book: micro:bit v2 edition](https://docs.rust-embedded.org/discovery-mb2/)
- 🔌 [probe-rs documentation](https://probe.rs/docs/)
- 🎁 [`microbit-v2` crate docs](https://docs.rs/microbit-v2) · 🛠️ [`nrf52833-hal`](https://docs.rs/nrf52833-hal) · 🧩 [`nrf52833-pac`](https://docs.rs/nrf52833-pac) · 🔌 [`embedded-hal`](https://docs.rs/embedded-hal)
- 🟩 [micro:bit v2 hardware & schematic](https://tech.microbit.org/hardware/)
- 🧠 [Nordic nRF52833 product page (datasheet)](https://www.nordicsemi.com/Products/nRF52833)

---

## 📄 License

Licensed under the [Apache License, Version 2.0](LICENSE). 📜

Happy hacking! 🦀💡
