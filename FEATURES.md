# Omarchy Kernels

Linux 7.2.3 in three scheduler flavours. Each flavour has a
standard x86_64 package and an Apple T2 package. Unless a section
names a flavour or hardware target, it applies to all six packages.

## Flavours

### Base

- Packages: `linux-omarchy` and `linux-omarchy-t2`.
- Uses the upstream EEVDF scheduler. sched-ext is enabled for
  pluggable schedulers from `scx-scheds`.

### BORE

- Packages: `linux-omarchy-bore` and `linux-omarchy-t2-bore`.
- Uses BORE 6.8.0 on EEVDF. sched-ext remains enabled.
- Adds the ADIOS 3.2.0 I/O scheduler as a module. The default I/O
  scheduler is unchanged.

### MuQSS

- Packages: `linux-omarchy-muqss` and `linux-omarchy-t2-muqss`.
- Uses MuQSS 0.31 from the Con Kolivas ck patch series. MuQSS
  replaces the mainline scheduler build, so sched-ext is not
  available.

## Configuration

- Base and BORE: HZ=1000, NO_HZ_FULL, PREEMPT_DYNAMIC. BORE sets
  the minimum base slice to 2,000,000 ns (CONFIG_MIN_BASE_SLICE_NS).
- MuQSS: HZ=100, NO_HZ_IDLE, PREEMPT_DYNAMIC. CONFIG_RQ_MC=y and
  CONFIG_SHARERQ=2 select the shared runqueue; CONFIG_SMT_NICE and
  CONFIG_MUQSS_IOTIME are enabled. Core scheduling, automatic NUMA
  balancing, utilization clamping, scheduler autogroup, CFS group
  scheduling, CFS bandwidth control and cgroup CPU accounting are
  not enabled.
- RCU callbacks offloaded to kthreads on all CPUs
  (RCU_NOCB_CPU_DEFAULT_ALL): no RCU softirq work on the queueing
  CPU, longer C-state residency at idle.
- MGLRU on by default; THP always; zswap on by default; zram as a
  module.
- NTSYNC, ksmbd, WireGuard and VirtualBox guest as modules.
- Kyber and BFQ built in; ZSTD kernel and module compression; Rust
  enabled.

## Scheduling and latency

### Standard base and BORE

`linux-omarchy` and `linux-omarchy-bore` carry the complete 7.3
scheduler pull, 33 commits, on the 7.2.3 base:

- Cgroup scheduling is flattened onto one runqueue per CPU. Tasks
  from every cgroup are picked from one queue. The group weight is
  applied per task through `cgroup_mode`, which defaults to
  `concur`.
- Slice protection follows the running task's lag. An eligible
  short-slice wakeup cancels the protection. A delayed-dequeue task
  cannot preempt.
- The follow-up fixes floor `tg_cpus()` at 1, charge the current
  task during pick and yield, and read the per-level current during
  throttling and bandwidth distribution.
- Active balance does not start while the source CPU's current task
  is off its runqueue. PSI skips irqtime accounting when no
  interrupt time elapsed.

The series author's test: minimum frame rate 3.8 to 20.6 fps under
eight niced spinners, with the flat cgroup scheduler and a shorter
slice.

### Base and BORE on both hardware targets

- Load balancing spreads work across small-core clusters on Intel
  hybrid CPUs and prefers fully idle cores.
- Cache-aware balancing does not move a task from a CPU that fits
  it to a smaller CPU that does not. RT and deadline push selection
  skips migrate-disabled tasks.
- `detach_tasks()` does not scan the same tasks twice. `avg_idle`
  does not retain a saturated value that stops idle CPUs from
  pulling work.
- Approximately 20 functions in the task-switch path compile
  always-inline. The patch author reports a larger gain when Spectre
  mitigations are enabled.

### BORE

- The standard BORE package runs BORE 6.8.0 on the flat runqueue.
  Its burst penalty, sleep accounting and wakeup preemption use the
  flat runqueue's task weights.
