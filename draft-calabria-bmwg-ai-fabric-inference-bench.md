---
title: "Benchmarking Methodology for AI Inference Serving Network Fabrics"
abbrev: "AI Inference Fabric Benchmarking"
category: info

docname: draft-calabria-bmwg-ai-fabric-inference-bench-latest
submissiontype: IETF
number:
date: 2026-02-24
consensus: true
v: 3
area: "Operations and Management"
workgroup: "Benchmarking Methodology Working Group"
keyword:
 - benchmarking
 - AI
 - inference
 - LLM
 - network fabric
 - RDMA
 - KV cache
 - MoE
 - disaggregated serving

venue:
  group: "BMWG"
  type: "Working Group"
  mail: "bmwg@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/bmwg/"
  github: "fcalabri/bmwg-ai-fabric-inference-bench"
  latest: "https://fcalabri.github.io/bmwg-ai-fabric-inference-bench/draft-calabria-bmwg-ai-fabric-inference-bench.html"

author:
 - fullname: Fernando Calabria
   initials: F.
   surname: Calabria
   organization: Cisco
   country: United States
   email: fcalabri@cisco.com
 - fullname: Carlos Pignataro
   initials: C.
   surname: Pignataro
   organization: Blue Fern Consulting
   country: United States
   email: carlos@bluefern.consulting
 - fullname: Qin Wu
   initials: Q.
   surname: Wu
   organization: Huawei
   country: China
   email: bill.wu@huawei.com
 - fullname: Giuseppe Fioccola
   initials: G.
   surname: Fioccola
   organization: Huawei
   country: Italy
   email: giuseppe.fioccola@huawei.com

normative:
  RFC1242:
  RFC2544:
  RFC2889:
  RFC6349:
  RFC6815:
  RFC8238:
  RFC8239:
  TERMINOLOGY: I-D.calabria-bmwg-ai-fabric-terminology
  TRAINING-BENCH: I-D.calabria-bmwg-ai-fabric-training-bench
  UEC-SPEC:
    title: "UEC Specification 1.0"
    author:
      org: "Ultra Ethernet Consortium"
    date: 2024

informative:
  RFC7432:
  RFC3849:

...

--- abstract

This document defines benchmarking terminology, methodologies, and Key
Performance Indicators (KPIs) for evaluating Ethernet-based AI inference
serving network fabrics. As Large Language Model (LLM) inference deployments
scale to disaggregated prefill/decode architectures spanning hundreds or
thousands of accelerators (GPUs/XPUs), the interconnect fabric becomes the
critical bottleneck determining Time to First Token (TTFT), Inter-Token
Latency (ITL), and aggregate throughput in tokens per second (TPS). This
document establishes vendor-independent, reproducible test procedures for
benchmarking fabric-level performance under realistic AI inference workloads.

Coverage includes RDMA-based KV cache transfer between disaggregated prefill
and decode workers, Mixture-of-Experts (MoE) expert parallelism AllToAll
communication, request routing and load balancing for inference serving,
congestion management under bursty inference traffic patterns, and scale/soak
testing. The methodology enables direct, equivalent comparison across
implementations, NIC transport stacks (RoCEv2, UET), and fabric architectures.

This document is a companion to {{!TRAINING-BENCH}}, which addresses training
workloads.

--- middle

# Introduction

Large Language Model (LLM) inference serving has emerged as a dominant consumer
of datacenter network capacity, with fundamentally different fabric requirements
compared to training workloads. While training workloads are characterized by
bulk synchronous collective operations (AllReduce, AllGather) with predictable
periodicity, inference workloads exhibit bursty, latency-sensitive
request/response patterns with strict Service Level Objectives (SLOs) on
per-token latency and time-to-first-token.

The advent of disaggregated serving architectures, where the computationally
intensive prefill phase (prompt processing) is physically separated from the
memory-bound decode phase (token generation), introduces a new class of
fabric-critical data movement: KV cache transfer. A single large prompt
processed by a typical large-scale model generates multiple gigabytes of KV
cache state that must be transferred from prefill workers to decode workers
within a fraction of the target TTFT SLO.

As clusters scale with thousands of concurrent requests, this creates sustained
multi-terabyte-per-second aggregate transfer demands on the fabric.
Simultaneously, Mixture-of-Experts (MoE) architectures introduce expert
parallelism (EP), which distributes expert sub-networks across GPUs and requires
AllToAll communication for token-to-expert routing. Wide EP configurations
(e.g., 96-way EP across 12 nodes of 8 GPUs each) generate fine-grained,
latency-sensitive inter-node traffic that contends with KV cache transfers on
shared fabric links.

This document defines vendor-independent benchmarking methodologies for
evaluating how well a network fabric supports these inference-specific traffic
patterns. All tests are designed for controlled laboratory environments using
either hardware traffic generators or software workload emulators capable of
reproducing inference serving traffic profiles.

## Requirements Language

{::boilerplate bcp14-tagged}

## Scope and Applicability

The scope covers Layer 2/3 fabric performance (switch forwarding, link utilization,
 congestion management), RDMA transport performance (one-sided PUT/GET operations
 for KV cache transfer, two-sided SEND/RECV for expert parallelism dispatch), and
the interaction between fabric behavior and application-level inference metrics
 (TTFT, ITL, TPS).

The DUT boundary for all measurements in this document is defined as the NIC-to-NIC
Ethernet fabric segment — specifically, the path from the point of packet transmission
 by the source NIC Ethernet port to the point of packet reception at the destination NIC
 Ethernet port.

Intra-node transfer segments (proprietary accelerator interconnects GPU-to-GPU, and PCIe/CXL GPU-to-NIC) are explicitly
OUT OF SCOPE as primary benchmarked entities.  Where intra-node transfer contributes
 measurably to an end-to-end latency measurement (e.g., TTFT decomposition in {{end-to-end-disaggregated-ttft}}), implementers report intra-node transfer time as a separately labelled component
 so that the fabric contribution can be isolated.  See Section 3.2 for DUT boundary diagram.

The document does NOT address benchmarking of individual accelerator (GPU/XPU) compute performance, model accuracy or quality metrics benchmarking of the inference serving
 software stack in isolation from the fabric.

All methodologies assume controlled laboratory conditions per BMWG convention.

## Relationship to Existing BMWG Work

This document builds upon the foundational BMWG benchmarking framework
established by {{RFC1242}}, {{RFC2544}}, {{RFC2889}}, and {{RFC6349}}.

The test structure follows RFC 2544 conventions for trial duration (minimum 60
seconds), statistical repetition (minimum 20 trials for latency, 50 for burst),
and reporting format (graphical and tabular).

The methodologies extend RFC 2544 Section 26 benchmarks (throughput, latency,
frame loss rate, back-to-back frames, system recovery, reset) to
inference-specific scenarios including KV cache transfer, expert parallelism
dispatch, and disaggregated serving request routing.

## Relationship to Companion Documents

This document is a companion to {{!TRAINING-BENCH}}, which defines benchmarking
methodologies for AI training network fabrics. Both documents share common
terminology (Section 2), test topology conventions (Section 3), and reporting
formats ({{reporting}}). Both documents use the terminology defined in
{{!TERMINOLOGY}}, which provides the common terminology base for AI fabric
benchmarking.

Where training workloads are dominated by bulk synchronous collective
communication (AllReduce, AllGather) with high bandwidth utilization and
periodic synchronization barriers, inference workloads are dominated by bursty,
latency-sensitive point-to-point transfers (KV cache) and fine-grained AllToAll
dispatch (MoE expert parallelism). Implementers deploying converged fabrics that
serve both training and inference workloads should run both test suites.

# Terminology

Terminology used in this document is defined in {{!TERMINOLOGY}}. Readers should consult that document before applying the methodology defined here. Where a term overlaps with {{!RFC1242}} or {{!RFC8238}}, the terminology document provides AI fabric context extensions; the foundational definitions in those RFCs remain authoritative for general network benchmarking.

The following terms are bench-specific extensions used only in this document and are not redefined in {{!TERMINOLOGY}}:

| Term | Definition |
|---|---|
| **TTFT_fabric** | The fabric-segment contribution to Time to First Token (TTFT), measured at the DUT-PD boundary. Comprises the KV cache transfer time over the Ethernet fabric only; excludes intra-node (PCIe/CXL/accelerator-interconnect) contributions. Reported alongside SUT-E TTFT to enable fabric/non-fabric decomposition. |
| **ITL_fabric** | The fabric-segment contribution to Inter-Token Latency (ITL), measured at the DUT-F boundary. Comprises the per-decode-step EP dispatch round-trip over the fabric; excludes intra-node and compute contributions. |
| **DUT-S** | Single-switch DUT configuration; see {{dut-id}}. |
| **DUT-F** | Complete-fabric DUT configuration; see {{dut-id}}. |
| **DUT-N** | NIC-transport DUT configuration; see {{dut-id}}. |
| **DUT-PD** | Prefill-Decode-path DUT configuration; see {{dut-id}}. |
| **SUT-E** | End-to-end inference SUT configuration; see {{dut-id}}. |
{: #tab-terminology title="Bench-Specific Terminology Extensions"}

The scope of the DUT for the tests defined in this document is the Ethernet fabric segment connecting prefill and decode workers (and, where applicable, expert-parallel groups), consistent with the Fabric DUT Boundary defined in {{!TERMINOLOGY}}.

Worked examples of the S_KV formula and KV cache size computation for representative model architectures are provided in the appendix.



# Test Topology and Architecture

## Reference Fabric Topologies

The reference topologies from the companion training document (2-Tier Clos,
3-Tier Clos, Rail-Optimized) remain applicable. Inference serving introduces
additional topology considerations related to disaggregated prefill/decode
placement and MoE expert distribution.

### Topology A: 2-Tier Clos (Leaf-Spine)

Applicable to inference clusters up to approximately 2,048 accelerators. Prefill
and decode worker groups are placed on separate leaf switches (or separate
leaf switch groups) to isolate KV cache transfer traffic from decode-to-client
response traffic. Expert parallelism (EP) traffic within a single MoE dispatch
group is confined to a single leaf switch or a minimal number of leaf
switches to minimize spine-hop latency.

### Topology B: 3-Tier Clos (Leaf-Spine-Superspine)

Required for inference clusters exceeding 2,048 accelerators or for multi-model
serving deployments where different model instances occupy different fabric pods.
KV cache transfer traffic between prefill and decode workers in different pods
traverses the superspine tier, making superspine bandwidth and latency critical.

### Topology C: Disaggregated Prefill/Decode Placement

A topology variant specific to inference serving in which prefill workers and
decode workers are placed in distinct physical locations within the fabric,
connected by a dedicated KV cache transfer network segment. This topology enables
independent scaling of prefill and decode resources and allows heterogeneous
hardware (e.g., high-compute GPUs for prefill, high-memory-bandwidth GPUs for
decode).

~~~
          +----------------------+
          |   Request Router     |
          |   (KV-Aware LB)      |
          +--------+-------------+
                   |
      +------------+--------------+
      |                           |
+-----v-------+         +---------v-----+
| Prefill Pool|         |  Decode Pool  |
| (xP workers)|         |  (yD workers) |
| High Compute|         | High Mem BW   |
| TP=8, DP=N/8|         | TP=8, DP=M/8  |
+------+------+         +-------+-------+
       |                        |
       | KV Cache RDMA Transfer |
       | (One-sided PUT/Signal) |
       +------------------------+
~~~
{: #fig-pd-topology title="Disaggregated Prefill/Decode Inference Topology"}

## Disaggregated Prefill/Decode Topology

The disaggregated topology separates the inference pipeline into physically
distinct pools connected by the fabric. The test topology includes the
following components:

* **Prefill Worker Pool:** N Prefill nodes, each containing G accelerators with
  high-compute capability. These workers execute the prefill phase and generate
  KV cache state. Tensor Parallelism (TP) is applied within each node; Data
  Parallelism (DP) is applied across nodes. Each prefill worker communicates
  with one or more decode workers via RDMA-based KV cache transfer.

* **Decode Worker Pool:** M Decode nodes, each containing G accelerators with high
  memory bandwidth. These workers receive KV cache state from prefill workers and
  execute the autoregressive decode phase. DP Attention may partition the KV
  cache across DP ranks within the decode pool, requiring AllToAll communication
  during decode.

* **KV Cache Transfer Network:** The Ethernet fabric segment connecting prefill and decode worker pools. This segment carries one-sided RDMA PUT operations (or PUT-with-signal) transferring KV cache blocks from prefill GPU memory to decode GPU memory via RDMA over Converged Ethernet (RoCEv2) or Ultra Ethernet Transport (UET).

  The end-to-end transfer from GPU memory to remote GPU memory traverses three segments:

  - (1) GPU-to-NIC: PCIe/CXL (intra-node, out of scope as DUT);
  - (2) NIC-to-NIC: Ethernet fabric (the DUT, in scope);
  - (3) NIC-to-GPU: PCIe/CXL at destination (intra-node, out of scope as DUT).

  Benchmarking procedures in {{test-cat1}} and {{test-cat2}} measure fabric-segment latency and throughput exclusively. When end-to-end measurements are reported (e.g., TTFT decomposition), the intra-node segments are labelled separately.

  ~~~~
  GPU Memory --> [PCIe/CXL] --> NIC --> [ETHERNET FABRIC] --> NIC --> [PCIe/CXL] --> GPU Memory
  <---intra-node (out of scope)--->|<------DUT (in scope)------->|<---intra-node (out of scope)--->
  ~~~~

* **Request Router:** A network-layer or application-layer load balancer that
  assigns incoming inference requests to prefill workers and subsequently routes
  KV cache to the appropriate decode workers. KV-aware routing and prefix-aware
  caching policies are under test.

## Device Under Test (DUT) Identification {#dut-id}

The following table defines the DUT configurations tested in this document:

| DUT ID | Description | Components Under Test |
|--------|-------------|----------------------|
| DUT-S | Single Switch | Individual leaf or spine switch forwarding inference traffic. Measures per-hop latency, buffer absorption, ECN marking accuracy. |
| DUT-F | Complete Fabric | End-to-end fabric from prefill NIC egress to decode NIC ingress. Measures fabric-level KV cache transfer latency, throughput, and congestion behavior. |
| DUT-N | NIC Transport | NIC RDMA transport stack processing KV cache transfer operations. Measures RDMA verb completion latency, one-sided PUT bandwidth, QP scaling. |
| DUT-PD | Prefill-Decode Path | The complete data path from prefill GPU memory through NIC, fabric, NIC, to decode GPU memory. Measures end-to-end KV cache transfer including proprietary accelerator-interconnect, PCIe/CXL, and fabric segments. |
| SUT-E | End-to-End System | Complete inference serving system including inference serving software, RDMA transfer libraries, fabric, and accelerators. Measures TTFT, ITL, TPS as functions of fabric performance. |
{: #tab-dut title="DUT Configuration Definitions"}

## Traffic Generator and Workload Emulator Requirements

Tests in this document require one or both of the following traffic generation
modes. The mode used is documented in all test reports.

### Hardware Traffic Generator (RT) - Minimum Requirements

The hardware traffic generator satisfies all of the following:

* RDMA traffic generation supporting RoCEv2 and, where tested, UET transport;
  configurable RDMA verb types (one-sided PUT, PUT-with-signal, two-sided
  SEND/RECV).

* Configurable message sizes from 4 KB (minimum KV cache page) to 256 MB
  (large KV cache block).

* Configurable QP counts from 1 QP to a minimum of 256 QPs per
  source-destination port pair.

### Software Workload Emulator (WE) - Minimum Requirements

A software workload emulator runs on actual accelerators and generates realistic
inference workloads. The WE supports all of the following:

* Configurable prompt length distributions: uniform, Zipf, and trace-replay
  modes.

* Configurable output length distributions and configurable request arrival
  rates: Poisson, bursty, and trace-replay.

* Disaggregated prefill/decode execution with actual RDMA-based KV cache
  transferring between prefill and decode worker pools.

* MoE expert parallelism with actual AllToAll dispatch where MoE-specific tests
  ({{test-cat3}}) are performed.

* Measurement instrumentation providing per-request TTFT and ITL with timestamp
  accuracy <= 1 millisecond.

When a software workload emulator is used, the complete software configuration
is documented per {{dut-id}}, as framework version, RDMA library version,
and GPU driver version materially affect results.

# KPI Framework and Metrics Taxonomy {#kpi-framework}

This section defines the Key Performance Indicators measured across all test
categories. KPIs are organized into four tiers: Primary Latency KPIs
(end-user-facing response time metrics), Primary Throughput KPIs (system-level
capacity metrics), Fabric-Level KPIs (network-specific measurements), and Fabric
Health Indicators (operational monitoring metrics).

> NOTE: Per BMWG charter, the definition of acceptance criteria or performance requirements is explicitly outside the scope of this Working Group. The KPI tables in this section define what is measured; they do not set thresholds. Indicative non-normative reference values reflecting current industry observations are provided in {{indicative-reference-values}}; those values MUST NOT be used as pass/fail thresholds in vendor evaluations.

## Primary Latency KPIs

| KPI | Unit | Definition | Measurement Point |
|-----|------|------------|-------------------|
| TTFT | ms | Time from request arrival to first output token emission | SUT-E request/response boundary |
| ITL | ms | Time between successive output tokens | SUT-E token emission timestamps |
| TTFT_fabric | ms | Fabric contribution to TTFT (KV cache transfer latency) | DUT-PD NIC-to-NIC measurement |
| ITL_fabric | ms | Fabric contribution to ITL (EP dispatch latency per decode step) | DUT-F EP dispatch round-trip |
| E2E_latency | ms | End-to-end request latency from arrival to completion of all output tokens | SUT-E request/response boundary |
{: #tab-latency-kpis title="Primary Latency KPIs"}

## Primary Throughput KPIs

| KPI | Unit | Definition | Measurement Point |
|-----|------|------------|-------------------|
| TPS_input | tokens/s | Aggregate input (prefill) tokens processed per second across all workers | SUT-E prefill completion events |
| TPS_output | tokens/s | Aggregate output (decode) tokens generated per second across all workers | SUT-E token emission events |
| TPS_per_GPU | tokens/s/GPU | Output tokens per second normalized by number of decode GPUs | SUT-E per-worker counters |
| Goodput | GB/s or tokens/s | See the Goodput definition in {{!TERMINOLOGY}} Reports use Inference_Goodput for token-rate measurements and Fabric_Goodput for byte-rate fabric measurements | SUT-E successful completion events |
| KV_BW | GB/s | Aggregate KV cache transfer bandwidth between prefill and decode pools | DUT-PD RDMA counters |
| Request_Rate | req/s | Maximum sustained request arrival rate meeting all latency SLOs | SUT-E admission control boundary |
{: #tab-throughput-kpis title="Primary Throughput KPIs"}

## Fabric-Level KPIs

| KPI | Unit | Definition | DUT |
|-----|------|------------|-----|
| KV_xfer_latency | us | One-sided RDMA PUT completion time for a single KV cache block transfer | DUT-N |
| KV_xfer_bandwidth | GB/s | Sustained unidirectional KV cache transfer throughput per NIC port | DUT-N |
| EP_alltoall_latency | us | Round-trip time for a complete MoE expert parallelism AllToAll dispatch | DUT-F |
| EP_alltoall_bandwidth | GB/s | Aggregate AllToAll bandwidth across all EP ranks during dispatch | DUT-F |
| Fabric_FCT | us | Flow completion time for a KV cache transfer flow through the fabric | DUT-F |
| Buffer_utilization | % | Peak switch buffer utilization during KV cache transfer bursts | DUT-S |
| ECN_marking_rate | % | Fraction of packets marked with ECN-CE during inference traffic | DUT-S |
| PFC_frame_count | frames | Number of PFC PAUSE frames generated per unit time | DUT-S |
| Link_utilization | % | Average and peak link utilization on fabric links carrying inference traffic | DUT-F |
| Packet_drop_rate | ppm | Packets dropped per million due to buffer overflow or transport error | DUT-F |
{: #tab-fabric-kpis title="Fabric-Level KPIs"}

## Fabric Health Indicators

| Indicator | Threshold | Description |
|-----------|-----------|-------------|
| CPU Utilization (switch) | < 30% | Control plane CPU usage on switches under inference traffic load |
| Memory Usage (switch) | < 70% | TCAM, buffer, and control plane memory usage |
| FEC Error Rate | < 1e-12 post-FEC BER | Forward Error Correction effectiveness on fabric links |
| CRC Error Count | 0 | Layer 2 CRC errors on any fabric link |
| BGP/OSPF Stability | 0 flaps | Routing protocol adjacency stability under inference load |
| NIC QP State | 100% active | All RDMA Queue Pairs in active state (no error/reset) |
| GPU-NIC PCIe BW | > 90% of theoretical | PCIe Gen5 x16 bandwidth utilization between GPU and NIC |
{: #tab-health title="Fabric Health Indicators"}

# Test Category 1: RDMA KV Cache Transfer Benchmarks {#test-cat1}

KV cache transfer between disaggregated prefill and decode workers is the
defining fabric workload for inference serving. Unlike training collectives
(AllReduce, AllGather) which are periodic and predictable, KV cache transfers
are event-driven (triggered by prefill completion) and bursty.

## Point-to-Point KV Cache Transfer Throughput

**Objective:** To determine the maximum sustained KV cache transfer throughput
between a single prefill worker NIC and a single decode worker NIC across the
DUT fabric.

**Procedure:** Configure a single RDMA connection (QP) between the prefill and
decode endpoints. Send a sequence of one-sided RDMA PUT operations with message
sizes corresponding to KV cache block sizes. The message size sequence
includes: 64 KB (single attention page), 256 KB, 1 MB, 4 MB, 16 MB, 64 MB,
256 MB (large prompt KV cache), and 1 GB. For each message size, transmit at
the maximum rate sustainable by the NIC for a minimum of 60 seconds per trial.
Repeat for 1, 4, 8, 16, 32, 64, and 128 concurrent QPs. The DUT is the fabric
path from NIC to NIC.

**Measurement:** Record throughput (GB/s), CPU utilization on both endpoints,
GPU memory-to-NIC transfer overhead, and NIC hardware offload utilization. The
test is repeated a minimum of 20 times per configuration and the average
reported.

**Reporting Format:** Results are reported as a multi-line graph with
message size (log scale) on the X axis and throughput (GB/s) on the Y axis.
Separate lines for each QP count. A reference line showing theoretical NIC line
rate is included.

## KV Cache Transfer Latency

**Objective:** To determine the latency of individual KV cache block transfers
across the DUT fabric under varying load conditions.

**Procedure:** Using the same endpoint configuration as Test 5.1, measure the
completion time of individual RDMA PUT operations. Latency is measured from the
initiation of the PUT verb on the prefill NIC to receipt of the completion
signal on the decode NIC (for PUT-with-signal) or to polling of the remote
completion queue. Measure latency under unloaded conditions (single outstanding
operation) and under loaded conditions (background traffic at 25%, 50%, 75%,
and 90% of fabric capacity). Message sizes include 64 KB, 1 MB, 16 MB,
and 256 MB.

**Measurement:** Report latency at P50, P95, P99, and P99.9 percentiles. The
test is repeated a minimum of 20 trials of at least 120 seconds each per
configuration. The difference between P99 and P50 (tail latency spread) is
reported as a derived metric.

**Reporting Format:** Results are reported as a table with columns for
message size, background load level, and latency at each percentile. A
complementary CDF plot of latency distribution for selected configurations
is included.

## Concurrent KV Cache Transfer Scaling

**Objective:** To characterize how aggregate KV cache transfer performance
scales as the number of concurrent prefill-to-decode transfer pairs increases.

**Procedure:** Configure N concurrent prefill-decode endpoint pairs, where N
ranges from 1 to the maximum supported by the fabric (e.g., 1, 2, 4, 8, 16,
32, 64, 128 pairs). Each pair executes continuous KV cache transfers of 16 MB
messages (representative of a medium-length prompt). Measure aggregate
throughput and per-pair latency as N increases.

**Measurement:** Report aggregate throughput (GB/s), per-pair median latency
(us), per-pair P99 latency (us), Jain Fairness Index across pairs, and maximum
fabric link utilization observed. The test is repeated a minimum of 20
times per value of N.

**Reporting Format:** Results are reported as a dual-axis graph with N
(concurrent pairs) on the X axis, aggregate throughput on the left Y axis, and
P99 latency on the right Y axis. The JFI value for each N is annotated.

## Multi-Tier Storage Transfer Characterization

**Objective:** To characterize KV cache transfer performance across the
memory/storage hierarchy: GPU HBM to GPU HBM (inter-node RDMA), GPU HBM to
remote CPU DRAM (offload), CPU DRAM to GPU HBM (reload), and GPU HBM to
NVMe/SSD (persistent cache).

**Procedure:** For each tier pair, measure unidirectional transfer throughput
and latency for message sizes of 1 MB, 16 MB, and 256 MB. Use zero-copy
transfers where supported (GPU-direct storage paths for NVMe and GPU-direct RDMA for inter-node, where the implementation provides them).

**Measurement:** Report throughput (GB/s) and latency (P50, P99) for each tier
pair and message size. Report the tier throughput ratio relative to GPU-to-GPU
RDMA as a derived metric.

**Reporting Format:** Results are reported as a table with rows for each
tier pair and columns for throughput and latency at each message size.

# Test Category 2: Prefill/Decode Disaggregation Benchmarks {#test-cat2}

Disaggregated prefill/decode serving separates the two phases onto distinct
hardware pools to enable independent optimization and scaling. This section
benchmarks the fabric's ability to support the resulting KV cache transfer
traffic patterns and their impact on end-to-end inference metrics.

## End-to-End Disaggregated TTFT

**Objective:** To measure TTFT as a function of prompt length in a disaggregated
serving configuration, isolating the fabric contribution.

**Procedure:** Configure a disaggregated serving system (SUT-E) with a specified
xPyD ratio (e.g., 3P9D for a 12-node cluster). Submit inference requests with
prompt lengths of 128, 512, 1024, 2048, 4096, 8192, and 16384 tokens. For each
prompt length, measure the total TTFT and decompose it into: T_prefill (prefill
compute time), T_transfer (KV cache fabric transfer time, measured at DUT-PD),
and T_decode_init (first decode step time).

**Measurement:** Report TTFT (ms) and its decomposition at P50, P95, and P99
percentiles. The ratio T_transfer/TTFT (fabric fraction) is reported as
a derived metric. The test is repeated a minimum of 20 trials per prompt
length.

**Reporting Format:** Results are reported as a stacked bar chart with
prompt length on the X axis and TTFT (ms) on the Y axis, with bars decomposed
into T_prefill, T_transfer, and T_decode_init. A table of numerical values accompanies the chart.

## xPyD Ratio Optimization

**Objective:** To determine the optimal prefill-to-decode resource ratio for a
given model, prompt distribution, and latency SLO, as limited by fabric transfer
capacity.

**Procedure:** For a fixed total number of nodes N (e.g., 12), iterate over
xPyD ratios: 1P11D, 2P10D, 3P9D, 4P8D, 6P6D, 8P4D, 10P2D, 11P1D. For each
ratio, submit a sustained request stream matching a target request rate with a
specified prompt length distribution (e.g., Zipf with alpha=1.0 over
\[128, 8192\] tokens). Measure TTFT P99, ITL P99, TPS_output, and Goodput for
each configuration.

**Measurement:** Report all four metrics for each xPyD ratio and request rate.
Identify the Pareto-optimal ratio(s) that maximize TPS_output while meeting
TTFT P99 < 500 ms and ITL P99 < 50 ms.

**Reporting Format:** Results are reported as a multi-panel figure with
one panel per request rate, each showing xPyD ratio on the X axis and metrics
on dual Y axes (TTFT/ITL on left, TPS on right). The Pareto frontier is
highlighted.

## Heterogeneous Parallelism Configuration

**Objective:** To evaluate the fabric impact of using different parallelism
strategies on prefill vs. decode pools in a disaggregated configuration.

**Procedure:** Test the following parallelism configurations:

* Prefill TP=8, Decode TP=8 (baseline, same parallelism)

* Prefill TP=8, Decode TP=4 with DP_Attention=2 (reduced TP, added DP)

* Prefill TP=4 with DP=2, Decode TP=2 with DP_Attention=4 (aggressive DP)

**Measurement:** Report the number of concurrent RDMA flows, aggregate bandwidth
(GB/s), TTFT (ms), and ITL (ms) at P50 and P99 for each configuration.

## Prefill Queue Depth Impact on Transfer Latency

**Objective:** To measure how queuing of prefill requests (due to compute
contention) affects KV cache transfer burstiness and fabric congestion.

**Procedure:** Oversubscribe the prefill pool by submitting requests at a rate
exceeding prefill capacity. Measure the resulting KV cache transfer burst
characteristics: burst size, burst duration, inter-burst gap, and peak fabric
bandwidth demand. Vary the oversubscription ratio from 1.0x (saturated) to 2.0x
in 0.25x increments.

**Measurement:** Report burst size distribution, peak and average fabric
bandwidth, KV transfer latency P99, and ECN/PFC event counts as functions of
oversubscription ratio.

# Test Category 3: MoE Expert Parallelism Benchmarks {#test-cat3}

Mixture-of-Experts models distribute expert sub-networks across GPUs and route
tokens to the appropriate experts via AllToAll communication. This section
benchmarks the fabric's ability to support the resulting fine-grained,
latency-sensitive inter-GPU traffic patterns.

## AllToAll Dispatch Throughput

**Objective:** To determine the maximum AllToAll dispatch throughput for MoE
expert parallelism across the DUT fabric.

**Procedure:** Generate a synthetic MoE dispatch workload where each GPU sends token embeddings to the experts selected by a top-k routing function.
The dispatch payload per GPU per MoE layer is:

T_dispatch = (B * k * H_model * P_bytes) / N. where B = batch size (tokens), k = top-k routing count,
H_model = hidden dimension, P_bytes = precision bytes (BF16=2), N = EP group size

**Canonical MoE Test Matrix**

| Config                                                | E (experts)                                   | k (top-k) | H_model | T_dispatch (B=128, BF16, N=96) |
| ----------------------------------------------------- | --------------------------------------------- | --------- | ------- | ------------------------------ |
| M1                                                    | 8                                             | 2         | 4096    | 2.1 MB/GPU                     |
| M2                                                    | 64                                            | 4         | 7168    | 29  MB/GPU                     |
| M3                                                    | 256                                           | 2         | 7168    | 14  MB/GPU                     |
| M4                                                    | 256                                           | 8         | 7168    | 58  MB/GPU                     |
| M5                                                    | (implementer-defined — report all parameters) |           |         |                                |
{: #tbl-moe-matrix title="Canonical MoE Test Matrix"}

**Measurement:** Report aggregate bandwidth (GB/s), per-dispatch latency (us)
at P50 and P99, and GPU idle time waiting for dispatch completion. The test is repeated a minimum of 20 times per configuration.

**Reporting Format:** Results are reported as a heatmap with EP group size
on the Y axis, batch size on the X axis, and throughput (GB/s) as the color
dimension. A companion latency table is included. Reports state which config row(s) were used. For M5, the values of E, k, H_model, P_bytes, and N are included in the results table.

NOTE: When per-accelerator normalized throughput (BusBW) is reported alongside EP_alltoall_bandwidth, BusBW is computed per the BusBW definition in {{!TERMINOLOGY}}; algo_factor is fixed per collective type and does not depend on the algorithm the library selects at runtime. The runtime algorithm in use is verified via library tracing and documented as part of the test conditions.

## Routing Mode and Dispatch Mode Comparison

**Objective:** To compare fabric performance across dispatch modes and routing policies. Tests cover Normal Dispatch and Low-Latency Dispatch.  Tests should additionally cover at least one alternative routing mode from {{tbl-routing-modes}}.

**Routing Mode Taxonomy**

| Mode                                                     | Description | Traffic Impact |
| -------------------------------------------------------- | ----------- | -------------- |
| Standard Top-k                                           | Each token routed to k. highest-scoring experts | Fixed, uniform AllToAll dispatch volume |
| Expert Choice (EC)                                       | Experts select tokens; ensures load balance | Non-uniform message sizes; tests HOL-blocking resilience |
| Top-k with Token Drop                                    | Overloaded experts drop excess tokens | Lower peak traffic; unpredictable under load |
| Auxiliary Loss Top-k                                     | Load-balanced top-k via training loss | Near-uniform AllToAll; lower hot-spot risk |
{: #tbl-routing-modes title="MoE Routing Mode Taxonomy"}

**Measurement:** Measure dispatch latency, fabric bandwidth, and routing mode impact on AllToAll traffic distribution and fabric congestion per {{tbl-routing-modes}}. Results from different routing modes are reported in separate result tables with the routing mode labelled.

## Wide Expert Parallelism Scaling

**Objective:** To characterize AllToAll dispatch performance as EP group size
scales beyond a single node (wide EP), requiring inter-node fabric communication.

**Procedure:** Scale the EP group from intra-node only (EP=8) to wide EP (EP=16, 32, 48, 64, 96 spanning 2, 4, 6, 8, 12 nodes). Use a fixed batch size of 128 tokens and at least one configuration from the canonical MoE test matrix {{tbl-moe-matrix}}.
The selected config row is identified in the results.

**Measurement:** Report total dispatch latency (us), inter-node bandwidth
(GB/s), and latency decomposition (intra-node vs. inter-node fraction). Report
the scaling efficiency: (EP=8 latency) / (EP=N latency) * (N/8).

## Expert Parallelism and KV Cache Transfer Contention

**Objective:** To measure the mutual interference between EP AllToAll dispatch
traffic and KV cache transfer traffic when both share the same fabric links.

**Procedure:** On a shared fabric, simultaneously execute: (a) continuous KV
cache transfers at a sustained rate (e.g., 50%, 75% of fabric capacity), and
(b) periodic EP AllToAll dispatches (one per MoE layer forward pass).

**Measurement:** Report KV_xfer_latency P99 (us) and EP_alltoall_latency P99
(us) for the isolated and contended cases. Report the contention penalty as the
ratio of contended P99 to isolated P99 for each traffic class. Report ECN/PFC
event counts during contention.

# Test Category 4: Congestion Management Benchmarks {#test-cat4}

Inference traffic patterns differ from training in their burstiness,
heterogeneity (mixed KV cache transfers and EP dispatches), and latency
sensitivity.

## ECN Marking Under Inference Incast

**Objective:** To verify that ECN marking thresholds are correctly applied when
multiple prefill workers simultaneously transfer KV cache blocks to a single
decode worker (incast pattern).

**Procedure:** Configure M prefill workers (M = 2, 4, 8, 16, 32) to
simultaneously transfer 16 MB KV cache blocks to a single decode worker port.
Repeat for ECN marking thresholds of 100 KB, 500 KB, 1 MB, and 5 MB. The DUT
is the individual leaf switch (DUT-S).

**Measurement:** Report the ECN marking rate (fraction of marked packets), the
onset of marking, queue depth at marking onset, and aggregate throughput
achieved. Repeat a minimum of 20 times per configuration.

## PFC Behavior Under Bursty KV Cache Transfers

**Objective:** To characterize PFC PAUSE frame generation and propagation under
bursty KV cache transfer patterns typical of disaggregated serving.

**Procedure:** Generate KV cache transfer bursts: N_burst concurrent transfers
(N_burst = 4, 8, 16, 32), each of size 16 MB, arriving within a window of
T_arrival (100 us, 1 ms, 10 ms). Vary the PFC threshold from 10 KB to 1 MB.

**Measurement:** Report PFC frame count, total PAUSE duration (us),
head-of-line blocking delay imposed on other traffic classes (us), and KV cache
transfer completion time.

## Congestion Control Convergence for Mixed Traffic

**Objective:** To measure the convergence time of DCQCN (or UET congestion
control) when KV cache transfer traffic and EP AllToAll dispatch traffic share
fabric capacity.

**Procedure:** Establish a sustained KV cache transfer at 80% of fabric
capacity. Introduce EP AllToAll dispatch traffic on the same fabric links.
Measure the convergence time to stable rate allocation. Repeat with the roles
reversed.

**Measurement:** Report convergence time (ms) to within 5% of steady-state
rates, steady-state bandwidth allocation between traffic classes, packet loss
during convergence, and Jain Fairness Index of the steady-state allocation.

## PFC Storm and Deadlock Resilience

**Objective:** To verify that the fabric does not enter a PFC storm or deadlock
condition under adversarial inference traffic patterns.

**Procedure:** Per the companion training document, generate a PFC storm
scenario by creating circular buffer dependency across multiple switches.
Simultaneously inject KV cache transfer traffic on all affected paths. Monitor
for PFC storm propagation, deadlock, and recovery time. The test duration is at least 300 seconds.

**Measurement:** Report whether PFC storm occurred (yes/no), deadlock occurred
(yes/no), maximum PAUSE propagation depth (number of hops), maximum
zero-throughput duration (ms), and recovery time (ms).

# Test Category 5: Request Routing and Load Balancing {#test-cat5}

Inference serving introduces application-layer routing decisions that interact
with fabric-layer load balancing (ECMP, flowlet, packet spray).

## KV-Aware Request Routing Efficacy

**Objective:** To measure the effectiveness of KV-aware request routing, where
the request router considers decode worker KV cache memory occupancy and fabric
path congestion when assigning requests.

**Procedure:** Configure a request router with KV-aware routing enabled. Submit
a sustained request stream at rates of 10, 50, 100, and 200 req/s. Compare
against round-robin routing (baseline).

**Measurement:** Report the coefficient of variation (CV) of decode worker
memory utilization, P99 TTFT, P99 ITL, KV cache eviction rate, and Goodput for
both KV-aware and round-robin routing.

## Prefix-Aware Cache Hit Rate

**Objective:** To measure the fabric bandwidth savings achieved by prefix-aware
caching, where requests with common prefixes are routed to workers that already
hold the corresponding KV cache segment.

**Procedure:** Generate a request workload where P% of requests share a common
prefix of L tokens (P = 25%, 50%, 75%, 90%; L = 256, 512, 1024, 2048). Compare
against non-prefix-aware routing.

**Measurement:** Report cache hit rate (%), fabric bandwidth reduction (%),
TTFT reduction (ms), and TPS improvement (%) for each (P, L) combination.

## ECMP and Dynamic Load Balancing Under Inference Traffic

**Objective:** To evaluate fabric-layer load balancing effectiveness under
inference traffic patterns characterized by a mix of large KV cache flows and
small EP dispatch flows.

**Procedure:** Measure link utilization uniformity under: (a) KV cache transfers
only (large flows, 16 MB+), (b) EP AllToAll dispatches only (small flows,
< 1 MB), (c) mixed KV cache and EP traffic.

**Measurement:** Report JFI, maximum link utilization (%), minimum link
utilization (%), and the oversubscription ratio for each scenario and load
balancing algorithm.

## Jain Fairness Index for Decode Worker Utilization

**Objective:** To measure how evenly the fabric distributes KV cache transfer
load across decode workers.

**Procedure:** With N_D decode workers (N_D = 8, 16, 32, 64), submit a
sustained request stream and measure per-worker KV cache receive rate, GPU
utilization, and output TPS.

**Measurement:** Report JFI for KV cache receive rate, GPU utilization, and
output TPS. Report the max/min ratio for each metric.

# Test Category 6: Latency Benchmarks

Inference latency is the primary user-facing quality metric. This section
defines benchmarks that isolate the fabric's contribution to end-to-end
inference latency.

## TTFT Under Varying Prompt Lengths

**Objective:** To characterize TTFT as a function of prompt length, isolating
the fabric-dependent KV cache transfer component.

**Procedure:** Submit single requests (no concurrent load) with prompt lengths
of 128, 256, 512, 1024, 2048, 4096, 8192, and 16384 tokens. Measure TTFT and
decompose into T_prefill, T_transfer, and T_decode_init. As a refernce the following table
is provided.

| Config ID      | Model Profile                                             | S_KV @ 4K ctx     | S_KV @ 32K ctx     | S_KV @ 128K ctx     |
| -------------- | --------------------------------------------------------- | ----------------- | ------------------ | ------------------- |
| CFG-A          | Small: L=32, H_kv=8 (GQA), D=128, BF16                    | 0.25 GB           | 2.0 GB             | 8.0 GB              |
| CFG-B          | Mid: L=80, H_kv=8 (GQA), D=128, BF16 (~70B-parameter dense class)         | 1.3 GB            | 10.5 GB            | 42.0 GB             |
| CFG-C          | Large MHA: L=96, H_kv=64 (MHA), D=128, BF16               | 12.3 GB           | 98.6 GB            | >300 GB             |
| CFG-D          | Mid INT8: L=80, H_kv=8 (GQA), D=128, INT8 (quantized)     | 0.67 GB           | 5.4 GB             | 21.5 GB             |
| CFG-E (custom) | Implementer-defined:  L=___, H_kv=___, D=___, P=___       | Computed          | Computed           | Computed            |
{: #tab-conf-matrix title="Reference Configuration Matrix"}

**Measurement:** Report TTFT, T_transfer, and T_transfer/TTFT at P50, P95, P99
for each prompt length. The test is repeated a minimum of 100 times per
prompt length

**Reporting Format:** Results specify the configuration ID (CFG-A through
 CFG-E) or provide complete values for L, H_kv, D, C, and P_bytes for any test that
specifies KV cache message sizes. Results are reported as a line graph with
prompt length on the X axis and TTFT (ms) on the Y axis, with separate lines for P50
and P99. The T_transfer component is shown as a shaded region.

## ITL Characterization and Tail Latency

**Objective:** To characterize inter-token latency distribution and identify
fabric-induced tail latency during the decode phase.

**Procedure:** Submit a single long-output request (e.g., 2048 output tokens)
and record the timestamp of each emitted token. Repeat under: (a) unloaded
fabric, (b) loaded fabric (50% of capacity), and (c) heavily loaded fabric (90%
of capacity plus concurrent EP dispatches).

**Measurement:** Report ITL at P50, P95, P99, P99.9, and maximum for each load
condition. Report the number of tokens exhibiting ITL > 100 ms (stall events).
The test generates at least 10,000 ITL samples per condition.

## End-to-End Latency Under Multi-Tenant Load

**Objective:** To measure inference latency when multiple models or model
instances share the same fabric.

**Procedure:** Deploy two or more model instances on separate worker pools
sharing the same fabric. Submit requests to both instances concurrently.

**Measurement:** Report per-instance TTFT P99, ITL P99, and the interference
penalty: (multi-tenant metric - single-tenant metric) / single-tenant metric *
100%.

## Latency Sensitivity to Fabric Congestion

**Objective:** To establish the relationship between fabric congestion level and
inference latency degradation.

**Procedure:** Inject controlled background traffic on the fabric at levels from
0% to 95% of capacity in 5% increments. At each level, submit inference requests
at a fixed rate and measure TTFT and ITL.

**Measurement:** Report TTFT P99 and ITL P99 as functions of background traffic
level. Identify the inflection point at which latency begins to degrade
significantly. Report the latency degradation factor at 50%, 75%, and 90%
background load.

# Test Category 7: Throughput Benchmarks

Inference throughput determines the cost-effectiveness of the serving
deployment.

## Aggregate Tokens Per Second

**Objective:** To determine the maximum sustained aggregate TPS achievable while
meeting latency SLOs.

**Procedure:** Increase the request arrival rate from 1 req/s to the point where
either TTFT P99 exceeds 500 ms or ITL P99 exceeds 50 ms. At each rate, measure
TPS_output, TPS_input, Goodput, and all latency KPIs.

**Measurement:** Report TPS_output, TPS_input, Goodput, TTFT P99, ITL P99, and
fabric utilization at the SLO-bounded throughput. Report the fabric utilization
at the SLO boundary as a key efficiency metric.

## Batch Size Scaling and Continuous Batching Impact

**Objective:** To measure the interaction between inference batch size,
continuous batching, and fabric transfer patterns.

**Procedure:** Configure the serving system with varying maximum batch sizes
(1, 4, 8, 16, 32, 64, 128). For each batch size, measure: (a) the number of
concurrent KV cache transfers, (b) aggregate fabric bandwidth consumed,
(c) TPS_output, and (d) TTFT P99. Enable continuous batching and repeat.

**Measurement:** Report TPS_output, TTFT P99, fabric bandwidth (GB/s), and peak
concurrent transfers for each batch size, with and without continuous batching.

## Goodput Under Preemption and Eviction

**Objective:** To measure the Goodput loss when fabric congestion forces KV
cache eviction or request preemption.

**Procedure:** Oversubscribe the system beyond the SLO-bounded throughput (at
110%, 125%, 150%, and 200% of the rate found in Test 11.1). Measure the rate of
KV cache evictions, request preemptions, and the resulting Goodput reduction.

**Measurement:** Report Goodput, eviction rate (evictions/s), preemption rate
(preemptions/s), wasted fabric bandwidth (GB/s), and the Goodput/TPS_output
ratio (efficiency).

# Test Category 8: Scale and Autoscaling {#test-cat8}

Inference serving clusters must scale dynamically to match request demand.

## Fabric Scale Limits for Inference Clusters

**Objective:** To determine the maximum inference cluster size supportable by
the DUT fabric while meeting performance requirements.

**Procedure:** Progressively scale the cluster from a minimal configuration
(e.g., 2 nodes, 16 GPUs) to the fabric's capacity (e.g., 1024 nodes, 8192
GPUs). At each scale point (following powers of two), measure KV cache transfer
throughput and latency, EP AllToAll dispatch latency, fabric control plane
convergence time, routing table size, and end-to-end TTFT and TPS.

**Measurement:** Report all KPIs at each scale point. Identify the scale limit
as the point where any KPI degrades by more than 10% from the
minimal-configuration baseline.

## Dynamic Autoscaling Response Time

**Objective:** To measure the time required for the fabric to accommodate
dynamic scaling of inference worker pools (adding/removing prefill or decode
workers).

**Procedure:** Starting from a stable serving state, trigger a scale-up event
(e.g., adding 4 decode nodes). Measure: (a) fabric convergence time, (b) time
from fabric convergence to first KV cache transfer on new nodes, (c) time to
reach steady-state throughput. Repeat for scale-down events.

**Measurement:** Report fabric convergence time (ms), first-transfer time (ms),
and time to steady-state (ms) for scale-up and scale-down events. Report any
packet loss or latency spikes during the scaling transition.

## Link Failure Convergence Impact on Serving

**Objective:** To measure the impact of fabric link failures on inference
serving performance and the convergence time to restore full service.

**Procedure:** During sustained inference serving at 80% of SLO-bounded
throughput, fail a single fabric link on: (a) a leaf-spine link carrying KV
cache traffic, (b) a spine-spine link, (c) a link on the decode worker's leaf
switch. Measure traffic disruption and recovery time. Repeat for dual link
failures.

**Measurement:** Report traffic disruption duration (ms), convergence time (ms),
TTFT degradation during convergence (ms above baseline P99), TPS reduction
during convergence (%), and time to full recovery (ms). The test is
repeated a minimum of 20 times per failure scenario.

# Test Category 9: Soak and Stability

Long-running inference serving deployments must maintain performance without
degradation over time.

## 24-Hour Sustained Inference Load

**Objective:** To verify that the fabric maintains performance under continuous
inference serving load for 24 hours.

**Procedure:** Configure the SUT-E at 80% of the SLO-bounded throughput
determined in Test 11.1. Run a continuous request stream for 24 hours with a
realistic prompt length distribution. Sample the following metrics every 15
minutes: TTFT P99, ITL P99, TPS_output, KV_xfer_latency P99, fabric link
utilization, switch CPU/memory usage, NIC counters (RDMA retransmissions, QP
errors), and PFC/ECN event counts.

**Measurement:** Report the trend of all sampled metrics over the 24-hour
period. The DUT is expected to exhibit zero NIC QP errors, zero routing flaps, and less than
1% variation in TTFT P99 over the test duration.

## KV Cache Memory Leak Detection

**Objective:** To detect memory leaks in the KV cache management subsystem that
may manifest as fabric performance degradation over time.

**Procedure:** Monitor GPU memory, CPU memory, NIC registered memory regions,
and RDMA memory region counts on all prefill and decode workers during the
24-hour soak test. Record the number of active KV cache pages, RDMA memory
registrations, and pinned memory at each sampling interval.

**Measurement:** Report the trend of each monitored metric. Flag any monotonic
increase as a potential leak. Report the maximum observed memory usage and the
usage at the end of the 24-hour period.

## Long-Running Serving Stability

**Objective:** To verify that fabric-dependent components remain stable under
continuous inference serving.

**Procedure:** During the 24-hour soak test, monitor: NIC QP state transitions,
switch buffer utilization trend, FEC error rate trend, BGP/OSPF adjacency
stability, and RDMA retransmission rate. At the 12-hour mark, trigger a
controlled perturbation (single link flap) and verify recovery.

**Measurement:** Report the count of any QP state transitions, maximum switch
buffer utilization, FEC error trend, adjacency flap count, and RDMA
retransmission count. Report the recovery time from the 12-hour link flap
perturbation.

# Reporting Format {#reporting}

All test results are reported following the conventions established in
{{!RFC2544}} Section 26. In addition, the following inference-specific reporting
elements apply:

* **System Configuration Report:** the report includes: model name and
  parameter count, parallelism strategy (TP, DP, EP, PP configuration for both
  prefill and decode pools), xPyD ratio, inference serving framework name and
  version, KV cache transfer library name and version, accelerator type and
  count, NIC type and firmware version, switch ASIC and software version, fabric
  topology, and link speeds.

* **Workload Characterization Report:** the report includes: prompt length
  distribution (mean, P50, P99, distribution type), output length distribution,
  request arrival rate and distribution, number of concurrent requests, and
  prefix sharing percentage.

* **Results Reporting:** for each test, results include: the specific test
  identifier (e.g., Test 5.1), the DUT/SUT configuration tested, the number of
  trials, all measured KPI values with confidence intervals, and any anomalies
  observed.

| Report Element | Format | Required? |
|----------------|--------|-----------|
| System Configuration | Structured table per above | Yes (required) |
| Workload Parameters | Structured table per above | Yes (required) |
| KPI Summary Table | Table with all measured KPIs | Yes (required) |
| Latency Distribution Plots | CDF or histogram per test section | Recommended |
| Throughput vs. Scale Graphs | Line chart per test section | Recommended |
| Fabric Health Indicators | Table per Section 4.4 | Yes (required) |
| Raw Data Appendix | Machine-readable format (CSV, JSON) | Optional |
{: #tab-reporting title="Reporting Format Requirements"}

# Security Considerations

This document defines benchmarking methodology for controlled laboratory environments and does not specify any protocol mechanism. It therefore introduces no new protocol-level security considerations beyond those of the underlying technologies it references. The considerations below follow the BMWG convention established in {{!RFC8238}} and align with the companion terminology document {{!TERMINOLOGY}}.

Benchmarking activities as described in this document are limited to technology characterization of AI inference serving fabrics using controlled stimuli in a laboratory environment, with dedicated address space and the constraints specified herein.

The benchmarking network topology will be an independent test setup and MUST NOT be connected to devices that may forward the test traffic into a production network or misroute traffic to the test management network. This isolation requirement is particularly important for AI fabric benchmarking because the lossless transport modes referenced in this document (PFC, DCQCN, CBFC) propagate congestion hop-by-hop and can extend the blast radius of a misconfigured test beyond the immediate DUT.

Benchmarking is performed on a "black-box" basis, relying solely on measurements observable external to the DUT as defined in {{!TERMINOLOGY}}.

Special capabilities SHOULD NOT exist in the DUT specifically for benchmarking purposes. Any implications for network security arising from the DUT SHOULD be identical in the lab and in production networks. In particular, RDMA memory-region permissions and KV cache telemetry exposure are properties of the deployed configuration, not of the benchmarking methodology, and SHOULD reflect production posture during testing.

Per {{!RFC6815}}, the tests defined herein MUST NOT be performed on production networks. The use of dedicated test IP address ranges per {{!RFC2544}} Appendix C (198.18.0.0/15 for IPv4; 2001:db8::/32 per {{?RFC3849}} for IPv6) is RECOMMENDED to prevent accidental interaction with production infrastructure.

The following considerations are specific to inference-serving benchmarking:

- **Synthetic prompt inputs:** The KV cache contains intermediate state derived from prompt content. Synthetic inputs SHOULD be used for all tests in this document so that no production prompt content is processed in the test environment. KV cache transfer benchmarks use payload patterns that do not reflect real user data.
- **One-sided RDMA write semantics:** KV cache transfers in this document use one-sided RDMA PUT operations to remote NIC memory. Such operations bypass remote-CPU authorization at the data path; generators that leak onto adjacent fabrics could write arbitrary bytes to remote NICs. Line-rate RDMA traffic generators MUST be confined to the test fabric.
- **PFC leakage:** PFC PAUSE frames generated under bursty KV cache or AllToAll incast conditions ({{test-cat4}}) that escape the test environment can hang adjacent production switches sharing the same priority class. Physical or VLAN-based isolation of the test fabric is required.
- **RDMA QP and PDC namespace isolation:** when RDMA/RoCEv2 traffic is used, the test environment SHOULD be isolated from production RDMA fabrics to prevent QP number space collisions or inadvertent PFC propagation. When UET traffic is used, the test environment MUST ensure that UDP port 4793 traffic does not leak to production networks and that PDC identifier spaces are isolated.
- **UET transport security sub-layer (TSS):** SHOULD NOT be enabled during performance benchmarking unless transport security overhead is explicitly being measured.

# IANA Considerations

This memo includes no request to IANA.

--- back

# KPI-to-Test Mapping Summary

The following table provides a cross-reference from each KPI defined in
{{kpi-framework}} to the test(s) in which it is measured.

| KPI | Primary Test(s) | DUT/SUT |
|-----|-----------------|---------|
| TTFT | 6.1, 6.2, 10.1, 10.3 | SUT-E |
| ITL | 10.2, 10.3, 10.4 | SUT-E |
| TPS_output | 6.2, 11.1, 11.2, 11.3 | SUT-E |
| TPS_input | 11.1 | SUT-E |
| Goodput | 11.1, 11.3 | SUT-E |
| KV_xfer_latency | 5.2, 5.3, 6.1, 6.4 | DUT-N, DUT-PD |
| KV_xfer_bandwidth | 5.1, 5.3, 5.4 | DUT-N, DUT-PD |
| EP_alltoall_latency | 7.1, 7.2, 7.3, 7.4 | DUT-F |
| EP_alltoall_bandwidth | 7.1, 7.3 | DUT-F |
| Fabric_FCT | 5.2, 5.3 | DUT-F |
| Buffer_utilization | 8.1, 8.2 | DUT-S |
| ECN_marking_rate | 8.1, 8.3 | DUT-S |
| PFC_frame_count | 8.2, 8.4 | DUT-S |
| Link_utilization | 5.3, 9.3, 12.1 | DUT-F |
| Packet_drop_rate | 8.1, 8.2, 12.3 | DUT-F |
| Request_Rate | 11.1 | SUT-E |
| Prefix Cache Hit Rate | 9.2 | SUT-E |
| JFI (Decode Worker) | 9.4 | SUT-E |
{: #tab-kpi-mapping title="KPI-to-Test Mapping"}

# Indicative Reference Values (Non-Normative) {#indicative-reference-values}

This appendix provides indicative reference values for the KPIs defined in {{kpi-framework}}, reflecting current industry observations for interactive inference workloads as of 2025-2026. These values are NON-NORMATIVE and do not constitute benchmarking acceptance criteria or performance requirements. Per the BMWG charter, the definition of acceptance criteria or performance requirements is explicitly outside the scope of this Working Group. Implementers may use these values as contextual references when interpreting results; they MUST NOT be used as pass/fail thresholds in vendor evaluations. Deployment-specific SLOs will vary by application, model architecture, and operator requirements.

| KPI | Indicative Reference (Interactive) |
|---|---|
| TTFT | < 500 ms P99 |
| ITL | < 50 ms P99 |
| TTFT_fabric | < 300 ms P99 |
| ITL_fabric | < 5 ms P99 |
| E2E_latency | varies by output length |
{: #tab-indicative-values title="Indicative Reference Values for Interactive Inference Serving (Non-Normative)"}

# Inference Serving Framework Capability Categories (Informational)

This appendix describes the inference serving framework capability categories
relevant to AI fabric benchmarking. This appendix is intended to guide
documentation of SUT-E configurations and is NOT normative. Implementers using
a Software Workload Emulator (SUT-E tests) document which of the
following capabilities their serving framework supports.

| Capability Category | Description | Relevance to Fabric Benchmarking |
|--------------------|-------------|----------------------------------|
| Disaggregated Prefill/Decode (PD) | Physical separation of prefill and decode execution across different accelerator pools | Determines whether DUT-PD topology tests apply (Section 6) |
| KV Cache Transfer Protocol | Protocol and library used for prefill-to-decode KV state transfer (one-sided PUT, two-sided SEND/RECV, GPU-initiated) | Determines RDMA verb types under test and applicable frame formats (Appendix C) |
| MoE Expert Parallelism (EP) Support | Distribution of MoE expert sub-networks across GPUs and AllToAll dispatch mode support | Determines whether MoE EP tests apply (Section 7) |
| Continuous Batching | Dynamic request admission to active inference batches | Affects request arrival rate distributions and load balancing tests in {{test-cat5}} |
| Prefix / KV Cache Sharing | Reuse of KV cache segments for requests with common prefixes | Determines applicability of the prefix cache hit rate test in {{test-cat5}} |
| RDMA Transport Support | Underlying transport(s) supported: RoCEv2, UET, or other | Documented in the test report; affects congestion management test interpretation in {{test-cat4}} |
| GPU-Initiated Networking (GIN) Support | Ability for GPU threads to directly initiate RDMA operations without CPU involvement | Affects RDMA primitive choice in MoE dispatch tests (Section 7) |
| Container Orchestration Integration | Native support for container-based deployment and horizontal scaling | Relevant for autoscaling tests in {{test-cat8}} |
| Maximum Reported Scale | Maximum cluster scale at which the framework has been validated | Documents applicability of fabric scale tests |
{: #tab-framework-caps title="Framework Capability Categories"}

NOTE: The specific framework name, version, and configuration are documented in all test reports. Results obtained with different frameworks are not directly comparable; framework identity is a required reporting parameter per {{reporting}}.

# KV Cache Transfer Frame Format {#kv-frame}

This appendix defines the reference frame format for KV cache transfer benchmarking over RoCEv2 using one-sided RDMA WRITE (PUT) operations. The frame format follows the standard RoCEv2 encapsulation defined in the InfiniBand Architecture Specification Volume 1 Annex A17 (RoCEv2).

| Offset | Field | Size | Value / Description |
|---|---|---|---|
| 00 | Ethernet Dst MAC | 6B | DUT next-hop MAC |
| 06 | Ethernet Src MAC | 6B | Test equipment MAC |
| 12 | EtherType / TPID | 2B | 0x0800 (IPv4) or 0x86DD (IPv6) when untagged; 0x8100 (TPID) when 802.1Q-tagged |
| 14 | 802.1Q Tag (optional) | 4B | When tagged: TCI (PCP for RDMA priority class, VID) followed by inner EtherType 0x0800 or 0x86DD. Omit this row when untagged and shift subsequent offsets back by 4B |
| 18 | IPv4 / IPv6 Header | 20B (IPv4) or 40B (IPv6) | DSCP=26 (AF31), ECN=ECT(0), Proto=17 (UDP) |
| 38 / 58 | UDP Header | 8B | DstPort=4791 (RoCEv2), SrcPort=entropy for ECMP, UDP Length, UDP Checksum |
| 46 / 66 | BTH (Base Transport Header) | 12B | OpCode=0x0A (RDMA WRITE Only) or 0x0B (RDMA WRITE with Immediate Data) at the last packet of a PUT-with-signal sequence; SE, M, Pad, TVer flags; PKey; Destination QP Number (24 bits); A flag; PSN (24 bits) |
| 58 / 78 | RETH (RDMA Extended Transport Header) | 16B | Virtual Address (64 bits), R_Key (32 bits), DMA Length (32 bits). The DMA Length indicates the size of the KV cache block transferred by this WRITE operation |
| 74 / 94 | KV Cache Payload | variable, up to MTU | Key/value attention state data |
| var | ICRC | 4B | Invariant CRC |
| var+4 | FCS | 4B | Ethernet Frame Check Sequence |
{: #tab-kv-frame title="RoCEv2 KV Cache Transfer Frame (One-Sided RDMA WRITE)"}

Notes:

- The UDP Source Port uses entropy-based values for ECMP load distribution across fabric paths.
- The RETH carries the remote virtual address, remote key, and DMA length for the one-sided WRITE operation. For KV cache transfers, the DMA Length field indicates the size of the KV cache block being transferred.
- Typical MTU for RoCEv2 deployments is 4096 bytes; larger KV cache blocks (e.g., 64 KB pages) are segmented into multiple packets by the NIC. The first packet of a segmented WRITE carries OpCode 0x06 (RDMA WRITE First) and a RETH; intermediate packets carry OpCode 0x07 (RDMA WRITE Middle); the last packet carries OpCode 0x08 (RDMA WRITE Last) or 0x0B (RDMA WRITE Last with Immediate Data) for PUT-with-signal completion signalling.
- For UET-based KV cache transfers, the frame format defined in {{!TRAINING-BENCH}} Appendix D ("UET Frame Format") applies; the DUT IP port is 4793 and the transport service indicator selects between ROD and RUD per test.

# MoE AllToAll Communication Pattern

This appendix describes the AllToAll communication pattern used for MoE expert
parallelism dispatch and its fabric-level traffic characteristics. In a
Mixture-of-Experts model with M total experts distributed across N GPUs (each
GPU holds M/N experts), a single MoE layer forward pass generates an AllToAll
communication pattern where each GPU sends a variable-size payload to every
other GPU.

| Parameter | Normal Dispatch (Prefill) | Low-Latency Dispatch (Decode) |
|-----------|--------------------------|-------------------------------|
| Batch Size | 128 - 512 tokens | 1 - 16 tokens |
| Payload per GPU pair | Variable (depends on routing) | Fixed (padded to max) |
| Shape Compatibility | Dynamic (symbolic) | Static (graph-capturable) |
| QP Parallelism | 24 QPs per connection | 8 - 16 QPs per connection |
| RDMA Primitive | Two-sided SEND/RECV or one-sided PUT | One-sided PUT (GPU-direct RDMA, GIN) |
| GPU Initiation | CPU-initiated or GIN | GIN (device-initiated, GPU-to-NIC direct) |
| Typical per-dispatch size | 1 - 10 MB aggregate | 10 KB - 1 MB aggregate |
| Dispatch Frequency | Once per MoE layer (prefill) | Once per MoE layer per token (decode) |
| Latency Target | < 1 ms per dispatch | < 200 us per dispatch |
{: #tab-moe-dispatch title="MoE Dispatch Traffic Characteristics by Mode"}

For a representative dense MoE configuration (M3: E=256, k=2, H_model=7168, EP=96 across 12 nodes of 8 accelerators, BF16; representative of a large publicly described MoE-class architecture), the inter-node traffic per MoE layer dispatch using T_dispatch = (B * k * H_model * 2) / N is approximately

* Normal Dispatch (prefill, batch=256): 256 \* 2 \* 7168 \* 2 bytes / 96 GPUs
  = ~76 KB per GPU pair, ~870 MB aggregate across all pairs.
* Low-Latency Dispatch (decode, batch=8): 8 \* 2 \* 7168 \* 2 bytes / 96 GPUs
  = ~2.4 KB per GPU pair, ~27 MB aggregate.

With 61 MoE layers (representative of a publicly described large-scale MoE architecture) and a decode iteration time target of ~30 ms, the decode
phase requires 61 AllToAll dispatches within 30 ms, yielding ~2,000 dispatches
per second per decode step, consuming approximately 54 GB/s aggregate inter-node
bandwidth for the Low-Latency Dispatch path.

# Model Architecture Parameters

This appendix provides a sample calculation for the S_KV formula already provided.
It's based on a '70B parameter model at FP16 with 4K context' model

~~~
Parameter                      Symbol   Value   Source
Transformer layers             L        80      Published architecture
KV attention heads (GQA-8)     H_kv     8       H_total=64 / GQA_ratio=8
Per-head dimension             D        128     model_dim(8192) / H_total(64)
Context length                 C        4,096   Given
Precision                      P_bytes  2       FP16 = 2 bytes/element

Step-by-Step Calculation

S_KV = 2  ×  L  ×  H_kv  ×   D   ×    C    × P_bytes

 = 2  ×  80  ×   8   ×  128  ×  4,096  ×    2

Step 1:  2  × 80           =         160   (K + V tensors × layers)

Step 2:  160 × 8           =       1,280   (× KV heads)

Step 3:  1,280 × 128       =     163,840   (× head dimension)

Step 4:  163,840 × 4,096   = 671,088,640   (× context tokens)

Step 5:  671,088,640 × 2   = 1,342,177,280 bytes
~~~

{:numbered="false"}

# Acknowledgments
{:numbered="false"}

This work has benefited from the discussions that occurred during the joint IPPM and BMWG meeting and on the BMWG mailing list. Thanks to Carsten Rossenhoevel, Mohamed Boucadair, and Sowjanya Reddy for valuable review and comments.
