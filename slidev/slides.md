---
theme: neversink
layout: cover
author: Sven Tennie
---

# GHC's RISC-V Native Code Generation Backend

- Haskell Implementors' Workshop 2025
- Sven Tennie

---
layout: section
color: sky
align: c
---

# RISC-V Overview

---
layout: top-title
align: c
color: light
---

:: title ::

# RISC-V

:: content ::

- 32bit **R**educed **I**nstruction **S**et as base
  - RV32I *Base Integer Instruction Set* -> ~40 instructions, ~6 formats
- Augmented by many extensions (sub-standards)
  - ISA like playing with Lego bricks
- Custom extensions are anticipated by the ISA
  - Ideal research vehicle for computer architectures

---
layout: top-title
align: c
color: light
---

:: title ::

# RISC-V

:: content ::

- Basic interpreter can be built in an afternoon
- ISA is open source, implementations (SOCs) not necessarily
  - License: _Creative Commons Attribution 4.0 International_
  - Development on GitHub
  - Conceptualization in working groups at _RISC-V International_ foundation
    - Free membership for individuals
- Everyone is free to build a RISC-V processor:
  - Several vendors
  - Hobbiests
    - Fun fact: Some hobbiests even tape out their designs via e.g. tinytapeout.com
---
layout: top-title
align: c
color: light
---

:: title ::

# RISC-V Status

:: content ::

- Standard (ISA, Calling Convention, ...) pretty complete
- Vibrant community
- Lack of powerful hardware
  - No good cloud options -> No native cloud CI
  - Cores comparable to ARM A55 (2017)
    - Your smartphone might be more powerful than RISC-V SBCs

---
layout: top-title
align: c
color: light
---

:: title ::

# RISC-V Status

:: content ::

- Lot's of movement though
  - New boards and chips appear frequently
  - Many manufacturers
  - Research all over the world
    - E.g. EU grant for RISC-V HPC research
      - DARE (Digital Autonomy with RISC-V in Europe)
      - Funding: 240 Million Euros

---
layout: top-title
align: c
color: light
---

:: title ::

# RISC-V Status

:: content ::

- There are still some dragons ...
  - Tools don't support the full instruction set
  - Tools sometimes still have bugs ...
  - Cores may have bugs
  - Core may not adhere to the ratified standards because it pre-dates it

---
layout: top-title
align: c
color: light
---

:: title ::

# ISA naming scheme

:: content ::

- Start with a base ISA: RV32I, RV64I or RV64E
- Add the extensions in canonical order
- The ISA is pretty new, so extensions' versions can usually be ignored
  - If not, the format is `<extension><major>p<minor>`
- Reduce common extensions to sets (e.g. *G*eneral for _IMAFDZicsr_Zifencei_)
- E.g. `RV64IMAFDZicsr_ZifenceiV1p0` -> `RV64GV1p0` -> `RV64GV1` -> `RV64GV`

---
layout: top-title
align: c
color: light
---

:: title ::
# Profiles

:: content ::

