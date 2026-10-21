# What each panel means

A dashboard is only useful if you know what "normal" looks like. This page goes through
every panel: what it shows, what's normal, and when to look closer.

Some panels also have a small **ⓘ** next to their title in Grafana, hover over it for a
short version.

Every graph that shows a rate (per second, or a percentage of time) averages over the
last **5 minutes**. A spike that lasted 10 seconds looks smaller than it was. That's on
purpose, it hides noise and shows trends.

- [Overview](#overview)
- [Containers](#containers)
- [CPU](#cpu)
- [Memory](#memory)
- [Disk](#disk)
- [Network](#network)
- [Hardware](#hardware)
- [Pressure (PSI)](#pressure-psi)

## Overview

The machine at a glance. If something here looks wrong, the sections below tell you why.

| Panel | What it shows | Look closer when |
| --- | --- | --- |
| **Uptime** | Time since the machine last booted | It's much shorter than you expected: the machine restarted (power cut, crash, auto-update) |
| **CPU busy** | Percentage of time the CPU wasn't idle, all cores together | It stays above 90% when you're not doing anything heavy |
| **Memory used** | Memory that's really in use, not counting cache Linux can free | It stays above 90% |
| **Root disk used** | How full the disk with `/` on it is | It's above 80%. At 90% the `DiskAlmostFull` alert fires |
| **Battery** | Battery charge | Empty on machines without a battery, that's normal |
| **Load (1m)** | How many processes wanted the CPU, averaged over the last minute | It's well above your number of cores for a long time. See [Load average](#cpu) |

## Containers

Every Docker container on the machine, one line each, including this stack's own.
This is how you find out which of your lab containers is eating the machine.

| Panel | What it shows | Look closer when |
| --- | --- | --- |
| **Container CPU** | CPU per container. **100% is one full core**, so on a 4-core machine a container can reach 400% | One container is high all the time and you don't know why |
| **Container memory** | The memory each container is really using (its "working set"). It's what Docker's memory limit and the out-of-memory killer go by | A line keeps going up and never comes down: that's usually a memory leak. If the container has a limit, it gets killed when it reaches it (`ContainerOomKilled` alert) |
| **Container network** | Traffic in and out of each container | A container you thought was idle is sending lots of data |
| **Container disk I/O** | How much each container reads from and writes to disk | A container is writing all the time (often logs or a database) and the disk feels slow |

**Containers on the host network** (like this stack's own) share the machine's network,
so their network graph shows the whole machine's traffic, not just their own. That's
expected.

A container that keeps restarting shows up as a line that keeps breaking and starting
again. The `ContainerRestarting` alert catches it.

## CPU

| Panel | What it shows | Look closer when |
| --- | --- | --- |
| **CPU by mode** | Where CPU time goes, stacked, so the top of the stack is how busy the CPU is overall | See the modes below |
| **Per-core busy** | One line per core | One core is stuck at 100% while the others are calm. The average on the overview hides this: a program that can only use one core is maxed out |
| **CPU frequency** | The speed each core runs at right now | It stays low under load, or drops when the machine is hot (thermal throttling). Empty in virtual machines |
| **Load average** | Processes running or waiting to run, averaged over 1, 5 and 15 minutes | It stays above your number of cores: things are queueing up. Compare the three lines: 1m above 15m means it's getting busier |

The modes in **CPU by mode**:

| Mode | Means | High means |
| --- | --- | --- |
| `user` | Running your programs | Normal when you're doing work |
| `system` | The kernel working for your programs | Lots of small file or network operations |
| `iowait` | Idle, waiting for the disk | The disk is the bottleneck, not the CPU |
| `nice` | Low-priority programs | Background jobs (indexing, compiling with `nice`) |
| `irq`, `softirq` | Handling hardware, mostly network | Very heavy network traffic |
| `steal` | A virtual machine waiting for the real CPU | Only on VMs: the host is overloaded |

To see how many cores you have: `nproc`.

## Memory

| Panel | What it shows | Look closer when |
| --- | --- | --- |
| **Memory breakdown** | Used, cache, buffers and free, stacked | "Used" keeps growing. A big cache is fine |
| **Swap** | How much swap is in use, and how much there is | Swap use keeps growing while memory is full: the machine is short on memory and slowing down |

**A big cache is normal and good.** Linux uses spare memory to cache files, so they open
faster next time, and gives it back the moment a program needs it. "Free" memory is
wasted memory. That's why **Memory used** on the overview leaves the cache out.

Some swap in use, sitting still, is also fine: Linux moved things nobody was using out of
the way. It's swap that keeps **changing** that means trouble. The Pressure panel shows
that more clearly.

## Disk

| Panel | What it shows | Look closer when |
| --- | --- | --- |
| **Disk throughput** | Megabytes per second read and written, per disk | Something is writing all the time and you don't know what |
| **Disk IOPS** | Read and write operations per second | Very high with low throughput: lots of tiny reads and writes, which is what slows hard drives down most |
| **Disk utilisation** | Percentage of time the disk was busy with at least one request | It sits at 100% and the machine feels slow. On SSDs and NVMe drives 100% doesn't always mean full, they handle many requests at once, so check Pressure too |
| **Filesystem used** | How full each filesystem is | Above 80%. One line per disk, even with btrfs subvolumes |
| **Filesystem free space** | The same, in bytes left | It's going down steadily. `DiskWillFillIn24h` fires when the trend says full within a day |

Disk names: `sda`, `sdb` are SATA disks or USB drives, `nvme0n1` is an NVMe SSD,
`mmcblk0` is an SD card (Raspberry Pi). `dm-0` and similar are encrypted or LVM volumes
sitting on top of a real disk.

## Network

| Panel | What it shows | Look closer when |
| --- | --- | --- |
| **Network throughput** | Bytes per second in and out, per interface | Traffic you didn't expect, especially going out |
| **Network packets** | Packets per second | Lots of packets but little data: many tiny messages, often chatty services or a scan |
| **Network errors and drops** | Broken or dropped packets | **Anything above zero for a while.** It should sit flat on zero. Climbing errors usually mean a bad cable, a weak Wi-Fi signal or a failing network card |

Interface names: `wlan0` or `wlp…` is Wi-Fi, `eth0` or `enp…`/`eno…` is Ethernet,
`tailscale0`/`wg0` are VPNs. Docker's virtual interfaces (`veth…`, `br-…`, `docker0`)
are left out on purpose, every container would add a new one.

## Hardware

| Panel | What it shows | Look closer when |
| --- | --- | --- |
| **Battery charge** | Battery level over time | It drops much faster than it used to: the battery is wearing out |
| **Power source** | Plugged in or on battery | Empty on desktops and servers |
| **Thermal zones** | Temperatures the kernel uses to decide when to slow the CPU down | Close to their limit for long stretches |
| **Hardware sensors (hwmon)** | Every temperature sensor the chips report: CPU package, NVMe drive, chipset, sometimes the GPU | Any sensor within 10°C of its critical limit. That's when `HighTemperature` fires |

Rough guide for laptops: CPUs often sit at 40–60°C idle and 80–95°C under full load, and
that's within spec. NVMe drives start slowing themselves down around 70–80°C. If it's hot
when idle, clean the fans or check what's running.

Sensors are named by the chip that reports them, e.g. `coretemp` (Intel CPU), `k10temp`
(AMD CPU), `nvme` (SSD), `acpitz` (motherboard). Empty in virtual machines.

## Pressure (PSI)

| Panel | What it shows | Look closer when |
| --- | --- | --- |
| **Resource pressure** | Percentage of time programs were **stuck waiting** for CPU, memory or disk | Memory pressure above a few percent for a while (`MemoryPressureHigh` fires at 10%). Disk pressure that stays high |

This is the most honest panel on the dashboard. The other panels say how **busy**
something is, this one says whether it's **slowing you down**. A disk at 100% that only
one program uses has no pressure. A CPU at 70% with lots of programs fighting over it
does.

- **CPU pressure:** more work than cores. Normal in short bursts, like compiling.
- **Memory pressure:** programs waiting while Linux frees memory or swaps. Even a few
  percent makes the machine feel sluggish.
- **IO pressure:** programs waiting for the disk.

Empty on kernels older than 4.20 or built without PSI.
