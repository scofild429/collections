---
craft_id: 05CD040B-D59E-4D8D-9569-D99604DD67B2
craft_source: "craftdocs://open?spaceId=f0e27734-d8b8-47ce-be9d-9b35def0cb70&blockId=05CD040B-D59E-4D8D-9569-D99604DD67B2"
imported: 2026-09-27
---

# HPC

### wide vector register

> - 256 or 512 bits
> - for SIMD (Single Instruction, Multiple Data) operations.
> - Use PF32, one vector register can pack twice number of values than PF64
  > -  also for the cache.
  > - inaccurate accumulation between big and small number 
  > - overflow because of the limited numerical range

### FLOPs

- Fused Multiply-Add: multiplying two numbers and adding the result to an accumulator
- Modern CPUs fuse this into a single instruction taking a single clock cycle
- counts as two floating-point operations (FLOPs)
- example:
  - 512 bits register for FP32: 16 numbers
  - FMA for 2 float operation:  32 times
  - two FMA units per core: 64 times
  - 32-core CPU :  2048 times
  - running at 3.0 GHz:  6.14 * 10^12 →  6.14 TFlops

### SIMD

- 512 bits register for FP32, we have 16 number
- corresponding to 16 math circuits(ALU: Arithmetic logic units) for 16 operation at the same time

#### SSD vs. HDD

- Now SSD use NVMe protocol, direct access memory through PCIe, require few CPUs instruction for fast velocity.

*****

#### Luster

- complex
- default POSIX file system, separate metadata daemon (on SSD) and storage daemon

#### BeeGFS

- easy
- default POSIX file system, separate metadata daemon (on SSD) and storage daemon

#### Ceph

- default object store, 
- Ceph File System (CephFS) use its Metadata server with CRUSH map rules
  - CRUSH map Metedata to SSD 
- Most used in the HOME directory in the HPC center



#### OpenMP vs. MPI

- *OpenMP →  shared memory system*

-  MPI →  distributed system*

*****

# Architecture for NUMA and UMA

  - Cluster

      collect multiple nodes, Geographical Homogeneity

  - Node

      single operating system (OS) kernel instance 

      shares a unified, cache-coherent physical memory address space (NUMA architecture)

  - Motherboard

      normally only one for a node

      the standard way to access RAM though bus (Northbridge) 

  - Socket

      the slot in motherboard for installing CPU, can be scaled up,  to 2, 4...

  - CPU

      Physical unit hardware 

  - Core

      processor unit

      be ware of hpyer-thread, one physical processor for 2 logical processors 

******

## UMA: 

- Unified memory access, a memory controller for a OS
- Historically also applied to multiple socket system, dispatched because of the access bottleneck of Bus in motherboard. 
- Current, more only used in the single socket (CPU) consumer product 

## NUMA:

- For a node with multiple socket CPU 
- Each CPU has its own dedicated memory with fast memory access
- Cross CPU memory access 
  - send request to other CPU,
  - other CPU read the data with its own memory controller
  - other  CPU delivery the data to the requiring CPU 
  - this delivery has slower memory access
+ IF the data is modified by requiring CPU, coherence protocol will mark it, so that other CPU need to update the data for consistency.
