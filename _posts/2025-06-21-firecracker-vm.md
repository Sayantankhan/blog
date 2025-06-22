---
layout: post
title:  "🔥 Firecracker: The Lightweight Guard of Serverless Computing"
date:   2025-06-20 10:00:00 +0530
tags:   [vm, serverless computing, aws, lambda, "little's law"]
categories: ["vm"]
---

In the world of virtualization, we're often caught between two extremes:

- 🛡️ **Virtual Machines (VMs)**: Offer strong isolation and security—but come with significant overhead.
- ⚡ **Containers**: Lightweight and fast—with minimal overhead—but weaker isolation guarantees.

Now enter **serverless computing**, which aspires to blend the best of both:
✔️ **High security like VMs**, and  
✔️ **Snappy performance like containers**.

But here's the challenge: Running **thousands of tiny workloads (functions or containers)** per second across **multi-tenant servers**, while keeping them **secure**, **isolated**, and **lightweight**.

---

## ☁️ How AWS (EC2) Tackled It Before

Cloud providers like **AWS EC2** faced similar challenges early on:

- ✅ Solved them using **hypervisor-based virtualization** (e.g., `QEMU/KVM`, `Xen`)
- 🔒 Avoided **multi-tenancy** where possible
- 🖥️ Offered **bare-metal instances**

These options worked, but weren't built for **millisecond-level workloads** that **serverless computing** demands.


---

## 🧱 How Linux Containers Tackle Density

Linux containers use built-in isolation features to efficiently run multiple workloads:

- **cgroups**  
  ➤ Group processes and manage resource usage (CPU, memory, etc.)

- **namespaces**  
  ➤ Isolate Linux kernel resources like PIDs, UIDs, and network interfaces

- **seccomp-bpf**  
  ➤ Restrict syscall access and arguments (critical for security)

- **chroot**  
  ➤ Provide an isolated filesystem view

Together, these tools provide a lightweight method for workload isolation—but still rely on **one shared kernel**, leading to **security vs. compatibility trade-offs**.

This introduces difficult tradeoffs: implementors of serverless and container services can choose
- **Hypervisor-based virtualization**  
    - High isolation, **but comes with high overhead**
- **Linux containers**  
    - Efficient and fast, **but lower security**

---

## ✅ What AWS Needed

To support the scaling demands of **Lambda** and **Fargate**, AWS needed a solution that offered:

- 🔐 **Strong isolation** between functions
- 📈 **High-density** to run thousands of workloads per host
- ⚡ **Fast startup** (<125 ms)
- 🧩 **Compatibility** with unmodified Linux binaries
- 🔄 **Rapid lifecycle** for workloads

These requirements drove AWS to reimagine the virtualization stack. Their answer: **Firecracker, a specialized Virtual Machine Monitor (VMM) built atop KVM, designed for microVMs.**

### What Makes Firecracker Special?
- 💡 Keeps **KVM**, but replaces **QEMU**
- ⚙️ Provides a minimalist **device model** and **API** for managing VMs
- 🐦 Launches **MicroVMs** with:
  - < **5 MB** memory overhead
  - < **125 ms** boot time
  - Up to **150 MicroVMs/sec/host**

This allowed AWS to meet its performance, security, and density goals — without compromising flexibility or isolation.

---
## 🔥 Firecracker - Performance, Job Density, Isolation

Firecracker was purpose-built for **serverless** and **container-based** workloads, focusing on speed, efficiency, and isolation.

Unlike traditional virtual machines, **Firecracker takes a minimalist approach**:

- 🚫 No BIOS  
- 🚫 Cannot boot arbitrary kernels  
- 🚫 No legacy device emulation (USB, PCI, etc.)  
- 🚫 No support for VM migration  
- 🚫 No built-in orchestration or packaging  

Instead, Firecracker focuses solely on **launching and managing lightweight microVMs**, leaving other responsibilities to:

- 🧠 **Higher-level tools** like Kubernetes, Docker, and containerd (for orchestration and metadata management)
- 🧱 **Purposeful omissions** (e.g., no PCI, sound, video, or CPU emulation) because these are **not required** for serverless/container workloads

Lower-level features, such as additional devices (USB, PCI, sound, video, etc), BIOS, and CPU instruction emulation are simply not implemented because they are not needed by typical serverless container and function workloads.

### ⚡ Key Performance Highlights

- 🪶 **Small memory footprint**: ~**5 MiB per microVM**
- 🚀 **Fast boot time**: <**125 ms** to launch user-space
- 🧵 **High throughput**: ~**150 microVMs/sec per host**

