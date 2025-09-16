# GPFS Training – Hands-on Labs

## Slide 4 - Node Status  Health Check

```bash
[root@dc1vse0081 ~]# mmgetstate -av
```

 Node number  Node name  GPFS state  
-------------------------------------
           1  gpfs01     active
           2  gpfs02     active
         155  n131       active
         156  n095       active
         183  master     active
         184  login01    active
         185  login02    active
         187  n096       active
         188  n097       active

mmgetstate: The following nodes could not be reached:

n114

```bash
[root@bruclu016-n01 ~]# mmhealth node show
```

Node name:      gpfs01
Node status:    TIPS
Status Change:  2 days ago

Component      Status        Status Change     Reasons & Notices
------------------------------------------------------------------------------
GPFS           TIPS          2 days ago        total_memory_small
NETWORK        HEALTHY       212 days ago      -
FILESYSTEM     HEALTHY       35 days ago       -
DISK           HEALTHY       145 days ago      -
CES            HEALTHY       145 days ago      -

What to Say (Explanation for Training):

“This output is from mmhealth node show. It shows the health status of several subsystems GPFS monitors. Most components are marked HEALTHY, but here the GPFS component is in TIPS state.

TIPS means: it’s not a failure or warning, but a recommendation. In this case, the tip is: total_memory_small. That means this node is below IBM's recommended minimum RAM for optimal GPFS performance.

However, if we look at the actual memory with free -hh...”


```bash
[root@bruclu016-n01 ~]# free -hh
```
              total        used        free      shared  buff/cache   available
Mem:            92G         25G         47G        730M         19G         64G
Swap:           15G          0B         15G


✅ Optional Follow-up Commands:
mmhealth node flush --component GPFS
(Clears the tip message if verified as not relevant.)

💡 Bonus Tip to Say:

“Always match health messages with free, top, and actual performance before acting.
 GPFS sometimes triggers TIPS based on static thresholds.”


```bash
[root@bruclu016-n01 ~]# mmhealth node eventlog
```
Node name:      gpfs01
Timestamp                         Event Name                             Severity             Details
2025-08-05 17:23:59.005961 CEST   eventlog_cleared                       INFO    On the node gpfs01, the eventlog was cleared.
2025-08-13 16:18:30.203111 CET    cluster_connections_down               WARNING Connection to cluster node 10.25.80.143 has all 2 connection(s) down. (Maximum 2).
2025-08-13 16:18:39.251816 CET    cluster_connections_down               WARNING Connection to cluster node 10.25.80.143 has all 1 connection(s) down. (Maximum 1).
2025-08-13 16:19:16.895462 CET    cluster_connections_ok                 INFO    All connections are good for target ip 10.25.80.143.
2025-08-20 15:53:19.603811 CEST   cluster_connections_bad                WARNING Connection to cluster node 10.25.80.139 has 1 bad connection(s). (Maximum 2).

```bash
[root@bruclu016-n01 ~]# mmlscluster
```

GPFS cluster information
========================
  GPFS cluster name:         GPFS.ESI-Group
  GPFS cluster id:           1194318350905388747
  GPFS UID domain:           GPFS.ESI-Group
  Remote shell command:      /usr/bin/ssh
  Remote file copy command:  /usr/bin/scp
  Repository type:           CCR

 Node  Daemon node name  IP address       Admin node name  Designation
-----------------------------------------------------------------------
   1   gpfs01            10.25.80.15      gpfs01           quorum-manager
   2   gpfs02            10.25.80.16      gpfs02           quorum-manager
 155   n131              10.25.80.131     n131             
 156   n095              10.25.80.95      n095             
 183   master            10.25.80.11      master           
 184   login01           10.25.80.12      login01          
 185   login02           10.25.80.13      login02          

