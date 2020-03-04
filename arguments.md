# Xous Arguments

Xous arguments are passed by giving a page-aligned address pointed to
by register `$a0`.  This uses IFF-style tags with the following format:

```rust
struct Tag {
    /// Ascii-printable name, not null-terminated, in little endian format.
    tag: u32,

    /// CRC16 of the data section, using CCITT polynomial.
    crc16: u16,

    /// Size of the data section, in 4-byte words.
    size: u16,
}
```

The size is in little endian byte order, and does not include the 8
bytes describing the tag type or the size.  The data must be padded to
4-bytes, so the `size` field is in 4-byte words.  That is, a tag with
four bytes of contents would have a `size` of `1`.

The `XArg` block **must** be first.

## Tag Types

| Tag | Description
| ---- | ------------
| XArg | Xous Args.  This indicates the size of the entire block (including the XArg tag and size), as well as the user RAM area, and can be used to determine how many pages to save for this block.  This must be the first argument.
| BFlg | Bootloader config flags.
| MREx | Extra memory ranges.  This is a series of offset/size pairs indicating additional memory regions in the system beyond RAM, as well as a code name for the memory page.  It does not include system RAM.
| XKrn | Kernel source specification.  Includes the offset of the kernel in RAM as well as its size.  Does not need to be page-aligned, unless NO_COPY is 1.
| Init | Initial program specification.  This includes the offset of PID1 in RAM as well as its size.  Does not need to be page-aligned unless NO_COPY is 1.  May appear more than once.

### XArg

The overall size of the argument block.  This must come first.  This tag
has four words of contents, which is the number of words of data plus
the system memory definition.  Therefore, a minimum boot tag structure
would have an `XArg` size of 5: One word for the tag, one for the
crc+size, and four for the contents of the `XArg` region.

| Offset | Size | Name | Description
| ------ | ---- | ---- | -----------
|    0   |   4  | Arg Size | The size of the entire args structure, including all headers, but excluding any trailing data (such as executables)
|    4   |   4  | Version   | Version of the XArg structure.  Currently `1`.
|    8   |   4  | RAM Start | The origin of system RAM, in bytes
|    12   |   4  | RAM Size  | The size of system RAM, in bytes
|    16   |   4  | RAM Name  | A printable name for system RAM

### Bflg

This configures various bootloader flags.  It consists of a single word
of data.

* 0x00000001 `NO_COPY`  -- Skip copying data to RAM.
* 0x00000002 `ABSOLUTE` -- All program addresses are absolute.
  Otherwise, they're relative to the start of the config block.
* 0x00000004 `DEBUG`    -- Allow the kernel to access memory inside user
  programs, which allows a debugger to run in the kernel.

### MREx

Extra memory regions.  See [memory.md](memory.md) for more information.

### Init

The `Init` argument describes how to load initial processes.  It has the
following values:

* LOAD_OFFSET -- Position in RAM relative to the start of the arguments
  block where this program is stored
* TEXT_OFFSET -- Virtual memory address where this program expects the
  program image to live
* TEXT_SIZE -- Size (in bytes) of the text section, not including this header
* DATA_OFFSET -- Virtual memory address where this program expects the
  .data/.bss section to be
* DATA_SIZE -- Size of the .data section
* BSS_SIZE -- Size of the .bss section, assumed to immediately follow .data
* ENTRYPOINT - Virtual memory address of the `_start()` function

Note that the .bss and .data sections are combined together.  This
merely indicates how pages will get allocated for this new program.
Also note that these sections will be cleared to 0 due to how Xous
processes start up.

The bootloader will copy `LOAD_SIZE` bytes of data from `LOAD_OFFSET` to
a new series of pages in memory, which will be mapped to `TEXT_OFFSET`.
Additionally, `DATA_SIZE` pages will be allocated at `DATA_OFFSET`, plus
a single page at `STACK`.

Init programs **cannot** access the first 4 megabytes, as this memory
is reserved for the kernel.

### XKrn

This describes the kernel image.  This image will get mapped into every
process within the final 4 megabytes, and therefore the text and data
offsets must be in the range `0xffc0_0000` - `0xfff0_0000`.

* LOAD_OFFSET -- Physical address (or offset) where the kernel is stored
* TEXT_OFFSET -- Virtual memory address where the kernel expects the
  program image to live.  This should be `0xffd00000`
* TEXT_SIZE -- Size of the text section.  This indicates how many bytes
  to copy from the boot image.
* DATA_OFFSET -- Virtual memory address where the kernel expects the
  .data/.bss section to be.  This should be above `0xffd00000` and below
  `0xffe00000`
* DATA_SIZE -- Size of the .data section
* BSS_SIZE -- The size of the .bss section, which immediately follows .data
* ENTRYPOINT -- Virtual address of the `_start()` function

The kernel will run in Supervisor mode, and have its own private stack.
