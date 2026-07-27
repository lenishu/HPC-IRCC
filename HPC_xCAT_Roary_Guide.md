# HPC Roary Cluster — xCAT Administration Guide

> **Audience:** HPC technicians performing cluster provisioning, node management, and maintenance.
> **xCAT master node:** `hpcxcat` — IP `192.168.246.2`
> **Cluster domain:** `roary.net`

---

## 0. xCAT Installation

> **When to use this section:** Follow these steps when installing xCAT on a fresh AlmaLinux 9 management node. Skip if xCAT is already installed — jump to Section 1.

### Prerequisites
- AlmaLinux 9 installed and network-configured on the management node (`hpcxcat`)
- Management NIC `eth3` assigned IP `192.168.246.2`
- Root access

### Add the xCAT Repository
```bash
# xCAT core packages
wget https://raw.githubusercontent.com/xcat2/xcat-core/master/xCAT-core.repo \
    -P /etc/yum.repos.d/

# xCAT dependency packages (AlmaLinux 9 / RHEL 9)
wget https://raw.githubusercontent.com/xcat2/xcat-dep/master/rh9/x86_64/xCAT-dep.repo \
    -P /etc/yum.repos.d/
```

### Install xCAT
```bash
dnf clean metadata
dnf install xCAT
```

### Start xCAT
```bash
# Load xCAT environment variables and commands into PATH
source /etc/profile.d/xcat.sh

# Enable and start the xCAT daemon
systemctl enable xcatd
systemctl start xcatd

# Verify installation
xcatversion
```

### Initial Site Configuration
Set cluster-wide defaults once after installation:
```bash
chdef -t site clustersite \
    master=192.168.246.2 \
    domain=roary.net \
    nameservers=192.168.246.2 \
    ntpservers=192.168.246.2 \
    forwarders=192.168.248.60 \
    dhcpinterfaces=eth3 \
    dnsinterfaces=eth3 \
    dnshandler=ddns \
    dnsupdatemethod=ddns \
    installdir=/install \
    tftpdir=/tftpboot \
    timezone=America/New_York \
    sshbetweennodes=ALLGROUPS \
    precreatemypostscripts=1 \
    sharedtftp=1
```

Verify:
```bash
lsdef -t site clustersite
```

### Initialize Network Services (run once)
```bash
makedhcp -n      # generate initial /etc/dhcp/dhcpd.conf
makedns -n       # generate initial DNS zones under /var/named/
makehosts        # populate /etc/hosts from xCAT DB
```

### Service Management
```bash
service xcatd stop
service xcatd start
service xcatd restart
service xcatd status

# Confirm version and connectivity
xcatversion
```

---

## 1. Cluster Overview

### Management Node (hpcxcat)
| Attribute | Value |
|-----------|-------|
| Hostname | `hpcxcat` |
| IP / VIP | `192.168.246.2` |
| Role | xCAT management, DHCP, DNS, TFTP, NFS |
| DHCP/DNS interface | `eth3` |
| DNS handler | ddns (dynamic) |
| NTP server | `192.168.246.2` |
| DNS forwarder | `192.168.248.60` |
| Domain | `roary.net` |
| Install directory | `/install` |
| TFTP directory | `/tftpboot` |
| Timezone | `America/New_York` |

### Key Infrastructure Nodes
| Node | IP | Groups | Role |
|------|----|--------|------|
| hpcxcat | 192.168.246.2 | xcatmn | xCAT management node |
| hpcslurm | — | slurmmn | SLURM resource manager |
| login1 | 192.168.246.11 | login, alma9, dell | User login node |
| login2 | 192.168.246.12 | login, alma9, dell | User login node |
| dm01 | — | dm | Data management node |
| v003 | — | viz, R720 | Visualization node |

### Network Layout
| Interface | VLAN | Subnet | Purpose |
|-----------|------|--------|---------|
| DRAC/IPMI | 421 | 192.168.254.x | BMC / out-of-band management |
| Management (MGMT) | 421 | 192.168.246.x | xCAT provisioning, in-band mgmt |
| Data | 418 | 192.168.233.x | MPI / application traffic |

