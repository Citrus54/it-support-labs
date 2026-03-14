# Name Resolution vs Connectivity

This lab demonstrates how to separate a DNS failure from a general network connectivity failure.

The goal is to prove that a system can still reach the internet by IP address while failing to resolve hostnames because of an invalid DNS server configuration.

---

## Ticket

**Issue:** User reports that the internet is down, but some connectivity tests still work.
---

## Initial Hypothesis

Based on the initial report, the problem could be caused by one of several issues:

- DNS server failure
- Incorrect DNS server configuration on the client
- Default gateway or routing failure
- General internet outage
- Multi-adapter network misconfiguration

At the start of troubleshooting, the exact cause is unknown. The key objective is to determine whether the issue is:

- **connectivity-related** (the client cannot reach external networks), or
- **name resolution-related** (the client can reach external IPs but cannot resolve hostnames)

---

## Business Impact

If DNS is not working correctly:

- Users may report that “the internet is down” even when the network path is still functional
- Websites and cloud services accessed by hostname may fail to load
- Software updates, downloads, and external service connections may fail
- SaaS applications may appear unavailable
- Troubleshooting can be delayed if the issue is mistaken for a full network outage

In a real business environment, this can affect productivity, delay access to web-based tools, and create unnecessary escalation if the issue is not quickly isolated to DNS.

---

## Environment

- Client OS: Windows 11 VM
- Hypervisor: Oracle VirtualBox
- Adapter 1: Host-only network
- Adapter 2: NAT network
- NAT adapter provides internet connectivity
- Host-only adapter remains connected to the lab network but is not used for internet access in this ticket

---

## Skills Demonstarted

Labs in this section help develop experience with:

- DNS troubleshooting
- Differentiating name resolution failures from routing/connectivity failures
- Reading multi-adapter `ipconfig /all` output
- Verifying default route with `route print`
- Testing with `ping` and `nslookup`

---

## Commands Used

- `ipconfig /all`
- `ping 8.8.8.8`
- `ping google.com`
- `nslookup google.com`
- `route print`

---

## Working State Verification

### Confirm Adapter Roles

Run on the client:

`ipconfig /all`

Expected:

- Host-only adapter has internal lab IP only
- NAT adapter has a NAT-assigned IPv4 address
- NAT adapter has a working default gateway
- DNS is working normally before the break

### Verify Internet Reachability by IP

Run:

`ping 8.8.8.8`

Expected result:

`Reply from 8.8.8.8`

This confirms the system can reach the internet at Layer 3.

### Verify Name Resolution

Run:

`ping google.com`

Expected result:

`Reply from <resolved IP>`

This confirms DNS name resolution is working before the break.

### Verify Default Route

Run:

`route print`

Expected key route:

`0.0.0.0          0.0.0.0          <NAT gateway>`

This confirms external traffic is routed through the NAT adapter.

---

## Breaking the Environment

### Goal

Create a failure where:

- `ping 8.8.8.8` works
- `ping google.com` fails
- `nslookup google.com` fails

### Change Only the DNS Server

On the client, open:

- Control Panel
- Network and Internet
- Network Connections
- Right-click the **NAT adapter**
- Properties
- Internet Protocol Version 4 (TCP/IPv4)
- Properties

Keep:

- **Obtain an IP address automatically**

Change:

- **Use the following DNS server addresses**

Set:

- Preferred DNS server: `10.1.10.250`
- Alternate DNS server: blank

### Why This Breaks the Lab

`10.1.10.250` does not exist as a working DNS server in this environment.

The client still has:

- a valid IP address
- a valid subnet mask
- a valid default gateway
- a valid default route

So external traffic can still be routed by IP.

But DNS queries now go to a dead server, so hostname lookups fail.

---

## Symptoms After Break

### IP Connectivity Still Works

Run:

`ping 8.8.8.8`

Observed result:

`Reply from 8.8.8.8`

This proves the machine still has internet reachability.

### Hostname Resolution Fails

Run:

`ping google.com`

Observed result:

`Ping request could not find host google.com. Please check the name and try again.`

This proves the system cannot resolve the hostname.

### DNS Lookup Fails

Run:

`nslookup google.com`

Observed result:

- `DNS request timed out.`
- `Server: Unknown`
- `Address: 10.1.10.250`
- `*** Request to UnKnown timed-out`

This shows the client is explicitly trying to use the broken DNS server.

### Routing Table Still Shows Valid Internet Route

Run:

`route print`

Observed key route:

`0.0.0.0          0.0.0.0          10.0.3.2          10.0.3.15`

This confirms the issue is not the default route.

---

## Root Cause

The client’s DNS server on the NAT adapter was manually changed to an invalid IP address.

Because the DNS server was unreachable:

- hostname lookups failed
- `ping google.com` failed
- `nslookup google.com` timed out

However, the client still had:

- a valid NAT IP address
- a valid default gateway
- a valid default route

So traffic sent directly to an IP address still worked.

This is a DNS failure, not a routing or internet connectivity failure.

---

## Resolution

Restore the NAT adapter’s DNS settings.

Open the NAT adapter IPv4 settings and either:

### Option 1

Set:

- **Obtain DNS server address automatically**

### Option 2

Manually enter a valid DNS server for the adapter

For this lab, the simplest fix is:

- **Obtain DNS server address automatically**

---

## Verification After Fix

### Confirm DNS Settings Restored

Run:

`ipconfig /all`

Expected:

- NAT adapter no longer points to `10.1.10.250`

### Confirm Internet Reachability by IP

Run:

`ping 8.8.8.8`

Expected:

`Reply from 8.8.8.8`

### Confirm Hostname Resolution Works Again

Run:

`ping google.com`

Expected:

`Reply from <resolved IP>`

### Confirm DNS Queries Succeed

Run:

`nslookup google.com`

Expected:

- `Non-authoritative answer:`
- `Name: google.com`
- `Addresses: ...`

---

## Verification Summary

The issue was confirmed to be a DNS failure, not a general connectivity failure.

Evidence collected showed:

- the client retained a valid IP address on the NAT adapter
- the client retained a valid default gateway and default route
- `ping 8.8.8.8` succeeded, proving internet reachability by IP
- `ping google.com` failed, proving hostname resolution was broken
- `nslookup google.com` timed out against `10.1.10.250`, confirming the configured DNS server was invalid or unreachable

After restoring valid DNS settings, hostname resolution and normal internet access by name were restored.

---

## Key Concepts

### Connectivity vs Name Resolution

A machine can still have working internet access even when DNS is broken.

If routing/connectivity is broken:

`ping 8.8.8.8`

fails

If DNS is broken:

`ping 8.8.8.8`

works, but:

- `ping google.com`
- `nslookup google.com`

fail

### Why IP Tests Matter

Testing by IP helps isolate whether the issue is:

- DNS-related
- gateway/routing-related
- broader internet connectivity failure

### Why `nslookup` Matters

`nslookup` shows which DNS server the client is actually trying to use.

In this lab, that was critical because it verified the client was sending queries to:

`10.1.10.250`

### Why `route print` Matters

`route print` confirmed the machine still had a valid default route:

`0.0.0.0 -> 10.0.3.2`

That ruled out the default gateway as the source of failure.

---

## Evidence Collected

Screenshots included:

- Broken `ipconfig /all`
- NAT adapter showing invalid DNS server `10.1.10.250`
- Successful `ping 8.8.8.8`
- Failed `ping google.com`
- Failed `nslookup google.com`
- `route print` showing valid default route
- Restored adapter DNS settings
- Successful `ping google.com` after fix
- Successful `nslookup google.com` after fix

---

## Final Takeaway

This lab demonstrates a classic troubleshooting scenario:

A user may say “the internet is down,” but the real issue may be only DNS.

The correct troubleshooting path is:

1. Test connectivity by IP
2. Test connectivity by hostname
3. Query DNS directly
4. Check the routing table
5. Isolate whether the issue is name resolution or network path

