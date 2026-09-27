# RSI 编程代理开发战略

> 基于 11 个开源项目/方法论（loop-engineering、LongHorizon-Harness、LoopTroop、ralphex、fractal、awesome-agent-orchestrators、awesome-ralph、how-to-ralph-wiggum、loopy、ralph-orchestrator、hive）的整合分析。
>
> 设计约束回顾：**性能要求不高**；优先**命令行调用**现有开源项目；中大型需求开发前先搜索是否有现成项目；不整合没有整合价值的插件；最终目标 = 编排出一个 **RSI（递归自我改进）循环**，让编程代理循环迭代优化自己。
>
> 本地已克隆并验证 CLI 入口：`fractal/`、`ralphex/`、`repo/`(ralph-orchestrator)、`hive-ref/`、`loop-engineering/`、`loopy/`、`looptroop-src/`。

---

## 0. TL;DR 一句话结论

**RSI 的核心不是"让代理改自己的代码"，而是"让一个外层 meta-loop 以切片化、可审计、可回滚的方式，每轮只改进循环的一个组件（提示词/计划/预设/技能/门槛），且改进必须通过独立验证器才能合入"。**

因此战略重心是：
- **不自研 agent 运行时**（11 个项目里已有多套成熟运行时，性能要求不高 → 直接 CLI 组合）；
- **自研三件小事**：① meta-loop 外壳（bash/sh 几十行 + 指纹僵局检测）；② Debrief→最小改进生成器（四因诊断）；③ 回执(receipt)+门槛门(gate)策略文件；
- **CLI 组合**：`ralphex`（任务循环）+ `ralph-orchestrator`（循环编排+多后端）+ `fractal`（层级节点+预算硬限）+ `loop-engineering` 的 `loop-*` 包（审计/预算/门槛/漂移检测）作为横切工具层。

---

## 1. 共同设计模式提炼（跨 11 个项目，10 个）

