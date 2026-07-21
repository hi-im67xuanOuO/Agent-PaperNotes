# 🦍 Gorilla: Large Language Model Connected with Massive APIs（微調 LLaMA-7B 精準生成 API 呼叫，並以檢索感知訓練適應文件變動、大幅降低幻覺）

> #100篇經典挑戰 No.06 ｜ ⭐⭐⭐⭐ ｜ Patil et al., 2023（arXiv 預印本；後續 NeurIPS 2024）

| 欄位 | 內容 |
|:---|:---|
| **作者 / 機構** | Shishir G. Patil、Tianjun Zhang、Xin Wang、Joseph E. Gonzalez / UC Berkeley、Microsoft Research |
| **發表** | arXiv:2305.15334（初版 2023/05） |
| **連結** | [arXiv abs](https://arxiv.org/abs/2305.15334) · [ar5iv HTML](https://ar5iv.labs.arxiv.org/html/2305.15334) · [專案頁](https://gorilla.cs.berkeley.edu/) |
| **領域標籤** | `#tool-use` `#api` `#retrieval` `#hallucination` |
| **重要度** | ⭐⭐⭐⭐ — 把工具使用推到「大規模真實 API」規模，並正面處理 API 呼叫的正確性與幻覺問題 |
| **閱讀狀態** | 精讀 |

Gorilla 是微調 LLaMA-7B 的模型，專門**生成正確的 API 呼叫**。作者蒐集 TorchHub、TensorHub、HuggingFace 三大機器學習套件庫共 1,645 個真實 API 組成 **APIBench** 資料集，用來微調模型；並提出**檢索感知訓練（retriever-aware training）**，讓模型學會使用提供的文件、進而在測試時適應 API 文件的版本變動。結果在 zero-shot 寫 API 呼叫上超越 GPT-4，並把幻覺率從約 37% 降到約 7%。

## 問題與核心貢獻

- **痛點（Before）**：即使是 GPT-4，寫 API 呼叫也常出錯——**參數給錯**，或**憑空捏造不存在的 API（幻覺）**。加上 API 會頻繁改版，模型記憶的用法很快過時。
- **核心洞見（Aha）**：
  1. 在**真實 API 文件**上做指令微調，讓小模型專精於「寫對 API 呼叫」；
  2. **檢索感知訓練**——訓練時就把「參考文件」放進 prompt，模型學會讀文件，測試時只要換上最新文件就能**適應改版**；
  3. 用 **AST（抽象語法樹）子樹匹配**精確定義並量測幻覺，區分「捏造的 API」與「用錯的真 API」。
- **核心貢獻**：Gorilla 模型、APIBench 資料集、AST 幻覺評估法、檢索感知訓練。
- **成果（After）**：zero-shot 整體準確率贏 GPT-4 約 **+20.43%**；TorchHub 幻覺率 **6.98%**（GPT-4 為 36.55%）。

![GPT-4／Claude／Gorilla 對比](https://raw.githubusercontent.com/hi-im67xuanOuO/Agent-PaperNotes/main/notes/assets/gorilla-fig1-examples.png)

> **Figure 1**　同一個需求（用 Torch Hub 找語音轉文字 API）：GPT-4 捏造參數（Hallucinate）、Claude 選錯函式庫（Wrong library）、Gorilla 產生完全正確、可直接執行的 API 呼叫。

## 方法拆解

- **APIBench 資料集**：TorchHub 95 個、TensorFlow Hub v2 626 個、HuggingFace 925 個（每領域取前 20 熱門），共 **1,645 個 API**。每個 API 用 **Self-Instruct**（以 GPT-4 生成、僅用 18 個手寫範例引導、每庫 6 個）產生 10 組合成的「使用者指令 ↔ API」配對，共 **16,450 筆**。API 文件轉成 JSON（domain、api_call、api_arguments、example_code…）。
- **底層模型與訓練**：LLaMA-7B 監督式指令微調（lr 2e-5、batch 64、5 epochs、序列長 2048、8×A100）。
- **檢索感知訓練（retriever-aware）**：訓練時在指令前加上「Use this API documentation for reference: <檢索到的 API 文件 JSON>」，讓模型學會**解析並使用提供的文件**；測試時換上更新後的文件即可適應改版。變體：zero-shot（無檢索）／Oracle（給正確文件）／BM25／GPT-index。
- **AST 子樹匹配（量測幻覺）**：把生成的 API 呼叫解析成抽象語法樹，檢查它是否為資料庫中某個 API 的**子樹**（必填參數要對、可忽略選填如 `pretrained=True`）。
  - **幻覺（Hallucination）**：不是任何 API 的子樹——等於呼叫一個**想像出來的工具**；
  - **錯誤（Error）**：呼叫了**真實 API 但用錯**（如給錯模型名）。

![Gorilla 系統架構](https://raw.githubusercontent.com/hi-im67xuanOuO/Agent-PaperNotes/main/notes/assets/gorilla-fig2-architecture.png)

> **Figure 2**　上排（訓練資料）：抓三大套件庫文件 → 轉 JSON → Self-Instruct 生成 16,450 組指令-API 配對 → 訓練 Gorilla-7B。下排（推論）：使用者需求可走「檢索器查 API 資料庫、把參考文件一起餵給 Gorilla」或直接 zero-shot，最後輸出可執行的 API 呼叫。

## 實驗與結果

- **設定**：zero-shot 寫 API 呼叫；對照 GPT-4、GPT-3.5、Claude、原始 LLaMA-7B。

### Zero-shot 準確率：小模型贏過 GPT-4

> **Table 1**　zero-shot（無檢索）API 呼叫準確率（%）。

| 模型 | TorchHub | HuggingFace | TensorFlow Hub |
|:---|:---:|:---:|:---:|
| **Gorilla（7B）** | **59.13** | **71.68** | **83.79** |
| GPT-3.5 | 48.38 | 16.81 | 41.75 |
| GPT-4 | 38.70 | 19.80 | 18.20 |
| Claude | 18.81 | 6.19 | 9.19 |
| LLaMA-7B | 0 | 0 | 0 |

- Gorilla 整體較 GPT-4 高約 **+20.43%**、較 GPT-3.5 高約 +10.75%。

### 幻覺率：從 ~37% 降到 ~7%

> **Table 2**　zero-shot 幻覺率（%，越低越好）。

| 模型 | TorchHub | HuggingFace | TensorFlow Hub |
|:---|:---:|:---:|:---:|
| **Gorilla（7B）** | **6.98** | **10.95** | **5.40** |
| GPT-3.5 | 18.81 | 35.73 | 47.88 |
| GPT-4 | 36.55 | 37.16 | 78.65 |
| Claude | 65.59 | 77.65 | 88.46 |

- GPT-4 在 TensorFlow Hub 幻覺高達 78.65%，常捏造不存在的模型名或 GitHub repo。

### 搭配 Oracle 檢索器：與閉源模型接近

> **Table 3**　給正確文件（Oracle retriever）時的準確率（%）。

| 模型 | TorchHub | HuggingFace | TensorFlow Hub |
|:---|:---:|:---:|:---:|
| **Gorilla（7B）** | **67.20** | **91.26** | 94.16 |
| GPT-3.5 | 66.31 | 89.71 | **95.03** |
| GPT-4 | 66.12 | 85.07 | 55.91 |
| Claude | 63.44 | 77.21 | 74.74 |

![準確率 vs 幻覺散點圖](https://raw.githubusercontent.com/hi-im67xuanOuO/Agent-PaperNotes/main/notes/assets/gorilla-fig3-acc-vs-hallucination.png)

> **Figure 3**　四種設定（0-shot／BM25／GPT／Oracle 檢索）下各模型的準確率 vs 幻覺散點。Gorilla（綠）在 zero-shot 位於左上角（高準確、低幻覺）；有了好的檢索器差距縮小。

- **值得注意**：加檢索**不一定**更好——用弱檢索器（BM25）時，檢索感知訓練的模型反而可能比 zero-shot 差；唯有搭配好的（Oracle）檢索才發揮最大效益。
- **適應測試時文件變動**：文件改版（如 ResNet-50 → ResNet-101、模型倉庫從 pytorch/vision 換到 NVIDIA/DeepLearningExamples）時，Gorilla 無需重訓即可跟上。
- **約束感知選擇**：能依「參數量 <10M 且 ImageNet 準確率 ≥70%」這類多重約束挑選合適模型。

## 可借鑑之處

- **可借用 / 可復用的方法點**：
  1. **檢索感知微調**：訓練時就讓模型「讀參考文件」，測試時換文件即可適應工具改版，不必重訓。
  2. **AST 子樹匹配**是嚴謹、可自動化的「API 幻覺」偵測法，並清楚區分「捏造工具」與「用錯真工具」。
  3. 在**明確定義的窄能力**上，微調過的小模型可勝過通用大模型。
- **適用場景**：要在大量且會改版的 API/工具集上，建立可靠的工具呼叫能力。

## 局限與延伸

1. **檢索不一定有幫助**：檢索感知模型搭配弱檢索器可能比 zero-shot 更差，效益高度依賴檢索品質。
2. **領域侷限**：只涵蓋三大 ML 套件庫的 API，對任意領域 API 的泛化未在此驗證。
3. **靜態快照**：APIBench 為特定時間點的資料集，需持續維護。
4. **單次 API 呼叫**：聚焦「選對並寫對一個 API」，不涉及多工具、多步驟編排（那是 ToolLLM 的範圍）。

### 名詞速查

| 名詞 | 一句白話 |
|:---|:---|
| **APIBench** | 由 TorchHub/TensorHub/HuggingFace 共 1,645 個真實 API 組成的評測資料集 |
| **Retriever-aware training** | 訓練時把參考文件放進 prompt，讓模型學會讀文件、適應文件變動 |
| **AST 子樹匹配** | 把生成的 API 呼叫解析成語法樹，檢查是否為某真實 API 的子樹，用以判定幻覺 |
| **Hallucination（幻覺）** | 呼叫一個資料庫中不存在、憑空想像的 API |
| **Self-Instruct** | 用 LLM 依少量範例自動生成大量「指令-回應」訓練資料的方法 |

## 歷史定位與展望

- **歷史定位**：
  - 接在 `[[Toolformer (Schick et al., 2023)]]` 之後，把工具使用從「少數固定工具」推到「上千個真實 API」規模，並正面處理**正確性、幻覺、文件改版**。
  - 與 `[[Toolformer (Schick et al., 2023)]]` 的差異：Toolformer 用自監督決定「何時呼叫少數工具」；Gorilla 聚焦「在大規模 API 中選對、寫對一個呼叫」，並用檢索適應改版。
- **延伸閱讀**：
  - 往前：`[[Toolformer (Schick et al., 2023)]]`、`[[ReAct (Yao et al., 2022)]]`
  - 往後：`[[ToolLLM / ToolBench (Qin et al., 2023)]]`（16k+ 真實 API、多工具編排）、各家 function calling
- **展望**（依論文與後續發展）：
  - 「檢索增強的工具選擇 + 跟上文件變動」成為實務上可靠工具呼叫的基礎；
  - 後續朝更大工具集、多步驟工具編排、以及與 agent 迴圈結合發展。
