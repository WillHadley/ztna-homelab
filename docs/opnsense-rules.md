# OPNsense ruleset v2: least-privilege segmentation

This replaces the ruleset documented in the April 2026 writeup. A review on 2026-11-10 found these problems:

- Pass-any rules shadowed the block rules on the Servers and Management tabs.
- ATTACK_NET could reach the home LAN and the Proxmox GUI through `WAN net`.
- Clients could reach ATTACK_NET on ports 80–443.
- Almost no rules logged.

## Lab addressing (as found)

| Segment | Bridge | Subnet | Hosts |
|---|---|---|---|
| WAN (home LAN) | vmbr0 | 192.168.1.0/24 | OPNsense 192.168.1.19, Proxmox 192.168.1.168, router 192.168.1.1 |
| Management | vmbr10 | 10.10.10.0/24 | OPNsense .1, mgmt Kali .50 |
| Servers | vmbr20 | 10.20.20.0/24 | OPNsense .1, Keycloak .10 (:8080), Wazuh .20 |
| Clients | vmbr30 | 10.10.30.0/24 | OPNsense .1, **no VMs** |
| ATTACK_NET | vmbr40 | 10.40.40.0/24 | OPNsense .1, kaliattack .50 |

## Design principles

1. **Default deny, logged.** Each tab ends with an explicit deny-all that has logging on, so every blocked attempt is evidence.
2. **Allow to a named service, not to a network.** Pass rules name a host alias and a port alias. "Servers net, any port" is never allowed.
3. **Blocks before broad passes.** The only broad pass rule is "internet", written as *not RFC1918* (`!PRIVATE_NETS`) so it can't reach other segments, the home LAN or the hypervisor.
4. **Rules go on the tab where traffic enters.** A rule on the Management tab only sees traffic *from* Management. Each segment is protected by the block rules on the *other* tabs.
5. **Every rule has a description.** The firewall log shows it, so a blocked packet explains itself.

---

## Step 0: Prerequisites

1. **Back up the config:** System → Configuration → Backups → **Download configuration**. Keep the file. If something breaks, restore from it.
2. **Remove the hypervisor's IPs from the lab bridges.**
   - In Proxmox, go to pve → System → Network and edit **vmbr10, vmbr20 and vmbr30**. Clear *IPv4/CIDR* (192.168.10.1, .20.1, .30.1), then click **Apply Configuration**.
   - Proxmox management stays on vmbr0, so this doesn't lock you out.
   - Why: while these IPs exist, the host has a Layer 2 presence in every segment that OPNsense can't filter.
3. **Optional: renumber Clients to 10.30.30.1/24** to match the writeup and the 10.X0.X0 scheme. Do it under Interfaces → [Clients], and update any DHCP range on that interface. The segment is empty, so nothing breaks.
4. **Check server DNS.** On Keycloak and Wazuh, run `resolvectl dns`. The new Servers rules allow DNS **only to the firewall** (10.20.20.1).
   - If a server points at 8.8.8.8 or similar, change its netplan to use `10.20.20.1` before applying.
   - Otherwise `apt` and `docker pull` will fail.

## Step 1: Aliases

Create these under Firewall → Aliases. Keep the existing `PROXMOX`, `PROXMOX_GUI`, `KEYCLOAK_SERVER`, `KEYCLOAK_HTTP` and `WAZUH`, and check their values.

**How to create an alias:**

1. Go to Firewall → Aliases and click the orange **+** at the bottom right of the table.
2. Fill in the form:
   - Leave **Enabled** ticked.
   - **Name:** e.g. `LAB_NETS`
   - **Type:** e.g. *Network(s)*
   - **Content:** type one entry and press **Enter**. It turns into a tag. Repeat for each entry.
   - **Description**
3. Click **Save**. When all the aliases are done, click **Apply** under the table. An alias does nothing until a rule uses it, so creating these is safe.

**Clients subnet:** in `LAB_NETS`, use `10.30.30.0/24` if you renumbered Clients in step 0.3, or `10.10.30.0/24` if you didn't. To check, Interfaces → [Clients] shows the current address.

