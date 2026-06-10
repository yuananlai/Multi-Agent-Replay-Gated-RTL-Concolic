# Multi-Agent-Replay-Gated-RTL-Concolic

本项目是一个面向 Verilog RTL 设计的 concolic verification 框架。框架以传统 concolic execution 为核心，结合多 agent 调度、运行时反馈记忆、blocked-prefix memory、replay gate 和 prefix-preserving recovery，用于提升 RTL 分支覆盖率，并减少无效 SMT 求解开销。

项目重点处理两类 RTL concolic testing 中常见的低效现象：

1. **Repeated UNSAT Prefix**  
   多个候选分支反复落入相同或高度相似的不可行路径前缀，导致重复 Z3 调用。

2. **SAT but Replay Has No Coverage Gain**  
   SMT 求解结果为 SAT，且外部输入发生变化，但在统一 RTL replay 后没有新增 coverage，说明该 SAT 候选在真实执行中没有产生有效推进。

框架的核心思想是：**SMT SAT 不是最终成功，只有 replay-confirmed coverage gain 才被接受。**

---

## 1. Overall Workflow

整体流程如下：

```text
RTL Code
  ↓
Parse / Instrument / Static RTL Summary
  ↓
Candidate Pool Construction
  ↓
Multi-Agent Scheduling and Reranking
  ↓
Concolic Core
  ├─ Simulation
  ├─ Path Constraint Collection
  ├─ Branch Constraint Mutation
  ├─ SMT Solving
  └─ Input Vector Generation
  ↓
Replay Gate
  ├─ replay has coverage gain → accept and save input
  └─ replay has no coverage gain → reject and record failure context
  ↓
Runtime Memory / Reflection / Recovery
```

传统 concolic core 负责仿真、路径约束构建、分支约束变异和 SMT 求解；多 agent 层负责候选调度、失败模式记忆、recovery 触发和 replay 反馈利用。

---

## 2. Key Ideas

### 2.1 Blocked-Prefix Memory for Repeated UNSAT Prefix

当某个 candidate 的 prefix constraints 被判定为 UNSAT 时，系统不只把它当作一次失败，而是记录其结构化上下文：

- target id
- candidate id
- branch id / assign id
- clock
- prefix range
- state region
- condition region
- constraint hash
- involved signals
- recent input and state summary

后续如果遇到相同或相似的不可行 prefix，scheduler 可以对该 candidate 执行 skip、down-rank 或 budget pruning，从而减少重复求解。

相关直接指标包括：

- `skipped_candidates`
- `avoided_z3_calls`
- `budget_pruned_candidates`
- `prefix_unsat_total`
- `repeated_unsat_prefix`
- `repeated_unsat_prefix_rate`
- `avg_solver_calls_per_target`

这些指标会被写入 metrics summary，可用于消融实验和论文分析。

### 2.2 Replay-No-Coverage Handling for SAT No-Op Candidates

当 SMT solver 返回 SAT 且 input vector 确实发生变化，但 replay 后没有新增 coverage 时，框架将该结果记录为：

```text
SAT_REPLAY_NO_NEW_COV
```

这类失败通常说明：

- 符号约束层面看似可行；
- 真实 RTL 执行没有进入新路径；
- 输入变化可能没有影响目标状态转移；
- 或者路径对齐、时序窗口、控制信号组合仍然不足。

系统会记录以下信息：

- `path_signature`
- `predicate_signature`
- `state_signature`
- `input_delta`
- `focus_signals`
- `change_signature`
- `replay_path_hash`
- `total_changes`

这些信息用于后续 replay-no-cov fast skip、candidate penalty 和 recovery hint。

### 2.3 Prefix-Preserving Recovery Beam

当普通搜索发生 coverage stall、repeated state、repeated condition 或 replay-no-cov pressure 时，recovery agent 会触发 prefix-preserving recovery：

```text
Stall Detection
  ↓
Preserve Useful Prefix
  ↓
Mutate Late Input Window
  ↓
Generate Beam Candidates
  ↓
Replay and Select Best Candidate
```

Recovery 不从头随机生成输入，而是尽量保留已经有效的前缀，只修改后半段 late window。这样可以避免破坏前面已经形成的状态，同时给后续控制信号、数据窗口或协议交互提供逃出局部停滞的机会。

---

## 3. Multi-Agent Design

框架中的 agent 不是替代 concolic execution，而是围绕 concolic core 提供反馈控制。

### Agent-1: Static RTL Summary / Instrumentation Agent

负责 RTL 静态分析和 observation-only 插桩：

- 解析 Verilog AST；
- 提取 CFG、branch predicates、state registers、control signals；
- 提取 target-related signals 和 dependency features；
- 生成 state summary、predicate monitor、condition monitor；
- 提供后续 scheduler 和 recovery 使用的静态特征。

