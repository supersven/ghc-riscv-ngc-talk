# GHC's RISC-V Native Code Generation Backend

RISC-V is an exciting, new architecture that has received a lot of attention by
the Open Source community lately. We will start by understanding where this
excitement comes from and why RISC-V is not just "yet another architecture".
Here we will also introduce some basic vocabulary to better understand
requirements for compiler engineers and software packagers.

Then, we will take a look at the current state of RISC-V support in GHC with a
focus on the Native Code Generation Backend ("NCG".) Furthermore, future
improvements will be discussed (finally, we're at the dawn of Zurihac 2025 ;)
.) A look at the innovative management facilities for variable vector registers
lengths in the instruction set may even inspire further research.

Last (but not least), implementing the RISC-V NCG was a challenge regarding
software development environment setup, debugging and problem-solving
strategies. There are a couple of tricks and hints be shared with future NCG
developers. Most of them could be useful to tackle other tasks in the lower
parts of the compiler pipeline as well.