- _Profiles_ (e.g. RVA23) define minimum requirements to simplify this
  - Otherwise, buying and building for a consumer computer could be a nightmare
  - (It still is, because many vendors don't mention profiles yet on their marketing pages)
  - Linux distributions handle this by relying on a small extension set (usually _RV64GC_)
    - *G*: General
    - *C*: Compressed instructions

---
layout: section
color: sky
align: c
---

# GHC Implementation Status


---
layout: top-title
align: c
color: light
---

:: title ::


# GHC RISC-V History

:: content ::

- LLVM backend by Andreas Schwab (October 2020; GHC 9.2)
- Moritz Angerman and Sven Tennie accidentally started NCG at the same time

  - Moritz switched to mentor role
  - Sven continued to hack
  - Andreas built CI support at SuSE with patch files
  - Available from GHC 9.12

- Strong advice: Reach out and team up
  - I wouldn't have imagined that such great collaboration between former strangers would be possible!
  - It is!

---
layout: top-title
align: c
color: light
---

:: title ::

# GHC RISC-V status

:: content ::

- LLVM Backend
- RTS Linker
- Native Code Generation Backend
  - Fullfills whole testsuite
- Tier 3
  - Due to lack of powerful hardware (CI), there are no official binary distributions, yet
  - Probably not much used yet
    - Happy to receive bug reports!
- SIMD (Vector) in NCG support WIP

---
layout: section
color: sky
align: c
---

# Vector (SIMD) Support

---
layout: top-title
align: c
color: light
---

:: title ::

# Vector Register Configuration

:: content ::

- Problem: Applications need very different vector sizes

  - Embedded chips should save silicon
  - HPC may need big vectors
  - usually a tradeoff
  - usually max vector sizes are bound to ISA features
  - Standard allows 32 (*Zvl32b*) to 65,536 bits per vector register

---
layout: top-title
align: c
color: light
---

:: title ::

# Vector Register Configuration

:: content ::

- RISC-V approach:

  1. Make effective register width configurable -> **grouping**
      - Combine multiple vector registers to one effective
  2. Tell when a configuration doesn't fit -> **strip mining**
      - Iterate over vector chunks

- Benefits:
  - Application can dynamically react on the vector register width (VLEN)
  - HPC software can run on embedded CPUs and vice versa without recompilation

---
layout: top-title
align: c
color: light
---

:: title ::

# Vector Register Configuration Instruction(s)

:: content ::

`vsetivli <VL>, <AVL>, <SEW>, <LMUL>, <tail>, <mask>`

- `VL`: New, effective **V**ector **L**ength (in elements)
- `AVL`: **A**pplication **V**ector **L**ength
  - The desired VL
- `SEW`: **S**ingle **E**lement **W**idth
  - Width of an element: `e8`, `e16`, `e32`, `e64` (bits)
- `LMUL`: **L**ength **Mul**tiplier
  - `mf8` (LMUL=1/8), `mf4` (LMUL=1/4), `mf2` (LMUL=1/2)
  - `m1` (LMUL=1), `m2` (LMUL=2), `m4` (LMUL=4), `m8` (LMUL=8)

---
layout: two-cols-title
columns: is-6
---

:: title ::

# Vector configuration - Grouping

- Task: Increment each element of a _8bit x 8_ vector by one (128bit register width)

:: left ::

```c
void plus_one(uint8_t b[8]) {
    for(int i = 0; i < 8; i++) {
        b[i]++;
    }
}
```

:: right ::

```asm
plus_one:
        vsetivli zero, 8, e8, mf2, ta, ma
        # Load v8 as 8-bit elements at address in a0
        vle8.v v8, (a0)
        # v8[i] = v8[i] + 1
        vadd.vi v8, v8, 1
        # Store to address in a0
        vse8.v v8, (a0)
        ret
```

:: default ::

- `mf2` grouping: 1/2 * 128 = 64
- required bits: 8 * 8 = 64

<!-- https://godbolt.org/z/dGfxY7dv3 -->

---
layout: two-cols-title
columns: is-6
---

:: title ::
# Vector configuration - Grouping (2)

- Task: Increment each element of a _8bit x 16_ vector by one (128bit register width)

:: left ::

```c
void plus_one(uint8_t b[16]) {
    for(int i = 0; i < 16; i++) {
        b[i]++;
    }
}
```
:: right ::

```asm
plus_one:
        vsetivli zero, 16, e8, m1, ta, ma
        # Load v8 as 8-bit elements at address in a0
        vle8.v v8, (a0)
        # v8[i] = v8[i] + 1
        vadd.vi v8, v8, 1
        # Store to address in a0
        vse8.v v8, (a0)
        ret
```

:: default ::

- `m1` grouping: 1 * 128 = 128
- required bits: 8 * 16 = 128

<!-- https://godbolt.org/z/jWGvWsbE4 -->

---
layout: two-cols-title
columns: is-6
---

:: title ::

# Vector configuration - Grouping (3)

- Task: Increment each element of a _8bit x 32_ vector by one (128bit register width)

:: left ::

```c
void plus_one(uint8_t b[32]) {
    for(int i = 0; i < 32; i++) {
        b[i]++;
    }
}
```

:: right ::

```asm
plus_one:
        # 32 doesn't fit into an immediate, use a register
        li a1, 32
        vsetvli zero, a1, e8, m2, ta, ma
        # Load v8 as 8-bit elements at address in a0
        vle8.v v8, (a0)
        # v8[i] = v8[i] + 1
        vadd.vi v8, v8, 1
        # Store to address in a0
        vse8.v v8, (a0)
        ret
```

:: default ::
- `m2` grouping: 2 * 128 = 256
- required bits: 8 * 32 = 256


<!-- https://godbolt.org/z/bjr57ohsr -->

<!-- https://llvm.org/devmtg/2023-10/slides/techtalks/Lau-VectorCodegenInTheRISC-VBackend.pdf -->

---
layout: two-cols-title
columns: is-6
---

:: title ::

# Vector configuration - Strip-Mining

- Task: Increment each element of a _8bit x 32_ vector by one (128bit register width)

:: left ::
```c
void plus_one(uint8_t b[32]) {
    for(int i = 0; i < 32; i++) {
        b[i]++;
    }
}
```

- Iterations:
  1. `t0` = 16; `a1` = 32; `a0` = `b[0]` = `b`
  1. `t0` = 16; `a1` = 16; `a0` = `b[16]` = `b + 16`

:: right ::

```asm
plus_one:
        # Start with 32 elements
        li a1, 32
loop:
        # Configure to get the real VL in t0
        vsetvli t0, a1, e8, m1, ta, ma
        # Perform computation on chunk
        vl1r.v v8, (a0)
        vadd.vi v8, v8, 1
        vs1r.v v8, (a0)
        # Update pointers and counters for next chunk
        # Move pointer a0 forward by VL*element_size
        add a0, a0, t0
        # Reduce remaining elements (a1 -= VL)
        sub a1, a1, t0
        # Repeat if there are remaining elements
        bnez a1, loop
end:
        ret
```

---
layout: top-title
align: c
color: light
---

:: title ::


# Vectors: Questions to investigate

:: content ::

- How can we allocate register groups? (Virtual registers that cover multiple consecutive registers)
- How to optimize for minimal vector re-configuration?
  - My naive approach is to fold over the final instructions in the Assembly emitting stage (`Ppr.hs`)

---
layout: section
color: sky
---

# NCG development: Tipps & Tricks

---
layout: top-title
align: c
color: light
---

:: title ::


# Compiler Explorer (Godbolt)

:: content ::

- Learn from others
- C and LLVM IR are good choices
- Intrinsics are a typed way to play with Assembly

---
layout: top-title
align: c
color: light
---

:: title ::

# ghc.nix

:: content ::

- https://gitlab.haskell.org/ghc/ghc.nix
- Nix env to build GHC
  - Cross-compiler envs possible

---
layout: top-title
align: c
color: light
---

:: title ::

# Run test cross with Qemu

:: content ::

---
layout: top-title
align: c
color: light
---

:: title ::

# test-primops

:: content ::

- https://gitlab.haskell.org/ghc/test-primops
- QuickCheck tests for PrimOps
- Compares your GHC to another version
  - Cross possible

---
layout: top-title
align: c
color: light
---

:: title ::

# Build Compiler with LLVM

:: content ::

- Focus on small bits: One at a time
- Build GHC itself and libraries with `-fllvm`
- Focus on one test / features at a time

---
layout: top-title
align: c
color: light
---

:: title ::

# Reduce problems

:: content ::

- Adjust tests
  - Build the smallest reproducer possible
  - Reading a lot of Assembly or Cmm can be very exhausting
- Run testsuite subsets
- Write small Cmm reproducers by hand

---
layout: top-title
align: c
color: light
---

:: title ::

# Your are not alone!

:: content ::

- Matrix group
- Mailing list
- Discourse

---
layout: top-title
align: c
color: light
---

:: title ::

# Hunting Heisenbugs

:: content ::

- Bugs that disappear when you "look" at them
  - Trace logs and debuggers (GDB) change the timing of programs and execution at CPU-level
- Trace instructions and/or CPU state with Qemu
- My worst Heisenbug was a missing memory barrier (program cache flush, `fence.i` instruction) in the linker
  - Illegal instruction exceptions at weird places
  - Gone when the timing changed e.g. by adding trace logs

---