| Name | Type | Content | Description |
|---|---|---|---|
| `PRIVATE_NETS` | Network(s) | `10.0.0.0/8` `172.16.0.0/12` `192.168.0.0/16` | RFC1918: every lab segment, home LAN and hypervisor |
| `LAB_NETS` | Network(s) | `10.10.10.0/24` `10.20.20.0/24` `10.30.30.0/24` *(or `10.10.30.0/24`; see above)* `10.40.40.0/24` | All lab segments |
| `WEB_PORTS` | Port(s) | `80` `443` | HTTP/HTTPS only. Replaces the 80–443 *range*, which opened 364 ports. |
| `WAZUH_AGENT` | Port(s) | `1514` `1515` | Agent events and enrolment |
| `WAZUH_DASH` | Port(s) | `443` | Wazuh dashboard (change it if yours differs) |
| `KEYCLOAK_SERVER` | Host(s) | `10.20.20.10` | *(exists; check the value)* |
| `WAZUH` | Host(s) | `10.20.20.20` | *(exists; check the value)* |
| `PROXMOX` | Host(s) | `192.168.1.168` | *(exists)* |

## Step 2: Rules per tab

How to read the tables:

- **All rules:** Direction **in**, quick (first match), IPv4 unless the rule says otherwise.
- **Log column:** ✅ means tick *Log packets that are handled by this rule*.
- **`!` prefix:** tick the *Invert* box next to the destination.
- **`This Firewall`:** a built-in destination meaning any of OPNsense's own IPs.

**How to rebuild:** a tab has eight rules at most, so it's quicker to **delete the custom rules on a tab and re-add them in order** than to re-order them. Do the tabs in this order: **Servers → Clients → ATTACK_NET → Management**. Management goes last because you're working from it. The automatic anti-lockout rule keeps the GUI reachable throughout. Click **Apply changes** after each tab.

### Servers

Servers expose services and reach out only for updates. They never start connections into other segments.

| # | Action | Proto | Source | Destination | Dest port | Log | Description |
|---|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | Servers net | This Firewall | 53 (DNS) | | SRV: DNS to firewall |
| 2 | Block | any | Servers net | Management net | any | ✅ | SRV: no access to admin net |
| 3 | Block | any | Servers net | Clients net | any | ✅ | SRV: no lateral movement to clients |
| 4 | Block | any | Servers net | ATTACK_NET net | any | ✅ | SRV: no reverse pivot |
| 5 | Block | any | Servers net | This Firewall | any | ✅ | SRV: no firewall admin |
| 6 | Block | any | Servers net | `PRIVATE_NETS` | any | ✅ | SRV: no home LAN / hypervisor |
| 7 | Pass | UDP | Servers net | `!PRIVATE_NETS` | 123 (NTP) | | SRV: time sync |
| 8 | Pass | TCP | Servers net | `!PRIVATE_NETS` | `WEB_PORTS` | | SRV: OS/container updates (internet only) |
| 9 | Block | any (IPv4+IPv6) | any | any | any | ✅ | SRV: default deny |

> **Delete** the old rule 1 (Servers net → any, pass). That single rule disabled all segmentation from this segment. Also delete the old "Allow updates" rule, which had its 80–443 range in the *source* port field.

### Clients

Clients are ordinary user devices. They get the identity provider plus the apps they're entitled to, and nothing else internal.

| # | Action | Proto | Source | Destination | Dest port | Log | Description |
|---|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | Clients net | This Firewall | 53 (DNS) | | CLI: DNS to firewall |
| 2 | Pass | TCP | Clients net | `KEYCLOAK_SERVER` | `KEYCLOAK_HTTP` | ✅ | CLI: sign-in (Keycloak) |
| 3 | Pass | TCP | Clients net | `WAZUH` | `WAZUH_AGENT` | | CLI: Wazuh agent telemetry |
| 4 | Block | any | Clients net | Management net | any | ✅ | CLI: no access to admin net |
| 5 | Block | any | Clients net | Servers net | any | ✅ | CLI: only entitled server apps |
| 6 | Block | any | Clients net | ATTACK_NET net | any | ✅ | CLI: no peer access |
| 7 | Block | any | Clients net | This Firewall | any | ✅ | CLI: no firewall admin |
| 8 | Block | any | Clients net | `PRIVATE_NETS` | any | ✅ | CLI: no home LAN / hypervisor |
| 9 | Pass | UDP | Clients net | `!PRIVATE_NETS` | 123 (NTP) | | CLI: time sync |
| 10 | Pass | TCP | Clients net | `!PRIVATE_NETS` | `WEB_PORTS` | | CLI: web browsing (internet only) |
| 11 | Block | any (IPv4+IPv6) | any | any | any | ✅ | CLI: default deny |

> The old rules allowed Clients → *all of* Servers net on 80/443. Nothing on Servers serves 80/443 to users; the Wazuh dashboard is on 443, and users shouldn't reach the SIEM. So this ruleset allows **specific apps** instead. When you add an app (e.g. an identity-aware proxy), add one pass rule for it above rule 4.

### ATTACK_NET

