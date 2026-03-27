# Domain Verification TXT Record Failure

## Ticket

**Title:** Customer cannot verify their domain in the platform  
**User reports:** “We added the TXT record exactly as instructed, but the platform still says verification failed.”  
**Impact:** Customer cannot complete domain setup or enable the domain-based feature  
**Priority:** Medium

---

## Environment

### Server VM
- Windows Server 2022
- DNS Server role installed

### Client VM
- Windows 11
- DNS configured to point to the Windows Server 2022 VM

### Lab Zone
- `curtislab.local`

### Expected Verification Record
- **Host/Name:** `_verify`
- **FQDN:** `_verify.curtislab.local`
- **Type:** `TXT`
- **Expected Value:** `saas-verification=abc123xyz`

---

## Lab Setup

### 1. Verify Basic Connectivity

On the Windows Server 2022 VM, find the server IP:

```powershell
ipconfig
```

On the Windows 11 client, verify the client can reach the server:

```powershell
ping <server-ip>
```

Example:

```powershell
ping 192.168.56.10
```

### 2. Install DNS Server Role on Windows Server 2022

If DNS is already installed, skip this section.

- Open **Server Manager**
- Click **Manage**
- Click **Add Roles and Features**
- Choose **Role-based or feature-based installation**
- Select the local server
- Check **DNS Server**
- Add required features if prompted
- Complete the wizard and install

After installation:
- Open **Server Manager**
- Go to **Tools**
- Open **DNS**

### 3. Create the DNS Zone

- In **DNS Manager**, expand the server
- Right-click **Forward Lookup Zones**
- Click **New Zone**
- Choose **Primary Zone**
- Name the zone:

```text
curtislab.local
```

Finish the wizard.

### 4. Point the Windows 11 Client to the Server for DNS

On the Windows 11 client:

- Press `Windows + R`
- Type `ncpa.cpl`
- Open the active network adapter
- Open **Properties**
- Select **Internet Protocol Version 4 (TCP/IPv4)**
- Click **Properties**
- Under DNS, choose **Use the following DNS server addresses**
- Set **Preferred DNS server** to the IP of the Windows Server 2022 VM

Verify on the client:

```powershell
ipconfig /all
```

Confirm the client is using the server VM as its DNS server.

---

## Tools Used

- `ipconfig /all`
- `nslookup -type=TXT`
- DNS Manager
- optional: `nslookup` interactive mode

---

## Broken Scenarios

This lab tests 3 common reasons TXT-based domain verification fails.

### Scenario 1: TXT Record Missing

#### What You Break
Do not create the `_verify` TXT record yet.

#### Test from the Client

```powershell
nslookup -type=TXT _verify.curtislab.local
```

#### Expected Result
The lookup should return no TXT result, no record found, or non-existent domain.

#### Root Cause
The required TXT verification record was missing, so the platform could not confirm domain ownership.

#### Resolution
Create the required TXT record at the correct hostname with the correct value.

---

### Scenario 2: TXT Record Exists but Value Is Wrong

#### What You Break
Create a TXT record at `_verify` with the wrong value.

#### Broken Example
- **Host/Name:** `_verify`
- **TXT Value:** `saas-verification=wrongvalue`

#### Test from the Client

```powershell
nslookup -type=TXT _verify.curtislab.local
```

#### Expected Result
The TXT record exists, but the value does not match the expected token.

#### Root Cause
The TXT verification record existed, but the value did not match the expected verification token, so validation failed.

#### Resolution
Edit the TXT record so the value exactly matches:

```text
saas-verification=abc123xyz
```

---

### Scenario 3: TXT Record Created on the Wrong Hostname

#### What You Break
Instead of creating the TXT record at `_verify.curtislab.local`, create it at the root of the zone:

- `curtislab.local`

#### Test from the Client

Check the expected hostname:

```powershell
nslookup -type=TXT _verify.curtislab.local
```

Then check the zone root:

```powershell
nslookup -type=TXT curtislab.local
```

#### Expected Result
- `_verify.curtislab.local` returns no valid record
- `curtislab.local` may return the TXT value

#### Root Cause
The TXT record was created on the wrong hostname, so the platform’s verification lookup did not find the required record.

#### Resolution
Move the TXT record from `curtislab.local` to `_verify.curtislab.local`.

---

## Troubleshooting Process

### 1. Confirm the Client Is Using the Correct DNS Server

Run:

```powershell
ipconfig /all
```

Check the **DNS Servers** field.

If the client is using the wrong DNS server, TXT lookup results may be wrong even if the server-side configuration is correct.

### 2. Query the TXT Record from the Client

Run:

```powershell
nslookup -type=TXT _verify.curtislab.local
```

This simulates how support would validate whether the customer’s record is present and correct.

### 3. Compare Expected vs Actual

Check:
- does the TXT record exist?
- is it on the correct hostname?
- does the value exactly match the expected verification token?

Expected:

```text
_verify.curtislab.local
TXT
saas-verification=abc123xyz
```

### 4. Determine the Root Cause

Possible root causes in this lab:
- record missing
- record value incorrect
- record created on wrong hostname
- client querying wrong DNS server

### 5. Apply the Fix

Fix the exact issue:
- create missing record
- correct wrong value
- move record to correct hostname
- point client to correct DNS server

### 6. Verify the Fix

From the client, run:

```powershell
nslookup -type=TXT _verify.curtislab.local
```

Verification is successful when:
- the TXT record resolves
- the hostname is correct
- the value exactly matches the expected token

---

## Verification

### Command Used

```powershell
nslookup -type=TXT _verify.curtislab.local
```

### Successful Result
The lookup should return the correct TXT record value for the expected hostname.

---

## Screenshots to Include

### Broken State
- DNS Manager showing missing, wrong, or misplaced TXT record
- `ipconfig /all` from client
- failed or incorrect `nslookup -type=TXT _verify.curtislab.local`

### Fixed State
- corrected TXT record in DNS Manager
- successful TXT lookup from client
- optional side-by-side expected vs actual comparison

---

## What I Learned

- A TXT record stores text data in DNS that outside systems can read.
- SaaS platforms use TXT records to verify domain control by checking for a specific token.
- Domain verification can fail even if a TXT record exists if:
  - the value is wrong
  - the record is placed on the wrong hostname
  - the client is querying the wrong DNS server
- Support troubleshooting should validate:
  - record presence
  - hostname placement
  - token accuracy
  - client-side DNS path


