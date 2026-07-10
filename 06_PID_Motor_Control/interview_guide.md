# 6. PID Motor Control (PID 馬達平滑控制)

## 📌 核心概念
當 AF 演算法算出目標 DAC 位置後，如何驅動音圈馬達 (VCM) 抵達該位置是一門學問。不能推太快導致「衝過頭 (Overshoot)」，也不能推太慢導致「對焦遲緩」。PID 控制器是解決這個問題的工業標準。

---

## 🗣️ Basic Interview Question

**Interviewer:** "Once your autofocus algorithm outputs a target DAC value, how do you drive the Voice Coil Motor (VCM) to that exact position quickly without overshooting?"

**Candidate:** "I use a **PID (Proportional-Integral-Derivative) controller**. 
- The **Proportional (P)** term drives the motor quickly when the error distance is large. 
- To prevent overshooting past the target, the **Derivative (D)** term acts as a damper, slowing down the motor as the velocity increases or as it approaches the peak. 
- Finally, the **Integral (I)** term accumulates small residual errors over time to eliminate any steady-state error caused by physical friction or gravity acting on the lens."

---

## 💻 Code Implementation (C++)

```cpp
class PIDController {
private:
    float Kp, Ki, Kd;
    float integral_sum = 0.0f;
    float previous_error = 0.0f;
    float max_integral = 1000.0f; // 防呆機制：防止積分飽和 (Anti-windup)

public:
    PIDController(float p, float i, float d) : Kp(p), Ki(i), Kd(d) {}

    float calculate(float target_dac, float current_dac, float dt) {
        float error = target_dac - current_dac;
        
        // 1. Proportional (比例項) - 給予與誤差成正比的推力
        float P = Kp * error;
        
        // 2. Integral (積分項) - 消除靜態誤差
        integral_sum += error * dt;
        
        // Anti-windup 保護：防止長時間卡住導致積分值爆炸
        if (integral_sum > max_integral) integral_sum = max_integral;
        if (integral_sum < -max_integral) integral_sum = -max_integral;
        float I = Ki * integral_sum;
        
        // 3. Derivative (微分項) - 阻尼器，防止衝過頭
        float derivative = (error - previous_error) / dt;
        float D = Kd * derivative;
        
        previous_error = error;
        
        // 回傳馬達實際驅動力量 (DAC 增量或 PWM Duty Cycle)
        return P + I + D; 
    }
};
```

---

## 🔥 Deep Dive Follow-up Questions

### Follow-up 1: Integral Windup (積分飽和問題)
**Interviewer:** "In your code, you cap the `integral_sum`. What problem are you preventing?"
**Candidate:** "I am preventing **Integral Windup**. If the motor gets physically stuck or is trying to reach an impossible position (like hitting the hard macro stop), the integral term will continuously accumulate a massive error sum over time. Once the blockage is cleared, this massive stored energy will cause a violent overshoot. Capping the integral sum prevents this dangerous build-up."

### Follow-up 2: Orientation / Posture Dependency (手機姿態影響)
**Interviewer:** "How does gravity affect VCM control, and how do you handle it when the user points the phone straight down versus straight up?"
**Candidate:** "Gravity significantly affects the VCM because it has to push the lens mass either against gravity or with it. Pointing straight up requires more force than pointing down. To handle this, we read data from the IMU (Gyro/Accelerometer) to detect the device's posture. Based on the posture, we dynamically add a **Feed-forward gravity compensation term** to the motor drive current, ensuring the PID controller behaves consistently regardless of orientation."
