# IMX477 (HQ Camera) kernel driver for Raspberry Pi

[![Build](https://github.com/Kurokesu/imx477-rpi-driver/actions/workflows/build.yml/badge.svg)](https://github.com/Kurokesu/imx477-rpi-driver/actions/workflows/build.yml)
[![Code style](https://github.com/Kurokesu/imx477-rpi-driver/actions/workflows/code-style.yml/badge.svg)](https://github.com/Kurokesu/imx477-rpi-driver/actions/workflows/code-style.yml)
[![Release](https://img.shields.io/github/v/release/Kurokesu/imx477-rpi-driver)](https://github.com/Kurokesu/imx477-rpi-driver/releases/latest)
[![Kurokesu apt archive](https://img.shields.io/badge/apt-Kurokesu_archive-D70A53?logo=debian)](https://apt.kurokesu.com)
[![RPi OS Bookworm | Trixie](https://img.shields.io/badge/RPi_OS-Bookworm_%7C_Trixie-blue?logo=raspberrypi)](https://www.raspberrypi.com/software/operating-systems/)
[![Kernel 6.12+](https://img.shields.io/badge/kernel-6.12%2B-blue?logo=raspberrypi)](https://github.com/raspberrypi/linux/tree/rpi-6.12.y)

Extended Raspberry Pi kernel driver for Sony IMX477, a 12.3 MP 1/2.3" stacked CMOS sensor with 1.55 µm pixels. Drop-in replacement for in-kernel `imx477` driver, works with Raspberry Pi HQ Camera and other IMX477 modules.

Additions over in-kernel driver:

- 4-lane MIPI CSI-2
- Refined PLL settings across full 150 to 1050 MHz link frequency range
- Refined line length limits

Supported framerates (4 lanes at 750 MHz):

- 4056×3040 @ 30 fps (full resolution)
- 4056×2160 @ 42 fps (4K crop)
- 2028×1520 @ 119 fps (2×2 binning)
- 2028×1080 @ 166 fps (1080p, 2×2 binning crop)
- 1332×990 @ 271 fps (2×2 binning crop)

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

*Output shown for `4lane` at 750 MHz on Pi 5.*

Start live preview:

```bash
rpicam-hello -t 0
```

On headless systems, capture a still image instead:

```bash
rpicam-still -o test.jpg
```

## In-kernel driver

DKMS installs this driver under `updates/`, which takes precedence over in-kernel `imx477` module. Check which one loads:

```bash
modinfo -n imx477
```

Expected output:

```
/lib/modules/<kernel>/updates/dkms/imx477.ko.xz
```

In-kernel driver loads from `kernel/drivers/media/i2c/` instead. Removing this driver brings in-kernel one back.

Stock `dtoverlay=imx477` works with this driver as well, limited to 2 lanes. Use `kurokesu-imx477` overlay for `4lane` option.

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

Example:

```ini
dtoverlay=kurokesu-imx477,4lane,link-frequency=750000000
```

Highest usable link frequency on Pi 5 is 750 MHz (1500 Mbps/lane), limited by its CSI-2 receiver. Pi 4 accepts full range, but its ISP caps output at ~25 fps @ full resolution.

#### Output formats

| Link frequency | Lanes | 8-bit | 10-bit | 12-bit |
|---|---|---|---|---|
| **4056×3040 (full resolution)** | | | | |
| 450 MHz | 2 | 17 fps | 14 fps | 11 fps |
| 750 MHz | 2 | 28 fps | 22 fps | 19 fps |
| 750 MHz | 4 | 30 fps | 30 fps | 30 fps |
| **4056×2160 (4K crop)** | | | | |
| 450 MHz | 2 | 24 fps | 19 fps | 16 fps |
| 750 MHz | 2 | 39 fps | 32 fps | 26 fps |
| 750 MHz | 4 | 42 fps | 42 fps | 42 fps |
| **2028×1520 (2×2 binning)** | | | | |
| 450 MHz | 2 | 66 fps | 53 fps | 45 fps |
| 750 MHz | 2 | 106 fps | 86 fps | 73 fps |
| 750 MHz | 4 | 119 fps | 119 fps | 90 fps |
| **2028×1080 (1080p, 2×2 binning crop)** | | | | |
| 450 MHz | 2 | 92 fps | 74 fps | 62 fps |
| 750 MHz | 2 | 147 fps | 120 fps | 101 fps |
| 750 MHz | 4 | 166 fps | 166 fps | 125 fps |
| **1332×990 (2×2 binning crop)** | | | | |
| 450 MHz | 2 | 147 fps | 120 fps | 101 fps |
| 750 MHz | 2 | 232 fps | 191 fps | 136 fps |
| 750 MHz | 4 | 271 fps | 271 fps | 136 fps |

*Measured on Pi 5.*

> [!NOTE]
> RPi5/RP1 has a 380 Mpix/s processing limit. Effective framerate is capped at ~30.3 fps @ full resolution regardless of the link frequency configured here.

### always-on

`always-on` keeps camera regulator permanently enabled, preventing kernel from powering off sensor during runtime PM suspend. Useful for debugging hardware issues, since it forces `CAM_GPIO` high constantly.

```ini
dtoverlay=kurokesu-imx477,always-on
```

### Frame sync

Multiple IMX477 cameras can run in hardware sync, sharing frame timing through `XVS` pin. One sensor runs as `sync-source` and drives `XVS`, others run as `sync-sink` and follow it:

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

> [!TIP]
> Options can be combined. Example (cam0, 4-lane, 750 MHz link):
>
> ```ini
> dtoverlay=kurokesu-imx477,cam0,4lane,link-frequency=750000000
> ```

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

## FAQ

### Does Raspberry Pi HQ Camera support 4 lanes?

No. HQ Camera board wires out 2 lanes. It works with this driver at 2 lanes, where higher [link frequency](#link-frequency) raises framerates. 4 lanes need an IMX477 module wired for 4 lanes on a 4-lane camera port, see [4lane](#4lane).

### How do I fix "only 2 data lanes are currently supported"?

In-kernel `imx477` driver rejects 4-lane configuration at probe:

```
imx477 10-001a: only 2 data lanes are currently supported
imx477 10-001a: probe with driver imx477 failed with error -22
```

Install this driver and add `4lane` to overlay line:

```ini
dtoverlay=kurokesu-imx477,4lane
```

If log also shows `subdevice requires 4 data lanes when 2 are supported`, camera port itself has only 2 lanes. Use 2 lanes there or move camera to a 4-lane port.

### How do I record 1080p at 120 fps with HQ Camera?

Set link frequency to 750 MHz:

```ini
dtoverlay=kurokesu-imx477,link-frequency=750000000
```

Record 10 seconds to MJPEG:

```bash
rpicam-vid -n -t 10000 --mode 2028:1080:10:P --width 2028 --height 1080 --framerate 120 --codec mjpeg -o video.mjpeg
```

### Why does rpicam-hello say "No cameras available!"?

Check [Install](#install) steps first: `camera_auto_detect=0`, overlay line under `[all]` and `cam0` option matching the port in use. Then check kernel log:

```bash
dmesg | grep imx477
```

`failed to read chip id 477` means sensor does not answer on I2C. Check camera port and cable seating, with Pi powered off.

## Special thanks

- [Raspberry Pi](https://www.raspberrypi.com/) for original [imx477 driver](https://github.com/raspberrypi/linux/blob/rpi-6.18.y/drivers/media/i2c/imx477.c)
