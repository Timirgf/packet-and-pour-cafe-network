# Packet & Pour Café Network Design

## Project Overview

This project presents a small-business network designed and configured in Cisco Packet Tracer for a fictional coffee shop. The network separates management devices, point-of-sale systems, guest wireless clients, and network-management traffic into different VLANs. A router-on-a-stick design provides gateway services between VLANs, DHCP supplies addressing to client devices, and an extended access control list restricts guest access to internal business resources.

The project demonstrates practical CCNA-level skills in network planning, VLAN segmentation, 802.1Q trunking, IPv4 addressing, DHCP, wireless security, access control, device hardening, troubleshooting, and configuration verification.

> **Lab note:** All usernames, passwords, and wireless keys shown in the screenshots are fictional training credentials created specifically for this Cisco Packet Tracer lab. They are included to document the complete configuration process and are not used on any real device or service.

## Business Requirements

The café requires separate network areas for:

- Management workstations and office devices
- Point-of-sale terminals and receipt printers
- Customer guest Wi-Fi
- Network-device management
- Upstream connectivity through an edge router

The design must allow each department to use the appropriate local resources while preventing guest devices from reaching sensitive internal networks.

## Technologies and Skills Demonstrated

- Cisco Packet Tracer
- Cisco IOS command-line configuration
- VLAN creation and access-port assignment
- IEEE 802.1Q trunking
- Router-on-a-stick inter-VLAN routing
- IPv4 subnetting with `/24` networks
- DHCP pools and excluded address ranges
- Static IPv4 addressing for infrastructure devices
- WPA2-PSK wireless security with AES encryption
- Extended IPv4 access control lists
- SSH version 2 device management
- Basic Cisco device hardening
- Ping and interface-status verification
- Saving running configurations to startup configuration

## Network Segmentation Plan

| VLAN | Name | IPv4 network | Default gateway | Purpose |
| --- | --- | --- | --- | --- |
| 10 | `Management_OFFICE` | `192.168.10.0/24` | `192.168.10.1` | Management computer and office printer |
| 20 | `POS` | `192.168.20.0/24` | `192.168.20.1` | Point-of-sale terminal and receipt printer |
| 30 | `Guest_wifi` | `192.168.30.0/24` | `192.168.30.1` | Customer wireless devices |
| 99 | `network_management` | `192.168.99.0/24` | `192.168.99.1` | Management of network infrastructure |

The switch management interface uses `192.168.99.2/24`. Static addresses are assigned to the office and receipt printers, while end-user devices obtain addresses from DHCP.

## Configuration Commands and Function Overview

The command blocks below present the completed configuration by function. They remove the mistyped commands visible in the troubleshooting screenshots and use descriptive placeholders for the lab credentials. The screenshots later in this README show the actual configuration process.

### 1. Switch identity and basic hardening

**Purpose:** Identifies the switch, prevents mistyped commands from triggering DNS lookups, protects privileged access, encrypts plaintext line passwords in the configuration, and displays an access warning.

```cisco
enable
configure terminal
hostname CoffeeShop-SW
no ip domain-lookup
enable secret <LAB_ENABLE_SECRET>
service password-encryption
banner motd #Unauthorized access is prohibited.#
```

### 2. Switch console protection

**Purpose:** Requires a password for local console access and prevents system messages from disrupting commands being typed.

```cisco
line console 0
password <LAB_CONSOLE_PASSWORD>
login
logging synchronous
exec-timeout 10 0
exit
```

### 3. VLAN creation

**Purpose:** Creates four separate Layer 2 broadcast domains so office, payment, guest, and infrastructure traffic do not share one flat network.

```cisco
vlan 10
name Management_OFFICE
exit

vlan 20
name POS
exit

vlan 30
name Guest_wifi
exit

vlan 99
name network_management
exit
```

### 4. Switch access-port assignment

**Purpose:** Places each wired endpoint or access point into the VLAN that matches its business function. The ranges reflect the assignments displayed by `show vlan brief` in the lab.

