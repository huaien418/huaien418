# RVV 工程設計筆記

邱懷恩 Evan Chiu · C／RISC-V RVV · 2026-10-22 文件整理

比較 8×8 DCT、量化與 IDCT 影像重建的三種計算映射。個人負責 Parts 1-3；原兩人團隊的 CUDA Parts 4-5 由隊友完成。本筆記依既有程式、封存統計與已保存的影像檢查整理。

## 計算映射

| 版本 | lane 對應的工作 | 實作要點 |
| --- | --- | --- |
| Part 1 純量 | 巢狀迴圈逐項累加 | 建立效能與數值比較基準 |
| Part 2 歸約 | 同一個部分內積的八項乘積 | vle32、vfmul、vfredusum；歸約後再由外層累加 |
| Part 3 獨立輸出 | 八個不同係數或輸出像素 | DCT 以 vlse32 取基底欄；各 lane 用 vfadd 累加，最後 vse32 |

### Part 3 工作分配示意

```text
# DCT mapping sketch: fixed u; lanes correspond to v = 0..7
acc[0:8] = 0
for x in 0..7:
    for y in 0..7:
        acc[0:8] += basis[u][x] * basis[0:8][y] * shifted[x][y]
```

上方為工作分配示意，省略量化及其他流程。Part 3 的 DCT 以跨步載入讀取基底欄；IDCT 以連續載入讀取基底列。少了向量歸約是程式可觀察的設計差異；各差異對效能的個別影響尚未隔離量測。

## 封存效能

封存環境：gem5 23.1.0.0、TimingSimpleCPU、VLEN 256、ELEN 64；L1 指令／資料快取各 32 KiB、8-way，cache line 32 bytes。輸入為 5184×3456 灰階影像，共 279,936 個區塊。

| 指標 | Part 1 純量 | Part 2 歸約 | Part 3 獨立輸出 |
| --- | --- | --- | --- |
| CPU cycles | 75,329,661,396 | 22,809,755,208 | 15,961,219,274 |
| Instructions | 21,374,974,987 | 6,105,866,037 | 3,659,225,278 |
| 純量／該版本週期數 | 1.00x | 3.30x | 4.72x |
| 純量／該版本指令數 | 1.00x | 3.50x | 5.84x |
| CPI | 3.524198 | 3.735709 | 4.361907 |

Part 3 使用約 21.2% 的純量週期數，指令數比值為 5.84x。CPI 從 3.524198 升至 4.361907，總週期仍下降；因此本例需要同時看指令數與 CPI。此處的比值描述指定模擬工作負載，不能直接當成實體板子的加速結果。

## 數值驗證

| 檢查 | 既有結果 | 判讀 |
| --- | --- | --- |
| Part 1 與 Part 3 | 保存的重建 PGM 逐位元組一致 | 支持這組保存輸出的一致性 |
| Part 2 與 Part 1 | 有 5,826 個像素不同 | 不能宣稱三版本逐像素相同 |
| PSNR | 三者記錄值皆為 45.154236 dB | 相同四捨五入數值不足以證明輸出相同 |

原團隊報告將 Part 2 的數值差異歸因於浮點累加順序。程式確實改成先歸約部分內積、再由外層累加；這是有依據的解釋方向。既有檢查未逐一追蹤係數差異，也未重新執行模擬，因此尚未證實每個差異的根因。

## 面試說明

### 為什麼選擇每個 lane 計算獨立輸出？

可將 lane 維持為獨立累加器，省去水平歸約；同時需要配合基底矩陣的記憶體配置。4.72x 是這份封存結果，個別瓶頸仍需 profiling 才能定位。

### 為什麼同時檢查 checksum、影像與 PSNR？

它們回答不同問題：checksum 可提示結果不同，影像位元組比較檢查保存輸出，PSNR 描述重建誤差。本例 Part 2 的 PSNR 相同但仍有像素差異。

### 如果之後補驗證，會先做什麼？

先修正重現路徑、準備小型可重現輸入，記錄編譯器／gem5 版本與指令；再區分純量 fallback、實際 RVV 執行及向量長度測試。這些是後續計畫。

## 限制與來源

證據範圍：本次只整理文件與既有統計，未新增程式執行。保存影像一致不代表所有中間係數或所有輸入皆相同；程式要求八個 FP32 lane，其他 VLEN 的適用性尚未建立。

- [README 與結果說明](https://github.com/huaien418/Computer-Architecture-Project/blob/main/README.md)
- [Part 2 歸約核心](https://github.com/huaien418/Computer-Architecture-Project/blob/main/part2/main.c#L28-L72)
- [Part 3 獨立輸出核心](https://github.com/huaien418/Computer-Architecture-Project/blob/main/part3/main.c#L28-L102)
- [封存統計](https://github.com/huaien418/Computer-Architecture-Project/tree/main/benchmarks/archived)
