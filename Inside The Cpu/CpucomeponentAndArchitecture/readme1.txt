# Inside the CPU — Master Notes

---

## 0. The Big Picture

```
CPU is not one thing. It is several specialized units
working together at billions of operations per second.

┌─────────────────────────────────────────┐
│                  CPU CORE               │
│  ┌─────┐  ┌─────┐  ┌───────────────┐   │
│  │ ALU │  │ CU  │  │   Registers   │   │
│  └─────┘  └─────┘  └───────────────┘   │
│              ↕              ↕           │
│         ┌────────────────────┐          │
│         │   MMU  │   TLB    │          │
│         └────────────────────┘          │
│              ↕                          │
│         ┌────────┐                      │
│         │   L1   │  (per core, ~128KB)  │
│         └────────┘                      │
│         ┌────────┐                      │
│         │   L2   │  (per core, ~256MB)  │
│         └────────┘                      │
└─────────────────────────────────────────┘
          ┌────────┐
          │   L3   │  (shared all cores, ~64MB)
          └────────┘
               ↕
          ┌──────────────┐
          │  Main Memory │  (DRAM, 50–100ns)
          └──────────────┘
```

---

## 1. ALU — Arithmetic Logic Unit

```
The actual compute engine. Does the real work.

Operations:
  Arithmetic:  +  -  *  /
  Logic:       AND  OR  XOR  NOT
  Comparison:  ==  >  

That's it. ALU only does math and logic.
Everything else (fetch, decode, memory) is handled by other units.

Key insight:
  ALU operates on REGISTERS only.
  It cannot directly touch RAM.
  Data must be loaded into registers first,
  then ALU operates,
  then result written back if needed.
```

---

## 2. CU — Control Unit

```
The conductor of the orchestra.
Tells every other unit what to do and when.

Three jobs:

  FETCH    → read next instruction from memory (at address in PC)
  DECODE   → figure out what the instruction means
             (is it ADD? LOAD? JUMP? STORE?)
  EXECUTE  → tell the right unit to do it
             (tell ALU to add, tell MMU to load, etc.)

This is the FETCH → DECODE → EXECUTE cycle.
Runs billions of times per second.
Every instruction your program runs goes through this cycle.
```

---

## 3. Registers

```
Fastest storage in the entire computer.
Live inside the CPU core itself.
~0.3ns access. Zero latency from ALU's perspective.

Size: 32-bit or 64-bit per register.
Count: typically 16–32 general purpose registers on x86-64.

Types:
  PC  (Program Counter)    → address of NEXT instruction to fetch
  IR  (Instruction Register) → current instruction being executed
  SP  (Stack Pointer)      → top of the current stack
  BP  (Base Pointer)       → base of current stack frame
  General purpose          → RAX, RBX, RCX, RDX etc. (x86-64)
                             used for arithmetic, passing arguments,
                             storing intermediate results

Key insight:
  ALU can ONLY operate on registers.
  Your C++ variable lives in RAM.
  To add two variables, CPU must:
    LOAD var1 from RAM → register
    LOAD var2 from RAM → register
    ADD register + register → register
    STORE register → RAM (if needed)
```

---

## 4. MMU — Memory Management Unit

```
Responsible for ALL memory access.
Sits between the CPU and RAM.

One job: translate virtual address → physical address.

Every single memory access goes through MMU:
  CPU says "I want address 0x1000"
  MMU translates → "that's physical address 0x45000"
  MMU fetches from physical RAM

Contains the TLB (Translation Lookaside Buffer):
  Caches recent virtual → physical translations.
  TLB hit  → translation in 1 cycle
  TLB miss → walk page table in RAM → slow

TLB must be flushed on context switch:
  Process A's translations are wrong for Process B.
  Flush = TLB goes cold = next N accesses are all misses.
  (ASID/PCID tagging avoids this on modern CPUs)
```

---

## 5. L Caches — L1, L2, L3