```cisco
interface range fastethernet 0/1-5
description MANAGEMENT_OFFICE_DEVICES
switchport mode access
switchport access vlan 10
spanning-tree portfast
exit

interface range fastethernet 0/6-10
description POS_DEVICES
switchport mode access
switchport access vlan 20
spanning-tree portfast
exit

interface fastethernet 0/11
description GUEST_WIRELESS_ACCESS_POINT
switchport mode access
switchport access vlan 30
spanning-tree portfast
exit
```

### 5. Switch-to-router 802.1Q trunk

**Purpose:** Carries traffic from VLANs 10, 20, 30, and 99 across one physical link between the switch and router. The router identifies the VLAN from the 802.1Q tag.

```cisco
interface gigabitethernet 0/1
description TO_COFFEESHOP_ROUTER
switchport trunk encapsulation dot1q
switchport mode trunk
switchport trunk allowed vlan 10,20,30,99
no shutdown
exit
```

> Some Packet Tracer switch models select 802.1Q automatically and do not accept `switchport trunk encapsulation dot1q`. On those models, begin with `switchport mode trunk`.

### 6. Switch management interface

**Purpose:** Gives the switch an IPv4 address in the dedicated network-management VLAN so an administrator can manage it remotely.

```cisco
interface vlan 99
description SWITCH_MANAGEMENT
ip address 192.168.99.2 255.255.255.0
no shutdown
exit
ip default-gateway 192.168.99.1
```

### 7. Secure SSH management on the switch

**Purpose:** Creates the prerequisites for encrypted remote administration, enables SSH version 2, uses the local account database, and blocks Telnet on the VTY lines.

```cisco
ip domain-name lab.local
username admin privilege 15 secret <LAB_ADMIN_SECRET>
crypto key generate rsa modulus 1024
ip ssh version 2

line vty 0 15
login local
transport input ssh
exec-timeout 10 0
exit
```

> The lab uses a 1024-bit RSA key because of Packet Tracer device limitations. A supported production device should use a stronger key size and follow the organization's current security standard.

### 8. Router identity and basic hardening

**Purpose:** Applies the same baseline protections to the café router.

```cisco
enable
configure terminal
hostname King_Coffee_shop-RTR
no ip domain-lookup
enable secret <LAB_ENABLE_SECRET>
service password-encryption
banner motd #Unauthorized access is prohibited.#

line console 0
password <LAB_CONSOLE_PASSWORD>
login
logging synchronous
exec-timeout 10 0
exit
```

### 9. Physical router interfaces

**Purpose:** Enables the upstream interface and the physical interface carrying the VLAN trunk. An IP address is not placed directly on `G0/1` because its subinterfaces provide the VLAN gateways.

```cisco
interface gigabitethernet 0/0
description TO_ISP
no shutdown
exit

interface gigabitethernet 0/1
description TO_COFFEESHOP_SWITCH
no ip address
no shutdown
exit
```

### 10. Router-on-a-stick subinterfaces

**Purpose:** Creates one logical router interface for each VLAN. Each subinterface uses the matching 802.1Q VLAN tag and becomes the default gateway for that subnet.

```cisco
interface gigabitethernet 0/1.10
description MANAGEMENT_OFFICE_GATEWAY
encapsulation dot1Q 10
ip address 192.168.10.1 255.255.255.0
exit

interface gigabitethernet 0/1.20
description POS_GATEWAY
encapsulation dot1Q 20
ip address 192.168.20.1 255.255.255.0
exit

interface gigabitethernet 0/1.30
description GUEST_WIFI_GATEWAY
encapsulation dot1Q 30
ip address 192.168.30.1 255.255.255.0
exit

interface gigabitethernet 0/1.99
description NETWORK_MANAGEMENT_GATEWAY
encapsulation dot1Q 99
ip address 192.168.99.1 255.255.255.0
exit
```

### 11. DHCP excluded addresses

**Purpose:** Prevents DHCP from leasing the first twenty addresses in each client subnet. These addresses remain available for gateways, printers, access points, and other devices that need predictable static addresses.

```cisco
ip dhcp excluded-address 192.168.10.1 192.168.10.20
ip dhcp excluded-address 192.168.20.1 192.168.20.20
ip dhcp excluded-address 192.168.30.1 192.168.30.20
```

### 12. DHCP pools

**Purpose:** Automatically gives office, POS, and guest clients an address, subnet mask, default gateway, and DNS server appropriate to their VLAN.

