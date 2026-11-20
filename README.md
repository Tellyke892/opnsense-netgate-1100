# OPNsense on the Netgate 1100

Replace pfSense Plus on a **Netgate 1100** with **OPNsense 26.7 (aarch64)**, running from the internal eMMC and booting on its own.

Every step was tested on a real 1100 (U-Boot `2018.03-devel-18.12.3`, eMMC MMCHC DG4008, switch Marvell 88E6341), in the order that works. Dead ends from the first attempt are left out and explained under [Reference](#reference).

> **Unofficial.** OPNsense has no official ARM support; the image is a community build by Maurice Walker ([maurice-w/opnsense-vm-images](https://github.com/maurice-w/opnsense-vm-images/releases)). Netgate doesn't support this either. You can always go back to pfSense ([see below](#going-back-to-pfsense)).

**Status:** booting, eMMC, switch, VLANs and console all work.

**Updated 2026-11-20** with three tips from Maurice: use the serial image, erase the eMMC before writing, and leave the firmware mirror at its default. These three changes were not re-run on the test unit.

## Contents

- [What you need](#what-you-need)
- [Before you start](#before-you-start)
- [Part 1 — Prepare the USB stick](#part-1--prepare-the-usb-stick-on-the-pc)
- [Part 2 — Boot OPNsense from USB](#part-2--boot-opnsense-from-usb-u-boot)
- [Part 3 — First boot: login, GUI, SSH](#part-3--first-boot-from-usb-login-gui-ssh)
- [Part 4 — Install to the eMMC](#part-4--install-to-the-emmc)
- [Part 5 — U-Boot autoboot from the eMMC](#part-5--make-u-boot-boot-opnsense-from-the-emmc)
- [Part 6 — Configure the switch](#part-6--configure-the-switch-wan--lan--opt)
- [Part 7 — VLAN interfaces, assignment and IPs](#part-7--vlan-interfaces-assignment-and-ips)
- [Part 8 — After the install](#part-8--after-the-install)
- [Reference](#reference)

Ready-to-paste files are in [`files/`](files/):

| File | What it is |
| --- | --- |
| [`files/uboot-usb-boot.txt`](files/uboot-usb-boot.txt) | U-Boot commands for a one-off boot from USB (Part 2) |
| [`files/uboot-opnboot.txt`](files/uboot-opnboot.txt) | U-Boot environment for autobooting from the eMMC (Part 5) |
| [`files/05-switch`](files/05-switch) | Switch/VLAN startup script (Part 6) |

## What you need

| Item | Notes |
| --- | --- |
| Netgate 1100 | Works with a blank or pfSense eMMC |
| Micro-USB cable | For the serial console (the 1100's "Console" port) |
| Windows PC | Serial terminal (PuTTY or Tera Term), 7-Zip, qemu-img, OSFMount, Rufus |
| USB stick, 8 GB or more | A **USB 2.0** stick is strongly preferred; a Kingston DataTraveler 3.0 dropped off the bus repeatedly |
| Ethernet cable | PC to the 1100 |
| Netgate store account | Free; needed to download the Netgate installer (for the `.dtb` and for going back to pfSense) |

## Before you start

Save your way back first: the Netgate installer and a copy of the U-Boot environment let you return to pfSense at any time.

1. **Back up your pfSense config** (Diagnostics → Backup & Restore), if the box still runs pfSense.
2. **Download the Netgate installer** for the 1100 from shop.netgate.com (aarch64, e.g. `netgate-installer-v1.2-RELEASE-aarch64.img.gz`). Keep it: it holds the `.dtb` and reinstalls pfSense.
3. **Connect the serial console.** Micro-USB from the 1100's Console port to the PC. Open the COM port at **115200 8N1** (Silicon Labs CP210x driver if no COM port appears). Only one program can hold the port: close PuTTY before opening Tera Term, and vice versa.
4. **Get to U-Boot.** Power on and press a key at `Hit any key to stop autoboot`. The prompt is `Marvell>>`.
5. **Dump the environment and save it to a text file on the PC:**

```
printenv
```

The stock boot command is `bootcmd=run emmcboot; run scsiboot; run net;`. Keep the `ethaddr`/`eth1addr`… lines too. **Never run `env default -a`** (it wipes the board's MAC addresses) and **never run `usbrecovery`** unless you mean to erase the eMMC.

## Part 1 — Prepare the USB stick (on the PC)

The stick gets the OPNsense VM image, converted to raw, with the 1100's device tree (`.dtb`) added to its EFI partition before it is written.

1. **Download the image** from [maurice-w/opnsense-vm-images releases](https://github.com/maurice-w/opnsense-vm-images/releases): the file with **ufs + serial + vm + aarch64**, e.g. `OPNsense-26.7-ufs-serial-vm-aarch64.qcow2`. Use UFS, not ZFS (1 GB RAM). The serial variant is built for boards without a video output: its console is the serial port, so you get a login there from the first boot.
2. **Install qemu-img** (QEMU for Windows from qemu.weilnetz.de/w64) and convert to raw:

```powershell
& "C:\Program Files\qemu\qemu-img.exe" info .\OPNsense-26.7-ufs-serial-vm-aarch64.qcow2
& "C:\Program Files\qemu\qemu-img.exe" convert -p -O raw .\OPNsense-26.7-ufs-serial-vm-aarch64.qcow2 .\opnsense.img
```

The virtual size must be under 7.3 GiB to fit the eMMC (26.7 was 4.25 GiB).

3. **Extract the device tree** from the Netgate installer with 7-Zip: open the `.img.gz` → the `.img` → its FAT partition, and copy out **`armada-3720-netgate-1100.dtb`** (about 18 KB).
4. **Add the `.dtb` to the image with OSFMount** (PassMark, free). Windows cannot write to a USB stick's EFI partition (diskpart refuses on removable media, and this U-Boot has no `fatwrite`), so edit the image file instead:
   - Mount new → `opnsense.img` → pick the small FAT/EFI partition (260 MB) → **untick "Read-only drive"** → Mount.
   - Copy the `.dtb` to the root of that drive, next to the `efi` folder.
   - In Explorer, eject the drive; then OSFMount → **Dismount all**.
5. **Note the image hash** for Part 4: `Get-FileHash .\opnsense.img -Algorithm SHA256`.
6. **Write the stick with Rufus:** select `opnsense.img`. Rufus greys out partition scheme and file system for raw images; that is DD mode. Check the target drive, then START.

After writing, Windows shows no drive letter for the stick (EFI + UFS partitions). That is normal; cancel any "format this drive" prompts.

## Part 2 — Boot OPNsense from USB (U-Boot)

U-Boot loads the `.dtb`, patches it in memory, then starts FreeBSD's EFI loader. Plug the stick into the 1100's **USB-A port** (the serial cable stays in the micro-USB Console port). Cold-start, stop autoboot, then at `Marvell>>`:

```
usb reset
fatls usb 0:1 /
```

The listing must show `efi/` and `armada-3720-netgate-1100.dtb`. If `usb reset` reports `0 Storage Device(s) found`, run it again (USB detection on this U-Boot is flaky). Then:

```
load usb 0:1 $fdt_addr_r armada-3720-netgate-1100.dtb
fdt addr $fdt_addr_r
fdt resize 4096
fdt set / sn $serial
fdt set /soc/internal-regs/ethernet local-mac-address $ethaddr
fdt rm /soc/internal-regs@d0000000/mmc@d8000 mmc-hs400-1_8v
fdt rm /soc/internal-regs@d0000000/mmc@d8000 mmc-ddr-1_8v
fdt set /soc/internal-regs@d0000000/mmc@d8000 vqmmc-supply <0x3f>
fdt set /soc/internal-regs@d0000000/mmc@d8000 quirks <0x2000000>
load usb 0:1 $kernel_addr_r efi/boot/bootaa64.efi
bootefi $kernel_addr_r $fdt_addr_r
```

The first two `fdt set` lines copy Netgate's own boot script (serial number and MAC). The four `mmc@d8000` lines make the eMMC run at 50 MHz instead of 0.4 MHz ([why](#why-each-device-tree-change-exists)). **Do not add `max-frequency`.**

**At the OPNsense boot menu**, press **Space**, then **3** (loader prompt). At `OK`, give the USB stick time to appear:

```
set kern.cam.boot_delay=10000
set vfs.mountroot.timeout=30
boot
```

If you still land at `mountroot>`, type `ufs:/dev/gpt/rootfs` and Enter. Never press Enter on an empty line there: that aborts and panics.

## Part 3 — First boot from USB: login, GUI, SSH

The VM image is already an installed system: there is no `installer` login. Log in as **root / opnsense**.

On first boot OPNsense grows its partition to fill the stick and runs interface assignment. Answer: LAGGs **n**, VLANs **n**, WAN **Enter** (none), LAN **`mvneta0`**, then **y**. LAN comes up as 192.168.1.1/24 on plain `mvneta0`. At this stage the switch is in its default mode, so all three front ports reach it.

With the serial image the login prompt appears on the serial console straight away. **If you used the efi image instead**, it defaults to a video console and shows no login on serial: plug the PC into any front port, open https://192.168.1.1, and set **System → Settings → Administration → Console**: Primary console = **Serial console**, speed **115200**. Save and reboot.

**Enable SSH** for Part 4: System → Settings → Administration → Secure Shell → enable, **Permit root user login**, **Permit password login**, Save.

Useful checks from the shell (menu option 8):

```
ifconfig mvneta0
ls /dev/mmcsd*
dmesg | grep -iE 'mmcsd0|e6000'
```

Expected: `mvneta0` with MAC `f0:ad:4e:…` (your `ethaddr`), and `mmcsd0: 8GB <…> at mmc0 50.0MHz/8bit`. If the eMMC line says **0.4MHz**, the device tree fixes from Part 2 were not applied.

## Part 4 — Install to the eMMC

Copy the clean image from the PC to the 1100 and write it to the eMMC; don't clone the stick, because its partition already grew to the stick's size on first boot.

**On the PC** (PC plugged into a front port, SSH enabled):

```powershell
scp .\opnsense.img root@192.168.1.1:/root/
```

**On the 1100** (shell), verify the copy against the hash from Part 1 (takes 2–3 minutes):

```
ls -la /root/opnsense.img
sha256 /root/opnsense.img
```

If the hashes match, erase the eMMC's old partition table first, then write the image. On an erased disk there is no stale backup GPT header left to repair:

```
gpart destroy -F mmcsd0
dd if=/root/opnsense.img of=/dev/mmcsd0 bs=1m status=progress
gpart show mmcsd0
```

`gpart destroy` prints an error if the eMMC has no partition table; that is fine. If `gpart show` still marks the disk `CORRUPT`, run `gpart recover mmcsd0`.

Expected time: about 12 minutes at roughly 6 MB/s (reading the USB stick and writing the eMMC at once). Don't interrupt it. `gpart show` should list `efi (260M)`, `freebsd-ufs (4.0G)` and about 3 GB free; OPNsense grows into the free space on first boot.

Then `poweroff` and **remove the USB stick**. The stick and the eMMC both carry a partition labelled `rootfs`; with both present FreeBSD may mount the wrong one.

## Part 5 — Make U-Boot boot OPNsense from the eMMC

One saved variable, `opnboot`, does everything Part 2 did by hand, but from the eMMC (`mmc 1:1`). Cold-start, stop autoboot, then at `Marvell>>` paste this as **one line** (the single quotes matter). The same commands are in [`files/uboot-opnboot.txt`](files/uboot-opnboot.txt).

```
setenv opnboot 'mmc dev 1; load mmc 1:1 $fdt_addr_r armada-3720-netgate-1100.dtb; fdt addr $fdt_addr_r; fdt resize 4096; fdt set / sn $serial; fdt set /soc/internal-regs/ethernet local-mac-address $ethaddr; fdt rm /soc/internal-regs@d0000000/mmc@d8000 mmc-hs400-1_8v; fdt rm /soc/internal-regs@d0000000/mmc@d8000 mmc-ddr-1_8v; fdt set /soc/internal-regs@d0000000/mmc@d8000 vqmmc-supply <0x3f>; fdt set /soc/internal-regs@d0000000/mmc@d8000 quirks <0x2000000>; load mmc 1:1 $kernel_addr_r efi/boot/bootaa64.efi; bootefi $kernel_addr_r $fdt_addr_r'
setenv bootcmd_pfsense 'run emmcboot; run scsiboot; run net;'
saveenv
```

This saves the new variable and a copy of the stock boot command; autoboot is not changed yet. **Test it:**

```
run opnboot
```

It should boot to the OPNsense console without any input. Check in the shell that root is on the eMMC:

```
glabel status | grep rootfs
```

Expected: `gpt/rootfs  N/A  mmcsd0p2`. **Then make it the default:** reboot, stop autoboot, and run:

```
setenv bootcmd 'run opnboot'
saveenv
reset
```

From now on the 1100 boots OPNsense by itself, 2 seconds after power-on.

## Part 6 — Configure the switch (WAN / LAN / OPT)

The 1100 has one NIC (`mvneta0`) wired to a Marvell 88E6341 switch; a startup script splits the three front ports into separate VLANs, tagged towards the CPU. Do this from the **serial console**: once the switch changes mode, plain `mvneta0` LAN (and the GUI) stops working.

| Case label | Switch port | VLAN |
| --- | --- | --- |
| WAN | port3 | 4090 |
| LAN | port2 | 4091 |
| OPT | port1 | 4092 |

Port 0 is the CPU link and carries all three VLANs tagged. To confirm the mapping on your unit: `kldload e6000sw`, plug a cable into one port at a time, and run `dmesg | tail -3` (look for `e6000sw0portN: link state changed to UP`).

**1. Load the switch driver at every boot:**

```
echo 'e6000sw_load="YES"' >> /boot/loader.conf.local
```

**2. Create the startup script** ([`files/05-switch`](files/05-switch)). The root shell is `csh`, which mishandles here-documents, so switch to `sh` first:

```
sh
mkdir -p /usr/local/etc/rc.syshook.d/early
cat > /usr/local/etc/rc.syshook.d/early/05-switch << 'EOF'
#!/bin/sh
kldload -n e6000sw
etherswitchcfg config vlan_mode dot1q
etherswitchcfg vlangroup0 vlan 4090 members 0t,3
etherswitchcfg vlangroup1 vlan 4091 members 0t,2
etherswitchcfg vlangroup2 vlan 4092 members 0t,1
etherswitchcfg port3 pvid 4090
etherswitchcfg port2 pvid 4091
etherswitchcfg port1 pvid 4092
EOF
chmod +x /usr/local/etc/rc.syshook.d/early/05-switch
cat /usr/local/etc/rc.syshook.d/early/05-switch
exit
```

**3. Run it once and check:**

```
sh -x /usr/local/etc/rc.syshook.d/early/05-switch
etherswitchcfg info
```

Expected: `VLAN mode: DOT1Q`, port1/2/3 with pvid 4092/4091/4090, and vlangroups with members `0t,3`, `0t,2`, `0t,1`.

## Part 7 — VLAN interfaces, assignment and IPs

OPNsense needs one VLAN interface per port on top of `mvneta0`. FreeBSD interface names are limited to **15 characters**, and the names the console menu generates (`mvneta0_vlan4090`) have 16. The rename fails, the VLANs stay empty, and LAN, WAN and the firewall all fail. Step 2 renames them to `vlan0.4090`–`vlan0.4092`.

**1. Assign from the console** (menu option **1**). Auto-detection (`a`) cannot work here, because `mvneta0` is always linked to the switch; type the names.

- LAGGs: **n**; VLANs: **y**, then parent `mvneta0` with tags **4090**, **4091**, **4092**, then Enter on an empty line.
- WAN: `mvneta0_vlan4090`; LAN: `mvneta0_vlan4091`; Optional 1: `mvneta0_vlan4092`; confirm **y**.

**2. Shorten the VLAN device names (required).** In the shell, rename them in `config.xml`, where they appear in both the VLAN section and the interface assignments:

```
cp /conf/config.xml /root/config.xml.before-vlan-fix
sed -i '' -e 's/mvneta0_vlan4090/vlan0.4090/g' -e 's/mvneta0_vlan4091/vlan0.4091/g' -e 's/mvneta0_vlan4092/vlan0.4092/g' /conf/config.xml
grep -n 'vlan0\.409' /conf/config.xml
```

The `grep` must show 6 lines (three `<vlanif>`, three `<if>`). Reboot (option 6). The console summary then shows `LAN (vlan0.4091)`, and the log contains `vlan1: changing name to 'vlan0.4091'`. Check:

```
ifconfig -l
ifconfig vlan0.4091
```

Expected: `vlan0.4090 vlan0.4091 vlan0.4092` in the list, and `vlan0.4091` with `parent interface: mvneta0`, its tag, and `status: active`.

**3. Set the IPs** (menu option **2**), per interface. If LAN reports *"This IP address conflicts with another interface"*, an old address is still on `mvneta0`: Ctrl+C, run `ifconfig mvneta0 inet 192.168.1.1 -alias` in the shell, and try again. A WAN left on DHCP with nothing plugged in logs `dhclient … FAIL`; that is harmless.

**4. Reboot once more and verify** with nothing typed by hand:

```
ifconfig -a | grep -A6 '^vlan'
pfctl -s info | head -1
```

Pass criteria: each VLAN shows `parent interface: mvneta0` with its tag, LAN has its address, the console summary lists IPs, `pfctl` reports `Status: Enabled`, and no `Configuring firewall … failed` appears at boot. Only then connect the internet to **WAN**.

## Part 8 — After the install

Back up the three things this setup depends on outside OPNsense's own config, then keep the box light.

- **Change the root password** (console option 3) and turn SSH password login off again if you don't need it.
- **Back up the config:** System → Configuration → Backups. Also keep on the PC: the `05-switch` script, `/boot/loader.conf.local`, and U-Boot's `printenv` output (with `opnboot`).
- **Updates:** this is a community aarch64 build. The "(default)" mirror in System → Firmware → Settings already points to Maurice's aarch64 repo, so leave it as it is. Back up the config before each update. After an update, check that `05-switch` and `loader.conf.local` are still there.
- **Resources:** 1 GB RAM and 2 × Cortex-A53. Idle use was 540/979 MB. Skip heavy plugins (Suricata, Zenarmor, ntopng). The dashboard itself spikes the CPU; that is normal.
- **No hardware clock:** time is wrong until NTP reaches the internet over WAN.
- **Re-mapping ports later** means editing `05-switch` (pvid and vlangroup members) to match the new assignment.

## Reference

### Why each device tree change exists

Without these, FreeBSD's `sdhci_xenon` driver left the eMMC at 0.4 MHz (0.37 MB/s writes). With them: 50 MHz, 8-bit, 38.5 MB/s.

| Change | Reason |
| --- | --- |
| `fdt set / sn $serial` and `local-mac-address $ethaddr` | Same as Netgate's own `pfsenseboot`; gives `mvneta0` the real MAC |
| `fdt resize 4096` | Adds free space so properties can be added |
| `fdt rm … mmc-hs400-1_8v` and `mmc-ddr-1_8v` | Stops FreeBSD trying HS400/DDR modes |
| `vqmmc-supply <0x3f>` | The eMMC node has no regulator, so every voltage switch failed (`Failed to set VCCQ`). `0x3f` is the unused SD slot's regulator; the eMMC pad is fixed at 1.8 V in hardware, so nothing physical changes |
| `quirks <0x2000000>` | `SDHCI_QUIRK_BROKEN_MMC_HS200`: HS200 tuning fails on this board, so FreeBSD falls back to HS52. The loader tunable `hw.sdhci.quirk_set` does not work: the Xenon driver overwrites slot quirks from the device tree |
| **No** `max-frequency` | The Xenon driver uses it as the controller's base clock, not a limit; it breaks every clock divider (CRC errors, failed card ID) |
| Not used: `no-1-8-v`, `marvell,xenon-phy-slow-mode` | The first is ignored by the driver; the second caused CRC errors at 52 MHz |

### Which prompt am I at?

| Prompt | What it is | Notes |
| --- | --- | --- |
| `Marvell>>` | U-Boot | `setenv`, `fdt`, `load`, `bootefi` |
| `OK` | FreeBSD loader | `set name=value`, `boot`, `lsdev` |
| `mountroot>` | Kernel looking for root | `ufs:/dev/gpt/rootfs`; never an empty Enter |
| `db>` | Kernel debugger after a panic | `reset` |
| `root@OPNsense:~ #` | OPNsense shell (csh) | Run `sh` for here-documents |

### Troubleshooting

| Symptom | Cause and fix |
| --- | --- |
| `0 Storage Device(s) found` | Stick not in the 1100's USB-A port, or flaky detection: run `usb reset` again |
| Stick drops mid-boot (`umass0 … disconnected`) | USB 3.0 stick on this port: use a USB 2.0 stick or a USB 2.0 hub/extension |
| `EFI: Watchdog timeout` | U-Boot resets an EFI program after about 5 minutes; the loader was stuck (wrong disk or waiting at a prompt) |
| `FDT_ERR_BADMAGIC` / `BADSTRUCTURE` | No valid `.dtb` at `$fdt_addr_r`; reload it |
| No serial login prompt | You used the efi image (video console by default): use the serial image, or set Serial console in the GUI (Part 3) |
| eMMC at 0.4 MHz, `Failed to set VCCQ` | Device tree fixes not applied (Part 2 / Part 5) |
| eMMC errors after many test boots | Cold-start (unplug power about 10 s); `reboot` does not reset the eMMC |
| LAN unreachable after VLAN assignment | VLAN names longer than 15 characters: rename them (Part 7, step 2) |
| Rufus / OSFMount | Rufus writes raw images in DD mode automatically; eject in Explorer before dismounting OSFMount |

### Going back to pfSense

At `Marvell>>`: `setenv bootcmd 'run emmcboot; run scsiboot; run net;'` then `saveenv`. Then follow Netgate's reinstall guide with the Netgate installer on a USB stick (it uses `run usbrecovery`, which erases the eMMC). Restore your pfSense config backup afterwards.

### Sources

- [OPNsense aarch64 VM images (maurice-w)](https://github.com/maurice-w/opnsense-vm-images/releases)
- [FreeBSD `sdhci_xenon.c`](https://github.com/freebsd/freebsd-src/blob/main/sys/dev/sdhci/sdhci_xenon.c) and [`sdhci.h`](https://github.com/freebsd/freebsd-src/blob/main/sys/dev/sdhci/sdhci.h) (quirk values)
- [Netgate 1100: Reinstalling pfSense Plus](https://docs.netgate.com/pfsense/en/latest/solutions/sg-1100/reinstall-pfsense.html)

## Credits

Thanks to Maurice Walker for the aarch64 images and the review on the OPNsense forum.
