# Huawei Japan R&D Center (Shinagawa) - Camera AF Interview Strategy (華為品川研究所面試戰略指南)

This document provides a targeted analysis and interview preparation strategy for the **Senior Camera Algorithm Engineer (AF)** role at Huawei's Japan Research Center in Shinagawa, Tokyo.

本指南專為**華為日本品川研究所**的「資深相機演算法工程師（AF）」職缺量限設計，協助您針對華為的產品線技術與面試風格進行重點準備。

---

## 1. Company Background & Context (華為品川研究所背景分析)

*   **Location:** Shinagawa, Tokyo (品川，東京). A major R&D hub for Huawei's consumer device business.
*   **Product Focus:** Flagship smartphones (Mate Series, Pura Series) and advanced imaging technologies under the **XMAGE** brand.
*   **Team & Culture:** 
    *   Highly execution-focused, high-performance R&D environment.
    *   Cross-border collaboration: The team consists of elite Chinese engineers, Japanese optical/sensor experts, and global researchers.
    *   **Your Language Advantage:** Being a native Mandarin speaker with JLPT-N2 Japanese makes you an ideal bridge for the Shinagawa R&D team.

---

## 2. Technical Alignment: Matching Huawei's Flagsip Technologies (華為旗艦技術對齊)

Huawei is a pioneer in several mobile camera technologies. Your experience directly aligns with their highest-priority R&D targets:

### ① Physical Variable Aperture (可變光圈技術)
*   **Huawei's Product:** The Pura 70 and Mate 60 series feature a physical variable aperture (e.g., f/1.4 to f/4.0) to control depth-of-field and light intake.
*   **Your Match:** Your recent work on **Variable Aperture (VA) Focus Peak Shift & Thermal Drift Evaluation** (f/1.46, f/1.68, f/4.0) is a direct hit. 
*   **Key Message to Highlight:**
    *   "I have hands-on experience developing automation tools to calibrate and evaluate focal peak shifts caused by spherical aberration changes during variable aperture transitions across various temperatures."

### ② Periscope Telephoto & Heavy Lens VCM Control (潛望式長焦與大鏡頭控制)
*   **Huawei's Product:** High-magnification optical zoom (e.g., 5X/10X periscope telephoto modules) using large, heavy glass elements.
*   **Your Match:** Your analysis of Voice Coil Motor (VCM) DAC behavior, thermal drift, and gravity compensation (G-Sensor offset).
*   **Key Message to Highlight:**
    *   "I verified Lens Position-to-Diopter mapping table designs and analyzed VCM DAC thermal drift to ensure focus precision for heavy telephoto lens groups and multi-camera transitions (Dual-Cam Sync)."

### ③ Sensor Fusion & Hybrid AF (多感測器融合對焦)
*   **Huawei's Product:** Combining custom RYYB sensors, Laser AF (ToF), and PDAF.
*   **Your Match:** Verifying autofocus target calculations that merge ToF physical distance measurements with PDAF defocus values in Diopter space.
*   **Key Message to Highlight:**
    *   "I verified GAF target calculations that integrate ToF ranging with PDAF disparity inputs in Diopter space, optimizing focus convergence in low-light and low-contrast environments."

---

## 3. Anticipated Interview Technical Questions (預測技術面試題)

Huawei's technical interviewers (hiring managers and principal engineers) will drill down into **log analytics, edge cases, and automation design**.

### Q1: "How did you measure and calculate the Variable Aperture (VA) peak shift?"
*   *Answer Outline:* 
    *   We control the camera module through adb scripts (`FullSweep_Auto_Run`) to perform automated lens sweeping (0 to 292 cPos).
    *   We run multiple iterations (up to 40 runs) under heating/cooling cycles to ensure statistical stability.
    *   We parse the raw DOF/kernel logs (`parse_dof_logs`) to extract `temp`, `cPos`, and focus contrast values.
    *   By fitting the focus contrast values to find the peak for each aperture value (e.g., f/1.46 vs f/4.0) across temperature states, we identify the exact Peak Shift offset (DAC delta) that the compensation algorithm must apply.

### Q2: "Why do we use Diopter instead of physical distance for focus mapping in Dual-Cam zoom transition?"
*   *Answer Outline:* 
    *   Physical distance has a hyperbolic, non-linear relationship with lens displacement ($1/x$). It is difficult to interpolate.
    *   Diopter ($1/\text{distance in meters}$) is highly linear with respect to lens displacement (DAC position).
    *   Using Diopter allows the focus coordinator to perform simple linear translations: $DAC_{Wide} \rightarrow \text{Diopter} \rightarrow DAC_{Tele}$, enabling seamless pre-focusing within 1 frame ($<30\text{ms}$) during zoom switching.

### Q3: "What are the common challenges when validating VCM actuators under extreme temperatures?"
*   *Answer Outline:* 
    *   **Thermal Shift:** The camera barrel expands/contracts, shifting the physical focus plane. The infinity DAC position shifts (usually requiring less current/lower DAC to focus).
    *   **VCM Hysteresis:** Friction shifts under high temperature.
    *   *Our Verification:* By checking the driver log outputs (`AF LTC` / `updated AF raw lens`) and correlating them with image metadata, we verify if the compensation offsets correctly center the focus peak.

---

## 4. Next Steps & Action Plan (下一步行動方案)

1.  **Resume Submission:**
    *   Alice will submit your resume to the Huawei Shinagawa Camera Team. 
2.  **Manager Review:**
    *   The hiring manager will review your technical bullet points. Your "AF Comparator" and "Variable Aperture (VA) Peak Shift" experiences will stand out significantly.
3.  **Bilingual Technical Prep:**
    *   Review the [af_technical_interview_guide.md](file:///Users/cboris/Downloads/interview/af_technical_interview_guide.md) to brush up on PDAF and VCM physics.
    *   Practice describing the VA project in Chinese and English.
