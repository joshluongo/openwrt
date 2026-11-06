# MERCUSYS MB118-4G v1: bring-up findings

Recorded: 2026-11-03

Status: the port builds on OpenWrt v25.12.6 (see "Implemented port" below)
and **runs on the owner's unit** since 2026-11-06. On 2026-11-05 the owner
supplied a full 8 MiB flash dump; its findings are under "Full flash dump"
and they resolved the per-unit data blocker for a TFTP-recovery install (see
"Installation method"). That install worked, and later images were installed
with `sysupgrade`. Verified on the device: boot, Ethernet (SSH and LuCI on the
single port), 2.4 GHz Wi-Fi (in client mode), the modem's USB functions, and
LTE data through the modem helper, including automatic reconnection.

## Device identity and live checks

The stock CLI and authenticated web API on the owner's device reported:

- Model: `MB118-4G`.
- Hardware: `MB118-4G v1.0 00000001`.
- Firmware: `1.6.0 0.9.1 v0001.0 Build 250523 Rel.57832n`.
- Hardware ID: `803286B9E7248CA006F100729FF813BC`.
- Modem software: `EC200AELLAR01A05M16_TP`, indicating the Quectel EC200A-EL
  family. The modem model-name field was empty; USB IDs remain unknown.
- LTE was connected on the stock firmware. The API reported RSRP -84 dBm and
  RSRQ -10 dB at the time of the initial check.
- HTTP and Telnet were reachable; SSH and HTTPS were not listening at the time
  of the check. The administrator password authenticated to the stock CLI.

The CLI permits device-version and basic status queries, but did not provide an
administrator shell. `sh` and `enable` were rejected in the tested CLI mode.
The web API successfully returned firmware, LTE and Ethernet status and system
logs. Those logs did not establish RAM size, live MTD geometry or USB IDs.

The device label (EU hardware v1.0) confirms further non-sensitive details:

- "300Mbps Wireless N" — a single 2.4 GHz 802.11n radio, **no 5 GHz**. This
  matches the port's use of only the integrated MT7628 WMAC and the omission of
  the MB130-4G's MT7663 PCIe 5 GHz block.
- Power: 12 V / 1 A DC, and PoE both passive (12 V / 1 A) and 802.3af
  (44-57 V / 0.35 A). Enclosure rated IP65. Default access `http://mwlogin.net`.
- The label MAC uses a TP-Link OUI and its last octets match the default SSID
  suffix, i.e. the label MAC is the device base MAC. Per-unit MAC, Wi-Fi PIN,
  IMEI and serial are deliberately kept out of this document; the label MAC is
  retained privately as a post-flash verification reference for the `romfile`
  nvmem offset.

The single Ethernet connector must remain usable for LAN management in the
default configuration. Its alternate WAN role needs a separate test. One stock
WAN object reported interface `eth0.5`; this alone does not establish the switch
port number or PHY wiring. The vendor product page advertises a gigabit port;
the actual PHY and supported link speeds still need confirmation.

Subscriber identifiers, credentials and raw configuration backups are deliberately
excluded from this document.

## Matching vendor firmware

Primary sources:

