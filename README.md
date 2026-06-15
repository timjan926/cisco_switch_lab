# cisco_switch_lab

## Overview

This project documents the security hardening of two Cisco switches in a small isolated network environment. The purpose of the lab was to create a dedicated management network, restrict administrative access, disable insecure management protocols and reduce the attack surface of the switches.

The environment consisted of two Cisco switches and one MacBook used for configuration, connectivity testing and administration.

## Objectives
The objectives of the lab were to:

* Create a dedicated management VLAN.
* Configure management IP addresses on both switches.
* Configure a trunk connection between the switches.
* Permit only the management VLAN across the trunk.
* Enable SSH version 2.
* Disable Telnet access.
* Use local authentication for administrative access.
* Restrict SSH access to the management subnet.
* Shut down unused switch ports.
* Place unused ports in an isolated parking VLAN.
* Verify the configuration through connectivity and port tests.

## Lab Environment
### Hardware

* Two Cisco switches
* One MacBook
* USB-C Ethernet adapter
* Console connection for initial switch configuration
* Ethernet cable between the switches

### Software and tools

* Cisco IOS
* macOS Terminal
* OpenSSH client
* Netcat
* Ping

## Network Topology

```text
MacBook
192.168.99.100/24
        |
        | Access port VLAN 99
        |
      SW1
VLAN 99: 192.168.99.11/24
        |
        | 802.1Q trunk
        | Allowed VLAN: 99
        |
      SW2
VLAN 99: 192.168.99.12/24
```

## IP Addressing

| Device  | Interface   | VLAN | IP address        |
| ------- | ----------- | ---: | ----------------- |
| MacBook | Ethernet    |   99 | 192.168.99.100/24 |
| SW1     | VLAN 99 SVI |   99 | 192.168.99.11/24  |
| SW2     | VLAN 99 SVI |   99 | 192.168.99.12/24  |

No default gateway was required because all management devices were located in the same isolated subnet.

## VLAN Design

| VLAN | Name         | Purpose                  |
| ---: | ------------ | ------------------------ |
|   99 | MGMT         | Switch management        |
|  999 | PARKING_LOT  | Unused switch ports      |
|   10 | TEST_CLIENTS | Temporary isolation test |

* VLAN 99 was used exclusively for management traffic.

* VLAN 999 was used as a parking VLAN for interfaces that were not required during the lab. These interfaces were also administratively shut down.

* VLAN 10 was temporarily used to verify that the MacBook could not reach the management interface while connected to a different VLAN.

## Security Controls
### Dedicated management VLAN

The switch management interfaces were placed in VLAN 99 instead of the default VLAN.

This separates administrative traffic from ordinary client traffic and reduces exposure of the switch management plane.

### SSH-only management

SSH version 2 was enabled on both switches.

The VTY configuration used:

```text
login local
transport input ssh
```

The `transport input ssh` command prevents Telnet connections from being accepted.

### Local authentication

A local administrative account with privilege level 15 was configured on both switches.

### Management access control list

A standard access control list named `MGMT_ONLY` was created:

```text
ip access-list standard MGMT_ONLY
 permit 192.168.99.0 0.0.0.255
 deny any log
```

The ACL was applied to the VTY lines:

```text
line vty 0 4
 access-class MGMT_ONLY in
 login local
 transport input ssh

line vty 5 15
 access-class MGMT_ONLY in
 login local
 transport input ssh
```

This restricts remote administrative access to devices located in the `192.168.99.0/24` management subnet.

### Disabled unused ports

Unused FastEthernet ports were:

* Assigned to VLAN 999.
* Given an unused-port description.
* Administratively shut down.

Example:

```text
interface range FastEthernet0/3 - 24
 description UNUSED_SHUTDOWN
 switchport mode access
 switchport access vlan 999
 shutdown
```

This reduces the risk of unauthorized devices being connected to an active switch port.

### Restricted trunk

The connection between SW1 and SW2 was configured as an 802.1Q trunk.

Only VLAN 99 was permitted across the trunk:

```text
interface FastEthernet0/1
 description TRUNK_TO_OTHER_SWITCH
 switchport mode trunk
 switchport trunk allowed vlan 99
```

Restricting the allowed VLAN list reduces unnecessary VLAN exposure across the trunk.

## Implementation Summary

The following configuration process was completed on both switches:

