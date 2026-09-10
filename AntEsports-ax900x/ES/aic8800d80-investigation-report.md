# Technical Investigation Report: Chipset Identification and Linux Driver Enablement for an Unbranded "AE900X" USB WiFi 6 + BLE Adapter

**Classification:** Hardware / Driver Reverse-Engineering Case Study
**Target Device:** AE900X-branded (rebadged) USB 2.4/5GHz WiFi 6 + BLE 5.4 adapter
**Host Environment:** CachyOS (Arch-based), kernel `7.2.0-1-cachyos`
**Investigator:** anonmahaa
**Date:** September 2026

---

## 1. Executive Summary

An unbranded USB WiFi 6 + BLE 5.4 adapter, sold under the generic label "AE900X," was procured with no chipset markings, no manufacturer documentation, and no indexed community identification. The device exhibited USB mass-storage behavior on first connection — a zero-driver delivery mechanism — obscuring its true identity behind a shared/placeholder USB Vendor:Product ID (`1111:1111`) commonly reused by unrelated white-label peripherals.

Community pattern-matching based on visual and marketing similarity to other "AX900"-class products led to an initial, incorrect chipset hypothesis (Realtek RTL8851BU). This hypothesis was falsified through static binary analysis of the vendor-supplied Windows driver package, which was extracted directly from the device's own storage partition without requiring a Windows environment. Analysis of the extracted `.inf` driver metadata positively identified the chipset as an **AICSemi AIC8800D80** WiFi 6 + Bluetooth combo SoC, and revealed a three-stage USB enumeration/mode-switch sequence required to reach functional state.

Using this ground-truth identification, two independent open-source Linux driver implementations were evaluated, cross-referenced, and combined to produce a working WiFi driver on the target kernel. Two independent defects were identified and resolved during this process: (1) a stale pre-built binary package that predated upstream kernel-compatibility fixes present in the same project's source repository, and (2) a udev rule file-shadowing condition that silently disabled a required device mode-switch rule. As of this report, WiFi scanning is functional; WiFi association and Bluetooth/BLE support remain open items, the latter confirmed to be unimplemented in the current driver rather than defective.

## 2. Scope & Objective

**In scope:**
- Identification of the actual silicon/chipset inside an unmarked, unbranded USB peripheral
- Static analysis of the vendor's Windows driver distribution package
- Enablement of Linux (Arch/CachyOS) kernel-level support for the identified chipset
- Root-cause analysis of build and configuration failures encountered during enablement

**Out of scope:**
- Firmware-level or RF-level analysis of the chipset itself
- Security assessment of the vendor driver package (e.g., malware analysis) — the package was treated as a legitimate driver payload, not a suspect binary
- Bluetooth/BLE enablement (identified as a future work item, not attempted)

## 3. Methodology

1. **Passive enumeration reconnaissance** — `lsusb`, `lsblk`, and `dmesg` used to characterize the device's USB identity and behavior across power cycles, without installing any driver.
2. **Open-source intelligence (OSINT) cross-referencing** — community reports on visually/nominally similar products used to form an initial chipset hypothesis.
3. **Static binary analysis** — the vendor's Windows driver installer (`Wifi6_install_bt.exe`, an NSIS self-extracting archive) was extracted using `7z` and inspected for `.inf` driver metadata, without executing the binary.
4. **Comparative source analysis** — two independent third-party Linux driver implementations for the identified chipset were located, cloned, and compared against each other and against the target kernel's API surface.
5. **Build log triage** — DKMS compilation failures were traced to their root compiler errors (as opposed to relying on summary/tail output) to distinguish environmental issues from genuine source incompatibilities.
6. **Source-vs-package differential analysis** — the installed binary package was compared against the same project's live git repository to determine whether a build/release lag existed.
7. **System configuration audit** — udev rule directories were audited for file-name collisions and priority-based shadowing after a compiled and loaded driver failed to bind to the target device.

## 4. Timeline of Investigation