```
Problem: RAM is 50–100ns. CPU runs at ~0.3ns per cycle.
         CPU is 200× faster than RAM.
         Without cache, CPU spends 99% of time waiting.

Solution: SRAM caches between CPU and RAM.

          Location        Size        Speed     Shared?
L1        Inside core     ~128KB      ~1ns      Per core (private)
L2        Inside core     ~256KB–2MB  ~5ns      Per core (private)
L3        Outside core    ~64MB       ~15ns     ALL cores share this
Main RAM  DIMM stick      GBs         50–100ns  All cores

L1 is TWO caches:
  L1D  (Data cache)        → stores data your program reads/writes
  L1I  (Instruction cache) → stores instructions being executed
  Split so CU and ALU can both work simultaneously without conflict.

L2, L3:
  Unified (data + instructions together).
  L2 is private to each core — fast but not shared.
  L3 is shared — slower but allows cores to share data without RAM.

Cache invalidation challenge:
  Core 1 writes to address X → L1 cache updated.
  Core 2 reads address X     → gets stale value from its own L1.
  Hardware cache coherency protocol (MESI) keeps caches in sync.
  This is one of the hardest problems in CPU design.
```

---

## 6. Multi-Core Architecture

```
Modern CPUs have multiple cores.
Each core = independent ALU + CU + Registers + MMU + L1 + L2.
All cores share L3 and main memory.

Single CPU (2 cores):
  ┌──────────────────────────────────┐
  │  Core 1              Core 2      │
  │  ALU CU Reg MMU L1   ALU CU...  │
  │  L2                  L2         │
  │  ──────────L3──────────          │
  └──────────────────────────────────┘
                  ↕ Bus
            Main Memory (DRAM)

Multi-CPU (NUMA):
  CPU 1 (Core 1, Core 2) → DIMM1 (local, fast)
  CPU 2 (Core 3, Core 4) → DIMM2 (local, fast)
  CPU 1 accessing DIMM2  → goes through bus → slow
  CPU 2 accessing DIMM1  → goes through bus → slow

  DSM = Distributed Shared Memory
  Coordinates access across multiple CPUs and their local RAM.
  NUMA-aware programs always try to use local memory.
```

---

## 7. The Fetch-Decode-Execute Cycle in Detail

```
PC = 0x640 (next instruction address)

FETCH:
  CU reads instruction at address PC (0x640) from memory/cache.
  Loads it into IR (Instruction Register).
  PC = PC + instruction_size (advance to next instruction).

DECODE:
  CU looks at IR and figures out:
    What operation? (ADD, LOAD, STORE, JUMP...)
    What operands?  (which registers? which memory address?)
    What mode?      (immediate value? register? memory?)

EXECUTE:
  CU signals the right unit:
    Arithmetic?    → tell ALU
    Memory access? → tell MMU → goes to cache/RAM
    Jump?          → update PC directly

Repeat. Forever. At ~3 billion times per second per core.
```

---

## 8. Full Speed Comparison

```
Unit              Speed           Why
──────────────────────────────────────────────────
Registers         ~0.3 ns         SRAM inside core
L1 cache          ~1 ns           SRAM, on-die, tiny
L2 cache          ~5 ns           SRAM, on-die, larger
L3 cache          ~15 ns          SRAM, shared, larger
Main RAM (DRAM)   50–100 ns       Off-chip, capacitors
SSD               50–100 µs       Flash, controller overhead
HDD               5–10 ms         Mechanical seek time
```

---

## 9. One-Line Interview Answers

**"What does the ALU do?"**
Performs all arithmetic and logic operations — add, subtract, multiply, divide, AND, OR, XOR — exclusively on register values.

**"What does the Control Unit do?"**
Fetches the next instruction from memory, decodes what it means, and signals the appropriate unit (ALU, MMU, etc.) to execute it — the fetch-decode-execute cycle.

**"What are registers?"**
Tiny, ultrafast storage inside the CPU core (~0.3ns) that hold the values the ALU operates on — variables in RAM must be loaded into registers before the CPU can compute with them.

**"What is the MMU?"**
The hardware unit that translates every virtual memory address to a physical address before RAM access, containing the TLB to cache recent translations for near-zero overhead.

**"What is L1D vs L1I?"**
L1 is split into a data cache (L1D) for data reads/writes and an instruction cache (L1I) for fetching instructions — the split lets the CU fetch instructions and the ALU access data simultaneously.

**"What is NUMA?"**
Non-Uniform Memory Access — a multi-CPU architecture where each CPU has local RAM it can access fast, but accessing another CPU's RAM requires going through a bus, making it significantly slower.