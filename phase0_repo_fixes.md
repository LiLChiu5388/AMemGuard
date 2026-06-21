# Phase 0 — A-MemGuard repo 整合修正記錄

> 對象:`AMemGuard/`(clone 自 github.com/TangciuYueng/AMemGuard,full history 僅 4 commits,HEAD = `d6082c7 add model provider`)。
> 範圍:讓 **ReAct-StrategyQA + GPT(gpt-4o-mini API)+ A-MemGuard 防禦** 路徑能跑(Phase 0 復現基準用)。
> 全部修改見 `phase0_repo_fixes.patch`(`git apply` 可重現)。所有改動皆**機械修正或介面重建,不更動 A-MemGuard 的防禦演算法本身**。
> ⚠️ 重要事實:`check_consistency` 在本 repo **任何 commit 都未被定義**(`git log --all -S"def check_consistency"` 零結果),代表作者發布版本從未能跑通防禦路徑;bug #1 是「重建」而非「還原」。

| # | 檔案:行 | 問題(before) | 修正(after) | 風險 |
|---|---|---|---|---|
| 1 | `ReAct/consistency.py` | `local_wikienv.py` 與 `medagent.py` 都 `from consistency import check_consistency`,但全 repo 無 `def check_consistency`(只有 `ConsistencyChecker` class)→ 防禦路徑 ImportError | 新增 `check_consistency(query, memories, selected_indexes, mode, knn, method)` adapter:快取一個 provider(預設 openai/gpt-4o-mini),包住 `ConsistencyChecker.check(...)`,回傳格式與兩呼叫端消費的 `consistent_memories`/`inconsistent_memories`(含 `memory`/`reasoning_chain`/`index`)完全吻合。backend/model/method 以環境變數覆寫(ablation 用) | **中**:重建作者原始 glue;以「論文 >95% 砍幅」驗收保真 |
| 2 | `ReAct/local_wikienv.py:16` | `from auditor_token import audit_and_sanitize_item`(無 `auditor_token.py`) | 改為 `from auditor import ...`(函式實際在 `auditor.py:132`) | 低 |
| 3 | `ReAct/local_wikienv.py:63-67` | retriever 硬寫本地路徑 `"/dpr-ctx_encoder-single-nq-base"`、`"realm-cc-news-pretrained-embedder"` | 改 HF hub id:`facebook/dpr-ctx_encoder-single-nq-base`、`google/realm-cc-news-pretrained-embedder` | 低 |
| 4 | `ReAct/run_strategyqa_gpt3.5.py:27-28` | `OPENAI_API_KEY=""`、`BASE_URL=""`(空字串會壞 OpenAI client) | 改讀 `os.getenv("OPENAI_API_KEY")`;`BASE_URL` 空時設 `None`(用官方預設端點) | 低 |
| 5 | `ReAct/run_strategyqa_gpt3.5.py:43-54` | `gpt()` 帶 `logprobs=1` 且只 `return content`(1 值),但 `react()` 以 `out, probs = llm(...)` 解包 2 值 → unpack 失敗 | 移除未用的 `logprobs=1`;`return_probs=True` 時回 `(content, None)`(probs 不被 `eval.py` 使用) | 低 |
| 6 | `ReAct/run_strategyqa_gpt3.5.py:121-126` | `trigger_token_list` 被註解掉、後面卻直接引用 → NameError | 改讀環境變數 `TRIGGER_TOKENS`(空白分隔);未設則明確報錯。優化後的 trigger 由此注入 | 低 |

## 驗收
- 三檔 `python -m py_compile` 通過(語法層)。
- 真正驗收需在 Colab GPU 跑「無防禦 adv vs A-MemGuard adv」,對照論文 ASR 砍幅 >95%(go/no-go gate #2)。

## 尚待(非 bug)
- **trigger 來源**:跑 `algo/trigger_optimization.py --agent qa --algo ap`(Colab GPU,DPR encoder 上 hotflip)產出 trigger tokens;或先用 golden/手動 trigger 做 pipeline smoke test。
- EhrAgent 路徑同樣有 `check_consistency` 問題(`medagent.py`),Phase 0 暫不處理(避開 MIMIC)。
- `audit_method=="ro"`(`auditor.audit_and_sanitize_item`)與 `distil`/`ppl` baseline 之後再驗。
