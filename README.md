# 🧩 Automated JML (Joiner • Mover • Leaver) with Active Directory, Entra Sync & PowerShell

**Goal**  
HR actions in **Active Directory** (create, move, disable a user) automatically update **group access**, which flows to **Microsoft Entra ID** through sync — without opening tickets or manually assigning groups.

This lab runs entirely in **one Windows Server VM** acting as:
- Domain Controller (AD DS)
- Entra Connect Sync server
- Automation host (PowerShell + Task Scheduler)

<img width="1536" height="1024" alt="ChatGPT Image Feb 4, 2026 at 11_40_06 PM" src="https://github.com/user-attachments/assets/ff35adf3-40c9-43c0-b3cf-ff0c8f9bc470" />

---

## 🏗️ Environment

- Azure VM — Windows Server 2022
- Active Directory Domain Services (`lab.local`)
- Entra Connect Sync
- PowerShell automation
- Task Scheduler for continuous enforcement

---

## ✅ What this proves

- AD OU structure drives access
- Security groups represent role-based access
- PowerShell enforces JML automatically
- Scheduled task keeps identities in the correct state
- Entra reflects changes through sync
- No tickets, no manual group assignment

---

## 🗂️ AD Structure (Source of Truth)

### OUs
- `OU_HR`
- `OU_Finance`
- `OU_IT`
- `OU_Disabled`

### Security Groups
- `AD-HR-Users`
- `AD-Finance-Users`
- `AD-IT-Admins`

### Test Users
- `Felipe-HR1`
- `Felipe-FIN1`
- `Felipe-IT1`

📸 **Screenshots**