- [Product specifications](https://www.mercusys.com/au/product/mobile-wifi/mb118-4g/v1/).
- [Firmware download page](https://www.mercusys.com/au/download/mb118-4g/).
- [Matching EU v1 firmware archive](https://static.mercusys.com/software/MB118-4G(EU)_V1_1.6.0_Build_25052320250903072616.zip).
- [GPL download query](https://www.mercusys.com/en/support/gpl-code/?model=MB118-4G),
  which listed no source archive when checked.

The downloaded archive contains:

```text
MB118-4G(EU)v1.0_1.6.0_0.9.1_[250523-rel57832]_up_boot_all_Release_2025-05-23_16.03.14.bin
```

Binary length: 15,485,460 bytes. SHA-256 of that extracted binary:

```text
b4334a2109349b9f4bc5bac06d4306cf44dc254d8886e955e160e611df7cc003
```

Offline extraction established:

- Bootloader: `U-Boot 1.1.3 (May 23 2025 - 15:53:36)`, with an `MT7628 #`
  prompt string and MT7628 PHY initialization messages.
- Kernel: LZMA stream at file offset `0x20400`, decompressing to 3,081,360 bytes.
- Kernel version: Linux 2.6.36, MIPS32 R2, little endian, built May 23, 2025.
- SquashFS: file offset `0x150200`, XZ compression, 262,144-byte block size.
- Stock modem drivers include USB serial `option`, `cdc_ether` and `rndis_host`.
  Their presence does not identify which driver the live modem uses.

The archive's length is not a measurement of the router's SPI flash capacity.
The router image occupies file offsets `0x200`–`0x6c0200` (a `0x200`-byte
header, then data padded with `0xff` from `0x684200`). Everything from
`0x6c0200` to the end, about 7.8 MB, is an ASR modem firmware image (`HMIT`/TIM
headers and ASR PMIC and charger strings), i.e. the bundled EC200A update. The
router part therefore fits the 8 MiB table below. The file contains no
`romfile` or `radio` data, which are per-unit.
Likewise, a bundled USB driver is not proof that it is bound to the live modem.

## Vendor kernel flash table

The decompressed kernel contains the following nine-entry static partition
table at offset `0x2bea08`. The SPI initialization code passes that table to
the MTD partition registration function. The full flash dump below matches it
partition for partition.

| Partition | Offset | Size |
| --- | --- | --- |
| boot | `0x000000` | `0x020000` |
| kernel | `0x020000` | `0x130000` |
| rootfs | `0x150000` | `0x560000` |
| ispconfig | `0x6b0000` | `0x010000` |
| config | `0x6c0000` | `0x010000` |
| configbak | `0x6d0000` | `0x010000` |
| romfile | `0x6e0000` | `0x010000` |
| radio | `0x6f0000` | `0x010000` |
| userdata | `0x700000` | `0x100000` |

This table spans 8 MiB. The firmware area before `ispconfig` is `0x690000`
bytes, starting at `0x020000`. Do not substitute the supported MB130-4G's
16 MiB layout, calibration offsets or image-size limit.

## Full flash dump

The owner supplied nine per-partition files (plus a duplicate of `boot`) that
concatenate to exactly 8 MiB, with the sizes in the table above. The dump is
kept privately outside the repository; it contains per-unit data and must not
be committed or shared. Findings:

- **Integrity:** dump bytes `0x000000`–`0x6b0000` (U-Boot, firmware header,
  kernel and rootfs) are byte-identical to the matching vendor firmware file
  above. This validates the read method for the rest of the dump.
- **`ispconfig`** is entirely `0xff`. **`config`/`configbak`** hold vendor
  settings. **`userdata`** is a JFFS2 filesystem.
- **`romfile`** holds the base MAC at `0xf100`. It is globally administered and
  matches the label MAC. Other non-`0xff` data is at `0xf200`, `0xf500`,
  `0xf600` and `0xfc00`.
- **`radio`** holds an MT7628 EEPROM (chip ID `0x7628`) in its first `0x400`
  bytes, with the remainder `0xff`. It differs from the firmware's bundled
  default EEPROM in 24 bytes of the first `0x200`, consistent with per-unit
  calibration. Its own MAC field is the MediaTek default `00:0c:43:…`, so the
  MAC must come from `romfile`, as the DTS does.
- **The stock firmware header is little-endian throughout.** It declares
  `fw_length = 0x6a0000`, a kernel offset/length of `0x200`/`0xf291f`, and a
  rootfs at offset `0x130000` with length `0x534000`. OpenWrt's `tplink-v2`
  header is big-endian apart from the swapped (`-e`) load and entry addresses.

A root shell on the running stock firmware later confirmed:
- `/proc/mtd` matches the nine-partition table.
- The MD5 of each of `mtdblock3`–`mtdblock8` equals the corresponding dump
  file, so the per-unit region was read correctly and is current.
- The SoC is an MT7628 with `MemTotal` 61,848 kB (64 MiB RAM).
- The LAN side uses VLANs 2–5 on `eth0`, bridged from `eth0.2`.

The bootloader's boot path (`bootm 0xbc020000`) was disassembled from the dump.
It always decompresses LZMA from header offset `0x200`. It reads only
`kernel_la` (`0x68`), `kernel_ep` (`0x6c`) and `kernel_len` (`0x78`), as
little-endian words, and passes `kernel_len` only as the decompressor's input
bound. It ignores `kernel_ofs` and `fw_length` and checks no checksum. OpenWrt's
swapped load/entry addresses therefore decode correctly. Its big-endian
`kernel_len` decodes as an oversized bound, which is harmless because the LZMA
stream carries its own size. This is the same mechanism other `tplink-v2`
MT7628 devices rely on, but it has not been booted on this unit.

### Recovery discrepancy

The vendor bootloader contains environment defaults `ipaddr=192.168.0.2` and
`serverip=192.168.0.225`, and this recovery command string:

```text
tftp 0x80060000 tp_recovery.bin;erase tplink 0x20000 0x7a0000;cp.b 0x80080000 0x20000 0x7a0000
```

The dumped bootloader confirms this string and its trigger. At power-up it
polls GPIO 38 (the reset button, low when pressed) every 100 ms. If the button
is still held after 50 polls (about 5 s), it runs the recovery command;
otherwise it boots normally. The destination interval, `[0x020000, 0x7c0000)`,
overlaps the configuration partitions, `romfile`, `radio` and part of
`userdata`. A padded OpenWrt image would erase the device's unique data. Only a
recovery image that writes that data back unchanged is safe (see
"Installation method").

The bootloader also advertises a distinct "Load system code to SDRAM via TFTP"
menu option, alongside options that explicitly write flash. Whether and how
that RAM-only option can be selected requires console verification. Ethernet
TFTP transport alone does not prove console-free RAM boot is available.

## GPIO findings from the vendor kernel

The kernel registers `/proc/tplink/` handlers for modem power, boot mode, DC,
antenna selection, reset, LEDs and GPIO status. All of them drive a 12-entry
GPIO table through a common routine at kernel address `0x80133578`, assuming
the raw kernel's `0x80000000` load base. A periodic timer writes the table's
states to the pins. The pin-write routine inverts levels only for GPIOs listed
at `0x80260e50`: `46, 44, 43, 42, 41, 40, 39`, the LEDs. These are offline
findings from disassembly, not physical pin testing:

| Control | GPIO | Pin level when active / stock steady state |
| --- | --- | --- |
| Power LED | 46 | on = high |
| Internet LED (follows `/proc/tplink/led_internet`) | 44 | on = high |
| 2.4 GHz WLAN / WPS LED | 43 | on = high |
| LAN LED (driver link check, switch ports 1–4) | 42 | on = high |
| Signal indicator, increasing strength | 41, then 40, then 39 | on = high |
| Reset button (kernel poller and U-Boot) | 38 | pressed = low |
| Modem USB power control (`usb_power`) | 36 | high; writing 1 pulses low for 1 s |
| Modem DC control (`usb_dc`) | 37 | high |
| Modem antenna control (`usb_antenna`) | 5 | high (never changed by userspace) |
| Modem boot control (`usb_boot`) | 3 | low; high only for modem firmware download |
| Modem reset control (`usb_reset`) | 4 | low; high asserts reset |

For the modem lines, a userspace value of 1 drives the pin high. The stock
`libcmm` writes `usb_power` 0 only on its shutdown path, after unloading the
USB drivers. It pulses `usb_boot`/`usb_reset` only to run the modem
firmware updater (`qdloader`). The stock `cos` daemon configures the modem
with `AT+QCFG="usbnet",3` (RNDIS), `AT+QCFG="nat",1` and `AT+QNETDEVCTL`
auto-dial. The modem retains its USB network mode across reboots.

GPIO 44 is on the `wled_an` pin, so the DTS muxes that group to GPIO. GPIO
assignments differ materially from `mt7628an_mercusys_mb130-4g-v1.dts`.

## Information needed before implementing a flashable port

Items 2 and 3 and most of 4 are now answered by the flash dump and the kernel
analysis above, and the modem's USB identity by the live system. Still open:
item 1 (a boot log) and item 5. The Ethernet switch port is port 4.

1. A complete cold-boot log: actual SoC, RAM size, SPI flash identity, live MTD
   boundaries, switch/PHY setup and modem USB enumeration.
2. A verified full-flash backup, retained privately, including bootloader,
   calibration, MAC addresses and vendor data. A web configuration export is
   not a substitute for a flash backup.
3. Confirmed Wi-Fi EEPROM and MAC locations within the live flash layout.
4. Confirmed Ethernet wiring, modem USB interface/driver, power/reset polarity
   and button/LED behavior.
5. A verified RAM-only boot procedure, followed by validation before any
   persistent installation. Prefer Ethernet-only access where possible; UART
   may be required to collect the missing evidence and select RAM boot.

No PCB revision, UART pad assignment or serial wiring has been established.
Do not infer those from another MERCUSYS model.

## Software-only investigation

The stock CLI accepts the hidden, read-only `mdlog info` command after normal
administrator authentication. This was verified on the owner's router and
reported the same device and modem firmware identities listed above.

Offline analysis of `/usr/bin/cli` also identified `mdlog prepare`. Its first
step parses the Telnet peer address from `netstat -nt`, then requests
`conf.json` from that peer using TFTP. Later stages prepare a temporary working
directory, download `mdlog_bridge.bin`, verify the bridge's RSA signature,
extract it and fetch modem diagnostic configuration. `mdlog start` invokes
the bridge from the working directory. The working directory is taken from
the JSON property `router.workdir` and is interpolated into shell command
strings. This is a candidate for temporary diagnostic access, not a verified
shell or installation procedure. The signed bridge cannot simply be replaced
with an unsigned executable through the normal loader path.

A bounded live transfer probe was performed with a temporary TFTP listener
on the SSH host. The listener served no files and terminated after 25 seconds.
The router's CLI reported:

```text
pcIP=192.168.1.100
[1]Upload and parse config file...
Failed to upload config file.
cmd:FAIL
```

No TFTP request reached the SSH host at `172.16.50.53`. The current routed
connection therefore does not deliver the diagnostic callback to that host.
This probe stopped at the initial file transfer, before working-directory
preparation or diagnostic-tool loading. No router settings or flash contents
were changed. A direct Ethernet connection, or an explicitly configured
callback path through the intervening router, is needed to test this loader
further. Direct Ethernet is preferred because it avoids changing the existing
gateway's configuration. It does not require opening the MB118 enclosure.

The owner subsequently moved the modem to VLAN 20. A temporary VLAN interface
`enp1s0f0.20` was created on the SSH host with address `192.168.1.225/24`.
The modem answered ARP and HTTP at `192.168.1.1` through that interface.
The host's existing `172.16.50.53` address and IPv4 default route were retained.
IPv6 router-advertisement acceptance and autoconfiguration were disabled on
the temporary interface, and its newly learned IPv6 default route was removed
to avoid changing the host's default routing. This VLAN setup is not persistent
across host reboots.

On 2026-11-04 the owner moved the modem to VLAN 5. The VLAN 20 interface was
removed and replaced with `enp1s0f0.5` using the same address and IPv6
settings. The modem answered ping and HTTP at `192.168.1.1` through it, with a
neighbour MAC matching the label's base MAC. The host's default route was
unchanged.

Repeating the no-file TFTP probe succeeded: the router reported
`pcIP=192.168.1.225`, and the host received its TFTP read request for
`conf.json` from `192.168.1.1`. The probe deliberately returned a file-not-found
error, then closed its listener. Thus the transfer-path blocker is resolved;
no diagnostic payload, shell access or full-flash backup has yet been tested.

A private, 6,072-byte stock settings export was obtained through the normal
authenticated web backup function. It is stored outside the repository with
owner-only permissions. Its SHA-256 is:

```text
886073b3332d1fee4e89a754a2c02a08227b9f76158360d8bd5cb66c0cbfac1b
```

The web interface constructs its actual backup URL from model, hardware
version, build date/time and signing status; the literal `/cgi/conf.bin`
returned 404 on this firmware. The model-specific URL successfully returned
the export. This settings export has not been restore-tested and does not
include a verified complete flash backup or calibration backup.

Offline DES decryption of the firmware's bundled default configuration
recovered the vendor's XML-like data format. It corroborates interface names
`eth0.2`, `eth0.3`, `eth0.4`, `eth0.5` and modem interface `lte0`, but does not
prove physical port wiring or the names assigned by OpenWrt. Credentials and
decrypted configuration contents are not included here.

## Enclosure access

The model-specific [quick installation guide, page 1](https://static.mercusys.com/manual/7100000055_MB118-4G-Outdoor_QIG_REV1.0.020250320055947.pdf)
instructs: "Gently lift the tab on the waterproof cover and pull it outward to
detach the waterproof cover." Disconnect both DC and PoE power before
investigating the enclosure.

This procedure opens the connector/SIM compartment, not the main enclosure
or PCB. No reliable MB118-4G v1 full teardown, internal photographs or UART
pinout was found in the searches performed. Main-enclosure screw locations,
clips and the UART pads must be established from the actual unit rather than
inferred from another model.

## Implemented port

A first-draft `ramips/mt76x8` target was added:

- `target/linux/ramips/dts/mt7628an_mercusys_mb118-4g-v1.dts`
- `target/linux/ramips/image/mt76x8.mk` — `Device/mercusys_mb118-4g-v1`
  (`IMAGE_SIZE := 6720k`, `TPLINK_FLASHLAYOUT := 8Mmtk`).
- `mt76x8/base-files/etc/board.d/02_network` — single `eth0` LAN (switch
  registered with VLANs disabled, as on other single-port mt76x8 boards), LTE
  as the standard `wan`/`wan6` on `usb0` (DHCP over RNDIS).
- `mt76x8/base-files/etc/board.d/01_leds` — internet LED bound to `usb0`; LAN
  LED as a switch-link trigger on port 4 (mask `0x10`).
- `mt76x8/base-files/etc/init.d/mercusys-modem` and
  `mt76x8/base-files/usr/libexec/mercusys-modem` — a procd service, active
  only on this board, that starts the LTE data call and drives the signal
  LEDs (see "Modem helper service"). `comgt` is added to `DEVICE_PACKAGES`
  for its `gcom` AT-command tool.

Design choices and their basis:

- **8 MiB layout** from the stock kernel's static MTD table; kernel+rootfs
  merged into a `tplink,firmware` partition; all vendor partitions kept
  read-only. The region math is exact to 8 MiB.
- **2.4 GHz only** via the integrated MT7628 WMAC (no PCIe/5 GHz card).
- **EC200A-EL as a USB Ethernet modem** (ASR platform, no QMI): DHCP on the
  modem's network interface, with `option` AT ports for control. Stock firmware
  leaves the modem in RNDIS mode (`AT+QCFG="usbnet",3`), so `rndis_host` is the
  driver expected on first boot. `cdc_ether` is kept for ECM mode
  (`"usbnet",1`).
- **Modem interface named `usb0`** (observed on OpenWrt v25.12.6,
  2026-11-06: `rndis_host 1-1:1.0 usb0: ... RNDIS device`). The modem is USB
  `2c7c:6005`; interface 0 (class `e0/01/03`) and 1 bind to `rndis_host`,
  interfaces 2–4 to `option`. OpenWrt's
  `781-usb-net-rndis-support-asr.patch` does add `2c7c:6005` with
  `asr_rndis_info` (`FLAG_WWAN`, which would name it `wwan0`), but inserts it
  after the generic `USB_INTERFACE_INFO(USB_CLASS_WIRELESS_CONTROLLER, 1, 3)`
  entry. The first match wins, so the generic `rndis_info` binds and the
  device gets the default `usb%d` name. (Main's 6.18 patch has the same
  order.) The Cudy LT500 Outdoor in this tree also uses `usb0`. If the modem
  is switched to ECM (`"usbnet",1`), `cdc_ether` binds it instead and the name
  may change.
- **Ethernet on switch port 4 only** (`mediatek,portdisable = <0x2f>`). On
  the running stock firmware, `/proc/mt7628/esw_cnt` showed traffic only on
  port 4. The link register (`0x10110080` = `0xa180a110`) had the port 4 link
  bit (29) set, with no link on ports 0–3.

Verified so far: file/shell/DTS syntax, layout arithmetic, and name
consistency across DTS, recipe and board.d; **and a full toolchain image build**
(ramips/mt76x8) completed on an x86_64 Ubuntu host, first on SNAPSHOT and
again on the OpenWrt v25.12.6 release (`r33311-981c978eff`, kernel 6.12.112).
The v25.12.6 build produced:

| Image | Size | Purpose |
| --- | --- | --- |
| `...-squashfs-sysupgrade.bin` | 5249 KiB | Upgrade from a running OpenWrt |
| `...-initramfs-kernel.bin` | 5199 KiB | RAM-only boot (no verified way to load it yet; see "Recovery discrepancy") |

That first build also produced a `tftp-recovery.bin`; it was removed from the
recipe because it would erase the calibration/MAC partitions (see
"Calibration-overwrite risk" below). Any copy left over from that build must
not be flashed.

The DTS compiled under the kernel, the sysupgrade fits the 6720 KiB firmware
region, and the image metadata declares `supported_devices: mercusys,mb118-4g-v1`.
The included driver set is correct for the hardware: `kmod-mt7603` + `kmod-mt76`
(2.4 GHz WMAC), and `kmod-usb2` / `kmod-usb-net-rndis` /
`kmod-usb-net-cdc-ether` / `kmod-usb-serial-option` / `kmod-usb-serial-wwan`
for the EC200A modem (RNDIS as left by stock, or ECM, plus AT control).
On-device results (2026-11-06): boot, Ethernet, Wi-Fi client mode, the
modem's `rndis_host`/`option` functions and LTE data work, and all LED
devices register under their DTS names, and the Wi-Fi MAC matches the
owner's expectation for the label. Not yet checked: the Ethernet MAC, the
radio's calibration and range, Wi-Fi access-point
operation, the visual state of each LED, the reset button under OpenWrt,
and the modem power/antenna lines.

The images are build-host artifacts and are not committed to the repository.

### Assumptions that require on-device confirmation

1. **MAC/EEPROM offsets** (`romfile` `0xf100`, `radio` `0x0`) are confirmed
   from the flash dump. On OpenWrt the Wi-Fi MAC is correct; the Ethernet MAC
   and the radio calibration have not been checked yet.
2. **GPIO polarity and modem line levels** come from disassembly of the stock
   GPIO driver (see "GPIO findings"). They are expected to be correct, but no
   pin has been observed on hardware.
3. **Switch port** for the single Ethernet connector is port 4, observed on
   the running stock firmware; confirmed under OpenWrt (SSH, LuCI and TFTP
   recovery all work through it).
4. **Reset button** is GPIO 38, active-low, read by both the stock kernel and
   U-Boot.
5. **First-flash method:** a per-unit TFTP-recovery image (see "Installation
   method") is the only known path that preserves the unique data. It worked
   on the owner's unit on 2026-11-06.
6. **Modem network interface name** is `usb0`, observed under OpenWrt (see
   "Design choices"). An earlier reading of the driver table predicted
   `wwan0`; that was wrong.

## Installation method

> **⚠ Do not use the MB130 TFTP-recovery method on the MB118-4G.** On this
> model it permanently erases the per-unit MAC and Wi-Fi calibration (see
> below). The only known safe first-flash path is a per-unit recovery image
> built from a full flash dump of the same unit (see "Per-unit recovery
> image").
>
> The built `sysupgrade.bin` and `initramfs-kernel.bin` also declare
> `fw_length = 0x7a0000` in their TP-Link header (inherited from the `8Mmtk`
> layout), although this device's firmware partition is only `0x690000`.
> OpenWrt ignores that field, but a stock flasher that trusts it would write
> into `romfile`/`radio`. Only give these images to a running OpenWrt
> `sysupgrade`, a verified RAM-only load, or wrap them in the per-unit
> recovery image — never give them directly to the stock web UI or the U-Boot
> menu options that write flash.

For reference only, the supported MB130-4G v1 installs with no disassembly via
its bootloader's TFTP recovery (OpenWrt PR #17819): a `tftp-recovery.bin` is
served as `tp_recovery.bin`, and the bootloader fetches and flashes it when
Reset is held at power-up.

The MB118-4G runs the same U-Boot 1.1.3 with the same recovery mechanism, but
the method does **not** carry over safely. There are two confirmed differences
and one blocking risk:

- **Recovery subnet differs.** The MB118 bootloader environment defaults are
  `ipaddr=192.168.0.2` / `serverip=192.168.0.225` (the MB130 uses
  `192.168.1.x`). So the PC must be `192.168.0.225/24` and the device recovers
  at `192.168.0.2`.
- **Copy length exceeds the firmware partition.** The bootloader copies a
  fixed `0x7a0000` bytes (the same length as the firmware region of
  firmware-utils' `8Mmtk` layout), but this device's firmware partition is only `0x690000`. Changing
  the image layout cannot fix this, because the copy length does not depend on
  the file size.
- **⚠ Calibration-overwrite risk (blocking).** The MB118 recovery command
  `erase 0x20000 0x7a0000; cp.b … 0x20000 0x7a0000` writes `[0x20000,
  0x7c0000)`, which overlaps `romfile`, `radio` (Wi-Fi calibration) and
  `userdata`. A recovery image built the MB130 way (`pad-extra 128k |
  factory.bin`) is exactly 0x7c0000 bytes and is `0xff` from `0x6b0000`
  onward, so this command would deterministically blank `ispconfig`, `config`,
  `configbak`, all of `romfile` and `radio`, and the first 768 KiB of
  `userdata` — permanently destroying the per-unit MAC and Wi-Fi calibration.
  DTS `read-only` flags do not apply to the bootloader. For this reason the
  port does **not** build a `tftp-recovery.bin`. Whether the MB130's recovery
  avoids the problem by erasing only its firmware partition has not been
  verified from this tree.

### Per-unit recovery image (requires your own flash dump)

With a verified dump of the same unit, a recovery image can carry that unit's
own bytes for `[0x6b0000, 0x7c0000)`. The bootloader then rewrites
`ispconfig`, `config`, `configbak`, `romfile`, `radio` and the start of
`userdata` with identical contents. Such an image contains per-unit data, so
build it locally, never commit or share it, and only flash it on the unit the
dump came from.

Layout (`0x7c0000` bytes; the bootloader skips the first `0x20000`):

| File offset | Content |
| --- | --- |
| `0x000000` | `0x20000` bytes of `0xff` (not written) |
| `0x020000` | OpenWrt `sysupgrade.bin` with its fwtool metadata trailer removed, as `sysupgrade` would write it |
| up to `0x6b0000` | `0xff` padding |
| `0x6b0000` | dump bytes `0x6b0000`–`0x7c0000`, copied verbatim |

```python
import struct
dump = open("flash-dump.bin", "rb").read()  # concatenated 8 MiB dump
fw = open("openwrt-ramips-mt76x8-mercusys_mb118-4g-v1-squashfs-sysupgrade.bin", "rb").read()
assert len(dump) == 0x800000
while fw[-16:-12] == b"FWx0":  # strip fwtool trailers
    fw = fw[:-struct.unpack(">I", fw[-4:])[0]]
assert fw.endswith(b"\xde\xad\xc0\xde") and len(fw) <= 0x690000
img = b"\xff" * 0x20000 + fw + b"\xff" * (0x690000 - len(fw)) + dump[0x6b0000:0x7c0000]
assert len(img) == 0x7c0000 and img[0x6b0000:] == dump[0x6b0000:0x7c0000]
open("tp_recovery.bin", "wb").write(img)
```

Procedure, derived from the dumped bootloader:

1. Connect a PC directly to the Ethernet port and give it `192.168.0.225/24`.
   The bootloader uses `192.168.0.2`, not the MB130's `192.168.1.x`.
2. Serve `tp_recovery.bin` from a TFTP server on that address.
3. Power off, hold Reset, power on, and keep holding for more than 5 s until
   the TFTP transfer starts.
4. Wait for the erase and write of `0x7a0000` bytes to finish and the device
   to reboot. OpenWrt then answers at `192.168.1.1`.

`config`/`configbak` are restored to their state when the dump was taken.
That only matters if you later return to stock firmware. This path worked on
the owner's unit (2026-11-06), served by the macOS built-in TFTP server
(`/private/tftpboot`, `launchctl load -F /System/Library/LaunchDaemons/tftp.plist`;
the file must be world-readable while served, and should be removed afterwards
because it holds per-unit data). After that, upgrade with `sysupgrade` from
OpenWrt instead. The dump's integrity check covers only `0x0`–`0x6b0000`;
the per-unit region is additionally confirmed by matching per-partition MD5
sums from the live system. Re-check those sums immediately before flashing,
in case stock firmware has changed `config` since.

## Remaining software work

The hardware port gives OpenWrt the modem's network interface (`usb0`), its
AT ports (`option`, interfaces 2–4, which become `ttyUSB0`–`ttyUSB2`,
confirmed on OpenWrt), and the LEDs. The stock firmware does more in
userspace. This section records what stock does and what OpenWrt needs. The
LTE data call and the signal LEDs are implemented by the modem helper service
below. SMS, USSD and network selection are not implemented.

### Modem helper service

`/usr/libexec/mercusys-modem`, started by `/etc/init.d/mercusys-modem` with
procd respawn, only on `mercusys,mb118-4g-v1`. Every 15 s it:

1. Finds the AT port by USB interface number (interface 3, the stock
   firmware's AT port), not by `ttyUSB` index. If no port is present, it
   turns the signal LEDs off and retries.
2. If `wan` is the default setup (`proto dhcp`, `device usb0`, not
   disabled), it reads `AT+CGDCONT?` and picks the first context with a
   non-empty, non-IMS APN (2 with the stock profile). If none can be read, it
   does not dial. It then queries `AT+QNETDEVCTL?`. Unless the link is up
   (last field 1) on that context, it disconnects any other context, sends
   `AT+CGACT=1,<cid>` and `AT+QNETDEVCTL=1,<cid>,1`, and on `OK` signals
   `udhcpc` (SIGUSR1) to restart DHCP discovery on `wan`. State changes are
   logged with tag `mercusys-modem`. If `wan` is switched to another protocol
   (for example `ncm`, to set a custom APN) or disabled, it stops dialling
   and leaves the modem alone.
3. Sets `green:signal1`–`green:signal3` from `AT+CESQ` RSRP (≥ −95 dBm → 3,
   ≥ −105 → 2, ≥ −115 → 1), or from `AT+CSQ` (≥ 20 / ≥ 12 / ≥ 5) when RSRP is
   unknown.

It never writes the APN or any other stored modem setting. AT commands go
through `gcom` with `/usr/share/mercusys-modem/at.gcom`, serialized with
`flock`. comgt's own `/etc/gcom/runcommand.gcom` cannot be used: it only
reports success or failure and never prints the modem's reply (comgt's
received-character echo is commented out), so a first version of the helper
parsed nothing and fell back to dialling context 1. `at.gcom` instead
captures everything the modem sends for 2 s after the command. `ttyUSB2`
stays free for users and tools such as `sms-tool`.

Testing: the parsers and the connect sequence (with a stubbed AT transport)
are tested against replies captured from this modem under OpenWrt, under
both macOS `sh` and BusyBox `ash`; the real `gcom` with `at.gcom` was also run
against a fake EC200A on a pty. The manual AT sequence the helper performs
was confirmed on the device (see "LTE data connection").

### Space budget

The current sysupgrade payload, including `comgt` and the helper service, is
`0x520004` bytes (about 5.1 MiB) of the `0x690000`-byte firmware partition on
v25.12.6 (it was `0x560004` on SNAPSHOT with kernel 6.18).
That leaves about 1.44 MiB for the JFFS2
overlay. A build from source does not include the LuCI web interface unless
`CONFIG_PACKAGE_luci=y` is set (release images get it from the build-server
config); with LuCI the payload is `0x580004` bytes, leaving about 1.06 MiB for the
overlay, which holds both configuration and any installed packages. Anything
substantial should be built into the image rather than installed with `apk`,
and the image must still pass `check-size`. ModemManager
(`feeds/packages/net/modemmanager`, plus GLib, D-Bus, libqmi and libmbim)
does not fit. A small shell script using AT commands is the realistic
approach.

### LTE data connection (background for the helper service)

What stock does (from strings in `/usr/bin/cos` and `/usr/bin/mobile`):

- Puts the modem in RNDIS mode with `AT+QCFG="usbnet",3`. The modem keeps
  this across reboots, which is why it appears as `2c7c:6005` with
  `rndis_host`.
- Enables the modem's NAT/router mode with `AT+QCFG="nat",1`.
- Starts the data call with `AT+QNETDEVCTL=<op>,<cid>,<urc>`; the strings
  `=1,1,1` and `=1,2,1` are both connect (op 1) requests, for context 1 and
  context 2 respectively. The `mobile` daemon sends them over `ttyUSB1`.
- A live read-only query on the second AT port (`ttyUSB2`) on 2026-11-05
  returned:
  - `+QNETDEVCTL: 1,2,1,1` — connected (last field 1) on **context 2**,
    with URCs enabled.
  - `+CGDCONT`: context 2 holds the carrier's internet APN (`IPV4V6`) and
    context 8 is IMS. **No context 1 is defined**, so OpenWrt must dial
    context 2 (or whichever context holds the APN on another SIM).
  - `+CPIN: READY` (no SIM PIN), `+QCFG: "usbnet",3` and `+QCFG: "nat",1`.
  - `+CSQ: 27,99`. `AT+QENG="servingcell"` is not implemented on this ASR
    firmware; it answered `+EEMGINFO : 3, 2`.
- Then runs DHCP on the network interface (`lte0` on stock). The live stock
  system had a carrier-grade NAT IPv4 address (`100.64.0.0/10` range, /24
  mask) and a global IPv6 address on that interface. The modem therefore
  passes the carrier addressing through, with IPv6 by SLAAC.

What OpenWrt does with the current port: `wan` (DHCP) and `wan6` (DHCPv6/RA)
on `usb0`. Observed on the device on 2026-11-06:

- The modem does **not** bring up its internet context by itself without
  the stock daemon. Only context 1 (the network's default LTE bearer,
  IPv6-only, not listed by `AT+CGDCONT?`) and context 8 (IMS) are active
  (`AT+CGACT?`), and context 2 has no address.
- `usb0` links up, but DHCP gets no reply until a context is linked to the
  network interface.
- `AT+QNETDEVCTL=1,2,1` fails with `+CME ERROR: 0` while context 2 is
  inactive. `AT+CGACT=1,2` (→ `OK`) followed by `AT+QNETDEVCTL=1,2,1`
  (→ `OK`) gives `+QNETDEVCTL: 1,2,1,1`, and DHCP on `usb0` then returns the
  carrier's CGNAT address (`100.64.0.0/10` range, /8 mask).
- `AT+QNETDEVCTL=0,1,1` (disconnect context 1) answered `ERROR`, yet the
  link state afterwards was `0,0,0,0`.
- Registration: `+CEREG: 0,1` (home), `+COPS: 0,0,"<operator>",7` (LTE),
  `+CPIN: READY`.

The helper service above implements the dial step. The in-tree `ncm`
protocol (`comgt-ncm`) has a generic `quectel` entry, which connects with
`AT+qnetdevctl=1,${profile},1` and disconnects with `=0,${profile},0`. It is
not the default because:
- it always rewrites the context's APN with `AT+QICSGP` (blanking it when
  no APN is configured);
- `config_generate` gives `ncm` interfaces profile 1 and `pdptype ipv4`,
  while this modem's data context is 2.
It remains the way to set a custom APN: set `wan` to `proto ncm`,
`device /dev/ttyUSB2`, `profile 2`, `pdptype ipv4v6` and `apn`, and install
`comgt-ncm`.

The original design notes for a dial step follow:

1. A procd init script, or a hotplug script on the `usb0` add event. It
   waits for the AT port, then sends
   `AT+CGDCONT=<cid>,"IPV4V6","<apn>"` (only if an APN is configured;
   the stock profile already uses cid 2), then `AT+CPIN="<pin>"` (only if
   `AT+CPIN?` does not report `READY`), then `AT+QNETDEVCTL=1,<cid>,1`.
   Finally it brings `wan`/`wan6` up, or lets netifd retry.
2. A UCI section (for example `network.wan.apn`, `pin`, `cid`, `device`), so
   these settings survive sysupgrade and could later be exposed in LuCI.
   Default `cid` to 2, matching the stock profile.
3. A watchdog: if DHCP fails or traffic stops, query `AT+QNETDEVCTL?`; if
   the last field is not 1, re-send the connect. The disconnect operation
   value has not been confirmed for this firmware (Quectel documents
   `<op>` 0 = disconnect for this command family), so do not rely on it
   untested. As a last resort, pulse modem reset through the exported
   `modem_reset` GPIO (high asserts reset, as on stock).
4. For the AT transport, use `sms-tool` (`feeds/packages/utils/sms-tool`, a
   small C binary that has an `at` command) or `comgt`'s `gcom`
   (`package/network/utils/comgt`). Plain `echo`/`cat` on the tty is fragile
   because unsolicited result codes interleave with replies.

A cleaner long-term option is a netifd protocol handler, for example
`proto quectel-rndis`, modelled on `uqmi`'s `qmi.sh`. It would run the dial
in `proto_setup` and then hand `usb0` to DHCP. That is more code, but it is
how upstream would expect it to be integrated.

Open questions:
- AT ports: on the running stock firmware the `mobile` daemon holds
  `/dev/ttyUSB1` (USB interface 3), and `/dev/ttyUSB2` (interface 4) also
  answers AT commands. Under OpenWrt, interfaces 2–4 become
  `ttyUSB0`–`ttyUSB2` (confirmed); the helper uses interface 3 (`ttyUSB1`),
  leaving `ttyUSB2` for interactive use. The role of `ttyUSB0` is
  unconfirmed.
- Resolved: the modem does not reconnect its internet context by itself
  without the stock daemon (see above), so the dial step is required.
- Whether the modem's NAT mode (`"nat",1`) interferes with IPv6 or with
  bridging. The evidence suggests it does not: `mobile_cli get_rt_param`
  showed the router receiving the carrier's own addressing (CGNAT IPv4
  address and gateway, carrier DNS, IPv6 DNS), and the interface had a
  global IPv6 address.

### Signal-strength LEDs (implemented by the helper service)

On stock, `/usr/bin/mobile` reads the signal and writes a level of 0–3 to
`/proc/tplink/led_signal_strength`. The kernel maps that level to GPIOs:
level 1 lights GPIO 41, level 2 lights 41+40, level 3 lights 41+40+39. In
OpenWrt these are `green:signal1`–`green:signal3`, which no trigger drives.

Needed: a small procd service that polls every 10–30 s over the AT port and
writes `/sys/class/leds/green:signal{1,2,3}/brightness`. It can poll
`AT+CSQ`, which returns RSSI 0–31 or 99 for unknown, or `AT+CESQ` for
RSRQ/RSRP. `AT+QENG="servingcell"` is not available on this ASR firmware
(see above). It can share the
AT helper with the dial script, but must serialize access to the port. The
stock thresholds have not been extracted from `mobile`. Stock uses two
scales: `mobile_cli get_sig_lvl` reports 0–4, while the LED level written to
`/proc/tplink/led_signal_strength` is 0–3. One live sample (2026-11-05)
had `+CESQ: 51,99,255,255,20,50`, i.e. RSRQ about −10 dB (field 5 = 20)
and RSRP about −90 dBm (field 6 = 50, decoded as −140 + n dBm). At those
values the signal level was 4 and the LED level 3. A conventional
mapping is RSRP ≥ −95 dBm → 3, ≥ −105 → 2, ≥ −115 → 1, else 0, or by CSQ
≥ 20 / ≥ 12 / ≥ 5. Extracting the stock table from `mobile` would match the
vendor behaviour exactly.

The stock kernel also has an `led_lte` control, but its write handler only
records a mode. No separate LTE LED GPIO was found in the GPIO table, so the
internet LED (GPIO 44, `netdev` on `usb0`) covers that role.

### SMS, USSD and network selection (stock web UI features)

On stock, `mobile` and `cos` implement SMS inbox/send, USSD, PIN management,
manual or automatic network selection (`AT+COPS`), band locking
(`AT+QCFG="band",…`, `"nwscanmode"`) and data-usage statistics.

OpenWrt options:
- `sms-tool` covers SMS read/send/delete, USSD and raw AT commands from the
  shell. Its packages-feed LuCI front-end, `luci-app-sms-tool`, was not found
  in this tree's feeds. Any LuCI page would have to be written or taken from
  a third-party feed, and space-checked.
- Network selection, band locking and scan modes need only raw AT commands.
  They can be scripted, but have no UI.
- Data-usage statistics: `vnstat`/`nlbwmon` can count `usb0`, at an
  additional space cost.

None of this is required to boot or route. All of it is required for parity
with the stock web UI, and the acceptance criteria below list it separately.

### Modem antenna and DC lines

`modem_antenna` (GPIO 5) and `modem_dc` (GPIO 37) are exported at the stock
steady-state levels (high). Stock never changes `usb_antenna` from
userspace, so the internal/external antenna choice is presumably fixed by
the board. Its meaning (which level selects which antenna) is unverified.
`usb_dc` is written from a configuration value (`echo %d >
/proc/tplink/usb_dc`) whose UI meaning has not been traced.

### Data still to collect from the running stock firmware

The AT port and the modem's stored settings were collected on 2026-11-05
(see above). The stock shell has no `printf`; `echo -ne "AT…\r" >
/dev/ttyUSB2` with a background `cat /dev/ttyUSB2` works. Still useful, all
read-only:
- The stock signal level alongside raw values, to recover the stock bar
  thresholds: `cat /proc/tplink/led_signal_strength` together with `AT+CSQ`
  and `AT+CESQ`, ideally at a few different signal strengths.

## Feature acceptance criteria

"All features work" requires on-device validation, not just a successful build:

- Boot reliably, retain unique MAC addresses and calibrated 2.4 GHz Wi-Fi,
  and recover from a cold power cycle.
- Ethernet LAN management and WAN mode, correct link negotiation and traffic
  forwarding through the physical port.
- Wi-Fi association, encryption, guest isolation, regulatory settings and
  sustained traffic.
- LTE registration, APN and PIN handling, IPv4/IPv6, DNS, modem restart,
  reconnection after signal loss and power cycling; SMS and network selection
  need suitable modem-control software and explicit tests.
- Modem power/reset sequencing, reset-button behavior, all LEDs and signal
  indicators, including recovery after a modem disconnect.
- DC and supported PoE power sources, tested physically.
- Configuration persistence, sysupgrade, failsafe and an independently
  verified stock-recovery procedure that preserves unique flash data.
- Stock software features such as guest networking, scheduled access,
  filtering, DDNS, UPnP and traffic limits need explicit OpenWrt package and
  configuration choices, followed by functional tests and an image-size check.

MERCUSYS app integration, vendor cloud management, vendor cloud firmware
updates and vendor ISP-file updates are firmware-specific services. They are
not preserved automatically by an OpenWrt hardware port. Exact parity for
those services has not been established and must not be promised.