```bash
[root@bruclu016-n01 ~]# mmlsconfig
```
Configuration data for cluster GPFS.ESI-Group:
----------------------------------------------
clusterName GPFS.ESI-Group
clusterId 1194318350905388747 ##dentify thois cluster uniquely -- used in logs, license tracking
autoload yes #Automatically mounts the filesystem when GPFS starts.“This ensures uptime without manual intervention.”
dmapiFileHandleSize 32
ccrEnabled yes ## CCR Cluster Configuration Repository - This means cluster config is managed in a distributed,
```bash
               ## fault tolerant way. It's the modern replacement for old flat files like mmsdrfs
			   ## The CCR state consists of files and directories under /var/mmfs/ccr
			   #Ref: https://www.ibm.com/docs/en/storage-scale/5.2.3?topic=architecture-cluster-configuration-repository
```
			   
```bash
## Security & tuning Parameters
```
cipherList AUTHONLY ## only encrypted/authenticated traffic between GPFS nodes.
maxMBpS 12000 ## artificial throttle to avoid saturating fabric or backend.
maxblocksize 4M ## optimal for large file HPC workloads (max GPFS block size).
workerThreads 1024 ## controls concurrency of GPFS tasks on nodes
socketMaxListenConnections 1024
cesSharedRoot /gpfs/ces
cifsBypassTraversalChecking yes
syncSambaMetadataOps yes
cifsBypassShareLocksOnRename yes
[nsdNodes]
maxFilesToCache 256000
maxStatCache 256000
```bash
##Network Fabric Highlight
```
[ibServers,nsdNodes,opaServers]
verbsRdma enable
[nsdNodes]
verbsPorts mlx5_0/1/0  hfi1_0/1/1
[opaServers]
verbsPorts hfi1_0/1/1
[ibServers]
verbsPorts mlx5_0/1/0
```bash
##This section defines which ports are used for RDMA (Remote Direct Memory Access) over high-performance fabrics like Infiniband and Omni-Path.
##The presence of mlx5_0/1/0 shows this system uses Mellanox Infiniband, and hfi1_0/1/1 refers to Intel Omni-Path.
##Enabling verbsRdma allows GPFS to bypass the TCP stack for inter-node communication, giving much lower latency and higher throughput.
##This is especially critical in HPC where performance bottlenecks in metadata or token traffic would otherwise slow the cluster.”
```
[gpfs01,gpfs02]
traceRecycle local
[nsdNodes]
pagepool 32G
[service]
pagepool 2G
[compute]
pagepool 4G
[gpfs01,gpfs02]
trace all 9 # All major GPFS activities are traced with level 9 verbosity — 
```bash
            # useful for debugging but can be heavy on disk. It's important to monitor trace size.”
```
[cesNodes]
trace all 9 iolite 0
[gpfs01,gpfs02,cesNodes]
tracedevTimeFormat calendar
[common]
minReleaseLevel 5.1.6.0
tiebreakerDisks GPFS_MD_LUN01;GPFS_MD_LUN03;GPFS_MD_LUN04
```bash
# “If quorum is lost, GPFS consults these disks to determine
# which partition of the cluster should stay active. It avoids data corruption during split-brain events.
```
cesCidrPool <IP=10.25.88.14><ATTRIBUTE=><GROUP=><PREFIX=>+<IP=10.25.88.200><ATTRIBUTE=><GROUP=><PREFIX=>+<IP=10.25.80.17><ATTRIBUTE=><GROUP=><PREFIX=>
adminMode central

File systems in cluster GPFS.ESI-Group:
---------------------------------------
/dev/gpfs

## Slide 6 - Detecting LUN failures, storage disconnections, and SAN issues

🎯 Goal:

Detect and investigate a simulated LUN failure or disconnected storage path in a GPFS cluster.
🔹 Step 1 – List Disks by Filesystem
mmlsfs gpfs -d
flag                value                    description
------------------- ------------------------ -----------------------------------
 -d                 GPFS_MD_LUN01;GPFS_MD_LUN02;GPFS_DATA_LUN01;GPFS_DATA_LUN02;GPFS_MD_LUN03;GPFS_MD_LUN04;GPFS_DATA_LUN03;GPFS_DATA_LUN04  Disks in file system