This segment models an **untrusted device on the network**. It can see the one published service (the Keycloak login) and the internet. A disabled, clearly labelled rule opens SSH for the brute-force scenario only while that test runs.

| # | Action | Proto | Source | Destination | Dest port | Log | Description |
|---|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | ATTACK_NET net | This Firewall | 53 (DNS) | | ATK: DNS to firewall |
| 2 | Pass | TCP | ATTACK_NET net | `KEYCLOAK_SERVER` | `KEYCLOAK_HTTP` | ✅ | ATK: published app (login page) |
| 3 | Pass, **disabled** | TCP | ATTACK_NET net | `KEYCLOAK_SERVER` | 22 | ✅ | ATK: TEST ONLY: SSH brute-force scenario |
| 4 | Block | any | ATTACK_NET net | Management net | any | ✅ | ATK: no access to admin net |
| 5 | Block | any | ATTACK_NET net | Clients net | any | ✅ | ATK: no lateral movement to clients |
| 6 | Block | any | ATTACK_NET net | Servers net | any | ✅ | ATK: no access to other server services |
| 7 | Block | any | ATTACK_NET net | This Firewall | any | ✅ | ATK: no firewall admin |
| 8 | Block | any | ATTACK_NET net | `PRIVATE_NETS` | any | ✅ | ATK: no home LAN / hypervisor |
| 9 | Pass | TCP | ATTACK_NET net | `!PRIVATE_NETS` | `WEB_PORTS` | | ATK: tool updates (internet only) |
| 10 | Block | any (IPv4+IPv6) | any | any | any | ✅ | ATK: default deny |

> The old "Allow outbound internet → WAN net" rule actually allowed the **home LAN**, which includes the Proxmox GUI at 192.168.1.168:8006. Rules 8–9 give real internet access without it.
>
> The old "Allow testing on servers" rule (any port to all of Servers net) is replaced by rules 2–3. Enable rule 3 only while the Hydra scenario runs, and record in your notes that you did. That's a realistic change-control habit to mention in interviews.
>
> If kaliattack still has a Wazuh agent, it will show as disconnected. That's intended: an untrusted device shouldn't reach the SIEM's ingest port.

### Management

Management is the only segment allowed to administer things. Each admin path is its own rule.

| # | Action | Proto | Source | Destination | Dest port | Log | Description |
|---|---|---|---|---|---|---|---|
| — | *(automatic)* | | | | | | *Anti-lockout rule: leave it* |
| 1 | Pass | any | Management net | This Firewall | any | | MGT: firewall admin, DNS, NTP |
| 2 | Pass | TCP | Management net | `PROXMOX` | `PROXMOX_GUI` | ✅ | MGT: Proxmox GUI |
| 3 | Pass | TCP | Management net | `KEYCLOAK_SERVER` | `KEYCLOAK_HTTP` | | MGT: Keycloak admin console |
| 4 | Pass | TCP | Management net | `WAZUH` | `WAZUH_DASH` | | MGT: Wazuh dashboard |
| 5 | Pass | TCP | Management net | `WAZUH` | `WAZUH_AGENT` | | MGT: Wazuh agent telemetry |
| 6 | Pass | TCP | Management net | `LAB_NETS` | 22 (SSH) | ✅ | MGT: SSH administration |
| 7 | Block | any | Management net | `PRIVATE_NETS` | any | ✅ | MGT: no other internal access |
| 8 | Pass | TCP | Management net | `!PRIVATE_NETS` | `WEB_PORTS` | | MGT: internet (updates, docs, Tailscale) |
| 9 | Block | any (IPv4+IPv6) | any | any | any | ✅ | MGT: default deny |

> **Delete** "Internet access" (any → any). It shadowed every rule below it.
>
> **Delete** "Protect Management" (any → Management net). On this tab it can only match traffic *from* Management *to* Management, which never crosses the firewall. Management is protected by the block rules on the Servers, Clients and ATTACK_NET tabs.
>
> **Delete** the two disabled "Default allow LAN" rules. They're noise.
>
> Tailscale works through rule 8: its DERP relays use 443. For faster direct connections, add a pass for UDP 41641 to `!PRIVATE_NETS`.

### WAN and Floating

Leave them empty. Nothing on WAN answered from the home LAN (443, 80, 22 and ping were all refused or dropped), which is correct.

---

## Step 3: Send firewall logs to Wazuh

This makes blocked traffic show up in the SIEM alongside the endpoint alerts, so one dashboard tells the whole story of an attack.

1. **OPNsense:** System → Settings → Logging → **Remote**. Add a destination:
   - Transport: UDP(4)
   - Applications: **filter (filterlog)**
   - Hostname: `10.20.20.20`
   - Port: `514`
