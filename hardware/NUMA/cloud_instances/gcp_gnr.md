# GNR (C4) on GCP

Notes on Google Cloud's Intel-based machine series, focused on **C4 with Intel Xeon 6
(Granite Rapids)**: NUMA/compute-die topology, the SPR→GNR single-socket delta, and the
shape/feature comparison against C4-EMR.

See also: [GNR (8i) on AWS](aws_gnr.md) — the same processor generation on AWS.

---

## Contents

- [NUMA Topology](#numa-topology)
- [Intel Architecture Instance Types on Google Cloud](#intel-architecture-instance-types-on-google-cloud)
- [Machine Series Comparison: C4-EMR vs C4-GNR](#machine-series-comparison-c4-emr-vs-c4-gnr)
- [What C4 on Xeon 6 Adds](#what-c4-on-xeon-6-adds)
- [SPR to GNR in a Single Socket](#spr-to-gnr-in-a-single-socket)
- [Identifying a GNR-backed C4](#identifying-a-gnr-backed-c4)

---

## NUMA Topology

Each 72-core socket is presented as **3 NUMA nodes** — each with 24 cores and local memory. This
provides optimal memory latencies for NUMA-aware applications.

### SNC3 architecture

A GNR socket contains **3 compute dies** and **12 memory channels**, plus IO dies. With **SNC3**,
each compute die is presented as its own NUMA node:

```
                   C4-GNR socket (12 memory channels)

   +-------------+   +-------------+   +-------------+
   |  c-die 0    |   |  c-die 1    |   |  c-die 2    |
   |  24c / 48T  |   |  24c / 48T  |   |  24c / 48T  |
   |  4 mem ch   |   |  4 mem ch   |   |  4 mem ch   |
   +-------------+   +-------------+   +-------------+
     NUMA node 0       NUMA node 1       NUMA node 2

   +---------------------+   +---------------------+
   |        IO die       |   |        IO die       |
   +---------------------+   +---------------------+
```

A full 2-socket system is 6 compute dies and 6 NUMA nodes.

### Per-socket comparison

| | Cores per socket | vCPUs per socket | NUMA nodes per socket |
| --- | --- | --- | --- |
| C3 | 44 | 88 | 2 |
| C4-GNR | 72 | 144 | **3** |

### Sockets and NUMA nodes by VM size

| vCPUs | SPR shape (c3) | c3 sockets | c3 NUMA | GNR shape (c4-GNR) | c4 sockets | c4 NUMA |
| --- | --- | --- | --- | --- | --- | --- |
| 4 | `c3-standard-4` | 1 | 1 | `c4-standard-4` | 1 | 1 |
| 8 | `c3-standard-8` | 1 | 1 | `c4-standard-8` | 1 | 1 |
| 16 | NA | NA | NA | `c4-standard-16` | 1 | 1 |
| 22 | `c3-standard-22` | 1 | 1 | NA | NA | NA |
| 24 | NA | NA | NA | `c4-standard-24` | 1 | 1 |
| 32 | NA | NA | NA | `c4-standard-32` | 1 | 1 |
| 44 | `c3-standard-44` | 1 | 1 | NA | NA | NA |
| 48 | NA | NA | NA | `c4-standard-48` | 1 | 1 |
| 88 | `c3-standard-88` | 1 | 2 | NA | NA | NA |
| 96 | NA | NA | NA | `c4-standard-96` | 1 | 2 |
| 144 | NA | NA | NA | `c4-standard-144` | 1 | **3** |
| 176 | `c3-standard-176` | 2 | 4 | NA | NA | NA |
| 192 | `c3-standard-192-metal` | 2 | 4 | `c4-standard-192` | 2 | 4 |
| 288 | NA | NA | NA | `c4-standard-288` | 2 | **6** |

### C4-GNR NUMA node and compute die layout by shape

| Shape | vCPUs | Sockets | NUMA nodes | vCPUs/node | Cores/node | Compute die coverage |
| --- | --- | --- | --- | --- | --- | --- |
| `c4-standard-4` | 4 | 1 | 1 | 4 | 2 | Partial compute die |
| `c4-standard-8` | 8 | 1 | 1 | 8 | 4 | Partial compute die |
| `c4-standard-16` | 16 | 1 | 1 | 16 | 8 | Partial compute die |
| `c4-standard-24` | 24 | 1 | 1 | 24 | 12 | Partial compute die |
| `c4-standard-32` | 32 | 1 | 1 | 32 | 16 | Partial compute die |
| `c4-standard-48` | 48 | 1 | 1 | 48 | 24 | 1 full compute die |
| `c4-standard-96` | 96 | 1 | **2** | 48 | 24 | 2 full compute dies |
| `c4-standard-144` | 144 | 1 | **3** | 48 | 24 | Single socket (3 compute dies) |
| `c4-standard-192` | 192 | 2 | **4** | 48 | 24 | 1 socket + 1 compute die |
| `c4-standard-288` | 288 | 2 | **6** | 48 | 24 | Full system (6 compute dies) |

> **Note** — the vCPUs/node and Cores/node columns are derived (vCPUs ÷ NUMA nodes, halved for
> Hyper-Threading); the source deck states sockets, NUMA nodes, and die coverage directly. A
> compute die is 24 cores, so `c4-standard-48-lssd` is 48 vCPUs = 24 cores = one full compute die.

### Practical guidance

- **Shapes up to `c4-standard-48` are a single NUMA node**, so NUMA placement and pinning are moot
  there. `c4-standard-48` is exactly one full compute die.
- **Node width is a uniform 48 vCPUs / 24 cores** at every multi-node shape — simpler than C3,
  where a node is 44 vCPUs.
- **`c4-standard-192` is asymmetric.** It reports 2 sockets and 4 NUMA nodes while covering
  "1 socket + 1 compute die" — three dies on one socket and a single die on the other. The four
  nodes are *not* evenly distributed, unlike `c3-standard-192-metal` (an even 2 sockets × 2 nodes).
  Worth accounting for in thread placement.
- **`c4-standard-144` is the largest shape with no cross-socket traffic** (3 nodes, single socket).
- Verify the topology on a running instance with `lscpu` or `numactl -H` — these are general Linux
  tools, not something the source deck specifies.

### Cross-cloud comparison

The same processor generation is configured differently per cloud, so a NUMA node is **not** the
same width on AWS and GCP:

| | AWS 8i | GCP C4-GNR |
| --- | --- | --- |
| Cores per socket | 96 | 72 |
| vCPUs per socket | 192 | 144 |
| NUMA nodes per socket | 3 (SNC3) | 3 (SNC3) |
| **Cores per NUMA node** | **32** | **24** |
| **vCPUs per NUMA node** | **64** | **48** |
| Memory channels per socket / per node | 12 / 4 | 12 / 4 |
| Memory speed (as stated per cloud) | DDR5-7200 | 6400 MT/s |
| All-core turbo | 3.9 GHz | 3.9 GHz |
| Smallest multi-node size | 24xl (96 vCPU, 2 nodes) | `c4-standard-96` (2 nodes) |
| Largest single-socket size | 48xl (192 vCPU, 3 nodes) | `c4-standard-144` (144 vCPU, 3 nodes) |
| Largest system | 96xl (384 vCPU, 6 nodes) | `c4-standard-288` (288 vCPU, 6 nodes) |

> **Note** — code that hard-codes a 32-core NUMA node on AWS will mis-place threads on GCP, where a
> node is 24 cores. Size thread pools and pinning from the node width reported at runtime rather
> than from an assumed socket layout.

---

## Intel Architecture Instance Types on Google Cloud

| Category | Purpose |
| --- | --- |
| **General Compute** (N1, N2, N4) | Balanced compute, memory, and network resources for general-purpose workloads: analytics, databases, enterprise applications. |
| **Compute-Optimized** (C2, C3, C4, H3, C3.Metal) | Enhanced execution resources for compute-bound workloads: data science, ML/AI inference, gaming, HPC. |
| **Memory-Optimized / Large Memory / Storage-Optimized** (M1, M2, M3, M4, X4, Z3) | Enhanced memory or storage for memory-bound / IOPS-bound workloads: in-memory databases and analytics, SQL, HANA. |
| **VMware** (VE1, VE2) | Google Cloud VMware Engine (GCVE), optimized for VMware workloads. |

### Processor generation mapping

| Intel generation | Google Cloud machine series |
| --- | --- |
| 5th / 6th Gen Intel Xeon Scalable | N4, C4, X4, M4, C3 w/ Confidential Compute (Intel TDX), C3.Metal |
| 4th Gen Intel Xeon Scalable | C3, H3, VE2, Z3 |
| 3rd Gen Intel Xeon Scalable | N2 w/ Ice Lake |
| 2nd Gen Intel Xeon Scalable | N2, C2, M2, VE1 |
| Intel Xeon Scalable processors | N1, M1 |
| Intel Xeon v4 and earlier | N1, M1 (earlier variants) |

---

## Machine Series Comparison: C4-EMR vs C4-GNR

| | **C4 (EMR)** | **C4 (GNR)** |
| --- | --- | --- |
| **Predefined shapes** | Up to 192 vCPU, 1,536 GB RAM | **Up to 288 vCPU, 2,232 GB RAM** |
| **Custom shapes** | N/A | N/A |
| **Implementation** | Shapes aligned to processor architecture, enabling maximum isolation and consistency | Shapes aligned to processor architecture, enabling maximum isolation and consistency |
| **Frequency** | Up to 4.0 GHz (single-core max turbo) | **Up to 4.2 GHz (single-core max turbo)** |
| **Hyperdisk Balanced** | Up to 320K IOPS, up to 10 GB/s | Up to 320K IOPS, up to 5 GB/s |
| **Hyperdisk Extreme** | Up to 500K IOPS, up to 10 GB/s | Up to 500K IOPS, up to 10 GB/s |
| **Hyperdisk Throughput** | Planned for post-GA | N/A |
| **Local SSD** | Standard, Highmem (planned post-GA) | **Standard, Highmem** |
| **Networking (standard)** | Up to 100 Gbps | Up to 100 Gbps |
| **Networking (Tier_1)** | Up to 200 Gbps | Up to 200 Gbps |
| **Maintenance** | Advanced maintenance | Standard maintenance |
| **Additional features** | Sole Tenancy, compact + spread placement, Confidential Compute (post-GA) | Sole Tenancy (coming soon), spread placement |
| **Billing / consumption** | Standard CUDs, Flex CUDs, Spot, Reservations | Standard CUDs, Flex CUDs, Spot, Reservations |
| **Relative pricing** | 3.5% higher than N2 (+6% delta vs N4) | Priced similar to C4-EMR, with higher perf per vCPU |
| **Relative performance** (SIR-17, perf/vCPU) | 40% better vs N2; 32% better vs C3 (+11% delta vs N4) | 32% better vs C3; higher than C4-EMR |

With Titanium, C4 also offers up to **80% better CPU responsiveness** compared to previous
generations for real-time workloads, including high-frequency workloads.

The high performance of C4 is a good fit for:

- Databases and caches
- Network appliances
- High-traffic web and application servers, analytics
- Real-time CPU-based inference, with Intel AMX

---

## What C4 on Xeon 6 Adds

The C4 machine series expands to the latest Intel Xeon 6 processor (Granite Rapids), delivering new
capabilities, more shape options, and greater flexibility — targeting databases, analytics, gaming,
real-time platforms, and inference.

- **C4 with Titanium Local SSD** — new VM shapes featuring Titanium Local SSDs for I/O-intensive
  applications, with up to **35% lower local SSD latency**.
- **C4 Bare Metal** — for customers needing maximum control and flexibility; up to **35% better
  performance** than previous-generation bare metal instances.
- **Larger C4 shapes** — higher frequencies, larger cache sizes, and up to **2.2 TB of memory**,
  featuring the **highest frequency of any Google Compute Engine VM (up to 4.2 GHz)**. Enables
  databases, data analytics, and other memory-bound or license-constrained workloads to scale
  effectively.

---

## SPR to GNR in a Single Socket

| | SPR | GNR |
| --- | --- | --- |
| Frequency (all-core turbo) | 3.0 GHz | **3.9 GHz** |
| vCPUs | 88 | **144** |
| Memory channels | 8 | **12** |
| Memory speed | 4800 MT/s | **6400 MT/s** |
| SNC | SNC2 or SNC4 | **SNC3** |
| Hyper-Threading | On | On |
| SSD | N/A | **Yes** |

Example SPR instance: `c3-standard-88`  ·  Example GNR instance: `c4-standard-144`

---

## Identifying a GNR-backed C4

- C4 instances on GNR are differentiated from EMR by the **inclusion of local SSD**, or by starting
  the instance with `--min-cpu-platform="Intel Granite Rapids"`.
- GNR-backed C4 instance types display **"Intel Granite Rapids"** on the GCP instance page. The
  `C4` name alone **does not guarantee GNR** unless the page shows "Intel Granite Rapids".
