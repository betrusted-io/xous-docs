# Kernel Arguments

Kernel arguments are passed by giving a page-aligned address pointed to by
register `$a0`.  This uses IFF-style tags with the following format:

```rust
struct Tag {
    /// Ascii-printable namem, not null-terminated, in little endian format.
    tag: u32,

    /// Reserved for a crc16 in a future version.
    _reserved: u16,

    /// Size of the data section, in 4-byte words.
    size: u16,
}
```

The size is in little endian byte order, and does not include the 8 bytes describing the tag type or the size.  The data must be padded to 4-bytes,
so the `size` field is in 4-byte words.  That is, a tag with four bytes of
contents would have a `size` of `1`.

## Tag Types

| Tag | Description
| ---- | ------------
| XASZ | Xous Args Size.  This indicates the size of the entire block (including the XASZ tag and size), and can be used to determine how many pages to save for this block.  This should be the first argument.
| BFlg | Bootloader config flags.
| MBLK | Memory page blocks.  This is a series of offset/size pairs indicating memory regions in the system, as well as a code name for the memory page.  The first region is defined to be "RAM", and is where memory will be allocated from.
| XKRN | Kernel memory specification.  Includes the offset of the kernel in RAM as well as its size.  Does not need to be page-aligned.
| Init | Initial program specification.  This includes the offset of PID1 in RAM as well as its size.  Does not need to be page-aligned.  May appear more than once.

### XASZ

The overall size of the argument block.  This must come first.  This tag
has one word of contents, which is the number of words of data.  Therefore,
a minimum boot tag structure would have an `XASZ` size of 3: One word for the
tag, one for the crc+size, and one for the contents of the `XASZ` region.

### Bflg

This configures various bootloader flags.

* 0x00000001 `NO_COPY` -- Set this flag to skip copying data to RAM.
* 0x00000002 `ABSOLUTE` -- If set, all addresses are absolute.  Otherwise, they're relative to the start of the config block.

### MBLK

Define memory block regions.  See [memory.md](memory.md) for more information.

### Init

The `Init` argument describes how to load initial processes.  It has the following values:

* LOAD_OFFSET -- Position in RAM relative to the start of the arguments block where this program is stored
* LOAD_SIZE -- Size (in bytes) of the binary, not including this header
* TEXT_OFFSET -- Virtual memory address where this program expects the program image to live
* DATA_OFFSET -- Virtual memory address where this program expects the .data/.bss section to be
* DATA_SIZE -- Size of the .data+.bss section
* ENTRYPOINT - Virtual memory address of the main() function
* STACK -- 32-bits of stack address, where the program expects `$sp` to point

Note that the .bss and .data sections are combined together.  This merely indicates how pages will get allocated for this new program.  Also note that these sections will be cleared to 0 due to how Xous processes start up.

The bootloader will copy `LOAD_SIZE` bytes of data from `LOAD_OFFSET` to a new series of pages in memory, which will be mapped to `TEXT_OFFSET`.  Additionally, `DATA_SIZE` pages will be allocated at `DATA_OFFSET`, plus a single page at `STACK`.

### XKRN

This describes the kernel.

* LOAD_OFFSET -- Position in RAM relative to the start of the arguments block
* LOAD_SIZE -- Size (in bytes) of the kernel binary, not including this header
* TEXT_OFFSET -- Virtual memory address where the kernel expects the program image to live.  This should be `0x008000000`
* DATA_OFFSET -- Virtual memory address where the kernel expects the .data/.bss section to be.  This should be `0x00c00000`
* DATA_SIZE -- Size of the .data+.bss section
* ENTRYPOINT -- Virtual address of the main() function
* STACK -- 32-bits of stack address, where the program expects `$sp` to point.  This will be used as the exception stack.