```cisco
ip dhcp pool Management_OFFICE
network 192.168.10.0 255.255.255.0
default-router 192.168.10.1
dns-server 8.8.8.8
exit

ip dhcp pool POS
network 192.168.20.0 255.255.255.0
default-router 192.168.20.1
dns-server 8.8.8.8
exit

ip dhcp pool Guest_wifi
network 192.168.30.0 255.255.255.0
default-router 192.168.30.1
dns-server 8.8.8.8
exit
```

> The configuration screenshot shows `0.0.0.0` entered as the DNS server during the lab. The completed command block uses a valid example DNS address so the documented configuration is operational.

### 13. Guest-network access control

**Purpose:** Allows guest devices to obtain DHCP information, blocks the guest subnet from reaching the office, POS, and network-management subnets, and permits traffic to other destinations. ACL order matters because IOS evaluates entries from top to bottom and stops at the first match.

```cisco
ip access-list extended GUEST_RESTRICTIONS
permit udp any eq bootpc any eq bootps
deny ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255
deny ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255
deny ip 192.168.30.0 0.0.0.255 192.168.99.0 0.0.0.255
permit ip 192.168.30.0 0.0.0.255 any
exit

interface gigabitethernet 0/1.30
ip access-group GUEST_RESTRICTIONS in
exit
```

The ACL is applied inbound because the router should evaluate guest traffic as it enters the guest gateway. This prevents unauthorized packets from being routed deeper into the internal network.

### 14. Wireless access-point settings

**Purpose:** Creates a recognizable guest wireless network and requires encrypted WPA2 authentication. Packet Tracer configures this access point through its graphical interface rather than Cisco IOS.

| Setting | Lab value |
| --- | --- |
| SSID | `coffeeshop-guest` |
| Authentication | WPA2-PSK |
| Encryption | AES |
| Pre-shared key | Fictional lab credential shown in the screenshot |
| Switch VLAN | VLAN 30, Guest Wi-Fi |

### 15. Static printer configurations

**Purpose:** Uses predictable addresses for business printers so authorized devices can consistently locate them. Both addresses are inside the router's DHCP-excluded ranges.

| Device | IPv4 address | Mask | Gateway | VLAN |
| --- | --- | --- | --- | --- |
| Office printer | `192.168.10.10` | `255.255.255.0` | `192.168.10.1` | 10 |
| Receipt printer | `192.168.20.10` | `255.255.255.0` | `192.168.20.1` | 20 |

These settings are entered under the endpoint's **Config** tab in Packet Tracer; they are not IOS commands.

### 16. Endpoint DHCP configuration

**Purpose:** Allows the guest laptops and other dynamic clients to request the correct network settings from the router.

In Packet Tracer, open the endpoint and select:

```text
Desktop > IP Configuration > DHCP
```

For a wireless laptop, first select `coffeeshop-guest`, enter the fictional lab pre-shared key, connect, and then request DHCP addressing on `Wireless0`.

### 17. Verification and testing commands

**Purpose:** Confirms that the configuration exists, interfaces are operational, clients receive leases, and the guest ACL enforces the intended policy.

Run on the switch:

```cisco
show vlan brief
show interfaces trunk
show ip interface brief
show running-config
show ip ssh
```

Run on the router:

```cisco
show ip interface brief
show ip dhcp pool
show ip dhcp binding
show access-lists GUEST_RESTRICTIONS
show running-config
```

Run from a guest PC command prompt:

```text
ipconfig
ping 192.168.30.1
ping 192.168.10.21
ping 192.168.20.21
```

The expected result is successful communication with the guest gateway and failed communication with protected office and POS destinations.

### 18. Saving the configuration

**Purpose:** Copies the active running configuration into NVRAM so it survives a device restart.

```cisco
end
copy running-config startup-config
```

When IOS asks for the destination filename, press **Enter** to accept `startup-config`.

## Logical Topology

The completed topology connects an upstream cloud to the café router, the router to the central switch, and the switch to wired business devices and a wireless access point. The access point serves two guest laptops. The central switch carries VLAN traffic toward the router, while router subinterfaces act as the default gateways for the four VLANs.

