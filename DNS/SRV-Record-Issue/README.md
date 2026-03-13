# DNS — Domain Controller Discovery Failure (SRV Record)

## Ticket

**Issue:**
User cannot sign in to the domain.

**Error:**
"The domain controller could not be contacted."

Business Impact
Users could not authenticate to the domain, new machines could not join the domain, and Group Policy processing would fail. In a production environment, this would affect user access, workstation management, and domain-dependent services.

---
## Initial Hypothesis
The client may be unable to locate a domain controller due to a DNS-related discovery failure, most likely involving missing or incorrect SRV records used by Active Directory.

## Environment

* Domain: `curtisitproject.com`
* Domain Controller: `IL-DC-01`
* DC IP: `10.1.10.2`
* Client OS: Windows 11
* Server OS: Windows Server 2022
* DNS hosted on Domain Controller

---

## Skills Demonstrated

* DNS troubleshooting
* Active Directory DC discovery
* SRV records
* `nltest`
* `nslookup`
* Netlogon DNS registration
* Command-line diagnostics
* Service Recovery
* Root cause analysis
* Technical Documentation
  
---

## Working State Verification

### Check Domain Controller Discovery

Run on the client:

```
nltest /dsgetdc:curtisitproject.com
```

Expected result:

```
DC: \\IL-DC-01.curtisitproject.com
Address: \\10.1.10.2
```

---

## Check SRV Record Resolution

```
nslookup -type=SRV _ldap._tcp.dc._msdcs.curtisitproject.com
```

Expected result:

```
_ldap._tcp.dc._msdcs.curtisitproject.com
port = 389
svr hostname = il-dc-01.curtisitproject.com
```

This confirms the client can discover the domain controller through DNS.

---

## Breaking the Environment

### Location of SRV Records

Open:

```
DNS Manager
Forward Lookup Zones
_msdcs.curtisitproject.com
dc
_tcp
```

Delete the following SRV records associated with the domain controller:

```
_ldap._tcp.dc._msdcs.curtisitproject.com
_ldap._tcp.Default-First-Site-Name._sites.dc._msdcs.curtisitproject.com
```

Multiple SRV records may exist.
All LDAP locator records must be removed to fully break DC discovery.

---

## Symptoms After Break

### SRV Lookup Failure

```
nslookup -type=SRV _ldap._tcp.dc._msdcs.curtisitproject.com
```

Result:

```
*** Non-existent domain
```

---

## Domain Controller Discovery Failure

```
nltest /dsgetdc:curtisitproject.com
```

Result:

```
Getting DC name failed: Status = 1355 0x54b ERROR_NO_SUCH_DOMAIN
```

The client cannot locate a domain controller.

---
## Lab Configuration Note

NetBIOS over TCP/IP was disabled on the client during this lab.

Reason:
NetBIOS can allow clients to locate domain controllers through legacy name resolution even when DNS SRV records are missing. Disabling NetBIOS ensures the client relies strictly on DNS for domain controller discovery.

Client configuration:

Network Adapter → IPv4 → Advanced → WINS → Disable NetBIOS over TCP/IP

## Root Cause

Domain Controller discovery relies on **LDAP SRV records stored in DNS**.

These records advertise which servers provide LDAP directory services for the domain.

When the `_ldap._tcp.dc._msdcs` records are missing:

* Clients cannot discover a domain controller
* Authentication fails
* Domain join operations fail
* Group Policy processing fails

Even if the domain controller is reachable by IP, clients cannot locate it without the SRV records.

---

## Resolution

Restart Netlogon on the Domain Controller to re-register DNS records.

```
net start netlogon
```

Force DNS registration:

```
nltest /dsregdns
```

Netlogon automatically recreates the required SRV records.

---

## Verification After Fix

### Verify SRV Records

```
nslookup -type=SRV _ldap._tcp.dc._msdcs.curtisitproject.com
```

Expected:

```
_ldap._tcp.dc._msdcs.curtisitproject.com
port = 389
svr hostname = il-dc-01.curtisitproject.com
```

---

## Verify Domain Controller Discovery

```
nltest /dsgetdc:curtisitproject.com
```

Expected:

```
DC: \\IL-DC-01.curtisitproject.com
Address: \\10.1.10.2
```

Domain discovery is restored.

## Verification Summary

After restarting Netlogon and forcing DNS registration, the missing LDAP SRV records were recreated in DNS. The client successfully resolved _ldap._tcp.dc._msdcs.curtisitproject.com and nltest /dsgetdc:curtisitproject.com returned the correct domain controller, confirming domain controller discovery was restored.

---

## Commands Used

```
nltest /dsgetdc:curtisitproject.com
nslookup -type=SRV _ldap._tcp.dc._msdcs.curtisitproject.com
net stop netlogon
net start netlogon
nltest /dsregdns
```

---

## Key Concepts

### What SRV Records Do in Active Directory

SRV records allow clients to discover services on the network.

Active Directory uses SRV records to advertise:

* LDAP directory services
* Kerberos authentication
* Global Catalog servers
* Domain controllers

Example LDAP SRV record:

```
_ldap._tcp.dc._msdcs.curtisitproject.com
```

This tells clients which server provides LDAP services for the domain.

Without these records, domain controller discovery fails.

---

## Screenshots

1. Working `nltest` output
2. Working SRV lookup
3. DNS Manager location before deletion
4. DNS records after deletion
5. Failed SRV lookup
6. Failed `nltest`
7. Netlogon repair command
8. Successful SRV lookup after fix
9. Successful `nltest` after fix
10. Disabled Net BIOS
