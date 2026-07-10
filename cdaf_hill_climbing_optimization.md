# Deep Dive: CDAF Hill Climbing Algorithm & Optimization

This document analyzes the mathematical and algorithmic intricacies of **Contrast Detection Auto Focus (CDAF)**. While Phase Detection (PDAF) is faster, CDAF remains the ultimate ground truth for focus accuracy in mobile camera systems. Mastering its optimization is crucial for low-light performance and macro photography.

---

## 1. Defining the Focus Value (FV)

CDAF works on the principle that an image in sharp focus has higher local contrast (more high-frequency components) than a blurred image.

### 1.1 The Operator
To extract high-frequency data, we apply a spatial filter (kernel) to the Region of Interest (ROI). Common operators include:
*   **Sobel:** Good for real-time processing, strong response to directional edges.
*   **Laplacian (2nd Derivative):** Highly sensitive to fine details but also sensitive to noise.
*   **Tenengrad:** Computes the gradient magnitude squared $(G_x^2 + G_y^2)$. Often the preferred choice for AF due to robust peak formation.

$$
FV = \sum_{x,y \in ROI} \text{Tenengrad}(I(x,y))
$$

### 1.2 Noise Filtering
In low light, sensor noise introduces false high-frequency signals.
*   **Solution:** Apply a Gaussian Blur or a Temporal IIR filter to the ROI *before* calculating the FV, or introduce a noise threshold (only sum gradient values > Threshold).

---

## 2. The Hill Climbing State Machine

The lens must physically move to sample the FV at different depths. The algorithm searches for the global maximum of the FV curve.

### 2.1 The Search Strategy
1.  **Direction Search (Wobble):** Move the lens one step forward. If $FV_{new} > FV_{old}$, continue forward. If $FV_{new} < FV_{old}$, reverse direction.
2.  **Coarse Search (Fast Climbing):** Move the lens in large steps to quickly traverse the curve.
3.  **Peak Detection (Drop-off):** We only know we've passed the peak when the FV starts dropping. Detect when $FV_{n} < FV_{n-1}$.
4.  **Fine Search (Reversal & Overshoot Correction):** Reverse the lens and take micro-steps back towards the peak to pinpoint the exact maximum.

### 2.2 The Overshoot Problem (Backlash)
When reversing the VCM (Voice Coil Motor) direction for the Fine Search, mechanical hysteresis (backlash) occurs. The algorithm must apply a specific $DAC_{hysteresis}$ offset when changing directions to ensure the physical lens returns to the exact mathematical peak.

---

## 3. Sub-Step Peak Interpolation

The true optical peak rarely falls exactly on a discrete lens step.

To find the true continuous peak, we use **Quadratic Interpolation** using the highest FV point ($V_2$) and its two neighbors ($V_1, V_3$).

Let their corresponding lens positions be $x_1, x_2, x_3$. The true peak $x_{peak}$ can be approximated by fitting a parabola:

$$
x_{peak} = x_2 + \frac{(V_3 - V_1)}{2 \cdot (2V_2 - V_1 - V_3)} \cdot \Delta x
$$

*Implementation Note:* This formula avoids complex matrix inversion and can be implemented using fast integer arithmetic, making it ideal for the camera HAL.

---

## 4. Embedded C/C++ Optimization (ISP Level)

Because CDAF processes massive amounts of pixel data, the C/C++ implementation must be highly optimized.

### 4.1 Memory Access Patterns
Image data is stored in Row-Major order. Nested loops must iterate with `y` (rows) on the outside and `x` (cols) on the inside to maximize CPU Cache Hit Rates.

### 4.2 Loop Unrolling & SIMD
Instead of computing the 3x3 kernel pixel-by-pixel, unroll the loop and utilize ARM NEON vector instructions to process 4 or 8 pixels simultaneously.

```cpp
// Example: Optimizing the inner loop
for (int x = 1; x < width - 1; x += 4) {
    // Load 4 pixels using SIMD
    // Apply Sobel Gx and Gy simultaneously
    // Accumulate into FV registers
}
```

### 4.3 Fixed-Point Arithmetic
Floating-point operations (FP32) are costly on DSPs/microcontrollers. Convert the interpolation formulas and kernel weights into Fixed-Point math (e.g., Q16.16 format) using bitwise shifts (`>> 16`) to maintain precision with integer ALU speeds.
