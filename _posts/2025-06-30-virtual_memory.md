---
layout: post
title: "Understanding Virtual Memory, Page Tables, and Cache Hierarchy"
date: 2025-06-30
tags: ["virtual memory", "page table"]
---

Modern operating systems and processors handle memory through a complex, elegant dance of abstraction and hardware support. Whether you're a systems developer or just curious about how your computer makes applications run smoothly, understanding **virtual memory**, **page tables**, **L1/L2 cache**, and **RAM** is foundational. Let's break it down.

---

## 🏡 What is Virtual Memory?

**Virtual memory** is an abstraction layer that gives processes the illusion they have their own continuous and isolated memory space. Instead of accessing physical RAM directly, processes operate within this virtual space. The OS and MMU (Memory Management Unit) map virtual addresses to physical addresses behind the scenes.

### Benefits:

- **Isolation**: Each process can't touch another's memory.
- **Efficiency**: Allows overcommitting memory.
- **Flexibility**: Enables swapping and memory-mapped files.

---

## How Does It Work?

Virtual memory is divided into chunks called **pages** (usually 4KB each). To translate virtual addresses to physical addresses, the OS uses a structure called the **page table**.

Each **page table entry (PTE)** holds:

- The physical frame number (PFN)
- Flags (e.g., valid, read/write, dirty, accessed)

### Address Translation (Simplified):

```
Virtual Address = [Page Number | Offset]
-> Page Number indexes into Page Table
-> Entry gives Physical Frame
-> Final Address = Frame + Offset
```

For large address spaces, we use **multi-level page tables** (2-level or 4-level on x86_64) to reduce memory overhead.

### Why Not 1:1 Mapping?

We avoid a **1:1 mapping** from every virtual page to a physical page because this would consume an enormous amount of memory, especially for large address spaces. Instead, we map **virtual memory segments** to **physical segments**. This way, we use offsets and calculate the final physical address, saving memory and improving efficiency.

---

## 🔎 Real Examples Using `/proc`

Use the following commands to inspect memory mappings of a running process (replace `<pid>`):

```bash
cat /proc/<pid>/maps
```

Example output:

```
57bb281e5000-57bb281e8000 r--p 00000000 08:02 20203174 /usr/sbin/cron
```

### Field Breakdown:

- `57bb281e5000-57bb281e8000`: Virtual address range
- `r--p`: Permissions (read-only, private)
  - r = readable
  - w = writable
  - x = executable
  - s = shared, p = private (copy-on-write)
- `00000000`: Offset in the file that is mapped
- `08:02`: Device identifier (major:minor)
- `20203174`: Inode on the device
- `/usr/sbin/cron`: The file backing the mapping (if any)

To map virtual addresses to physical ones:

```bash
sudo cat /proc/<pid>/pagemap
```

Each entry in `/proc/<pid>/pagemap` represents a page. It contains:
- Bits 0–54: Page Frame Number (PFN)
- Bit 63: Page present

To decode:

```bash
sudo dd if=/proc/<pid>/pagemap bs=8 skip=<page_index> count=1 | hexdump
```

You can also inspect memory usage using:

```bash
sudo cat /proc/<pid>/smaps
```

This includes memory stats like RSS, PSS, clean/dirty shared/private regions, etc.

---

### 🕵️️ Example: Mapping `/usr/sbin/cron`

From `/proc/<pid>/maps`:

```
57bb281e5000-57bb281e8000 r--p 00000000 08:02 20203174 /usr/sbin/cron
```

From `/proc/<pid>/smaps`:

```
Size:                 12 kB
Rss:                  12 kB
Private_Clean:        12 kB
Referenced:           12 kB
```

This region is:

- Read-only
- 12KB (3 pages of 4KB)
- Fully loaded into memory

To get the PFN and final physical address:

```bash
VPN = 0x57bb281e5000 / 4096
Physical Address = PFN * 4096 + Offset
```

---

## 🐿 Page Table Access is Slow — We Need Caches

Looking up a virtual address through the page table can take **10+ CPU cycles**, especially with multi-level tables. Modern CPUs speed this up using:

- **TLB (Translation Lookaside Buffer)**: Caches address translations.
- **L1/L2/L3 Caches**: Store actual memory data close to the CPU.

---

## ⚡️ TLB: Translation Lookaside Buffer

The **TLB** is a fast cache for virtual-to-physical address translations.

- A **TLB hit** is fast (1 cycle).
- A **TLB miss** requires a page table walk.

Modern CPUs have:

- Separate TLBs for instructions/data.
- Multi-level TLBs (like L1/L2 caches).

### Why We Still Need L1/L2/L3:

- **TLB** maps addresses.
- **L1/L2/L3** cache the actual data.

Analogy:

- **TLB** = Finds the location of the book.
- **Caches** = Bring the book close to you.

---

## 📊 Cache Hierarchy

| Level | Size     | Latency | Description        |
|-------|----------|---------|--------------------|
| L1    | 32-64KB  | ~1 cycle| Per-core, fastest  |
| L2    | ~256KB   | ~10 cyc | Per-core, slower   |
| L3    | 2-32MB   | ~30-50c | Shared across cores|

---

## 📀 RAM: Main Memory

RAM is where active process data lives. It's slow compared to caches (~100ns), so the system tries to avoid fetching from RAM unless absolutely needed.

---

## 🔧 perf Output Sample

```
 1.14%  node                  [.] 0x0000000000cb8d7e
 1.01%  perf                  [.] io__get_char
 0.81%  [kernel]              [k] __irqentry_text_end
...
```

You can use this to identify hot paths, syscall pressure, or instruction stalls.

---

## ⚖️ Summary

- **Virtual Memory**: Isolates processes & expands address space.
- **Page Tables**: Map virtual → physical addresses.
- **TLB**: Avoids walking the page table repeatedly.
- **Caches**: L1/L2/L3 store data close to CPU.
- **RAM**: Main memory.
