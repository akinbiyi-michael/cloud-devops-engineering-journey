# Workstation Inventory & Asset Audit

A full inventory of the machine this entire programme runs on — compute, memory, storage, running
state, and what it is already serving to the network. Captured from the command line, not from a
specification sheet.

**Why this exists.** Before deciding what a machine can run, measure it. This audit is the reason
later phases use WSL2 rather than multiple virtual machines, and target a managed Kubernetes cluster
rather than a local one. Those are not preferences — they follow from the numbers below.

| | |
|---|---|
| **Performed** | 8 November 2026 |
| **Host** | Personal workstation (hostname redacted) |
| **OS** | Windows 11 Pro |
| **Method** | PowerShell, read-only commands only. Nothing installed, nothing modified. |

> Network addresses in section 6 are partially redacted. Publishing a host's full internal
> addressing serves no purpose and narrows an attacker's work for them.

---

## 1 · Compute

```powershell
Get-CimInstance Win32_Processor |
  Format-List    Name, NumberOfCores, NumberOfLogicalProcessors, MaxClockSpeed
```

| Item | Value |
|---|---|
| CPU | Intel(R) Core(TM) i7-4600U CPU @ 2.10GHz |
| Physical cores | **2** |
| Logical processors | **4** |
| Max clock speed | **2701 MHz** reported max · 2.10 GHz base (the gap is Turbo Boost) |
| Generation | Haswell (2013), ultra-low-voltage mobile part |

**Cores vs logical processors — what is the difference on this machine, and why?**

> _your answer_

**Note on the model name.** "i7" is a market tier, not a capability. The `U` suffix denotes a
low-power mobile part, and this is a dual-core chip from 2013 — a more recent mid-range desktop
processor would outperform it substantially. The specification matters, the name does not.

---

## 2 · Memory

```powershell
Get-CimInstance Win32_OperatingSystem |
  Select-Object @{n='FreeRAM_GB';e={[math]::Round($_.FreePhysicalMemory/1MB,2)}},
                @{n='TotalRAM_GB';e={[math]::Round($_.TotalVisibleMemorySize/1MB,2)}}
```

| Item | Value |
|---|---|
| Total physical RAM | **11.88 GB** |
| Free at capture | **3.92 GB** |
| In use at capture | ~7.96 GB |

**If this machine lost power right now, what happens to the contents of RAM, and why does that
matter for a database server?**

> **The contents are erased.** RAM is volatile — the data is gone the instant power stops, not gradually.
>
> **Instructor addition:** why a database in particular cares — if it confirmed `saved` while the data existed only in RAM, a power cut would lose a transaction the user was told had succeeded. So it forces the write to physical disk (`fsync`) *before* acknowledging. That is durability, the **D** in ACID, and it is why databases are slower than raw disk throughput suggests they should be.

---

## 3 · Storage

```powershell
Get-Volume | Where-Object { $_.DriveLetter } |
  Select-Object DriveLetter, FileSystem,
    @{n='Size_GB';e={[math]::Round($_.Size/1GB,1)}},
    @{n='Free_GB';e={[math]::Round($_.SizeRemaining/1GB,1)}}
```

| Volume | Filesystem | Size (GB) | Free (GB) | Used |
|---|---|---|---|---|
| C: | NTFS | 237.7 | 111.7 | 53% |
| D: | NTFS | 0.7 | 0.1 | **86%** |

**On the D: volume at 86%.** On a production server a disk at 86% would raise an alert. Here it
means nothing: D: is a 700 MB system volume that was always near capacity and is not growing. A
monitoring rule that fires on "any volume over 80%" would raise a false alarm on this host every
day — and teams that receive daily false alarms learn to ignore alarms.

Thresholds without context produce noise, not signal. This is a design consideration for the
baseline audit toolkit built later in the programme.

**Which is faster, RAM or this disk, and roughly by what factor?**

> **RAM is faster, by roughly 1000×.** On a human scale: if a RAM read took 1 second, the equivalent read from this SSD would take about 17 minutes. That factor is the reason caches, buffers and in-memory layers exist at all.

---

## 4 · Operating system

```powershell
Get-CimInstance Win32_OperatingSystem |
  Format-List Caption, Version, BuildNumber, OSArchitecture

(Get-Date) - (Get-CimInstance Win32_OperatingSystem).LastBootUpTime
```

| Item | Value |
|---|---|
| OS name & edition | **Microsoft Windows 11 Pro** |
| Version / build | **10.0.22621** · build **22621** |
| Architecture | **64-bit** (x64) |
| Uptime at capture | **23 h 49 m** |

---

## 5 · Running state

```powershell
Get-Process | Sort-Object WorkingSet64 -Descending |
  Select-Object -First 10 Name, Id, @{n='Mem_MB';e={[math]::Round($_.WorkingSet64/1MB)}}
```

