# HPC Roary Cluster — Intern Onboarding Guide

> **Who this is for:** New team members (interns, junior admins) joining the FIU HPC team with little or no prior HPC sysadmin experience.
> **Goal:** Give you enough context and hands-on knowledge to work confidently on the cluster within your first few weeks.
> **For deep command reference:** See `HPC_xCAT_Roary_Guide.md` (the companion admin guide).

---

## 1. What Is This Project?

### The Roary HPC Cluster
Florida International University runs a High-Performance Computing (HPC) cluster called **Roary** for faculty and student researchers. HPC clusters are large collections of computers working together so researchers can run jobs (simulations, data analysis, machine learning training) that are too big for a laptop or a single server.

As a technician on this team, your job is to keep the cluster healthy:
- Provision (install the OS on) new nodes when hardware arrives
- Diagnose and repair nodes that fail
- Maintain the provisioning infrastructure (xCAT, DNS, DHCP)
- Help keep the cluster available for researchers

### Scale of the Cluster
- **~50+ compute nodes** across multiple hardware generations
- **7 GPU nodes** (NVIDIA A100 GPUs) for deep learning and GPU-accelerated work
- **2 login nodes** (where researchers log in and submit jobs)
- **1 xCAT management node** (`hpcxcat`) that controls everything
- **1 SLURM management node** (`hpcslurm`) that schedules jobs
- Operating system: **AlmaLinux 9** (a RHEL-compatible Linux distro)

---

## 2. Cluster Architecture Overview

### Physical Layout (simplified)

```
                    ┌─────────────────────────────────┐
                    │         hpcxcat                 │
                    │   (xCAT Management Node)        │
                    │   IP: 192.168.246.2             │
                    │   Runs: xcatd, DHCP, DNS,       │
                    │         TFTP, NFS               │
                    └───────────────┬─────────────────┘
                                    │ Management Network (246.x)
              ┌─────────────────────┼──────────────────────┐
              │                     │                      │
     ┌────────┴──────┐    ┌────────┴──────┐    ┌──────────┴──────┐
     │  HPE Compute  │    │  Dell Compute │    │   GPU (Raptor)  │
     │  n126–n133    │    │  n095–n102    │    │ gpu-a100-01..07 │
     │  (8 nodes)    │    │  n001–n012    │    │   (7 nodes)     │
     └───────────────┘    └───────────────┘    └─────────────────┘
              │                     │                      │
              └─────────────────────┼──────────────────────┘
                                    │ Data Network (233.x)
                    ┌───────────────┴─────────────────┐
                    │       Lustre Parallel            │
                    │       Filesystem (Storage)       │
                    │   /home  /scratch                │
                    └─────────────────────────────────┘

     login1 (246.11), login2 (246.12) — researchers log in here
     hpcslurm — SLURM job scheduler (alma9 group)
```

### Three Networks — Why Three?

Every compute node has connections to **three separate networks**, each with a specific job:

| Network | Subnet | VLAN | Used for |
|---------|--------|------|---------|
| **IPMI / DRAC** | 192.168.254.x | 421 | Out-of-band management — power on/off, console access even when the OS is crashed |
| **Management (MGMT)** | 192.168.246.x | 421 | xCAT provisioning, SSH, in-band management. This is the "primary" address. |
| **Data** | 192.168.233.x | 418 | MPI job traffic, Lustre filesystem access — high-bandwidth, low-latency |

**Why separate?** If a node's OS crashes, you can still reach its IPMI interface to power-cycle it or read the console. The data network carries heavy traffic (parallel jobs) which you want isolated from management traffic.

### Software Stack

