# Building and Installing the Omarchy Kernels

Sources, patches, configurations, and signing keys:

_Note: temporary location, content will move in the future._

https://github.com/kwilczynski/omarchy-kernels

The repository holds six kernel packages:

- `linux-omarchy/` - default kernel (EEVDF)
- `linux-omarchy-bore/` - BORE scheduler, ADIOS module
- `linux-omarchy-muqss/` - MuQSS scheduler
- `t2-linux/linux-omarchy-t2/` - default kernel for Apple T2 Macs
- `t2-linux/linux-omarchy-t2-bore/` - BORE for Apple T2 Macs
- `t2-linux/linux-omarchy-t2-muqss/` - MuQSS for Apple T2 Macs

Each directory builds two packages, named after the directory: the
kernel (`<NAME>-<VERSION>-x86_64.pkg.tar.zst`) and its headers
package (`<NAME>-headers-<VERSION>-x86_64.pkg.tar.zst`, needed for
out-of-tree modules and DKMS). For example, `linux-omarchy-bore/`
builds `linux-omarchy-bore-7.2.2-3-x86_64.pkg.tar.zst` and
`linux-omarchy-bore-headers-7.2.2-3-x86_64.pkg.tar.zst`. What each kernel
contains: see `FEATURES.md`; per-patch details: each directory's
`FILES.txt`.

Three ways to get a kernel, from fastest to most manual:

1. Install the prebuilt binary packages.
2. Build the packages from source with `makepkg`.
3. Apply the patch stack to a raw kernel tree (development only).

---

## 1. Install the prebuilt binary packages

Every kernel variant ships with its own manifest, named after the
variant and listing its two packages (kernel and headers):

    linux-omarchy.SHA256SUMS
    linux-omarchy-bore.SHA256SUMS
    linux-omarchy-muqss.SHA256SUMS
    linux-omarchy-t2.SHA256SUMS
    linux-omarchy-t2-bore.SHA256SUMS
    linux-omarchy-t2-muqss.SHA256SUMS

Each manifest has a detached GPG signature beside it, named by
appending `.3DE334E7.sig` (key of Krzysztof Wilczynski).
`<MANIFEST>` below stands for one of these six filenames.
Download the packages, manifests, and signatures from:

https://drive.google.com/drive/folders/1_pKMyMsWyXPe7wKfFJgk82AA8GHKk5d0

### 1.1 Verify

Import the signing key once (it ships in the repository):

    gpg --import linux-omarchy/keys/pgp/12D27D5D8C8E9BF1AABC7C7C7C64768D3DE334E7.asc

Verify the manifest signature, then the artifacts against the
manifest:

    gpg --verify <MANIFEST>.3DE334E7.sig <MANIFEST>
    sha256sum -c <MANIFEST>

Expect `Good signature` from the key above and `OK` on every line.
Do not install artifacts that fail either check.

### 1.2 Install

Install the kernel and its headers package together:

    sudo pacman -U \
        linux-omarchy-bore-7.2.2-3-x86_64.pkg.tar.zst \
        linux-omarchy-bore-headers-7.2.2-3-x86_64.pkg.tar.zst

The packages install the kernel image and modules under
`/usr/lib/modules/<RELEASE>/`. Your distribution's pacman hooks
build the initramfs and refresh the boot entries; check that your
bootloader configuration lists the new kernel before you reboot.

Different variants are different packages and coexist: you can keep
`linux-omarchy` installed as a fallback while you test
`linux-omarchy-bore`. Upgrading the same variant replaces it.

### 1.3 Confirm after reboot

    uname -r

Expect `7.2.2-3-omarchy`, `7.2.2-3-omarchy-bore`,
`7.2.2-3-omarchy-t2`, or the matching variant name. A release string
without the `-omarchy` suffix means you did not boot these packages.

### 1.4 Extra drivers through DKMS

The headers package exists so that DKMS modules build against these
kernels. Install the headers package for every kernel you boot, and
DKMS drivers from the Arch repositories compile automatically at
package install and at every kernel upgrade. The official DKMS
packages, and what they give you:

- `nvidia-open-dkms` - the open NVIDIA kernel module (Turing and
  newer GPUs).