Which LUNs (NSDs) are metadata vs data

Match them with SANtricity volumes --> Show web acceess
https://bruclu015-e2800b.vbr.is.keysight.com/sm/en-US/#/home
https://bruclu015-e2812a.vbr.is.keysight.com/sm/en-US/#/home

🔹 Step 2 – View Disk State in GPFS
mmlsdisk gpfs -Y

mmlsdisk::HEADER:version:reserved:reserved:nsdName:driverType:sectorSize:failureGroup:metadata:data:status:availability:diskID:storagePool:remarks:numQuorumDisks:readQuorumValue:writeQuorumValue:diskSizeKB:diskUID:thinDiskType:replicaType:
mmlsdisk::0:1:::GPFS_MD_LUN01:nsd:512:-1:Yes:No:ready:up:1:system:desc:5:3:3:387826316:C0A8F6FB5E72304E:::
mmlsdisk::0:1:::GPFS_MD_LUN02:nsd:512:-1:Yes:No:ready:up:2:system:desc:5:3:3:387815832:C0A8F6FB5E723050:::
mmlsdisk::0:1:::GPFS_DATA_LUN01:nsd:512:-1:No:Yes:ready:up:3:nlsas:desc:5:3:3:31210004736:C0A8F6FB5E723052:::
mmlsdisk::0:1:::GPFS_DATA_LUN02:nsd:512:-1:No:Yes:ready:up:4:nlsas::5:3:3:31210004736:C0A8F6FB5E723054:::
mmlsdisk::0:1:::GPFS_MD_LUN03:nsd:512:-1:Yes:No:ready:up:5:system:desc:5:3:3:387815832:C0A8F6FC63877608:::
mmlsdisk::0:1:::GPFS_MD_LUN04:nsd:512:-1:Yes:No:ready:up:6:system:desc:5:3:3:387805344:C0A8F6FC6387760C:::
mmlsdisk::0:1:::GPFS_DATA_LUN03:nsd:512:-1:No:Yes:ready:up:7:nlsas::5:3:3:31209994240:C0A8F6FC6387760A:::
mmlsdisk::0:1:::GPFS_DATA_LUN04:nsd:512:-1:No:Yes:ready:up:8:nlsas::5:3:3:31209994240:C0A8F6FC6387760E:::

🗣️ “A failed or disconnected LUN will appear in this output with an error status.”
status=down, missing, or notReady



🔎 Look at:
IOPS drop
Health status: disk errors, failed controller, degraded pool

```bash
##Step 3 – Check Physical Path (Optional)
```
multipath -ll
lsblk

LUNs are seen by the OS
Multipath reports healthy paths



✅ TL;DR (Quick Answer):

“We separate metadata and data into different partitions in GPFS to optimize performance and reliability.
 Metadata operations are small and frequent, while data operations are typically large and sequential.
 Splitting them allows each to be tuned and scaled independently.”

Deeper Explanation (What to Say in Training):

“GPFS lets us define different storage pools: one for metadata and one for user data. Each pool can be backed by different LUNs or disks.
Metadata includes things like file names, directories, permissions, inode structures — it's heavily accessed even when you're just listing files or checking access. These are small, random I/O operations.
Data is the actual file content — jobs reading or writing GBs or TBs. These are large, sequential I/O operations.
By placing them on separate disks, we prevent I/O contention. Metadata stays fast and responsive even when the data LUNs are handling heavy job throughput.
This also improves recovery time: if a data disk fails, we don’t lose the metadata structure, and vice versa.
In NetApp or other SAN setups, we usually place metadata on faster, low-latency LUNs, and data on high-throughput or larger-capacity LUNs — depending on performance goals.”
Because this metadata are stored on SSD drive, to maximise performance
      +---------------------------+
      |        GPFS Filesystem    |
      +---------------------------+
           |             |
    [Metadata Pool]   [Data Pool]
     LUN01, LUN02     LUN03, LUN04
     Fast SAS/SSD     High-cap HDDs
	 
