### [Kernel level exception handling in Linux](Documentation/x86/exception-tables.txt)
The old version of verifying the user space address and its size against process's vma has a performance overhead (overhead for every case, even on valid addresses).

Whenever the kernel tries to access an address that is currently not accessible, the CPU generates a page fault exception and calls the page fault handler `do_page_fault`. In `arch/x86/mm/fault.c`. The parameters on the stack are set up by the low level assembly glue in arch/x86/kernel/entry_32.S. The parameter regs is a pointer to the saved registers on the stack, error_code contains a reason code for the exception

`do_page_fault first` obtains the unaccessible address from the CPU control register `CR2`.

To be studied: the details of the `fixup`
