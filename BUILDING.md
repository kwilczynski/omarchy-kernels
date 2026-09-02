# Building and Installing the Omarchy Kernels

Sources, patches, configurations, and signing keys:

_Note: temporary location, content will move in the future._

https://github.com/kwilczynski/omarchy-kernels

The repository has three scheduler flavours for two hardware
targets:

| Flavour | Standard x86_64 | Apple T2 |
| --- | --- | --- |
| Base (EEVDF) | `linux-omarchy/` | `t2-linux/linux-omarchy-t2/` |
| BORE 6.8.0 | `linux-omarchy-bore/` | `t2-linux/linux-omarchy-t2-bore/` |
| MuQSS 0.31 | `linux-omarchy-muqss/` | `t2-linux/linux-omarchy-t2-muqss/` |

The base and BORE flavours enable sched-ext. BORE also builds the
ADIOS 3.2.0 I/O scheduler as a module. MuQSS replaces the mainline
scheduler build and does not enable sched-ext. See `FEATURES.md` for
the complete configuration and scheduler differences.

Each directory builds two packages. Its final path component is the
package base. The outputs are the kernel
(`<NAME>-<VERSION>-x86_64.pkg.tar.zst`) and its headers
package (`<NAME>-headers-<VERSION>-x86_64.pkg.tar.zst`, needed for
out-of-tree modules and DKMS). For example, `linux-omarchy-bore/`
builds `linux-omarchy-bore-7.2.3-1-x86_64.pkg.tar.zst` and
`linux-omarchy-bore-headers-7.2.3-1-x86_64.pkg.tar.zst`. What each kernel
contains: see `FEATURES.md`; per-patch details: each directory's
`FILES.txt`.

Three ways to get a kernel, from fastest to most manual:

1. Install the prebuilt binary packages.
2. Build the packages from source with `makepkg`.
3. Apply the patch stack to a raw kernel tree (development only).

---

## 1. Install the prebuilt binary packages

Each published kernel package has its own manifest. The manifest is
named after the package and lists its kernel and headers artifacts:

    linux-omarchy.SHA256SUMS
    linux-omarchy-bore.SHA256SUMS
    linux-omarchy-muqss.SHA256SUMS
    linux-omarchy-t2.SHA256SUMS
    linux-omarchy-t2-bore.SHA256SUMS
    linux-omarchy-t2-muqss.SHA256SUMS

Each manifest has a detached GPG signature beside it, named by
appending `.3DE334E7.sig` (key of Krzysztof Wilczynski).
`<MANIFEST>` below stands for one of these six filenames.
Standard and Apple T2 releases are in separate directories. A
release directory can contain only the packages published for that
release. Check that it contains the manifest for the package you
selected.

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

Install the kernel and its headers package together. This example
installs the standard BORE flavour. Use both filenames from the
manifest for the package you selected:

    sudo pacman -U \
        linux-omarchy-bore-7.2.3-1-x86_64.pkg.tar.zst \
        linux-omarchy-bore-headers-7.2.3-1-x86_64.pkg.tar.zst

The packages install the kernel image and modules under
`/usr/lib/modules/<RELEASE>/`. Your distribution's pacman hooks
build the initramfs and refresh the boot entries; check that your
bootloader configuration lists the new kernel before you reboot.

Different flavours and hardware targets use different package names,
so they can coexist. You can keep `linux-omarchy` installed as a
fallback while you test `linux-omarchy-bore`. Upgrading the same
package replaces it.

### 1.3 Confirm after reboot

    uname -r

Expect the release for the package you installed:

- Base: `7.2.3-1-omarchy`
- BORE: `7.2.3-1-omarchy-bore`
- MuQSS: `7.2.3-1-omarchy-muqss`
- Apple T2 base: `7.2.3-1-omarchy-t2`
- Apple T2 BORE: `7.2.3-1-omarchy-t2-bore`
- Apple T2 MuQSS: `7.2.3-1-omarchy-t2-muqss`

A release string without the `-omarchy` suffix means you did not
boot these packages.

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
- `broadcom-wl-dkms` - proprietary Broadcom Wi-Fi. Does not build
  against kernel 7.2: the driver calls the removed `strncpy()`.
  Debian's broadcom-sta `47-linux72.patch` applied to the DKMS
  source restores the build.
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

