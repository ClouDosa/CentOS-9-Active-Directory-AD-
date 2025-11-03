# CentOS 9 ↔ Active Directory (AD) – What Worked For Me

**Goal:** Log in to a CentOS Stream 9 box with AD accounts, get Kerberos SSO for SSH, give sudo to an AD group, and (optionally) auto‑mount home folders.

> I'm a junior sysadmin and I kept notes while I got this working. This is not “the only right way,” just the steps that finally worked for me. If you see something off, please open an issue/PR 🙏

---

## My Test Setup
- **Linux:** CentOS Stream 9 (fully updated)
- **AD:** Windows Server 2019/2022 domain
- **Tools:** `realmd`, `sssd`, `adcli`, `oddjob-mkhomedir`, `krb5-workstation`, `authselect`
- **Time zone:** America/Chicago
- **Last checked:** 2025‑11‑02

> Tip: Kerberos is super picky about **DNS** and **time**. I had most problems when those weren’t right.

---

## TL;DR (the short version)

```bash
# Fill these in for your environment
export AD_DOMAIN="example.com"      # DNS domain, lower-case
export AD_REALM="EXAMPLE.COM"       # Kerberos realm, upper-case
export AD_JOIN_USER="adjoiner"      # AD user who can join computers

# 1) Packages
sudo dnf -y update
sudo dnf -y install realmd sssd sssd-tools adcli oddjob oddjob-mkhomedir   krb5-workstation authselect chrony samba-common-tools openldap-clients   sudo policycoreutils-python-utils

# 2) Make sure time is synced (Kerberos hates clock drift)
sudo timedatectl set-ntp true
sudo systemctl enable --now chronyd

# 3) Discover & join
realm discover "$AD_DOMAIN"
sudo realm join "$AD_DOMAIN" -U "$AD_JOIN_USER"

# 4) Use SSSD + auto-create home dirs on first login
sudo authselect select sssd with-mkhomedir --force
sudo systemctl enable --now sssd

# 5) Limit who can log in (optional but I like it)
sudo realm permit -g "Domain Users"
# or: sudo realm permit -g "linux-login"

# 6) Enable Kerberos SSO for SSH
sudo bash -c 'cat >/etc/ssh/sshd_config.d/10-gssapi.conf <<EOF
GSSAPIAuthentication yes
GSSAPICleanupCredentials yes
GSSAPIKeyExchange yes
GSSAPIStrictAcceptorCheck no
EOF'
sudo systemctl restart sshd

# 7) Give sudo to an AD group
sudo bash -c 'cat >/etc/sudoers.d/ad-sudoers <<EOF
%linux-sudoers ALL=(ALL) ALL
EOF'
sudo visudo -c   # sanity check

# 8) Quick tests
kinit "$USER@${AD_REALM}"
id someuser@"$AD_DOMAIN"   # should return an NSS line
ssh -K localhost           # SSO works if you already kinit'd
```

That was enough for me to log in with an AD user and run `sudo` if they’re in the `linux-sudoers` group.

---

## What I Changed (and Why)

### 1) SSSD defaults
`realm join` writes `/etc/sssd/sssd.conf`. I only tweaked a few basics to keep it simple:

```ini
[sssd]
services = nss, pam, sudo, autofs, ssh
config_file_version = 2
domains = example.com

[domain/example.com]
id_provider = ad
access_provider = ad

# I like short logins (just "jdoe" instead of "jdoe@example.com")
use_fully_qualified_names = False

fallback_homedir = /home/%u
default_shell = /bin/bash
cache_credentials = True
ldap_id_mapping = True

# sudo and autofs providers (enabled by services above)
sudo_provider = ad
autofs_provider = ad
```

Then I set permissions and restarted SSSD:

```bash
sudo chmod 600 /etc/sssd/sssd.conf
sudo systemctl restart sssd
```

### 2) SSH GSSAPI (for Kerberos SSO)
I added the small drop‑in file under `/etc/ssh/sshd_config.d/`. After restart, if I have a ticket (`kinit`), `ssh -K host` logs me in without typing a password.

### 3) sudo via AD groups
I made an AD security group called **linux-sudoers**. On the Linux side I mapped it in `/etc/sudoers.d/ad-sudoers` as shown above.  

> If your group name has a space, escape it (example: `%Domain\ Admins`). I avoid spaces when I can.

---

## Optional: Automount Home Directories (I tested NFS; SMB notes below)

I started with **local** home dirs (created automatically by `oddjob-mkhomedir`) because it’s easier. When I tried **NFS** homes, this worked:

1) Install autofs and NFS bits
```bash
sudo dnf -y install autofs nfs-utils
sudo systemctl enable --now autofs
```

2) Point `/home` to an automap
```text
# /etc/auto.master.d/home.autofs
/home  /etc/auto.home --timeout=120 --ghost
```

3) Map each user to `filesrv.example.com:/export/home/<user>`
```text
# /etc/auto.home
*  -fstype=nfs4,sec=krb5p,rw,soft,intr filesrv.example.com:/export/home/&
```
```bash
sudo systemctl reload autofs
```

Now going to `/home/jdoe` triggers a Kerberos‑protected NFS mount.

**SMB/CIFS note:** I didn’t fully test DFS, but people use something like:
```text
*  -fstype=cifs,sec=krb5i,cruid=${UID},multiuser,cache=none,vers=3.1.1 ://filesrv.example.com/home/&
```
Make sure `cifs-utils` is installed and users have a Kerberos ticket. YMMV with DFS referrals.

---

## Troubleshooting I Actually Used

```bash
# See joined realm info
realm list

# Kerberos tickets
klist
kinit user@EXAMPLE.COM
klist -kte /etc/krb5.keytab    # host keytab entries

# SSSD health
sudo systemctl status sssd
sudo journalctl -u sssd -f
getent passwd user@example.com
getent group linux-sudoers

# SSH SSO
ssh -vvv -K host.example.com

# Autofs logs
sudo journalctl -u autofs -f
```

### Things that tripped me up
- **DNS**: Forward and reverse must match. Wrong PTR gave me Kerberos errors (`…MODIFIED` or SPN issues).
- **Time**: If clocks drift >5 minutes, SSO breaks. `chronyd` fixed it.
- **Names**: With `use_fully_qualified_names=False`, I login as `jdoe`. If you prefer `jdoe@example.com`, set it to `True`.

---

## Security / Policy Notes (basic, from my perspective)
- I kept **SELinux enforcing**. For NFS homes: `sudo setsebool -P use_nfs_home_dirs=on` (only if you need it).  
- I only opened firewall ports if the box was serving something. As a client, normal egress was enough.  
- I used an AD group for sudo instead of listing users individually.

---

## Undo / Leave the Domain
```bash
sudo realm leave example.com
sudo rm -f /etc/krb5.keytab
sudo systemctl stop sssd && sudo rm -rf /var/lib/sss/db/* && sudo systemctl start sssd
```

---

## What I Still Want To Learn
- Managing sudo rules directly from AD/LDAP (schema extensions)
- Cleaner DFS home shares with Kerberos (SMB) across sites
- Best practices for nested groups and SSSD caching

---

## Credits
Big thanks to docs and forum posts I read while figuring this out. If you’ve got improvements or corrections, please send them my way.

---

## License
MIT — free to use. Please test in a lab first before rolling to prod :)
