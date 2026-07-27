# 📓 HPC Lab Notebook: xCAT Node Provisioning Template (Masked)

## Purpose

This document is a reusable provisioning template for adding new compute nodes into an xCAT-managed HPC cluster.

It is designed to:

- Prevent exposure of full IP/MAC schema
- Standardize provisioning workflow
- Allow fast node onboarding
- Serve as an operational checklist for HPC deployment

---

# 🧭 Section 1: Cluster Network Architecture

HPC clusters isolate traffic into multiple network planes.

## 1. Out-of-Band Management Network

- Purpose: IPMI / BMC / hardware control
- Example subnet: `192.168.254.xx`
- Example node: `rv100-xx`

---

## 2. In-Band Provisioning Network

- Purpose: PXE boot + OS install
- Example subnet: `192.168.246.xx`
- Example node: `gpu-v100-xx`

Interface:

ens3f0

---

## 3. High-Speed Data Network

- Purpose: MPI / Lustre / cluster traffic
- Example subnet: `192.168.233.xx`

Interface:

ens3f0.418

---

# 📋 Section 2: Pre-Provisioning Requirements

## Node Worksheet (Fill Before Provisioning)

| Field | Example (Masked) |
|------|----------------|
| Compute Hostname | gpu-v100-xx |
| BMC Hostname | rv100-xx |
| Compute IP | 192.168.246.xx |
| BMC IP | 192.168.254.xx |
| Data IP | 192.168.233.xx |
| MAC Address | 0C:42:A1:C7:9A:xx |
| Hardware Model | Dell:R740 |
| Group | raptor |
| Install NIC | ens3f0 |
| Data NIC | ens3f0.418 |
| OS Image | alma9.7-x86_64-netbootIB-cuda-compute |

---

## Minimum Checklist

□ Hostname assigned  
□ BMC hostname assigned  
□ Compute IP assigned  
□ BMC IP assigned  
□ Data IP assigned  
□ MAC recorded  
□ Hardware verified  
□ Group selected  
□ OS image selected  
□ BMC reachable  

---

# 🔄 Section 3: Variable Fields (Change Per Node)

- Hostname → gpu-v100-xx  
- IP addresses → 192.168.xxx.xx  
- MAC address → 0C:42:A1:C7:9A:xx  
- BMC mapping → rv100-xx  
- Cluster group → raptor / compute / login  

---

# ⚙️ Section 4: Verify OS Images

lsos

Example:
alma9.7-x86_64-netbootIB-cuda-compute

---

# 🏗️ Section 5: Create Node Objects

lsdef gpu-v100-xx  
lsdef rv100-xx  

nodeadd gpu-v100-xx groups=raptor  
nodeadd rv100-xx groups=mgt  

---

# 🛠️ Section 6: Configure Nodes

chdef -t node rv100-xx ip=192.168.254.xx mgt=ipmi hostnames=rv100-xx mac=^  

chdef -t node gpu-v100-xx ip=192.168.246.xx bmc=rv100-xx mgt=ipmi netboot=xnba installnic=ens3f0 mac=0C:42:A1:C7:9A:xx provmethod=alma9.7-x86_64-netbootIB-cuda-compute  

---

# 📡 Section 7: Rebuild Services

makehosts gpu-v100-xx,rv100-xx  
makedns gpu-v100-xx  
makedhcp gpu-v100-xx  

---

# 🚀 Section 8: Deploy

nodeset gpu-v100-xx osimage=alma9.7-x86_64-netbootIB-cuda-compute  
rpower gpu-v100-xx reset  
rcons gpu-v100-xx  

---

# 🎯 Final Reveal

Compute IP: 192.168.246.210  
BMC IP: 192.168.254.210  
Data IP: 192.168.233.210  
MAC: 0C:42:A1:C7:9A:E4  
