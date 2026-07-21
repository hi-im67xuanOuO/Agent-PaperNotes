# 🗳️ Self-Consistency Improves Chain of Thought Reasoning in Language Models（對同一題取樣多條推理路徑，再用多數決選出最一致的答案）

> #100篇經典挑戰 No.03 ｜ ⭐⭐⭐⭐⭐ ｜ Wang et al., 2022（ICLR 2023）

| 欄位 | 內容 |
|:---|:---|
| **作者 / 機構** | Xuezhi Wang、Jason Wei、Dale Schuurmans、Quoc Le、Ed Chi、Sharan Narang、Aakanksha Chowdhery、Denny Zhou / Google Research（Brain Team） |
| **發表** | ICLR 2023（arXiv:2203.11171，初版 2022/03） |
| **連結** | [arXiv abs](https://arxiv.org/abs/2203.11171) · [ar5iv HTML](https://ar5iv.labs.arxiv.org/html/2203.11171) |
| **領域標籤** | `#reasoning` `#prompting` `#decoding` `#self-consistency` |
| **重要度** | ⭐⭐⭐⭐⭐ — 推理可靠性的基礎技巧，是 CoT 之後的標配；「取樣多條路徑再聚合」也是後續 test-time compute 的雛形 |
| **閱讀狀態** | 精讀 |

Self-Consistency 是一種**解碼策略**，用來取代 Chain-of-Thought（CoT）原本的 greedy decoding。做法是：對同一題**取樣出多條不同的推理路徑**，再對這些路徑的**最終答案做多數決**，選出最一致的答案。方法建立在 CoT few-shot 之上，不需要任何訓練、額外模型或人工標註。

## 問題與核心貢獻

- **痛點（Before）**：CoT 用 greedy decoding，一題只走**一條**推理路徑；若這條路徑中途算錯，答案就錯，且完全沒利用「同一題可以有多條不同推理路徑」這個特性。
- **核心洞見（Aha）**：複雜推理題通常**允許多條不同的正確推理路徑，最後都會收斂到同一個正確答案**；而錯誤的路徑則傾向各自發散、彼此不一致。
  - 類比：一題數學多找幾個人用不同解法算，大家都算到同一個數字，那個數字很可能就是對的。
- **核心貢獻**：
  - 提出 self-consistency 解碼（sample-and-marginalize）：取樣多條路徑 → 對答案邊際化（多數決）；
  - 免訓練、可直接疊加在任何 CoT few-shot 之上；
  - 在多個算術與常識 benchmark 上大幅提升 CoT 表現。
- **成果（After）**：相對 CoT greedy 的絕對增益——GSM8K **+17.9%**、SVAMP **+11.0%**、AQuA **+12.2%**、StrategyQA **+6.4%**、ARC-challenge **+3.9%**。

## 方法拆解

三個步驟：

- **Step 1**：用 CoT few-shot prompt 提問（跟 CoT 一樣）。
- **Step 2**：**不用 greedy**，改用**取樣解碼**（temperature sampling + top-k）產生一組**多樣**的推理路徑，每條都含「推理鏈 + 最終答案」。
- **Step 3**：只抽出每條的**最終答案**，做**多數決**（對推理路徑邊際化），票數最多的答案即輸出。

- **聚合方式**：預設**未加權多數決**。也試過用 token log-prob 正規化後加權，結果幾乎相同（模型把這些生成視為「差不多可能」）；未正規化的加權則明顯更差。
- **取樣設定**：UL2-20B / LaMDA-137B 用 T=0.5、k=40；PaLM-540B 用 T=0.7、k=40；GPT-3 用 T=0.7、不做 top-k。主實驗每題取樣 **40 條路徑**、跑 10 次取平均。

![Self-Consistency 方法示意](https://raw.githubusercontent.com/hi-im67xuanOuO/Agent-PaperNotes/main/notes/assets/sc-fig1-method.png)

> **Figure 1**　上：CoT 用 greedy 只解出一條路徑，一旦算錯（$14）答案就錯。下：Self-Consistency 取樣多條路徑（$18／$26／$18），對最終答案做多數決 → 選出最一致的 $18。

## 實驗與結果

- **測試模型**：UL2-20B、GPT-3（code-davinci-001/002）、LaMDA-137B、PaLM-540B。
- **對照**：同模型的 CoT greedy decoding。

### 主結果：全面且大幅超越 CoT greedy

> **Table 1**　PaLM-540B 在八個推理任務上，CoT greedy 與 Self-Consistency（40 條路徑）的正確率（%）對比。

| 任務 | 類型 | CoT greedy | Self-Consistency | 增益 |
|:---|:---|:---:|:---:|:---:|
| **GSM8K** | 算術 | 56.5 | **74.4** | +17.9 |
| SVAMP | 算術 | 79.0 | 86.6 | +7.6 |
| ASDiv | 算術 | 74.0 | 81.9 | +7.9 |
| AQuA | 算術（選擇題） | 35.8 | 48.3 | +12.5 |
| MultiArith | 算術 | 94.7 | 99.3 | +4.6 |
| StrategyQA | 常識 | 75.3 | 81.6 | +6.3 |
| ARC-challenge | 常識 | 85.2 | 88.7 | +3.5 |
| CommonsenseQA | 常識 | 79.0 | 80.7 | +1.7 |

![PaLM 540B 主結果](https://raw.githubusercontent.com/hi-im67xuanOuO/Agent-PaperNotes/main/notes/assets/sc-palm540b-results.png)

> **Figure 2**　（自繪，數據取自論文回報值）PaLM-540B 上 Self-Consistency（紅）相對 CoT greedy（灰）的提升；GSM8K、AQuA 這類需要多步計算的任務增益最大。

### 隨取樣路徑數擴展：單調上升、約 10~20 條後飽和

![路徑數擴展曲線](https://raw.githubusercontent.com/hi-im67xuanOuO/Agent-PaperNotes/main/notes/assets/sc-scaling.png)

> **Figure 3**　（LaMDA-137B）三個任務上，準確率隨取樣路徑數（1→40）變化。藍線（Self-Consistency）單調上升、約 10~20 條後趨於飽和；橘線（greedy 單一路徑）為固定水平。

### 與其他「多樣性 / 整體」方法比較

- **Sample-and-rank**（取最高機率的一條）增益極小，遠不如多數決。
- **Beam search** 多樣性低：AQuA（UL2）beam search 23.6% vs Self-Consistency 取樣 26.9%；即使用 beam 產生多樣性再投票也只有 24.2%。
- **Prompt 整體法**（GSM8K、LaMDA-137B）：

> **Table 2**　GSM8K（LaMDA-137B）上 Self-Consistency 與 prompt 整體法的比較。

| 方法 | GSM8K |
|:---|:---:|
| 40 組 prompt 排列整體 | 19.2 |
| 3 組 multi-prompt 整體 | 18.6 |
| **Self-Consistency（40 路徑）** | **27.7** |

### 穩健性

- **取樣參數**：溫度 [0.5–1.0]、top-k [10–100] 範圍內表現穩定。
- **不完美 prompt**：範例推理步驟被隨機數字破壞時，greedy CoT 從 17.1% 掉到 14.9%（GSM8K, LaMDA），Self-Consistency 反而回到 23.4%。
- **Zero-shot CoT**：套上 self-consistency，GSM8K（PaLM）從 43.0% 提升到 69.2%（+26.2%）。

## 可借鑑之處

- **可借用 / 可復用的方法點**：
  1. 「取樣多條 + 對答案多數決」是**免訓練、可疊加在任何 CoT 之上**的可靠性提升法。
  2. 要多樣性用**取樣**（temperature/top-k）而非 beam search；beam 的輸出多樣性較低。
  3. 用「答案一致性」當作信心訊號——收斂度高的答案較可信。
- **適用場景**：**最終答案可聚合／可比對**的任務（算術、選擇題、是非題）；以準確率換取數倍推理算力。

## 局限與延伸

1. **推理成本數倍**：40 條路徑等於約 40× 推理算力；增益隨路徑數飽和（約 10~20 條後邊際遞減）。
2. **需要可聚合的答案**：多數決要求答案能被抽取並比對；**開放式生成**（無單一標準答案）不直接適用（論文自陳）。
3. **仍依賴底層 CoT 品質**：若模型根本不會對該題推理，取樣再多也難投出正確答案。

### 名詞速查

| 名詞 | 一句白話 |
|:---|:---|
| **Self-Consistency** | 對同一題取樣多條推理路徑，再對最終答案多數決的解碼策略 |
| **Greedy decoding** | 每步都選機率最高的 token，只產生單一確定路徑 |
| **Sampling decoding** | 依機率分布抽樣 token（配 temperature / top-k），可產生多樣輸出 |
| **Marginalize（邊際化）** | 對「推理路徑」加總／忽略，只看最終答案的分布 |
| **Sample-and-rank** | 取樣多條後選機率最高的一條（本文對照組，效果較差） |

## 歷史定位與展望

- **歷史定位**：
  - 直接建立在 `[[Chain-of-Thought Prompting (Wei et al., 2022)]]` 之上，把它的 greedy 解碼換成「取樣 + 多數決」；
  - 成為 CoT 之後的推理標配，並被 `[[ReAct (Yao et al., 2022)]]` 直接採用（ReAct↔CoT-SC 結合）。
- **延伸閱讀**：
  - 往前：`[[Chain-of-Thought Prompting (Wei et al., 2022)]]`
  - 往後：`[[Tree of Thoughts (Yao et al., 2023)]]`（把「取樣多路徑」擴成有結構的搜尋）、`[[ReAct (Yao et al., 2022)]]`、以及後續的 verifier / weighted voting 方向
- **展望**（依論文與後續發展）：
  - 「取樣多條解、再聚合」是後續 **test-time compute / inference scaling**（如 o1、R1 一類推理模型）的雛形之一；
  - 後續研究朝「用 verifier 或加權投票取代單純多數決」「降低取樣成本」等方向延伸。
