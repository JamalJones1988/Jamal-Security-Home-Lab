# Active Directory Domain Services Lab — Azure Windows Server 2022

**Project 01 | Jamal Jones Security Home Lab**

---

## Objective

Deploy and administer Active Directory Domain Services on Windows Server 2022 in Azure, designing a realistic OU structure, department-based security groups, and Group Policy enforcement — then use Windows Security event logging to detect and document a privileged group membership change.

---

## Environment

| Component | Details |
|---|---|
| Cloud Platform | Microsoft Azure (Azure for Students) |
| Operating System | Windows Server 2022 Datacenter |
| VM Size | Standard B2als_v2 (2 vCPUs, 4GB RAM) |
| Domain | lab.local |
| Tools Used | Active Directory Users & Computers, Group Policy Management, Event Viewer |

---

## What I Built and Why

### OU Structure
Designed a department-based Organizational Unit structure mirroring a small company environment:

- **I.T.** — IT department users and admin accounts
- **Human Resources** — HR department users
- **Sales** — Sales department users

Real enterprises organize AD this way to apply different policies per department and delegate administration cleanly. Flat AD environments (everything dumped in the default Users container) are harder to manage and harder to secure.

### User Accounts
Created 8 user accounts across the three OUs with full organizational attributes populated — Department, Job Title, and organizational properties. Leaving these blank is common in lab environments but fails to reflect real enterprise AD.

### Security Groups
Created one security group per department:
- **IT-Staff**
- **Human Resources-Staff**
- **Sales-Staff**

Users were added to their respective department groups. During this process I caught a privilege creep error — a user was accidentally added to a second department group. This was identified during a group membership audit and corrected immediately. This is a realistic scenario: unauthorized group membership is a common finding in enterprise AD audits and a potential attacker persistence mechanism.

### Admin Accounts
Created a dedicated admin account (admin.smith) and added it to the **Domain Admins** built-in group, following the principle of least privilege — admin accounts are kept separate from day-to-day user accounts.

### Group Policy Objects
Configured four GPOs linked to the domain:

| GPO | Key Setting |
|---|---|
| Password-Policy | Minimum 12 characters, complexity enabled, 90-day maximum age |
| Account-Lockout-Policy | Lock after 5 failed attempts, 30-minute duration |
| Screen-Lock-Policy | Screen saver timeout at 15 minutes |
| Audit-Policy | Logon success/failure, account management, privilege use, policy change |

Verified GPO application using `gpupdate /force` and `gpresult /r`.

### Audit Policy & Event Logging
Enabled Windows Security auditing to capture:

| Event ID | Description |
|---|---|
| 4624 | Successful logon |
| 4625 | Failed logon attempt |
| 4634 | Account logoff |
| 4672 | Special privileges assigned (admin logon) |
| 4728 | Member added to security-enabled global group |
| 4688 | Process creation |

Generated and confirmed live event capture in Event Viewer, including a deliberate failed logon (4625) and a privileged group membership change (4728) to validate the full audit pipeline.

---

## Detection Notes — Why SOC Analysts Monitor Event ID 4728

**Event ID 4728: "A member was added to a security-enabled global group"**

This event fires any time a user is added to a security group in Active Directory. For high-privilege groups like **Domain Admins**, this is one of the most critical events a SOC analyst monitors.

**Why it matters:**
- Attackers who compromise a low-privilege account will attempt to add it to Domain Admins to gain full domain control
- Insider threats may quietly escalate their own privileges
- Legitimate IT changes should be authorized and expected — an unexpected 4728 on Domain Admins at 2AM is a high-priority alert

**In a real SOC environment**, this event would be:
1. Ingested into a SIEM (Splunk, Sentinel, etc.)
2. Trigger an automated alert
3. Routed to an analyst for review within minutes

This lab demonstrates the full chain: GPO enables the audit → Windows generates the log → analyst reviews in Event Viewer. Project 02 extends this by forwarding these logs into Splunk for centralized detection.

---

## Screenshots

**OU Tree and Department Structure**
![OU Tree](Image%201_.png)

**I.T. OU — Users and Admin Account**
![IT OU](Image%202.png)

**Sales OU**
![Sales OU](Image%203.png)

**Human Resources OU**
![HR OU](Image%204.png)

**IT-Staff Group Members**
![IT Staff Group](Image%205.png)

**Sales-Staff Group Members**
![Sales Staff Group](Image%206.png)

**Human Resources-Staff Group Members**
![HR Staff Group](Image%207.png)

**Domain Admins — Privileged Group Members**
![Domain Admins](Image%208.png)

**GPO Overview — All Policies Linked to lab.local**
![GPO Overview](Image%209.png)

**GPO Verification — gpresult /r Output**
![GPResult](Image%2010.png)

**Event ID 4728 — Member Added to Domain Admins**
![Event 4728](Image%2011.png)

**Event ID 4625 — Failed Logon Detection**
![Event 4625](Image%2012.png)

---

## Lessons Learned

- **Privilege creep happens fast.** Bulk-adding users to groups is efficient but requires an immediate membership audit — I caught a user in the wrong security group and it reinforced why AD auditing exists.
- **Azure time sync fights you.** Azure VMs default to UTC via Hyper-V time synchronization. Disabling the VMICTimeProvider and pointing to an external NTP server (time.nist.gov) was required to get accurate log timestamps — a real operational concern in any cloud-hosted lab.
- **GPOs need verification, not just creation.** Creating a GPO doesn't mean it applied. Running `gpresult /r` to confirm application is a habit worth building — it's the difference between thinking you're protected and knowing you are.
- **OU design reflects security posture.** How you structure OUs determines what policies apply where and who can administer what. A flat structure is a red flag in any enterprise AD review.
- **Event Viewer is the foundation.** Before any SIEM, the raw Windows Security log is where the truth lives. Being able to navigate, filter, and interpret Event Viewer without a SIEM is a core SOC analyst skill.

---

## Repository Structure

```
jamal-security-home-lab/
└── 01-active-directory-lab/
    ├── README.md
    └── Images 1-12 (lab screenshots)
```

---

## Next Project

**Project 02 — Ubuntu Server + Splunk SIEM:** Deploy Splunk on Ubuntu Server in Azure, configure a Universal Forwarder on this Windows Server VM to ship Security logs, and build an SPL alert for Event ID 4728.
