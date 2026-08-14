# Active Directory Enterprise Lab — Build Manual

This guide explains how to build a small Windows enterprise environment using **Windows Server 2022, Windows 11 Pro, Active Directory Domain Services (AD DS), DNS, Group Policy, security groups, and SMB file sharing**.

The finished lab will contain:

```text
                    VMware Workstation
                           |
                  192.168.170.0/24
                           |
             +-------------+-------------+
             |                           |
           DC01                       CLIENT01
    Windows Server 2022              Windows 11 Pro
      192.168.170.10                    DHCP
             |
         AD DS + DNS
             |
       abctech.local
             |
     +-------+--------+
     |       |        |
   Users   Groups    OUs
             |
       File Permissions
```

> **Important:** The IP addresses in this guide match my VMware NAT environment. VMware may create a different subnet on your computer. Check your VMnet8 configuration before assigning static addresses.

---

# 1. Prerequisites

You will need:

- VMware Workstation
- Windows Server 2022 Evaluation ISO
- Windows 11 Pro ISO
- Approximately 100–120 GB of free storage
- At least 8 GB RAM available for the lab
- Administrator access to the host computer

Recommended host specifications:

```text
RAM:       16 GB+
Storage:   150 GB+ free
CPU:       4+ cores
```

This lab uses VMware NAT networking so both virtual machines can communicate while still having internet connectivity through the host.

---

# 2. Create the Windows Server VM

Create a new virtual machine in VMware Workstation.

Suggested configuration:

| Setting | Value |
|---|---|
| VM Name | DC01 |
| Operating System | Windows Server 2022 |
| RAM | 4 GB |
| CPU | 2 cores |
| Disk | 60 GB |
| Network | NAT |
| Firmware | UEFI |

Attach the Windows Server 2022 Evaluation ISO.

During installation select:

**Windows Server 2022 Standard Evaluation (Desktop Experience)**

Desktop Experience provides the Windows graphical interface, which makes the environment easier to learn for a first Active Directory lab.

Complete the installation and configure a strong lab-only Administrator password.

---

# 3. Rename the Server

After logging into Windows Server, rename the computer:

```text
DC01
```

This can be done through:

**Server Manager → Local Server → Computer Name**

Restart the server after changing the name.

Verify using:

```powershell
hostname
```

Expected result:

```text
DC01
```

---

# 4. Examine the VMware Network

Before manually assigning an IP address, determine which subnet VMware is using.

Inside DC01 run:

```powershell
ipconfig /all
```

In my environment VMware created:

```text
Network:       192.168.170.0/24
Gateway:       192.168.170.2
DHCP Server:   192.168.170.254
```

Also check:

**VMware Workstation → Edit → Virtual Network Editor → VMnet8**

Review the DHCP allocation range.

Choose a static server address that:

1. Belongs to the VMnet8 subnet.
2. Is outside the DHCP allocation pool.
3. Is not already being used.

For this lab I selected:

```text
192.168.170.10
```

---

# 5. Configure DC01 with a Static IP

On DC01 press:

**Windows + R**

Run:

```text
ncpa.cpl
```

Navigate to:

**Ethernet → Properties → Internet Protocol Version 4 (TCP/IPv4) → Properties**

Configure:

```text
IP Address:       192.168.170.10
Subnet Mask:      255.255.255.0
Default Gateway:  192.168.170.2
Preferred DNS:    192.168.170.10
```

The DNS address points to DC01 because this server will soon host the DNS service used by Active Directory.

Verify:

```powershell
ipconfig /all
```

You should see:

```text
DHCP Enabled:     No
IPv4 Address:     192.168.170.10
Subnet Mask:      255.255.255.0
Default Gateway:  192.168.170.2
DNS Server:       192.168.170.10
```

Test the gateway:

```powershell
ping 192.168.170.2
```

---

# 6. Install Active Directory Domain Services

Open:

**Server Manager → Manage → Add Roles and Features**

Select:

**Role-based or feature-based installation**

Select DC01 and enable:

**Active Directory Domain Services**

When Windows asks to install required features, select:

**Add Features**

Continue through the wizard and install the role.

Installing AD DS does **not** automatically make the server a Domain Controller.

The process is:

```text
Windows Server
      |
      v
Install AD DS
      |
      v
AD DS Components Installed
      |
      v
Promote Server
      |
      v
Domain Controller
```

---

# 7. Promote DC01 to a Domain Controller

After AD DS installation, Server Manager should display a notification.

