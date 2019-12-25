# High latency in MSSQL DB hosted on Nutanix after Nutanix cluster upgrade

 

### Goal achieved

We fixed the SQL crash issue caused by high disk latency on ISCSI
storage hosted on Nutanix.

### Problem

There was latency of around 20000 ms on MSSQL cluster shared Disk. Issue
started just after the   Nutanix Cluster upgrade. Resulting SQL server
frequent crash.

### Error Message

*SQL Server has encountered 26063 occurrence(s) of I/O requests taking
longer than 15 seconds to complete on file .*

*The OS file handle is 0x000006A4. The offset of the latest long I/O is:
0x00000*

![](latency%20in%20SQL.fld/image001.png)

### Client Infrastructure
-------------------------------------------------------
**MS SQL cluster**

MS SQL cluster contains two nodes windows 2012 servers. Both nodes
hosted as VMs on Nutanix cluster.

**Shared Drive**

MS SQL Cluster share drives are ISCSI luns targeted on Nutanix storage
volume group. Target LUNS are accessible from CVM through Nutanix
cluster Network.

**Nutanix Cluster**

Nutanix Cluster contain 3 Nodes. Nutanix nodes interconnected with 10G
same VLAN network. AOS running in CVMs on all Nutanix nodes, group and
emulate the local disks connected on Nutanix nodes into shared storage
leveraging 10G cluster communication. This shared storage is presented
to all Nutanix nodes using virtual HBAs in CVM. All communication to
shared storage from Nutanix node occur using CVM virtual HBAs. This
redundant Shared storage can be used and divided into VMs storage, ISCSI
targets, luns and file server. As all data traffic occur using CVMs,
CVMs and Physical node must be on same 10G network.

 
### <span style="color:blue">Observation</span>

Checked the Nutanix luns on volume group, latency was not more than 2
ms, Checked the VM disks, low latency. However high latency on shared
issci drive on VMs. Nutanix cluster is running on 10G network and
connected to 10G switch, however router for inter VLAN communication is
running on 1G network.

### <span style="color:blue">Finding</span>

MS SQL and Nutanix CVM are running on different VLAN. after upgrade,
cluster reboot reset the tiered caching in underlying VM vdisk. As
caching reset, all the IOPS must be served from remote iSCSI LUN using
network. ISCSI initiator not getting required network throughput due
bottleneck at router. VMs shared disk experiencing high latency thus SQL
server was kept crashing.

### <span style="color:blue">Resolution</span>

As VMs and CVM are on different VLAN all iscsi trafice must go to
router, as router is running on 1G network, resulting slowness in
network communication. To prevent iscsi traffic on inter VLAN routing,
one VNIC in each VMs has been added. VLAN tagging was the same as in CVM
VLAN so that traffic must remain and leverage 10G Nutanix Cluster
network. As iscsi network started running on 10 G, latency reduced to 2
ms dramatically.

![](latency%20in%20SQL.fld/image002.png)
