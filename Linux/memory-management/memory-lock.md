
## Effective Synchronization on Linux/NUMA Systems by Christoph Lameter

### Basic Atomicity
> The NUMA interlink uses a hardware cache consistency protocol to provide a coherent view of memory

> The NUMA interlink uses a hardware cache consistency protocol to provide a coherent view of memory in the system as a whole to all processors.

### Cache Line
> The contents of the cache line are then locally available through the cache of cache lines in the node. A cache line may be either be acquired as a shared cache line that only allows read access or as an exclusive cache line.

> Write access is only allowed on a cache line that is held with exclusive access. This guarantees a cache line level atomicity of writes across the hardware consistency domain.

> One cache line can then be acquired by a processor and all related data can be processed from the same cache line without any additional memory operations.

> The copies that other processors are holding must be invalidated if one processor wants to hold the line as exclusive

> a bouncing cache line. The constant renegotiation of the ownership of the cache line may cause lots of traffic across the NUMA link which may
become a performance bottleneck.

### Atomic nature of processor operations

> The processor interfaces with the cache in a way that guarantees the atomicity of certain operations. Itanium the guarantee is that operations up to 64 bit are atomic

#### RCU in list update
> We need a write barrier between the setup of the new entry and the update of the pointer. The write barrier insures that all writes before the barrier become visible to other processors before any writes after the barrier. Without write ordering the new start pointer value could become visible to other processors before the content of the new entry.

> Read barrier insures that data read after the barrier are actually retrieved after the read operations before the barrier.

> The read and write barriers are Linux functions that are mapped to the same Itanium memory fence instruction.

> A memory operation with acquire semantics will insure that the access is visible before all subsequent accesses. Release semantics imply that all prior memory accesses are made visible before the memory access in the instruction.7 These semantics will obviously not affect values cached in registers by the code generated through the compiler. This means that the compilers must also in these cases cooperate to have the proper effect

rcu_read_unlock()
> These are not real lock operations. The functions are used to maintain a counter of the number of active readers. If the number of readers reaches zero then it is guaranteed that no link to the list element exists anymore and the freed list element can be finally deallocated.

> Barriers and the regular loads and stores (which are atomic) are the most efficient means for synchronization in a NUMA system since they do not require slow atomic semaphore instructions. However, the elements to control atomicity discussed so far cannot guarantee that a single processor knows that only itself caused a state change in a memory location.

### Atomic semaphore instructions
> The Itanium processor provides a number of atomic read modify write operations called Semaphore Instructions. Semaphore instructions are expensive because they acquire an
exclusive cache line and then do a read modify write cycle on a cache line atomically. Semaphore instruction are always non-speculative. This means that atomic
semaphore instructions result in pipeline stalls.

> Without semaphore instructions multiple processors may change a memory location but there is no way for the processor to tell that it was the unique processor whose store
effected the change.

#### Compare and exchange
> The compare and exchange instruction allows one to specify the content that a memory location is expected to have and a new value that is to be placed into that memory
location. The atomic operation is performed in the following way. First an exclusive cache line is acquired and all status changes to the cache line are stopped. Then the
contents of the memory location are compared with the expected value. If the memory location has the expected value then the new value is written to the memory location.
Only then is the cache line allowed to change state or be acquired by another processor.

> certain that only one processor obtains exclusive access to some resources and others are excluded from a resource protected

#### Fetchadd
> tchadd adds a value to a memory location atomically and returns the result.

### Spinlocks
> Spinlocks are designed to be fast and simple. Only limited nesting of locks is allowed, there is no deadlock prevention mechanism and no explicit management of contention.

> A variety of spinlock types exists to deal with concurrent interrupts or bottom handlers

> The spinlock insures that there are no concurrent operations in the critical section

> Spinlocks are realized on Itanium using a single 32 bit value that is either 0 (unlocked) or 1 (locked). The state change from 0 -> 1 is done using a CMPXCHG. Then a read barrier is needed to insure that data protected by the lock is reread by the processor so that the state after the possible completion of another critical section can be obtained. Unlocking is simply a write barrier to insure that modifications done within the lock are visible to others before the lock appears to be available. Then a zero is written to the lock.

> If an attempt to acquire the lock using CMPXCHG fails then we enter into a wait loop. A CMPXCHG requires the processor to acquire the cache line containing the lock for
exclusive access. If the operation fails then it is likely that the other processor holding the lock will read or write to data in the cache line and thus exclusive access must be transferred back to the processor holding the lock leading to the cache line bouncing back and forth.
> If the CMPXCHG would simply be retried on failure then there is a high likelihood that the cache line will continually bounce between the processor holding the lock and the processor trying to acquire the lock until the lock is acquired by another processor. In order to limit cache line bouncing, retries simply read the lock value and wait using regular load instruction while the contents of the lock value is not zero. If the contents are zero then another CMPXCHG is attempted to acquire the lock.
