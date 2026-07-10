# 深度解析：CDAF 爬山演算法與底層優化

本文件詳細分析了**反差自動對焦 (CDAF)** 的數學基礎與演算法細節。儘管相位對焦 (PDAF) 在速度上佔優勢，但 CDAF 依然是行動裝置相機系統中，確保對焦絕對準確度（Ground Truth）的最終防線。掌握 CDAF 的底層優化，對於解決低照度與微距攝影的挑戰至關重要。

---

## 1. 定義對焦值 (Focus Value, FV)

CDAF 的核心原理是：準確對焦的影像，其局部對比度（高頻分量）會高於模糊的影像。

### 1.1 算子 (The Operator)
為了萃取高頻數據，我們將空間濾波器 (Kernel) 應用於感興趣區域 (Region of Interest, ROI)。常見的算子包括：
*   **Sobel:** 適合即時處理，對方向性邊緣有強烈響應。
*   **Laplacian (二階導數):** 對微小細節極度敏感，但也容易放大雜訊。
*   **Tenengrad:** 計算梯度的平方和 ( Gx^2 + Gy^2 )。因為能形成穩定的峰值，通常是 AF 系統的首選算子。

FV = sum( Tenengrad( I(x,y) ) ) for x,y in ROI

### 1.2 雜訊濾除 (Noise Filtering)
在低照度環境下，感光元件雜訊會產生虛假的高頻訊號。
*   **解決方案:** 在計算 FV *之前*，先對 ROI 應用高斯模糊 (Gaussian Blur) 或時間 IIR 濾波器 (Temporal IIR filter)；或者設定雜訊門檻（僅累加梯度值大於 Threshold 的像素）。

---

## 2. 爬山狀態機 (Hill Climbing State Machine)

鏡頭必須進行物理移動，以在不同的景深位置對 FV 進行採樣。演算法的目標是尋找 FV 曲線的全局最大值 (Global Maximum)。

### 2.1 搜尋策略 (Search Strategy)
1.  **方向探測 (Wobble):** 將鏡頭往前推動一步。若 FV_new > FV_old，則繼續前進；若 FV_new < FV_old，則反轉方向。
2.  **粗略搜尋 (Fast Climbing):** 以較大的步距移動鏡頭，快速攀爬對比度曲線。
3.  **峰值偵測 (Drop-off):** 我們只有在越過峰值、FV 開始下降時，才能確認峰值的存在。偵測條件為 FV_n < FV_{n-1}。
4.  **精細搜尋 (Reversal & Overshoot Correction):** 反轉鏡頭方向，以微步距向峰值回推，精準定位最高點。

### 2.2 過衝與遲滯問題 (The Overshoot Problem)
在精細搜尋階段反轉 VCM (音圈馬達) 方向時，會發生機械遲滯現象 (Hysteresis/Backlash)。演算法在改變方向時，必須套用特定的 DAC_hysteresis 補償偏移量，以確保物理鏡頭能準確回到數學計算出的峰值位置。

---

## 3. 亞步距峰值插值法 (Sub-Step Peak Interpolation)

真實的光學最高點，極少會剛好落在離散的馬達步距上。

為了找出連續空間中的真實峰值，我們使用最高 FV 點 (V_2) 及其相鄰的兩點 (V_1, V_3) 進行**二次插值 (Quadratic Interpolation)**。

假設這三點對應的鏡頭位置為 x_1, x_2, x_3。真實峰值 x_peak 可以透過擬合拋物線來近似：

x_peak = x_2 + [ (V_3 - V_1) / (2 * (2 * V_2 - V_1 - V_3)) ] * Delta_x

*實作備註:* 這個公式避開了複雜的矩陣反轉，且可以使用快速的整數運算來實作，非常適合用於相機硬體抽象層 (HAL)。

---

## 4. 嵌入式 C/C++ 優化 (ISP Level)

由於 CDAF 需要處理海量的像素數據，C/C++ 的實作必須經過極致優化。

### 4.1 記憶體存取模式 (Memory Access Patterns)
影像數據在記憶體中是依 Row-Major (列優先) 順序儲存的。在撰寫巢狀迴圈時，外層迴圈必須是 `y` (Row)，內層迴圈必須是 `x` (Col)，以最大化 CPU Cache 的命中率。

### 4.2 迴圈展開與 SIMD (Loop Unrolling & SIMD)
與其逐個像素計算 3x3 kernel，不如將迴圈展開，並利用 ARM NEON 向量指令集，一次同時處理 4 個或 8 個像素。

```cpp
// 範例: 優化內部迴圈
for (int x = 1; x < width - 1; x += 4) {
    // 使用 SIMD 一次載入 4 個像素
    // 同時套用 Sobel Gx 與 Gy 權重
    // 將結果累加至 FV 暫存器
}
```

### 4.3 定點數運算 (Fixed-Point Arithmetic)
浮點數運算 (FP32) 在 DSP 或微控制器上的成本極高。利用位元移位操作 (`>> 16`)，將插值公式和 kernel 權重轉換為定點數數學 (例如 Q16.16 格式)，如此既能保持精度，又能獲得整數算術邏輯單元 (ALU) 的高速執行效率。
