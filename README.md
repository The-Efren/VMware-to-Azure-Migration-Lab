# VMware to Azure Migration Lab

## Objective

This lab aimed to establish a hybrid cloud identity environment simulating a real-world enterprise migration from on-premises Active Directory to Microsoft Azure. The primary focus was to deploy and configure Active Directory Domain Services and synchronize identities to Microsoft Entra ID using Entra Connect. I also implemented cloud security controls including Role-Based Access Control (RBAC) and Multi-Factor Authentication (MFA). This hands-on experience was designed to deepen my understanding of hybrid identity architecture, cloud migration methodology, and Azure administration.

<img width="1532" height="1245" alt="azure-migration-diagram drawio" src="https://github.com/user-attachments/assets/65ff93fa-3623-4f8f-9b45-e85a6a37a259" />



---

### Skills Learned

- Deployment and configuration of Active Directory Domain Services (AD DS) on Windows Server 2022
- Design and implementation of Organizational Unit (OU) hierarchies following enterprise best practices
- Understanding and application of the Principle of Least Privilege through service account management
- Configuration of Microsoft Entra Connect Sync with Password Hash Synchronization (PHS)
- Implementation of OU-level filtering to control identity sync scope
- Azure Role-Based Access Control (RBAC) role assignment across synced identities
- Multi-Factor Authentication (MFA) enforcement via Entra ID Security Defaults
- Hybrid identity architecture design connecting on-premises AD to Microsoft Entra ID
- Cloud migration planning, documentation, and troubleshooting methodology
- Static IP and DNS configuration for critical infrastructure servers
- PowerShell automation for AD object creation and management

---

### Tools Used

- **VMware Workstation Pro** — Local virtualization platform for hosting DC01
- **Windows Server 2022** — Operating system for the Domain Controller
- **Active Directory Domain Services (AD DS)** — On-premises identity provider
- **Active Directory Users and Computers (ADUC)** — GUI management console for AD
- **Microsoft Entra ID** — Cloud identity platform (formerly Azure AD)
- **Microsoft Entra Connect Sync** — Hybrid identity synchronization engine
- **Microsoft Azure Portal** — Cloud administration interface (portal.azure.com)
- **Microsoft Entra Admin Center** — Entra ID administration interface (entra.microsoft.com)
- **PowerShell** — Automation and configuration scripting
- **DNS Manager** — Domain name resolution configuration
- **Azure RBAC** — Cloud resource access control

---

## Steps

---

### Phase 1 — Active Directory Setup

---

#### Step 1 — VMware VM Creation & Windows Server 2022 Installation

Created a new Virtual Machine in VMware Workstation Pro configured with 2 vCPUs, 4-8GB RAM, and a 60GB thin-provisioned disk. Installed Windows Server 2022 Standard with Desktop Experience. The VM was named DC01 and configured with two network adapters — NAT for internet access and Host-Only for internal domain traffic.

| Property | Value |
|---|---|
| VM Name | DC01 |
| vCPUs | 2 |
| RAM | 4-8 GB |
| Disk | 60 GB (Thin Provisioned) |
| OS | Windows Server 2022 Standard (Desktop Experience) |
| NIC 1 | NAT — Internet access |
| NIC 2 | Host-Only — Internal domain network |

*Ref 1: VMware VM settings showing dual NIC configuration (NAT + Host-Only)*  

<img width="888" height="467" alt="vmware net settings " src="https://github.com/user-attachments/assets/b63ab720-b899-4b7a-ae9f-560590db7538" />


---

#### Step 2 — Static IP Configuration

Configured a static IP address on the Host-Only NIC (Ethernet1) to ensure the Domain Controller always has a predictable, consistent address. Dynamic IPs on a DC would break DNS resolution, Kerberos authentication, and AD replication — making static assignment non-negotiable for any infrastructure server.

| Interface | IP Address | Type | Purpose |
|---|---|---|---|
| Ethernet0 | 192.168.10.132 | DHCP | NAT — Internet access |
| Ethernet1 | 192.168.10.10 | Static | Host-Only — Internal domain |
| Loopback | 127.0.0.1 | Static | Self DNS reference |

*Ref 2: PowerShell output confirming dual NIC configuration with static IP on Ethernet1*  

<img width="965" height="157" alt="IPs" src="https://github.com/user-attachments/assets/2a1ce175-8bdf-4d6b-8242-696e0f17f351" />


---

#### Step 3 — AD DS Role Installation & Domain Controller Promotion

Installed the Active Directory Domain Services and DNS Server roles via Server Manager. Promoted DC01 to a Domain Controller by creating a new forest with the root domain `efrenlab.com`. Forest and domain functional levels were set to Windows Server 2016(a requirement for Microsoft Entra Connect compatibility).