| # | 模式 | 核心主张 | 主要来源项目 | 在 RSI 中的用途 |
|---|------|----------|--------------|-----------------|
| P1 | **生成-验证角色硬分离** | 写代码的代理不能给自己打分；Auditor/verifier 是独立上下文、独立模型（交叉模型）；验证器默认 REJECT | LongHorizon-Harness（Manager/Executor/Auditor 三角色）；loop-engineering（Maker/Checker）；ralphex（codex 审 Claude 代码）；awesome：kodo（独立验证）、Ouroboros（受审自我修改） | RSI 内环每次自我修改必须过独立审查才能合入，防自我奖励 |
| P2 | **状态全外置（文件/git 即共享内存）** | 循环本身无状态：进度、计划、ledger、round 记录全部落在 markdown/git/JSONL；崩溃可恢复 | ralphex（状态外置到文件+不可逆信号契约）；how-to-ralph-wiggum（IMPLEMENTATION_PLAN.md 是唯一持久状态）；LongHorizon（round ledger 持久化+checkpoint）；ralph-orchestrator（PersistedLoopState 断点续跑） | RSI 的"自我改进历史"= git 仓库 + 每轮回执文件，可 diff、可回滚 |
| P3 | **每轮新鲜上下文 / 上下文清刷** | 宁可用文件重置 + 全新会话，不延续污染会话；失败→紧凑失败笔记 + git 重置 + 重跑 | awesome-ralph（每次干净上下文）；LoopTroop（context wipe 优于会话续命）；LongHorizon（Executor 每轮新鲜上下文重读）；how-to-ralph（主上下文=调度器，40-60% smart zone） | RSI 内环任务循环用"外部 while + 每轮新会话"（Ralph 模式），避免上下文漂移 |
| P4 | **机器强制的预算与硬限** | 不信任 agent 自我配合：per-run/per-iter/per-step 三层预算、树深/子节点上限在 spawn 时机器强制；token 成本可观测 | fractal（三层预算+10% reserve 收尾窗口+max_children/max_depth 硬执行）；loop-engineering（loop-cost token 预算）；ralph-orchestrator（MaxIterations/MaxRuntime/MaxCost）；ralphex（--session-timeout/--idle-timeout） | RSI 每轮自我改进消耗必须有机器上限，防止预算烧穿 |
| P5 | **收敛保护（僵局/震荡检测）** | HEAD+diff 指纹 N 轮不变即停；n-gram 检测死循环；分类的"卡住"原因 | ralphex（僵局检测）；ralph-orchestrator（LoopThrashing/LoopStale 枚举）；hive（n-gram 死循环检测） | RSI 内环与 meta-loop 双层都要收敛保护，防止无限自我折腾 |
| P6 | **命名终止态 + 可审计回执** | 循环结束必须是具名状态（Success/Blocked/Exhausted/No-progress/Approval-required...），并产出可复现回执（循环定义版本+SHA-256+验收条件+动作-证据）；创建者不自批 | loopy（六步反馈循环+回执+四因诊断 Debrief）；ralph-orchestrator（TerminationReason 八枚举+Landing/Handoff）；loop-engineering（失败故事强制入库：token 成本+根因+补救） | RSI 每轮自我改进的"证据"就是回执；Debrief 指导下一轮改什么 |
| P7 | **git worktree 隔离 + 合并队列** | 无人值守的代码改动实验全部在 worktree 里做；分支=节点的拓扑学；双速合并（节点内快合、跨节点慢合） | fractal（节点树=git worktree 树，squash-merge）；loop-engineering（loop-worktree 隔离）；ralph-orchestrator（并行 worktree+wave scatter-gather+合并队列）；ralphex（--worktree 隔离执行） | RSI 的每次自我改进 diff 都在隔离 worktree 中，验证通过才合入主线 |
| P8 | **机器策略门 + 漂移检测 + 逐级放权** | gate.yaml 是机器可解析策略（哪些改动允许/需人批）；策略与声明脱节时漂移检测报警；L1(报告)→L2(辅助)→L3(自动) 逐级毕业 | loop-engineering（gate.yaml+loop-sync 漂移检测+loop-gate+L1-L3）；ralph-orchestrator（预设策略库 review.yml/code-assist.yml）；fractal（requires_approval 步骤门控+父节点 approve） | RSI 的放权阶梯：人批→人抽检→自动合入，且循环的 gate 配置本身被漂移监控 |
| P9 | **切片化自我改进 + 质量棘轮** | 一次只改一个东西（一个 prompt/一个测试/一个模块）；Debrief 只推荐"能改变结果的最小改动"；已过的验收标准永不回退（棘轮） | loop-engineering（Post-Run Critique：每轮强制产出"改进下一轮的一个变化"；变异测试 loop-drill）；loopy（Debrief 四因诊断→最小改动；分离优化信号与验收门）；awesome：toryo（质量棘轮）；Ouroboros（受审自我修改） | RSI 的核心循环体：每轮恰好一个改进 diff + 棘轮保证单调不退步 |
| P10 | **最小单元分解 + 技能/记忆沉淀** | 工作拆成"最小可独立实现单元"(bead)，JSONL 持久化+依赖拓扑；好的做法沉淀为 skill/playbook/markdown 记忆，按结果召回 | LoopTroop（Beads 原子分解+LLM Council）；hive（工具门控习得技能 create_learned_skill、分域 markdown 记忆、先试点再系统化）；awesome：Mnemoverse（按结果召回记忆）、hermes-agent（技能沉淀） | RSI 长期价值 = 把验证过的改进固化成 skill/preset 文件（in-band 自我改进，不改核心代码） |

> 模式间关系：**P1+P9+P7 是 RSI 的骨架**（改进必须被独立审查、必须切片、必须隔离）；**P2+P6 是 RSI 的神经系统**（状态外置 + 回执让每轮可审计）；**P3+P5 是收敛保障**；**P4+P8 是安全边界**；**P10 是长期资产化**。

---

## 2. RSI 实现方向与路径假设（保守 → 激进）

### 路径 A（保守，推荐起步）：外置 meta-loop，"循环在环外"

> "Loop outside the loop"：任务循环完全交给现成 CLI，RSI 逻辑是它外面的一个薄 bash 脚本 + 一份改进生成提示词。