![Completed logical topology](<Screenshots/topology.png>)

## Implementation and Screenshot Review

### 1. Initial switch hardening

The switch is renamed `CoffeeShop-SW`, DNS lookup is disabled, an enable secret is configured, and password encryption is enabled. These settings establish a recognizable device identity and reduce accidental DNS lookups caused by mistyped IOS commands.

![Initial switch hardening](<Screenshots/1.png>)

### 2. Message-of-the-day banner

This screenshot shows the creation of a message-of-the-day banner warning that unauthorized access is prohibited. The first attempts generate an error because the command is entered from the wrong IOS mode. The final attempt succeeds after entering global configuration mode, demonstrating troubleshooting through command-context awareness.

![Switch MOTD banner](<Screenshots/2.png>)

### 3. Console access configuration

The switch console line is protected with authentication, and `logging synchronous` is enabled. Logging synchronization keeps system messages from interrupting commands being typed at the console.

![Switch console access configuration](<Screenshots/3.png>)

### 4. Developing the physical and logical layout

This image captures the network during development. The router, switch, wired endpoints, printers, and wireless area are being organized into a practical small-business topology before the final configuration and testing stages.

![Early topology development](<Screenshots/5.png>)

### 5. Configuring the switch trunk

GigabitEthernet `G0/1` is described as the link to the café router and configured as an 802.1Q trunk. VLANs 10, 20, 30, and 99 are explicitly allowed across the trunk. The screenshot also records the correction made after IOS rejected trunk mode while encapsulation remained set to automatic.

![Switch trunk configuration](<Screenshots/6.png>)

### 6. Management SVI configuration

The switch creates VLAN interface 99 as its management SVI and assigns `192.168.99.2/24`. This gives the switch a dedicated management address that is separated from user and point-of-sale traffic.

![Switch management SVI](<Screenshots/7.png>)

### 7. RSA keys and SSH prerequisites

The switch receives a local domain name and a privileged local administrator account before generating a 1024-bit RSA key pair. These are prerequisites for SSH access in the Packet Tracer lab.

![SSH RSA key generation](<Screenshots/8.png>)

### 8. SSH version 2 and VTY restrictions

SSH version 2 is enabled, the VTY lines use the local user database, and remote access is restricted to SSH. This avoids insecure Telnet access and demonstrates safer remote-administration practices.

![SSH version 2 and VTY lines](<Screenshots/9.png>)

### 9. VTY inactivity timeout

An execution timeout of ten minutes is applied to the VTY lines. The screenshot includes several incorrect command attempts before the command is entered in the correct line-configuration mode, documenting the troubleshooting process.

![VTY inactivity timeout](<Screenshots/10 - end of witch config.png>)

### 10. VLAN and port verification

The `show vlan brief` output verifies that VLANs 10, 20, 30, and 99 exist and are active. It also shows the access-port ranges assigned to the management, POS, guest Wi-Fi, and network-management segments.

![VLAN brief verification](<Screenshots/11- Vlan brief.png>)

### 11. Switch interface-status verification

The `show ip interface brief` output is used to check physical and logical interface states. Connected access ports appear up/up, while unused ports remain down. The VLAN 99 SVI has the expected management address; its protocol status should be rechecked after confirming that VLAN 99 has an active forwarding port.

![Switch interface status](<Screenshots/12- .png>)

### 12. Running-configuration review

The switch running configuration is reviewed to confirm that the hostname, encrypted-password setting, interface configuration, VLAN information, and remote-management settings were entered as intended.

![Switch running configuration](<Screenshots/13.png>)

### 13. Router identity and baseline security

The router is renamed `King_Coffee_shop-RTR`, DNS lookup is disabled, an enable secret is configured, password encryption is enabled, and an unauthorized-access banner is added. This applies a consistent security baseline to the routing device.

![Router baseline configuration](<Screenshots/14.png>)

### 14. Router console security

The router console is configured to require authentication, and logging synchronization is enabled. The screenshot also demonstrates correcting a command that was initially entered from privileged EXEC mode instead of global configuration mode.

![Router console security](<Screenshots/15.png>)

### 15. Upstream router interface

