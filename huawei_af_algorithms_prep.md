# Huawei Camera R&D - Autofocus Algorithm & Math Preparation Guide (華為對焦演算法與數學原理準備指南)

This guide provides deep technical reviews of autofocus (AF) control loops, mathematical modeling, and sensor fusion algorithms commonly tested in senior-level camera R&D interviews.

本指南專為自動對焦演算法工程師設計，深入剖析控制迴路、數學模型、濾波融合等面試核心考點。

---

## 1. Contrast AF: Climb-Hill & Peak Fitting Algorithms (爬山演算法與峰值擬合)

Contrast AF is the ultimate "ground truth" of focus sharpness. Interviewers will ask about search efficiency and peak interpolation.

### ① Step-Size Control Strategy (步長控制策略)
To avoid focus hunting and minimize search frames, the climb-hill algorithm uses dynamic step-sizing:
*   **Far from Peak (Low Contrast):** Large step sizes (e.g., 20-30 DAC units) to quickly identify contrast trends.
*   **Near Peak (High Contrast):** Small step sizes (e.g., 4-8 DAC units) to avoid overshooting.
*   **Trigger criteria:** Step size decreases when the change rate of Focus Value (FV) drops or when the slope changes.

### ② Mathematical Peak Interpolation (數學峰值擬合)
Because the lens moves in discrete steps, the absolute peak of the contrast curve might lie between two measured positions. Moving the lens back and forth to find it is too slow.
*   **Parabolic Fitting (拋物線擬合):** 
    Assume the contrast curve around the peak follows a quadratic equation:
    $$y = a x^2 + b x + c$$
    By taking three measured points $(x_1, y_1), (x_2, y_2), (x_3, y_3)$ around the peak, we solve the system of linear equations for coefficients $a, b, c$.
    The estimated peak lens position ($x_{peak}$) is:
    $$x_{peak} = -\frac{b}{2a}$$
    The algorithm immediately drives the lens to $x_{peak}$ without further scanning.

```
Contrast (FV)
  ^
  |          * (x_peak, y_peak) [Calculated]
  |        /   \
  |    y2 *     * y3
  |      /       \
  |  y1 *
  +---------------------------> Lens Position (DAC)
       x1   x2   x3
```

---

## 2. PDAF: Disparity & Cross-Correlation Algorithms (相位差與互相關演算法)

PDAF is a sensor-level triangulation method. Interviewers want to know how raw PD pixel outputs are translated into a lens move command.

### ① Phase Correlation Calculation (相位相關計算)
The image sensor embeds left-shielded ($L$) and right-shielded ($R$) pixels. The disparity (shift $s$) between the two arrays represents the defocus state.
*   **Sum of Absolute Differences (SAD) Algorithm:**
    To find the disparity, we slide one array relative to the other and calculate the correlation error $SAD(s)$:
    $$SAD(s) = \sum_{i} |L(i) - R(i + s)|$$
    The shift $s$ that minimizes $SAD(s)$ is the calculated phase difference (Disparity).
*   **Sub-pixel Interpolation:**
    To achieve sub-pixel accuracy, we fit a parabola around the minimum SAD value to find the fractional shift $s_{sub}$.

### ② Defocus to Diopter Mapping
Once the disparity is calculated, it is converted to Defocus in Diopter space:
$$\text{Defocus (Diopter)} = \text{Disparity} \times \text{K-Factor}$$
*(The K-Factor is a calibration constant determined by the lens aperture, focal length, and pixel size).*

---

## 3. Sensor Fusion Algorithms: PDAF + ToF (感測器融合演算法)

In flagship systems, PDAF and ToF are fused. You need to explain the algorithm used to merge these inputs.

### ① Confidence-Weighted Fusion (信賴度加權融合)
Different sensors are accurate under different conditions. The final focus target is computed as a weighted average:
$$D_{final} = W_{ToF} \cdot D_{ToF} + W_{PDAF} \cdot D_{PDAF}$$
*   **ToF Weight ($W_{ToF}$):** High in low-light (low lux), close range ($< 1.5\text{m}$), and low-contrast scenes. Degrades to $0$ at long distances or when multi-path reflections are detected.
*   **PDAF Weight ($W_{PDAF}$):** High in bright environments (high lux) and high-contrast texture targets. Degrades to $0$ in pitch black.

### ② Kalman Filter Tracking (卡爾曼濾波)
For moving subjects (e.g. tracking a running pet or person), the target position is continuously updated using a Kalman Filter to predict the next position and velocity:

```
+-------------------------------------------------------------------+
|                       KALMAN FILTER LOOP                          |
+-------------------------------------------------------------------+
                                 |
                                 v
                     [ Predict State (x_hat) ]
                                 |
                                 v
                   [ Calculate Kalman Gain (K) ]
                                 |
                                 v
          [ Update State with Sensor Inputs (ToF / PDAF) ]
                                 |
                                 v
                 [ Update Error Covariance (P) ]
```

---

## 4. Multi-Camera Parallax Correction Algorithm (多鏡頭視差校正)

When switching from Wide to Tele, the physical distance between the two lenses (baseline $B$) causes a perspective shift (Parallax).

$$\text{Diopter}_{Tele} = \text{Diopter}_{Wide} - \Delta \text{Diopter}_{parallax}$$

### Parallax Correction Formula:
If focusing on a near object at distance $d$, the focus plane shift $\Delta$ due to parallax is modeled as:
$$\Delta = \frac{B \cdot \cos(\theta)}{d}$$
*(Where $B$ is the baseline distance, and $\theta$ is the angle of camera tilt).*
The algorithm must calculate this shift in real-time to adjust the pre-focus DAC position of the incoming camera.