- **核心机制**：
  - 内环：`ralphex docs/plans/feature.md`（无人值守跑计划 → 内部 5 并行子代理审查 → codex 外部审查 → 收尾），信号协议 `RALPHEX:ALL_TASKS_DONE / TASK_FAILED / REVIEW_DONE`。
  - meta-loop（自研，~100 行 bash）：`while :; do 跑内环 → 落回执(receipt, loopy 式: 版本SHA+验收条件+退出枚举) → Debrief 四因诊断 → LLM 生成"下一轮恰好一个最小改进"(改 PROMPT.md / 计划模板 / gate 配置 / preset yaml) → 改进 diff 走 ralphex `--review` 交叉模型审查 + `fractal`/worktree 隔离 → 审查通过才 git commit → 指纹检测 N 轮无变化即停 → 重跑内环`。
- **使用哪些开源项目（CLI）**：`ralphex`（内环引擎+交叉审查）、`loop-engineering` 的 `loop-audit`/`loop-cost`/`loop-gate`（审计/预算/门槛横切工具）、`loopy` 的 SKILL.md（Debrief 直接作为提示词素材，纯文本引用即可）。
- **需要自研**：仅 ① meta-loop 外壳脚本；② 回执 schema（一个 JSON 模板 + 落盘逻辑）；③ 最小改进提示词 + 四因诊断模板。无自研运行时、无新服务。
- **风险**：改进粒度受"只能改提示词/配置"限制，天花板有限；依赖 ralphex 信号契约稳定。
- **收益**：两周内可跑通最小 RSI 闭环；每一轮改进可 git 审计、可回滚；完全符合"性能要求不高、CLI 优先"约束。

### 路径 B（中等）：in-band 技能沉淀（hive 模式）

> 代理在运行中把验证过的做法写成 skill/playbook/记忆文件（"工具门控习得技能"），meta-loop 负责审查与合并；自我改进发生在**文件层**而非**核心代码层**。

- **核心机制**：在路径 A 之上增加"试点→因式分解→系统化"（hive：Queen 亲自 pilot → 把成功做法抽成 skill+playbook → 后续轮次直接调用技能而非重新推理）；技能/记忆为分域 markdown，按任务结果召回；每次新 skill 合入必须过 P1 独立审查 + P8 门控（哪些 skill 类别允许自动合入由 gate.yaml 声明）。
- **使用哪些开源项目（CLI）**：路径 A 全部 + `fractal`（`fractal init` 提供项目级双知识库：node 私有 memory + 共享 wiki；`fractal radio` 作为跨节点消息面）、`hive`（`uv run hive` 作为可选运行时后端，Windows 下用 `hive.ps1`；主要借用其四级可靠性机制：park/resume 游标、结构保持 compaction、n-gram 死循环检测——以配置/参考方式实现，不嵌入其代码）。
- **需要自研**：① 技能仓库目录约定 + 工具门控加载器（读 manifest，只加载被白名单放行的技能）；② 四因诊断引擎（把 loopy 的 Debrief 提示词产品化为一个小 CLI：输入回执，输出"下一轮最小改动建议"）；③ 技能合入审查流水线（可复用 ralphex `--review`）。
- **风险**：技能/记忆污染（坏经验被反复召回）需要"按结果召回+定期淘汰"；hive 与 fractal 双知识库若混用易漂移。
- **收益**：循环能力随轮次**单调资产化**——技能库是比提示词持久、比代码修改安全的中间层；这是"自我改进"最健康的形态。

### 路径 C（激进）：层级分形自治改进（fractal 树 + 受审自修改）

> 把"改进循环自身"也变成节点：RSI 树 = 任务节点（执行功能）+ 改进节点（产出循环自身补丁，Ouroboros 式受审自修改），硬树限 + L1→L2→L3 放权。

