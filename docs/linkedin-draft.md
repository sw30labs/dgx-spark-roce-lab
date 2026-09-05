# LinkedIn draft — Dual DGX Spark fabric (2026-09-05)

TwinOps-shaped: medium length, direct claim hook, numbers first, punchline close. Karpathy cadence: measure → compare → pick. Edit before post.

---

Two DGX Sparks. One QSFP DAC. A lot of folklore about "the right" NCCL env soup.

I didn't want a literature digest. I wanted the config that wins **on this kernel, today**.

So I pointed Grok Bot at the cluster with a simple mandate: inventory first, then measure → compare → pick. Human-in-the-loop before any Netplan apply.

**What we measured (right↔right, dual CX7 rails, MTU 9000):**

- RDMA solo: ~109 Gbit/s per rail
- RDMA concurrent: ~196 Gbit/s aggregate (~90% of solo sum)
- Multi-QP (q4/q8): flat. No free lunch.
- NCCL all_reduce (2 ranks, 1 GPU each): dual-HCA **~23.1 GB/s** busbw peak
- Single HCA: ~half. Dual rails earn their keep.

**What lost:**

- Channel / QP / algo-proto knobs: none beat the dual-HCA baseline on peak
- PFC/ECN on a p2p DAC: global pause already on; PFC apply not worth the risk
- GPU Direct RDMA: dead end on Spark GB10/UMA — `nvidia_peermem` returns EINVAL on the open driver; CUDA reports GDR/dma-buf unsupported. Env vars cannot override silicon/API reality.

**Winning session config (no Netplan change):**

```bash
export NCCL_IB_HCA=rocep1s0f1,roceP2p1s0f1
export NCCL_SOCKET_IFNAME=enP7s7
export NCCL_IB_GID_INDEX=3
# mpirun: pin MCA btl/oob to enP7s7 (mgmt)
```

Leave the live `192.168.100/101` fabric. Don't renumber to the runbook's `172.28` for cosmetics.

**Takeaway:** on current Spark kernel/stack, the best reachable collective config is dual-rail RoCE + mgmt bootstrap + GID 3. The next real unlock is hardware (second DAC for left↔left) or a future NVIDIA UMA GDR path — not more YAML folklore.

Full lab notes + sudoers appendix: https://sw30labs.github.io/dgx-spark-roce-lab/

#DGX #NCCL #RoCE #HPC #AIInfra