Select:

**Promote this server to a domain controller**

Because this is a completely new environment, choose:

**Add a new forest**

Root domain name:

```text
abctech.local
```

> `.local` is suitable for this isolated training lab. A new production environment would normally use a DNS namespace based on a domain owned by the organization.

Continue to Domain Controller Options.

Ensure the following are selected:

```text
DNS Server
Global Catalog (GC)
```

Create a strong **Directory Services Restore Mode (DSRM)** password.

Keep the default NetBIOS name:

```text
ABCTECH
```

Leave the default database locations:

```text
Database: C:\Windows\NTDS
Logs:     C:\Windows\NTDS
SYSVOL:   C:\Windows\SYSVOL
```

Complete the prerequisites check and install.

DC01 will restart.

---

# 8. Verify the Domain

After restarting, log in using the domain Administrator account.

Open:

**Server Manager → Tools → Active Directory Users and Computers**

You should see:

```text
abctech.local
```

Expand:

**Domain Controllers**

You should see:

```text
DC01
```

At this point DC01 is functioning as the first Domain Controller for the environment.

---

# 9. Create the Organizational Unit Structure

In **Active Directory Users and Computers**, create:

```text
ABC Technologies
|
+-- Users
|   |
|   +-- HR
|   +-- Finance
|   +-- Sales
|   +-- IT
|   +-- Management
|
+-- Computers
|
+-- Groups
```

Organizational Units help organize Active Directory objects and provide locations where Group Policies can be targeted.

---

# 10. Create Domain Users

Create test users inside their appropriate department OUs.

Example:

| Employee | Username | Department |
|---|---|---|
| Sarah Brown | sarah.brown | HR |
| Emma Davis | emma.davis | Finance |
| John Smith | john.smith | Sales |
| Alex Carter | alex.carter | IT |
| Michael Scott | michael.scott | Management |

Use lab-only passwords.

These are fictional accounts created only for the training environment.

---

# 11. Create Security Groups

Inside the Groups OU create:

```text
GG_HR
GG_FINANCE
GG_SALES
GG_IT
GG_MANAGEMENT
```

Configure each as:

```text
Group Scope: Global
Group Type:  Security
```

Add users to their corresponding departmental group.

For example:

```text
Sarah Brown
     |
     v
   GG_HR
```

Security groups will later control access to departmental resources.

---

# 12. Why Use Groups Instead of Direct Permissions?

Avoid configuring file permissions like:

```text
Sarah Brown → HR Folder
James Wilson → Finance Folder
```

Instead use:

```text
Sarah Brown
     |
     v
   GG_HR
     |
     v
HR Resources
```

When another HR employee joins the company, simply add them to `GG_HR`.

This provides much more manageable role-based access.

---

# 13. Create CLIENT01

Create another VMware virtual machine.

Suggested configuration:

| Setting | Value |
|---|---|
| VM Name | CLIENT01 |
| Operating System | Windows 11 Pro |
| RAM | 4 GB |
| CPU | 2 cores |
| Disk | 50 GB |
| Network | NAT |

Install **Windows 11 Pro**.

Windows 11 Home cannot join a traditional Active Directory domain.

Name the computer:

```text
CLIENT01
```

---

# 14. Configure CLIENT01 Networking

Unlike DC01, CLIENT01 does not require a static IP.

Allow VMware DHCP to provide:

```text
IP Address
Subnet Mask
Default Gateway
```

However, CLIENT01 must use the Active Directory DNS server.

Open:

```text
ncpa.cpl
```

Navigate to the IPv4 configuration.

Keep:

```text
Obtain an IP address automatically
```

Configure:

```text
Preferred DNS Server:
192.168.170.10
```

This is critical.

CLIENT01 must use DC01's DNS service to locate the Active Directory domain.

---

# 15. Test Domain Connectivity

From CLIENT01:

```powershell
ping 192.168.170.10
```

Then test DNS:

```powershell
nslookup abctech.local
```

You can also test:

```powershell
ping DC01
```

If these tests succeed, CLIENT01 can communicate with the Domain Controller and resolve the internal domain.

---

# 16. Join CLIENT01 to the Domain

Open Windows system settings and join the computer to:

```text
abctech.local
```

When credentials are requested, provide an account authorized to join computers to the domain, such as the lab Domain Administrator.

Example:

```text
ABCTECH\Administrator
```

You should receive:

```text
Welcome to the abctech.local domain
```

