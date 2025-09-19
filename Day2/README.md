## Module 4 - CES Services

### **Step 1** – Confirm CES Is Running on Nodes:

```bash
mmces node list

Node Number   Node Name   Node Groups   Node Flags
------------- ----------- ------------- ------------ 
1             gpfs01                    none
2             gpfs02                    none
```

This output shows which nodes are part of the CES subsystem. We can see that `gpfs01` and `gpfs02` are *CES-enabled nodes*.

> **Meaning:** they can run services like **NFS, CIFS, or Object (S3)**
> Even through the CES node list shows `none` in the `Node Flags` above.

- The next command that will see below is `mmces service list`:

```bash
[root@bruclu016-n01 ~]# mmces service list
Enabled services: NFS SMB
NFS is running, SMB is running
```

`NFS` and `SMB` services are **up** and **running**.

This happens because CES can enable services cluster-wide without assigning them to a specific node groups or flags unless explicit configured.

> So the takeaway is: don't rely solely on `mmces node list` to confirm if a service is running, always follow up with `mmces service list` .

Also, CES floating IPs will be automatically assigned to eligible nodes when services are active

### **Step 2** – List Floating IPs (CES Addresses)

```bash
mmces address list

#Output:
Address        Node     Ces Group   Attributes
-------------- -------- ----------- ------------
10.25.80.17    gpfs02   none        none
10.25.88.14    gpfs01   none        none
10.25.88.200   gpfs02   none        none
```

🧠 Why it matters:
“If these IPs are missing, clients won’t be able to reach NFS/CIFS shares even if services are up.”

These IPs are floating IP addresses managed by CES. They are not permanently tied to the physical interfaces
but instead move between CES-enabled nodes to ensure high availability for NFS and SMB (CIFS) services.

In this output, we see three IPs:
- `10.25.88.14` currently assigned to `gpfs01`
- `10.25.80.17` and `10.25.88.200` assigned to `gpfs02`

Even though CES services (NFS and SMB) are running, if none of these floating IPs are present or bound to any node, then clients trying to mount exports via those addresses **would fail**.

> **Note**:
> So in a possible **troubleshooting**, if a user says:
> I can't mount the NFS export, and **CES services are running**, the very next thing to check is this **list of IPs**.
> If it’s empty or the IP they’re using is missing here, CES may have failed over or the IP wasn’t reassigned properly

What to Compare:
Run `ip a` on both `gpfs01` and `gpfs02`, and confirm that:
- `10.25.88.14` is present on `gpfs01`
- `10.25.80.17` and `10.25.88.200` are present on `gpfs02`

### **Step 3** – View Exported Paths for NFS:

```bash
mmnfs export list
#output
Path                Delegations  Clients  
------------------  -----------  -------  
/gpfs/scratch-mstr  NONE         *     
```

**Explanation:**
Displays the NFS export definitions: paths, allowed clients, options, and permissions.

- **Final Step** – Confirm CES IP Configuration in GPFS:

```bash
mmlsconfig | grep ces

#Output (from your cluster):
cesSharedRoot /gpfs/ces
cesCidrPool <IP=10.25.88.14><ATTRIBUTE=><GROUP=><PREFIX=>+<IP=10.25.88.200><ATTRIBUTE=><GROUP=><PREFIX=>+<IP=10.25.80.17><ATTRIBUTE=><GROUP=><PREFIX=>
```

This output above confirms that GPFS is properly configured to use three floating CES IPs:
1. `10.25.88.14`
2. `10.25.88.200`
3. `10.25.80.17`

The IPs are part of the `cesCidrPool`, which is the pool of addresses that the **CES** can assign dynamically to export services like NFS and SMB. GPFS will assign these IPs to CES nodes (like `gpfs01` or `gpfs02`) based on load failover or service enablement.
This configuration matches the output we saw in `mmces` address list and `ip a`
which means everything is **consistent**.


## Module 5 - Quota & Fileset Troubleshooting in GPFS:
### Goal: Learn how to identify quota problems for users and filesets, and how to simulate/explore a quota-exceeded situation



### **Step 1** – View Quota Enabled Status for Filesystem:

```bash
mmlsfs gpfs -Q
#output

flag                value                    description
------------------- ------------------------ -----------------------------------
 -Q                 user;group;fileset       Quotas accounting enabled
                    user;group;fileset       Quotas enforced
                    none                     Default quotas enabled
```

> This tells us that all types of quota tracking and enforcement are turned on. 
> So, if a user exceeds their limit, GPFS will actively block write operations.
> There’s no default quota, which means each user must be assigned a quota manually or by policy.



### **Step 2** – List Quota Usage by a Specific User:

```bash
mmlsquota -u <your-username> --block-size auto`
# Using the option --block-size will be more humanized output

                         Block Limits                                               |     File Limits
Filesystem Fileset    type         blocks      quota      limit   in_doubt    grace |    files   quota    limit in_doubt    grace  Remarks
gpfs       scratch-mstr USR          540.2M       300G       360G          0     none |       13       0        0        0     none 
```

The user is using `540.2MB` of their `300GB soft quota`. Hard quota limit is `360GB` — writes beyond this will be blocked immediately, because GPFS enforces the hard limit. Only 13 files are stored — and `0 file` count `quota` is enforced. *No data* is `in_doubt` (i.e., accounting is up-to-date).

> **Note:**
> `in_doubt` shows how much disk usage has occurred but has not yet been fully confirmed by all the nodes responsible for quota accounting in the GPFS. This can happen due to delays in synchronization, or if the usage happened on a node that is offline or currently unreachable.

> **Important to know:**
> This is where you’ll confirm whether the error came from a `soft` or `hard limit` breach.


- ### **Step 3** – View Quotas per Fileset (Optional):

```bash
mmlsquota -j scratch-mstr gpfs

                         Block Limits                                    |     File Limits
