# 2. PDAF Disparity Calculation (SAD 視差計算)

## 📌 核心概念
相位對焦 (Phase Detection AF) 透過比較感測器上左半遮蔽 (Left-looking) 與右半遮蔽 (Right-looking) 像素的相位差 (Disparity) 來計算失焦量。最常見的演算法為 SAD (絕對差值和)。

---

## 🗣️ Basic Interview Question

**Interviewer:** "Can you explain the mathematical approach you use to calculate Phase Detection disparity?"

**Candidate:** "The sensor outputs two separate 1D arrays of pixel intensities: left-looking and right-looking pixels. I use the **Sum of Absolute Differences (SAD)** algorithm to find the phase disparity. I digitally shift the right array against the left array across a defined search window. At each shift, I calculate the absolute difference. The shift amount that yields the **minimum SAD** is the disparity, which linearly correlates to the lens focus error."

---

## 💻 Code Implementation (Python)

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
            # Boundary check: 確保加上 shift 後沒有超出陣列邊界
            if 0 <= i + shift < n:
                sad += abs(left_arr[i] - right_arr[i + shift])
                count += 1
        
        # 使用平均 SAD 避免重疊面積不同造成的計算偏差
        avg_sad = sad / count if count > 0 else float('inf')
        
        # 尋找極小值 (Minimum SAD)
        if avg_sad < min_sad:
            min_sad = avg_sad
            best_shift = shift
            
    return best_shift # 回傳計算出的視差 (Disparity)
```

---

## 🔥 Deep Dive Follow-up Questions

### Follow-up 1: Picket Fence Effect (柵欄效應)
**Interviewer:** "What causes PDAF to calculate the wrong disparity, and how do you detect it?"
**Candidate:** "The most common issue is the **Picket Fence Effect** caused by repetitive patterns (like a striped shirt). The SAD curve will show **multiple local minima**, tricking the algorithm into locking onto the wrong phase shift. I detect this by analyzing the shape of the SAD curve—if there are multiple valleys with similar minimum values, I label the PDAF confidence as low and fallback to Contrast AF."

### Follow-up 2: Low Contrast vs Defocus (低對比度與失焦的差別)
**Interviewer:** "How does the SAD curve look when the subject is completely out of focus versus when the subject just has low contrast?"
**Candidate:** "When the subject is completely out of focus, the SAD curve is very wide and shallow, but it still has a discernible minimum. However, when the subject has extremely low contrast, the SAD values across all shifts are almost identical, resulting in a flat line. In that case, the disparity is entirely unreliable."
