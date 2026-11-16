# Realtek RTL8111 quad-port card fails in the chipset slot of a Dell OptiPlex 3050 SFF (Proxmox VE 9.2)

**Status:** unresolved. Workaround: use the CPU-fed x16 slot. **Keywords:** r8169, r8168, RTL8111, RTL8125, `NETDEV WATCHDOG: transmit queue 0 timed out`, `rtl_rxtx_empty_cond == 0`, `AER: PCIe Bus Error: severity=Uncorrectable (Fatal), type=Inaccessible`, `AER: device recovery failed`, `can't disable ASPM; OS doesn't have ASPM control`, Dell OptiPlex 3050, B250, Proxmox 9, quad-port NIC, PCIe switch.

## Summary

A quad-port gigabit PCIe x1 card (Realtek RTL8111, behind an on-card PCIe switch) fitted in the small x1/x2 slot of a Dell OptiPlex 3050 SFF stalls within about two minutes of the link coming up. The device becomes inaccessible on the PCIe bus (fatal AER error), the downstream port reset fails to recover it, and the driver's transmit watchdog then fires every few seconds until reboot.

The same card works in the x16 slot. A quad-port 2.5 GbE card (RTL8125, also behind an on-card PCIe switch) also works in the x16 slot. The onboard Realtek gigabit port works. With both add-in cards fitted at the same time, neither worked.

We did not find the cause. This page records what we tried, so others do not repeat it.

## Environment

- Dell OptiPlex 3050 SFF, Intel B250 chipset, BIOS 1.32.0 (the latest available at the time of writing).

- Proxmox VE 9.2 (Debian 13 "trixie"), kernel `7.0.14-28-pve`. An older Proxmox kernel (6.17 series) was also tried. 

- Card A: quad-port gigabit, RTL8111 chips (`lspci`: RTL8111/8168/8211/8411, rev 0c), x1 card with an on-card PCIe switch. \[add make/model\]

- Card B: quad-port 2.5 GbE, RTL8125 (rev 05), on-card PCIe switch. Only fits the x16 slot. 

- Onboard NIC: Realtek RTL8111/8168 (rev 15), driver `r8169`. Works throughout.

- Slots: the x16 slot is fed by the CPU. The small x1/x2 slot hangs off chipset (PCH) root port `00:1c.6`.

Topology of card A in the chipset slot (`lspci -tv`, trimmed):

```
+-1c.6-[03-08]----00.0-[04-08]--+-01.0-[05]----00.0  Realtek RTL8111/8168...
                                +-03.0-[06]----00.0  Realtek RTL8111/8168...
                                +-05.0-[07]----00.0  Realtek RTL8111/8168...
                                \-07.0-[08]----00.0  Realtek RTL8111/8168...
```

## Symptoms

Cable plugged into one port of card A, in the chipset slot, bridged into `vmbr0`. Within about 20 to 120 seconds of boot:

```
r8169 0000:0c:00.0: AER: PCIe Bus Error: severity=Uncorrectable (Fatal), type=Inaccessible, (Unregistered Agent ID)
r8169 0000:0c:00.0: AER: can't recover (no error_detected callback)
pcieport 0000:0a:03.0: AER: Downstream Port link has been reset (0)
pcieport 0000:0a:03.0: AER: device recovery failed
r8169 0000:0c:00.0 nic6: NETDEV WATCHDOG: CPU: 0: transmit queue 0 timed out 5001 ms
r8169 0000:0c:00.0: can't disable ASPM; OS doesn't have ASPM control
r8169 0000:0c:00.0 nic6: rtl_rxtx_empty_cond == 0 (loop: 42, delay: 100).
```

The sequence repeats every five seconds or so until reboot. The port never recovers.

Related lines at boot:

```
ACPI FADT declares the system doesn't support PCIe ASPM, so disable it
acpi PNP0A08:00: FADT indicates ASPM is unsupported, using BIOS configuration
pci 0000:0a:01.0: ASPM: current common clock configuration is inconsistent, reconfiguring
```

(The "inconsistent common clock" message appeared for all four downstream ports of the card's switch.)

## What we tried

| \# | Change | Result |
| - | - | - |
| 1 | BIOS: Advanced configurations, ASPM set from Auto to **Disabled** | `lspci -vv` then showed `LnkCtl: ASPM Disabled` on the endpoint. Failure unchanged. |
| 2 | Runtime power management off for the NICs (`echo on > /sys/class/net/nicN/device/power/control`) | No change. |
| 3 | Kernel parameter `pcie_port_pm=off` (one boot) | No change. Failure about 20 seconds after boot. |
| 4 | Different kernel (6.17 series, instead of 7.0) | No change. |
| 5 | Check for a newer BIOS | 1.32.0 is the latest. |
| 6 | Card A alone in the chipset slot (card B removed) | Same failure. |
| 7 | Card A and card B moved between slots | Both cards work in the **x16 slot**, alone. |
| 8 | Both cards fitted together | Neither card worked. |
| 9 | Out-of-tree driver: `r8168-dkms` (8.053.00-1 from Debian trixie non-free), `r8169` blacklisted, plus `r8168.aspm=0 r8168.eee_enable=0 pcie_aspm=off loglevel=3` | The fatal AER messages stopped appearing (note that `loglevel=3` can hide console messages, so check `dmesg`). The transmit stall remained: watchdog every 6 to 9 seconds, `Transmit timeout reset Device!`, `Device reseting!`, then `link up`. The device now recovered each time, but the port was unusable. |
| 10 | Offloads disabled (`ethtool -K nicN tso off gso off gro off sg off tx off rx off`) | No change. |


Moving the management cable to the onboard port (which uses the same `r8169` driver) always worked.

Not yet tried: forcing the PCIe link speed, and a non-Realtek card in the chipset slot.

## What we suspect

We do not have a root cause. Our reading:

- The kernel-level fatal error, "Inaccessible", means the endpoint stopped answering on the PCIe link and the port reset could not recover it. That is below the NIC driver, so driver swaps did not cure it.

- The fault follows the **slot**, not the card: card A fails in the chipset slot and works in the x16 slot. The onboard Realtek port, on its own chipset root port, is fine.

- The failing configuration always has an on-card **PCIe switch** behind a chipset root port on a narrow link. The switch, its clocking (the "inconsistent common clock" warnings), or the narrow chipset link are suspects. This is unproven.

- With both cards installed neither works, which suggests a shared resource (chipset lanes, power, or PCIe resources). Also unproven.

- The Arch Linux thread at [https://bbs.archlinux.org/viewtopic.php?id=258867](https://bbs.archlinux.org/viewtopic.php?id=258867) reports the same `rtl_rxtx_empty_cond == 0` message on RTL8111 cards, cured by `r8168`. It did not cure ours.

## Workaround

Put the card in the CPU-fed x16 slot and leave the chipset slot empty. A single card in the x16 slot worked for us.

If you need more ports from the small slot, an Intel card with no PCIe switch (for example a dual-port 82576 x1 card) is what we plan to try next. We have not tested it yet.

## If you have seen this

Please comment with your motherboard or system, BIOS version, kernel, card make and model, and anything that worked.

