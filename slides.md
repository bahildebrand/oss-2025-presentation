## Efficient On-Device Core Dump Processing for IoT: A Rusty Implementation

---

Presented By: Blake Hildebrand

June 24th, 2025

---

## What do we get from a coredump?

--

<!-- .slide: data-auto-animate -->

```c
gdb memfaultd core.elf
```
<!-- .element: data-id="1" -->

--

<!-- .slide: data-auto-animate -->

```c
gdb memfaultd core.elf

(gdb) bt
#0  core::ptr::write<i32> ...
#1  core::ptr::mut_ptr::{impl#0}::write<i32> ...
#2  memfaultd::cli::memfaultctl::coredump::trigger_crash ..
```
<!-- .element: data-id="1" -->

---

## Stages of Coredump Collection Development

<ol>
<li class="fragment fade-up">Normal Core Passthrough</li>
  <ul class="fragment fade-up">
    <li>Setting the stage for future improvements</li>
    <li>Allow for added metadata</li>
  </ul>
<li class="fragment fade-up">Stack Only</li>
  <ul class="fragment fade-up">
    <li>Shrinks the size</li>
  </ul>
<li class="fragment fade-up">On-Device Unwind</li>
  <ul class="fragment fade-up">
    <li>Prevents PII from leaving the device</li>
    <li>Makes the core even smaller</li>
</ol>

---

<!-- .slide: data-auto-animate -->

## What's In a Core?

--

<!-- .slide: data-auto-animate -->
<div class="r-vstack">
<div>

```bash [1| 3-9]
readelf -l core.elf

Program Headers:
  Type  Offset             VirtAddr
        FileSiz            MemSiz
  NOTE  0x0000000000000a50 0x0000000000000000
        0x0000000000002234 0x0000000000000000
  LOAD  0x0000000000002c88 0x00006119ededc000
        0x00000000000003d4 0x00000000000003d4
                 ...
```

</div>
<div class="fragment fade-up">

```c
typedef struct
{
  Elf64_Word p_type;   /* Segment type */
  Elf64_Word p_flags;  /* Segment flags */
  Elf64_Off p_offset;  /* Segment file offset */
  Elf64_Addr p_vaddr;  /* Segment virtual address */
  Elf64_Addr p_paddr;  /* Segment physical address */
  Elf64_Xword p_filesz;  /* Segment size in file */
  Elf64_Xword p_memsz;  /* Segment size in memory */
  Elf64_Xword p_align;  /* Segment alignment */
} Elf64_Phdr;
```

</div>
</div>

--

<div class="r-hstack">
<div>

#### Note Header

```c
typedef struct {
  Elf64_Word n_namesz;
  Elf64_Word n_descsz;
  Elf64_Word n_type;
} Elf64_Nhdr;
```
<!-- .element: style="margin: 1em; padding: 1em;" -->

</div>

<div>

#### `prstatus` Note

```c
struct elf_prstatus_common {
  struct elf_siginfo pr_info;
  short pr_cursig;
  unsigned long pr_sigpend;
  unsigned long pr_sighold;
  pid_t pr_pid;
  pid_t pr_ppid;
  pid_t pr_pgrp;
  pid_t pr_sid;
  struct __kernel_old_timeval pr_utime;
  struct __kernel_old_timeval pr_stime;
  struct __kernel_old_timeval pr_cutime;
  struct __kernel_old_timeval pr_cstime;
  elf_gregset_t pr_reg;
  int pr_fpvalid;
};
```
<!-- .element: style="margin: 1em; padding: 1em;" -->

</div>
</div>

--

## Where Is It Coming From?

--

## Signal Triggers

- `SIGABRT`: Abnormal termination of the program (null)
- `SIGFPE`: Floating-point exception.
- `SIGSEGV`: Invalid memory reference.

--

```bash [2]
cat /proc/sys/kernel/core_pattern
|/usr/sbin/memfault-core-handler -c /etc/memfaultd.conf %P %I %s
```

---

## What is a Linux Coredump?

--

A Linux coredump represents a **snapshot** of the crashing process' memory

- Written as an **ELF file**
- Can be loaded into programs like **GDB**
- Inspects the state of the process at the time of crash

---

## Complete Signal List

The signals that cause a coredump:

- `SIGABRT`: Abnormal termination (abort call)
- `SIGBUS`: Bus error (bad memory access)
- `SIGFPE`: Floating-point exception
- `SIGILL`: Illegal instruction
- `SIGQUIT`: Quit from keyboard
- `SIGSEGV`: Invalid memory reference
- `SIGSYS`: Bad system call
- `SIGTRAP`: Trace/breakpoint trap

---

## Enabling Coredumps

--

### Kernel Configuration

```c
CONFIG_COREDUMP=y
CONFIG_CORE_DUMP_DEFAULT_ELF_HEADERS=y
```

Most distros enable these by default

--

### Set Resource Limits

```bash
ulimit -c unlimited
```

Sets the maximum size of a coredump

---

## `core_pattern` Configuration

--

### Write to File

```bash
/tmp/core.%e.%p
```

- `%e` = process name
- `%p` = process PID

--

### Pipe to Program

```bash
|/usr/sbin/memfault-core-handler -c /etc/memfaultd.conf %P %I %s
```

- Stream coredump via `stdin`
- Access to crashing process' `procfs`
- More flexible processing

---

## `procfs` Benefits

While the pipe program is running:

- Direct access to `/proc/<pid>/mem`
- Access to `/proc/<pid>/cmdline`
- Read-only access to kernel data structures
- Additional process metadata

---

## ELF Core File Layout

--