1. Configured the switch hostname.
2. Disabled DNS lookup.
3. Created VLAN 99 and VLAN 999.
4. Configured the VLAN 99 management interface.
5. Assigned a management IP address.
6. Configured the trunk between SW1 and SW2.
7. Allowed only VLAN 99 across the trunk.
8. Assigned the MacBook-facing interface to VLAN 99.
9. Moved unused interfaces to VLAN 999.
10. Shut down unused interfaces.
11. Configured a local privileged administrator.
12. Generated RSA keys.
13. Enabled SSH version 2.
14. Disabled Telnet on all VTY lines.
15. Applied the `MGMT_ONLY` ACL to the VTY lines.
16. Saved the running configuration.

The complete example configurations are available in the `configs` directory.

## Verification and Test Results

| Test                                               | Expected result        | Result |
| -------------------------------------------------- | ---------------------- | ------ |
| Ping SW1 from VLAN 99                              | Reachable              | Pass   |
| Ping SW2 from VLAN 99                              | Reachable              | Pass   |
| TCP port 23 on SW1                                 | Refused or unavailable | Pass   |
| TCP port 23 on SW2                                 | Refused or unavailable | Pass   |
| TCP port 22 on SW1                                 | Open                   | Pass   |
| TCP port 22 on SW2                                 | Open                   | Pass   |
| VLAN 99 SVI on SW1                                 | Up/up                  | Pass   |
| VLAN 99 SVI on SW2                                 | Up/up                  | Pass   |
| VLAN 99 across trunk                               | Active and forwarding  | Pass   |
| SSH version 2                                      | Enabled                | Pass   |
| VTY transport protocol                             | SSH only               | Pass   |
| Local VTY authentication                           | Configured             | Pass   |
| Management ACL                                     | Applied                | Pass   |
| Unused switch ports                                | Administratively down  | Pass   |
| Management reachability from VLAN 10               | Unreachable            | Pass   |
| Management reachability after returning to VLAN 99 | Reachable              | Pass   |

## VLAN Isolation Test

The MacBook-facing port was temporarily moved from VLAN 99 to VLAN 10. While connected to VLAN 10, the MacBook could not reach the SW1 management address at `192.168.99.11`.

The interface was then returned to VLAN 99. Connectivity to the management address was restored. This test demonstrated Layer 2 isolation between the VLANs in the current lab topology.

Because no inter-VLAN router was included, this test did not independently verify the ACL against routed traffic. The ACL configuration was verified directly from the switch configuration.

## SSH Compatibility Limitation

An interactive SSH login from the MacBook was not completed.

The switches offered legacy SHA-1-based key exchange algorithms and the older `ssh-rsa` host key type. These algorithms are disabled by default in modern OpenSSH clients because they no longer meet current cryptographic recommendations.

The following were successfully verified:

* The MacBook could reach TCP port 22 on both switches.
* SSH version 2 was enabled on both switches.
* The VTY lines accepted SSH only.
* Local authentication was configured.
* The management ACL was applied.

The failure occurred during the cryptographic SSH negotiation rather than at the network or TCP layer.

Legacy SSH algorithms were not enabled globally on the MacBook.

## Security Relevance

This lab demonstrates several principles that are also important in cloud security:

* Management-plane isolation
* Least-privilege access
* Reduced attack surface
* Secure administrative protocols
* Network segmentation
* Explicit access control
* Disabled unused services and interfaces
* Configuration verification

In cloud environments, similar controls are implemented using private management networks, security groups, network access control lists, identity policies and restricted administrative endpoints.

## Skills Demonstrated

* Cisco IOS configuration
* VLAN creation and management
* 802.1Q trunk configuration
* Management network isolation
* Access control lists
* SSH configuration
* Telnet removal
* Switch-port hardening
* Network connectivity testing
* macOS command-line tools
* Troubleshooting SSH compatibility
* Security documentation

## Repository Contents

```text
configs/
```

Contains configurations for SW1 and SW2.

```text
evidence/
```

Contains terminal output and screenshots showing the verification results.

```text
diagrams/
```

Contains the network topology diagram.

## Conclusion

The lab successfully hardened the management plane of two Cisco switches by separating management traffic, disabling Telnet, enabling SSH version 2, restricting administrative access, shutting down unused interfaces and verifying the resulting network behavior.

The lab also identified a compatibility issue between legacy Cisco SSH algorithms and the security defaults used by modern OpenSSH clients.
