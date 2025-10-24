# HTB: thefrizz — User Shell (sanitized)
Date: 2025-10-24
Status: User shell obtained (sanitized)

TL;DR
- Obtained user access via enumeration and credential discovery. All sensitive data omitted.

High-level Methodology (no spoilers)
1. Enumeration: host/service and web enumeration.
2. Initial access: credential discovery (no code shown).
3. Post-access: basic file inspection for privilege escalation hints.

Detection / SOC notes
- SIEM example:
  index=auth sourcetype=linux "Accepted password for" | stats count by user, src_ip

Notes
- No IPs, outputs or exploit code included in this public repo.
