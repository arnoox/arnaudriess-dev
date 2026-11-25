+++
title = "\"Rust is safe\" stops at extern \"C\""
date = 2026-11-25
description = "Rust's guarantees depend on the declarations at the C boundary, and the compiler cannot check them. Two small bugs show what that means in practice."
+++

When a team adds Rust to an existing C code base, someone often says that the code is now memory safe. For the new Rust code, this is mostly true. At the point where Rust calls C, it is not true. In a mixed code base, this boundary carries most of the risk, and the compiler cannot see across it.

<!-- more -->

## The compiler believes what you write

An `extern "C"` block is a promise. You tell the Rust compiler that a function exists with this name, these arguments and this return type. The compiler cannot check this promise. It never sees the C header, and the linker only compares symbol names. If the promise is wrong, the program still compiles and links. All Rust guarantees then depend on a false statement.

Here is the header of a small sensor driver:

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

And here is a Rust binding that a developer wrote by hand:

```rust
#[repr(C)]
#[derive(Clone, Copy, Debug, PartialEq)]
pub enum SensorMode { Off = 0, Idle = 1, Run = 2 }

unsafe extern "C" {
    fn sensor_mode() -> SensorMode;
    fn sensor_read(channel: u8, out: *mut i32) -> i32;
}
```

The code compiles and links, and the tests pass. But both declarations are wrong.

## Bug 1: the integer width

The header declares `int16_t *out`. The binding declares `*mut i32`. C writes two bytes, and Rust reads four bytes.

```rust
let mut raw: i32 = 0;
let rc = unsafe { sensor_read(0, &mut raw) };
```

On a little-endian target, with `raw` set to zero before the call, every positive value is correct. This is why the tests pass. Then the sensor reports -5. C writes `0xFFFB` into the two low bytes, the two high bytes stay zero, and Rust reads 65531. The program does not crash and shows no warning. The wrong value goes directly into the control logic.

The linker cannot detect this error, because a C symbol is only a name. In my hazard catalogue, this hazard is `HAZARD_NO_LINK_TIME_TYPE_CHECK`.

## Bug 2: the enum

The second bug is more dangerous, although it looks harmless. Rust and C both have enums, so it seems natural to mirror one with the other. But the two enums do not mean the same thing. In C, an enum is an integer, and it can hold any integer value. A Rust `SensorMode` can only have the values 0, 1 or 2. Any other value causes undefined behaviour as soon as Rust reads it.

Now the supplier releases a firmware update that adds `SENSOR_FAULT = 3`. Nobody changes the Rust code, because nothing seems to require it.

```rust
match unsafe { sensor_mode() } {
    SensorMode::Off  => power_down(),
    SensorMode::Idle => wait(),
    SensorMode::Run  => sample(),
}
```

The match covers all three variants, so the compiler accepts it. The compiler may also assume that one of these three branches always runs, and it may optimise the code based on this assumption. The fault state is the case where a safe reaction is most important, and it is the one case that this code cannot express. The behaviour of the program is then not defined, and it can be different in debug and release builds. (`HAZARD_INVALID_VALUE_REPRESENTATION`)

Note where the bug is. The `match` is safe Rust. One line in the `extern` block created a false assumption, and the safe code relies on it.

## A correct version

The fix is simple. Accept from C only what C actually guarantees, and convert the values at the boundary:

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

The new firmware value now arrives as an `Err`, and the caller must handle it. You can test this code path and discuss it in a review. You cannot do either with undefined behaviour.

One limitation: `c_int` assumes that your C compiler stores this enum as an `int`. Most compilers do this, but the C standard does not guarantee it. Check it on your toolchain.

## The Rust language agrees

Since the 2024 edition, Rust requires `unsafe extern "C"` instead of `extern "C"`. The keyword is not there because a call to C is dangerous. It is there because the declaration itself is a claim, and only you can confirm that it is true. The language now shows the warning at the same place where this post locates the problem.

## What this means for a safety case

"We use Rust" is a good argument for the code inside the crate. It says nothing about the boundary. An assessor will ask why your declarations are correct on your compiler and your target. "bindgen generated them" is only a part of the answer. A later post will explain why.

Three rules keep the boundary under control:

- **Keep `unsafe` in one place.** One thin layer communicates with C and checks all values. The code above this layer uses `#![forbid(unsafe_code)]`. The next post shows the three layers of a binding.
- **Treat every declaration as a claim that needs evidence.** This includes sizes, signedness, offsets and enum values. Check them on the real toolchain and target, not only on a development machine.
- **Convert at the boundary.** Raw integers and pointers come in, and validated Rust types go out.

Both bugs are entries in my public catalogue of Rust ↔ C hazards. The catalogue has nineteen entries so far. Each entry has a stable ID that you can use in coding guidelines and review checklists. <!-- TODO: link the catalogue once it is public -->

Rust is safe up to the last line that the compiler can check. In a mixed code base, this line is your `extern` block.
