# 🧠 Chain-of-Thought Prompting（在 few-shot 範例中加入中間推理步驟，提升大模型多步推理能力）

> #100篇經典挑戰 No.01 ｜ ⭐⭐⭐⭐⭐ ｜ Wei et al., 2022（NeurIPS 2022）

| 欄位 | 內容 |
|:---|:---|
| **作者 / 機構** | Jason Wei、Xuezhi Wang 等 9 人 / Google Research（Brain Team） |
| **發表** | NeurIPS 2022（arXiv:2201.11903，初版 2022/01、v6 2023/01） |
| **連結** | [arXiv abs](https://arxiv.org/abs/2201.11903) · [ar5iv HTML](https://ar5iv.labs.arxiv.org/html/2201.11903) |
| **領域標籤** | `#reasoning` `#prompting` `#emergent-ability` `#in-context-learning` |
| **重要度** | ⭐⭐⭐⭐⭐ — LLM 推理研究的起點；後續 ReAct、Self-Consistency、Tree of Thoughts 等多沿用其思路 |
| **閱讀狀態** | 精讀 |

本篇提出 **Chain-of-Thought（CoT，思維鏈）prompting**：在 few-shot 範例中，除了「問題→答案」，還把中間推理過程一併寫出來作為示範。模型在作答前會先生成一串推理步驟，藉此提升算術、常識、符號等多步推理任務的正確率。方法屬於**純提示（prompting）**，不需要任何訓練或微調。

## 問題與核心貢獻

- **痛點（Before）**：LLM 在「一步到位」的任務表現好，但多步推理（數學應用題、常識推論）表現差。當時的兩條主流解法都成本高：
  - 用「題目＋完整解題步驟」的標註資料做 fine-tune；
  - 單純放大模型規模——但論文指出，這在算術／常識／符號推理上「幾乎沒有平滑增益」。
- **核心洞見（Aha）**：把「解題過程」也寫進 few-shot 範例當示範，模型會模仿「先拆步驟、再給答案」的行為。
  - 類比：如同人類解應用題會先在**草稿紙**上一步步計算，而非直接寫答案。
- **核心貢獻**：
  - 提出 CoT prompting 這個簡單、免訓練的方法；
  - 在多個推理 benchmark 上顯著提升表現；
  - 指出 CoT 的效益是**隨模型規模湧現（emergent）**的現象。
- **成果（After）**：PaLM 540B 在 GSM8K（小學數學應用題）從標準提示的 **17.9% 提升到 56.9%**，超越當時 SOTA（微調 GPT-3 175B + 驗證器，55%）。

## 方法拆解

- **本體**：一種 prompt 格式，差別只在 few-shot 範例的「答案欄」——從「只有答案」改成「推理過程＋答案」。
- **輸入 / 輸出**：輸入是含 CoT 示範的 few-shot prompt；輸出是「一串自然語言推理步驟＋最終答案」，答案位於最後一句。
- **不改動的部分**：不改架構、不改權重、不加解碼器；測試時模型自動在 `A:` 後生成整串推理鏈。

範例對照：

```text
【標準 few-shot】
Q: Roger 有 5 顆網球，又買了 2 罐、每罐 3 顆，現在有幾顆？
A: 答案是 11。                                        ← 只示範「結果」
Q: <新題>
A:  → 模型直接輸出一個數字（多步題常錯）

【Chain-of-Thought】
Q: Roger 有 5 顆網球，又買了 2 罐、每罐 3 顆，現在有幾顆？
A: Roger 一開始有 5 顆。2 罐 × 每罐 3 顆 = 6 顆。5 + 6 = 11。答案是 11。   ← 連「過程」一起示範
Q: <新題>
A:  → 模型先輸出推理鏈，最後才給答案
```

- **作者列出的四項特性**：
  - 把難題拆成中間步驟，等於分配更多運算到多步題；
  - 可解釋（可看出推理停在哪一步）；
  - 適用面廣（凡是人類能用語言講清楚的任務原則上都適用）；
  - 免訓練，現成大模型改範例格式即可。

![CoT 架構圖](assets/cot-fig1-schematic.png)

> **Figure 1**　左為標準提示：範例只示範答案（11），模型面對新題直接輸出數字 → 錯（27）。右為 CoT：範例中把推理過程（藍字）一併寫出，模型面對新題跟著逐步推理（綠字）→ 對（9）。兩者唯一差異在範例答案是否含推理過程。

## 實驗與結果

- **測試模型（5 個家族，350M→540B）**：
  - GPT-3（350M / 1.3B / 6.7B / 175B）
  - LaMDA（422M / 2B / 8B / 68B / 137B）
  - PaLM（8B / 62B / 540B）
  - UL2 20B、Codex（code-davinci-002）
- **範例數（k-shot）**：算術任務 8 個手寫 CoT 範例；AQuA（選擇題）4 個；常識／符號 6–10 個。全程零訓練。
- **對照組**：同模型的標準 few-shot 提示；GSM8K 另比當時 SOTA（微調 GPT-3 175B + 驗證器）。
- **任務分三類**：算術（GSM8K、SVAMP、ASDiv、AQuA、MAWPS）、常識（CommonsenseQA、StrategyQA、Date、Sports）、符號（Last Letter Concatenation、Coin Flip）。

### 主結果

- GSM8K：標準 17.9% → CoT **56.9%**（約 3.2×），超越 SOTA（55%）。
- 規律：**愈需要多步推理的任務，CoT 增益愈大**；接近一步到位的任務（如 ASDiv 72→74）增益小。

![PaLM 540B 主結果](assets/cot-palm540b-results.png)

> **Figure 2**　（自繪，數據取自論文回報值）PaLM 540B 在八個任務上 CoT（紅）與標準提示（灰）的對比；GSM8K 一格越過虛線（前 SOTA 55%）。

> **Table 1**　PaLM 540B 在八個推理任務上，標準提示與 CoT 的正確率（%）對比。

| 任務 | 類型 | 標準 | CoT | 提升 |
|:---|:---|:---:|:---:|:---:|
| **GSM8K** | 算術 | 17.9% | **56.9%** | +39.0（超越前 SOTA 55%） |
| SVAMP | 算術 | 69.4% | 79.0% | +9.6 |
| ASDiv | 算術 | 72.1% | 73.9% | +1.8 |
| AQuA | 算術（選擇題） | 25.2% | 35.8% | +10.6 |
| MAWPS | 算術 | 79.2% | 93.3% | +14.1 |
| StrategyQA | 常識 | 68.6% | 77.8% | +9.2 |
| Date Understanding | 常識 | 49.0% | 65.3% | +16.3 |
| Sports Understanding | 常識 | 80.5% | 95.4% | +14.9 |
| CommonsenseQA | 常識 | 78.1% | 79.9% | +1.8 |
| Last Letter Concat.（in-domain） | 符號 | 7.6% | **99.4%** | +91.8 |
| Coin Flip（in-domain） | 符號 | 98.1% | 100.0% | +1.9 |

- 符號任務 Last Letter（7.6 → 99.4）的重點在**長度外推**：範例為 2 個詞，測試時換成 3、4 個詞，CoT 仍能靠逐步推理泛化，標準提示則大幅下降。

### 湧現能力：效益隨規模出現

- 論文結論：CoT 在約 **100B 參數以下幾乎無效、甚至有害**，過此門檻才明顯有益。
  - 原文：*"chain-of-thought prompting does not positively impact performance until used with models of ∼100B parameters."*
- 小模型會生成「**文句通順但邏輯不通**」的推理鏈，反而拉低分數。
- 此觀察延伸為另一篇專論 **Emergent Abilities of LLMs**。

### Ablation：效益來自哪個因素

作者以三個變體排除替代解釋（LaMDA 137B、PaLM 540B on GSM8K）：

> **Table 2**　三個變體的消融結果，逐一排除「運算量／token 數／知識喚醒」等替代解釋。

| 變體 | 做法 | 結果 | 說明 |
|:---|:---|:---|:---|
| **Equation only** | 範例只放算式，不放自然語言推理 | 幾乎無幫助 | 語意複雜的題目需要自然語言把關係想清楚 |
| **Variable compute only** | 用等長「…」佔位，無實際內容 | ≈ baseline | 效益不是來自「多花運算 / 多輸出 token」 |
| **Chain of thought after answer** | 先給答案、再寫推理 | ≈ baseline | 效益不是來自「喚醒知識」，而是**作答前的循序推理** |

- 結論：效益歸因於「**在給答案之前，按順序生成推理步驟**」，而非運算量、token 數或知識喚醒。

![錯誤類型分析](assets/cot-fig8-error-types.png)

> **Figure 3**　將 62B 模型的錯誤分為三類（語意理解錯、漏一步、其他），並統計放大到 540B 後修正的數量。規模放大主要修正「語意理解」與「漏步驟」這兩類推理錯誤。

### 穩健性

- **不同標註者**：三位作者各自獨立撰寫範例，三套皆大幅優於標準提示。
- **不同範例來源**：從訓練集隨機抽的範例效果與手寫相當。
- **範例順序**：跨 5 個隨機種子的標準差在幾乎所有任務都很小。

## 可借鑑之處

- **可借用 / 可復用的方法點**：
  1. 在 few-shot / in-context 任務中「示範過程，而非只示範結果」，常比增加輸入→輸出對更有效。
  2. 用「等長佔位（variable-compute-only）」與「順序對調（after-answer）」這類 ablation，驗證效益究竟來自哪個因素。
  3. 評估「推理型」技巧時需考慮模型規模門檻，小模型上的失敗不代表方法無效。
- **適用場景**：能用自然語言逐步描述的推理任務（數學、常識、符號操作）。

## 局限與延伸

1. **可解釋性有限**：論文自陳，模型生成的推理鏈不保證反映其內部真實計算（可能答對但推理是事後合理化）；後續 faithfulness 研究對此有進一步探討。
2. **仍需人工範例**：每個任務需人工撰寫 8~10 個高品質 CoT 範例；此成本由後續 Zero-shot CoT、Auto-CoT 嘗試降低。
3. **純文字推理的天花板**：需要精密符號運算、長程狀態追蹤的任務，文字推理會累積誤差；後續以外接工具（PAL、Program-of-Thoughts、ReAct）緩解。

### 名詞速查

| 名詞 | 一句白話 |
|:---|:---|
| **Chain of Thought（思維鏈）** | 模型在給最終答案前，先生成的一串中間推理步驟（自然語言） |
| **Few-shot / In-context learning** | 不更新權重，只在 prompt 裡放幾個示範，讓模型仿照作答 |
| **Emergent ability（湧現能力）** | 小模型沒有、模型放大過某門檻後才出現的能力，隨規模非線性跳升 |
| **GSM8K** | Grade School Math 8K，8,500 題小學數學應用題，需多步算術 |
| **Verifier（驗證器）** | 前 SOTA 方案用的額外模型，對候選答案打分重排 |
| **Ablation（消融實驗）** | 拿掉／替換某個零件，觀察效果變化，用以判斷哪個設計起作用 |

## 歷史定位與展望

- **歷史定位**：
  - 繼承自 Nye 等人的 `[[Scratchpad (Nye et al., 2021)]]`（讓模型寫出中間計算）與 Cobbe 等人的 GSM8K + verifier；
  - 將「靠訓練取得推理」轉為「靠提示引導推理」；
  - 啟發後續一系列工作：`[[Zero-shot CoT (Kojima et al., 2022)]]`、`[[Self-Consistency (Wang et al., 2022)]]`、`[[Tree of Thoughts (Yao et al., 2023)]]`、`[[ReAct (Yao et al., 2022)]]`。
- **延伸閱讀**：
  - 往前：`[[Scratchpad (Nye et al., 2021)]]`、`[[STaR (Zelikman et al., 2022)]]`
  - 往後：`[[Self-Consistency (Wang et al., 2022)]]`、`[[Tree of Thoughts (Yao et al., 2023)]]`、`[[ReAct (Yao et al., 2022)]]`、`[[Emergent Abilities of LLMs (Wei et al., 2022)]]`
- **展望**（依論文與後續發展）：
  - 「先推理再作答」的思路後續發展為 test-time compute / inference scaling（如 o1、R1 一類推理模型）；
  - 「多步推理會累積誤差」推動了工具使用與 agent 方向；
  - 「能力隨規模湧現」成為 emergent abilities 討論的重要案例。
