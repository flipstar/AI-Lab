# cmp50hx-rom.md

# The VBIOS swap

The stock 50HX idles like it's still mining: 62–64 W doing nothing. The
MSI-tuned ROM from
[wmantly/turing-multi-gpu-llm-server](https://github.com/wmantly/turing-multi-gpu-llm-server)
fixes that — ~31 W idle, 25% minimum fan, same silicon. If
`cmp-idle-governor.service` is already running, this is optional; it just
makes the card behave before the OS ever gets involved.

| Piece | Pick | Why |
|---|---|---|
| ROM | `msi_cmp50hx_90.02.60.00.17.rom` | MSI-tuned idle + fan curve |
| Tool | nvflash 5.692 | Newer builds refuse the job; 2021 does it |
| Flag | `-6` | The ROM carries an MSI subsystem ID (1462:371F); `-6` overrides |

The one real gotcha: nvflash versions. The latest release would not do the
flash. 5.692 would. When in doubt, go old.

## The sequence

Unload the driver first — nvflash wants the cards free:

```bash
sudo rmmod nvidia_uvm nvidia_drm nvidia_modeset nvidia
```

Find the target. Index 2 is the B:2D slot; index 1 is B:21:

```text
<0> GeForce GT 710       (10DE,128B,19DA,5360) S:00,B:17,D:00,F:00
<1> Graphics Device      (10DE,1E09,10DE,1554) S:00,B:21,D:00,F:00
<2> Graphics Device      (10DE,1E09,10DE,1554) S:00,B:2D,D:00,F:00
```

Backup before anything else:

```bash
sudo ./x64/nvflash --index=2 --save cmp50hx_backup.rom
```

Stock came back as `90.02.60.00.1A` on an ISSI IS25WP080 chip, built
04/22/21. That file is the recovery path — keep it in the repo.

Then flash:

```bash
sudo ./x64/nvflash --index=2 -6 msi_cmp50hx_90.02.60.00.17.rom
```

nvflash warns about the subsystem ID mismatch — 1462.371F in the ROM,
10DE.1554 on the card. Expected: it's an MSI part number on an NVIDIA board.
Answer `y`, then `y` again:

```text
Firmware image updated.
- New version: 90.02.60.00.17
- Old version: 90.02.60.00.1A
A reboot is required for the update to take effect.
```

Reboot, then confirm with `sudo ./x64/nvflash --index=2` that it reads
`90.02.60.00.17`. Repeat for index 1.

## Recovery

Bricked a card? Same command, backup in place of the MSI ROM:

```bash
sudo ./x64/nvflash --index=2 -6 cmp50hx_backup.rom
sudo reboot
```

The `-6` is harmless on the backup — its subsystem ID already matches.