📌 💬 Debrief Questions:

Was any disk marked as down or failed?
What could cause a disk to go missing in GPFS?
What would happen to jobs if a metadata disk goes offline?
How would GPFS self-heal if replication is enabled?


```bash
#✅ Slide 9 – Title:
# Module 3: Network Diagnostics – InfiniBand & Omni-Path
```

🧪 LAB: Diagnosing RDMA Fabric Issues in GPFS
🧠 Goal: Detect whether InfiniBand or Omni-Path is properly configured and functional on a GPFS node, 
and verify the interfaces GPFS uses for RDMA.

✅ Step 1 – Check IB or OPA Interface Status

🖥️ For InfiniBand Nodes:
ibstat

```bash
#output
```
CA 'mlx5_0'
    Port 1:
        State: Active
        Physical state: LinkUp

🗣️ “If the state is Down or Initializing, there’s a cable, port, or switch issue.”

🖥️ For Omni-Path Nodes:
opainfo

✅ Check:
Active ports (e.g. Port 1 Active)
Link speed and width

✅ Step 2 – Test Connectivity Between Nodes

ibping -S
```bash
# On another node
```
ibping -c 5 -C <hostname>
✅ Expect successful response — latency in microseconds.

```bash
#“Use ibping to test basic connectivity between two nodes at the InfiniBand level (not IP) is called verb,
# indeed it's a transport layer specific to IB (such as MAC fo Ethernet, but different)
```
.”

```bash
#✅ Step 3 – Discover the Topology (Optional Advanced)
```
ibnetdiscover
  ✅ Shows a graph of nodes and switches — can help detect cabling issues or switch ports down.

```bash
#✅ Step 4 – Check What Ports GPFS Is Using
```
mmlsconfig | grep verbsPorts
```bash
#✅ Example Output:
```
[nsdNodes]
verbsPorts mlx5_0/1/0  hfi1_0/1/1


```bash
#What's NSD/RDMA ?
```

```bash
#🗣️ “This confirms what interface(s) GPFS is using for RDMA. 
#Mismatched or disabled ports can cause performance or availability issues.”
```

```bash
#NSD stands for Network Shared Disk — it’s a core concept in IBM Spectrum Scale (GPFS).
#🔹 Definition:
#An NSD is a logical representation of a physical disk that is accessible over the network.
#It abstracts storage access from the physical location and allows GPFS to access disks transparently over the network, whether they are local or remote.
#🧱 How It Works:
#NSD Servers: Nodes that own physical access to the disk (e.g., via SAS, FC, NVMe).
#NSD Clients: Nodes that access that disk over the network, not directly.
#All nodes access the disk via the GPFS protocol, regardless of local or remote status.
#This slide shows the internal communication flow between an NSD Client and an NSD Server when using RDMA for I/O.
#On the left side, we have a client node, which is typically a compute node that doesn’t own the disk. It accesses the disk through the RDMA fabric via an NSD server, shown on the right.
#Here's what's important:
#GPFS uses RPC (Remote Procedure Call) requests over RDMA to talk between client and server.
#Page Pool memory is used to buffer and transfer data.
#With verbsRDMA enabled (as we saw in mmlsconfig | grep verbsPorts), GPFS uses specialized network interfaces like mlx5_0 or hfi1_0 for high-speed, zero-copy data transfer.
#This avoids the traditional TCP/IP stack, reducing overhead.
#So effectively, this model lets us do disk I/O over the network with much lower CPU usage, faster performance, and reduced latency — ideal for HPC
```



Slide 11 | Module 4 - CES Services

✅ Step 1 – Confirm CES Is Running on Nodes
mmces node list

Node Number   Node Name   Node Groups   Node Flags
------------- ----------- ------------- ------------ 
1             gpfs01                    none
2             gpfs02                    none