- **核心机制**：`fractal` 的层级 node 树（子分支=父分支.node 名，worktree 树=分支拓扑）；改进节点 spawn 在根节点下，其 charter 是"提升 root 循环的下轮分数"；三层预算（per-run/iter/step）+ reserve 10% 收尾窗口机器强制；step 文件化（NN-*.md）使"改进循环的步骤"本身可被下一轮重写而不碰代码；双速合并：改进节点 squash-merge 回 trunk 必须通过 root 的 gate + 变异测试（`loop-drill`）+ 质量棘轮（分数不得低于基线）。
- **使用哪些开源项目（CLI）**：路径 B 全部 + `fractal` 全量（`fractal node init parser --max-cost 10 --max-iters 10 --timeout 24h` → `fractal node start` → `fractal node merge`）、`loop-drill`（护栏变异演练：注入坏变更，验证 gate 真的会 REJECT）、`loop-metrics`（分数曲线）。
- **需要自研**：① 改进节点 charter 模板 + "meta 改进 diff"的 schema（限改哪些文件：prompts/、presets/、gate.yaml、skills/，核心代码目录禁改）；② L2→L3 放权判定器（连续 K 轮 REJECT 率为 0 + 棘轮不回退 + 成本下降 → 自动升级）；③ 全系统回归基线（每 N 轮跑一次基准任务集，分数棘轮比较）。
- **风险**：自我修改安全（最危险路径，必须 P8 门控 + 变异测试 + 人工 L3 门）；fractal 依赖 tmux（Windows 需 WSL）；层级预算爆炸。
- **收益**：真正的"编程代理循环迭代优化自己"——循环自身的每一步都是可 diff、可审计、可回滚的文件，且权限逐级放权。

**路径关系**：A ⊂ B ⊂ C。战略上 A→B→C 逐阶段递进，每阶段都保持独立可交付。

---

## 3. 分阶段开发战略

### Phase 0 — MVP：单循环 + 独立验证 + 外置 meta-loop（路径 A，第 1–2 周）

**目标**：一条 bash 命令跑通"任务循环 → 交叉审查 → 回执 → 最小改进 → 合入"的完整 RSI 单圈，改进对象仅限提示词/计划文件。

**调用的开源项目（命令行）**：

```bash
# 内环引擎：ralphex（Go，go install github.com/umputun/ralphex/cmd/ralphex@latest 或 brew）
ralphex --init                                  # 生成项目 .ralphex/ 配置
ralphex docs/plans/feature.md                   # 任务循环+内部审查+codex 外部审查
ralphex --review --base-ref develop             # 对已有 diff 做多层审查（RSI 改进 diff 走这里）
ralphex --review-patience=3 --session-timeout=30m   # 僵局保护 + 挂起保护

# 横切工具层：loop-engineering（npm 包，可 npx 直接调）
npx @cobusgreyling/loop-audit . --suggest       # 循环成熟度打分 L0-L3（MVP 验证指标）
npx @cobusgreyling/loop-cost .                  # token/成本观测
```

**自研模块（小而薄）**：
1. `rsi/meta-loop.sh`（~100 行）：while 循环 + HEAD+diff 指纹僵局检测（P5）+ 退出码映射（P6 命名终止态）。
2. `rsi/receipt.json` schema + 落盘脚本：每轮记录内环版本 SHA-256、验收条件、RALPHEX 信号、token 成本、TerminationReason（P2/P6）。
3. `rsi/prompts/debrief.md`：四因诊断模板（loopy 素材直接引用）+ "只允许改 {prompts,plans} 目录、恰好一个 diff"的硬约束（P9）。

**验证标准（阶段完成判据）**：
- 给定 5 个基准任务（参考 `repo/bench/tasks.json` 的任务结构），RSI 闭环跑 3 圈后基准通过率 ≥ 无 meta-loop 的 ralphex 直跑基线；
- 每圈必产出一条完整回执，`git log` 可完整重放 3 圈的改进 diff；
- 故意喂入停滞场景（重复同一失败），meta-loop 在 ≤3 圈内被指纹检测触发 No-progress 终止态；
- `npx @cobusgreyling/loop-audit .` 分数较初版提升且 L1 达标。

### Phase 1 — 增强：门控策略 + 预算可观测 + 多后端（路径 A 完整形态，第 3–4 周）

**目标**：RSI 有"安全边界"——gate.yaml 机器策略、漂移检测、token 预算上限；支持 Claude/Codex 双后端交叉审查常态化。

