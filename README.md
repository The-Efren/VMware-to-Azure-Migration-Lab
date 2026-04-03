# VMware to Azure Cloud Migration Lab

## Objective

This lab project aimed to establish a hybrid cloud identity environment simulating a real-world enterprise migration from on-premises Active Directory to Microsoft Azure. The primary focus was to deploy and configure Active Directory Domain Services, synchronize identities to Microsoft Entra ID using Entra Connect, and implement cloud security controls including Role-Based Access Control (RBAC) and Multi-Factor Authentication (MFA). This hands-on experience was designed to deepen my understanding of hybrid identity architecture, cloud migration methodology, and Azure administration.

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
> 📸 Insert screenshot of VMware VM settings → Network Adapter configuration here

---

#### Step 2 — Static IP Configuration

Configured a static IP address on the Host-Only NIC (Ethernet1) to ensure the Domain Controller always has a predictable, consistent address. Dynamic IPs on a DC would break DNS resolution, Kerberos authentication, and AD replication — making static assignment non-negotiable for any infrastructure server.

| Interface | IP Address | Type | Purpose |
|---|---|---|---|
| Ethernet0 | 192.168.10.132 | DHCP | NAT — Internet access |
| Ethernet1 | 192.168.10.10 | Static | Host-Only — Internal domain |
| Loopback | 127.0.0.1 | Static | Self DNS reference |

*Ref 2: PowerShell output confirming dual NIC configuration with static IP on Ethernet1*
> 📸 Insert screenshot of PowerShell output: `Get-NetIPAddress -AddressFamily IPv4 | Select-Object InterfaceAlias, IPAddress`

---

#### Step 3 — AD DS Role Installation & Domain Controller Promotion

Installed the Active Directory Domain Services and DNS Server roles via Server Manager. Promoted DC01 to a Domain Controller by creating a new forest with the root domain `efrenlab.com`. Forest and domain functional levels were set to Windows Server 2016 — a requirement for Microsoft Entra Connect compatibility.

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
> 📸 Insert screenshot of ADUC (dsa.msc) showing efrenlab.com in the left panel

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
> 📸 Insert screenshot of ADUC with the full OU tree expanded showing _Computers, _Employees (with IT, HR, Finance), and _ServiceAccounts

---

#### Step 5 — User Account Creation

Created 7 realistic test users across the three department OUs using PowerShell. Users were given proper attributes including display name, UPN, job title, and department to simulate a real enterprise directory and ensure Entra Connect syncs metadata correctly alongside identity objects.

**Naming Convention:**

| Property | Format | Example |
|---|---|---|
| SAMAccountName | firstname.lastname | alice.carter |
| UPN | firstname.lastname@efrenlab.com | alice.carter@efrenlab.com |
| Display Name | Firstname Lastname | Alice Carter |

**Users Created:**

| Display Name | Username | Department | Title | OU |
|---|---|---|---|---|
| Alice Carter | alice.carter | IT | IT Manager | IT |
| Bob Stevens | bob.stevens | IT | Systems Administrator | IT |
| Carol Dean | carol.dean | IT | Help Desk Technician | IT |
| David Mills | david.mills | HR | HR Manager | HR |
| Emma Clarke | emma.clarke | HR | HR Coordinator | HR |
| Frank Nguyen | frank.nguyen | Finance | Finance Manager | Finance |
| Grace Lee | grace.lee | Finance | Financial Analyst | Finance |

*Ref 5: ADUC showing users listed under each department OU*
> 📸 Insert screenshot of ADUC showing IT, HR, and Finance OUs with users visible inside each

*Ref 6: PowerShell verification of all 7 users with Department and Title*
> 📸 Insert screenshot of PowerShell output: `Get-ADUser -Filter * -Properties Department, Title | Select-Object Name, SamAccountName, Department, Title | Sort-Object Department`

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
> 📸 Insert screenshot of PowerShell output: `Get-ADUser -Identity "svc_entrasync" -Properties * | Select-Object Name, SamAccountName, UserPrincipalName, PasswordNeverExpires, Enabled`

*Ref 8: PowerShell output confirming Domain Users only group membership*
> 📸 Insert screenshot of PowerShell output: `Get-ADPrincipalGroupMembership "svc_entrasync" | Select-Object Name`

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
> 📸 Insert screenshot of PowerShell output: `Get-DnsServerForwarder`

---

### Phase 2 — Azure & Entra ID Configuration

---

#### Step 8 — Entra ID Tenant Setup & Cloud Admin Account

Located the Microsoft Entra ID tenant associated with the Azure free trial. Created a dedicated cloud-only Global Administrator account to follow security best practices — isolating billing account credentials from administrative operations.

| Property | Value |
|---|---|
| Primary Domain | efrenmartineztechgmail.onmicrosoft.com |
| Tenant ID | [Your Tenant ID] |
| Subscription Type | Azure Free Trial |
| Global Admin Account | cloudadmin@efrenmartineztechgmail.onmicrosoft.com |

