```python
cat << 'EOF' | sudo tee /var/openflexure/extensions/microscope_extensions/self_aware_diagnostics/aging_telemetry.py
import os
import time
import json
import logging
import urllib.request
import numpy as np
import cv2


class LEDAgingLifecycleMonitor:
    """
    Objective 2B: Solid-State LED Aging Factor & Lifecycle Monitor
    Measures physical photon flux with exposure locked and logs clean audit reports.
    """
    def __init__(self, microscope_component, output_dir="/var/openflexure/data/diagnostics/objective_2"):
        self.microscope = microscope_component
        self.output_dir = output_dir
        os.makedirs(self.output_dir, exist_ok=True)

        self.DEFAULT_FACTORY_BASELINE = 243.10
        self.DEFAULT_WARNING_THRESHOLD = 85.0
        self.DEFAULT_REPLACE_THRESHOLD = 70.0
        self.DEFAULT_ROI_SIZE = 100

    def _grab_raw_live_frame(self):
        camera = getattr(self.microscope, "camera", None)
        if camera:
            try:
                if hasattr(camera, "lock_exposure"):
                    camera.lock_exposure()
                elif hasattr(camera, "exposure_mode"):
                    camera.exposure_mode = 'off'
            except Exception as e:
                logging.debug(f"[AgingMonitor] Could not lock exposure: {e}")

            if hasattr(camera, "grab_image"):
                try:
                    img_pil = camera.grab_image()
                    return cv2.cvtColor(np.array(img_pil), cv2.COLOR_RGB2GRAY)
                except Exception as e:
                    logging.debug(f"[AgingMonitor] cam.grab_image failed: {e}")

        try:
            req = urllib.request.Request("http://127.0.0.1:5000/api/v2/streams/snapshot")
            with urllib.request.urlopen(req, timeout=3.0) as response:
                img_bytes = response.read()
                img_np = np.frombuffer(img_bytes, np.uint8)
                frame_gray = cv2.imdecode(img_np, cv2.IMREAD_GRAYSCALE)
                if frame_gray is not None and frame_gray.size > 0:
                    return frame_gray
        except Exception as e:
            logging.debug(f"[AgingMonitor] Snapshot route failed: {e}")

        raise RuntimeError("Unable to acquire live frame from OpenFlexure camera subsystem.")

    def evaluate_aging_profile(self, factory_baseline=None, warning_threshold=None, replace_threshold=None, roi_size=None):
        I_0 = float(factory_baseline) if factory_baseline is not None else self.DEFAULT_FACTORY_BASELINE
        warn_th = float(warning_threshold) if warning_threshold is not None else self.DEFAULT_WARNING_THRESHOLD
        repl_th = float(replace_threshold) if replace_threshold is not None else self.DEFAULT_REPLACE_THRESHOLD
        r_size = int(roi_size) if roi_size is not None else self.DEFAULT_ROI_SIZE

        timestamp_str = time.strftime("%Y%m%d_%H%M%S")

        frame = self._grab_raw_live_frame()
        h, w = frame.shape
        xs = max(0, int((w - r_size) // 2))
        ys = max(0, int((h - r_size) // 2))

        roi_readings = []
        for _ in range(3):
            f = self._grab_raw_live_frame()
            roi_readings.append(float(np.mean(f[ys:ys + r_size, xs:xs + r_size])))
            time.sleep(0.05)

        current_flux = float(np.mean(roi_readings))

        soh_pct = (current_flux / I_0) * 100.0
        kappa_age = 1.0 - (current_flux / I_0)
        c_gain = (I_0 / current_flux) if current_flux > 0 else 1.0

        if soh_pct >= warn_th:
            status_text = "HEALTH_OPTIMAL"
            action_required = False
            ui_message = f"LED Health: {soh_pct:.1f}% (Optimal). Radiant flux operating nominally."
        elif soh_pct >= repl_th:
            status_text = "HEALTH_WARNING"
            action_required = False
            ui_message = f"LED Aging Detected: {soh_pct:.1f}% SOH. Schedule maintenance soon."
        else:
            status_text = "HEALTH_REPLACE_CRITICAL"
            action_required = True
            ui_message = f"L70 Critical EOL: {soh_pct:.1f}% SOH. Flux severely degraded (<70%). Replace LED unit."

        result = {
            "timestamp": timestamp_str,
            "overall_status": status_text,
            "state_of_health_pct": round(soh_pct, 2),
            "aging_factor_kappa": round(kappa_age, 4),
            "compensation_gain": round(c_gain, 4),
            "measured_steady_flux_dn": round(current_flux, 2),
            "factory_baseline_dn": round(I_0, 2),
            "warning_threshold_pct": round(warn_th, 1),
            "replace_threshold_pct": round(repl_th, 1),
            "action_required": action_required,
            "ui_notification": ui_message
        }

        latest_json = os.path.join(self.output_dir, "latest_led_health_metrics.json")
        with open(latest_json, "w") as f_out:
            json.dump(result, f_out, indent=4)

        report_path = os.path.join(self.output_dir, f"led_aging_report_{timestamp_str}.txt")
        latest_report = os.path.join(self.output_dir, "latest_led_aging_report.txt")
        
        
        report_text = (
            "OPENFLEXURE SELF-AWARE DIAGNOSTIC HEALTH REPORT\n"
            "LED AGING AND LIFECYCLE AUDIT\n\n"
            f"Timestamp: {time.strftime('%Y-%m-%d %H:%M:%S')}\n"
            f"Measured Steady Flux (I): {current_flux:.2f} DN\n"
            f"Factory Baseline Flux (I_0): {I_0:.2f} DN\n\n"
            f"State of Health (SOH): {soh_pct:.2f} %\n"
            f"Aging Degradation Factor: {kappa_age:.4f} ({kappa_age*100:.1f}% flux loss)\n"
            f"Photometric Gain Comp: {c_gain:.4f}x\n\n"
            f"DIAGNOSTIC STATUS: {status_text}\n"
            f"MAINTENANCE ACTION: {'REPLACE LED IMMEDIATELY' if action_required else 'CONTINUE OPERATION'}\n"
            f"NOTIFICATION: {ui_message}\n"
        )
        with open(report_path, "w") as f_rep:
            f_rep.write(report_text)
        with open(latest_report, "w") as f_rep:
            f_rep.write(report_text)

        return result
EOF
```
