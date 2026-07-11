# 📚 Agent-PaperNotes

> 我的 AI Agent 領域論文閱讀筆記。追求**深入淺出的骨架 × 有觀點的部落格語氣 × 硬核的技術深度**——外行看得懂在解決什麼，同行看得到怎麼做到的。

每篇筆記都盡量做到：能讓同行**大致復現**核心思路（給狀態變數、演算法步驟、關鍵超參數、精確數字、ablation 結論），並附上**歷史定位**與**個人判斷**，而不是中立摘要。

---

## 📝 筆記索引（[`notes/`](notes/)）

| # | 論文 | 年份 | 主題 | 我的評分 | 筆記 |
|---|---|---|---|---|---|
| 01 | **Chain-of-Thought Prompting** (Wei et al.) | 2022 | `#reasoning` `#emergent-ability` | ⭐⭐⭐⭐⭐ | [開啟](notes/2022-Chain-of-Thought-Prompting.md) |

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

## 🧭 這些筆記怎麼讀

- **🟢 第一層 是什麼**：30 秒電梯簡報，講給沒讀過的人聽。
- **🟢 第二層 怎麼做**：方法拆解、技術細節、關鍵設計選擇（為什麼）、精確實驗數字與 ablation。
- **🟢 第三層 對我而言**：可借用的點子、金句、相關論文連結。
- **🔵 深入欄位**：批判性思考、名詞速查、歷史定位與展望。

圖片一律用論文原圖（附白話圖說）；若為自繪圖表會標明「自繪」並註明數據來源。相關論文用 `[[論文名]]` 串成知識網。

## 📂 目錄結構

```text
Agent-PaperNotes/
├─ README.md                                # 你正在看的這頁
└─ notes/                                    # 論文筆記
   ├─ 2022-Chain-of-Thought-Prompting.md
   └─ assets/                                # 各篇筆記引用的圖
```
