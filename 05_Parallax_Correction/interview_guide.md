# 5. Multi-Camera Parallax Correction (多鏡頭視差校正)

## 📌 核心概念
在多鏡頭系統（如廣角切換長焦）中，兩顆鏡頭物理位置有差距（基線 Baseline）。當拍攝近物時，這會造成視角上的偏差（視差 Parallax）。因此，在切換鏡頭的瞬間，必須對焦平面進行補償計算，以達到無縫縮放 (Seamless Zoom) 的體驗。

---

## 🗣️ Basic Interview Question

**Interviewer:** "When smoothly zooming from the Wide lens to the Tele lens, how do you ensure the Tele lens is instantly in focus without having to hunt for the peak again?"

**Candidate:** "I use the focus data from the Wide lens and apply a **Parallax Correction algorithm**. Since the two lenses are separated by a physical baseline distance, focusing on a close object creates a parallax angle. I take the current focus distance (diopter) of the Wide lens, use trigonometry with the known baseline distance, and calculate the shifted target distance for the Tele lens. I then pre-drive the Tele lens motor to this exact position before the zoom transition is fully completed."

---

## 💻 Code Implementation (Python)

```python
import math

def calculate_parallax_dac_correction(wide_diopter, baseline_mm, tilt_angle_deg):
    # 屈光度 (Diopter) 為距離的倒數 (1/meters)
    if wide_diopter <= 0: 
        return wide_diopter # 無窮遠沒有視差，不需修正
        
    # 將屈光度轉回物理距離 d (轉為毫米)
    d_m = 1.0 / wide_diopter
    d_mm = d_m * 1000.0 
    
    # 計算視差位移 (Parallax Shift)
    # 這裡考慮了鏡頭模組安裝時的微小物理傾斜角 (Tilt Angle)
    tilt_rad = math.radians(tilt_angle_deg)
    parallax_shift_mm = (baseline_mm * math.cos(tilt_rad)) / d_mm
    
    # 計算修正後的目標屈光度 (給長焦鏡頭使用)
    correction_diopter = 1000.0 / (d_mm + parallax_shift_mm)
    return correction_diopter
```

---

## 🔥 Deep Dive Follow-up Questions

### Follow-up 1: Macro vs Infinity (微距與無窮遠的差異)
**Interviewer:** "Is parallax correction always necessary, regardless of the subject distance?"
**Candidate:** "No, parallax error is inversely proportional to the subject distance. At infinity (diopter approaches 0), the parallax angle is essentially zero, so no correction is needed. The correction is absolutely critical during **Macro photography**, where the subject distance is comparable to the baseline distance between the cameras, causing severe focal plane mismatches."

### Follow-up 2: Factory Calibration (工廠校準)
**Interviewer:** "The baseline distance on paper might be 15mm, but manufacturing tolerances happen. How do you account for hardware variations?"
**Candidate:** "We rely on **Factory Calibration (工廠端校準)**. During manufacturing, each camera module undergoes a calibration process where the actual physical tilt and optical center shift are measured and stored in the device's EEPROM. My algorithm reads these device-specific calibration values instead of relying on hardcoded theoretical baseline numbers."