- `virtualbox-host-dkms` - VirtualBox host modules. The guest
  modules are already inside these kernels.
- `v4l2loopback-dkms` - virtual video devices (OBS virtual camera).
- `linux-apfs-rw-dkms` - the APFS filesystem driver, for the
  standard kernels only. Do not install it on a machine that boots
  a T2 kernel: the T2 kernels build this driver in-tree, and the
  DKMS package builds for every kernel with headers installed
  (`AUTOINSTALL`) and lands in the module directory that depmod
  prefers - it silently replaces the in-tree T2 driver. On a
  machine that mixes T2 and standard kernels, skip the package, or
  remove its build for the T2 release after every kernel upgrade -
  read the built version from `dkms status -m linux-apfs-rw`, then
  `sudo dkms uninstall linux-apfs-rw/<VERSION> -k <T2-RELEASE>`.
- `acpi_call-dkms` - calls arbitrary ACPI methods from userspace;
  used by tools like TLP for battery thresholds on some laptops.
- `bbswitch-dkms` - powers the discrete NVIDIA GPU off and on in
  older Optimus laptops.
- `vhba-module-dkms` - virtual host bus adapter (cdemu).
- `broadcom-wl-dkms` - proprietary Broadcom Wi-Fi. Known broken
  against kernel 7.2 until Arch imports the driver-side strncpy
  fix that Debian already ships (broadcom-sta `47-linux72.patch`);
  applying that fix to the DKMS source restores the build,
  verified on a MacBookPro9,2.
- `bcachefs-dkms` - the bcachefs filesystem, maintained out of
  tree.
- `openrazer-driver-dkms` - Razer keyboard, mouse and peripheral
  drivers for the OpenRazer userspace tools.
- `scap-dkms` - the system-call capture driver for Falco and
  sysdig.
- `deepin-anything-dkms` - the kernel module for Deepin's file
  search.
- `ndiswrapper-dkms` - runs Windows NDIS Wi-Fi drivers on legacy
  wireless hardware. Broken against kernel 7.2 the same way as
  `broadcom-wl-dkms` - the driver still calls the removed
  `strncpy()` - and here no distribution ships a fix yet.

One general caution: DKMS packages in the Arch repositories track
the Arch kernel version, and these kernels run ahead of it - a DKMS
package can lag a new base until its upstream or Arch packaging
catches up, as `broadcom-wl-dkms` does today.

---

## 2. Build the packages from source

### 2.1 Requirements

- An Arch Linux environment (host, container, or chroot) with
  `base-devel`. `makepkg -s` installs the remaining build
  dependencies declared in the PKGBUILD (compiler, rust, pahole,
  and so on).
- Disk: keep at least 60 GB free per build. A kernel build tree with
  debug info is large, and a build that runs out of disk space dies
  at the final link after most of an hour. Build one kernel at a
  time.
- Time: about 10 minutes per kernel on 32 modern cores; scale up for
  smaller machines.

### 2.2 Import the verification keys