| Stage | Action | Outcome |
|---|---|---|
| 1 | Device enumerated via `lsusb`/`lsblk` on first connection | Identified as USB mass storage, VID:PID `1111:1111` (generic/shared placeholder ID) |
| 2 | Chipset hypothesis formed via product-similarity OSINT | Hypothesized Realtek RTL8851BU (later disproven) |
| 3 | `rtw89-dkms-git` installed based on hypothesis | Installed successfully but irrelevant — wrong chipset family |
| 4 | Manual `eject`-based mode-switch attempted | Transient effect only; did not persist across reconnection |
| 5 | Vendor Windows installer extracted directly from device storage partition via `7z` | `.inf` metadata revealed true chipset: AICSemi AIC8800D80, VID `368b` |
| 6 | Secondary `.inf` (`aicloadfw.Inf`) analyzed | Revealed intermediate bootROM stage `a69c:8d80` requiring firmware push |
| 7 | `rtw89-dkms-git` removed | Cleanup of incorrect prior assumption |
| 8 | Driver candidate 1 (`kilam994/aic8800d80-linux-driver`) installed | Correct device-specific `usb_modeswitch` config installed; DKMS build failed on target kernel |
| 9 | Driver candidate 2 (`ronnyf/AIC8800-Linux-Driver`) installed via maintainer's pacman repository | Build failed; root cause isolated via `make.log` to two kernel-API incompatibilities |
| 10 | Git source of candidate 2 cloned and diffed against installed package | Confirmed both incompatibilities already fixed upstream; installed binary release was stale |
| 11 | Driver built manually from current git source | Compiled and loaded successfully; modules registered with USB core |
| 12 | Device still not transitioning out of storage mode | Root-caused to udev rule file-shadowing between two vendor-supplied rule files of identical name in different priority directories |
| 13 | Missing device-specific switch rule merged into the effective (higher-priority) rule file | Device successfully transitioned through all three USB stages; WiFi interface exposed |
| 14 | Functional verification | WiFi network scanning confirmed operational |

## 5. Technical Findings

### 5.1 Finding: Chipset Misidentification Risk via Generic/Shared USB Identifiers

The device enumerated under VID:PID `1111:1111`, a non-unique identifier reused across unrelated products by vendors who have not obtained a registered USB Vendor ID. This is a materially different situation from a misconfigured or spoofed ID — it reflects a broader pattern among low-cost, unbranded OEM/ODM hardware in which USB identification data cannot be trusted as a reliable chipset fingerprint. Community-sourced pattern-matching (visual/marketing similarity to other products) produced an incorrect chipset hypothesis and led to installation of an irrelevant driver package before ground-truth verification was performed.

**Risk/impact:** Time and system-state cost (unnecessary package installation); in a security-sensitive context, reliance on unverified community identification of hardware could plausibly be leveraged to mask the true nature of a device — a broader supply-chain-adjacent consideration beyond the scope of this specific investigation.

### 5.2 Finding: Multi-Stage USB Mode-Switch Mechanism

The chipset implements a three-stage USB enumeration sequence rather than presenting its functional interface directly:

```
1111:1111  (generic mass-storage placeholder, driver delivery stage)
      -> a69c:8d80  (bootROM stage, awaiting firmware push)
            -> 368b:8d81  (functional WiFi 6 + BT composite device, MI_02 = WLAN)
```

The transition from stage 1 to stage 2 requires a vendor-specific raw SCSI Command Block Wrapper, not a generic SCSI `eject`/media-removal command (which was empirically confirmed to have only a transient, non-persistent effect). The exact command was recovered from a third-party `usb_modeswitch` configuration file:

```
DefaultVendor=0x1111
DefaultProduct=0x1111
TargetVendor=0xa69c
TargetProduct=0x8d80
MessageContent="555342438765432100000000000010fd0000000000000000000000000000f2"
```

### 5.3 Finding: Driver Distribution Channel Staleness

The maintainer-provided binary package (`aic8800-fdrv-dkms`, via a third-party pacman repository) failed to compile against the target kernel due to two upstream Linux kernel API changes:

1. Removal of the implicit-declaration allowance for `strncpy()` (kernel enforces explicit declaration under `-Wimplicit-function-declaration` as an error under Clang/LLVM builds)
2. A signature change to `struct cfg80211_ops.remain_on_channel`, which gained an additional `rx_addr` parameter

Both issues were confirmed, via direct inspection of the project's `main` branch, to already be resolved in source — the distributed binary package simply had not been rebuilt/re-released to reflect these fixes. This represents a **build-artifact/release synchronization gap** rather than a defect in the upstream project's active source.

