# Omarchy Kernels

Linux 7.2.2 base. Six kernels from one patch baseline with
scheduler-specific and Apple T2 overlays.

## Kernels

- **linux-omarchy** - upstream EEVDF scheduler; sched-ext for
  pluggable schedulers (scx-scheds).
- **linux-omarchy-bore** - BORE 6.8.0, a burst-oriented EEVDF
  variant biased toward interactive tasks; sched-ext available.
  Adds the ADIOS I/O scheduler as a module; the default I/O
  scheduler is unchanged.
- **linux-omarchy-muqss** - MuQSS 0.31 (Con Kolivas ck lineage);
  no sched-ext.
- **linux-omarchy-t2**, **linux-omarchy-t2-bore**,
  **linux-omarchy-t2-muqss** - the same three schedulers plus the
  Apple T2 hardware stack.

## Configuration

- HZ=1000 with NO_HZ_FULL on four kernels; the MuQSS pair runs
  HZ=100 with NO_HZ_IDLE. PREEMPT_DYNAMIC on all six.
- RCU callbacks offloaded to kthreads on all CPUs
  (RCU_NOCB_CPU_DEFAULT_ALL): no RCU softirq work on the queueing
  CPU, longer C-state residency at idle.
- MGLRU on by default; THP always; zswap on by default; zram as a
  module.
- NTSYNC, ksmbd, WireGuard and VirtualBox guest as modules.
- Kyber and BFQ built in; ZSTD kernel and module compression; Rust
  enabled.

## Scheduling and latency

- Load balancing spreads across small-core clusters on Intel
  hybrid CPUs and prefers fully idle cores.
- detach_tasks() no longer scans the same tasks twice; avg_idle no
  longer saturates with a stale value that kept idle cores from
  pulling work.
- The task-switch path (~20 functions) compiles always-inline; the
  gain grows with Spectre mitigations enabled.
- Cross-CPU function calls and waits run preemptible; TLB-flush
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

- 47 commits from the 7.3 sound fixes pull: HDA and USB audio
  quirks, a USB MIDI out-of-bounds write fix, AMD ASoC DMI
  entries, SoundWire and core fixes.
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

The standard set minus the Dell panel patches (T2-order forks
replace them), plus:

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
