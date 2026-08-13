---
title: "Wrap Up – GSoC '26"
layout: default
---

# Dynamic DICOM Endpoints with Diomede - Final Report

Over the summer of 2026, I worked on **Diomede**, a load‑aware orchestration system for dynamic DICOM endpoint selection. This page serves as my official GSoC work product summary.

Static DICOM routing still impacts many real clinical environments, where scanners send studies to a single hard‑coded endpoint. That model works only as long as the destination node is healthy, uncongested, and not running out of disk space. When it isn’t, radiology workflows stall. The goal of this project is to replace those static assumptions with a dynamic, telemetry‑driven routing layer that preserves reliability under realistic failure conditions.

---

## Project overview

**Project:** Towards Dynamic DICOM Endpoints with Diomede  
**Areas:** Distributed systems, medical imaging, DICOM, cloud infrastructure  

At a high level, Diomede introduces a self‑healing, load‑aware orchestration layer on top of the open‑source Orthanc PACS server. Instead of pushing every study to a single fixed endpoint, scanners (via an edge agent) talk to an orchestrator that continuously monitors multiple regional Orthanc nodes and chooses the most suitable destination for each study in real time. 

Key design goals:

- **Avoid static configuration:** No more hard-coding a single IP + AE Title for every scanner.
- **Use live telemetry:** Make routing decisions using queue depth, disk usage, and network latency.
- **Stay compatible with existing DICOM devices:** Use an edge Orthanc + forwarder so legacy scanners don’t need changes.
- **Preserve clinical reliability:** Maintain delivery success under node overload, failures, and recoveries.
---

## Architecture and components

Diomede is built as a system of cooperating services; the main components include: 

- **Regional Orthanc nodes**  
  Cloud‑hosted Orthanc instances (e.g., in us‑east1, eu‑west1, asia‑northeast1, af‑south1) act as regional PACS endpoints. They expose standard DICOM services plus a REST API for telemetry and ingesting instances via HTTP. 

- **Telemetry daemon**  
  A background process that polls each Orthanc node’s `/statistics`, `/jobs?expand`, and `/system` endpoints to collect metrics such as queue depth, disk usage, and configuration parameters. It writes this data into Redis with a time‑to‑live (TTL), so stale node metrics expire automatically. 

- **FastAPI orchestrator**  
  Maintains the live node registry in Redis and implements an HTTP endpoint (`/get-best-node`) that evaluates a scoring function over all healthy nodes to choose the best routing destination for each incoming study. 

- **Edge agent (Orthanc + forwarder)**  
  A local Orthanc instance acts as a DICOM receiver for existing scanners. A Python forwarder monitors new instances, retrieves image bytes, asks the orchestrator for a destination, pushes the study to the chosen cloud Orthanc node, and then removes the local copy to avoid disk exhaustion. This preserves compatibility with legacy DICOM devices while centralizing routing logic in the orchestrator.

- **Simulation scanner**  
  A Python `asyncio` + `httpx` harness that generates synthetic DICOM traffic and failure scenarios without requiring physical imaging equipment. I used this extensively for load and failover testing. 

All services are containerized using Docker and Docker Compose for reproducible deployments across regions, and Redis provides lightweight in‑memory state for node telemetry with automatic cleanup via TTL. 

---

## Routing model and scoring function

The orchestration logic is deliberately simple and transparent. The telemetry daemon periodically queries each node to estimate:

- **Queue depth (q):** Derived from Orthanc’s job state (e.g., number of running/pending jobs).  
- **Free disk fraction (free/total):** Derived from node statistics such as `TotalDiskSizeMB` and `TotalUncompressedSizeMB`.  
- **Network round‑trip time (RTT):** Measured via HTTP probes against `/system`. 

Each node $n$ is scored as:

$$S(n) = w_q \cdot \frac{1}{q + 1} + w_d \cdot \frac{\text{free}}{\text{total}} + w_r \cdot \frac{1}{\frac{\text{RTT}}{\text{RTT}_{\text{ref}}} + 1}$$

where $\text{RTT}_{\text{ref}} = 100\text{ ms}$, and the default weights are:  
$w_q = 0.5$, $w_r = 0.35$, and $w_d = 0.15$.

Intuitively:

- Low queue depth and low latency are favored over merely empty disks.
- The highest‑scoring node at the time of decision becomes the routing target.
- Redis TTL acts as a dead‑node detector: if telemetry stops arriving, the node quietly drops out of the candidate pool. 

The scoring model is **pluggable**. In practice, this means an operator can prioritize latency, storage safety, or queue avoidance differently by adjusting weights or swapping the function, without changing the DICOM edge behavior. 