**UPN Mapping — On-Premises to Cloud:**

| On-Premises UPN | Cloud UPN |
|---|---|
| alice.carter@efrenlab.com | alice.carter@efrenmartineztechgmail.onmicrosoft.com |
| bob.stevens@efrenlab.com | bob.stevens@efrenmartineztechgmail.onmicrosoft.com |
| carol.dean@efrenlab.com | carol.dean@efrenmartineztechgmail.onmicrosoft.com |
| david.mills@efrenlab.com | david.mills@efrenmartineztechgmail.onmicrosoft.com |
| emma.clarke@efrenlab.com | emma.clarke@efrenmartineztechgmail.onmicrosoft.com |
| frank.nguyen@efrenlab.com | frank.nguyen@efrenmartineztechgmail.onmicrosoft.com |
| grace.lee@efrenlab.com | grace.lee@efrenmartineztechgmail.onmicrosoft.com |

*Ref 10: Microsoft Entra ID Overview page showing tenant domain and Tenant ID*
> 📸 Insert screenshot of entra.microsoft.com → Overview page showing primary domain and Tenant ID

*Ref 11: Entra ID Users page showing cloudadmin with Global Administrator role assigned*
> 📸 Insert screenshot of cloudadmin user properties showing Global Administrator role

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
> 📸 Insert screenshot of PowerShell output from the prerequisites check commands

---

#### Step 10 — Entra Connect Download & Installation

Downloaded Microsoft Entra Connect Sync from the Microsoft Entra Admin Center. Note: As of recent updates, new versions are no longer available on the Microsoft Download Center and must be downloaded from entra.microsoft.com → Microsoft Entra Connect → Get Started → Manage tab.

Installed using the **Custom** path (not Express) to maintain full control over OU filtering, sync scope, and configuration decisions.

*Ref 13: Entra Connect installer — Install Required Components screen*
> 📸 Insert screenshot of the "Install required components" screen with all options unchecked

*Ref 14: Entra Connect — User Sign-In screen showing Password Hash Sync selected*
> 📸 Insert screenshot of User Sign-In screen with Password Hash Synchronization selected

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
> 📸 Insert screenshot of the OU filtering screen showing _Employees checked and _ServiceAccounts unchecked

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
| Alice Carter | Windows Server AD | ✅ Synced |
| Bob Stevens | Windows Server AD | ✅ Synced |
| Carol Dean | Windows Server AD | ✅ Synced |
| David Mills | Windows Server AD | ✅ Synced |
| Emma Clarke | Windows Server AD | ✅ Synced |
| Frank Nguyen | Windows Server AD | ✅ Synced |
| Grace Lee | Windows Server AD | ✅ Synced |
| svc_entrasync | N/A | ❌ Correctly excluded |

*Ref 16: Entra Connect "Configuration complete" screen*
> 📸 Insert screenshot of the "Configuration complete" success screen

*Ref 17: Entra ID Users page showing all 7 synced users with Source = Windows Server AD*
> 📸 Insert screenshot of entra.microsoft.com → Users → All Users showing synced users with Source column

---

### Phase 4 — Azure RBAC & MFA

---

#### Step 13 — Azure Resource Group & RBAC Role Assignments

Created a Resource Group (`rg-efrenlab-prod`) as the scope for RBAC assignments. Assigned roles to all 7 synced users based on job function following the Principle of Least Privilege — IT staff received Contributor access for resource management while business department users received Reader access for visibility without write permissions.

| User | Role | Scope |
|---|---|---|
| Alice Carter | Contributor | rg-efrenlab-prod |
| Bob Stevens | Contributor | rg-efrenlab-prod |
| Carol Dean | Reader | rg-efrenlab-prod |
| David Mills | Reader | rg-efrenlab-prod |
| Emma Clarke | Reader | rg-efrenlab-prod |
| Frank Nguyen | Reader | rg-efrenlab-prod |
| Grace Lee | Reader | rg-efrenlab-prod |

*Ref 18: Azure Portal — rg-efrenlab-prod resource group overview*
> 📸 Insert screenshot of rg-efrenlab-prod overview page in portal.azure.com

*Ref 19: Access Control (IAM) → Role assignments showing all 7 users with their roles*
> 📸 Insert screenshot of rg-efrenlab-prod → Access Control (IAM) → Role assignments tab

---

#### Step 14 — Multi-Factor Authentication via Security Defaults

Enabled MFA across the entire tenant using Entra ID Security Defaults — Microsoft's free baseline security policy. Tested MFA registration end-to-end as Alice Carter using Microsoft Authenticator push notification.

| Protection | Status |
|---|---|
| MFA required for all administrators | ✅ Enforced |
| MFA registration required for all users | ✅ Enforced (14 day grace period) |
| Legacy authentication protocols blocked | ✅ Enforced |
| MFA required for Azure management | ✅ Enforced |

