---
title: GHC's RISC-V Native Code Generation Backend
subtitle: Haskell Implementors' Workshop 2025
author: Sven Tennie
...

# RISC-V

- 32bit *R*educed *I*nstruction *S*et as base
  - RV32I Base Integer Instruction Set -> ~40 instructions
  - Basic interpreter can be built in an afternoon
- Augmented by many extensions (sub-standards)
- ISA like playing with Lego bricks
- ISA is open source, implementations (SOCs) not necessarily
  - License: _Creative Commons Attribution 4.0 International_
  - Development on GitHub
  - Conceptualization in working groups at _RISC-V International_ foundation
    - Free membership for individuals
- Everyone is free to build a RISC-V processor:
  - Several vendors
  - Hobbiests
    - Fun fact: Some hobbiests even tape out their designs via e.g. tinytapeout.com

# RISC-V Status

- Standard (ISA, Calling Convention, ...) pretty complete
- Vibrant community
- Lack of powerful hardware
  - No good cloud options -> No native cloud CI
- Lot's of movement though
  - New boards and chips appear frequently
  - Many manufacturers
- There are still some dragons ...
  - Tools don't support the full instruction set
  - Tools sometimes still have bugs ...
  - Cores may have bugs
  - Core may not adhere to the ratified standards because it pre-dates it

# ISA naming scheme

- Start with a base ISA: RV32I, RV64I or RV64E
- Add the extensions in canonical order
- The ISA is pretty new, so extensions' versions can usually be ignored
  - If not, the format is `<extension><major>p<minor>`
- Reduce common extensions to sets (e.g. *G*eneral for _IMAFDZicsr_Zifencei_)
- E.g. `RV64IMAFDZicsr_ZifenceiV1p0` -> `RV64GV1p0` -> `RV64GV1` or `RV64GV`
- _Profiles_ (e.g. RVA23) define minimum requirements to simplify this
  - Otherwise, buying and building for a consumer computer could be a nightmare
  - (It still is, because many vendors don't mention profiles yet on their marketing pages)
  - Linux distributions handle this by relying on a small extension set (usually _RV64GC_)

# GHC RISC-V History

- LLVM backend by Andreas Schwab
- Moritz Angerman and Sven Tennie accidentally started NCG at the same time

  - Moritz switched to mentor role
  - Sven continued to hack
  - Andreas built CI support at SuSE with patch files

- Strong advice: Reach out and team up
  - I wouldn't have imagined that such great collaboration between former strangers would be possible!
  - It is!

# GHC RISC-V status

- LLVM Backend
- RTS Linker
- Native Code Generation Backend
  - Fullfills whole testsuite
  - Released: 9.12
- Tier 3
  - Due to lack of powerful hardware (CI), there are no official binary distributions, yet
  - Probably not much used yet
    - Happy to receive bug reports!

# Vector Register Configuration

- Problem: Applications need very different vector sizes

  - Embedded chips should save silicon
  - HPC may need big vectors
  - usually a tradeoff
  - usually max vector sizes are bound to ISA features
  - Standard allows 32 to 65,536 bits per vector register (TODO: Check this!)

- RISC-V approach:

  1. Make effective register width configurable -> grouping
  1. Tell when a calculation doesn't fit -> strip mining

- Benefits:
  - Application can dynamically react on the vector register width (VLEN)
  - HPC software can run on embedded CPUs and vice versa without recompilation

# Vector configuration

- Task: Increment each element of a _8bit x 8_ vector by one (128bit register width)

```c
#include <stdlib.h>
#include <riscv_vector.h>

uint8_t* plus_one(uint8_t b[8]) {
    for(int i = 0; i < 8; i++) {
        b[i]++;
    }
    return b;
}
```

=>

```asm
plus_one:
        vsetivli        zero, 8, e8, mf2, ta, ma
        vle8.v  v8, (a0)
        vadd.vi v8, v8, 1
        vse8.v  v8, (a0)
        ret
```

<!-- https://godbolt.org/z/dGfxY7dv3 -->

# Vector configuration (2)

- Task: Increment each element of a _8bit x 16_ vector by one (128bit register width)

```c
uint8_t* plus_one(uint8_t b[16]) {
    for(int i = 0; i < 16; i++) {
        b[i]++;
    }
    return b;
}
```

=>

```asm
plus_one:
        vl1r.v  v8, (a0)
        vsetivli        zero, 16, e8, m1, ta, ma
        vadd.vi v8, v8, 1
        vs1r.v  v8, (a0)
        ret
```

<!-- https://godbolt.org/z/jWGvWsbE4 -->

# Vector configuration (3)

- Task: Increment each element of a _8bit x 32_ vector by one (128bit register width)

```c
uint8_t* plus_one(uint8_t b[32]) {
    for(int i = 0; i < 32; i++) {
        b[i]++;
    }
    return b;
}
```

=>

```asm
plus_one:
        vl2r.v  v8, (a0)
        vsetvli a1, zero, e8, m2, ta, ma
        vadd.vi v8, v8, 1
        vs2r.v  v8, (a0)
        ret
```

<!-- https://godbolt.org/z/bjr57ohsr -->

<!-- https://llvm.org/devmtg/2023-10/slides/techtalks/Lau-VectorCodegenInTheRISC-VBackend.pdf -->

# Vectors: Questions to investigate

- How can we allocate register groups? (Virtual registers that cover multiple consecutive registers)
- How to optimize for minimal vector re-configuration?

# NCG development: Tipps & Tricks

## Compiler Explorer (Godbolt)

- C and LLVM IR
- Intrinsics are a typed way to play with Assembly

## ghc.nix

- https://gitlab.haskell.org/ghc/ghc.nix
- Nix env to build GHC
  - Cross-compiler envs possible

## test-primops

- https://gitlab.haskell.org/ghc/test-primops
- QuickCheck tests for PrimOps
- Compares your GHC to another version
  - Cross possible

## Build Compiler with LLVM

- Focus on small bits: One at a time
- Build GHC itself and libraries with `-fllvm`
- Focus on one test / features at a time

## Reduce problems

- Adjust tests
  - Build the smallest reproducer possible
  - Reading a lot of Assembly or Cmm can be very exhausting
- Run testsuite subsets
- Write small Cmm reproducers by hand

## Your are not alone!

- Matrix group
- Mailing list
- Discourse

## Hunting Heisenbugs

- Bugs that disappear when you "look" at them
  - Trace logs and debuggers (GDB) change the timing of programs and execution at CPU-level
- Trace instructions and/or CPU state with Qemu
- My worst Heisenbug was a missing memory barrier (program cache flush, `fence.i` instruction) in the linker
  - Illegal instruction exceptions at weird places
  - Gone when the timing changed e.g. by adding trace logs