GigabitEthernet `G0/0` is described as the ISP-facing interface and enabled with `no shutdown`. The link transitions to the up state, confirming Layer 1 and Layer 2 connectivity to the simulated upstream cloud.

![Router upstream interface](<Screenshots/16.png>)

### 16. VLAN 10 router subinterface

Subinterface `G0/1.10` is created for the management/office VLAN. It uses 802.1Q tag 10 and the gateway address `192.168.10.1/24`. This begins the router-on-a-stick configuration.

![VLAN 10 router subinterface](<Screenshots/17.png>)

### 17. VLAN 20 router subinterface

Subinterface `G0/1.20` is assigned to the POS network using 802.1Q tag 20 and gateway address `192.168.20.1/24`. This keeps business payment devices logically separated from office and guest systems.

![VLAN 20 router subinterface](<Screenshots/18.png>)

### 18. VLAN 30 router subinterface

Subinterface `G0/1.30` is configured for guest Wi-Fi with address `192.168.30.1/24`. An initial overlapping-address entry is corrected, showing how IOS error feedback was used to fix the configuration.

![VLAN 30 router subinterface](<Screenshots/19.png>)

### 19. VLAN 99 router subinterface

Subinterface `G0/1.99` is configured with 802.1Q tag 99 and address `192.168.99.1/24`. This becomes the default gateway for the network-management VLAN.

![VLAN 99 router subinterface](<Screenshots/20.png>)

### 20. Router subinterface verification

The router's interface summary confirms that `G0/1.10`, `G0/1.20`, `G0/1.30`, and `G0/1.99` are configured with the correct gateway addresses and are operational. This verifies that tagged traffic can reach the appropriate Layer 3 gateway.

![Router interface verification](<Screenshots/21.png>)

### 21. DHCP address exclusions

The first twenty addresses in VLANs 10, 20, and 30 are excluded from dynamic assignment. Reserving these ranges prevents DHCP from assigning addresses intended for gateways, printers, access points, or other infrastructure.

![DHCP exclusions](<Screenshots/22.png>)

### 22. Management DHCP pool

The management/office DHCP pool is created for `192.168.10.0/24` with `192.168.10.1` as its default gateway. The screenshot shows the pool-building process and the correction of an initially mistyped DHCP command.

![Management DHCP pool](<Screenshots/23.png>)

### 23. POS and guest DHCP pools

Separate DHCP pools are configured for the POS and guest networks. Each pool uses its VLAN-specific network and default gateway, allowing clients to receive addressing appropriate to their security zone.

![POS and guest DHCP pools](<Screenshots/24.png>)

### 24. DHCP pool verification

DHCP pool output confirms the address ranges and exclusions for the three client networks. This provides a control-plane check before validating leases from the endpoint side.

![DHCP pool verification](<Screenshots/25 dhcp pool.png>)

### 25. Guest access-control policy

An extended ACL named `GUEST_RESTRICTIONS` is created for the guest subnet. Its intent is to allow DHCP, block guest traffic to the management, POS, and network-management networks, and permit other guest traffic. Before publication, capture `show access-lists GUEST_RESTRICTIONS` to verify the final wildcard masks, rule order, and hit counters.

![Guest ACL creation](<Screenshots/26 Assess control.png>)

### 26. Applying the guest ACL

The guest restriction ACL is applied inbound on router subinterface `G0/1.30`. Placing the extended ACL near the guest source prevents unauthorized traffic from traveling farther into the internal network.

![Applying the guest ACL](<Screenshots/27 ACL.png>)

### 27. Saving the router configuration

The running configuration is copied to startup configuration. This ensures that the router retains the completed lab configuration after a reload.

![Saving the router configuration](<Screenshots/28 saving Config.png>)

### 28. Access-point wireless security

The access point is configured to use WPA2-PSK authentication with AES encryption. This protects the guest wireless network from unauthenticated association in the lab.

![Access point WPA2 configuration](<Screenshots/Access point password config.png>)

### 29. Guest DHCP validation

The wireless laptop reports a successful DHCP request on its wireless interface. This validates the path from the wireless client through the access point and switch to the router's guest DHCP service.

![Guest DHCP validation](<Screenshots/DHCP CHECK to pc.png>)

### 30. Guest SSID configuration

