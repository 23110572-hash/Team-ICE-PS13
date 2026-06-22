# Air-Gapped Predictive Copilot for Secure MPLS Operations

> **Bharatiya Antariksh Hackathon 2026 — Problem Statement 13 — Cybersecurity / Secure Network Operations**

We build an offline, autonomous, predictive NOC copilot for **SD-WAN-over-MPLS** networks in secure, air-gapped enterprise and government deployments. The system forecasts faults — congestion build-up, BGP/OSPF routing instability, tunnel health degradation, controller policy drift — *before* they impact service, explains each prediction in plain English using a locally hosted LLM grounded in the operator's own runbooks and topology, and recommends the corrective action. It runs entirely inside the air-gap with zero outbound network dependency.

---

## Table of Contents

- [Overview](#overview)
- [The Problem](#the-problem)
- [What We Build](#what-we-build)
- [How It Works](#how-it-works)
- [Technology](#technology)
- [Target Hardware](#target-hardware)
- [Validation Scenarios](#validation-scenarios)
- [Air-Gap Approach](#air-gap-approach)
- [Build Timeline](#build-timeline)
- [Team](#team)
- [Documentation](#documentation)
- [License](#license)

---

## Overview

Modern enterprise and government networks run SD-WAN overlays on top of MPLS underlays to connect branches, hubs, datacentres, and cloud edges. Our copilot moves NOC operations from reactive to predictive and answers three operational questions in real time:

- **What is likely to fail next, and when?** — forecasting with time-to-impact.
- **Why is risk elevated, and which signals contributed?** — grounded explanation.
- **What corrective action should be taken before SLA or security impact?** — recommended remediation.

We forecast degradation with enough lead time for the NOC to intervene, and the copilot communicates that forecast in operator-ready language, fully offline.

### The Three Operational Questions

The brief frames success around three questions. Each maps to a concrete system output:

| Question | System output | Component |
|---|---|---|
| **Q1 — What will fail next, and when?** | Fault probability + time-to-impact | Predictive Analytics Engine (Layer 4) |
| **Q2 — Why is risk elevated, which signals contributed?** | Grounded explanation + root-cause hypothesis, with citations | Offline LLM Copilot (Layer 2) |
| **Q3 — What action to take before impact?** | Prioritised, cited playbook action sequence | NOC Workflow Automation (Layer 3) |

A NOC engineer interacts with all of this through a natural-language chatbox. They can ask plain questions — "why is tunnel T3 degrading?", "what's the risk on the Branch-2 link?", "what should I do about the BGP flapping on PE2?" — and get a direct, accurate answer grounded in the operator's own runbooks and topology, with citations. The quality and accuracy of these answers is the primary measure of the copilot, so grounding and correctness are treated as first-class requirements, not the breadth of what the operator must type.

## The Problem

Secure networks face two compounding gaps:

- **Reactive detection.** Threshold-based alerts fire only after a breach, leaving no time to intervene.
- **Air-gap constraints.** Regulated and government environments prohibit cloud-connected AI inference, so operators have no intelligent guidance exactly where security matters most.

A problem in the MPLS underlay — a flapping LSP, a congested hub-spoke link, a drifting controller policy — often surfaces as degraded overlay performance after users already feel it. We close that visibility gap by detecting the precursor conditions that lead to failure and surfacing them early.

## What We Build

Five components deliver the four objectives of the brief:

1. **Simulated SD-WAN-over-MPLS environment** — a reproducible multi-site topology (branch, hub, datacentre) with CE/PE/P device roles, MPLS forwarding, VPN segmentation, IPSec overlay tunnels, BGP/OSPF routing, QoS, application traffic, and configurable fault injection.
2. **Predictive fault analytics engine** — models that detect precursor conditions and produce per-entity fault probability and time-to-impact across congestion forecasting, routing instability, and tunnel health degradation.
3. **Offline LLM NOC copilot** — a self-hosted quantized LLM with retrieval over internal artifacts (topology maps, runbooks, past incidents) that explains the correlated incident from the workflow layer (with its localised root cause), produces structured, grounded responses, and answers free-form natural-language questions. An engineer asks plain questions in a chatbox ("why is tunnel T3 degrading?") and gets a cited, accurate answer without needing to know how to query the network.
4. **Integrated NOC workflow** — topology-aware event correlation, confidence-scored alert prioritisation, playbook suggestion, and operator-ready incident summaries.
5. **Operator dashboard** — live topology, prioritised alerts with lead time, copilot explanations with cited artifacts, and recommended actions.

## How It Works

The platform is organised in six layers inside a single air-gapped boundary. Data flows up: the simulation emits telemetry, the pipeline normalises and stores it, the engine forecasts faults with lead time, the workflow layer correlates them into a single incident and localises the root cause, the copilot explains that incident with grounded retrieval, and the dashboard presents it to the operator.

```
┌─────────────────────────────────────────────────────────────────────┐
│              Layer 1 — NOC Operator Interface (Streamlit)           │
│        Live topology · prioritised alerts · NL query · citations    │
├─────────────────────────────────────────────────────────────────────┤
│                    Layer 2 — Offline LLM Copilot                    │
│   Internal artifacts → MiniLM → FAISS · incident → retriever →      │
│   Llama-3-8B (Ollama) → structured response with citation gate      │
├─────────────────────────────────────────────────────────────────────┤
│                Layer 3 — NOC Workflow Automation                    │
│   NetworkX correlation + root-cause · prioritisation · playbooks    │
├─────────────────────────────────────────────────────────────────────┤
│                 Layer 4 — Predictive Analytics Engine               │
│   Prophet + LSTM forecaster + Isolation Forest + graph anomaly      │
│        → XGBoost ensemble → fault probability + time-to-impact      │
├─────────────────────────────────────────────────────────────────────┤
│                    Layer 5 — Ingestion Pipeline                     │
│  SNMP + syslog (BGP/OSPF) + NetFlow/IPFIX + controller telemetry    │
│                       → normaliser → DuckDB                         │
├─────────────────────────────────────────────────────────────────────┤
│                  Layer 6 — SD-WAN over MPLS Simulation              │
│  Containerlab + FRR · branch/hub/DC · CE/PE/P · IPSec overlay ·     │
│            MPLS underlay · BGP/OSPF · fault injection               │
└─────────────────────────────────────────────────────────────────────┘
```

The full component reference and data-flow description are in [ARCHITECTURE.md](ARCHITECTURE.md) and [DOCUMENTATION.md](DOCUMENTATION.md).

## Confidence Scoring

The confidence score in every copilot response comes from the predictive engine, not the language model. The XGBoost ensemble outputs a probability of fault within each horizon (5, 15, and 30 minutes). That probability is calibrated on a held-out validation set — using isotonic regression — so the number is meaningful: a reported 0.8 means that, historically, about 80% of predictions at that level were real faults. The score is reinforced when the independent detectors agree: the LSTM forecast crossing its threshold, the Isolation Forest precursor-anomaly score, and the graph-based anomaly score all pointing the same way raise confidence, while disagreement lowers it.

The copilot passes this calibrated number through to the operator and never invents its own confidence. Combined with the citation gate, this keeps both the explanation and its confidence grounded in measured model behaviour rather than the language model's self-assessment.

## Technology

Every component runs offline.

| Layer | Component | Technology |
|---|---|---|
| Simulation | Network simulation | Containerlab + FRRouting |
| Simulation | SD-WAN overlay | strongSwan IPSec tunnels (CE, kernel XFRM) + closed-loop controller (vtysh + tc) |
| Simulation | Traffic + faults | iperf3 + tc/netem injection |
| Ingestion | SNMP | snmp_exporter + snmptrapd (interface counters) |
| Ingestion | Node metrics | tc qdisc stats + active probes (queue, latency, jitter) |
| Ingestion | Syslog + routing events | rsyslog → JSON |
| Ingestion | NetFlow / IPFIX | softflowd (per-interface IPFIX export) → GoFlow2 |
| Ingestion | Controller telemetry | controller state + config poller (drift diff) |
| Ingestion | Historical store | DuckDB |
| Ingestion | Live UI buffer | in-memory recent-window (Pandas) |
| Prediction | Time-series decomposition | Prophet (trend/residual extraction) |
| Prediction | Forecasting | LSTM forecaster (PyTorch) |
| Prediction | Anomaly detection | Isolation Forest (per-entity precursors) |
| Prediction | Graph-based anomaly detection | Neighbour-relative deviation over topology → Isolation Forest |
| Prediction | Lead-time classification | XGBoost ensemble |
| Workflow | Event correlation | NetworkX graph correlator |
| RAG | Embeddings | all-MiniLM-L6-v2 |
| RAG | Vector store | FAISS |
| RAG | LLM runtime | Ollama |
| RAG | LLM model | Llama-3-8B-Instruct (Q4_K_M GGUF) |
| RAG | Orchestration | LangChain (offline) |
| UI | Dashboard | Streamlit |
| Packaging | Runtime | Docker Compose |

## Target Hardware

The whole stack runs on a single laptop, which is our demo configuration.

| Resource | Specification |
|---|---|
| CPU | 8+ cores |
| RAM | 32 GB |
| Disk | 60 GB |
| GPU | Optional; helps LLM latency |

A 4-bit quantized Llama-3-8B uses about 4–5 GB of RAM at inference. Alongside it the host runs Containerlab (~9 FRR containers), the traffic generator, the ingestion pipeline, DuckDB, and the Streamlit dashboard. For a responsive live demo we run in demo mode: models are pre-trained and loaded, the FAISS index is pre-built, Ollama is pre-warmed, and scenarios are replayed from the labelled dataset. Heavy model training is done ahead of time.

## Validation Scenarios

Four scenarios form the validation set. Each is a reproducible script that ends in: fault injected → precursor detected with lead time → copilot explains with citation → playbook suggested.

| # | Scenario | Injection method | Key precursor signals |
|---|---|---|---|
| 1 | Progressive hub-spoke congestion | tc qdisc ramp + staged iperf3 load | Utilisation slope, queue-depth slope, drops, jitter trend, RTT slope |
| 2 | BGP route flap → reroute cascade | Cyclic BGP prefix withdraw/clear | BGP adjacency changes, advertise/withdraw rate, convergence time, path asymmetry |
| 3 | Intermittent MPLS underlay + tunnel degradation | Flap core link / RSVP-TE path; degrade IPSec | LSP reroutes, Fast Reroute switchover, IPSec loss progression, jitter, rekey anomalies |
| 4 | Controller misconfiguration → policy drift | Push inconsistent SD-WAN policy | Controller telemetry deltas, policy-vs-device mismatch, unexpected path changes |

Supporting precursor classes (LDP session flap, OSPF instability, link/node failure) and an adversarial scenario (rogue-CE BGP route injection inside the VPN) strengthen the labelled training set.

## Air-Gap Approach

The system is designed to prove zero outbound dependency at runtime:

1. Every dependency is staged offline before the finale — Docker images, Python wheels, Ollama model weights, the FAISS index, and the labelled training data — and loaded from a golden USB.
2. Ollama's update checker and telemetry are disabled in the daemon configuration so it never attempts to phone home for model updates.
3. The demo runs with the network interface down and outbound traffic blocked, allowing loopback only.
4. A `verify_airgap.sh` script captures egress with tcpdump and ss during a full inference cycle to demonstrate zero outbound traffic.

## Build Timeline

| Date | Milestone |
|---|---|
| 10 June – 1 July 2026 | Registration and idea submission |
| 20 July 2026 | Shortlist announced |
| 21 July 2026 | Induction session |
| 6–7 August 2026 | 30-hour Grand Finale |

The pre-finale build order:

1. Stand up the SD-WAN-over-MPLS Containerlab topology with FRR, BGP, OSPF, LDP, RSVP-TE, IPSec overlay, and the closed-loop controller (vtysh + tc).
2. Wire the four-source ingestion pipeline (SNMP, syslog, NetFlow/IPFIX, controller telemetry) into DuckDB.
3. Implement the four scenarios plus supporting precursor classes and produce a labelled dataset.
4. Engineer features and train the models — Prophet decomposition, the LSTM forecaster, Isolation Forest, the graph-based anomaly detector, and the XGBoost ensemble; evaluate on temporal hold-outs and report lead time per scenario.
5. Author the runbook corpus, build the FAISS index, wire Ollama and LangChain, and design the structured-response prompt with citation enforcement.
6. Build the Streamlit dashboard, graph-based event correlation, alert prioritisation, and playbook sequencing.
7. Package for the air-gap and verify offline on a clean machine.

## Team

A four-member team:

| Role | Responsibilities |
|---|---|
| Network / DevOps | Containerlab topology, FRR configs, ingestion pipeline, fault injection, Docker packaging |
| ML | Feature engineering, forecasting, anomaly detection, ensemble, event correlation, evaluation |
| LLM / RAG | Runbook authoring, FAISS index, LangChain pipeline, Ollama setup, prompt design |
| Integration / UI | Streamlit dashboard, alert-to-copilot wiring, demo flow, documentation |

## Documentation

- [ARCHITECTURE.md](ARCHITECTURE.md) — layered architecture, component reference, and data flow.
- [DOCUMENTATION.md](DOCUMENTATION.md) — full build reference: glossary, deliverables, dataset and simulation strategy, step-by-step pipeline, roadmap, and evaluation mapping.
- [Problem Statement .md](Problem%20Statement%20.md) — the official problem statement and evaluation criteria.

## License

Original project code is Apache-2.0. The bundled open-weight LLM (Llama-3-8B-Instruct) retains its upstream Meta Llama 3 Community Licence.