该 agent 的插桩主要作为 observation log，不应改变 DUT 原始赋值、时序和控制流。

### Agent-2: Online Scheduler / Rerank Agent

负责 candidate branch 的排序和预算控制：

- 维护 candidate pool；
- 融合静态距离、依赖关系、运行时状态、历史失败；
- 根据 blocked-prefix memory 对重复失败 candidate 降权或跳过；
- 根据 solver pressure 调整 solver budget；
- 根据 commander objective 选择当前轮策略。

常见 round objective 包括：

- `increase_target_alignment`
- `escape_repeated_state_region`
- `reduce_repeated_unsat`
- `probe_new_predicates`
- `mutate_data_path_history`
- `break_instruction_pointer_stall`

### Agent-3: Recovery Beam Agent

负责 stall-triggered recovery：

- 只在普通搜索停滞时触发；
- 保留有用 prefix；
- 只扰动 late input window；
- 生成多个 recovery beam candidates；
- 通过 replay 选择 coverage gain 最大或最符合目标的候选。

### Replay Judge

负责统一 replay 验证：

- SMT SAT 不直接视为成功；
- input vector 必须经过 RTL replay；
- 只有 replay-confirmed coverage gain 才被接受；
- SAT 但 replay 无新增覆盖会被记录为 failure context。

### Memory / Reflection Agent

负责持久化运行经验和生成反馈：

- 保存 blocked prefix；
- 保存 SAT replay no coverage trace；
- 保存 candidate attempt / rerank trace；
- 保存 recovery beam trace；
- 保存 round/action/lesson JSONL；
- 根据一轮结果生成 reflection lesson，反馈给后续 scheduler 和 recovery。

---

## 4. Repository Structure

```text
.
├── main.py                     # 项目入口；解析参数、配置模式、执行 concolic 流程
├── Globals.py                  # 全局配置、benchmark 配置、agent mode、消融开关
├── GlobalVar.py                # 运行时全局状态、覆盖率指标、solver 指标、blocked-prefix 直接指标
├── Simulate.py                 # concolic 主循环、仿真、SMT 求解、replay-no-cov 处理、recovery 调用
├── SMTlib.py                   # RTL 到 SMT 的核心建模、表达式、约束和分支结构
├── Utilies.py                  # InputVector、约束辅助、输入向量生成和高层参数展开
├── DQN.py                      # candidate 排序、策略网络及部分 RL 相关逻辑
├── RL.py                       # 仿真、build_stack、constraint stack 构建等基础执行逻辑
├── DutCreate.py                # DUT 生成、RTL 插桩、datapath extraction
├── TbCreate.py                 # 单输入 testbench 生成
├── MultiTbCreate.py            # 多输入 replay testbench 生成
├── AgentGuidance.py            # AgentFeedbackBus、静态分析、candidate 调度、recovery 模板
├── AgentMemory.py              # agent memory JSONL 持久化
├── CommanderAgent.py           # round objective 与 high-level profile 选择
├── ReflectionEngine.py         # post-round lesson 生成
├── FrontierStateBank.py        # recovery 可复用 frontier prefix 存储
├── ProtocolProfiles.py         # benchmark/protocol-specific profile 和 focus signals
├── LLM.py                      # LLM 调用、JSON schema、planning / recovery / datapath analysis
├── RandomTest.py               # random baseline 及 fixed rare branch 相关辅助
└── IMPLEMENTATION_NOTES.md     # agent enhancement 实现记录
```

---

## 5. Requirements

### 5.1 System Tools

需要安装至少一种 RTL 仿真器：

- `iverilog` + `vvp`，默认使用；
- 或 `verilator`，需要在 `Globals.py` 中切换 `simulator`。

示例：

```bash
sudo apt-get update
sudo apt-get install -y iverilog verilator
```

### 5.2 Python Packages

建议使用 Python 3.8+。主要依赖包括：

```bash
pip install z3-solver pyverilog numpy torch tqdm requests memory-profiler lightning litgpt
```

如果只运行 `local` 模式，通常不需要配置 LLM API key。`hybrid` 和 `llm_full` 可能调用 LLM 相关功能，具体是否调用取决于 `Globals.py` 中的开关和当前 benchmark。

---

## 6. Quick Start

### 6.1 Run with Default Benchmark

默认 benchmark 在 `Globals.py` 中设置，例如：

```python
testbench = 'b22'
```

直接运行：

```bash
python main.py
```

### 6.2 Run Baseline Concolic Mode

```bash
python main.py --agent-mode local
```

`local` 模式关闭 LLM、agent memory、blocked-prefix memory、rerank、reflection 和 recovery，适合作为 baseline。

### 6.3 Run Hybrid Mode

```bash
python main.py --agent-mode hybrid
```

