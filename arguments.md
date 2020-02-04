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
| MBLK | Memory page blocks.  This is a series of offset/size pairs indicating memory regions in the system, as well as a code name for the memory page.  The first region is defined to be "RAM", and is where memory will be allocated from.
| XKRN | Kernel memory specification.  Includes the offset of the kernel in RAM as well as its size.  Does not need to be page-aligned.
| PID1 | Initial program specification.  This includes the offset of PID1 in RAM as well as its size.  Does not need to be page-aligned.

### XASZ

The overall size of the argument block.  This must come first.  This tag
has one word of contents, which is the number of words of data.  Therefore,
a minimum boot tag structure would have an `XASZ` size of 3: One word for the
tag, one for the crc+size, and one for the contents of the `XASZ` region.

### MBLK

Define memory block regions.  See [memory.md](memory.md) for more information.

### PID1

The `PID1` argument describes how to load PID1.  It has the following values:

* LOAD_OFFSET -- Address in RAM where PID1 is stored
* LOAD_SIZE -- Size (in bytes) of PID1 binary, not including this header
* TEXT_OFFSET -- Virtual memory address where PID1 expects the program image to live
* DATA_OFFSET -- Virtual memory address where PID1 expects the .data/.bss section to be
* DATA_SIZE -- Size of the .data+.bss section
* ENTRYPOINT - Virtual memory address of the main() function
* STACK -- 32-bits of stack address, where the program expects `$sp` to point

Note that the .bss and .data sections are combined together.  This merely indicates how pages will get allocated for this new program.  Also note that these sections will be cleared to 0 due to how Xous processes start up.

### XKRN

This describes the kernel.

* LOAD_OFFSET -- Address in RAM where the kernel is stored
* LOAD_SIZE -- Size (in bytes) of the kernel binary, not including this header
* TEXT_OFFSET -- Virtual memory address where the kernel expects the program image to live.  This should be `0x02000000`
* DATA_OFFSET -- Virtual memory address where the kernel expects the .data/.bss section to be.  This should be `0x04000000`
* DATA_SIZE -- Size of the .data+.bss section
* TRAP_HANDLER -- Virtual memory of the `stvec` trap handler
