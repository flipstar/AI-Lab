# Headless BIOS Mode

**Quick summary:** How to make an HP Z6 G4 POST and boot with no GPU
installed. HP hides the option in firmware as `Headless Boot`, and the normal
F10 save/load workaround does not set it on G4 machines. Enabled here from
Linux using HP's unsupported firmware tools, then the GPU was removed.

| Summary | Detail |
|---|---|
| Goal | Run the Z6 G4 with no display adapter installed |
| Method | Set hidden BIOS `Headless Boot = Enable` |
| Tooling | `hpuefi-mod` + `hp-repsetup` on Linux |
| Verified | BIOS `P60_0300.bin`; Ubuntu 26.x amd64; kernel `7.0.0-31-generic` |
| Result | Machine boots with no display adapter |
| Gotcha | F10 setup save/load does not apply this G4 hidden setting |

**Fastest path:**

```bash
# Disable Secure Boot in F10 if enabled
make
sudo make install
sudo modprobe hpuefi
sudo /lib/modules/$(uname -r)/kernel/drivers/hpuefi/mkdevhpuefi
/opt/hp/hp-flash/hp-repsetup -s HeadlessBoot_enable.utf16le
sudo reboot
```

**Safety note:** Back up BIOS settings first and keep a GPU handy until
headless boot is verified. HP ships these tools as-is; firmware changes are
at your own risk.

> **Get the files first:** the `.tgz` SoftPaqs below are not mirrored here.
> Download them from the official HP support page for this machine:
> <https://support.hp.com/de-de/drivers/hp-z6-g4-workstation/16449901>
> Look under *BIOS* and *HP Linux Firmware Update Utility* (or similar —
> HP reshuffles the page layout every few months). Grab the newest BIOS image
> and the latest `hp-flash` / `hpuefi-mod` bundle.

## Why bother

HP workstations treat a display adapter the way a bouncer treats a guest
without an invitation: no card, no POST. For a headless lab box that's the
first wall. The fix exists in firmware — `Headless Boot`, tucked under
`Advanced -> Error Handling Settings` — and the F10 UI will never show it to
you. You have to go out-of-band.

## What the firmware actually says

