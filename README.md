# Rehearse

> 本地、离线、免费、开源的语音练习教练 —— 练英语口语对话，也能凭记忆复述任意 Markdown 笔记。在 Apple Silicon Mac 上跑。
> A fully-local, offline, free, open-source voice coach for Apple Silicon Macs: practice English conversation, and recall any Markdown doc from memory — out loud.

**Status:** 🟢 v1（英语对话）+ Phase 2（跨会话间隔复习持久化）+ markdown-recall 模式已交付。
`uv run pytest` 全绿：**284 passed / 6 skipped**（共 290 项；跳过的是需要 `REHEARSE_LLM_TESTS=1`
真模型的门控用例，以及本机未装 `--extra vad` 时的 Silero 用例）。
唯一无法自动化、只能人工验证的路径是真麦克风闭环（开发环境无麦克风）。

## 这是什么 / What it is

**模式一：英语对话练习。** 练 **自然连贯的日常英语对话**——不是补词汇，而是把已经背过的高频句子
（如 AnkiApp 句库）织进真实对话里，做"对话式间隔复习"（conversational spaced repetition）。
每轮挑"最久没练"的目标句（默认 3 句，`--n-targets` 可调，prompt 侧硬上限 4）喂进 system
prompt，助手自然地引导你用到它们；你说的话会被
本地 embedding 打分，命中即记一次"练过"，**跨会话持久化到 SQLite**，下次继续挑最欠练的。

**模式二：凭记忆复述任意 Markdown。** 指定任意 Markdown 文件（简历、面试准备、术语表、演讲稿……），
本地 LLM 把它抽成"回忆议程"，然后逐条口头考你。覆盖度评分是**诚实的**：一个要点只有在你的回答
**既语义贴题、又说出了它的硬事实**（数字/缩写，或足够多的关键词）时才算命中——空泛地绕不给分。

100% 本地离线，无任何云 API。

<details>
<summary><b>English summary</b></summary>

A voice loop (you speak → it speaks back) that runs entirely on an Apple Silicon Mac. Two modes:

- **English conversation practice** — weaves your existing high-frequency sentence bank (an AnkiApp
  XML export) into natural conversation, so you *use* those sentences in context instead of
  passively reviewing flashcards. Each turn injects the least-practiced sentences into the system
  prompt (3 by default, `--n-targets`, hard-capped at 4); your transcript is scored against them
  with a local embedding model, and hits are
  persisted to SQLite so scheduling continues across sessions — real conversational spaced repetition.
- **Markdown recall** — point it at any Markdown file (resume, interview prep, glossary, speech) and
  it interviews you to recall the key points from memory. Coverage scoring is honest: a point counts
  only if your answer is *both* semantically on-topic *and* contains its hard facts (numbers /
  acronyms, or enough of its key words). The coach LLM never receives the expected answers, so it
  structurally cannot leak them.

