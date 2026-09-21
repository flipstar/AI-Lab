# Local AI Lab

A short, honest build log: one retired HP Z6 G4 workstation, two cheap mining
GPUs, and the question of how far local AI can go without a datacenter, a
monitor, or a credit-card bill.

## Why

I've been watching the AI transformation from the outside for a while now.
Watching builds headlines; understanding builds intuition. And the fastest
way to understand the details of any stack is to get your fingers dirty on
every single layer of it — firmware, drivers, thermals, all of it. So: a lab
box, documented as I go.

## The plan

| Piece | Pick | Why |
|---|---|---|
| Host | HP Z6 G4 workstation | Room for several full-length cards, dual-socket headroom, big PSU, cheap used |
| Compute | 2× NVIDIA CMP 50HX (TU102), 10 GB each | RTX 2080 Ti silicon at mining-card prices — 20 GB aggregate |
| Display | GT 710 (1-slot, low-profile) | Case is tight; the small card fills the leftover slot and gives a screen |
| OS | Ubuntu amd64 | Boring is a feature |

## The GPU gamble

The CMP 50HX is NVIDIA's mining SKU: the same TU102 as a 2080 Ti, deliberately
crippled for the mine. Locked PCIe (Gen1 x4), no ReBAR, throttled SM issue
rate, and an idle draw it will not let itself shed. Buy two and you've got
serious compute for the price of a used midrange card each — *if* you can get
past the lockout.

You can. The [cmp50hx-unlock](https://github.com/xrip/cmp50hx-unlock) project
patches the NVIDIA open kernel modules (plus an optional pre-OS UEFI path) to
unlock full compute, a 16 GiB BAR1 via ReBAR, and a real PCIe Gen2 x4 link at
5.0 GT/s. One-click install, verified to RTX 2080 Ti-class performance — an
A/B run measured FP32 go from 0.43 to 13.5 TFLOP/s, a ~31× jump. The honest
fine print: the 56 reported RT cores are a reporting patch, not real — the
physical RT fuse is still dead, and the card idles at 62–64 W until you add
their idle governor, which drops it to ~1.8 W.

That trade — mining price, 2080 Ti compute, no RT cores — is exactly the sweet
spot for a local inference lab. So we take it.

## Fitting it all in

The catch is physical. Two TU102 mining cards are big, and the Z6 G4's
expansion area is tight. Fitting both means careful slot spacing, and it leaves
only room for a 1-slot, low-profile card in what's left over. That's where the
GT 710 lives — a dirt-cheap display adapter that gives the box a screen and
keeps the fit clean. Headless boot (below) means I'm never hostage to it: pull
the GT 710 and the machine still boots, which is the whole point of having
unlocked it.

## Topics

| File | What's in it |
|---|---|
| [headless-bios-mode.md](HP_Z6_G4/headless-bios-mode.md) | Making the Z6 G4 boot with no display card installed |
| [cmp50hx-rom.md](CMP50HX/cmp50hx-rom.md) | Swapping in the MSI-tuned VBIOS: the 5.692 quirk, the `-6` flag, recovery |
| [cmp50hx-unlock.md](CMP50HX/cmp50hx-unlock.md) | Installing and verifying the 50HX unlock, before/after numbers |
| [platform.md](HP_Z6_G4/platform.md) | Hardware as-built: CPU, RAM, slot layout, PSU |
| [os-and-drivers.md](os-and-drivers.md) | Ubuntu, kernel, NVIDIA driver versions |
| [compute-stack.md](compute-stack.md) | Two-card / 20 GB VRAM budget, what actually runs |
| [thermals-and-power.md](thermals-and-power.md) | Idle governor, power caps, PSU headroom |
| [storage.md](storage.md) | Drives, where models and datasets live |
| [gotchas.md](gotchas.md) | Every mistake, with the fix |

## Status

- [x] Headless boot verified (boots with no display card)
- [ ] 50HX unlock installed
- [ ] Two-card fit + GT 710 in the case, idle + load test
- [ ] First local model end-to-end
