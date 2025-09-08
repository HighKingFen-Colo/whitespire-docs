# Link Aggregation Control Protocol (LACP)

## LACP Basics
Link aggregation is a technique that bonds multiple physical network interfaces into a single logical interface to increase bandwidth and provide redundancy, while Link Aggregation Control Protocol (802.3ad) is a protocol used to dynamically manage link aggregations.

### How LACP Works
LACP works by negotiating which physical network interfaces form part of a Link Aggregation Group (LAG). Each LAG can have a max of 8 active interfaces and an additional 8 standby interfaces. The active interfaces are the only ones actively transmitting data, and in the case one of the active interfaces goes down, a standby interface can become promoted to active. If there are no standby interfaces, LACP will simply reuse the remaining active interfaces.

LACP dynamically adjusts to the available active interfaces by using a hashing algorithm to decide which interface to route a connection down. LACP, however, will not be able to route the packets belong to one network connection down multiple interfaces. LACP has three primary hashing algorithms:

- L2: hashes MAC Address of both the client and server
- L2 + L3: hashes MAC Address + IP Address of both the client and server
    - This means multiple IPs on the same server (different services?) will possibly be routed to different interfaces
    - This also means multiple servers sharing the same IP (e.g. load balancing) will possibly be routed to different interfaces
- L3 + L4: hashes IP Address + port of both the client and server
    - This allows multiple connections from the same service to route to different interfaces, as the server IP, server port, and client IP will stay the same, but client port will change
    - This feature is only typically supported on high end enterprise switches
    - This feature also technically violates the LACP RFC, as it can cause packets to become delivered out-of-order. While this won't break any of your connections (as out-of-order delivery is an unavoidable problem on networks), the latency penalty can be significant

These hashing algorithms only apply to egress. That means the switch has to pick a hashing algorithm for how it sends packets to the server, and the server for the packets it sends to the switch.

### LACP Benefits
- Increased Bandwidth: Aggregate multiple connections (e.g., 2 x 2.5 Gbps = 5 Gbps)
- Redundancy: If one link fails, you still remain connected (albeit at reduced bandwidth)

### LACP Drawbacks
- Switch Support: LACP requires support from your switch. Most enterprise switches will support LACP, and the entire UniFI lineup other than the  USW-Flex, USW-Flex-Mini and USW-Ultra
- Spotty Sleep Support: LACP is primarily designed for applications where the device is not expected to go to sleep. When a device goes to sleep, it is possible that the LACP negotiation stops, and the device effectively becomes disconnected. Re-negotiation can take 30+ seconds, making the delay significant when waking from sleep. This can also break Wake-on-LAN (WOL) - which is a feature to wake up your server using a magic network packet - as your device is disconnected
- No Bandwidth Increase for Single Connection: LACP routes an entire TCP/UDP connection down a single physical network interface, which it picks using a hashing algorithm. You will need at least as many connections as you have physical network interfaces to utilize the full bandwidth
    - If your only goal was for a single VM to access your NAS with twice the bandwidth, LACP will not provide that with the default hash algorithm

## Plugging in the cables
Plug in the two ethernet cables between the GMKtec K9 and the UniFi Enterprise 24 PoE. Make sure the two ports chosen on the switch are adjacent to each other.

!!! warning "UniFi switches only support aggregation between adjacent ports"

## Configuring Proxmox
!!! warning "Configure LACP on Proxmox First"
    You should always configure LACP from the leaf (edge-most device) to the root (router). This is because once you configure something for LACP, you will lose access to all downstream devices unless the downstream devices are also configured for LACP.

Open your Proxmox WebUI : https://yourproxmoxIP:8006 and select the Proxmox server, then the System / Network option.

Edit the vmbr0 Linux bridge and delete the network interface in the bridge ports section.

!!! warning "Do not hit apply until configuration is complete. An incomplete configuration will cause you to lose web access to your server"

Create a Linux bond. In the slaves section, type out the names of your network interfaces. Set LACP as the mode, and configure layer2+3 as the hash policy.

!!! question "Why layer2+3?"
    While layer3+4 will help with multi-connection file transfers, it will only help write throughput. UniFi switches only support layer2 hashing, so reads will remain bottlenecked. This combined with the possible latency hit means it's probably not worthwhile.

Now go back to editing the vmbr0 Linux bridge and add bond0 in the bridge ports section. We needed to remove the previous network interface so that we could add it to the Linux bond.

Now hit apply.

## Configuring UniFi
Open the port manager on your UniFi web interface.

Select the switch you are trying the manage.

Select one of the two ports in the LAG.

In the operation dropdown, select the Aggregate option. Select the other port you wish to combine in the LAG.

Now hit apply changes.

!!! info "You many need to wait several minutes for the LAG to form"

!!! info "UniFi layer3+4 Hashing"
    Some UniFi switches (including the UniFi Enterprise 24 PoE) unofficially support layer3+4 hashing via the command line configuration. However, this setting does not persist across reboots and is not officially supported

## LACP Alternatives

### Static Aggregation
!!! info "This is not supported on UniFi switches"

For more information, refer to the following Wikipedia article: [link aggregation types](https://en.wikipedia.org/wiki/Link_aggregation#Linux_drivers)

### Multi-Chassis Link Aggregation Group (MLAG)
!!! info "This is not supported on UniFi switches"

LACP only supports aggregation between a server an a single switch, which means the switch serves as a single point of failure. MLAG allows aggregation between ports on different switches, which removes the switch-side single point of failure

### Multi-path I/O (MPIO)
MPIO refers to when the application itself is aware of the multiple physical paths between the client and server. Instead of relying on LAG to present a virtual interface, the application is capable of using the additional information about the physical layout of the network to improve throughput and resiliency. For example, for a storage protocol like iSCSI, an application aware of two distinct physical links can chunk a file into two distinct parts, open a connection on each interface, and re-combine the file on the other end. This guarantees that the application is able to make full use of available bandwidth.

However, most applications do not support MPIO, so this does not provide additional resiliency or bandwidth for those applications. MPIO is most commonly used for storage applications, where you may have a single client and server that you wish to maximize the bandwidth of. The classic setup is LACP for the control plane and application network, and MPIO for a dedicated storage network.

As the GMKtec K9 only has two ports, the value of resiliency and improved bandwidth in every other application outweighs the benefits of increased throughput for point-to-point storage applications

!!! info "Virtual MPIO"
    It is possible to give your VM two virtual NICs instead, and setup MPIO on the VM. The NICs would have two distinct MAC addresses, which could be hashed differently and take different paths down the LACP link. This would not help if your Proxmox install itself was the one that needed MPIO (in the case you wanted network storage for your VM datastore)