### Node Naming Convention
| Prefix | Interface | Example |
|--------|-----------|---------|
| `rxxx` | DRAC/IPMI BMC | `r126`, `raptor01b` |
| `nxxx` / `gxxx` / `vxxx` | MGMT (compute / GPU / viz) | `n126`, `gpu-a100-01`, `v003` |
| `<node>-data` | Data VLAN 418 (auto-generated hostname) | `n126-data`, `gpu-a100-01-data` |
| `<node>-mgt` | MGMT VLAN 421 alias | `n126-mgt` |

### DNS / Hosts Example (n126)
```
192.168.246.196   n126         n126.roary.net          # MGMT
192.168.233.196   n126-data    n126-data.roary.net     # Data VLAN 418
```

---

## 2. xCAT Basic Concepts

### Architecture
xCAT (Extreme Cluster Administration Toolkit) follows a management-node-centric model:

```
hpcxcat (Management Node)
  ├── Runs: xcatd, DHCP, DNS (ddns), TFTP, NFS
  ├── Stores: all cluster config in the xCAT database
  ├── Controls: provisioning, power, console for all nodes
  └── Serves: OS images, postscripts, syncfiles to nodes
```

Nodes boot via **xnba** (xCAT Network Boot Agent) over the management network (VLAN 421, 192.168.246.x), pull their OS image from NFS/TFTP, and run postscripts on first boot.

### xCAT Database
All configuration is stored in a relational database on hpcxcat. xCAT exposes it through **object definitions** — a higher-level view that joins multiple tables. The key object types are:

| Object Type | What it stores |
|-------------|---------------|
| `node` | All attributes of a single node (IP, MAC, OS, groups, postscripts, NIC config, …) |
| `group` | A named collection of nodes; shared attributes can be set at group level |
| `osimage` | Diskless or diskful OS image definition (paths, pkglists, provmethod) |
| `network` | Subnet definitions (gateway, mask, DHCP range) |
| `site` | Global cluster-wide settings (domain, master, DNS, NTP, …) |
| `policy` | Access control rules for xCAT commands |
| `route` | Static route definitions |

### Key Underlying Tables
These tables back the `node` object — understanding them helps when using `tabdump` or `man <tablename>`:

| Table | Content |
|-------|---------|
| `nodelist` | Node-to-group membership |
| `nodetype` | OS, arch, profile, provmethod per node |
| `nodehm` | Hardware management: power method, console, IPMI settings |
| `noderes` | Boot resources: netboot method, installnic, xcatmaster, tftpserver |
| `mac` | MAC address to node mapping |
| `hosts` | IP, hostname, aliases for DNS/hosts generation |
| `networks` | Network definitions used by DHCP and DNS |
| `passwd` | Credentials (IPMI admin password, system passwords) |
| `postscripts` | Postscript and postbootscript assignments |

### Provisioning Methods
| Method | Description | Used for |
|--------|-------------|---------|
| `netboot` | Diskless — node boots entirely from RAM image over network | Most compute nodes |
| `install` | Diskful — full OS installed to local disk | login1, login2, dm01 |
| `statelite` | Hybrid — diskless with persistent overlay | Not currently active |

---

## 3. xCAT Global Configuration

The **site** object holds cluster-wide defaults. View with:
```bash
lsdef -t site clustersite
```

### Current Site Table (hpcxcat)
| Setting | Value | Description |
|---------|-------|-------------|
| `master` | `192.168.246.2` | xCAT management node IP (VIP) |
| `domain` | `roary.net` | Cluster DNS domain |
| `nameservers` | `192.168.246.2` | DNS server for nodes |
| `ntpservers` | `192.168.246.2` | NTP server for nodes |
| `forwarders` | `192.168.248.60` | Upstream DNS forwarder |
| `dhcpinterfaces` | `eth3` | Interface xCAT manages DHCP on |
| `dnsinterfaces` | `eth3` | Interface xCAT manages DNS on |
| `dnshandler` | `ddns` | Dynamic DNS update method |
| `dnsupdatemethod` | `ddns` | DNS update mechanism |
| `installdir` | `/install` | Root of OS images, packages, postscripts |
| `tftpdir` | `/tftpboot` | TFTP root for netboot files |
| `timezone` | `America/New_York` | Timezone set on provisioned nodes |
| `sshbetweennodes` | `ALLGROUPS` | SSH key distribution scope |
| `precreatemypostscripts` | `1` | Pre-generate postscript bundles |
| `sharedtftp` | `1` | Single shared TFTP server |
| `dhcplease` | `43200` | DHCP lease time (seconds = 12 hrs) |
| `xcatdport` | `3001` | xCAT daemon port |
| `xcatiport` | `3002` | xCAT info port |

