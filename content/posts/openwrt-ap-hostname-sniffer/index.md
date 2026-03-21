---
title: "Fixing the '?' Hostname Problem on OpenWrt Access Points"
date: 2026-03-09T22:00:00-07:00
draft: false
description: "OpenWrt dumb APs show '?' for client hostnames because /tmp/dhcp.leases is empty. A tcpdump script sniffs DHCP Option 12 from bridge traffic and writes lease entries that LuCI can display."
tags:
  - openwrt
  - networking
  - home-automation
  - dhcp
takeaways:
  - LuCI shows '?' hostnames on dumb APs because /tmp/dhcp.leases is empty without a local DHCP server
  - A tcpdump script can sniff DHCP Option 12 hostnames from bridge traffic on br-lan
  - The script writes to the lease file format that rpcd-mod-luci expects, so LuCI displays hostnames normally
  - A procd init script and uci-defaults hook ensure the solution survives reboots and firmware upgrades
cover:
  image: before-after.png
  alt: "Before and after: OpenWrt Associated Stations page showing unknown hostnames resolved to device names"
  relative: true
---

If you run OpenWrt as a dumb access point, you've probably noticed that LuCI's **Network > Wireless > Associated Stations** page shows `?` for every client hostname. This is because the AP doesn't run a DHCP server - another device (your router) handles that - so `/tmp/dhcp.leases` is empty and LuCI has nothing to look up.

The hostnames are right there, though. DHCP packets from clients flow through the AP's bridge, and they contain the hostname in Option 12. We just need to capture them.

## The Problem

LuCI resolves hostnames for wireless clients via `rpcd-mod-luci`, which reads `/tmp/dhcp.leases`. On a full router running dnsmasq as the DHCP server, this file is populated automatically. On a dumb AP with DHCP disabled, it's always empty.

The clients *are* sending their hostnames - every DHCP Discover and Request includes [Option 12 (Hostname)][1]. The packets transit the AP's bridge (`br-lan`) on their way to the router. We can sniff them with `tcpdump` and write the results to the lease file ourselves.

## Verifying the Data is There

Before writing any scripts, confirm you can see hostnames in the DHCP traffic:

```bash
tcpdump -l -i br-lan -n -e -vv 'udp port 67'
```

You should see output like:

```
BOOTP/DHCP, Request from aa:bb:cc:dd:ee:f1, length 291
      Client-IP 192.168.1.42
      Client-Ethernet-Address aa:bb:cc:dd:ee:f1
      ...
      Hostname (12), length 9: "my-laptop"
```

If `Hostname (12)` appears, you're good.

## The Approach

The DHCP exchange between a client and server looks like this:

1. **Client Request** - contains the hostname (Option 12), MAC address, and sometimes a requested IP
2. **Server ACK** - contains the granted IP (`Your-IP`), actual lease time (`Lease-Time`), and the client's MAC

Both packets pass through `br-lan`. The script correlates them: it stores the hostname from the Request, then when the ACK arrives for the same MAC, it grabs the actual lease time and IP, and writes a complete lease entry.

The script also learns the DHCP server's MAC automatically from the first Reply packet it sees, so it can filter out the server's own DHCP traffic (my upstream router was spamming Discovers every few seconds and polluting the lease file).

## The Script

Install `tcpdump-mini`:

```bash
apk add tcpdump-mini
```

Create `/usr/sbin/dhcp-hostname-sniff`:

```bash
#!/bin/sh
LEASE_FILE=/tmp/dhcp.leases
IFACE=${1:-br-lan}
SELF_MAC=$(cat /sys/class/net/br-lan/address)
FIFO=/tmp/dhcp-sniff.fifo
SERVER_MAC=""

rm -f "$FIFO"
mkfifo "$FIFO"

cleanup() {
    kill "$TCPDUMP_PID" 2>/dev/null
    rm -f "$FIFO"
    exit 0
}
trap cleanup TERM INT

tcpdump -l -i "$IFACE" -n -e -vv 'udp port 67' > "$FIFO" 2>/dev/null &
TCPDUMP_PID=$!

ether_src=""
is_reply=0
mac=""
hostname=""
ip=""
lease_time=""

while IFS= read -r line; do
    case "$line" in
        [0-9][0-9]:*)
            ether_src="${line#* }"
            ether_src="${ether_src%% *}"
            ;;
        *"BOOTP/DHCP, Reply"*)
            is_reply=1
            mac=""
            ip=""
            lease_time=""
            if [ -z "$SERVER_MAC" ]; then
                SERVER_MAC="$ether_src"
            fi
            ;;
        *BOOTP/DHCP*)
            is_reply=0
            mac=""
            hostname=""
            ip=""
            if [ -n "$SERVER_MAC" ] && [ "$ether_src" = "$SERVER_MAC" ]; then
                is_reply=2
            fi
            ;;
        *"Client-Ethernet-Address"*)
            mac="${line##* }"
            ;;
        *"Client-IP"*)
            ip="${line##* }"
            ;;
        *"Requested-IP"*)
            ip="${line##* }"
            ;;
        *"Your-IP"*)
            ip="${line##* }"
            ;;
        *"Hostname (12), length"*)
            _h="${line#*: \"}"
            hostname="${_h%%\"*}"
            ;;
        *"Lease-Time (51)"*)
            _lt="${line##* }"
            lease_time="$_lt"
            ;;
    esac

    # Client request: store hostname by MAC
    if [ "$is_reply" = 0 ] && [ -n "$mac" ] && [ -n "$hostname" ]; then
        eval "host_$(echo "$mac" | tr ':' '_')=\"$hostname\""
        eval "ip_$(echo "$mac" | tr ':' '_')=\"$ip\""
        mac=""
        hostname=""
    fi

    # Server reply: write lease
    if [ "$is_reply" = 1 ] && [ -n "$mac" ] && [ -n "$lease_time" ]; then
        if [ "$mac" = "$SELF_MAC" ]; then
            mac=""
            lease_time=""
            continue
        fi
        _key=$(echo "$mac" | tr ':' '_')
        eval "stored_host=\$host_$_key"
        if [ -n "$stored_host" ]; then
            if [ -z "$ip" ]; then
                eval "ip=\$ip_$_key"
            fi
            if [ -z "$ip" ]; then
                ip="0.0.0.0"
            fi
            expiry=$(($(date +%s) + lease_time))
            now=$(date +%s)
            tmp=$(mktemp /tmp/dhcp.leases.XXXXXX)
            awk -v m="$mac" -v n="$now" '$2 != m && ($1 > n || $1 == 0)' \
                "$LEASE_FILE" > "$tmp" 2>/dev/null
            echo "$expiry $mac $ip $stored_host *" >> "$tmp"
            mv "$tmp" "$LEASE_FILE"
        fi
        mac=""
        lease_time=""
        ip=""
    fi
done < "$FIFO"
```