**调用的开源项目（命令行）**：

```bash
# loop-engineering 策略工具（从 loop-engineering/ 本地构建或用 npx）
npx @cobusgreyling/loop-gate .          # gate.yaml 机器策略检查（哪些改动需人批）
npx @cobusgreyling/loop-sync .          # 策略-实际 漂移检测
npx @cobusgreyling/loop-init .          # L1 起步脚手架（gate.yaml/STATE.md 模板）

# ralph-orchestrator（Rust CLI，npm i -g @ralph-orchestrator/ralph-cli 或 cargo install ralph-cli）
ralph init --backend claude
ralph plan "RSI meta-loop v2"
ralph run -p "Implement in .ralph/specs/"     # 事件循环+hat 角色+记忆/任务
# 预设策略直接复用：repo/presets/review.yml、code-assist.yml、wave-review.yml
```

**自研模块**：
1. `rsi/gate.yaml` 生成器：由 Debrief 输出自动派生（改进类别→自动/人批 映射），并接入 loop-sync 漂移监控（P8）。
2. 成本看板（单文件 HTML，读 loop-cost 输出）：预算烧穿预警（P4）。
3. Debrief CLI 化：`rsi-debrief <receipt> → improvement-patch`，把最小改进从"提示词素材"变成可重放命令（P9）。

**验证标准**：
- 验证器默认 REJECT 率统计可查；gate.yaml 中"需人批"类别的实际拦截率 ≥ 90%（用一次故意越权改动测试）；
- 单轮 token 成本可预测（预算上限被机器触发 ≤ 1 次事故）；
- 双后端（Claude 执行 + codex 审查）跑通 3 个基准任务且互审记录入库。

### Phase 2 — 高级：层级节点 + 技能资产化（路径 B，第 5–8 周）

**目标**：RSI 从"改提示词"升级为"沉淀技能"：改进节点在 fractal 树上并行孵化，成功做法固化为工具门控技能（in-band 自我改进），主线质量单调棘轮。

**调用的开源项目（命令行）**：

```bash
# fractal（pip install plasma-fractal，Windows 需 WSL/tmux；无 tmux 环境用 --headless）
fractal init --agent claude
fractal commit "baseline" --init
fractal node init improver-1 --title "Improve meta-loop debounce" \
        --max-cost 10 --max-iters 10 --timeout 24h --max-children 2
fractal node start improver-1 --headless
fractal node cost remaining improver-1        # 三层预算观测
fractal node merge improver-1                 # 验证后 squash 回主线（合并队列）

# hive（借用其可靠性机制与技能模式；uv run hive，Windows 用 hive.ps1）
./hive open          # 仪表板观察
./hive run queen     # 试点任务（先 pilot 再系统化为 skill）
```

**自研模块**：
1. 技能仓库：`rsi/skills/<domain>/*.md` + `skills/manifest.yaml`（工具门控：只允许白名单工具加载；新技能合入 = 独立审查 + 棘轮基线通过）（P10）。
2. 双知识库接线（fractal 约定直接复用，不自研）：node 私有 memory + 项目共享 wiki。
3. 基准任务集 runner：`rsi-bench`（单脚本，调 bench/tasks.json，跑分写回执，分数棘轮比较——分数只升不降即质量棘轮 toryo 模式）。

**验证标准**：
- ≥ 3 个改进节点并行孵化并成功 merge（合并冲突率 < 10%）；
- 技能库命中率：第 3 轮后，同类任务调用已有技能的成功率 ≥ 60%；
- `rsi-bench` 分数连续 4 轮单调不降（棘轮），且成本/轮不升（同任务更快）。

### Phase 3 — 自治：受审自修改 + 逐级放权（路径 C，第 9 周起）

**目标**：允许 RSI 改进对象扩展到循环自身的**配置层**（presets/gate/prompt 模板/loop 外壳脚本），仍禁核心审查代码；L2→L3 放权自动化，变异测试守门。

**调用的开源项目（命令行）**：

