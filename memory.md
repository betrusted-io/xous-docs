# Memory Management

In general, memory cannot be mapped to more than one process.  This includes RAM, storage, and io peripherals.

A process can request specific memory ranges to be allocated.  For example, a `uart_server` might request the UART memory region be allocated so that it can handle that device and provide a service.  This region cannot be re-mapped to another process until it is freed.

A process can request more memory for its heap.  This will pull memory from the global pool and add it to that process' `heap_size`.  Processes start out with a `heap_size` of 0, which does not include the contents of the `.text` or `.data` sections.

If a process intends to spawn multiple threads, then it must malloc that memory prior to creating the thread.

## Memory Whitelist

Memory is kept in a whitelist.  That is, when calling `sys_memory_allocate()`, the address is first validated against a list of known ranges.  This has two major benefits:

1. It prevents attacks where memory mirrors can be reused to access another process' memory.  For example, on Litex, the peripheral space is mirrored at both `0x70000000` and `0xe0000000`.  Without special handling, two different processes could map these two spaces and share memory.
2. We can limit the amount of memory that is needed to keep track of memory.  For example, if we had generic tables that expanded every time an invalid region was accessed, then a process could use up the kernel's memory by simply requesting every possible address.  In having a whitelist, we can statically allocate memory blocks to track memory usage.

## Memory Tables

Each valid memory page has an associated table entry.  This entry simply contains a `XousPid`, to indicate which process the memory block belongs to.  A `XousPid` of `0` is invalid, and indicates the region is free.  A `XousPid` of `1` indicates the page belongs to the kernel.

In a system with ample amounts of memory, all valid memory page would get its own memory table.  However, in resource-constrained systems, a simple array is not suitable, and so a programmatic lookup table is used instead.
