# avon-barksdale — OPNsense firewall

The firewall and router for my home lab, running OPNsense on a Dell OptiPlex.

> **Status:** living project, updated as the lab changes.
> **Last verified against the running system:** 2026-10-26

> **TL;DR:** Deny by default: every allowed flow is a rule I wrote on purpose.
> Each device group gets only the internet and the firewall's DNS, and nothing else
> passes through the firewall between groups. The network is still flat, so devices on
> the same switch can reach each other directly until VLANs are in place.
> DNS is validated, encrypted and forced through the firewall.

Known gaps are listed in [Known limitations](#known-limitations). Improvements are
welcome as an [issue](../../issues).

> **Sanitized:** passwords, keys, certificates, usernames and public IPs are removed
> from the [config export](config/config-avon-barksdale-sanitized.xml), and screenshots
> are redacted.

![Network overview](images/network-diagram.png)

---

## How this was built

- **On my own, as a beginner:** the first setup on Proxmox, the firewall rules, device
  groups and aliases, Unbound with DNSSEC, DNS-over-TLS, and Quad9 with its backup server.
- **With an LLM as a tutor:** most of what came after, including forced local DNS,
  remote logging, Unbound tuning, hardening and the freeze investigation. I can explain
  what each part does, and I'm still learning the details.

---

## Design decisions

Each choice below has a reason, and most had an alternative I decided against.

### 1. Deny by default

OPNsense ships with "allow LAN to anything" rules for IPv4 and IPv6. **I disabled both.** Every allowed flow is now a rule I wrote on purpose, and everything else hits the default deny, which is logged.

*Rejected alternative:* keeping the default rules and blocking known-bad traffic. That's allow-by-default with extra steps.

### 2. Full isolation instead of trust tiers

Every device group is isolated. No group can reach another group. Each one gets the internet and the firewall's DNS, and nothing else. The only exception is ADMIN, because I need to manage everything from somewhere.

Default deny plus 8 allow rules replaced about 60 block rules. Rules are evaluated top-down, and the first match wins:

| # | Source | Destination | Action |
|---|---|---|---|
| 1 | any | any, DNS-over-TLS (853) | block, logged |
| 2 | ADMIN | any | allow |
| 3 | any | firewall, DNS (53) | allow |
| 4–9 | HYPERVISOR, HOMELAB, LAN, MOBILE, IOT, REST | internet only (`! PRIVATE_NETS`) | allow |
| – | everything else | – | default deny, logged |

![OPNsense LAN rules](images/opnsense-firewall-rules.png)
*The live LAN rule list. The default allow rules are disabled at the top.*

- **"Internet only" really means internet only.** `PRIVATE_NETS` covers RFC 1918, CGNAT (`100.64.0.0/10`) and link-local, so a group can't sneak into another private range either.
- **NETWORK (`.201–.250`) gets no allow rule at all.** Switches and access points don't need the internet, so they don't get it.
- **The DoT block sits above ADMIN on purpose.** Not even my own devices can skip the firewall's DNS.
- **IPv6 is disabled entirely** rather than half-secured.

*Rejected alternative:* trust tiers, where each group is blocked only from groups more trusted than itself. That gives fewer rules to think about, but more allowed paths, and every allowed path is something an attacker can use. Full isolation flips that: there are fewer allowed paths, and every one of them is a rule I chose on purpose.

**Honest scope:** this isolates traffic that passes *through* the firewall. The network is still a flat /24, so devices on the same switch can reach each other directly. VLANs close that gap (see [Roadmap](#roadmap)).

### 3. Named for VLANs before VLANs exist

Each group already has its planned VLAN ID. When the new switch is in, moving to real VLANs is a migration, not a redesign. *(The full table is in [Reference](#reference).)*

### 4. IDS/IPS (Suricata): designed, tested, and off for now

**The design:** not every rule set deserves the power to drop traffic, so I split them by how often they're wrong:

- **Drop:** high-confidence lists with few false positives (abuse.ch feeds, known botnet and Cobalt Strike C2, exploits, phishing, worms).
- **Alert only:** broad behavioral rules (scans, malware, coinminers, tor), which I review before I trust them.

It runs inline (netmap) on WAN, with nightly rule updates and hardware offloading disabled, as inline mode requires.

**Why it's off: two reasons.**
1. **Latency.** I game, and inline inspection added noticeable delay. For a home network, I chose responsiveness over deep inspection and kept the firewall rules as the main defense.
2. **An open bug.** On bare metal, inline Suricata makes the firewall hard-freeze. The investigation is below, and it's still open.

#### Troubleshooting: firewall freezing every 12–24 hours (open)

- **Symptom:** after I moved OPNsense from a Proxmox VM to bare metal, it hard-froze every 12–24 hours. No GUI, no SSH, no logs. Only a reboot brought it back.
- **First suspect: the Intel I226-V,** the new 2.5 GbE card and a known troublemaker on FreeBSD. Further research ruled it out.
- **The clue:** Suricata ran inline for two weeks as a VM on Proxmox with no freezes. There, Linux drove the hardware and OPNsense only saw virtual NICs. The freezes started only when FreeBSD drove the hardware directly.
- **Caveat:** the two setups weren't identical. The VM was my beginner setup, with far more rules and a lot of experimenting, so the comparison is a strong clue, not proof.
- **Isolation:** on bare metal, I disabled Suricata and changed nothing else. The freezes stopped.
- **Current theory:** a FreeBSD kernel-level problem with inline mode (netmap) on this hardware, not the card and not my configuration.
- **Lesson:** I changed two things at once (bare metal *and* inline IPS), which made my first diagnosis wrong. Now I change one thing at a time.
- **Next steps:** kernel crash dumps, console output during test windows, and a hardware watchdog. Logs already go to a lab VM over syslog, so the lead-up to a freeze survives the crash.

### 5. DNS that can't be read, forged or skipped

- **DNSSEC validation** in Unbound, hardened with hidden identity and version, aggressive NSEC, prefetching and serve-expired.
- **DNS-over-TLS only,** to Quad9 (`9.9.9.9`, with `149.112.112.112` as backup). The certificate is verified against `dns.quad9.net`, and there's no plain-text upstream.
- **An ad and tracker blocklist,** refreshed on a schedule.
- **Forced local DNS:** plain DNS to *any* server is redirected to the firewall by a NAT rule, and outbound DNS-over-TLS is blocked. Devices with hardcoded resolvers still get filtered answers.
- **Internal names** (`*.internal`) through host overrides.

**Proof it works:** I asked Google's resolver for a name that only my firewall knows:

![nslookup omar-little.internal 8.8.8.8 returns 192.168.0.2](images/nslookup-screenshot.png)
*Google can't know `*.internal`, so getting an answer proves the firewall intercepted the query.*

### 6. Right-sized, not oversized

I use enterprise patterns on hardware that fits an apartment. A low-power i5-10500T is plenty for routing and filtering, and FreeRADIUS will run on a Raspberry Pi instead of a server rack.

---

## Known limitations

Here's the honest list, because a design is only as good as its weakest part:

- **The network is flat.** Groups are IP ranges, so a device that self-assigns an ADMIN address gets admin access. VLANs and 802.1X fix this.
- **DNS-over-HTTPS can bypass forced DNS,** because it shares port 443 with normal web traffic. (pointed out by the LLM on a review, still dont understand it 100%)
- **The anti-lockout rule** leaves the GUI and SSH ports reachable from the whole LAN. Login is still admin-only.
- **There's no 2FA on the GUI yet,** and config backups are manual.
- **There's no alerting.** Logs are collected, but nothing notifies me.

---

## Roadmap

1. **A VLAN-capable switch:** TP-Link Omada, with 8 × 2.5 GbE ports, 2 × SFP+ and a dedicated management port.
2. **VLANs, one group at a time,** then disable the anti-lockout rule.
3. **802.1X (EAP-TLS) with FreeRADIUS** on a Raspberry Pi, so devices land in their VLAN automatically, plus MAB for devices that can't do 802.1X.
4. **WireGuard** for remote access, on its own subnet with its own rules.
5. **Automated off-box backups and alerting.**
6. **Finish the freeze investigation,** and move threat lists to firewall aliases.
7. **A 5G backup line** (multi-WAN), with a rule that blocks heavy traffic to save mobile data.

---

## What I learned, and what I'm still asking

Building this with an LLM at my side made it feel like nothing was out of reach. It also showed me how big the gap is between "it works" and how a professional would run it. That gap is exactly what I want to close.

**Questions I'm still chasing:**
- How do professionals tune IDS/IPS rule sets without drowning in false positives, or in latency?
- When is a virtualized firewall the *right* choice in production, and not just a workaround?

If you have answers, I'd love to hear them.

---

## Reference

<details>
<summary><b>Hardware</b></summary>

Dell OptiPlex 5090 Micro: i5-10500T, 16 GB RAM, 256 GB SSD on ZFS.

The OptiPlex has only one network port. I added a second by putting an Intel I226-V 2.5 GbE card in the M.2 Wi-Fi slot (Key E carries a PCIe lane), cabled to an RJ45 port mounted in the case.

![Front](images/optiplex-front.png)
![Inside: I226-V, SSD, M.2 A+E slot](images/optiplex-inside.png)

</details>

<details>
<summary><b>Device groups and planned VLANs</b></summary>

| Group      | VLAN | Range     | Role |
| ---------- | ---- | --------- | ---- |
| INFRA      | 100  | .1        | The firewall |
| HYPERVISOR | 200  | .2–.10    | Proxmox hosts |
| ADMIN      | 300  | .11–.20   | Admin devices |
| HOMELAB    | 400  | .21–.40   | Lab VMs and services |
| LAN        | 500  | .41–.60   | Everyday computers |
| MOBILE     | 600  | .61–.80   | Phones and tablets |
| IOT        | 700  | .81–.100  | Smart devices |
| REST       | 800  | .101–.200 | Anything unknown |
| NETWORK    | –    | .201–.250 | Switches and access points (no internet) |

</details>

<details>
<summary><b>Management, logging and hardening</b></summary>

- **Web GUI:** HTTPS, LAN only, 15-minute timeout.
- **SSH:** key-only, on a non-default port, with named personal accounts. Root login is disabled.
- **Logging:** remote syslog to a lab VM (rsyslog, daily rotation, kept for 14 days).
- **WAN:** no allow rules, so nothing is reachable from outside. Private and bogon networks are blocked.
- **Traffic shaping:** fq_codel on WAN against bufferbloat.
- **Snapshots:** a ZFS snapshot before every larger change, so a failed update rolls back in minutes.

![ZFS snapshots](images/opnsense-snapshots.png)

</details>
