# 核心對焦演算法程式碼工具箱 (Core AF Algorithms Code Cookbook)

本工具箱提供通用對焦演算法的 **Python** 與 **C++** 實作範例，專注於通用對焦系統的邏輯設計。

---

## 1. 陀螺儀角速度積分偵測手機晃動 (Gyro SCD)
*   **用途：** 讀取 IMU 數據並積分，判斷手機旋轉角度是否超過觸發自動對焦的閾值。

### 🐍 Python 實作
```python
class GyroSceneChangeDetector:
    def __init__(self, threshold_degrees: float = 2.0):
        self.threshold = threshold_degrees
        self.accumulated_angle = 0.0 # 累計旋轉角度 (度)
        
    def process_gyro_data(self, angular_velocity_deg_s: float, dt_sec: float) -> bool:
        """
        處理每一格的陀螺儀角速度
        angular_velocity_deg_s: 當前角速度 (度/秒)
        dt_sec: 兩幀之間的時間差 (秒)
        回傳值: True 代表觸發重對焦 (SCD)
        """
        # 進行角速度對時間積分
        delta_angle = abs(angular_velocity_deg_s) * dt_sec
        self.accumulated_angle += delta_angle
        
        # 若累計角度大於設定閾值，觸發對焦並重設累計器
        if self.accumulated_angle >= self.threshold:
            self.accumulated_angle = 0.0
            return True
            
        # 緩慢衰減累計值，防止靜止狀態下微幅雜訊隨時間無限累積
        self.accumulated_angle *= 0.95 
        return False

# 測試範例
detector = GyroSceneChangeDetector(threshold_degrees=2.0)
# 模擬手機快速轉動 0.5 秒 (角速度 6度/秒, 每幀 1/30 秒)
dt = 1/30.0
for frame in range(15):
    triggered = detector.process_gyro_data(6.0, dt)
    print(f"Frame {frame+1}: Angle={detector.accumulated_angle:.2f}° | Triggered={triggered}")
```

---

## 2. 馬達反向間隙補償 (VCM Backlash Compensation)
*   **用途：** 當鏡頭改變移動方向時，補償機械摩擦遲滯，確保精準停在目標 DAC。

### 💻 C++ 實作
```cpp
#include <iostream>
#include <cmath>

class VCMActuator {
private:
    int current_dac = 0;
    int backlash_compensation = 8; // 馬達反向間隙補償值 (DAC 單位)
    
    // 模擬底層驅動寫入馬達暫存器
    void write_dac_hardware(int dac) {
        std::cout << "  [HW] Writing DAC to VCM: " << dac << std::endl;
        current_dac = dac;
    }

public:
    VCMActuator(int initial_dac) : current_dac(initial_dac) {}

    void drive_to(int target_dac) {
        std::cout << "Drive Request: " << current_dac << " -> " << target_dac << std::endl;
        
        if (target_dac == current_dac) return;

        // 判斷移動方向 (正向為靠近 macro, 反向為靠近 infinity)
        bool moving_forward = (target_dac > current_dac);

        // 為確保每次都從同一方向 settling (例如正向移動)
        if (!moving_forward) {
            // 如果要反向移動：先過衝 (Overshoot)，再拉回
            int overshoot_dac = target_dac - backlash_compensation;
            std::cout << "  [Compensation] Overshoot applied." << std::endl;
            write_dac_hardware(overshoot_dac);
        }
        
        // 最終寫入目標值
        write_dac_hardware(target_dac);
    }
};

int main() {
    VCMActuator vcm(400);
    // 情境一：同向移動 (正向)，直接寫入
    vcm.drive_to(450);
    // 情境二：反向移動 (反向)，執行補償
    vcm.drive_to(420);
    return 0;
}
```

---

## 3. 動態適應型 IIR 濾波器 (Dynamic Adaptive IIR Filter)
*   **用途：** 平滑 PDAF / ToF 的數據流，在距離變化大時快速響應，接近目標時減少噪訊抖動。

