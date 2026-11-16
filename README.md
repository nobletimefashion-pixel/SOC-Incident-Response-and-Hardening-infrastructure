# Linux Server & Host Security Hardening

This repository documents the step-by-step security hardening and verification performed on a Linux host (`systemd`-based environment). The objective of this task is to minimize the network attack surface, enforce strict firewall policies, secure remote SSH access, configure automated intrusion prevention, and verify the posture using a full-port network scan.

---

## Table of Contents
1. [Attack Surface Audit (`ss`)](#1-attack-surface-audit-ss)
2. [Firewall Configuration (`ufw`)](#2-firewall-configuration-ufw)
3. [SSH Hardening (`/etc/ssh/sshd_config`)](#3-ssh-hardening-etcsshsshd_config)
4. [Intrusion Prevention (`fail2ban`)](#4-intrusion-prevention-fail2ban)
5. [Hardening Verification (`nmap`)](#5-hardening-verification-nmap)
6. [Key Takeaways](#6-key-takeaways)

---

## 1. Attack Surface Audit (`ss`)

Before applying security controls, active listening sockets and their owning processes were audited using `ss` (Socket Statistics).

### Command
```bash
sudo ss -tulpn
```
* `-t` / `-u`: Display TCP and UDP sockets.
* `-l`: Show only listening sockets.
* `-p`: Show the process name and PID using each socket.
* `-n`: Display numeric port numbers instead of resolving service names.

### Audit Findings
* **Local-only services (`127.0.0.x`):** Services such as `systemd-resolve` (port `53`), `tor` (`9050`), and `containerd` (`42993`) were bound strictly to loopback interfaces, making them unreachable from external networks.
* **Network-exposed interfaces (`0.0.0.0` / `[::]`):** Only local name resolution protocols (`5353` mDNS and `5355` LLMNR) were listening across all interfaces.
* **SSH Daemon:** `sshd.service` was initially inactive and subsequently enabled and started for hardened remote access:
  ```bash
  sudo systemctl enable --now sshd.service
  ```

---

## 2. Firewall Configuration (`ufw`)

Uncomplicated Firewall (`ufw`) was configured using a **Default Deny** security model—blocking all unsolicited inbound connections while permitting outbound traffic and explicitly allowed services.

### Commands
```bash
# Allow SSH before enforcing default deny to prevent lockout
sudo ufw allow 22/tcp

# Set baseline policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Enable firewall at boot and immediately
sudo ufw enable

# Verify active rules
sudo ufw status verbose
```

### Verified Status
* **Status:** Active (`enabled on system startup`)
* **Default Policy:** `deny (incoming)`, `allow (outgoing)`, `deny (routed)`
* **Inbound Rules:** `22/tcp` (`ALLOW IN` from `Anywhere`)

---

## 3. SSH Hardening (`/etc/ssh/sshd_config`)

To eliminate brute-force password attacks against port `22`, SSH authentication was restricted exclusively to Ed25519 cryptographic key pairs.

### Key Setup & Verification
1. Confirmed existing Ed25519 key pair (`~/.ssh/id_ed25519` and `~/.ssh/id_ed25519.pub`).
2. Installed the public key into `~/.ssh/authorized_keys` and verified key-based login:
   ```bash
   ssh-copy-id $USER@localhost
   ssh $USER@localhost
   ```

### Configuration Changes (`/etc/ssh/sshd_config`)
Updated `/etc/ssh/sshd_config` with the following directives:
```text
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
```

### Validation & Reload
Tested configuration syntax before reloading the daemon:
```bash
sudo sshd -t
sudo systemctl reload sshd.service
```

### Proof of Enforcement
Attempted login with public-key authentication explicitly disabled:
```bash
ssh -o PubkeyAuthentication=no $USER@localhost
# Output: darklord@localhost: Permission denied (publickey).
```

---

## 4. Intrusion Prevention (`fail2ban`)

Fail2ban was installed and configured to monitor SSH authentication events and dynamically ban offending IP addresses via UFW.

### Log Backend Discovery
On this `systemd`-based distribution, `/var/log/auth.log` is not used; instead, authentication events are written directly to the `systemd` journal (`journald`). Therefore, Fail2ban was configured with `backend = systemd`.

### Local Jail Configuration (`/etc/fail2ban/jail.local`)
Created `/etc/fail2ban/jail.local` (preserving `/etc/fail2ban/jail.conf` from package update overwrites):

```ini
[DEFAULT]
# Ban IP for 1 hour after 4 failed attempts within 10 minutes
bantime = 1h
findtime = 10m
maxretry = 4
banaction = ufw

[sshd]
enabled = true
port = ssh
backend = systemd
```

### Service Activation & Verification
```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

**Verified Jail Output:**
```text
Status for the jail: sshd
Status for the jail: sshd
|- Filter
|  |- Currently failed:	0
|  |- Total failed:	0
|  `- Journal matches:	_SYSTEMD_UNIT=sshd.service + _COMM=sshd + _COMM=sshd-session
`- Actions
   |- Currently banned:	0
   |- Total banned:	0
   `- Banned IP list:
```

---

## 5. Hardening Verification (`nmap`)

Finally, a full 65,535-port TCP SYN scan (`-sS`) was executed against the host's network interface IP (`192.168.100.89`) to validate firewall filtering from a network perspective.

### Command
```bash
sudo nmap -sS -p- 192.168.100.89
```

### Scan Results
```text
Nmap scan report for 192.168.100.89
Host is up (0.00013s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT     STATE SERVICE
22/tcp   open  ssh
5355/tcp open  llmnr

Nmap done: 1 IP address (1 host up) scanned in 105.61 seconds
```

# results

 sudo journalctl -u sshd -n 8 --no-pager
Nov 16 10:08:21 darklord-hpmt41 systemd[1]: Starting OpenSSH Daemon...
Nov 16 10:08:23 darklord-hpmt41 sshd[587]: Server listening on 0.0.0.0 port 22.
Nov 16 10:08:23 darklord-hpmt41 sshd[587]: Server listening on :: port 22.
Nov 16 10:08:23 darklord-hpmt41 systemd[1]: Started OpenSSH Daemon.
Nov 16 11:13:39 darklord-hpmt41 sshd-session[19231]: Connection closed by 192.168.70.70 port 42989 [preauth]
Nov 16 11:13:39 darklord-hpmt41 sshd-session[19236]: Connection closed by 192.168.70.70 port 41413 [preauth]
Nov 16 11:13:40 darklord-hpmt41 sshd-session[19239]: Connection closed by 192.168.70.70 port 40101 [preauth]
Nov 16 11:13:40 darklord-hpmt41 sshd-session[19242]: Connection closed by 192.168.70.70 port 54721 [preauth]

# bash
  sudo fail2ban-client status sshd
  sudo journalctl -u fail2ban -n 10 --no-pager
  sudo ufw status numbered

Status for the jail: sshd
|- Filter
|  |- Currently failed:	0
|  |- Total failed:	0
|  `- Journal matches:	_SYSTEMD_UNIT=sshd.service + _COMM=sshd + _COMM=sshd-session
`- Actions
   |- Currently banned:	0
   |- Total banned:	1
   `- Banned IP list:	
Nov 16 10:08:21 darklord-hpmt41 systemd[1]: Started Fail2Ban Service.
Nov 16 10:08:52 darklord-hpmt41 fail2ban-server[585]: Server ready
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] Anywhere                   REJECT IN   192.170.70.70              
[ 2] 22                         ALLOW IN    Anywhere                  
[ 3] 51820/udp                  ALLOW OUT   Anywhere                   (out)
[ 4] 443/tcp                    ALLOW OUT   Anywhere                   (out)
[ 5] 22 (v6)                    ALLOW IN    Anywhere (v6)             
[ 6] 51820/udp (v6)             ALLOW OUT   Anywhere (v6)              (out)
[ 7] 443/tcp (v6)               ALLOW OUT   Anywhere (v6)              (out)




### Result Analysis
* **`65533 filtered tcp ports`:** Confirms UFW is silently dropping unsolicited incoming packets across the port range.
* **`22/tcp open ssh`:** Intentionally open for key-only SSH access, protected by Fail2ban.
* **`5355/tcp open llmnr`:** Local Link-Local Multicast Name Resolution socket (`systemd-resolve`), visible only because the scan originated from the local host itself.

---

## 6. Key Takeaways

* **Defense in Depth:** Combining network-level filtering (`ufw`), strong cryptographic authentication (`sshd`), and automated log-based banning (`fail2ban`) ensures no single failure compromises the host.
* **Safe Configuration Practices:** Always verify SSH key logins and run `sshd -t` before disabling password authentication to prevent remote lockouts, and always store Fail2ban overrides in `jail.local`.
* **System-Specific Auditing:** Checking actual log paths (`/var/log/auth.log` vs. `journald`) and service unit names (`sshd.service` vs. `ssh.service`) prevents silent configuration failures.



