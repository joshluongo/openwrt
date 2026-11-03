# MERCUSYS MB118-4G v1: bring-up findings

Recorded: 2026-11-03

Status: a first-draft target was added to this tree and **now builds cleanly**
(see "Implemented port" below). It has **not** been booted or flashed on
hardware. No flash or configuration writes were performed on the device during
this investigation. An Ethernet-only installation method has not been verified,
and the first flash remains the single irreversible step.

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
the MTD partition registration function. These are offline findings, pending
confirmation against the running device and a complete flash backup.

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

### Recovery discrepancy

The vendor bootloader contains environment defaults `ipaddr=192.168.0.2` and
`serverip=192.168.0.225`, and this recovery command string:

```text
tftp 0x80060000 tp_recovery.bin;erase tplink 0x20000 0x7a0000;cp.b 0x80080000 0x20000 0x7a0000
```

This is evidence, not an installation instruction. Its destination interval,
`[0x020000, 0x7c0000)`, overlaps the static kernel table's configuration,
`romfile`, `radio` and part of `userdata`. It is not safe to prescribe this
recovery path using a padded OpenWrt image without resolving that discrepancy
and preserving the device's unique data.

The bootloader also advertises a distinct "Load system code to SDRAM via TFTP"
menu option, alongside options that explicitly write flash. Whether and how
that RAM-only option can be selected requires console verification. Ethernet
TFTP transport alone does not prove console-free RAM boot is available.

## GPIO findings from the vendor kernel

The kernel registers `/proc/tplink/` handlers for modem power, boot mode, DC,
antenna selection, reset, LEDs and GPIO status. The following GPIO numbers
come from their control routines, not physical pin testing:

| Control | GPIO number(s) |
| --- | --- |
| Power LED | 46 |
| 2.4 GHz WLAN LED | 43 |
| Internet LED | 42 |
| Signal indicator, increasing strength | 41, then 40, then 39 |
| Modem USB power control | 36 |
| Modem boot control | 3 |
| Modem DC control | 37 |
| Modem antenna control | 5 |
| Modem reset control | 4 |

The common GPIO control routine is at kernel address `0x80133578`, assuming
the raw kernel's `0x80000000` load base. For example, the `usb_power` write
handler at `0x80133a64` selects GPIO 36 and performs a one-second state
transition when asked to power the modem. Those abstract states must not be
treated as verified physical GPIO polarity or a validated OpenWrt power sequence.

The reset-button mapping, Ethernet LED behavior, GPIO polarity, pin multiplexing
and startup states still need validation. GPIO assignments differ materially
from `mt7628an_mercusys_mb130-4g-v1.dts`.

## Information needed before implementing a flashable port

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
  as the standard `wan`/`wan6` on `eth1` (DHCP, ECM).
- `mt76x8/base-files/etc/board.d/01_leds` — internet LED bound to `eth1`.

Design choices and their basis:

- **8 MiB layout** from the stock kernel's static MTD table; kernel+rootfs
  merged into a `tplink,firmware` partition; all vendor partitions kept
  read-only. The region math is exact to 8 MiB.
- **2.4 GHz only** via the integrated MT7628 WMAC (no PCIe/5 GHz card).
- **EC200A-EL treated as ECM** (ASR platform, no QMI): DHCP on the modem's
  network interface, with `option` AT ports for control. Corroborated by the
  stock firmware bundling `cdc_ether`/`rndis_host` and no QMI driver.
- **Modem interface named `eth1`**, matching the in-tree MB130-4G, which
  carries the same EC200A-EL with `kmod-usb-net-cdc-ether`. usbnet names the
  device `eth*` when the modem reports a globally administered MAC and `usb*`
  otherwise, so this is unconfirmed for this unit; if it enumerates as `usb0`,
  change `wan` and the internet LED accordingly.
- **esw portmap left at the SoC default** (all PHYs active) so the single
  Ethernet port is reachable on first boot regardless of which physical port
  it maps to.

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
(2.4 GHz WMAC), and `kmod-usb2` / `kmod-usb-net-cdc-ether` /
`kmod-usb-serial-option` / `kmod-usb-serial-wwan` for the EC200A modem (ECM data
plus AT control), with `kmod-usb-net-rndis` as a fallback in case the modem
enumerates in RNDIS mode. **Not** verified: any on-device behaviour (boot, MAC/calibration
nvmem offsets, GPIO polarity, modem bring-up, Ethernet, Wi-Fi, LTE).

The images are build-host artifacts and are not committed to the repository.

### Assumptions that require on-device confirmation

1. **MAC/EEPROM offsets** (`romfile` `0xf100`, `radio` `0x0`) are carried from
   the 16 MiB MB130-4G and are unconfirmed for this unit. Wrong offsets mean
   wrong/zero MAC addresses and uncalibrated Wi-Fi.
2. **GPIO polarity and the modem power/reset sequence** are from the stock
   kernel's handlers, not pin testing. The DTS defaults power the modem on and
   release reset; these may need inverting.
3. **Switch port number** for the single Ethernet connector is unknown.
4. **Reset button GPIO** was not found in the stock GPIO handlers. The DTS
   provisionally declares GPIO38 (active-low, the MB130 mapping) as the reset
   key; this is unverified, so the failsafe button may not work.
5. **First-flash method** is unresolved. The stock web UI likely rejects an
   unsigned image, and the vendor U-Boot TFTP recovery overwrites a region that
   overlaps `romfile`/`radio`/`userdata` (see "Recovery discrepancy"), which
   would destroy per-unit MAC and calibration data. A safe install path must be
   established before flashing, since no UART recovery is available.
6. **Modem network interface name** is assumed to be `eth1`, as on the MB130.
   If it enumerates as `usb0`, update `wan` in `02_network` and the internet
   LED in `01_leds` together.

## Installation method

> **⚠ Do not use the MB130 TFTP-recovery method on the MB118-4G.** On this
> model it permanently erases the per-unit MAC and Wi-Fi calibration (see
> below). There is currently no known safe first-flash path.
>
> The built `sysupgrade.bin` and `initramfs-kernel.bin` also declare
> `fw_length = 0x7a0000` in their TP-Link header (inherited from the `8Mmtk`
> layout), although this device's firmware partition is only `0x690000`.
> OpenWrt ignores that field, but a stock flasher that trusts it would write
> into `romfile`/`radio`. Only give these images to a running OpenWrt
> `sysupgrade` or a verified RAM-only load — never to the stock web UI or the
> U-Boot menu options that write flash.

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

Because the enclosure is water-sealed (no UART to read the real erase size or
to back up flash), a safe first flash requires resolving this before writing:
either confirm the live bootloader actually limits the write to the firmware
region, or obtain verified per-unit contents of every vendor partition in
`[0x6b0000, 0x7c0000)` so a recovery image can be built that writes them back
unchanged. Image size alone does not establish preservation. Until then, the first flash is not
demonstrably non-destructive on this model.

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
