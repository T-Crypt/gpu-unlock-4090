# GPU unlock 1.0.0 — RTX 4090 / AD102

This Linux kernel module enables the SM speed-select override before the stock
NVIDIA driver starts. RTX 4090 (`10de:2684`, AD102) support has demonstrated
approximately **2× FP16 and BF16 tensor throughput with FP32 accumulation**:

| Mode | Before | After |
|---|---:|---:|
| FP16 → FP32 accumulate | 165.6 TF/s | 331.6 TF/s |
| BF16 → FP32 accumulate | 165.5 TF/s | 330.6 TF/s |

The override survived PCIe Function Level Reset (FLR), stock NVIDIA `615.78.08`
initialization and CUDA benchmarks. INT8, `dp4a` and scalar FP32 stayed near their
baseline rates. This is one tested RTX 4090 and one successful cold power-on;
independent cold-boot reproduction and application numerical validation remain
pending. The measured firmware transaction preceded the repository cleanup;
the cleaned module has host-side validation and still needs that hardware repeat.
See the [technical report](rtx4090-register-report.md) and
[sanitized evidence](docs/evidence/rtx4090/README.md).

The AD102 payload and its debug-booter analysis come from
**[TOTORONG/AD103_win_unlocker](https://huggingface.co/TOTORONG/AD103_win_unlocker)**.
This fork integrates that published payload into
**[Zagitta/gpu-unlock](https://github.com/Zagitta/gpu-unlock)**'s Linux module,
firmware generation and boot packaging. The payload is essential to the result.

## Supported targets

The module selects the AD102 firmware and payload automatically for RTX 4090
`10de:2684`. The original GeForce/CMP GA102/GA104 device list and GA102 payload
are retained. Upstream reported RTX 3090 success; this fork's AD102 work does not
revalidate every legacy board. Other Ada device IDs, professional RTX A-series
and data-centre parts are not accepted. Shared firmware across Ada chips does
not establish support for untested cards.

The RTX 4090 path runs the signed firmware, checks the payload's completion marker and
required permissions, writes SS0 `0x88888888` and SS1 `0x8`, then performs FLR
before the unmodified NVIDIA driver binds. It requires the expected
effective speed-select readout before and after FLR. No patched NVIDIA driver or UEFI
application is needed. The payload uses instruction addresses derived from
published decrypted debug-booter analysis; the production booter is authenticated
and decrypted inside the GPU.

## Build and install

Install the package for your distribution, including its build dependencies:

```sh
make install-arch   # makepkg -sif
make install-deb    # apt-get build-dep, dpkg-buildpackage, apt-get install
make install-rpm    # dnf builddep, rpmbuild, dnf install
```

`make arch`, `make deb` and `make rpm` build without installing. Packages are
built from the committed `HEAD`; Debian and RPM outputs go to `build/deb` and
`build/rpm`. Kernel headers and a working DKMS toolchain are required.

Builds fetch the pinned NVIDIA `610.43.03` source and binary inputs over HTTPS
and verify their SHA-256 hashes. `make inputs` prepares the cache;
`make firmware` generates assets from cached inputs; `make assets` does both.
The default cache is `$HOME/.cache/gpu-unlock` and can be overridden with
`CACHE=`, `INPUTS=` and `ASSETS=`. A clean-chroot Debian build needs
`GPU_UNLOCK_INPUTS` pointing to the input cache because `dpkg-buildpackage`
does not fetch inputs itself.

The source archive produced by `make dist`, the Debian source package and the
`.nosrc.rpm` contain no NVIDIA archives or generated firmware. See
[firmware/README.md](firmware/README.md) for the input and asset layout.

## Enable at boot

The package installs the module and firmware into the initramfs so the unlock
runs before NVIDIA binds the card. It remains inactive unless the kernel
command line opts in:

```text
gpu_unlock.targets=0000:03:00.0
gpu_unlock.targets=0000:03:00.0,0000:04:00.0
gpu_unlock.targets=all
```

These PCI addresses are examples. Use the full PCI names printed by `lspci -D`.
`all` selects every accepted unbound card, including legacy GA102/GA104 targets;
an explicit address is useful when selecting one RTX 4090. Do not mix `all` with
addresses. Invalid entries are logged and ignored.

The packages support these initramfs generators:

- **mkinitcpio:** `/etc/mkinitcpio.conf.d/zz-gpu-unlock.conf` puts `gpu_unlock`
  immediately before `nvidia` in `MODULES`, or last if NVIDIA is absent. Images
  are rebuilt on installation. A custom configuration selected with `-c` or a
  preset `_config` does not read drop-ins; add `gpu_unlock` before `nvidia` in
  that configuration's `MODULES` yourself.
- **initramfs-tools:**
  `/usr/share/initramfs-tools/modules.d/gpu-unlock.conf`.
- **dracut:** `/usr/lib/dracut/dracut.conf.d/gpu-unlock.conf`.

There is no `softdep nvidia pre:` rule because kmod uses only the first matching
soft dependency line, which can conflict with other packages.

**Power the machine off and start it again for the first attempt.** The module
needs a card that no GPU driver has initialized since power-on. A warm reboot
does not provide that state. It skips cards that are already bound, powered down
or bus mastering, and refuses cards whose firmware state shows earlier use.
Loading the module manually after NVIDIA has bound the card cannot unlock it.

Avoid `nvidia-smi`, CUDA, NVML and `nvidia-modprobe` before a manual cold-card
attempt: they can load NVIDIA by name. Ordinary modprobe blacklist rules do not
block explicit named loads. The packaged initramfs flow avoids the need for
manual firmware staging and research parameters.

The original display-card handoff is retained: the firmware framebuffer is
removed just before FLR, and the display stays blank until `nvidia-drm` takes
over. Display handoff and multi-card operation were not part of the RTX 4090
validation.

## Check and disable

Inspect the kernel log after boot:

```sh
journalctl -k -b | grep gpu_unlock
lspci -nnk -s 0000:03:00.0  # replace the example address
```

A loaded module or successful `modprobe` return alone does not prove an unlock:
look for the selected card's `gpu_unlock: unlocked` message and the payload,
selector and post-FLR checks. The report describes independent register and
performance confirmation. The optional read-only sampler lives in
`diagnostics/peek`; the CUDA instruction benchmark is in `diagnostics/bench`.
Never load a second BAR0 sampler while `gpu_unlock` is performing its firmware
transaction.

If the module reports that firmware did not stop and its DMA buffers are being
kept, power off before trying again. The module intentionally retains buffers
that firmware may still use. Do not force-unload it in this state.

Remove `gpu_unlock.targets=` from the kernel command line to disable future
attempts. It can be removed for one boot in the bootloader editor. Removing the
packages removes the module and initramfs integration; power off to return to a
fresh hardware state.

## Development and credits

`make test` runs the host-side suite without accessing a GPU. With generated
firmware available, it also exercises complete unlock transactions against mocked
hardware. `make LLVM=1 -C kernel/gpu_unlock` builds the module with LLVM against
the running kernel's headers.

The Linux module and packaging derive from
[Zagitta/gpu-unlock](https://github.com/Zagitta/gpu-unlock). The AD102 four-PLM
payload derives from
[TOTORONG/AD103_win_unlocker](https://huggingface.co/TOTORONG/AD103_win_unlocker),
including its published payload generator and debug-booter analysis. The GA102
V67 chain derives from
[WildFlash1st/cmp90hx-unlock](https://github.com/WildFlash1st/cmp90hx-unlock).
The earlier CMP unlock work, NVIDIA's published loader code, Nouveau and
[envytools](https://github.com/envytools/envytools) also underpin the loader and
investigation.

The kernel module, packaging and payload integration are GPL-2.0-only. NVIDIA
firmware remains under its own terms. See [firmware/NOTICE.md](firmware/NOTICE.md)
and [licenses/](licenses/) for provenance and preserved license notices.
