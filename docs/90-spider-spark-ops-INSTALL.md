# Install scoped sudoers on sparkone + sparktwo

File: `90-spider-spark-ops` → `/etc/sudoers.d/90-spider-spark-ops`

## On each Spark (interactive)

```bash
sudo visudo -f /etc/sudoers.d/90-spider-spark-ops
```

Paste the drop-in body (no markdown fences), save, then:

```bash
sudo chmod 440 /etc/sudoers.d/90-spider-spark-ops
sudo visudo -c
sudo -n /usr/sbin/lsmod >/dev/null && echo "kmod NOPASSWD ok"
sudo -n /usr/bin/cat /etc/netplan/40-cx7.yaml >/dev/null && echo "netplan read ok" || echo "cat netplan failed (file missing or still needs password)"
```

Or copy from Mac after scp:

```bash
# from Mac
scp ~/Desktop/90-spider-spark-ops.sudoers sparkone:/tmp/
scp ~/Desktop/90-spider-spark-ops.sudoers sparktwo:/tmp/
# on each Spark
sudo cp /tmp/90-spider-spark-ops.sudoers /etc/sudoers.d/90-spider-spark-ops
sudo chmod 440 /etc/sudoers.d/90-spider-spark-ops
sudo visudo -c   # MUST say parsed OK — if not, sudo rm the file immediately
```

## What it allows (NOPASSWD)
- modprobe / rmmod / lsmod
- netplan (generate/try/apply — still treat apply as HOTL in chat)
- tee/rm only under `/etc/modules-load.d/`
- cat only under `/etc/netplan/`

## What it does NOT allow
- apt, shell, arbitrary tee/cat/rm, reboot
- Does not touch Sync `gb10-clock-cap` drop-in on sparkone

## Note
sparkone already has broader Sync NOPASSWD (`systemctl` unrestricted). This drop-in only adds kmod/netplan/modules-load/netplan-read.
