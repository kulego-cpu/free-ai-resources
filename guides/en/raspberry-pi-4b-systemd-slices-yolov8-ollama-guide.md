---
title: "Running Computer Vision and LLMs on 8GB Raspberry Pi: Fixing Real-time Telemetry Jitter with systemd slices"
description: "Running Signal K marine telemetry, YOLOv8 vision detection, and Ollama offline manuals simultaneously on an 8GB Raspberry Pi 4B routinely saturates four cores and drops sensor packets. This guide breaks down why CPUAffinity fails under full load and demonstrates how to isolate workloads using Linux systemd slices, CPUWeight priorities, and CPUQuota limits."
pubDate: 2026-11-07
author: "Xiao Ke"
category: "Hands-on Guide"
tags: ["Raspberry-Pi", "systemd", "Ollama", "YOLOv8", "Resource-Isolation", "Edge-Computing", "Sysadmin-Guide"]
featured: true
---

> 📌 **Original Source**: This article is syndicated from [FreeAI Dog Official Platform](https://freeaidog.com/blog/raspberry-pi-4b-systemd-slices-yolov8-ollama-guide/). For live LLM availability radar, latency benchmarks, and interactive API testers, visit the official site: [https://freeaidog.com/](https://freeaidog.com/).


# Running Computer Vision and LLMs on 8GB Raspberry Pi: Fixing Real-time Telemetry Jitter with systemd slices

Squeezing multi-task performance out of low-power edge hardware frequently risks starvation for critical real-time workflows.

I maintain an 8GB Raspberry Pi 4B board deployed in an open source marine monitoring installation. This headless Debian node juggles three primary workloads simultaneously. First is a Signal K server, ingesting high-frequency NMEA telemetry for vessel heading, water depth, wind speed, and engine RPM. Second is a background YOLOv8 nano model running via ONNX to scan a bow camera feed at five frames per second. Third is an Ollama daemon serving Llama 3.2 1B to index marine maintenance manuals locally.

This edge co-location pushes quad-core capacity to the brink of continuous overload, causing computing shortfalls whenever requests spike.

When only the vision model runs, four-core CPU utilization hovers around 50 percent, and the aluminum enclosure warms to 68 degrees Celsius, barely skirting thermal throttling limits.

However, the moment an operator submits a manual query via the local web dashboard, Ollama initiates multi-threaded inference, instantly spiking all four cores to 100 percent.

Operational disruption follows immediately. The dashboard loses control.

The smooth NMEA telemetry stream drops packets aggressively. Update latency surges from tens of milliseconds into multiple seconds, triggering repeated disconnection alarms across bridge instruments.

During navigation or industrial inspection, dropping real-time sensor packets while waiting for a language model response represents an unacceptable operational hazard. Who could tolerate sensor dropouts on live instruments?

Can an 8GB Raspberry Pi genuinely host vision processing alongside a local language model?

The open source engineering community recently shared a robust operating system level solution. A developer documented how to resolve this three-workload conflict using Linux systemd slices.

Primary discussion and implementation records are available here:
https://www.reddit.com/r/ollama/comments/1xpeb8t/

In-depth post-mortem analysis reveals that standard core affinity tricks become largely useless once total compute capacity saturates.

Engineers must leverage Linux cgroups v2 resource slicing to enforce deterministic boundaries.

---

## Why Relying on CPUAffinity Alone Fails Completely

When multiple workloads compete for CPU cycles, sysadmins instinctively reach for core pinning.

The project author initially attempted this exact strategy.

He specified CPUAffinity parameters inside the system service definition, pinning the real-time Signal K daemon strictly to core 0 in an attempt to corral background workloads onto the remaining three cores.

To prevent 8GB of physical RAM from thrashing under concurrent model weights, he allocated a 2GB swapfile while lowering swappiness to 10, reserving maximum physical memory for active processes.

Did this standard optimization sequence eliminate telemetry packet drops?

The answer was a clear negative. Packet loss persisted without improvement.

When Ollama and YOLOv8 run at full capacity, their worker threads avoid core 0, yet kernel device driver interrupts, disk I/O, timers, and network software interrupts continue dispatching across all cores.

Once the remaining three cores run fully pinned at 100 percent, operating system context-switching overhead escalates exponentially.

Whenever the telemetry daemon on core 0 issues a system call or requests a shared kernel lock, the scheduler must wait for sibling cores to release shared memory buses and spinlocks.

Cache lines thrash continuously. Context switches spike.

Core pinning failed to provide an isolated environment, creating severe core imbalances instead.

When all four cores hit saturation, single-thread pinning cannot ensure sub-millisecond dispatch responsiveness. The instrumentation lagged regardless.

Engineers must throttle total compute demand at the source. Can we truly clamp compute consumption cleanly? We had plenty of doubts initially.

---

## The Core Solution: Building Two Defenses with systemd slices

A proposal from community contributor Geekomcomputer targeted the fundamental resource control mechanisms in modern Linux.

The Broadcom quad-core processor on the Raspberry Pi 4B provides a fixed ceiling of compute power. The solution does not lie in finding idle cores for individual processes, but in preventing background inference jobs from consuming full compute capacity simultaneously.

The implementation relies on custom systemd resource slices, constructing two operational defenses:

- First Defense: establishing absolute scheduling priority with CPUWeight;
- Second Defense: setting hard compute boundaries on inference workloads using CPUQuota.

The configuration layout is straightforward. Create two custom slice definition files under systemd directories.

The first unit file is telemetry.slice, dedicated to Signal K and critical real-time daemons:

```ini
[Unit]
Description=Telemetry and Critical Realtime Slice

[Slice]
CPUWeight=1000
```

The second unit file is inference.slice, designed specifically to host YOLOv8 and Ollama background processes:

```ini
[Unit]
Description=AI Inference Background Slice

[Slice]
CPUWeight=50
CPUQuota=200%
```

Auditing the resource budget makes the benefits evident.

By default, standard systemd services inherit a CPU weight of 100. We elevate the telemetry slice to 1000 while depressing the inference slice to 50.

During scheduling contention, whenever telemetry processes receive inbound sensor packets, the kernel scheduler prioritizes their execution cycles without hesitation.

Even more crucial is the hard ceiling imposed by CPUQuota set to 200%.

The quad-core Raspberry Pi provides a cumulative 400% CPU time allocation across four cores. Capping the inference slice at 200% guarantees that regardless of inbound request volume, YOLOv8 and Ollama combined can never consume more than two physical cores of compute time.

This permanently guarantees two full cores of compute headroom for the operating system, network stack, log daemons, and real-time telemetry.

---

## Production Verification and Thermal Improvements

After placing slice unit files, reload systemd configuration and assign target services to their respective slices.

Reload the systemd manager:

```bash
sudo systemctl daemon-reload
```

Assign slice ownership inside unit configuration files, or launch controlled daemons directly via the command line:

```bash
sudo systemd-run --slice=inference.slice ollama serve
```

We subjected the modified configuration to high-load regression testing on the physical Raspberry Pi.

Triggering extensive language model inference via the web dashboard produced a completely different operational profile:

Regardless of inference thread intensity, overall CPU utilization in htop remained pinned near 250 percent across distributed cores.

Cores 0 and 1 operated under comfortably light loads.

Inspecting instrument telemetry revealed immediate improvements. Heading angle and depth graphs updated smoothly, network packet latency remained below 15 milliseconds, and instrument disconnect alarms ceased entirely.

Execution proceeded without incident. Telemetry data flowed smoothly once more.

Beyond latency stability, thermal metrics showed marked improvements.

Previously, running dual inference workloads at full throttle caused aluminum enclosure temperatures to approach the 70 degrees Celsius thermal throttling threshold.

Under the 200% quota cap, even after ten minutes of continuous manual querying, enclosure temperatures stabilized near 61 degrees Celsius, staying safely clear of hardware throttling zones.

Is trading a slight decrease in model token generation speed for total embedded system reliability worthwhile? Who could stay calm when critical instruments freeze? The answer is obvious. Trading generation speed for system stability is well worth the investment.

---

## Technical Boundaries and Deployment Checklist for Edge Co-location

Field testing establishes clear operational boundaries when co-locating real-time services with local inference models on low-power boards.

When deploying workloads on a Raspberry Pi 4B or comparable single-board hardware, verify three basic principles:
- Respect hardware physical ceilings. Never expect low-power boards to run multiple heavy models at unconstrained capacity; enforce explicit limits;
- Abandon single-thread pinning assumptions. In interrupt-heavy environments, prioritize cgroups weights and quota enforcement over raw core affinity;
- Monitor operating temperatures closely. Sustained temperatures exceeding 65 degrees Celsius indicate impending throttling risks; resource throttling proves far more effective than adding fans.

While edge hardware provides modest baseline compute, disciplined operating system resource governance allows single-board computers to deliver remarkable reliability.

When managing low-power boards straining under concurrent inference, how large a CPUQuota limit do you plan to configure for background models in your next edge deployment?

---

> 💡 **Source & Live Radar**：
> - Originally published on [FreeAI Dog Official Blog](https://freeaidog.com/blog/raspberry-pi-4b-systemd-slices-yolov8-ollama-guide/)
> - For real-time free LLM endpoint health check and latency benchmarks, visit [FreeAI Dog Availability Radar](https://freeaidog.com/radar/)

