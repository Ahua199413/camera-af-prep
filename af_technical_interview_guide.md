# Autofocus Technical & Theory Interview Guide (自動對焦技術與理論面試指南)

This guide provides deep technical reviews of autofocus (AF) theory, actuator physics, and project STAR-method breakdowns to help you prepare for technical discussions with recruiters and engineering managers.

本指南提供自動對焦（AF）理論、馬達物理學的深度解析，以及採用 STAR 原則整理的專案成果說明，供您在技術關卡與主管面談時進行複習。

---

## 1. Core AF Technologies: Theory & Principles (核心對焦技術理論)

A senior AF engineer must explain how different AF sensors and algorithms cooperate in a modern smartphone.

### ① Contrast AF (反差對焦 / 降噪對焦)
*   **Principle:** Focus is determined by finding the lens position that yields the highest image sharpness (contrast). The ISP evaluates high-frequency components of the raw pixels within the Focus Window (ROI).
*   **Mechanism:** The lens is driven step-by-step through a range (sweeping). At each step, a Focus Value (FV) is calculated. The lens returns to the position with the peak FV.
*   **Pros & Cons:**
    *   *Pros:* Extremely high accuracy; does not require special sensors.
    *   *Cons:* Very slow (requires multiple frame reads); prone to "focus hunting" (lens swings back and forth to find the peak).

### ② PDAF - Phase Detection AF (相位對焦)
*   **Principle:** Left-shielded and right-shielded pixels (PD pixels) are embedded on the image sensor. The phase difference (disparity) between the left and right light paths is calculated to determine the defocus direction and magnitude.
*   **Disparity to Defocus Mapping:**
    $$\text{Defocus (Diopters)} = \text{Disparity} \times \text{Conversion Factor}$$
*   **Pros & Cons:**
    *   *Pros:* Extremely fast; knows exactly which direction and how far to move the lens in 1 frame.
    *   *Cons:* Performance degrades in low-light (noisy PD signals) and low-contrast target environments.

### ③ ToF / Laser AF (飛行時間/雷射對焦)
*   **Principle:** An infrared laser diode emits a pulse, and a Time-of-Flight sensor measures the round-trip time of the light bouncing off the target.
*   **Distance to Diopter Conversion:**
    $$\text{Diopter} = \frac{1000}{\text{Distance (mm)}}$$
*   **Pros & Cons:**
    *   *Pros:* Active sensing; works in pitch-black environments; instant distance measurement.
    *   *Cons:* Short range (typically limited to 1.5 - 5 meters); vulnerable to glass reflections or multi-path interference (e.g. rain/fog).

### ④ Hybrid AF (混合對焦系統)
*   **Coordination Strategy:**
    1.  **Coarse Stage (Immediate):** Use **ToF** to get a rough distance and instantly drive the lens close to the target (e.g., from infinity to 1.2m).
    2.  **Fine Stage (Precision):** Use **PDAF** to calculate the fine defocus offset and drive the lens to near-focus.
    3.  **Fallback/Verification Stage:** If PDAF confidence is low (low light), use **Contrast AF** to scan and lock onto the peak focus.

---

## 2. Actuator (VCM) Physics & Diopter Calibration (馬達物理與屈光度標定)

### Voice Coil Motor (VCM) Characteristics
*   **How it works:** Current passing through the coil interacts with permanent magnets, creating Lorentz Force ($F = I \cdot L \times B$) that pushes the lens carrier forward against a mechanical spring.
*   **DAC control:** The coil current is regulated by a DAC (Digital-to-Analog Converter), typically 10-bit (0 - 1023).
*   **Hysteresis (滯後):** The spring has mechanical memory, and the carrier has sliding friction. Driving the lens from $0 \rightarrow 500$ DAC versus $1000 \rightarrow 500$ DAC results in slightly different lens positions.
*   **Thermal Expansion:** Heat from the sensor (CIS) and VCM coil expands the camera module housing. This changes the distance between the lens and the sensor, shifting the infinity focus position (usually requiring the lens to move further back, meaning a lower DAC current is needed to focus at infinity).

