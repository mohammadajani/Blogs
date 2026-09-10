# Getting an Unlabeled AX900-Type WiFi + BLE Dongle Working on Arch

I picked up a USB adapter called the AE900X — WiFi 6, 2.4 and 5GHz, BLE 5.4, all printed on the box, nothing printed on the actual device. No chipset name anywhere on the shell, no datasheet, nothing. Plug it into Windows or Linux and before you even install anything, it shows up as a tiny 1.9MB storage drive carrying the driver — the classic zero-CD trick a lot of cheap dongles and old USB modems use to hand you a driver without needing an internet connection. Install whatever's on that drive and it flips into being an actual WiFi adapter. Simple enough in theory.

Since nothing on the case told me what was actually inside it, I did what anyone does and looked it up. A lot of visually identical "AX900" dongles kept turning up as Realtek RTL8851BU under different seller names — same shape, same plastic, same everything, just rebranded. So that became my working guess. Reasonable enough starting point, I thought.

Then I actually checked what Linux itself was seeing, and it didn't match.

```
$ lsusb
Bus 001 Device 014: ID 1111:1111 Pandora International Ltd. 88M80

$ lsblk
sdb      8:16   1   1.9M  1 disk
└─sdb1   8:17   1   1.9M  1 part
```

Nothing Realtek at all. `1111:1111` isn't even a real company — it's one of those generic placeholder IDs a bunch of unrelated cheap devices reuse because nobody bothered registering a proper vendor ID. `dmesg` filled in a bit more:

```
usb 1-4: New USB device found, idVendor=1111, idProduct=1111
usb 1-4: Product: 88M80   Manufacturer: AIC
scsi 4:0:0:0: Direct-Access     LGX      WIFI6            2.30
```

None of that matched Realtek, but at that point I still didn't have anything solid to go on either, so the guess stood.

I tried the obvious fix first — ejecting the storage partition to force it to switch modes, the same trick that works on USB modems:

```
sudo eject /dev/sdb1
```

It did something, briefly — dmesg logged `sdb: detected capacity change from 3968 to 0` — but the moment I unplugged and replugged it, it went right back to being a plain storage drive again. Whatever this thing actually needed to wake up, a plain eject wasn't it.

I went ahead and installed a Realtek driver anyway, on the strength of the original guess:

```
paru -S rtw89-dkms-git
```

It installed cleanly, DKMS registered it against both my kernels. It also did absolutely nothing, because it turned out later to be for a completely different chip.

This is where it got genuinely annoying for a while. Nothing was working, the device was still sitting there as a fake CD drive, and I didn't actually know what chip I was even dealing with — I was debugging blind. At some point it clicked that the driver installer itself was sitting right there on the device, in that 1.9MB partition, and I didn't need Windows at all to get at it. So I mounted it, pulled off the `.exe` (`Wifi6_install_bt.exe`, turned out to be an NSIS installer), and cracked it open with an archive tool instead of running it:

```
7z x -y -oextracted Wifi6_install_bt.exe
```

That's where the real answer was. Buried in the driver's `.inf` file was this line:

```
%AIC.DeviceDesc_368b_8d81% = aic.ndi, USB\VID_368b&PID_8d81&MI_02
AIC.DeviceDesc_368b_8d81 = "Wifi6 802.11ax USB Adapter"
```

Vendor `368b` — that's AICSemi, not Realtek. The chip was an AIC8800D80, a completely different WiFi6 + Bluetooth combo chipset than the one I'd been chasing this whole time. A second file in there (`aicloadfw.Inf`) explained the storage-drive behaviour too:

```
USB\VID_A69C&PID_8d80.DeviceDesc="AIC Load Fw Driver"
```

There's a hidden middle step where the chip sits in a sort of bootloader state (`a69c:8d80`) waiting for firmware to be pushed to it over USB, and only after that does it become the real WiFi interface (`368b:8d81`). Three stages total, not one switch. That explained why a plain eject was never going to do anything — it wasn't a mode switch, it needed an actual firmware push.

