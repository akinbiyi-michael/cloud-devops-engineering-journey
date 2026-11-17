# Process, scheduling and memory — a utilisation audit

**Phase 1 · 17 November 2026**
Measured on my own workstation. Hardware profile in [workstation-inventory.md](workstation-inventory.md).

---

## What this audit asked

My machine has **2 physical cores** and runs **109 services** plus 23 listening sockets. A core executes one instruction stream at a time, so the obvious question is how 109 services coexist on 2 cores.

The usual answer is time-slicing: the scheduler gives each process a few milliseconds and switches thousands of times a second. That answer is correct and incomplete. This audit set out to measure the real one.

---

## Method

Four read-only commands. Nothing installed, nothing changed, nothing to clean up.

```powershell
# 1 — uptime, derived rather than counted
$os = Get-CimInstance Win32_OperatingSystem; $os.LastBootUpTime; (Get-Date) - $os.LastBootUpTime

# 2 — total core-seconds consumed by every process since it started
$cpu = (Get-Process | Measure-Object CPU -Sum).Sum; [math]::Round($cpu)

# 3 — instantaneous utilisation across all processors
(Get-Counter '\Processor(_Total)\% Processor Time').CounterSamples.CookedValue

# 4 — where that CPU time went, with memory alongside
Get-Process | Sort-Object CPU -Descending | Select-Object -First 10 Name, CPU,
  @{N='RAM_MB';E={[math]::Round($_.WorkingSet/1MB)}} | Format-Table -AutoSize
```

**Note on command 1:** the OS does not count seconds since boot. It stores the boot timestamp and subtracts it from the current time on demand. Uptime is arithmetic, not a counter.

**Note on command 4:** `@{N=...;E={...}}` builds a calculated column — `WorkingSet` is reported in bytes, so it is divided by 1 MB to be readable.

---

## Raw measurements

| Measurement | Value |
|---|---|
| Last boot | 12 November 2026, 12:54:08 |
| Uptime | **420,614 seconds** — 4 d 20 h 50 m 13 s |
| Physical cores / logical processors | **2 / 4** |
| Core-seconds available (420,614 × 4) | **1,682,456** |
| CPU seconds consumed, all processes | **21,013** |
| Instantaneous utilisation while actively working | **6.39%** |

### Top ten processes by CPU time

| Process | CPU seconds | RAM (MB) |
|---|---|---|
| Spotify | 5,802 | 272 |
| chrome | 3,614 | 349 |
| chrome | 2,515 | 101 |
| chrome | 2,327 | 797 |
| ChatGPT Classic | 2,212 | 485 |
| ChatGPT Classic | 1,136 | 95 |
| Spotify | 813 | 43 |
| claude | 745 | 653 |
| claude | 625 | 254 |
| PhoneExperienceHost | 622 | 288 |
| **Total** | **20,414** | **3,337** |

---

## Findings

### 1 · Utilisation over nearly five days was 1.25%

```
21,013 ÷ 1,682,456  =  1.25%
```

The CPU was idle **98.75%** of the time, across 4 days 20 hours, with 109 services running, 23 sockets listening, a browser open and several applications in use. Measured against 2 physical cores rather than 4 logical the figure is still only ~2.5%.

### 2 · Ten processes accounted for 97.1% of all CPU time

| | CPU seconds |
|---|---|
| Top 10 processes | 20,414 |
| **Every other process combined** | **599** |

Five days of uptime, and everything outside the top ten did about ten minutes of work between them.

### 3 · Not one service appears in the top ten

Every entry is a user-facing application. All **109 services** sit inside that 599-second tail.

This is the real answer to the opening question. The 109 services do not compete for 2 cores because they are not running — they are **blocked**, waiting on a network connection, a timer, a keystroke or a disk read. A blocked process is not queued for a core and consumes no CPU at all.

> **Almost nothing is running. Almost everything is waiting.**

A listening socket is the clearest case: the process makes a blocking syscall, the kernel records that it owns the port, and the process is descheduled until a connection arrives. The alternative — **busy-waiting**, polling the port in a loop — would consume a full core per listener. One would cost ~50% of this machine. Twenty-three would be impossible.

### 4 · CPU time and memory are independent

```
chrome   3,614 s CPU  →  349 MB RAM
chrome   2,327 s CPU  →  797 MB RAM     less CPU, more than twice the memory
```

Two separate resources with two separate exhaustion modes. Worth keeping distinct alongside the three properties of RAM — **volatile**, **fast**, **finite** — each of which answers a different question.

### 5 · Four logical processors are not four cores

`NumberOfCores: 2`, `NumberOfLogicalProcessors: 4`. Hyperthreading duplicates the per-thread bookkeeping (registers tracking execution position) while the arithmetic and floating-point units remain shared. The second logical processor on each core fills stalls while the first waits on memory — and memory is roughly 1000× slower than the core, so there are many stalls to fill.

Realistic gain: **15–30%**, occasionally negative when two threads contend for the same unit. Amdahl's law and coordination overhead then apply on top of that, but the first constraint is simply that half the advertised processors are not execution hardware.

### 6 · The constraint on this machine is memory, not CPU

The ten processes above hold **3.26 GB** of 11.88 GB while having done almost no recent work. A blocked process still holds its memory.

Set against the **2.08 GB shortfall** calculated for three 2 GB virtual machines, the conclusion is concrete: closing the browser and music player before a lab recovers more than a gigabyte, and **CPU was never the limiting factor.**

---

## Consequences

**For this machine.** Memory is the budget to manage, not CPU. Close applications before labs. The 2.08 GB shortfall stands, and WSL2 rather than multiple VirtualBox guests remains the correct choice for Phase 2.

**For infrastructure generally.** Single-digit utilisation on dedicated hardware is the economic basis of virtualisation, containers and cloud. Running one application per physical server wastes almost all of it. Virtual machines pack several operating systems onto one host; containers pack many applications onto one kernel, more densely still; cloud platforms pool the waste across tenants; orchestrators schedule work across a fleet to raise utilisation. Each is a response to the number measured above.

**One caveat.** 1.25% is an average, and averages hide collisions. Capacity planning uses peaks, because workloads become busy at the same moments. This is why right-sizing an instance is a real cost discipline rather than an arithmetic exercise.

**For operations.** Idle services cost **memory and attack surface**, not CPU. Every listening socket is a reachable entry point, which makes "does this need to be reachable for the machine to do its job?" a security question before it is a performance one.

**One runaway process is enough to break the arrangement.** A process in a tight loop never blocks, so it is permanently *ready* and the scheduler — working correctly, being fair — hands it a slice every round. That removes roughly half this machine's compute, and on a 2013 ultra-low-voltage part sustained load also triggers thermal throttling below the measured 2,701 MHz ceiling. The machine does not crash; it degrades. This is the standard shape of a runaway-process incident.

---

## Still open

1. What happens to the uptime figure if the machine is suspended rather than shut down?
2. Two further explanations for a 64 GB, 32-core server being slow. The first is established above — a single-threaded bottleneck or runaway process means core count is irrelevant.
3. Identifying all 23 listening processes individually, and deciding which would be shut off on a production web server. *(Phase 2, on Linux with `ss -tlnp`.)*
