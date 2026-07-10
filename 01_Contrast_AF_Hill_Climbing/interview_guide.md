# 1. Contrast AF & Hill Climbing (反差評估與爬山演算法)

## 📌 核心概念 (結合 autofocus.cpp 實作)
Contrast AF 依賴分析影像的高頻特徵來判斷清晰度。在你的 C++ 實作中，包含了幾個非常實用的工業級防呆機制：
1. **空間低通濾波 (Spatial Low-Pass Filter):** 不逐像素計算梯度，而是將 ROI 切分為 8x4 區塊計算平均值 (`a = a >> 4`)，有效降低高頻雜訊干擾。
2. **動態步長 (Dynamic Step Size):** 根據對比度上升或下降的趨勢，動態調整 `focus_step`。
3. **雜訊容忍度 (Noise Tolerance):** 使用 `rate_epsylon` 過濾微小的數值跳動。
4. **峰值確認 (Peak Confirmation):** 利用 `epoch_tomax` 計數器，避免被局部最佳解欺騙。

---

## 🗣️ Basic Interview Question

**Interviewer:** "How do you evaluate image sharpness in a Contrast AF system, and how do you find the optimal focus lens position?"

**Candidate:** "I evaluate sharpness by calculating the contrast within a specific Region of Interest (ROI). To prevent sensor noise from corrupting the data, my implementation divides the ROI into 8x4 pixel blocks and calculates the block-averaged intensity differences. This acts as an implicit spatial low-pass filter. 

To find the optimal position, I use a **Hill Climbing algorithm**. The lens moves step-by-step; if the contrast increases, it continues in the same direction and increases the step size. Once it detects a contrast drop, it reverses direction with a smaller step size to accurately locate the peak."

---

## 💻 Code Implementation (C++)

```cpp
// 爬山演算法核心邏輯 (擷取自 autofocus.cpp)
void focusstrategy() {
    int delta_rate = focus_rate - last_focus_rate;

    // 發現對比度下降 (走過頭了)
    // 使用 rate_epsylon 來抵抗數值微小跳動的雜訊
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
        
        // 紀錄連續上升次數，用於後續確認是否為真實峰值
        ++epoch_tomax; 
    }
}
```

---

## 🔥 Deep Dive Follow-up Questions

### Follow-up 1: Local Maximum (局部最佳解)
**Interviewer:** "How do you avoid getting stuck in a local maximum?"
**Candidate:** "In my algorithm, I implemented a threshold and an epoch counter (`epoch_tomax`). The contrast must drop continuously for a certain number of steps, or drop below a specific tolerance threshold (`rate_epsylon`), before the system confirms it's the true peak. In a hybrid system, we also use PDAF to immediately point us to the global maximum vicinity."

### Follow-up 2: Flat Scenes (無紋理場景)
**Interviewer:** "What happens if the scene is completely flat or textureless, like a blank white wall?"
**Candidate:** "Pure Contrast AF will fail because a flat scene results in a completely flat contrast curve with no peaks. The lens would perform a full sweep from macro to infinity, find nothing, and eventually park at a default hyperfocal distance. To solve this, we must use active sensors like ToF or a Laser assist beam."

### Follow-up 3: Motor Hysteresis (馬達磁滯/齒輪間隙)
**Interviewer:** "When your algorithm detects a drop and reverses direction, what mechanical issues do you face?"
**Candidate:** "When reversing the VCM motor, we face **Motor Hysteresis (磁滯效應)** and **Backlash (齒輪間隙)**. Returning 1 step backwards doesn't perfectly equal 1 step forwards physically. To solve this, the algorithm must overshoot the target in the reverse direction, and then approach the peak again from the original direction."
