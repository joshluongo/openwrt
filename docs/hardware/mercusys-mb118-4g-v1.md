# MERCUSYS MB118-4G v1: bring-up findings

Recorded: 2026-11-03

Status: a first-draft target was added to this tree and **now builds cleanly**
(see "Implemented port" below). It has **not** been booted or flashed on
hardware. No flash or configuration writes were performed on the device during
this investigation. On 2026-11-05 the owner supplied a full 8 MiB flash dump;
its findings are under "Full flash dump" and they resolve the per-unit data
blocker for a TFTP-recovery install (see "Installation method"). That install
is still untested and remains the single irreversible step.

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
  as the standard `wan`/`wan6` on `wwan0` (DHCP over RNDIS).
- `mt76x8/base-files/etc/board.d/01_leds` — internet LED bound to `wwan0`; LAN
  LED as a switch-link trigger on port 4 (mask `0x10`).

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
- **Modem interface named `wwan0`.** The live stock system (2026-11-05)
  shows the modem as USB `2c7c:6005`, bound to `rndis_host` and `option`, with
  a locally administered MAC on its network interface. In this tree's kernel
  (6.18), `rndis_host` matches `2c7c:6005` to `asr_rndis_info`, which sets
  `FLAG_WWAN`, and usbnet always names such devices `wwan%d`. The MB130-4G's
  `eth1` therefore does not apply while the modem is in RNDIS mode. If the
  modem is switched to ECM (`"usbnet",1`), `cdc_ether` binds it instead and
  the name changes.
- **Ethernet on switch port 4 only** (`mediatek,portdisable = <0x2f>`). On
  the running stock firmware, `/proc/mt7628/esw_cnt` showed traffic only on
  port 4. The link register (`0x10110080` = `0xa180a110`) had the port 4 link
  bit (29) set, with no link on ports 0–3.

Verified so far: file/shell/DTS syntax, layout arithmetic, and name
consistency across DTS, recipe and board.d; **and a full toolchain image build**
(ramips/mt76x8, SNAPSHOT) completed on an x86_64 Ubuntu host. The build produced:

| Image | Size | Purpose |
| --- | --- | --- |
| `...-squashfs-sysupgrade.bin` | 5440 KiB | Upgrade from a running OpenWrt |
| `...-initramfs-kernel.bin` | 5398 KiB | RAM-only boot (no verified way to load it yet; see "Recovery discrepancy") |

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
**Not** verified: any on-device behaviour (boot, modem bring-up, Ethernet,
Wi-Fi, LTE, LEDs).

The images are build-host artifacts and are not committed to the repository.

### Assumptions that require on-device confirmation

1. **MAC/EEPROM offsets** (`romfile` `0xf100`, `radio` `0x0`) are confirmed
   from the flash dump; first boot should still show the label MAC and a
   calibrated radio.
2. **GPIO polarity and modem line levels** come from disassembly of the stock
   GPIO driver (see "GPIO findings"). They are expected to be correct, but no
   pin has been observed on hardware.
3. **Switch port** for the single Ethernet connector is port 4, observed on
   the running stock firmware; untested under OpenWrt.
4. **Reset button** is GPIO 38, active-low, read by both the stock kernel and
   U-Boot.
5. **First-flash method:** a per-unit TFTP-recovery image (see "Installation
   method") is the only known path that preserves the unique data. It is
   untested.
6. **Modem network interface name** is `wwan0`, derived from the live USB ID
   and the kernel's driver table (see "Design choices"). It has not yet been
   observed under OpenWrt.

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
That only matters if you later return to stock firmware. This path has not
been tested on hardware. The dump's integrity check covers only `0x0`–`0x6b0000`;
the per-unit region is additionally confirmed by matching per-partition MD5
sums from the live system. Re-check those sums immediately before flashing,
in case stock firmware has changed `config` since.

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
