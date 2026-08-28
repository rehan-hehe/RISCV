# xv6 Operating Systems Labs

A collection of Operating Systems laboratory implementations built by
modifying and extending the **xv6 RISC-V kernel**.

These labs explore operating system internals by directly implementing
mechanisms across process management, CPU scheduling, inter-process
communication, virtual memory, Copy-on-Write, demand paging, page
replacement, file systems, and symbolic links.

Each lab is documented through its corresponding report, covering the
problem statement, design decisions, kernel modifications,
implementation details, testing, and results.

------------------------------------------------------------------------

## 📚 Labs at a Glance

| Lab | Topic | Key Concepts | Report |
|:---:|---|---|---|
| **Lab 1** | Prime PID & Process Monitoring | Process management, system calls, CPU accounting | [View Report](./Prime-PID-and-Process-Monitoring.pdf) |
| **Lab 2** | Weighted Round Robin Scheduling | CPU scheduling, priorities, Modified SRTF | [View Report](./Weighted-Round-Robin-Scheduling.pdf) |
| **Lab 3** | Inter-Process Communication | Shared memory, mailboxes, synchronization | [View Report](./Inter-Process-Communication.pdf) |
| **Lab 4** | Copy-on-Write & Demand Paging | Page faults, COW, MRU replacement, swapping | [View Report](./Copy-on-Write-and-Demand-Paging.pdf) |
| **Lab 5** | File System Extensions | Doubly-indirect blocks, symbolic links | [View Report](./File-System-Extensions.pdf) |

------------------------------------------------------------------------

# Lab 1 --- Prime PID & Process Monitoring

📄 [Detailed Report](./Prime-PID-and-Process-Monitoring.pdf)

This lab focused on how xv6 manages processes internally and how
information maintained inside the kernel can be exposed safely to
user-space programs.

## Objectives

### Prime PID Allocation

The default PID allocation mechanism was modified so that every newly
created process receives a **prime number as its Process ID**.

Instead of sequential allocation:

``` text
1, 2, 3, 4, 5, 6, ...
```

processes receive:

``` text
2, 3, 5, 7, 11, 13, 17, ...
```

This required modifying the kernel's PID allocation logic and
introducing a mechanism to determine the next valid prime number while
preserving correct synchronization.

### Custom `top` Command

A simplified version of the Linux `top` command was implemented for xv6.
The command displays:

``` text
PID
Process State
CPU Ticks
Process Name
```

The implementation required collecting process information from the
kernel and exposing it to a user-space program through the system call
interface.

## Main Components

-   Modified PID allocation logic
-   Prime number checking
-   Process information collection
-   CPU tick tracking
-   New kernel interfaces
-   System call integration
-   User-space `top` command

## What I Learned

``` text
User Program
     ↓
System Call Interface
     ↓
Kernel Data Structures
     ↓
Process Management
```

This lab provided practical experience with process structures, PID
management, kernel synchronization, system calls, and safe communication
between kernel and user space.

------------------------------------------------------------------------

# Lab 2 --- Weighted Round Robin Scheduling

📄 [Detailed Report](./Weighted-Round-Robin-Scheduling.pdf)

This lab focused on modifying xv6's scheduler and implementing two
custom CPU scheduling policies:

1.  **Weighted Round Robin**
2.  **Modified Shortest Remaining Time First**

## Weighted Round Robin

Traditional Round Robin scheduling provides equal scheduling
opportunities to runnable processes. This implementation introduced
**process priorities as weights**.

``` text
Process A → Priority 5
Process B → Priority 10
Process C → Priority 20
```

Processes with larger weights receive proportionally more CPU scheduling
opportunities. The implementation involved extending the process
structure with priority information and introducing interfaces for
processes to modify and inspect their priority.

## Modified SRTF Scheduling

A modified version of **Shortest Remaining Time First** scheduling was
also implemented. Each process maintains a CPU burst estimate, and the
scheduler selects the runnable process with the smallest remaining
burst.

The implementation uses a lazy scheduling approach, avoiding
unnecessarily aggressive preemption while still prioritizing shorter
jobs at scheduling decision points.

## Main Components

-   Process priority management
-   Weighted CPU allocation
-   Priority-related system calls
-   CPU burst estimates
-   Modified SRTF scheduling
-   Scheduler selection logic
-   Process inheritance testing

## What I Learned

The lab demonstrated trade-offs between:

``` text
Fairness
Latency
Throughput
Responsiveness
Context Switching
Starvation
```

Different scheduling strategies optimize for different goals while
operating on the same underlying process-management infrastructure.

------------------------------------------------------------------------

# Lab 3 --- Inter-Process Communication

