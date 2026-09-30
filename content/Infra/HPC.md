---
craft_id: 05CD040B-D59E-4D8D-9569-D99604DD67B2
craft_source: "craftdocs://open?spaceId=f0e27734-d8b8-47ce-be9d-9b35def0cb70&blockId=05CD040B-D59E-4D8D-9569-D99604DD67B2"
imported: 2026-09-27
---

# HPC

## SIMD, vector registers, and precision

SIMD means **Single Instruction, Multiple Data**: one instruction applies an operation to multiple data elements. On x86, examples include 256-bit AVX and 512-bit AVX-512 vectors; widths and supported operations depend on the architecture.

| Vector width | FP32 elements (32 bits each) | FP64 elements (64 bits each) |
|---|---:|---:|
| 256 bits | 8 | 4 |
| 512 bits | 16 | 8 |

Use **FP32/FP64**, not PF32/PF64. At a fixed register width, FP32 packs twice as many elements. It also halves raw array storage compared with FP64, so a fixed-size cache can hold twice as many elements, ignoring other data and overhead.

Sixteen vector lanes do not necessarily mean sixteen separate physical ALUs complete everything in one cycle. A processor can split a wide instruction across narrower execution units or multiple cycles. See [Intel AVX-512 overview](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-avx-512-instructions.html).

### Numerical tradeoffs

FP32 has less precision and a smaller finite range than FP64. Adding a very small value to a much larger value can lose the small contribution through rounding. Overflow, underflow, and cancellation are distinct problems; FP64 can experience them too. Parallel reductions may change the order of addition and therefore the rounded result.

For IEEE binary32, this example demonstrates lost precision at `2**24` without requiring NumPy:

```python
import struct

def fp32(value):
    return struct.unpack("f", struct.pack("f", value))[0]

large = fp32(2**24)
assert fp32(large + 1.0) == large
assert fp32(large + 2.0) == large + 2.0
```

Choose precision based on error tolerance and numerical stability, not performance alone.

## FLOPs, FLOP/s, and FMA

A **FLOP** is a floating-point operation; **FLOP/s** is a rate. Fused multiply-add computes `a * b + c` with one final rounding and conventionally counts as **two FLOPs per active element**.

A single FMA instruction is not necessarily a one-cycle-latency operation. **Latency** measures how long until a result can be used; **throughput** measures how frequently independent instructions can execute. Multiple independent accumulators can help use pipelined FMA units. Consult the specific processor's throughput and latency data. See [Intel optimization reference](https://cdrdv2-public.intel.com/814199/356477-Optimization-Reference-Manual-V2-002.pdf).

### Example: theoretical CPU peak

Assume all of the following:

- 512-bit vectors operating on FP32: 16 elements per instruction.
- Two full-width vector FMA instructions per core per cycle.
- 32 physical cores operating at a sustained 3.0 GHz for this workload.

Then:

$$
P = 16\ \frac{\text{elements}}{\text{instruction}}
\times 2\ \frac{\text{FLOPs}}{\text{element}}
\times 2\ \frac{\text{instructions}}{\text{core}\cdot\text{cycle}}
\times 32\ \text{cores}
\times 3.0\times10^9\ \frac{\text{cycles}}{\text{s}}
= 6.144\times10^{12}\ \text{FLOP/s}.
$$

This is **6.144 TFLOP/s FP32**, or **3.072 TFLOP/s FP64** under the same full-width instruction-throughput assumptions. The original arithmetic was approximately correct; its interpretation needed these conditions.

This is a theoretical arithmetic ceiling, not measured application performance. Memory bandwidth, instruction mix, dependencies, vectorization, and power limits can reduce achieved throughput. SIMD-heavy clock frequency may differ from the advertised clock. SMT threads share a core's execution resources and should not be counted as additional physical cores in this calculation.

## Cluster and processor topology

| Term | Meaning |
|---|---|
| Cluster | Networked nodes managed together; hardware need not be homogeneous or geographically colocated |
| Compute node | A resource that runs work, commonly one server/OS instance; exact boundaries depend on the platform |
| Motherboard | Connects sockets, memory, and I/O; multi-node chassis need not have one board per logical node |
| Socket | Physical connection for a processor package; a package may contain multiple dies/chiplets |
| CPU package | Physical processor containing cores, caches, and usually memory controllers |
| Core | Execution core within a CPU package |
| Logical CPU / hardware thread | OS scheduling context; several may share a physical core with SMT |

Intel's Hyper-Threading is one SMT implementation, often exposing two logical CPUs per core. SMT is not universal and does not double all compute resources. Modern mainstream CPUs generally have integrated memory controllers; the external northbridge memory controller is a historical design, not a universal current arrangement.

## UMA and NUMA

### UMA: Uniform Memory Access

Memory access characteristics are approximately uniform across processors accessing the shared memory. UMA does **not** mean “Unified Memory Access,” nor is it defined by having one memory controller or one OS. Historical symmetric multiprocessors could be UMA; a modern single-socket system is not automatically UMA.

### NUMA: Non-Uniform Memory Access

Memory is grouped into locality domains. Access latency and available bandwidth depend on the relationship between the executing CPU and the memory's location. A socket may contain multiple NUMA nodes; some nodes can be memory-only. See [Linux NUMA documentation](https://docs.kernel.org/mm/numa.html).