```
┌──────────────────────────────────────────────────────┐
│              Researcher Experience                   │
│   SLURM (job scheduler) — submit, queue, run jobs    │
├──────────────────────────────────────────────────────┤
│              Cluster Services                        │
│   Lustre (parallel filesystem: /home, /scratch)      │
│   LDAP (authentication — user accounts)              │
│   Nagios (monitoring — alerts when nodes go down)    │
│   Munge (authentication token for SLURM)             │
├──────────────────────────────────────────────────────┤
│              Infrastructure Layer                    │
│   xCAT (node provisioning, power, console mgmt)      │
│   DHCP + DNS (network identity for every node)       │
│   NFS/TFTP (serving OS images during boot)           │
├──────────────────────────────────────────────────────┤
│              Hardware                                │
│   HPE ProLiant / Dell PowerEdge / Raptor GPU nodes   │
│   IPMI (out-of-band management on every node)        │
└──────────────────────────────────────────────────────┘
```

As a technician, you mostly work at the **Infrastructure Layer** — xCAT, DHCP, DNS — and occasionally at the hardware layer (IPMI, physical swap).

---

## 3. Node Inventory

### Node Types

| Type | Nodes | Hardware | OS Image | What they do |
|------|-------|----------|---------|--------------|
| HPE Compute | n126–n133 | HPE ProLiant | alma9.5/9.7-netboot-compute | CPU-heavy research jobs |
| Dell Compute | n095–n102 | Dell PowerEdge | alma9.5-netboot-compute | CPU-heavy research jobs |
| Legacy Dell | n001–n012 | Dell PowerEdge | alma9.5-netboot-compute | Older compute capacity |
| GPU (Raptor) | gpu-a100-01–07, g009, cn1 | Dell R7525 + NVIDIA A100 | alma9.5-netboot-cudaruntime | Deep learning, GPU jobs |
| Login | login1, login2 | Dell | alma9.5-install-compute | Researcher login, job submission |
| SLURM Mgmt | hpcslurm | — | alma9 | Job scheduling |
| xCAT Mgmt | hpcxcat | — | — | Cluster management (YOU work here) |
| Data Mgmt | dm01 | Dell | alma9.5-install-compute | Data movement |
| Visualization | v003 | Dell R720 | alma9.5-netboot-desktop | Visual applications |

### Node Groups (xCAT)
Nodes are organized into groups. A node can belong to multiple groups — one for hardware type, one for OS, one for role.

```
hpe       → n126–n133 (HPE hardware)
dell      → n095–n102, n001–n012, login1, login2, dm01 (Dell hardware)
raptor    → gpu-a100-01 through gpu-a100-07, cn1, g009 (GPU nodes)
alma9     → all nodes running AlmaLinux 9
login     → login1, login2
slurmmn   → hpcslurm
xcatmn    → hpcxcat
```

To query live group membership:
```bash
nodels hpe          # list all HPE nodes
nodels raptor       # list all GPU nodes
lsdef -t group hpe  # show group definition
```

---

## 4. How Provisioning Works (End-to-End)

Understanding this flow is the single most important thing for debugging. When a node boots:

```
1. Node powers on
      │
2. BIOS/UEFI sends a DHCP request on the management NIC
      │
3. hpcxcat (DHCP server) responds with:
      - IP address for the node
      - Pointer to the TFTP server (also hpcxcat)
      │
4. Node downloads the xnba bootloader from TFTP
      (/tftpboot/xcat/xnba/...)
      │
5. xnba reads the node's boot config
      (/tftpboot/xcat/xnba/nodes/<node-ip>)
      - This file was written by `nodeset`
      │
6. Node fetches the OS kernel + initrd from TFTP
      │
7. OS boots and mounts the root filesystem image from NFS
      (/install/netboot/alma9.x/x86_64/compute/rootimg)
      │
8. Postscripts run (one-time setup: hostname, networking,
      LDAP, Lustre, SLURM, repo config, etc.)
      │
9. Node is up and available
```

**What can go wrong at each step:**
- Step 2–3: Node not in DHCP → run `makedhcp -a <node>`
- Step 4–5: Boot config missing → run `nodeset <node> osimage=<image>` then `makegocons`
- Step 6: TFTP error → check `/tftpboot` permissions or xCAT daemon
- Step 7: NFS mount failure → check NFS exports on hpcxcat
- Step 8: Postscript failure → check console with `rcons <node>` for error output
- Step 9: `lsdef <node>` shows `status=failed` → read the console log, check postscripts

