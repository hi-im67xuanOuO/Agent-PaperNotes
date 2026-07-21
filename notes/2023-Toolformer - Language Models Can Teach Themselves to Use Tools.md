# 🔧 Toolformer: Language Models Can Teach Themselves to Use Tools（用自監督篩選 API 呼叫，讓 LLM 自己學會何時、如何用工具）

> #100篇經典挑戰 No.05 ｜ ⭐⭐⭐⭐⭐ ｜ Schick et al., 2023（NeurIPS 2023）

| 欄位 | 內容 |
|:---|:---|
| **作者 / 機構** | Timo Schick、Jane Dwivedi-Yu、Roberto Dessì、Roberta Raileanu、Maria Lomeli、Luke Zettlemoyer、Nicola Cancedda、Thomas Scialom / Meta AI Research |
| **發表** | NeurIPS 2023（arXiv:2302.04761，初版 2023/02） |
| **連結** | [arXiv abs](https://arxiv.org/abs/2302.04761) · [ar5iv HTML](https://ar5iv.labs.arxiv.org/html/2302.04761) |
| **領域標籤** | `#tool-use` `#self-supervised` `#api` `#finetuning` |
| **重要度** | ⭐⭐⭐⭐⭐ — 工具使用的開山之作：用自監督讓 LLM 自己學會呼叫外部 API，免大量人工標註 |
| **閱讀狀態** | 精讀 |

Toolformer 讓語言模型透過**自監督**學會決定「何時、如何」呼叫外部 API（計算機、問答、搜尋、翻譯、日曆）。核心做法：先讓模型自己在文本中標註候選 API 呼叫，只保留「能降低後續 token 預測損失」的呼叫，再用這批資料微調模型。以 GPT-J（6.7B）為底，zero-shot 表現在多個任務上追平甚至勝過大 25 倍的 GPT-3（175B），且不損害原本的語言建模能力。

## 問題與核心貢獻

- **痛點（Before）**：LLM 擅長 few-shot，卻在**算術、事實查詢、當前日期、低資源語言**等基本功上出錯。過去要教模型用工具，多半需要**大量人工標註**「該在哪裡呼叫什麼工具」，或只針對單一工具特製。
- **核心洞見（Aha）**：讓模型**自己教自己**——用一個自監督準則判斷 API 呼叫是否真的有用：把「呼叫 + 結果」插進去後，若能**降低模型對後續文字的預測損失（perplexity）**，就代表這個呼叫有幫助、值得保留。如此完全不需要人工標「何時該呼叫」。
- **核心貢獻**：
  - 提出「取樣 → 執行 → 依損失篩選」的自監督資料生成流程；
  - 用內嵌於文本的 API 呼叫標記，微調後模型能在推論時自主呼叫工具；
  - 整合 5 種工具，並顯示工具使用的效益隨模型規模浮現。
- **成果（After）**：GPT-J 6.7B 版 Toolformer 在 LAMA T-REx（53.5 vs GPT-3 39.8）、ASDiv（40.4 vs GPT-3 14.0）等 zero-shot 任務勝過 175B 的 GPT-3，且 WikiText perplexity 幾乎不變。

![Toolformer API 呼叫範例](https://raw.githubusercontent.com/hi-im67xuanOuO/Agent-PaperNotes/main/notes/assets/toolformer-fig1-examples.png)

> **Figure 1**　模型自主在生成文字中插入不同 API 呼叫並用回傳結果：QA 問出版者、Calculator 算 400/1400、MT 翻譯 tortuga、WikiSearch 查 Brown Act。呼叫以 `[工具(輸入) → 結果]` 形式嵌在文字裡。

## 方法拆解

- **API 呼叫格式**：以特殊標記 `<API> 工具(輸入) → 結果 </API>` 表示（實作上用 `[工具(輸入) → 結果]`），直接嵌入文字序列，**不需擴充詞表**。
- **三步驟自監督資料生成**：
  - **Step 1 取樣（Sample）**：用少量手寫範例 few-shot 提示模型，在文本中「模型認為可能要呼叫 API」的位置（機率超過門檻 τ_s）取樣最多 m 個候選呼叫。
  - **Step 2 執行（Execute）**：把每個候選呼叫送去對應後端執行，取得實際結果。
  - **Step 3 篩選（Filter）**：只保留能降低後續 token 加權交叉熵損失達門檻 τ_f（通常 =1.0）的呼叫，即 **L⁻ − L⁺ ≥ τ_f**——其中 L⁺ 是「有呼叫+結果」的損失，L⁻ 是「完全不呼叫」或「有呼叫但不給結果」兩者的較小損失。
- **微調**：把篩選後的增強資料集 C\*（原文＋保留的 API 呼叫）用**標準語言建模目標**微調 GPT-J；因分布貼近預訓練，保住通用能力。
- **底層模型**：GPT-J（6.7B，於 The Pile 預訓練）。

![三步驟自監督流程](https://raw.githubusercontent.com/hi-im67xuanOuO/Agent-PaperNotes/main/notes/assets/toolformer-fig2-pipeline.png)

> **Figure 2**　以「Pittsburgh is also known as the Steel City」為例：① 取樣候選 API 呼叫（問 Pittsburgh 的別名／所在國）→ ② 執行取得結果（Steel City／United States）→ ③ 依損失篩選，只有「問別名」那個呼叫降低了損失而被保留 → 產生含 API 呼叫的訓練樣本。

### 整合的 5 種工具

| 工具 | 功能 | 後端 |
|:---|:---|:---|
| **Question Answering** | 回答事實型問題 | Atlas（在 Natural Questions 微調） |
| **Calculator** | 四則運算 | Python 運算式求值 |
| **Wikipedia Search** | 維基全文檢索 | BM25 檢索（KILT 索引） |
| **Machine Translation** | 非英文翻成英文 | NLLB（600M，200 語言） |
| **Calendar** | 回傳當前日期 | 決定性查詢 |

## 實驗與結果

- **設定**：全部 **zero-shot**；比較 Toolformer（6.7B）、GPT-J（6.7B，無工具）、GPT-3（175B）。

### 事實補全（LAMA）：6.7B 勝過 175B GPT-3

> **Table 1**　LAMA 事實補全準確率（%）。

| 模型 | SQuAD | Google-RE | T-REx |
|:---|:---:|:---:|:---:|
| GPT-J（無工具） | 17.8 | 4.9 | 31.9 |
| GPT-3（175B） | 26.8 | 7.0 | 39.8 |
| **Toolformer（6.7B）** | **33.8** | **11.5** | **53.5** |

### 數學推理：大幅超越，得益於計算機

> **Table 2**　數學任務準確率（%，首數字匹配）。

| 模型 | ASDiv | SVAMP | MAWPS |
|:---|:---:|:---:|:---:|
| GPT-J（無工具） | 7.5 | 5.2 | 9.9 |
| GPT-3（175B） | 14.0 | 10.0 | 19.8 |
| **Toolformer（6.7B）** | **40.4** | **29.4** | **44.0** |

### 問答（QA）：勝 GPT-J，但未超越 GPT-3

> **Table 3**　開放式問答準確率（%）；QA 工具本身能力有限，故未追上 GPT-3。

| 模型 | WebQS | NQ | TriviaQA |
|:---|:---:|:---:|:---:|
| GPT-J（無工具） | 18.5 | 12.8 | 43.9 |
| **Toolformer（6.7B）** | 26.3 | 17.7 | 48.8 |
| GPT-3（175B） | **29.0** | **22.6** | **65.9** |

- **多語言 QA（MLQA，西班牙文）**：GPT-J 15.2% → Toolformer 20.6%；依語言不同，工具使用率 63.8%–94.9%。
- **時序推理**：Dateset 上日曆工具被呼叫 54.8%，帶來 +23.4 分（GPT-J 3.9 → Toolformer 27.3）。
- **不損害語言建模**：WikiText perplexity GPT-J 9.9 →（關閉工具）Toolformer 10.3，差 0.4 屬雜訊等級。

![工具效益隨規模浮現](https://raw.githubusercontent.com/hi-im67xuanOuO/Agent-PaperNotes/main/notes/assets/toolformer-fig4-scaling.png)

> **Figure 3**　工具效益的規模曲線（LAMA／Math／QA）。藍＝Toolformer、粉＝關閉工具、紫虛線＝GPT-3。小模型（GPT-2 small/medium）用不用工具差不多；約 775M 以上工具才明顯有益，且到 6.7B 差距仍大。

## 可借鑑之處

- **可借用 / 可復用的方法點**：
  1. **「這個工具呼叫有沒有降低損失」** 是一個免人工標註、可自動判斷「工具是否有用」的訊號，可移植到其他「要不要用某個外部能力」的決策。
  2. 把工具呼叫**內嵌成文本標記再微調**，模型就能在推論時自主呼叫——工具能力寫進權重、不靠 prompt。
  3. **小模型 + 工具**可在知識／計算密集任務上追上大模型，是省成本的路線。
- **適用場景**：想把確定性或外部能力（計算、檢索、日期、翻譯）補進一個基礎 LM。

## 局限與延伸

1. **不支援鏈式／組合式工具使用**：每個 API 呼叫是**獨立**取樣與篩選的，模型無法「先看工具結果、再決定下一步呼叫」（論文自陳），這正是 `[[ReAct (Yao et al., 2022)]]` 的互動迴圈所補足的。
2. **沒有回饋迴圈**：工具結果不會回頭影響推理再觸發新呼叫。
3. **樣本成本**：要建訓練資料需大量取樣＋執行候選呼叫。
4. **工具集在訓練時固定**：新增工具需重新標註與微調。

### 名詞速查

| 名詞 | 一句白話 |
|:---|:---|
| **Toolformer** | 用自監督學會呼叫外部 API 的語言模型 |
| **Self-supervised filtering** | 用「呼叫是否降低後續 token 損失」自動篩選有用的 API 呼叫，免人工標註 |
| **API 呼叫標記** | 內嵌文字的 `[工具(輸入) → 結果]`，微調後模型會自主生成 |
| **Perplexity（困惑度）** | 模型對文字的預測損失，越低代表越「不意外」 |
| **GPT-J** | 6.7B 參數的開源自迴歸模型（本文的底模） |

## 歷史定位與展望

- **歷史定位**：
  - 工具使用的**開山之作**之一；相較先前需重度監督的工具增強（如 WebGPT），Toolformer 用自監督把「何時該呼叫」的標註成本拿掉。
  - 與 `[[ReAct (Yao et al., 2022)]]` 互補：ReAct 在**推論時用提示**交錯推理與行動（不微調）；Toolformer 把工具呼叫**用自監督微調寫進權重**。
- **延伸閱讀**：
  - 往前：`[[ReAct (Yao et al., 2022)]]`、`[[Chain-of-Thought Prompting (Wei et al., 2022)]]`
  - 往後：`[[Gorilla (Patil et al., 2023)]]`（大規模 API 呼叫）、`[[ToolLLM / ToolBench (Qin et al., 2023)]]`（多工具編排）、以及各家的 function calling
- **展望**（依論文與後續發展）：
  - 「工具使用」自此成為 LLM 的核心能力，後續發展出 function calling 標準、大規模 API 選擇（Gorilla）與多工具編排（ToolLLM）；
  - 主要延伸方向：組合式／互動式工具使用、擴充工具集不需重訓、降低資料生成成本。
