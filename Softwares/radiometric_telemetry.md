```python
cat << 'EOF' | sudo tee /var/openflexure/extensions/microscope_extensions/self_aware_diagnostics/radiometric_telemetry.py
import os
import time
import json
import csv
import logging
import urllib.request
import numpy as np
import cv2

# Set non-interactive backend for server daemon
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt


class RadiometricStabilityMonitor:
    """
    Objective 2: Radiometric Stability & Thermal Characterization
    Auto-terminates once stability (Delta I_bar <= threshold) is achieved 
    for 3 consecutive sampling cycles, bypassing long fixed timeouts.
    """
    def __init__(self, microscope_component, stability_threshold_pct=0.1, output_dir="/var/openflexure/data/diagnostics/objective_2"):
        self.microscope = microscope_component
        self.stability_threshold_pct = float(stability_threshold_pct)
        self.output_dir = output_dir
        os.makedirs(self.output_dir, exist_ok=True)

    def _grab_grayscale_frame(self):
        """Pulls active frame directly from the running microscope snapshot stream."""
        try:
            req = urllib.request.Request("http://127.0.0.1:5000/api/v2/streams/snapshot")
            with urllib.request.urlopen(req, timeout=2.0) as response:
                img_bytes = response.read()
                img_np = np.frombuffer(img_bytes, np.uint8)
                frame_gray = cv2.imdecode(img_np, cv2.IMREAD_GRAYSCALE)
                if frame_gray is not None and frame_gray.size > 0:
                    return frame_gray
        except Exception as e:
            logging.debug(f"[RadiometricMonitor] Snapshot HTTP route failed: {e}")

        if hasattr(self.microscope, "camera") and hasattr(self.microscope.camera, "grab_image"):
            try:
                img_pil = self.microscope.camera.grab_image()
                return cv2.cvtColor(np.array(img_pil), cv2.COLOR_RGB2GRAY)
            except Exception as e:
                logging.debug(f"[RadiometricMonitor] cam.grab_image failed: {e}")

        raise RuntimeError("Unable to grab frame from OpenFlexure camera subsystem.")

    def run_radiometric_profile(self, duration_minutes=20, sampling_interval_sec=10, stability_threshold=None, consecutive_target=3, roi_size=100, generate_plots=True):
        """
        Executes dynamic radiometric characterization:
        - Monitors Delta I_bar (%/min)
        - Stops immediately upon 3 consecutive stable readings
        - Writes CSV logs, text audit report, and 600 DPI publication plots
        """
        active_th = float(stability_threshold) if stability_threshold is not None else self.stability_threshold_pct
        self.stability_threshold_pct = active_th

        timestamp_str = time.strftime("%Y%m%d_%H%M%S")
        sampling_interval = max(1, int(sampling_interval_sec))
        max_duration_sec = max(sampling_interval * 3, int(duration_minutes * 60))
        max_samples = int(max_duration_sec / sampling_interval)

        csv_path = os.path.join(self.output_dir, f"objective_2_radiometric_log_{timestamp_str}.csv")
        latest_csv_path = os.path.join(self.output_dir, "latest_radiometric_log.csv")

        times_min = []
        mean_intensities = []
        rates_of_change = []
        system_states = []

        consecutive_stable_count = 0
        early_terminated = False
        start_time = time.time()

        with open(csv_path, mode='w', newline='') as f_csv:
            writer = csv.writer(f_csv)
            writer.writerow(["Elapsed_Minutes", "Mean_Intensity_DN", "Rate_Of_Change_Pct_Per_Min", "System_State"])

            for idx in range(max_samples):
                current_time_sec = time.time() - start_time
                current_time_min = current_time_sec / 60.0

                frame = self._grab_grayscale_frame()
                h, w = frame.shape

                # Isolate central 100x100 ROI
                xs = max(0, int((w - roi_size) // 2))
                ys = max(0, int((h - roi_size) // 2))
                roi = frame[ys:ys + roi_size, xs:xs + roi_size]

                I_bar = float(np.mean(roi))
                mean_intensities.append(I_bar)
                times_min.append(current_time_min)

                if len(mean_intensities) > 1:
                    look_back = min(len(mean_intensities) - 1, 6)
                    I_past = mean_intensities[-1 - look_back]
                    time_diff_min = (look_back * sampling_interval) / 60.0

                    if I_past > 0 and time_diff_min > 0:
                        raw_pct_change = (abs(I_bar - I_past) / I_past) * 100.0
                        delta_I = float(raw_pct_change / time_diff_min)
                    else:
                        delta_I = 0.0
                else:
                    delta_I = 0.0

                rates_of_change.append(delta_I)

                if idx > 0 and delta_I <= active_th:
                    consecutive_stable_count += 1
                    state = f"Stable Phase (Lock {consecutive_stable_count}/{consecutive_target})"
                else:
                    consecutive_stable_count = 0
                    state = "System Unstable (Warming Up)"

                system_states.append(state)
                writer.writerow([round(current_time_min, 3), round(I_bar, 4), round(delta_I, 4), state])
                f_csv.flush()

                if consecutive_stable_count >= consecutive_target:
                    early_terminated = True
                    break

                time.sleep(sampling_interval)

        with open(csv_path, 'r') as src, open(latest_csv_path, 'w') as dst:
            dst.write(src.read())

        actual_duration_min = round((time.time() - start_time) / 60.0, 2)
        final_I_bar = mean_intensities[-1]
        final_delta_I = rates_of_change[-1]
        avg_drift_rate = float(np.mean(rates_of_change[1:])) if len(rates_of_change) > 1 else rates_of_change[-1]
        is_stable = early_terminated or (final_delta_I <= active_th)
        overall_status = "STABLE" if is_stable else "TIMEOUT_UNSTABLE"

        result = {
            "timestamp": timestamp_str,
            "duration_minutes": actual_duration_min,
            "max_timeout_minutes": float(duration_minutes),
            "sample_count": len(times_min),
            "consecutive_stable_locks": consecutive_stable_count,
            "sampling_interval_sec": sampling_interval,
            "final_mean_intensity_DN": round(final_I_bar, 4),
            "final_drift_rate_pct_per_min": round(final_delta_I, 4),
            "average_drift_rate_pct_per_min": round(avg_drift_rate, 4),
            "stability_threshold_pct_per_min": active_th,
            "is_stable": is_stable,
            "overall_status": overall_status
        }

        report_path = os.path.join(self.output_dir, f"radiometric_report_{timestamp_str}.txt")
        latest_report_path = os.path.join(self.output_dir, "latest_radiometric_report.txt")
        report_text = (
            "===========================================================================\n"
            " OPENFLEXURE SELF-AWARE DIAGNOSTIC HEALTH REPORT FOR RADIOMETRIC STABILITY \n"
            "===========================================================================\n"
            f"Timestamp            : {time.strftime('%Y-%m-%d %H:%M:%S')}\n"
            f"Convergence Time     : {actual_duration_min} min ({len(times_min)} samples @ {sampling_interval}s interval)\n"
            f"Termination Reason   : {'3x Consecutive Convergence Achieved' if early_terminated else 'Maximum Timeout Reached'}\n"
            "---------------------------------------------------------------------------\n"
            f"Mean ROI Intensity (I): {final_I_bar:.2f} DN (8-bit Gray Scale: 0-255)\n"
            f"Final Drift Rate dI  : {final_delta_I:.4f} % / min\n"
            f"Average Drift Rate   : {avg_drift_rate:.4f} % / min\n"
            f"Stability Limit (dI_th): {active_th:.4f} % / min\n"
            "---------------------------------------------------------------------------\n"
            f"DIAGNOSTIC STATUS    : {overall_status}\n"
            f"THERMAL INTEGRITY    : {'PASS (Thermodynamic Equilibrium Lock)' if is_stable else 'FAIL (Thermal Creep Active)'}\n"
            "===========================================================================\n"
        )
        with open(report_path, "w") as f:
            f.write(report_text)
        with open(latest_report_path, "w") as f:
            f.write(report_text)

        latest_json_path = os.path.join(self.output_dir, "latest_radiometric_metrics.json")
        with open(latest_json_path, "w") as f:
            json.dump(result, f, indent=4)

        if generate_plots:
            plt.rcParams.update({
                'font.family': 'sans-serif',
                'font.size': 11,
                'axes.labelsize': 12,
                'axes.titlesize': 13,
                'xtick.labelsize': 10,
                'ytick.labelsize': 10
            })

            fig1, ax1 = plt.subplots(figsize=(9, 5), dpi=300)
            ax1.plot(times_min, mean_intensities, 'g-o', markersize=4, linewidth=2, label=r'Mean Intensity ($\bar{I}$)')
            if early_terminated:
                ax1.axvline(times_min[-1], color='black', linestyle=':', label=f'Equilibrium Lock ({actual_duration_min}m)')
            ax1.set_title("Objective II: Radiometric Stability Characterization", fontsize=12, fontweight='bold', pad=12)
            ax1.set_xlabel("Elapsed Time (Minutes)", fontsize=11, fontweight='bold')
            ax1.set_ylabel("Mean Pixel Intensity (DN)", fontsize=11, fontweight='bold')
            ax1.legend(loc="upper right", frameon=True, edgecolor='gray')
            ax1.grid(True, linestyle='--', alpha=0.7)
            fig1.tight_layout()

            plot1_path = os.path.join(self.output_dir, f"intensity_vs_time_{timestamp_str}_600DPI.png")
            latest_plot1 = os.path.join(self.output_dir, "latest_intensity_vs_time.png")
            fig1.savefig(plot1_path, dpi=600, bbox_inches='tight')
            fig1.savefig(latest_plot1, dpi=600, bbox_inches='tight')
            plt.close(fig1)

            fig2, ax2 = plt.subplots(figsize=(9, 5), dpi=300)
            ax2.plot(times_min, rates_of_change, 'b-o', markersize=4, linewidth=2, label=r'Rate of Change ($\Delta\bar{I}$)')
            ax2.axhline(y=active_th, color='r', linestyle='--', linewidth=1.5, label=f'Stability Threshold ({active_th}%/min)')
            if early_terminated:
                ax2.axvline(times_min[-1], color='black', linestyle=':', label=f'3x Target Reached ({actual_duration_min}m)')
            ax2.set_title("Objective II: Elapsed Temporal Illumination Drift Rate", fontsize=12, fontweight='bold', pad=12)
            ax2.set_xlabel("Elapsed Time (Minutes)", fontsize=11, fontweight='bold')
            ax2.set_ylabel("Variation per Minute (%)", fontsize=11, fontweight='bold')
            ax2.legend(loc="upper right", frameon=True, edgecolor='gray')
            ax2.grid(True, linestyle='--', alpha=0.7)
            fig2.tight_layout()

            plot2_path = os.path.join(self.output_dir, f"stability_rate_vs_time_{timestamp_str}_600DPI.png")
            latest_plot2 = os.path.join(self.output_dir, "latest_stability_rate_vs_time.png")
            fig2.savefig(plot2_path, dpi=600, bbox_inches='tight')
            fig2.savefig(latest_plot2, dpi=600, bbox_inches='tight')
            plt.close(fig2)

        return result
EOF
```