`hybrid` 模式启用多 agent 调度、blocked-prefix memory、commander/reflection、高层输入 profile 和 LLM-assisted policy/path alignment，但默认不启用 recovery。

### 6.4 Run Full Mode

```bash
python main.py --agent-mode llm_full
```

`llm_full` 模式启用完整框架，包括 recovery。

### 6.5 Run a Specific Verilog Design

```bash
python main.py \
  -f ./test/b15/b15.v \
  -t b15 \
  -c CLOCK \
  -e RESET \
  -d 1 \
  -u 80 \
  -s 8 \
  --agent-mode llm_full
```

参数说明：

| 参数 | 含义 |
|---|---|
| `-f`, `--file_pth` | Verilog 文件路径 |
| `-t`, `--top-module` | top module 名称 |
| `-c`, `--clk-name` | clock 信号名 |
| `-e`, `--reset-name` | reset 信号名 |
| `-d`, `--reset-edge` | reset 有效边/有效电平配置 |
| `-u`, `--unroll` | 仿真展开长度 |
| `-s`, `--steps` | 每轮探索步数 |
| `-l`, `--limit_iter` | concolic 迭代限制 |
| `-r`, `--random-sim` | 初始 random simulation 次数 |
| `--agent-mode` | `local` / `hybrid` / `llm_full` |
| `--rerank-ablation-mode` | candidate-rerank/framework 消融阶段 |

---

## 7. Ablation Study

框架支持 candidate-rerank / framework ablation：

```bash
python main.py --rerank-ablation-mode A0_baseline
python main.py --rerank-ablation-mode A1_local_semantic
python main.py --rerank-ablation-mode A2_blocked_memory
python main.py --rerank-ablation-mode A3_recovery
python main.py --rerank-ablation-mode A4_llm
```

各阶段含义：

| Stage | 含义 |
|---|---|
| `A0_baseline` | baseline concolic |
| `A1_local_semantic` | 加入本地语义 candidate rerank |
| `A2_blocked_memory` | 加入 blocked-prefix memory |
| `A3_recovery` | 加入 prefix-preserving recovery，关闭 LLM recovery |
| `A4_llm` | 完整 LLM-enhanced framework |

建议实验组织方式：

```text
A0 baseline concolic
  → A1 + local semantic policy
  → A2 + blocked-prefix memory
  → A3 + prefix-preserving recovery
  → A4 + optional LLM enhancement
```

这样可以分别观察 blocked-prefix memory、recovery 和 LLM enhancement 对覆盖率、solver 调用数、repeated UNSAT prefix rate 的影响。

---

## 8. Outputs and Logs

运行时会按不同模式隔离输出目录，避免不同方法之间共享输入和 memory。

典型输出包括：

```text
run_<method>/
  ├── conquest_dut.v
  ├── conquest_tb.v
  ├── data.mem
  ├── sim.log
  ├── mergesim.log
  ├── branch_id.txt
  └── metrics_summary.json

inputs_history_<method>/
  └── input_x/data.mem

agent_memory_<method>/
  ├── candidate_attempt_trace.jsonl
  ├── candidate_rerank_trace.jsonl
  ├── rerank_ablation_trace.jsonl
  ├── recovery_ablation_trace.jsonl
  ├── recovery_beam_trace.jsonl
  ├── sat_replay_no_cov_trace.jsonl
  ├── frontier_state_bank.jsonl
  ├── protocol_profile_trace.jsonl
  └── blocked_prefix_memory.jsonl
```

其中最重要的是：

- `metrics_summary.json`：覆盖率、solver 统计、blocked-prefix 直接指标、recovery beam 指标；
- `blocked_prefix_memory.jsonl`：重复 UNSAT prefix 和 replay-no-cov failure context；
- `sat_replay_no_cov_trace.jsonl`：SAT 但 replay 无 coverage gain 的详细记录；
- `recovery_beam_trace.jsonl`：recovery beam candidate 的 replay 结果；
- `candidate_rerank_trace.jsonl`：candidate rerank 前后排序及原因。

---

## 9. Metrics to Report

建议在实验报告或论文中至少汇报以下指标：

### Coverage Metrics

- total branch count
- covered branch count
- branch coverage rate
- fixed rare branch total
- fixed rare branch covered count
- rare branch coverage rate

### Solver Metrics

- total solver calls
- total solver time
- average solver time
- prefix solver calls
- branch solver calls
- prefix UNSAT count
- branch UNSAT count

### Blocked-Prefix Direct Metrics

- skipped candidates
- avoided Z3 calls
- repeated UNSAT prefix rate
- average solver calls per target
- skipped-by-reason distribution

### Recovery Metrics

- recovery trigger count
- recovery beam candidate count
- recovery success count
- selected gain total
- recovery score total

