# Windows Server Full Lab (FINAL.LOCAL)

A complete Windows Server 2022 lab built on VMware Workstation. It covers Active Directory, Group Policy, DHCP with failover, DNS load balancing, file services, printing, Hyper-V Replica, and backup. Every step is documented with screenshots.

   Implemented hands-on by Amjad Sayer ALjuaid, based on a Windows Server training course, as part of practicing cloud and infrastructure fundamentals.
## Contents

1. [Lab topology](#lab-topology)
2. [Requirements checklist](#requirements-checklist)
3. [Implementation details and screenshots](#implementation-details-and-screenshots)
   - [Base network setup and Server Core](#1-base-network-setup-and-server-core)
   - [Active Directory](#2-active-directory-ous-users-and-groups)
   - [Additional Domain Controller](#3-server-core-as-additional-domain-controller-adc)
   - [Group Policy](#4-group-policy)
   - [DHCP and Failover](#5-dhcp-and-failover)
   - [IIS and DNS load balancing](#6-iis-and-dns-load-balancing)
   - [File services](#7-file-services-shared-folders-drive-maps-file-screening-and-quota)
   - [Printing](#8-printing)
   - [Hyper-V and Replica](#9-hyper-v-and-hyper-v-replica)
   - [Backup](#10-backup)

## Lab Topology

| Machine | Role | OS | IP |
|---|---|---|---|
| **PDC** | Primary Domain Controller, DNS, DHCP, IIS, File/Print, Hyper-V | Windows Server 2022 Standard (Desktop) | 192.168.1.2/24 |
| **CORE** (ADC) | Additional Domain Controller, DHCP failover partner, IIS, Hyper-V, backup target | Windows Server 2022 Datacenter (Server Core) | 192.168.1.3/24 |
| **Client-PC1** | Domain-joined client | Windows 10 | DHCP (192.168.1.40+) |

- **Domain:** `FINAL.LOCAL`
- **Gateway:** 192.168.1.1
- **Hypervisor:** VMware Workstation (nested virtualization enabled for Hyper-V)

## Requirements Checklist

- [x] Domain `FINAL.LOCAL` on server `PDC` (192.168.1.2/24, GW 192.168.1.1)
- [x] OUs for HR, Sales, Dev, IT
- [x] Users per department, one security group per department
- [x] Password policy: max age 60 days, min length 6, complexity on, history 3
- [x] Account lockout: 4 failed attempts, locked for 60 minutes
- [x] Remote Desktop enabled for all domain computers (GPO)
- [x] Remote Assistance for all computers, IT Group as helpers (GPO)
- [x] Block external storage, Task Manager, Control Panel (GPO)
- [x] DHCP scope 192.168.1.40 - 192.168.1.230, exclusion .80 - .85, 10-day lease
- [x] DHCP failover with Server Core; Core promoted to Additional Domain Controller (192.168.1.3)
- [x] IIS installed on PDC and Core (default page)
- [x] DNS round-robin load balancing for `www.final.local` (192.168.1.2 and 192.168.1.3)
- [x] Shared folders for Dev and HR (create/edit, no delete) with file screening (audio, video, executables)
- [x] Mapped network drives for Dev and HR
- [x] Public share mapped for all domain users with 2 GB quota
- [x] Shared printer, black and white only, available 9:00 AM - 4:00 PM
- [x] Active Directory Recycle Bin enabled
- [x] Hyper-V on PDC and Core, Core VM on PDC, Hyper-V Replica PDC -> Core
- [x] Client joined to the domain
- [x] Daily full server backup at 11:00 PM to the ADC

## Implementation Details and Screenshots

### 1. Base network setup and Server Core

- PDC uses the static address `192.168.1.2/24` with gateway `192.168.1.1`.
- The Server Core machine is configured with `sconfig`: computer name `core`, static IP `192.168.1.3/24`, DNS `192.168.1.2`, ping response and remote management enabled.
- The Windows 10 client is renamed `Client-PC1` and starts on DHCP.

#### PDC

![PDC - Server Manager dashboard with the installed roles (AD DS, DNS, File and Storage Services).](screenshots/001-pdc-server-manager.png)

*PDC - Server Manager dashboard with the installed roles (AD DS, DNS, File and Storage Services).*

![PDC - static IPv4 settings: 192.168.1.2 / 255.255.255.0, gateway 192.168.1.1, preferred DNS 192.168.1.2.](screenshots/002-pdc-static-ip.png)

*PDC - static IPv4 settings: 192.168.1.2 / 255.255.255.0, gateway 192.168.1.1, preferred DNS 192.168.1.2.*


#### Server Core (CORE)

![Server Core - initial `sconfig` menu (workgroup mode, default computer name).](screenshots/003-core-sconfig-menu.png)

*Server Core - initial `sconfig` menu (workgroup mode, default computer name).*

![Server Core - renaming the computer to `core` from `sconfig`.](screenshots/004-core-rename.png)

*Server Core - renaming the computer to `core` from `sconfig`.*

![Server Core - Network adapter settings: switching the adapter from DHCP to a static address.](screenshots/005-core-network-settings.png)

*Server Core - Network adapter settings: switching the adapter from DHCP to a static address.*

![Server Core - static IP 192.168.1.3 applied with subnet mask 255.255.255.0.](screenshots/006-core-static-ip.png)

*Server Core - static IP 192.168.1.3 applied with subnet mask 255.255.255.0.*

![Server Core - preferred DNS server set to the PDC (192.168.1.2).](screenshots/007-core-dns.png)

*Server Core - preferred DNS server set to the PDC (192.168.1.2).*

![Server Core - remote management enabled and server response to ping enabled.](screenshots/008-core-enable-ping.png)

*Server Core - remote management enabled and server response to ping enabled.*


#### Windows 10 client

![Windows 10 client - opening System Properties with `sysdm.cpl`.](screenshots/009-client-run-sysdm.png)

*Windows 10 client - opening System Properties with `sysdm.cpl`.*

![Windows 10 client - System Properties, Computer Name tab (default name, WORKGROUP).](screenshots/010-client-system-properties.png)

*Windows 10 client - System Properties, Computer Name tab (default name, WORKGROUP).*

![Windows 10 client - computer renamed to `Client-PC1`.](screenshots/011-client-rename.png)

*Windows 10 client - computer renamed to `Client-PC1`.*

![Windows 10 client - opening Network Connections with `ncpa.cpl`.](screenshots/012-client-run-ncpa.png)

*Windows 10 client - opening Network Connections with `ncpa.cpl`.*

![Windows 10 client - IPv4 set to obtain an IP address and DNS server automatically (DHCP).](screenshots/013-client-ipv4-auto.png)

*Windows 10 client - IPv4 set to obtain an IP address and DNS server automatically (DHCP).*


---

### 2. Active Directory: OUs, users and groups

- Domain: `FINAL.LOCAL`. One OU per department: **HR, Sales, Dev, IT** (plus a **Groups** OU).
- Example users: `ahmed.mohammed` (HR) and `sara.abdullah` (IT). Each user is added to the matching department group (`HR-Group`, `IT-Group`, `Dev-Group`).
- The **Active Directory Recycle Bin** is enabled from Active Directory Administrative Center (this cannot be undone).

#### Users and groups

![Active Directory Users and Computers showing the `FINAL.LOCAL` domain.](screenshots/014-aduc-domain.png)

*Active Directory Users and Computers showing the `FINAL.LOCAL` domain.*

![Creating Organizational Units (right-click the domain, New, Organizational Unit).](screenshots/015-aduc-new-ou.png)

*Creating Organizational Units (right-click the domain, New, Organizational Unit).*

![Creating user `ahmed.mohammed` inside the HR OU.](screenshots/016-new-user-ahmed.png)

*Creating user `ahmed.mohammed` inside the HR OU.*

![Setting the password for the new user.](screenshots/017-new-user-password.png)

*Setting the password for the new user.*

![Confirmation summary before the user is created.](screenshots/018-new-user-confirm.png)

*Confirmation summary before the user is created.*

![Right-click the user, then Add to a group.](screenshots/019-user-add-to-group.png)

*Right-click the user, then Add to a group.*

![Select Groups dialog.](screenshots/020-select-groups-dialog.png)

*Select Groups dialog.*

![Adding the user to the department group (`HR-Group`).](screenshots/021-select-hr-group.png)

*Adding the user to the department group (`HR-Group`).*

![Creating user `sara.abdullah` inside the IT OU.](screenshots/022-new-user-sara.png)

*Creating user `sara.abdullah` inside the IT OU.*


#### Active Directory Recycle Bin

![Active Directory Administrative Center - Enable Recycle Bin confirmation.](screenshots/102-recycle-bin-confirm.png)

*Active Directory Administrative Center - Enable Recycle Bin confirmation.*

![Recycle Bin enabling started for the forest (refresh AD Administrative Center).](screenshots/103-recycle-bin-enabled.png)

*Recycle Bin enabling started for the forest (refresh AD Administrative Center).*


---

### 3. Server Core as Additional Domain Controller (ADC)

- `core` is joined to `final.local`, then added to Server Manager on the PDC for remote management.
- AD DS is installed remotely and the server is promoted as an additional domain controller (replicating from `PDC.FINAL.LOCAL`).

#### Join the domain and manage it from the PDC

![Server Core - joining `final.local` with `sconfig` (Successfully joined domain).](screenshots/023-core-join-domain.png)

*Server Core - joining `final.local` with `sconfig` (Successfully joined domain).*

![Server Manager, Add Servers - searching Active Directory for `core`.](screenshots/036-add-servers-search.png)

*Server Manager, Add Servers - searching Active Directory for `core`.*

![`core` found (Windows Server 2022 Datacenter Evaluation) and selected.](screenshots/037-add-servers-found.png)

*`core` found (Windows Server 2022 Datacenter Evaluation) and selected.*

![All Servers view listing PDC (192.168.1.2) and CORE (192.168.1.3).](screenshots/038-all-servers.png)

*All Servers view listing PDC (192.168.1.2) and CORE (192.168.1.3).*

![Right-click CORE, then Add Roles and Features (managed remotely from the PDC).](screenshots/039-core-add-roles-menu.png)

*Right-click CORE, then Add Roles and Features (managed remotely from the PDC).*

![Add Roles and Features wizard with destination server `core.FINAL.LOCAL`.](screenshots/040-core-add-roles-wizard.png)

*Add Roles and Features wizard with destination server `core.FINAL.LOCAL`.*

![Selecting `core.FINAL.LOCAL` from the server pool.](screenshots/041-core-select-destination.png)

*Selecting `core.FINAL.LOCAL` from the server pool.*

![Selecting the Active Directory Domain Services role.](screenshots/042-core-select-adds.png)

*Selecting the Active Directory Domain Services role.*


#### Promote to Additional Domain Controller

![AD DS Configuration Wizard - "Add a domain controller to an existing domain" (`FINAL.LOCAL`) on target server `core.FINAL.LOCAL`.](screenshots/062-core-adds-deployment.png)

*AD DS Configuration Wizard - "Add a domain controller to an existing domain" (`FINAL.LOCAL`) on target server `core.FINAL.LOCAL`.*

![Additional options - replicate from `PDC.FINAL.LOCAL`.](screenshots/063-core-adds-replicate-from.png)

*Additional options - replicate from `PDC.FINAL.LOCAL`.*


---

### 4. Group Policy

| GPO | Purpose |
|---|---|
| Default Domain Policy | Password policy (60 days, min 6, complexity, history 3) and account lockout (4 attempts, 60 minutes) |
| RemoteDesktop | Allow Remote Desktop connections + predefined inbound firewall rules |
| AllowRemoteAssistance | Offer Remote Assistance with `FINAL\IT-Group` and Domain Admins as helpers + firewall rules |
| DisableControlPanel | Prohibit access to Control Panel and PC settings |
| RemoveTaskManager | Remove Task Manager |
| DisableExternalStorage | Deny access to removable storage classes |
| HR-MapDrive / Dev-Map / PublicFolder-Map | Mapped network drives (Group Policy Preferences) |
| Printers | Deploys the shared printer to users |

#### Remote Desktop

![Group Policy Management console for the forest `FINAL.LOCAL`.](screenshots/043-gpmc-console.png)

*Group Policy Management console for the forest `FINAL.LOCAL`.*

![Creating a GPO in the domain and linking it.](screenshots/044-gpo-create-and-link.png)

*Creating a GPO in the domain and linking it.*

![GPO `RemoteDesktop` opened in the Group Policy Management Editor.](screenshots/045-gpo-remotedesktop-editor.png)

*GPO `RemoteDesktop` opened in the Group Policy Management Editor.*

![Computer Configuration, Administrative Templates, Windows Components, Remote Desktop Services, Remote Desktop Session Host, Connections.](screenshots/046-gpo-rds-connections.png)

*Computer Configuration, Administrative Templates, Windows Components, Remote Desktop Services, Remote Desktop Session Host, Connections.*

![Selecting the policy "Allow users to connect remotely by using Remote Desktop Services".](screenshots/047-gpo-allow-remote-connections.png)

*Selecting the policy "Allow users to connect remotely by using Remote Desktop Services".*

![Policy set to Enabled.](screenshots/048-gpo-allow-remote-enabled.png)

*Policy set to Enabled.*

![Security node - Network Level Authentication setting.](screenshots/049-gpo-rds-security-nla.png)

*Security node - Network Level Authentication setting.*

![Windows Defender Firewall in the GPO - New Inbound Rule wizard.](screenshots/050-firewall-new-inbound-rule.png)

*Windows Defender Firewall in the GPO - New Inbound Rule wizard.*

![Predefined rule: Remote Desktop.](screenshots/051-firewall-predefined-remote-desktop.png)

*Predefined rule: Remote Desktop.*

![Resulting inbound rules for Remote Desktop (User Mode TCP/UDP and Shadow) - Enabled, Allow.](screenshots/052-firewall-remote-desktop-rules.png)

*Resulting inbound rules for Remote Desktop (User Mode TCP/UDP and Shadow) - Enabled, Allow.*


#### Password and account lockout policy

![Default Domain Policy, Password Policy - default values before the change.](screenshots/053-password-policy-before.png)

*Default Domain Policy, Password Policy - default values before the change.*

![Password Policy after the change: history 3, maximum age 60 days, minimum length 6, complexity enabled.](screenshots/054-password-policy-after.png)

*Password Policy after the change: history 3, maximum age 60 days, minimum length 6, complexity enabled.*

![Account Lockout Policy - default values before the change.](screenshots/055-lockout-policy-before.png)

*Account Lockout Policy - default values before the change.*

![Account Lockout Policy after the change: threshold 4 invalid attempts, duration 60 minutes, reset counter after 60 minutes.](screenshots/056-lockout-policy-after.png)

*Account Lockout Policy after the change: threshold 4 invalid attempts, duration 60 minutes, reset counter after 60 minutes.*


#### Block Control Panel, Task Manager and external storage

![Creating the `DisableControlPanel` GPO.](screenshots/057-gpo-disable-control-panel-new.png)

*Creating the `DisableControlPanel` GPO.*

![User Configuration, Control Panel - "Prohibit access to Control Panel and PC settings".](screenshots/058-gpo-prohibit-control-panel.png)

*User Configuration, Control Panel - "Prohibit access to Control Panel and PC settings".*

![User Configuration, System, Ctrl+Alt+Del Options - "Remove Task Manager" set to Enabled.](screenshots/059-gpo-remove-task-manager.png)

*User Configuration, System, Ctrl+Alt+Del Options - "Remove Task Manager" set to Enabled.*

![`DisableExternalStorage` GPO (security filtering: Authenticated Users).](screenshots/060-gpo-disable-external-storage.png)

*`DisableExternalStorage` GPO (security filtering: Authenticated Users).*

![Removable Storage Access - deny access for CD/DVD, WPD devices and all removable storage classes.](screenshots/061-gpo-removable-storage-access.png)

*Removable Storage Access - deny access for CD/DVD, WPD devices and all removable storage classes.*


#### Remote Assistance (IT Group as helpers)

![Add Roles and Features on the PDC - selecting the destination server.](screenshots/064-ra-select-destination.png)

*Add Roles and Features on the PDC - selecting the destination server.*

![Select features list.](screenshots/065-ra-features-list.png)

*Select features list.*

![Remote Assistance feature selected.](screenshots/066-ra-feature-selected.png)

*Remote Assistance feature selected.*

![Group Policy Objects in `FINAL.LOCAL` including the Remote Assistance GPO.](screenshots/067-ra-gpo-list.png)

*Group Policy Objects in `FINAL.LOCAL` including the Remote Assistance GPO.*

![Configure Offer Remote Assistance - Enabled, helpers: `FINAL\IT-Group` and `FINAL\Domain Admins`.](screenshots/068-ra-offer-helpers.png)

*Configure Offer Remote Assistance - Enabled, helpers: `FINAL\IT-Group` and `FINAL\Domain Admins`.*

![Configure Solicited Remote Assistance - Enabled, allow helpers to remotely control the computer.](screenshots/069-ra-solicited.png)

*Configure Solicited Remote Assistance - Enabled, allow helpers to remotely control the computer.*

![Security Option: User Account Control - Allow UIAccess applications to prompt for elevation, set to Enabled.](screenshots/070-ra-uac-uiaccess.png)

*Security Option: User Account Control - Allow UIAccess applications to prompt for elevation, set to Enabled.*

![Firewall - predefined inbound rule: Remote Assistance.](screenshots/071-ra-firewall-predefined.png)

*Firewall - predefined inbound rule: Remote Assistance.*

![Remote Assistance inbound rules created (Enabled, Allow).](screenshots/072-ra-firewall-rules.png)

*Remote Assistance inbound rules created (Enabled, Allow).*


---

### 5. DHCP and Failover

- Scope: `192.168.1.40 - 192.168.1.230` (/24), exclusion `192.168.1.80 - 192.168.1.85`, lease duration 10 days.
- Options: router `192.168.1.1`, DNS `192.168.1.2`, domain `FINAL.LOCAL`.
- DHCP role installed on `core`, authorized in AD, then DHCP failover configured between `pdc.final.local` and `core` (Hot standby, message authentication with a shared secret).
- The client received `192.168.1.40` from the scope and was then joined to the domain.

#### Create the scope on the PDC

![DHCP console on the PDC (`pdc.final.local`).](screenshots/024-dhcp-console.png)

*DHCP console on the PDC (`pdc.final.local`).*

![Right-click IPv4, then New Scope.](screenshots/025-dhcp-new-scope-menu.png)

*Right-click IPv4, then New Scope.*

![New Scope Wizard.](screenshots/026-dhcp-scope-wizard.png)

*New Scope Wizard.*

![Scope name.](screenshots/027-dhcp-scope-name.png)

*Scope name.*

![IP address range 192.168.1.40 - 192.168.1.230 with a /24 mask.](screenshots/028-dhcp-ip-range.png)

*IP address range 192.168.1.40 - 192.168.1.230 with a /24 mask.*

![Exclusion range 192.168.1.80 - 192.168.1.85.](screenshots/029-dhcp-exclusion.png)

*Exclusion range 192.168.1.80 - 192.168.1.85.*

![Lease duration set to 10 days.](screenshots/030-dhcp-lease.png)

*Lease duration set to 10 days.*

![Router (default gateway) option: 192.168.1.1.](screenshots/031-dhcp-router.png)

*Router (default gateway) option: 192.168.1.1.*

![Parent domain `FINAL.LOCAL` and DNS server 192.168.1.2 distributed to clients.](screenshots/032-dhcp-dns.png)

*Parent domain `FINAL.LOCAL` and DNS server 192.168.1.2 distributed to clients.*


#### Client receives an address and joins the domain

![Client - `ipconfig /all` shows 192.168.1.40 leased by the DHCP server 192.168.1.2.](screenshots/033-client-ipconfig.png)

*Client - `ipconfig /all` shows 192.168.1.40 leased by the DHCP server 192.168.1.2.*

![Client - Network Connection Details confirming the address, gateway, DHCP and DNS servers.](screenshots/034-client-connection-details.png)

*Client - Network Connection Details confirming the address, gateway, DHCP and DNS servers.*

![Client - "Welcome to the final.local domain" after joining the domain.](screenshots/035-client-joined-domain.png)

*Client - "Welcome to the final.local domain" after joining the domain.*


#### DHCP role on core and failover

![Add Roles and Features - destination server `core.FINAL.LOCAL`.](screenshots/152-core-dhcp-destination.png)

*Add Roles and Features - destination server `core.FINAL.LOCAL`.*

![Server roles list on `core`.](screenshots/153-core-dhcp-roles.png)

*Server roles list on `core`.*

![DHCP Server role selected on `core`.](screenshots/154-core-dhcp-selected.png)

*DHCP Server role selected on `core`.*

![DHCP post-install configuration - authorizing the DHCP server in AD DS.](screenshots/159-dhcp-authorization.png)

*DHCP post-install configuration - authorizing the DHCP server in AD DS.*

![Adding `core` to the DHCP console.](screenshots/160-dhcp-add-core.png)

*Adding `core` to the DHCP console.*

![Configure Failover wizard - select the scope(s) to protect.](screenshots/161-dhcp-failover-intro.png)

*Configure Failover wizard - select the scope(s) to protect.*

![Selecting the partner server (`core`, 192.168.1.3).](screenshots/162-dhcp-failover-partner.png)

*Selecting the partner server (`core`, 192.168.1.3).*

![Creating the failover relationship (`pdc.final.local-core`, Hot standby, message authentication enabled).](screenshots/163-dhcp-failover-relationship.png)

*Creating the failover relationship (`pdc.final.local-core`, Hot standby, message authentication enabled).*


---

### 6. IIS and DNS Load Balancing

- IIS installed on the PDC (Server Manager) and on Core (PowerShell) with the default page.
- Two `A` records named `www` in the `FINAL.LOCAL` zone point to `192.168.1.2` and `192.168.1.3` (DNS round-robin).

#### IIS on PDC and Core

![Add Roles and Features on the PDC - Web Server (IIS) with management tools.](screenshots/073-iis-pdc-role.png)

*Add Roles and Features on the PDC - Web Server (IIS) with management tools.*

![Server Core after joining - `sconfig` shows Domain: FINAL.LOCAL and computer name CORE; option 15 opens PowerShell.](screenshots/074-core-sconfig-domain.png)

*Server Core after joining - `sconfig` shows Domain: FINAL.LOCAL and computer name CORE; option 15 opens PowerShell.*

![Server Core PowerShell - first install attempt (typo in the cmdlet name returned an error).](screenshots/075-core-iis-first-attempt.png)

*Server Core PowerShell - first install attempt (typo in the cmdlet name returned an error).*

![Server Core PowerShell - `Install-WindowsFeature -Name web-server -IncludeManagementTools` finished with Success = True.](screenshots/076-core-iis-installed.png)

*Server Core PowerShell - `Install-WindowsFeature -Name web-server -IncludeManagementTools` finished with Success = True.*


#### DNS records for www.final.local

![DNS Manager, Forward Lookup Zones, `FINAL.LOCAL` - existing host records (pdc, core, Client-PC1).](screenshots/077-dns-zone-records.png)

*DNS Manager, Forward Lookup Zones, `FINAL.LOCAL` - existing host records (pdc, core, Client-PC1).*

![Creating the first `www` host (A) record.](screenshots/078-dns-new-host-www-1.png)

*Creating the first `www` host (A) record.*

![Creating the second `www` host (A) record pointing to 192.168.1.3 (round-robin).](screenshots/079-dns-new-host-www-2.png)

*Creating the second `www` host (A) record pointing to 192.168.1.3 (round-robin).*


---

### 7. File Services: shared folders, drive maps, file screening and quota

- Folders `HR` and `Dev` on volume `E:`, `Public` on volume `F:`.
- Share permissions: department group = Change; Domain Admins = Full Control.
- Dev: explicit **Deny** on *Delete* and *Delete subfolders and files* for `Dev-Group`, so users can create and edit but not delete.
- **FSRM file screening (Active)** using the template `Audio+Video+Exe` on `E:\Dev` and `E:\HR`.
- **Quota:** 2 GB per user on the Public volume, warning at 1700 MB.
- Drive maps by GPO: HR -> `H:`, Dev -> `I:`, Public -> `P:`.

#### HR folder and drive map

![HR folder on volume E:.](screenshots/080-hr-folder-properties.png)

*HR folder on volume E:.*

![HR share permissions - adding the Domain Admins group.](screenshots/081-hr-select-domain-admins.png)

*HR share permissions - adding the Domain Admins group.*

![HR share permissions - Domain Admins and HR-Group.](screenshots/082-hr-share-permissions.png)

*HR share permissions - Domain Admins and HR-Group.*

![Domain Admins - Full Control on the share.](screenshots/083-hr-domain-admins-full-control.png)

*Domain Admins - Full Control on the share.*

![NTFS permission entry for Domain Admins on HR - Full control.](screenshots/084-hr-ntfs-domain-admins.png)

*NTFS permission entry for Domain Admins on HR - Full control.*

![NTFS permission entry for HR-Group on HR.](screenshots/085-hr-ntfs-hr-group.png)

*NTFS permission entry for HR-Group on HR.*

![HR-MapDrive GPO, Drive Maps - `\\pdc\hr` mapped as drive H:.](screenshots/086-hr-drive-map-properties.png)

*HR-MapDrive GPO, Drive Maps - `\\pdc\hr` mapped as drive H:.*

![Drive Maps entry H: (Update) pointing to `\\Pdc\hr`.](screenshots/087-hr-drive-map-list.png)

*Drive Maps entry H: (Update) pointing to `\\Pdc\hr`.*


#### Dev folder and drive map

![Dev share permissions - Dev-Group and Domain Admins.](screenshots/088-dev-share-permissions.png)

*Dev share permissions - Dev-Group and Domain Admins.*

![Advanced Security Settings for the Dev folder - existing permission entries.](screenshots/089-dev-advanced-security.png)

*Advanced Security Settings for the Dev folder - existing permission entries.*

![Adding `Dev-Group` to the NTFS permissions.](screenshots/090-dev-select-group.png)

*Adding `Dev-Group` to the NTFS permissions.*

![Explicit Deny for `Dev-Group` on Delete and Delete subfolders and files (users can create and edit but not delete).](screenshots/091-dev-deny-delete.png)

*Explicit Deny for `Dev-Group` on Delete and Delete subfolders and files (users can create and edit but not delete).*

![Dev Properties, Security tab.](screenshots/092-dev-security-tab.png)

*Dev Properties, Security tab.*

![Dev-Map GPO, Drive Maps - `\\PDC\Dev` mapped as drive I:.](screenshots/093-dev-drive-map.png)

*Dev-Map GPO, Drive Maps - `\\PDC\Dev` mapped as drive I:.*


#### Public folder

![Public share permissions - Domain Admins and Domain Users.](screenshots/095-public-share-permissions.png)

*Public share permissions - Domain Admins and Domain Users.*

![PublicFolder-Map GPO - `\\PDC\Public` mapped as drive P:.](screenshots/096-public-drive-map.png)

*PublicFolder-Map GPO - `\\PDC\Public` mapped as drive P:.*


#### File screening and quota (FSRM)

![Adding the File Server Resource Manager role service.](screenshots/094-fsrm-role.png)

*Adding the File Server Resource Manager role service.*

![FSRM file screen on `E:\Dev` - active screening blocking Audio and Video files and Executable files.](screenshots/097-fsrm-file-screen.png)

*FSRM file screen on `E:\Dev` - active screening blocking Audio and Video files and Executable files.*

![Saving the file screen properties as the template `Audio+Video+Exe`.](screenshots/098-fsrm-save-template.png)

*Saving the file screen properties as the template `Audio+Video+Exe`.*

![File screens applied to `E:\Dev` and `E:\HR` from the template.](screenshots/099-fsrm-file-screens-list.png)

*File screens applied to `E:\Dev` and `E:\HR` from the template.*

![Disk quota on the Public volume - limit 2 GB, deny disk space beyond the limit, warning level 1700 MB.](screenshots/100-public-quota.png)

*Disk quota on the Public volume - limit 2 GB, deny disk space beyond the limit, warning level 1700 MB.*


---

### 8. Printing

- Print and Document Services role on the PDC; printer set to **Black & White**, with an availability schedule.
- The printer is deployed to domain users through Group Policy (Print Management, Deploy with Group Policy).

#### Print server and printer

![Adding the Print and Document Services role.](screenshots/101-print-role.png)

*Adding the Print and Document Services role.*

![Print Management - Add Printer.](screenshots/104-print-add-printer.png)

*Print Management - Add Printer.*

![Network Printer Installation wizard.](screenshots/105-print-install-wizard.png)

*Network Printer Installation wizard.*

![Printer Properties, Advanced tab - availability schedule.](screenshots/106-print-availability.png)

*Printer Properties, Advanced tab - availability schedule.*

![Printing Preferences - Black & White selected.](screenshots/107-print-black-white.png)

*Printing Preferences - Black & White selected.*

![Browsing for the GPO (`Printers`) used to deploy the printer.](screenshots/108-print-browse-gpo.png)

*Browsing for the GPO (`Printers`) used to deploy the printer.*

![Printer `\\PDC\Printer` deployed with Group Policy (per user).](screenshots/109-print-deploy-gpo.png)

*Printer `\\PDC\Printer` deployed with Group Policy (per user).*


---

### 9. Hyper-V and Hyper-V Replica

- Nested virtualization was enabled in VMware after the Hyper-V validation error.
- Hyper-V installed on PDC (Server Manager) and on Core (PowerShell); an external virtual switch was created on each host.
- VM **CoreVM2** (Generation 2, Windows Server 2022 Server Core) was created on the PDC.
- Core is configured as a Replica server (Kerberos/HTTP, port 80) and the *Hyper-V Replica HTTP Listener* firewall rule is enabled.
- Replication PDC -> Core every 15 minutes, initial copy over the network.

#### Install Hyper-V (enable nested virtualization)

![Add Roles and Features on the PDC - selecting the destination server.](screenshots/110-hyperv-select-destination.png)

*Add Roles and Features on the PDC - selecting the destination server.*

![Validation error - Hyper-V cannot be installed because the processor does not expose virtualization.](screenshots/111-hyperv-validation-error.png)

*Validation error - Hyper-V cannot be installed because the processor does not expose virtualization.*

![VMware VM settings - "Virtualize Intel VT-x/EPT or AMD-V/RVI" is greyed out.](screenshots/112-vmware-vt-disabled.png)

*VMware VM settings - "Virtualize Intel VT-x/EPT or AMD-V/RVI" is greyed out.*

![VMware VM settings - "Virtualize Intel VT-x/EPT or AMD-V/RVI" enabled (nested virtualization).](screenshots/113-vmware-vt-enabled.png)

*VMware VM settings - "Virtualize Intel VT-x/EPT or AMD-V/RVI" enabled (nested virtualization).*

![Hyper-V role selected.](screenshots/114-hyperv-role-selected.png)

*Hyper-V role selected.*

![Create Virtual Switches - selecting the network adapter.](screenshots/115-hyperv-virtual-switch.png)

*Create Virtual Switches - selecting the network adapter.*

![Server Core - installing Hyper-V with `Install-WindowsFeature -Name Hyper-V -IncludeManagementTools -Restart`.](screenshots/116-core-hyperv-powershell.png)

*Server Core - installing Hyper-V with `Install-WindowsFeature -Name Hyper-V -IncludeManagementTools -Restart`.*

![PDC Network Connections - new vEthernet adapter created by the virtual switch.](screenshots/117-pdc-vethernet.png)

*PDC Network Connections - new vEthernet adapter created by the virtual switch.*

![Virtual switch adapter renamed to `Ext192-V-Switch`.](screenshots/118-pdc-vswitch-renamed.png)

*Virtual switch adapter renamed to `Ext192-V-Switch`.*

![IPv4 settings on the virtual switch adapter: 192.168.1.2, gateway 192.168.1.1, DNS 192.168.1.2 / 192.168.1.3.](screenshots/119-pdc-vswitch-ipv4.png)

*IPv4 settings on the virtual switch adapter: 192.168.1.2, gateway 192.168.1.1, DNS 192.168.1.2 / 192.168.1.3.*


#### Virtual switch and Core VM on PDC

![Copying the Windows Server ISO into the VM to use as installation media for the Core VM.](screenshots/120-iso-copy-to-vm.png)

*Copying the Windows Server ISO into the VM to use as installation media for the Core VM.*

![Virtual Switch Manager on PDC - creating a new External virtual switch.](screenshots/121-vswitch-manager-new.png)

*Virtual Switch Manager on PDC - creating a new External virtual switch.*

![External virtual switch bound to the physical network adapter.](screenshots/122-vswitch-external.png)

*External virtual switch bound to the physical network adapter.*

![New Virtual Machine Wizard - name `CoreVM2`.](screenshots/123-vm-name.png)

*New Virtual Machine Wizard - name `CoreVM2`.*

![Generation 2 selected.](screenshots/124-vm-generation.png)

*Generation 2 selected.*

![Startup memory assigned.](screenshots/125-vm-memory.png)

*Startup memory assigned.*

![Network connection: the external virtual switch.](screenshots/126-vm-networking.png)

*Network connection: the external virtual switch.*

![`CoreVM2` created on PDC - right-click, Connect.](screenshots/127-vm-created.png)

*`CoreVM2` created on PDC - right-click, Connect.*

![Virtual Machine Connection window (VM turned off, ready to start).](screenshots/128-vm-connection.png)

*Virtual Machine Connection window (VM turned off, ready to start).*

![Creating the virtual hard disk `CoreVM2.vhdx`.](screenshots/129-vm-virtual-disk.png)

*Creating the virtual hard disk `CoreVM2.vhdx`.*

![Installing the operating system from the bootable ISO file.](screenshots/130-vm-install-iso.png)

*Installing the operating system from the bootable ISO file.*

![Windows Server 2022 setup - language and keyboard.](screenshots/131-vm-os-setup-language.png)

*Windows Server 2022 setup - language and keyboard.*

![Selecting Windows Server 2022 Standard Evaluation (Server Core).](screenshots/132-vm-os-setup-edition.png)

*Selecting Windows Server 2022 Standard Evaluation (Server Core).*

![Installing Windows Server 2022 in the VM.](screenshots/133-vm-os-installing.png)

*Installing Windows Server 2022 in the VM.*

![CoreVM2 - `sconfig` after the Server Core installation.](screenshots/151-corevm-sconfig.png)

*CoreVM2 - `sconfig` after the Server Core installation.*


#### Replica server on CORE

![Hyper-V Manager - connecting to another server: selecting `CORE` from the domain computers.](screenshots/134-hyperv-connect-core.png)

*Hyper-V Manager - connecting to another server: selecting `CORE` from the domain computers.*

![Hyper-V Manager managing both PDC and CORE.](screenshots/135-hyperv-manager-both.png)

*Hyper-V Manager managing both PDC and CORE.*

![Virtual Switch Manager on CORE - new External virtual switch.](screenshots/136-core-vswitch-manager.png)

*Virtual Switch Manager on CORE - new External virtual switch.*

![External virtual switch created on CORE.](screenshots/137-core-vswitch-external.png)

*External virtual switch created on CORE.*

![Hyper-V Settings for CORE - Replication Configuration not yet enabled.](screenshots/138-core-replication-disabled.png)

*Hyper-V Settings for CORE - Replication Configuration not yet enabled.*

!["Enable this computer as a Replica server" selected.](screenshots/139-core-replica-enabled.png)

*"Enable this computer as a Replica server" selected.*

![Replica authentication: Kerberos (HTTP) on port 80, allow replication from any authenticated server.](screenshots/140-core-replica-kerberos.png)

*Replica authentication: Kerberos (HTTP) on port 80, allow replication from any authenticated server.*

![Hyper-V Settings for PDC - Replication Configuration.](screenshots/141-pdc-replica-settings.png)

*Hyper-V Settings for PDC - Replication Configuration.*

![Windows Defender Firewall - enabling the "Hyper-V Replica HTTP Listener (TCP-In)" rule.](screenshots/142-firewall-replica-http.png)

*Windows Defender Firewall - enabling the "Hyper-V Replica HTTP Listener (TCP-In)" rule.*

![Hyper-V Replica listener inbound rules in Windows Defender Firewall.](screenshots/143-firewall-replica-https.png)

*Hyper-V Replica listener inbound rules in Windows Defender Firewall.*


#### Enable replication for CoreVM2

![Enable Replication for CoreVM2 - specifying the replica server `core`.](screenshots/155-replication-replica-server.png)

*Enable Replication for CoreVM2 - specifying the replica server `core`.*

![Connection parameters: `core.final.local`, port 80, Kerberos (HTTP) authentication.](screenshots/156-replication-connection.png)

*Connection parameters: `core.final.local`, port 80, Kerberos (HTTP) authentication.*

![Replication frequency: every 15 minutes.](screenshots/157-replication-frequency.png)

*Replication frequency: every 15 minutes.*

![Initial replication: send the initial copy over the network, start immediately.](screenshots/158-replication-initial.png)

*Initial replication: send the initial copy over the network, start immediately.*


---

### 10. Backup

- Windows Server Backup feature installed on the PDC.
- Scheduled daily full backup of the server to a shared network folder on the ADC, using a domain administrator account.

#### Windows Server Backup schedule

![Adding the Windows Server Backup feature.](screenshots/144-backup-feature.png)

*Adding the Windows Server Backup feature.*

![Windows Server Backup console (Local Backup).](screenshots/145-backup-console.png)

*Windows Server Backup console (Local Backup).*

![Backup Schedule Wizard - Getting Started.](screenshots/146-backup-schedule-wizard.png)

*Backup Schedule Wizard - Getting Started.*

![Backup time - once a day.](screenshots/147-backup-time.png)

*Backup time - once a day.*

![Destination type - back up to a shared network folder.](screenshots/148-backup-destination-type.png)

*Destination type - back up to a shared network folder.*

![Remote shared folder on the ADC (`\\core\c$`).](screenshots/149-backup-remote-folder.png)

*Remote shared folder on the ADC (`\\core\c$`).*

![Registering the backup schedule with a domain administrator account.](screenshots/150-backup-credentials.png)

*Registering the backup schedule with a domain administrator account.*



## Repository Structure

```
.
|-- README.md
`-- screenshots/     # 163 numbered screenshots (001 ... 163) in the order the lab was built
```

## Skills Demonstrated

Active Directory, Group Policy, DNS, DHCP Failover, IIS, File Server Resource Manager, Print Management, Hyper-V Replica, Windows Server Backup, Server Core administration, VMware Workstation.

## Notes

- Lab environment only. Passwords and credentials used here are for testing and are not included in this repository.
- Windows Server 2022 Evaluation editions were used.
