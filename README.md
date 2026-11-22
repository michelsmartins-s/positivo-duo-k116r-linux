# Ubuntu 24.04 on the Positivo Duo K116R

Getting Ubuntu 24.04 fully working on the **Positivo Duo K116R** (2-in-1, Intel Celeron Gemini Lake, 11.6"), including the problems that most guides do not cover: missing Wi-Fi interface, touchscreen offset, screen flicker and tablet mode.

> Português: veja [README.pt-BR.md](README.pt-BR.md)

## Status

| Component | Status | Notes |
|---|---|---|
| Video (Intel UHD, i915) | Works | Needs kernel parameters, see below |
| Wi-Fi (RTL8723BU) | Works, intermittent | Interface may be missing after a boot (EFuse error). Power reset fixes it |
| Bluetooth | Works | Same chip as Wi-Fi |
| Touchscreen (Silead MSSL1680) | Works | Needs firmware + range correction |
| Keyboard / touchpad | Works | Disabling them in tablet mode needs a script |
| Camera | Detected | Not tested further |
| Audio | Detected | Not tested further |
| Screen rotation | Not tested | |

## Tested system

- Ubuntu 24.04, kernel `6.8.0-40-generic`
- BIOS: AMI Aptio, version 2.20.1273 (2021), UEFI only (no CSM)
- Boot menu: `F7`. Setup: `Del` or `F2`
- Storage: eMMC (`mmcblk1`)

## Hardware

| Part | Detail |
|---|---|
| CPU/GPU | Intel Celeron (Gemini Lake), Intel UHD Graphics |
| Wi-Fi/BT | Realtek RTL8723BU (USB `0bda:b720`) |
| Touchscreen | Silead MSSL1680 (I2C/ACPI) |
| Card reader | Realtek RTS5129 |

## 1. Screen flickers / restarts the graphical session

Add kernel parameters in `/etc/default/grub`:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash i915.enable_psr=0 i915.enable_dc=0 usbcore.autosuspend=-1"
```

Then:

```
sudo update-grub
```

Do **not** use `acpi=off`: it disables the touchscreen, keyboard and the eMMC.

## 2. Wi-Fi interface missing (no `wlan0`)

Symptoms: `lsusb` shows the RTL8723BU, `rtl8xxxu` is loaded, firmware exists, but `ip link` shows only `lo`. In `dmesg`:

```
usb 1-4: Fatal - failed to read EFuse
rtl8xxxu: probe of 1-4:1.2 failed with error -5
```

Fix that worked:

1. Shut down.
2. Unplug the charger.
3. Hold the power button for 15 to 20 seconds (nothing will happen on screen).
4. Wait 1 minute, plug the charger, boot.

The interface then appeared as `wlx<mac>`. This was combined with `usbcore.autosuspend=-1`. The problem may come back on a later boot, so the power reset is a workaround, not a confirmed permanent fix. Do not call `modprobe rtl8xxxu` repeatedly while the chip is in a bad state: it can hang.

## 3. Touchscreen does not work

`dmesg` shows:

```
silead_ts i2c-MSSL1680:00: Direct firmware load for silead/mssl1680.fw failed with error -2
```

The firmware is **not** in the `linux-firmware` package. Get it from the community repository:

```
sudo apt install -y git
git clone https://github.com/onitake/gsl-firmware.git
sudo mkdir -p /lib/firmware/silead
sudo cp $(find gsl-firmware -name "gsl1680-positivo-c4128b.fw" | head -1) /lib/firmware/silead/mssl1680.fw
sudo update-initramfs -u
```

The file `gsl1680-positivo-c4128b.fw` (from another Positivo model) worked on the K116R. The firmware file is not included in this repository.

## 4. Touchscreen works but pointer is offset / scaled

Symptom: top-left corner is correct, center lands near the top-left, bottom-right lands near the center.

Cause: the driver advertises `0-4095` on both axes, but the panel only reports about `0-1890` (X) and `0-1258` (Y). Measure with:

```
sudo evtest /dev/input/eventN | grep --line-buffered ABS_MT_POSITION
```

Create `/etc/udev/hwdb.d/99-silead-touch.hwdb` (see [files/99-silead-touch.hwdb](files/99-silead-touch.hwdb)):

```
evdev:name:silead_ts:*
 EVDEV_ABS_35=0:1900:1
 EVDEV_ABS_36=0:1270:1
```

Each `EVDEV_ABS` line must start with one space. Apply:

```
sudo systemd-hwdb update
sudo udevadm trigger
sudo reboot
```

Note: `LIBINPUT_CALIBRATION_MATRIX` udev rules had no effect under GNOME here.

## 5. Disable keyboard and touchpad in tablet mode

Whether the firmware reports the fold event to Linux was not tested (run `sudo libinput debug-events` while folding the screen to check). Until then, a manual toggle is used: `evtest --grab` on the internal keyboard plus disabling the touchpad via `gsettings`. See [files/toggle-teclado](files/toggle-teclado) and [files/toggle-teclado.desktop](files/toggle-teclado.desktop).

Requires a sudoers rule:

```
youruser ALL=(root) NOPASSWD: /usr/bin/evtest, /usr/bin/pkill
```

Tip: keep an external keyboard or the on-screen keyboard available when testing.

Install:

```
sudo cp files/toggle-teclado /usr/local/bin/toggle-teclado
sudo chmod +x /usr/local/bin/toggle-teclado
mkdir -p ~/.local/share/applications
cp files/toggle-teclado.desktop ~/.local/share/applications/
```

Then add the sudoers rule above with `sudo visudo -f /etc/sudoers.d/toggle-teclado`. The GitHub web interface does not keep the executable bit, so `chmod +x` is required.

## 6. Recovering a boot problem (EFI Shell)

If the machine drops into `Shell>`, boot manually:

```
map -r
FS0:
FS0:\EFI\ubuntu\shimx64.efi
```

Then fix it permanently:

```
sudo grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu
sudo update-grub
```

## 7. Ubuntu 26.04 note

An early test with Ubuntu 26.04 booted with working video, but no Wi-Fi. Ubuntu 24.04 is what this guide is based on.

## Contributing

Tested on one unit only. Issues and pull requests with results from other units or distributions are welcome.

## License

MIT. Firmware files belong to their respective authors.
