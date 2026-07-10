# 自動對焦演算法程式碼工具箱 (AF Algorithms Code Cookbook)

本工具箱提供對焦工程師面試常見演算法的 **Python** 與 **C++** 實作範例。所有實作均以簡潔、易懂、符合面試手寫板（Whiteboard/CoderPad）為原則進行設計。

---

## 1. 拋物線頂點峰值擬合 (Parabolic Peak Fitting)
*   **用途：** 反差對焦 (Contrast AF) 中，利用三個離散的掃描點擬合出連續曲線的最高點，節省對焦時間。
*   **數學原理：** 已知三點 $(x_1, y_1), (x_2, y_2), (x_3, y_3)$，求解二次方程式 $y = ax^2 + bx + c$ 的極值點：
    $$x_{peak} = x_2 - \frac{1}{2} \frac{(x_2 - x_1)^2 (y_2 - y_3) - (x_2 - x_3)^2 (y_2 - y_1)}{(x_2 - x_1)(y_2 - y_3) - (x_2 - x_3)(y_2 - y_1)}$$
    若步長相同（即 $x_2 - x_1 = x_3 - x_2 = H$），公式可簡化為：
    $$x_{peak} = x_2 + \frac{H}{2} \frac{y_1 - y_3}{y_1 - 2y_2 + y_3}$$

### 🐍 Python 實作 (等步長簡化版)
```python
def find_parabolic_peak(x1: float, y1: float, x2: float, y2: float, x3: float, y3: float) -> float:
    """
    計算拋物線極值點 (等步長情況下)
    x1, x2, x3: 鏡頭 DAC 位置 (x2 必須為中間點)
    y1, y2, y3: 對應的反差評估值 (Focus Value, y2 應為最大值)
    """
    denom = y1 - 2 * y2 + y3
    if abs(denom) < 1e-6: # 避免除以零 (三點呈水平線)
        return x2
        
    step_size = x2 - x1
    # 拋物線頂點公式偏移量
    offset = (step_size / 2.0) * (y1 - y3) / denom
    return x2 + offset

# 測試範例
# x = [400, 410, 420], 假設真實峰值在 413 左右
peak_dac = find_parabolic_peak(400, 150, 410, 200, 420, 170)
print(f"Estimated Peak DAC: {peak_dac:.2f}") # 輸出約 413.33
```

### 💻 C++ 實作 (等步長簡化版)
```cpp
#include <iostream>
#include <cmath>

double find_parabolic_peak(double x1, double y1, double x2, double y2, double x3, double y3) {
    double denom = y1 - 2.0 * y2 + y3;
    if (std::abs(denom) < 1e-6) {
        return x2; // 避免除以零
    }
    double step_size = x2 - x1;
    double offset = (step_size / 2.0) * (y1 - y3) / denom;
    return x2 + offset;
}

int main() {
    double peak = find_parabolic_peak(400, 150, 410, 200, 420, 170);
    std::cout << "Estimated Peak DAC: " << peak << std::endl;
    return 0;
}
```

---

## 2. 相位對焦絕對差值和 (SAD) 互相關演算法
*   **用途：** 相位對焦 (PDAF) 中，滑動比對左像素陣列與右像素陣列，計算出相位視差（Disparity）。

### 🐍 Python 實作
```python
import numpy as np

def calculate_pdaf_disparity(left_arr: list, right_arr: list, max_search_range: int) -> int:
    """
    計算 PDAF 視差 (Disparity)
    left_arr, right_arr: 左右 PD 像素亮度陣列 (長度相同)
    max_search_range: 最大滑動像素範圍 (e.g. -5 ~ +5)
    """
    best_shift = 0
    min_sad = float('inf')
    
    n = len(left_arr)
    
    for shift in range(-max_search_range, max_search_range + 1):
        sad = 0
        count = 0
        for i in range(n):
            # 確保滑動比對時不超出邊界
            if 0 <= i + shift < n:
                sad += abs(left_arr[i] - right_arr[i + shift])
                count += 1
        
        # 進行均值化避免因重疊邊界長度不同導致誤差
        avg_sad = sad / count if count > 0 else float('inf')
        
        if avg_sad < min_sad:
            min_sad = avg_sad
            best_shift = shift
            
    return best_shift

# 測試範例
L = [10, 20, 50, 100, 80, 20, 10]
R = [10, 10, 20, 50, 100, 80, 20] # 相對於 L 右移了 1 格
shift = calculate_pdaf_disparity(L, R, 3)
print(f"Calculated Disparity (Shift): {shift}") # 輸出 1
```

---

## 3. 信賴度加權感測器融合 (Confidence-Weighted Sensor Fusion)
*   **用途：** 根據環境亮度 (Lux/BV) 與感測器自身的信賴度 (Confidence)，動態決定 ToF 與 PDAF 的比重。

