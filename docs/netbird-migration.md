# Replacing Tailscale with self-hosted NetBird (internal CA)

**Goal:** one identity system for the whole lab. Today, Tailscale signs in with a personal Google account while the lab's own identity provider (Keycloak) sits unused by the network layer. After this migration:

> **Keycloak** (identity, MFA) → **NetBird** (network access by identity and device posture) → **OPNsense** (segmentation) → **Wazuh** (detection)

Everything is self-hosted and fully offline. The lab runs its own certificate authority (CA), and every layer checks identity.

Sources, read on 2026-11-11:
- docs.netbird.io: the self-hosted quickstart, the external reverse proxy page, the self-hosted guide, and the Keycloak setup page;
- `getting-started.sh` from the NetBird repo.

The docs **do not cover private CAs**. Steps marked *(untested)* are based on how Go programs normally find trusted certificates, so check them as you go.

---

## What NetBird needs (from the docs)

| Requirement | Detail |
|---|---|
| Server | Linux with **≥1 CPU, 2 GB RAM**, Docker and the compose plugin, `jq`, `curl` |
| Install | `getting-started.sh` generates `docker-compose.yml` and `config.yaml` for one combined `netbird-server` container (management, signal, relay and STUN) plus the dashboard |
| Reverse proxy | Script option **[2] Nginx** writes `nginx-netbird.conf` with `ssl_certificate` / `ssl_certificate_key` placeholders. This is the documented way to use your own certificate files; the built-in Traefik (option 0) only does Let's Encrypt. |
| Ports **to** the server | **TCP 443** (dashboard, API, gRPC, relay WebSocket) and **UDP 3478** (STUN). STUN can't go through the proxy; Docker publishes it directly. |
| Ports on peers | **None inbound.** Peers dial out. |
| Domain | The docs ask for a "public domain". The script accepts `netbird.lab.internal` (it only checks the format) and just warns if the name doesn't resolve publicly. |
| Identity | Embedded IdP (Dex) as a break-glass login, plus **Keycloak added** under Dashboard → Settings → Identity Providers |
| Keycloak | **Must be served over HTTPS.** Docs: "Otherwise, the setup won't work." |

---

## Design

### Names
`.internal` was reserved by ICANN in 2024 for private networks, so it can never clash with a real domain.

| Name | Resolves to (OPNsense Unbound override) | Serves |
|---|---|---|
| `netbird.lab.internal` | 10.20.20.30 | NetBird dashboard, API and gRPC |
| `keycloak.lab.internal` | 10.20.20.30 | Keycloak over HTTPS, proxied to 10.20.20.10:8080 |

### New server
| Item | Value |
|---|---|
| Type | Proxmox LXC from the existing **Debian 13** template |
| ID / name | 106 / `netbird` |
| Network | vmbr20 (Servers), **10.20.20.30/24**, gateway and DNS 10.20.20.1 |
| Resources | 2 cores, **2 GB RAM**, 512 MB swap, 16 GB disk on `local-lvm` |
| Options | Unprivileged; features `nesting=1,keyctl=1` (needed for Docker in LXC); start at boot |

**Nginx runs on the LXC itself.** It holds the one certificate and terminates TLS for both names:
- `netbird.lab.internal` goes to the NetBird containers on 127.0.0.1;
- `keycloak.lab.internal` goes to Keycloak on linuxserver.

Keycloak gets HTTPS without any certificate config of its own.

### The CA (OPNsense → System → Trust)
| Item | Value |
|---|---|
| Root CA | `ZTNA-LAB Root CA`, RSA 4096, SHA-256, 10 years. **The private key never leaves OPNsense.** |
| Server certificate | `netbird-proxy`, type *Server*, RSA 2048, SHA-256, **397 days**. SANs: `netbird.lab.internal`, `keycloak.lab.internal` |

### Where the CA must be trusted
This is the fiddly part of an internal CA. Anything that makes an HTTPS connection to either name needs the root CA.

