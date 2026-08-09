> Ref: [On Disk IO, Part 1: Flavors of IO](https://medium.com/databasss/on-disk-io-part-1-flavours-of-io-8e1ace1de017)

There are different flavors of IO: 

- System IO / Syscalls : (e.g open, write, read, fsync, sync, close, etc.)

These mainly comprise of IO operations that directly impacts the storage layers of the kernel's address space via kernel syscall interface.

- Standard IO : (e.g fopen, fwrite, fread, fflush, fclose, etc.)

These mainly comprise of IO operations at the application layer, where the data lives in the buffer of the application's address space.

- Vectored IO: (e.g writev, readv, etc.)

Here, reading is done from multiple streams and written to a single stream or writing is done to multiple streams and read from a single stream. This is therefore also called the scatter/gather method.

- Memory mapped IO: (e.g munmap, mmap, open, close, msync, etc. )

These are similar to System IO, except here instead of copying the pages from the page cache into the application buffer, the pages are directly mapped into the virtual address of the application via page tables. This way, the application gets a pointer to these pages in memory.

## Sectors, Blocks, Pages

Sectors are the smallest unit of data transfer for block devices, e.g hard disks, pen drives, sd cards, etc. In most block devices, sectors are of size 512 bytes. 

A block, a foundational unit of the filesystem, consists of multiple adjacent sectors and therefore its size is a multiple of 512 bytes.

IO is done through virtual memory, which caches requested filesystem blocks in memory and serves as a buffer for intermediate operations. Virtual memory works with pages, which map to filesystem blocks. So, a page is a physical unit of memory/RAM, size of which is a multiple of filesystem blocks.

![An example of sectors, blocks and pages](sector_block_page.png)

Different types of IO based two independent questions:

1. Does my data use page cache? (Buffered vs Direct I/O)
2. Does the thread wait? (Synchronous vs Asynchronous I/O)

`Buffered IO`: involves the use of a page cache for faster read/writes from memory. 

![The flow of buffered IO](buffered_io.png)

Only, exception is in the case of memory mapped IO: where a pointer to the memory pages required by the process is returned instead of a copy of the pages.

`Direct I/O`: doesn't involve a page cache. Instead here, the read/write operations happen between the application and the disk. Uses the O_DIRECT flag when opening a file.

![The flow of direct IO](direct_io.png)

Because Direct IO involves direct access to disk, bypassing intermediate kernel buffers, it is important that all operations are aligned to sector boundary, i.e. a data written onto the disk/the buffer size has to be a multiple of 512 and every operation has to have a starting offset of 512 as well (implying sector size = 512 bytes).

![How sector boundary should be maintained](sector_boundary.png)


`Synchronous IO`: It makes the process/thread blocking, i.e it waits for the entire read/write operation to complete. There exists a special flag called O_NONBLOCK, but it is effectively ignored for regular files because the kernel waits for storage operation to complete, instead of treating it as a readiness event.

`Asynchronous IO` : Although, in modern Unix systems, we can make IO async using interfaces like io_uring. This way the thread doesn't wait for the storage operation to complete and is notified upon completion.

> ==When performing a write operation that is backed by the kernel or by a library buffer, it is important to make sure that the data actually reaches stable storage, i.e disk. The errors will only appear when the data is finally stored in disk, which can be while *fsyncing* or closing the file. So, it becomes really important to observe the error code to know where it actually failed.==