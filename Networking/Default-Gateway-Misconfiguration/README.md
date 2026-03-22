# Networking — Default Gateway Misconfiguration

## Ticket

**Issue:**
User can reach local systems but cannot access the internet or remote networks.

### Business Impact

The user could still communicate with local lab systems, but internet access and connectivity to remote networks were unavailable. In a production environment, this would prevent access to cloud services, external websites, updates, remote systems, and any business applications hosted outside the local subnet.

### Initial Hypothesis

The client may have a routing issue on its internet-facing adapter, most likely caused by an incorrect default gateway or IP configuration preventing traffice from leaving the local subnet.

---

## Environment

* Client OS: Windows 11 VM
* Hypervisor: Oracle VirtualBox
* Adapter 1: Host-only network (10.1.10.x lab network)
* Adapter 2: NAT (internet access)

NAT adapter provides the external gateway used to reach the internet.

---

## Skills Demonstrated

* IP configuration troubleshooting
* Default gateway logic
* Basic routing concepts
* Connectivity testing with command-line tools

Commands used:

```
ipconfig /all
ping
route print
```

---

## Working State Verification

Interpretation:
The client has a valid IP configuration and a reachable default gateway on the NAT network, allowing traffic to leave the local subnet and reach the internet.

### Confirm IP Configuration

Run on the client:

```
ipconfig /all
```

Example output:

```
IPv4 Address: 10.0.1.15
Subnet Mask: 255.255.255.0
Default Gateway: 10.0.3.2
```

This confirms the client has a valid IP configuration.

---

### Verify Gateway Reachability

```
ping 10.0.3.2
```

Expected result:

```
Reply from 10.0.3.2
```

The client can reach its default gateway.

---

### Verify Internet Connectivity

```
ping 8.8.8.8
```

Expected result:

```
Reply from 8.8.8.8
```

This confirms the system can route traffic outside the local network.

---

### Observe Routing Table

```
route print
```

Key route:

```
0.0.0.0          0.0.0.0        10.0.3.2
```

This is the **default route**, which tells the system where to send traffic destined for networks outside the local subnet.

---

# Breaking the Network

Open:

```
Network Connections
→ Ethernet (NAT adapter)
→ Properties
→ Internet Protocol Version 4 (IPv4)
```

Change configuration from DHCP to manual and set:

```
IP address:      10.0.1.15
Subnet mask:     255.255.255.0
Default gateway: 10.0.1.250
DNS server:      8.8.8.8
```

The gateway `10.0.1.250` does not exist on the network.

---

# Symptoms After Break

### Internet Connectivity Failure

```
ping 8.8.8.8
```

Possible Result:

```
Destination host unreachable or failed replies, indicating the client cannot forward traffic beyond the local subnet.
```

The system cannot route traffic outside the local subnet.

---

### Verify Routing Table

```
route print
```

Default route now shows:

```
0.0.0.0          0.0.0.0        10.0.1.250
```

This indicates the system is attempting to send external traffic to a non-existent gateway.

---
### Verification Summary
After restoring the correct default gateway, the client was again able to reach off-subnet IP addresses. The routing table showed the default route pointing to 10.0.2.2, and external connectivity was successfully restored.

## Root Cause

The default gateway was incorrectly configured.

The default gateway is responsible for forwarding packets destined for networks outside the local subnet. When the gateway address is invalid, packets cannot be routed beyond the local network.

---

## Resolution

Restore the correct gateway configuration.

```
Default Gateway: 10.0.3.2
```

Alternatively, re-enable DHCP so the correct configuration is assigned automatically.

---

# Verification

Run:

```
ping 8.8.8.8
```

Expected:

```
Reply from 8.8.8.8
```

Confirm routing table:

```
route print
```

Default route should again show:

```
0.0.0.0          0.0.0.0        10.0.3.2
```

Internet connectivity is restored.

---

# Key Concepts

### Default Gateway

A default gateway is the router responsible for forwarding packets to networks outside the local subnet.

If the gateway is incorrect, off-subnet communication fails.

---

### Routing Logic

```
Same subnet → direct communication (ARP)
Different subnet → send traffic to gateway
```

If the gateway is invalid, packets destined for external networks have no route.

---

# Difference Between Gateway Issue vs DNS Issue

Gateway failure:

```
ping 8.8.8.8 → fails
```

DNS failure:

```
ping 8.8.8.8 → works
ping google.com → fails
```

Testing with IP addresses helps isolate routing problems from DNS problems.

---

# Evidence Collected

Screenshots included:

1. Working `ipconfig /all`
2. Successful ping to gateway
3. Successful ping to 8.8.8.8
4. Working routing table
5. Modified IPv4 gateway configuration
6. Failed ping to 8.8.8.8
7. Incorrect default route shown in `route print`
8. Restored gateway configuration
9. Successful connectivity after repair