### High-Level View

```text
┌─────────────────┐
│   ELF Header    │
├─────────────────┤
│ Program Headers │
├─────────────────┤
│   PT_NOTE 1     │
│   PT_NOTE 2     │
│      ...        │
├─────────────────┤
│   PT_LOAD 1     │ ← Stack Memory
│   PT_LOAD 2     │ ← Heap Memory  
│      ...        │ ← Other Segments
└─────────────────┘
```

---

## ELF Header

```c
typedef struct {
  unsigned char e_ident[EI_NIDENT];
  Elf32_Half e_type;        // ET_CORE for coredumps
  Elf32_Half e_machine;     // Architecture (ARM, x86, etc.)
  Elf32_Word e_version;
  Elf32_Addr e_entry;
  Elf32_Off e_phoff;        // Offset to program headers
  // ... more fields
} Elf32_Ehdr;
```

---

## Program Headers

--

```c
typedef struct {
  Elf32_Word p_type;      // PT_NOTE or PT_LOAD
  Elf32_Off p_offset;     // File offset
  Elf32_Addr p_vaddr;     // Virtual address
  Elf32_Addr p_paddr;     // Physical address
  Elf32_Word p_filesz;    // Size in file
  Elf32_Word p_memsz;     // Size in memory
  Elf32_Word p_flags;
  Elf32_Word p_align;
} Elf32_Phdr;
```

--

### Two Main Types

- **`PT_NOTE`**: Metadata about the process
- **`PT_LOAD`**: Memory segments (stack, heap, etc.)

---

## PT_NOTE Layout

```text
┌──────────────────────────────────┐
│ namesz (4 bytes) │ descsz (4 bytes) │
├──────────────────┼─────────────────┤
│     type (4 bytes)                │
├───────────────────────────────────┤
│              name                 │
│         (null-terminated)         │
├───────────────────────────────────┤
│              desc                 │
│         (note data)               │
└───────────────────────────────────┘
```

- **namesz/descsz**: Size of name and descriptor
- **name**: String identifying note type
- **desc**: Actual note data
- **type**: Note type identifier

---

## Coredumps at Memfault: Rev. 1

--

### Goals

- Symbolicated backtrace for each thread
- Register values at crash time
- Symbolicated local variables at each frame
- **No source modifications required**

--

### Our Approach

1. Use existing Linux coredump system
2. Add metadata via custom `PT_NOTE` segment
3. Stream memory to avoid allocations
4. Maintain compatibility with standard tools

---

## The Handler Process

--

### Core Pattern Setup

```bash
|/usr/sbin/memfault-core-handler -c /path/to/config %P %e %I %s
```

- `%P`: PID of crashing process
- `%e`: Process name
- `%I`: UID of crashing process
- `%s`: Signal that caused crash

--

### Processing Steps

1. Read all program headers into memory
2. Save all `PT_NOTE` segments
3. Stream `PT_LOAD` segments from `/proc/<pid>/mem`
4. Add custom metadata note
5. Write modified ELF coredump

---

## Modified ELF Layout

### Before vs After

```text
Original ELF Core        Modified ELF Core
┌─────────────────┐     ┌─────────────────┐
│   ELF Header    │     │   ELF Header    │
├─────────────────┤     ├─────────────────┤
│ Program Headers │     │ Program Headers │
├─────────────────┤     ├─────────────────┤
│   PT_NOTE       │     │   PT_NOTE       │
│                 │ --> │ + Custom Note   │ ← Added
├─────────────────┤     ├─────────────────┤
│   PT_LOAD       │     │   PT_LOAD       │
│   (Memory)      │     │   (Memory)      │
└─────────────────┘     └─────────────────┘
```

Only difference: Added custom metadata note

---

## Why This Approach?

--

### Benefits

- **Metadata injection**: Device identification and versioning
- **Future flexibility**: Sets stage for advanced processing
- **Memory efficiency**: Streaming prevents large allocations
- **Compatibility**: Standard ELF format works with existing tools

--

### The Problem

**Coredumps can be quite large!**

- Processes with many threads
- Large memory allocations
- Problematic for embedded devices with limited storage

---

## Conclusion

- **Traditional coredumps:** Powerful but large and can leak sensitive data.
- **Shrinking the Core (Part 2):** Significantly reduced size by stripping heap and limiting stack depth.
  - ~35x size reduction.
- **On-Device Unwinding (Part 3):** The ultimate solution for privacy and size.
  - No memory sections leave the device.
  - Only program counters (PCs) and necessary metadata are collected.
  - Leverages `.eh_frame` and `addr2line` for local stack unwinding.

- **Result:** Efficient, privacy-preserving crash capture for embedded Linux IoT devices.

---

<!-- .slide: data-auto-animate -->

## Common Challenges For Coredumps On Embedded Systems

--

<ul>
<li class="fragment fade-up">Size</p>
<li class="fragment fade-up">Size</p>
<li class="fragment fade-up">Size</p>
</ul>

---

## Q&A

- **Thank you!**
- **Further Reading:**
  - [Linux Coredumps (Part 1) - Introduction](https://interrupt.memfault.com/blog/linux-coredumps-part-1)
  - [Linux Coredumps (Part 2) - Shrinking the Core](https://interrupt.memfault.com/blog/linux-coredumps-part-2)
  - [Linux Coredumps (Part 3) - On Device Unwinding](https://interrupt.memfault.com/blog/linux-coredumps-part-3)
- **Source Code:** [memfault/memfaultd](https://github.com/memfault/memfaultd/tree/main/memfaultd/src/cli/memfault_core_handler/stack_unwinder)
