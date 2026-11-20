# Packet Tracer - HSRP Configuration Guide (with Redundant DHCPv4)

## 📌 Project Overview

This repository contains the complete configuration documentation, custom split-scope redundant DHCPv4 implementation, and lab verification answers for the **Packet Tracer - HSRP Configuration Guide** activity.

In addition to configuring **Hot Standby Router Protocol (HSRP)** on **R1** (Active) and **R3** (Standby) to provide first-hop gateway redundancy (`192.168.1.254`), this implementation replaces static host addressing with a **non-overlapping split-scope redundant DHCPv4 architecture** across both gateway routers so end devices automatically receive the HSRP Virtual IP as their default gateway.

## 📋 Addressing Table

| Device | Interface | IP Address | Subnet Mask / Prefix | Default Gateway |
| :--- | :--- | :--- | :--- | :--- |
| **R1** | `G0/0` | `10.1.1.1` | `255.255.255.252` (`/30`) | N/A |
| **R1** | `G0/1` | `192.168.1.1` | `255.255.255.0` (`/24`) | N/A |
| **R1** | `G0/2` | `10.1.1.9` | `255.255.255.252` (`/30`) | N/A |
| **R2** | `G0/0` | `10.1.1.2` | `255.255.255.252` (`/30`) | N/A |
| **R2** | `G0/1` | `10.1.1.5` | `255.255.255.252` (`/30`) | N/A |
| **R2** | `G0/2` | `10.100.100.1` | `255.255.255.252` (`/30`) | N/A |
| **R3** | `G0/0` | `192.168.1.3` | `255.255.255.0` (`/24`) | N/A |
| **R3** | `G0/1` | `10.1.1.6` | `255.255.255.252` (`/30`) | N/A |
| **R3** | `G0/2` | `10.1.1.10` | `255.255.255.252` (`/30`) | N/A |
| **I-Net** *(Cloud)* | `G0/1` | `10.100.100.2` | `255.255.255.252` (`/30`) | N/A |
| **HSRP Virtual Gateway** | `Virtual` | `192.168.1.254` | `255.255.255.0` (`/24`) | N/A |
| **S1** | `VLAN 1` | `192.168.1.11` | `255.255.255.0` (`/24`) | `192.168.1.254` *(Updated from `.1`)* |
| **S3** | `VLAN 1` | `192.168.1.13` | `255.255.255.0` (`/24`) | `192.168.1.254` *(Updated from `.3`)* |
| **PC-A** | `NIC` | *DHCP (`192.168.1.129` – `.253`)* | `255.255.255.0` (`/24`) | `192.168.1.254` *(via DHCP)* |
| **PC-B** | `NIC` | *DHCP (`192.168.1.14` – `.128`)* | `255.255.255.0` (`/24`) | `192.168.1.254` *(via DHCP)* |
| **Web Server** | `NIC` | `209.165.200.226` | `255.255.255.224` (`/27`) | `209.165.100.225` |

## ⚙️ Device Configurations

### 1. Router `R1` — HSRP Active Router & Primary DHCPv4 Server

`R1` is configured as the **HSRP Group 1 Active Router** (priority `150` with `preempt`) and hosts the DHCP pool **`LAN-S1`**. To prevent IP address conflicts with `R3` on the same Layer 2 broadcast domain, `R1` excludes `.1`–`.128` and `.254`, leasing addresses from the upper half of the subnet (**`192.168.1.129` to `192.168.1.253`**).

```ios
R1# configure terminal
! Configure HSRP Version 2 on LAN Interface G0/1
R1(config)# interface GigabitEthernet0/1
R1(config-if)# standby version 2
R1(config-if)# standby 1 ip 192.168.1.254
R1(config-if)# standby 1 priority 150
R1(config-if)# standby 1 preempt
R1(config-if)# exit

! Configure DHCPv4 Server (Upper Half Split-Scope: .129 - .253)
R1(config)# ip dhcp excluded-address 192.168.1.1 192.168.1.128
R1(config)# ip dhcp excluded-address 192.168.1.254
R1(config)# ip dhcp pool LAN-S1
R1(dhcp-config)# network 192.168.1.0 255.255.255.0
R1(dhcp-config)# default-router 192.168.1.254
R1(dhcp-config)# end
R1# copy running-config startup-config
```

### 2. Router `R3` — HSRP Standby Router & Secondary DHCPv4 Server

`R3` is configured as the **HSRP Group 1 Standby Router** (default priority `100`, preemption disabled) and hosts the secondary DHCP pool **`LAN-S3`**. It excludes static device addresses (`192.168.1.1`–`192.168.1.13`) as well as `R1`'s DHCP scope and the virtual gateway (`192.168.1.129`–`192.168.1.254`), cleanly leasing addresses from the lower half of the subnet (**`192.168.1.14` to `192.168.1.128`**) while advertising the HSRP Virtual IP (`192.168.1.254`) as the default gateway.

```ios
R3# configure terminal
! Configure HSRP Version 2 on LAN Interface G0/0
R3(config)# interface GigabitEthernet0/0
R3(config-if)# standby version 2
R3(config-if)# standby 1 ip 192.168.1.254
R3(config-if)# exit

! Configure DHCPv4 Server (Lower Half Split-Scope: .14 - .128)
R3(config)# ip dhcp excluded-address 192.168.1.1 192.168.1.13
R3(config)# ip dhcp excluded-address 192.168.1.129 192.168.1.254
R3(config)# ip dhcp pool LAN-S3
R3(dhcp-config)# network 192.168.1.0 255.255.255.0
R3(dhcp-config)# default-router 192.168.1.254
R3(dhcp-config)# end
R3# copy running-config startup-config
```