| Place | Why it connects over HTTPS | How |
|---|---|---|
| LXC 106 (OS) | Testing with `curl`; source of the bundle for the containers | `/usr/local/share/ca-certificates/` + `update-ca-certificates` |
| `netbird-server` container | The embedded Dex fetches Keycloak's discovery, keys and tokens | Mount the host's bundle read-only and set `SSL_CERT_FILE` *(untested)* |
| linuxserver (OS) | The NetBird client talks to the management service | `update-ca-certificates` |
| mgmt Kali (OS **and** Firefox) | The NetBird client, plus the browser logins to NetBird and Keycloak | `update-ca-certificates`; Firefox → Settings → Certificates → Import |
| Windows PC (later) | Same as mgmt Kali | Import into *Trusted Root Certification Authorities* |
| **kaliattack: deliberately not** | — | It sees a certificate warning, as an untrusted device should |

**Security note for the writeup.** Any device that trusts this CA will accept any certificate it signs, so the CA's private key is a high-value secret. Mitigations:
- the key stays on the firewall;
- the CA is installed only on lab devices;
- server certificates are short-lived (397 days).

### Firewall (OPNsense)
New alias `NETBIRD` = 10.20.20.30, plus port aliases `NETBIRD_TCP` = 443 and `NETBIRD_UDP` = 3478.