| Process | PID | Memory (MB) | What it is |
|---|---|---|---|
| Memory Compression | 2224 | **1128** | Kernel memory compression — see below |
| chrome | 14912 | 670 | Browser |
| claude | 1252 | 599 | Desktop application |
| ChatGPT Classic | 15444 | 547 | Desktop application |
| chrome | 7156 | 329 | Browser |
| chrome | 5000 | 249 | Browser |
| PhoneExperienceHost | 10984 | 245 | Windows Phone Link |
| explorer | 7512 | 235 | Windows shell |
| chrome | 7908 | 222 | Browser |
| claude | 13924 | 211 | Desktop application |

**Running services:** 109

### Two observations worth recording

**1. The largest single consumer is Memory Compression, at 1,128 MB.** This is the kernel
compressing pages rather than paging them to disk, and it only engages under memory pressure. With
a browser and two desktop applications open, this host is already short of memory. That is the
measured basis for the capacity conclusion in section 8.

**2. Four `chrome` processes, four distinct PIDs, separate memory each.** One *program* on disk;
multiple *processes* in memory. The isolation between them is enforced by the kernel, which is why
one tab crashing does not take the others with it.

---

## 6 · This machine as a server

```powershell
Get-NetTCPConnection -State Listen |
  Select-Object LocalAddress, LocalPort, OwningProcess,
    @{n='Process';e={ (Get-Process -Id $_.OwningProcess -ErrorAction SilentlyContinue).Name }} |
  Sort-Object LocalPort
```

**23 sockets in LISTEN state.** Every one is a server process by definition: a program waiting for
an inbound connection.

| Local address | Port | PID | Process | Purpose | Network-reachable? |
|---|---|---|---|---|---|
| `::` / `0.0.0.0` | 135 | 1080 | svchost | RPC endpoint mapper | **Yes** |
| `192.168.x.x` (×3) | 139 | 4 | System | NetBIOS session service | Per-interface |
| `::` | 445 | 4 | System | SMB file sharing | **Yes** |
| `127.0.0.1` | 4709 | 6500 | wpscloudsvr | WPS Office sync | No — loopback only |
| `0.0.0.0` | 5040 | 7348 | svchost | Connected Devices Platform | **Yes** |
| `::1` | 42050 | 9932 | OneDrive.Sync.Service | OneDrive sync | No — loopback only |
| `0.0.0.0` / `::` | 49664 | 876 | lsass | RPC (security subsystem) | **Yes** |
| `0.0.0.0` / `::` | 49665 | 776 | wininit | RPC | **Yes** |
| `0.0.0.0` / `::` | 49666–49667 | 1836, 1976 | svchost | RPC | **Yes** |
| `0.0.0.0` / `::` | 49668 | 3364 | spoolsv | Print spooler RPC | **Yes** |
| `0.0.0.0` / `::` | 49669 | 852 | services | RPC | **Yes** |
| `0.0.0.0` | 49690 | 10276 | SpotifyLauncher | Spotify | **Yes** |

### The bind address matters more than the port

| Address | Meaning | Reachable from the network |
|---|---|---|
| `0.0.0.0` | Every IPv4 interface | **Yes** |
| `::` | Every IPv6 interface | **Yes** |
| `127.0.0.1` / `::1` | Loopback only | **No** |
| A specific IP | That interface only | That network only |

`wpscloudsvr` and `OneDrive.Sync.Service` are bound to loopback and cannot be reached from the
network at all. Everything bound to `0.0.0.0` or `::` can.

**One finding of note:** a `192.168.56.x` address appears among the interfaces — the VirtualBox
host-only adapter default. This host has VirtualBox installed, and it contributes network
interfaces whether or not a virtual machine is running.

**Is this laptop a server? Defend your answer in three sentences.**

> _your answer_

---

## 7 · This machine as a client

```powershell
Resolve-DnsName example.com | Select-Object Name, Type, IPAddress
Test-NetConnection example.com -Port 443
$r = Invoke-WebRequest -Uri https://example.com -Method Head; $r.StatusCode; $r.Headers
```

**DNS resolution — four records for one name:**

| Name | Type | Address |
|---|---|---|
| example.com | AAAA | 2606:4700:10::6814:179a |
| example.com | AAAA | 2606:4700:10::ac42:93f3 |
| example.com | A | 172.66.147.243 |
| example.com | A | 104.20.23.154 |

Two IPv6 (`AAAA`) and two IPv4 (`A`) records. Multiple addresses for a single hostname is DNS
distributing load across servers — the simplest form of load balancing there is, operating before
a connection is even opened.

**TCP reachability on port 443:** `TcpTestSucceeded : True` — connected to `172.66.147.243:443` via interface `Wi-Fi 2`. Source address redacted (`192.168.x.x`, private). Note this is the **same IPv4 address as the third `A` record above** — the resolver selected one of the two available, which is the load-distribution point in practice rather than in theory.

**HTTP response:**

