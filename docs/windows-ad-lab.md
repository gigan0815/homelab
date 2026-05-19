# Windows Server 2022 AD Lab

This document describes the Windows Server lab environment in the homelab. The lab provides hands-on practice with Microsoft Server, Active Directory Domain Services, DNS, and Group Policy, complementing the existing Linux-focused setup.

## Purpose

- Practice Microsoft Server administration tasks hands-on.
- Build PowerShell muscle for AD operations, not just GUI clicking.
- Provide a realistic workload running on Proxmox alongside the existing Linux LXCs.

## Scope

Currently single-DC, single-domain. Designed to grow: future additions (member server, second DC, domain-joined client) fit cleanly into the existing OU structure without restructuring.

## VM specifications

| Setting | Value |
| --- | --- |
| VMID | 200 |
| Hostname | dc01 |
| IP | 192.168.1.50 (DHCP with MAC reservation in UniFi) |
| OS | Windows Server 2022 Datacenter Evaluation, Desktop Experience |
| CPU | 2 vCPU, host type |
| RAM | 4 GB |
| Disk | 60 GB on `lvm` storage, VirtIO SCSI single, discard + iothread + SSD emulation |
| BIOS | OVMF (UEFI) with EFI disk and Secure Boot keys pre-enrolled |
| Machine | q35 |
| TPM | v2.0 |
| Network | virtio on vmbr0 |
| Locale | English UI, German locale, German keyboard |
| Guest Agent | QEMU Guest Agent installed |

Resource footprint is intentionally small. The DC handles directory and DNS for a lab population without strain.

## Network and identity decisions

The DC uses DHCP with a MAC-based reservation in UniFi, rather than a static IP set inside Windows. Both achieve the requirement "the DC's IP must not change," but the reservation pattern has a single source of truth for IP assignments, survives VM restore cleanly, and avoids scattered static configs across hosts.

DNS on the DC points to itself (127.0.0.1). External resolution goes through DNS forwarders 1.1.1.1 and 1.0.0.1.

Locale split (English UI, German locale, German keyboard) keeps error messages and documentation consistent with the system language while keeping daily input and formatting regional.

## Forest and domain

| Property | Value |
| --- | --- |
| Forest | homelab.local |
| Domain | homelab.local |
| NetBIOS name | HOMELAB |
| Forest functional level | Windows Server 2016 (WinThreshold) |
| Domain functional level | Windows Server 2016 (WinThreshold) |
| FSMO roles | All five held by dc01 |
| Global Catalog | dc01 |
| DNS | AD-integrated, zone `homelab.local`, forwarders 1.1.1.1 + 1.0.0.1 |

## OU structure

```
homelab.local
├── _Admin/                  privileged accounts (sorts to top alphabetically)
├── Corp/                    wrapper for all corporate objects
│   ├── Users/
│   │   ├── IT/
│   │   ├── Sales/
│   │   └── Support/
│   ├── Groups/
│   │   ├── Security/
│   │   └── Distribution/
│   ├── Workstations/
│   └── Servers/
└── Domain Controllers/      built-in
```

All OUs are created with `-ProtectedFromAccidentalDeletion $true`.

A `Corp` wrapper OU is used rather than placing departmental OUs at the domain root. This avoids collision with the built-in `CN=Users` container, and provides a single point of delegation if a junior admin needs rights over all corporate objects without touching DCs or built-in containers.

## Users

| SamAccountName | OU | Role |
| --- | --- | --- |
| ahenkel-adm | _Admin | Privileged admin (member of Domain Admins) |
| ahenkel | Corp/Users/IT | Daily-driver account, Department IT, Title Systemadministrator |
| aschmidt | Corp/Users/Sales | Test user, Department Sales |
| tbecker | Corp/Users/Support | Test user, Department Support |

The `-adm` suffix on the admin account follows the common convention of making privileged accounts visibly distinct from regular accounts in logs and reports. Daily AD work is done as `ahenkel-adm`, not as the built-in Administrator.

## Groups

| Name | Scope | Members | Purpose |
| --- | --- | --- | --- |
| GG-IT-Staff | Global | ahenkel | IT staff role group |
| GG-Sales-Staff | Global | aschmidt | Sales staff role group |
| DL-FileShare-Marketing-RW | DomainLocal | GG-Sales-Staff (nested) | Resource group, would grant RW on a Marketing share |

Naming convention: `GG-` prefix for Global groups, `DL-` for Domain Local. The scope is visible at a glance.

The Sales staff group is nested into the Marketing share group, demonstrating the AGDLP pattern (Account into Global, Global into Domain Local, Domain Local granted Permission). No actual file share exists yet; the structure is in place for when one is added.

## Group Policy