Out went the Realtek driver (`paru -R rtw89-dkms-git`). Now that I actually knew what chip this was, I found two different open-source Linux drivers written specifically for it: [kilam994/aic8800d80-linux-driver](https://github.com/kilam994/aic8800d80-linux-driver) and [ronnyf/AIC8800-Linux-Driver](https://github.com/ronnyf/AIC8800-Linux-Driver). The first one even documented this exact "clone" variant by name, ID and all, and shipped the precise low-level command needed to trigger that first switch. The second was a more actively maintained project with its own package repository, tested specifically on the same Linux distro I run. Between the two I figured I had everything I needed.

I was wrong, just in a more specific way this time. kilam994's installer laid down the firmware and udev rules fine, but the DKMS build itself died:

```
git clone https://github.com/kilam994/aic8800d80-linux-driver.git
sudo ./install.sh
...
[FAIL] DKMS build failed.
make[4]: *** [...] aic8800_fdrv Error 2
```

Too old for how new my kernel is. So I tried ronnyf's, via their own pacman repo:

```
sudo pacman -S aic8800/aic8800-fdrv-dkms
```

Failed too, but this time the actual compiler error was readable, and it pointed at two very specific things:

```
error: call to undeclared library function 'strncpy' ...
error: incompatible function pointer types initializing '.remain_on_channel = rwnx_cfg80211_remain_on_channel' ...
```

A function that no longer exists the way it used to in newer kernels, and a driver callback whose expected argument list had changed. Both very fixable, in theory — except the packaged version I'd installed didn't have the fix. So I went and looked at the project's actual current source code directly, and both problems were already patched there. The maintainer just hadn't repackaged a new release yet. The fix existed, it just hadn't reached the download I'd used.

Built it by hand straight from that source instead of the package:

```
git clone https://github.com/ronnyf/AIC8800-Linux-Driver.git
cd AIC8800-Linux-Driver
make LLVM=1 -C drivers/aic8800
sudo make -C drivers/aic8800 install
sudo modprobe aic_load_fw
sudo modprobe aic8800_fdrv
```

And this time it actually compiled. `lsmod | grep aic` showed `aic8800_fdrv`, `aic_load_fw`, `cfg80211` all loaded, nothing errored out. For the first time in this whole process something had gone right on the first attempt.

Except the dongle was still sitting there as a plain storage drive. Driver loaded, nothing to drive.

Turned out ronnyf's driver only knew how to auto-trigger the switch for three specific hardware variants, and mine wasn't one of them — it was kilam994's driver, the one with the failed build, that actually had the right trigger for my exact device. And it had installed its version correctly. The problem was that both drivers happened to name their udev rule file identically — `aic.rules` — and Linux keeps a couple of different folders it checks for these in a fixed order:

```
/etc/udev/rules.d/aic.rules       (ronnyf's — wins, doesn't know about my device)
/usr/lib/udev/rules.d/aic.rules   (kilam994's — correct, but shadowed and ignored)
```

The higher-priority folder wins completely, silently, no warning that anything else even existed. So the correct trigger for my device had been sitting there the entire time, just invisible. I merged the missing piece straight into the winning file:

```
KERNEL=="sd*", ATTRS{idVendor}=="1111", ATTRS{idProduct}=="1111", RUN+="/usr/sbin/usb_modeswitch -c /etc/usb_modeswitch.d/1111:1111"
```

reloaded udev, unplugged and replugged the dongle one more time — and this time `lsusb` didn't show `1111:1111` anymore. A real WiFi interface showed up instead, and a network scan actually returned real networks around me.

It's not fully there yet — I can see networks but can't connect to one just yet, still working through that, and Bluetooth/BLE isn't supported by this particular driver at all right now, confirmed straight from its own documentation rather than something broken on my end. But getting from an unmarked dongle with a fake vendor ID all the way to a working WiFi scan, on a driver I ended up half-building myself from source most people never open, feels like the actual hard part is behind me. The rest is just tuning from here.
