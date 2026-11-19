# dynamic memory allocator 

built a performant (high utilization & high throughput), multi-threaded dynamic memory allocator in C++20. benchmarked against glibc's malloc, jemalloc, tcmalloc, mimalloc. profiled using gprof, gcov, & perf.

## how it works
 
memory flows from OS -> 4MiB chunks -> 4KiB pages -> either a heap full of pages (medium) or a page split into same size blocks (small). large requests skip chunks entirely

huge allocation cutoffs (4MiB chunks with 4KiB pages):
- small (up to half a page): subpage granularity (bitmap) 
- medium (2KiB - 1 MiB): page granularity (free list for free page runs)
- large (> 1 MiB): own mmap / munmap system calls 

## design decisions & tradeoffs
 
### 1. memory
 
each process has its own virtual address space where the OS lazily hands out memory in 4KiB pages (page fault when first touched). asking for memory really means asking the kernel to make a range of addresses valid for calling the process.
 
**naive:** sbrk: grows the heap by n bytes moving its brk pointer
- problem: one shared region (a single brk pointer) so memory can only be returned from the top and must be locked across threads as its a shared state

**mine:** mmap: asks the kernel for a new region of n bytes (page-aligned); where the kernel looks for any unused gap in the address space
- enables multiple regions of memory so each thread can get its own regions (no shared state => no lock) and can give memory in any order (not just from the top like from sbrk)

**tradeoff:** syscalls take 1 us so allocator asks for big chunks rarely to split into pages
 
mmap will return 4 MiB (not KiB) bytes for each system call (will experiment on bigger and smaller sizes when benchmarked)
 
giving back memory to the OS: naive options either never give memory back (stuck at peak) or munmap right away (give back and take again over and over). production allocators (jemalloc, mimalloc) keep a small cache with a few empty chunks per thread ready for reuse and anything above gets released (madvise) (used same technique). common reuse case needs no system calls and memory use stays bounded.

### 2. block layout & metadata

**naive**: problem with existing malloc:
- information about free blocks are kept in free blocks (header, footer, succ/pred pointers) meaning malloc must touch the memory of every free block it visits (problematic if working set of program is already too large causing cache & TLB misses)
- malloc allocations may require two pages and two TLB entries if allocated between two pages boundaries (straddles the boundaries)

**mine**: instead, use the phkmalloc design (pginfo + page directory)
- keep malloc metadata separate (in each chunk's header, not inside the blocks)
- two level page allocator (large allocations at page granularity, small allocations performed at sub-page granularity)
- pages are 4KiB (same as OS pages)

use page directory to make free(p) O(1) as its the same as indexing into an array (know the start of heap, pointer to page, page aligned), otherwise would be O(all pages) to find the page we want to free traveresing the heap

phkmalloc uses a single page directory but since our allocator is multi-threaded that would require tons of synchronization which hinders our performance. instead, we create multiple page directories for each 4MiB chunk (4MiB algined) so each thread can just access its own page directory without needing to be synchronized. 

**tradeoff**: requires each chunk to hold bytes for its header

### 3. allocation tiers / cutoffs

**naive:** no cutoff meaning everything that fits goes in a chunk. large allocations on page granularity would cause external fragmentation (i.e with no cutoff that means a 2 MiB block in a 4 MiB chunk leaves 2 MiB that a 2.5 MiB request can't use)

**mine:** cutoff at 1MiB caps the worst case at ~25% per chunk. anything bigger gets its own mmap, where the 1 us syscall is small next to the block's own page faults
 
**tradeoff:** a syscall on every alloc/free over 1MiB (will try other variant sizes)

### 4. page level allocator

**naive**: scan the page directory for enough free pages, O(all pages)

**mine**: use free list to make allocate(n) O(free page runs) rather than O(all pages) as it stores only runs of contiguous free pages so just find number of pages needed to serve the allocation request

page level allocator: add page run to free list in address order with coalescing (to combine multiple free pages runs into one larger one to reduce external fragmentation)

**tradeoff**: inserting in address orders requires serach for the spot though

### 5. sub-page allocator

powers of 2 for sub-page level allocation as at subpage level theres no splitting, coalescing overhead, and sizes divide 4KiB pages evenly

track free blocks at subpage granularity, we'll use a bitmap for allocated to avoid malloc requiring to page free blocks to free a free list when working set is already big causing cache misses

sub-page level allocator: clear subpages bit in the bitmap as all chunks are same size in page on purpose (no coalescing)

**tradeoff**: powers of 2 can waste about half of a block if allocation request is just slightly above half way of a block

### 6. thread safety & per-thread heaps

ideally, we want each thread to not share data to begin with to avoid needing locks. however, in an allocator, threads must share, so we need to synchronize through narrow channels like the remote free list

from jeff deans numbers: mutex lock/unlock takes 25ns (worse on contention)

**naive**: one global lock would require one mutex around malloc and free. unscalable (1 thread uses the locks leaves the n - 1 sitting idle)

**mine**: per thread caches: each thread has its own heap. most malloc and free calls touch the calling thread's data only so theres no lock or atomic operations. only global pools (empty chunks and free heaps) use a lock, which is on the slow path (malloc/free remain lock free) 

**per-thread heaps**: each thread gets its own heap (4 MiB chunks, page allocator, subpage allocator)

no lock contention on small and medium paths so throughput increases; cache stays warm as thread's memory comes from its own chunks and prevents false sharing (no ping pong effect of reading and writing same cache lines over and over across threads). 

**tradeoff**: though it lowers utilization as each heap now fragments independent of each other

### 7. remote free lists

only the owning thread may modify its heap's bookkeeping (bitmaps, page lists, page-run
free list), so the owner's malloc/free need no locks or atomics

if a thread calls free(p) on a block whose chunk belongs to another thread's heap, it must not touch that heap's bookkeeping
instead:
- each heap has one shared field: a remote-free list head
- the freeing thread pushes the block onto that list atomically (CAS)
- when the owner runs out of free blocks, it atomically takes the whole list and frees each block through its normal local path

**tradeoff**: costs one atomic operation on a remote free but malloc/free on owner thread don't require locks or atomics. allowing threads to modify another threads heap would require locks on every operation

### 8. thread exits

if a thread exits then its heap still holds blocks other threads may be using and will free later. remote free list held blocks that are meant to free at a later time, but now the owner thread has exited

**naive**: doing nothing to clean up could cause memory to leak. freeing all the memory in the heap would cause segfaults as other threads may have blocks allocated in the heap

**mine**: instead, on exit, thread does clean up itself:
- frees the blocks on its remote free list
- returns empty chunks to global empty chunk pool 
- marks heap as unowned and pushes it onto a global free heap pool so its memory can be reused by the next thread

## development
 
- **platform**: linux x86-64
- **language**: C++20
  - avoiding hidden runtime costs (exceptions, RTTI, virtual functions, std containers that allocate)
  - enables using constexpr, std::atomic, alignas, and templates
- **testing & correctness**: GoogleTest & debug/release builds
- **CI**: GitHub Actions builds and runs the tests (debug, release, ThreadSanitizer)

## benchmarks
 
for throughput, peak memory, and thread scaling vs glibc, jemalloc, tcmalloc, mimalloc