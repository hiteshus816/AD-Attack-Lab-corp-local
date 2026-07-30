# Diagram Recreation Instructions

These are text-based specifications so each diagram can be recreated in draw.io, Excalidraw, Mermaid, or any diagramming tool. Mermaid code blocks are included where the diagram type supports it directly.

---

## 1. Network Diagram

**Layout:** Star topology. One central "Attacker" node (Kali Linux) connected via a shared internal switch/subnet to three target nodes.

**Nodes:**
- Kali Linux (Attacker) — `192.168.100.x` — icon: Linux penguin or terminal
- DC01 — `192.168.100.10` — Windows Server 2022, Domain Controller — icon: server with AD logo/shield
- SALES-PC — `192.168.100.20` — Windows 10 workstation — icon: desktop PC
- USER-PC — `192.168.100.30` — Windows 10 workstation — icon: desktop PC

**Connections:** All four nodes connect to a central "192.168.100.0/24 (Internal Network)" switch/cloud shape. Label the connecting network segment.

**Annotations:** Add a small note near DC01: "SMB Signing: Enabled | SMBv1: Disabled." Add a note near the network cloud: "Isolated VirtualBox host-only/internal network — no external routing."

```mermaid
graph TB
    NET["192.168.100.0/24<br/>Internal Network"]
    ATK["Kali Linux (Attacker)<br/>192.168.100.x"]
    DC["DC01<br/>192.168.100.10<br/>Windows Server 2022 (DC)<br/>corp.local"]
    SALES["SALES-PC<br/>192.168.100.20<br/>Windows 10"]
    USER["USER-PC<br/>192.168.100.30<br/>Windows 10"]

    ATK --- NET
    DC --- NET
    SALES --- NET
    USER --- NET
```

---

## 2. Active Directory Architecture Diagram

**Layout:** Top-down organizational tree.

**Level 1 (root):** `corp.local` domain

**Level 2 (OUs):** IT | HR | Finance | Sales | Servers | Workstations | Managed Service Accounts | Users (default container)

**Level 3 (objects under each OU):**
- IT → `john smith (js)` → member of **IT Admins**
- HR → `mary hr (hr)` → member of **HR Group**
- Finance → `bob finance (bob)` → member of **Finance**
- Sales → `alice sales (as)` → member of **Sales**
- Servers → (computer objects — none custom beyond DC01)
- Workstations → `SALES-PC`, `USER-PC`
- Users (default) → `Administrator`, `Guest`, `help desk (hd)`, `intern 1`, `intern 2`, `svc_backup`, `svc_sql`, `krbtgt`

**Highlight box:** Draw `svc_sql` with a red/warning-colored border and a dual arrow showing it belongs to **both** `SQL Admin` (intended scope) **and** `Domain Admins` (unintended/critical finding). This dual-membership visual is the single most important element of this diagram.

```mermaid
graph TD
    ROOT[corp.local]
    ROOT --> IT[IT OU]
    ROOT --> HR[HR OU]
    ROOT --> FIN[Finance OU]
    ROOT --> SALES[Sales OU]
    ROOT --> SRV[Servers OU]
    ROOT --> WS[Workstations OU]
    ROOT --> USERS[Users container]

    IT --> JS[john smith / js]
    JS -.member of.-> ITADM[IT Admins]

    HR --> HRU[mary hr / hr]
    HRU -.member of.-> HRGRP[HR Group]

    FIN --> BOB[bob finance / bob]
    BOB -.member of.-> FINGRP[Finance]

    SALES --> AS[alice sales / as]
    AS -.member of.-> SALESGRP[Sales]

    WS --> SALESPC[SALES-PC]
    WS --> USERPC[USER-PC]

    USERS --> SVCSQL[svc_sql]
    USERS --> SVCBK[svc_backup]
    USERS --> HD[help desk / hd]
    USERS --> I1[intern 1]
    USERS --> I2[intern 2]

    SVCSQL -.member of.-> SQLADM[SQL Admin - intended]
    SVCSQL ==CRITICAL: also member of==> DA["Domain Admins ⚠️"]

    style SVCSQL fill:#ffcccc,stroke:#cc0000,stroke-width:2px
    style DA fill:#ffcccc,stroke:#cc0000,stroke-width:2px
```

---

## 3. Attack Flow Diagram

**Layout:** Left-to-right (or top-to-bottom) swimlane/sequence showing 10 stages.