### 🧩 Process-per-VM Model
Unlike traditional virtual machines or containers that may run multiple processes inside a single OS instance, **Firecracker adopts a "Process-per-VM" model**:

> 💡 **Each microVM runs exactly one workload (function or container) in complete isolation.**

This model offers a **tighter security boundary** and simplifies resource tracking and billing. Here's why it matters:

- 🛡️ **Security Isolation**  
  Each workload runs in its own microVM with its own kernel. This significantly reduces the blast radius in case of a compromise, especially important in multi-tenant environments.

- ⚖️ **Fine-Grained Resource Accounting**  
  Because each microVM maps 1:1 with a function or container, it's easier to throttle, monitor, and charge based on exact CPU, memory, and I/O usage.

- 🧼 **Clean Startup and Teardown**  
  Since each microVM has its own lifecycle, AWS can quickly start or destroy them without leaking state across workloads.

- 📦 **Stateless Serverless Compatibility**  
  Serverless functions are meant to be short-lived and stateless. The Process-per-VM model maps perfectly to this by providing fresh, isolated environments per invocation (or set of invocations).

This design was fundamental to AWS Lambda and Fargate being able to scale to **thousands of workloads per host**—securely and efficiently.

## 🛤️ Journey of Lambda – λ

In its early days, **AWS Lambda** used:

- 🐧 **Linux containers** to isolate functions from the same customer  
- 🧱 **Virtual Machines (VMs)** to isolate workloads from **different customer accounts**

### 📦 Initial Approach

- Multiple functions **from the same customer** ran inside a single VM
- Functions **from different customers** always ran in **separate VMs**

While this architecture worked, AWS quickly became **unsatisfied** with the model due to:

- ❌ The trade-off between **security and compatibility** that containers impose
- ❌ **Inefficiencies in packing** workloads onto fixed-size VMs
- ❌ Lack of flexibility and granularity in managing per-function resource usage

### 🔄 Requirement for Redesigning the Isolation Model

AWS needed a model that delivered all of the following:

#### 🔐 Isolation  
- Safe for multiple functions to run on **shared hardware**
- Protection against:
  - **Privilege escalation**
  - **Information disclosure**
  - **Covert channels**
  - **Microarchitectural & side-channel attacks**

#### ⚙️ Overhead & Density  
- Ability to run **thousands of functions per machine**  
- Minimal compute/memory waste  
- Support **soft allocation**: over-commit CPU/memory based on usage, not entitlement

#### ⚡ Performance  
- Near-native execution speeds  
- Consistent performance across invocations  
- No interference from noisy neighbors on shared infrastructure

#### 🧩 Compatibility  
- Support for arbitrary **Linux binaries and libraries**  
- No need for code modifications or recompilation

#### 🔄 Fast Switching  
- Must be able to **quickly start new functions**  
- **Efficient cleanup** of completed or idle functions

### 🧠 Observations That Shaped Firecracker

Modern commodity servers have:

- Up to **1 TB of RAM**
- Lambda functions can run with **as little as 128 MB**

To utilize full RAM capacity, AWS needs to support **8,000+ concurrent functions per host**  
(and even more thanks to **soft allocation**).

📊 For example, for a function with **1024 MB memory**, AWS aimed for **<10% overhead**  
→ Meaning **only ~102 MB** RAM consumed for Firecracker-related tasks

This drove the vision for **Firecracker**:  
A **minimal, secure, high-density VM solution** designed to serve serverless at scale.

### Isolation Options and Workloads for Linux

There are three major models for isolating workloads on Linux:
- **Containers**:, in which all workloads share a kernel and some combination of kernel mechanisms are used to isolate them
- **Virtualization**:, in which workloads run in their own VMs under a hypervisor
- **Language VM isolation**:, in which the language VM is responsible for either isolating workloads from each other or from the operating system. 

![Linux Virtualization Model](/assets/img/linux_vm_model.png)

#### 🐳 Containers
In Linux containers:
- Untrusted code calls the host kernel directly, possibly with the kernel surface area restricted (such as with seccomp-bpf).
- It also interacts directly with other services provided by the host kernel, like filesystems and the page cache


#### 💻 Virtualization
In virtualization:
- Untrusted code has full access to a guest kernel, using all kernel features.
- The guest kernel is explicitly treated as untrusted.
- **Hardware virtualization** and the **VMM** limit the guest's access to the host kernel and privileged domain.

### Problems

