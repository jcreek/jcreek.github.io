---
tags:
  - opnsense
  - nintendo switch
  - nat
---

# Opnsense Nintendo Switch Settings

_2025-04-19_

When trying to play some (mostly older) Nintendo Switch games with online multiplayer there are specific settings needed in Opnsense to ensure that the games can connect to other players and the Nintendo servers. This guide documents how I set up Opnsense to work with the Nintendo Switch.

To see if this applies to you, you can do an Internet Test on the Nintendo Switch. If you get a NAT Type of A or B, then you are good to go. If you get a NAT Type of C or D, then you will need to follow the steps below.

Alternatively, when trying to play the game Mario Kart 8 Deluxe, you may get the below error message:

```text
Error Code: 2618-0516

Unable to connect to the other console(s).
NAT traversal process has failed.
Please try again later.

If the problem persists, your network conditions may not be suited to stable online play.
```

By following these steps I changed from a NAT Type of D to a NAT Type of B.

This guide is based on:

- a post on the [Opnsense forums](https://forum.opnsense.org/index.php?topic=17499.0) and enabled me to play Mario Kart 8 Deluxe online with friends.
- a thread on [Reddit](https://www.reddit.com/r/OPNsenseFirewall/comments/g3sx2l/tip_opnsense_and_nintendo_switch_nat_rules/)
- another thread on [Reddit](https://www.reddit.com/r/NintendoSwitch/comments/lzfhyi/help_nat_type_d_error/)
- a post on [Nintendo's support site](https://en-americas-support.nintendo.com/app/answers/detail/a_id/22455/)

## 1. Assign a Static IP Address

Go to <https://10.0.0.1/services_dhcp.php?if=lan> where `10.0.0.1` is the IP address of your Opnsense router. Click on the `+` button to add a new DHCP static mapping.

It's important to note that your Nintendo Switch has two network interfaces: one for the Wi-Fi and one for the Ethernet. You will need to set up a static IP address for whichever interface you're intending to use for online multiplayer. You can find the mac address of the interface by going to `System Settings > Internet Settings > Internet Settings > Connection Test` on the Nintendo Switch.

## 2.Set up a NAT rule

Go to <https://10.0.0.1/firewall_nat_out.php> where `10.0.0.1` is the IP address of your Opnsense router. I selected the`Hybrid outbound NAT rule generation` mode. Add a new manual NAT rule by clicking the `+` button.

- Disabled: Unchecked
- Do Not NAT: Unchecked
- Interface: WAN
- TP/IP Version: IPv4
- Protocol: Any
- Source Invert: Unchecked
- Source Address: Nintendo switch (an alias to the switch's IP)
- Source Port: Any
- Destination Invert: Unchecked
- Destination Address:Any
- Destination Port: Any
- Translation/Target: WAN address
- Log: Unchecked
- Translation/port: Blank
- Static port: Checked
- Pool Options: Default

The remaining fields are all blank (Set Local Tag, Match Local Tag, No XMLRPC Sync and Desription).

## 3. Port Forward

Go to <https://10.0.0.1/firewall_nat.php> where `10.0.0.1` is the IP address of your Opnsense router. Click on the `Port Forward` tab and then click the `+` button to add a new port forward rule.

- Disabled: Unchecked
- No RDR(NOT): Unchecked
- Interface: WAN
- TCP/IP Version: IPv4
- Protocol: UDP
- Source:
- Destination/Invert: Unchecked
- Destination: Single host or network
- Destination Address: 10.0.0.104 (the IP address of the Nintendo Switch)
- Destination Port Range: 45000-65535
- Redirect target IP: 10.0.0.104 (the IP address of the Nintendo Switch)
- Redirect target port: 45000
- Pool Options: Default
- NAT Reflection: Use system default
- Filter rule association: Add associated filter rule
- Description: Nintendo Switch Online Multiplayer
- Log: Unchecked
- No XMLRPC Sync: Unchecked
