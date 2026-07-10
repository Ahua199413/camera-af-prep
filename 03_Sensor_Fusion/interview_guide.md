# 3. Sensor Fusion / Hybrid AF (信賴度加權融合)

## 📌 核心概念
沒有一種感測器是完美的。PDAF 在低光源下雜訊大；ToF 會被玻璃反射且有距離限制；CDAF 速度慢。混合對焦 (Hybrid AF) 需要透過「信賴度 (Confidence)」來動態決定各個感測器的權重。

---

## 🗣️ Basic Interview Question

**Interviewer:** "We have a PDAF sensor and a Time-of-Flight (ToF) sensor. How do you combine their data to get the best focus result in a Hybrid AF system?"

**Candidate:** "I use a **Confidence-Weighted Sensor Fusion** approach. Both sensors output a target DAC position and a confidence score. Instead of hard-switching between them, I calculate a dynamically weighted average. For instance, if the ambient Lux index indicates a dark environment, PDAF pixel noise increases, so its confidence is heavily penalized. The system naturally shifts the weight towards the ToF sensor, which remains highly reliable in the dark because it uses its own active infrared illumination."

---

## 💻 Code Implementation (Python)

```python
def fuse_sensors(pdaf_dac, pdaf_conf, tof_dac, tof_conf, lux_index):
    # 1. 動態權重調整 (Dynamic Weighting)
    if lux_index > 400: 
        # 環境極暗，PDAF 信賴度衰減
        pdaf_weight = pdaf_conf * 0.1
        tof_weight = tof_conf * 0.9
    else:
        # 一般光源，正常混合
        pdaf_weight = pdaf_conf * 0.6
        tof_weight = tof_conf * 0.4
        
    total_weight = pdaf_weight + tof_weight
    
    # 防呆：如果兩者信賴度都趨近於零
    if total_weight < 1e-5: 
        return pdaf_dac 
        
    # 2. 進行加權平均 (Weighted Average)
    fused_dac = (pdaf_dac * pdaf_weight + tof_dac * tof_weight) / total_weight
    return fused_dac
```

---

## 🔥 Deep Dive Follow-up Questions

### Follow-up 1: Glass Window Problem (玻璃反射問題)
**Interviewer:** "What if the ToF sensor detects a glass window between the camera and the subject?"
**Candidate:** "The infrared light will reflect off the glass, causing ToF to focus on the window (macro). Meanwhile, PDAF will see through the glass and show a sharp peak at the subject (infinity). The fusion algorithm must detect this **distance mismatch**. If ToF says 'macro' but PDAF says 'infinity' with high confidence, the algorithm should recognize the glass reflection and prioritize PDAF."

### Follow-up 2: ToF Range Limit (ToF 測距極限)
**Interviewer:** "ToF sensors usually have a maximum range, say 5 meters. What happens when tracking a subject moving from 3 meters to 7 meters away?"
**Candidate:** "As the subject crosses the 5-meter boundary, the ToF return signal drops drastically, causing its confidence score to plummet. The fusion algorithm will see the ToF weight drop and automatically hand over the tracking authority to PDAF. To ensure a smooth transition without motor jerks, the handover must be blended over several frames using exponential smoothing."
