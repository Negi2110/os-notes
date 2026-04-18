# Read/Write from Memory — Master Notes

---

## 0. The Core idea

```
CPU cannot operate on RAM directly.
It must explicitly request reads and writes through the memory bus.
Every access goes: CPU → memory controller → DRAM → back to CPU.
Access time: 50–100 ns (one of the biggest bottlenecks in computing).
```

---

## 1. Read from Memory — Step by Step

```
Setup (example stack frame at address 1024):
  bp  = 0      (4 bytes) at address 2044
  lr  = 0      (4 bytes) at address 2040
  ptr = 1024   (4 bytes) at address 2036   ← pointer to address 1024
  *ptr = 11                                ← value 11 lives at address 1024

Goal: CPU wants to execute instruction at address 640 (the PC)
```

**Step 1 — Kernel requests instruction fetch:**
```
Program Counter (PC) = 640
Kernel tells CPU: load instruction at address 640
CPU sends READ REQUEST → address 640 → to RAM
```

**Step 2 — RAM responds with a burst:**
```
RAM does NOT return just 1 byte or 1 instruction.
RAM returns 64 bytes (one full cache line) starting at address 640.
This takes 50–100 ns.

Why 64 bytes?
  → Burst length = 64 bytes (prefetch buffer design, matches cache line)
  → Fetching neighbours is nearly free once the row is open
  → Spatial locality means you'll probably need them anyway
```

**Step 3 — Cache stores the burst:**
```
The 64 bytes are stored in the CPU's L1/L2/L3 cache.
Subsequent reads to nearby addresses (641, 642, ...) hit the cache.
Cache hit = ~1–4 cycles.  Cache miss = back to RAM = 50–100 ns.
```

```
Timeline:

  PC = 640
  CPU ──[read addr 640]──────────────────→ RAM
  RAM ──[64 bytes burst, 50–100ns]───────→ CPU L cache
                                            ↓
                                   instruction at 640 executes
                                   641, 642... already in cache
```

---

## 2. Read from Memory — Reading a Value (*ptr)

```
Goal: read the value at address 1024 (*ptr = 11)

CPU ──[read addr 1024]──→ RAM
RAM ──[64 bytes burst]──→ CPU cache
                           value 11 at offset within the 64-byte block
                           cached alongside its neighbours
```

**Key insight:**
```
You never fetch 4 bytes.
You always fetch 64 bytes.
The 4 bytes you wanted are somewhere inside that 64-byte block.
The rest go into cache — useful if you access nearby addresses next.
```

---

## 3. Write to Memory — Step by Step

```
Goal: write value 22 to address 1024 (*ptr = 22), 4 bytes

Step 1 — CPU sends write request:
  CPU ──[write addr 1024, value 22, 4 bytes]──→ RAM

Step 2 — RAM performs the write:
  Memory controller opens the bank → opens the row → writes to column 1024
  Write goes to sense amplifiers first
  Committed to cells on precharge

Step 3 — Check for errors:
  Write is verified (ECC on DDR5, optional on DDR4)
  Error returned to CPU if write failed
```

```
Timeline:

  CPU ──[write to 1024, value=22, size=4]──→ RAM
  RAM ──[ack / error]──────────────────────→ CPU
```

---

## 4. Write vs Read — Key Differences

```
READ:
  → Returns 64 bytes (burst), not just what you asked for
  → Data cached in CPU L caches
  → Latency: 50–100 ns first access, ~1 ns if cached

WRITE:
  → Writes exact size (e.g. 4 bytes) to exact address
  → Also updates the cache line if that line is already cached
  → Must be verified (error checking)
  → No burst on write — only the specified bytes change
```

---

## 5. Memory Alignment

