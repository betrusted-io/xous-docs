# Xous Startup Process

This document describes how a running system is established.

## Pre-Boot Environment

A pre-boot environment is needed to pass kernel arguments.  It simply loads the argument structure into RAM, sets `$a0`, and jumps to the stage-1 kernel.

## Stage 1: Allocating memory pages

This stage-1 kernel allocates space for various kernel data structures according to kernel arguments.  At the end of this, the kernel will not rely on anything from the pre-boot environment.

1. Determine how many pages to allocate from the end of RAM by reading `XASZ` and adding the total of `MBLK`.
1. Allocate that number of pages from the end of RAM
1. Copy the args to the first page
1. Create PageAllocations tables from args, just following the args array
1. Allocate one more page for stack, and set $sp to point there.

At this point, we don't need to rely on external memory anymore.  There is no MMU, so we're still running in Machine Mode.  We still need to copy data off of the SPI flash to set up things like the kernel and PID1.

## Stage 2: Loading the kernel and creating the first process

The first process will eventually get turned into PID1, however at the start we're running without an associated userspace process.  In fact, we're even running without kernel loaded.

1. Allocate pages for main kernel, assigning them to PID1
1. Copy main kernel into newly-allocated pages
1. Allocate page for PID1 SATP
1. Allocate second-level pages for kernel
1. Map our current stack to PID1
1. Delegate all interrupts to supervisor mode
1. Set up a `trap_handler` to handle machine mode traps, even though they should never happen
1. Return to kernel and set the stack pointer, enabling MMU

The kernel is now running in Supervisor mode.  The MMU is enabled, but there still is no PID1.

## Stage 3: Getting the first process running

In this final stage, PID1 is transformed into a full-fledged process.

1. Allocate process ID for PID1
1. Allocate stack pages for PID1
1. Map text section for PID1
1. Enable MMU by returning to PID1 -- free existing stack beforehand.
