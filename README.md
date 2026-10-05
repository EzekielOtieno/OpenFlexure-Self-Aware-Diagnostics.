# Embedded Self-Awareness for the OpenFlexure Microscope: Real-Time Metrological Assurance in Distributed Telepathology

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Target Journal](https://img.shields.io/badge/HardwareX-Elsevier-orange.svg)](https://www.journals.elsevier.com/hardwarex)
[![Platform](https://img.shields.io/badge/Platform-Raspberry%20Pi%204B-red.svg)](https://www.raspberrypi.com/)
[![Quality Standard](https://img.shields.io/badge/Compliance-ISO%2014971%20%7C%20ISO%2013485-green.svg)](#regulatory-alignment)

**Authors:** Ezekiel Otieno¹'*, Dickson Mwenda Kinyua², Daniel Maitethia Memeu¹  
¹ *Department of Physical Sciences (Physics), Meru University of Science and Technology, Meru, Kenya*  
² *Department of Pure and Applied Sciences, Kirinyaga University, Kerugoya, Kenya*  
*\*Corresponding Author*

---

## Overview

The democratization of digital pathology via low-cost, open-source robotic platforms such as the **OpenFlexure Microscope (v7.0.0-beta)** significantly lowers capital barriers for decentralized telepathology. However, these systems traditionally operate as passive instruments, leaving them vulnerable to undetected hardware degradation including optomechanical spatial drift, thermal settling instability, and radiometric LED decay. 

This repository provides **Self-Aware Diagnostics**, an embedded, closed-loop server extension that transforms the OpenFlexure Microscope into an active, self-monitoring instrument running entirely on native single-board edge hardware (Raspberry Pi 4B).

---

## Key Experimental Benchmarks

Validated across dual physical builds with controlled fault-injection routines:

* **Optomechanical Noise Floor:** Baseline spatial drift noise floor of $\mu_D = 1.67 \times 10^{-3}\text{ px}$, maintaining stability two orders of magnitude below the pre-set fault threshold.
* **Thermodynamic Settling Lock:** Reliable convergence to thermodynamic equilibrium within 20 minutes across cold-start cycles.
* **Physical State-of-Health (SOH):** Accurate characterization of emitter degradation ($R^2 = 0.9109$ exponential fit), successfully classifying a 58.2% lumen depreciation as an actionable critical fault (41.8% SOH) prior to clinical sample acquisition.

---

## Deterministic Three-Stage State Machine
1. **Dual-Gate Alignment Lock (`alignment_telemetry.py`):** Sub-pixel tracking of illumination centroid coordinates to identify optomechanical creep, vibration, and mechanical relaxation.
2. **Radiometric Thermal Lock (`radiometric_telemetry.py`):** Dynamic monitoring of sensor flux and thermal expansion until equilibrium conditions are achieved.
3. **Continuous SOH Classification (`aging_telemetry.py`):** Evaluates chronic LED emission decay, updating baseline calibration logs and halting acquisition if flux drops below diagnostic limits.

---

## Regulatory & Quality Standard Alignment

* **ISO 14971 (Clauses 7.1–7.3 - Risk Control):** Proactively prevents clinical false positives and negatives caused by underexposure, illumination inhomogeneity, and lateral mechanical slip during automated tile scanning.
* **ISO 13485 (Clause 7.6 - Control of Monitoring and Measuring Devices):** Automates the continuous verification, status adjustment, and audit archiving of optoelectronic metrological integrity without requiring external laboratory equipment.

---

## Repository Architecture

```text
OpenFlexure-Self-Aware-Diagnostics/
├── software/                       # Embedded diagnostic logic and server hooks
│   ├── __init__.py
│   ├── alignment_telemetry.py      # Dual-gate optomechanical tracking routine
│   ├── radiometric_telemetry.py    # Radiometric settling & thermal lock
│   ├── aging_telemetry.py          # Emitter degradation & SOH classification
│   └── requirements.txt            # Python environment dependencies
├──
├── data/                           # Experimental validation & repeatability logs
│   ├── radiometric_full_20min_20260828.csv
│   └── repeatability_dataset_illumination.csv
├                       
│   └── Bill_of_Materials.csv       # Standardized BOM (dual builds + workstation)
├── LICENSE                         # GNU General Public License v3.0
└── README.md                       # Master documentation file
---

## Installation & Deployment

### 1. Prerequisites
* **Operating System:** Raspberry Pi OS (Debian 64-bit / 32-bit)
* **Python Runtime:** Python >= 3.8
* **Microscope Stack:** Operational `openflexure-microscope-server`

### 2. Clone and Setup
```bash
git clone [https://github.com/EzekielOtieno/OpenFlexure-Self-Aware-Diagnostics.git](https://github.com/EzekielOtieno/OpenFlexure-Self-Aware-Diagnostics.git)
cd OpenFlexure-Self-Aware-Diagnostics
pip install -r software/requirements.txt
sudo cp -r software /var/openflexure/extensions/microscope_extensions/self_aware_diagnostics
sudo systemctl restart openflexure-microscope-server
# Phase 1: Dual-gate optomechanical centroid verification
python3 -m software.alignment_telemetry

# Phase 2: Radiometric settling & thermal lock
python3 -m software.radiometric_telemetry

# Phase 3: Physical LED aging and State-of-Health (SOH) evaluation
python3 -m software.aging_telemetry

---

### Action on GitHub
1. Paste the block directly at the bottom of the editor.
2. Scroll to the bottom of the page.
3. Enter a commit message (e.g., `Complete README with setup, execution, and deployment steps`).
4. Click the green **Commit changes** button.