```bash
# loop-engineering 护栏演练与度量
npx @cobusgreyling/loop-drill .     # 变异测试：注入坏 diff，验证 gate 一定 REJECT
npx @cobusgreyling/loop-metrics .   # 分数/成本曲线入回执
npx @cobusgreyling/loop-audit .     # L2→L3 毕业评分（每 N 轮自动跑）
# fractal 全量树管理
fractal node list --json && fractal radio read --channel inboxes --unread
```

**自研模块**：
1. **自修改 schema 与文件白名单**：meta diff 只能触碰 `prompts/ presets/ gate.yaml meta-loop.sh skills/`；白名单外路径在合并队列处机器拒收（Ouroboros 受审自修改 + loop-engineering"验证器默认 REJECT"）。
2. **放权判定器**：连续 K=5 轮 [REJECT 率=0 ∧ 棘轮不回退 ∧ 成本不升 ∧ 漂移检测静默] → 自动 L2→L3；任一违反回退一级并落失败故事（loop-engineering 失败故事规范：token 成本+根因+补救）。
3. **全系统回归基线**：每 N 轮跑 rsi-bench + loop-drill，失败即冻结自动合入（人工门）。

**验证标准**：
- 完成一轮**全自治** RSI 迭代：meta-loop 自主改进配置层 diff → 独立审查 → 变异测试 → 自动合入 → 分数上升，全程无人工批；
- 故意注入 3 种恶意/退化 diff（放宽验收标准、删除门禁、回退提示词），loop-drill + 棘轮 100% 拦截；
- `loop-audit` 达到 L3 且 gate 漂移检测连续 10 轮零告警。

---

## 4. 开源项目调用矩阵

| 项目（本地目录） | 形态 | 命令行直接调用 | 需要整合 | 纯方法论 | RSI 中的角色 | 采用阶段 |
|---|---|:--:|:--:|:--:|---|---|
| **ralphex** (`ralphex/`) | Go CLI（`go install`/brew/releases） | ✅ `ralphex docs/plans/x.md`、`--review`、`--worktree`、`--codex` | ❌ | — | 内环引擎 + 交叉模型审查 + 僵局/挂起保护（P1/P3/P5） | Phase 0 起 |
| **loop-engineering** (`loop-engineering/`) | 17 个 TS CLI 包（npx 直调，`@cobusgreyling/loop-audit` 等） | ✅ `npx @cobusgreyling/loop-{audit,cost,gate,sync,drill,metrics}` | ❌ | — | 横切工具层：审计打分/预算/门策略/漂移/变异测试（P4/P8/P9） | Phase 0/1/3 |
| **ralph-orchestrator** (`repo/`) | Rust CLI（npm/cargo install `ralph-cli`） | ✅ `ralph init/plan/run/web`、presets yaml | ❌（预设 yaml 直接复制进 gate 生成器参考） | — | 事件循环 + hat 角色 + 记忆/任务 + 分类终止枚举 + 可恢复断点（P2/P6） | Phase 1 |
| **fractal** (`fractal/`) | Python CLI（pip/uv，需 WSL 或 `--headless`） | ✅ `fractal init / node init / start / merge / cost / radio` | ❌ | — | 层级节点树 + 三层预算硬限 + 双速合并 + 双知识库（P4/P7/P10） | Phase 2 |
| **hive** (`hive-ref/`) | uv 包装 CLI（`uv run hive`，Windows `hive.ps1`） | ⚠️ 可跑（`hive open/run`） | ⚠️ 仅借机制（park/resume 游标、n-gram 死循环检测、技能/playbook 模式——以配置与自研薄层实现，不嵌其代码） | 部分（"one loop many loops"思想） | 试点→系统化流水线 + 工具门控技能（P10）；可选运行时后端 | Phase 2（参考为主） |
| **loopy** (`loopy/`) | 技能文件 + loop-library（无 CLI） | ❌ | ❌ | ✅ 直接引用 `loopy/skills/loopy/SKILL.md` 六步循环 + 命名终止态 + 回执 schema + 四因诊断 | Debrief 提示词素材 + 回执规范（P6/P9） | Phase 0 起（文本引用） |
| **LoopTroop** (`looptroop-src/`) | GUI+daemon TS 服务（`looptroop start/status`） | ⚠️ 有 server CLI 但核心是 GUI 本地编排引擎 | ❌（"slow and perfect" 定位与性能要求不高的 RSI 主线不匹配） | ✅ 借 Beads 原子分解 + 上下文清刷 + LLM Council | 备选：bead JSONL 最小单元格式（P10）、多模型议会（P1） | 仅参考 |
| **LongHorizon-Harness**（未本地克隆） | 框架库（多后端 harness） | ❌ | ❌（computer-use 长时程场景，与本项目无关，不引入） | ✅ 三角色（Manager/Executor/Auditor）设计直接作为自研审查流水线的蓝本（P1） | 验证器架构参考 | 设计参考 |
| **awesome-agent-orchestrators**（未本地克隆） | 分类清单 | ❌ | ❌ | ✅ 选型雷达：kodo/toryo/Ouroboros/Mnemoverse/hermes 的设计思想吸收 | 防重复造轮子的检索源（每阶段开工前跑一遍） | 持续 |
| **awesome-ralph**（未本地克隆） | 资源清单 | ❌ | ❌ | ✅ 文件结构规范（loop.sh+PROMPT_*.md+AGENTS.md≤60行）与 backpressure 理论 | Phase 0 仓库骨架规范 | Phase 0 |
| **how-to-ralph-wiggum**（未本地克隆） | 方法论 Playbook | ❌ | ❌ | ✅ 双提示词分离（build/plan）+ 上下文经济学 + 可丢弃计划 | meta-loop 提示词设计准则 | 全程 |

