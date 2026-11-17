# dynamic memory allocator 

built a high ultilization & throughput dynamic memory allocator from scratch in C. benchmarked against glibc's malloc.

## design decisions

**thread safety**: ideally, we want each thread to not share data to begin with to avoid needing locks. however, in an allocator, threads must share, so we need to synchronize through narrow channels like SPSC queues.

from jeff deans numbers: mutex lock/unlock takes 25ns on a 2GHZ processor

- one global lock would require one mutex around malloc and free. unscalable (1 thread uses the locks leaves the n - 1 sitting idle)
- per thread caches: each thread has its own free list. most malloc and free calls touch the calling thread's data only so theres no lock or atomic operations. use a shared heap that uses a lock ONLY to refill thread's cache or take back excess free blocks (on the slow path). 
       
**memory**: each process has its own virtual address space where the OS lazily hands out memory in 4KiB pages (page fault when first touched). asking for memory really means asking the kernel to make a range of addresses valid for calling the process.

sbrk: grows the heap by n bytes moving its brk pointer
- problem: one shared region (a single brk pointer) so memory can only be returned from the top and must be locked across threads as its a shared state

mmap: asks the kernel for a new region of n bytes (page-aligned); where the kernel looks for any unused gap in the address space
- enables multiple regions of memory so each thread can get its own regions (no shared state => no lock) and can give memory in any order (not just from the top like from sbrk)
- syscalls take 1 us so allocator asks for big chunks rarely to split into pages

mmap will return 4 MiB (not KiB) bytes for each system call (will experiment on bigger and smaller sizes when benchmarked)

giving back memory to the OS: production allocators (jemalloc, mimalloc) keep a small cache with a few empty chunks per thread ready for resuse and anything above gets released (used same technique)

**block layout & metadata**: 

## development

**platform**: linux x86-64
**language**: C++20
-       avoiding hidden runtime costs (execeptions, RTTI, virtual functions, std) 
-       enables using constexpr, std::atomic, alignas, and templates