# 🛠️ ToolLLM: Facilitating Large Language Models to Master 16000+ Real-world APIs（用 16k+ 真實 API 建 ToolBench、訓練 ToolLLaMA，並以 DFSDT 搜尋樹做多工具編排）

> #100篇經典挑戰 No.07 ｜ ⭐⭐⭐⭐⭐ ｜ Qin et al., 2023（ICLR 2024）

| 欄位 | 內容 |
|:---|:---|
| **作者 / 機構** | Yujia Qin、Shihao Liang 等 19 人 / 清華大學、人民大學、Yale、微信 AI 等 |
| **發表** | ICLR 2024（arXiv:2307.16789，初版 2023/07） |
| **連結** | [arXiv abs](https://arxiv.org/abs/2307.16789) · [ar5iv HTML](https://ar5iv.labs.arxiv.org/html/2307.16789) · [code](https://github.com/OpenBMB/ToolBench) |
| **領域標籤** | `#tool-use` `#benchmark` `#multi-tool` `#agent` |
| **重要度** | ⭐⭐⭐⭐⭐ — 把工具使用推到「真實世界 16k+ API、多工具編排」規模，並提供資料集、模型、搜尋演算法與評測一整套 |
| **閱讀狀態** | 精讀 |

ToolLLM 是一套涵蓋**資料建構、模型訓練、評測**的完整工具使用框架。它用 ChatGPT 自動建出 **ToolBench** 指令微調資料集（涵蓋 RapidAPI Hub 上 16,464 個真實 API、49 個類別），訓練 LLaMA-2-7B 得到 **ToolLLaMA**；並提出 **DFSDT（深度優先搜尋決策樹）** 取代 ReAct 的線性推理鏈，讓模型能探索多條路徑與回溯；再用 **ToolEval** 自動評測。ToolLLaMA 的工具使用能力可媲美 ChatGPT，並能泛化到未見過的 API 與分布外的 APIBench。

## 問題與核心貢獻

- **痛點（Before）**：開源 LLM（LLaMA）工具使用能力弱，因為指令微調多聚焦基本語言任務、忽略工具使用領域；閉源模型（ChatGPT）則很強。過去的工具資料集規模小、API 少、多為單工具，缺乏「真實世界規模 + 多工具編排 + 妥善評測」。
- **核心貢獻（四件套 + 檢索器）**：
  - **ToolBench**：用 ChatGPT 自動建的工具使用指令微調資料集；
  - **ToolLLaMA**：微調 LLaMA-2-7B 得到的工具使用模型；
  - **DFSDT**：讓模型評估多條推理路徑、可回溯的搜尋演算法；
  - **ToolEval**：以 ChatGPT 為裁判的自動評測（pass rate + win rate）；
  - **神經 API 檢索器**：從上千工具中推薦相關 API。
- **成果（After）**：ToolLLaMA 平均 pass rate **66.7%**，與 ChatGPT（64.8%）相當、逼近 GPT-4（71.1%）；並能泛化到未見過的 API 與分布外資料集 APIBench。

![ToolLLM 整體框架](https://raw.githubusercontent.com/hi-im67xuanOuO/Agent-PaperNotes/main/notes/assets/toolllm-fig1-framework.png)

> **Figure 1**　整體框架。左：資料建構三階段（API 收集 → 指令生成 → 解答路徑標註）產出 ToolBench，用來 SFT 出 ToolLLaMA 與 API 檢索器。右：推論時檢索器先挑出相關 API，ToolLLaMA 多輪呼叫真實 RapidAPI，最後由 ToolEval 評測。

## 方法拆解

### ToolBench 資料集（三階段，全程用 ChatGPT 自動建）

- **① API 收集**：從 RapidAPI Hub 篩選 10,853 個工具（53,190 個 API）→ 保留 3,451 個功能完整的工具、**16,464 個 API**、涵蓋 **49 個類別**（金融、運動、天氣…），並整理每個 API 的說明、參數、範例回應。
- **② 指令生成**：用少量種子（12 個單工具、36 個多工具）few-shot 提示 ChatGPT，生成三種指令：
  - **I1 單工具**（87,413 筆）
  - **I2 同類別多工具**（84,815 筆）
  - **I3 同集合多工具**（25,251 筆）
- **③ 解答路徑標註**：讓 ChatGPT 透過多輪對話、實際呼叫 API，為每條指令搜出可行的 API 呼叫序列。最終得 **126,486 組指令-解答**、過程中執行了 **469,585 次真實 API 呼叫**。

![RapidAPI 階層與指令生成](https://raw.githubusercontent.com/hi-im67xuanOuO/Agent-PaperNotes/main/notes/assets/toolllm-fig2-dataset.png)

> **Figure 2**　左：RapidAPI 的「類別 → 工具 → API」階層，每個 API 附完整文件（名稱、參數、範例）。右：I1/I2/I3 三種指令從階層樹上取樣 API 組合，交給 ChatGPT 生成「指令 + 相關 API」。

### DFSDT（深度優先搜尋決策樹）

- **ReAct／CoT 的問題**：只走**一條線性路徑**；一旦某次 API 呼叫出錯，錯誤會**沿路傳播**、無法復原，最後只能放棄。
- **DFSDT 的做法**：把推理步驟展開成**決策樹**，可產生多個候選行動、評估後選最佳，遇到死路就**回溯**換條路；用前序遍歷（pre-order）控制成本。→ 探索空間更大、能從錯誤中恢復。

![DFSDT vs ReAct/CoT](https://raw.githubusercontent.com/hi-im67xuanOuO/Agent-PaperNotes/main/notes/assets/toolllm-fig3-dfsdt.png)

> **Figure 3**　左：CoT／ReAct 是單一線性鏈，第 3 步 Error 後一路失敗。中：DFSDT 展開多分支，遇 Error 回溯，最終走到 Success。右：實際多輪對話範例（呼叫失敗時 server not available，模型改走別條路）。

### ToolLLaMA 與檢索器

- **底模與訓練**：LLaMA-2 7B；context 由 4096 用位置內插擴到 **8192**；2 epochs、lr 5e-5、batch 64；保留 ChatGPT 的多輪對話格式。
- **神經 API 檢索器**：以 Sentence-BERT 做對比學習，從上千工具中推薦 top-5 相關 API，免把 16k API 全塞進 context。

### ToolEval（自動評測）

- **Pass Rate**：在預算內是否完成指令（與人工標註一致度 **87.1%**）。
- **Win Rate**：兩個解答路徑的成對比較（與人工一致度 **80.3%**），看資訊豐富度、事實性、推理細節、成本效率等。

## 實驗與結果

### 主結果：開源 7B 追平 ChatGPT

> **Table 1**　ToolEval pass rate（%，DFSDT 設定；I*-Inst 為未見指令子集）。

| 模型 | I1-Inst | I2-Inst | I3-Inst | 平均 |
|:---|:---:|:---:|:---:|:---:|
| Claude-2 | 20.5 | 17.0 | 28.0 | 22.6 |
| Text-Davinci-003 | 43.5 | 37.0 | 46.0 | 43.1 |
| ChatGPT | 54.5 | 75.0 | 62.0 | 64.8 |
| **ToolLLaMA（7B）** | **57.0** | **77.0** | **66.0** | **66.7** |
| GPT-4 | 60.0 | 79.5 | 84.0 | 71.1 |

### DFSDT vs ReAct：多步工具任務大幅提升

> **Table 2**　同一個 ChatGPT 下 ReAct 與 DFSDT 的 pass rate（%）。

| 指令類型 | ReAct | DFSDT | 增益 |
|:---|:---:|:---:|:---:|
| I1（單工具） | 37.8 | 54.5 | +16.7 |
| I2（同類別多工具） | 40.6 | 75.0 | +34.4 |
| I3（同集合多工具） | 27.6 | 62.0 | +34.4 |

### API 檢索器：遠勝傳統檢索

> **Table 3**　API 檢索 NDCG@5。

| 方法 | I1 | I2 | I3 |
|:---|:---:|:---:|:---:|
| BM25 | 19.7 | 11.0 | 20.4 |
| OpenAI Ada | 58.8 | 30.7 | 46.8 |
| **ToolLLM 神經檢索器** | **89.7** | **77.9** | **87.1** |

- **分布外泛化（APIBench）**：ToolLLaMA 在 Gorilla 的 APIBench 上有 zero-shot 泛化能力，幻覺率（6.48–15.70%）低於 Gorilla-ZS baseline（17.20–46.90%）。

## 可借鑑之處

- **可借用 / 可復用的方法點**：
  1. **用 ChatGPT 自動大規模建工具使用資料**（指令生成 + 解答路徑標註）——是做 tool-use SFT 資料的實用配方。
  2. **DFSDT**：給 agent「搜尋 + 回溯」而非單一線性鏈，在多步工具任務上增益顯著（I2/I3 +34pp）。
  3. **神經檢索器**：從上千工具挑 top-k，免把所有工具塞進 context，是擴展到大工具集的關鍵。
  4. 有對的資料，**開源 7B 也能在工具使用上追平 ChatGPT**。
- **適用場景**：建構與評測「多工具、多步驟」的真實世界 agent。

## 局限與延伸

1. **高度依賴 ChatGPT**：資料建構與 ToolEval 評測都以 ChatGPT 為核心，帶入其偏誤與成本；pass/win rate 由 LLM 裁判，非絕對 ground-truth。
2. **真實 API 不穩定**：RapidAPI 上的 API 會出錯、改版、下架，資料含噪。
3. **DFSDT 成本較高**：探索多分支等於更多 API 呼叫與 token。
4. **評測指標的侷限**：以「是否完成、路徑優劣」為主，不等同嚴格的答案正確性。

### 名詞速查

| 名詞 | 一句白話 |
|:---|:---|
| **ToolBench** | 用 ChatGPT 自動建、涵蓋 16k+ 真實 API 的工具使用指令微調資料集 |
| **ToolLLaMA** | 在 ToolBench 上微調 LLaMA-2-7B 得到的工具使用模型 |
| **DFSDT** | 深度優先搜尋決策樹，讓模型展開多條推理路徑並可回溯 |
| **ToolEval** | 以 ChatGPT 為裁判的自動評測（pass rate + win rate） |
| **I1 / I2 / I3** | 單工具／同類別多工具／同集合多工具三種指令 |
| **RapidAPI Hub** | 匯集大量真實 REST API 的平台，本文 API 的來源 |

## 歷史定位與展望

- **歷史定位**：
  - 接續 `[[Toolformer (Schick et al., 2023)]]`（少數固定工具）與 `[[Gorilla (Patil et al., 2023)]]`（1,645 個 ML API、單次呼叫正確性），把工具使用推到 **16k+ 真實 API、多工具、多步驟編排**；
  - 把 `[[ReAct (Yao et al., 2022)]]` 的線性推理，用類似 `[[Tree of Thoughts (Yao et al., 2023)]]` 的樹搜尋（DFSDT）升級。
- **延伸閱讀**：
  - 往前：`[[Toolformer (Schick et al., 2023)]]`、`[[Gorilla (Patil et al., 2023)]]`、`[[ReAct (Yao et al., 2022)]]`、`[[Tree of Thoughts (Yao et al., 2023)]]`
  - 往後：後續的 agent 工具使用 benchmark 與函式呼叫排行榜（如 Berkeley Function-Calling Leaderboard）
- **展望**（依論文與後續發展）：
  - 「大規模真實工具 + 搜尋式推理 + 自動評測」成為工具型 agent 的基礎範式；
  - 後續朝更真實穩定的工具環境、更嚴謹的評測、以及與長程 agent 規劃結合發展。