---

## 5. The xCAT Management Node (hpcxcat)

Everything runs through `hpcxcat`. Log into it first for any cluster work:

```bash
ssh hpcxcat    # or ssh 192.168.246.2
```

### What Runs on hpcxcat
| Service | Purpose |
|---------|---------|
| `xcatd` | xCAT daemon — the core engine |
| `dhcpd` | DHCP server for all nodes |
| `named` | DNS server (ddns) |
| `tftp` | Serves boot files to nodes |
| `nfsd` | Serves OS root images to diskless nodes |
| `conserver` | Console aggregator (used by `rcons`) |

> **How xCAT was installed:** xCAT was installed from the official xCAT repository onto AlmaLinux 9 (`dnf install xCAT`). The xCAT daemon (`xcatd`) is the core engine — if it stops, all provisioning commands fail. Restart it with `service xcatd restart` and verify with `xcatversion`. Full installation steps are in `HPC_xCAT_Roary_Guide.md` Section 0.

### Key Directories

| Path | What's there |
|------|-------------|
| `/install/` | Root of everything xCAT-managed |
| `/install/alma9.5/x86_64/` | AlmaLinux 9.5 package tree (from ISO) |
| `/install/alma9.7/x86_64/` | AlmaLinux 9.7 package tree |
| `/install/netboot/alma9.x/x86_64/compute/` | Diskless root image for compute nodes |
| `/install/post/otherpkgs/alma9.x/x86_64/` | Extra RPMs added to diskless images |
| `/install/custom/netboot/alma/` | Custom pkglists, postinstall scripts |
| `/tftpboot/xcat/xnba/nodes/` | Per-node boot config (written by `nodeset`) |
| `/opt/xcat/share/xcat/netboot/alma/` | xCAT-provided pkglists, exlists, and postinstall templates for AlmaLinux |
| `/etc/xcat/` | xCAT daemon config files |
| `/etc/dhcp/dhcpd.conf` | DHCP config — **do not edit manually** |
| `/var/named/` | DNS zone files — **do not edit manually** |
| `/home/share/scripts/xCAT/` | xCAT DB backups |

> **Rule:** Never manually edit `/etc/dhcp/dhcpd.conf` or `/var/named/`. These are generated by xCAT (`makedhcp`, `makedns`). Manual edits will be overwritten and can break things.

---

## 6. Daily Operations — Commands You Will Actually Use

### Checking a Node's Status
```bash
lsdef -t node n126                          # full node definition from xCAT DB
lsdef -t node n126 -i ip,os,status         # just the key fields
rpower n126 stat                            # actual power state (on/off)
rinv n126 all                               # hardware inventory (model, serial, firmware)
```

### Opening a Node's Console
```bash
# IMPORTANT: console definition must be created first
makegocons n126                             # create/update console definition
rcons n126                                 # open serial console
# Exit the console: Ctrl-e, c, .
```

### Running Commands Across Multiple Nodes
```bash
xdsh n126 "uptime"                         # single node
xdsh hpe "df -h"                           # all HPE nodes
xdsh n126-n133 "systemctl status slurmd"   # node range
xdsh all "hostname"                        # every node (use carefully)
```

### Provisioning / Reprovisioning a Node
```bash
nodeset n126 osimage=alma9.5-x86_64-netboot-compute   # set what to install
rpower n126 reset                                      # reboot into new image
makegocons n126 && rcons n126                         # watch it boot
```

### DNS and DHCP Update Sequence
When adding or changing a node, always follow this order:

```bash
# 1. Update DHCP (node can now get an IP)
makedhcp -a <node>

# 2. Generate /etc/hosts from xCAT DB
makehosts                    # MUST come before makedns

# 3. Update DNS (reads the /etc/hosts entries makehosts just wrote)
makedns <node>
```

> **Common mistake:** Running `makedns` before `makehosts`. DNS will be stale or wrong. Always `makehosts` first.

---

## 7. Adding a New Node — Checklist

