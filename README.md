# avon-barksdale — OPNsense firewall

Bare-metal OPNsense firewall running my home network: design, reasoning, and a [sanitized config export](config/config-avon-barksdale-sanitized.xml).

**Highlights:** default-deny rules with device groups · DNSSEC + DNS-over-TLS with forced local DNS · a documented root-cause investigation · planned move to VLANs and 802.1X.

> Work in progress. Mistakes and suggestions welcome. Open an [issue](../../issues).

![Network overview](images/network-diagram.png)

## How this was built

- **My own work:** firewall rules, device groups and aliases, DNSSEC validation and DNS-over-TLS in Unbound.
- **Built with LLM help:** forced local DNS (NAT redirect + DNS-over-TLS block), remote logging, remaining Unbound tuning. I understand what they do and am still learning the details.

## Hardware

Dell OptiPlex 5090 Micro (i5-10500T, 16 GB RAM, 256 GB SSD on ZFS).

The OptiPlex has only one network port. I added a second by putting an Intel I226-V 2.5 GbE card in the M.2 Wi-Fi slot (Key E carries a PCIe lane), cabled to an RJ45 port mounted in the case.

<details><summary>Photos</summary>

![Front](images/optiplex-front.png)
![Inside: I226-V, SSD, M.2 A+E slot](images/optiplex-inside.png)
</details>

## Firewall rules

Default deny, plus 8 allow rules. These replaced about 60 block rules.

| # | Source | Destination | Action |
|---|---|---|---|
| 1 | any | any, DNS-over-TLS (853) | block, logged |
| 2 | ADMIN | any | allow |
| 3 | any | firewall, DNS (53) | allow |
| 4–9 | each other group | internet only (`! PRIVATE_NETS`) | allow |
| – | everything else | – | default deny, logged |

![OPNsense LAN rules](images/opnsense-firewall-rules.png)

- **Groups** are IP-range aliases (ADMIN, HYPERVISOR, HOMELAB, LAN, MOBILE, IOT, REST, NETWORK). Each one maps to a planned VLAN.
- **`PRIVATE_NETS`** contains RFC 1918, CGNAT (`100.64.0.0/10`) and link-local, so "internet only" really means internet only.
- **WAN** has no allow rules, so nothing is reachable from outside. IPv6 is disabled entirely rather than half-secured.
- **Honest scope:** the network is still flat. Rules control traffic *through* the firewall, not between devices on the same switch. Real isolation comes with VLANs.

## DNS (Unbound)

- DNSSEC validation; upstream to Quad9 over DNS-over-TLS only.
- Ad/tracker blocklist, refreshed on a schedule.
- **Forced local DNS:** plain DNS to any server is redirected to the firewall, and outbound DNS-over-TLS is blocked. Devices with hardcoded resolvers still get filtered answers.
- Internal names (`*.internal`) via host overrides.

**Proof it works:** I asked Google's resolver for a name only my firewall knows:

![nslookup omar-little.internal 8.8.8.8 returns 192.168.0.2](images/nslookup-screenshot.png)

Google can't know `*.internal`, so the answer shows the firewall intercepted the query.

## Troubleshooting: firewall freezing every 12–24 h

- **Symptom:** after moving from a Proxmox VM to bare metal, the firewall hard-froze. No GUI, no SSH, no logs. Only a reboot recovered it.
- **Two things had changed at once:** bare metal, and Suricata running inline (netmap). I first suspected the new NIC.
- **Isolation:** I disabled Suricata and changed nothing else. The freezes stopped.
- **Conclusion:** the trigger is inline IPS, not the NIC. A silent total hang points to a kernel-level (netmap) issue.
- **Lesson:** change one thing at a time.
- **Status: open.** Next steps are kernel crash dumps, console output during test windows, and a hardware watchdog. Logs now go to a lab VM over syslog so the lead-up survives a freeze.

## Known limitations

- **Flat network:** groups are IP ranges, so a device that self-assigns an ADMIN address gets admin access. VLANs and 802.1X fix this.
- **DNS-over-HTTPS** can bypass forced DNS, because it shares port 443 with normal web traffic.
- **Anti-lockout** leaves GUI/SSH ports reachable from the whole LAN (login is still admin-only).
- **No GUI 2FA** yet, and config backups are manual.
- **No alerting:** logs are collected, but nothing notifies me.

## Roadmap

1. VLANs, one group at a time, then disable anti-lockout.
2. 802.1X (EAP-TLS) with FreeRADIUS; MAB for devices that can't do 802.1X.
3. Automated off-box backups and alerting.
4. Finish the freeze investigation; move threat lists to firewall aliases.

<details><summary>Management, logging and hardening details</summary>

- Web GUI: HTTPS, LAN only, logins limited to the ADMIN range, 15-min timeout.
- SSH: key-only, non-default port, named personal accounts, root disabled.
- Remote syslog to a lab VM (rsyslog, daily rotation, 14 days).
- fq_codel traffic shaping on WAN against bufferbloat.
- ZFS snapshot before every larger change, so a failed update rolls back in minutes.

![ZFS snapshots](images/opnsense-snapshots.png)
</details>

<details><summary>IDS/IPS (Suricata): off by decision</summary>

Rules are split into *drop* (high-confidence lists like abuse.ch and known C2) and *alert only* (broad behavioral rules). Suricata is off for daily use: inline mode caused the freezes and added noticeable latency. It's only enabled during planned test windows.
</details>