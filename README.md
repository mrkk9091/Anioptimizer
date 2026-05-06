<div align="center">
![Banner](banner.png)

  
<img src="https://img.shields.io/badge/AniOptimizer-V5.0-blueviolet?style=for-the-badge&logo=android&logoColor=white"/>
<img src="https://img.shields.io/badge/KernelSU-Supported-green?style=for-the-badge"/>
<img src="https://img.shields.io/badge/Magisk-Supported-orange?style=for-the-badge"/>
<img src="https://img.shields.io/badge/Root-Required-red?style=for-the-badge"/>

# 🎮 AniOptimizer V5.0
### Intelligent Gaming Performance Daemon for Android

**By [mrk](https://t.me/mrkGL01)**  
📢 Community Channel: [t.me/optimizeph](https://t.me/optimizeph)

</div>

---

## 📖 What is AniOptimizer?

AniOptimizer is a **root-level system daemon** (`anioptimizerd`) that runs silently in the background and intelligently manages your device's hardware resources in real-time — specifically tuned for gaming. Unlike static performance tweaks, AniOptimizer makes **live decisions** every few milliseconds based on what your device is actually doing right now.

It monitors CPU load, GPU usage, temperature, battery level, frame rate, frame pacing, and memory pressure, then dynamically moves between five power states to keep your games smooth without unnecessarily frying your battery or thermals.

---

## ⚙️ How It Works

### Five Power States

AniOptimizer uses a five-tier state machine. It steps between states smoothly — never jumping more than two levels at once — with cooldown timers to prevent flickering.

| State | Description |
|-------|-------------|
| 🟤 **SAFE** | Emergency thermal / critically low battery — absolute minimum |
| 🔵 **LOW** | Menu / lobby — conserve power, nothing demanding happening |
| 🟢 **BALANCED** | Default idle state, moderate tasks |
| 🟡 **PERFORMANCE** | Active gameplay, moderate load |
| 🔴 **TURBO** | Heavy GPU/CPU demand, severe jank detected |

### What It Optimizes

- **CPU** — Applies per-state frequency governors and nice values per game thread class (main, render, physics, AI, audio, network, binder)
- **GPU** — Frequency floor boosts, efficiency scoring, and throttle detection
- **DDR / RAM** — Memory bus frequency scaled per state
- **SurfaceFlinger** — Compositor thread elevation, display refresh rate management, FPSGO integration
- **Thermals** — Decoupled frequency capping prevents oscillation; precool adjustments per SoC profile
- **I/O** — Storage scheduler tuned per state
- **Memory** — ZRAM writeback, memory compaction, page cache drops under PSI pressure, LMKD gaming mode
- **WiFi** — Power-save disabled during gaming sessions, restored afterward
- **Charging** — Charging current limited at configurable battery % to reduce heat during play
- **IRQ Affinity** — Interrupts redistributed for lower gaming latency

---

## 🧠 Smart Features

### 🔍 Game Detection
AniOptimizer reads the foreground app every loop tick. When a package from your game list is detected, it instantly switches context — locking game memory, pinning threads to big cores, compacting RAM, and boosting display Hz.

### 📚 Per-Session Learning
After enough gaming sessions, AniOptimizer learns the **minimum state** your game actually needs and uses it as a floor. Learned data persists across reboots. This means the module gets smarter the more you play.

### 🖥️ Render API Detection (Vulkan vs GLES)
Every 30 seconds during a game session, the daemon scans `/proc/<pid>/maps` to detect whether the game uses **Vulkan** or **GLES**. Vulkan games get a GPU floor boost; GLES games get a CPU governor nudge to compensate for driver overhead.

### 📐 Touch-to-Frame Latency Measurement
AniOptimizer tracks the time between your touch input and the next delivered frame. If the rolling average exceeds the threshold, it escalates state to at least PERFORMANCE — so your inputs never feel laggy.

### 📊 GPU Efficiency Scoring
While gaming, the daemon records FPS-per-MHz for each power state. Over time it learns the most power-efficient state that still delivers smooth frames, and prefers it when thermal headroom exists.

### 💧 Proactive Memory Management (PSI)
When memory pressure (`/proc/pressure/memory`) rises above the threshold, AniOptimizer proactively:
1. Triggers kernel memory compaction
2. Drops clean page caches
3. Initiates ZRAM idle writeback

All with cooldowns to avoid hammering the system.

### 🧪 Passive Benchmarking Mode
A built-in benchmarking mode freezes hardware state changes and records FPS, CPU%, temperature, battery, and jank score over time — without affecting the game's performance. Outputs a CSV you can analyze later.

### 🧯 Frame Pacing & Jank Detection
The daemon tracks frame delivery timing (via `schedstat` context switches on the render thread), computing a jank score and jank class (None / Mild / Moderate / Severe). Severe jank immediately triggers TURBO state; Moderate jank escalates to PERFORMANCE.

### 🔒 Safe Mode
If the device becomes thermally or electrically unstable, AniOptimizer enters a timed safe mode that locks the state machine to prevent further escalation, then automatically re-evaluates after a cooldown.

---

## 🎮 Supported Games (Built-in)

AniOptimizer ships with pre-configured profiles for these titles out of the box:

**Shooters:** Call of Duty Mobile, PUBG Mobile, Free Fire, Critical Ops, Delta Force, Phantom Forces  
**Open World / Heavy GPU:** Genshin Impact, Honkai: Star Rail, Zenless Zone Zero, Wuthering Waves, Tower of Fantasy, Infinity Nikki, Reverse: 1999  
**MOBA / Strategy:** Wild Rift, Mobile Legends: Bang Bang, Honor of Kings, Clash of Clans, Clash Royale  
**RPG / Action:** Arknights, Epic Seven, Sky: Children of the Light  
**Rhythm:** Cytus II, Cytus, Arcaea, Muse Dash, Deemo, Bang Dream  

> You can add any game by dropping its package name into the game list config.

---

## 📦 Installation

1. Download the latest `.zip` from the [Releases](../../releases) page.
2. Open **Magisk** or **KernelSU Manager**.
3. Go to **Modules → Install from storage**.
4. Select the downloaded `.zip` and flash it.
5. **Reboot** your device.
6. The daemon starts automatically on boot. No app required.

> **Minimum requirement:** Android 10+ with a working root via Magisk or KernelSU. The daemon requires root (`uid=0`). It will refuse to start without it.

---

## 🔧 Configuration

Config files are stored in `/data/adb/anioptimizer/` after first boot:

| File | Purpose |
|------|---------|
| `config.conf` | Global settings (temp limits, poll intervals, PSI thresholds, etc.) |
| `games.list` | Package names of games to optimize |
| `profiles.conf` | Per-game overrides (nice value, target FPS, force_performance flag) |
| `learn/` | Persisted learning data per package |

Editing these files takes effect after sending `SIGHUP` to the daemon, or on next reboot. No need to reflash.

---

## 🔋 Does the Daemon Loop Drain My Battery?

**No — and here's exactly why.**

A common concern with background daemons is that a "loop running constantly" must be eating CPU cycles and therefore battery. AniOptimizer is specifically designed to avoid this, and the architecture proves it.

### The loop sleeps, not spins

The main loop does **not** run at full CPU speed. At the end of every single iteration, the daemon calls `adaptive_sleep_get_ms()` and goes to sleep for a variable interval before doing anything again. This is a real OS-level sleep — the process is parked by the kernel and consumes zero CPU during that time, exactly like a notification light blinking: it does work, sleeps, does work, sleeps.

The sleep duration adapts to what is actually happening:

| Situation | Loop interval |
|-----------|--------------|
| Game in menu / lobby (FPS < 15) | `poll_menu_ms` — longest interval |
| SAFE or LOW state | `poll_safe_ms` / `poll_low_ms` — long |
| BALANCED (no game active) | `poll_balanced_ms` — moderate |
| PERFORMANCE state | `poll_perf_ms` — shorter |
| TURBO (heavy gaming) | `poll_turbo_ms` — shortest |

In other words: the daemon polls *fastest* precisely when you're already gaming hard — when the CPU is already awake, your screen is on at full brightness, and you're drawing peak GPU power. The tiny cost of one extra wakeup per few hundred milliseconds is completely invisible against that baseline. When you're not gaming, the loop stretches out to long idle intervals and contributes essentially nothing.

### When no game is detected, everything is restored

The moment your game session ends, AniOptimizer immediately:
- Restores CPU and GPU frequencies to defaults
- Drops DDR back to BALANCED
- Re-enables WiFi power save
- Disables LMKD gaming mode
- Returns display refresh rate to 120 Hz
- Releases all thread pinning and CPU set overrides
- Resets charging limits

The device returns to its normal idle profile. The daemon then stretches its poll interval out and does almost nothing until the next game is detected.

### The daemon itself is a tiny native C process

`anioptimizerd` is a single compiled C binary — no JVM, no JavaScript runtime, no background Android service, no wakelock held. Its entire job per tick is to read a handful of sysfs files (lightweight kernel memory reads, no disk I/O) and occasionally write a value back. The process footprint is measured in kilobytes of RAM and its CPU time between sleeps is microseconds.

### It prevents battery waste, it doesn't cause it

The biggest real source of battery drain during gaming is **thermal throttling**. When a device overheats, the kernel slashes CPU and GPU clocks, the game stutters, and the player pushes the device harder trying to compensate — burning even more power in a vicious cycle. AniOptimizer's thermal management breaks that cycle by proactively capping frequencies before the SoC hits its panic threshold, keeping thermals stable and power draw predictable. The module is designed to save battery compared to running without it, not spend more of it.

---

## ❌ What NOT To Do

> ⚠️ **Read this section carefully. Ignoring it is the #1 cause of issues.**

### ❌ Do NOT mix with other performance modules

**Do not install AniOptimizer alongside:**
- GameMode / GameBooster-type modules
- Thanox / Scene (if using its game engine)
- Any module that modifies CPU governors, GPU frequencies, or scheduler tunables
- FDE.AI, MAGNETAR, or similar daemon-based optimizers
- Any Magisk module that edits `/sys/devices/system/cpu/` or `/proc/sys/vm/`

**Why:** AniOptimizer runs a live loop that continuously applies and re-applies hardware settings. If another module or daemon touches the same sysfs nodes, they will fight each other hundreds of times per second. The result is oscillating performance, excessive heat, and potential instability. There is no safe way to "combine" these — they are fundamentally incompatible.

### ❌ Do NOT use with per-app performance apps

Apps like PUBG GFX Tool, Game Booster+, or OEM game modes (Poco Game Turbo, ROG Armoury Crate mode, etc.) that lock CPU/GPU clocks will conflict with the state machine. The daemon will keep trying to adjust what those apps have locked.

### ❌ Do NOT modify kernel governor manually while gaming

If you manually write to `/sys/devices/system/cpu/cpu*/cpufreq/scaling_governor` while a session is active, AniOptimizer may overwrite your change within milliseconds. Make governor changes via `config.conf` instead.

### ❌ Do NOT flash on a non-rooted device

The daemon checks `uid == 0` at startup and exits immediately if not root. Flashing on a broken root environment will result in the module doing nothing.

### ❌ Do NOT expect miracles on LOW-end devices

AniOptimizer detects your device's performance class on startup and caps the maximum reachable state accordingly (LOW-end → BALANCED cap, MID → PERFORMANCE cap, HIGH → TURBO allowed). This is intentional. Forcing TURBO on a budget SoC causes thermal throttling that makes performance *worse*, not better.

---

## 📋 Logs

Logs are written to `/data/adb/anioptimizer/anioptimizer.log`.  
Enable verbose output by setting `debug_log=1` in `config.conf`.

---

## 👤 Author

<div align="center">

Made by **[mrk](https://t.me/mrkGL01)**

💬 Telegram: [@mrkGL01](https://t.me/mrkGL01)  
📢 Channel: [t.me/optimizeph](https://t.me/optimizeph)

*Join the channel for updates, changelogs, and support.*

</div>

---

<div align="center">
<sub>AniOptimizer V5.0 · KernelSU & Magisk · Root Required · Android 10+</sub>
</div>
