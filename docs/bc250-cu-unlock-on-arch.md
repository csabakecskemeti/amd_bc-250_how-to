---
title: Unlock all 40 CUs on BC-250 on Arch / Manjaro
updated: 2025-09-05
---

# Unlock all 40 CUs on BC-250 on Arch / Manjaro

Complete walkthrough for applying `patch/bc250-40cu-amdgpu.patch` on an Arch-family distro, with an explanation of what each step does and why it is needed.

> **Prerequisites.** You need access to the CU-unlock patch source repo:
> `https://github.com/csabakecskemeti/bc250-40cu-unlock`
>
> ```bash
> git clone git@github.com:csabakecskemeti/bc250-40cu-unlock.git
> ```
>
> All subsequent references to `~/bc250-40cu-unlock` assume this clone path.

## On the bundled scripts

`scripts/bc250-enable-40cu.sh` is Debian-only (it looks for `linux-source-*` and calls `apt-get`) and fails immediately on Arch. The repo also ships `scripts/bc250-enable-40cu-arch.sh`, which the README does not mention — it handles kernel.org source download, `tar --wildcards`, and extracting the full `drivers/gpu/drm/amd/` tree. However it does **not** regenerate the initramfs (`mkinitcpio`), which on Manjaro means the patched module is built and installed correctly and then never loaded. See step 4e — that is the failure mode that looks exactly like "the patch doesn't work."

This document does the whole thing by hand so each step is inspectable.

**Verified on:** Manjaro, kernel `6.18.8-1-MANJARO`, gcc 15.2.1, BC-250 (`0000:01:00.0`, PCI ID `0x13fe`) — result: `active_cu_number 40`, 40/40 CUs active, up from the stock 24.

---

## Background: what you are actually doing

The BC-250 ships with 16 of its 40 CUs fused off by firmware policy. The patch re-enables them by writing two hardware registers during driver init (`CC_GC_SHADER_ARRAY_CONFIG` for enumeration, `SPI_PG_ENABLE_STATIC_WGP_MASK` for wave dispatch — both are required).

Practically, this means:

- **The change lives in the `amdgpu` kernel module**, so you need to compile a modified `amdgpu.ko` and get the kernel to load *yours* instead of the stock one.
- **It is gated behind a module parameter** (`bc250_cc_write_mode`, default `0` = off), so installing the module alone changes nothing until you also set the parameter via a modprobe config.
- **Nothing is permanent.** No firmware is written. Removing the modprobe config, or restoring the stock module, returns the board to 24 CU.

Three things must all be true for the unlock to take effect. Every failure in this document is one of them being false:

1. The patched module is built and matches your running kernel (`vermagic`).
2. The kernel actually loads *that* module — not a stale copy from the initramfs.
3. `bc250_cc_write_mode=3` is set at load time.

---

## Why the bundled Debian script fails

`sudo ./scripts/bc250-enable-40cu.sh build` fails in `find_source()`:

```
[+] Kernel source not found locally. Trying apt...
[E] Cannot find kernel source for 6.18.8-1-MANJARO. Install: apt install linux-source-6.18.8
```

Three independent Debian assumptions, each of which would break on Arch:

1. **Source discovery.** It searches only `/usr/src/linux-source-*` and then calls `apt-get`. Arch/Manjaro have neither. Manjaro's kernel is a versioned package (`linux618` / `linux618-headers`) rather than Arch's plain `linux`, so even generic Arch-specific advice doesn't transfer verbatim.
2. **Extraction pattern.** Its `tar xf ... '*/drivers/gpu/drm/amd/amdgpu/'` calls omit `--wildcards`. GNU tar does not glob by default, so it silently extracts nothing and the build proceeds as if no source existed.
3. **Incomplete tree.** It extracts only the `amdgpu/` subdirectory, which cannot compile — the amdgpu Makefile also pulls in `../include`, `../display`, `../pm`, and `../amdkfd`.

**Why not rebuild the kernel via PKGBUILD?** Manjaro builds from its own packaging rather than Arch or CachyOS PKGBUILDs, so there is no ready-made PKGBUILD to drop the patch into. It also rebuilds the entire kernel (30–60+ min) when only one driver file changed, and replaces your kernel package rather than one swappable file. Building just the module takes a few minutes and is trivially reversible.

**The approach used here:** compile *only* `amdgpu` out-of-tree against the already-installed kernel headers. Using the headers' `Module.symvers` keeps symbol CRCs consistent with your running kernel, which is what makes a vanilla-source module loadable on Manjaro's patched kernel.

---

