---
layout: page
title: fork(), Copy-on-Write, and smaps
---

# fork(), Copy-on-Write, and smaps
...

On Linux, `fork()` is one of those deceptively simple system calls that hides a lot of interesting behavior. It creates a new process, but it does *not* duplicate memory immediately. Instead, it relies on **Copy-on-Write (COW)** to make process creation fast and cheap.
Understanding this mechanism is essential if you want to reason about memory usage, performance, and what tools like `/proc/[pid]/smaps` actually report.

## What fork() really does

When a process calls `fork()`, the kernel creates a child process that initially looks identical to the parent. The virtual address space layout is the same: same mappings, same code, same heap, same stack.
The crucial detail is that **no physical memory is copied at fork time**.
Parent and child initially **share the same physical pages**. These pages are marked read-only. As long as neither process writes to them, they remain shared.
Once a process writes to a shared page, a page fault is triggered. The kernel allocates a new physical page, copies the contents, and updates the page table of the writing process. From that moment on, the page is private to that process. This is Copy-on-Write.

## Why Copy-on-Write matters

Without COW, `fork()` would have to copy the entire address space eagerly, which would make process creation extremely expensive.
With COW, `fork()` is mostly about duplicating page tables. Actual memory copying only happens when data is modified. This is why the classic `fork()` + `exec()` pattern is viable at all.
It also explains why memory usage after `fork()` often looks confusing if you only look at high-level metrics.

## Relevant smaps fields for COW analysis

To understand what is really happening in memory, `/proc/[pid]/smaps` is the right tool. For Copy-on-Write behavior, a few fields are especially relevant.

### RSS (Resident Set Size)

RSS is the amount of physical memory currently mapped into the process.
After `fork()`, RSS can appear high for both parent and child, even though many pages are still shared. RSS does **not** distinguish between shared and private memory and is therefore misleading on its own.

### PSS (Proportional Set Size)

PSS divides each shared page by the number of processes sharing it.
This makes PSS the most meaningful metric for COW analysis. If a page is shared by two processes, each process is charged half a page. When Copy-on-Write happens, PSS shifts toward the process that triggered the write.

### Private_Dirty

Private_Dirty represents pages that are private to the process and have been modified.

This is where Copy-on-Write becomes visible. When a process writes to a previously shared page, that page moves into `Private_Dirty`. Growth here means real memory duplication has occurred.

### Shared_Clean and Shared_Dirty

These fields represent pages that are still shared between processes.

Immediately after `fork()`, most anonymous memory appears as shared. As processes start writing, shared pages decrease and private dirty pages increase.

# Hands On Lab
## Step 1: No heap
We start with a minimal process that does almost nothing. It simply spins forever.
```c
int main(void) {
    while (1);
    exit(EXIT_SUCCESS);
}
```

When inspecting `/proc/<pid>/smaps` for this process, there is **no `[heap]` section**.

This is expected. On Linux, the heap is created lazily. A heap mapping only appears once the process actually requests dynamic memory, typically via `malloc()`, which internally uses `brk()` or `mmap()`.

Since this program never allocates heap memory and only uses the stack and existing file-backed mappings, the kernel has no reason to create a `[heap]` region.

We established a clean baseline: no anonymous heap pages and no Copy-on-Write–relevant memory yet.

## Step 2: Heap created, but mostly untouched

Now we introduce a single heap allocation:

```c
int main(void) {
    uint8_t *mem = malloc(ALLOCATE_N_BYTES);
    if (mem == NULL) {
        perror("malloc");
        exit(EXIT_FAILURE);
    }
    while (1);
    exit(EXIT_SUCCESS);
}
```

This immediately creates a `[heap]` mapping in `/proc/<pid>/smaps`:

```
[heap]
Size:            132 kB
KernelPageSize:    4 kB
Rss:               8 kB
Pss:               8 kB
Private_Dirty:     8 kB
```

### What is happening here

Even though `malloc(64 KiB)` was called, only **8 kB RSS** are resident.
With **4 kB pages**, that means **2 pages are physically present**.

