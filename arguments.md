# Kernel Arguments

Kernel arguments are passed by giving a page-aligned address pointed to by
register `$a0`.  This uses IFF-style tags with the following format:

```
TagType,Size
```

The size is in native byte order, and does not include the 8 bytes describing the tag type or the size.

## Tag Types

| Tag | Description
| ---- | ------------
| XASZ | Xous Args Size.  This indicates the size of the entire block (including the XASZ tag and size), and can be used to determine how many pages to save for this block.  This should be the first argument.
| BFlg | Bootloader config flags.
| MBLK | Memory page blocks.  This is a series of offset/size pairs indicating memory regions in the system, as well as a code name for the memory page.  The first region is defined to be "RAM", and is where memory will be allocated from.
| XKRN | Kernel memory specification.  Includes the offset of the kernel in RAM as well as its size.  Does not need to be page-aligned.
| Init | Initial program specification.  This includes the offset of PID1 in RAM as well as its size.  Does not need to be page-aligned.  May appear more than once.

### Bflg

This configures various bootloader flags.

* NO_COPY -- Set this flag to skip copying data to RAM.
* ABSOLUTE -- If set, all addresses are absolute.  Otherwise, they're relative to the start of the config block.

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

### XKRN

This describes the kernel.

* LOAD_OFFSET -- Position in RAM relative to the start of the arguments block
* LOAD_SIZE -- Size (in bytes) of the kernel binary, not including this header
* TEXT_OFFSET -- Virtual memory address where the kernel expects the program image to live.  This should be `0x008000000`
* DATA_OFFSET -- Virtual memory address where the kernel expects the .data/.bss section to be.  This should be `0x00c00000`
* DATA_SIZE -- Size of the .data+.bss section
* ENTRYPOINT -- Virtual address of the main() function
* STACK -- 32-bits of stack address, where the program expects `$sp` to point.  This will be used as the exception stack.