### Replay-No-Coverage Metrics

- number of `SAT_REPLAY_NO_NEW_COV` events
- repeated no-cov signature count
- no-cov candidate ids
- applied signals and input delta
- replay path hash diversity

---

## 10. LLM Configuration

如果运行需要调用 LLM 的模式，需要配置对应 API key。代码支持 OpenAI-compatible、Gemini 和 DeepSeek 风格接口，常用环境变量包括：

```bash
export OPENAI_API_KEY="your_key"
export OPENAI_API_URL="https://.../v1"
export OPENAI_CHAT_API_URL="https://.../v1/chat/completions"

export GEMINI_API_KEY="your_key"
export GEMINI_API_URL="https://..."

export DEEPSEEK_API_KEY="your_key"
export DEEPSEEK_API_URL="https://.../chat/completions"
```

如果没有配置 API key，相关 LLM task 会被跳过或退回本地策略。为了做公平消融，建议在 `A0` 到 `A3` 阶段保持 LLM 关闭，只在 `A4_llm` 中打开。

---

## 11. Adding a New Benchmark

添加新 benchmark 时，建议修改 `Globals.py` 中的 `configurations`：

```python
configurations = {
    "new_design": {
        "file_path": "./test/new_design/new_design.v",
        "top_module": "new_design",
        "clk_name": "clk",
        "reset_name": "rst",
        "reset_edge": "1",
    }
}
```

然后运行：

```bash
python main.py \
  -f ./test/new_design/new_design.v \
  -t new_design \
  -c clk \
  -e rst \
  -d 1 \
  --agent-mode local
```

建议先跑 `local`，确认基础 testbench、DUT 生成和 branch id 稳定，再跑 `hybrid` 和 `llm_full`。

---

## 12. Recommended Experiment Protocol

为避免不同方法之间结果互相污染，建议：

1. 每个 mode 单独运行；
2. 保持同一 benchmark 的原始 RTL 不变；
3. 检查不同 mode 的 `branch_id.txt` 是否一致；
4. 确认 `run_<method>`、`inputs_history_<method>`、`agent_memory_<method>` 已隔离；
5. 使用 `metrics_summary.json` 汇总结果；
6. 对 recovery case，额外查看 `recovery_beam_trace.jsonl` 和 `sat_replay_no_cov_trace.jsonl`；
7. 对 blocked-prefix memory 消融，重点查看 `skipped_candidates`、`avoided_z3_calls`、`repeated_unsat_prefix_rate` 和 `avg_solver_calls_per_target`。

---

## 13. Troubleshooting

### 13.1 运行后覆盖率在不同模式之间异常接近

可能原因：

- 不同 mode 共享了 `inputs_history`；
- 不同 mode 共享了 `agent_memory`；
- 运行产物没有隔离，复用了旧的 `data.mem` / `sim.log` / `branch_id.txt`。

建议检查：

```text
run_<method>/
inputs_history_<method>/
agent_memory_<method>/
```

是否独立存在。

### 13.2 SMT SAT 但没有覆盖提升

检查：

```text
agent_memory_<method>/sat_replay_no_cov_trace.jsonl
agent_memory_<method>/blocked_prefix_memory.jsonl
```

重点关注：

- `candidate_id`
- `branch_id`
- `clock`
- `focus_signals`
- `input_delta`
- `path_signature`
- `predicate_signature`

如果重复出现同类 no-cov，可以考虑提高 replay-no-cov penalty 或更早触发 recovery。

### 13.3 Recovery 没有明显提升覆盖率

检查：

```text
agent_memory_<method>/recovery_beam_trace.jsonl
```

重点关注：

- 是否真的触发 recovery；
- beam candidates 是否有 input delta；
- replay 是否有 branch gain；
- 是否误破坏了已有 rare branch；
- recovery window 是否过短或过晚；
- focus signals 是否与目标 branch 相关。

### 13.4 LLM 请求失败或超时

如果只想跑 deterministic ablation，可使用：

```bash
python main.py --rerank-ablation-mode A0_baseline
python main.py --rerank-ablation-mode A1_local_semantic
python main.py --rerank-ablation-mode A2_blocked_memory
python main.py --rerank-ablation-mode A3_recovery
```

这些阶段应尽量保持 LLM-free。只有 `A4_llm` 用于评估 LLM enhancement。

---

## 14. Notes

- 本框架中的 coverage 以 replay-confirmed coverage 为准。
- Blocked-prefix memory 不应直接修改 DUT 行为，只用于调度、跳过、降权和日志分析。
- Recovery 应在 stall 或 replay-no-cov pressure 出现时触发，不建议让 LLM 全程 rerank 所有 candidate。
- 做消融实验时，应确保 branch target set、instrumentation 和 replay 环境一致。