The access point SSID is renamed `coffeeshop-guest` and paired with WPA2-PSK/AES security. A descriptive SSID helps users identify the intended guest service while the VLAN and ACL enforce separation behind it.

![Guest SSID configuration](<Screenshots/SSID Renamee.png>)

### 31. Guest gateway reachability and internal blocking

The guest workstation successfully reaches its own gateway at `192.168.30.1`, proving local VLAN and gateway connectivity. Attempts to reach an internal management address fail, which supports the intended guest-isolation policy.

![Guest ACL test](<Screenshots/acl check on guest pc.png>)

### 32. Guest-to-POS blocking

The guest workstation attempts to reach a POS address at `192.168.20.21` and receives destination-unreachable responses from the guest gateway. This provides endpoint-level evidence that guest traffic cannot access the payment network.

![Guest to POS ACL test](<Screenshots/acl check on guest pc ii.png>)

### 33. Discovering the secured wireless network

The laptop detects `coffeeshop-guest` as a WPA2-PSK wireless network. This verifies SSID broadcast, wireless coverage, and recognition of the configured security type.

![Wireless network discovery](<Screenshots/connect to access point from laptop i.png>)

### 34. Entering the wireless pre-shared key

The laptop is prompted for the WPA2 pre-shared key before joining the network. This demonstrates that wireless security is being enforced rather than allowing an open guest association.

![Wireless authentication](<Screenshots/connect to access pont from laptop.png>)

### 35. Wireless laptop hardware configuration

The laptop configuration shows the endpoint prepared for wireless connectivity. This supports the later screenshots that demonstrate SSID discovery, authentication, and DHCP assignment.

![Wireless laptop configuration](<Screenshots/laptop config.png>)

### 36. Receipt-printer gateway and DNS settings

The receipt printer is assigned the POS gateway `192.168.20.1` and a DNS server. A fixed network configuration is appropriate for a business printer because terminals and administrators need a predictable destination address.

![Receipt printer gateway configuration](<Screenshots/pos printer static ip config.png>)

### 37. Receipt-printer static IPv4 address

The receipt printer uses static address `192.168.20.10/24`. The address falls within the DHCP-excluded infrastructure range, preventing a duplicate assignment to a dynamic client.

![Receipt printer static IPv4 address](<Screenshots/pos printer static ip config ii.png>)

### 38. Office-printer gateway and DNS settings

The office printer is assigned management gateway `192.168.10.1` and a DNS server. This places the device in the management/office network rather than the guest or payment segment.

![Office printer gateway configuration](<Screenshots/printer static ip config for management.png>)

### 39. Office-printer static IPv4 address

The office printer uses static address `192.168.10.10/24`. Like the receipt printer, it is placed inside the excluded address range to protect the static assignment from DHCP conflicts.

![Office printer static IPv4 address](<Screenshots/printer static ip config for management ii.png>)

### 40. Connectivity troubleshooting and validation

The command prompt records initial timed-out tests followed by successful replies from `192.168.10.1`. The first tests were confusing because the endpoint was still using a previous static IPv4 configuration. After changing the endpoint to DHCP and confirming that it received the correct address, mask, and gateway for its VLAN, the gateway ping succeeded.

![Connectivity testing](<Screenshots/test connectivity.png>)

### 41. Final topology review

The final logical view shows the upstream cloud, router, central switch, wireless access point, guest laptops, management workstation and printer, POS terminal, and receipt printer. It gives recruiters a quick visual summary of the completed design and the business purpose of each endpoint.

![Final network topology](<Screenshots/topology.png>)


## Verification Summary

| Test | Evidence | Result |
| --- | --- | --- |
| VLAN creation and port assignment | `show vlan brief` | VLANs 10, 20, 30, and 99 are active |
| Router subinterfaces | `show ip interface brief` | Four VLAN gateways are up/up |
| Guest DHCP | Laptop IP configuration | DHCP request succeeds on the wireless interface |
| Guest Wi-Fi security | Wireless monitor and AP configuration | WPA2-PSK with AES is enabled |
| Guest gateway access | Ping to `192.168.30.1` | Successful |
| Guest-to-management isolation | Ping to `192.168.10.21` | Blocked/unreachable |
| Guest-to-POS isolation | Ping to `192.168.20.21` | Blocked/unreachable |
| Management gateway access | Ping to `192.168.10.1` | Successful after troubleshooting |
| Configuration persistence | `copy running-config startup-config` | Router configuration saved |