Stack: faster-whisper `small.en` (CPU) → MLX Qwen3.5-4B-4bit or Ollama `qwen3.5:4b` (GPU,
non-thinking) → Kokoro-82M via mlx-audio (GPU) → Silero VAD (CPU), plus `nomic-embed-text` via
Ollama for all scoring. Half-duplex by default (mic muted while the assistant speaks) so it works
on the built-in speaker without echo. No asyncio, no streaming TTS, no GPU concurrency — all
deliberate, see [Key design decisions](#关键设计取舍--key-design-decisions).
</details>

## 目标硬件 / Target hardware

Mac Studio · Apple M1 Max · 32GB unified memory（Apple Silicon 通用）。
输入实测：DJI Mic Mini 近场领夹麦；输出：Mac 内置音箱（半双工）或 AirPods（可选全双工）。

## 快速开始 / Quick start

前置：

- `uv`
- **Ollama**（`ollama pull qwen3.5:4b nomic-embed-text`）。即使 coach 跑在 MLX 上，**所有评分都走
  Ollama 的 `nomic-embed-text`**，所以实际上是必装的：Markdown 复述模式**硬性要求**它（拿不到
  embedding 直接拒绝启动，覆盖度评分是这个模式的全部意义）；英语模式会降级——照常对话，但
  "练过"追踪整场关闭。`--smoke` 也会探它（探不到只是 WARN，但 LLM 探针本身走 Ollama，探不通 smoke 直接 FAIL）。
- 英语模式还需要 AnkiApp XML 导出放进 `data/`（Markdown 复述模式不需要卡组）。

首次运行会联网拉一次模型（Whisper、Kokoro；首次用 MLX 后端时还有约 2.5GB 的 Qwen3.5-4B-MLX），
之后完全离线。

```bash
uv sync --extra audio --extra vad   # 安装 ASR/TTS/LLM/VAD 运行时（一次性；vad 会拉 torch，较重）
uv run rehearse --smoke             # 健康检查，不需要麦克风（建议先跑这个）
uv run rehearse --menu              # 交互式菜单：选模式，不用记 flag
uv run rehearse                     # 英语对话（自动模式）——直接开口说
```

> ⚠️ 本机 shell 的 `python` 被 `~/miniconda3` 抢占，`source .venv/bin/activate` 会失效。
> 本仓库**一律用 `uv run ...`**，不要手动 activate venv（否则 misaki 等依赖会 ImportError）。

**加个 alias**（从任何目录一条命令开菜单；跑在子 shell 里，不改变当前目录）。
必须**在仓库根目录**执行，因为它会把仓库的绝对路径写死进去：

```bash
cd /path/to/Rehearse
echo "alias rehearse='(cd \"$(pwd)\" && uv run rehearse --menu)'" >> ~/.zshrc && source ~/.zshrc
rehearse   # 菜单：英语练习 / 手动轮次 / 手动+简短 / Markdown 复述 / 冒烟 / 退出
```

## 项目结构 / Repository layout

```
rehearse/                  # 唯一的 Python 包（无子包，全部平铺）
  main_loop.py             # CLI 外壳：argparse + 菜单 + 3 条会话循环（run_loop / run_manual /
                           #   run_recall）+ smoke()；麦克风、播放、线程都在这一层
  loop_core.py             # 帧流 → 完整话语的可测胶水（io_rates / drain_utterance /
                           #   UtteranceAssembler / is_stop）——从 main_loop 抽出来好测
  pipeline.py              # ★ 一轮对话的核心：speak_turn()（ASR→LLM→TTS，无评分，两模式共用）
                           #   + respond()（英语模式 = speak_turn + D3 练过评分）
  _exit.py                 # CLI 退出闸门：os._exit 跳过 abseil/sentencepiece 的 SIGBUS

  ── 音频层 ────────────────────────────────────────────────────────────
  audio_io.py              # 单声道化 / soxr 重采样 / 预滚 RingBuffer；采样率常量
  vad.py                   # EndpointDetector 纯状态机（可用合成概率序列单测）+ SileroVad 封装
  duplex.py                # HalfDuplexGate：播放时静音麦克风 + 150ms guard（消回声）
  asr.py                   # WhisperASR：faster-whisper small.en，CPU/int8
  tts.py                   # KokoroTTS：Kokoro-82M via mlx-audio，Apple GPU，24kHz

  ── LLM / embedding 后端 ──────────────────────────────────────────────
  llm_client.py            # Ollama /api/chat：think:false、keep_alive=-1、流式、format(JSON schema)
                           #   + warmup() / think_probe()（非思考 + 暖 TTFT 契约，不达标启动即失败）
  mlx_llm.py               # mlx-lm 后端（同签名 drop-in）+ resolve_coach_chat() 后端解析 + 暖机探针
  embeddings.py            # nomic-embed-text via Ollama /api/embed + cosine（两个评分器共用）

  ── 英语对话模式 ──────────────────────────────────────────────────────
  anki_loader.py           # AnkiApp XML → Sentence（lang 属性会说谎，角色按内容判）
  session_seeder.py        # select_targets()：按 (练过次数, 上次练的时间) 升序挑 n 句（默认 3）
  prompt_builder.py        # 对话伙伴 system prompt + 目标句注入（JSON 转义，防提示注入）
  practiced_scorer.py      # D3「练过」评分：用户转写 vs 活跃目标句 cosine ≥ 0.50
  practice_store.py        # SQLite 跨会话持久化（schema v2）；损坏库隔离重建、WAL、并发迁移自愈
  unaided.py               # T-P2-2b「无提示自发产出」检测（影子模式，默认关闭，阈值 0.65）
  stats_report.py          # `rehearse-stats`：离线读库，最多/最少练的句子 + unaided 审计

  ── Markdown 复述模式 ─────────────────────────────────────────────────
  markdown_extractor.py    # 分块 → 本地 LLM(JSON) → PracticeItem；按内容哈希缓存；写可见议程
  practice_item.py         # PracticeItem（expected_points 只用于评分，绝不进 coach prompt）
  coverage.py              # CoverageTracker：cosine ≥ 0.62 AND 硬事实锚点命中，逐要点、粘性
  recall_session.py        # 议程walker：问 → 追问 → 卡住给提示 → 放行；coach prompt 结构性无泄漏

  ── 共用工具 ──────────────────────────────────────────────────────────
  text_sanitize.py         # 剥 markdown / emoji / 链接，TTS 不会念符号
  sentence_chunker.py      # 切成可朗读的句子块（缩写不误切、无标点长句按词边界截断）
  latency.py               # TurnTrace / LatencyAggregator：每轮 4 段计时 + 会话 p50/p95
  bench_asr.py             # `rehearse-bench-asr`：whisper vs parakeet WER + GPU 争用探针
                           #   —— 纯评估工具，不在出货管线里

  __init__.py              # 仅版本/包声明

tests/                     # 33 个测试文件，290 项；确定性模块纯单测 + 真模型集成（门控）
  fixtures/                # 合成 AnkiApp XML（真实 deck 是私有的，永不入库）
experiments/               # 一次性探针，不出货：真麦采集、mlx-vs-ollama 循环/抽取基准
docs/                      # DESIGN.md（v1 架构+对抗审查）、DESIGN-tui-interview.md（F1/F2）、
                           #   ASR-evaluation.md（模型选型决策记录）
data/                      # 你的 AnkiApp XML 导出（gitignored，私有）
debug/                     # --debug 落盘的每轮 WAV + 转写（gitignored）
HANDOFF.md · TODOS.md      # 交接说明 / Phase 2 待办
```

## 架构 / Architecture

### 模型栈

| Stage | 选型 | Device | 为什么 |
|---|---|---|---|
| ASR | faster-whisper `small.en`（int8, `beam_size=5`, `vad_filter`） | **CPU** | 英语专用；留在 CPU 不和 LLM/TTS 抢 Metal GPU |
| LLM | **MLX** `mlx-community/Qwen3.5-4B-MLX-4bit`（非思考）<br>Ollama `qwen3.5:4b` 作回退 | **GPU** | 解码吞吐 ~2.8x；TTFT 持平。auto = 有 mlx-lm 就用 MLX |
| TTS | Kokoro-82M (`prince-canuma/Kokoro-82M`, `af_heart`) via mlx-audio | **GPU**（与 LLM 串行） | 82M 够小、RTF≈0.03、Apache-2.0 |
| VAD | Silero VAD v5（512 样本 / 32ms 窗口） | CPU | 端点检测 |
| Embedding | `nomic-embed-text` via Ollama `/api/embed` | — | 两个模式的评分基础；**Ollama 因此不可移除** |
| 编排 | 同步阻塞主循环 + PortAudio 回调线程（+ 自动模式下的 Enter 监听线程） | — | **没有 asyncio**；全串行，GPU 零并发 |

### 一轮对话的数据流（英语自动模式）

```
麦克风 ──PortAudio 回调线程──> audio_q ──主循环──> HalfDuplexGate（播放期间丢弃这些块）
   └─> 设备原生采样率→16k 重采样（io_rates 查询，Mac 上通常是 48k；设备全程只开一次）
   └─> SileroVad.prob ─> EndpointDetector（起始 64ms voiced / 收尾 1000ms 静音 / 单句上限 20s）
        └─> UtteranceAssembler（"start" 时补上 200ms 预滚，不切掉开头）
             └─> pipeline.respond()
                   ├─ asr.transcribe            (faster-whisper, CPU)
                   ├─ build_system_prompt(select_targets 选出的最久未练目标句，默认 3)
                   ├─ chat_fn                   (MLX 或 Ollama，非思考，num_predict 硬上限)
                   ├─ sanitize_for_tts → chunk_sentences
                   ├─ tts.synth 每个 chunk → 拼成整段回复音频
                   └─ score_practiced           (nomic cosine ≥ 0.50)
             └─> play()：24k→设备重采样，按 100ms 音频块写出；块与块之间可被 Enter 截停
                           （两种模式都行），或被语音 barge-in 截停（仅 --full-duplex）
             └─> finally: apply_practiced(内存 stats) + store.record_practiced(SQLite)
             └─> （可选、默认关）影子 unaided 检测，播放之后跑，永不影响排程
```

**注意 TTS 不是流式的**：`speak_turn` 先把回复切成句块、逐块交给 Kokoro 合成，**全部合成完
再拼成一段整体音频返回**，然后才开始播放。切句的作用是让 Kokoro 每次处理一个短句，并单独计出
`tts_ttfa`（第一块的合成耗时，唯一影响体感的 TTS 延迟）——**不是**边合成边播。截停的粒度是播放
时的 100ms 音频块，与句子边界无关。这是 D1「全串行」的直接后果，也是 `docs/ASR-evaluation.md`
里三方独立评审后明确冻结、不再优化的取舍。

### Markdown 复述模式的数据流

```
加载（一次性）：
  doc.md ──sha256(版本|后端|模型|内容)──> ~/.cache/rehearse/recall/<hash>.json 命中？
     │未命中：chunk_markdown（有标题按标题切并打包，无标题按 6000 字窗口+200 重叠）
     │        └─每块一次 LLM 调用（Ollama 走 format=JSON schema；MLX 走宽容 JSON 解析）
     │        └─失败则回退到 heading+bullet 正则解析器
     │        └─_merge_substantive：滤掉无实质内容的点、按 key 去重
     └─> 写 <doc>.md.recall.json（可见议程 = 透明度功能，跟缓存是两回事，命中/未命中都会写）

每轮：
  RecallSession.coach_prompt()   ← 只含当前 item 的 prompt（+ 卡住时的 support_snippet）
      ★ expected_points 结构性地从不进入 prompt —— coach 想泄题也泄不了
  speak_turn(...) → 用户回答
  CoverageTracker.score(item, 累计回答)
      每个要点：cosine ≥ 0.62  AND  硬事实门（有锚点则全部锚点命中；无锚点则内容词覆盖 ≥50%）
      → hit / partial / miss；粘性：命中过就不会被后面的废话冲掉
  进度：全 hit → 下一条；有进展 → 停留追问；卡住 2 轮且该条有 support_snippet → 给一次提示，
        第 3 轮仍无进展 → 放行；若该条根本没有 snippet，卡住 2 轮就直接放行，不把人困住
```

### 关键设计取舍 / Key design decisions

这些都经过 Codex 对抗审查加固，完整推导见 `docs/DESIGN.md`。

- **不全用 MLX** — ASR 留在 CPU，避免 ASR+LLM+TTS 三者抢同一块 Metal GPU。
- **GPU 上 LLM 与 TTS 串行**，不并发——并发恰恰是 TTFT 抖动最严重的时刻。
- **半双工默认**（TTS 播放时静音麦克风 + 150ms guard）根除外放回声；戴耳机可 `--full-duplex`。
- **非思考是结构性的，不是靠 prompt** — Ollama 原生 `think:false` / MLX `enable_thinking=False`，
  启动探针断言无 `<think>` 块且暖 TTFT < 3s，不达标**直接启动失败**，绝不静默降级。
- **不显式纠错** — 自然对话伙伴，靠地道措辞和复述潜移默化（纯语音闭环无法评判发音，这是固有边界）。
- **评分必须可证伪** — "练过"是 cosine ≥ 0.50（在真实 nomic 上标定：改写 0.55-1.0，无关 ≤0.454）；
  复述覆盖度是 cosine ≥ 0.62 **加**事实锚点门。纯 cosine 会给空话高分，那是表演不是度量。
- **持久化永不阻塞练习** — 损坏库自动隔离重建、写失败降级成一行警告、并发迁移自愈重试；
  `--no-persist` 完全退回 v1 纯内存行为。
- **延迟靠实测** — 每轮埋 4 段时间戳，退出时打 p50/p95。`<1.5s` 是冲刺目标不是验收标准。
- **架构是自写的，不是 fork** — 设计阶段考虑过 fork `eauchs/speech-to-speech-pipeline`，
  对抗审查发现它只有 3 个 commit、无 release，判定"当可挖的代码，不当地基"，最终全部自写。

## 状态落在哪 / Where state lives

| 内容 | 路径 | 说明 |
|---|---|---|
| 跨会话练习统计 | `~/.local/share/rehearse/practice.db`（尊重 `XDG_DATA_HOME`） | SQLite；`--practice-db` 可指定 |
| Markdown 抽取缓存 | `~/.cache/rehearse/recall/<hash>.json` | 键 = 抽取器版本+后端+模型+文件内容 |
| 可见回忆议程 | `<你的文档>.md.recall.json`（就在源文件旁边） | 透明度：打开就能看到会被考什么 |
| `--debug` 落盘 | `debug/`（英语）、`recall-debug/`（复述） | 每轮的 WAV + 转写 + 计时。⚠️ `.gitignore` 只挡了 `debug/`；`recall-debug/` **未被忽略**（`.wav` 因 `*.wav` 规则安全，但 `.txt` 转写不是），而它装的是你个人文档的内容 |
| 你的 Anki 卡组 | `data/*.xml` | gitignored，私有，**永不入库** |

## CLI 参考 / CLI reference

### `rehearse`

```bash
uv run rehearse                        # 英语对话，自动轮次（VAD 判断你说完了）
uv run rehearse --manual-turns         # 手动轮次：Enter 开始 / Enter 结束，零时间压力
uv run rehearse --manual-turns --brief # + 约 15 词的简短回复：更快，你说得更多
uv run rehearse --full-duplex          # 语音插话打断（配 AirPods 用，外放会回声）
uv run rehearse --content markdown --path /abs/path/notes.md   # 凭记忆复述
uv run rehearse --smoke                # 健康检查 + TTS→ASR 往返，不需要麦克风
uv run rehearse --menu                 # 交互菜单（不能和其它模式 flag 混用，会报错退出 2）
```

| Flag | 默认 | 作用 |
|---|---|---|
| `--content {english,markdown}` | `english` | 内容源；`markdown` 必须配 `--path` |
| `--path PATH` | — | 要复述的 Markdown 文件 |
| `--decks [PATH ...]` | `data/*.xml` | AnkiApp XML 卡组 |
| `--coach-backend {auto,mlx,ollama}` | `auto` | 实时 coach 的 LLM 后端（auto = 能导入 mlx-lm 就用 MLX） |
| `--model ID` | 按后端 | coach 模型 id，在所选后端的命名空间里解释 |
| `--extract-backend {auto,mlx,ollama,fast}` | `auto` | Markdown 抽取策略；**`fast` = 不调 LLM 的即时标题+要点解析器** |
| `--extract-model ID` | 按后端 | 抽取模型 id |
| `--n-targets N` | `3` | 每轮注入的目标句数（2-4 合理） |
| `--voice NAME` | `af_heart` | Kokoro 音色 |
| `--asr-model NAME` | `small.en` | `medium.en`（更准更慢）/ `distil-large-v3`（最准） |
| `--end-silence-ms MS` | `1000` | 自动模式下多久静音算你说完；觉得被打断就调大 |
| `--brief` | off | 一句话回复 |
| `--full-duplex` | off | 语音 barge-in |
| `--manual-turns` | off | 手动轮次 |
| `--practice-db PATH` | XDG 默认 | 指定统计库 |
| `--no-persist` | off | 纯内存，不读不写历史（v1 行为） |
| `--enable-unaided` | off | **实验/影子**：记录"自发产出"事件供后续标定，**不影响排程** |
| `--debug` | off | 每轮 WAV + 转写 + 逐轮延迟行落盘 |
| `--smoke` / `--menu` | — | 冒烟测试 / 交互菜单 |

> **markdown 模式只用其中一部分 flag。** `--content markdown` 走的是 `run_recall()`，它只接受
> `--path` `--extract-backend` `--extract-model` `--coach-backend` `--model` `--voice` `--asr-model`
> `--debug`。英语模式专属的 `--decks` `--n-targets` `--brief` `--manual-turns` `--full-duplex`
> `--end-silence-ms` `--practice-db` `--no-persist` `--enable-unaided` 在该模式下**会被静默忽略**
> （复述模式本身就是手动轮次，回复长度固定 120 token，不写练习库）。

### `rehearse-stats`（离线，只读，不需要模型和麦克风）

```bash
uv run rehearse-stats                       # 最多/最少练的句子 + 总计 + unaided 审计
uv run rehearse-stats --practice-db /path/db
```

### `rehearse-bench-asr`（评估工具，不碰出货管线）

```bash
uv sync --extra audio --extra parakeet
uv run rehearse-bench-asr record --refs sentences.txt --out bench_audio/
uv run rehearse-bench-asr run --wav-dir bench_audio/              # whisper vs parakeet：WER + 延迟
uv run rehearse-bench-asr run --wav-dir bench_audio/ --contention # GPU 争用对 coach TTFT 的影响
```

换 ASR 的门槛（见 `docs/ASR-evaluation.md`）：必须**同时**在你自己的嗓音上显著降低 WER
**且**不劣化 p95 首音延迟。省几毫秒 ASR 时间不构成换模型的理由。

## 使用提示 / Tips

**觉得被打断 / 来不及想：** `--end-silence-ms 1500`，或直接用 `--manual-turns`（零时间压力）。

**识别不准（口音/噪声）：** ASR 默认已开 `vad_filter` + `beam_size=5`，若仍不行换
`--asr-model medium.en`（约 2.3s vs small.en 的 0.9s）或 `distil-large-v3`；`--debug` 落盘听回放。

**默认行为：** 半双工（助手说话时静音麦克风，外放不回声）、连续对话直到你说 "stop/bye/quit"
或 Ctrl-C、播放中按 Enter 可截停。退出时打印延迟 p50/p95 和本次练过的统计。

**间隔复习是跨会话的：** 每句练过的记录写进 `practice.db`，下次开机继续挑最欠练的——
真正的对话式间隔复习，不是每次重置。

**Markdown 抽取很慢？** 长文档是"每块一次 LLM 调用"，首次可能几分钟（有 `chunk N/total` 进度，
Ctrl-C 可中止且不会写脏缓存）。结构良好的文档（简历、要点笔记、术语表）用
`--extract-backend fast` 可以**完全跳过 LLM**，瞬间完成。抽取结果按内容哈希缓存，改了文档才重抽。

## 测试 / Testing

```bash
uv run pytest                                   # 290 项：284 passed / 6 skipped（跳过=门控用例）
REHEARSE_LLM_TESTS=1 uv run pytest tests/test_integration_real.py -s   # 真模型 E2E（无 mock）
```

> `pyproject.toml` 里已有 `addopts = "-q"`，再手动加一个 `-q` 会变成 `-qq`，**连最后那行
> "N passed" 汇总都会被吞掉**。想看汇总就别再补 `-q`。

分层：
- **确定性模块纯单测** —— VAD 状态机（合成概率序列）、重采样数学、切句、抽取解析、覆盖度打分、
  SQLite 存储（33 个用例，含损坏库/锁库/并发迁移）、CLI 参数分派、菜单映射。
- **无麦克风的音频集成** —— Kokoro 合成已知文本 → faster-whisper 转写 ≈ 原文（TTS→ASR 往返），
  覆盖整条音频链路。
- **真模型 E2E（门控）** —— `REHEARSE_LLM_TESTS=1` 时跑真 TTS + 真 ASR + 真 nomic-embed 的
  "练过评分 → SQLite 持久化 → 跨会话重排序"和"无提示自发产出检测"，无 mock。
- **只能手测的** —— 真麦克风闭环（开发环境无麦克风）。`experiments/mic_probe.py` 是给有硬件时用的探针。

## 路线图 / Roadmap

- **Phase 1（MVP）✅** 全本地闭环（faster-whisper + qwen3.5:4b + Kokoro + Silero）、连续半双工对话、
  Anki 句子注入 + nomic「练过」评分、延迟埋点、原生冒烟测试。
- **Phase 2 ✅（部分）**
  - markdown-recall 模式：本地 LLM 抽取 + 诚实（cosine + 事实锚点）覆盖度评分。
  - 交互式启动菜单 `--menu` + `rehearse` alias。
  - T-P2-2a 跨会话持久化（SQLite）——间隔复习真正闭环。
  - T-P2-2b「无提示自发产出」检测——**影子模式已落地**，默认关闭。
  - `rehearse-stats`：离线跨会话进度报告 + unaided 审计。
- **性能与选型工作 ✅**（2026-06）
  - coach 与抽取双双迁移到 MLX（Ollama 保留为回退 + embedding 后端），解码吞吐 ~2.8x。
  - 每轮 4 段延迟埋点 + 会话 p50/p95（`rehearse/latency.py`）——先测量，再谈换模型。
  - `rehearse-bench-asr` + `docs/ASR-evaluation.md`：whisper vs parakeet 的本地决策工具；
    结论是**保持现状**，并把 LLM 后端冻结在 mlx-primary/Ollama-fallback。
  - `--extract-backend fast`：结构良好的文档可完全跳过 LLM 抽取。
- **待办**（详见 `TODOS.md`）
  - T-P2-2b 剩余：用真实会话的 `practice_event` 审计日志标定 0.65 阈值，再决定是否接入排程。
  - T-P2-1：nomic + bce-reranker 语义检索注入（当前是"最久未练"排序，不是话题相关）。
  - T-P2-3：全双工外放 + 真 WebRTC AEC。
  - T-P2-4：菜单的运行时实时状态显示（live status TUI）。

## 文档 / Docs

- [`docs/DESIGN.md`](docs/DESIGN.md) — v1 完整设计、Codex 对抗审查（v2 加固）、工程评审锁定（v3）、
  真麦调优（v4）。
- [`docs/DESIGN-tui-interview.md`](docs/DESIGN-tui-interview.md) — F1（TUI）/ F2（Markdown 复述）设计与
  保真度修正（C1-C9，含"coach 不能看到答案"的结构性约束由来）。
- [`docs/ASR-evaluation.md`](docs/ASR-evaluation.md) — ASR/TTS/LLM 选型决策记录；mlx-lm vs Ollama 实测；
  为什么冻结在 mlx-primary/Ollama-fallback。
- [`HANDOFF.md`](HANDOFF.md) — 构建过程交接说明、已锁决策、环境坑。
- [`TODOS.md`](TODOS.md) — Phase 2 待办的完整论证。

## License

TBD（意向为宽松许可）。所用组件为 MIT / Apache-2.0。
