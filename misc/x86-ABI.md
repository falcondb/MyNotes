## Instruction set architecture
### x86-64  
  - Registers: 16 * 64-bit general purpose registers; 16 * 128-bit SSE (Streaming SIMD Extensions) registers; 8 * 80b-it floating registers
  - Stack frame:
    - 0(%rbp):previous %rbp of the caller; 8(%rbp): return address; 8n+16(%rbp): function arugments in 8 bytes
    - -8(%rbp) --> 0(%rsp): local variables
  - Register usage:
    - `rax`: 1st return register;  `rbp`: caller-saved; `rcp`: 4th integer argument; `rdx`: 3rd integer argument / 2nd return register; `rsi rdi r8 r9` 2nd 1st 5th 6th integer argument; `r15` _GOT_ base pointer
  - Address space
      - 48 bit to 0x0000 7fff ffff ffff
      - Text segment from 0x40 0000; Stack segment downward from 0x800 0000 0000
  - _rFLAGS_: `CF` no carry; `ZF` No zero result; `OF` no overflow; `TF` Trap flag; `0x0200 IF` interrupt enable


  - On x86-64 Linux, %rbp, %rbx, %r12, %r13, %r14, and %r15 are callee-saved, as (sort of) are %rsp and %rip. The other registers are caller-saved.

### Arm
#### Arm32
  - Registers
    - 16 * 32-bit general purpose integer registers
  - Register usage:
    - `X0-X3` Parameter `X0 X1` Result Registers
    - `X11` Frame base
    - `X12` used by linkers to insert veneers
    - `X13` Stack pointer
    - `X14` _link register_
    - `X15` Program counter _PC_
#### Arm64
  - Registers
    - 31 * 64-bit general purpose registers
    - `XZR WZR` zero registers
    - `sp` stack pointer, `PC` program counter (not general purpose registers)
    - `MSR` instruction to read/write _system registers_ from/to general purpose register
  - Register usage:
      - `X0-X7` Parameter and `X0 X1` Result Registers.
      - `XR (X8)` to the memory allocated by the caller for returning the struct.
      - `X9-X15` Corruptible Registers
      - `IP0 (X16)` `IP1 (X17)` used by linkers to insert veneers between the caller and callee
      - `X19-X28` Callee-saved Registers
      - `FP (X29)` Frame
      - `LR (X30)` _link register_ for function return addr
      - `TPIDRRO_EL0` CPU number
      - The program counter and the stack pointer aren't indexed registers
  - Pipeline: IF, ID are in order execution; EX, MEM, WB are out of order execution
  - System call:
    - `SVC` Supervisor call: Used by an application to call the OS
    - `HVC` Hypervisor call: Used by an OS to call the hypervisor
    - `SMC` Secure monitor call: Used by an OS or hypervisor to call the EL3 firmware
  - Endian: most of the Arm implements use little endian  
