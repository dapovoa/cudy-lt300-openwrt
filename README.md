# Cudy LT300 v3.0 on OpenWrt — 4G router, MT7628AN

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![OpenWrt](https://img.shields.io/badge/OpenWrt-25.12.6-00B5E2)](https://openwrt.org/)
[![Target](https://img.shields.io/badge/target-ramips%2Fmt76x8-blue)](https://openwrt.org/docs/techref/targets/ramips)
[![Device](https://img.shields.io/badge/device-Cudy%20LT300%20v3.0-informational)](https://openwrt.org/toh/cudy/lt300_v3.0)

Flashing this box, getting a shell on it, and getting traffic through its LTE modem
without surprises. Measured on hardware, not copied from a wiki.

SoC MT7628AN, 1 core. 58 MB RAM, 16 MB flash. Wi-Fi 2.4 GHz only, 2x2. Two 100 Mbps
ports. Modem is a Marvell `2dee:4d57` LTE Cat.4 in RNDIS mode.

Ports, left to right on the case: `0 = LAN/WAN` (VLAN 1), `1 = LAN` (VLAN 2).

## Flash

Image for this device, release 25.12.6. 6095122 bytes, sha256
`f3416aa0bb507d4f34035e67d18292cf4015dfa403670f3664babfe3e6d65bfa`:

```
https://downloads.openwrt.org/releases/25.12.6/targets/ramips/mt76x8/openwrt-25.12.6-ramips-mt76x8-cudy_lt300-v3-squashfs-sysupgrade.bin
```

- checksums: https://downloads.openwrt.org/releases/25.12.6/targets/ramips/mt76x8/sha256sums
- release directory: https://downloads.openwrt.org/releases/25.12.6/targets/ramips/mt76x8/

Stock firmware will not accept that image, it only takes RSA-signed ones. The first flash
has to come from Cudy's signed intermediate:

- https://www.cudy.com/en-us/blogs/faq/openwrt-software-download

Check the profile exists for the release you intend to flash. Do not assume:

```sh
curl -s https://downloads.openwrt.org/releases/<REL>/targets/ramips/mt76x8/profiles.json \
  | grep -o '"cudy_lt300[^"]*"'
```

| Release | `cudy_lt300-v3` |
|---|---|
| 23.05.5 | no |
| 24.10.4 | no |
| 25.12.6 | yes |

Install chain, because stock firmware only accepts RSA-signed images:

```
OEM firmware -> Cudy-signed intermediate -> mainline sysupgrade
```

The intermediate is a full OpenWrt build with LuCI and a plausible `/etc/openwrt_release`.
It is a stepping stone, and it ships without a USB stack, so the modem never enumerates.
Recognise it:

```sh
cat /etc/openwrt_release     # 23.05.5 on a device 23.05.5 never supported
uci show network             # only loopback + lan, no wan/wwan
ls /sys/bus/usb              # no such directory
dmesg | grep -i usbcore      # empty
```

The hardware is present, the devicetree has `ehci@101c0000 ohci@101c1000 usbphy@10120000`.
Only the kernel drivers are missing.

Flash mainline:

```sh
scp openwrt-25.12.6-ramips-mt76x8-cudy_lt300-v3-squashfs-sysupgrade.bin root@192.168.1.1:/tmp/
ssh root@192.168.1.1 'sysupgrade -n /tmp/openwrt-25.12.6-*.bin'
```

Verify the image yourself before writing, because the 23.05 flasher does not:

```sh
sha256sum /tmp/openwrt-*.bin
strings -n 8 /tmp/openwrt-*.bin | tail -3
# { "metadata_version":"1.1", "supported_devices":["cudy,lt300-v3","R100"], ... }
```

On 23.05, `/sbin/sysupgrade` never calls `fwtool_check_image`. Only LuCI does, through
`/usr/libexec/validate_firmware_image`. The 25.12 image carries a trailer with magic
`FWx0`, which the 23.05 `fwtool` cannot parse, so LuCI reports `Image metadata not present`
and, since the target sets `REQUIRE_IMAGE_METADATA=1`, you must tick Force. The image is
fine, the validator is old.

`fwtool -i` takes `<output> <firmware>`. Passing only the image truncates it to 0 bytes.

Package manager on 25.12 is apk, not opkg:

```sh
apk update
apk add <pkg>
```

## Access

Dropbear and uhttpd ship listening on every interface. Bind them to the LAN:

```sh
uci set dropbear.main.Interface='lan'
uci set dropbear.main.GatewayPorts='off'
uci set uhttpd.main.listen_http='192.168.2.1:80'
uci set uhttpd.main.listen_https='192.168.2.1:443'
uci -q add_list dhcp.@dnsmasq[0].notinterface='usb0'
uci commit
```

LAN IP, with the prefix. Without `/24` you get a /32 and dnsmasq has no subnet to lease
from:

```sh
uci set network.lan.ipaddr='192.168.2.1/24'
uci commit network
```

### When the router shares 192.168.1.0/24 with the ISP router

Two devices on the same subnet, reachable over different paths, gives ARP flux and
asymmetric routing. It presents as a radio fault. Measured on one client, same AP:

| Client setup | tx retries | window | rate |
|---|---|---|---|
| Wi-Fi only | 155 | 146 s | ~1/s |
| cable + Wi-Fi, same subnet | 4732 | 71 s | ~67/s |

Thirty times the retransmissions at −28 dBm. Move one router to another subnet.

To reach the colliding router when you cannot add routes, go over IPv6 link-local:

```sh
nmcli con modify "<wired>" ipv4.method disabled ipv6.method auto ipv6.never-default yes
nmcli con up "<wired>"
ip -6 neigh show dev <iface>
ssh root@fe80::xxxx:xxxx:xxxx:xxxx%<iface>
```

IPv4 and DNS keep working through the other interface, so the session survives.

After a router reboot or a profile toggle, NetworkManager sometimes activates a stale
lease: interface up, `carrier=1`, an address, but no default route and 100% loss.
Waiting does not help:

```sh
nmcli con down "<wired>" && nmcli con up "<wired>"
```

## Communication

The stock image already wires the LTE modem as WAN, nothing to create:

```
network.wwan    device usb0   proto dhcp
network.wwan6   device usb0   proto dhcpv6
firewall zone wan = wan wan6 wwan wwan6, masq=1, forwarding lan -> wan
```

The modem is RNDIS (`rndis_host 1-1:1.0 usb0`), leases an address from itself, and owns
the APN. There is no APN field in OpenWrt for this device and that is correct.

It also exposes nothing you can talk to:

| Attempt | Result |
|---|---|
| `ATI`, `AT+CGMM`, `AT+COPS?` on `ttyUSB0..3` at 115200, with and without DTR | no response |
| Spontaneous traffic on all four ports over 4 s | 0 bytes |
| Modem gateway web UI, ports 80/443 | nothing listening |
| `ubus list \| grep -i modem` | only `network.interface.wwan` |

No band information, no band lock, no APN control, no QMI or MBIM switch.

### Path MTU is 1464, not 1500

Small requests succeed, large transfers stall at 0 bytes until timeout. The router itself
downloads fine, only forwarded traffic dies. PMTU black hole: the carrier drops oversized
packets and the ICMP fragmentation-needed never returns.

Measure with the DF bit set from a client behind the router:

```sh
for s in 1400 1420 1440 1450 1460 1470; do
  ping -c1 -W2 -M do -s $s -I <lan-iface> 8.8.8.8 >/dev/null 2>&1 \
    && echo "MTU $((s+28)) ok" || echo "MTU $((s+28)) fail"
done
```

```
MTU 1464 ok
MTU 1465 fail
```

Fix it on the interface. fw4's MSS clamping (`maxseg size set rt mtu`) then derives the
right MSS:

```sh
uci set network.wwan.mtu='1464'
uci commit network
ifup wwan
```

| | before, MTU 1500 | after, MTU 1464 |
|---|---|---|
| 6 MB download | 0 B, timeout at 30 s | 6037353 B in 1.23 s |
| throughput | none | 4.9 MB/s, about 39 Mbps |

Clamping alone does not save you, it clamps to the route MTU, so an interface claiming
1500 still black-holes. Re-run the DF loop after any PDP renegotiation.

Free CPU per forwarded packet, there is no hardware PPE on MT7628:

```sh
uci set firewall.@defaults[0].flow_offloading='1'
uci set firewall.@defaults[0].flow_offloading_hw='0'
uci commit firewall
```

## Optional: radio tuning

Channel census on a dense street, `iwinfo <iface> scan` and count:

| Channel | Neighbours |
|---|---|
| 1 | 20 |
| 6 | 16 |
| 11 | 10 |

Moving 1 to 11 gained 25.7 to 27.3 Mbps. Channel choice reshuffles contention, it does not
fix saturation.

HT40 is refused by the standard's coexistence rule:

```
20/40 MHz operation not permitted on channel pri=11 sec=7 based on overlapping BSSes
```

`option noscan '1'` makes the AP report 40 MHz and gains nothing, the client still
negotiates HT20 at MCS 15.

Two clients, same AP, one at a time:

| Client | Radio | Signal | Negotiated | Download |
|---|---|---|---|---|
| phone | unknown | −32 dBm | 144.4 MCS15 | 26.8 Mbps |
| laptop | MT7921 Wi-Fi 6 2x2 | −31 dBm | 144.4 MCS15 | 20.1 Mbps |

A better card scored equal or worse, so the client is not the limit. Retransmissions during
a transfer:

```
+19055 retries in 64 s, peak 3307 in 2 s (about 1653/s)
CPU average 29.7%, peak 57.3%
```

Nine to ten percent of roughly 111000 frames, and every retry costs a CSMA backoff window.

Flow offloading moves CPU, not throughput:

| | CPU avg | CPU peak | Download |
|---|---|---|---|
| off | n/a | 70% | 26.8 Mbps |
| on | 22.8% | 51.7% | 21-23 Mbps |

Bufferbloat, measured before reaching for SQM/cake:

| State | avg | max |
|---|---|---|
| idle | 62.5 ms | 324.9 ms |
| under load | 65.0 ms | 119.4 ms |

Loaded equals idle, so there is nothing to fix, and `usb0` already runs `fq_codel`. cake
would add per-packet CPU on a 580 MHz SoC for no gain.

Two clients per radio is the practical ceiling here, 20 to 27 Mbps. Only different
hardware fixes it, not configuration.

Wi-Fi and timezone, both mandatory to set explicitly:

```sh
uci set wireless.radio0.country='PT'
uci set wireless.default_radio0.ssid='<SSID>'
uci set wireless.default_radio0.encryption='psk2'
uci set wireless.default_radio0.key='<KEY>'
uci set wireless.default_radio0.disabled='0'
uci set system.@system[0].timezone='WET-0WEST,M3.5.0/1,M10.5.0'
uci set system.@system[0].zonename='Europe/Lisbon'
uci commit
```

The image also advertises IPv6 with an IPv4-only uplink, so clients try AAAA first and
stall:

```sh
uci -q delete network.lan.ip6assign
uci set dhcp.lan.dhcpv6='disabled'
uci set dhcp.lan.ra='disabled'
uci set network.wwan6.disabled='1'
/etc/init.d/odhcpd disable
uci commit
```

## Not possible here

5G, no 5G radio, Cat.4 hardware. Band lock or band info, serial ports dead. Manual APN,
lives in modem firmware. Extroot, the only USB bus is the internal modem. More flash, 16 MB
fully partitioned. Overclock, no cpufreq and the CPU is not the bottleneck. Modem firmware,
no recovery path.

## Package operations on 58 MB RAM

`apk add mwan3` plus its LuCI app, 24 packages, was OOM-killed mid-transaction. Left behind:
an orphan `/etc/apk/world` entry, a truncated index in `/var/cache/apk`, orphan policy
routing rules and mwan3 chains in iptables and nft.

```sh
ip rule show
# 2061: from all fwmark 0x3d00/0x3f00 blackhole
# 2062: from all fwmark 0x3e00/0x3f00 unreachable
```

Symptom is misleading, the router loses all outbound connectivity (`ping: sendto: Network
unreachable`, `wget: Operation not permitted`, DNS REFUSED) while LAN access and Wi-Fi keep
working. It looks like a broken package feed, it is not.

Deleting the package files does not remove kernel state. Only a reboot does.

```sh
sysupgrade -b /tmp/backup.tar.gz
echo 3 > /proc/sys/vm/drop_caches
apk add <one-package>
free -m
```

One package at a time, backup first, reboot instead of unpicking kernel state by hand.

## Buying for OpenWrt and LTE

Minimums learned from the limits above.

| Component | Minimum | Ideal | Why |
|---|---|---|---|
| RAM | 128 MB | 256-512 MB | 58 MB OOMs apk |
| Flash | 16 MB | 128 MB | 16 MB gives a 9.7 MB overlay |
| Wi-Fi | 5 GHz ac | Wi-Fi 6 | 2.4 GHz only gave 20-27 Mbps |
| Ethernet | 1x GbE | 4-5x GbE | 100 Mbps caps local traffic |
| CPU | dual 880 MHz | dual/quad A53 1.3 GHz | one core hit 50% at 23 Mbps |
| USB | optional | yes | extroot |
| Modem | mPCIe/M.2 with QMI or MBIM | Cat 6+ | RNDIS gives no control |

"OpenWrt-based" in marketing is not mainline support, that is the trap above. Silent
hardware revisions bite: WR3000 v2.0 is unsupported, some WR3000P units ship a newer
SPI-NAND that older images cannot boot.

Profiles verified present in 25.12.6:

```
mediatek/filogic : cudy_wr3000p-v1, cudy_wr3000h-v1, cudy_wr3000e-v1
mediatek/mt7622  : linksys_e8450, linksys_e8450-ubi
ipq806x/generic  : netgear_r7800
ramips/mt7621    : xiaomi_mi-router-4a-gigabit
ramips/mt76x8    : cudy_lt300-v3
```

Re-check before buying, this list ages fast.

## References

- https://openwrt.org/toh/cudy/lt300_v3.0
- https://firmware-selector.openwrt.org/
- https://openwrt.org/docs/guide-user/base-system/start
- https://openwrt.org/docs/guide-user/installation/generic.sysupgrade
- https://openwrt.org/docs/guide-user/troubleshooting/failsafe_and_factory_reset
- https://openwrt.org/docs/guide-user/network/wan/wwan/ltedongle

## License

This project is under the MIT License. See the [LICENSE](LICENSE) file for details.
