## Why NSX
NSX-T Data Center network virtualization programmatically creates and manages virtual networks.

With network virtualization, the functional equivalent of a network hypervisor reproduces the complete set of Layer 2 through Layer 7 networking services (for example, switching, routing, access control, firewalling, QoS) in software.

**NSX-T Data Center** works by implementing three separate but integrated planes: **management, control, and data**. These planes are implemented as a set of processes, modules, and agents residing on two types of nodes: **NSX Manager and transport nodes.**
Every node hosts a management plane agent.
NSX Manager nodes host API services and the management plane cluster daemons.
NSX Controller nodes host the central control plane cluster daemons.
Transport nodes host local control plane daemons and forwarding engines.

## Component
**NSX Manager**
- The NSX Manager provides the graphical user interface (GUI) and the REST APIs for creating, configuring, and monitoring NSX components, such as controllers, segments, and edge nodes.

**NSX Controller**
- The NSX Controller is an advanced distributed state management system that controls virtual networks and overlay transport tunnels. The NSX Controller function operates as a separate process within the NSX Manager cluster.

## Other components
### Transport Node
Any hypervisor or physical server including edge nodes which are participating in a NSX-T datacenter are called NSX-T transport nodes.
### Transport zone
**VLAN**
- A segment created in a VLAN transport zone will be a *VLAN backed segment*. This means that traffic between two VMs on two different hosts attached to this segment will be carried over a VLAN between the two hosts in native IEEE encapsulation. The resulting constraint is that an appropriate VLAN needs to be provisioned in the physical infrastructure for those two VMs to communicate at layer2 over this segment.
**Overlay**
- A segment created in an Overlay transport zone will be an *Overlay segment*. This means that two VMs on two different hosts attached to this segment will have their layer2 traffic carried by tunnel between their hosts. Once the VLAN used by the tunnel has been provisioned on the physical infrastructure, there is no need for additional segment-specific configuration on the physical infrastructure.

### T0 or Tier-0
data which goes out our datacenter is usually called as north-south. As NSX is software defined network solution, it uses two types of routers. First one is T0 or Tier-0, which establishes neighbor ship with underlay network. Any communication which comes to NSX-T datacenter for data networks comes using T0 routers. Hence for NSX-T datacenter T0 takes care of all north-south data traffic.

### T1 or Tier-1
Takes care of communication with in NSX-T datacenter, which is termed as east-west traffic. For example machine A with ip 192.168.1.2/24 gateway 192.168.1.254 is talking to 192.168.1.3/24 gateway 192.168.1.254 is east west traffic which will be with in same broadcast domain.
### Segment - Layer 2 broadcast domain
- Just like port group in vSwitch but for NSX

### Edge Sub-Cluster
This refers to two Edge nodes that are in a sub-cluster. It is automatically created and defined by the NSX manager. Note that when you create an Edge cluster there must be an even number of Edge nodes; otherwise, the sub-cluster will end up with a single Edge node. You do not need to configure anything for creating the sub-cluster, as it will be automatically defined on the Edges. VMware NSX manager will take of creating the Edge sub-cluster.
### Interface group 
A group with external (uplink) interfaces or service interfaces across service routers. During configuration, place all uplink interfaces and service interfaces in this Interface group. By default, there is only one Interface group.
### Shadow Interface 
Shadow interface is used by the Inter-Edge Logical Switch to provide a path for traffic that is punted between Edges. This interface will be used to send traffic to the desired edge.

### Backup Interface
This is the same as the shadow interface used by the Inter-Edge Logical Switch to provide a path for traffic that is punted between Edges. This interface will be used to receive traffic from the Edge.
### Uplink interface 
The interface used for connecting VLAN (virtual LAN)-based segments to the top-of- the-rack router.

## Bidirectional Forwarding Detection (BFD)
Nó được thiết kế để **phát hiện cực nhanh khi đường truyền bị mất** (dù về mặt Layer 2 vẫn thấy "link up").
### Cơ chế hoạt động
- Hai thiết bị (router, NSX edge, switch, v.v.) **trao đổi BFD packets định kỳ rất nhanh**, ví dụ:
    - Gửi gói BFD mỗi **50ms**
    - Timeout nếu không nhận phản hồi trong **150ms** (3 lần mất liên tiếp)
- Nếu một bên không phản hồi → **coi như đường truyền dead** → **routing protocol drop session ngay lập tức**.
### So sánh
| Tiêu chí                | BGP mặc định              | BGP + BFD            |
| ----------------------- | ------------------------- | -------------------- |
| Phát hiện link down     | 3s – 180s (hold time)     | < 1s                 |
| Mất bao lâu để failover | Rất lâu (đôi khi cả phút) | Gần như ngay lập tức |
| Tốc độ phản ứng         | Chậm                      | Siêu nhanh ⚡         |
## VRF

**VRF** là một kỹ thuật trong mạng giúp tạo ra **nhiều bảng định tuyến riêng biệt** trên cùng một router hoặc gateway. Điều này cho phép:

- Tách biệt lưu lượng giữa các tenant hoặc ứng dụng khác nhau.
- Mỗi VRF hoạt động như một router ảo riêng, có NAT, firewall, gateway riêng.
- Ví dụ: web và app ở hai VRF khác nhau không "nhìn thấy" nhau dù cùng nằm trên một Tier-0.