### Modify a site setting
```bash
chdef -t site clustersite <attribute>=<value>
# Example: update NTP server
chdef -t site clustersite ntpservers=192.168.246.2
```

---

## 4. Manipulating DB Tables and Objects

### Object vs. Table Commands
xCAT provides two levels of access:

| Level | Commands | Use when |
|-------|----------|---------|
| **Object** (recommended) | `lsdef`, `mkdef`, `chdef`, `rmdef` | Day-to-day work — cleaner, joins multiple tables |
| **Table** (raw) | `tabdump`, `tabedit`, `chtab` | Bulk edits, scripting, inspecting raw data |

### Object Commands
```bash
# List objects
lsdef -t node n126                        # all attributes of n126
lsdef -t node -i ip,mac n126             # specific attributes only
lsdef -t group raptor                     # group definition
lsdef -t osimage                          # list all OS images
lsdef -t site clustersite                 # global config
lsdef -t network                          # all network definitions
lsdef -a                                  # everything

# Create
mkdef -t node newnode groups=hpe,alma9 ip=192.168.246.xx mac=<mac>

# Modify
chdef -t node n126 os=alma9.7            # change single attribute
chdef -t node n126 ip=192.168.246.196 mac=d4:04:e6:06:04:ac  # multiple

# Remove
rmdef -t node retirednode
rmdef -t group oldgroup
```

### Node Shorthand Commands
```bash
nodels                          # list all nodes
nodels raptor                   # list nodes in a group
nodels n126 groups              # show group membership of n126
nodeadd n127 groups=hpe,alma9   # add node with groups
nodech n126 os=alma9.7          # shorthand for chdef -t node
noderm retirednode              # shorthand for rmdef -t node
```

### Node Range Syntax
xCAT accepts flexible node ranges in any command:

```bash
n126                    # single node
n126,n127,n128          # comma-separated list
n126-n133               # numeric range (expands to n126,n127,...,n133)
hpe                     # all nodes in group hpe
hpe,dell                # union of two groups
all                     # every node in the DB
```

### Raw Table Commands
```bash
tabdump nodetype                          # dump nodetype table as CSV
tabdump -d nodetype                       # dump with column descriptions
tabedit nodetype                          # open table in $EDITOR
chtab node=n126 nodetype.os=alma9.7      # set one cell directly
```

### Getting Help
```bash
man lsdef                # object command reference
man mkdef
man xcatdb               # complete database schema reference
man nodelist             # schema for a specific table
man nodetype
man nodehm
man noderes
```

---

## 5. Node Groups

Query with: `nodels` or `lsdef -t group`

> Postscripts are assigned **per node**, not per group in this cluster.

| Group | Members / Purpose |
|-------|-------------------|
| `xcatmn` | hpcxcat — xCAT management node |
| `slurmmn` | hpcslurm — SLURM manager |
| `login` | login1, login2 |
| `hpe` | n126–n133 (HPE ProLiant compute nodes) |
| `dell` | n095–n102, n001–n012, login1, login2, dm01 (Dell compute nodes) |
| `alma9` | All nodes running AlmaLinux 9 |
| `raptor` | gpu-a100-01 through gpu-a100-07, cn1, g009 (GPU nodes) |
| `raptor_ipmi` | raptor01b–raptor07b (IPMI interfaces for raptor nodes) |
| `16C_128G` | n001, n003, n006, n007, n009, n010, n011, n012 (16-core/128 GB nodes) |
| `dm` | dm01 |
| `viz` | v003 |
| `vizmgt` | rv003 |
| `mgt` | All rack BMC nodes (r126, r127, … rlogin1, rlogin2, rdm01, …) |
| `mgmt` | rg009 |
| `hpcdns` | l01, l02, l03 |
| `ldap` | LDAP server — IP 192.168.246.4 |
| `mirror` | Package mirror — IP 192.168.246.5 |
| `R720` | v003 (Dell R720 hardware) |

---

## 6. OS Images