Why so little?

* `malloc()` mainly reserves **virtual address space**
* physical pages are committed **lazily** (on first write / page fault)
* we never write to `mem`, so most of the requested 64 KiB stay non-resident

So where do the **2 pages** come from?

The allocator (glibc `malloc`) touches a small amount of heap memory for **bookkeeping** (metadata, management, alignment). Those writes fault in a couple of pages and mark them **Private_Dirty**, which is exactly what we see here.

At this point:

* most of the heap exists only virtually
* Copy-on-Write is not involved yet (still one process)
* this step demonstrates **virtual allocation vs. physical residency (RSS)**

Next step: write into the allocated region and watch RSS and `Private_Dirty` scale with the number of pages we actually touch.

## Step 3: Touching memory makes it real

Now we allocate **and write** to the whole buffer:

```c
int main(void) {
    uint8_t *mem = malloc(ALLOCATE_N_BYTES);
    if (mem == NULL) {
        perror("malloc");
        exit(EXIT_FAILURE);
    }

    if (!memfill(mem, ALLOCATE_N_BYTES, ALLOCATE_N_BYTES, 1)) {
        return EXIT_FAILURE;
    }

    write(STDOUT_FILENO, "PARENT: ", strlen("PARENT: "));
    write_pid();

    while (1);
}
```

After this, `/proc/<pid>/smaps` shows a much higher resident set for `[heap]`:

```
[heap]
Size:            132 kB
KernelPageSize:    4 kB
Rss:              68 kB
Pss:              68 kB
Private_Dirty:    68 kB
Anonymous:        68 kB
```

### What is happening here

`malloc()` still only reserves virtual address space. The big change comes from `memfill()`:

* `memset()` writes into the allocated region
* each first write to a page triggers a **page fault**
* the kernel allocates a physical page and maps it in
* the page becomes **Anonymous** and **Private_Dirty**

So the heap pages you touched became resident.

### Why RSS is not 64 kB (and not 132 kB)

We allocated **64 KiB**, but RSS is **68 kB**. With 4 kB pages, that is **17 pages**.

The extra pages are normal and come from allocator overhead and alignment. The heap mapping is also **132 kB**, which is the allocator’s current heap segment size, not our requested size.

Key takeaway:

* **Size** = virtual heap mapping size
* **Rss** = pages actually backed by RAM (only after you touch them)
* **Private_Dirty** = pages you modified (exactly what `memset` does)

This step is the final “single-process baseline” before `fork()`: now we have real anonymous pages that can be shared and later duplicated via Copy-on-Write.

## Step 4: After `fork()` the heap becomes shared (COW baseline)

Now we allocate, touch the memory (so pages are resident), and then `fork()`:

```c
uint8_t *mem = malloc(ALLOCATE_N_BYTES);
if (mem == NULL) {
    perror("malloc");
    exit(EXIT_FAILURE);
}
if (!memfill(mem, ALLOCATE_N_BYTES, ALLOCATE_N_BYTES, 1)) {
    return EXIT_FAILURE;
}

pid_t pid = fork();
if (pid < 0) {
    perror("fork");
    exit(EXIT_FAILURE);
} else if (pid > 0) {
    write(STDOUT_FILENO, "PARENT: ", strlen("PARENT: "));
    write_pid();
    while (1);
} else {
    write(STDOUT_FILENO, "CHILD: ", strlen("CHILD: "));
    write_pid();
    while (1);
}
```

### What smaps shows (important parts)

Both parent and child report the same `[heap]` mapping and the same RSS:

* `Rss: 68 kB`

But PSS is **split**:

* `Pss: 34 kB` (parent)
* `Pss: 34 kB` (child)

And the pages are now reported as shared:

* `Shared_Dirty: 68 kB`
* `Private_Dirty: 0 kB`

### What is happening here

After `fork()`, the parent and child still map the **same physical pages** for anonymous heap memory. The kernel marks those pages as **Copy-on-Write** and typically read-only internally (even if `smaps` still shows `rw-p` for the mapping).

