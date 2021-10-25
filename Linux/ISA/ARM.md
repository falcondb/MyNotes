## Arm32
[MS:ARM32 ABI conventions](https://docs.microsoft.com/en-us/cpp/build/overview-of-arm-abi-conventions?view=msvc-160)
###Registers
  - 16 * 32-bit general purpose integer registers
###Register usage:
  - `X0-X3` Parameter `X0 X1` Result Registers
  - `X11` Frame base
  - `X12` used by linkers to insert veneers
  - `X13` Stack pointer
  - `X14` _link register_
  - `X15` Program counter _PC_

###Parameter passing
  Refer to the doc
###Return values
  - Integer type values are returned in r0, optionally extended to r1 for 64-bit return values. VFP/NEON floating-point or SIMD type values are returned in s0, d0, or q0
###Stack
  - 4-byte aligned, and must be 8-byte aligned at any function boundary
###Red zone
The 8-byte area immediately below the current stack pointer is reserved for analysis and dynamic patching.

## ARM64
[MS:ARM64 ABI conventions](https://docs.microsoft.com/en-us/cpp/build/arm64-windows-abi-conventions?view=msvc-160)
###Registers
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
###Parameter passing
 - Initialization
  - Next General-purpose Register Number (NGRN) ==> 0
  - Next SIMD and Floating-point Register Number (NSRN)  ==> 0
  - next stacked argument address (NSAA) = `SP`
- Pre-padding and extension of arguments
  - If the argument type is a Composite Type whose size can't be statically determined by both the caller and the callee, the argument is copied to memory and the argument is replaced by a pointer to the copy
  - no changes for _HFA (Homogeneous Floating-point Aggregate) HVA (Homogeneous Short-Vector Aggregate)_
  - If the argument type is a Composite Type larger than 16 bytes, then the argument is copied to memory allocated by the caller, and the argument is replaced by a pointer to the copy
  - If the argument type is a Composite Type, then the size of the argument is rounded up to the nearest multiple of 8 bytes
- Assignment of arguments to registers and stack
  - Steps for allocating floating, integer and composite arguments to floating, integer registers or stack (when registers are taken). Refer to the doc for the details

###Return values
  - Integral values are returned in `x0`. floating-point values are returned in `s0, d0, or v0`, as appropriate.
  - Floating values, refer to the doc
  - Composite types by non-member functions or static member functions
    - sizeof() <= 8, `x0`
    - sizeof() <= 16, `x0 x1`
    - sizeof() > 16, caller reserves a memory block, passes in `x8`
  - Other types
    - caller reserves a memory block, passes in `x0`; if `x0` is `this`, passes it in `x1`.

###Stack
  - 16-byte aligned
###Red zone
  - The 16-byte area immediately below the current stack pointer is reserved for use by analysis and dynamic patching scenarios.  
