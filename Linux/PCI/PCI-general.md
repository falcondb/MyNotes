## PCI General

### [PCI Interrupts for x86 Machines under FreeBSD](https://people.freebsd.org/~jhb/papers/bsdcan/2007/article/article.html)
>Exceptions, Non-Maskable Interrupts (NMIs), Inter-Processor Interrupts (IPIs), and device interrupts all use the same interrupt mechanism on x86 CPUs.

>The operating system provides an Interrupt Descriptor Table (IDT) to the CPU that contains an array of handlers.

>Exceptions and NMIs are assigned static IDT vectors

>Thus, at one end we have a PCI interrupt line being signaled by a PCI function that needs attention. At the other end we have a CPU receiving an IDT vector. In the middle is one of those ``and then a miracle occurs'' mysteries. In this case the mystery hardware in the middle is known collectively as interrupt controllers.

Programmable Interrupt Routers (PCI Link Devices)
  - A programmable interrupt router is used to route PCI interrupt signals to interrupt inputs on another interrupt controller. A router contains several input signals and output signals. M-2-M input and output signal mapping
  - System software, such as the BIOS or operating system, is responsible for programming the interrupt router. Programming the router consists of routing each input signal that is in use to an output signal.
  - Many x86 systems, including most recent systems, include a second set of interrupt controllers known as APICs. In these systems, each CPU includes a local APIC which receives interrupt messages and uses them to assert interrupts on the CPU. The chipset includes one or more I/O APICs which are responsible for converting device interrupt signals into messages that are delivered to one or more local APICs.

*TO BE CONTINUE*


### External Interrupts in the x86 system

#### [External Interrupts in the x86 system. Part 1. Interrupt controller evolution](https://habr.com/en/post/446312/)
IRQ 0 — system timer
IRQ 1 — keyboard controller
IRQ 2 — cascade (interrupt from slave controller)
IRQ 3 — serial port COM2
IRQ 4 — serial port COM1
IRQ 5 — parallel port 2 and 3 or sound card
IRQ 6 — floppy controller
IRQ 7 — parallel port 1
IRQ 8 — RTC timer
IRQ 9 — ACPI
IRQ 10 — open/SCSI/NIC
IRQ 11 — open/SCSI/NIC
IRQ 12 — mouse controller
IRQ 13 — math co-processor
IRQ 14 — ATA channel 1
IRQ 15 — ATA channel 2

#### [External Interrupts in the x86 system. Part 2. Linux kernel boot options](https://habr.com/en/post/501660/)


#### [External Interrupts in the x86 system. Part 3. Interrupt routing setup in a chipset, with the example of coreboot](https://habr.com/en/post/501912/)
