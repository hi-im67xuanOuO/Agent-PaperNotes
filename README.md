# 📚 Agent-PaperNotes

> AI Agent 領域論文閱讀筆記。追求**深入淺出的骨架 × 客觀中立的說明 × 硬核的技術深度**——外行看得懂在解決什麼，同行看得到怎麼做到的。

每篇筆記以**擷取、翻譯、說明**論文內容為主，客觀中立、不摻個人主觀評價；多用項目符號，並盡量做到讓同行**大致復現**核心思路（狀態變數、演算法步驟、關鍵超參數、精確數字、ablation 結論）。

---

## 📝 筆記索引（[`notes/`](notes/)）

| # | 論文 | 年份 | 主題 | 評分 | 筆記 |
|---|---|---|---|---|---|
| 01 | **Chain-of-Thought Prompting** (Wei et al.) | 2022 | `#reasoning` `#emergent-ability` | ⭐⭐⭐⭐⭐ | [開啟](notes/2022-Chain-of-Thought%20Prompting%20Elicits%20Reasoning%20in%20Large%20Language%20Models.md) |
| 02 | **ReAct: Synergizing Reasoning and Acting** (Yao et al.) | 2022 | `#agent` `#tool-use` | ⭐⭐⭐⭐⭐ | [開啟](notes/2022-ReAct%20-%20Synergizing%20Reasoning%20and%20Acting%20in%20Language%20Models.md) |
| 03 | **Self-Consistency** (Wang et al.) | 2022 | `#reasoning` `#decoding` | ⭐⭐⭐⭐⭐ | [開啟](notes/2022-Self-Consistency%20Improves%20Chain%20of%20Thought%20Reasoning%20in%20Language%20Models.md) |
| 04 | **Tree of Thoughts** (Yao et al.) | 2023 | `#reasoning` `#planning` `#search` | ⭐⭐⭐⭐⭐ | [開啟](notes/2023-Tree%20of%20Thoughts%20-%20Deliberate%20Problem%20Solving%20with%20Large%20Language%20Models.md) |
| 05 | **Toolformer** (Schick et al.) | 2023 | `#tool-use` `#self-supervised` | ⭐⭐⭐⭐⭐ | [開啟](notes/2023-Toolformer%20-%20Language%20Models%20Can%20Teach%20Themselves%20to%20Use%20Tools.md) |
| 06 | **Gorilla** (Patil et al.) | 2023 | `#tool-use` `#api` `#retrieval` | ⭐⭐⭐⭐ | [開啟](notes/2023-Gorilla%20-%20Large%20Language%20Model%20Connected%20with%20Massive%20APIs.md) |
| 07 | **ToolLLM / ToolBench** (Qin et al.) | 2023 | `#tool-use` `#benchmark` `#multi-tool` | ⭐⭐⭐⭐⭐ | [開啟](notes/2023-ToolLLM%20-%20Facilitating%20Large%20Language%20Models%20to%20Master%2016000%2B%20Real-world%20APIs.md) |

> 清單會持續往下長；讀一篇、補一列。

## 🗺️ 閱讀路線圖

依「地基 → 應用 → 綜述」的順序往下讀，逐步補齊 AI Agent 的全局視野：

1. **地基（Reasoning）**：Chain-of-Thought → ReAct → Self-Consistency → Tree of Thoughts
2. **Tool Use**：Toolformer → Gorilla → ToolLLM
3. **Reflection**：Reflexion → Self-Refine
4. **Memory & 終身學習**：Generative Agents → Voyager → MemGPT
5. **Planning**：Plan-and-Solve → LATS
6. **Multi-Agent**：CAMEL → AutoGen → MetaGPT → ChatDev
7. **Applications**：SWE-agent → WebArena → AI Scientist
8. **Survey**：Xi et al. Survey → CoALA（把上面全部串起來）

---

## 🧭 每篇筆記的結構

每篇以六個描述式段落呈現：

- **問題與核心貢獻**：這篇之前的痛點、核心洞見、提出什麼、關鍵成果數字。
- **方法拆解**：核心方法、輸入/輸出、關鍵機制，配架構圖。
- **實驗與結果**：實驗設定、主結果（圖＋表＋倍數）、ablation、穩健性。
- **可借鑑之處**：可復用的方法點與適用場景。
- **局限與延伸**：論文自陳或後續研究指出的局限、名詞速查表。
- **歷史定位與展望**：繼承誰、啟發誰、後續發展方向。

圖片一律用論文原圖（附白話圖說）；若為自繪圖表會標明「自繪」並註明數據來源。相關論文用 `[[論文名]]` 串成知識網。

## 📂 目錄結構

```text
Agent-PaperNotes/
├─ README.md                                     # 你正在看的這頁
└─ notes/                                         # 論文筆記（檔名 = 年份-論文全名）
   ├─ 2022-Chain-of-Thought Prompting Elicits ….md
   ├─ 2022-ReAct - Synergizing Reasoning and ….md
   └─ assets/                                     # 各篇筆記引用的圖
```