| Property | Value |
|---|---|
| Domain Name (FQDN) | efrenlab.com |
| NetBIOS Name | EFRENLAB |
| Forest Functional Level | Windows Server 2016 |
| Domain Functional Level | Windows Server 2016 |
| DC Hostname | DC01 |
| DC Roles | Primary Domain Controller, DNS Server |
| DNS Self Reference | 127.0.0.1 |

*Ref 3: Active Directory Users and Computers showing efrenlab.com domain after promotion*  

<img width="555" height="212" alt="aduc" src="https://github.com/user-attachments/assets/9b0cf070-90a1-4125-a4de-d4c3194d5495" />


---

#### Step 4 — Organizational Unit Hierarchy

Created a professional OU structure using PowerShell. All custom OUs are prefixed with `_` to sort them above default AD containers in the ADUC console. Accidental deletion protection was enabled on all OUs.

```
efrenlab.com
├── _Computers
├── _Employees
│   ├── Finance
│   ├── HR
│   └── IT
└── _ServiceAccounts
```

**PowerShell commands used:**

```powershell
$domain = "DC=efrenlab,DC=com"

New-ADOrganizationalUnit -Name "_Employees"       -Path $domain -ProtectedFromAccidentalDeletion $true
New-ADOrganizationalUnit -Name "_ServiceAccounts" -Path $domain -ProtectedFromAccidentalDeletion $true
New-ADOrganizationalUnit -Name "_Computers"       -Path $domain -ProtectedFromAccidentalDeletion $true

$employeesOU = "OU=_Employees,$domain"
New-ADOrganizationalUnit -Name "IT"      -Path $employeesOU -ProtectedFromAccidentalDeletion $true
New-ADOrganizationalUnit -Name "HR"      -Path $employeesOU -ProtectedFromAccidentalDeletion $true
New-ADOrganizationalUnit -Name "Finance" -Path $employeesOU -ProtectedFromAccidentalDeletion $true
```

*Ref 4: ADUC showing full OU hierarchy expanded under efrenlab.com*  

<img width="553" height="457" alt="OUs" src="https://github.com/user-attachments/assets/7412a8cc-9f3a-4502-a16e-00e84292ef27" />


---

#### Step 5 — User Account Creation

Created 7 realistic test users across the three department OUs using PowerShell. Users were given proper attributes including display name, UPN, job title, and department to simulate a real enterprise directory and ensure Entra Connect syncs metadata correctly alongside identity objects.

**Naming Convention:**

| Property | Format | Example |
|---|---|---|
| SAMAccountName | FirstinitialLastname | emartinez |
| UPN | firstname.lastname@efrenlab.com | emartinez@efrenlab.com |
| Display Name | Firstname Lastname | Efren Martinez |

**Users Created:**

| Display Name | Username | Department | Title | OU |
|---|---|---|---|---|
| Efren Martinez | emartinez | IT | IT Manager | IT |
| Bob Stevens | bstevens | IT | Systems Administrator | IT |
| Carol Dean | cdean | IT | Help Desk Technician | IT |
| David Mills | dmills | HR | HR Manager | HR |
| Emma Clarke | eclarke | HR | HR Coordinator | HR |
| Frank Nguyen | fnguyen | Finance | Finance Manager | Finance |
| Grace Lee | glee | Finance | Financial Analyst | Finance |

*Ref 5: ADUC showing users listed under each department OU*  

<img width="1066" height="302" alt="users1" src="https://github.com/user-attachments/assets/336826f2-591c-4a3a-9615-a8eda971a853" />
<img width="1051" height="302" alt="users2" src="https://github.com/user-attachments/assets/782886ea-0b86-4d8d-83e1-190c1c383be3" />
<img width="1093" height="297" alt="users3" src="https://github.com/user-attachments/assets/333f449d-0def-445a-9c69-95d1247eac26" />  

*Ref 6: PowerShell verification of all 7 users with Department and Title*  

<img width="641" height="291" alt="ps users" src="https://github.com/user-attachments/assets/b11ffbd1-e7df-4f78-9152-0a77dc1056fb" />


---

#### Step 6 — Service Account Creation

Created a dedicated service account `svc_entrasync` in the `_ServiceAccounts` OU for use by Microsoft Entra Connect. Following the Principle of Least Privilege, the account was granted only the minimum permissions required for directory synchronization and configured to be excluded from cloud sync via OU filtering.