## Step 0 — Prerequisites and baseline

**Confirm you are on a BC-250.** The patch is guarded by PCI device ID `0x13FE` and does nothing on other hardware.

```bash
lspci -nn | grep -i 13fe
# 01:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Device [1002:13fe]
```

**Record your starting point,** so you can prove afterwards that something changed.

```bash
sudo dmesg | grep active_cu_number      # active_cu_number 24
```

**Note your exact kernel version.** Everything downstream must match it.

```bash
uname -r                                # 6.18.8-1-MANJARO
```

**Install build dependencies.** Substitute the headers package matching your kernel series — kernel `6.18.x` → `linux618-headers`, `6.12.x` → `linux612-headers`.

```bash
sudo pacman -S --needed base-devel zstd curl pciutils linux618-headers
```

Why each: `base-devel` provides gcc/make; `zstd` because Arch stores kernel modules zstd-compressed; `curl` to fetch source; the headers package provides the kernel build system, `.config`, and `Module.symvers` you compile against.

**Verify unsigned modules can load.** If `CONFIG_MODULE_SIG_FORCE` were set, a locally built unsigned module would be rejected and you would need to sign it.

```bash
grep -E "CONFIG_MODULE_SIG_FORCE|CONFIG_MODULE_SIG=" /usr/lib/modules/$(uname -r)/build/.config
# CONFIG_MODULE_SIG=y
# # CONFIG_MODULE_SIG_FORCE is not set     <- good: unsigned modules load
```

**Check whether the initramfs carries its own amdgpu.** This determines whether step 4 needs `mkinitcpio -P` — on a default Manjaro install it does.

```bash
grep -E "^HOOKS=" /etc/mkinitcpio.conf            # a `kms` hook pulls amdgpu in
sudo lsinitcpio /boot/initramfs-*.img | grep -i amdgpu
```

> Run that with `sudo`. The images are mode `600`; without root `lsinitcpio` prints `ERROR: Unable to read file` which greps as an empty result and looks exactly like "amdgpu is not in the initramfs." That false negative is what makes the failure in step 6 so confusing.

---

## Step 1 — Get matching kernel source

```bash
mkdir -p ~/bc250build && cd ~/bc250build
curl -LO https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.18.8.tar.xz
mkdir -p ksrc
tar xf linux-6.18.8.tar.xz -C ksrc --strip-components=1 --wildcards \
    'linux-6.18.8/drivers/gpu/drm/amd/*'
```

**Why vanilla source works.** Manjaro's kernel is close to upstream for this driver, and because you compile against the *installed* headers, the resulting module inherits your kernel's `vermagic` and symbol CRCs. Substitute your own version throughout — the tarball must match `uname -r` stripped of the `-1-MANJARO` suffix, i.e. `6.18.8-1-MANJARO` → `linux-6.18.8.tar.xz`.

**Why `--wildcards`.** GNU tar treats the pattern literally without it and extracts nothing, with no useful error.

**Why the whole `amd/*` tree** rather than just `amdgpu/`: the amdgpu Makefile includes headers from sibling directories (`../include/asic_reg`, `../display`, `../pm`, `../amdkfd`). Extracting only `amdgpu/` fails to compile.

Only the `amd` subtree is extracted (~500 MB), not the whole kernel.

---

## Step 2 — Apply the patch

```bash
patch -p1 -d ~/bc250build/ksrc < ~/bc250-40cu-unlock/patch/bc250-40cu-amdgpu.patch
```

Expected output:

```
patching file drivers/gpu/drm/amd/amdgpu/gfx_v10_0.c
Hunk #2 succeeded at 10132 (offset -2 lines).
```

**The offset is normal and not a warning sign.** `patch` locates hunks by surrounding code context, not line numbers, so it self-corrects when the target file has shifted. A *fuzz* message would mean weaker matching; a clean offset is fine.

**Why `-p1` and not `-p5`.** The paths in the patch are `a/drivers/gpu/drm/amd/amdgpu/gfx_v10_0.c`. From the `ksrc` root you strip one leading component (`a/`), so `-p1`. The original README's `-p5` applies only if you first `cd` into the `amdgpu/` directory, stripping all five leading components.

**Why `-d` instead of `cd`.** `patch -d` sets the working directory explicitly. Running `patch` from the wrong directory produces the confusing `can't find file to patch` prompt rather than a clear error.

Verify the patch is really in the source:

```bash
grep -c bc250_cc_write_mode ~/bc250build/ksrc/drivers/gpu/drm/amd/amdgpu/gfx_v10_0.c   # 7
```

---