- The T2 BORE package runs BORE 6.8.0 without the 7.3 scheduler
  changes.
- Measured on the maintainer's machine against 7.2.1, seven runs:
  stress-ng context switching +5% to +12%, fork +4% to +8%.

### MuQSS

- MuQSS replaces the fair, RT and deadline scheduler classes, so
  the 7.3 scheduler changes above do not apply to it. The shared
  cpuset, isolation, PSI and stop-machine changes do.
- The T2 MuQSS package runs MuQSS 0.31 without the 7.3 scheduler
  changes.

### All flavours

- Cross-CPU function calls and waits run preemptible. TLB-flush
  data moved to the stack as groundwork.

## Power management

- amd-pstate: the complete six-fix 7.3 batch; includes caching the
  firmware-programmed EPP at init, closing the case where hardware
  kept the firmware setting while sysfs reported performance.
- intel_pstate: correct frequency in scaling_cur_freq, correct
  starting P-state.
- intel_idle: no too-deep C-states while a core goes offline.
- Intel display: FBC on larger planes from Lunar Lake on; PSR2
  early-transport panels accepted instead of refused; a 20 GB/s
  bandwidth point on Panther Lake lets the memory fabric clock
  lower.

## Gaming

- FUTEX_WAIT_MULTIPLE (opcode 31) restored on the merged futex
  engine for old game-runtime builds, with ABI repairs (shared
  bit, wake mask, fixed structure layout); futex_waitv and NTSYNC
  serve current runtimes.
- AMD HDMI 2.1 VRR and ALLM from AMD's maintainer-reviewed series:
  gaming capabilities parsed from the HDMI-Forum block, variable
  refresh driven through standard timing-metadata packets, on TMDS
  as well as FRL links; FRL on by default.
- DSC uses the sink's declared fixed bits-per-pixel from the
  DisplayID VESA block; mode lookup via drm_mode_match() with
  timings, clock and flags.

## Memory management

- 41 commits from the 7.3 mm cycle: mm core fixes, memcg leak and
  reparenting fixes, MGLRU counter fixes, zram hardening, zsmalloc
  class lookup, zswap shrinker performance, and a scan-balance
  series measuring ~60% less lru_lock wait.
- MGLRU keeps executable folios of running programs from being
  evicted first under memory pressure.
- Faster KSM reverse-map walk; less zsmalloc lock contention; zstd
  probes for BMI2 once instead of per compression context.

## Security

- The userspace crypto socket (AF_ALG) accepts only an allowlist
  of algorithms, controlled by the `crypto.af_alg_restrict`
  sysctl and restricted by default. The allowlists cover the
  known users - iwd, bluez, cryptsetup, iproute2. Set the sysctl
  to 0 for the previous behaviour, or 2 to disable AF_ALG.

## Filesystems

- Btrfs, 72 commits from the 7.3 merge: direct io through the
  iomap bounce buffer (~2x), extent buffer tracking on a local LRU
  (~3x), the 1-jiffy sleep removed from non-SSD log commit (~5x),
  fsync skips hole detection on files without holes (~5x), plus
  fixes.
- FUSE: the tail of the old last page zeroed on file growth (no
  stale-data exposure); larger single read requests; a failed
  write-through no longer invalidates a good page; one queued
  request wakes per freed background slot.

## Audio

- 47 of 53 commits from the 7.3 sound fixes pull: HDA and USB audio
  quirks, a USB MIDI out-of-bounds write fix, AMD ASoC DMI entries,
  SoundWire and core fixes. The six omitted commits change code
  absent from 7.2.3.
- 24 laptop audio fixes from the 7.3 cycle: speaker output,
  headset microphones, mute LEDs and jack detection on HP,
  Lenovo, Acer, ASUS, Razer, Dell, LG, Beelink, Minisforum and
  Positivo machines.
- Awinic AW88399 amplifier driver (HDA side codec) with Lenovo
  Legion support.
