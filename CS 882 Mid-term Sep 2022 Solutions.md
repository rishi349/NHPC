# CS 882: Network-Based Computing for HPC
## Mid-term Examination Solutions (Sep 2022)

---

### Question 1 (5 points)
**What is a deadlock? Define how deadlocks can occur in distributed applications that use MPI. How do we overcome deadlocks?**

#### 1. Definition
A **deadlock** in a distributed system is a state where two or more processes are permanently blocked because each is waiting for an event (such as a message receipt, buffer release, or lock acquisition) that can only be triggered by another blocked process in the dependency cycle.

#### 2. How Deadlocks Occur in MPI
In MPI, deadlocks primarily arise from cyclic dependencies involving **blocking point-to-point communication** (`MPI_Send` and `MPI_Recv`).
* **Cyclic Blocking Pattern**:
  Suppose Process 0 and Process 1 both execute:
  ```c
  // Process 0:
  MPI_Send(..., 1, ...); // Blocked waiting for P1 to post Recv
  MPI_Recv(..., 1, ...);

  // Process 1:
  MPI_Send(..., 0, ...); // Blocked waiting for P0 to post Recv
  MPI_Recv(..., 0, ...);
  ```
* **Eager vs. Rendezvous Pitfall**:
  * For small messages, `MPI_Send` may complete immediately because the MPI runtime copies the payload into an internal eager buffer. The code appears to work.
  * For large messages, MPI switches to the **Rendezvous Protocol**, requiring a handshake acknowledgement from a matching `MPI_Recv` before transmitting. Because neither process reaches `MPI_Recv`, both block indefinitely in `MPI_Send`, causing a deadlock.
* **Collective Ordering Mismatch**: Deadlocks also occur when processes invoke collective operations in differing orders (e.g., P0 calls `MPI_Bcast` then `MPI_Reduce`, while P1 calls `MPI_Reduce` then `MPI_Bcast`).

#### 3. How to Overcome Deadlocks
1. **Reorder Send/Recv Calls (Symmetric Alternation)**:
   Break the cycle by pairing even and odd ranks:
   - Even ranks: `MPI_Send` followed by `MPI_Recv`.
   - Odd ranks: `MPI_Recv` followed by `MPI_Send`.
2. **Atomic Combined Operations (`MPI_Sendrecv`)**:
   Use `MPI_Sendrecv` or `MPI_Sendrecv_replace`, which guarantees internal deadlock-free scheduling of concurrent send and receive operations.
3. **Non-Blocking Communication (`MPI_Isend` / `MPI_Irecv` + `MPI_Waitall`)**:
   Post asynchronous requests non-blockingly so the network hardware initiates transfers without blocking CPU progression, followed by an explicit synchronization fence (`MPI_Waitall`).
4. **Persistent Communication Requests**: Using `MPI_Send_init` / `MPI_Recv_init` with `MPI_Startall`.

---

### Question 2 (5 points)
**What are the two protocols that MPI uses internally for exchanging point-to-point messages? What are the advantages and disadvantages of each of these protocols?**

The two internal protocols are the **Eager Protocol** and the **Rendezvous Protocol**, separated by an implementation threshold (typically 8 KB to 64 KB).

#### Comparison Table

| Feature | Eager Protocol | Rendezvous Protocol |
| :--- | :--- | :--- |
| **Target Message Size** | Small messages ($< \text{Threshold}$) | Large messages ($\ge \text{Threshold}$) |
| **Mechanism** | Sender transmits data immediately without waiting for receiver confirmation. Receiver stores it in a temporary system buffer if `MPI_Recv` is not yet posted. | 3-way handshake: Sender transmits a Request-to-Send (RTS); Receiver sends Clear-to-Send (CTS) with buffer address; Sender transfers bulk data directly into destination buffer; Finishes with ACK. |
| **Advantages** | 1. **Minimal Latency**: No round-trip handshake penalty ($T \approx \alpha + M\beta$).<br>2. **Decoupled Progression**: Sender completes immediately and continues local computation. | 1. **Zero-Copy Transfer**: Eliminates intermediate system buffer copies directly into user space via RDMA.<br>2. **Bounded Memory Footprint**: Receiver memory cannot be exhausted by unbounded unexpected messages.<br>3. **High Bandwidth Utilization**: Ideal for streaming large tensors/arrays. |
| **Disadvantages** | 1. **Memory Bloat**: Receiver can run out of memory if unexpected messages pile up in system queues.<br>2. **Double Copying**: Sender buffer $\to$ Network $\to$ Receiver system buffer $\to$ User target buffer (consumes memory bandwidth). | 1. **High Latency Overhead**: Incurs a full round-trip time (RTT) penalty for the RTS/CTS handshake.<br>2. **Strict Synchronization**: Sender cannot complete its send call until the receiver explicitly posts the matching receive. |

---

### Question 3 (5 points)
**Modern networks use advanced features like RDMA read or write for sending and receiving messages across systems, especially large messages. What are the pros and cons of using RDMA as compared to traditional methods like TCP?**

#### 1. Pros of RDMA over TCP
1. **Kernel Bypass (Zero OS Overhead)**:
   * **TCP**: Requires context switching into the OS kernel, traversing the TCP/IP network stack (socket layer, sk_buff allocation, TCP flow control, IP routing).
   * **RDMA**: User-space applications submit work requests (Work Queue Elements - WQEs) directly to the network interface card (NIC) hardware via Memory-Mapped I/O (MMIO), bypassing the kernel entirely.