When new hardware arrives, follow these steps in order. Refer to `HPC_xCAT_Roary_Guide.md` Section 8 for the full commands.

```
[ ] 1. Cable the node and power it on in BIOS setup mode
[ ] 2. Configure DRAC/IPMI:
        - Assign IP from 192.168.254.x range
        - Enable IPMI over LAN as Administrator
        - Create user "adminuser" with cluster IPMI password
[ ] 3. Capture MAC addresses for all NICs from DRAC GUI
[ ] 4. Assign IPs:
        - DRAC:  192.168.254.xx  → name: rxxx  (e.g. r134)
        - MGMT:  192.168.246.xx  → name: nxxx  (e.g. n134)
        - Data:  192.168.233.xx  → hostname suffix: -data
[ ] 5. Record everything in the inventory spreadsheet (OneDrive: fiu_hpc_detailsv10)
[ ] 6. Add node to xCAT (nodeadd + chdef commands)
[ ] 7. Configure NIC attributes (bond0 for HPE, ens3f0 for GPU)
[ ] 8. Set postscripts and postbootscripts
[ ] 9. Update DHCP and DNS:
        makedhcp -a <node>
        makehosts
        makedns <node>
[ ] 10. Set the OS image: nodeset <node> osimage=<image>
[ ] 11. Create console definition: makegocons <node>
[ ] 12. Power on and watch console: rpower <node> on && rcons <node>
[ ] 13. Verify node comes up, postscripts complete, status=booted in lsdef
```

---

## 8. Reprovisioning an Existing Node

Reprovisioning means wiping and reinstalling the OS on a node — useful when a node is misbehaving, needs an OS upgrade, or its image is corrupt.

```bash
# 1. Confirm which image to use
lsdef -t node n126 -i provmethod,os

# 2. Set the target image (change osimage if upgrading)
nodeset n126 osimage=alma9.5-x86_64-netboot-compute

# 3. Make sure console is ready to watch
makegocons n126

# 4. Reboot the node
rpower n126 reset

# 5. Watch the boot on console
rcons n126

# 6. After boot, verify
lsdef -t node n126 -i status,updatestatus
```

For diskless nodes (most compute nodes), "reprovisioning" just reboots them into a fresh copy of the image — no data is lost because there is no local disk. For diskful nodes (login1, login2, dm01), reprovisioning wipes the local disk.

---

## 9. Troubleshooting Common Problems

### Node won't boot / stuck at PXE
1. Confirm the node definition exists: `lsdef -t node <node>`
2. Check DHCP: `makedhcp -q <node>` — does it return an IP?
3. Check nodeset was run: look in `/tftpboot/xcat/xnba/nodes/` for the node's IP file
4. Re-run: `nodeset <node> osimage=<image>` then `rpower <node> reset`

### `rcons` gives "no console" error
```bash
makegocons <node>   # create/refresh console definition first
rcons <node>
```

### DNS not resolving a new node
```bash
makehosts           # regenerate /etc/hosts first
makedns <node>      # then update DNS
# Verify:
host <node>.roary.net
```

### Node shows `status=failed` in lsdef
This means postscripts failed during provisioning.
```bash
makegocons <node>
rcons <node>        # read the console output for the error
# Common causes:
#   - Repo not reachable (roary-repo-alma9 failed)
#   - LDAP not reachable (roary-ldap failed)
#   - Network misconfigured (confignetwork failed)
```

### Can't ping a node that should be up
```bash
rpower <node> stat              # is it actually on?
makedhcp -q <node>              # is it getting the right IP?
xdsh <node> "ip addr"          # what IP does the node think it has?
```

### xCAT daemon not responding
```bash
service xcatd status
service xcatd restart           # last resort
```

---

## 10. Best Practices — Things to Remember

These were learned the hard way. Follow them.

### Before making bulk changes
```bash
# Always dump the xCAT database first
dumpxCATdb -a -p "/home/share/scripts/xCAT/hpcxcat-dump_$(date +%m%d%y)"
```
If you break something, `restorexCATdb` gets you back.