Query all images: `lsdef -t osimage`

### Complete Image Inventory
| Image Name | Type | Use |
|------------|------|-----|
| `alma9.5-x86_64-install-compute` | diskful | login1, login2, dm01 |
| `alma9.5-x86_64-install-service` | diskful | service nodes |
| `alma9.5-x86_64-netboot-compute` | diskless | HPE/Dell compute (n095–n133) |
| `alma9.5-x86_64-netboot-cudaruntime` | diskless | GPU nodes (raptor) |
| `alma9.5-x86_64-netboot-desktop` | diskless | v003 (visualization) |
| `alma9.5-x86_64-statelite-compute` | statelite | — |
| `alma9.6-x86_64-install-compute` | diskful | — |
| `alma9.6-x86_64-install-service` | diskful | — |
| `alma9.6-x86_64-netboot-compute` | diskless | — |
| `alma9.6-x86_64-netbootIB-compute` | diskless+IB | — |
| `alma9.6-x86_64-statelite-compute` | statelite | — |
| `alma9.7-x86_64-install-compute` | diskful | — |
| `alma9.7-x86_64-install-service` | diskful | — |
| `alma9.7-x86_64-netboot-compute` | diskless | newer compute nodes |
| `alma9.7-x86_64-netboot-cudaruntime` | diskless | newer GPU nodes |
| `alma9.7-x86_64-netbootIB-compute` | diskless+IB | IB-connected nodes (n128–n133, n001) |
| `alma9.7-x86_64-statelite-compute` | statelite | — |

### alma9.5-x86_64-netboot-compute (active — standard compute)
```
Object name: alma9.5-x86_64-netboot-compute
    exlist=/opt/xcat/share/xcat/netboot/alma/compute.alma9.x86_64.exlist
    imagetype=linux
    osarch=x86_64
    osdistroname=alma9.5-x86_64
    osname=Linux
    osvers=alma9.5
    otherpkgdir=/install/post/otherpkgs/alma9.5/x86_64
    pkgdir=/install/alma9.5/x86_64
    pkglist=/opt/xcat/share/xcat/netboot/alma/compute.alma9.x86_64.pkglist
    postinstall=/opt/xcat/share/xcat/netboot/alma/compute.alma9.x86_64.postinstall
    profile=compute
    provmethod=netboot
    rootimgdir=/install/netboot/alma9.5/x86_64/compute
```

### alma9.7-x86_64-netbootIB-compute (active — InfiniBand nodes)
```
Object name: alma9.7-x86_64-netbootIB-compute
    exlist=/opt/xcat/share/xcat/netboot/alma/compute.alma9.x86_64.exlist
    kernelver=5.14.0-611.54.3.el9_7.x86_64
    imagetype=linux
    osarch=x86_64
    osdistroname=alma9.7-x86_64
    osname=Linux
    osvers=alma9.7
    otherpkgdir=/install/post/otherpkgs/alma9.7/x86_64
    otherpkglist=/install/custom/netboot/alma/ib.alma9.7.doca.pkglist
    permission=755
    pkgdir=/install/alma9.7/x86_64,/install/kernels/5.14.0-611.54.3.el9_7.x86_64
    pkglist=/opt/xcat/share/xcat/netboot/alma/compute.alma9.x86_64.pkglist
    postinstall=/opt/xcat/share/xcat/netboot/alma/compute.alma9.x86_64.postinstall
    profile=compute
    provmethod=netboot
    rootimgdir=/install/netboot/alma9.7/x86_64/compute-ib
```

---

## 7. xCAT Command Reference

### Database Object Commands
| Command | Purpose |
|---------|---------|
| `lsdef -t node <node>` | Show all attributes of a node |
| `lsdef -t group <group>` | Show group definition |
| `lsdef -t osimage` | List all OS images |
| `lsdef -t osimage <name>` | Show OS image details |
| `lsdef -t site clustersite` | Show global xCAT site configuration |
| `mkdef -t node <node> <attrs>` | Create a new node definition |
| `chdef -t node <node> <attrs>` | Modify node attributes |
| `rmdef -t node <node>` | Remove a node definition |
| `nodeadd <node> groups=<g1>,<g2>` | Add node and assign to groups |
| `nodels` | List all nodes |
| `nodels <group>` | List nodes in a group |

