# Overview: Tenant Isolation in Storage Area Network
Ceph is a widely used, multi-tenant, high-throughput, distributed storage system. It provides logical isolation through identity-based authentication and authorization. However, all client and server nodes still share a single public Storage Area Network (SAN). 
To achieve defense-in-depth, we developed ***Secure Ceph***, which locks down RADOS connections by partitioning the Layer-3 public network among tenants using VLANs. Each tenant's clients are confined to a dedicated VLAN, whereas servers asymmetrically aggregate and flatten multiple VLANs into a single unified Layer-3 space—without exposing tenants to each other. We term this asymmetric design <ins>*L3 lockdown*</ins>. This network-level isolation also strengthens authentication by validating requests against their VLAN origin. ***Secure Ceph*** *thereby enables confidentiality-sensitive workloads in one's own isolated storage pool.* 

# Features:
**Enforce dual control planes:** ***Secure Ceph*** combines identity-based authorization with <ins>*L3 lockdown*</ins>. This dual mechanism provides both logical and network-level isolation: even if credentials are compromised, VLAN isolation prevents cross-tenant reachability, IP spoofing, and lateral movement.

**Provide scalable security:** The enhanced security layer consists of [niiceph-ovsctrl](https://github.com/NII-cloud-operation/niiceph-ovsctrl/), [niiceph-gateway](https://github.com/NII-cloud-operation/niiceph-gateway/), and Open vSwitch (OVS). These components operate across all Ceph nodes in a distributed manner without sacrificing throughput.

**Doesn't disrupt Ceph cluster management:** No additional client-side administration is required, except for setting the tenant's VLAN ID as a prefix for a pool name; and no extra IP resources are needed, unlike isolations of Lustre LNet or CIFS VIF. 

<table>
  <tr>
    <td align="center">
      <img src="https://github.com/user-attachments/assets/50cfab34-7886-48d1-a664-1ecbcdfbe3d9" width="480">
    </td>
    <td style="width: 40px;"></td>
    <td align="center">
      <img src="https://github.com/user-attachments/assets/b0f44aad-bf31-422b-84d3-bf2d4a5594cf" width="480">
    </td>
  </tr>
  <tr style="background-color: #f0f8ff;"> <!-- ← この行の色を指定 -->
    <td align="center"><b>OSD Internals</b></td>
    <td></td>
    <td align="center"><b>Ceph Cluster Configuration</b></td>
  </tr>
</table>

# Architecture: *L3 lockdown* as Dual Security Planes

**Partitioning:** VLAN partitions a single Layer-3 network segment into dedicated IP subsets: one for Ceph cluster nodes and separate subsets for each tenant's client nodes.

**Client-side isolation:** Within a tenant, each RADOS client uses its assigned IP subset. VLAN enforcement occurs at the switch port—preventing client nodes from reaching or spoofing those of other tenants.

**Server-side asymmetry:** In contrast, Ceph nodes receive all tenant VLANs over a trunk and asymmetrically flatten them into a unified Layer-3 space, while still preserving tenant isolation. Layer-3 partitioning is therefore invisible to Ceph's Monitor (MON) and Object Storage Device (OSD) services.

**Transport lockdown:** The OVS bridge on each Ceph node enforces <ins>*L3 lockdown*</ins> routing in a distributed fashion—removing VLAN tags on ingress and re-adding them back on egress, i.e., static routing based on tenants' IP subsets. [niiceph-ovsctrl](https://github.com/NII-cloud-operation/niiceph-ovsctrl/) manages this distributed OVS control. 
Authorization binding: Authentication requests to MON are intercepted by [niiceph-gateway](https://github.com/NII-cloud-operation/niiceph-gateway/) and validated against VLAN/tenant origins, ensuring credentials cannot be misused across VLANs.
