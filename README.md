<p align="right">
  <img src="docs/aineko.svg" alt="" width="56" />
</p>

# Dual DGX Spark RoCE / NCCL lab notes

Lab notes from a 2026-09-05 measure→compare→pick session on a one-cable dual NVIDIA DGX Spark cluster (GB10 / current nvidia-open kernel).

**Physical interconnect:** NVIDIA **N911-class** QSFP DAC (Micro Center carries them), right↔right.


<p align="center">
  <img src="docs/netplan-cx7-fabric-hotl.png" alt="Netplan CX7 fabric inventory — sparkone/sparktwo enp1s0f1np1 and enP2p1s0f1np1 at MTU 9000" width="720" />
</p>

<p align="center"><em>Figure: Live CX7 fabric already matches <code>/etc/netplan/40-cx7.yaml</code> on both Sparks (100/101 subnets @ MTU 9000). Netplan apply held HOTL — leave alone / backup only / align to §6A 172.28.</em></p>

**GitHub Pages:** https://sw30labs.github.io/dgx-spark-roce-lab/

Contents:
- [Lab notes](docs/index.md) — results, winning NCCL config, dead ends
- [LinkedIn draft](docs/linkedin-draft.md)
- [Scoped sudoers drop-in](docs/90-spider-spark-ops.sudoers)

Author: Nicolas Cravino · Org: [sw30labs](https://github.com/sw30labs)