So at this point:

* the pages are **shared between two processes**
* they are still **anonymous** (not file-backed)
* they are considered **dirty** (because we wrote to them before the fork)
* but **not private** anymore, because both processes reference the same pages

That is why `Private_Dirty` drops to `0 kB` and `Shared_Dirty` becomes `68 kB`.

### Why RSS stays the same but PSS halves

* **RSS** counts the physical pages mapped into the process, even if they are shared.
  So both processes can show the full `68 kB` RSS.

* **PSS** divides shared pages across sharers.
  With exactly two processes sharing, each gets half: `68 kB / 2 = 34 kB`.

This is the cleanest “COW baseline” you can get: shared anonymous pages with zero private dirty memory.

### Note about `Referenced`

Our child shows `Referenced: 0 kB` while the parent shows `Referenced: 68 kB`.

That is normal: the referenced bits depend on whether pages were recently accessed (and on when you sampled `smaps`). Since the child does basically nothing after fork, it may not trigger any access that sets those bits.

Next step is where COW becomes visible: write to `mem` in only one process and watch `Shared_Dirty` shrink while `Private_Dirty` grows in the writing process.

## Step 5: Triggering COW by writing in the child

Now the child writes to the same heap buffer after `fork()`. That single change is enough to force **Copy-on-Write**.

```c
int main(void) {
    uint8_t *mem = malloc(ALLOCATE_N_BYTES);
    if (mem == NULL) {
        perror("malloc");
        exit(EXIT_FAILURE);
    }
    if (!memfill(mem, ALLOCATE_N_BYTES, ALLOCATE_N_BYTES, 1)) {
        return EXIT_FAILURE;
    }

    pid_t pid = fork();
    if (pid < 0) {
        perror("fork");
        exit(EXIT_FAILURE);
    } else if (pid > 0) {
        write(STDOUT_FILENO, "PARENT: ", strlen("PARENT: "));
        write_pid();
        while (1) { }
    } else {
        write(STDOUT_FILENO, "CHILD: ", strlen("CHILD: "));
        write_pid();

        /* This write triggers Copy-on-Write */
        if (!memfill(mem, ALLOCATE_N_BYTES, ALLOCATE_N_BYTES, 2)) {
            return EXIT_FAILURE;
        }

        while (1);
    }
}
```

### What you see in smaps

After the child writes, both processes show:

* `Shared_Dirty: 0 kB`
* `Private_Dirty: 68 kB`
* `Pss: 68 kB`

### What happened

Before the child write, the heap pages were **shared** between parent and child (COW baseline).
When the child executes `memset()`, it writes into every heap page:

* each write to a shared page triggers a COW page fault
* the kernel allocates a **new private page** for the writing process
* the child ends up with its **own copy** of those pages

Because the child touched essentially all resident heap pages, the shared set is gone. The heap is now fully duplicated:

* parent has its private dirty pages
* child has its private dirty pages
* nothing is shared anymore, so PSS no longer halves

### Important detail: why the parent also shows `Private_Dirty`

After COW, the original physical pages remain mapped by the parent only, so they become private to the parent as well. They were already dirty from the initial `memfill(mem, ..., 1)` before the fork, so the parent’s pages naturally show up as `Private_Dirty`.

**Key takeaway:** one-sided writes after `fork()` convert shared anonymous pages into private pages, and `smaps` shows it directly via `Shared_Dirty -> 0` and `Private_Dirty -> RSS`.

## Conclusion

This lab shows what `fork()` and Copy-on-Write actually mean in practice, not in theory.

* `malloc()` reserves virtual address space, not physical memory
* pages become real only when they are written to
* after `fork()`, anonymous pages are shared and accounted via PSS
* a single write is enough to break sharing and trigger full duplication

`/proc/<pid>/smaps` makes this visible:
**RSS shows what is mapped, PSS shows what is owned, and `Private_Dirty` shows where COW actually happened.**

Due to this lab, `fork()` stops being “magic” and becomes just page tables, faults, and accounting.