2. **Zero-Copy Architecture**:
   * **TCP**: Incurs at least two memory copies (User buffer $\to$ Kernel socket buffer $\to$ DMA to NIC on sender; NIC DMA $\to$ Kernel buffer $\to$ User buffer on receiver), thrashing CPU cache.
   * **RDMA**: Network adapter hardware DMAs data directly between the remote host's physical memory and the local host's physical memory without intermediate CPU staging buffers.
3. **CPU Offload**:
   * RDMA packet parsing, CRC checks, sequence verification, and flow control are executed entirely inside the network adapter ASIC (e.g., Mellanox/NVIDIA ConnectX). Host CPU utilization drops close to 0%.
4. **Ultra-Low Latency and High Throughput**:
   * Latency drops from $\sim 10\text{–}30\ \mu\text{s}$ (TCP) to $< 1\text{–}2\ \mu\text{s}$ (InfiniBand/RoCE), fully saturating 200/400 Gbps line rates.

#### 2. Cons / Challenges of RDMA
1. **Memory Registration Overhead & Pinning**:
   * All memory buffers must be registered with the NIC and page-pinned (`mlock`) to physical RAM to prevent OS virtual memory paging. Registration is an expensive OS call ($O(\text{pages})$).
2. **Lossless Fabric Requirement (Go-Back-N / PFC)**:
   * RoCE (RDMA over Converged Ethernet) requires a lossless network fabric via Priority-based Flow Control (PFC) and Explicit Congestion Notification (ECN). Packet drops cause catastrophic performance degradation due to Go-Back-N retransmission.
3. **Scalability of Connection State (Queue Pairs)**:
   * Reliable Connected (RC) transport requires each node to maintain dedicated Queue Pairs (QP contexts) for every peer ($O(N^2)$ state across the cluster), exhausting on-chip NIC cache and causing QP-cache thrashing at large scale.
4. **Complex Error Recovery & Programming Model**:
   * Asynchronous event queues (Completion Queues), memory keys (rkey/lkey), and strict synchronization make application development and debugging far more complex than standard POSIX sockets.

---

### Question 4 (5 points)
**Describe the communication challenges in HPC applications like stencils? What are the techniques that researchers use for achieving better scaling?**

#### 1. Communication Challenges in Stencils
1. **Surface-to-Volume Ratio (Halo Exchange)**:
   * Stencils update a grid point using its nearest neighbors. When the domain is partitioned across processes, each process must exchange boundary layers (**ghost cells / halo zones**) at every time step.
   * As subdomain size decreases, the communication surface ($O(N^{d-1})$) dominates the local computation volume ($O(N^d)$), causing low compute-to-communication ratios.
2. **Strided Memory Access (Packing/Unpacking Overhead)**:
   * In multi-dimensional grids (e.g., 2D/3D Fortran/C arrays), boundary data along non-contiguous dimensions (e.g., Y and Z planes) is non-contiguous in memory. Sending them directly requires expensive CPU packing into contiguous buffers or non-contiguous MPI Datatypes with high copy overhead.
3. **Fine-Grained Synchronization Barrier**:
   * Time-stepping requires all nodes to complete their halo exchanges before the next iteration can advance. A single straggler node (due to OS jitter or network congestion) stalls the entire cluster.
4. **High Network Hop Contention**:
   * Simultaneous boundary exchanges among neighbors can cause bisection link congestion if the logical topology does not match the physical network topology.

#### 2. Scaling Techniques
1. **Communication-Computation Overlap (Non-blocking Isend/Irecv)**:
   * Decompose the local domain into an **inner core** and an **outer boundary (halo)**.
   * Step a: Post asynchronous halo receives and sends (`MPI_Irecv`, `MPI_Isend`).
   * Step b: Immediately compute the inner core domain (which has no dependencies on ghost cells).
   * Step c: Wait for communication to complete (`MPI_Waitall`).
   * Step d: Compute the outer boundary domain using the newly arrived ghost cells.
2. **Optimal Domain Decomposition (Square/Cubic Slabs vs. Pencils)**:
   * Partitioning a 3D grid into cubes ($P_x \times P_y \times P_z$) minimizes the total perimeter surface area compared to 1D planar slicing, minimizing communication volume.
3. **Derived MPI Datatypes & Hardware Packing**:
   * Use `MPI_Type_vector` and `MPI_Type_create_subarray` offloaded to GPU/NIC Direct Memory Access (e.g., GPUDirect Async) to pack strided memory without CPU intervention.
4. **Temporal Blocking / Stencil Tiling (Diamond Tiling)**:
   * Compute multiple time steps ($k$ steps) within an expanded local halo before performing communication, reducing network communication frequency by a factor of $k$ at the expense of redundant local compute.
5. **Topology-Aware Mapping**:
   * Use `MPI_Cart_create` and graph communicators to map logically adjacent stencil processes to physically adjacent cores or switches.

---

### Question 5 (5 points)
**Assume we fix the global stencil size as $8 \times 8 \times 8$ grid, can you describe how one can divide this problem across $2, 4, 8, 16, 32, 64, 128$ nodes? What could be the scaling challenges that one should consider while solving such problems?**

#### 1. Domain Decomposition across Node Counts
Total grid points = $8 \times 8 \times 8 = 512$ points.
The optimal decomposition splits along axes ($P_x \times P_y \times P_z = P$) while keeping subdomains as cubic as possible to minimize the surface-to-volume ratio:

| Nodes ($P$) | Grid Decomposition ($P_x \times P_y \times P_z$) | Subdomain Size per Node | Points per Node | Active Ghost Faces |
| :---: | :---: | :---: | :---: | :---: |
| **2** | $2 \times 1 \times 1$ | $4 \times 8 \times 8$ | 256 | 2 faces ($X$) |
| **4** | $2 \times 2 \times 1$ | $4 \times 4 \times 8$ | 128 | 4 faces ($X, Y$) |
| **8** | $2 \times 2 \times 2$ | $4 \times 4 \times 4$ | 64 | 6 faces ($X, Y, Z$) |
| **16** | $4 \times 2 \times 2$ | $2 \times 4 \times 4$ | 32 | 6 faces |
| **32** | $4 \times 4 \times 2$ | $2 \times 2 \times 4$ | 16 | 6 faces |
| **64** | $4 \times 4 \times 4$ | $2 \times 2 \times 2$ | 8 | 6 faces |
| **128** | $8 \times 4 \times 4$ | $1 \times 2 \times 2$ | 4 | 6 faces (1 pt thick) |

*(Note: Beyond 512 nodes, there would be fewer grid points than nodes).*

#### 2. Scaling Challenges for this Small Problem
1. **Severe Surface-to-Volume Catastrophe**:
   * At $P = 64$, each node holds a tiny $2 \times 2 \times 2 = 8$ grid points.
   * To update these 8 points, each node must exchange 6 boundary faces of size $2 \times 2 = 4$ points each $\implies$ Total halo data sent = $6 \times 4 = 24$ points.
   * **Communication volume (24 points) is $3\times$ larger than computation volume (8 points)!**
2. **Loss of Overlap Potential (No Interior Core)**:
   * When subdomain dimension along an axis drops to $1$ or $2$ (e.g., at $P=64$ and $P=128$), **every single point is on the boundary**.
   * There are **zero interior points**. It is mathematically impossible to hide communication latency behind compute because all points depend on incoming ghost cells.
