```python
cat << 'EOF' | sudo tee /var/openflexure/extensions/microscope_extensions/self_aware_diagnostics/__init__.py
import os
import json
import logging
from flask import abort, jsonify, request
from labthings import fields, find_component
from labthings.extensions import BaseExtension
from labthings.views import ActionView, PropertyView
from openflexure_microscope.api.utilities.gui import build_gui

from .alignment_telemetry import OptomechanicalAlignmentMonitor
from .radiometric_telemetry import RadiometricStabilityMonitor
from .aging_telemetry import LEDAgingLifecycleMonitor

_alignment_monitor = None
_radiometric_monitor = None
_aging_monitor = None

OUTPUT_DIR_OBJ1 = "/var/openflexure/data/diagnostics/objective_1"
OUTPUT_DIR_OBJ2 = "/var/openflexure/data/diagnostics/objective_2"

CURRENT_DRIFT_THRESHOLD_PX = 0.2
CURRENT_MIN_INTENSITY = 200
CURRENT_STABILITY_RATE_THRESHOLD = 0.1

def get_alignment_monitor():
    global _alignment_monitor
    microscope = find_component("org.openflexure.microscope")
    if _alignment_monitor is None:
        _alignment_monitor = OptomechanicalAlignmentMonitor(
            microscope,
            drift_threshold_px=CURRENT_DRIFT_THRESHOLD_PX,
            min_intensity_threshold=CURRENT_MIN_INTENSITY,
            output_dir=OUTPUT_DIR_OBJ1
        )
    return _alignment_monitor

def get_radiometric_monitor():
    global _radiometric_monitor
    microscope = find_component("org.openflexure.microscope")
    if _radiometric_monitor is None:
        _radiometric_monitor = RadiometricStabilityMonitor(
            microscope,
            stability_threshold_pct=CURRENT_STABILITY_RATE_THRESHOLD,
            output_dir=OUTPUT_DIR_OBJ2
        )
    return _radiometric_monitor

def get_aging_monitor():
    global _aging_monitor
    microscope = find_component("org.openflexure.microscope")
    if _aging_monitor is None:
        _aging_monitor = LEDAgingLifecycleMonitor(
            microscope,
            output_dir=OUTPUT_DIR_OBJ2
        )
    return _aging_monitor


# =========================================================================
# OBJECTIVE 1: ILLUMINATION ALIGNMENT VIEWS
# =========================================================================
class CheckAlignmentView(ActionView):
    args = {
        "duration": fields.Number(missing=5, example=5, description="Monitoring Duration (s)"),
        "min_intensity": fields.Number(missing=200, example=200, description="Min Intensity Gate (DN)"),
        "drift_threshold": fields.Number(missing=0.2, example=0.2, description="Drift Limit D_th (px)")
    }

    def post(self, args=None):
        global CURRENT_DRIFT_THRESHOLD_PX, CURRENT_MIN_INTENSITY
        args = args or {}
        duration = args.get("duration", 5)
        drift_th = args.get("drift_threshold", CURRENT_DRIFT_THRESHOLD_PX)
        min_i = args.get("min_intensity", CURRENT_MIN_INTENSITY)

        CURRENT_DRIFT_THRESHOLD_PX = float(drift_th)
        CURRENT_MIN_INTENSITY = int(min_i)

        monitor = get_alignment_monitor()
        res = monitor.run_alignment_profile(
            duration_seconds=int(duration),
            drift_threshold=CURRENT_DRIFT_THRESHOLD_PX,
            min_intensity=CURRENT_MIN_INTENSITY,
            generate_plot=True
        )

        status = res.get("overall_status")
        drift = res.get("euclidean_drift_D_px")
        peak = res.get("peak_intensity")
        msg = (
            f"ALIGNMENT RESULT: {status}\n\n"
            f"Drift: {drift} px (Limit: <= {CURRENT_DRIFT_THRESHOLD_PX} px)\n"
            f"Peak Flux: {peak} DN (Gate: >= {CURRENT_MIN_INTENSITY} DN)\n"
        )
        abort(400, description=msg)


class AlignmentReportProperty(PropertyView):
    def get(self):
        report_path = os.path.join(OUTPUT_DIR_OBJ1, "latest_alignment_metrics.json")
        if os.path.exists(report_path):
            try:
                with open(report_path, "r") as f:
                    return json.load(f)
            except Exception:
                pass
        return {"overall_status": "NOT_TESTED", "euclidean_drift_D_px": 0.0}


class DownloadAlignmentReportView(ActionView):
    args = {}
    def post(self, args=None):
        txt_path = os.path.join(OUTPUT_DIR_OBJ1, "latest_alignment_report.txt")
        if os.path.exists(txt_path):
            with open(txt_path, "r") as f:
                abort(400, description=f"\n{f.read()}")
        abort(400, description="No Objective 1 diagnostic report found on disk.")


# =========================================================================
# OBJECTIVE 2A: RADIOMETRIC STABILITY VIEWS
# =========================================================================
class RadiometricStabilityView(ActionView):
    args = {
        "max_timeout_minutes": fields.Number(missing=20, example=20, description="Max Timeout Ceiling (min)"),
        "sampling_interval_sec": fields.Number(missing=10, example=10, description="Sampling Interval (s)"),
        "stability_threshold": fields.Number(missing=0.1, example=0.1, description="Stability Rate (%/min)")
    }

    def post(self, args=None):
        global CURRENT_STABILITY_RATE_THRESHOLD
        args = args or {}
        timeout_min = args.get("max_timeout_minutes", 20)
        sampling_sec = args.get("sampling_interval_sec", 10)
        stab_th = args.get("stability_threshold", CURRENT_STABILITY_RATE_THRESHOLD)

        CURRENT_STABILITY_RATE_THRESHOLD = float(stab_th)

        monitor = get_radiometric_monitor()
        res = monitor.run_radiometric_profile(
            duration_minutes=float(timeout_min),
            sampling_interval_sec=int(sampling_sec),
            stability_threshold=CURRENT_STABILITY_RATE_THRESHOLD,
            consecutive_target=3,
            generate_plots=True
        )

        status = res.get("overall_status")
        flux = res.get("final_mean_intensity_DN")
        t_lock = res.get("duration_minutes")
        rate = res.get("final_drift_rate_pct_per_min")
        msg = (
            f"RADIOMETRIC STABILITY RESULT: {status}\n\n"
            f"Steady Flux: {flux} DN\n"
            f"Convergence Time: {t_lock} min\n"
            f"Final Drift Rate: {rate} %/min\n"
        )
        abort(400, description=msg)


class RadiometricReportProperty(PropertyView):
    def get(self):
        report_path = os.path.join(OUTPUT_DIR_OBJ2, "latest_radiometric_metrics.json")
        if os.path.exists(report_path):
            try:
                with open(report_path, "r") as f:
                    return json.load(f)
            except Exception:
                pass
        return {"overall_status": "NOT_TESTED", "final_drift_rate_pct_per_min": 0.0}


class DownloadRadiometricReportView(ActionView):
    args = {}
    def post(self, args=None):
        txt_path = os.path.join(OUTPUT_DIR_OBJ2, "latest_radiometric_report.txt")
        if os.path.exists(txt_path):
            with open(txt_path, "r") as f:
                abort(400, description=f"\n{f.read()}")
        abort(400, description="No Objective 2 diagnostic report found on disk.")


# =========================================================================
# OBJECTIVE 2B: CLINICAL ON-SCREEN ALERT DIALOG
# =========================================================================
class EvaluateLEDAgingView(ActionView):
    args = {
        "factory_baseline": fields.Number(missing=243.10, example=243.10, description="Pristine Baseline (DN)"),
        "warning_threshold": fields.Number(missing=85.0, example=85.0, description="Warning Threshold SOH (%)"),
        "replace_threshold": fields.Number(missing=70.0, example=70.0, description="L70 Limit SOH (%)")
    }

    def post(self, args=None):
        args = args or {}
        base_flux = float(args.get("factory_baseline") or 243.10)
        warn_th = float(args.get("warning_threshold") or 85.0)
        repl_th = float(args.get("replace_threshold") or 70.0)

        monitor = get_aging_monitor()
        res = monitor.evaluate_aging_profile(
            factory_baseline=base_flux,
            warning_threshold=warn_th,
            replace_threshold=repl_th
        )

        status = res.get("overall_status")
        soh = res.get("state_of_health_pct", 0.0)
        flux = res.get("measured_steady_flux_dn", 0.0)
        kappa = res.get("aging_factor_kappa", 0.0)

        if status == "HEALTH_OPTIMAL":
            status_desc = "OPTIMAL EMISSION"
            action = "NO ACTION REQUIRED. Illumination output is optimal for diagnostic scanning."
        elif status == "HEALTH_WARNING":
            status_desc = "WARNING (AGED EMITTER)"
            action = "SCHEDULE LED REPLACEMENT. Emitter has degraded by >15%. Ongoing scans permitted."
        else:
            status_desc = "REPLACE IMMEDIATELY"
            action = "REPLACE LED. Radiant flux breached L70 standard (<70%). Diagnostic scanning locked."

        screen_dialog = (
            "HOSPITAL CLINICAL DIAGNOSTIC REPORT\n\n"
            f"DIAGNOSTIC STATUS: {status_desc}\n"
            f"STATE OF HEALTH: {soh:.1f} %\n"
            f"RADIANT FLUX: {flux:.2f} DN (Baseline: {base_flux:.1f} DN)\n"
            f"LUMEN LOSS: {kappa * 100:.1f} %\n\n"
            f"RECOMMENDED ACTION:\n{action}\n"
        )
        abort(400, description=screen_dialog)


class ReadAuditReportOnScreenView(ActionView):
    args = {}

    def post(self, args=None):
        txt_path = os.path.join(OUTPUT_DIR_OBJ2, "latest_led_aging_report.txt")
        if os.path.exists(txt_path):
            with open(txt_path, "r") as f:
                abort(400, description=f"\n{f.read()}")

        monitor = get_aging_monitor()
        monitor.evaluate_aging_profile()
        if os.path.exists(txt_path):
            with open(txt_path, "r") as f:
                abort(400, description=f"\n{f.read()}")

        abort(400, description="No clinical audit report found.")


class LEDAgingReportProperty(PropertyView):
    def get(self):
        report_path = os.path.join(OUTPUT_DIR_OBJ2, "latest_led_health_metrics.json")
        if os.path.exists(report_path):
            try:
                with open(report_path, "r") as f:
                    return json.load(f)
            except Exception:
                pass
        return {"overall_status": "NOT_TESTED", "state_of_health_pct": 0.0}


# =========================================================================
# OPENFLEXURE eV GUI CONFIGURATION
# =========================================================================
extension_gui = {
    "icon": "local_hospital",
    "forms": [
        {
            "name": "1. Illumination Alignment",
            "route": "/check-alignment",
            "isTask": True,
            "isCollapsible": True,
            "submitLabel": "Run Alignment Check",
            "schema": [
                {"fieldType": "numberInput", "name": "duration", "label": "Duration (s)", "default": 5},
                {"fieldType": "numberInput", "name": "min_intensity", "label": "Min Intensity (DN)", "default": 200},
                {"fieldType": "numberInput", "name": "drift_threshold", "label": "Drift Tolerance (px)", "default": 0.2}
            ]
        },
        {
            "name": "2. Radiometric Thermal Stability",
            "route": "/radiometric-stability",
            "isTask": True,
            "isCollapsible": True,
            "submitLabel": "Run Thermal Stability Lock",
            "schema": [
                {"fieldType": "numberInput", "name": "max_timeout_minutes", "label": "Timeout (min)", "default": 20.0},
                {"fieldType": "numberInput", "name": "sampling_interval_sec", "label": "Sampling Interval (s)", "default": 10},
                {"fieldType": "numberInput", "name": "stability_threshold", "label": "Stability Threshold (%/min)", "default": 0.1}
            ]
        },
        {
            "name": "3. Solid-State LED Health & SOH Monitor",
            "route": "/evaluate-led-aging",
            "isTask": True,
            "isCollapsible": False,
            "submitLabel": "Run Physical LED Health Check",
            "schema": [
                {"fieldType": "numberInput", "name": "factory_baseline", "label": "Pristine Baseline (DN)", "default": 243.10},
                {"fieldType": "numberInput", "name": "warning_threshold", "label": "Warning Threshold SOH (%)", "default": 85.0},
                {"fieldType": "numberInput", "name": "replace_threshold", "label": "L70 Replacement SOH (%)", "default": 70.0}
            ]
        },
        {
            "name": "4. Display Full Clinical Report on Screen",
            "route": "/display-clinical-report",
            "isTask": True,
            "isCollapsible": False,
            "submitLabel": "Open Full Audit Report Dialog",
            "schema": []
        }
    ]
}

# Instantiate BaseExtension
self_aware_diagnostics = BaseExtension(
    "org.openflexure.self_aware_diagnostics",
    version="1.0.0"
)

# Objective 1 Routes
self_aware_diagnostics.add_view(CheckAlignmentView, "/check-alignment")
self_aware_diagnostics.add_view(DownloadAlignmentReportView, "/fetch-alignment-report")
self_aware_diagnostics.add_view(AlignmentReportProperty, "/report/alignment")
self_aware_diagnostics.add_view(AlignmentReportProperty, "/report")

# Objective 2A Routes
self_aware_diagnostics.add_view(RadiometricStabilityView, "/radiometric-stability")
self_aware_diagnostics.add_view(DownloadRadiometricReportView, "/fetch-radiometric-report")
self_aware_diagnostics.add_view(RadiometricReportProperty, "/report/radiometric")

# Objective 2B Clinical Routes
self_aware_diagnostics.add_view(EvaluateLEDAgingView, "/evaluate-led-aging")
self_aware_diagnostics.add_view(ReadAuditReportOnScreenView, "/display-clinical-report")
self_aware_diagnostics.add_view(LEDAgingReportProperty, "/report/led-aging")

# Attach Web GUI
self_aware_diagnostics.add_meta("gui", build_gui(extension_gui, self_aware_diagnostics))
EOF
```