### Provisioning Commands
| Command | Purpose |
|---------|---------|
| `nodeset <node> osimage=<image>` | Set next boot OS image |
| `rpower <node> stat` | Check power state |
| `rpower <node> on` | Power on |
| `rpower <node> off` | Power off |
| `rpower <node> reset` | Reboot |
| `rcons <node>` | Open serial console (exit: `Ctrl-e c .`) |

### DHCP / DNS Commands
| Command | Purpose |
|---------|---------|
| `makedhcp -n` | Regenerate dhcpd.conf (run once) |
| `makedhcp -a <node>` | Add node to DHCP |
| `makedhcp -d <node>` | Remove node from DHCP |
| `makedhcp -q <node>` | Query node DHCP entry |
| `makedns -n` | Regenerate DNS zones (run once) |
| `makedns <node>` | Add/update node in DNS |
| `makedns -d <node>` | Remove node from DNS |
| `makehosts` | Sync /etc/hosts from xCAT DB |

### Image Commands
| Command | Purpose |
|---------|---------|
| `copycds -n <osver> -a x86_64 <iso>` | Import ISO into xCAT |
| `genimage <osimage>` | Generate diskless image |
| `packimage <osimage>` | Pack diskless image for deployment |

### Parallel / Remote Commands
| Command | Purpose |
|---------|---------|
| `xdsh <noderange> <cmd>` | Run command on multiple nodes concurrently |
| `xdcp <noderange> <src> <dst>` | Copy files to multiple nodes |

### Backup / Restore
| Command | Purpose |
|---------|---------|
| `dumpxCATdb -a -p <dir>` | Dump all xCAT DB tables to CSV |
| `restorexCATdb -a -p <dir>` | Restore xCAT DB from CSV dump |

---

## 8. Adding a New Compute Node (Step-by-Step)

### Step 1 — Configure DRAC/IPMI on the physical node
1. Assign a free IP from DRAC VLAN 421: `192.168.254.xx`
2. Enable IPMI over LAN as Administrator
3. Create user `adminuser` with the password matching the xCAT `passwd` table IPMI entry
4. Capture MAC addresses for all network interfaces from the DRAC GUI

### Step 2 — Assign IPs for all interfaces
Plan three IPs and record them in the cluster inventory spreadsheet on OneDrive (`fiu_hpc_detailsv10`):
- **DRAC/IPMI** (`192.168.254.xx`) — BMC interface
- **MGMT** (`192.168.246.xx`) — xCAT provisioning interface
- **Data** (`192.168.233.xx`) — application/MPI traffic (becomes `<node>-data`)

### Step 3 — Add node definitions in xCAT

#### HPE compute node (bond NIC pattern — e.g. n126)
HPE nodes use a bonded interface (eth0+eth1, 802.3ad LACP) with VLAN sub-interfaces.

```bash
# Add to groups
nodeadd n126 groups=hpe,alma9

# Core attributes
chdef -t node n126 \
    ip=192.168.246.xx \
    mac=<mgmt_mac> \
    installnic=<mgmt_mac> \
    netboot=xnba \
    os=alma9.5 \
    arch=x86_64 \
    mgt=ipmi \
    bmc=r126 \
    xcatmaster=hpcxcat \
    serialport=0 \
    serialspeed=115200 \
    addkcmdline="biosdevname=0 net.ifnames=0"

# Bond NIC configuration
chdef -t node n126 \
    nicdevices.bond0="eth0|eth1" \
    nicdevices.bond0.418=bond0 \
    nictypes.bond0=bond \
    nictypes.eth0=Ethernet \
    nictypes.eth1=Ethernet \
    nictypes.bond0.418=vlan \
    nictypes.bond0.421=vlan \
    nicips.bond0=192.168.246.xx \
    nicips.bond0.418=192.168.233.xx \
    nicnetworks.bond0=net427 \
    nicnetworks.bond0.418=data \
    nichostnamesuffixes.bond0.418=-data \
    nichostnamesuffixes.bond0.421=-mgt \
    "nicextraparams.bond0=mode=802.3ad miimon=100 ipv4.dns=192.168.246.2 ipv4.dns-search=roary.net"

# Postscripts and postbootscripts
chdef -t node n126 \
    postscripts="syslog,remoteshell,syncfiles,setupntp,roary-sethostname,confignetwork,roary-repo-alma9,roary-ldap,roary-nouserlogin,roary-setuplustre,roary-slurm" \
    postbootscripts=otherpkgs

# Add DRAC/BMC interface
nodeadd r126 groups=mgt
chdef -t node r126 ip=192.168.254.xx mgt=ipmi
```

