# Memory Management

In general, memory cannot be mapped to more than one process.  This includes RAM, storage, and io peripherals.

A process can request specific memory ranges to be allocated.  For example, a `uart_server` might request the UART memory region be allocated so that it can handle that device and provide a service.  This region cannot be re-mapped to another process until it is freed.

A process can request more memory for its heap.  This will pull memory from the global pool and add it to that process' `heap_size`.  Processes start out with a `heap_size` of 0, which does not include the contents of the `.text` or `.data` sections.

If a process intends to spawn multiple threads, then it must malloc that memory prior to creating the thread.
