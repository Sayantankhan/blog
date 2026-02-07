---
layout: post
title: "Understanding Page Tables, Virtual Memory, MMU, TLB, and IOMMU"
date: 2025-06-30
tags: ["os", "virtual-memory", "mmu", "iommu", "linux"]
---
Modern operating systems give every process the illusion that it owns a large, private, contiguous memory space. In reality, physical memory (RAM) is shared, fragmented, and limited. The magic that makes this illusion possible is virtual memory, backed by page tables, enforced by the MMU, accelerated by the TLB, and extended to devices using IOMMU.

Let’s walk through this step by step, from the CPU’s point of view.

--- 
## Virtual Memory: Pages and Frames

A process never sees physical memory directly. Instead, it operates in a virtual address space, which is divided into fixed-size blocks called pages. Physical memory, on the other hand, is divided into blocks of the same size called frames.
So the core idea is simple:

> Virtual pages are mapped to physical frames

This mapping is not sequential, not guaranteed to be contiguous, and not even guaranteed to exist at all times.

### Page Tables: The Translation Map
To translate virtual pages into physical frames, the OS uses a data structure called the page table. Every process has its own page table.

A typical Page Table Entry (PTE) contains:
```mathematica
Virtual Page → { Frame Number, Protection Bits, Status Bits }
```

#### But number of entry in page table could be huge
Let’s do some quick math.
- Virtual address space: 8 GB
- Page size: 4 KB
- Total pages = 8GB / 4KB ≈ 2 million pages

That means:
- ~2 million page table entries per process
- If each PTE is 4 bytes → ~8 MB per process
- Multiply that by 50 processes → 400 MB just for page tables

That’s unacceptable. To solve this, operating systems use:
- Multi-level page tables (most common)
- Inverted page tables
- Hashed page tables

