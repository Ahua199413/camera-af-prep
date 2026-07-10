# Autofocus (AF) Algorithm Interview Cheat Sheet

這份文件整理了 AF 面試中最常考的 6 大核心演算法，包含英文擬答 (Q&A) 與對應的程式碼片段 (Code Snippets)。

---

## 1. Contrast AF & Hill Climbing (反差評估與爬山演算法)

**🗣️ Interviewer:** "How do you evaluate image sharpness in a Contrast AF system, and how do you find the optimal focus lens position?"

**🎯 Candidate:** "I evaluate sharpness by calculating the contrast within a specific Region of Interest (ROI). In my implementation, I divide the ROI into small blocks, calculate the intensity difference between adjacent blocks, and sum the maximum contrast. To find the optimal position, I use a **Hill Climbing algorithm**. The lens moves step-by-step; if the contrast increases, it continues in the same direction. Once it detects a contrast drop, it reverses direction with a smaller step size to accurately locate the peak."

### C++ Code Snippet:
```cpp
// 爬山演算法核心邏輯
void focusstrategy() {
    int delta_rate = focus_rate - last_focus_rate;

    // 發現對比度下降 (走過頭了)
    if (delta_rate < -rate_epsylon) {
        move_direction = -move_direction; // 反轉方向
        // 縮小步長，精確尋找
        focus_step = (focus_step - 1 < min_focus_step ? min_focus_step : focus_step - 1); 
        last_focus_rate = focus_rate;
    }
    // 對比度持續上升 (繼續往峰值前進)
    if (delta_rate > rate_epsylon) {
        step_tomax += focus_step;
        // 加大步長，加速尋找
        focus_step = (focus_step + 1 > max_focus_step ? max_focus_step : focus_step + 1); 
        last_focus_rate = focus_rate;
        ++epoch_tomax;
    }
}
```

---

## 2. PDAF Disparity Calculation (SAD 視差計算)

**🗣️ Interviewer:** "Can you explain the mathematical approach you use to calculate Phase Detection disparity?"

**🎯 Candidate:** "Yes. The sensor outputs two separate 1D arrays of pixel intensities: one for left-looking pixels and one for right-looking pixels. I use the **Sum of Absolute Differences (SAD)** algorithm to find the phase disparity. I digitally shift one array against the other across a search window. At each shift, I calculate the SAD. The shift amount that produces the **minimum SAD** is the disparity, which directly correlates to the focus error and motor displacement."

### Python Code Snippet:
```python
def calculate_pdaf_disparity(left_arr, right_arr, max_search_range):
    best_shift = 0
    min_sad = float('inf')
    n = len(left_arr)
    
    # 掃描 Search Range
    for shift in range(-max_search_range, max_search_range + 1):
        sad = 0
        count = 0
        for i in range(n):
            # Boundary check
            if 0 <= i + shift < n:
                sad += abs(left_arr[i] - right_arr[i + shift])
                count += 1
        
        # 使用平均 SAD 避免重疊面積不同造成的偏差
        avg_sad = sad / count if count > 0 else float('inf')
        
        # 尋找極小值 (Minimum SAD)
        if avg_sad < min_sad:
            min_sad = avg_sad
            best_shift = shift
            
    return best_shift # 視差 (Disparity)
```

---

## 3. Sensor Fusion / Hybrid AF (信賴度加權融合)

**🗣️ Interviewer:** "We have a PDAF sensor and a Time-of-Flight (ToF) sensor. How do you combine their data to get the best focus result?"

**🎯 Candidate:** "I use a **Confidence-Weighted Sensor Fusion** approach. Both sensors provide a target lens position and a confidence score. Instead of a hard switch, I calculate a weighted average. The weights are dynamically adjusted based on the environment. For instance, if the Lux index indicates a dark environment, PDAF noise increases, so its confidence score is penalized. The system will then naturally favor the ToF sensor’s target, which remains highly confident in the dark."

### Python Code Snippet:
```python
def fuse_sensors(pdaf_dac, pdaf_conf, tof_dac, tof_conf, lux_index):
    # 1. 根據環境光調整權重
    if lux_index > 400: # 暗處衰減 PDAF 置信度
        pdaf_weight = pdaf_conf * 0.1
        tof_weight = tof_conf * 0.9
    else:
        pdaf_weight = pdaf_conf * 0.6
        tof_weight = tof_conf * 0.4
        
    total_weight = pdaf_weight + tof_weight
    if total_weight < 1e-5: return pdaf_dac 
        
    # 2. 進行加權平均 (Weighted Average)
    fused_dac = (pdaf_dac * pdaf_weight + tof_dac * tof_weight) / total_weight
    return fused_dac
```