**VRF Route Leaking** là kỹ thuật cho phép **trao đổi lưu lượng giữa các VRF khác nhau** trong môi trường mạng NSX. Mặc định, mỗi VRF có bảng định tuyến riêng biệt và **cô lập hoàn toàn** với nhau. Nếu không cấu hình route leaking, các máy trong VRF A sẽ không thể giao tiếp với máy trong VRF B.

**Local Inter-VRF (Virtual Routing and Forwarding) routing** is a feature that allows route(s) to be shared between separate routing instances through the use of Static Routes. This can be used to allow communication between different routing instances for a particular use case. Perhaps an application that may be shared between customers or zones.


## Stateful Active/Active Services
- Support from NSX 4.1
- Possible to use Stateful services and T0 or T1 Gateway running in Active/Active High Availability (HA) Mode
- Scale out up to 8 edge node
### Service in gateway

The **supported stateful gateway services** are:

- Gateway Firewall L3-L4
- APP-ID (L7)
- User-ID
- URL Filtering
- TLS Inspection
- IDS/IPS
- Malware Detection and Sandboxing
- NAT
- DHCP Relay Server
- DHCP Server

The **unsupported services** are:

- FQDN Analysis
- L2VPN
- IPSecVPN
- Gateway Network Introspection
- Local DHCP Server
- Service Interface

## Security
- **Distributed Firewall (DFW)**:
    - **Vị trí**: Hoạt động tại mức **vNIC** (virtual Network Interface Card) của từng máy ảo (VM) trong hypervisor (ESXi). DFW được tích hợp trực tiếp vào kernel của hypervisor, cho phép xử lý lưu lượng ngay tại nguồn.
    - **Phạm vi**: Bảo vệ lưu lượng **East-West** (lưu lượng nội bộ giữa các VM hoặc workload trong cùng một trung tâm dữ liệu hoặc SDDC).
    - **Mục đích**: Được thiết kế để thực hiện **micro-segmentation**, cho phép áp dụng các chính sách bảo mật chi tiết đến từng VM hoặc workload cụ thể, ngăn chặn sự di chuyển ngang (lateral movement) của các mối đe dọa trong mạng.liquidweb.comcloud13.ch
    - **Ví dụ**: Kiểm soát lưu lượng giữa các máy ảo trong cùng một phân đoạn mạng hoặc giữa các phân đoạn mạng trong SDDC.
- **Gateway Firewall (GFW)**:
    - **Vị trí**: Hoạt động tại **Tier-0 hoặc Tier-1 Gateway** trong NSX, thường nằm ở ranh giới của mạng (perimeter) hoặc tại các điểm kết nối giữa các mạng.
    - **Phạm vi**: Bảo vệ lưu lượng **North-South** (lưu lượng vào/ra khỏi SDDC hoặc giữa các vùng/mạng khác nhau).
    - **Mục đích**: Được sử dụng để bảo vệ ranh giới mạng, kiểm soát lưu lượng giữa SDDC và mạng bên ngoài (như Internet, on-premises, hoặc các đám mây khác). GFW cũng hỗ trợ các dịch vụ như NAT, DHCP, IPsec VPN, và L2 VPN.vmc.techzone.vmware.comvmware.com
    - **Ví dụ**: Kiểm soát lưu lượng từ một máy ảo trong SDDC ra Internet hoặc giữa các tenant trong môi trường multi-tenant.

```
  

  

+----------------------------+                +----------------------------+

|        TOR Switch 1       |                |        TOR Switch 2       |

|  VLAN 10: Management       |                |  VLAN 10: Management       |

|  VLAN 20: vMotion          |  LACP/vLAG     |  VLAN 20: vMotion          |

|  VLAN 30: vSAN             | <----------->  |  VLAN 30: vSAN             |

|  VLAN 40: Overlay (Geneve) |                |  VLAN 40: Overlay (Geneve) |

|  VLAN 50: Edge Uplink      |                |  VLAN 50: Edge Uplink      |

+-------------+--------------+                +--------------+-------------+

              |                                                |

              |                                                |

        +-----+------------+                            +------+------------+

        |   ESXi Host #1   |                            |   NSX Edge Node   |

        +------------------+                            +-------------------+

        | vmnic0 / vmnic1  | -> vDS -> trunk all VLANs  | uplink1 / uplink2 |

        | vmnic2 / vmnic3  |                            |                   |

        +------------------+                            +-------------------+

                |                                               |

                |                                               |

        +--------------------+                         +----------------------+

        | Distributed PortGrp|                         | T0 Router Uplink     |

        | Mgmt / vMotion /   |                         | Overlay + Uplink     |

        | vSAN / Geneve      |                         +----------------------+

        +--------------------+
```

## NSX commandline

### ESXI
esxcli network ip interface list 
ping -I vmk10 -S vxlan 10.100.12.6

### Edge node
get interface
get transport-node status
get tunnel-interface
get tunnel-endpoints
get tunnel-statistics
get tunnel-port <uuid>
get tunnel-port <uuid> stats
get tunnel-ports stats

### NSX Manager