In a typical cache-coherent multi-socket server, each socket has nearby memory controllers and DRAM. A remote access travels through the interconnect to the relevant memory/coherence machinery. It does not require a software task on the other CPU to read and forward the data. A coherent cache may also supply the requested line, rather than DRAM.

Cache coherence manages ownership and visibility of cache lines across cores. A write can invalidate other cached copies; it does not necessarily send an immediate updated value to every core. Coherence is also not a substitute for synchronization or the language's memory model.

### Practical consequences

- Keep threads and the memory they use close when possible; locality depends on allocation policy and access patterns.
- On Linux, first-touch placement is common under default policy, but explicit policies, migration, and memory pressure can change placement. See [Linux NUMA memory policy](https://docs.kernel.org/admin-guide/mm/numa_memory_policy.html).
- Pinning threads without considering memory placement can still leave most accesses remote.

Inspect the actual Linux topology rather than assuming “one socket = one NUMA node”:

```bash
lscpu
lscpu -e=CPU,CORE,SOCKET,NODE
numactl --hardware  # Requires numactl
numastat           # Requires numactl; system-level NUMA statistics
```

These are inspection commands; they do not change CPU affinity or memory policy. See [`numactl`](https://man7.org/linux/man-pages/man8/numactl.8.html).

## OpenMP and MPI

| Model | Common use | Qualification |
|---|---|---|
| OpenMP | Threads cooperating over shared memory within a node | Also supports tasks and accelerator offload; it is not limited to simple CPU loops |
| MPI | Communicating processes across nodes | Also works within one node and includes shared-memory and one-sided communication facilities |
| Hybrid MPI + OpenMP | MPI between groups of processes; threads within each process | Requires attention to rank/thread placement and MPI thread-support level |

“OpenMP = shared memory; MPI = distributed memory” is a useful starting point, not an exclusive hardware restriction. See [OpenMP scope](https://www.openmp.org/spec-html/5.2/openmpse1.html) and [MPI background](https://www.mpi-forum.org/docs/mpi-4.1/mpi41-report/node13.htm).

## Storage hardware: HDD, SSD, and NVMe

**HDD/SSD describe storage technology; SATA/SAS/NVMe describe interfaces or protocols.** SSDs can use SATA, SAS, or NVMe. NVMe commonly uses PCIe locally and also supports network transports through NVMe over Fabrics.

NVMe uses queues and can reduce command-processing overhead compared with older storage protocols. Controllers commonly transfer data with DMA, but this does not turn an SSD into directly load/store-addressable RAM, eliminate drivers, or remove all CPU work. See [NVM Express specifications](https://nvmexpress.org/specifications/) and [NVMe protocol overhead](https://nvmexpress.org/faq-items/why-does-nvme-technology-have-lower-latency-than-sata-or-sas/).

HDDs can offer economical capacity, while SSDs typically provide much lower random-access latency. Actual performance depends on workload, queue depth, device/controller limits, and the storage network.

## Distributed and parallel filesystems

### Lustre

The correct name is **Lustre**, not “Luster.” Lustre presents a POSIX filesystem interface to clients and separates metadata services/targets (**MDS/MDT**) from object storage services/targets (**OSS/OST**) for file contents. It also has management services. Some features can place small-file data on metadata targets, so the separation is not an absolute rule for every byte.

Fast metadata storage is common, but “metadata is always on SSD” is not a defining property. See [Lustre architecture](https://wiki.lustre.org/Lustre_Architecture_for_Admins).

### BeeGFS

BeeGFS exposes a filesystem through its client and separates metadata and storage services, with a management service coordinating the system. Metadata and storage roles can be placed according to deployment needs. Whether it is “easy” depends on scale, failure handling, and operational requirements; it is not an architectural guarantee. See [BeeGFS architecture](https://doc.beegfs.io/latest/architecture/overview.html).

### Ceph and CephFS

Ceph's distributed storage foundation is **RADOS**; see [Ceph architecture](https://docs.ceph.com/en/latest/architecture/). It supports object, block, and filesystem interfaces; **CephFS** is its filesystem interface. CephFS metadata servers (**MDS**) coordinate filesystem metadata, while persistent metadata and file data live in RADOS pools backed by OSDs.

CRUSH rules determine pool data placement over OSDs/failure domains; they do not inherently map all metadata to SSD. A metadata pool can be assigned an SSD/NVMe placement rule. Ceph recommends fast storage for that pool. See [creating a CephFS filesystem](https://docs.ceph.com/en/latest/cephfs/createfs/) and [CRUSH maps](https://docs.ceph.com/en/pacific/rados/operations/crush-map/).

The original claim that Ceph is “most used for home directories in HPC centers” is not established by these notes. Home, scratch, and project storage choices are site-specific. Likewise, labels such as “Lustre is complex” and “BeeGFS is easy” need a particular deployment context to be meaningful.

## Review scope

Reviewed on 2026-09-30. This note describes general architecture and an illustrative peak-performance calculation; it does not assert the topology, sustained FLOP/s, or filesystem configuration of a particular HPC center. Original Craft provenance remains in the frontmatter.