📄 [Detailed Report](./Inter-Process-Communication.pdf)

This lab extended xv6 with two fundamental Inter-Process Communication
mechanisms:

1.  **Shared Memory**
2.  **Mailboxes**

## Shared Memory

A shared memory mechanism was implemented that allows multiple processes
to access the same physical memory region. The interface supports
creating a region, attaching to an existing region, and detaching from a
region.

A global shared-memory structure tracks active regions and their
associated physical memory. Reference counting ensures that a shared
page is only released once the final process detaches.

``` text
                Physical Page
                      │
             ┌────────┼────────┐
             │        │        │
         Process A Process B Process C
```

## Mailboxes

A mailbox-based messaging system was implemented to allow processes to
exchange messages through bounded message queues.

``` text
Empty Mailbox: Receiver blocks until a message arrives
Full Mailbox : Sender blocks until space becomes available
```

The implementation uses xv6's synchronization mechanisms to coordinate
blocked and awakened processes.

## Coordinated Traversal

The shared memory and mailbox mechanisms were then combined to solve a
coordinated traversal problem. Two processes move through an intertwined
structure stored in shared memory. Each process communicates its current
position to the other, which then uses the shared structure to determine
its own next position.

``` text
Process 1: Send → Receive
Process 2: Receive → Send
```

This asymmetric ordering avoids a situation where both processes wait
indefinitely for the other to act first.

## Main Components

-   Shared memory regions
-   Physical memory mapping
-   Reference counting
-   Mailbox creation
-   Message queues
-   Blocking send and receive
-   `sleep()` and `wakeup()`
-   Process synchronization
-   Deadlock avoidance

## What I Learned

``` text
Shared Memory
→ Processes communicate through shared data

Message Passing
→ Processes communicate through explicit messages
```

Combining both mechanisms highlighted the importance of synchronization
and communication ordering in concurrent systems.

------------------------------------------------------------------------

# Lab 4 --- Copy-on-Write & Demand Paging

📄 [Detailed Report](./Copy-on-Write-and-Demand-Paging.pdf)

This lab focused on xv6's virtual memory subsystem through:

1.  **Copy-on-Write Fork**
2.  **Demand Paging with MRU Page Replacement**

The corresponding implementation snapshots are included in:

-   `xv6-riscv-cow`
-   `xv6-riscv-dempg`

## Copy-on-Write Fork

A traditional `fork()` duplicates a process's memory. Copy-on-Write
avoids immediately copying physical pages.

``` text
Parent ─────┐
            ├── Shared Physical Page
Child ──────┘
```

The shared page is protected from direct modification. When a process
attempts to write:

``` text
Write Attempt
      ↓
Page Fault
      ↓
Allocate New Page
      ↓
Copy Existing Data
      ↓
Update Page Table
      ↓
Resume Execution
```

This delays memory copying until it is actually required.

## Reference Counting

Since multiple processes may reference the same physical page, the
implementation tracks the number of active references. A physical page
is released only when its reference count reaches zero.

This required modifications to physical memory management and allocation
logic.

## Demand Paging

Demand paging delays physical memory allocation until a page is actually
accessed.

``` text
Process Requests Memory
          ↓
Virtual Address Space Grows
          ↓
No Physical Page Yet
          ↓
First Access
          ↓
Page Fault
          ↓
Physical Page Allocation
```

The page fault handler therefore becomes an important part of normal
memory allocation.

## MRU Page Replacement

When physical memory is constrained, the implementation uses a **Most
Recently Used (MRU)** policy.

``` text
Memory Pressure
      ↓
Select MRU Page
      ↓
Swap / Store Page
      ↓
Reuse Physical Memory
```

The implementation tracks recently accessed pages and memory-related
events such as page faults and swap activity.

## Main Components

-   Copy-on-Write page sharing
-   Reference counting
-   Page table modifications
-   Write fault handling
-   Demand allocation
-   Page fault handling
-   MRU page tracking
-   Page eviction
-   Swap-like storage management
-   Memory statistics

## What I Learned

``` text
fork()
  │
  ├── Page Tables
  ├── Physical Memory
  ├── Reference Counting
  ├── Page Faults
  └── Virtual Memory
```

This lab demonstrated that page faults are not only error
conditions---they can also be deliberately used to implement advanced
memory-management strategies.

------------------------------------------------------------------------

# Lab 5 --- File System Extensions

📄 [Detailed Report](./File-System-Extensions.pdf)

This lab extended xv6's file system in two major directions:

1.  **Doubly-Indirect Block Mapping**
2.  **Symbolic Links**

Relevant implementation snapshots are included in:

-   `xv6-riscv_bmap`
-   `xv6-riscv-slink`
-   `xv6-riscv-5-2`