Filesystem type             KB      quota      limit   in_doubt    grace |    files   quota    limit in_doubt    grace  Remarks
gpfs       FILESET     no limits                                    
```

This output tells us that the `scratch-mstr` fileset has no defined quota limits at the **fileset level**. 

> In the GPFS, you can apply quotas at three levels:

1. Per user
2. Per group
3. Per fileset

What we’re seeing here is that the fileset itself is not restricted — users can fill it as long as they remain within their personal or group limits. This is common in shared scratch areas, where individual user quotas are enforced but the fileset is left open for flexibility.”

### **Step 4** – Full Report of All Users

```bash
mmrepquota -u -v gpfs --block-size auto
```

**Great for:**
- Quota audits;
- Detecting heavy users;
- Preparing reports;

> **Tip:**
> GPFS doesn’t sort the output natively, but we can use Unix tools like sort to identify the top users by disk usage.
> This is very helpful in quota audits, or when a filesystem is nearly full and you need to know who’s consuming the most.”

- **Organizing by space used:**

```bash
mmrepquota -u -v gpfs --block-size auto | sort -k4 -h -r | more
# To show only top 15 users
```
```bash
mmrepquota -u -v gpfs --block-size auto | sort -k4 -h -r | head -n 15

Part						Purpose
-u	    					User quota
-v							Verbose output
--block-size auto			Human-readable sizes (KB, MB, GB)
sort -k4					Sort by 4th column (blocks used)
-h							Human-readable sort (handles MB/GB/etc.)
-r                          Reverse the result of comparisons
```			

## Module 6 – Node Management Hands-On Lab

### **Step 1**: How to show all nodes per group:

```bash
mmlsnodeclass
#output

Node Class Name       Members
--------------------- -----------------------------------------------------------
opaServers            gpfs01,gpfs02,n095,n097,n098,n099,n101,n103,n104,...
compute               n131,n095,n097,n098,n099,n101,n103,n104,n106,...
service               gpfs01,gpfs02
ibServers             gpfs01,gpfs02,n131,n132,n133,n134,n135,n136,...
```

- **How your nodes are organized by roles:**
    1. `opaServers` → Nodes with Omni-Path interfaces; 
    2. `ibServers` → Nodes with InfiniBand interfaces;
    3. `compute` → Regular compute nodes;
    4. `service` → Metadata / CES servers.

> **🧠 Training Insight:**
> This command helps us understand how the GPFS cluster is logically segmented.
> If you're troubleshooting I/O latency or connection issues, you’ll know which nodes use which interconnect (OPA vs IB), and which are providing services like CES or metadata.
> It’s also important when adding new nodes — we need to know which class they should belong to.

### **Step 2** – Check Node Status in Real Time

```bash
mmgetstate -a
```
> **🧠 Explanation:**
> Use this to quickly detect unhealthy or unreachable nodes.
> Nodes in unknown state may have networking or service issues.

### **Step 3** – View Node Licensing (Safe Read-Only)

```bash
mmlslicense
#output
Summary information 
---------------------
Number of nodes defined in the cluster:                         85
Number of nodes with server license designation:                 2
Number of nodes with FPO license designation:                    0
Number of nodes with client license designation:                83
Number of nodes still requiring server license designation:      0
Number of nodes still requiring client license designation:      0
This node runs IBM Spectrum Scale Standard Edition
```


Using mmlslicense, we get a quick overview of our license distribution. We have 85 nodes in total: 2 are acting as servers (most likely gpfs01 and gpfs02), and the rest are clients. No nodes are unlicensed, which is good — if we were scaling the cluster, we’d need to revisit this to ensure we stay compliant.

We're also using the Standard Edition, which covers all the GPFS functionality we use today, such as *quotas*, *filesets*, and *CES*. 
Advanced features would require licensing upgrades



## Module 7 - Investigate Logs for Issues

How to identify GPFS problems with key log files



### **Step 1** – Check if GPFS is Healthy

- Make sure all nodes are active using `mmgetstate -a`

### **Step 2** – Tail the Main Log

```bash
tail -n 20 /var/adm/ras/mmfs.log.latest
#output

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
```

> **Tip:**
> 🔍 Look for recent activity, mount events, quorum messages, or node failures


### **Step 3** – Search for Errors:

```bash
grep -i error /var/adm/ras/mmfs.log.latest | tail -n 10

#output
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
```

> **Vocabulary**
> `VERBS RDMA closed connection` >> An RDMA connection between two nodes was unexpectedly closed.
> `mlx5_0 port 1` >> Refers to the InfiniBand (or RoCE) interface and port — mlx5_0 is the device name.
> `fabnum, index, cookie` >> Internal identifiers for the RDMA fabric and session — mostly for IBM support.
> `error 233` >> Indicates a connection timeout or remote host unreachability.
> `error 73` >> Typically means software caused connection abort — could be related to high congestion, reset, or failure on remote side.


### **Step 4** – Additional informations:

You can check other logs from `mmfs.log` listing all logs including the rotate logs also.

```bash
ls -lh /var/adm/ras/mmfs.log.*
```