```mermaid
flowchart LR
    A[1. Initial Access<br/>js:Password123!] --> B[2. SMB/LDAP/AD<br/>Enumeration]
    B --> C[3. BloodHound<br/>Collection]
    C --> D[4. BloodHound Analysis:<br/>Kerberoastable Accounts Query]
    D --> E[5. Kerberoasting<br/>impacket-GetUserSPNs]
    E --> F[6. Offline Cracking<br/>hashcat + rockyou.txt]
    F --> G[7. Recovered Password<br/>svc_sql:Password@123]
    G --> H[8. WinRM Access<br/>evil-winrm]
    H --> I[9. Privilege Confirmation<br/>whoami /all → Domain Admins]
    I --> J[10. Post-Exploitation<br/>Enumeration]

    style D fill:#ffe4b5,stroke:#cc6600
    style I fill:#ffcccc,stroke:#cc0000,stroke-width:2px
```

**Annotation:** Mark step 4 (BloodHound Analysis) as the "decision point" and step 9 (Privilege Confirmation) as "Full Domain Compromise" in a distinct color (as shown above).

---

## 4. Kerberoasting Workflow Diagram

**Layout:** Sequence diagram between three actors: Attacker, Domain Controller (KDC), Offline Cracking Host.

```mermaid
sequenceDiagram
    participant A as Attacker (js)
    participant KDC as Domain Controller (KDC)
    participant H as Hashcat (offline)

    A->>KDC: TGT request (already authenticated as js)
    KDC-->>A: TGT issued
    A->>KDC: TGS-REQ for MSSQLSvc/DC01.corp.local:1433 (svc_sql's SPN)
    KDC-->>A: TGS-REP encrypted with svc_sql's NTLM hash (RC4/etype 23)
    A->>A: Extract $krb5tgs$23$... hash (GetUserSPNs -request)
    A->>H: Submit hash + rockyou.txt (hashcat -m 13100)
    H-->>A: Cracked: Password@123
    Note over A,H: No elevated privilege needed for the TGS-REQ step —<br/>any authenticated domain user can request a ticket<br/>for any account with an SPN.
```

---

## 5. BloodHound Privilege Path Diagram

**Layout:** Directed graph, matching the actual BloodHound visualization captured in the lab.

```mermaid
graph LR
    JS[js / john smith]
    SVCSQL[svc_sql]
    SQLADM[SQL Admin group]
    DA[Domain Admins]
    DC01[DC01.corp.local]

    SVCSQL -->|HasSPN: MSSQLSvc/DC01:1433<br/>Kerberoastable| KERB{{Kerberoastable}}
    SVCSQL -->|MemberOf| SQLADM
    SVCSQL ==MemberOf ⚠️==> DA
    DA -->|GenericAll| DC01

    style SVCSQL fill:#ffe4b5,stroke:#cc6600,stroke-width:2px
    style DA fill:#ffcccc,stroke:#cc0000,stroke-width:2px
    style KERB fill:#fff2cc,stroke:#cc9900
```

**Caption:** "BloodHound's 'Shortest Path to Domain Admins from Kerberoastable Users' query returns a single-hop result: `svc_sql` is both Kerberoastable and a direct member of Domain Admins."

---

## 6. MITRE ATT&CK Flow Diagram

**Layout:** Horizontal kill-chain style, one box per tactic (top row), technique beneath each.

```mermaid
flowchart LR
    subgraph Discovery
        T1["T1087.002<br/>Account Discovery"]
        T2["T1135<br/>Network Share Discovery"]
        T3["T1018<br/>Remote System Discovery"]
    end
    subgraph CredAccess["Credential Access"]
        T4["T1558.003<br/>Kerberoasting"]
        T5["T1110.002<br/>Password Cracking"]
        T6["T1552.006<br/>Unsecured Credentials"]
    end
    subgraph PrivEsc["Privilege Escalation"]
        T7["T1078.002<br/>Valid Accounts: Domain"]
    end
    subgraph LatMove["Lateral Movement"]
        T8["T1021.006<br/>WinRM"]
    end

    Discovery --> CredAccess --> PrivEsc --> LatMove

    style T4 fill:#ffcccc,stroke:#cc0000,stroke-width:2px
    style T7 fill:#ffcccc,stroke:#cc0000,stroke-width:2px
```