Strings recovered from the BIOS UEFI volume with
[UEFIExtract](https://github.com/LongSoft/UEFITool/releases) (A75):

```bash
curl -sL -o uefiextract.zip \
  "https://github.com/LongSoft/UEFITool/releases/download/A75/UEFIExtract_NE_A75_x64_linux.zip"
unzip -o uefiextract.zip
./ueefiextract P60_0300.bin
```

Hunt the UTF-16LE needle:

```bash
grep -rl -a -P "H\x00e\x00a\x00d\x00l\x00e\x00s\x00s\x00" P60_0300.bin.dump
```

Found in `CDBB7B35-6833-4ED6-9AB2-57D2ACDDF6F0 / 595 PlatformRasConfigSmm /
1 PE32 image body.bin`. The string table reads:

```
M.2 SSD0 Error Handling
M.2 SSD1 Error Handling
Headless Boot
\Advanced\Error Handling Settings
NVMe Write Endurance Masking
Advanced Error Control
Accept Liability for Circumventing POST Error Messages
Slot 1 Error Handling
... (Slot 2..10)
Disable Enable Decline Accept        <- shared option strings
```

Naming trivia: HP Zx40 / Z4 G4 call it `Headless Mode`. The Z6 G4 says
`Headless Boot` (`Disable` / `Enable`). Same idea, different label — check
the name on *your* machine before grepping.

## Toolbox

| SoftPaq | Contents | Role |
|---|---|---|
| `sp171387.tgz` | `hp-flash-3.26`, `hpuefi-mod-3.07` | The workhorses (Linux) |
| `sp173131.tgz` | BIOS `P60_0300.bin` + tools | Newest BIOS in one box |
| `sp87572.tgz` | `UefiBiosConfig.efi`, `CustomLogoApp.efi` | UEFI-Shell route (no OS required) |

Kernel warning: `hpuefi-mod` officially supports kernels up to **6.18**. It
built and loaded fine on **7.0.0-31-generic** — the build script regenerates
`hpuefi.c`/`.h` from templates and chases kernel API changes
(`vm_flags_set` >= 6.3, `memdesc_flags_t` / `flags.f` >= 6.18), so it keeps
up better than its release notes admit.

## Procedure (Linux route, as actually run)

Everything on the Z6 G4 itself, as a normal user with `sudo`.

### 1. Prerequisites

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
ls /lib/modules/$(uname -r)/build   # headers must match the running kernel
```

### 2. Unpack the tools

```bash
mkdir -p ~/hptools && cd ~/hptools
tar xzf /path/to/sp171387.tgz
tar xzf non-rpms/hp-flash-3.26_x86_64.tgz
tar xzf non-rpms/hpuefi-mod-3.07.tgz
```

### 3. Handle Secure Boot

`hpuefi.ko` is **unsigned**. Under Secure Boot it refuses to load, and it
would be right to.

```bash
mokutil --sb-state
```

If `enabled`: disable it in F10 (`Security -> Secure Boot Configuration ->
Disabled`) and reboot — or sign the module and enroll a MOK key. For a lab
box, disabling was the boring, correct choice.

### 4. Build and load the kernel module

```bash
cd ~/hptools/hpuefi-mod-3.07
make
sudo make install
sudo modprobe hpuefi
sudo /lib/modules/$(uname -r)/kernel/drivers/hpuefi/mkdevhpuefi
ls -l /dev/hpuefi     # must exist
```

### 5. Install the user-space tools

```bash
cd ~/hptools/hp-flash-3.26_x86_64
sudo ./install.sh     # installs to /opt/hp/hp-flash, picks matching OpenSSL build
/opt/hp/hp-flash/hp-repsetup -h
```

### 6. Backup — always

```bash
cd ~/hptools
/opt/hp/hp-flash/hp-repsetup -g HPSETUP_orig.TXT   # UCS-2 / UTF-16LE
iconv -f UTF-16LE -t UTF-8 HPSETUP_orig.TXT | grep -A2 -i "Headless Boot"
```

Expected block (`*` marks the current selection):

```
Headless Boot
*Disable
Enable
```

### 7. Flip the setting

Full-file route (edit a copy, convert back):

```bash
iconv -f UTF-16LE -t UTF-8 HPSETUP_orig.TXT > h.txt
sed -i '/^Headless Boot$/{n;s/^\*Disable/ Disable/;s/ Enable/\*Enable/}' h.txt
iconv -f UTF-8 -t UTF-16LE h.txt > HPSETUP_mod.TXT
iconv -f UTF-16LE -t UTF-8 HPSETUP_mod.TXT | grep -A2 -i "Headless Boot"  # verify *Enable
```

Lazy route (and the recommended one): a minimal file containing just this
block —

```
Headless Boot
*Enable
Disable
```

— saved as UTF-16LE **without** byte-order mark (HP's native format). A
ready-made `HeadlessBoot_enable.utf16le` ships with this repo.

### 8. Apply and reboot

```bash
/opt/hp/hp-flash/hp-repsetup -s HPSETUP_headless.TXT   # minimal file recommended
sudo reboot
```

The `-s` run will spew `Access Denied` / `Unsupported` / `Invalid Parameter`
for read-only and inventory fields (serial numbers, versions, DIMM labels).
Expected, benign, faintly delightful. Success = `Headless Boot` is **not**
among the failures.

### 9. Verify, then pull the GPU

```bash
/opt/hp/hp-flash/hp-repsetup -g HPSETUP_check.TXT
iconv -f UTF-16LE -t UTF-8 HPSETUP_check.TXT | grep -A2 -i "Headless Boot"
```

Reads `*Enable`? Shut down, disconnect power, remove the GPU, boot. No
display adapter, no complaint — the Z6 G4 comes up.

## Gotchas, collected

- **The F10 text-file trick is a dead end** on G4s. Out-of-band it is.
- **Setting names are platform-specific** (`Headless Mode` vs `Headless Boot`).
- **Secure Boot** blocks the unsigned module; plan for it before you need it.
- **Kernel drift**: headers must match the running kernel exactly. The hpuefi
  build scripts handle 6.3+/6.18+ API changes — not your missing headers.
- **`-s` error spam is normal.** Judge success by what is *not* failing.

## Disclaimer

`hp-flash`, `hp-repsetup`, and `hpuefi-mod` are HP's, shipped as-is and
unsupported. Firmware surgery is on you. Snapshot the original settings first
(`hp-repsetup -g`), and keep a GPU on the shelf until headless boot is
verified.