A few things worth noting:

- **FIFO instead of a pipe** - the `while read` loop must run in the main shell process, not a subshell, so that the `trap` handler works and [procd can actually stop the service][2]. A pipe (`tcpdump | while read`) creates a subshell that never receives SIGTERM.
- **Shell parameter expansion** - `${line##* }` and `${line%%\"*}` instead of spawning `awk` or `sed` for each line. On a resource-constrained AP, avoiding subprocesses in the hot loop matters.
- **`eval` for hostname storage** - busybox ash has no associative arrays. We store hostnames keyed by sanitized MAC (`aa:bb:cc:dd:ee:ff` becomes `aa_bb_cc_dd_ee_ff`) using `eval`. It's ugly but it works.
- **Atomic writes** - `mktemp` + `mv` prevents partial reads if LuCI queries the lease file mid-write.
- **Expired entry cleanup** - the `awk` command that removes the old entry for the current MAC also drops any entries past their expiry timestamp.

## The Lease File Format

The script writes entries in the format `rpcd-mod-luci` [expects][3]:

```
<expiry_epoch> <MAC> <IPv4> <hostname> <client-id>
```

For example:

```
1773120600 aa:bb:cc:dd:ee:f1 192.168.1.42 my-laptop *
```

LuCI's `getDHCPLeases` function reads every entry, computes remaining time as `expiry - now`, and displays it. It does *not* filter out expired entries - they show as "expired" but remain visible until removed.

## Init Script

Create `/etc/init.d/dhcp-hostname-sniff`:

```bash
#!/bin/sh /etc/rc.common
USE_PROCD=1
START=95
STOP=10

start_service() {
    procd_open_instance
    procd_set_param command /usr/sbin/dhcp-hostname-sniff br-lan
    procd_set_param respawn 3600 5 0
    procd_set_param stderr 1
    procd_close_instance
}
```

Enable and start:

```bash
chmod +x /usr/sbin/dhcp-hostname-sniff /etc/init.d/dhcp-hostname-sniff
service dhcp-hostname-sniff enable
service dhcp-hostname-sniff start
```

Hostnames will populate as clients renew their DHCP leases. With a typical 600-second lease time, all devices should appear within 10 minutes.

## Surviving Firmware Upgrades

Three things need to persist across sysupgrade:

**1. The script itself** - `/usr/sbin/` is not preserved by default:

```bash
echo /usr/sbin/dhcp-hostname-sniff >> /etc/sysupgrade.conf
```

**2. The init script** - `/etc/init.d/` is under `/etc/`, so it's preserved automatically.

**3. Re-enabling the service and reinstalling tcpdump** - sysupgrade recreates `/etc/rc.d/` from installed packages, so the symlink that enables our service gets lost. A uci-defaults script seems like the right fix - it runs once on first boot after upgrade - but it races with package initialization. The package manager rebuilds `/etc/rc.d/` *after* uci-defaults has already run and deleted itself, wiping our symlink for good.

`/etc/rc.local` runs after everything else and on every boot, so it won't get stomped. The operations are all idempotent:

```bash
# Add to /etc/rc.local, before "exit 0":
if [ -x /etc/init.d/dhcp-hostname-sniff ]; then
    /etc/init.d/dhcp-hostname-sniff enable 2>/dev/null
    if ! command -v tcpdump >/dev/null 2>&1; then
        apk add tcpdump-mini
    fi
    /etc/init.d/dhcp-hostname-sniff start 2>/dev/null
fi
```

`/etc/rc.local` is under `/etc/`, so it's preserved across sysupgrade.

## Gotchas

- **Devices that don't send hostnames** - some devices with privacy features (e.g., Apple's Private Wi-Fi Address) may not include Option 12 at all. There's no fix for this short of static leases on the router.
- **The AP's own MAC** - `SELF_MAC` filters out the AP itself if it appears in DHCP traffic. The server's MAC is learned dynamically from the first Reply packet.
- **Multiple APs** - each AP runs its own sniffer and only sees traffic from clients connected to it. This is fine - each AP populates its own lease file for its own LuCI instance.

[1]: https://www.rfc-editor.org/rfc/rfc2132#section-3.14
[2]: https://forum.openwrt.org/t/solved-procd-service-doesnt-kill-its-children-processes/29622
[3]: https://github.com/openwrt/luci/blob/master/libs/rpcd-mod-luci/src/luci.c