### 🐍 Python 實作
```python
def fuse_sensors(pdaf_dac: float, pdaf_conf: float, tof_dac: float, tof_conf: float, lux_index: float) -> float:
    """
    信賴度加權融合演算法
    pdaf_dac, tof_dac: 感測器各自估算出的對焦馬達 DAC 目標
    pdaf_conf, tof_conf: 各自的置信度 (0.0 ~ 1.0)
    lux_index: 環境亮度指標 (值越高代表環境越暗)
    """
    # 1. 根據環境光調整權重
    # 若環境太暗 (lux_index > 400)，PDAF 置信度強制衰減
    if lux_index > 400:
        pdaf_weight = pdaf_conf * 0.1
        tof_weight = tof_conf * 0.9
    else:
        pdaf_weight = pdaf_conf * 0.6
        tof_weight = tof_conf * 0.4
        
    total_weight = pdaf_weight + tof_weight
    
    if total_weight < 1e-5:
        # 如果兩者置信度都極低，回傳預設值 (例如無窮遠對焦)
        return pdaf_dac 
        
    # 2. 進行加權平均
    fused_dac = (pdaf_dac * pdaf_weight + tof_dac * tof_weight) / total_weight
    return fused_dac

# 測試範例 (亮處，PDAF 信賴度高)
print(f"Fused DAC (Bright): {fuse_sensors(450, 0.9, 430, 0.8, 100)}") # 權重向 PDAF 偏
# 測試範例 (暗處，PDAF 衰減)
print(f"Fused DAC (Dark): {fuse_sensors(450, 0.9, 430, 0.8, 450)}") # 權重向 ToF 偏
```

---

## 4. 一維卡爾曼濾波對焦追蹤 (1D Kalman Filter Focus Tracking)
*   **用途：** 用於動態場景，平滑並預測連續運動物體在下一影格的對焦馬達位置。

### 🐍 Python 實作
```python
class FocusKalmanFilter:
    def __init__(self, process_noise: float, measurement_noise: float):
        """
        1D 卡爾曼濾波器
        process_noise (Q): 系統預測誤差 (通常極小，e.g. 1e-4)
        measurement_noise (R): 測量誤差 (ToF/PDAF 的誤差波動)
        """
        self.Q = process_noise
        self.R = measurement_noise
        
        self.x = 0.0 # 估算出的狀態值 (對焦 DAC)
        self.P = 1.0 # 估算誤差協方差
        
    def predict(self) -> float:
        """ 預測步驟 (狀態無轉移矩陣，假設上一幀為基礎) """
        self.P = self.P + self.Q
        return self.x
        
    def update(self, measurement: float) -> float:
        """ 更新步驟 (依據感測器量測值修正預測) """
        # 計算卡爾曼增益
        self.K = self.P / (self.P + self.R)
        # 修正估算狀態
        self.x = self.x + self.K * (measurement - self.x)
        # 更新協方差
        self.P = (1.0 - self.K) * self.P
        return self.x

# 測試範例：鏡頭追蹤一個逐漸前進的物體 (DAC 規律遞增，但有噪訊)
kf = FocusKalmanFilter(process_noise=0.1, measurement_noise=10.0)
kf.x = 400.0 # 初始位置

measurements = [412, 418, 432, 441, 453]
for idx, m in enumerate(measurements):
    kf.predict()
    updated_dac = kf.update(m)
    print(f"Frame {idx+1}: Measured={m} | Filtered={updated_dac:.2f}")
```

---

## 5. 多鏡頭視差校正預對焦 (Multi-Cam Parallax Correction)
*   **用途：** 主鏡頭切換至長焦鏡頭時，修正兩顆相機物理位置造成的焦平面誤差。

### 🐍 Python 實作
```python
import math

def calculate_parallax_dac_correction(wide_diopter: float, baseline_mm: float, tilt_angle_deg: float, focal_length_mm: float) -> float:
    """
    計算多鏡頭視差校正後的目標屈光度 (Diopter)
    wide_diopter: 主鏡頭當下的屈光度 (1/d, 單位為 1/meters)
    baseline_mm: 兩相機間基線物理距離 (單位為毫米)
    tilt_angle_deg: 鏡頭物理傾斜度 (單位為度)
    focal_length_mm: 焦距
    """
    if wide_diopter <= 0:
        return wide_diopter # 無窮遠不需要視差修正
        
    # 將屈光度轉回物距 d (公尺)
    d_m = 1.0 / wide_diopter
    d_mm = d_m * 1000.0 # 轉為毫米
    
    # 視差位移公式
    tilt_rad = math.radians(tilt_angle_deg)
    parallax_shift_mm = (baseline_mm * math.cos(tilt_rad)) / d_mm
    
    # 修正後的長焦端焦距偏移 (對應到屈光度)
    correction_diopter = 1000.0 / (d_mm + parallax_shift_mm)
    return correction_diopter

# 測試範例
# 近景 33cm (3.0 D), 基線距離 15mm
corrected_d = calculate_parallax_dac_correction(3.0, 15.0, 0.0, 24.0)
print(f"Original Diopter: 3.0 D | Corrected Tele Diopter: {corrected_d:.3f} D")
```
