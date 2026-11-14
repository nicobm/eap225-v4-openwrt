# TP-Link EAP225 v4 → OpenWrt

**From stock firmware 5.2.x to OpenWrt — flashing over the serial console, plus a minimal first-boot setup.**

![Hardware](https://img.shields.io/badge/hardware-TP--Link%20EAP225%20v4-0ea5e9?style=flat-square)
![OpenWrt](https://img.shields.io/badge/OpenWrt-23.05.5-00b5e2?style=flat-square)
![Target](https://img.shields.io/badge/target-ath79%2Fgeneric-2dd4bf?style=flat-square)
[![License](https://img.shields.io/badge/license-GPL--2.0-64748b?style=flat-square)](https://www.gnu.org/licenses/old-licenses/gpl-2.0.html)

> **Interactive guide:** <https://nicobm.github.io/eap225-v4-openwrt>
>
> The web version walks through every step with code blocks you can copy, safety callouts, and the multimeter check for the USB-UART adapter. It is bilingual: English by default, with a Spanish toggle.
>
> This README is the condensed version.

This repository is intentionally minimalist: **only the steps that change are documented. Everything not mentioned is fine on its defaults.**

| Part | What it covers | Serial access required |
|---|---|---|
| **1. Flashing** | Replace the stock TP-Link firmware with OpenWrt over the serial console + U-Boot + TFTP. Specific to the EAP225 v4 hardware. | Yes |
| **2. First boot** | Minimal setup after OpenWrt is running. | No |

---

## Contents

- [Part 1 — Flashing OpenWrt](#part-1--flashing-openwrt)
  - [Requirements](#requirements)
  - [Before you start](#before-you-start)
  - [Firmware situation](#firmware-situation)
  - [Specifications](#specifications)
  - [Materials](#materials)
  - [Verify the USB-UART adapter](#verify-the-usb-uart-adapter)
  - [Flash steps](#flash-steps)
  - [Back to stock / last resort](#back-to-stock--last-resort)
- [Part 2 — First boot & basic configuration](#part-2--first-boot--basic-configuration)
- [Credits and license](#credits-and-license)

---

## Part 1 — Flashing OpenWrt

> This is the only part that needs hardware access (serial console + soldering). The whole setup in Part 2 is done through the LuCI web GUI.

### Requirements

- TP-Link Omada EAP225 **hardware v4.0** (any hardware revision; only the firmware matters).
- Stock firmware **5.2.3** (Build 20250709) — the guide targets this version. Firmware 5.2.x blocks the software method; serial is required.
- A Linux PC, a USB-to-UART adapter (3.3V), dupont wires, soldering iron, ethernet cable, and a PoE injector/switch.
- Tools: `picocom`, `dnsmasq`, `wget`, `scp`, `ssh`.

### Before you start

> [!WARNING]
> **Use a 3.3V USB-UART adapter.** The EAP225 v4 SoC (QCA9563) runs at 3.3V. A 5V adapter can permanently damage it. Verify with a multimeter before connecting anything.

> [!CAUTION]
> **There is no factory recovery after flashing.** Keep a full flash backup if you want to go back to stock; otherwise restoring requires an SPI programmer.

> [!CAUTION]
> **A power loss can brick the device.** Use a stable PoE source and do not disconnect anything during the process.

### Firmware situation

On firmware ≤ 5.1.6, SSH gave a full Linux shell where you could run `cliclientd stopcs` and then flash OpenWrt from the web UI. From 5.2.x onward, TP-Link removed the Linux shell, removed `cliclientd`, and blocked downgrade. Hence the serial method.

| Stock firmware | SSH + `cliclientd` + web | Serial + U-Boot | TFTP recovery (reset button) |
|---|---|---|---|
| ≤ 5.1.6 | Works | Works | Not documented for v4 |
| 5.2.x | Blocked | **Works ← your path** | Not documented for v4 |

### Specifications

| Component | Detail |
|---|---|
| SoC | QCA9563 @ 775 MHz |
| RAM | 128 MiB DDR2 |
| Flash | 16 MiB SPI-NOR |
| WiFi 2.4 GHz | b/g/n 3x3 (integrated, driver `ath9k`) |
| WiFi 5 GHz | a/n/ac 2x2 MU-MIMO (QCA9886, driver `ath10k`) |
| Ethernet | 1× GbE (PHY RTL8211FS), 802.3at PoE |
| Serial | PCB J4 — Pin1 TXD, Pin2 RXD, Pin3 GND, Pin4 VCC |
| Serial config | **3.3V**, 115200 baud, 8N1 |

### Materials

**Hardware:** 3.3V USB-UART adapter (CP2102/FT232RL/CH340G), dupont wires, soldering iron + solder, ethernet cable, PoE injector/switch.

**Software:** `picocom`, `dnsmasq`, `wget`, OpenWrt images for EAP225 v4.

Install the tools with your package manager:

```bash
sudo apt install picocom dnsmasq wget   # Debian/Ubuntu
sudo dnf install picocom dnsmasq wget   # Fedora
sudo pacman -S picocom dnsmasq wget     # Arch
```

Add your user to the `dialout` group to access `/dev/ttyUSB*` (log out and back in):

```bash
sudo usermod -aG dialout $USER
```

### Verify the USB-UART adapter

Plug the adapter in (nothing connected). Measure with a multimeter in DC voltage mode:

- **VCC vs GND:** expected ~3.3V (3.0–3.6). If ~5.0V → do not use.
- **TXD vs GND (idle):** expected ~3.3V.

If any reading is above 3.6V, do not connect that adapter to the EAP225.

### Flash steps

#### 1. Download the OpenWrt images

```bash
mkdir -p ~/openwrt-eap225v4 && cd ~/openwrt-eap225v4
wget https://downloads.openwrt.org/releases/23.05.5/targets/ath79/generic/openwrt-23.05.5-ath79-generic-tplink_eap225-v4-initramfs-kernel.bin -O initramfs.bin
wget https://downloads.openwrt.org/releases/23.05.5/targets/ath79/generic/openwrt-23.05.5-ath79-generic-tplink_eap225-v4-squashfs-sysupgrade.bin -O sysupgrade.bin
```

Check the latest stable release on the firmware selector searching "EAP225 v4" and adjust the URLs.

#### 2. Open the case and wire the serial

Disconnect power. Open the case, locate header J4 (`Pin1 TXD, Pin2 RXD, Pin3 GND, Pin4 VCC`). You may need to bridge unpopulated resistors R225 (TXD) and R237 (RXD). Solder pins/wires, then connect:

```
EAP225 J4 Pin1 (TXD)  →  RXD of the USB-UART adapter
EAP225 J4 Pin2 (RXD)  →  TXD of the USB-UART adapter
EAP225 J4 Pin3 (GND)  →  GND of the USB-UART adapter
EAP225 J4 Pin4 (VCC)  →  DO NOT CONNECT (powered by PoE)
```

#### 3. Open the serial terminal

```bash
ls /dev/ttyUSB*
picocom -b 115200 /dev/ttyUSB0
```

Exit picocom with `Ctrl+A` then `Ctrl+X`. Do not power the EAP yet.

#### 4. Configure the network and TFTP server

Connect the PC directly to the EAP225. Set a static IP:

```bash
sudo ip addr add 192.168.1.10/24 dev eth0   # replace eth0 with your interface
```

Open the firewall for TFTP and start dnsmasq (keep running):

```bash
sudo firewall-cmd --add-service=tftp   # or: sudo ufw allow 69/udp
mkdir -p /tmp/tftp
cp ~/openwrt-eap225v4/initramfs.bin /tmp/tftp/
sudo dnsmasq --no-daemon --port=0 --enable-tftp --tftp-root=/tmp/tftp --listen-address=192.168.1.10
```

#### 5. Access U-Boot

Have picocom open. Connect PoE power. Immediately press `Ctrl+B` repeatedly. On success you get:

```
ath>
```

If it boots into stock firmware, disconnect PoE, wait, reconnect while spamming `Ctrl+B`.

#### 6. Load OpenWrt over TFTP

At the `ath>` prompt:

```
ath> setenv ipaddr 192.168.1.1
ath> setenv serverip 192.168.1.10
ath> tftp 0x80800000 initramfs.bin
ath> bootelf $fileaddr
```

This runs OpenWrt from RAM (flash untouched). The dnsmasq terminal shows the transfer; the serial terminal shows the kernel booting.

#### 7. Install OpenWrt permanently

OpenWrt is now reachable at `192.168.1.1`.

**Option A — via SSH (recommended):**

```bash
ping -c 3 192.168.1.1
scp ~/openwrt-eap225v4/sysupgrade.bin root@192.168.1.1:/tmp/
ssh root@192.168.1.1
# inside the device:
sysupgrade -v /tmp/sysupgrade.bin
```

**Option B — via LuCI:** open `http://192.168.1.1` → System → Backup/Flash Firmware → upload `sysupgrade.bin` → confirm.

The device reboots and this time boots OpenWrt from internal flash.

#### 8. First login

Wait for the LED to stabilize. Browse to `http://192.168.1.1`. User `root`, no password. Set a password immediately under System → Administration.

#### 9. Clean up

```bash
sudo ip addr del 192.168.1.10/24 dev eth0
# stop dnsmasq: Ctrl+C in its terminal
# exit picocom: Ctrl+A, Ctrl+X
```

### Back to stock / last resort

There is no TP-Link recovery mode after flashing OpenWrt. To restore stock firmware, use an SPI programmer (clip SOIC-8 + CH341A at 3.3V, or Raspberry Pi + `flashrom`) with a full flash backup. U-Boot is not blocked on this device, so the serial method remains available even after a bad flash.

---

## Part 2 — First boot & basic configuration

Minimal steps after OpenWrt is running. The EAP225 is an access point, so keep it simple.

- Wait for the LED to stabilize, then browse to `http://192.168.1.1` and log in as `root`.
- Set a root password: System → Administration.
- Enable Wi-Fi if needed: Network → Wireless → enable the radios (radio0 2.4 GHz, radio1 5 GHz).
- For a dumb AP behind an existing router: disable DHCP on LAN, assign a static IP in your subnet, bridge wireless interfaces to LAN.
- Set timezone/NTP: System → System → Timezone; Services → NTP (or install `chrony-nts` for NTS).
- Update packages: System → Software → Update lists, then upgrade.

---

## Credits and license

Flashing method based on the OpenWrt community work for the EAP225 (serial console + U-Boot + TFTP). See the [OpenWrt wiki](https://openwrt.org/toh/tp-link/eap225) and forum threads. Licensed under [GPL-2.0](https://www.gnu.org/licenses/old-licenses/gpl-2.0.html).