**矩阵结论**：
- **直接 CLI 调用（6 个）**：ralphex、loop-engineering 17 包、ralph-orchestrator、fractal、hive（可选）、LoopTroop server（仅参考不用）——满足"优先命令行"约束，**零新框架**。
- **需要整合（嵌入代码）**：**无强制项**。hive 的技能/park-resume 机制若自研，也只是"薄层实现参考"，不是代码级依赖。
- **纯方法论（9 个项目中占 5 个）**：loopy、LongHorizon、三个 awesome/how-to 类——全部以"提示词素材 + 设计蓝本 + 检索雷达"方式消化，**不增加任何运行时依赖**。
- 按"不整合没有整合价值的插件"约束：**LoopTroop 明确不整合**（GUI 定位+slow-and-perfect 与 RSI 主线冲突）；**LongHorizon 不整合**（computer-use 域错位）。

---

## 5. 关键假设与风险

### 关键假设（不成立则路径需修正，共 7 条）

| # | 假设 | 失败时的后果 / 备选 |
|---|------|--------------------|
| A1 | 现成 coding CLI（claude/codex）的**单次能力**足以作为 RSI 的基本单元——自我改进的增益来自"循环结构"而非"模型更强" | 若不成立：换更强模型或加 LoRA/上下文增强，路径不变（CLI 约束保证可替换） |
| A2 | **文件/git 是唯一可靠共享状态**（P2）：markdown+JSONL+git 足以承载循环状态，无需数据库 | 状态规模爆炸（>10k 回执）时引入 LoopTroop 的 JSONL 索引或 hive 的 SQLite tracker |
| A3 | **交叉模型独立审查**（Claude 生成 / codex 审查或反向）能有效防奖励黑客（P1） | 若两模型同源偏差：引入第三模型（gemini/hive council 式 LLM Council 投票，LoopTroop 素材） |
| A4 | **"恰好一个最小 diff"（P9）是自我改进的正确粒度**——改进收敛且可归因 | 若改进需要多文件联动：放宽为"一个 logical change"（可含 N 个文件但同一验收条件），仍保棘轮 |
| A5 | 预算/树限等**机器硬限可执行**（P4：fractal 三层预算、ralphex timeout） | 需要把 loop-cost 纳入 meta-loop 每轮前置检查（gate 先于 spawn） |
| A6 | **Windows 可用**：fractal 需 WSL（tmux）、hive 有 .ps1、ralphex/ralph CLI 原生 Windows | 备选：全部在 WSL2 内运行（一行 setup 脚本）；或用 ralph-orchestrator `--headless` 路线替代 tmux |
| A7 | 每 N 轮的**基准任务集**（rsi-bench）能真实反映"循环变强"——分数与代理能力单调相关 | 基准被循环针对优化（Goodhart）→ 定期轮换基准任务（loopy"分离优化信号与验收门"原则） |