**Risk/impact:** Consumers relying solely on the packaged binary (the maintainer's own "recommended" installation path) would encounter a hard failure with no indication that a fix already existed one layer upstream. This is a general risk pattern for any project distributing pre-built binaries alongside actively developed source: without version pinning or a public changelog cross-reference, end users cannot distinguish "unsupported configuration" from "unreleased fix."

### 5.4 Finding: Silent udev Rule Shadowing Across Priority Directories

Two independent, correctly-authored udev rule files — one from each of the two evaluated driver projects — were both named `aic.rules`, but installed to different directories in the udev rule search path:

- `/usr/lib/udev/rules.d/aic.rules` (lower priority; contained the required rule for this device's specific mode-switch case)
- `/etc/udev/rules.d/aic.rules` (higher priority; did not contain a rule for this device's specific case)

udev resolves rule files by name across its search directories using directory priority, with no merging of same-named files. The higher-priority file completely masked the lower-priority file's contents, silently disabling the correct mode-switch trigger with no error, warning, or log output indicating the conflict.

**Risk/impact:** This is a general configuration-management risk applicable beyond this specific case: combining udev rules (or any priority-directory-resolved configuration, e.g. systemd unit drop-ins, sudoers.d, polkit rules) from multiple independently-authored sources without auditing for filename collisions can result in a fully silent loss of intended functionality, with no diagnostic signal pointing to the cause.

## 6. Artifacts / Indicators

| Type | Value |
|---|---|
| USB VID:PID (storage/delivery stage) | `1111:1111` |
| USB VID:PID (bootROM/firmware-load stage) | `a69c:8d80` |
| USB VID:PID (functional WiFi+BT stage) | `368b:8d81` (WLAN interface: `MI_02`) |
| SCSI manufacturer string | `AIC` |
| SCSI product string | `88M80` |
| SCSI INQUIRY vendor/model | `LGX` / `WIFI6` |
| Confirmed chipset | AICSemi AIC8800D80 |
| Driver source (WiFi, kernel 7.x compat) | `github.com/ronnyf/AIC8800-Linux-Driver` |
| Driver source (device-specific mode-switch config) | `github.com/kilam994/aic8800d80-linux-driver` |
| Host kernel | `7.2.0-1-cachyos` |

## 7. Recommendations / Lessons Learned

1. **Do not treat community/OSINT chipset identification as ground truth for unbranded hardware.** Where feasible, extract and statically analyze the vendor's own driver package before committing to a driver installation — in this case the vendor's driver was directly retrievable from the device's own storage partition, requiring no Windows environment.
2. **When an out-of-tree driver's packaged binary fails to build, check the project's live source before concluding the project is unmaintained or unsupported.** A build failure may reflect a release/packaging lag rather than an actual gap in upstream support.
3. **Always inspect full build logs, not summary/tail output**, when triaging compilation failures — the root-cause `error:` lines in this investigation were not present in the DKMS-provided summary and required direct log inspection.
4. **Audit for filename collisions when combining configuration from multiple independently-authored sources**, particularly in priority-directory-resolved systems (udev, systemd, polkit, sudoers.d). Absence of an error does not imply absence of a conflict.

## 8. Current Status / Open Items

| Item | Status |
|---|---|
| Chipset identification | Complete |
| Mode-switch chain reverse-engineered | Complete |
| Driver build (kernel 7.2.0-1-cachyos) | Complete |
| WiFi scan functionality | Verified working |
| WiFi association/connection | Open — under active debugging (power-save and MAC-randomization interactions suspected) |
| Bluetooth/BLE support | Not implemented in current driver (confirmed via upstream documentation; not a defect) |

## 9. Conclusion

This investigation demonstrates that reliable hardware enablement for unbranded, unidentified USB peripherals is achievable without vendor documentation, provided the vendor's own driver distribution mechanism is treated as a legitimate source of ground-truth technical data rather than relying solely on community inference. The methodology applied here — enumeration reconnaissance, static analysis of vendor driver packages, comparative source review, and systematic build/configuration failure triage — generalizes to similar hardware bring-up problems beyond this specific device.