3. **Network Latency Dominance ($\alpha \gg M\beta$)**:
   * Messages are minuscule ($1\times 2$ or $2\times 2$ floats $\approx 16\text{–}32$ bytes).
   * Bandwidth is irrelevant; execution time is completely dominated by network injection latency $\alpha$ and software overheads. Speedup collapses (Amdahl's law in the strong scaling limit).

---

### Question 6 (5 points)
**Achieving overlap of communication and computation while scaling MPI applications on high-end HPC systems has been hard. Can you briefly describe what are the practical challenges and what optimizations are available for achieving better performance?**

#### 1. Practical Challenges
1. **Lack of True Hardware Asynchrony (Progress Engine Stalls)**:
   * In many MPI libraries, non-blocking calls (`MPI_Isend`, `MPI_Irecv`) do not actually transmit data in the background unless the CPU explicitly calls an MPI function (e.g., `MPI_Test`, `MPI_Wait`). Without hardware offload, the communication stalls while the CPU is busy computing.
2. **Host CPU Core / Thread Contention**:
   * If MPI uses helper progress threads to advance communication, these threads fight with computation threads (e.g., OpenMP worker threads) for CPU cores, L3 cache, and memory bus bandwidth.
3. **Memory Bus Saturation**:
   * High-speed computation (e.g., streaming stencil sweeps or GEMM) fully saturates the memory bus (DRAM bandwidth). Concurrent DMA operations by the NIC over PCIe compete for memory controllers, slowing down computation.
4. **Coarse Problem Granularity / Dependencies**:
   * In algorithms where computation strictly depends on boundary exchange, the independent "core" computation is often too small to cover the network round-trip time.

#### 2. Optimizations Available
1. **Hardware Offloaded Communication Progress**:
   * Modern RDMA NICs (e.g., InfiniBand Core-Direct, NVIDIA SHARP) execute triggered operations and progress engines on the NIC ASIC without CPU involvement.
2. **Dedicated Core Reservation (Progress Cores)**:
   * Allocate one physical CPU core per node exclusively to the MPI communication progress engine (e.g., setting `MPICH_ASYNC_PROGRESS=1` or `MV2_ENABLE_AFFINITY=1`).
3. **Splitting Compute into Outer Boundary and Inner Core**:
   * Restructure algorithms into a two-stage pipeline: compute the boundary-independent core while asynchronous non-blocking transfers are in flight, then process the boundary once `MPI_Wait` resolves.
4. **CUDA Streams & GPUDirect RDMA (for GPUs)**:
   * Enqueue communication (via NCCL or GPUDirect) on a dedicated CUDA communication stream concurrent with kernel execution on computational CUDA streams.

---

### Question 7 (5 points)
**What are pros and cons of static and switched networks?**

#### Comparison

| Network Type | Architecture / Topology | Pros | Cons |
| :--- | :--- | :--- | :--- |
| **Static Networks (Direct / Point-to-Point)** | Nodes are directly wired to other nodes via fixed point-to-point links (e.g., 2D/3D Torus, Mesh, Hypercube, Ring). Nodes act as both compute entities and routers. | 1. **No External Switch Cost**: Eliminates expensive multi-stage spine/leaf switches.<br>2. **Predictable Local Latency**: Ultra-fast single-hop communication for nearest neighbors (ideal for stencils/domain decomposition).<br>3. **High Bisection Scalability**: Incremental scaling (add nodes + local cables together). | 1. **High Diameter**: Distant nodes require multi-hop store-and-forward routing ($O(N^{1/d})$ hops).<br>2. **Routing Overhead & Contention**: Forwarding traffic steals memory/link bandwidth from intermediate compute nodes.<br>3. **Rigid Topology**: Poor performance for non-localized patterns like All-to-All or random shuffle. |
| **Switched Networks (Indirect / Multi-Stage)** | Compute nodes connect exclusively to dedicated switching hardware (e.g., Crossbar switches, Fat-Tree, Clos networks, Spine-Leaf). | 1. **Low, Uniform Latency**: Any node can communicate with any other node in a fixed small number of hops (e.g., 3-stage Clos = 3 hops).<br>2. **Flexible Traffic Patterns**: Efficiently handles arbitrary permutation traffic, All-to-All, and dynamic deep learning workloads.<br>3. **Zero Host Forwarding Overhead**: Compute nodes do not route third-party traffic. | 1. **High Hardware Cost**: Switches, optical transceivers, and cable harnesses scale super-linearly with cluster size ($O(N \log N)$).<br>2. **Bisection Bottleneck Risk**: In oversubscribed fabrics, cross-switch spine links suffer from severe congestion. |

---

### Question 8 (5 points)
**What is the difference between ToR and EoR architecture?**

#### Detailed Comparison

| Attribute | Top of Rack (ToR) | End of Row (EoR) |
| :--- | :--- | :--- |
| **Switch Location** | Dedicated 1U/2U switch placed directly inside **each individual server rack** (usually at the top). | Centralized, large modular chassis switch placed in a dedicated cabinet at the **end of a server row**. |
| **Cabling Within Rack** | **Short, inexpensive copper cables (DAC)** connect servers directly to the in-rack switch. Clean and contained. | Long bundles of copper/fiber cables run from **every individual server across the row** through overhead cable trays into the EoR rack. |
| **Uplink Cabling** | Only a few high-speed uplinks leave each rack to connect to the aggregation/spine tier. | Extremely dense, heavy cable bundles in underfloor or overhead trays (cable management nightmare). |
| **Port Utilization & Cost** | Can result in wasted ports if a rack has fewer servers than switch ports. Higher number of total switches to purchase and manage. | Highly efficient port utilization (modular line cards added as needed). Fewer switches to manage. |
| **Failure Domain** | Switch failure only affects the **single rack** it resides in. | Central EoR switch/chassis failure can take down an **entire row** of servers. |
| **Modern HPC Standard** | **Predominant architecture in modern HPC/AI clusters** due to clean cabling and high-radix switches. | Legacy data center design; largely phased out for dense multi-GPU clusters. |

---

### Question 9 (10 points)
**Fat-Tree challenge: Assuming each node has a single NIC, how would you build a 1024-node fat-tree cluster using 128-port switches with full bisection bandwidth? Assume $\alpha$ is the latency, $B$ is the bandwidth of NIC, and the switch latency is $\beta$, can you describe the estimated latency, bandwidth of such a system across two nodes? Also, what is the bisection bandwidth of this cluster?**

#### 1. Topology Design (2-Level Fat-Tree / Clos Network)
* Switch Radix $k = 128$ ports.
* For **Full Bisection Bandwidth (non-blocking 1:1 ratio)**, each Leaf (Edge) switch must split its ports equally:
  * Downlink ports to compute nodes = $k / 2 = 128 / 2 = \mathbf{64}$.
  * Uplink ports to Spine (Core) switches = $k / 2 = 128 / 2 = \mathbf{64}$.

**Number of Leaf (Edge) Switches**:
$$\text{Leaf Switches} = \frac{\text{Total Nodes}}{\text{Downlinks per Leaf}} = \frac{1024}{64} = \mathbf{16 \text{ Leaf Switches}}$$

**Number of Spine (Core) Switches**:
* Total uplinks originating from all 16 Leaf switches = $16 \times 64 = 1024$ uplinks.
* Each Spine switch has 128 ports, all serving as downlinks to Leaf switches.
$$\text{Spine Switches} = \frac{\text{Total Leaf Uplinks}}{\text{Ports per Spine Switch}} = \frac{1024}{128} = \mathbf{8 \text{ Spine Switches}}$$

**Cabling Interconnect Rule**:
* Every Leaf switch connects $\frac{64}{8} = 8$ parallel physical links to **each** of the 8 Spine switches.
* Total Switches = $16 \text{ (Leaf)} + 8 \text{ (Spine)} = \mathbf{24 \text{ switches}}$.

---

#### 2. Latency and Bandwidth across Two Nodes

##### Case A: Intra-Switch Communication (Two nodes on the same Leaf switch)
* Path: $\text{Node A} \to \text{Leaf Switch} \to \text{Node B}$ (1 switch traversal).
* **Latency**:
  $$\text{Latency} = \alpha + \beta$$
* **Bandwidth**: Limited by the single NIC line rate:
  $$\text{Bandwidth} = B$$

##### Case B: Inter-Switch Communication (Two nodes on different Leaf switches)
* Path: $\text{Node A} \to \text{Leaf Switch 1} \to \text{Spine Switch} \to \text{Leaf Switch 2} \to \text{Node B}$ (3 switch traversals).
* **Latency**:
  $$\text{Latency} = \alpha + 3\beta$$
* **Bandwidth**: Because the fabric has full bisection bandwidth (non-blocking 1:1), each flow achieves full wire speed without oversubscription:
  $$\text{Bandwidth} = B$$

---

#### 3. Bisection Bandwidth of this Cluster
* In a balanced 1024-node cluster, bisecting the system splits the nodes into two equal halves of 512 nodes each (8 Leaf switches on the left, 8 Leaf switches on the right).
* All 8 Leaf switches in Half 1 have $8 \times 64 = 512$ uplinks running into the Spine tier.
* Because the network is full bisection (non-blocking), exactly 512 links cross between the two halves simultaneously.
$$\text{Bisection Bandwidth} = \frac{N}{2} \times B = \frac{1024}{2} \times B = \mathbf{512 \cdot B}$$
*(e.g., if $B = 200\text{ Gbps}$, Bisection Bandwidth = $512 \times 200\text{ Gbps} = 102.4\text{ Tbps}$ bi-directional).*

---

### Question 10 (5 points)
**Can you describe the different ways of parallelism in distributed deep learning training applications?**

#### 1. Data Parallelism (DP / DDP)
* **Strategy**: The complete model is replicated identically across all $P$ worker GPUs. The global training batch is sliced into $P$ mini-batches.
* **Execution**: Each worker independently computes a forward and backward pass on its local mini-batch to generate local weight gradients $\nabla W_i$.
* **Communication**: Workers execute an **`MPI_Allreduce` (Sum)** across all GPUs to average gradients before updating weights:
  $$W^{(t+1)} = W^{(t)} - \eta \cdot \frac{1}{P}\sum_{i=1}^P \nabla W_i$$

#### 2. Tensor Parallelism (TP / Megatron-LM)
* **Strategy**: Intra-layer model parallelism. Large individual weight matrices (Linear/MLP layers, Self-Attention projections) are split across multiple GPUs.
* **Execution**:
  * **Column Parallel Linear Layer**: Matrix $W$ is sliced vertically: $W = [W_1 \mid W_2]$.
  * **Row Parallel Linear Layer**: Matrix $W$ is sliced horizontally. Requires an `Allreduce` across the output to sum partial dot-products: $Y = X_1 W_1 + X_2 W_2$.
* **Communication**: Requires high-frequency `Allreduce` or `Allgather` inside every single transformer layer. Demands ultra-fast intra-node interconnects (NVLink).

#### 3. Pipeline Parallelism (PP / GPipe / PipeDream)
* **Strategy**: Inter-layer model parallelism. The layers of a deep neural network are partitioned sequentially across multiple GPUs (e.g., Layers 1–8 on GPU 0, Layers 9–16 on GPU 1).
* **Execution**: Activations flow forward via point-to-point transfers ($P_i \to P_{i+1}$), and gradients flow backward ($P_{i+1} \to P_i$).
* **Mitigating Bubbles**: Splitting mini-batches into **micro-batches** (GPipe 1F1B schedule) keeps pipeline stages busy, reducing the idle pipeline bubble fraction to $\frac{P-1}{m + P - 1}$.

#### 4. Hybrid / 3D Parallelism (TP + PP + DP) & ZeRO
* **Strategy**: Combines TP within a node (over NVLink), PP across nodes (over InfiniBand), and DP across pipeline replicas.
* **ZeRO (Zero Redundancy Optimizer)**: Shards optimizer states (ZeRO-1), gradients (ZeRO-2), and model parameters (ZeRO-3) across data-parallel ranks using `Reduce-Scatter` and `Allgather`, eliminating memory duplication.

---

### Question 11 (5 points)
**Explain how one can achieve overlap of communication with computation in distributed deep learning applications with data parallelism?**

#### 1. The Core Insight: Layer-by-Layer Backward Progression
In deep neural networks, the backward pass computes gradients in reverse topological order (from the output layer $L$ back to the input layer $1$):
$$\text{Loss} \to \nabla W_L \to \nabla W_{L-1} \to \dots \to \nabla W_2 \to \nabla W_1$$
* **The Opportunity**: As soon as layer $L$'s gradient $\nabla W_L$ is computed, its value is finalized. It does **not** depend on the gradients of earlier layers ($L-1, L-2, \dots, 1$).
* Therefore, the system does not need to wait for the entire backward pass to finish before starting communication.

#### 2. Implementation Mechanics (Bucket-based Pipelined AllReduce)
PyTorch DDP (Distributed Data Parallel) achieves near 100% overlap through the following pipeline:
1. **Autograd Hook Registration**:
   * DDP attaches asynchronous hooks (`autograd_hook`) to every parameter tensor.
   * The moment backpropagation finishes computing gradients for a layer, the hook fires immediately.
2. **Gradient Bucketing**:
   * Firing an individual `Allreduce` for every tensor creates massive latency overhead ($\alpha$).
   * DDP groups parameter gradients into contiguous memory buffers called **Buckets** (typically 25 MB).
   * Once a bucket fills with finalized gradients, an asynchronous non-blocking **`NCCL AllReduce`** is launched immediately on a separate CUDA communication stream.
3. **Dual-Stream Execution (Compute Stream + Communication Stream)**:
   * The default CUDA Compute Stream proceeds backwards computing $\nabla W_{L-1}, \nabla W_{L-2}$.
   * Concurrently, the CUDA Communication Stream drives the hardware NIC via GPUDirect RDMA to reduce Bucket $L$ over the network.
   * **Result**: Communication time for most layers is completely hidden behind backward computation.

---

### Question 12 (10 points)
**Describe the steps of Allreduce algorithms Ring, Rabenseifner, pair-wise exchange algorithm using the following example:**

Given 8 processes ($P1$ to $P8$), each holding an array of 8 integers:
```
Index:   0  1  2  3  4  5  6  7
P1:    [ 0, 1, 2, 3, 0, 1, 2, 3 ]
P2:    [ 0, 1, 2, 3, 0, 1, 2, 3 ]
P3:    [ 0, 1, 2, 3, 0, 1, 2, 3 ]
P4:    [ 0, 1, 2, 3, 0, 1, 2, 3 ]
P5:    [ 0, 1, 2, 3, 0, 1, 2, 3 ]
P6:    [ 0, 1, 2, 3, 0, 1, 2, 3 ]
P7:    [ 0, 1, 2, 3, 0, 1, 2, 3 ]
P8:    [ 0, 1, 2, 3, 0, 1, 2, 3 ]
```
*(Summed target across all 8 processes at each index: $8 \times [0, 1, 2, 3, 0, 1, 2, 3] = \mathbf{[0, 8, 16, 24, 0, 8, 16, 24]}$)*.

Let $N = 8$, message length $M = 8$ integers.

---

#### 1. Ring AllReduce
Divide the array of 8 elements into $N = 8$ chunks of size 1 ($S_0, S_1, \dots, S_7$).
Arrange processes in a logical ring: $P1 \to P2 \to P3 \to P4 \to P5 \to P6 \to P7 \to P8 \to P1$.

##### Phase 1: Reduce-Scatter ($N - 1 = 7$ steps)
In step $k$ ($k=0 \dots 6$), process $i$ sends chunk $(i - k) \pmod 8$ to its right neighbor, receives from its left neighbor, and sums it into its local chunk.
* **Step 1**: $P1$ sends chunk 0 to $P2$; $P2$ sends chunk 1 to $P3$; ...; $P8$ sends chunk 7 to $P1$.
* ... (Repeats for 7 steps)
* **End of Phase 1**: Each process holds the fully reduced global sum for exactly **one** chunk:
  * $P1$ holds reduced chunk $S_1 = 1 \times 8 = \mathbf{8}$.
  * $P2$ holds reduced chunk $S_2 = 2 \times 8 = \mathbf{16}$.
  * $P3$ holds reduced chunk $S_3 = 3 \times 8 = \mathbf{24}$.
  * $P4$ holds reduced chunk $S_4 = 0 \times 8 = \mathbf{0}$.
  * $P5$ holds reduced chunk $S_5 = 1 \times 8 = \mathbf{8}$.
  * $P6$ holds reduced chunk $S_6 = 2 \times 8 = \mathbf{16}$.
  * $P7$ holds reduced chunk $S_7 = 3 \times 8 = \mathbf{24}$.
  * $P8$ holds reduced chunk $S_0 = 0 \times 8 = \mathbf{0}$.

##### Phase 2: Allgather ($N - 1 = 7$ steps)
Each process circulates its fully reduced chunk around the ring for 7 steps without performing arithmetic, overwriting old values.
* **Total Steps**: $7 + 7 = \mathbf{14 \text{ steps}}$.
* **Data transferred per step**: 1 integer ($M/N$).

---

#### 2. Rabenseifner's Algorithm (Recursive Halving & Recursive Doubling)

##### Phase 1: Recursive Halving (Reduce-Scatter) in $\log_2 8 = 3$ steps
At step $k$ ($k=1, 2, 3$), processes pair up at distance $2^{3-k}$. One process reduces the lower half, the other reduces the upper half:
* **Step 1 (Distance 4, Chunk size = 4 integers)**:
  * Pairs: $(P1, P5), (P2, P6), (P3, P7), (P4, P8)$.
  * $P1$ exchanges with $P5$:
    * $P1$ sends upper half (indices 4..7) to $P5$; $P5$ sends lower half (indices 0..3) to $P1$.
    * $P1$ computes sum of indices $0..3$: $[0+0, 1+1, 2+2, 3+3] = [0, 2, 4, 6]$.
    * $P5$ computes sum of indices $4..7$: $[0+0, 1+1, 2+2, 3+3] = [0, 2, 4, 6]$.
  * Similarly, pairs $(P2, P6), (P3, P7), (P4, P8)$ do the same.
* **Step 2 (Distance 2, Chunk size = 2 integers)**:
  * Pairs: $(P1, P3), (P2, P4)$ on lower halves; $(P5, P7), (P6, P8)$ on upper halves.
  * $P1$ sends indices 2..3 to $P3$; $P3$ sends indices 0..1 to $P1$.
  * $P1$ accumulates sum for indices $0..1$: $[0+0, 2+2] = [0, 4]$.
  * $P3$ accumulates sum for indices $2..3$: $[4+4, 6+6] = [8, 12]$.
* **Step 3 (Distance 1, Chunk size = 1 integer)**:
  * Pairs: $(P1, P2), (P3, P4), (P5, P6), (P7, P8)$.
  * $P1$ and $P2$ exchange: $P1$ gets index 0 ($0 \times 8 = \mathbf{0}$); $P2$ gets index 1 ($1 \times 8 = \mathbf{8}$).
  * At the end of Step 3, every node holds exactly 1 fully reduced element.

##### Phase 2: Recursive Doubling (Allgather) in $\log_2 8 = 3$ steps
Reverse the process:
* **Step 4 (Distance 1)**: $P1$ and $P2$ exchange their 1 element $\to$ each holds indices $0..1$ ($[0, 8]$).
* **Step 5 (Distance 2)**: $(P1, P2)$ exchange 2 elements with $(P3, P4) \to$ all hold indices $0..3$ ($[0, 8, 16, 24]$).
* **Step 6 (Distance 4)**: $(P1..P4)$ exchange 4 elements with $(P5..P8) \to$ all hold complete array $[0, 8, 16, 24, 0, 8, 16, 24]$.
* **Total Steps**: $3 + 3 = \mathbf{6 \text{ steps}}$.

---

#### 3. Pair-Wise Exchange Algorithm (XOR-Based Shift)
Ranks are indexed $0 \dots 7$ ($P1=0 \dots P8=7$). The message is split into 8 chunks of 1 integer.
The algorithm runs in two distinct phases:

##### Phase 1: Reduce-Scatter via XOR Schedule ($N-1 = 7$ rounds)
In each round $k = 1 \dots 7$, node $i$ pairs with partner $j = i \oplus k$.
* The chunk exchanged is determined by schedule indices:
  * In round 1 ($k=1$): $(P1, P2), (P3, P4), (P5, P6), (P7, P8)$ exchange and accumulate disjoint chunks.
  * In round 2 ($k=2$): $(P1, P3), (P2, P4), (P5, P7), (P6, P8)$ exchange and accumulate.
  * ... continues through round 7 ($k=7$).
* After 7 steps, each node holds exactly 1 fully reduced integer.

##### Phase 2: Allgather via XOR Schedule ($N-1 = 7$ rounds)
The exact same XOR pairing sequence is executed, but nodes exchange fully reduced chunks without summing.
* **Total Steps**: $7 + 7 = \mathbf{14 \text{ steps}}$.

---

### Question 13 (15 points)
**Let's assume $B$ is the bandwidth of the NIC on each node, $M$ is the message size and $N$ is the total number of nodes in the system. Analyze the ring, rabenseifner and pair-wise exchange allreduce algorithms in terms of message size per step, total number of steps required, is it latency optimized and/or bandwidth optimized, whether the algorithm is congestion friendly and whether the algorithm is easy to handle non-power of 2 nodes and non-power of 2 message sizes.**

#### Comprehensive Analytical Comparison Table

| Feature / Metric | Ring AllReduce | Rabenseifner AllReduce | Pair-wise Exchange AllReduce |
| :--- | :--- | :--- | :--- |
| **Message Size per Step** | **Constant**: $\frac{M}{N}$ | **Variable (Halves/Doubles)**: Starts at $\frac{M}{2}$, drops to $\frac{M}{N}$, doubles back to $\frac{M}{2}$ | **Constant**: $\frac{M}{N}$ |
| **Total Number of Steps** | **$2(N - 1)$ steps**<br>($(N-1)$ Reduce-Scatter + $(N-1)$ Allgather) | **$2 \log_2 N$ steps**<br>($\log_2 N$ Rec-Halving + $\log_2 N$ Rec-Doubling) | **$2(N - 1)$ steps**<br>(or $2\log_2 N$ if hypercube-based) |
| **Total Comm. Time ($T$)** | $2(N - 1)\alpha + 2\frac{N - 1}{N}\frac{M}{B}$ | $2\log_2 N \cdot \alpha + 2\frac{N - 1}{N}\frac{M}{B}$ | $2(N - 1)\alpha + 2\frac{N - 1}{N}\frac{M}{B}$ |
| **Latency Optimized?** | **No ($O(N)$)**: Poor for large $N$ due to long chain of sequential dependencies. | **Yes ($O(\log N)$)**: Highly latency optimized; minimal step count. | **No ($O(N)$)**: Linear step count hurts on large node counts. |
| **Bandwidth Optimized?** | **Yes ($100\%$)**: Transmits optimal volume $2\frac{N-1}{N}M \approx 2M$; fully saturates bidirectional NIC. | **Yes ($100\%$)**: Transmits optimal volume $\sum \frac{M}{2^k} = 2\frac{N-1}{N}M \approx 2M$. | **Yes ($100\%$)**: Transmits optimal volume $2\frac{N-1}{N}M \approx 2M$. |
| **Congestion Friendly?** | **Extremely Friendly**: Zero network congestion. Only nearest neighbors on the 1D ring communicate. Contention-free. | **Prone to Congestion**: Stride-based pairing ($N/2, N/4$) causes concurrent cross-switch spine traffic on Fat-Trees. | **Severe Congestion**: Round-dependent XOR pairings cross bisection switches simultaneously (e.g., `0 0 0 0 8 8 8 8`). |
| **Non-power of 2 Nodes?** | **Trivial**: Works natively for any integer $N$ simply by constructing a ring of length $N$. | **Difficult**: Requires isolating $r = N - 2^{\lfloor \log_2 N \rfloor}$ surfeit nodes, executing pre-checkin and post-checkout steps. | **Difficult**: XOR pairing breaks; requires virtual rank expansion or shift-based fallback. |
| **Non-power of 2 Msg Size?** | **Moderate**: Requires $M$ divisible by $N$. If not, uneven chunk sizes or padding with dummy bytes is required. | **Difficult**: Array must be divisible by $2^{\log_2 N} = N$ at every level of recursive division; needs memory padding. | **Moderate**: Requires $M$ divisible by $N$. |

---

### Question 14 (10 points)
**Describe the three algorithms of all2all including pair-wise exchange, linear exchange and optimal algorithm and explain the pros and cons of each algorithm.**

In `MPI_Alltoall`, every node holds $N$ distinct personalized messages of size $m$ (total data $M = N \cdot m$) and sends a unique piece to every other node.

---

#### 1. Pair-Wise Exchange (XOR-Scheduled Exchange)
* **Mechanism**: Runs in $N - 1$ rounds. In round $k$ ($k = 1 \dots N-1$), process $i$ communicates bi-directionally with process $j = i \oplus k$.
* **Pros**:
  * Clean, contention-free port pairing on flat crossbar switches (every node has exactly 1 send and 1 receive partner per round; no receiver incast).
* **Cons**:
  * **Severe Bisection Congestion on Multi-Stage Networks**: On multi-switch topologies (like Fat-Trees), rounds where $k \ge N/2$ force all nodes on Switch A to send across the bisection link simultaneously. For $N=8$, bisection traffic profile is `0 0 0 0 8 8 8 8` (Slide 25).
  * Strictly requires $N$ to be a power of 2.

---

#### 2. Linear Shift Exchange (Round-Robin Shift)
* **Mechanism**: Runs in $N - 1$ rounds. In round $k$ ($k = 1 \dots N-1$), process $i$ sends to $j = (i + k) \pmod N$ and receives from $(i - k) \pmod N$.
* **Pros**:
  * Trivial implementation that works natively for **any non-power-of-2 $N$**.
  * Eliminates sudden 4-round deadlock slams seen in XOR.
* **Cons**:
  * **Pyramid Bisection Bottleneck**: Cross-switch traffic rises to a peak at distance $k = N/2$. For $N=8$, bisection profile is `0 2 4 6 8 6 4 2` (Slide 26). The middle round saturates bisection bandwidth and causes queue bloat.

---

#### 3. Optimal Algorithm (Bruck’s Algorithm / Node-Ordered Balanced Exchange)
Depending on the message regime, the "Optimal" algorithm takes two forms:

##### A. For Small Messages: Bruck's Algorithm ($\lceil \log_2 N \rceil$ phases)
* **Mechanism**: Expresses target ranks in binary. Performs local array rotation, followed by $\lceil \log_2 N \rceil$ communication rounds where node $i$ sends packed blocks of size $2^k \cdot m$ to $(i + 2^k) \pmod N$, finishing with inverse local rotation.
* **Pros**:
  * **Latency Optimal**: Cuts network latency hops from $N-1$ down to **$\lceil \log_2 N \rceil$**.
* **Cons**:
  * Transmits redundant data ($\approx \frac{N}{2}\log_2 N \cdot m$), making it inefficient for large messages. Incurs local memory copy overhead for index rotation.

##### B. For Large Messages: Node-Ordered Balanced Exchange (Slide 27)
* **Mechanism**: Staggers the communication pairs across nodes so that in *every single round*, exactly $\frac{N}{2}$ messages cross the bisection link and $\frac{N}{2}$ stay local.
* **Pros**:
  * **Bandwidth & Bisection Optimal**: Bisection traffic profile is completely flat: **`4 4 4 4 4 4 4 4`** (Slide 27). Eliminates traffic spikes, buffers never overflow, and NICs operate at 100% steady-state wire speed.
* **Cons**:
  * Requires offline topology discovery to compute optimal permutation schedules.

---

### Question 15 (5 points)
**Do we have a global clock in distributed systems? What are some of the practical challenges associated with clocks in distributed systems?**

#### 1. Do we have a global clock?
**No.** There is no single, physically unified, shared global clock hardware in distributed systems. Every independent compute node possesses its own local physical quartz crystal oscillator that counts clock ticks independently.

#### 2. Practical Challenges Associated with Clocks

1. **Clock Drift and Clock Skew**:
   * **Clock Skew**: The difference in absolute time readings between two clocks at a given instant.
   * **Clock Drift**: The difference in frequency (tick rate) between two clocks caused by physical crystal imperfections, temperature fluctuations, and component aging (typically drifts $1\text{–}100\ \mu\text{s}$ per second). Clocks inevitably diverge over time.
2. **Network Uncertainty in Synchronization (NTP / PTP Limitations)**:
   * Protocols like NTP (Network Time Protocol) and PTP (Precision Time Protocol / IEEE 1588) synchronize clocks by exchanging timestamped packets over the network.
   * Because network queueing delays, switch buffering, and OS interrupt latencies are non-deterministic and asymmetric, synchronization cannot guarantee zero error. PTP achieves sub-microsecond accuracy only with hardware timestamping NICs and PTP-aware switches.
3. **Relativity and Event Ordering (Violation of Causality)**:
   * If Node A sends a message to Node B at local time $T_A$, Node B might receive it at local time $T_B < T_A$ due to clock skew.
   * In distributed databases and debugging traces, physical timestamps can make an effect appear to occur *before* its cause.
4. **Failure of Monotonicity (Time Jumps)**:
   * Naive clock adjustments that synchronize a lagging clock forward or backward can produce non-monotonic time (time moving backwards), breaking timers, schedulers, and log analysis. (Systems require slow frequency adjustments / clock slewing to prevent this).
5. **Logical Solution**:
   * Because physical global time is impossible to guarantee perfectly, distributed algorithms rely on **Lamport Logical Timestamps**, **Vector Clocks**, or bounded-uncertainty physical clocks (e.g., Google TrueTime with atomic clocks and GPS receivers).