#### GPU node (single NIC pattern — e.g. gpu-a100-01)
Raptor/GPU nodes use a single Ethernet NIC (ens3f0) with a VLAN sub-interface for data.

```bash
nodeadd gpu-a100-01 groups=raptor,alma9

chdef -t node gpu-a100-01 \
    ip=192.168.246.xx \
    mac=<mgmt_mac> \
    netboot=xnba \
    os=alma9.5 \
    arch=x86_64 \
    mgt=ipmi \
    bmc=raptor01b \
    xcatmaster=hpcxcat

# NIC configuration
chdef -t node gpu-a100-01 \
    nicdevices.ens3f0.418=ens3f0 \
    nictypes.ens3f0=Ethernet \
    nictypes.ens3f0.418=Vlan \
    nicips.ens3f0=192.168.246.xx \
    nicips.ens3f0.418=192.168.233.xx \
    nicnetworks.ens3f0=net427 \
    nicnetworks.ens3f0.418=data \
    nichostnamesuffixes.ens3f0.418=-data

chdef -t node gpu-a100-01 \
    postscripts="syslog,remoteshell,syncfiles,setupntp,confignetwork,roary-ldap,roary-nouserlogin,roary-setuplustre,roary-slurm" \
    postbootscripts=otherpkgs
```

### Step 4 — Update DHCP and DNS
```bash
makedhcp -a n126
makedhcp -a r126
makehosts           # must run first — populates /etc/hosts
makedns n126        # reads entries from /etc/hosts
makedns r126
```

Verify DHCP entry:
```bash
makedhcp -q n126
# Expected: n126: ip-address = 192.168.246.xx, hardware-address = <mac>
```

Verify /etc/hosts entry:
```bash
grep n126 /etc/hosts
# 192.168.246.xx   n126      n126.roary.net
# 192.168.233.xx   n126-data n126-data.roary.net
```

### Step 5 — Set boot state and provision
```bash
# Set OS image for next boot
nodeset n126 osimage=alma9.5-x86_64-netboot-compute

# Verify definition
lsdef n126

# Power on to trigger xnba boot
rpower n126 on
```

`nodeset` writes the boot config to `/tftpboot/xcat/xnba/nodes/`.

### Step 6 — Monitor installation
```bash
makegocons n126     # create console definition first
rcons n126          # watch console (exit: Ctrl-e c .)
rpower n126 stat    # check power state
```

---

## 9. Postscripts Reference

Postscripts are assigned per node (not per group). Set with:
```bash
chdef -t node <node> postscripts="<comma-separated-list>"
chdef -t node <node> postbootscripts=otherpkgs
```

### Common postscripts by node type

| Script | Compute (HPE/Dell) | GPU (raptor) | Login |
|--------|--------------------|--------------|-------|
| syslog | ✓ | ✓ | ✓ |
| remoteshell | ✓ | ✓ | ✓ |
| syncfiles | ✓ | ✓ | ✓ |
| setupntp | ✓ | ✓ | ✓ |
| roary-sethostname | ✓ | — | ✓ |
| confignetwork | ✓ | ✓ | ✓ |
| setroute replace | — | — | ✓ |
| roary-repo-alma9 | ✓ | — | ✓ |
| roary-ldap | ✓ | ✓ | ✓ |
| roary-nouserlogin | ✓ | ✓ | ✓ |
| roary-setuplustre | ✓ | ✓ | ✓ |
| roary-slurm | ✓ | ✓ | — |
| roary-motd | — | — | ✓ |
| roary-nagios | — | — | ✓ |
| roary-admin | — | — | ✓ |
| roary-ondemand | — | — | ✓ |
| roary-munge | — | — | ✓ |

**postbootscripts:** `otherpkgs` (all node types)

---

## 10. Diskless Image Management

