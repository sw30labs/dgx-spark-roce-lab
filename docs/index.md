---
title: "Dual DGX Spark performance lab notes — 2026-09-05"
description: "Measure→compare→pick on a one-cable dual DGX Spark cluster — RDMA, NCCL, GDR dead-ends, scoped sudoers."
---

# Dual DGX Spark performance lab notes — 2026-09-05

Companion to the LinkedIn draft. Operator runbook of what we did, what won, and how to reproduce.

**Owner:** Nicolas Cravino  
**Agent:** Spark Cluster Engineer (Grok Bot)  
**Mandate:** measure → compare → pick. HOTL before Netplan apply.

---

## 1. Topology (as tested)

| Item | Value |
|---|---|
| Nodes | sparkone (`192.168.86.44`), sparktwo (`192.168.86.38`) |
| Fabric | 1× QSFP DAC, **right↔right** |
| Rails | `enp1s0f1np1` / `rocep1s0f1` → `192.168.100.10↔.11` |
|  | `enP2p1s0f1np1` / `roceP2p1s0f1` → `192.168.101.10↔.11` |
| Link | 200000 Mb/s, MTU **9000**, GID idx **3** (RoCEv2 IPv4) |
| Left cages | unused (no second cable) |
| Netplan | `/etc/netplan/40-cx7.yaml` (not runbook `90-spark-manual.yaml`) |
| Stack | Ubuntu 24.04.4, kernel `6.17.0-1032-nvidia`, driver **580.173.02** (open), CUDA 13.0, NCCL 2.30.7, CX7 FW 28.45.4028 |

Canonical desk runbook (Mac): `/Users/spider/Desktop/dgx-spark-manual-cluster-runbook.md` (§6A). Live IPs differ from §6A examples — **leave live IPs**.

SSH from Mac: `ssh sparkone` / `ssh sparktwo` (Sync `nvsync.key`).

---

## 2. Method

1. Inventory both nodes (`ibdev2netdev`, link/speed, Netplan ownership)
2. Score vs §6A → **PARTIAL** (topology match; IP/filename/MTU differ)
3. RDMA: solo T1/T2, concurrent T3, multi-QP
4. NCCL: dual vs single HCA, GDR probe, channel/QP/algo sweep
5. PFC/ECN read-only on DAC
6. Post-update GDR/stack recheck; peermem attempt → abandon
7. libmlx5/rdma-core inventory → skip install (UMA: GDR unsupported)
8. Scoped passwordless sudo for future ops

---

## 3. Results (headline)

| Test | Result |
|---|---|
| RDMA solo / path | ~**109** Gbit/s |
| RDMA concurrent agg | ~**196** Gbit/s (~90% of solo sum) |
| Multi-QP concurrent | flat vs q1 |
| NCCL V1 dual HCA (post-update) | ~**23.1** GB/s peak busbw @512M |
| NCCL single HCA | ~**11.2** GB/s |
| GDR | **unsupported** on Spark UMA / nvidia-open |
| PFC apply | **not recommended** on p2p DAC |

**Winner:** dual HCA + `NCCL_SOCKET_IFNAME=enP7s7` + `NCCL_IB_GID_INDEX=3` + mpirun MCA pin to mgmt. No Netplan change.

Reproduce:

```bash
export LD_LIBRARY_PATH=$HOME/nccl/build/lib
export NCCL_IB_HCA=rocep1s0f1,roceP2p1s0f1
export NCCL_SOCKET_IFNAME=enP7s7
export NCCL_IB_GID_INDEX=3
export NCCL_DEBUG=WARN
# hostfile: 192.168.86.44 + 192.168.86.38
mpirun -np 2 --hostfile HOSTFILE \
  --mca btl_tcp_if_include enP7s7 --mca oob_tcp_if_include enP7s7 \
  -x LD_LIBRARY_PATH -x NCCL_IB_HCA -x NCCL_SOCKET_IFNAME -x NCCL_IB_GID_INDEX -x NCCL_DEBUG \
  $HOME/nccl-tests/build/all_reduce_perf -b 8M -e 512M -f 2 -g 1
```

---

## 4. Dead ends (document so we don't re-learn)

- `nvidia_peermem`: module exists under `nvidia-580-open/`; `modprobe` → **EINVAL** both nodes. Do not persist `modules-load.d`.
- `mlx5dv_reg_dmabuf_mr`: absent in Ubuntu `ibverbs-providers` 50.0; needs rdma-core ≥54; apt has no bump. CUDA `GPU_DIRECT_RDMA_SUPPORTED=0`, `DMA_BUF_SUPPORTED=0`.
- NVIDIA Spark/UMA guidance: classic peermem/dma-buf/GDR path not the play — host-reg paths instead.
- Netplan renumber to `172.28.*`: cosmetics only.

---

## 5. Artifacts

| Path | What |
|---|---|
| Desktop `dgx-spark-cluster-scorecard-2026-09-05.md` | Session scorecard |
| Desktop `dgx-spark-manual-cluster-runbook.md` | Manual cluster runbook |
| Desktop `dgx-spark-raw-mirrors-20260905/` | Archived raw dumps |
| Desktop `90-spider-spark-ops.sudoers` + `-INSTALL.md` | Scoped sudoers |
| Box `/workspace/spark-inventory-20260905/` | Full lab tree |

---

## Appendix A — Scoped passwordless sudo

**Why:** agent ops (kmod, read Netplan, modules-load) without handing out `NOPASSWD: ALL`.

**File:** `/etc/sudoers.d/90-spider-spark-ops` (mode `440`)

```
Cmnd_Alias SPARK_KMOD = /usr/sbin/modprobe, /usr/sbin/rmmod, /usr/sbin/lsmod
Cmnd_Alias SPARK_NETPLAN = /usr/sbin/netplan
Cmnd_Alias SPARK_MODULES_LOAD = /usr/bin/tee /etc/modules-load.d/*, /usr/bin/rm /etc/modules-load.d/*
Cmnd_Alias SPARK_NETPLAN_READ = /usr/bin/cat /etc/netplan/*

spider ALL=(root) NOPASSWD: SPARK_KMOD, SPARK_NETPLAN, SPARK_MODULES_LOAD, SPARK_NETPLAN_READ
```

**Install (from Mac, each node):**

```bash
scp ~/Desktop/90-spider-spark-ops.sudoers sparkone:/tmp/
ssh -t sparkone 'sudo cp /tmp/90-spider-spark-ops.sudoers /etc/sudoers.d/90-spider-spark-ops && sudo chmod 440 /etc/sudoers.d/90-spider-spark-ops && sudo visudo -c && sudo -n /usr/sbin/lsmod >/dev/null && echo OK'
# repeat for sparktwo
```

**Verify:** `sudo -n /usr/bin/cat /etc/netplan/40-cx7.yaml` and `sudo -n /usr/sbin/lsmod`.

**Still HOTL:** any `netplan apply` / `netplan try` — show plan + rollback in chat first.

**Does not grant:** apt, arbitrary tee/cat/rm, reboot, shell as root.

---

## Appendix B — Netplan fabric (`40-cx7.yaml`)

sparkone: `.10` on both subnets. sparktwo: `.11`. MTU 9000. `dhcp4: no`. Management stays on `enP7s7` / 192.168.86.0/24 — do not rewrite default route.

---

## Appendix C — Second cable (optional later)

L↔L + R↔R can add rails. Expect more raw aggregate and multi-job headroom; **not** 2× on today's 2-rank all_reduce while GDR stays off. Buy for fabric headroom/redundancy, not a magic NCCL doubling.