- Realtek RT766/RT767 SDCA SoundWire codec driver. Upstream has
  no sound-card wiring for this codec yet.
- Intel SOF links powered correctly at probe (fixes
  no-audio-after-boot); Dell XPS 13 (SKU 0E53) sidecar amplifiers.

## Hardware fixes

- Input: Intel THC response bounds and power-down teardown;
  psmouse use-after-free at disconnect; focaltech coordinate
  underflow.
- USB and Type-C: control characters stripped from device strings
  (input devices no longer rejected under Wayland); DisplayPort
  and Thunderbolt altmodes refused on cables that declare they
  cannot carry them; USB4STREAM fixes plus an opt-in per-stream
  busy-poll mode.
- DRM core: monitor refresh range from the DisplayID adaptive-sync
  block; xe framebuffers allocated outside stolen memory (fixes
  hangs); dmem cgroup for GPU memory limits.
- WireGuard keeps tstamp_type on encapsulated packets (no fq qdisc
  log spam).
- PCI: Target Speed quirk skipped on clamped ports without a link,
  removing a fixed 2 s boot delay.

## Platform

- 152 commits from the 7.3 platform-drivers-x86 pull, 58 of
  them fixes.
- New drivers: AMD Halo RGB LED, Lenovo Yoga Book 9i keyboard
  dock, Uniwill laptops.
- AMD: PMF support for Family 1AH Model 80H; HSMP protocol v7;
  PMC smart trace buffer (mp1_stb).
- hp-wmi GPU MUX switching; Huawei Fn-lock.
- Fixes across Dell, ASUS, Lenovo, Panasonic, LG, Toshiba and
  others: probe leaks, races, an out-of-bounds write, fan
  reporting, hotkeys.

## Apple T2 kernels

Each T2 package has the scheduler configuration of its standard
counterpart, without the 7.3 scheduler changes. The three Dell
display patches are not included. The applesmc cache fix is part of
the T2 applesmc series. They add:

- T2 buffer copy engine drivers: internal keyboard, trackpad and
  audio path.
- Asahi input stack: SPI and DockChannel HID transports, RTKit
  helper, magicmouse MTP multitouch.
- Keyboard backlight on the standard :white:kbd_backlight LED
  name, level preserved across resume.
- Touch Bar via appletb-kbd; Fn double-press switches the default
  layer.
- applesmc on the T2 mmio interface: sensors, fan control, iMac
  Pro whitelist entry, battery charge limiter.
- apple-gmux GPU switching on dual-GPU Macs; MacBookPro15,1 dGPU
  runtime PM (suspend with the GPU off).
- Thunderbolt device links fix Apple topology probe order.
- APFS as a module: read, experimental write. The same driver
  ships as linux-apfs-rw-dkms; the DKMS build shadows the in-tree
  driver - do not install it alongside a T2 kernel.
- Dead CPU threads offlined early; AP cache state restored when
  CPUs return online (removes multi-second resume stalls).

## DKMS

The headers packages are DKMS build targets; Arch DKMS packages
build automatically at install and kernel upgrade. Official
packages:

- **nvidia-open-dkms** - open NVIDIA kernel module
- **virtualbox-host-dkms** - VirtualBox host modules
- **v4l2loopback-dkms** - virtual video devices
- **linux-apfs-rw-dkms** - APFS; standard kernels only, shadows
  the T2 in-tree driver
- **acpi_call-dkms** - ACPI method calls from userspace
- **bbswitch-dkms** - Optimus dGPU power switching
- **vhba-module-dkms** - virtual host bus adapter (cdemu)
- **bcachefs-dkms** - bcachefs, out of tree
- **openrazer-driver-dkms** - Razer peripherals
- **scap-dkms** - system-call capture (Falco, sysdig)
- **deepin-anything-dkms** - Deepin file-search kernel module

Not building against 7.2 (removed strncpy()): broadcom-wl-dkms
(Debian ships the fix, broadcom-sta 47-linux72.patch) and
ndiswrapper-dkms (no fix shipped anywhere).