This output shows which nodes are part of the CES subsystem.We can see that gpfs01 and gpfs02 are CES-enabled nodes.
Meaning: they can run services like NFS, CIFS, or Object (S3)
Even through the CES node list shows none in the flags, we can see from 

mmces service list

that both NFS and SMB services are up and running.

This happens because CES can enable services cluster-wide without assgning thm to specific node groups or flags unless explicity configured.

so the takeway is: dont rely solely on mmces node list to confirm if a service is running -- always follow up with mmces service list.

Also, CES floating IPs will be automatically assigned to eligible nodes when services are active

✅ Step 2 – List Floating IPs (CES Addresses)
mmces address list
```bash
#Output:
```
Address        Node     Ces Group   Attributes
-------------- -------- ----------- ------------
10.25.80.17    gpfs02   none        none
10.25.88.14    gpfs01   none        none
10.25.88.200   gpfs02   none        none


🧠 Why it matters:
“If these IPs are missing, clients won’t be able to reach NFS/CIFS shares even if services are up.”

These IPs are floating IP addresses managed by CES. They are not permanently tied to the physical interfaces
but instead move between CES-enabled nodes to ensure high availability for NFS and SMB (CIFS) services.

In this output, we see three IPs:
10.25.88.14 currently assigned to gpfs01
10.25.80.17 and 10.25.88.200 assigned to gpfs02

Even though CES services (NFS and SMB) are running, if none of these floating IPs are present or bound to any node, then clients trying to mount exports via those addresses would fail.

So in troubleshooting, if a user says:
I cant mount the NFS export,
and CES services are running, the very next thing to check is this list of IPs.
If it’s empty or the IP they’re using is missing here, CES may have failed over or the IP wasn’t reassigned properly

📌 What to Compare:
Run ip a on both gpfs01 and gpfs02, and confirm that:
10.25.88.14 is present on gpfs01
10.25.80.17 and 10.25.88.200 are present on gpfs02

Step 3 – View Exported Paths for NFS
mmnfs export list

Path                Delegations  Clients  
------------------  -----------  -------  
/gpfs/scratch-mstr  NONE         *     

📌 Explained:
Displays the NFS export definitions: paths, allowed clients, options, and permissions.

✅ 🔚 Final Step – Confirm CES IP Configuration in GPFS
mmlsconfig | grep ces

✅ Output (from your cluster):
cesSharedRoot /gpfs/ces
cesCidrPool <IP=10.25.88.14>+<IP=10.25.88.200>+<IP=10.25.80.17>

This confirms that GPFS is properly configured to use three floating CES IPs:
10.25.88.14
10.25.88.200
10.25.80.17

The Ips are part of the cesCidrPool, which is the pool of addresses that CES can assign dynamically to export services like NFS and SMB
GPFS will assign these IPs to CES nodes (like gpfs01 or gpfs02) based on load failover, or service enablement.
This configuration matches the output we saw in mmces address list and ip a
which means wverything is consistent


```bash
#SLIDE 13 🧪 LAB: Quota & Fileset Troubleshooting in GPFS | Module 5
## 🎯 Goal: Learn how to identify quota problems for users and filesets, and how to simulate/explore a quota-exceeded situation
```

```bash
# ✅ Step 1 – View Quota Enabled Status for Filesystem
```
mmlsfs gpfs -Q
```bash
#output
```
flag                value                    description
------------------- ------------------------ -----------------------------------
 -Q                 user;group;fileset       Quotas accounting enabled
                    user;group;fileset       Quotas enforced
                    none                     Default quotas enabled

```bash
# This tells us that all types of quota tracking and enforcement are turned on. 
# So if a user exceeds their limit, GPFS will actively block write operations.
# There’s no default quota, which means each user must be assigned a quota manually or by policy.
```

```bash
# ✅ Step 2 – List Quota Usage by a Specific User
```
mmlsquota -u <your-username> --block-size auto

