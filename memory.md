
# Memory Layout

Xous assumes a memory-mapped IO system.  Furthermore, it assumes there
is one section of "general-purpose RAM", and zero or more additional
memory sections.

Memory is divided into "pages" of 4096 bytes, and are allocated based on
this number.  It is considered an error to request memory that isn't
aligned to this address, and it is an error to request a multiple of pages
that is different from this number.  For example, you cannot request 4097
bytes -- you must request 8192 bytes.

Memory is allocated on a first-come, first-served basis.  Physical addresses
may be specified when allocating memory, in which case they are taken from
that physical address.  Otherwise, they are pulled from the "general-purpose
RAM" section.

A process can request specific memory ranges to be allocated.  For example, a `uart_server` might request the UART memory region be allocated so that it can handle that device and provide a service.  This region cannot be re-mapped to another process until it is freed.

A process can request more memory for its heap.  This will pull memory from the global pool and add it to that process' `heap_size`.  Processes start out with a `heap_size` of 0, which does not include the contents of the `.text` or `.data` sections.

If a process intends to spawn multiple threads, then it must malloc that memory prior to creating the thread.

## Special Virtual Memory Addresses

These addresses are statically mapped in virtual memory.  They are only visible in "Supervisor" mode.  However, they are globally mapped, and are available in every process.

| Address    | Description
| ---------- | -----------
| 0x01000000 | Kernel arguments, allocation tables
| 0x02000000 | Kernel binary image
| 0x04000000 | Kernel data section

## Memory Whitelist

Memory is kept in a whitelist.  That is, when calling `sys_memory_allocate()`, the address is first validated against a list of known ranges.  This has two major benefits:

1. It prevents attacks where memory mirrors can be reused to access another process' memory.  For example, on Litex, the peripheral space is mirrored at both `0x70000000` and `0xe0000000`.  Without special handling, two different processes could map these two spaces and share memory.
2. We can limit the amount of memory that is needed to keep track of memory.  For example, if we had generic tables that expanded every time an invalid region was accessed, then a process could use up the kernel's memory by simply requesting every possible address.  In having a whitelist, we can statically allocate memory blocks to track memory usage.

## Memory Tables

Each valid memory page has an associated table entry.  This entry simply contains a `XousPid`, to indicate which process the memory block belongs to.  A `XousPid` of `0` is invalid, and indicates the region is free.  A `XousPid` of `1` indicates the page belongs to the kernel.

In a system with ample amounts of memory, all valid memory page would get its own memory table.  However, in resource-constrained systems, a simple array is not suitable, and so a programmatic lookup table is used instead.

## Kernel Arguments

There are several arguments that specify where kernel structures should go.

### Memory Blocks

The kernel needs to know the range of pages.  This is passed to the kernel
as a list in the following form:

```
MBLK,$count,
$start1,$len1,$name
$start2,$len2,$name
...
```

The first memory range that is listed is always RAM, which is the range that will be used for dealing with `sbrk` and unspecified memory allocations.  The name should be printable ASCII, and is primarily used for debugging.

### Kernel Memory

This structure specifies how kernel memory is laid out.

## Allocation Tables

Each page of memory has an entry in the allocation tables.  When allocating
a new page, Xous ensures that page is not currently allocated to another
process.  This ensures that each page of memory is only assigned to one
process at a time, unless that page is handed out as shared.

Allocation tables have the following layout:

```rust
struct AllocationEntry {
    /// PID that owns this page.  `0` if this
    /// page is unallocated.
    pid: XousPid,
}

struct PageRange {
    /// A slice of all allocations within this range.
    entries: &[AllocationEntry],
}

struct PageAllocations {
    /// Each range of memory gets its own allocation table.
    /// ranges[0] is always defined as RAM,
    /// and is where memory comes from when
    /// no physical address is specified.
    ranges: &[PageRange],
}
```

## Page Tables

Each process requires its own page table.  The kernel will be mapped to a fixed offset in each process, in order to save some RAM and make
context switches easier.