---

## 4. Predictive Focus Tracking (預測性對焦追蹤)

**🗣️ Interviewer:** "When tracking a moving subject, the camera always focuses behind the subject due to system latency. How do you fix this?"

**🎯 Candidate:** "I fix this by implementing **Predictive Tracking**, typically using a **1D Kalman Filter**. The filter uses historical focus positions to estimate the subject's velocity. During the prediction phase, it calculates where the subject *will be* in the next frame, completely offsetting the system's mechanical and processing latency. Then, in the update phase, it uses the new sensor reading to correct any estimation errors."

### Python Code Snippet:
```python
class FocusKalmanFilter:
    def __init__(self, process_noise, measurement_noise):
        self.Q = process_noise      # 系統預測誤差
        self.R = measurement_noise  # 感測器量測雜訊
        self.x = 0.0                # 狀態值 (DAC 位置)
        self.P = 1.0                # 誤差協方差
        
    def predict(self):
        # Prediction Step: 預測下一步的狀態
        self.P = self.P + self.Q
        return self.x
        
    def update(self, measurement):
        # Update Step: 根據感測器量測值修正預測
        self.K = self.P / (self.P + self.R) # 卡爾曼增益
        self.x = self.x + self.K * (measurement - self.x)
        self.P = (1.0 - self.K) * self.P
        return self.x
```

---

## 5. Multi-Camera Parallax Correction (多鏡頭視差校正)

**🗣️ Interviewer:** "When smoothly zooming from the Wide lens to the Tele lens, how do you ensure the Tele lens is instantly in focus without hunting?"

**🎯 Candidate:** "I use the focus data from the Wide lens and apply a **Parallax Correction algorithm**. Since the two lenses are physically separated by a baseline distance, looking at a close object creates a parallax angle. I take the focus distance of the Wide lens, use trigonometry with the known baseline distance, and calculate the shifted target distance for the Tele lens. I can then pre-drive the Tele lens motor to this exact position before the zoom transition finishes."

### Python Code Snippet:
```python
import math

def calculate_parallax_dac_correction(wide_diopter, baseline_mm, tilt_angle_deg):
    if wide_diopter <= 0: return wide_diopter 
        
    # 將屈光度轉回物理距離 d (轉為毫米)
    d_m = 1.0 / wide_diopter
    d_mm = d_m * 1000.0 
    
    # 計算視差位移 (Parallax Shift)
    tilt_rad = math.radians(tilt_angle_deg)
    parallax_shift_mm = (baseline_mm * math.cos(tilt_rad)) / d_mm
    
    # 計算修正後的目標屈光度 (給長焦鏡頭使用)
    correction_diopter = 1000.0 / (d_mm + parallax_shift_mm)
    return correction_diopter
```

---

## 6. PID Motor Control (PID 馬達平滑控制)

**🗣️ Interviewer:** "Once your algorithm outputs a target DAC value, how do you drive the Voice Coil Motor (VCM) to that position quickly without overshooting?"

**🎯 Candidate:** "I use a **PID (Proportional-Integral-Derivative) controller**. The Proportional term drives the motor fast when the error is large. To prevent overshooting past the target, the Derivative term acts as a damper, slowing down the motor as it approaches the peak. The Integral term ensures we eliminate any steady-state error caused by physical friction or gravity acting on the lens."

### C++ Code Snippet:
```cpp
class PIDController {
private:
    float Kp, Ki, Kd;
    float integral_sum = 0.0f;
    float previous_error = 0.0f;
    float max_integral = 1000.0f; // 防止積分飽和 (Anti-windup)

public:
    PIDController(float p, float i, float d) : Kp(p), Ki(i), Kd(d) {}

    float calculate(float target_dac, float current_dac, float dt) {
        float error = target_dac - current_dac;
        
        // Proportional (比例項)
        float P = Kp * error;
        
        // Integral (積分項)
        integral_sum += error * dt;
        if (integral_sum > max_integral) integral_sum = max_integral;
        if (integral_sum < -max_integral) integral_sum = -max_integral;
        float I = Ki * integral_sum;
        
        // Derivative (微分項)
        float derivative = (error - previous_error) / dt;
        float D = Kd * derivative;
        
        previous_error = error;
        
        return P + I + D; // 回傳馬達驅動力量
    }
};
```