| Tab | New rules (above that tab's block rules) | Why |
|---|---|---|
| Management | Pass TCP → NETBIRD 443; pass UDP → NETBIRD 3478 | The admin peer can enrol and connect, and reach Keycloak through the proxy |
| Clients | The same two rules. **Replace** the Keycloak 8080 rule with these. | User devices reach Keycloak only through the proxy |
| ATTACK_NET | Pass TCP → NETBIRD 443 *(logged)*. Remove any Keycloak 8080 rule. | An untrusted device can reach the **login page only**, as it would on the internet |
| Servers | none | linuxserver to NetBird is same-subnet traffic |

Peers in different segments will connect through NetBird's **relay** (WebSocket over 443), because OPNsense blocks direct WireGuard UDP between segments. That's fine for a lab. Direct peer-to-peer would need an extra UDP 51820 rule.

### NetBird groups and policies (default deny)
| Group | Members |
|---|---|
| `admins` | mgmt Kali (and later the Windows PC). Synced from the Keycloak group `admins` via the JWT `groups` claim. |
| `servers` | linuxserver (routing peer for 10.20.20.0/24) |
| `users` | client VM (later), synced from the Keycloak group `users` |

| Policy | Allows |
|---|---|
| admins → servers | TCP 22 |
| admins → network `servers-net` (10.20.20.0/24) | TCP 8080 (Keycloak direct), 443 (Wazuh) |
| *(delete NetBird's default "All → All" policy)* | — |

Posture checks for `admins`: a minimum NetBird client version and OS version.

---

## Phases

Tailscale stays as it is (offline, keys expired) until Phase 5, so nothing is lost if the migration takes several sessions.

### Phase 0: Prerequisites
1. **Tailscale:** remove **kaliattack** (admin console → Machines → kaliattack → ⋯ → Remove).
2. **Keycloak hardening.**
   - In the **master** realm: create a permanent admin user and delete the temporary one.
   - In the **ZTNA-LAB** realm:
     - turn on brute-force detection (Realm settings → Security defenses);
     - add a password policy (Authentication → Policies), e.g. length 12, not username, password history;
     - make *Configure OTP* a default required action;
     - turn on user and admin events (Realm settings → Events).
3. **Keycloak backup** (on linuxserver). The H2 database lives inside the container, so take this before the container is ever recreated:
   ```bash
   sudo docker inspect keycloak > ~/keycloak-inspect-2026-11.json
   sudo docker commit keycloak keycloak-backup:2026-11
   sudo docker images keycloak-backup
   ```
4. **Proxmox:**
   - back up VM 101 (and VMs 104 and 105, which have never been backed up);
   - set the backup job's retention to Keep Last 1–2.

### Phase 1: CA, DNS and firewall (all in OPNsense; nothing else depends on these yet)
5. **Create the root CA:** System → Trust → Authorities → **+**.
   - Method *Create an internal Certificate Authority*
   - Name `ZTNA-LAB Root CA`
   - RSA 4096, SHA256, 3650 days
   - Common name `ZTNA-LAB Root CA`
   - Fill in country, etc. as you like
6. **Create the server certificate:** System → Trust → Certificates → **+**.
   - Method *Create an internal Certificate*
   - Name `netbird-proxy`
   - Issuer `ZTNA-LAB Root CA`
   - Type *Server Certificate*
   - RSA 2048, SHA256, 397 days
   - Common name `netbird.lab.internal`
   - Alternative names: DNS `netbird.lab.internal` and DNS `keycloak.lab.internal`
7. **Download** to mgmt Kali (Firefox in VM 103):
   - **the CA certificate only** (never its key) → `ztna-lab-root-ca.crt`;
   - the `netbird-proxy` certificate;
   - the `netbird-proxy` key.

   The key is a secret. Delete it from `~/Downloads` once it's on the LXC (Phase 2).
8. **DNS:** Services → Unbound DNS → Overrides → Hosts.
   - Add host `netbird`, domain `lab.internal`, type A, 10.20.20.30.
   - Add host `keycloak`, same settings.
   - Apply.
   - Check from mgmt Kali: `host netbird.lab.internal` and `host keycloak.lab.internal` should both return 10.20.20.30.
9. **Firewall:** add the aliases and rules from the table above, then Apply. Keep the old Keycloak 8080 rules until Phase 2 step 15 is verified, then remove them from Clients and ATTACK_NET.

### Phase 2: NetBird server and Keycloak behind HTTPS
10. **Create LXC 106** as in the design table.
    - Inside it, run `apt update && apt full-upgrade -y && apt install -y curl jq nginx ca-certificates sudo openssh-server`.
    - Create a sudo user for SSH (`adduser <name>`, then `usermod -aG sudo <name>`).
    - Install Docker: `curl -fsSL https://get.docker.com -o get-docker.sh`, read it, then `sh get-docker.sh`.
11. **Copy the certificates in**, from mgmt Kali, then delete the key from Kali:
    ```bash
    scp ~/Downloads/ztna-lab-root-ca.crt ~/Downloads/netbird-proxy.crt ~/Downloads/netbird-proxy.key <name>@10.20.20.30:~
    ```
    Then on the LXC:
    ```bash
    sudo cp ~/ztna-lab-root-ca.crt /usr/local/share/ca-certificates/ && sudo update-ca-certificates
    sudo mkdir -p /etc/nginx/tls && cat ~/netbird-proxy.crt ~/ztna-lab-root-ca.crt | sudo tee /etc/nginx/tls/fullchain.pem >/dev/null
    sudo install -m 600 -o root ~/netbird-proxy.key /etc/nginx/tls/privkey.pem && rm ~/netbird-proxy.key
    ```
12. **Install NetBird:**
    - download `https://github.com/netbirdio/netbird/releases/latest/download/getting-started.sh`, read it, and run it with `bash`;
    - domain **`netbird.lab.internal`**;
    - reverse proxy **[2] Nginx**;
    - if asked for trusted proxy IPs, use `127.0.0.1`.
13. **Configure Nginx for NetBird:** install the generated `nginx-netbird.conf` as `/etc/nginx/sites-available/netbird`.
    - Point it at `/etc/nginx/tls/fullchain.pem` and `/etc/nginx/tls/privkey.pem`.
    - Keep `client_header_timeout 1d; client_body_timeout 1d;`.
    - Enable it and remove the `default` site.
    - Run `sudo nginx -t && sudo systemctl reload nginx`.
14. **Make NetBird trust the CA** *(untested)*: in `docker-compose.yml`, under the `netbird-server` service:
    ```yaml
        volumes:
          - /etc/ssl/certs/ca-certificates.crt:/etc/ssl/certs/ca-certificates.crt:ro
        environment:
          - SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt
    ```
    Add these to the existing lists, then run `sudo docker compose up -d`.
15. **Put Keycloak behind the proxy.**
    - Add an Nginx server block for `keycloak.lab.internal`: listen 443 ssl, the same certificate, and `proxy_pass http://10.20.20.10:8080` with `X-Forwarded-For`, `-Proto`, `-Host` and `-Port` headers.
    - Then recreate Keycloak on linuxserver **from the backup image, with its data moved onto a Docker volume** (a new, empty named volume is filled from the image's copy):
    ```bash
    sudo docker stop keycloak && sudo docker rename keycloak keycloak-old
    sudo docker run -d --name keycloak --restart unless-stopped -p 8080:8080 \
      -v keycloak-data:/opt/keycloak/data \
      keycloak-backup:2026-11 start-dev \
      --proxy-headers xforwarded --hostname https://keycloak.lab.internal
    ```
    - Check that `https://keycloak.lab.internal/admin` loads from mgmt Kali with a padlock and that the ZTNA-LAB users are still there.
    - Only then, weeks later, remove `keycloak-old`.
    - **Bonus:** the data now survives the container being recreated.
16. Open `https://netbird.lab.internal/setup` and create the **local break-glass admin**. Store its password in a password manager.

### Phase 3: Keycloak as NetBird's identity provider
17. In Keycloak (ZTNA-LAB):
    - create the groups `admins` (secadmin) and `users` (alice, bob); leave redteam out of both;
    - create a client scope `groups` with a *Group Membership* mapper, claim name `groups`, *Full group path* off, added to the ID token, access token and userinfo.
18. In NetBird: Settings → Identity Providers → Add → Keycloak.
    - It shows the **redirect URI**. Create the confidential client `netbird` in Keycloak with that URI and the `groups` scope.
    - Paste the client ID and secret back into NetBird.
    - Issuer: `https://keycloak.lab.internal/realms/ZTNA-LAB`.
19. Turn on JWT group sync (claim `groups`).
20. Test: log out, then sign in with **Keycloak + OTP** as secadmin.

### Phase 4: Enrol peers and set policy
21. **Trust the CA** on linuxserver and mgmt Kali: copy `ztna-lab-root-ca.crt` to `/usr/local/share/ca-certificates/`, then run `sudo update-ca-certificates`. On Kali, also import it into Firefox.
    - Doing linuxserver over SSH from mgmt Kali lets you paste commands instead of typing them in the console.
22. **Install the client:** download `https://pkgs.netbird.io/install.sh`, read it, and run it.
23. **linuxserver:**
    - create a one-off **setup key** in the dashboard with auto-group `servers`;
    - run `sudo netbird up --management-url https://netbird.lab.internal --setup-key <key>`;
    - Dashboard → Networks → add `servers-net` with resource 10.20.20.0/24 and routing peer linuxserver.
24. **mgmt Kali:** run `sudo netbird up --management-url https://netbird.lab.internal`, then log in through the browser as secadmin.
25. Create the policies above, **delete All → All**, and add posture checks.

### Phase 5: Retire Tailscale
26. On linuxserver and mgmt Kali:
    - `sudo tailscale down`;
    - `sudo apt purge tailscale`;
    - remove `/etc/apt/sources.list.d/tailscale*`.
27. **Proxmox host:** remove the Tailscale apt repo. It's enabled even though the host isn't in the tailnet.
28. Delete the machines from the Tailscale admin console, and optionally delete the tailnet.

### Phase 6: Prove it (evidence for the writeup)
| Test | Expected |
|---|---|
| kaliattack → `https://netbird.lab.internal` | **Certificate warning** (untrusted device). The login page can't be used without Keycloak credentials and OTP. |
| kaliattack → 10.20.20.0/24 other than :443 on .30 | Blocked by OPNsense (unchanged) |
| mgmt Kali (NetBird on) → Keycloak, Wazuh through `servers-net` | Allowed |
| mgmt Kali → linuxserver TCP 80 through NetBird | Denied by NetBird policy |
| Disable secadmin in Keycloak, or remove them from `admins` | The peer loses access. Screenshot the NetBird peer/group change. This shows **revocation by identity**. |
| Failed Keycloak logins | Visible in Keycloak events, and later in Wazuh |
| `openssl s_client -connect netbird.lab.internal:443 -servername netbird.lab.internal` | The chain ends at `ZTNA-LAB Root CA` and verifies OK |

### Phase 7 (optional, the strongest Zero Trust result): NetBird as the only way in
29. Once NetBird works, remove the Management tab's direct allows to Keycloak, Wazuh and SSH. Keep **Proxmox GUI** and **the firewall** allowed so you can't lock yourself out.
    - Management can then reach servers **only** through NetBird, with identity, MFA and policy, instead of being trusted for its network location.
    - Rerun the tests.

---

## Risks and notes
- **Keycloak data:** never `docker rm` the old container until the new one is verified (step 15).
- **CA trust in containers** is the most likely thing to break.
  - Symptom: the Keycloak IdP fails to connect, with x509 "unknown authority" in `docker compose logs netbird-server`.
  - Fix: step 14.
- **Kali on NetBird:** Kali isn't officially listed. `install.sh` (Debian path) should work; it's untested.
- **Certificate renewal:** the server certificate expires after 397 days. Diary a reminder; renewal is reissue in OPNsense, then repeat step 11.
- **Keycloak still runs `start-dev`.** Moving to production mode with a PostgreSQL database is a later improvement.