### 🔒 Container-Based Approach
In environments running untrusted code, container isolation is not only concerned
with preventing privilege escalation, but also in preventing information disclosure side channels and preventing communication between functions over covert channels

### 🔠 Language VM Isolation Approach
Some language VMs (such as V8’s isolates and the JVM’s security managers) aim
to run multiple workloads within a single process, an approach which introduces significant tradeoffs between functionality, compatibility
and resistance to side channel attacks such as Spectre


### 🖥️ Virtualization Approach

#### Challenges
- Two key, and related, challenges of virtualization are density and overhead
The VMM and kernel associated with each guest consumes some amount of CPU and memory before it does useful work, limiting density. 

- Another challenge is startup time, with typical VM startup times in the range of seconds. 

These challenges are particularly important in the Lambda environment, where functions are small (so relative overhead is larger), and workloads are constantly changing.

##### 🚫 Unikernels?
One way to address startup time is to boot something smaller than a full OS kernel, such as a unikernel. Our requirement for running unmodified code targeting Linux meant that we could not apply this approach.

- Another challenge in virtualization is the implementation itself: hypervisors and virtual machine monitors (VMMs), and therefore the required trusted computing base (TCB), can be large and complex, with a significant attack surface

This complexity comes from the fact that VMMs still need to either provide some OS functionality themselves (type 1) or depend on the host operating system (type 2) for functionality. 

In the type 2 model, The VMM depends on the host kernel to provide IO, scheduling, and other low-level functionality, potentially exposing the host kernel and side-channels through shared data structures

The popular combination of Linux Kernel Virtual Machine (KVM) and QEMU clearly illustrates the complexity. QEMU contains > 1.4 million lines of code, and can require up to ~270 unique
syscalls (more than any other package on Ubuntu Linux 15.04). The KVM code in Linux adds another ~120,000 lines.

Efforts have been made to significantly reduce the size of the Hypervisor and VMM, but none of these minimized solutions offer the platform independence, operational characteristics, or maturity that we needed at AWS.


### 🔥 Firecracker’s Solution
Firecracker’s approach to these problems is to use KVM., but replace the VMM
with a minimal implementation written in a safe language. Minimizing the feature set of the VMM helps reduce surface
area, and controls the size of the TCB. Firecracker contains approximately 50k lines of Rust code (96% fewer lines than
QEMU), including multiple levels of automated tests, and auto-generated bindings

Virtualization provides many compelling benefits. From an isolation perspective, the most compelling benefit is that it moves the security-critical interface from the OS boundary to a boundary supported in hardware and comparatively simpler software. 
It removes the need to trade off between kernel features and security: 
- The guest kernel can supply its full feature set with no change to the threat model. 
- VMMs are much smaller than general-purpose OS kernels, exposing a small number of well-understood abstractions without compromising on software compatibility or requiring software to be modified

--- 

## The Firecracker VMM

Firecracker is a Virtual Machine Monitor (VMM), which uses the Linux Kernel’s KVM virtualization infrastructure to provide minimal virtual machines (MicroVMs)
supports Linux and OSv guests

Firecracker provides a REST based configuration API; device emulation for disk, networking and serial console; configurable rate limiting for network and disk 
throughput and request rate. One Firecracker process runs per MicroVM, providing a simple model for security isolation.

Our other philosophy in implementing Firecracker was to rely on components built into Linux rather than re-implementing our own, where the Linux components offer the
right features, performance, and design.

For example we pass block IO through to the Linux kernel, depend on Linux’s process scheduler and memory manager for handling contention
between VMs in CPU and memory, and we use TUN/TAP virtual network interfaces

It has been choose for two reasons
- One was implementation cost: high-quality operating system components, such as schedulers, can take decades to get right, especially when they need to deal with multi-tenant workloads on multi-processor machines
    
- The other reason was operational knowledge: within AWS, operators are highly experienced at operating, automating, optimizing Linux systems

For example, running ps on a Firecracker host will include all the MicroVMs on the host in the process list, and tools like top, vmstat and even kill work as operators expect. 

###  Jailer
- Firecracker’s jailer implements an additional level of protection against unwanted VMM behavior
- The jailer implements a wrapper around Firecracker which places it into a restrictive sandbox before it boots the guest,
including running it in a chroot, isolating it in pid and network namespaces, dropping privileges, and setting a restrictive seccomp-bpf profile
-  The sandbox’s chroot contains only the Firecracker binary, /dev/net/tun, cgroups control files, and
any resources the particular MicroVM needs access to (such as its storage image). The seccomp-bpf profile whitelists 24
syscalls, each with additional argument filtering, and 30 ioctls (of which 22 are required by KVM ioctl-based API).

