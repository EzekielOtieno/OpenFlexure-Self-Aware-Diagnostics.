```python
cat << 'EOF' | sudo tee /var/openflexure/extensions/microscope_extensions/self_aware_diagnostics/alignment_telemetry.py
import os
import time
import json
import logging
import urllib.request
import numpy as np
import cv2

# Set non-interactive backend for server daemon
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt


class OptomechanicalAlignmentMonitor:
    """
    Objective 1 Control Framework:
    Dynamically receives intensity and drift thresholds, measures Luminance Centroid (x_bar, y_bar)
    relative to Calibrated Baseline (Tx, Ty), and updates all outputs and audit reports.
    """
    def __init__(self, microscope_component, drift_threshold_px=0.2, min_intensity_threshold=200, output_dir="/var/openflexure/data/diagnostics/objective_1"):
        self.microscope = microscope_component
        self.drift_threshold_px = float(drift_threshold_px)
        self.min_intensity_threshold = int(min_intensity_threshold)
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
            logging.debug(f"[AlignmentMonitor] Snapshot route failed: {e}")

        if hasattr(self.microscope, "camera") and hasattr(self.microscope.camera, "grab_image"):
            try:
                img_pil = self.microscope.camera.grab_image()
                return cv2.cvtColor(np.array(img_pil), cv2.COLOR_RGB2GRAY)
            except Exception as e:
                logging.debug(f"[AlignmentMonitor] cam.grab_image failed: {e}")

        raise RuntimeError("Unable to grab frame from OpenFlexure camera subsystem.")

    def run_alignment_profile(self, duration_seconds=5, roi_size=120, drift_threshold=None, min_intensity=None, generate_plot=True):
        """
        Executes diagnostic evaluation using dynamic thresholds:
        - Evaluates peak intensity vs. min_intensity threshold
        - Evaluates Euclidean drift D vs. drift_threshold
        - Writes synchronized text reports, JSON metrics, and publication plots.
        """
        active_drift_th = float(drift_threshold) if drift_threshold is not None else self.drift_threshold_px
        active_min_int = int(min_intensity) if min_intensity is not None else self.min_intensity_threshold
        
        self.drift_threshold_px = active_drift_th
        self.min_intensity_threshold = active_min_int

        timestamp_str = time.strftime("%Y%m%d_%H%M%S")

        # Step 1: Baseline Acquisition & Calibration (t = 0)
        initial_frame = self._grab_grayscale_frame()
        h, w = initial_frame.shape
        xs, ys = max(0, int((w - roi_size) // 2)), max(0, int((h - roi_size) // 2))

        baseline_targets = []
        for _ in range(5):
            cal_frame = self._grab_grayscale_frame()
            roi = cal_frame[ys:ys + roi_size, xs:xs + roi_size]
            smoothed_roi = cv2.GaussianBlur(roi, (21, 21), 0)
            M = cv2.moments(smoothed_roi)
            if M["m00"] > 0:
                baseline_targets.append(((M["m10"] / M["m00"]) + xs, (M["m01"] / M["m00"]) + ys))
            time.sleep(0.05)

        if baseline_targets:
            Tx, Ty = np.mean(baseline_targets, axis=0)
        else:
            Tx, Ty = float(w / 2.0), float(h / 2.0)

        Tx, Ty = float(Tx), float(Ty)
        target_y_idx = int(np.clip(round(Ty), 0, h - 1))

        # Step 2: Temporal Sampling Loop
        start_time = time.time()
        measured_centroids_x = []
        measured_centroids_y = []
        peak_intensities = []
        final_frame = initial_frame.copy()
        sample_count = 0

        while (time.time() - start_time) < max(1, duration_seconds):
            frame = self._grab_grayscale_frame()
            roi = frame[ys:ys + roi_size, xs:xs + roi_size]
            
            peak_intensities.append(int(np.max(roi)))
            smoothed_roi = cv2.GaussianBlur(roi, (21, 21), 0)
            M = cv2.moments(smoothed_roi)

            if M["m00"] > 0:
                cx = (M["m10"] / M["m00"]) + xs
                cy = (M["m01"] / M["m00"]) + ys
            else:
                cx, cy = Tx, Ty

            measured_centroids_x.append(cx)
            measured_centroids_y.append(cy)
            final_frame = frame.copy()
            sample_count += 1
            time.sleep(0.15)

        x_bar = float(np.mean(measured_centroids_x))
        y_bar = float(np.mean(measured_centroids_y))
        avg_peak_intensity = float(np.mean(peak_intensities))

        # Step 3: Euclidean Drift Calculation
        dx = float(x_bar - Tx)
        dy = float(y_bar - Ty)
        D = float(np.sqrt(dx**2 + dy**2))

        # Step 4: Decision Rules
        intensity_passed = avg_peak_intensity >= active_min_int
        drift_passed = D <= active_drift_th

        if not intensity_passed and not drift_passed:
            status_text = "FAILED_BOTH_GATES"
            integrity_label = f"FAIL (Intensity {avg_peak_intensity:.1f} < {active_min_int} & Drift {D:.4f}px > {active_drift_th}px)"
        elif not intensity_passed:
            status_text = "INSUFFICIENT_ILLUMINATION"
            integrity_label = f"FAIL (Peak Intensity {avg_peak_intensity:.1f} < {active_min_int})"
        elif not drift_passed:
            status_text = "MISALIGNED"
            integrity_label = f"FAIL (Drift {D:.4f}px > {active_drift_th}px)"
        else:
            status_text = "ALIGNED"
            integrity_label = "PASS (Optimal Intensity & Sub-pixel Stability)"

        result = {
            "timestamp": timestamp_str,
            "duration_seconds": duration_seconds,
            "sample_count": sample_count,
            "calibrated_baseline_x": round(Tx, 4),
            "calibrated_baseline_y": round(Ty, 4),
            "measured_centroid_x": round(x_bar, 4),
            "measured_centroid_y": round(y_bar, 4),
            "delta_x_px": round(dx, 4),
            "delta_y_px": round(dy, 4),
            "euclidean_drift_D_px": round(D, 4),
            "drift_threshold_px": active_drift_th,
            "peak_intensity": round(avg_peak_intensity, 1),
            "min_intensity_threshold": active_min_int,
            "intensity_passed": intensity_passed,
            "drift_passed": drift_passed,
            "is_aligned": (intensity_passed and drift_passed),
            "overall_status": status_text
        }

        # Step 5: Save Audit Text Report
        report_path = os.path.join(self.output_dir, f"alignment_report_{timestamp_str}.txt")
        latest_report_path = os.path.join(self.output_dir, "latest_alignment_report.txt")
        report_text = (
            "===========================================================================\n"
            " OPENFLEXURE SELF-AWARE DIAGNOSTIC HEALTH REPORT FOR ILLUMINATION ALIGNMENT\n"
            "===========================================================================\n"
            f"Timestamp            : {time.strftime('%Y-%m-%d %H:%M:%S')}\n"
            f"Monitoring Duration  : {duration_seconds} s ({sample_count} samples)\n"
            "---------------------------------------------------------------------------\n"
            f"Peak Optical Flux (I): {avg_peak_intensity:.1f} / 255 (Required: >= {active_min_int})\n"
            f"Illumination Quality : {'SUFFICIENT' if intensity_passed else 'UNDER-ILLUMINATED'}\n"
            "---------------------------------------------------------------------------\n"
            f"Calibrated Baseline  : ({Tx:.4f}, {Ty:.4f}) px\n"
            f"Luminance Centroid   : ({x_bar:.4f}, {y_bar:.4f}) px\n"
            f"Offset Vector (dx,dy): ({dx:.4f}, {dy:.4f}) px\n"
            "---------------------------------------------------------------------------\n"
            f"Euclidean Drift (D)  : {D:.4f} px\n"
            f"Drift Threshold (D_th): {active_drift_th:.4f} px\n"
            "---------------------------------------------------------------------------\n"
            f"DIAGNOSTIC STATUS    : {status_text}\n"
            f"METROLOGICAL STATUS  : {integrity_label}\n"
            "===========================================================================\n"
        )
        with open(report_path, "w") as f:
            f.write(report_text)
        with open(latest_report_path, "w") as f:
            f.write(report_text)

        latest_json_path = os.path.join(self.output_dir, "latest_alignment_metrics.json")
        with open(latest_json_path, "w") as f:
            json.dump(result, f, indent=4)

        # Step 6: Generate Publication Plots (600 DPI PNG & Vector PDF)
        if generate_plot:
            baseline_profile = initial_frame[target_y_idx, :]
            final_profile = final_frame[target_y_idx, :]

            plt.rcParams.update({
                'font.family': 'sans-serif',
                'font.size': 11,
                'axes.labelsize': 12,
                'axes.titlesize': 13,
                'xtick.labelsize': 10,
                'ytick.labelsize': 10
            })

            fig, ax = plt.subplots(figsize=(9, 5.5))
            ax.plot(baseline_profile, color='#D9534F', linestyle='--', linewidth=2.0, alpha=0.85,
                    label='Calibrated Baseline (t = 0 s)')
            ax.plot(final_profile, color='#0275D8', linestyle='-', linewidth=1.8,
                    label=f'Evaluation Profile (t = {duration_seconds} s)')

            ax.axhline(active_min_int, color='#28A745', linestyle=':', linewidth=1.5,
                       label=f'Min Intensity Gate (I >= {active_min_int})')

            ax.set_xlim(0, w)
            ax.set_ylim(0, 260)
            ax.set_xlabel("Pixel Index along X-axis (X)")
            ax.set_ylabel("Pixel Intensity (8-bit Gray Level: 0–255)")
            ax.set_title(
                f"Horizontal Illumination Profile Comparison (Y = {target_y_idx})\n"
                f"Status: [{status_text}] | Peak I = {avg_peak_intensity:.1f} | Drift D = {D:.4f} px (D_th = {active_drift_th} px)",
                pad=35
            )
            ax.grid(True, which='major', linestyle=':', linewidth=0.7, alpha=0.7)
            ax.legend(
                loc='lower center',
                bbox_to_anchor=(0.5, 1.02),
                ncol=3,
                frameon=True,
                edgecolor='gray',
                facecolor='white',
                fontsize=9
            )

            plt.tight_layout()
            
            png_path = os.path.join(self.output_dir, f"illumination_line_profile_{timestamp_str}.png")
            pdf_path = os.path.join(self.output_dir, f"illumination_line_profile_{timestamp_str}.pdf")
            latest_png = os.path.join(self.output_dir, "latest_drift_profile.png")
            
            plt.savefig(png_path, dpi=600, bbox_inches='tight')
            plt.savefig(pdf_path, dpi=600, bbox_inches='tight')
            plt.savefig(latest_png, dpi=600, bbox_inches='tight')
            plt.close(fig)

        return result
EOF
```