### Order matters for DNS/DHCP
```
makehosts  →  makedns    ← correct
makedns    →  makehosts  ← WRONG — makedns will use stale /etc/hosts
```

### Console before `rcons`
```
makegocons <node>  →  rcons <node>   ← correct
rcons <node>  (without makegocons)   ← will fail or connect to wrong console
```

### Test on one node first
```bash
# Bad: immediately run on all 50 nodes
xdsh all "my-new-script.sh"

# Good: test on one, then expand
xdsh n126 "my-new-script.sh"          # test
xdsh hpe "my-new-script.sh"           # expand to group
```

### Never edit these files manually
| File | Use instead |
|------|-------------|
| `/etc/dhcp/dhcpd.conf` | `makedhcp` |
| `/var/named/*.zone` | `makedns` |
| `/etc/hosts` | `makehosts` |
| `/tftpboot/xcat/xnba/nodes/*` | `nodeset` |

### Verify before you provision
```bash
lsdef -t node n126    # confirm all attributes are correct before nodeset
```
A missing `mac` or wrong `ip` in the node definition means the node won't boot.

### Understand noderanges
xCAT runs commands on all nodes in a range simultaneously. `xdsh all "reboot"` will reboot every node at once. Always double-check your noderange.

---

## 11. Understanding the Software Stack (Brief)

You won't configure these daily but you need to know they exist and what they do.

### SLURM (Simple Linux Utility for Resource Management)
The job scheduler. Researchers submit jobs (`sbatch`, `srun`) and SLURM decides which nodes to run them on. The SLURM daemon (`slurmd`) runs on compute nodes, installed by the `roary-slurm` postscript. The SLURM manager runs on `hpcslurm`. If a compute node's `slurmd` dies, jobs can't run on it.

```bash
xdsh dell "systemctl status slurmd"    # check SLURM on all Dell nodes
xdsh hpe "systemctl restart slurmd"    # restart if needed
```

### Lustre (Parallel Filesystem)
The shared storage mounted at `/home` and `/scratch` on all nodes. Provides fast parallel I/O for research data. Installed by `roary-setuplustre` postscript. If Lustre is unmounted, researchers can't access their files.

```bash
xdsh hpe "df -h /home /scratch"       # check Lustre mounts
```

### LDAP (Lightweight Directory Access Protocol)
User authentication. Every user account lives in LDAP, not on individual nodes. Installed by `roary-ldap` postscript. LDAP server is at `192.168.246.4`.

### Nagios
Monitoring system. Sends alerts when nodes go down or services fail. Installed on login nodes by `roary-nagios` postscript.

---

## 12. Useful References

| Resource | Where to find it |
|----------|-----------------|
| xCAT Admin Guide (detailed commands) | `HPC_xCAT_Roary_Guide.md` (this project directory) |
| Node inventory spreadsheet | OneDrive: `fiu_hpc_detailsv10` |
| xCAT documentation | https://xcat-docs.readthedocs.io/ |
| xCAT database schema | `man xcatdb` on hpcxcat |
| Table column descriptions | `man <tablename>` (e.g. `man nodetype`, `man nodehm`) |
| xCAT command man pages | `man lsdef`, `man nodeset`, `man xdsh`, `man makedns` |

### Quick Reference Card

```
DAILY COMMANDS
  lsdef -t node <node>            → show node config
  rpower <node> stat              → power state
  makegocons <node>               → prep console
  rcons <node>                    → open console (exit: Ctrl-e c .)
  xdsh <noderange> "<cmd>"        → run cmd on nodes

PROVISIONING
  nodeset <node> osimage=<image>  → set boot image
  rpower <node> on/reset          → power on/reboot
  nodeset + rpower                → reprovision

DNS/DHCP UPDATE ORDER
  makedhcp -a <node>
  makehosts                       ← always before makedns
  makedns <node>

BACKUP
  dumpxCATdb -a -p <dir>          → before any bulk change
  restorexCATdb -a -p <dir>       → to restore
```