---

## What I implemented during GSoC

During GSoC 2026, I focused on delivering an end‑to‑end, measurable system rather than simply a theoretical design. Concretely, I implemented: 

- **Orthanc‑based edge and regional topology**
  - Containerized deployments of Orthanc for regional PACS nodes and the edge.  
  - Configuration for persistent storage and secure DICOM transport (including TLS). 

- **Telemetry collection and Redis integration**
  - A polling daemon that periodically calls `/statistics`, `/jobs?expand`, and `/system` on each node.  
  - A Redis schema with TTL so stale node metrics expire automatically and dead nodes are removed without manual cleanup. 

- **FastAPI orchestrator**
  - `/get-best-node` endpoint that reads the current node registry, applies the scoring function, and returns the best candidate as a simple JSON payload.  
  - Health‑check logic to enforce a fixed monitoring interval (≈10 seconds) and a predictable upper bound on failover detection time. 

- **Edge forwarding workflow**
  - A Python forwarder that watches the local Orthanc for new instances, fetches image bytes, queries the orchestrator, forwards the study to the chosen cloud node, and deletes the local copy after success.  
  - PHI‑safe logging assumptions so that routing decisions and metrics can be audited without exposing sensitive data. 

- **Simulation and benchmarking harness**
  - A synthetic scanner implementation using `asyncio` and `httpx` to generate realistic burst and concurrent traffic.  
  - Scripts to automate routing latency measurements, routing correctness trials, failover tests, and end‑to‑end transfer benchmarks. 

---

## Evaluation: performance and reliability

I evaluated Diomede from four perspectives: routing decision latency, routing correctness, failover behavior, and end‑to‑end transfer success.

### Routing decision latency

To understand overhead introduced by the orchestration layer, I measured the `/get-best-node` endpoint with both single requests and sustained concurrent load.

- A **single routing request** completes in roughly **2 ms**.  
- Under a **60‑second run at 100 concurrent clients**, the orchestrator:  
  - Processed **84,334 requests**.  
  - Sustained **≈1,256 requests per second**.  
  - Achieved **p99 latency of 25 ms**.  
  - Recorded **zero failures**.

These results indicate the routing step is lightweight and does not become a bottleneck compared to typical DICOM transfer times. 

### Routing correctness

Speed is only useful if the system chooses the right node. I tested routing correctness by rotating synthetic queue‑depth levels across four Orthanc nodes while keeping network latency and disk space fixed, then issuing repeated routing requests after telemetry updates. 

- Across **35 load permutations**, the orchestrator:  
  - Selected a node from the expected **lowest‑load set in all 35 trials**.  
  - Adapted its selection whenever the least‑loaded node changed.

This confirms that routing decisions are sensitive to live telemetry, not a static preference for a single default node.

### Failover detection and recovery

I evaluated failover behavior by taking regional nodes offline under two failure modes: process stops and network partitions. For each region and failure mode, I measured the time to detect failure and the time to detect recovery after the node returned. 

- Across **40 trials** (4 regions × 2 failure modes × 5 repeats):  
  - Detection occurred in **≈10 seconds** on average.  
  - Recovery was recognized in a **similar ≈10‑second window** after restoration.  
  - Results were consistent across regions and failure types.

This behavior reflects the orchestrator’s fixed 10‑second health‑check interval and provides a predictable upper bound on how long a failed node may remain in the routing pool. 

### End‑to‑end transfer success

Finally, I validated the full forwarding pipeline from edge to cloud, including behavior under mid‑transfer failures. 

- Under baseline conditions:
  - **1,000/1,000 studies** were successfully delivered end‑to‑end.  
- With a **15‑second mid‑transfer node outage**:
  - **1,000/1,000 studies** were still delivered successfully.  
  - Both scenarios exceed a **99.5% success threshold**.

This shows that the system not only detects failures but continues to preserve delivery reliability during them, masking short‑lived outages from the perspective of the overall DICOM workflow.

---

## Code and merge history

Below is an outline of where my work lives upstream. 

### Core components