```bash
#output
```
                         Block Limits                                               |     File Limits
Filesystem Fileset    type         blocks      quota      limit   in_doubt    grace |    files   quota    limit in_doubt    grace  Remarks
gpfs       scratch-mstr USR          540.2M       300G       360G          0     none |       13       0        0        0     none 

```bash
#The user allasouz is using 540.2MB of their 300GB soft quota
#Hard quota limit is 360GB — writes beyond this will be blocked immediately
#Only 13 files are stored — and no file count quota is enforced
#No data is in doubt (i.e., accounting is up-to-date)
```

```bash
#Here, we’re checking quota usage for a real user, allasouz, in the scratch-mstr fileset.
#They’re using only 540 MB, so well within their limits. 
#But if this user tried writing more than 360 GB, they’d be blocked immediately because GPFS enforces the hard limit.
#There’s no grace period or file quota in this case, which means file count is unlimited, but disk space is strictly monitored and enforced
```

```bash
#🧠 “This is where you’ll confirm whether the error came from a soft or hard limit breach.
```

```bash
# ✅ Step 3 – View Quotas per Fileset (Optional)
```
mmlsquota -j scratch-mstr gpfs

```bash
# “This output tells us that the scratch-mstr fileset has no defined quota limits at the fileset level.
# In GPFS, you can apply quotas at three levels:
### Per user
### Per group
### Per fileset
# What we’re seeing here is that the fileset itself is not restricted — 
# users can fill it as long as they remain within their personal or group limits.
#This is common in shared scratch areas, where individual user quotas are enforced but the fileset is left open for flexibility.”
```

```bash
#✅ Step 4 – Full Report of All Users
```
mmrepquota -u -v gpfs --block-size auto

```bash
#✅ Great for:
#Quota audits
#Detecting heavy users
#Preparing reports
```

```bash
#GPFS doesn’t sort the output natively, but we can use Unix tools like sort to identify the top users by disk usage.
#This is very helpful in quota audits, or when a filesystem is nearly full and you need to know who’s consuming the most.”
```

```bash
# Organizing by space used
```
mmrepquota -u -v gpfs --block-size auto | sort -k4 -h -r | more
```bash
# To show only top 15 users
```
mmrepquota -u -v gpfs --block-size auto | sort -k4 -h -r | head -n 15

Part						Purpose
-u	    					User quota
-v							Verbose output
--block-size auto			Human-readable sizes (KB, MB, GB)
sort -k4					Sort by 4th column (blocks used)
-h							Human-readable sort (handles MB/GB/etc.)
-r			

```bash
# Slide 14 - 🧪 Module 6 – Node Management Hands-On Lab
```

mmlsnodeclass
```bash
#output
```
Node Class Name       Members
--------------------- -----------------------------------------------------------
opaServers            gpfs01,gpfs02,n095,n097,n098,n099,n101,n103,n104,...
compute               n131,n095,n097,n098,n099,n101,n103,n104,n106,...
service               gpfs01,gpfs02
ibServers             gpfs01,gpfs02,n131,n132,n133,n134,n135,n136,...

```bash
#How your nodes are organized by roles:
### opaServers → Nodes with Omni-Path interfaces
### ibServers → Nodes with InfiniBand interfaces
### compute → Regular compute nodes
### service → Metadata / CES servers
```


```bash
#🧠 Training Insight:
```

```bash
#This command helps us understand how the GPFS cluster is logically segmented.
#If you're troubleshooting I/O latency or connection issues, you’ll know which nodes use which interconnect (OPA vs IB), and which are providing services like CES or metadata.
#It’s also important when adding new nodes — we need to know which class they should belong to.
```

```bash
#✅ Step 2 – Check Node Status in Real Time
```
mmgetstate -a

```bash
#🧠 Explanation:
#Use this to quickly detect unhealthy or unreachable nodes.
#Nodes in unknown state may have networking or service issues.
```