Restart CLIENT01.

---

# 17. Test Domain Authentication

At the CLIENT01 login screen choose:

**Other user**

Sign in using:

```text
ABCTECH\sarah.brown
```

Alternatively:

```text
sarah.brown@abctech.local
```

If authentication succeeds, CLIENT01 is successfully using Active Directory for domain authentication.

---

# 18. Move CLIENT01 into the Computers OU

By default, a newly joined workstation may appear inside the default **Computers** container.

On DC01 open:

**Active Directory Users and Computers**

Move CLIENT01 to:

```text
ABC Technologies
|
+-- Computers
    |
    +-- CLIENT01
```

This will allow policies linked to the Computers OU to target CLIENT01.

---

# 19. Create a Group Policy

Open:

**Server Manager → Tools → Group Policy Management**

Locate:

```text
ABC Technologies → Computers
```

Create and link a new GPO:

```text
Workstation Security Policy
```

Edit the GPO.

Navigate to:

```text
Computer Configuration
→ Policies
→ Windows Settings
→ Security Settings
→ Local Policies
→ Security Options
```

Configure:

```text
Interactive logon: Machine inactivity limit
```

Set:

```text
600 seconds
```

This automatically locks company workstations after 10 minutes of inactivity.

---

# 20. Apply and Verify Group Policy

On CLIENT01 open an elevated Command Prompt or PowerShell window.

Run:

```powershell
gpupdate /force
```

Then verify:

```powershell
gpresult /r
```

Under **Applied Group Policy Objects**, you should see:

```text
Workstation Security Policy
```

This confirms that the policy was delivered from Active Directory to CLIENT01.

---

# 21. Create Department File Shares

On DC01 create:

```text
C:\CompanyShares\
|
+-- HR
+-- Finance
+-- Sales
+-- Public
```

Place a test document inside each folder.

Example:

```text
HR-Documents.txt
Finance-Documents.txt
```

---

# 22. Share the Folders

For the HR folder:

**Properties → Sharing → Advanced Sharing**

Enable:

```text
Share this folder
```

Share name:

```text
HR
```

The network path becomes:

```text
\\DC01\HR
```

Repeat for:

```text
\\DC01\Finance
\\DC01\Sales
\\DC01\Public
```

---

# 23. Configure Department Permissions

Configure departmental access using the security groups created earlier.

Example:

```text
HR
└── GG_HR

Finance
└── GG_FINANCE

Sales
└── GG_SALES
```

Administrators should retain administrative access.

For the Public share, you can provide appropriate access to:

```text
Domain Users
```

Remember that Windows file access can involve both:

- Share permissions
- NTFS permissions

The effective access is determined by the combination of these permission layers.

---

# 24. Test HR Access

Sign into CLIENT01 as:

```text
ABCTECH\sarah.brown
```

Open:

```text
\\DC01\HR
```

Sarah should be able to access the HR resource.

Then attempt:

```text
\\DC01\Finance
```

Sarah should not receive Finance access if the permissions have been configured correctly.

The expected model is:

```text
Sarah Brown
     |
     v
   GG_HR
     |
     +------ HR       ALLOWED
     |
     +------ Finance  DENIED
```

---

# 25. Test Another Department

Sign into CLIENT01 as the Finance user.

Example:

```text
ABCTECH\emma.davis
```

Test:

```text
\\DC01\Finance
```

Expected:

```text
Finance → Allowed
HR      → Denied
Public  → Allowed
```

This verifies that security group membership is controlling resource access.

---

# 26. Troubleshooting Exercise — Account Lockout

Scenario:

> A user entered their password incorrectly several times and can no longer log in.

On DC01 open:

**Active Directory Users and Computers**

Locate the user.

Check whether the account is locked.

If the user knows their password:

**Unlock the account.**

If they have forgotten the password:

1. Reset the password.
2. Provide a temporary password according to your lab procedure.
3. Enable **User must change password at next logon**.
4. Verify successful authentication.

The important distinction is:

```text
Locked Account
     ≠
Forgotten Password
```

A locked account does not automatically require a password reset.

---

# 27. Troubleshooting Exercise — New Employee

Scenario:

> James Wilson joins the Finance department.

Create:

```text
james.wilson
```

Place the account in:

```text
Users → Finance
```

Then add James to:

```text
GG_FINANCE
```

Do **not** manually assign James permissions to the Finance folder.

His access should come from:

```text
James Wilson
     |
     v
GG_FINANCE
     |
     v
Finance Share
```

