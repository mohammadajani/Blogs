# Chasing Ghosts: Getting an Unbranded AX900 WiFi 6 + BLE Dongle Working on Arch

I bought a USB adapter called the **AE900X** — 2.4GHz + 5GHz WiFi 6, BLE 5.4, no visible chipset markings anywhere on the shell. The kind of thing that shows up under a dozen different brand names on AliExpress for the price of a coffee. I wanted it running on my CachyOS box. It fought back the entire way, and along the way it turned out to be a completely different chip than I thought it was. Here's the whole trail.

## Step one: it's not even a network adapter yet

Plug it in on Windows or Linux, and before any driver is installed, it doesn't show up as a WiFi adapter at all — it enumerates as a tiny 1.9MB read-only USB storage drive, carrying the Windows driver installer. Classic zero-CD driver delivery, the same trick a lot of 3G/4G USB modems use. Until something flips it out of that mode, that's all it will ever be.

## The first guess (wrong)

Nothing on the case says what chip is inside, so I did what anyone does: pattern-matched against the internet. A huge number of visually identical "AX900" dongles turn out to use a Realtek RTL8851BU. Different sellers, same photos, same plastic shell, same spec sheet — it's a common rebrand pattern. So that became the working theory, and I went and installed `rtw89-dkms-git` from the AUR on the strength of it.

In hindsight: reasonable first guess, wrong chip entirely.

## `lsusb` throws a curveball

Before touching drivers, the sane move is to actually look at the device. `lsusb` didn't show anything obviously Realtek. Instead there was this:

```
Bus 001 Device 014: ID 1111:1111 Pandora International Ltd. 88M80
```

`1111:1111` isn't really "Pandora International" — it's a well-known placeholder VID:PID that a bunch of unrelated cheap peripherals reuse because nobody bothered registering a real one. `dmesg` filled in more: manufacturer string `AIC`, product string `88M80`, and the SCSI inquiry data reporting vendor `LGX`, model `WIFI6`. None of that screamed Realtek. It screamed "nobody's indexed this specific product online."

I tried the obvious trick — `eject`ing the storage partition to force a mode switch, the same move that works for USB modems. It did *something* (capacity briefly reported as zero), but it didn't stick. Unplug and replug, and it came right back as the same 1.9MB drive. Whatever this device needed, a plain eject wasn't it.

## Cracking open the installer

At that point the only honest move was to stop guessing and look at the actual driver. Since the storage partition *is* the driver delivery mechanism, the Windows installer (`Wifi6_install_bt.exe`) was sitting right there on the mounted drive — no need to touch Windows at all. Pulled it off, ran it through `7z` as an NSIS archive, and the `.inf` files inside told the real story:

```
%AIC.DeviceDesc_368b_8d81% = aic.ndi, USB\VID_368b&PID_8d81&MI_02
AIC.DeviceDesc_368b_8d81 = "Wifi6 802.11ax USB Adapter"
```

`368b` is AICSemi's real registered USB vendor ID. This wasn't a Realtek dongle at all — it's an **AICSemi AIC8800D80**, a WiFi 6 + BT5 combo chip. A second `.inf` (`aicloadfw.Inf`) revealed the missing piece: an intermediate bootROM stage at `a69c:8d80` that needs firmware pushed to it before the chip ever exposes the real `368b:8d81` WiFi interface. Three stages total: storage mode → bootROM → functional device. The plain `eject` trick was never going to trigger a firmware push.

Correcting course, `rtw89-dkms-git` came back out — wrong chip family entirely, harmless to have installed, useless for this hardware.

## Two drivers, two different kinds of broken

Digging around turned up two community efforts for this exact chip:

- **kilam994/aic8800d80-linux-driver** — explicitly documents this precise "Pandora clone" variant (`1111:1111 → a69c:8d80`, handled via a proper `usb_modeswitch` config with the raw SCSI command bytes), bundled with firmware and a DKMS module.
- **ronnyf/AIC8800-Linux-Driver** — a more actively maintained fork with an actual pacman repo, explicitly tested on CachyOS, with recent commits patching the driver for breaking changes in newer kernels.

First attempt: kilam994's installer. It correctly laid down firmware and udev rules, then the DKMS build itself died on `kernel 7.2.0-1-cachyos` — bleeding-edge kernel, driver source too old for it.

Second attempt: ronnyf's driver via their pacman repo. Same failure, different specifics — the build log showed two real compile errors: a call to `strncpy()` that no longer exists as an implicit declaration in modern kernels, and a function pointer signature mismatch on `.remain_on_channel`, which gained a new `rx_addr` parameter upstream. Both are exactly the kind of breaking changes that kill out-of-tree drivers overnight.

The interesting part: cloning ronnyf's repo fresh and reading the *actual current source* showed both bugs already fixed on the `main` branch — `strscpy()` in place, the new parameter handled. The **packaged pacman binary was just stale**, built before those fixes landed. So: skip the package, build straight from git `main` by hand. That compiled clean and loaded — `lsmod` showed `aic8800_fdrv`, `aic_load_fw`, and `cfg80211` all present, and dmesg confirmed both modules registering with the USB core.

## Loaded, but still stuck in storage mode

Modules loaded, device still showing as `1111:1111`. Turned out ronnyf's driver ships its own udev rule (`aic.rules`), but it only auto-ejects three *known* bootROM variants (`a69c:5721/5723/5724`) — none of which is the `1111:1111` placeholder this specific clone uses. kilam994's installer *had* written the correct rule for this — but to `/usr/lib/udev/rules.d/aic.rules`, while ronnyf's manual install wrote to `/etc/udev/rules.d/aic.rules`. Same filename, different priority directory — and `/etc/` silently wins, completely shadowing the other file. kilam994's correct `1111:1111 → a69c:8d80` switch rule had effectively gone dark the moment the second file landed.

Fix was simple once found: merge the missing `1111:1111` rule into the winning file, reload udev, replug.

## Where things stand right now

- **WiFi**: driver loads, the device finally leaves storage mode, and it can see and scan networks. Actually *connecting* to one is the current open problem — diagnosing whether it's a power-save, MAC-randomization, or handshake-level issue.
- **Bluetooth/BLE**: confirmed not supported by this driver fork at all — its own "Supported Features" list is WiFi-only, despite some dormant Bluetooth source files still sitting in the tree from the original vendor SDK. A project for later.

## What actually mattered, in hindsight

The single highest-leverage move in this whole process was pulling the driver installer off the device itself and reading the `.inf` files — that's what turned "probably Realtek, going by internet vibes" into a confirmed chipset, exact VID:PID chain, and the precise three-stage mode-switch mechanism, in one step. Everything after that was standard (if occasionally frustrating) Linux driver triage: reading DKMS build logs past the first error, checking whether a packaged binary matches its own upstream source, and — the one that cost the most time — remembering that two files with the same name in different udev directories don't merge, they shadow.

More soon, once the connection issue is sorted and BLE gets its own investigation.
