# axdtb

A no_std Device Tree Binary (DTB) parser implementation.

This crate provides functionality to parse Device Tree Binary (DTB) files in a no_std environment. The parser supports DTB format version 17 and provides a safe interface to traverse the device tree structure while extracting property values.

## Examples

```rust
#![no_std]
#![no_main]

#[macro_use]
extern crate axlog2;
extern crate alloc;
use alloc::string::String;
use alloc::vec::Vec;

use core::panic::PanicInfo;

#[no_mangle]
pub extern "Rust" fn runtime_main(_cpu_id: usize, dtb_pa: usize) {
    axlog2::init("info");
    info!("[rt_axdtb]: ...");

    axalloc::init();

    test_dtb(dtb_pa);

    info!("[rt_axdtb]: ok!");
    axhal::misc::terminate();
}

#[cfg(target_arch = "riscv64")]
fn test_dtb(dtb_pa: usize) {
    let mut cb = |name: String,
                  _addr_cells: usize,
                  _size_cells: usize,
                  props: Vec<(String, Vec<u8>)>| {
        match name.as_str() {
            "chosen" => {
                for prop in props {
                    match prop.0.as_str() {
                        "bootargs" => {
                            if let Ok(cmd) = core::str::from_utf8(&prop.1) {
                                let cmd = cmd.trim_end_matches(char::from(0));
                                assert!(cmd.len() > 0);
                                assert!(cmd.starts_with("init="));
                                let cmd = cmd.strip_prefix("init=").unwrap();
                                assert!(cmd == "/sbin/init" || cmd == "/btp/sbin/hello");
                            }
                        }
                        _ => (),
                    }
                }
            },
            _ => (),
        }
    };

    axdtb::parse(dtb_pa, &mut cb);
}

#[cfg(not(target_arch = "riscv64"))]
fn test_dtb(_dtb_pa: usize) {
}

#[panic_handler]
pub fn panic(info: &PanicInfo) -> ! {
    error!("{}", info);
    arch_boot::panic(info)
}

```

## Structs

### `DeviceTree`

```rust
pub struct DeviceTree {
    pub off_struct: usize,
    /* private fields */
}
```

Main structure representing a Device Tree Binary.
Contains information about the DTB header and provides methods to parse the tree structure.

## Enums

### `DeviceTreeError`

```rust
pub enum DeviceTreeError {
    BadMagicNumber,
    SliceReadError,
    VersionNotSupported,
    ParseError(usize),
    Utf8Error,
}
```

Represents possible errors that can occur during DTB parsing.

## Traits

### `SliceRead`

```rust
pub trait SliceRead {
    // Required methods
    fn read_be_u32(&self, pos: usize) -> DeviceTreeResult<u32>;
    fn read_be_u64(&self, pos: usize) -> DeviceTreeResult<u64>;
    fn read_bstring0(&self, pos: usize) -> DeviceTreeResult<&[u8]>;
    fn subslice(&self, start: usize, len: usize) -> DeviceTreeResult<&[u8]>;
}
```

A trait for safely reading binary data from a slice.
This trait provides methods to read big-endian integers and null-terminated strings from a byte slice, with bounds checking to ensure memory safety.

## Functions

### `parse`

```rust
pub fn parse<F>(dtb_va: usize, cb: F)
where
    F: FnMut(String, usize, usize, Vec<(String, Vec<u8>)>),
```

Convenience function to parse a DTB and process its nodes.
