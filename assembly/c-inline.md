[Brennan's Guide to Inline Assembly](http://www.delorie.com/djgpp/doc/brennan/brennan_att_inline_djgpp.html)
the underscore ("_") is how you get at static (global) C variables from assembler.
```
a        eax
b        ebx
c        ecx
d        edx
S        esi
D        edi
I        constant value (0 to 31)
q,r      dynamically allocated register (see below)
g        eax, ebx, ecx, edx or variable in memory
A        eax and edx combined into a 64-bit integer (use long longs)
```
_q_ causes GCC to allocate from `eax, ebx, ecx, and edx`. _r_ lets GCC also consider `esi` and `edi`. So make sure, if you use _r_ that it would be possible to use `esi` or `edi` in that instruction. If not, use _q_.

how to determine how the %n tokens get allocated to the arguments. It's a straightforward first-come-first-served, left-to-right thing, mapping to the _q_'s and _r_'s. But if you want to reuse a register allocated with a _q_ or _r_, you use "0", "1", "2".

If your assembly statement must execute where you put it, put the keyword `volatile` after `asm`


[Using Inline Assembly With gcc](https://www.cs.virginia.edu/~clc5q/gcc-inline-asm.pdf)
* Basic Asm
`inline`: If you use the inline qualifier, then for inlining purposes the size of the asm statement is taken as the smallest size possible

GCC does not parse the assembler instructions themselves and does not know what they mean or even whether they are valid assembler input.

a newline to break the line, plus a tab character (written as ‘\n\t’)

Extended asm statements have to be inside a C function, so to write inline assembly language at file scope (“top-level”), outside of C functions, you must use basic asm. Functions declared with the `naked` attribute also require basic `asm`

* Extended Asm - Assembler Instructions with C Expression Operands
With extended asm you can read and write C variables from assembler and perform jumps from assembler code to C labels.
`volatile`: The typical use of extended asm statements is to manipulate input values to produce output values. However, your asm statements may also produce side effects. If so, you may need to use the volatile qualifier to disable certain optimizations.

`Clobbers` A comma-separated list of registers or other values changed by the AssemblerTemplate, beyond those listed as outputs
Two special clobber arguments:
  - The "cc" clobber indicates that the assembler code modifies the flags register. On some machines, GCC represents the condition codes as a specific hardware register; "cc" serves to name this register. On other machines, condition code handling is different, and specifying "cc" has no effect. But it is valid no matter what the target.

  - The "memory" clobber tells the compiler that the assembly code performs memory reads or writes to items other than those listed in the input and output operands (for example, accessing the memory pointed to by one of the input parameters). To ensure memory contains correct values, GCC may need to flush specific register values to memory before executing the asm. Further, the compiler does not assume that any values read from memory before an asm remain unchanged after that asm; it reloads them as needed. Using the "memory" clobber effectively forms a read/write memory barrier for the compiler.

`%%` Outputs a single `%` into the assembler code. `%{ %| %}` Outputs the characters into the assembler code.

`%=` Outputs a number that is unique to each instance of the asm statement in the entire compilation. This option is useful when creating local labels and referring to them multiple times in a single template that generates multiple assembler instructions.

Reference the name in the assembler template by enclosing it in square brackets `%[Value]`. Use the position of the operand in the list of operands in the assembler template `%0` for the first operand.

Output constraints must begin with either ‘=’ (a variable overwriting an existing value) or ‘+’ (when reading and writing).
`r` for register and `m` for memory. When you list more than one possible location `=rm`, the compiler chooses the most efficient one based on the current context.
`+` constraint modifier count as two operands (that is, both as input and output)
Use the `&` constraint modifier (see Modifiers) on all output operands that must not overlap an input.

If the assembler code does modify anything, use the "memory" clobber to force the optimizers to flush all register values to memory and reload them if necessary after the asm statement. Also note that an asm goto statement is always implicitly considered volatile.

[How to Use Inline Assembly Language in C Code](https://gcc.gnu.org/onlinedocs/gcc/Using-Assembly-Language-with-C.html#Using-Assembly-Language-with-C)


[The Arm® Compiler User Guide provides information for users new to Arm Compiler 6.](https://www.keil.com/support/man/docs/armclang_intro/armclang_intro_Chunk146806297.htm)

[A guide to inline assembly for C and C++](https://www.ibm.com/developerworks/rational/library/inline-assembly-c-cpp-guide/index.html)
