# IMX477 kernel driver for Raspberry Pi

[![Build](https://github.com/Kurokesu/imx477-rpi-driver/actions/workflows/build.yml/badge.svg)](https://github.com/Kurokesu/imx477-rpi-driver/actions/workflows/build.yml)
[![Code style](https://github.com/Kurokesu/imx477-rpi-driver/actions/workflows/code-style.yml/badge.svg)](https://github.com/Kurokesu/imx477-rpi-driver/actions/workflows/code-style.yml)
[![Release](https://img.shields.io/github/v/release/Kurokesu/imx477-rpi-driver)](https://github.com/Kurokesu/imx477-rpi-driver/releases/latest)
[![Kurokesu apt archive](https://img.shields.io/badge/apt-Kurokesu_archive-D70A53?logo=debian)](https://apt.kurokesu.com)
[![RPi OS Bookworm | Trixie](https://img.shields.io/badge/RPi_OS-Bookworm_%7C_Trixie-blue?logo=raspberrypi)](https://www.raspberrypi.com/software/operating-systems/)
[![Kernel 6.12+](https://img.shields.io/badge/kernel-6.12%2B-blue?logo=raspberrypi)](https://github.com/raspberrypi/linux/tree/rpi-6.12.y)

Raspberry Pi kernel driver for Sony IMX477, a 12.3 MP 1/2.3" stacked CMOS sensor with 1.55 µm pixels. Extends Raspberry Pi in-kernel `imx477` driver with 4-lane MIPI CSI-2 for higher frame rates.

- 2-lane and 4-lane MIPI CSI-2
- 8-bit, 10-bit and 12-bit RAW output
- 4056×3040 @ 30 fps (full resolution)
- 2028×1520 @ 119 fps (2×2 binning)
- 1332×990 @ 271 fps (2×2 binning crop)
- Frame sync source and sink
- Drop-in replacement for in-kernel driver

![Kurokesu camera modules connected to a Raspberry Pi 5.](https://raw.githubusercontent.com/Kurokesu/imx477-rpi-driver/main/docs/kurokesu-on-pi.jpg)

*IMX477 camera modules are available at [kurokesu.com](https://www.kurokesu.com/item/477C-CSI)*

## Install

Connect camera to CSI port with Pi powered off.

Update OS and reboot:

```bash
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

> [!IMPORTANT]
> If driver or camera stack was previously built from source, run one-time cleanup before first apt install. See [migrating from a source install](#migrating-from-a-source-install).

Enable Kurokesu apt archive (skip if already enabled):

```bash
curl -fsSLO https://apt.kurokesu.com/setup.sh
sudo sh setup.sh
```

Install driver and camera stack:

```bash
sudo apt update
sudo apt install -y imx477-rpi-dkms rpicam-apps
```

*With archive enabled, apt resolves Kurokesu `rpicam-apps` and `libcamera` forks as updates to stock packages. Later updates arrive with regular `apt upgrade`.*

Edit boot configuration:

```bash
sudo nano /boot/firmware/config.txt
```

Make two changes:

1. Find `camera_auto_detect` near the top and set it to `0`:

```ini
camera_auto_detect=0
```

2. Add `dtoverlay=kurokesu-imx477` under the `[all]` section at the bottom of the file:

```ini
[all]
dtoverlay=kurokesu-imx477
```

*If camera is connected to cam0 port, use `dtoverlay=kurokesu-imx477,cam0` instead. See [cam0](#cam0).*

Save and exit.

`config.txt` changes take effect after reboot:

```bash
sudo reboot
```

Verify camera is detected:

```bash
rpicam-hello --list-cameras
```

Expected output (varies by link frequency and lane configuration):

```
Available cameras
-----------------
0 : imx477 [4056x3040 12-bit RGGB] (/base/axi/pcie@1000120000/rp1/i2c@70000/imx477@1a)
    Modes: 'SRGGB10_CSI2P' : 1332x990 [271.96 fps - (696, 528)/2664x1980 crop]
                             2028x1080 [166.14 fps - (0, 440)/4056x2160 crop]
                             2028x1520 [119.50 fps - (0, 0)/4056x3040 crop]
                             4056x2160 [42.43 fps - (0, 440)/4056x2160 crop]
                             4056x3040 [30.34 fps - (0, 0)/4056x3040 crop]
           'SRGGB12_CSI2P' : 1332x990 [136.43 fps - (696, 528)/2664x1980 crop]
                             2028x1080 [125.55 fps - (0, 440)/4056x2160 crop]
                             2028x1520 [90.31 fps - (0, 0)/4056x3040 crop]
                             4056x2160 [42.43 fps - (0, 440)/4056x2160 crop]
                             4056x3040 [30.34 fps - (0, 0)/4056x3040 crop]
           'SRGGB8' : 1332x990 [271.96 fps - (696, 528)/2664x1980 crop]
                      2028x1080 [166.14 fps - (0, 440)/4056x2160 crop]
                      2028x1520 [119.50 fps - (0, 0)/4056x3040 crop]
                      4056x2160 [42.43 fps - (0, 440)/4056x2160 crop]
                      4056x3040 [30.34 fps - (0, 0)/4056x3040 crop]
```

Start live preview:

```bash
rpicam-hello -t 0
```

On headless systems, capture a still image instead:

```bash
rpicam-still -o test.jpg
```

## In-kernel driver

Raspberry Pi kernel ships its own `imx477` module. DKMS installs this driver under `updates/`, which takes precedence. Check which one loads:

```bash
modinfo -F version imx477
```

This driver prints its package version. In-kernel driver prints nothing. Removing this driver brings in-kernel one back.

Stock `dtoverlay=imx477` works with this driver as well, limited to 2 lanes. `kurokesu-imx477` overlay has separate name so kernel updates, which restore stock overlays, never overwrite it.

## dtoverlay options

`kurokesu-imx477` overlay supports comma-separated options to override defaults:

| option | description | default |
|--------|-------------|---------|
| [`cam0`](#cam0) | Use cam0 port instead of cam1 | cam1 |
| [`4lane`](#4lane) | Use 4-lane MIPI CSI-2 (if wired) | 2 lanes |
| [`link-frequency=<Hz>`](#link-frequency) | Set MIPI CSI-2 link frequency (Hz) | 450000000 |
| [`always-on`](#always-on) | Keep regulator powered (prevents runtime PM power-off) | off |
| [`sync-source`](#frame-sync) | Output frame sync on `XVS` | off |
| [`sync-sink`](#frame-sync) | Follow external frame sync on `XVS` | off |
| `rotation=<deg>` | Sensor mounting rotation | 0 |
| `orientation=<n>` | Camera orientation (0 front, 1 back, 2 external) | 2 |
| `media-controller=0` | Use legacy unicam driver | on |

### cam0

If camera is connected to cam0 port, append `,cam0`:

```ini
dtoverlay=kurokesu-imx477,cam0
```

### 4lane

To enable 4-lane MIPI CSI-2, append `,4lane`:

```ini
dtoverlay=kurokesu-imx477,4lane
```

> [!WARNING]
> Before using `4lane`, confirm your camera port actually supports 4-lane MIPI CSI. Not all Raspberry Pi models and carrier boards provide 4-lane MIPI CSI on both ports.

### link-frequency

Default link frequency is 450 MHz (900 Mbps/lane). Driver supports higher link frequencies for higher frame rates:

| Range | Step |
|---|---|
| 150 to 262.5 MHz | 1.5 MHz |
| 300 to 525 MHz | 3 MHz |
| 600 to 1050 MHz | 6 MHz |

Other values fail probe with `No supported link frequency configured` in kernel log. Example:

```ini
dtoverlay=kurokesu-imx477,4lane,link-frequency=750000000
```

Highest usable link frequency on Pi 5 is 750 MHz (1500 Mbps/lane), limited by its CSI-2 receiver.

#### Output formats

| Link frequency | Data rate / lane | Lanes | Bit depth | Width | Height | Max FPS |
|---|---|---|---|---|---|---|
| **Full resolution** | | | | | | |
| 450 MHz | 900 Mbps | 2 | 8 | 4056 | 3040 | 17 fps |
| 450 MHz | 900 Mbps | 2 | 10 | 4056 | 3040 | 14 fps |
| 450 MHz | 900 Mbps | 2 | 12 | 4056 | 3040 | 11 fps |
| 450 MHz | 900 Mbps | 4 | 8 | 4056 | 3040 | 30 fps |
| 450 MHz | 900 Mbps | 4 | 10 | 4056 | 3040 | 27 fps |
| 450 MHz | 900 Mbps | 4 | 12 | 4056 | 3040 | 22 fps |
| 750 MHz | 1500 Mbps | 4 | 8 | 4056 | 3040 | 30 fps |
| 750 MHz | 1500 Mbps | 4 | 10 | 4056 | 3040 | 30 fps |
| 750 MHz | 1500 Mbps | 4 | 12 | 4056 | 3040 | 30 fps |
| **4K crop (16:9)** | | | | | | |
| 450 MHz | 900 Mbps | 2 | 8 | 4056 | 2160 | 24 fps |
| 450 MHz | 900 Mbps | 2 | 10 | 4056 | 2160 | 19 fps |
| 450 MHz | 900 Mbps | 2 | 12 | 4056 | 2160 | 16 fps |
| 450 MHz | 900 Mbps | 4 | 8 | 4056 | 2160 | 42 fps |
| 450 MHz | 900 Mbps | 4 | 10 | 4056 | 2160 | 38 fps |
| 450 MHz | 900 Mbps | 4 | 12 | 4056 | 2160 | 32 fps |
| 750 MHz | 1500 Mbps | 4 | 8 | 4056 | 2160 | 42 fps |
| 750 MHz | 1500 Mbps | 4 | 10 | 4056 | 2160 | 42 fps |
| 750 MHz | 1500 Mbps | 4 | 12 | 4056 | 2160 | 42 fps |
| **2028×1520 (2×2 binning)** | | | | | | |
| 450 MHz | 900 Mbps | 2 | 8 | 2028 | 1520 | 66 fps |
| 450 MHz | 900 Mbps | 2 | 10 | 2028 | 1520 | 53 fps |
| 450 MHz | 900 Mbps | 2 | 12 | 2028 | 1520 | 45 fps |
| 450 MHz | 900 Mbps | 4 | 8 | 2028 | 1520 | 119 fps |
| 450 MHz | 900 Mbps | 4 | 10 | 2028 | 1520 | 102 fps |
| 450 MHz | 900 Mbps | 4 | 12 | 2028 | 1520 | 86 fps |
| 750 MHz | 1500 Mbps | 4 | 8 | 2028 | 1520 | 119 fps |
| 750 MHz | 1500 Mbps | 4 | 10 | 2028 | 1520 | 119 fps |
| 750 MHz | 1500 Mbps | 4 | 12 | 2028 | 1520 | 90 fps |
| **1080p (2×2 binning crop)** | | | | | | |
| 450 MHz | 900 Mbps | 2 | 8 | 2028 | 1080 | 92 fps |
| 450 MHz | 900 Mbps | 2 | 10 | 2028 | 1080 | 74 fps |
| 450 MHz | 900 Mbps | 2 | 12 | 2028 | 1080 | 62 fps |
| 450 MHz | 900 Mbps | 4 | 8 | 2028 | 1080 | 166 fps |
| 450 MHz | 900 Mbps | 4 | 10 | 2028 | 1080 | 142 fps |
| 450 MHz | 900 Mbps | 4 | 12 | 2028 | 1080 | 120 fps |
| 750 MHz | 1500 Mbps | 4 | 8 | 2028 | 1080 | 166 fps |
| 750 MHz | 1500 Mbps | 4 | 10 | 2028 | 1080 | 166 fps |
| 750 MHz | 1500 Mbps | 4 | 12 | 2028 | 1080 | 125 fps |
| **1332×990 (2×2 binning crop)** | | | | | | |
| 450 MHz | 900 Mbps | 2 | 8 | 1332 | 990 | 147 fps |
| 450 MHz | 900 Mbps | 2 | 10 | 1332 | 990 | 120 fps |
| 450 MHz | 900 Mbps | 2 | 12 | 1332 | 990 | 101 fps |
| 450 MHz | 900 Mbps | 4 | 8 | 1332 | 990 | 271 fps |
| 450 MHz | 900 Mbps | 4 | 10 | 1332 | 990 | 224 fps |
| 450 MHz | 900 Mbps | 4 | 12 | 1332 | 990 | 136 fps |
| 750 MHz | 1500 Mbps | 4 | 8 | 1332 | 990 | 271 fps |
| 750 MHz | 1500 Mbps | 4 | 10 | 1332 | 990 | 271 fps |
| 750 MHz | 1500 Mbps | 4 | 12 | 1332 | 990 | 136 fps |

> [!NOTE]
> RPi5/RP1 has a 380 Mpix/s processing limit. Effective framerate is capped at ~30.3 fps @ full resolution regardless of the link frequency configured here.

> [!TIP]
> Options can be combined. Example (cam0, 4-lane):
> ```ini
> dtoverlay=kurokesu-imx477,cam0,4lane
> ```

### Frame sync

Multiple IMX477 sensors can share frame timing through `XVS` pin. One sensor runs as `sync-source` and drives `XVS`, others run as `sync-sink` and follow it:

```ini
dtoverlay=kurokesu-imx477,sync-source
```

```ini
dtoverlay=kurokesu-imx477,cam0,sync-sink
```

`XVS` and `XSTROBE` are **1.8 V logic level** pins wired directly to sensor.

Both signals are available on AUX connector:

![IMX477 module AUX connector pinout, XVS, XSTROBE and GND next to two unconnected pins.](https://raw.githubusercontent.com/Kurokesu/imx477-rpi-driver/main/docs/imx477-aux.png)

*Full module pinout is documented in [477x-CSI wiki page](https://wiki.kurokesu.com/books/mipi-csi2-camera-modules/page/477x-csi)*

---

Frame sync can also be set via module parameter. New value applies at next stream start:

```bash
# 0=off, 1=sync-source, 2=sync-sink
echo 1 | sudo tee /sys/module/imx477/parameters/trigger_mode
```

> [!NOTE]
> Module parameter is global: in a multi-camera setup it applies to all IMX477 sensors. To configure each sensor independently, use device tree overlay options instead. Device tree setting takes precedence over module parameter.

### always-on

`always-on` keeps camera regulator permanently enabled, preventing kernel from powering off sensor during runtime PM suspend.

```ini
dtoverlay=kurokesu-imx477,always-on
```

## Build from source

Install required tools:

```bash
sudo apt install -y git
sudo apt install -y --no-install-recommends dkms
```

Clone this repository:

```bash
cd ~
git clone https://github.com/Kurokesu/imx477-rpi-driver.git
cd imx477-rpi-driver/
```

If driver was installed from apt archive previously, remove it first:

```bash
sudo apt remove imx477-rpi-dkms
```

Run setup script:

```bash
sudo ./setup.sh
```

Camera stack, boot configuration and verification follow [Install](#install). Skip `imx477-rpi-dkms` there, only `rpicam-apps` is needed. To build `libcamera` and `rpicam-apps` from source as well, see [libcamera/BUILDING.md](https://github.com/Kurokesu/libcamera/blob/kurokesu/BUILDING.md).

## Migrating from a source install

One-time cleanup before first apt install.

Remove `imx477` driver modules installed by `setup.sh`:

```bash
dkms status | grep imx477 | cut -d, -f1 | sort -u | xargs -rI{} sudo dkms remove {} --all
```

Source-built `libcamera` and `rpicam-apps` install to `/usr/local` and shadow packaged binaries. Remove them:

> [!WARNING]
> Command below deletes everything under `/usr/local` with `libcamera`, `rpicam` or `libpisp` in its name, including custom scripts or files named after them.

```bash
sudo find /usr/local -depth \( -name '*libcamera*' -o -name '*rpicam*' -o -name '*libpisp*' \) -exec rm -rf {} +
```

Cleanup complete. Continue with [install steps](#install).

## Special thanks

- [Raspberry Pi](https://www.raspberrypi.com/) for original [`imx477` driver](https://github.com/raspberrypi/linux/blob/rpi-6.18.y/drivers/media/i2c/imx477.c)