## Troubleshooting Demonstrated

Building this lab involved several problems that required me to slow down, read the Cisco IOS messages, check my addressing, and test one layer at a time. I kept these issues in the project because they show the troubleshooting process behind the completed network rather than presenting only the final commands.

### DNS value entered during DHCP configuration

While creating the router's DHCP pools, I entered `0.0.0.0` as a placeholder for the DNS server because this Cisco Packet Tracer exercise focused on VLAN segmentation, DHCP address assignment, inter-VLAN routing, wireless connectivity, and access-control testing. The lab tests used IPv4 addresses directly, so DNS name resolution was not required to prove that the VLAN gateways and ACL were working.

`0.0.0.0` is not a functional DNS resolver. It did not prevent the IP-based ping tests in this lab, but a client receiving that value would not have a valid server for translating a domain name such as `example.com` into an IP address. The intended DNS value for the completed lab was Google's public DNS server, `8.8.8.8`, which I also used in the static printer configurations.

The corrected DHCP option is:

```cisco
dns-server 8.8.8.8
```

This taught me to verify every DHCP option from the client side instead of assuming that a successful address lease means all settings are correct. A client can receive an IPv4 address and default gateway successfully but still fail to resolve domain names if the DNS information is wrong.

In a real environment, the correct DNS choice would depend on the organization's design:

| DNS option | Example | Appropriate use |
| --- | --- | --- |
| Internal DNS server | An organization's Windows Server DNS address | Best when internal devices must resolve company hostnames or Active Directory services |
| Router or firewall DNS proxy | The LAN address of the router or firewall | Useful when the gateway forwards client DNS requests to approved upstream resolvers |
| ISP-provided DNS | Addresses supplied by the Internet service provider | Suitable when the organization chooses to use the provider's DNS service |
| Google Public DNS | `8.8.8.8` | A valid public resolver suitable for this completed demonstration lab |
| Cloudflare DNS | `1.1.1.1` | Another valid public resolver that could be used in a general lab |

For this project, I would replace the placeholder with:

```cisco
ip dhcp pool Management_OFFICE
 dns-server 8.8.8.8
 exit

ip dhcp pool POS
 dns-server 8.8.8.8
 exit

ip dhcp pool Guest_wifi
 dns-server 8.8.8.8
 exit
```

This explanation documents the exact lab condition while also showing that I understand what DNS does, why `0.0.0.0` is not a production DNS value, and what should be configured in a functional network.

### Endpoint left on static addressing

One of the most confusing problems occurred when I tested connectivity from an endpoint that was still configured with a static address. I had already built the DHCP pools, so I expected the computer to receive the new network settings automatically. Because I forgot to change the endpoint from **Static** to **DHCP**, the device continued using its old information and the first ping tests timed out.

I corrected the problem by opening the endpoint's IP configuration, selecting **DHCP**, and confirming that the device received:

- An address from the correct VLAN subnet
- The correct `255.255.255.0` subnet mask
- The VLAN's router subinterface as its default gateway
- The intended DNS server

After renewing the configuration, I repeated the ping test and received successful replies. This reinforced an important troubleshooting lesson: always verify the source device's IP configuration before assuming the router, switch, trunk, or ACL is causing the failure.

### Commands entered from the wrong IOS mode

Several commands initially failed because I entered them from the wrong command level. For example, global commands such as `banner motd` and interface or line settings must be entered from their correct configuration contexts. I used the router or switch prompt to determine my current mode, returned to global configuration with `configure terminal`, and then entered the appropriate interface or line mode.

### Switch trunk encapsulation set to automatic

When I first entered `switchport mode trunk`, IOS rejected the command because the trunk encapsulation was still set to `Auto`. I corrected it by explicitly selecting 802.1Q before enabling trunk mode:

```cisco
switchport trunk encapsulation dot1q
switchport mode trunk
switchport trunk allowed vlan 10,20,30,99
```

