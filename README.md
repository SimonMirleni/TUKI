# The New Pain — Distributed Operating System Simulator

A multi-process **operating system simulator** written in **C**, built as the final integrative project for the *Sistemas Operativos* course at UTN FRBA (Universidad Tecnológica Nacional, 1C 2023).

The project models a real OS as **five independent processes** that communicate over **TCP sockets**, exercising the core concepts of process scheduling, memory management, virtual memory, file systems, IPC, concurrency, and deadlock handling.

---

## What it does

The simulator runs user programs written in a small **pseudo-assembly language** (`SET`, `MOV_IN`, `MOV_OUT`, `I/O`, `WAIT`, `SIGNAL`, `F_OPEN`, `F_READ`, `F_WRITE`, `CREATE_SEGMENT`, etc.). Each program is submitted from a **Console**, scheduled and dispatched by the **Kernel**, executed instruction-by-instruction by a **CPU** (with its own MMU and CPU registers), backed by a **segmented Memory** module and a block-based **File System** built on top of a flat disk file.

A typical execution exercises every layer of an OS at once: scheduling decisions, segment allocation/compaction, MMU address translation, page-like I/O against a virtual disk, blocking syscalls, and resource synchronization across multiple concurrent programs.

---

## Architecture

```
       ┌──────────────┐         ┌──────────────┐
       │   Consola    │ ──────▶ │    Kernel    │ ◀──────┐
       │ (pseudo-asm) │         │ schedulers   │        │
       └──────────────┘         │ resources/IO │        │
                                └──────┬───────┘        │
                                       │                │
                            PCB │ instr.fetch           │
                                       ▼                │
                                ┌──────────────┐        │
                                │     CPU      │        │
                                │   MMU + regs │        │
                                └──────┬───────┘        │
                          read/write   │                │
                                       ▼                │
                  ┌────────────┐   ┌────────────┐       │
                  │  Memoria   │◀─▶│ FileSystem │───────┘
                  │ segmented  │   │ super/bmp/ │
                  │ FIRST/BEST/│   │ blocks/FCB │
                  │  WORST +   │   └────────────┘
                  │ compaction │
                  └────────────┘
```

Every arrow is a real TCP connection with a custom serialization protocol (op-codes + binary buffers) implemented in `shared/`.

---

## Modules

| Module | Responsibility | Key files |
|---|---|---|
| **Kernel** | Long-term & short-term schedulers (FIFO / HRRN), multiprogramming control, PCB lifecycle (NEW → READY → EXEC → BLOCK → EXIT), syscall dispatching, resource semaphores (`WAIT`/`SIGNAL`), deadlock-prone resource queues, file-open coordination, I/O blocking | `kernel/src/kernel.c`, `kernel/src/planificacion.c` |
| **CPU** | Fetch–decode–execute loop, MMU performing logical → physical address translation over the segment table, full register file (AX/BX/CX/DX, EAX–EDX, RAX–RDX), segment-fault detection | `cpu/src/cpu.c` |
| **Memoria** | Segmented memory with shared segment 0, allocation algorithms **FIRST / BEST / WORST**, free-hole tracking, **compaction** triggered on `OUT_OF_MEMORY`, configurable access latency | `memoria/src/memoria.c`, `memoria/src/utils.c` |
| **FileSystem** | Block-based FS over a flat `bloques.dat`, superblock + bitmap + per-file FCB metadata, `F_OPEN/F_CLOSE/F_SEEK/F_READ/F_WRITE/F_TRUNCATE`, R/W against a real disk-image file with configurable block-access delay | `fileSystem/src/fileSystem.c` |
| **Consola** | Loads a `.pseudo` program, ships the instruction list to the Kernel, waits for end-of-process notification | `consola/src/consola.c` |
| **Shared** | Cross-module utilities: socket server/client, packet serialization, PCB (de)serialization, op-code protocol | `shared/src/shared_utils.c` |