DKMS packages in the Arch repositories track the Arch kernel
version. A package can lag these kernels until its upstream or Arch
packaging catches up.

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

`makepkg` checks every source against its PKGBUILD checksum. It also
verifies each source that has a detached signature. Three public keys
verify the signed sources: Linus Torvalds and Greg Kroah-Hartman for
the kernel tarball, and Krzysztof Wilczynski for repository patches.
All three keys ship in the repository:

    gpg --import linux-omarchy/keys/pgp/ABAF11C65A2970B130ABE3C479BE3E4300411886.asc
    gpg --import linux-omarchy/keys/pgp/647F28654894E3BD457199BE38DBBDC86092693E.asc
    gpg --import linux-omarchy/keys/pgp/12D27D5D8C8E9BF1AABC7C7C7C64768D3DE334E7.asc

Without this step `makepkg` stops at source verification.

### 2.3 Build

Set `package_dir` to one source directory from the table at the top
of this document. This example selects the standard base flavour:

    package_dir=linux-omarchy
    cd "$package_dir"
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
  directory, so builds of multiple packages download it once.
- `PKGDEST=<DIRECTORY>` - collects the finished packages in one place.
- `MAKEFLAGS="-j$(nproc)"` - the PKGBUILD already parallelizes the
  kernel build itself.

### 2.4 What a correct build log shows

- Source verification: checksum validation reports `Passed` for each
  patch. Signature files report `Skipped` because the PKGBUILD does
  not checksum detached signatures. The later GPG stage reports
  `Passed` for each source that has a detached signature.
- Patch application: one `Applying patch ...` line per patch, with
  no `offset`, no `fuzz`, and no `FAILED` hunks. The application
  order is the `source()` array order in the PKGBUILD. Do not sort
  the array. Section 3 describes the current package layouts.
- Config: `Setting config...` runs `make olddefconfig` and prints a
  diff against the shipped `config.x86_64`. A small diff is normal:
  the version banner and the compiler-detection symbols
  (`CONFIG_CC_VERSION_TEXT`, `CONFIG_GCC_VERSION`, and related
  lines) record your compiler version, so they change with your
  toolchain. Changed lines outside that set mean the config no
  longer matches the tree - read them before you use the build.
- `Prepared linux-omarchy version 7.2.3-1-omarchy` - the release
  string the selected package will report. The package name and
  suffix change with `package_dir`.
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
  it). Keep them if you plan to build another package. All six
  packages use the same tarball.
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

Set `package_dir` to the source directory you want to test. Write the
ordered patch list to a file first. The order is the `source()` array
of that PKGBUILD:

    package_dir=linux-omarchy-bore
    awk '/^source=\(/,/^\)/' "$package_dir/PKGBUILD" | \
        grep -oE '[0-9]{4}[^ {]*\.patch' > patch-order.txt

The three standard package arrays use filename order. The T2 arrays
do not. They apply scheduler and Apple hardware patches according to
their dependencies. Do not sort any extracted patch list.

Extract a pristine tree next to the package directory and apply the
list in order, refusing fuzz so drift is visible. The tarball sits
in the package directory after any earlier `makepkg` run there, or
in `SRCDEST` when you set it; download it from kernel.org otherwise:

    tar -xf "$package_dir/linux-7.2.3.tar.xz"
    cd linux-7.2.3
    while read -r p; do
        patch -Np1 --fuzz=0 < "../$package_dir/$p" || break
    done < ../patch-order.txt
    cp "../$package_dir/config.x86_64" .config
    make olddefconfig
    make -j$(nproc)

Two requirements:

- **Keep `.git` away from the tree.** If the tree is inside or under
  a git checkout, `setlocalversion` appends `-dirty` to the release
  string and the kernel misidentifies itself. Build from a plain
  extracted tarball, or add an empty `.scmversion` file.
- **Release-string parity**: the packages add `localversion` files
  (`-<PKGREL>` and the package suffix). A raw tree without them
  reports plain `7.2.3`, which is easy to confuse with a vanilla
  kernel. For the BORE example above, create them as follows:

      echo "-1" > localversion.10-pkgrel
      echo "-omarchy-bore" > localversion.20-pkgname