| Property | Value |
|---|---|
| SAMAccountName | svc_entrasync |
| UPN | svc_entrasync@efrenlab.com |
| Location | OU=_ServiceAccounts,DC=efrenlab,DC=com |
| Password Never Expires | True |
| Cannot Change Password | True |
| Group Membership | Domain Users only |
| Purpose | Entra Connect directory synchronization |
| Synced to Entra ID | ❌ Excluded via OU filter |

**Permissions Granted:**
- Replicating Directory Changes (`1131f6aa-9c07-11d1-f79f-00c04fc2dcd2`)
- Replicating Directory Changes All (`1131f6ab-9c07-11d1-f79f-00c04fc2dcd2`)

*Ref 7: PowerShell output verifying svc_entrasync properties*  

<img width="1487" height="217" alt="ps serv" src="https://github.com/user-attachments/assets/cb8ff4f6-6267-4566-8497-1f3068213680" />


*Ref 8: PowerShell output confirming Domain Users only group membership*  

<img width="857" height="111" alt="ps serv2" src="https://github.com/user-attachments/assets/ee80913e-0c74-439d-bd7d-5f3bc7595edc" />


---

#### Step 7 — DNS Forwarders & AD Recycle Bin

Configured DNS forwarders to allow DC01 to resolve external Microsoft endpoints required by Entra Connect (`login.microsoftonline.com`, `provisioningapi.microsoftonline.com`). Enabled the AD Recycle Bin to protect against accidental object deletion — allowing full object restoration including all attributes within 180 days.

```powershell
# Configure DNS Forwarders
Add-DnsServerForwarder -IPAddress 8.8.8.8
Add-DnsServerForwarder -IPAddress 1.1.1.1

# Enable AD Recycle Bin
Enable-ADOptionalFeature "Recycle Bin Feature" `
  -Scope ForestOrConfigurationSet `
  -Target "efrenlab.com" `
  -Confirm:$false