| Header | Value | What it tells us |
|---|---|---|
| `Server` | cloudflare | The listening process identifying itself |
| `CF-RAY` | `...-AMS` | The responding edge node is in **Amsterdam** |
| `cf-cache-status` | HIT | Served from cache — the origin server was never reached |
| `Age` | 7 | The cached copy was 7 seconds old |
| `Content-Type` | text/html; charset=utf-8 | |
| `Date` | Sun, 08 Nov 2026 21:35:14 GMT | |
| `Last-Modified` | Sun, 08 Nov 2026 18:05:34 GMT | |

**Status code:** **200** — the request succeeded.

Request originated in Lagos; the response came from Amsterdam, from a cache, not from the origin.
There is an entire content delivery layer between client and origin server that a simple request
makes visible.

**Describe the full round trip in your own words — from pressing Enter to the status code
appearing. Name every participant.**

> 1. **Browser** — parses the URL, checks its cache and HSTS list, initiates the request.
> 2. **DNS resolver and authoritative nameservers** — resolve `example.com` to an IP address.
> 3. **Routers / ISP** — route IP packets across networks toward the destination.
> 4. **Edge / CDN / load balancer** — terminates TLS, applies caching and routing rules, forwards the request.
> 5. **Origin web server** — processes the request and returns a status code.
>
> **Instructor addition 1 — the TCP handshake** belongs between 3 and 4: SYN, SYN-ACK, ACK, three packets before any data moves. TLS cannot begin until TCP is established.
>
> **Instructor addition 2 — step 5 did not occur in this measurement.** The captured headers show `cf-cache-status: HIT` and `Age: 7`, meaning the Amsterdam edge answered from its own cache with a copy 7 seconds old. **The origin server was never contacted.** The chain above is correct in general and wrong for this specific request, and noticing that distinction is the point. `cf-cache-status` is among the first headers to check when a site serves stale content after a deployment.

---

## 8 · Capacity verdict

**Could this machine run three virtual machines at 2 GB RAM each, simultaneously, while remaining
usable? Show the arithmetic.**

> **No.** 3 VMs × 2 GB = **6 GB** required. Free RAM at measurement: **3.92 GB**. **Short by 2.08 GB.**
>
> That is the optimistic figure. It assumes zero hypervisor overhead and leaves the host with nothing spare, so the practical shortfall is larger. **Conclusion: Phase 2 uses WSL2 rather than three VirtualBox guests** — a decision derived from this arithmetic, not from preference.

**What breaks first — CPU, RAM, or disk? Why?**

> **RAM — and for a capacity reason, not a volatility one.** Total RAM is finite at 11.88 GB, and the shortfall is 2.08 GB before overhead.
>
> **CPU is the second constraint:** 2 physical cores / 4 threads shared between three guests and the host means severe scheduling contention even if the RAM existed. **Disk is not a constraint** — 111.7 GB free against roughly 30–60 GB of VM images.
>
> **Instructor note:** RAM has three properties worth keeping separate — *volatile* (contents lost when power stops), *fast* (~1000× disk), and *finite* (11.88 GB). Each answers a different question. This one is decided by the third; volatility does not bear on it.

### Consequences for the rest of this programme

> **This arithmetic sets four technical decisions for the whole programme:**
>
> 1. **Phase 2 Linux runs on WSL2**, not a VirtualBox guest. A full VM costs far more than WSL2 on 2 cores and 11.88 GB.
> 2. **Multi-VM labs are not viable.** Course material assuming a three-guest Vagrant setup must be reduced to one guest or moved to cloud.
> 3. **Local Kubernetes is marginal.** Single-node kind or minikube may run; multi-node will not. **Tier 3 therefore targets EKS**, with hourly charges to budget and tear down.
> 4. **Docker Desktop on WSL2 is workable** but will be the heaviest process running. Close the browser before container labs.

---

## 9 · Problems encountered

| What happened | Error text | How I diagnosed it | How I resolved it |
|---|---|---|---|
| CPU core count and max clock speed came back blank | No error text — the command succeeded, the columns were simply empty | Raised with instructor. Cause was PowerShell table auto-sizing silently dropping columns behind a 40-character `Name` field — the data was never missing | Re-ran with `Format-List` in place of `Select-Object`: it prints one field per line and cannot be squeezed by column width |

---

## 10 · Open questions

Things this audit raised that I cannot yet answer.

1. What are the 23 processes listening on this machine, and which of them genuinely need to be reachable from the network? *(Phase 2 — to be answered on Linux with `ss -tlnp`.)*
2. If one physical core provides two logical processors, does it perform twice the work? *(Lesson 2.)*
3. If RAM is roughly 1000× faster than disk, why is everything not simply run from RAM? *(Lesson 2.)*

---

## Evidence

Screenshots in [`evidence/`](evidence/). Hostnames, usernames and full internal addresses cropped
or redacted.