Verify:

```text
Finance → Allowed
HR      → Denied
```

---

# 28. Troubleshooting Exercise — Domain Cannot Be Contacted

Scenario:

CLIENT01 reports:

```text
The specified domain either does not exist or could not be contacted.
```

The client's DNS server was accidentally changed to:

```text
8.8.8.8
```

Do not immediately change settings.

Start by gathering information:

```powershell
ipconfig /all
```

Check:

```text
IPv4 Address
Default Gateway
DNS Servers
```

Test whether DC01 is reachable:

```powershell
ping 192.168.170.10
```

Then test DNS:

```powershell
nslookup abctech.local
```

If network connectivity to DC01 works but internal domain resolution fails, investigate the DNS configuration.

Restore CLIENT01's DNS server:

```text
192.168.170.10
```

Clear cached DNS information:

```powershell
ipconfig /flushdns
```

Retest:

```powershell
nslookup abctech.local
```

You can also test:

```powershell
ping DC01
```

This demonstrates an important troubleshooting principle:

```text
Network Connectivity
        ≠
DNS Resolution
```

A computer may be able to reach another IP address while still being unable to resolve its hostname.

---

# 29. Useful Commands

### Network Configuration

```powershell
ipconfig
ipconfig /all
```

### Connectivity

```powershell
ping 192.168.170.10
ping DC01
```

### DNS

```powershell
nslookup abctech.local
ipconfig /flushdns
```

### Computer Identity

```powershell
hostname
```

### Group Policy

```powershell
gpupdate /force
gpresult /r
```

### Active Directory PowerShell

On DC01:

```powershell
Get-ADUser -Filter *
```

For a cleaner view:

```powershell
Get-ADUser -Filter * | Select-Object Name,SamAccountName
```

---

# 30. Common Problems

### CLIENT01 Cannot Join the Domain

First check DNS:

```powershell
ipconfig /all
```

CLIENT01 should use:

```text
DNS → DC01
```

not a public DNS server such as:

```text
8.8.8.8
```

---

### CLIENT01 Cannot Reach DC01

Test:

```powershell
ping 192.168.170.10
```

Check that:

- Both VMs are running.
- Both use the same VMware NAT network.
- Their IP addresses belong to the same subnet.
- DC01 still has its static address.

---

### Group Policy Does Not Apply

Confirm CLIENT01 is inside the OU where the GPO is linked.

Then run:

```powershell
gpupdate /force
gpresult /r
```

Do not assume a GPO applied simply because it was created.

Always verify.

---

### User Cannot Access a Shared Folder

Check:

1. User identity.
2. Security group membership.
3. Share permissions.
4. NTFS permissions.
5. Whether the user has signed out/in after a recent group membership change.

Remember:

```text
User
 ↓
Security Group
 ↓
Permissions
 ↓
Resource
```

---

# 31. What You Have Built

After completing this guide your environment should resemble:

```text
                         ABC Technologies
                          abctech.local
                                |
                              DC01
                        192.168.170.10
                                |
                  +-------------+-------------+
                  |             |             |
                AD DS          DNS           GPO
                  |                           |
          +-------+-------+                   |
          |               |                   |
        Users           Groups          Computers OU
          |               |                   |
     Departments     Access Control         CLIENT01
                          |                   |
                    File Shares <------ Domain Users
```

You have now implemented:

- Windows Server 2022
- Active Directory Domain Services
- Domain Controller deployment
- DNS
- Static server addressing
- Windows 11 domain joining
- Domain authentication
- Organizational Units
- Domain users
- Security groups
- Group Policy
- SMB file sharing
- Group-based permissions
- Basic Active Directory troubleshooting

---

# 32. Next Steps

Once the core lab is working, possible extensions include:

- PowerShell user provisioning
- Additional Windows clients
- Password and account policies
- Drive mapping through Group Policy
- Software deployment through GPO
- Additional Domain Controller
- DHCP Server
- Windows Server backup and recovery
- Microsoft Entra ID
- Microsoft 365 administration
- Hybrid identity concepts

---

## Final Note

This lab is intended for **education and hands-on practice**.

The organization, employees, accounts, and business scenarios used throughout the guide are fictional.

The objective is not simply to follow configuration steps, but to understand how:

```text
Networking
    +
DNS
    +
Active Directory
    +
Authentication
    +
Group Policy
    +
Access Control
```

work together in a Windows enterprise environment.