2. **Wazuh manager:** in `/var/ossec/etc/ossec.conf`, add:
   ```xml
   <remote>
     <connection>syslog</connection>
     <port>514</port>
     <protocol>udp</protocol>
     <allowed-ips>10.20.20.1</allowed-ips>
   </remote>
   ```
   Then run `systemctl restart wazuh-manager`.
3. **Check:** Wazuh's built-in pfSense/OPNsense decoders should parse `filterlog` lines. Trigger a block (test A3 below) and look for it in Threat Hunting.

No extra firewall rule is needed: traffic from OPNsense itself isn't filtered by the interface rules.

---

## Step 4: Verification matrix

Run each test and record pass/fail with a screenshot of the command *and* the matching firewall log line, which shows the rule description. Together these form the new evidence set.

### From kaliattack (10.40.40.50)

| ID | Command | Expected | Log label |
|---|---|---|---|
| A1 | `nmap -Pn -p 22,80,443,8080 10.20.20.10` | 8080 open; others filtered | ATK: no access to other server services |
| A2 | `nmap -Pn -p 1514,1515,443 10.20.20.20` | all filtered | ATK: no access to other server services |
| A3 | `nmap -Pn -p 22,80,443 10.10.10.1 10.10.10.50` | all filtered | ATK: no access to admin net |
| A4 | `nmap -Pn -sn 10.10.30.0/24` | no hosts | ATK: no lateral movement to clients |
| A5 | `curl -k -m 5 https://192.168.1.168:8006` | timeout | ATK: no home LAN / hypervisor |
| A6 | `curl -m 5 http://192.168.1.1` | timeout | ATK: no home LAN / hypervisor |
| A7 | `curl -sI -m 5 https://www.kali.org` | HTTP 200/301 | *(pass, not logged)* |
| A8 | `nikto -h http://10.20.20.10:8080` | scan completes, "1 host(s) tested" | ATK: published app |
| A9 | Enable rule 3, then `hydra -l testuser -P /usr/share/wordlists/rockyou.txt -t 4 ssh://10.20.20.10`. Disable rule 3 afterwards. | Wazuh brute-force alert (rule 5763 / level 10) | ATK: TEST ONLY |

### From Keycloak server (10.20.20.10)

| ID | Command | Expected | Log label |
|---|---|---|---|
| S1 | `nc -zv -w 3 10.10.10.50 22` | fails | SRV: no access to admin net |
| S2 | `nc -zv -w 3 10.40.40.50 22` | fails | SRV: no reverse pivot |
| S3 | `curl -k -m 5 https://192.168.1.168:8006` | fails | SRV: no home LAN / hypervisor |
| S4 | `curl -k -m 5 https://10.20.20.1` | fails | SRV: no firewall admin |
| S5 | `sudo apt update` and `docker pull hello-world` | succeed | *(pass)* |

### From mgmt Kali (10.10.10.50)

| ID | Command | Expected |
|---|---|---|
| M1 | Open `https://192.168.1.168:8006`, `http://10.20.20.10:8080/admin`, `https://10.20.20.20` | all load |
| M2 | `ssh <user>@10.20.20.10` | connects |
| M3 | `curl -m 5 http://192.168.1.1` | fails (MGT: no other internal access) |
| M4 | `nc -zv -w 3 10.20.20.10 8443` | fails (MGT: no other internal access) |

### From a client VM (once one exists on Clients)

| ID | Command | Expected |
|---|---|---|
| C1 | Browse `http://10.20.20.10:8080/realms/ZTNA-LAB/account` | loads |
| C2 | `nmap -Pn -p 22,443,1514 10.20.20.10 10.20.20.20` | 22 and 443 filtered; 1514 open on .20 only |
| C3 | `nmap -Pn -p 80,443 10.40.40.50` | filtered. **This is the ATTACK_NET hole the old ruleset left open.** |

---

## Known limitations (to be addressed next)

- **Same-segment traffic isn't filtered.** Keycloak ↔ Wazuh traffic on vmbr20 never crosses OPNsense. Fix: Proxmox VM firewall rules (microsegmentation within a segment).
- **The Keycloak admin console is on the same port as the login page,** so anything that can sign in can also reach `/admin`. Fix: put a reverse proxy in front that blocks `/admin` from non-management sources, or use Keycloak's separate admin hostname.
- **Keycloak is on plain HTTP.** Credentials cross the network in cleartext. Fix: TLS on 8443, then switch the `KEYCLOAK_HTTP` alias.
- **Rules are static.** Next step: let Wazuh active response add offending IPs to an OPNsense alias through the API (automated containment).