Multi-level page tables allocate memory only for the portions of the address space actually used, drastically reducing RAM overhead.![alt text](https://upload.wikimedia.org/wikipedia/commons/3/32/Virtual_address_space_and_physical_address_space_relationship.svg)![alt text](https://i.sstatic.net/Iz7Ti.png)

### Context Switches and PTBR

When the CPU switches from one process to another (a context switch), it must also switch address spaces.

This is done using a special CPU register:
**PTBR** – Page Table Base Register

PTBR points to the root of the current process’s page table.
Changing PTBR effectively changes the entire virtual memory view of the CPU.

This register is privileged, meaning user programs can’t touch it.

---

### Inside a Page Table Entry
Each page table entry contains much more than just an address.

```mathematica
Virtual Page → { Frame Number, Protection Bits, Status Bits }
```

#### Protection / Permission Bits
A Page Table Entry (PTE) doesn’t just say where a page lives — it says who can touch it, how, and in what mode.
At a high level, permission bits answer these questions:
- Can this page be read?
- Can it be written?
- Can it be executed?
- Can user mode access it?
- Is it kernel-only?
- Can it be cached?
- Can it be shared safely?

These bits are enforced in hardware by the MMU, not by the OS after the fact.
| Permission | Meaning                   |
| ---------- | ------------------------- |
| **R**      | Read allowed              |
| **W**      | Write allowed             |
| **X**      | Execute allowed           |
| **U/S**    | User vs Supervisor access |

##### x86-64 Page Table Permission Bits
A typical x86-64 PTE contains (simplified):
| Bit    | Name                     | Meaning                            |
| ------ | ------------------------ | ---------------------------------- |
| Bit 0  | **P (Present)**          | Page exists in RAM                 |
| Bit 1  | **RW (Read/Write)**      | 0 = Read-only, 1 = Writable        |
| Bit 2  | **US (User/Supervisor)** | 0 = Kernel only, 1 = User allowed  |
| Bit 3  | **PWT**                  | Page Write Through (cache control) |
| Bit 4  | **PCD**                  | Page Cache Disable                 |
| Bit 5  | **A (Accessed)**         | Page was accessed                  |
| Bit 6  | **D (Dirty)**            | Page was written                   |
| Bit 7  | **PAT**                  | Page Attribute Table               |
| Bit 8  | **G (Global)**           | Not flushed on context switch      |
| Bit 63 | **NX (No Execute)**      | Instruction fetch forbidden        |

You can observe real permission mappings using:
```bash
cat /proc/self/maps
```

Typical permissions look like:
- r-x → code
- rw- → heap / stack
- r-- → constants

These map directly to page table permission bits.

#### Status Bits (Used by Hardware)
- Present / Valid bit – is the page in RAM?
- Dirty bit – has the page been modified?
- Accessed bit – recently read or written
- Global bit – shared across processes (e.g., kernel pages)

These bits are critical for paging, caching, and page replacement algorithms.

---

### The Cost of Address Translation

If the CPU had to access main memory just to translate a virtual address, every single load or store would require *two* memory accesses: one to read the page table entry and another to access the actual data. In other words, address translation itself would double the cost of every memory operation.

Yes — without any assistance, this would be painfully slow and completely unacceptable for modern systems. This fundamental problem is what led to the introduction of dedicated hardware for memory translation. This is where the **MMU** enters the picture.

### MMU: The Gatekeeper of Memory

The Memory Management Unit (MMU) is a hardware component responsible for translating virtual addresses generated by the CPU into physical addresses used to access main memory. Beyond simple translation, the MMU enforces access permissions, prevents processes from touching memory they do not own, and raises hardware exceptions when violations occur.

Conceptually, the MMU sits between the CPU and the memory subsystem. Every memory access generated by the processor flows through it. Depending on the microarchitecture, address translation may occur either before or after cache lookup, leading to different cache designs.

In a **physically indexed cache**, the virtual address is first translated by the MMU and the resulting physical address is then used to access the cache:

> CPU → MMU → Cache → Memory

In a **virtually indexed cache**, cache lookup happens using the virtual address, and translation occurs afterward:

> CPU → Cache → MMU → Memory

Modern CPUs typically use a hybrid design that combines aspects of both approaches in order to balance performance with correctness and avoid aliasing problems.

![alt text](https://homepage.cs.uiowa.edu/~jones/assem/notes/14f/chip.gif)

### TLB: The MMU’s Cache

To avoid repeated and expensive page table walks, the MMU relies on a small, ultra-fast cache known as the **Translation Lookaside Buffer (TLB)**. The TLB stores recently used page table entries so that virtual-to-physical address translation can be completed without accessing main memory in the common case.

The TLB is fully hardware-managed, extremely fast, and typically implemented as a fully associative structure to minimize translation conflicts. Because it is much smaller than the page table itself, it only retains translations that are likely to be reused in the near future.

Most modern CPUs separate the TLB into two logical components: an **Instruction TLB (iTLB)** used for instruction fetches, and a **Data TLB (dTLB)** used for load and store operations. To further improve hit rates, contemporary systems often implement multiple levels of TLBs, similar to multi-level cache hierarchies.

### ASID: Multiple Processes, One TLB

Without additional support, a context switch between processes would require flushing the entire TLB, since the virtual address space changes. Frequent TLB flushes would severely degrade performance.

To avoid this, modern architectures associate each TLB entry with an **ASID (Address Space Identifier)**. The ASID uniquely identifies the address space to which a translation belongs.

By tagging TLB entries with ASIDs, the hardware allows translations from multiple processes to coexist in the TLB simultaneously. This significantly reduces the need for TLB flushes during context switches and results in faster process switching and better overall system performance.


---

## How Address Translation Actually Happens
When a program executes an instruction that touches memory, the CPU does not generate a physical address. It generates a virtual address. The responsibility of converting this virtual address into a physical one belongs entirely to the MMU.

Assume the system uses a 4 KB page size. This immediately defines the structure of every virtual address. Since 4 KB equals $2^{12}$
 bytes, the lower 12 bits of the address represent the page offset — the exact byte location within a page. These bits are carried unchanged throughout translation. The remaining higher-order bits form the Virtual Page Number (VPN), which uniquely identifies the virtual page within the process’s address space.

The MMU receives the virtual address and splits it into these two components: the VPN and the page offset.

Before touching main memory, the MMU first consults the Translation Lookaside Buffer (TLB). The TLB is a small, hardware-managed cache that stores recently used page table entries from multiple processes. Each entry is tagged with an ASID (Address Space Identifier) so that translations from different processes can coexist without being flushed on every context switch.

If the VPN is found in the TLB and the entry is marked valid, the MMU extracts the corresponding physical frame number, replaces the VPN portion of the address with this frame number, preserves the offset bits, and forms the complete physical address. The memory access then proceeds immediately. This is the fast path, and it is what happens for the overwhelming majority of memory accesses in a healthy system.

If the VPN is not found in the TLB, a TLB miss occurs. In this case, the MMU performs a page table walk by reading the page table structures stored in main memory. Depending on the architecture, this may involve multiple levels of indirection. Once the correct page table entry is located, the MMU checks its validity and permission bits. If the entry is valid, the physical frame number is obtained, the translation is inserted into the TLB for future accesses, and the memory operation resumes using the newly constructed physical address.

If the page table entry indicates that the page is not present, the MMU cannot complete the translation and instead raises a page fault exception, transferring control to the operating system.

The OS first determines whether the faulting access is legal. If the process attempted to access memory outside its allocated address space or violated permission constraints, the OS terminates the process, typically with a segmentation fault. If the access is valid, the OS proceeds to resolve the fault by selecting a free physical frame, loading the required page from disk into memory, updating the page table with the new frame mapping, and inserting the translation into the TLB. Execution then resumes from the faulting instruction as if the interruption never occurred.

This disk-backed page fault path is orders of magnitude slower than a TLB hit or even a page table walk, which is why page faults are disastrous for latency-sensitive workloads.

Because the TLB is small, it cannot hold translations for all active pages. When it becomes full, the TLB controller must evict an existing entry to make room for a new one. The eviction decision follows a hardware-defined TLB replacement policy, commonly LRU, round-robin, or random replacement. The exact policy is architecture-specific, but the goal is always to retain translations that are likely to be reused in the near future.

This translation process — TLB lookup, page table walk, possible page fault, and replacement — occurs continuously, millions of times per second, entirely in hardware unless an exception forces the operating system to intervene.

----

### Linux: Observing Memory Translation
``` bash
# Page size
getconf PAGE_SIZE

# Virtual memory statistics
vmstat 1

# Page faults
perf stat -e page-faults ./your_program

# Process memory map
cat /proc/<pid>/maps

# Page table usage (root only)
cat /proc/meminfo | grep PageTables

```
---

## IOMMU: MMU for Devices

So far, address translation has been entirely CPU-centric. The CPU generates virtual addresses, the MMU translates them, and memory access proceeds under strict protection rules. This model breaks down the moment devices enter the picture.

Modern systems rely heavily on **DMA (Direct Memory Access)**. Network cards, GPUs, NVMe controllers, and RDMA-capable devices all perform memory accesses without CPU involvement. During DMA, a device issues memory reads and writes directly to RAM using addresses supplied by the operating system.

The problem is simple: devices do not understand CPU virtual memory. They operate on raw addresses. Without additional hardware support, a device performing DMA could read or overwrite arbitrary physical memory, intentionally or accidentally.

This is where the **IOMMU** (Intel VT-d / AMD-Vi) comes in.

An IOMMU performs address translation and access control for devices in the same way that an MMU does for CPUs. Instead of translating CPU virtual addresses, the IOMMU translates **device-visible virtual addresses** into physical addresses. Every DMA request issued by a device passes through the IOMMU before reaching memory.

At a high level, the IOMMU provides per-device isolation, enforces DMA permissions, and prevents devices from corrupting memory outside of the regions explicitly assigned to them. It is also a foundational requirement for secure virtualization and modern high-performance I/O.

Conceptually, it is best thought of as an MMU placed on the I/O path rather than the CPU path.

### DMA and What It Actually Needs

For DMA to work correctly and safely, three conditions must be met.

First, the memory used for DMA must remain stable for the duration of the operation. The device cannot tolerate the OS paging the memory out or moving it. This is why DMA memory is traditionally pinned — pinned pages cannot be swapped or remapped while the device is using them.

Second, the device must be restricted to accessing only the memory it has been explicitly granted. Without isolation, a buggy driver or compromised device could overwrite kernel memory or data belonging to other processes.

Third, DMA must be efficient. Excessive copying or bounce buffers defeat the purpose of DMA, especially for high-throughput devices such as NICs, GPUs, and NVMe controllers.

Without an IOMMU, the kernel must satisfy these requirements manually by pinning physical pages, exposing raw physical addresses to devices, and trusting that the device will behave correctly. This approach does not scale and is fundamentally unsafe.

### Why IOMMU Matters

Without an IOMMU, devices can issue DMA requests to any physical address. This creates a serious security risk, as a malicious or malfunctioning device can corrupt arbitrary memory. To mitigate this, the kernel is forced to permanently pin DMA buffers and carefully manage physical memory exposure, increasing memory pressure and reducing flexibility.

With an IOMMU enabled, DMA becomes both safe and scalable. The OS can present devices with a virtualized I/O address space, map only the required memory regions, and rely on hardware to enforce isolation. Devices are confined to their assigned mappings, and DMA faults are detected and handled by the system.

This enables safe DMA without exposing physical memory, strong isolation between devices, zero-copy I/O without permanent pinning, efficient RDMA and high-speed networking, and secure virtualization.

### How Modern Virtual Machines Use IOMMU

In virtualized environments, the role of the IOMMU becomes even more critical. A hypervisor must isolate not only processes, but entire guest operating systems — each with its own page tables, kernel, and drivers.

When a device is assigned to a virtual machine, either directly (device passthrough) or via virtual functions, the device must not be allowed to DMA into the host kernel or into the memory of other guests. The IOMMU makes this possible by introducing an additional layer of address translation on the I/O path.

From the device’s point of view, it issues DMA requests using addresses provided by the guest OS. These addresses are **guest-physical addresses**, not host-physical addresses. The IOMMU translates these guest-physical addresses into host-physical memory, using mappings installed by the hypervisor.

This creates a three-level translation pipeline:
- Guest virtual address → guest physical address (guest MMU)
- Guest physical address → host physical address (IOMMU)
- Host physical address → DRAM

Because the IOMMU enforces these mappings in hardware, the hypervisor does not need to intercept or emulate every DMA operation. Devices can perform DMA at full speed while remaining fully isolated.

This is the foundation for:
- PCI passthrough
- VFIO-based device assignment
- SR-IOV virtual functions
- GPU passthrough
- High-performance networking in VMs
- Near-native I/O performance in guests

Without IOMMU support, virtual machines must rely on paravirtualized drivers and software bounce buffers, which significantly limit performance and increase complexity.

In modern hypervisors, IOMMU is not optional. It is a hard requirement for secure device assignment, scalable virtualization, and safe high-performance I/O.

### Observing IOMMU in Linux

You can verify whether IOMMU support is enabled using:

```bash
dmesg | grep -i iommu
```

### Conclusion

Modern memory management is not a single mechanism but a carefully layered contract between hardware and software. Virtual memory gives processes the illusion of private, contiguous address spaces, while page tables and permission bits encode ownership, access rights, and intent. The MMU enforces these rules in hardware, and the TLB exists purely to make this enforcement fast enough to be practical.

As systems evolved, memory access stopped being a CPU-only concern. Devices began to move data directly using DMA, bypassing the processor entirely. Without additional controls, this broke the isolation guarantees that virtual memory relies on. The IOMMU restores those guarantees by extending address translation and protection into the I/O path, ensuring that devices are subject to the same discipline as CPUs.

In virtualized environments, these mechanisms become non-negotiable. Guest operating systems assume full control over memory and devices, while the host must retain ultimate authority. MMUs translate guest virtual memory, and IOMMUs translate device-visible addresses, together forming a complete isolation boundary that allows near-native performance without sacrificing safety.

The key takeaway is that performance, security, and scalability are not competing goals here — they are co-designed. TLBs exist because page tables are expensive. IOMMUs exist because DMA is dangerous without translation. Pinning, zero-copy I/O, RDMA, SR-IOV, and device passthrough are not special cases; they are natural outcomes of this architecture.

Understanding these layers as a single system — rather than isolated features — is essential for building databases, hypervisors, networking stacks, or any performance-critical infrastructure. Once you see the full pipeline, the trade-offs stop being mysterious and start becoming deliberate engineering choices.
