# Let's write a Malloc

This project aims to implement malloc and see how it works with existing programs. 

The project follows [Dan Luu's tutorial guide](https://danluu.com/malloc-tutorial/) on how to build a malloc. Whilst the basic theory of malloc and dynamic memory allocation is summarised onto this README, the implemented version of malloc is not complete as it does not solve fragmentation nor thread safety. 

## Why Dynamic Memory exists

Dynamic memory allocation is the process of allocating and managing memory during a program's execution (runtime), rather than when the program is compiled. It allows a program to request the exact amount of memory it needs while it is running and release it so that it can be reused.

You need `malloc` when the size of something is unknown at compile time, or when it must outlive the function that created it. C already gives you memory automatically in every other case.

A C program stores data in three main places:

|Where | What lives there | Lifetime | Who manages it |
| ----- | ----- | ----- | ----- |
| Stack | Local variables, function arguments, return addresses | Until the function returns | The compiler, automatically |
| Data/BSS segments | Global and static variables | The whole program run | Fixed at compile/load time |
| Heap | Anything you ` malloc` | Until you free it | You |

So `int arr[100];` or `struct node n;` inside a function needs no malloc: it lives on the stack. You utilise the heap with two conditions:

1. **Size known only at runtime.** The user types how many records to load, or a file is an unknown length.
2. **Lifetime longer than the function.** An `insert()` function creates a linked-list node that the caller keeps using after `insert()` returns. A stack variable would be destroyed on return, leaving a dangling pointer.

The price of the heap is that nothing cleans it up for you. Every `malloc` needs a matching `free`. 

---

### The heap, the program break, and brk/sbrk

The heap is a region of your process's virtual address space that grows upward, and its top edge is called the program break. Moving the break up gives your process more heap; moving it down gives some back.

<p align="center"> <img src="assets/Virtual address space.png" width="500"> </p>

Your allocator only ever works inside the highlighted heap region, growing it by pushing the break to the right.

**Virtual, not physical**. Every process sees its own private range of addresses. The kernel maps those virtual pages onto the physical RAM, and does it lazily: a page only gets real RAM the first time you touch it (a page fault). Pages can also be swapped to disk. 

> "The heap is part of the address space, backed by RAM on demand"

#### brk and sbrk

`brk` is a **system call**: a request from your program to the kernel. It is not a library that talks to hardware. The kernel updates your process's memory mappings and moves the break.

| `Call` | What it does | Returns on success | Returns on failure |
| ----- | ----- | ---- | ----- |
| `brk(addr)` | Sets the break to exactly `addr` | `0` | `-1`, errno = ENOMEM |
| `sbrk(n)` | Moves the break by `n` bytes | The **old** break (start of new memory) | `(void *) -1`, errno = ENOMEM |
| `sbrk(0)` | Moves nothing | The current break | — |

`sbrk` is a libc wrapper on Linux: it calls `brk` and remembers the old value so it can return it. Note the failure value is `(void *) -1`, **not** `NULL`. Checking for NULL is a classic bug.

**Why real allocators don't rely on sbrk alone**
 - The heap is one contiguous region, so the break can only shrink from the top. A freed block in the middle can never go back to the OS this way.
 - glibc's real malloc uses `sbrk` for small requests, but for requests of 128 KB or more it calls `mmap`, which creates a separate mapping that can be returned to the OS independently.

 ---
 
 ### What the four functions must do 

Each function has a contract that programs depend on and if the replacement breaks it, unrelated library code can crash.

| Function | Contract | Edge cases to handle | 
| ----- | ----- | ----- |
| `malloc(size)` | Returns a pointer to at least `size` bytes, suitably aligned. Contents are **not** initialised. | `size==0` --> NULL or a unique pointer that `free` accepts. Failure --> NULL, errno = ENOMEM. |
| `free(p)` | Releases a block from an earlier malloc/calloc/realloc. Returns nothing. | `p==NULL` --> do nothing. Double free or a foreign pointer --> undefined behaviour. |
| `calloc(n, size)` | Allocates `n x size` bytes, all set to zero. | `n x size` overflows --> must fail, not allocate a tiny block. |
| `realloc(p, size)` | Resizes a block, keeping contents up to the smaller of old and new size. May move it. | `p==NULL` --> acts as `malloc(size)`. Failure --> NULL, and the **old block is untouched**. |

**The hidden header**
`free(p)` is given no size, so the allocator must store the size itself. The standard trick is a **metadata header** placed just before the address handed to the user.

`[ header: size, next, free flag][user data .............] //malloc returns this address`

To free, the allocator steps back one header from `p` and reads the size and flags there. This is also why writing before the start of a malloc'd buffer is so destructive: it corrupts the allocator's own bookkeeping.

**Alignment**
The returned address must be suitably aligned for any built-in type. On x86-64 that means a multiple of 16 (glibc's guarantee). A misaligned `double` or SIMD value can be slow or crash outright.

Approach: round every requested size up to a multiple of 16, make the header size a multiple of 16, and make sure the very first block starts on a 16-byte boundary. Then every block that follows is aligned automatically.

**Uninitialised memory**
`malloc` hands back whatever bytes were there before, which may be data from a block you freed earlier. Never assume zeros. That is what `calloc` is for. 

---

### Freeing, Leaks, Fragmentation and Threads

**What free actually does**
`free` usually does not return memory to the OS. Because the heap is contiguous and the break only moves at the top, a freed block in the middle is simply **marked reusable**. The next `malloc` that fits can take it.

#### The three classic bugs

| Bug | What happens | Consequence |
| ----- | ----- | ----- |
|Memory Leak | You lose the last pointer to a block without freeing it | That block can never be freed or reused. Long-running programs (servers, daemons) slowly run out of memory. The OS reclaims everything when the process exits. |
|Double free | You call `free` twice on the same pointer | The allocator's list gets corrupted; later `malloc`s can return the same block twice |
|Use-after-free | You read or write a block after freeing it | You may overwrite another allocation's data or the allocator's headers. A common security exploit |

#### Fragmentation

- **External fragmentation**: total free memory is large enough, but it is split into smaller non-adjacent holes, so a big request cannot fit anywhere. Fixed by coalescing (merging neighbouring free blocks)
- **Internal fragmentation**: wasted space inside an allocated block. You ask for 10 bytes and get 16 because of alignment, or you reuse a 4KB free block for a 64-byte request. Fixed by **splitting** a large free block into the used part and a smaller free remainder. 

#### Thread safety

The allocator's linked list is shared state. If two threads call `malloc` at once, both can grab the same free block, or both can call `sbrk` and interleave. glibc protects its structures with mutexes and per-thread arenas.