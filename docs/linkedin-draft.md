# LinkedIn draft (simplified) — 2026-09-05

---

What's the best way to get real performance out of a two-Spark cluster with a single QSFP?

Not another NCCL env dump. Measure → compare → pick — on this kernel, today.

I ran that loop with Grok Bot on sparkone ↔ sparktwo — one **N911-class QSFP DAC** (Micro Center carries them), right↔right, dual CX7 rails:

- RDMA ~109 Gbit/s per rail · ~196 Gbit/s concurrent
- NCCL dual-HCA ~23.1 GB/s (single HCA ≈ half)
- QP/channel soup: no peak win
- GDR / peermem on GB10+UMA: dead end
- Netplan: leave live `40-cx7.yaml` alone

**Winner:** both HCAs + mgmt bootstrap (`enP7s7`) + GID 3. No Netplan change.

Lab notes: https://sw30labs.github.io/dgx-spark-roce-lab/

#DGX #NCCL #RoCE #HPC #AIInfra