## Step 3 — Build the module

```bash
mkdir -p ~/bc250build/ksrc/inc/trace
make -C /usr/lib/modules/$(uname -r)/build \
     M=$HOME/bc250build/ksrc/drivers/gpu/drm/amd/amdgpu \
     KCFLAGS="-I$HOME/bc250build/ksrc/inc/trace" \
     -j$(nproc) modules
```

**What this does.** `-C <headers>` runs the kernel's build system; `M=<dir>` tells it to build an external module from your patched source. This is why only amdgpu is compiled, not the kernel. Takes a few minutes for ~700 objects.

### Why the `inc/trace` directory is needed

Without it the build dies partway through:

```
/usr/lib/modules/6.18.8-1-MANJARO/build/include/trace/define_trace.h:118:42:
fatal error: ../../drivers/gpu/drm/amd/amdgpu/amdgpu_trace.h: No such file or directory
```

`amdgpu_trace.h` hardcodes `TRACE_INCLUDE_PATH ../../drivers/gpu/drm/amd/amdgpu`, a path that assumes an in-tree build where the full source sits under the build root. The installed headers package ships only `Kconfig` at that location, so the relative include cannot resolve.

Creating an empty directory two levels below `ksrc` and adding it to the include path makes the relative path land back inside your extracted tree: `ksrc/inc/trace/` + `../../drivers/gpu/drm/amd/amdgpu/` = `ksrc/drivers/gpu/drm/amd/amdgpu/`. This fixes the build without editing any kernel source.

### Confirm the build

```bash
modinfo ~/bc250build/ksrc/drivers/gpu/drm/amd/amdgpu/amdgpu.ko | grep -E "^vermagic|bc250"
# vermagic: 6.18.8-1-MANJARO SMP preempt mod_unload
# parm:     bc250_cc_write_mode:BC-250: 0=off 1=probe-SE0SH0 ... (int)
```

Check **both**: the `parm` line proves the patch compiled in, and `vermagic` must match `uname -r` exactly or the kernel will refuse to load the module.

---

## Step 4 — Install

### 4a. Strip debug info

```bash
cd ~/bc250build/ksrc/drivers/gpu/drm/amd/amdgpu
cp amdgpu.ko /tmp/amdgpu-40cu.ko
strip --strip-debug /tmp/amdgpu-40cu.ko        # ~650 MB -> ~31 MB
```

The freshly built module is ~650 MB because the kernel config enables full debug info; the stock module is 4.8 MB compressed. `--strip-debug` removes debug sections while keeping the symbol table the module loader needs — plain `strip` would be too aggressive. Working on a copy leaves your build intact.

### 4b. Back up the stock module

```bash
MODPATH=/lib/modules/$(uname -r)/kernel/drivers/gpu/drm/amd/amdgpu
sudo cp $MODPATH/amdgpu.ko.zst $MODPATH/amdgpu.ko.zst.backup
```

**Do not skip this.** It is your recovery path if the patched module misbehaves, and it is the only pristine copy you have short of reinstalling the kernel package.

### 4c. Install the patched module

```bash
sudo zstd -f /tmp/amdgpu-40cu.ko -o $MODPATH/amdgpu.ko.zst
sudo chown root:root $MODPATH/amdgpu.ko.zst
sudo depmod -a
```

Arch/Manjaro store modules zstd-compressed, hence `zstd` rather than a plain copy. `depmod -a` rebuilds the module dependency database so the loader resolves the new file.

**The `chown` is a security fix, not cosmetic.** `zstd -o` copies ownership from the source file in `/tmp`, leaving the installed module owned by your user with mode `644` — an unprivileged account could then rewrite a module the kernel loads as root at boot. Confirm with `ls -l $MODPATH` that both files are `root root`.

### 4d. Enable the unlock

```bash
echo 'options amdgpu bc250_cc_write_mode=3' | sudo tee /etc/modprobe.d/bc250-40cu.conf
```

The parameter defaults to `0` (off), so the patched module behaves exactly like the stock one until this file exists. Mode `3` = clear the harvest mask on all shader arrays.

### 4e. Rebuild the initramfs — REQUIRED

```bash
sudo mkinitcpio -P
```

**This is the step most likely to be missed, and it silently defeats everything above.** With the `kms` hook (default in mkinitcpio v33+), the initramfs contains its own copy of `amdgpu` and loads it early in boot for modesetting. If you don't regenerate it, that stale stock module loads and your patched module in `/lib/modules` is never consulted.

