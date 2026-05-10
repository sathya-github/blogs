# Diagnosing a Sluggish Intel Mac: A Field Manual

If you own an Intel MacBook Pro, you have probably noticed that "it just feels slow today" is a question without a good answer. Activity Monitor tells you something is up, but rarely *why*. Restart and the symptom often goes away — but you never quite know what it was, or whether it will come back.

The terminal does a much better job. Eight commands, clearly interpreted, will tell you the actual state of your machine: how much memory is free, whether you are swapping to disk, what is eating CPU, what background daemons are misbehaving, and whether any rogue virtual machines are quietly hogging resources.

This is a reference for those eight commands. Bookmark it. The next time your keyboard feels laggy, your fans are spinning for no apparent reason, or the cursor stutters between windows, run through this list.

## 1. The 30-second overview

**What it tells you:** A snapshot of every important system metric in one place — free RAM, swap activity, load average, and CPU idle time.

**Command:**
```bash
top -l 1 | head -10
```

**How to read it:** Three lines matter most.

The `Load Avg` line shows three numbers — load over the last 1, 5, and 15 minutes. On a 16-thread machine (Intel i9 with hyperthreading), sustained load above 8 means the CPU is queuing more work than it can serve. Watch the trend: if 1-min is higher than 15-min, things are getting worse, not stabilizing.

The `PhysMem` line shows used vs. unused RAM. On a 32 GB Intel Mac, you want at least **6 GB unused** during normal work. Below 3 GB and you are about to start swapping — which is when input lag begins.

The `VM` line shows cumulative swap activity since boot. The numbers in parentheses, like `(0)`, are the *current* rate. Anything other than `(0)` means swap is happening right now.

## 2. Are you swapping to disk?

**What it tells you:** macOS uses your SSD as overflow when RAM fills up. Swap is hundreds of times slower than RAM, and active swap is the single biggest cause of system-wide lag — typing, scrolling, window switching, all of it.

**Command:**
```bash
sysctl vm.swapusage
```

**How to read it:** A healthy output looks like:

```
vm.swapusage: total = 0.00M  used = 0.00M  free = 0.00M  (encrypted)
```

`used` is the number that matters. **Zero is ideal.** Up to a few hundred MB is acceptable on a long-running session. Multiple GB of swap used means your system is regularly hitting memory pressure, and you should hunt down what's eating RAM.

## 3. How is memory feeling?

**What it tells you:** A more nuanced view than "free RAM" — macOS aggressively uses idle RAM as disk cache, so "low free RAM" is not always a problem. This command asks the kernel directly whether memory is comfortable.

**Command:**
```bash
memory_pressure | head -5
```

**How to read it:** Look for the `System-wide memory free percentage` and the verdict at the bottom. You want:

- **"The system has plenty of free memory"** → green, no action needed
- **"warn"** state → tight but functional; close some apps when convenient
- **"critical"** state → actively swapping; close apps now or restart

If `memory_pressure` says "plenty" but `vm.swapusage` shows several GB used, that means you *had* memory pressure earlier and the kernel paged things out — but you are recovering. Once the pressure passes, swap usage stays elevated until reboot, which is harmless on its own.

## 4. What is using all my RAM?

**What it tells you:** A ranked list of which processes are holding the most memory. Use this when free RAM is low and you need to decide what to close.

**Command:**
```bash
ps aux | sort -k 4 -rn | head -10 | awk '{printf "%-6s PID:%-7s %s\n", $4"%", $2, $11}'
```

**How to read it:** Each line shows percentage of total system memory and the process. On a 32 GB machine, 1% = ~320 MB. Anything over 5% is using more than 1.5 GB.

Common offenders on a developer's machine:

- **Virtualization VMs** (Docker, OrbStack, UTM, Lima) — often 5–15% each
- **JetBrains IDEs** (IntelliJ, PyCharm, WebStorm) — 4–8%
- **Electron apps** (Slack, VS Code, Discord, Notion) — 1–3% each, but they stack up
- **Chrome / Chrome Helper (Renderer)** — multiple instances, 1–3% each
- **Java processes** (Gradle daemons, Kotlin daemons) — often persist after their original task finished

If you spot a Gradle daemon you no longer need, kill it cleanly with `./gradlew --stop` from any project directory.

## 5. What is using all my CPU?

**What it tells you:** A ranked list of which processes are running hot.

**Command:**
```bash
ps aux | sort -k 3 -rn | head -10 | awk '{printf "%-6s PID:%-7s %s\n", $3"%", $2, $11}'
```

**How to read it:** Anything sustained above **30% CPU when you are not actively doing work** is suspicious. Brief spikes from `WindowServer`, `opendirectoryd`, or your terminal during command execution are normal — they handle screen drawing and user lookups respectively.

Sustained high CPU from these is worth investigating:

