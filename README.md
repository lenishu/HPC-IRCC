# HPC Roary — IRCC (Documentation & tooling)

A collection of operational documentation, onboarding guides, and small utilities for managing the Roary HPC cluster (FIU). This repository contains step-by-step xCAT administration guides, an intern onboarding guide, provisioning templates, presentation slides, and helper scripts used by the cluster operations team.

What this is
- Operational knowledgebase and lightweight tooling for the Roary HPC cluster (xCAT-based provisioning, SLURM, Lustre, IPMI).
- Intended audience: HPC system administrators, new technicians/interns, and anyone onboarding onto the Roary cluster operations team.

Primary technologies & environment
- Cluster management: xCAT (provisioning), SLURM (scheduler), Lustre (parallel filesystem)
- OS images in use: AlmaLinux 9.x (diskless netboot and diskful install images)
- Scripts: Python 3 (presentation patching utilities)
- Documentation format: Markdown + PowerPoint slides

Repository contents (top-level)
- CLAUDE.md                      — Project behavioral guidelines for contributors
- HPC_Roary_Intern_Guide.md      — Onboarding guide for new technicians (hands-on)
- HPC_xCAT_Roary_Guide.md        — Detailed xCAT administration guide and references
- HPC_gpu_node_Template.md       — Masked provisioning template for GPU nodes
- nodecreation.txt               — Node provisioning log / notes
- "HPC - xCAT Overview - Roary.pptx" (and copies) — Presentation used for briefings
- update_slides.py               — Script to update the slides (CentOS→AlmaLinux etc.)
- patch_slides.py                — Targeted fixes for the presentation

Quick start
1. Browse the documentation locally:
   - Open HPC_Roary_Intern_Guide.md for a practical, hands-on introduction.
   - Open HPC_xCAT_Roary_Guide.md for the authoritative xCAT admin reference.

2. If you need to update the slides (requires Python 3 and python-pptx):

```bash
# (Optional) create a virtualenv
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt 2>/dev/null || pip install python-pptx lxml

# Run the update script (reads/outputs files in the repo root)
python3 update_slides.py

# Run the targeted patch script (edits the same pptx file in-place)
python3 patch_slides.py
```

Notes
- update_slides.py expects the input PowerPoint to be named `HPC - xCAT Overview.pptx` and will write `HPC - xCAT Overview - Roary.pptx`.
- patch_slides.py edits `HPC - xCAT Overview - Roary.pptx` in-place to apply further targeted corrections.

Contributing and changes
- This repo is primarily operational documentation. When submitting edits:
  - Follow the principle of surgical changes: edit only the portions you need to change.
  - For doc updates: prefer small, focused commits that change only the affected Markdown or slide content.
- If you add automation that modifies live cluster config, include clear success criteria and a rollback plan.

Missing pieces / recommendations
- No LICENSE file is present. Add one if you intend to share or reuse these materials outside the team.
- Consider adding a requirements.txt for the Python scripts (python-pptx, lxml) so installs are repeatable.

Contact
- Repository owner: lenishu (GitHub)
- For cluster-specific questions, refer to the internal Roary contact list (not included here).