---

## Engineering highlights

- **~4,300 lines of C** across five communicating processes plus a shared library.
- **Custom binary protocol** (op-codes + length-prefixed buffers) for sending PCBs, instruction sets, segment tables, and read/write requests between modules.
- **Concurrency**: POSIX threads, mutexes and counting semaphores guard the ready queue, multiprogramming degree, file-system serialization, and the global open-files table.
- **Schedulers**:
  - *Long-term* gates admission via the multiprogramming-degree semaphore.
  - *Short-term* implements **FIFO** and **HRRN** with α-weighted next-burst estimation and dynamic re-ordering of the ready queue.
- **Memory management**: dynamic segment table per process, three allocation policies, hole coalescing, and full **memory compaction** that rewrites live segment tables and propagates the new base addresses back to every PCB in the kernel.
- **Virtual memory / MMU** in the CPU: logical addresses split into `(segment, offset)`, translated through the per-PCB segment table, with explicit **SEG_FAULT** signaling back to the Kernel when an access overflows a segment.
- **File system on a flat disk image**: superblock + bitmap + numbered blocks (`bloques.dat`), per-file FCB, and pointer-driven `F_SEEK`/`F_READ`/`F_WRITE` whose payloads are copied directly to/from the Memory module's address space.
- **Deadlock-prone resource model**: configurable `RECURSOS=[…]` with `INSTANCIAS_RECURSOS=[…]`, per-resource blocked queues, and pseudo-programs designed to deliberately induce circular waits (`DEADLOCK_*.pseudo`).
- **Reproducible test matrix**: scripted scenarios for the base case, deadlock, memory pressure / compaction, file-system load, and error paths (`prueba_*.sh`, `setear_prueba.sh`, paired with versioned configs in `configs/`).

---

## Repository layout

```
.
├── kernel/        # scheduler, PCB lifecycle, syscall handlers
├── cpu/           # fetch-decode-execute, MMU, registers
├── memoria/       # segmented memory, allocation, compaction
├── fileSystem/    # superblock, bitmap, blocks, FCB
├── consola/       # pseudo-code loader / client
├── shared/        # sockets + serialization + protocol
├── configs/       # versioned configs per test scenario (v1..v5)
├── prueba_*.sh    # scripted test scenarios
├── makeall.sh     # builds every module
└── instalar_librerias.sh  # installs the so-commons-library dependency
```

---

## Build & run (Linux)

The project targets a Linux toolchain (`gcc`, `make`, POSIX threads, `readline`) and the [so-commons-library](https://github.com/sisoputnfrba/so-commons-library) used by the course.

```bash
# 1. install the shared commons lib used by all modules
./instalar_librerias.sh

# 2. build every module
./makeall.sh

# 3. pick a test scenario (base / deadlock / memory / fs / errors)
./setear_prueba.sh

# 4. start the modules in separate terminals
./memoria/memoria
./fileSystem/fileSystem
./cpu/cpu
./kernel/kernel

# 5. run a scenario
./prueba_base.sh        # or prueba_deadlock.sh / prueba_memoria.sh / ...
```

Scheduling algorithm, multiprogramming degree, memory size, segment count, allocation policy, latencies, and resource pool are all driven by the `*.config` files under `configs/` and selected automatically by `setear_prueba.sh`.

---

## Concepts exercised

Process scheduling · multiprogramming · PCB & context switching · IPC over TCP · custom binary serialization · POSIX threads, mutexes & semaphores · segmented virtual memory · MMU / address translation · memory compaction · file systems on a block device · syscalls and blocking I/O · deadlock conditions and recovery via timed scenarios.

---

## Course context

> **Universidad Tecnológica Nacional — Facultad Regional Buenos Aires**
> Sistemas Operativos — *Trabajo Práctico Integrador, 1er cuatrimestre 2023*
> Project codename: **The New Pain** (`tp-2023-1c-The-New-Pain`)