```bash
#✅ Step 3 – View Node Licensing (Safe Read-Only)
```
mmlslicense
```bash
#output
```
Summary information 
---------------------
Number of nodes defined in the cluster:                         85
Number of nodes with server license designation:                 2
Number of nodes with FPO license designation:                    0
Number of nodes with client license designation:                83
Number of nodes still requiring server license designation:      0
Number of nodes still requiring client license designation:      0
This node runs IBM Spectrum Scale Standard Edition

```bash
#Using mmlslicense, we get a quick overview of our license distribution.
#We have 85 nodes in total: 2 are acting as servers (most likely gpfs01 and gpfs02), and the rest are clients.
#No nodes are unlicensed, which is good — if we were scaling the cluster, we’d need to revisit this to ensure we stay compliant.
#We're also using the Standard Edition, which covers all the GPFS functionality we use today, 
#such as quotas, filesets, and CES. Advanced features would require licensing upgrades
```

```bash
## Slide 16 - Module 7 Investigate Logs for Issues
```

```bash
#How to identify GPFS problems with key log files
```

```bash
#✅ Step 1 – Check if GPFS is Healthy
```
mmgetstate -a

```bash
#✅ Make sure all nodes are active
```

```bash
#✅ Step 2 – Tail the Main Log
```
tail -n 20 /var/adm/ras/mmfs.log.latest

```bash
#output
```
2025-09-12_15:01:03.590+0200: [N] Connecting to 10.25.80.144 n144 <c0n7>:[0]
2025-09-12_15:01:03.599+0200: [I] Connected to 10.25.80.144 n144 <c0n7>:[0]
2025-09-12_15:31:47.668+0200: [I] Accepted and connected to 10.25.80.111 n111 <c0n42>:[0]
2025-09-12_15:44:31.525+0200: [I] Accepted and connected to 10.25.80.105 n105 <c0n20>:[0]
2025-09-12_15:52:47.870+0200: [I] Accepted and connected to 10.25.80.111 n111 <c0n42>:[1]
2025-09-12_15:53:01.670+0200: [I] Accepted and connected to 10.25.80.126 n126 <c0n35>:[0]
2025-09-12_15:53:55.673+0200: [I] Accepted and connected to 10.25.80.119 n119 <c0n47>:[1]
2025-09-12_15:54:05.983+0200: [I] Accepted and connected to 10.25.80.107 n107 <c0n25>:[1]
2025-09-12_15:54:05.984+0200: [N] Connecting to 10.25.80.107 n107 <c0n25>:[0]
2025-09-12_15:54:05.994+0200: [I] Connected to 10.25.80.107 n107 <c0n25>:[0]
2025-09-12_15:54:06.739+0200: [I] Accepted and connected to 10.25.80.126 n126 <c0n35>:[1]
2025-09-12_15:54:12.627+0200: [I] Accepted and connected to 10.25.80.119 n119 <c0n47>:[0]
2025-09-12_15:54:35.818+0200: [I] Accepted and connected to 10.25.80.110 n110 <c0n29>:[0]
2025-09-12_16:20:38.492+0200: [I] Accepted and connected to 10.25.80.150 n150 <c0n63>:[1]
2025-09-12_16:20:38.492+0200: [N] Connecting to 10.25.80.150 n150 <c0n63>:[0]
2025-09-12_16:20:38.502+0200: [I] Connected to 10.25.80.150 n150 <c0n63>:[0]
2025-09-12_16:24:30.319+0200: [I] Accepted and connected to 10.25.80.156 n156 <c0n8>:[0]
2025-09-12_16:24:30.320+0200: [N] Connecting to 10.25.80.156 n156 <c0n8>:[1]
2025-09-12_16:24:30.330+0200: [I] Connected to 10.25.80.156 n156 <c0n8>:[1]
2025-09-12_16:30:25.507+0200: [I] Accepted and connected to 10.25.80.105 n105 <c0n20>:[1]

```bash
#🔍 Look for recent activity, mount events, quorum messages, or node failures
```

