+++
title = "\"Rust is safe\" stops at extern \"C\""
date = 2026-11-25
description = "Rust's guarantees rest on the declarations at the C boundary, and the compiler cannot check them. Two small bugs show what that means in practice."
+++

When a team brings Rust into an existing C code base, someone usually says: "And now it's memory safe." For the new Rust code, mostly yes. At the line where Rust calls C, no. In a mixed code base that line carries most of the risk, and the compiler can't see across it.

<!-- more -->

## The compiler believes what you write

An `extern "C"` block is a promise. You tell the Rust compiler that a function exists with this name, these arguments and this return type. The compiler can't check it. It never sees the C header, and the linker only matches symbol names. If the promise is wrong, the program still compiles and links, and every Rust guarantee is built on a false statement.

Here is a small sensor driver header:

```c
// sensor.h
typedef enum {
    SENSOR_OFF  = 0,
    SENSOR_IDLE = 1,
    SENSOR_RUN  = 2,
} sensor_mode_t;

sensor_mode_t sensor_mode(void);
int32_t       sensor_read(uint8_t channel, int16_t *out);
```

And a Rust binding that someone wrote by hand on a Friday afternoon:

```rust
#[repr(C)]
#[derive(Clone, Copy, Debug, PartialEq)]
pub enum SensorMode { Off = 0, Idle = 1, Run = 2 }

unsafe extern "C" {
    fn sensor_mode() -> SensorMode;
    fn sensor_read(channel: u8, out: *mut i32) -> i32;
}
```

It compiles. It links. The tests pass. Both declarations are wrong.

## Bug one: the width

The header says `int16_t *out`. The binding says `*mut i32`. C writes two bytes, Rust reads four.

```rust
let mut raw: i32 = 0;
let rc = unsafe { sensor_read(0, &mut raw) };
```

On a little-endian target, with `raw` set to zero first, every positive reading comes back correct. That is why the tests pass. Then the sensor reports -5. C writes `0xFFFB` into the low two bytes, the high two stay zero, and Rust reads 65531. No crash, no warning. Just a wrong number on its way into your control logic.

The linker said nothing because it can't. A C symbol is only a name. In my hazard catalogue this is `HAZARD_NO_LINK_TIME_TYPE_CHECK`.

## Bug two: the enum

This one is worse, and it looks harmless. Rust and C both have enums, so mirroring one with the other feels natural. They don't mean the same thing. In C an enum is an integer, and any integer fits. A Rust `SensorMode` can only ever be 0, 1 or 2. Any other value is not just odd. It is undefined behaviour the moment Rust reads it.

Now the supplier ships a firmware update that adds `SENSOR_FAULT = 3`. Nobody touches the Rust side. Why would they?

```rust
match unsafe { sensor_mode() } {
    SensorMode::Off  => power_down(),
    SensorMode::Idle => wait(),
    SensorMode::Run  => sample(),
}
```

The match is exhaustive, so the compiler is happy. It is also allowed to assume that one of these three arms always runs, and to optimise on that. The fault state, the one case where a safe reaction matters most, is the one case your code cannot express. What happens next is anyone's guess, and it can differ between debug and release builds. (`HAZARD_INVALID_VALUE_REPRESENTATION`)

Look at where the bug lives. The `match` is safe Rust. One line in the `extern` block planted a false assumption, and safe code acted on it.

## A correct version

The fix isn't clever. Accept from C what C actually guarantees, and convert at the door:

```rust
use core::ffi::c_int;

unsafe extern "C" {
    fn sensor_mode() -> c_int;
    fn sensor_read(channel: u8, out: *mut i16) -> i32;
}

pub enum SensorMode { Off, Idle, Run }

pub struct UnknownMode(pub c_int);

pub fn mode() -> Result<SensorMode, UnknownMode> {
    match unsafe { sensor_mode() } {
        0 => Ok(SensorMode::Off),
        1 => Ok(SensorMode::Idle),
        2 => Ok(SensorMode::Run),
        other => Err(UnknownMode(other)),
    }
}
```

The new firmware value now arrives as an `Err` that the caller must handle. That is a code path you can test and discuss in a review. Undefined behaviour is neither.

One caveat: `c_int` assumes your C compiler stores this enum as an `int`. Most do, the standard doesn't promise it. Check it on your toolchain.

## Even Rust says so now

Since the 2024 edition, Rust makes you write `unsafe extern "C"` instead of plain `extern "C"`. The keyword is not there because calling C is dangerous. It is there because the declaration itself is a claim that only you can vouch for. The language put the warning exactly where this post says the problem is.

## What this means for a safety case

"We use Rust" is a good argument for the code inside the crate. It says nothing about the boundary. An assessor will ask what makes your declarations correct on your compiler and your target. "bindgen generated them" is part of an answer, not the whole one. More on that in a later post.

Three habits keep the boundary under control:

- **Keep `unsafe` in one place.** One thin layer talks to C and checks everything. The code above it uses `#![forbid(unsafe_code)]`. The next post shows the three layers of a binding.
- **Treat every declaration as a claim that needs evidence.** Sizes, signedness, offsets, enum values. Checked on the real toolchain and target, not on a laptop.
- **Convert at the door.** Raw integers and pointers come in, validated Rust types go out.

Both bugs above are entries in my public catalogue of Rust ↔ C hazards. It has nineteen so far, each with a stable ID you can cite in guidelines and review checklists. <!-- TODO: link the catalogue once it is public -->

Rust is safe up to the last line the compiler can check. In a mixed code base, that line is your `extern` block.
