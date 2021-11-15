## [Scatterlist Chaining at LWN](https://lwn.net/Articles/234617/)
> DMA assumes that the data to be transferred is stored contiguously in memory. When the I/O buffer is in kernel space, the kernel can often arrange for it to be physically contiguous - though that gets harder as the size of the buffers gets larger. If the buffer is in user space, it is guaranteed to be scattered around physical memory. So it would be nice if DMA operations could work with buffers which are split into a number of distinct pieces.

> Once the list has been filled in, the driver calls `dma_map_sg`
