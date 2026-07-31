# JML (Joiner-Mover-Leaver) Lifecycle Automation Lab

**Tools:** PowerShell, Microsoft Entra ID, Microsoft Graph API

## Overview

This lab simulates an enterprise identity lifecycle automation pipeline driven by an HR system of truth. Rather than manually provisioning, transferring, or offboarding accounts one at a time, a single PowerShell reconciliation script reads an HR export (CSV) and automatically applies the correct identity action based on each record's status — mirroring how real-world IGA platforms (SailPoint, Okta Workflows, etc.) operate under the hood.

The lab environment simulates a company ("FernandoTech") with 6 departments (modeled as soccer clubs for readability) and processes a 62-record HR batch covering the full identity lifecycle.

## Architecture

- **Source of truth:** `HR_Roster.csv` — simulates an HR system export with `Name, Club, Status, ManagerName, JobTitle` columns

<img width="450" height="239" alt="HR Roster source file" src="https://github.com/user-attachments/assets/c2281f1b-a8b7-47cc-a1bc-4a1c1059ebbf" />

- **Reconciliation engine:** A single PowerShell script reads the CSV and branches into Joiner / Mover / Leaver logic based on the `Status` field per record

<img width="653" height="144" alt="Reconciliation summary" src="https://github.com/user-attachments/assets/d4915b3a-2876-4458-8718-b3c85a389255" />

- **Identity platform:** Microsoft Entra ID (Microsoft Graph PowerShell SDK)
- **Access control:** Department-based security groups (RBAC), enforced via Conditional Access policies already active on the tenant (MFA, device compliance, legacy auth blocking)
- **Audit trail:** Every lifecycle event is logged with a timestamp to `JML_AuditLog.csv`

## Results

| Event Type | Count |
|---|---|
| Joiner | 50 |
| Mover | 8 |
| Leaver | 4 |
| **Total records processed** | **62** |

## Section 1: Joiner

50 new identities provisioned in a single automated pass. For each record:

- Account created in Entra ID with department and job title attributes set
- Manager attribute linked to the correct department manager (a real user object, not just a text field)
- Added to both the department-specific security group and a company-wide baseline group
- Event logged with timestamp

**Sample script logic:**

```powershell
$newUser = New-MgUser -DisplayName $record.Name -MailNickname $mailNickname -UserPrincipalName $upn `
    -AccountEnabled -JobTitle $record.JobTitle -Department $record.Club -PasswordProfile $pw

$manager = Get-MgUser -Filter "displayName eq '$($record.ManagerName)'"
Set-MgUserManagerByRef -UserId $newUser.Id -BodyParameter @{ "@odata.id" = "https://graph.microsoft.com/v1.0/users/$($manager.Id)" }

New-MgGroupMember -GroupId $clubGroup.Id -DirectoryObjectId $newUser.Id
New-MgGroupMember -GroupId $allPlayers.Id -DirectoryObjectId $newUser.Id
```

<img width="1620" height="773" alt="Joiner script output" src="https://github.com/user-attachments/assets/a4adf4e9-445f-4099-ab60-699e3f02fe4e" />

<img width="1202" height="859" alt="Entra ID groups and users after Joiner run" src="https://github.com/user-attachments/assets/e16ec4b2-7620-4b9a-a7ed-27df23ebfe90" />

## Section 2: Mover

8 identities transferred between departments, simulating promotions/role changes. For each transfer:

- Removed from old department's security group
- Added to new department's security group
- Department attribute updated
- Manager attribute reassigned to reflect new reporting line
- Full before/after state logged for audit purposes

**Sample script logic:**

```powershell
Remove-MgGroupMemberByRef -GroupId $oldGroup.Id -DirectoryObjectId $user.Id
New-MgGroupMember -GroupId $newGroup.Id -DirectoryObjectId $user.Id
Update-MgUser -UserId $user.Id -Department $record.Club
Set-MgUserManagerByRef -UserId $user.Id -BodyParameter @{ "@odata.id" = "https://graph.microsoft.com/v1.0/users/$($newManager.Id)" }
```

<img width="714" height="92" alt="Mover proof - Harry Kane department change" src="https://github.com/user-attachments/assets/10803112-0220-4587-9380-3df91f2d3d2b" />
<img width="864" height="99" alt="image" src="https://github.com/user-attachments/assets/f6e371f5-a4ec-4f4e-880f-74dbe9f9015f" />

## Section 3: Leaver

4 identities offboarded using a 6-step deprovisioning sequence:

1. Disable account
2. Randomize password
3. Remove all group memberships
4. Stamp audit timestamp
5. Hide from Global Address List
6. Move to Disabled Users group (cloud equivalent of moving to a Disabled OU)

**Sample script logic:**

```powershell
Update-MgUser -UserId $user.Id -AccountEnabled:$false
Update-MgUser -UserId $user.Id -PasswordProfile @{ Password = $randomPw; ForceChangePasswordNextSignIn = $true }
foreach ($g in $memberships) { Remove-MgGroupMemberByRef -GroupId $g.Id -DirectoryObjectId $user.Id }
Update-MgUser -UserId $user.Id -ShowInAddressList:$false
New-MgGroupMember -GroupId $disabledGroup.Id -DirectoryObjectId $user.Id
```

<img width="784" height="90" alt="Leaver proof - Messi account disabled" src="https://github.com/user-attachments/assets/82f652ca-cfe4-48a6-a8bd-7576e92c6d0b" />
<img width="868" height="104" alt="image" src="https://github.com/user-attachments/assets/ebdfdbf6-b4fd-4551-a7b4-18aaa3bba1e9" />

## Access Governance

<img width="839" height="171" alt="Audit log sample" src="https://github.com/user-attachments/assets/39d1cc58-0b31-4b51-9d5e-bd8617c4ef7a" />

All identities in this lab operate under existing tenant-wide Conditional Access policies:

- **CORP-Require-MFA-All-Users** — MFA enforced for all sign-ins
- **CORP-Require-Compliant-Device** — device compliance required for access
- **CORP-Block-Legacy-Authentication** — legacy auth protocols blocked

<img width="1103" height="619" alt="Conditional Access policies" src="https://github.com/user-attachments/assets/b81671de-450f-4008-b3fa-6e5d382c25cd" />

This demonstrates that provisioned identities inherit real security posture immediately upon creation, not as a separate manual step.

## Key Takeaways

- Built a reconciliation pattern (HR source of truth → automated identity actions) rather than three disconnected manual scripts
- Demonstrated full JML lifecycle: provisioning, mid-lifecycle attribute/group changes, and secure offboarding
- Full audit trail across all 62 events for compliance/audit-readiness
- Access governed by existing Conditional Access policies (MFA, device compliance, legacy auth blocking)