- **Orchestrator and telemetry code**
  - **Path**: [`src/orchestrator/`](https://github.com/KathiraveluLab/Diomede/tree/main/src/orchestrator)
  - **Key files**:
    - `main.py` – FastAPI app and routing logic.
    - `daemon.py` – Telemetry daemon process.
    - `scorer.py` & `weighted_scorer.py` – Scoring logic for orchestration.

- **Edge agent and forwarder**
  - **Path**: [`src/edge/`](https://github.com/KathiraveluLab/Diomede/tree/main/src/edge)
  - **Key files**:
    - `forwarder.py` – Python process that watches for new instances and forwards them.
    - `orthanc_source.py` – Orthanc integration module.
    - `transport.py` – Network transport handling.

- **Simulation and benchmarking harness**
  - **Path**: [`src/simulator/`](https://github.com/KathiraveluLab/Diomede/tree/main/src/simulator)
  - **Key files**:
    - `generate_dicom.py` – Synthetic DICOM/scanner data generator.
    - `send_dicom_native.py` & `send_dicom_rest.py` – Sending scripts for DICOM benchmarking across native and REST protocols.


### Selected pull requests and merge history

- **Implement dynamic routing endpoint and scoring model** – PR [#184](https://github.com/KathiraveluLab/Diomede/pull/184) (`feat: orchestrator api endpoints`) & PR [#183](https://github.com/KathiraveluLab/Diomede/pull/183) (`feat: add scorer`)
- **Add telemetry daemon with Redis TTL‑based node health tracking** – PR [#182](https://github.com/KathiraveluLab/Diomede/pull/182) (`feat: integrate orchestrator daemon`)
- **Edge forwarder integration with Orthanc and orchestrator** – PR [#185](https://github.com/KathiraveluLab/Diomede/pull/185) (`feat: edge agent forwarder`)
- **Load testing and failover harness** – PR [#190](https://github.com/KathiraveluLab/Diomede/pull/190) (`feat: add Locust load tests`), PR [#187](https://github.com/KathiraveluLab/Diomede/pull/187) (`feat: testing improvements`), & PR [#181](https://github.com/KathiraveluLab/Diomede/pull/181) (`Feat: simulate DICOM file creation and transfer`)
- **Performance & memory optimizations** – PR [#192](https://github.com/KathiraveluLab/Diomede/pull/192) (`enhancement: optimize memory footprint`)
- **Infrastructure, TLS, & CI security** – PR [#180](https://github.com/KathiraveluLab/Diomede/pull/180) (`feat: enable TLS`) & PR [#177](https://github.com/KathiraveluLab/Diomede/pull/177) (`enhance: CI workflow guards and change mypy 2.0`)

For a complete list of changes, see:

- **All merged PRs authored by me:** [`is:pr is:merged author:jess-tech-lab`](https://github.com/KathiraveluLab/Diomede/pulls?q=is%3Apr+is%3Amerged+author%3Ajess-tech-lab)
- **Full commit history:** [`commits?author=jess-tech-lab`](https://github.com/KathiraveluLab/Diomede/commits/main?author=jess-tech-lab)

---

## What’s left and future work

The current prototype is production‑oriented, but not enterprise-ready. If I were to continue this work beyond GSoC, my next steps would be: 

- **Policy‑aware routing:** Incorporate additional factors such as modality priority, study urgency, and patient or site‑specific policies into the scoring function instead of relying solely on queue depth, disk, and RTT. 

- **Reduce failover latency:** Experiment with shorter health‑check intervals or push‑based heartbeats to detect node failures faster without overloading nodes.

- **Real‑world workload validation:** Test the system against real scanner traffic and larger‑scale deployments to validate behavior beyond synthetic loads. 

- **Operational tooling and observability:** Extend metrics and dashboards to help operators understand congestion, failure patterns, and routing decisions over time. 

---

## Lessons learned

This project was as much about designing a robust pipeline as it was about writing code. A few key takeaways for me: 

- **Simple scoring beats complicated rules.**  
  A clear, tunable scoring function made debugging and tuning significantly easier than a black‑box approach. 

- **Telemetry and TTLs are powerful together.**  
Using Redis TTL as a way to detect inactive nodes turned out to be a clean way to handle nodes joining and leaving without adding complicated logic.

- **Evaluate the system, not just the components.**  
  Measuring end‑to‑end transfer success under failures gave a more honest view of reliability than looking at routing latency or correctness in isolation. 

- **Compatibility matters in healthcare.**  
  The edge Orthanc + forwarder architecture allows for improved routing without requiring any changes to existing scanners, which is essential in real deployments.

---

## Acknowledgments

I’m incredibly grateful to my mentors, Dr. Pradeeban Kathiravelu & Ananth Reddy, for their invaluable guidance, code reviews, and continuous support throughout this project. Their feedback helped refine both the architecture and the evaluation approach, and made this a much more rigorous and rewarding experience.

If you’d like to learn more or discuss the project, feel free to reach out to me at **j658huan@uwaterloo.ca**.