`makepkg` verifies every source against a detached signature. Three
keys sign the sources: Linus Torvalds and Greg Kroah-Hartman (kernel
tarball), and Krzysztof Wilczynski (every patch in this repository).
All three ship in the repository:

    gpg --import linux-omarchy/keys/pgp/*.asc

Without this step `makepkg` stops at source verification.

### 2.3 Build

    cd linux-omarchy-bore
    makepkg -s

`-s` installs the missing build dependencies through pacman and
removes nothing afterwards. Useful flag combinations:

- `makepkg -si` - build, then install the finished packages in one
  step (pacman asks for confirmation).
- `makepkg -sri` - the same, and remove the build dependencies that
  `-s` installed once the build succeeds.
- `makepkg -Cf` - remove a leftover `src/` tree first and force a
  rebuild over an existing package; use this for a clean rebuild
  after a failed or interrupted build.
- `makepkg -cf` - force a rebuild and delete the `src/` tree after
  the build; keeps the directory small at the cost of a full
  re-extract next time.

If you prefer to install the dependencies yourself instead of `-s`,
the full list is the `makedepends` array at the top of the PKGBUILD;
install it with `sudo pacman -S --needed <LIST>` and run plain `makepkg`.

Optional environment variables:

- `SRCDEST=<DIRECTORY>` - caches the kernel tarball outside the build
  directory, so six variant builds download it once.
- `PKGDEST=<DIRECTORY>` - collects the finished packages in one place.
- `MAKEFLAGS="-j$(nproc)"` - the PKGBUILD already parallelizes the
  kernel build itself.

### 2.4 What a correct build log shows

- Source verification: every patch line ends in `Passed` (signature)
  or `Skipped` (checksum covered by the signature).
- Patch application: one `Applying patch ...` line per patch, with
  no `offset`, no `fuzz`, and no `FAILED` hunks. The application
  order is the `source()` array order in the PKGBUILD - it is not
  numeric; do not re-sort it.
- Config: `Setting config...` runs `make olddefconfig` and prints a
  diff against the shipped `config.x86_64`. A small diff is normal:
  the version banner and the compiler-detection symbols
  (`CONFIG_CC_VERSION_TEXT`, `CONFIG_GCC_VERSION`, and related
  lines) record your compiler version, so they change with your
  toolchain. Changed lines outside that set mean the config no
  longer matches the tree - read them before you use the build.
- `Prepared linux-omarchy-bore version 7.2.2-3-omarchy-bore` - the
  release string the booted kernel will report.
- Kernel code compiles at zero warnings. Three known-benign warning
  classes come from host tools, not the kernel: bpftool/libbpf
  compiler probes, the deliberate `DEPMOD=/doesnt/exist` suppression,
  and headers-package `$srcdir` references.

### 2.5 Clean up after a build

After the build, the package directory holds:

- `src/` and `pkg/` - the extracted, patched, compiled kernel tree
  and the staged package root. This is the bulk of the disk use.
  Building with `-c` removes `src/` and is the primary
  cleanup mechanism. To remove them afterwards, enter the package
  directory first, then delete by explicit relative path:

      cd linux-omarchy-bore
      rm -Rf ./src ./pkg

- The kernel tarball and its signature (or in `SRCDEST` if you set
  it). Keep them if you plan to build another variant; every variant
  uses the same tarball.
- The finished `*.pkg.tar.zst` packages (or in `PKGDEST`). Keep what
  you install; you can delete the rest.

Build dependencies: if you built with `-sr`, they are already gone.
If you built with `-s` alone, pacman recorded them as dependencies,
and they become orphans once nothing else needs them. Review and
remove orphans with:

    pacman -Qdtq
    sudo pacman -Rns $(pacman -Qdtq)

Read the first command's list before running the second: it lists
all orphaned packages on the system, not only the ones this build
installed.

---

## 3. Apply the stack to a raw kernel tree (development)

For testing patches without packaging. Not for installation.

Write the ordered patch list to a file first - the order is the
`source()` array of the PKGBUILD, not the filename order:

    awk '/^source=\(/,/^\)/' linux-omarchy-bore/PKGBUILD | \
        grep -oE '[0-9]{4}[^ {]*\.patch' > patch-order.txt

Extract a pristine tree next to the package directory and apply the
list in order, refusing fuzz so drift is visible. The tarball sits
in the package directory after any earlier `makepkg` run there, or
in `SRCDEST` when you set it; download it from kernel.org otherwise:

    tar -xf linux-omarchy-bore/linux-7.2.2.tar.xz
    cd linux-7.2.2
    while read -r p; do
        patch -Np1 --fuzz=0 < "../linux-omarchy-bore/$p" || break
    done < ../patch-order.txt
    cp ../linux-omarchy-bore/config.x86_64 .config
    make olddefconfig
    make -j$(nproc)

Two lessons from this repository's own build history:

- **Keep `.git` away from the tree.** If the tree is inside or under
  a git checkout, `setlocalversion` appends `-dirty` to the release
  string and the kernel misidentifies itself. Build from a plain
  extracted tarball, or add an empty `.scmversion` file.
- **Release-string parity**: the packages add `localversion` files
  (`-<PKGREL>` and `-omarchy<VARIANT>`). A raw tree without them
  reports plain `7.2.2`, which is easy to confuse with a vanilla
  kernel. Create them if you need the tree to identify itself:

      echo "-3" > localversion.10-pkgrel
      echo "-omarchy-bore" > localversion.20-pkgname