| GPO | Linked to | Effect |
| --- | --- | --- |
| GPO-Logon-Banner | OU=Workstations,OU=Corp ; OU=Servers,OU=Corp | Sets LegalNoticeCaption and LegalNoticeText, displaying a banner before login on domain-joined machines |

The GPO is Computer-side, so it only applies to computer objects. It is currently demonstrative since no member machines exist yet to apply it.

An HTML report of the GPO can be generated for audit and handover purposes:

```powershell
Get-GPOReport -Name "GPO-Logon-Banner" -ReportType Html -Path "$env:USERPROFILE\gpo-logon-banner.html"
```

## Operational commands

### Health checks

```powershell
# Confirm DC status, FSMO roles, and IP
Get-ADDomainController

# Verify all core AD services are running
Get-Service NTDS, DNS, KDC, ADWS, Netlogon | Format-Table Name, Status

# Confirm SYSVOL and NETLOGON shares are online
Get-SmbShare | Where-Object Name -in 'SYSVOL','NETLOGON'

# Test internal DNS resolution (DC should resolve itself)
Resolve-DnsName dc01.homelab.local

# Test external DNS resolution via forwarders
Resolve-DnsName google.com
```

### OU management

```powershell
# Create a new OU with accidental deletion protection
New-ADOrganizationalUnit -Name "X" -Path "OU=Parent,DC=homelab,DC=local" -ProtectedFromAccidentalDeletion $true

# List all OUs sorted by distinguished name (shows hierarchy clearly)
Get-ADOrganizationalUnit -Filter * | Select-Object Name, DistinguishedName | Sort-Object DistinguishedName
```

### User management

```powershell
# Prompt for a password without it appearing in plain text in the session
$pw = Read-Host -AsSecureString -Prompt "Password"

# Create a new user in the IT OU
New-ADUser -Name "Display Name" -SamAccountName "samname" -UserPrincipalName "user@homelab.local" `
  -Path "OU=IT,OU=Users,OU=Corp,DC=homelab,DC=local" `
  -AccountPassword $pw -Enabled $true -ChangePasswordAtLogon $false

# List all users in the domain with department and title attributes
Get-ADUser -Filter * -SearchBase "DC=homelab,DC=local" -Properties Department, Title
```

### Group management

```powershell
# Create a new Global security group in the Security OU
New-ADGroup -Name "GG-X" -SamAccountName "GG-X" -GroupCategory Security -GroupScope Global `
  -Path "OU=Security,OU=Groups,OU=Corp,DC=homelab,DC=local"

# Add a user (or group) to a group
Add-ADGroupMember -Identity "GG-X" -Members "samname"

# List current members of a group
Get-ADGroupMember -Identity "GG-X"
```

### GPO management

```powershell
# Create an empty GPO
New-GPO -Name "GPO-X" -Comment "Purpose"

# Link the GPO to an OU so it takes effect there
New-GPLink -Name "GPO-X" -Target "OU=Workstations,OU=Corp,DC=homelab,DC=local"

# Set a registry-based policy value inside the GPO
Set-GPRegistryValue -Name "GPO-X" -Key "HKLM\Software\..." -ValueName "X" -Type String -Value "Y"

# Show which GPOs are linked to a given OU (including inherited ones)
Get-GPInheritance -Target "OU=..." | Select-Object -ExpandProperty GpoLinks

# Generate an HTML report of all settings inside a GPO
Get-GPOReport -Name "GPO-X" -ReportType Html -Path "$env:USERPROFILE\report.html"
```

## Credentials

All stored in Bitwarden:

- `Homelab dc01 - Administrator` — built-in domain administrator
- `Homelab dc01 - DSRM` — Directory Services Restore Mode password
- `Homelab dc01 - lab users` — shared password for ahenkel, ahenkel-adm, aschmidt, tbecker (lab convenience, would be unique per user in production)

## Known gaps

- No member server yet. Spinning up a second Windows VM and domain-joining it would let the GPO take effect and demonstrate the full join workflow.
- No second DC. A dc02 would enable practice with AD replication, FSMO role transfer, and DC failure scenarios.
- No domain-joined Windows 10/11 client. Useful for testing user-side GPOs.
- dc01 not yet included in the Proxmox backup job.
- krbtgt password rotation is a security best practice (annual). Not yet on a schedule.

## Maintenance

- After significant changes to OUs, users, groups, or GPOs: update this document.
- Periodically run `Get-ADDomainController` and the service health check to confirm core AD services are running.
- The Windows Server 2022 evaluation license is valid for 180 days. Can be extended up to 5 times with `slmgr /rearm` (~3 years total). Note: only relevant if the lab is kept long-term; otherwise rebuild is faster.