![](/assets/img/lambda_event.png)

### High-Level Architecture
- Invoke traffic arrives at the frontend via the Invoke REST API, where requests are authenticated and checked
for authorization, and function metadata is loaded.

- The execution of the customer code happens on the Lambda worker fleet, but to
improve cache locality, enable connection re-use and amortize the costs of moving and loading customer code, events
for a single function are sticky-routed to as few workers as possible.
This sticky routing is the job of the Worker Manager, a custom high-volume (millions of requests per second)
low-latency (<10ms 99.9th percentile latency) stateful router

- The Worker Manager replicates sticky routing information for a single function (or small group of functions) between
a small number of hosts across diverse physical infrastructure, to ensure high availability.

- Once the Worker Manager has identified which worker to run the code on, it advises the invoke service which sends the
 payload directly to the worker to reduce round-trips.

    - The Worker Manager and workers also implement a concurrency control protocol which resolves
the race conditions created by large numbers of independent invoke services operating against a shared pool of workers

- Each Lambda worker offers a number of slots, with each slot providing a pre-loaded execution environment for a function.

- Slots are only ever used for a single function, and a single concurrent invocation of that function, but are used for many serial invocations of the function

- Where a slot is available for a function, the Worker Manager can simply perform its lightweight concurrency control
protocol, and tell the frontend that the slot is available for use. 
    - Where no slot is available, either because none exists or because traffic to a function has increased to require additional slots,
      the Worker Manager calls the Placement service to request that a new slot is created for the function. The Placement service in turn optimizes
      the placement of slots for a single function across the worker fleet, ensuring that the utilization of resources including CPU, memory, network, and
      storage is even across the fleet and the potential for correlated resource allocation on each individual worker is minimized.

- Once this optimization is complete — a task which typically takes less than 20ms — the Placement service contacts a
worker to request that it creates a slot for a function. The Placement service uses a time-based lease protocol to
lease the resulting slot to the Worker Manager, allowing it to make autonomous decisions for a fixed period of time.

- The Placement service remains responsible for slots, including limiting their lifetime (in response to the life cycle of
the worker hosts), terminating slots which have become idle or redundant, managing software updates, and other similar
activities.

- Using a lease protocol allows the system to both maintain efficient sticky routing (and hence locality) and have
clear ownership of resources.

### Firecracker In The Lambda Worker
![Lambda Worker](/assets/img/lambda_worker.png)
- Figure 3 shows the architecture of the Lambda worker, where Firecracker provides the critical security boundary required to
run a large number of different workloads on a single server.
- Each worker runs hundreds or thousands of MicroVMs (each providing a single slot), with the number depending on the
configured size of each MicroVM, and how much memory, CPU and other resources each VM consumes.
- Each MicroVM contains a single sandbox for a single customer function, along with a minimized Linux kernel and userland, and a
shim control process
- The shim process in each MicroVM communicates through the MicroVM boundary via a TCP/IP socket with the MicroManager,
 a per-worker process which is responsible for managing the Firecracker processes
- MicroManager provides slot management and locking APIs to placement, and an event invoke API to the Frontend. 
-  Once the Frontend has been allocated a slot by the WorkerManager, it calls the MicroManager with the details of the slot and request payload, which
the MicroManager passes on to the Lambda shim running inside the MicroVM for that slot. 
-  On completion, the MicroManager receives the response payload (or error details in case of a failure), and passes these onto the Frontend for
response to the customer. 
- Communicating into and out of the MicroVM over TCP/IP costs some efficiency, but is consistent
with the design principles behind Firecracker: it keeps the MicroManager process loosely coupled from the MicroVM,
and re-uses a capability of the MicroVM (networking) rather than introducing a new device.
- The MicroManager’s protocol with the Lambda shim is important for security, because it is
the boundary between the multi-tenant Lambda control plane, and the single-tenant (and single-function) MicroVM
- The MicroManager also keeps a small pool of pre-booted MicroVMs, ready to be used when Placement requests a new
slot.
    - While the 125ms boot times offered by Firecracker are fast, they are not fast enough for the scale-up path of Lambda,
which is sometimes blocking user requests. Fast booting is a first-order design requirement for Firecracker, both because
boot time is a proxy for resources consumed during boot, and because fast boots allow Lambda to keep these spare pools
small.
    - The required mean pool size can be calculated with Little’s law : the pool size is the product of creation rate
