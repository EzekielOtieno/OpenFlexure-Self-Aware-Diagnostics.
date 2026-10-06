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

1. **Stage 1: Dual-Gate Alignment Lock (`software/alignment_telemetry.py`):** Monitors sub-pixel illumination centroid coordinates to detect optomechanical creep, vibration, and mechanical relaxation.
2. **Stage 2: Radiometric Thermal Lock (`software/radiometric_telemetry.py`):** Continuously samples sensor flux and calculates rates of change until thermodynamic equilibrium ($t_{\text{lock}}$) is achieved.
3. **Stage 3: Continuous SOH Classification (`software/aging_telemetry.py`):** Evaluates chronic LED emission decay against calibrated baseline levels, logging compliance and alerting before degraded illumination affects diagnostics.

---

## Regulatory & Quality Standard Alignment

* **ISO 14971 (Clauses 7.1–7.3 - Risk Control):** Proactively prevents clinical false positives and false negatives caused by underexposure, illumination inhomogeneity, and lateral mechanical slip during automated slide scanning.
* **ISO 13485 (Clause 7.6 - Control of Monitoring and Measuring Devices):** Automates the continuous verification, status adjustment, and audit archiving of optoelectronic metrological integrity without requiring external laboratory equipment.

---

## Repository Architecture

OpenFlexure-Self-Aware-Diagnostics/
├── software/
│   ├── __init__.py
│   ├── alignment_telemetry.py
│   ├── radiometric_telemetry.py
│   ├── aging_telemetry.py
│   └── requirements.txt
├── data/
│   ├── radiometric_full_20min_20260828.csv
│   └── repeatability_dataset_illumination.csv
├── images/
│   └── (state machine diagrams and GUI screenshots)
├── docs/
│   └── Bill_of_Materials.csv
├── LICENSE                         (GPLv3)
└── README.md

2. Clone and Setup
git clone [https://github.com/EzekielOtieno/OpenFlexure-Self-Aware-Diagnostics.git](https://github.com/EzekielOtieno/OpenFlexure-Self-Aware-Diagnostics.git)
cd OpenFlexure-Self-Aware-Diagnostics
pip install -r software/requirements.txt

3. Deploy Extension to OpenFlexure Server
To install the package into the microscope server daemon:
sudo cp -r software /var/openflexure/extensions/microscope_extensions/self_aware_diagnostics
sudo systemctl restart openflexure-microscope-server


Execution & Usage
Standalone Metrological Verification
You can execute diagnostic checks directly from the command line:

# Phase 1: Dual-gate optomechanical centroid verification
python3 -m software.alignment_telemetry

# Phase 2: Radiometric settling & thermal lock
python3 -m software.radiometric_telemetry

# Phase 3: Physical LED aging and State-of-Health (SOH) evaluation
python3 -m software.aging_telemetry

Web GUI Access
Once deployed, open the OpenFlexure Web Client or connect through OpenFlexure Connect. Navigate to Extensions > Self-Aware Diagnostics to run alignment checks, observe real-time radiometric curves, or export ISO compliance audit logs.

Bill of Materials Summary
A complete, market-verified Bill of Materials specifying both experimental builds and the benchtop workstation display (Vitron 32") is maintained at docs/Bill_of_Materials.csv.

License
This software and diagnostic framework is released under the GNU General Public License v3.0 (GPLv3). See LICENSE for complete terms.