```bash
#✅ Step 3 – Search for Errors (Ask Alain Cady)
```
grep -i error /var/adm/ras/mmfs.log.latest | tail -n 10

```bash
#output
```
2025-08-31_21:06:59.448+0200: [E] VERBS RDMA closed connection to 10.25.80.139 (n139) on mlx5_0 port 1 fabnum 0 index 21 cookie 529 error 233
2025-08-31_21:06:59.549+0200: [E] VERBS RDMA closed connection to 10.25.80.158 (n158) on mlx5_0 port 1 fabnum 0 index 4 cookie 528 error 233
2025-09-01_05:17:46.066+0200: [E] VERBS RDMA closed connection to 10.25.80.150 (n150) on mlx5_0 port 1 fabnum 0 index 22 cookie 557 error 233
2025-09-01_05:56:15.786+0200: [E] VERBS RDMA closed connection to 10.25.80.139 (n139) on mlx5_0 port 1 fabnum 0 index 21 cookie 586 error 233
2025-09-01_06:39:36.591+0200: [E] VERBS RDMA closed connection to 10.25.80.139 (n139) on mlx5_0 port 1 fabnum 0 index 21 cookie 589 error 233
2025-09-01_11:07:09.253+0200: [E] VERBS RDMA closed connection to 10.25.80.154 (n154) on mlx5_0 port 1 fabnum 0 index 1 cookie 550 error 233
2025-09-01_13:16:05.135+0200: [E] VERBS RDMA closed connection to 10.25.80.142 (n142) on mlx5_0 port 1 fabnum 0 index 1 cookie 591 error 233
2025-09-01_19:15:06.643+0200: [E] VERBS RDMA closed connection to 10.25.80.150 (n150) on mlx5_0 port 1 fabnum 0 index 22 cookie 588 error 73
2025-09-08_07:36:24.899+0200: [E] VERBS RDMA closed connection to 10.25.80.150 (n150) on mlx5_0 port 1 fabnum 0 index 22 cookie 593 error 233
2025-09-12_09:31:25.400+0200: [E] VERBS RDMA closed connection to 10.25.80.135 (n135) on mlx5_0 port 1 fabnum 0 index 20 cookie 505 error 233
```bash
[root@bruclu016-n01 ~]# 
```

```bash
#Explaining
#VERBS RDMA closed connection >> An RDMA connection between two nodes was unexpectedly closed.
#mlx5_0 port 1 >> Refers to the InfiniBand (or RoCE) interface and port — mlx5_0 is the device name.
#fabnum, index, cookie >> Internal identifiers for the RDMA fabric and session — mostly for IBM support.
#error 233 >> Indicates a connection timeout or remote host unreachability.
#error 73 >> Typically means software caused connection abort — could be related to high congestion, reset, 
#			or failure on remote side.
#🔎 Review the last 10 logged errors — are they related to:
##Disk I/O?
##Quorum?
##Node communication?
```

```bash
#✅ Step 4 – CES Health via mmwatch.log ((Ask Alain Cady))
```
grep -i fail /var/adm/ras/mmwatch.log

```bash
#output
[root@bruclu016-n01 ~]# less /var/adm/ras/mmwatch.log | tail -n10
```
"KAFKA_ENABLED" is undefined
"KAFKA_ENABLED" is undefined
"KAFKA_ENABLED" is undefined
"KAFKA_ENABLED" is undefined
"KAFKA_ENABLED" is undefined
"KAFKA_ENABLED" is undefined
"KAFKA_ENABLED" is undefined
"KAFKA_ENABLED" is undefined
"KAFKA_ENABLED" is undefined
"KAFKA_ENABLED" is undefined
```bash
[root@bruclu016-n01 ~]# 
```


```bash
# 📋 Look for CES-related startup problems or failed checks.
```

```bash
#✅ Step 5 – Bonus (Optional)
```
ls -lh /var/adm/ras/mmfs.log.*