- `softwareupdated` over 10% → macOS is staging system updates in the background; install them or wait
- `mds` / `mdworker_shared` over 20% → Spotlight is reindexing; usually self-resolves in 30–60 minutes
- `WindowServer` over 30% → graphical rendering is overloaded; reduce window count, disable transparency
- Java processes — usually a runaway build or a stuck IDE indexing job

## 6. Are any virtual machines running?

**What it tells you:** Whether Apple's Virtualization framework is active. On Intel Macs, VMs are particularly expensive — they reserve CPU cores and large RAM blocks. A forgotten VM can easily explain "my Mac feels slow" all by itself.

**Command:**
```bash
ps aux | grep -i Virtualization | grep -v grep
```

**How to read it:** **Empty output is what you want.** Any line returned means a VM is running. Common sources:

- Docker Desktop (look for `--memoryMiB` and `--cpus` flags showing how much it has reserved)
- OrbStack, Lima, Colima, UTM, Tart, Parallels
- macOS itself during system update staging (parented to launchd, PID 1)

If you see a VM you didn't intend to start, find what spawned it:

```bash
ps -o pid,ppid,user,etime,command -p <PID>
```

**Important:** If the parent process (PPID) is 1 (launchd), check whether macOS is preparing an update before killing it:

```bash
softwareupdate --list
```

If updates are pending, the VM is `MobileSoftwareUpdate.UpdateBrainService` and killing it can corrupt the staging. Apply the updates instead.

## 7. Is anything preventing sleep?

**What it tells you:** Background daemons can hold sleep-prevention assertions, which keep your Mac in a high-power idle state instead of letting it actually sleep. This is the cause of mysterious overnight battery drain on Intel Macs, but it also affects daytime responsiveness — a Mac that can't enter low-power states stays warmer and throttles sooner.

**Command:**
```bash
pmset -g | grep sleep
pmset -g assertions | grep -i prevent
```

**How to read it:** The first command shows whether anything is currently blocking system sleep. A healthy line is:

```
sleep    1
```

A problem line names the offender:

```
sleep    1 (sleep prevented by sharingd)
```

The most common chronic offender on Intel Macs is `sharingd` — the daemon for Apple Continuity features (Handoff, Universal Clipboard). Disabling Handoff in **System Settings → General → AirDrop & Handoff** typically fixes this.

The second command lists all current assertions. Brief, transient ones (Time Machine, GoogleUpdater) are normal. Persistent ones from third-party daemons (Dropbox, VPN clients, Logitech utilities) are worth investigating.

## 8. The all-in-one health check

When you want to run all eight checks at once, paste this into your terminal:

```bash
echo "=== Overview ==="; top -l 1 | head -10; \
echo -e "\n=== Swap ==="; sysctl vm.swapusage; \
echo -e "\n=== Memory pressure ==="; memory_pressure | head -5; \
echo -e "\n=== Top 5 by memory ==="; ps aux | sort -k 4 -rn | head -5 | awk '{printf "%-6s PID:%-7s %s\n", $4"%", $2, $11}'; \
echo -e "\n=== Top 5 by CPU ==="; ps aux | sort -k 3 -rn | head -5 | awk '{printf "%-6s PID:%-7s %s\n", $3"%", $2, $11}'; \
echo -e "\n=== Virtualization VMs ==="; ps aux | grep -i Virtualization | grep -v grep; \
echo -e "\n=== Sleep blockers ==="; pmset -g | grep ' sleep '; pmset -g assertions | grep -iE "prevent.*sleep" | head -5
```

Note the use of `;` rather than `&&` between commands — `&&` stops the chain when `grep` returns no matches (which is actually good news), so `;` is more robust for diagnostic scripts where empty results are expected.

Save it as a `macheck` shell alias and run it whenever the machine feels off. It takes about a second and gives you a full picture.

## Threshold reference at a glance

A quick sanity table to keep handy:

- **Free RAM:** healthy >6 GB, watch <4 GB, problem <2 GB
- **Swap used:** healthy 0 MB, watch 1–2 GB, problem >4 GB
- **Load Avg (1-min) on 16-thread Intel i9:** healthy <4, watch 4–8, problem >8
- **CPU idle:** healthy >80%, watch 50–80%, problem <50%
- **Single process CPU when idle:** suspicious >30% sustained

## Final thoughts

Intel Macs are a tighter platform than Apple Silicon — less efficient, less forgiving, and increasingly underserved by software defaults that assume modern hardware. A 32 GB i9 has plenty of capacity for serious development work, but only if you keep an eye on what's running. Default settings in apps like Docker Desktop will quietly claim half your machine; daemons like `sharingd` and `softwareupdated` will quietly drain your battery and your responsiveness.

The eight commands above let you see all of that at a glance. Run them when something feels off, and you'll usually find the answer in under a minute. The fix is almost always "quit the thing that shouldn't be running" or "lower the resource allocation of the thing that needs to keep running."

Your Intel Mac has more years of life left in it than the marketing suggests. A little observability goes a long way toward getting them.