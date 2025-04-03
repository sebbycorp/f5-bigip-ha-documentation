# F5 BIG-IP High Availability Configuration

This repository documents the high availability configuration for a pair of F5 BIG-IP devices with a sample web server virtual server setup.

## Network Architecture

### Device Information
- **Primary Device**: bigip1.example.com (172.16.10.10)
- **Secondary Device**: bigip2.example.com (172.16.10.11)
- **Software Version**: BIG-IP 17.5.0

### Network Layout

#### VLANs
- **External VLAN**: Interface 1.1
  - Used for client-facing traffic
  - Network: 172.16.20.0/24
- **Internal VLAN**: Interface 1.2
  - Used for server-facing traffic
  - Network: 172.16.30.0/24
- **HA VLAN**: Interface 1.3
  - Used for failover communication between BIG-IP devices
  - Network: 192.168.100.0/30

#### Self IPs
- **Primary Device**:
  - External Self IP: 172.16.20.5/24
  - Internal Self IP: 172.16.30.5/24
  - HA Self IP: 192.168.100.1/30
- **Secondary Device**:
  - External Self IP: 172.16.20.6/24
  - Internal Self IP: 172.16.30.6/24
  - HA Self IP: 192.168.100.2/30
- **Floating IPs** (Active on current primary):
  - External Floating IP: 172.16.20.10/24
  - Internal Floating IP: 172.16.30.10/24

#### Default Route
- Gateway: 172.16.20.1

## High Availability Configuration

### Device Trust
- A device trust has been established between the two BIG-IP devices
- Each device is aware of the other and can share configuration

### Device Group
- Name: `device-group-sync`
- Type: `sync-failover`
- Auto Sync: Enabled
- Network Failover: Enabled
- Devices: bigip1, bigip2

### ConfigSync Setup
- **Primary Device**: 
  - ConfigSync IP: 192.168.100.1
  - Mirror IP: 192.168.100.1
  - Unicast Address: 192.168.100.1:1026
- **Secondary Device**:
  - ConfigSync IP: 192.168.100.2
  - Mirror IP: 192.168.100.2
  - Unicast Address: 192.168.100.2:1026

## Security Settings

### Self IP Port Lockdown
- **Non-floating Self IPs**: Allow None
  - Restricts all traffic through these IPs
- **Floating IPs**: Allow Default
  - Allows standard F5 services through these IPs
- **HA Self IPs**: Allow Default
  - Required for proper failover communication

## Network Services

### DNS Configuration
- Primary DNS: 8.8.8.8
- Secondary DNS: 1.1.1.1

### NTP Configuration
- NTP Servers:
  - 0.north-america.pool.ntp.org
  - 1.north-america.pool.ntp.org
- Timezone: America/New_York

## Virtual Server Configuration

### HTTP Health Monitor
- Name: http-test-monitor
- Inherited From: /Common/http
- Interval: 5 seconds
- Timeout: 16 seconds
- Send String: `GET /\r\n`

### Server Pool
- Name: web-pool
- Monitor: http-test-monitor
- Load Balancing Method: Least Connections
- Members:
  - 172.16.30.100:80
  - 172.16.30.101:80
  - 172.16.30.102:80
  - 172.16.30.103:80
  - 172.16.30.104:80

### Virtual Server
- Name: web-vip
- Destination Address: 172.16.20.100:80
- Type: Standard
- Source Address Translation: Automap
- Profiles:
  - HTTP
  - TCP
- Pool: web-pool
- iRule: allow_subnet (restricts access to 172.16.20.0/24 subnet)

## Access Control iRule

The following iRule is applied to the web-vip virtual server to restrict access to only clients from the 172.16.20.0/24 subnet:

```tcl
when CLIENT_ACCEPTED {
    set client_ip [IP::client_addr]
    if { not [IP::addr $client_ip equals 172.16.20.0/24] } {
        log local0. "Connection rejected from disallowed IP: $client_ip"
        reject
    }
}
```

## Configuration Diagram

```
+---------------------------+                       +---------------------------+
|                           |                       |                           |
|     bigip1.example.com    |                       |     bigip2.example.com    |
|       (172.16.10.10)      |                       |       (172.16.10.11)      |
|                           |                       |                           |
|  +---------------------+  |                       |  +---------------------+  |
|  |                     |  |                       |  |                     |  |
|  |    Virtual Server   |  |                       |  |    Virtual Server   |  |
|  |   172.16.20.100:80  |  |                       |  |   172.16.20.100:80  |  |
|  |    (Active/Standby) |  |                       |  |    (Active/Standby) |  |
|  +---------------------+  |                       |  +---------------------+  |
|            |              |                       |            |              |
|            v              |                       |            v              |
|  +---------------------+  |                       |  +---------------------+  |
|  |                     |  |                       |  |                     |  |
|  |      web-pool       |  |                       |  |      web-pool       |  |
|  | (Least Connections) |  |                       |  | (Least Connections) |  |
|  +---------------------+  |                       |  +---------------------+  |
|                           |                       |                           |
+------------^--------------+                       +------------^--------------+
             |                                                   |
             | HA Sync, Failover                                 |
             |                                                   |
+------------+--------------+       +---------------------------+|
|                           |       |                           ||
|       External VLAN       |       |       External VLAN       ||
|   Self IP: 172.16.20.5/24 +-------+   Self IP: 172.16.20.6/24 ||
| Float IP: 172.16.20.10/24 | WAN   | Float IP: 172.16.20.10/24 ||
|                           |       |                           ||
+------------+--------------+       +------------+--------------+|
             |                                   |               |
             |                                   |               |
+------------+--------------+       +------------+--------------+|
|                           |       |                           ||
|       Internal VLAN       |       |       Internal VLAN       ||
|   Self IP: 172.16.30.5/24 |       |   Self IP: 172.16.30.6/24 ||
| Float IP: 172.16.30.10/24 |       | Float IP: 172.16.30.10/24 ||
|                           |       |                           ||
+------------+--------------+       +------------+--------------+|
             |                                   |               |
             |                                   |               |
+------------+--------------+       +------------+--------------+|
|                           |       |                           ||
|          HA VLAN          |       |          HA VLAN          ||
|   Self IP: 192.168.100.1/30+-------+   Self IP: 192.168.100.2/30|
|                           |       |                           |
+---------------------------+       +---------------------------+

 Web Servers (Pool Members):
 +-----------------------+    +-----------------------+    +-----------------------+
 |    172.16.30.100:80   |    |    172.16.30.101:80   |    |    172.16.30.102:80   |
 +-----------------------+    +-----------------------+    +-----------------------+
                             +-----------------------+    +-----------------------+
                             |    172.16.30.103:80   |    |    172.16.30.104:80   |
                             +-----------------------+    +-----------------------+
```

## Testing and Verification

To verify the HA functionality:

1. Access the virtual server at 172.16.20.100:80 from a client in the 172.16.20.0/24 subnet
2. Perform a failover test by shutting down the active device
3. Verify that traffic continues to flow through the new active device
4. Verify that the floating IPs move to the new active device

## Maintenance Considerations

- Schedule maintenance during low-traffic periods
- Use the Configuration Utility to force a failover before maintenance
- Verify synchronization status after making configuration changes
- Monitor logs for rejected connection attempts from unauthorized subnets