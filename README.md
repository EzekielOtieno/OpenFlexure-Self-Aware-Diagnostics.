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