**MFA Test Results:**

| Property | Value |
|---|---|
| Test Account | alice.carter@efrenmartineztechgmail.onmicrosoft.com |
| MFA Method | Microsoft Authenticator — Push Notification |
| Registration | ✅ Successful |
| Sign-in with MFA | ✅ Successful |

*Ref 20: Entra ID Security Defaults blade showing toggle Enabled*
> 📸 Insert screenshot of entra.microsoft.com → Identity → Overview → Properties → Manage Security Defaults showing Enabled

*Ref 21: MFA registration prompt displayed when signing in as alice.carter*
> 📸 Insert screenshot of the "More information required" MFA registration prompt

*Ref 22: Successful Azure portal login after MFA approval*
> 📸 Insert screenshot of portal.azure.com dashboard after successful MFA-protected login as alice.carter

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

*Ref 23: Azure Migrate project overview — efrenlab-migration*
> 📸 Screenshot to be added upon completion

*Ref 24: DC01 assessment results showing Azure readiness and recommended VM size*
> 📸 Screenshot to be added upon completion

---

#### Step 16 — Azure Virtual Network (Planned)

> 🟡 *This phase is currently in progress.*

**Planned Configuration:**

| Property | Value |
|---|---|
| VNet Name | vnet-efrenlab-prod |
| Address Space | 10.0.0.0/16 |
| Subnet Name | snet-identity |
| Subnet Range | 10.0.1.0/24 |
| Region | East US |
| Resource Group | rg-efrenlab-prod |

*Ref 25: Azure VNet configuration overview*
> 📸 Screenshot to be added upon completion

---

#### Step 17 — Cloud Domain Controller DC02 (Planned)

> 🟡 *This phase is currently in progress.*

**Planned Configuration:**

| Property | Value |
|---|---|
| VM Name | DC02 |
| VM Size | Standard_B2s |
| OS | Windows Server 2022 |
| Role | Additional Domain Controller — efrenlab.com |
| VNet | vnet-efrenlab-prod |
| Subnet | snet-identity |
| Static Private IP | 10.0.1.10 |
| Resource Group | rg-efrenlab-prod |

*Ref 26: DC02 Azure VM overview page*
> 📸 Screenshot to be added upon completion

*Ref 27: DC02 promoted as Domain Controller — ADUC showing both DC01 and DC02*
> 📸 Screenshot to be added upon completion

---

## Lessons Learned & Troubleshooting

| Issue Encountered | Root Cause | Resolution |
|---|---|---|
| `New-ADUser: Server unwilling to process request` | Domain DN was `DC=efrenlab,DC=com` not `DC=corp,DC=local` as initially configured | Ran `(Get-ADDomain).DistinguishedName` to confirm actual DN — updated all PowerShell scripts |
| Entra Connect not on Microsoft Download Center | Microsoft moved releases exclusively to Entra Admin Center | Downloaded from entra.microsoft.com → Microsoft Entra Connect → Get Started → Manage tab |
| UPN suffix warning in Entra Connect wizard | `efrenlab.com` is not a verified domain in the Entra ID tenant | Checked "Continue without matching all UPN suffixes to verified domains" — expected for unowned domains |
| PingSucceeded: False on connectivity test | Microsoft blocks ICMP ping on all their endpoints by design | Confirmed `TcpTestSucceeded: True` on port 443 — HTTPS connectivity confirmed, ping result is irrelevant |
| Tenant primary domain too long for practical use | Original tenant was created with a Gmail-derived name | Attempted to create new workforce tenant — feature moved in Azure Portal. Proceeded with existing tenant |

---

## Key Technical Decisions

| Decision | Rationale |
|---|---|
| Custom install over Express in Entra Connect | Express syncs all OUs with no filtering — Custom allows OU exclusions and full configuration visibility |
| Password Hash Sync over Pass-through Authentication | PHS is simpler, more resilient, widely deployed, and doesn't require on-prem infrastructure for auth |
| Let Azure manage source anchor | Modern recommended approach using mS-DS-ConsistencyGuid for stable, portable cloud identity linking |
| Excluded _ServiceAccounts OU from sync | Service accounts should never exist as cloud identities — security best practice and least privilege |
| Security Defaults over per-user MFA | Enforces consistent MFA baseline across all users without requiring Entra ID P1 licensing |
| Separate cloudadmin account for Azure administration | Isolates billing credentials from admin operations — mirrors enterprise security practices |
| Static IP on DC01 (192.168.10.10) | Domain Controllers require predictable addresses for DNS, Kerberos auth, and AD replication stability |

---

*This project is part of a cloud engineering portfolio demonstrating hybrid identity, Azure administration, and cloud migration skills. Built using VMware Workstation Pro and Microsoft Azure free trial.*
