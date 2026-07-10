# 4. Predictive Focus Tracking (預測性對焦追蹤)

## 📌 核心概念
在連續對焦 (AF-C) 情境下，系統的影像處理、演算法計算與馬達物理驅動都會產生延遲 (Latency)。如果直接瞄準當下測量的位置，鏡頭永遠會落在移動中物體的「後方」。卡爾曼濾波 (Kalman Filter) 透過預測未來位置來消弭延遲。

---

## 🗣️ Basic Interview Question

**Interviewer:** "When tracking a moving subject, the camera always focuses behind the subject due to system latency. How do you fix this?"

**Candidate:** "To fix system latency, I implement **Predictive Tracking**, typically using a **1D Kalman Filter**. The filter tracks the subject's historical focus positions to estimate its velocity. During the *Prediction step*, it calculates where the subject *will be* in the next frame, effectively offsetting the mechanical and processing delay. Then, during the *Update step*, it uses the new noisy sensor measurement to correct its internal state model."

---

## 💻 Code Implementation (Python)

```python
class FocusKalmanFilter:
    def __init__(self, process_noise, measurement_noise):
        self.Q = process_noise      # Q: 系統預測誤差 (Process Noise)
        self.R = measurement_noise  # R: 感測器量測雜訊 (Measurement Noise)
        self.x = 0.0                # x: 狀態估計值 (對焦 DAC 位置)
        self.P = 1.0                # P: 估計誤差協方差
        
    def predict(self):
        # Prediction Step: 根據自身模型預測下一步
        self.P = self.P + self.Q
        return self.x
        
    def update(self, measurement):
        # Update Step: 引入感測器實際讀數來修正模型
        # 計算卡爾曼增益 (Kalman Gain)
        self.K = self.P / (self.P + self.R) 
        
        # 修正狀態 (若 K 偏向 1，代表相信測量值；若 K 偏向 0，代表相信預測模型)
        self.x = self.x + self.K * (measurement - self.x)
        self.P = (1.0 - self.K) * self.P
        
        return self.x
```

---

## 🔥 Deep Dive Follow-up Questions

### Follow-up 1: Erratic Movement (不規則移動)
**Interviewer:** "How do you handle sudden, erratic movements where the subject stops or changes direction abruptly?"
**Candidate:** "Erratic movement breaks the prediction model. To handle this, I dynamically adjust the **Process Noise Covariance (Q)** and **Measurement Noise Covariance (R)**. If the actual sensor reading suddenly deviates massively from the Kalman prediction, I dynamically increase the process noise (Q), essentially telling the system to trust the prediction model less and rely more on the raw sensor measurements, until a new stable velocity vector is established."

### Follow-up 2: Occlusion Handling (物體遮蔽處理)
**Interviewer:** "What happens if a tree temporarily passes in front of the moving car you are tracking?"
**Candidate:** "This is an occlusion scenario. If the AF system suddenly reads a massive distance jump (e.g., from 10 meters to 1 meter), the tracking algorithm must suspend the *Update step*. It should rely purely on the *Prediction step* to let the lens coast along the predicted trajectory. Once the car reappears and the distance matches the predicted range again, normal updating resumes."