## Doubly-Indirect Block Mapping

The original xv6 inode structure supports:

``` text
Direct Blocks
      +
Single-Indirect Block
```

To support significantly larger files, the file system was extended
with:

``` text
Direct Blocks
      +
Single-Indirect Block
      +
Doubly-Indirect Block
```

The hierarchy becomes:

``` text
Inode
  │
  └── Doubly-Indirect Block
           │
           ├── Indirect Block
           │      ├── Data Block
           │      ├── Data Block
           │      └── ...
           │
           ├── Indirect Block
           │      └── Data Blocks
           │
           └── ...
```

This substantially increases the number of data blocks accessible
through a single inode and required changes to block mapping and block
deallocation logic.

## Symbolic Links

The file system was also extended with symbolic link support.

``` text
mylink ─────► target.txt
```

A new symbolic-link inode type and system call were introduced. When a
symbolic link is opened, the kernel resolves the stored target path.

``` text
open("mylink")
       ↓
Resolve Link
       ↓
Find Target
       ↓
Open Target
```

The implementation also handles `O_NOFOLLOW`, allowing the symbolic link
itself to be opened rather than automatically following its target. A
maximum resolution depth prevents infinite recursion caused by circular
or excessively deep links.

## Main Components

-   Extended inode block addressing
-   Doubly-indirect blocks
-   Block allocation
-   Block deallocation
-   Large-file support
-   New symbolic-link inode type
-   `symlink()` system call
-   Link resolution in `open()`
-   `O_NOFOLLOW`
-   Recursive link protection

## What I Learned

``` text
User File Operation
        ↓
System Call
        ↓
Path Resolution
        ↓
Inode Lookup
        ↓
Block Mapping
        ↓
Storage Blocks
```

This lab showed that extending a file system requires careful changes
across multiple interacting components rather than modifying a single
isolated function.

------------------------------------------------------------------------

# Repository Contents

The repository contains five detailed lab reports:

``` text
Copy-on-Write-and-Demand-Paging.pdf
File-System-Extensions.pdf
Inter-Process-Communication.pdf
Prime-PID-and-Process-Monitoring.pdf
Weighted-Round-Robin-Scheduling.pdf
```

It also contains xv6 implementation snapshots corresponding to the
implemented kernel modifications:

``` text
xv6-riscv-cow/
xv6-riscv-dempg/
xv6-riscv_bmap/
xv6-riscv-slink/
xv6-riscv-5-2/
```

The PDF reports are the primary documentation for understanding the
objectives, implementation approach, code changes, testing methodology,
and results of each lab.

------------------------------------------------------------------------

# Overall Learning Progression

``` text
Lab 1
│
├── Process Management
│   ├── PID Allocation
│   └── Process Monitoring
│
Lab 2
│
├── CPU Scheduling
│   ├── Weighted Round Robin
│   └── Modified SRTF
│
Lab 3
│
├── Inter-Process Communication
│   ├── Shared Memory
│   └── Mailboxes
│
Lab 4
│
├── Virtual Memory
│   ├── Copy-on-Write
│   ├── Page Fault Handling
│   ├── Demand Paging
│   └── MRU Replacement
│
Lab 5
│
└── File Systems
    ├── Doubly-Indirect Blocks
    └── Symbolic Links
```

Together, these labs provide hands-on experience with the internal
mechanisms behind a modern operating system.

------------------------------------------------------------------------

# Technologies

-   **Operating System:** xv6
-   **Architecture:** RISC-V
-   **Primary Language:** C
-   **Low-Level Development:** Kernel programming and system calls
-   **Core Areas:** Process management, scheduling, IPC, virtual memory,
    page faults, synchronization, and file systems

------------------------------------------------------------------------

# Key Takeaways

Through these implementations, I gained practical experience with:

-   Navigating and modifying an operating system kernel
-   Understanding xv6 process management
-   Designing and implementing system calls
-   Transferring information between kernel and user space
-   Building custom CPU schedulers
-   Managing process priorities and CPU burst estimates
-   Implementing shared memory
-   Building message-passing primitives
-   Synchronizing concurrent processes
-   Avoiding deadlock through communication design
-   Implementing Copy-on-Write memory management
-   Handling page faults
-   Designing demand-paging mechanisms
-   Implementing page replacement strategies
-   Extending inode block-addressing mechanisms
-   Supporting larger files
-   Implementing symbolic links and recursive path resolution

------------------------------------------------------------------------

> These implementations were developed as part of the **CS 344 Operating
> Systems Laboratory**, using the xv6 teaching operating system to
> explore the internal design and implementation of core operating
> system mechanisms.