![Image 2-4-26 at 22 14](https://github.com/user-attachments/assets/11da5902-32c7-40a7-8242-483d8b4959a4)

![Image 2-4-26 at 22 20](https://github.com/user-attachments/assets/661fbe0f-8c69-4c32-97be-b127881860ee)

![Image 2-4-26 at 22 28](https://github.com/user-attachments/assets/c9a79cf2-0036-405c-ab48-67cf58b4047b)

![Image 2-4-26 at 22 30](https://github.com/user-attachments/assets/a05fb69f-97d6-4524-bf01-e6f26426fe10)

![Image 2-4-26 at 22 30](https://github.com/user-attachments/assets/c5fa6624-aa2a-44c3-9a3f-c3de452b3827)

![Image 2-4-26 at 22 31](https://github.com/user-attachments/assets/6706641f-aab3-4b57-aee1-cf710020444a)

![Image 2-4-26 at 22 33](https://github.com/user-attachments/assets/4f1348b2-2bef-4bf2-a662-f370df08ae74)

![Image 2-4-26 at 22 34](https://github.com/user-attachments/assets/b79e9ee0-f5b0-41e5-a1c1-802aa6de2710)

![Image 2-4-26 at 22 35](https://github.com/user-attachments/assets/edb00db6-d37c-4f6e-8efe-b50f0c06c5a1)

---

## ⚙️ Automation Folder

C:\IAM-Automation
├── JML-AutoGroups.ps1
└── Logs

📸 **Screenshot**: Folder structure

![Image 2-4-26 at 22 38](https://github.com/user-attachments/assets/9c79494a-765d-464e-8026-77e73431965c)

![Image 2-4-26 at 22 46](https://github.com/user-attachments/assets/84985ef3-bf5f-4ac1-a64a-4d2c9290695c)

---

## 🧠 PowerShell — JML Enforcement Logic

This script:

- Scans each OU
- Ensures users are only in the correct group
- Removes users from incorrect groups when moved (Mover)
- Removes all access when placed in Disabled OU (Leaver)
- Writes actions to a log file for proof

```powershell
Import-Module ActiveDirectory

$HR_OU       = "OU=OU_HR,DC=lab,DC=local"
$FIN_OU      = "OU=OU_Finance,DC=lab,DC=local"
$IT_OU       = "OU=OU_IT,DC=lab,DC=local"
$DISABLED_OU = "OU=OU_Disabled,DC=lab,DC=local"

$HR_GROUP  = "AD-HR-Users"
$FIN_GROUP = "AD-Finance-Users"
$IT_GROUP  = "AD-IT-Admins"

$LOGFILE = "C:\IAM-Automation\Logs\JML-AutoGroups.log"

function Write-Log($msg) {
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    Add-Content -Path $LOGFILE -Value "$timestamp - $msg"
}

Write-Log "==== JML Automation Run Started ===="

$HR_Users  = Get-ADUser -SearchBase $HR_OU  -Filter * -Properties Enabled
$FIN_Users = Get-ADUser -SearchBase $FIN_OU -Filter * -Properties Enabled
$IT_Users  = Get-ADUser -SearchBase $IT_OU  -Filter * -Properties Enabled

function Ensure-GroupMembership($user, $correctGroup, $otherGroups) {
    if ($user.Enabled -eq $false) { return }

    try {
        Add-ADGroupMember -Identity $correctGroup -Members $user.SamAccountName -ErrorAction Stop
        Write-Log "Added $($user.SamAccountName) to $correctGroup"
    } catch {}

    foreach ($g in $otherGroups) {
        try {
            Remove-ADGroupMember -Identity $g -Members $user.SamAccountName -Confirm:$false
            Write-Log "Removed $($user.SamAccountName) from $g"
        } catch {}
    }
}

foreach ($u in $HR_Users)  { Ensure-GroupMembership $u $HR_GROUP @($FIN_GROUP, $IT_GROUP) }
foreach ($u in $FIN_Users) { Ensure-GroupMembership $u $FIN_GROUP @($HR_GROUP, $IT_GROUP) }
foreach ($u in $IT_Users)  { Ensure-GroupMembership $u $IT_GROUP @($HR_GROUP, $FIN_GROUP) }

$DisabledUsers = Get-ADUser -SearchBase $DISABLED_OU -Filter * -Properties Enabled
foreach ($u in $DisabledUsers) {
    foreach ($g in @($HR_GROUP,$FIN_GROUP,$IT_GROUP)) {
        try { Remove-ADGroupMember -Identity $g -Members $u.SamAccountName -Confirm:$false } catch {}
    }
}

Write-Log "==== JML Automation Run Finished ===="

```
📸 **Screenshots**

![Image 2-4-26 at 22 59](https://github.com/user-attachments/assets/d175cc11-2058-4736-a03c-1b43e12aa966)

![Image 2-4-26 at 22 59](https://github.com/user-attachments/assets/17b51fc9-5bd8-43c9-b8bf-ccd465e3df42)

![Image 2-4-26 at 23 01](https://github.com/user-attachments/assets/4d963dd5-7c0a-4cb7-a173-5e4960fb121a)

![Image 2-4-26 at 23 01](https://github.com/user-attachments/assets/6d60e5da-d2b0-49ef-a6ee-a873eee1e5c1)

![Image 2-4-26 at 23 02](https://github.com/user-attachments/assets/994171f7-0751-465f-a202-8feaf0739c84)

---

## ⏱️ Continuous Enforcement — Task Scheduler

A scheduled task runs every **5 minutes** to enforce JML automatically:

- Executes the PowerShell script silently
- Continuously validates correct group membership
- Writes every action to the log file for auditing

📸 **Screenshots**

![Image 2-4-26 at 23 04](https://github.com/user-attachments/assets/a8690208-e2ac-4d87-a7bb-4a3ea3624a8b)

![Image 2-4-26 at 23 06](https://github.com/user-attachments/assets/ac509318-e229-4557-8e37-6cd670214f46)

![Image 2-4-26 at 23 07](https://github.com/user-attachments/assets/60bc595b-a42a-49ba-9508-ef242a4378aa)

![Image 2-4-26 at 23 08](https://github.com/user-attachments/assets/0c91a06b-102c-46f5-ba22-64c88c21f500)

![Image 2-4-26 at 23 10](https://github.com/user-attachments/assets/3be8375c-d94f-43a6-8af0-458cee06d58a)

---

## 🔁 Mover Scenario (OU Change)

**Action**  
`Felipe-HR1` moved from `OU_HR` → `OU_Finance`

**Result**
- Removed from `AD-HR-Users`
- Added to `AD-Finance-Users`

📸 **Screenshots**

![Image 2-4-26 at 23 16](https://github.com/user-attachments/assets/8b30850b-479d-43c9-aaf4-10017acf24f6)

---

## 🚫 Leaver Scenario (Disable + Disabled OU)

**Action**  
`Felipe-FIN1` moved to `OU_Disabled` and account disabled

**Result**
- Removed from all access groups
- Logged as a **LEAVER** action

📸 **Screenshots**

![Image 2-4-26 at 23 20](https://github.com/user-attachments/assets/d279fb95-c1ab-40f5-9532-9d773bf48c4b)

![Image 2-4-26 at 23 21](https://github.com/user-attachments/assets/5057a9ed-7465-4e57-a8f4-69fb72a05886)

---

## 🧠 What This Demonstrates

This project shows how **Active Directory becomes the identity source of truth** and how automation enforces **Joiner, Mover, Leaver** without manual intervention.

It mirrors how enterprise IAM teams implement identity lifecycle management using:

- OU structure
- Security groups
- PowerShell automation
- Scheduled task enforcement
