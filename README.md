# sqm-scripts-nss

Latency control (SQM) for Qualcomm NSS routers (IPQ806x: Netgear R7800, TP-Link Archer C2600 and relatives). The script `nss-rk.qos` plugs into [sqm-scripts](https://github.com/tohojo/sqm-scripts) and does the shaping on the NSS cores, so a fast line can be shaped with little CPU load. Besides the single fq_codel queue per direction it can, optionally, split the **upload** into three priority classes and steer traffic into them by DSCP and by static rules on ports, addresses and protocols.

## Contents

* [Origin and credits](#origin-and-credits)
* [Recommended firmware](#recommended-firmware)
* [Requirements](#requirements)
* [Installation](#installation)
* [First setup](#first-setup)
* [Options](#options)
* [Upload classes](#upload-classes)
* [Static rules](#static-rules)
* [Checking that it works](#checking-that-it-works)
* [Hotplug helper 12-sqm-nss](#hotplug-helper-12-sqm-nss)
* [sqm-autorate-nss](#sqm-autorate-nss)
* [Example: the author's R7800](#example-the-authors-r7800)
* [What has been tested](#what-has-been-tested)
* [Limitations](#limitations)
* [License](#license)

## Origin and credits

This repository is a fork of [rickkdotnet/sqm-scripts-nss](https://github.com/rickkdotnet/sqm-scripts-nss), where `nss-rk.qos` comes from. rickk wrote that script on the foundation of `nss.qos` and `simple.qos` by ricsc. This fork builds on their work; thank you.

This fork keeps what the original script did and adds to it: a start that rolls back on failure and retries, readiness checks for the eth device and the WAN, handling of `nssifb` per kernel, a reload of a running sqm-autorate, the optional upload classes with DSCP and static rules, and a hotplug helper for DSA builds. The documentation in this README was written from scratch for this fork.

## Recommended firmware

The script is developed and tested on the **DSA** flavour of the NSS builds, where the switch ports are `lan1` to `lan4` and `wan` and SQM runs on `eth0`. Install one of these builds, the `dsa-main` branch of either repository:

* https://github.com/gyimesibalazs/openwrt-ipq806x-nss-25.12/tree/dsa-main
* https://github.com/nihilt/openwrt-ipq806x-nss-25.12/tree/dsa-main

Both are described the same way in their README (state of November 2026): OpenWrt master with Linux 6.18, the Qualcomm NSS offload for IPQ806x, DSA instead of swconfig with NSS still accelerating the traffic, aimed at the Netgear R7800 and the TP-Link Archer C2600. On the R7800 several users report the DSA branch stable; the C2600 has been tested by the author of the builds. They are community builds, not official OpenWrt images, so before flashing read the README of the repository you pick: it has the build steps, the installation notes and the notes on moving from an swconfig NSS image (marked experimental there). Take a configuration backup first (`sysupgrade -b`) and keep a way to recover the router.

The NSS qdisc modules this script needs (`nsstbl`, `nssfq_codel`, and `nssprio` for the classes) come with these builds.

## Requirements

* An OpenWrt build with the NSS qdisc support: the `qca-nss-qdisc` and `nss-ifb` kernel modules and the NSS-patched `tc`. The stock `tc-full` or `tc-bpf` do not know the NSS qdiscs; if one of them replaced the patched `tc`, the script refuses to start and the log says `Unknown qdisc nsstbl`.
* `sqm-scripts` and `jsonfilter` (the package depends on them). `luci-app-sqm` is optional and gives you the web interface.
* `nft` (nftables), only for the optional upload classes.

## Installation

### With the recommended builds

The builds include SQM. Check which script version is installed:

    head -3 /usr/lib/sqm/nss-rk.qos

The options in this README need `nss-rk.qos` version `20261120.2-dscp` or later. If your build has an older one, update it by hand as described next.

### By hand

Copy the files of this repository to the router (`scp -O` works with the Dropbear of OpenWrt, which has no sftp server):

    scp -O files/usr/lib/sqm/nss-rk.qos files/usr/lib/sqm/nss-rk.qos.help root@192.168.1.1:/usr/lib/sqm/
    scp -O files/etc/hotplug.d/iface/12-sqm-nss root@192.168.1.1:/etc/hotplug.d/iface/
    ssh root@192.168.1.1 'chmod 755 /etc/hotplug.d/iface/12-sqm-nss; /etc/init.d/sqm restart'

`nss-rk.qos.help` is the text LuCI shows next to the script choice. The hotplug helper is optional, see [below](#hotplug-helper-12-sqm-nss).

### As a package in your own build

    echo "src-git sqm_scripts_nss https://github.com/nihilt/sqm-scripts-nss.git" >> feeds.conf
    ./scripts/feeds update sqm_scripts_nss
    ./scripts/feeds install sqm-scripts-nss

Then enable `sqm-scripts-nss` in `make menuconfig` (it is listed under "Extra packages"). The package installs `nss-rk.qos`, `nss-rk.qos.help` and the hotplug helper.

## First setup

In LuCI open **Network → SQM QoS**, add or edit a queue and set:

* the interface to the physical WAN device, `eth0` on the R7800 and the C2600;
* the download and upload rates in kbit/s. The NSS shaper is exact, so use a value a little below what the line really delivers;
* on the *Queue Discipline* tab, `fq_codel` as the discipline and `nss-rk.qos` as the script;
* tick *Enable this SQM instance* and save.

The same thing in `/etc/config/sqm` (the section name is yours to choose):

    config queue 'eth1'
        option enabled '1'
        option interface 'eth0'
        option download '830000'
        option upload '80000'
        option qdisc 'fq_codel'
        option script 'nss-rk.qos'

Everything else is optional. Options that LuCI does not show go into the same section: `uci set sqm.<section>.<option>='<value>'`, `uci commit sqm`, `/etc/init.d/sqm restart`. [`configs/sqm`](configs/sqm) is a complete commented example that lists every option, set or not. UCI ignores option names it does not know, so a typo does nothing: compare with `uci show sqm`.

Every option can also be given as an environment variable with its name in upper case (`NSS_WAN_IFACE`, `NSS_DSCP_CLASSES`, ...). The variable wins over the config file and is only seen by an SQM start that was given it.

With the defaults the shaper is one fq_codel queue per direction:

    eth0   (upload)    nsstbl 1: root  ->  nssfq_codel 10: (default queue)
    nssifb (download)  nsstbl 1: root  ->  nssfq_codel 10: (default queue)

`nssifb` is the redirect device created by the `nss-ifb` module; the script creates it and attaches the download shaper to it.

## Options

All options are optional; the default is in the second column.

### Start, WAN readiness and nssifb

A start is all-or-nothing: when a step fails, everything installed so far is removed again and the start is repeated a few times, which rides out NSS firmware timeouts at boot. Before shaping, the script waits, with a limit, for the eth device and for the logical WAN interface, so the boot never hangs on it.

| Option | Default | Meaning |
|---|---|---|
| `root_burst_up` | 1000 us worth of the rate | burst of the upload root, in bytes |
| `root_burst_down` | 1000 us worth of the rate | burst of the download root, in bytes |
| `nss_wait_timeout` | `30` | seconds to wait for the eth device (a hard requirement) |
| `nss_wan_iface` | `wan` | logical interface to wait for before shaping; `none` skips the check |
| `nss_wan_timeout` | `20` | seconds to wait for it. On timeout the script warns and shapes anyway (soft requirement) |
| `nss_wan_required` | `0` | `1`: fail instead of shaping when the WAN is not up after the timeout |
| `nss_start_attempts` | `3` | start attempts before giving up |
| `nss_keep_ifb` | `1` on kernels 5.10 and 6.18, else `0` | `1`: leave `nssifb` in place on stop and restart; `0`: delete it on stop and create it again on start |
| `nss_reload_autorate` | `1` | `1`: after every successful start ask a running sqm-autorate to reload and apply its rates and burst again. It never starts autorate. `0`: leave it alone |

About `nss_keep_ifb`: on 5.10 deleting `nssifb` is known to crash the kernel now and then. On 5.15 the ingress qdisc only works the first time it is attached, so the interface has to be recreated. On 6.18, 20 restarts in a row on an R7800 neither crashed nor stopped the download shaper from counting, so it is kept. Other kernels are unverified and get the recreate behaviour. The option overrides the default.

If hotplug starts the script while a complete NSS shaper is already installed, the script leaves it alone: the shaper lives in hardware and does not care about `ifup`, while bouncing `nssifb` in quick succession can crash the router.

### Queue tuning (standard sqm options)

`upload` and `download` (kbit/s) are the rates; with sqm-autorate running they are only the start values. The options below tune the `nssfq_codel` leaves, on `eth0` for upload and on `nssifb` for download:

| Option | Default | Meaning |
|---|---|---|
| `eqdisc_opts` / `iqdisc_opts` | filled in by the script | leaf options for upload / download, such as `limit 350 flows 1024 quantum 304 target 3.5ms interval 50ms`. What you give wins; what is missing is filled in (interval 100ms, flows 1024, quantum from the MTU, limit and target from the speed) |
| `elimit` / `ilimit` | 100 ms worth at the link speed, at least 200 packets | queue limit in packets, used only when `eqdisc_opts` / `iqdisc_opts` have no `limit` |
| `etarget` / `itarget` | `5ms` | codel target, used only when `eqdisc_opts` / `iqdisc_opts` have no `target` |
| `overhead` | `18` (14 ethernet + 4 NSS) | per-packet overhead added to the device MTU for the shaper; leave it unset |

`debug_logging '1'` writes a detailed log of start and stop under `/var/run/sqm/` (keep it `0` normally) and `verbosity` (0-8, default 5) sets how much SQM writes to the system log. `qdisc_advanced` and `qdisc_really_really_advanced` are LuCI toggles that only show or hide fields; the script does not read them.

The script does not read `squash_dscp`, `squash_ingress`, `ingress_ecn`, `egress_ecn` or the link layer fields (`linklayer`, `linklayer_adaptation_mechanism`, `tcMTU`, `tcTSIZE`, `tcMPU`): they belong to other sqm scripts and have no effect here.

### Upload classes

| Option | Default | Meaning |
|---|---|---|
| `nss_dscp_classes` | `0` | `1`: three upload classes behind the root (see below) |
| `nss_dscp_high` | `cs4 cs5 cs6 cs7 ef va` | DSCP names or numbers (1-63) for the high class; `none` for an empty list |
| `nss_dscp_bulk` | `cs1 le` | same for the bulk class; a DSCP in both lists stays high |
| `nss_dscp_match` | empty | static rules by protocol, address and port (see [Static rules](#static-rules)) |
| `nss_dscp_wan_dev` | the l3 device of `nss_wan_iface` | device the tag rules match as outgoing interface (`wan` on DSA, `pppoe-wan` on PPPoE); set it only when it cannot be found |
| `nss_dscp_sched` | `nssprio` | experimental: the NSS scheduler between the classes, `nssprio`, `nsswrr` or `nsswfq` |
| `nss_dscp_weights` | `4 2 1` | shares of high, normal and bulk for `nsswrr` / `nsswfq`: three whole numbers 1-1000 (`nssprio` ignores them) |

## Upload classes

With `nss_dscp_classes '1'` the upload shaper becomes `nsstbl` → `nssprio` (3 bands) → one `nssfq_codel` per band:

* `11:` high (band 1)
* `12:` normal (band 2, the default queue)
* `13:` bulk (band 3)

Forwarded packets that leave through the WAN device get a tag (`meta priority set 11:0` or `13:0`) from their DSCP, in the nft table `nss_dscp`. The NSS firmware queues an accelerated flow in the class whose handle equals that tag; packets that are not accelerated (ICMP, for instance) are queued by their tag one by one. Untagged traffic goes to the normal class. The classes are priorities, not strict lanes: under load bulk is slowed down, not starved.

    uci set sqm.<section>.nss_dscp_classes='1'
    uci set sqm.<section>.nss_dscp_high='cs4 cs5 cs6 cs7 ef va'   # the default
    uci set sqm.<section>.nss_dscp_bulk='cs1 le'                  # the default
    uci commit sqm
    /etc/init.d/sqm restart

* DSCP lists take names (`cs1`-`cs7`, `af11`-`af43`, `ef`, `va`, `le`) or numbers 1-63. DSCP 0 and everything not listed goes to the normal class.
* The tag is set from the **first packets of a flow**, the moment the firmware records it. A flow keeps its class for its whole life, even when the DSCP of later packets changes, and connections already running when SQM (re)starts stay in the normal class until they are opened again.
* Only **forwarded** traffic (LAN to WAN) is tagged, not traffic of the router itself.
* **Download is not classified.** In a test with tagged return packets the `nssifb` shaper ignored the tags and every download flow stayed in the default queue, so the download side stays one fq_codel queue.
* If the classes cannot be built (no `nssprio` in tc or kernel, ...) the script warns and builds the single queue, as if the option were off.
* With `nss_dscp_classes` off the shaper and the firewall are exactly what you had without it.

### Schedulers (experimental)

`nssprio` (priority bands) is the default and the proven one. `nsswrr` (weighted round robin, quantum in packets) and `nsswfq` (weighted fair queueing, quantum in bytes) share the upload by `nss_dscp_weights` instead. The 11.2 firmware accepts them and they steer like `nssprio`, but they brought no advantage: in a race of three uploads, one per class, `nssprio` finished the high class about 38 % sooner, and the bulk upload ended at the same time with all three. `nsshtb` is not offered because the 11.2 firmware refuses its second class. If the scheduler you ask for cannot be built, the script warns and builds `nssprio`; if that fails too, the single queue. sqm-autorate only knows `nssprio`: stop it when you use `nsswrr` or `nsswfq`.

## Static rules

`nss_dscp_match` decides the class by protocol, address and port, for traffic that carries no useful DSCP: a backup PC that should always be bulk, a call application that does not mark its packets, and so on. The rules need `nss_dscp_classes '1'`. They are looked at **before** the DSCP lists and the first rule that matches decides (a matching rule stops the `nss_dscp` chain for that packet; other firewall chains are not affected).

A rule is:

    CLASS [PROTO] [saddr A] [daddr A] [sport P] [dport P]

* `CLASS`: `high`, `bulk`, or `normal` (no tag: the flow stays in the normal class whatever its DSCP says).
* `PROTO`: `tcp`, `udp`, `icmp` (IPv4 and IPv6), `esp`, `gre` or `sctp`, or a comma list of them. With a port and no protocol the rule means tcp and udp. Ports go with tcp and udp only.
* `A`: a comma list of addresses or prefixes of **one** IP version (`192.168.1.20`, `10.0.0.0/8`, `2001:db8::/32`). On upload `saddr` is the LAN side of the flow and `daddr` the Internet side.
* `P`: a comma list of ports or ranges (`443`, `50000-50100`). For traffic that starts in your LAN the source port is random, so match the destination port (`dport`), the port of the server.

Several rules, one `list` entry each:

    config queue 'eth1'
        option nss_dscp_classes '1'
        list nss_dscp_match 'high udp dport 3478,19302-19309'
        list nss_dscp_match 'bulk saddr 192.168.1.50'
        list nss_dscp_match 'normal tcp dport 443 daddr 203.0.113.0/24'

or `uci add_list sqm.<section>.nss_dscp_match='high udp dport 3478'`, or a single `option` with the rules separated by `;` or by new lines. The three rules above are examples: use your own addresses and ports.

* At most 32 rules and 16 values per list. A rule that is not valid is ignored with a warning in the log (`logread -e SQM`); the others stay. Only letters, digits, spaces and `. : , / - ;` are allowed, anything else drops all static rules, with a warning.
* A rule without `saddr` applies to every device behind the router. To single out one device use `saddr`, and give that device a static DHCP lease so its address does not change.
* Like the DSCP rules, static rules see only the first packets of a flow: flows already running when SQM restarts stay in the normal class.
* If nft refuses the static rules, the DSCP rules are loaded without them (warning in the log). Nothing is tagged only when nft refuses those too.

## Checking that it works

    logread -e SQM | tail          # "DSCP classes: out via wan, high (11:) DSCP ..., 1 static rule(s) first"
    tc -s qdisc show dev eth0      # bytes sent per class: 11: high, 12: normal, 13: bulk
    nft list table inet nss_dscp   # every rule has a counter

With the classes on, `tc qdisc show dev eth0` lists the root, the scheduler and the three leaves (output of an R7800 shaped to 80 Mbit/s):

    qdisc nsstbl 1: root refcnt 2 buffer/maxburst 10654b rate 80Mbit mtu 1522b accel_mode 0
    qdisc nssprio 10: parent 1: bands 3 accel_mode 0
    qdisc nssfq_codel 12: parent 10:2 target 3.5ms limit 350p interval 50ms flows 1024 quantum 304 set_default accel_mode 0
    qdisc nssfq_codel 11: parent 10:1 target 3.5ms limit 350p interval 50ms flows 1024 quantum 304 accel_mode 0
    qdisc nssfq_codel 13: parent 10:3 target 3.5ms limit 350p interval 50ms flows 1024 quantum 304 accel_mode 0

The counters of the nft rules only count packets that take the normal path: the first packets of an accelerated TCP or UDP flow (a handful, up to a few dozen) and every packet of traffic that is not accelerated, such as ICMP. The bytes of an accelerated flow show in the `Sent` counters of the classes.

The nft rules are also saved, with `@WAN@` in place of the WAN device, in `/var/run/sqm/<iface>.nss-dscp.nft`. The [hotplug helper](#hotplug-helper-12-sqm-nss) loads that file again when needed.

## Hotplug helper 12-sqm-nss

`/etc/hotplug.d/iface/12-sqm-nss` (`chmod 755`) is a safety net for DSA builds. sqm-scripts' own hotplug script acts on the device that hotplug reports, which on DSA is the user port (`wan`) while SQM runs on the eth device (`eth0`), so it never matches and SQM would only ever be started at boot. When the logical WAN interface (`nss_wan_iface`, default `wan`) comes up, this helper does two things and nothing else:

1. **SQM is not running** for an `nss-rk.qos` instance: it starts it, at most once per 60 seconds and instance.
2. **SQM is running** and `nss-rk.qos` saved its class rules: it loads them again when the nft table is gone, a chain is missing, or the table matches another WAN device than the interface has now (the WAN device was not known when SQM started, PPPoE came back under another name, ...). When the table is already right nothing is loaded, so its counters stay. The saved file holds the DSCP rules and the static rules.

It waits up to 30 seconds for a running SQM start or stop to finish, never restarts a running instance and never touches `tc` or `nssifb`. Its messages go to the system log:

    logread | grep sqm-nss

## sqm-autorate-nss

The patched `sqm-autorate.pl` of [sqm-autorate-nss](https://github.com/nihilt/sqm-autorate-nss) works with these scripts. It reads the counters of the `nsstbl` root and, when the root has an `nssprio`, changes only the rate of the root and leaves the three upload queues alone. Older versions read the wrong counters. After every successful SQM start the script asks a running autorate to reload (`nss_reload_autorate`), because SQM has just put the static rates back. Autorate only knows `nssprio`: stop it when you use `nsswrr` or `nsswfq`.

## Example: the author's R7800

Values from [`configs/sqm`](configs/sqm), a Netgear R7800 on a DSA build with an 830 Mbit/s down and 80 Mbit/s up line:

    config queue 'eth1'
        option enabled '1'
        option interface 'eth0'
        option download '830000'
        option upload '80000'
        option qdisc 'fq_codel'
        option script 'nss-rk.qos'
        option eqdisc_opts 'limit 350 flows 1024 quantum 304 target 3.5ms interval 50ms'
        option iqdisc_opts 'limit 3600 flows 1024 quantum 768 target 3.5ms interval 50ms'
        option nss_dscp_classes '1'

The interval and target are shorter than the defaults (100 ms and 5 ms), so codel reacts sooner. A small `quantum` makes the scheduler serve small packets such as calls and games sooner relative to bulk transfers. The limits follow the speed of each direction. Treat the numbers as a starting point and measure on your own line.

## What has been tested

On a Netgear R7800 (DSA, kernel 6.18, NSS firmware NSS.AK.11.2-9-002-R), IPv4, `nssprio`, 80 Mbit/s upload:

* Upload classes by DSCP: TCP flows, and a UDP flow (a WhatsApp call), land in the class their DSCP maps to.
* Static rules: a TCP rule by port moves a flow without DSCP into the high class, and a rule by address and port into the bulk class; a `normal` rule keeps an EF-marked flow in the normal class; a UDP rule moves a WhatsApp call into the bulk class; an ICMP rule moves pings into the high class; a flow that matches no static rule still follows the DSCP rules.
* **Not tested on the router:** IPv6 rules (only checked for syntax), PPPoE, and static rules for `esp`, `gre` and `sctp`.

## Limitations

* fq_codel is the only queue discipline the script accepts; it refuses the others.
* Traffic classes exist for **upload** only and are optional; the download side is one queue.
* ECN marking is not available, and DSCP squashing (`squash_dscp`, `squash_ingress`) is not handled by this script.
* Link layer compensation (the `linklayer` family of options) has no effect.
* The class of a flow is fixed from its first packets; a flow running when SQM starts stays in the normal class.
* The shaper works on the eth device that carries the WAN traffic (`eth0` on the R7800): use it as the interface, not a user port such as `wan`.

## License

The repository carries the GNU General Public License version 3, see [LICENSE](LICENSE).