### Diopter Table (Lens Position-to-Diopter Mapping)
*   **Purpose:** Maps the physical lens position (DAC) to optical focal distance (Diopter = 1/meters).
*   **Calibration Target:**
    *   **Infinity ($0\text{ D}$):** Focused at infinity (e.g., distant mountain).
    *   **Macro ($10\text{ D}$):** Focused at near limit (e.g., 10cm object).
*   **Multi-Cam Sync:**
    When switching from Wide to Tele during zoom, the system reads $DAC_{Wide}$, translates it to Diopter ($D$) using the Wide Diopter Table, and then uses the Tele Diopter Table to pre-drive the Tele lens to the exact corresponding $DAC_{Tele}$, avoiding focus drops.

### Variable Aperture (VA) Focus Peak Shift
*   **The Phenomenon:** Flagship cameras with physical variable apertures (changing between f/1.46, f/1.68, f/4.0) experience shifting focus peaks due to changing spherical aberration as the aperture opening changes.
*   **The Challenge:** When switching apertures, the focal plane shifts. The AF system must compensate by adjusting the target VCM DAC position to keep the image sharp. This requires mapping the shift offsets under various temperatures.

---

## 3. Project Walkthroughs (STAR Method)

Use these templates to answer technical questions about your projects.

### 🌟 Project 1: AF Comparator (對焦除錯與視覺化工具)
*   **Situation:** The engineering team had to manually parse huge text logs and inspection folders containing hundreds of JPEGs/MP4s to diagnose autofocus (GAF) failures. This manual debugging process was a major bottleneck in the tuning cycle.
*   **Task:** Build an automated, high-performance debug dashboard to extract GAF ROI metadata and enable dual-video synchronized analysis.
*   **Action:** 
    *   Developed a Flask-based web application.
    *   Implemented nested Protobuf parsing to extract real-time AEC, motion, and GAF ROI coordinates from images/videos.
    *   Optimized frame parsing performance by 10X using Python's `ThreadPoolExecutor` and prevented Flask thread-locking via non-blocking AJAX polling.
    *   Implemented synchronized video tracking (Side-by-Side Playback) and real-time ROI overlay rendering on the frontend.
*   **Result:** Reduced manual inspection time by 50%, enabling the team to analyze 78+ complex scenarios efficiently.

### 🌟 Project 2: Diopter Table Verification & Actuator Analysis (屈光度表與馬達分析)
*   **Situation:** Physical variations and thermal expansion in VCM actuators cause defocusing during multi-camera switching (Dual-Cam Sync) and sensor fusion (PDAF & ToF alignment).
*   **Task:** Analyze VCM DAC thermal drift and verify the accuracy of the Lens Position-to-Diopter mapping tables under diverse environment scenarios.
*   **Action:**
    *   Created log-correlation scripts to align high-level image metadata with low-level kernel/driver logs (`AF LTC`, `updated AF raw lens`).
    *   Calculated VCM DAC thermal drift rates ($DAC/^\circ C$) by correlating CIS temperature with matched log offsets.
    *   Verified the mapping table designs by comparing ToF physical distance (converted to Diopters) with GAF's computed Lens Position-to-Diopter values.
*   **Result:** Successfully verified focus accuracy limits and provided critical data to validate Lens Thermal Compensation (LTC) and multi-camera focus transitions.

### 🌟 Project 3: Variable Aperture (VA) Focus Peak Shift Evaluation (可變光圈焦平面偏移與溫飄評估)
*   **Situation:** Variable aperture camera modules suffer focus shifts (Aperture Peak Shift) due to spherical aberration variations when switching between f-stops (e.g., f/1.46 to f/4.0), which are further affected by environmental temperature changes.
*   **Task:** Establish an automated verification framework to measure and calibrate focus peak shifts under different apertures and temperatures.
*   **Action:**
    *   Wrote automated adb shell and Python scripts (`FullSweep_Auto_Run`) to control device camera settings, trigger heating/cooling cycles, and run 40-iteration focus sweeps (0 to 292 cPos) for each aperture.
    *   Developed log-parsing scripts (`parse_dof_logs`) to extract and organize real-time depth logs (cPos, temperature, VCM DAC) from the device.
    *   Calculated stabilized peak focus positions and mapped peak shift statistics across different aperture settings and temperature profiles.
*   **Result:** Enabled systematic validation of aperture-shift compensation algorithms, replacing manual sweeps and significantly accelerating tuning cycle speed.
