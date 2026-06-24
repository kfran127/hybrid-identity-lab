<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:0D1117,50:1F6FEB,100:58A6FF&height=120&section=header" width="100%" alt=""/>

# 🏢 Hybrid Identity Lab

**On-premises Active Directory integrated with Microsoft Entra ID via Entra Connect Sync, simulating a real enterprise hybrid identity environment with automated JML workflows.**

![Status](https://img.shields.io/badge/Status-Complete-2EA44F?style=for-the-badge)
![Security+](https://img.shields.io/badge/Security%2B-Certified-2EA44F?style=for-the-badge)
![SC-300](https://img.shields.io/badge/SC--300-In_Progress-E36209?style=for-the-badge)

<sub>Built by **Kency Francois**</sub>

<br>

<img src="https://img.shields.io/badge/Active_Directory-003087?style=flat-square" alt="Active Directory"/>
<img src="https://img.shields.io/badge/Microsoft_Entra_Connect-5C2D91?style=flat-square" alt="Microsoft Entra Connect"/>
<img src="https://img.shields.io/badge/PowerShell-2D2D30?style=flat-square" alt="PowerShell"/>
<img src="https://img.shields.io/badge/Windows_Server_2022-0078D4?style=flat-square" alt="Windows Server 2022"/>
<img src="https://img.shields.io/badge/Azure-0089D6?style=flat-square" alt="Azure"/>

</div>

<br>

## 📋 Overview

Hybrid Identity Lab demonstrates how enterprise organizations manage identities across both on-premises Active Directory and cloud-based Microsoft Entra ID. The project covers deploying a Domain Controller in Azure, bulk provisioning users via PowerShell, configuring Entra Connect Sync with Password Hash Sync, and automating the full Joiner/Mover/Leaver lifecycle with changes syncing in real time from on-prem to cloud.

<div align="center">

| | |
|:--|:--|
| **Domain** | `securebank.local` |
| **Cloud Tenant** | `slatterals.onmicrosoft.com` (Microsoft Entra ID P2) |

</div>

<br>

## 🏗️ Architecture

```text
                    Mac (Microsoft Remote Desktop)
                                │
                                ▼
                  Azure VM: SecureBank-DC1
                  Windows Server 2022
                  Active Directory Domain Services
                  Domain: securebank.local
                  15 users across 5 OUs
                                │
                                ▼
                  Microsoft Entra Connect Sync
                  Password Hash Sync
                                │
                                ▼
                  Microsoft Entra ID
                  slatterals.onmicrosoft.com
                  Users sync automatically from AD to cloud
                  Changes propagate within minutes
                                │
                                ▼
                  ✅ Hybrid Identity Complete
```

<br>

## 📑 Contents

- [Phase 1 — Active Directory Foundation](#-phase-1-active-directory-foundation)
- [Phase 2 — Entra Connect Sync](#-phase-2-entra-connect-sync)
- [Phase 3 — Hybrid JML Workflows](#-phase-3-hybrid-jml-workflows)
- [Skills Demonstrated](#-skills-demonstrated)
- [Related Projects](#-related-projects)

<br>

## 📁 Phase 1: Active Directory Foundation

![Status](https://img.shields.io/badge/Status-Complete-2EA44F?style=flat-square)

### Why this matters

Every enterprise identity strategy starts with a directory of record. Most organizations still run Active Directory on-premises to manage logins, group memberships, and permissions for company devices and internal resources. Before any identity can sync to the cloud, it has to exist somewhere authoritative on-prem. This phase builds that foundation: a real Domain Controller, a real domain, and a department-based structure that mirrors how a bank would actually organize its workforce.

### Step-by-step

1. Deployed a Windows Server 2022 virtual machine in Azure named `SecureBank-DC1`
2. Installed the Active Directory Domain Services role on the VM
3. Promoted the server to a Domain Controller and created the `securebank.local` domain
4. Built 5 Organizational Units (OUs) matching SecureBank's department structure
5. Created a `Disabled Users` OU to hold offboarded employee accounts
6. Wrote a PowerShell script to bulk import 15 users from a CSV file, assigning each user to the correct OU along with department, title, and company attributes
7. Ran the script and verified all 15 users appeared correctly across the 5 department OUs

### Organizational Units

| OU | Department | Users |
|:--|:--|:--|
| `SB-RetailBanking` | Retail Banking | James Carter, Maria Lopez, David Kim |
| `SB-Compliance` | Compliance | Sarah Johnson, Robert Chen, Lisa Brown |
| `SB-ITSecurity` | IT Security | Kevin Patel, Amanda Torres, Marcus Williams |
| `SB-LoanOfficers` | Loan Officers | Jennifer Davis, Michael Scott, Rachel Green |
| `SB-Executive` | Executive | Thomas Anderson, Patricia Moore, Steven Clark |
| `Disabled Users` | Offboarded | *(used during Leaver workflow)* |

**ADUC Overview — All OUs**
![ADUC Overview](screenshots/phase1-aduc-overview.png)

### Bulk User Creation Script

```powershell
$Password = ConvertTo-SecureString "TempPass123!@#" -AsPlainText -Force
$Domain = "DC=securebank,DC=local"
$Users = Import-Csv "C:\SecureBank-Users.csv"

foreach ($User in $Users) {
    $OU = "OU=$($User.OU),$Domain"

    New-ADUser `
        -Name "$($User.FirstName) $($User.LastName)" `
        -GivenName $User.FirstName `
        -Surname $User.LastName `
        -SamAccountName $User.Username `
        -UserPrincipalName "$($User.Username)@securebank.local" `
        -Path $OU `
        -Department $User.Department `
        -Title $User.Title `
        -Company "SecureBank" `
        -AccountPassword $Password `
        -ChangePasswordAtLogon $true `
        -Enabled $true

    Write-Host "JOINER: Created $($User.FirstName) $($User.LastName) in $($User.OU)"
}
```

**SB-RetailBanking Users**
![RetailBanking](screenshots/phase1-retailbanking.png)

**SB-Compliance Users**
![Compliance](screenshots/phase1-compliance.png)

**SB-ITSecurity Users**
![ITSecurity](screenshots/phase1-itsecurity.png)

**SB-LoanOfficers Users**
![LoanOfficers](screenshots/phase1-loanofficers.png)

**SB-Executive Users**
![Executive](screenshots/phase1-executive.png)

<br>

## 🔗 Phase 2: Entra Connect Sync

![Status](https://img.shields.io/badge/Status-Complete-2EA44F?style=flat-square)

### Why this matters

Most companies are not 100% cloud or 100% on-prem, they are both. Employees need to log into Microsoft 365, Entra-protected apps, and cloud resources using the same identity they use to log into their work computer. Without sync, IT would have to manually create and maintain two separate sets of user accounts, which is slow and error-prone and creates security gaps when an account is disabled in one place but not the other. Entra Connect Sync solves this by automatically and continuously pushing identity data from on-prem AD into Entra ID, so there is one source of truth.

### Step-by-step

1. Installed Microsoft Entra Connect Sync directly on the Domain Controller
2. Selected Password Hash Sync (PHS) as the authentication method, so credentials sync securely without exposing passwords in plain text
3. Connected the on-prem domain `securebank.local` to the cloud tenant `slatterals.onmicrosoft.com`
4. Ran an initial sync cycle and confirmed connectivity between AD and Entra ID
5. Verified all 15 users appeared in Entra ID with **On-premises sync = Yes**
6. Spot-checked department, title, and company attributes to confirm they synced correctly from AD

**Entra Connect Sync Service Running**
![Sync Service](screenshots/phase2-sync-service.png)

**Entra ID Users with On-Premises Sync = Yes**
![Synced Users](screenshots/phase2-synced-users.png)

### How It Works

```text
Every 30 minutes (or on-demand via PowerShell):

securebank.local (AD)
        │
        ▼
Entra Connect detects changes:
  • New users      → sync to cloud
  • Modified users  → update in cloud
  • Disabled users  → disable in cloud
        │
        ▼
slatterals.onmicrosoft.com (Entra ID)
```

**Force sync command:**

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

<br>

## 🔄 Phase 3: Hybrid JML Workflows

![Status](https://img.shields.io/badge/Status-Complete-2EA44F?style=flat-square)

### Why this matters

Joiner, Mover, Leaver (JML) is the core lifecycle every identity goes through at a real company: someone gets hired, someone changes roles, someone leaves. Each event has security consequences. A slow Joiner process delays a new hire's first day. A missed Mover update leaves someone with access to a department they no longer work in. A missed Leaver process is one of the most common causes of real data breaches, because a former employee's account is left active. This phase proves the full lifecycle works end-to-end and syncs correctly between on-prem and the cloud at every step.

<br>

### 🟢 Joiner — Bulk Provisioning

**What this simulates in a real company:**
A new class of 15 employees joins SecureBank on the same day. IT needs to create all of their accounts, place them in the correct department, and have them ready to log in and show up in cloud apps without manual one-by-one setup.

**The PowerShell script used:**

```powershell
$Password = ConvertTo-SecureString "TempPass123!@#" -AsPlainText -Force
$Domain = "DC=securebank,DC=local"
$Users = Import-Csv "C:\SecureBank-Users.csv"

foreach ($User in $Users) {
    $OU = "OU=$($User.OU),$Domain"

    New-ADUser `
        -Name "$($User.FirstName) $($User.LastName)" `
        -GivenName $User.FirstName `
        -Surname $User.LastName `
        -SamAccountName $User.Username `
        -UserPrincipalName "$($User.Username)@securebank.local" `
        -Path $OU `
        -Department $User.Department `
        -Title $User.Title `
        -Company "SecureBank" `
        -AccountPassword $Password `
        -ChangePasswordAtLogon $true `
        -Enabled $true

    Write-Host "JOINER: Created $($User.FirstName) $($User.LastName) in $($User.OU)"
}
```

**Before:** No user accounts exist yet in `securebank.local`.

**Script running:**

**PowerShell Bulk Creation Output**
![Joiner Output](screenshots/phase3-joiner-output.png)

**After:** All 15 users exist in the correct OUs in AD and synced to Entra ID within minutes (see Phase 1 and Phase 2 screenshots above for the resulting accounts).

**What this proves to an employer:** I can provision enterprise users at scale using PowerShell instead of manual clicking, and I understand how those accounts flow from on-prem AD into the cloud automatically.

<br>

### 🟡 Mover — Role Change

**What this simulates in a real company:**
James Carter gets promoted from Bank Teller in Retail Banking to Junior Loan Officer in the Loan Officers department. His access, department, and title all need to update together, on-prem and in the cloud, without creating a duplicate account or leaving stale access behind.

**Before the change:**

| | Value |
|:--|:--|
| **OU** | `SB-RetailBanking` |
| **Title** | Bank Teller |
| **Department** | Retail Banking |

**James Carter Before Mover (RetailBanking)**
![Before Mover](screenshots/phase3-mover-before.png)

**The PowerShell script used:**

```powershell
$Username = "jcarter"
$NewOU = "OU=SB-LoanOfficers,DC=securebank,DC=local"

Get-ADUser $Username | Move-ADObject -TargetPath $NewOU
Set-ADUser $Username -Title "Junior Loan Officer" -Department "Loan Officers"

Start-ADSyncSyncCycle -PolicyType Delta
```

**Script running:**

**PowerShell Mover Output**
![Mover Output](screenshots/phase3-mover-output.png)

**After the change:**

| | Value |
|:--|:--|
| **OU** | `SB-LoanOfficers` |
| **Title** | Junior Loan Officer |
| **Department** | Loan Officers |

**James Carter After Mover (Entra ID, Junior Loan Officer)**
![After Mover](screenshots/phase3-mover-after.png)

**What this proves to an employer:** I understand that access changes need to happen in AD first and sync to the cloud, and I can script a role change so that OU, title, and department all update consistently instead of relying on manual edits that drift out of sync.

<br>

### 🔴 Leaver — Offboarding

**What this simulates in a real company:**
James Carter leaves SecureBank. His account needs to be disabled immediately, removed from active OUs, and that change needs to reflect in the cloud right away so he loses access to email, Microsoft 365, and any connected apps.

**Before the change:** James Carter's account is active in `SB-LoanOfficers` and enabled in both AD and Entra ID.

**The PowerShell script used:**

```powershell
$Username = "jcarter"

# Disable account
Disable-ADAccount -Identity $Username

# Move to Disabled Users OU
Get-ADUser $Username | Move-ADObject -TargetPath "OU=Disabled Users,DC=securebank,DC=local"

# Log offboarding date
Set-ADUser $Username -Description "DISABLED - Offboarded $(Get-Date -Format 'MM/dd/yyyy')"

# Sync to cloud
Start-ADSyncSyncCycle -PolicyType Delta
```

**Script running:**

**PowerShell Leaver Output**
![Leaver Output](screenshots/phase3-leaver-output.png)

**After the change:**

1. Account disabled immediately in AD
2. Moved to the `Disabled Users` OU
3. Offboarding date logged in the description field
4. Changes synced to Entra ID automatically
5. Account shows disabled in the cloud

**James Carter Disabled in Entra ID**
![Leaver Entra](screenshots/phase3-leaver-entra.png)

**Disabled Users OU in ADUC**
![Disabled OU](screenshots/phase3-disabled-ou.png)

**What this proves to an employer:** I understand offboarding as a security-critical process, not just an admin task. Disabling, relocating, and logging the account, then confirming the change synced to the cloud, is the kind of audit trail a real compliance or security team would expect to see.

<br>

## 📊 Skills Demonstrated

| Skill | Implementation |
|:--|:--|
| Active Directory Administration | Deployed DC, created domain, OUs, and users |
| Hybrid Identity | Entra Connect Sync with Password Hash Sync |
| PowerShell Automation | Bulk user creation via CSV, JML scripts |
| User Lifecycle Management (JML) | Joiner, Mover, Leaver fully automated |
| Password Hash Sync | Secure credential sync from AD to Entra ID |
| On-Premises to Cloud Sync | Real-time identity propagation |
| Organizational Unit Design | Department-based OU structure plus Disabled Users OU |
| Windows Server Administration | Server 2022 VM deployment and AD DS configuration |
| Azure Infrastructure | VM deployment, resource group, networking |
| Identity Governance | Offboarding workflow with audit trail |

<br>

## 🔗 Related Projects

- [SecureBank IAM Lab](https://github.com/kfran127/securebank-iam-lab) — cloud-native IAM environment in Microsoft Entra ID

<br>

<div align="center">

### 🔗 Connect

<a href="https://linkedin.com/in/kency-francois">
  <img src="https://img.shields.io/badge/LinkedIn-Connect-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white" alt="LinkedIn"/>
</a>
&nbsp;
<a href="mailto:kfran127@fiu.edu">
  <img src="https://img.shields.io/badge/Email-Reach_Out-EA4335?style=for-the-badge&logo=gmail&logoColor=white" alt="Email"/>
</a>
&nbsp;
<a href="https://github.com/kfran127">
  <img src="https://img.shields.io/badge/GitHub-kfran127-181717?style=for-the-badge&logo=github&logoColor=white" alt="GitHub"/>
</a>

**Open to:** IAM Analyst and IAM Administrator roles · Remote and Hybrid · Available Immediately

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:58A6FF,50:1F6FEB,100:0D1117&height=100&section=footer" width="100%" alt=""/>

</div>
