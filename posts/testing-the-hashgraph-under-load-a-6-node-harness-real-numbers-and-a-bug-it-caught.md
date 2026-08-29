---
title: "Testing the hashgraph under load: a 6-node harness, real numbers, and a
  bug it caught"
---

This is a follow-up to [the previous post](#) on building JKaIN, my from-scratch hashgraph consensus engine in Rust ([repo](https://github.com/Jashk120/JX/)). That post covered the architecture and how it got built. This one is about what I did with it today: actually put it under load and see what happens.

## Building a real test harness

Up to this point I'd been testing correctness in isolation — unit tests, small local clusters, manually checking that checkpoints matched across nodes. Today I built a proper Python test harness to actually stress the thing: drive real `jkaind` binaries across a 6-node cluster, track them by PID over a control socket, and measure real gossip convergence, chaos recovery, and finality/throughput under load — not simulated, the actual compiled node.

The harness has three pieces worth naming: a `ClusterManager` that spins up and tears down the actual node processes, a `ControlClient` that talks to each node over its control socket to submit transactions and query state, and a `LatencyMesh` that measures pairwise gossip timing across the cluster rather than trusting any single node's view. 16 tests came out of it in total, tagged by what they exercise — smoke, gossip convergence, chaos (killing nodes mid-run), and finality/TPS.

## The numbers

Run on a 6-node direct LAN setup (`sync_interval 25ms`, my desktop — Ryzen 5 7600X, no GPU):

**Lowest latency (isolated, concurrency=1)**

| metric | value |
|---|---:|
| p50 decided | 0.536s |
| p95 decided | 1.14s |
| p99 decided | 1.82s |
| mean decided | 0.72s |
| best single tx | **0.210s** |

**Highest throughput (sustained, round-robin across all 6 nodes)**

| total tx | concurrency | submit TPS | decided TPS | submit dur | decided dur |
|---:|---:|---:|---:|---:|---:|
| 500 | 20 | 10,512 | 471 | 0.05s | 1.06s |
| 1,000 | 30 | 8,519 | 890 | 0.12s | 1.12s |
| 1,500 | 30 | 9,934 | 1,298 | 0.15s | 1.16s |
| 2,000 | 40 | 10,479 | 1,673 | 0.19s | 1.20s |
| **2,500** | **40** | **10,474** | **2,002** | **0.24s** | **1.25s** |

Peak: **2,002 TPS decided** (10,474 TPS submit) at 2,500 tx / concurrency 40.

The gap between "submit" and "decided" is the honest number here — submit TPS is just how fast you can throw transactions at the network, decided TPS is how fast the network actually reaches finalized consensus order on them. Anyone can post a big submit number. Decided is the one that matters.

## The tradeoff that actually matters: gossip interval

The 25ms sync interval above is the harness setting, not what I'd run in production — it's fast specifically so it stresses the graph and hits thermal limits on my hardware (CPU-bound gossip and signature verification will heat a desktop up fast without server cooling). The real lever is `sync_interval_ms`, and it's a straight tradeoff between latency, throughput, and heat:

| sync_interval | latency p50 | decided TPS | submit TPS | thermal |
|---:|---:|---:|---:|---:|
| 25ms (harness, stress) | 0.54s | 2,002 | 10,474 | 90–97°C |
| 80ms (prod default) | ~1.7s | ~625 | ~3,270 | ~70°C |
| 250ms | ~5.4s | ~200 | ~1,047 | ~50°C |
| 500ms | ~10.7s | ~100 | ~523 | ~45°C |

Latency scales roughly with `gap × log(N)`, throughput roughly with `1/gap`. It's a dial, not a fixed number — you're trading finality speed against network chatter and, on real hardware without a data center's cooling budget, against how hot your CPU runs. That's the kind of tradeoff table you only get by actually measuring, not by reading the paper.

## A real bug the harness caught

Building the harness wasn't just about producing benchmark numbers — it surfaced a real consensus-correctness bug. Checkpoints are chained via `prev_checkpoint_hash`, and I was reconstructing that hash from a pass-local map during finalization. The problem: two nodes with *identical* decided history could still end up in different finalization "pass groupings," so the pass-local map would substitute a pre-batch sentinel value for rounds below that pass's floor — meaning two nodes could sign byte-different checkpoint payloads for the same round. That's a quorum-breaking bug: it silently violates the "everyone signs the same thing" invariant the whole checkpoint scheme depends on.

The fix was to anchor the hash lookup in a node-lifetime map, seeded on restart and pruned alongside snapshots, instead of a map scoped to the current finalization pass. Small diff, real bug — the kind that only shows up when you're actually running multiple nodes with real restart/recovery paths, not when you're single-stepping through a test.

## What's next

The consensus core, checkpointing, and now a real multi-node benchmark harness are in place. Next is pushing scale — the design docs already sketch the path to 100/1,000 nodes (QUIC transport, smarter peer selection, parallel execution) — but the immediate value of today was proving the harness itself works and getting a real, reproducible number to compare future changes against.