### 风险（会阻断进度的 7 条，含缓解）

| # | 风险 | 概率 | 缓解措施 |
|---|------|:--:|----------|
| R1 | **自我奖励黑客**：循环学会放宽自己的验收标准而非真提升 | 高 | 验收门与优化信号分离（loopy）；验证器代码与配置白名单隔离（Ouroboros）；loop-drill 变异测试注入"放宽验收"恶意 diff 必须 100% 被 REJECT |
| R2 | **震荡不收敛**：改进 A → 分数降 → 回退 → 改进 B → 又降，来回打摆 | 高 | 质量棘轮（分数只升不降，toryo）+ HEAD/diff 指纹僵局 N 轮熔断（ralphex）+ 回退时强制 Debrief 换因（四因诊断不允许重复上次根因） |
| R3 | **预算爆炸**：层级节点 + 交叉审查使成本超线性增长 | 中 | fractal 三层预算 + reserve 10% 收尾窗口 + loop-cost 每轮观测 + L3 放权需成本不升作为前提（假设 A5） |
| R4 | **上游 CLI 破坏性变更**（ralphex 信号协议、ralph 预设格式） | 中 | 锁定版本安装（`go install @version` / `cargo install` 固定）；meta-loop 只依赖信号枚举，适配层单文件隔离 |
| R5 | **平台摩擦**（fractal tmux / hive bash 在 Windows） | 中 | WSL2 单命令 setup；`fractal node start --headless`；hive 用 `hive.ps1`（假设 A6） |
| R6 | **技能/记忆污染**：坏经验被沉淀后反复召回，负循环 | 中 | 技能合入=独立审查+棘轮通过；按结果召回（Mnemoverse 思想）+ 每 N 轮淘汰零命中技能；分域隔离（hive） |
| R7 | **自修改失控**（仅 Phase 3）：循环改了自己的 gate/审查配置 | 低（设计期已隔离） | 文件白名单机器拒收白名单外路径；gate 本身只允许"收紧"不允许"放宽"（单向棘轮）；L3 前人工门常开；失败故事强制入库 |

### 止损线（Stop-loss）

- Phase 0 若 3 圈内基准通过率 ≤ 无 meta-loop 基线：暂停自动化，降级为"人读 Debrief、人做改进"（保留回执规范，这是最低成本资产）。
- Phase 3 若恶意 diff 拦截率 < 90%：**永久停留在 L2**（人批合入），不追求自治——安全优先于自治。

---

## 附：目录约定（awesome-ralph 规范 + 本项目扩展）

```
rsi-loop/
├── meta-loop.sh            # 自研：RSI 外壳（Phase 0）
├── prompts/
│   ├── PROMPT_plan.md      # 双提示词分离（build/plan）
│   ├── PROMPT_debrief.md   # 四因诊断+最小改进生成
│   └── AGENTS.md           # ≤60 行循环行为契约
├── docs/plans/             # ralphex 计划（IMPLEMENTATION_PLAN.md 唯一持久状态）
├── receipts/               # 每轮回执（loopy schema + git 可重放）
├── gate.yaml               # 机器策略（loop-gate/loop-sync 监控）
├── skills/                 # Phase 2 技能仓库 + manifest（工具门控）
├── presets/                # ralph-orchestrator 预设（review.yml 等）
├── bench/                  # rsi-bench 基准任务集
└── stories/                # 失败故事（token 成本+根因+补救，loop-engineering 规范）
```

**下一步（Week 1）**：按 awesome-ralph 规范搭骨架 → `ralphex --init` + `npx @cobusgreyling/loop-audit . --suggest`（缺口建议直接照抄）→ 写 `meta-loop.sh` 单圈跑通 → 产第一条回执。