### Import a new AlmaLinux ISO
```bash
copycds -n alma9.5 -a x86_64 AlmaLinux-9.5-x86_64-dvd.iso
```
Creates by default:
- `alma9.5-x86_64-install-compute` — diskful (stateful) installation
- `alma9.5-x86_64-netboot-compute` — diskless (netboot/RAMdisk)

### Generate a diskless image
```bash
genimage alma9.5-x86_64-netboot-compute
```
Root filesystem built under `/install/netboot/alma9.5/x86_64/compute/`.

Optionally chroot to make manual adjustments before packing:
```bash
chroot /install/netboot/alma9.5/x86_64/compute/rootimg
```

### Pack and deploy
```bash
packimage alma9.5-x86_64-netboot-compute
nodeset <noderange> osimage=alma9.5-x86_64-netboot-compute
rpower <noderange> reset
```

### Add additional packages to an image
1. Place RPMs in `/install/post/otherpkgs/alma9.5/x86_64/`
2. Run `createrepo /install/post/otherpkgs/alma9.5/x86_64/`
3. Update `otherpkglist` attribute of the osimage: `chdef -t osimage alma9.5-x86_64-netboot-compute otherpkglist=<path>`
4. Re-run `genimage` and `packimage`

---

## 11. DHCP and DNS Configuration

### DHCP
Config file: `/etc/dhcp/dhcpd.conf` (managed by xCAT — do not edit manually)  
Interface: `eth3`

```bash
makedhcp -n                  # Rebuild dhcpd.conf from scratch (run once)
makedhcp -a <node>           # Add or update a node
makedhcp -d <node>           # Remove a node
makedhcp -q <node>           # Query current DHCP entry
```

### DNS
Zone files: `/var/named/` (dynamic DNS via ddns)  
Interface: `eth3`

```bash
makedns -n                   # Rebuild all DNS zones (run once)
makedns <node>               # Add or update a node
makedns -d <node>            # Remove a node from DNS
makehosts                    # Rebuild /etc/hosts from xCAT DB
```

After `makehosts`, verify:
```bash
grep n126 /etc/hosts
# 192.168.246.196   n126       n126.roary.net
# 192.168.233.196   n126-data  n126-data.roary.net
```

---

## 12. Parallel Commands (xdsh / xdcp)

```bash
# Run command on all Dell compute nodes
xdsh dell "df -h"

# Unmount Lustre on a node range
xdsh n095-n102 "umount -t lustre /home /scratch"

# Check uptime across all HPE nodes
xdsh hpe "uptime"

# Copy a file to all nodes in a group
xdcp hpe /local/path/file.conf /etc/file.conf
```

---

## 13. Backup, Restore, and Cleanup

### Backup xCAT Database
```bash
dumpxCATdb -a -p "/home/share/scripts/xCAT/hpcxcat-dump_$(date +%m%d%y)"
cp -pr /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf_$(date +%m%d%y)
cp -pr /etc/hosts /etc/hosts_$(date +%m%d%y)
cp -pr /var/lib/dhcpd/dhcpd.leases /var/lib/dhcpd/dhcpd.leases_$(date +%m%d%y)
```

### Restore xCAT Database
```bash
restorexCATdb -a -p /home/share/scripts/xCAT/hpcxcat-dump-<mmddyy>
```

### Remove a Retired Node
Always backup first, then:
```bash
makedns -d <nodename>
makehosts
makedhcp -d <nodename>
rmdef -t node <nodename>
```

If DNS entries are not removed automatically:
```bash
systemctl restart named
```

---

## 14. Node Reference

> For current node IPs and OS assignments, always query the live xCAT database:
> ```bash
> lsdef -t node <nodename>                     # full node detail
> lsdef -t node -i ip,os,provmethod <nodename> # key fields only
> nodels <group>                               # list nodes in a group
> ```

### Hardware Management (IPMI)
Serial console: port 0, baud 115200.

```bash
rpower <node> stat
rpower <node> on|off|reset
rcons <node>            # exit: Ctrl-e c .
```

---

## 15. xCAT Additional Resources

- xCAT documentation: https://xcat-docs.readthedocs.io/
- man pages on hpcxcat: `man lsdef`, `man chdef`, `man nodeset`, `man xdsh`, `man makedns`
- xCAT mailing list: https://sourceforge.net/projects/xcat/