and creation latency. Alternatively, at 125ms creation time,
one pooled MicroVM is required for every 8 creations per second.


## Little’s Law

Expressed algebraically the law is

> 𝐿=𝜆𝑊.


For a *stable* system, it ties the average request arrival rate, *λ*, the average time each request is processed by the system, *W*, and the number of concurrent requests pending in the system, *L.*

What’s remarkable about this result is that it does not depend on the precise distribution of the requests, the order in which requests are processed or any other variable that might have conceivably affected the result.

**Example -** if 1000 requests, on average, arrive each second, and each takes 0.5 seonds to process, on average, then our system is required to handle 1000*0.5 = 500 requests concurrently.

To handle more requests, **we need to increase *L*, our capacity,** or **decrease *W*, our processing time, or latency.**

What happens if requests arrive at a greater rate than *λ*? The system will no longer be stable. Requests will start queuing up. At first, they will experience much increased latency, but quickly system resources will be exhausted and the server will become unavailable.

### What Dominates the Capacity (*L*)

*L* is a feature of the environment (hardware, OS, etc.) and its limiting factors

- number of concurrent **TCP** connections the server can support.
- **bandwidth** that server can support
- **RAM** - the number of concurrent requests that can fit in RAM depends on how much memory each request consumes, but assuming we can keep this to well below 1MB, and given the low cost of RAM, this number is probably well over 1 million, and certainly over several hundreds-of-thousands.
- **CPU** - how many concurrent requests the CPU can support depends on the application logic, but given that most processing is done by the microservices and that most of the time our requests just wait for the microservices to responsd and don’t waste CPU, this number is anywhere between several hundreds of thousands and several millions.
    - Productions systems employing similar architectures rarely report CPU as their bottleneck in practice.
- So far, we have reason to believe we can keep *L* somewhere between 100K and 1 million. Sounds great, huh? But there is one more limiting factor: **the OS.**
    - A request runs on a single OS thread to completion, and when it’s done, the web server is free to use that thread to serve other requests. So the number of concurrent request we can handle is also limited by the number of threads the OS can handle.
    - So, how many such threads could the OS handle concurrently? That depends on the OS, but it is usually somewhere between 2K and 15K. Beyond that, thread scheduling will add significant latency to the requests, and once latency grows, *W* increases and *λ* drops again.
    ****

## The Role of Multi-Tenancy
- Soft-allocation (the ability for the platform to allocate resources on-demand, rather than at startup) and multi-tenancy
(the ability for the platform to run large numbers of unrelated workloads) are critical to the economics of Lambda.
- Each slot can exist in one of three states: initializing, busy,
and idle, and during their lifetimes slots move from initializing to idle, and then move between idle and busy as invokes flow to the slot 
-  When they are idle they consume memory, keeping the function state available. When they are
initializing and busy, they use memory but also resources like CPU time, caches, network and memory bandwidth and any other resources in the system
- Memory makes up roughly 40% of the capital cost of typical modern server designs, so idle slots should cost 40% of the cost of busy slots.
- Achieving this requires that resources (like CPU) are both soft-allocated and oversubscribed, so can be sold to other slots while one is
idle.
- Oversubscription is fundamentally a statistical bet: the platform must ensure that resources are kept as busy as possible, but some are available to any slot which receives work.
-  We set some compliance goal X (e.g., 99.99%), so that functions are able to get all the resources they need with no contention
X% of the time. Efficiency is then directly proportional to the ratio between the Xth percentile of resource use, and mean
resource use. Intuitively, the mean represents revenue, and the Xth percentile represents cost. Multi-tenancy is a powerful
tool for reducing this ratio, which naturally drops approximately with √N when running N uncorrelated workloads on
a worker.

Boot times of MicroVMs are important for serverless workloads like Lambda. While Lambda uses a small local pool
of MicroVMs to hide boot latency from customers, the costs of switching between workloads (and therefore the cost of
creating new MicroVMs) is very important to our economics.



## 📎 References:

- [Firecracker : Lightweight Virtualization
for Serverless Applications](https://www.usenix.org/system/files/nsdi20-paper-agache.pdf)
- [Firecreacker-MicroVm](https://github.com/firecracker-microvm/firecracker)
- [Firecracker-Youtube](https://www.youtube.com/watch?time_continue=2&v=cwruf1ERAKM&embeds_referring_euri=https%3A%2F%2Fwww.usenix.org%2F&source_ve_path=Mjg2NjY)
