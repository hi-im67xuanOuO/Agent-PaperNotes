# 🌳 Tree of Thoughts: Deliberate Problem Solving with Large Language Models（把推理從線性鏈擴展成可搜尋、可回溯的思路樹）

> #100篇經典挑戰 No.04 ｜ ⭐⭐⭐⭐⭐ ｜ Yao et al., 2023（NeurIPS 2023）

| 欄位 | 內容 |
|:---|:---|
| **作者 / 機構** | Shunyu Yao、Dian Yu、Jeffrey Zhao、Izhak Shafran、Thomas L. Griffiths、Yuan Cao、Karthik Narasimhan / Princeton、Google DeepMind |
| **發表** | NeurIPS 2023（arXiv:2305.10601，初版 2023/05） |
| **連結** | [arXiv abs](https://arxiv.org/abs/2305.10601) · [ar5iv HTML](https://ar5iv.labs.arxiv.org/html/2305.10601) · [code](https://github.com/princeton-nlp/tree-of-thought-llm) |
| **領域標籤** | `#reasoning` `#planning` `#search` `#prompting` |
| **重要度** | ⭐⭐⭐⭐⭐ — 把 LLM 推理從「線性鏈」擴展成「可搜尋、可回溯的樹」，是 planning／search 這條線的概念源頭 |
| **閱讀狀態** | 精讀 |

Tree of Thoughts（ToT）把 Chain-of-Thought 一般化：不再只走一條線性推理鏈，而是把問題求解視為**在「思路（thought）」組成的樹上搜尋**。LLM 同時扮演兩個角色——**生成候選思路**與**自我評估各個狀態**，再配合 BFS／DFS 搜尋，能做前瞻（lookahead）與回溯（backtracking）。以 GPT-4 為底，在 Game of 24 上把 CoT 的 4% 提升到 74%。

## 問題與核心貢獻

- **痛點（Before）**：LLM 推論是 token-level、由左至右的單向決策；CoT 只走一條線性鏈，**無法探索、無法前瞻、無法回溯**——早期一步走錯，整條鏈就毀了。Self-Consistency 雖取樣多條完整鏈再投票，但仍沒有「中途探索與回溯」。
- **核心洞見（Aha）**：把求解視為**在思路樹上的搜尋**（節點＝部分解），用 **LLM 自己當啟發式**——既生成候選思路、也評估每個狀態好不好，於是能刻意地前瞻與回溯，做出全域選擇。
  - 類比：解謎時在草稿紙上分岔嘗試幾種走法、覺得某條死路就劃掉回頭，而不是一條路走到黑。
- **核心貢獻**：
  - 提出通用的 ToT 框架（四個設計元件，見下）；
  - 用 LLM 本身當「生成器 + 評估器」，不需寫死規則（如 DeepBlue）或另訓模型（如 AlphaGo）；
  - 設計三個需要規劃／搜尋的新任務並驗證大幅增益。
- **成果（After）**：Game of 24（GPT-4）CoT 僅 4% → **ToT 74%**。

![IO / CoT / CoT-SC / ToT 對比](https://raw.githubusercontent.com/hi-im67xuanOuO/Agent-PaperNotes/main/notes/assets/tot-fig1-schematic.png)

> **Figure 1**　四種推理結構對比。(a) IO 直接輸入→輸出；(b) CoT 線性鏈；(c) CoT-SC 取樣多條鏈再多數決；(d) ToT 在思路樹上分岔探索（綠＝有希望、紅＝剪掉），可前瞻與回溯。

## 方法拆解

ToT 需要決定四個元件：

- **① 思路分解（Thought Decomposition）**：定義「一個思路」有多大——要小到 LLM 能生成多樣候選，又大到能被有意義地評估。三個任務分別是：一個等式（Game of 24）、一段寫作計畫或段落（Creative Writing）、一個填字答案（Crosswords）。
- **② 思路生成器 G**：
  - **Sample（取樣）**：獨立取樣多條 CoT 式思路——思路空間豐富時用；
  - **Propose（提議）**：用 proposal prompt 依序提出候選——空間受限時用，避免重複。
- **③ 狀態評估器 V**（用 LLM 自評，只要「大致有用」即可）：
  - **Value（獨立賦值）**：對每個狀態給分或分類（sure／likely／impossible），可含少量前瞻模擬；
  - **Vote（比較投票）**：讓 LLM 在多個候選中選出最好的。
- **④ 搜尋演算法**：
  - **BFS**：每步保留 b 個最有希望的狀態（適合淺樹 T≤3、小 beam b≤5）；
  - **DFS**：先深入最有希望的路徑，遇到評估為不可能（V ≤ 門檻）就剪枝、回溯。

- **關鍵超參數**：Game of 24 / Creative Writing 用 BFS，b∈{1,5}、深度 T≤3、每節點 k=5 候選、每個思路取 3 次評估；Mini Crosswords 用 DFS，k=5 提議、最多 100 步、含剪枝門檻。
- **底層模型**：全部用 **GPT-4**（temperature 0.7）。

![Game of 24 的生成與評估](https://raw.githubusercontent.com/hi-im67xuanOuO/Agent-PaperNotes/main/notes/assets/tot-fig2-game24-pipeline.png)

> **Figure 2**　Game of 24 的一步。(a) Propose Prompt：給現有數字，LLM 提議下一步可能的算式（如 4+9=13, left: 10 13）。(b) Value Prompt：LLM 評估「剩下的數字還能不能湊到 24」並標記 sure／likely／impossible，據此剪枝。

## 實驗與結果

三個需要規劃／搜尋的任務（皆 GPT-4）：Game of 24（4 個數字湊 24）、Creative Writing（用 4 個指定句尾寫 4 段連貫短文）、Mini Crosswords（5×5 填字）。

### Game of 24：CoT 4% → ToT 74%

> **Table 1**　Game of 24（100 題難題，GPT-4）各方法成功率。

| 方法 | 成功率 |
|:---|:---:|
| IO prompt | 7.3% |
| CoT prompt | 4.0% |
| CoT-SC（k=100） | 9.0% |
| ToT（b=1） | 45% |
| **ToT（b=5）** | **74%** |

- 即使給對照組 100 次機會取最好：IO best-of-100 = 33%、CoT best-of-100 = 49%，仍低於 ToT。

### Mini Crosswords：搜尋帶來數倍提升

> **Table 2**　Mini Crosswords（20 局，GPT-4）三種粒度的成功率。

| 粒度 | IO | CoT | ToT | ToT（+best state） |
|:---|:---:|:---:|:---:|:---:|
| 字母（Letters） | 38.7% | 40.6% | 78% | 82.4% |
| 單字（Words） | 14% | 15.6% | 60% | 67.5% |
| 整局（Games） | 0 | 1 局 | 20% | 35% |

- **Creative Writing**（連貫度 1–10，GPT-4 評分）：IO 6.19、CoT 6.93、**ToT 7.56**；人工評比 ToT 勝出 41/100，CoT 21/100（38 平手）。

![成功率隨搜尋節點數](https://raw.githubusercontent.com/hi-im67xuanOuO/Agent-PaperNotes/main/notes/assets/tot-fig3-scaling.png)

> **Figure 3**　Game of 24 成功率 vs 走訪的樹節點數。ToT（綠虛線，b=1…5）在相同節點預算下明顯高於 IO、CoT 的 best-of-k 曲線。

### Ablation：剪枝、回溯、生成品質都關鍵

- **剪枝 / 回溯**（Mini Crosswords）：拿掉剪枝或回溯，字母／單字／整局成功率都大幅下滑（例如 -backtrack：54.6% → 20% → 5%），證明「能剪枝、能回頭」是 ToT 的關鍵。
- **瓶頸在生成、不在評估**：拆開誰用 GPT-4——

> **Table 3**　Game of 24：把「生成」與「評估」分派給不同模型，看瓶頸在哪。

| 生成 / 評估 | 成功率 |
|:---|:---:|
| GPT-4 生成 + GPT-3.5 評估 | 64% |
| GPT-3.5 生成 + GPT-4 評估 | 31% |
| GPT-4 生成 + GPT-4 評估（完整） | 74% |

- 生成端換成弱模型掉最多（→31%），顯示**思路生成品質**是主要瓶頸。

## 可借鑑之處

- **可借用 / 可復用的方法點**：
  1. 當任務需要**探索／前瞻／回溯**時，把它結構化成「思路樹搜尋」，用 LLM 當生成器 + 評估器。
  2. LLM 自評（value 給分 / vote 比較）可當便宜的搜尋啟發式，只要「大致有用」即可。
  3. 依樹的深淺與分支選搜尋法：淺樹小分支用 BFS，需要深入與回溯用 DFS + 剪枝。
  4. 思路空間豐富用 sample、受限用 propose。
- **適用場景**：中間狀態**可被評估**、需要規劃或搜尋的任務。

## 局限與延伸

1. **推理成本高**：每個節點都要多次呼叫 LLM（生成 k 個 + 評估），總呼叫數遠高於 CoT／CoT-SC。
2. **需任務特製**：思路分解、生成／評估 prompt、搜尋設定都要針對任務設計，非即插即用。
3. **受限於生成品質**：換成較弱的底模（GPT-3.5）Game of 24 僅 19%（GPT-4 為 74%）。
4. **需可評估的狀態**：對「難以評估中間狀態」的開放式任務較難套用。

### 名詞速查

| 名詞 | 一句白話 |
|:---|:---|
| **Thought（思路）** | 一個連貫的中間推理單位（可以是一個字、一條等式、一段文字） |
| **State（狀態）** | 輸入加上目前已走的思路序列，對應樹上的一個節點 |
| **Thought generator** | 生成下一步候選思路的機制（sample 取樣 / propose 提議） |
| **State evaluator** | 用 LLM 評估狀態好壞（value 給分 / vote 比較），當搜尋啟發式 |
| **BFS / DFS** | 廣度優先（每步留 b 個最佳）／深度優先（深入、剪枝、回溯）搜尋 |
| **Pruning / Backtracking** | 剪掉評估為不可能的分支／回到上一個岔路換條路走 |

## 歷史定位與展望

- **歷史定位**：
  - 直接一般化 `[[Chain-of-Thought Prompting (Wei et al., 2022)]]`（線性鏈）與 `[[Self-Consistency (Wang et al., 2022)]]`（多條鏈投票），把它們納為特例；
  - 把 Newell & Simon 的「問題求解即搜尋」古典觀念，用 LLM 當啟發式重新實作。
- **延伸閱讀**：
  - 往前：`[[Chain-of-Thought Prompting (Wei et al., 2022)]]`、`[[Self-Consistency (Wang et al., 2022)]]`、`[[ReAct (Yao et al., 2022)]]`
  - 往後：`[[LATS: Language Agent Tree Search (Zhou et al., 2023)]]`（把 MCTS、反思、行動整合）、Graph of Thoughts
- **展望**（依論文與後續發展）：
  - 「在 LLM 思路上做搜尋」成為 agent 規劃與 test-time search 的基礎，後續發展出 MCTS 式的搜尋（如 LATS）與更強的推理模型；
  - 降低搜尋成本、自動化思路分解與評估器設計，是後續改進方向。