```
Rule: data types must be placed at addresses divisible by their size.

Type      Size    Must be at address divisible by
────────────────────────────────────────────────
char      1 byte  any address (1)
short     2 bytes address divisible by 2
int       4 bytes address divisible by 4
double    8 bytes address divisible by 8

Why?
  CPU fetches data in aligned chunks (1, 4, or 8 byte boundaries)
  Misaligned access may straddle two cache lines → two reads needed
  Some architectures fault on misaligned access entirely
  Aligned access = always 1 read, always fast
```

**Struct padding example:**
```c
// Wasteful layout (compiler adds padding):
struct Foo {
  char c;       // 1 byte at 0x00
  // 7 bytes padding ← double needs 8-byte alignment
  double d;     // 8 
  bytes at 0x08
  short s;      // 2 bytes at 0x10
  // 2 bytes padding ← int needs 4-byte alignment
  int i;        // 4 bytes at 0x14
};
// Total: 24 bytes (not 15)

// Efficient layout (reorder fields, largest first):
struct Foo {
  double d;     // 8 bytes at 0x00
  int i;        // 4 bytes at 0x08
  short s;      // 2 bytes at 0x0C
  char c;       // 1 byte at 0x0E
  // 1 byte padding to round to 8
};
// Total: 16 bytes
```

---

## 6. Full Read/Write Flow Diagram

```
CPU wants to READ address X:
  ┌─────────────────────────────────────────────────┐
  │ 1. Check L1 cache → HIT? return in ~1 ns        │
  │ 2. Check L2 cache → HIT? return in ~5 ns        │
  │ 3. Check L3 cache → HIT? return in ~20 ns       │
  │ 4. MISS → send read request to memory controller│
  │ 5. Controller → opens bank → row → column       │
  │ 6. DRAM returns 64-byte burst → 50–100 ns       │
  │ 7. Burst stored in L cache                      │
  │ 8. Requested bytes returned to CPU              │
  └─────────────────────────────────────────────────┘

CPU wants to WRITE value V to address X:
  ┌─────────────────────────────────────────────────┐
  │ 1. Send write request: address X, value V, size │
  │ 2. If X is in cache → update cache line too     │
  │ 3. Memory controller opens bank → row → column  │
  │ 4. Write V to DRAM cells (via sense amplifiers) │
  │ 5. Verify write (ECC) → return ack or error     │
  └─────────────────────────────────────────────────┘
```

---

## 7. Summary Table

```
Operation  │ Request            │ Response         │ Latency
───────────┼────────────────────┼──────────────────┼──────────────
Read       │ address            │ 64-byte burst    │ 50–100 ns
           │                    │ cached in L$     │ ~1 ns if hit
Write      │ address+value+size │ ack or error     │ 50–100 ns
Alignment  │ must be at addr    │ 1 memory access  │ fast
           │ divisible by size  │                  │
Misaligned │ straddles 2 lines  │ 2 memory accesses│ 2× slow
```

---

## 8. One-Line Interview Answers

**"What happens when the CPU reads from memory?"**
The CPU sends a read request with the address; DRAM returns a 64-byte burst (one cache line) which is stored in the L cache; subsequent nearby accesses hit the cache at ~1 ns instead of going back to RAM.

**"Why does a read return 64 bytes when you asked for 4?"**
DRAM burst length is fixed at 64 bytes (one cache line). Once a row is open, fetching neighbours costs almost nothing, and spatial locality means you'll likely need them — so the hardware always fetches the full line.

**"What is memory alignment?"**
Placing data at addresses divisible by the data type's size so the CPU can fetch it in one operation. Misaligned data may straddle two cache lines, requiring two reads instead of one.

**"What happens during a write?"**
The CPU sends the address, value, and byte count to the memory controller. The controller writes to DRAM (via sense amplifiers, committed on precharge), updates any cached copy of that line, and returns an acknowledgement or error.

**"Why is memory access 50–100 ns?"**
Opening a DRAM row (tRCD) + CAS latency (CL) + burst transfer time across the memory bus. This is the fundamental cost of a cache miss and the main reason CPU caches exist.