The symptom is precise and misleading: `modinfo amdgpu | grep bc250` succeeds (the right module *is* installed), yet after reboot `dmesg | grep bc250-40cu` is empty, `/sys/module/amdgpu/parameters/bc250_cc_write_mode` does not exist, and `active_cu_number` stays 24.

Upstream's Arch instructions omit this because the PKGBUILD route reinstalls the kernel package, which triggers an initramfs rebuild automatically. The module-only approach here does not, so it must be done by hand.

---

## Step 5 — Verify *before* rebooting

```bash
modinfo amdgpu | grep bc250          # MUST print the parm line
modprobe --showconfig | grep bc250   # options amdgpu bc250_cc_write_mode=3
ls -l $MODPATH                       # both files root root
```

`modinfo amdgpu` without a path resolves through the module database to whatever is actually installed in `/lib/modules`. Empty output means the install did not take and there is no point rebooting — this check catches the "built it but never installed it" failure.

Then:

```bash
sudo reboot
```

---

## Step 6 — Verify after reboot

```bash
cat /sys/module/amdgpu/parameters/bc250_cc_write_mode   # 3

# the running module must now match what is on disk
cat /sys/module/amdgpu/srcversion
modinfo amdgpu | grep srcversion

sudo dmesg | grep bc250-40cu     # or: journalctl -k -b | grep bc250-40cu
sudo dmesg | grep active_cu_number
```

**The `srcversion` comparison is the definitive diagnostic.** It is a hash of the module source, so it identifies *which build* is loaded. If the running value differs from the on-disk value, the kernel loaded a different module than the one you installed — in practice, the stale one from the initramfs (go back to 4e).

Actual output from a successful run on `6.18.8-1-MANJARO`:

```
amdgpu 0000:01:00.0: amdgpu: bc250-40cu-enable: mode=3 se=0 sh=0 CC=0xfff80000->0xffe00000 SPI=0x00000007->0x0000001f
amdgpu 0000:01:00.0: amdgpu: bc250-40cu-enable: mode=3 se=0 sh=1 CC=0xfff80000->0xffe00000 SPI=0x00000007->0x0000001f
amdgpu 0000:01:00.0: amdgpu: bc250-40cu-enable: mode=3 se=1 sh=0 CC=0xfff80000->0xffe00000 SPI=0x00000007->0x0000001f
amdgpu 0000:01:00.0: amdgpu: bc250-40cu-enable: mode=3 se=1 sh=1 CC=0xfff80000->0xffe00000 SPI=0x00000007->0x0000001f

amdgpu 0000:01:00.0: amdgpu: SE 2, SH per SE 2, CU per SH 10, active_cu_number 40
```

Expect four `bc250-40cu-enable` lines, one per shader array (SE0/SH0, SE0/SH1, SE1/SH0, SE1/SH1), each showing `CC=0xfff80000->0xffe00000` (24 CU mask → 40 CU mask) and `SPI=0x00000007->0x0000001f` (3 WGPs → 5 WGPs).

---

## Step 7 — Thermals and validation

More active CUs means more power and heat at the same clock. At the 2 GHz governor default the BC-250 draws ~181 W and reaches 96 °C; capping at 1500 MHz / 900 mV via `cyan-skillfish-governor` gives ~372 tok/s pp512 at 125 W and 83 °C.

### Health test

Newly unlocked CUs are not guaranteed healthy on every board. Check correctness before trusting them:

1. Quick test: `~/bc250-40cu-unlock/scripts/bc250-compute-verify.sh`
2. Thorough per-WGP isolation: `~/bc250-40cu-unlock/scripts/bc250-cu-health-test.sh start`

A contiguous harvest map (`■■■■■■□□□□`) suggests firmware policy rather than defective silicon. Scattered patterns (`■■□□■■□□■■`) are likelier to include bad CUs, which you can mask individually with `amdgpu.disable_cu` — see the CU-unlock repo README for details.

---

## Step 8 — Reverting

### Disable the unlock (keep patched module installed)

```bash
sudo rm /etc/modprobe.d/bc250-40cu.conf
sudo reboot
```

Sufficient on its own, because the parameter defaults to `0`.

### Full restore of the stock module

```bash
MODPATH=/lib/modules/$(uname -r)/kernel/drivers/gpu/drm/amd/amdgpu
sudo mv $MODPATH/amdgpu.ko.zst.backup $MODPATH/amdgpu.ko.zst
sudo rm -f /etc/modprobe.d/bc250-40cu.conf
sudo depmod -a
sudo mkinitcpio -P
sudo reboot
```

`mkinitcpio -P` is needed here too — otherwise the initramfs keeps serving the patched module.