### 🐍 Python 實作
```python
class AdaptiveIIRFilter:
    def __init__(self, initial_val: float):
        self.val = initial_val
        
    def filter_value(self, measured: float) -> float:
        """
        動態調適型一階 IIR 濾波器
        """
        diff = abs(measured - self.val)
        
        # 根據測量值與當前狀態的差值，動態計算 alpha 係數
        # 差值極大時 (大於 100 DAC)：alpha 接近 1.0 (快速收斂)
        # 差值極小時 (小於 10 DAC)：alpha 接近 0.1 (深度平滑)
        if diff > 100.0:
            alpha = 0.9
        elif diff < 10.0:
            alpha = 0.1
        else:
            # 線性內插
            alpha = 0.1 + 0.8 * ((diff - 10.0) / 90.0)
            
        self.val = alpha * measured + (1.0 - alpha) * self.val
        return self.val

# 測試範例
f = AdaptiveIIRFilter(400.0)
# 模擬目標突然跳變，隨後在小範圍內震盪的雜訊
measurements = [550, 548, 552, 549, 550]
for idx, m in enumerate(measurements):
    out = f.filter_value(m)
    print(f"Frame {idx+1}: Input={m} | Filtered={out:.2f}")
```

---

## 4. 反差爬山搜尋迴路模擬 (Contrast AF Climb-Hill State Machine)
*   **用途：** 模擬爬山演算法的步長控制、斜率判斷與峰值越過回退的狀態機控制迴路。

### 🐍 Python 實作
```python
class ClimbHillSearch:
    def __init__(self, start_dac: int, step_size: int = 16):
        self.current_dac = start_dac
        self.step = step_size
        self.last_fv = -1.0
        
        self.peak_dac = -1
        self.state = "INIT" # 狀態機: INIT -> SWEEP -> REVERSE -> DONE
        
    def next_step(self, current_fv: float) -> tuple:
        """
        根據輸入的反差評估值 (FV)，計算下一步馬達位置
        回傳值: (next_dac, is_done)
        """
        if self.state == "INIT":
            self.last_fv = current_fv
            self.state = "SWEEP"
            self.current_dac += self.step
            return self.current_dac, False
            
        elif self.state == "SWEEP":
            if current_fv > self.last_fv:
                # 反差值持續上升，繼續前進
                self.last_fv = current_fv
                self.current_dac += self.step
                return self.current_dac, False
            else:
                # 反差值下降，代表越過峰值！開始反向拉回
                print(f"  [Sweep] Peak detected. Reversing from {self.current_dac}")
                self.state = "REVERSE"
                # 反向移動且將步長減半以求精確
                self.step = -self.step // 2
                self.current_dac += self.step
                return self.current_dac, False
                
        elif self.state == "REVERSE":
            # 已經回到峰值位置，搜尋結束
            self.state = "DONE"
            return self.current_dac, True
            
        return self.current_dac, True

# 模擬測試：假設真實對焦峰值在 450 處的反差曲線
# 鏡頭位置: 400 -> 416 -> 432 -> 448 -> 464 (此時 FV 下降) -> 回退到 456
search = ClimbHillSearch(start_dac=400, step_size=16)

# 模擬影像反差響應 (模擬鏡頭位置對應的 Focus Value)
def mock_sensor_fv(dac):
    # 拋物線反差模擬：在 450 處有最大值 1000
    return 1000.0 - 0.1 * (dac - 450)**2

dac = 400
done = False
print(f"Start Search at DAC: {dac}")
while not done:
    fv = mock_sensor_fv(dac)
    dac, done = search.next_step(fv)
    if not done:
        print(f"Drive lens to: {dac} (Mocked FV: {fv:.1f})")
    else:
        print(f"Search Finished! Peak locked at DAC: {dac}")
```