### 3. Switches `S1` & `S3` — Update Management Default Gateway

Update the static default gateway on both Layer 2 switches so remote management traffic routes through the HSRP virtual IP (`192.168.1.254`):

```ios
! On S1
S1(config)# ip default-gateway 192.168.1.254

! On S3
S3(config)# ip default-gateway 192.168.1.254
```

### 4. End Hosts (`PC-A` & `PC-B`) — Enable DHCP Client

1. Open **Desktop > IP Configuration** on **PC-A** and **PC-B**.
2. Switch from **Static** to **DHCP**.
3. Verify each host receives a non-overlapping IPv4 address in `192.168.1.0/24` (either from `R1`'s `.129–.253` scope or `R3`'s `.14–.128` scope) and the **Default Gateway `192.168.1.254`**.

## ❓ Lab Questions & Answers

### Part 1: Verify Connectivity (Before HSRP)

1. **Step 1 — Which devices are on the path from PC-A to the Web Server?**
   > **Answer:** **PC-A** $\rightarrow$ **R1** (`192.168.1.1`) $\rightarrow$ **R2** (`10.1.1.2`) $\rightarrow$ **I-Net** (`10.100.100.2`) $\rightarrow$ **Web Server** (`209.165.200.226`). *(Note: Layer 2 switches `S1` and `S2` are transparent to `tracert`).*

2. **Step 2 — Which devices are on the path from PC-B to the Web Server?**
   > **Answer:** **PC-B** $\rightarrow$ **R3** (`192.168.1.3`) $\rightarrow$ **R2** (`10.1.1.6`) $\rightarrow$ **I-Net** (`10.100.100.2`) $\rightarrow$ **Web Server** (`209.165.200.226`).

3. **Step 3c — What are the results when the link between R3 and S3 is deleted?**
   > **Answer:** The `tracert` fails completely (`Request timed out`) because **PC-B** is configured with `R3` (`192.168.1.3`) as its sole static default gateway and has no automatic failover path to `R1`.

### Part 2: Configure HSRP Active and Standby Routers

4. **Step 1e — What will the HSRP priority of R3 be when it is added to HSRP group 1?**
   > **Answer:** **`100`** (the default Cisco HSRP priority value).

5. **Step 3a — Using the `show standby` output, answer the following:**
   * **Which router is the active router?**
     > **R1** (`192.168.1.1`).
   * **What is the MAC address for the virtual IP address?**
     > **`0000.0C9F.F001`** (HSRP Version 2 default virtual MAC format `0000.0C9F.Fxxx`, where `001` is Group 1 in hex).
   * **What is the IP address and priority of the standby router?**
     > IP Address: **`192.168.1.3`** (`R3`), Priority: **`100`**.

6. **Step 3c — Which default gateway address should be used for PC-A, PC-B, S1, and S3, and are pings to the Web Server successful?**
   > **Answer:** **`192.168.1.254`** (the HSRP Virtual IP address, now automatically distributed to PC-A and PC-B via DHCPv4). Yes, pings from both PCs to the Web Server (`209.165.200.226`) succeed.

### Part 3: Observe HSRP Operation

7. **Step 1 — Does the path from PC-B differ from the path used before HSRP was configured?**
   > **Answer:** **Yes.** Previously, PC-B routed through `R3` (`192.168.1.3`). Now that PC-B uses the HSRP Virtual Gateway (`192.168.1.254`) and `R1` is the Active HSRP router, traffic from PC-B travels through **`R1` (`192.168.1.1`)** $\rightarrow$ `R2` (`10.1.1.2`) $\rightarrow$ `I-Net` (`10.100.100.2`) $\rightarrow$ `Web Server`.

8. **Step 2b — How was the trace different immediately after breaking the link between R1 and S1?**
   > **Answer:** The first few probes timed out (`* * *`) while HSRP waited for the **Hold Timer (`10 seconds`)** to expire without receiving Hello packets from `R1`. Once the timer expired, **`R3` transitioned from Standby to Active** and took over `192.168.1.254`, rerouting PC-B's traffic through **`R3` (`192.168.1.3`)** $\rightarrow$ `R2` (`10.1.1.6`) $\rightarrow$ `I-Net` $\rightarrow$ `Web Server`.

9. **Step 3b — After restoring the link to R1, what path is used to reach the Web Server, and would the results be the same without `preempt`?**
   > **Answer:**
   > * **Path used:** Traffic returns to the original primary path through **`R1` (`192.168.1.1`)** $\rightarrow$ `R2` (`10.1.1.2`) $\rightarrow$ `I-Net` $\rightarrow$ `Web Server`.
   > * **Without `preempt`:** **No.** If `standby 1 preempt` had not been configured on `R1`, `R3` would have remained the Active router even after `R1` came back online with a higher priority (`150`), and traffic would have continued routing through `R3`.

## 🔍 Verification Commands

```ios
! Verify HSRP state, priority, preemption, and virtual MAC
R1# show standby
R1# show standby brief
R3# show standby brief

! Verify DHCPv4 pool allocations and active client leases
R1# show ip dhcp pool
R1# show ip dhcp binding
R3# show ip dhcp pool
R3# show ip dhcp binding
```