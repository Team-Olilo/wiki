---
title: MikroTik RouterOS Internet Setup Guide
description: Configuring your main WAN connection on MikroTik for Openreach, CityFibre, or Freedom Fibre
category: Setup
author: Aydan Abrahams
lastUpdated: 14/08/2026
---

# MikroTik RouterOS Internet Setup Guide

This covers your **main internet connection** on MikroTik RouterOS (tested against RouterOS 7.x). If you're after Olilo's L2TP tunnel service instead, see the separate **L2TP Guide**, that's a different, additional service, not your main line.

This assumes a **stock RouterOS default configuration** (the one applied by `/system reset-configuration` or on first boot), which already has `WAN`/`LAN` interface lists and a `srcnat ... action=masquerade` rule for the `WAN` list, and a DHCP client already running on `ether1`. Adjust interface names to match your own hardware.

Check [Which Network Am I On?](./which-network-am-i-on.md) first, the three networks are configured very differently.

---

## Openreach (PPPoE)

Stock config puts a DHCP client on `ether1`, remove it first, PPPoE replaces it.

Olilo supports a 1500-byte MTU over PPPoE. Set the physical interface MTU to 1508 to leave room for the 8-byte PPPoE overhead.

```routeros
# Remove the default DHCP client on ether1, Openreach uses PPPoE, not DHCP,
# on the physical port
/ip dhcp-client remove [find interface=ether1]

# Allow a 1500-byte IP MTU plus the PPPoE overhead
/interface ethernet
set [find name=ether1] mtu=1508

# Create the PPPoE client
/interface pppoe-client
add name=olilo-wan interface=ether1 user="your-pppoe-username" \
    password="your-pppoe-password" max-mtu=1500 max-mru=1500 \
    add-default-route=yes use-peer-dns=no disabled=no

# Swap WAN list membership from ether1 to the PPPoE interface
/interface list member
remove [find interface=ether1 list=WAN]
add list=WAN interface=olilo-wan comment="Olilo PPPoE"
```

Your credentials are supplied by email after your order goes live, and are case-sensitive.

### IPv6 at MTU 1492

The MTU 1500 settings above are optional. Without them, PPPoE normally negotiates an MTU of 1492. RouterOS adjusts TCP MSS automatically for IPv4, but not IPv6. If you use IPv6 with an MTU of 1492, add this rule:

```routeros
/ipv6 firewall mangle
add chain=forward action=change-mss new-mss=clamp-to-pmtu \
    protocol=tcp tcp-flags=syn out-interface=olilo-wan \
    comment="Olilo PPPoE: clamp IPv6 TCP MSS to PMTU"
```

Otherwise, you'll have connectivity issues when remote services send IPv6 packets sized for a 1500-byte path.

---

## CityFibre (DHCP + VLAN 911)

CityFibre needs a VLAN tag of `911` on the WAN port, untagged traffic won't get an address at all.

```routeros
# Remove the default untagged DHCP client, it won't get anything usable here
/ip dhcp-client remove [find interface=ether1]

# Create the VLAN 911 interface on top of ether1
/interface vlan
add name=olilo-wan interface=ether1 vlan-id=911

# DHCP client on the VLAN interface
/ip dhcp-client
add interface=olilo-wan add-default-route=yes disabled=no

# Swap WAN list membership
/interface list member
remove [find interface=ether1 list=WAN]
add list=WAN interface=olilo-wan comment="Olilo CityFibre"
```

---

## Freedom Fibre (DHCP)

This is the simplest case, no VLAN, no PPPoE. If you're on stock config, `ether1` already has a working DHCP client and WAN list membership, there's nothing to change.

If you've since modified it, this restores the default:

```routeros
/ip dhcp-client
add interface=ether1 add-default-route=yes disabled=no

/interface list member
add list=WAN interface=ether1 comment="Olilo Freedom Fibre"
```

---

## IPv6 (all networks)

Request your routed `/48` via DHCPv6 Prefix Delegation on whichever interface is your WAN (`olilo-wan`, or `ether1` on Freedom Fibre):

```routeros
/ipv6 dhcp-client
add interface=olilo-wan pool-name=olilo request=address,prefix
```

Then assign a `/64` out of your delegated `/48` to your LAN bridge:

```routeros
/ipv6 address
add address=::1/64 from-pool=olilo interface=bridge advertise=yes
```

## Verify

Go to **https://test-ipv6.com**. You should see your ISP listed as `OLILO`, and both an IPv4 and IPv6 address. If IPv6 doesn't show up, reconnect the test device to Wi-Fi/ethernet and try again.

## Still stuck?

- **Discord:** discord.gg/olilo
- **Email:** support@olilo.co.uk

## Related guides

- Which Network Am I On?
- L2TP Guide
- IPv6 Not Working? Troubleshooting
- PPPoE Won't Connect
