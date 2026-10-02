# Static BAR1 layout for CMP 170HX behind PLX switches

Boot-time tooling that lets two unlocked CMP 170HX cards share one PCIe switch with a full
64 GB BAR1 each, so the static-BAR1 P2P patches on this branch can work.

## When you need it

You need it only if **all** of these are true:

- you want GPU-to-GPU P2P (the `p2p-*` patches on this branch);
- two or more cards sit behind one PCIe switch (for example a PLX PEX 8747 with two cards);
- the BIOS leaves the switch window too small for two 64 GB BAR1 windows. On our host the BIOS put
  each switch window at the top of the 1 TB aperture, and the kernel could place only one 64 GB
  BAR1 behind each switch. The driver's P2P setup then failed with `-EIO`.

If each card has its own root port, or you do not need P2P, you do not need this directory. The
driver works without it; it loads in non-P2P ("safe") mode.

## What it does

1. `cmp-p2p-boot.service` runs early, before the NVIDIA driver loads.
2. `cmp-p2p-boot` stages a `kexec` image of the running kernel, then runs
   `cmp-bar-layout.py --unlock`. The script programs, with `setpci`:
   - the ReBAR capability of each card up to 64 GB (the same writes the unlock driver makes);
   - each card's 64 GB BAR1 at a 64 GB-aligned base, with its 32 MB BAR3 directly below BAR1;
   - the windows of each switch downstream port, switch upstream port and root port to cover
     their cards.
3. It `kexec`s once into the same kernel with `cmp_layout=done`, so the kernel adopts the layout
   as firmware assignments.
4. On the second pass the NVIDIA driver loads through `cmp-nvidia-guard`. The guard loads with
   static-BAR1 P2P only if every CMP 170HX shows a 64 GB BAR1. In all other cases it loads the
   safe, non-P2P driver.
5. It enables persistence mode and sets the power cap (`POWER_LIMIT=140` W, the default we use).

Every failure path ends in safe mode. The layout script exits 2 and writes nothing if a card is
missing, is not a CMP 170HX, already has a driver, or sits behind an unexpected port.
To turn the P2P path off: `touch /etc/cmp-p2p-disable`.

## Deriving the layout (`/etc/cmp-bar-layout.json`)

`cmp-bar-layout.example.json` contains placeholders only. Fill in your own values:

1. `lspci -tv` and `lspci -nn | grep 20c2`: for each card, note its BDF and its parent switch
   downstream port. List each switch upstream port and root port with the cards below it.
2. `grep -i "pci bus" /proc/iomem` and the BIOS settings: find the 64-bit MMIO aperture. Choose
   one 64 GB-aligned base per card, inside the aperture and clear of other devices. The 32 MB
   below each base holds BAR3, so leave it free too.
3. Run `cmp-bar-layout.py --dry-run --config=/etc/cmp-bar-layout.json` with no driver bound,
   and read the planned writes before you use `--unlock`.

## Install

```bash
install -m 755 cmp-p2p-boot cmp-nvidia-guard cmp-bar-layout.py /usr/local/sbin/
install -m 644 zz-cmp-guard.conf /etc/modprobe.d/
install -m 755 cmp-guard-initramfs-hook /etc/initramfs-tools/hooks/cmp-guard
install -m 644 cmp-p2p-boot.service /etc/systemd/system/
cp cmp-bar-layout.example.json /etc/cmp-bar-layout.json   # then edit it
update-initramfs -u && systemctl enable cmp-p2p-boot.service
```

Requirements: `kexec-tools`, `pciutils`, Python 3.

## Status

- **Measured** (one 4-card host, two PEX 8747 switches with two cards each, Broadwell Xeon,
  Proxmox VE 9.2.2, kernel 7.0.2-6-pve, driver 615.71.09 open): 12/12 GPU pairs pass P2P,
  5.79 GB/s; repeated cold boots with 0 crashes and all four cards in P2P mode.
- **Measured:** on this platform P2P makes GLM-5.3-Flash TP4 decode *slower* (−26 %), because the
  collectives cross the PLX switches. P2P helps TP2 (+11 %). Do not assume P2P is a win.
- **Untested:** other boards, other switch models, AMD hosts, other kernels.

WARNING: Wrong window values can make cards disappear from the bus until a full power cycle.
Use `--dry-run` first, and keep remote power control available.
