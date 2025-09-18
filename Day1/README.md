# GPFS Training – Hands-on Labs

## Module 1 - Node Status & Health Check

```bash
[root@dc1vse0081 ~]# mmgetstate -av

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
```
### 1.1 Health Check command

The commmand `mmhealth` will show us the health status of several subystems GPFS monitors. Most components below are marked `HEALTHY`, but here the GPFS component is in `TIPS` state.

**Meaning of TIPS**
> It's not a failure or warning, but a recomendation following IBM's best pratices. 
> In this case, the TIP is: `total_memory_small`. 
> That means this node is below IBM's minimum RAM ofr optimal GPFS performance

```bash
[root@bruclu016-n01 ~]# mmhealth node show


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

```

However, if we look at the actual memory with free -hh...”

```bash
[root@bruclu016-n01 ~]# free -hh

              total        used        free      shared  buff/cache   available
Mem:            92G         25G         47G        730M         19G         64G
Swap:           15G          0B         15G
```

> [Documentation of mmhealth command](https://www.ibm.com/docs/en/storage-scale/5.2.2?topic=reference-mmhealth-command)



> “Always match health messages with free, top, and actual performance before acting. GPFS sometimes triggers TIPS based on static thresholds.”


```bash
[root@bruclu016-n01 ~]# mmhealth node eventlog

#output
Node name:      gpfs01
Timestamp                         Event Name                             Severity             Details
2025-08-05 17:23:59.005961 CEST   eventlog_cleared                       INFO    On the node gpfs01, the eventlog was cleared.
2025-08-13 16:18:30.203111 CET    cluster_connections_down               WARNING Connection to cluster node 10.25.80.143 has all 2 connection(s) down. (Maximum 2).
2025-08-13 16:18:39.251816 CET    cluster_connections_down               WARNING Connection to cluster node 10.25.80.143 has all 1 connection(s) down. (Maximum 1).
2025-08-13 16:19:16.895462 CET    cluster_connections_ok                 INFO    All connections are good for target ip 10.25.80.143.
2025-08-20 15:53:19.603811 CEST   cluster_connections_bad                WARNING Connection to cluster node 10.25.80.139 has 1 bad connection(s). (Maximum 2).
```
### 1.2 Other valuable commands to check our Nodes or Cluster Status

```bash
[root@bruclu016-n01 ~]# mmlscluster

#output
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
```

> What is CCR (Cluster Configuration Repository)?

```bash
Repository type:           CCR
```

- *Meaning*:
    This means your GPFS cluster is using the Cluster Configuration Repository — a fault-tolerant, highly available, and distributed configuration store that replaces older methods (like mmsdrfs files) in newer GPFS (IBM Storage Scale) clusters.
- A fault tolerant way. It's the modern replacement for old flat files like mmsdrfs
- The CCR state consists of files and directories under /var/mmfs/ccr    
- More informations from IBM Documentation, [click here](https://www.ibm.com/docs/en/storage-scale/5.2.3?topic=architecture-cluster-configuration-repository)


The command `mmlsconfig`:

```bash
[root@bruclu016-n01 ~]# mmlsconfig
Configuration data for cluster GPFS.ESI-Group:
----------------------------------------------
clusterName GPFS.ESI-Group
clusterId 1194318350905388747 ##dentify thois cluster uniquely -- used in logs, license tracking
autoload yes #Automatically mounts the filesystem when GPFS starts.“This ensures uptime without manual intervention.”
dmapiFileHandleSize 32
ccrEnabled yes ## CCR Cluster Configuration Repository - This means cluster config is managed in a distributed,
## Security & tuning Parameters
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
##Network Fabric Highlight
[ibServers,nsdNodes,opaServers]
verbsRdma enable
[nsdNodes]
verbsPorts mlx5_0/1/0  hfi1_0/1/1
[opaServers]
verbsPorts hfi1_0/1/1
[ibServers]
verbsPorts mlx5_0/1/0
[gpfs01,gpfs02]
traceRecycle local
[nsdNodes]
pagepool 32G
[service]
pagepool 2G
[compute]
pagepool 4G
[gpfs01,gpfs02]
trace all 9 # All major GPFS activities are traced with level 9 verbosity — useful for debugging but can be heavy on disk. It's important to monitor trace size.
[cesNodes]
trace all 9 iolite 0
[gpfs01,gpfs02,cesNodes]
tracedevTimeFormat calendar
[common]
minReleaseLevel 5.1.6.0
tiebreakerDisks GPFS_MD_LUN01;GPFS_MD_LUN03;GPFS_MD_LUN04
# “If quorum is lost, GPFS consults these disks to determine
# which partition of the cluster should stay active. It avoids data corruption during split-brain events.
cesCidrPool <IP=10.25.88.14><ATTRIBUTE=><GROUP=><PREFIX=>+<IP=10.25.88.200><ATTRIBUTE=><GROUP=><PREFIX=>+<IP=10.25.80.17><ATTRIBUTE=><GROUP=><PREFIX=>
adminMode central
```
## Module 2 - Detecting LUN failures, storage disconnections, and SAN issues

Goal:

Detect and investigate a simulated LUN failure or disconnected storage path in a GPFS cluster.

- **Step 1** – List Disks by Filesystem:

```bash
mmlsfs gpfs -d
flag                value                    description
------------------- ------------------------ -----------------------------------
 -d                 GPFS_MD_LUN01;GPFS_MD_LUN02;GPFS_DATA_LUN01;GPFS_DATA_LUN02;GPFS_MD_LUN03;GPFS_MD_LUN04;GPFS_DATA_LUN03;GPFS_DATA_LUN04  Disks in file system
```

Which LUNs (NSDs) are metadata vs data

- Match them with SANtricity volumes in our GUI web access:
  - [NetApp E2800](https://bruclu015-e2800b.vbr.is.keysight.com/sm/en-US/#/home)
  - [NetApp E2812](https://bruclu015-e2812a.vbr.is.keysight.com/sm/en-US/#/home)

- **Step 2** – View Disk State in GPFS:

```bash
mmlsdisk gpfs

disk         driver   sector     failure holds    holds                            storage
name         type       size       group metadata data  status        availability pool
------------ -------- ------ ----------- -------- ----- ------------- ------------ ------------
GPFS_MD_LUN01 nsd         512          -1 Yes      No    ready         up           system       
GPFS_MD_LUN02 nsd         512          -1 Yes      No    ready         up           system       
GPFS_DATA_LUN01 nsd         512          -1 No       Yes   ready         up           nlsas        
GPFS_DATA_LUN02 nsd         512          -1 No       Yes   ready         up           nlsas        
GPFS_MD_LUN03 nsd         512          -1 Yes      No    ready         up           system       
GPFS_MD_LUN04 nsd         512          -1 Yes      No    ready         up           system       
GPFS_DATA_LUN03 nsd         512          -1 No       Yes   ready         up           nlsas        
GPFS_DATA_LUN04 nsd         512          -1 No       Yes   ready         up           nlsas   
```

> A failed or disconnected LUN will appear in this output with an error status.
> `status=down, missing, or notReady`


- Look at:
    IOPS drop
    Health status: disk errors, failed controller, degraded pool

- **Step 3** – Check Physical Path (Optional)

```bash
multipath -ll
lsblk
```
LUNs are seen by the OS
Multipath reports healthy paths

## Module 3 - Network Diagnostics – InfiniBand & Omni-Path


> LAB: Diagnosing RDMA Fabric Issues in GPFS
> Goal: Detect whether InfiniBand or Omni-Path is properly configured and functional on a GPFS node, and verify the interfaces GPFS uses for RDMA.

- **Step 1** – Check IB or OPA Interface Status:

For InfiniBand Nodes:

```bash
ibstat
#output

CA 'mlx5_0'
    Port 1:
        State: Active
        Physical state: LinkUp
```
> If the state is Down or Initializing, there’s a cable, port, or switch issue.

For Omni-Path Nodes:
```bash
opainfo
#output
hfi1_0:1                           PortGID:0xfe80000000000000:00117501010957de
   PortState:     Active
   LinkSpeed      Act: 25Gb         En: 25Gb        
   LinkWidth      Act: 4            En: 4           
   LinkWidthDnGrd ActTx: 4  Rx: 4   En: 3,4         
   LCRC           Act: 14-bit       En: 14-bit,16-bit,48-bit       Mgmt: True 
   LID: 0x00000005-0x00000005       SM LID: 0x00000001 SL: 0 
         QSFP Copper,       3m  Hitachi Metals    P/N IQSFP26C-30       Rev 02
   Xmit Data:            1490350 MB Pkts:            425442181
   Recv Data:           11551433 MB Pkts:           2801740517
   Link Quality: 5 (Excellent)
```

> **Check:**
> Active ports (e.g. Active)
> LinkSpeed, LinkWidth and Link Quality

**Note:**
- `LinkWidth Act: 4 En: 4`
  - Act: (Active) – The actual number of lanes currently in use.
  - En: (Enabled) – The number of lanes that the port is capable of using (if available on both ends).
  - LinkWidth = 4 means the connection is using 4 lanes (commonly referred to as 4x in InfiniBand terminology)
  - This improves bandwidth and fault tolerance.
  - Each lane supports a certain bandwidth (e.g., 25Gbps), so 4x25Gbps = 100Gbps total throughput is possible.

> Context for Troubleshooting or Tuning:
> If Act: < En:, there may be a cabling issue, dirty connector, switch port misconfiguration, or hardware problem.
> If both are equal (as in our example), the link is fully utilized

- **Step 2** – Test Connectivity Between Nodes and Topology

  - Check the LID in our topology using `ibnetdiscover`
    ```bash
    [25]    "H-98039b03007fdc20"[1](98039b03007fdc20)               # "bruclu016-n01 mlx5_0" lid 30 4xEDR
    Ca      1 "H-98039b03007fdc20"          # "bruclu016-n01 mlx5_0"
    ``` 

```bash
ibping -S #ran in one node as gpfs01 or other
```
Now in another diff node session will execute the `ibping` command getting the LID number from the `ibnetdiscover` command will work as a IP address to complete the `ibping` command, see our example below:

```bash
[root@bruclu016-n02 ~]# ibping -c 5 30 #30 is the LID number from gpfs01
Pong from bruclu016-n01.vbr.is.keysight.com.rungislinux (Lid 30): time 0.232 ms
Pong from bruclu016-n01.vbr.is.keysight.com.rungislinux (Lid 30): time 0.200 ms
Pong from bruclu016-n01.vbr.is.keysight.com.rungislinux (Lid 30): time 0.203 ms
Pong from bruclu016-n01.vbr.is.keysight.com.rungislinux (Lid 30): time 0.200 ms
Pong from bruclu016-n01.vbr.is.keysight.com.rungislinux (Lid 30): time 0.210 ms
```
**Expect successful response** — latency in microseconds.

> **Note about LID Numbers:**
> LID 0 is reserved
> Valid LID goes from 1 to 64k: 1-65535

> Use ibping to test basic connectivity between two nodes at the InfiniBand level (not IP) is called IB Verb,
> indeed it's a transport layer specific to IB (such as MAC fo Ethernet, but different)

.”

- **Step 3** – Check What Ports GPFS Is Using:

```bash
mmlsconfig | grep verbsPorts
#Output
[nsdNodes]
verbsPorts mlx5_0/1/0  hfi1_0/1/1
```