This showed me that the required trunk commands can depend on the switch model and its current encapsulation setting.

### Overlapping router subinterface address

During the router-on-a-stick configuration, I accidentally attempted to place the VLAN 30 subinterface in the VLAN 20 network. IOS reported that the address overlapped with `G0/1.20`. I reviewed the addressing plan and corrected `G0/1.30` to use `192.168.30.1/24`.

This error reinforced the importance of checking that each VLAN has a unique, non-overlapping subnet before configuring interfaces.

### Incomplete or mistyped verification commands

Some `show` commands failed because I used incomplete syntax or typed a space where IOS expected a hyphen. For example, the correct command is:

```cisco
show running-config
```

I used IOS error markers and command help to correct the syntax. I then verified the network with commands such as `show vlan brief`, `show ip interface brief`, and `show ip dhcp pool`.

### Guest ACL validation

After creating the guest ACL, I did not rely only on the configuration screen. I tested from a guest endpoint. The guest device could reach its gateway at `192.168.30.1`, but attempts to reach devices in the management and POS networks failed. These results helped separate an intentional ACL denial from a general connectivity failure.

### Configuration save command

I initially shortened the destination name incorrectly when saving the router configuration. I corrected the command to:

```cisco
copy running-config startup-config
```

Saving the configuration ensures that the router keeps the completed settings after a restart.

### Troubleshooting approach learned

The lab helped me develop a more organized testing process:

1. Check the endpoint's IP address, subnet mask, gateway, and DHCP/static setting.
2. Ping the local default gateway.
3. Verify the switch access VLAN and trunk.
4. Confirm that the router subinterface is up/up and uses the correct subnet.
5. Check DHCP bindings and pool settings.
6. Review ACL direction, sequence, and wildcard masks.
7. Retest after each correction and document the result.

The most important lesson was that a failed ping does not immediately identify the failed component. I learned to verify the endpoint first and then move through the network one layer at a time.

## Known Limitations and Recommended Improvements

The available screenshots support the switching, VLAN, DHCP, wireless, ACL, and local connectivity portions of the project. The following improvements would make the repository stronger:

1. Replace the temporary `0.0.0.0` DHCP DNS entry with the intended `8.8.8.8` value in the saved Packet Tracer configuration.
2. Add `show interfaces trunk` evidence to verify trunk state and allowed VLANs.
3. Add `show access-lists GUEST_RESTRICTIONS` after testing so rule order and hit counters are visible.
4. Confirm the switch VLAN 99 SVI is up/up and configure an appropriate switch default gateway if remote management is required.
5. Add NAT/PAT, a default route, and a successful outside-connectivity test before claiming Internet access.
6. Crop screenshots to Packet Tracer and the relevant output so desktop icons, Finder windows, browser content, and unrelated applications are not visible.
7. Add the Packet Tracer `.pkt` file and router/switch configuration exports to the repository.
8. Rename screenshots with short, consistent lowercase filenames before the final GitHub push.

## Recommended Repository Structure

```text
packet-and-pour-cafe-network/
├── README.md
├── packet-and-pour-cafe.pkt
├── configs/
│   ├── router-running-config.txt
│   └── switch-running-config.txt
└── screenshots/
    ├── 01-final-topology.png
    ├── 02-vlan-verification.png
    ├── 03-trunk-configuration.png
    ├── 04-router-subinterfaces.png
    ├── 05-dhcp-pools.png
    ├── 06-guest-wifi-security.png
    ├── 07-guest-dhcp-success.png
    ├── 08-guest-management-blocked.png
    └── 09-guest-pos-blocked.png
```

## Key Takeaways

This project strengthened my ability to translate a small-business scenario into a segmented network design. I practiced building VLANs, assigning ports, configuring a trunk, creating router subinterfaces, delivering addresses through DHCP, securing guest Wi-Fi, restricting guest access with an ACL, and validating the finished design from both Cisco IOS and endpoint devices. Most importantly, I documented the troubleshooting process and verified that guest users could reach their own gateway while remaining isolated from internal management and POS resources.

## Author

**Oluwatimilehin Daramola**  
Cybersecurity and networking professional building hands-on skills in Cisco networking, infrastructure support, troubleshooting, and security operations.