**If the machine won't boot:** pick an older kernel entry in GRUB (e.g. a `6.12` image, if installed), or append `modprobe.blacklist=amdgpu` to the kernel command line for a console-only boot, then restore the backup as above.

---

## After a kernel update

A `linux618` (or similar) package upgrade changes `uname -r` and installs a fresh stock `amdgpu.ko.zst`, so the unlock disappears until you repeat the process against the new version. Keep `~/bc250build` (~2.4 GB) to skip the download, or delete it and start from step 1.

Condensed rebuild for a new kernel version — set `KV` and run:

```bash
KV=$(uname -r); KVBASE=${KV%%-*}
cd ~/bc250build
curl -LO https://cdn.kernel.org/pub/linux/kernel/v${KVBASE%%.*}.x/linux-${KVBASE}.tar.xz
rm -rf ksrc && mkdir -p ksrc/inc/trace
tar xf linux-${KVBASE}.tar.xz -C ksrc --strip-components=1 --wildcards "linux-${KVBASE}/drivers/gpu/drm/amd/*"
patch -p1 -d ksrc < ~/bc250-40cu-unlock/patch/bc250-40cu-amdgpu.patch
make -C /usr/lib/modules/$KV/build M=$HOME/bc250build/ksrc/drivers/gpu/drm/amd/amdgpu \
     KCFLAGS="-I$HOME/bc250build/ksrc/inc/trace" -j$(nproc) modules
cp ksrc/drivers/gpu/drm/amd/amdgpu/amdgpu.ko /tmp/amdgpu-40cu.ko
strip --strip-debug /tmp/amdgpu-40cu.ko
MODPATH=/lib/modules/$KV/kernel/drivers/gpu/drm/amd/amdgpu
sudo cp $MODPATH/amdgpu.ko.zst $MODPATH/amdgpu.ko.zst.backup
sudo zstd -f /tmp/amdgpu-40cu.ko -o $MODPATH/amdgpu.ko.zst
sudo chown root:root $MODPATH/amdgpu.ko.zst
sudo depmod -a && sudo mkinitcpio -P
modinfo amdgpu | grep bc250 && sudo reboot
```

The modprobe config in `/etc/modprobe.d/` survives kernel updates and does not need recreating.

---

## Gotchas

| Symptom | Cause | Fix |
|---|---|---|
| `Cannot find kernel source ... apt install` | Bundled script is Debian-only | Fetch source from kernel.org (step 1) |
| `tar` extracts nothing, no error | GNU tar needs explicit globbing | Add `--wildcards` |
| `can't find file to patch` | Wrong working directory | Use `patch -p1 -d <ksrc>` |
| Missing `../include`, `../display` headers | Only `amdgpu/` extracted | Extract all of `drivers/gpu/drm/amd/*` |
| `amdgpu_trace.h: No such file` | Tracepoint path assumes in-tree build | Add `ksrc/inc/trace` to `KCFLAGS` (step 3) |
| Module is ~650 MB | Unstripped debug info | `strip --strip-debug` |
| `.ko` won't load, `vermagic` mismatch | Source version ≠ running kernel | Build against source matching `uname -r` |
| Installed `.ko.zst` owned by your user | `zstd -o` copies source ownership | `sudo chown root:root` |
| Built fine, still 24 CU after reboot | Module never installed | `modinfo amdgpu \| grep bc250` before rebooting (step 5) |
| `modinfo` shows the parm, but after reboot no `bc250-40cu` in dmesg and no `/sys/module/amdgpu/parameters/bc250_cc_write_mode` | Stale `amdgpu` in initramfs shadows `/lib/modules` | `sudo mkinitcpio -P`, reboot, confirm via `srcversion` |
| `lsinitcpio` shows nothing for amdgpu | Images are mode 600; the error greps as "no match" | Run `sudo lsinitcpio` |
| Unlock disappears after a system update | Kernel upgrade replaced the module | Repeat against the new `uname -r` |
| 40 CU active but crashes/artifacts under load | Possibly defective unlocked CUs | `bc250-compute-verify.sh`, then mask with `disable_cu` |

---

## Patch source

The CU-unlock patch and supporting scripts live in a separate repo:

- **Source:** https://github.com/csabakecskemeti/bc250-40cu-unlock
- **Patch file:** `patch/bc250-40cu-amdgpu.patch`
- **Health test scripts:** `scripts/bc250-compute-verify.sh`, `scripts/bc250-cu-health-test.sh`

Clone it alongside your working directory and point the patch path at the cloned copy (see the `KV` rebuild script above).