```

*Ref 9: PowerShell output confirming DNS forwarders configured*  

<img width="448" height="175" alt="forward" src="https://github.com/user-attachments/assets/579c827d-cf78-4823-ae02-c37fa78d16b4" />


---

### Phase 2 — Azure & Entra ID Configuration

---

#### Step 8 — Entra ID Tenant Setup & Cloud Admin Account

Located the Microsoft Entra ID tenant associated with the Azure free trial. Created a dedicated cloud-only Global Administrator account to follow security best practices — isolating billing account credentials from administrative operations.

| Property | Value |
|---|---|
| Primary Domain | <TENANT_DOMAIN> |
| Tenant ID | [Your Tenant ID] |
| Subscription Type | Azure Free Trial |
| Global Admin Account | <ADMIN_ACCOUNT> |

**UPN Mapping — On-Premises to Cloud:**

| On-Premises UPN | Cloud UPN |
|---|---|
| emartinez@efrenlab.com | emartinez@<TENANT_DOMAIN> |
| bstevens@efrenlab.com | bstevens@<TENANT_DOMAIN> |
| cdean@efrenlab.com | cdean@<TENANT_DOMAIN> |
| dmills@efrenlab.com | dmills@<TENANT_DOMAIN> |
| eclarke@efrenlab.com | eclarke@<TENANT_DOMAIN> |
| fnguyen@efrenlab.com | fnguyen@<TENANT_DOMAIN> |
| glee@efrenlab.com | glee@<TENANT_DOMAIN> |

*Ref 10: Microsoft Entra ID Overview page showing tenant domain and Tenant ID*  

<img width="1627" height="638" alt="entra" src="https://github.com/user-attachments/assets/b9b1b7ee-bf79-4d58-acc0-c3e538d5def5" />


*Ref 11: Entra ID Users page showing cloudadmin with Global Administrator role assigned*  

<img width="2547" height="415" alt="cloudadmin" src="https://github.com/user-attachments/assets/c3ae3e85-ed6c-4df0-8ce2-eee0824128cb" />


---

### Phase 3 — Microsoft Entra Connect Sync

---

#### Step 9 — Prerequisites Verification

Before installing Entra Connect, verified DC01 met all technical requirements including PowerShell version, .NET Framework version, TLS support, and network connectivity to Microsoft authentication endpoints.

| Check | Result | Status |
|---|---|---|
| PowerShell Version | 5.1.20348 | ✅ |
| .NET Framework | 4.8 (528449) | ✅ |
| TLS Support | SystemDefault | ✅ |
| login.microsoftonline.com:443 | TcpTestSucceeded | ✅ |
| provisioningapi.microsoftonline.com:443 | TcpTestSucceeded | ✅ |

> **Note:** PingSucceeded showed False for both Microsoft endpoints — this is expected as Microsoft blocks ICMP ping by design. TcpTestSucceeded: True on port 443 confirms HTTPS connectivity.

*Ref 12: PowerShell prerequisites check output*  

<img width="317" height="266" alt="ps version" src="https://github.com/user-attachments/assets/f9d49ca3-b7f4-43b3-be48-6ecb02d75dda" />


---

#### Step 10 — Entra Connect Download & Installation

Downloaded Microsoft Entra Connect Sync from the Microsoft Entra Admin Center. Note: As of recent updates, new versions are no longer available on the Microsoft Download Center and must be downloaded from entra.microsoft.com → Microsoft Entra Connect → Get Started → Manage tab.

Installed using the **Custom** path (not Express) to maintain full control over OU filtering, sync scope, and configuration decisions.

*Ref 13: Entra Connect installer — Install Required Components screen*  

<img width="887" height="630" alt="install" src="https://github.com/user-attachments/assets/884f63d8-fc80-4675-a558-204f341e151e" />



*Ref 14: Entra Connect — User Sign-In screen showing Password Hash Sync selected*  

<img width="890" height="627" alt="sign in" src="https://github.com/user-attachments/assets/036e2d0d-2d8a-48cf-af3c-756efcef0df7" />

---

#### Step 11 — Entra Connect OU Filtering Configuration

Configured OU-level filtering to control exactly which objects sync to Entra ID. This is a critical security configuration — it prevents service accounts and computer objects from being exposed in the cloud directory.

| OU | Sync Status | Reason |
|---|---|---|
| _Employees\IT | ✅ Included | User accounts — sync required |
| _Employees\HR | ✅ Included | User accounts — sync required |
| _Employees\Finance | ✅ Included | User accounts — sync required |
| _ServiceAccounts | ❌ Excluded | Prevents svc_entrasync from syncing to cloud |
| _Computers | ❌ Excluded | Computer objects not needed in Entra ID |
| Domain Controllers | ❌ Excluded | DC objects never synced |
| All other default containers | ❌ Excluded | Not required for this migration |

*Ref 15: Entra Connect Domain and OU Filtering screen showing selected OUs*  

<img width="890" height="626" alt="ou filter" src="https://github.com/user-attachments/assets/2678f40b-c639-4be6-8efb-96976f6b7a28" />


---

#### Step 12 — Entra Connect Configuration Complete & Sync Verification

Completed the Entra Connect configuration wizard. Triggered an initial sync cycle via PowerShell and verified all 7 users appeared in the Entra ID portal with Source = Windows Server AD, confirming the sync pipeline was functioning correctly end-to-end.

```powershell
Import-Module ADSync
Start-ADSyncSyncCycle -PolicyType Initial
Get-ADSyncScheduler | Select-Object SyncCycleEnabled, NextSyncCyclePolicyType
```

**Sync Results:**

| Display Name | Source | Status |
|---|---|---|
| Efren Martinez | Windows Server AD | ✅ Synced |
| Bob Stevens | Windows Server AD | ✅ Synced |
| Carol Dean | Windows Server AD | ✅ Synced |
| David Mills | Windows Server AD | ✅ Synced |
| Emma Clarke | Windows Server AD | ✅ Synced |
| Frank Nguyen | Windows Server AD | ✅ Synced |
| Grace Lee | Windows Server AD | ✅ Synced |
| svc_entrasync | N/A | ❌ Correctly excluded |

*Ref 16: Entra Connect "Configuration complete" screen*  

<img width="888" height="628" alt="config complete " src="https://github.com/user-attachments/assets/57a72712-791b-407a-bb37-8869b606ea83" />


*Ref 17: Entra ID Users page showing all 7 synced users with Source = Windows Server AD*  

<img width="1392" height="877" alt="azure users" src="https://github.com/user-attachments/assets/c5ab9202-259e-40cd-ac19-8783b38016d6" />


---

### Phase 4 — Azure RBAC & MFA

---

#### Step 13 — Azure Resource Group & RBAC Role Assignments

Created a Resource Group (`rg-efrenlab-prod`) as the scope for RBAC assignments. Assigned roles to all 7 synced users based on job function following the Principle of Least Privilege — IT staff received Contributor access for resource management while business department users received Reader access for visibility without write permissions.

| User | Role | Scope |
|---|---|---|
| Efren Martinez | Contributor | rg-efrenlab-prod |
| Bob Stevens | Contributor | rg-efrenlab-prod |
| Carol Dean | Reader | rg-efrenlab-prod |
| David Mills | Reader | rg-efrenlab-prod |
| Emma Clarke | Reader | rg-efrenlab-prod |
| Frank Nguyen | Reader | rg-efrenlab-prod |
| Grace Lee | Reader | rg-efrenlab-prod |

*Ref 18: Azure Portal — rg-efrenlab-prod resource group overview*  

<img width="2511" height="320" alt="rg" src="https://github.com/user-attachments/assets/45a12952-4c8d-4e87-861a-1e8f3f134128" />


*Ref 19: Access Control (IAM) → Role assignments showing all 7 users with their roles*  

<img width="1475" height="603" alt="IAM" src="https://github.com/user-attachments/assets/17d97252-d1a7-425a-911a-3051897905a8" />


---

#### Step 14 — Multi-Factor Authentication via Security Defaults

Enabled MFA across the entire tenant using Entra ID Security Defaults — Microsoft's free baseline security policy. Tested MFA registration end-to-end as Efren Martinez using Microsoft Authenticator push notification.

| Protection | Status |
|---|---|
| MFA required for all administrators | ✅ Enforced |
| MFA registration required for all users | ✅ Enforced (14 day grace period) |
| Legacy authentication protocols blocked | ✅ Enforced |
| MFA required for Azure management | ✅ Enforced |

**MFA Test Results:**

| Property | Value |
|---|---|
| Test Account | emartinez@<TENANT_DOMAIN> |
| MFA Method | Microsoft Authenticator — Push Notification |
| Registration | ✅ Successful |
| Sign-in with MFA | ✅ Successful |

*Ref 20: Entra ID Security Defaults blade showing toggle Enabled*  

<img width="533" height="430" alt="sec" src="https://github.com/user-attachments/assets/e96cc12d-a3d4-48e6-9b9a-43038999c529" />


---

### Phase 5 — Azure Migrate & Cloud DC (Next Steps)

---

#### Step 15 — Azure Migrate Assessment (Planned)

> 🟡 *This phase is currently in progress. Screenshots and results will be added upon completion.*

**Planned Actions:**
- Create Azure Migrate project `efrenlab-migration` in `rg-efrenlab-prod`
- Import DC01 server details via CSV discovery method
- Generate Azure VM readiness assessment for DC01
- Review recommended VM size, readiness status, and estimated monthly cost
- Document and screenshot assessment results

---

## Lessons Learned & Troubleshooting

| Issue Encountered | Root Cause | Resolution |
|---|---|---|
| `New-ADUser: Server unwilling to process request` | Domain DN was `DC=efrenlab,DC=com` not `DC=corp,DC=local` as initially configured | Ran `(Get-ADDomain).DistinguishedName` to confirm actual DN — updated all PowerShell scripts |
| Entra Connect not on Microsoft Download Center | Microsoft moved releases exclusively to Entra Admin Center | Downloaded from entra.microsoft.com → Microsoft Entra Connect → Get Started → Manage tab |
| UPN suffix warning in Entra Connect wizard | `efrenlab.com` is not a verified domain in the Entra ID tenant | Checked "Continue without matching all UPN suffixes to verified domains" — expected for unowned domains |
| PingSucceeded: False on connectivity test | Microsoft blocks ICMP ping on all their endpoints by design | Confirmed `TcpTestSucceeded: True` on port 443 — HTTPS connectivity confirmed, ping result is irrelevant |
| Tenant primary domain too long for practical use | Original tenant was created with a Gmail-derived name | Attempted to create new workforce tenant, feature moved in Azure Portal. Proceeded with existing tenant |

---

## Key Technical Decisions

| Decision | Rationale |
|---|---|
| Custom install over Express in Entra Connect | Express syncs all OUs with no filtering while Custom allows OU exclusions and full configuration visibility |
| Password Hash Sync over Pass-through Authentication | PHS is simpler, more resilient, widely deployed, and doesn't require on-prem infrastructure for authentication |
| Let Azure manage source anchor | Modern recommended approach using mS-DS-ConsistencyGuid for stable, portable cloud identity linking |
| Excluded _ServiceAccounts OU from sync | Service accounts should never exist as cloud identities. Security best practice and least privilege |
| Security Defaults over per-user MFA | Enforces consistent MFA baseline across all users without requiring Entra ID P1 licensing |
| Separate cloudadmin account for Azure administration | Isolates billing credentials from admin operations and mirrors enterprise security practices |
| Static IP on DC01 (192.168.x.x) | Domain Controllers require predictable addresses for DNS, Kerberos auth, and AD replication stability |

---

## 👋 About Me

I'm currently a Helpdesk Engineer transitioning into Azure Cloud Engineering, any feedback would be greatly appreciated!

Feel free to connect with me:

- LinkedIn: https://www.linkedin.com/in/efren-martinez-it/
- GitHub: https://github.com/The-Efren
