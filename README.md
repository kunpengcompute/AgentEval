# AgentEval

**确定性、证据可密封的 agent 轨迹评测框架。**

Deterministic, evidence-sealed agent trajectory evaluation built on Harbor.
（experimental · v0.3.0 · Apache-2.0）

> 项目名 **AgentEval**（本仓库）；发行包名是 `aeval-harbor`，CLI 与 import 名仍是 `aeval`。
> 下文与文档中出现的 `aeval` 均指同一个项目。

[![判分链路](docs/images/architecture.png)](https://gitcode.com/boostkit/AgentEval/blob/master/docs/diagrams/eval-chain.html)

> 交互版架构图（含引导视图 / 明暗主题 / 导出）：[在 GitCode 打开架构图](https://gitcode.com/boostkit/AgentEval/blob/master/docs/diagrams/eval-chain.html)

aeval 把执行委托给 Harbor（PyPI `harbor[e2b]==0.23.0`），自己只做一件事：**让分数可信**。

- 每份采集产物逐一 sha256 校验，并绑定运行时锁；证据门不通过就整条拒绝（fail loud）。
- 密封 bundle 可用 `recompute` 独立复算，篡改即报错。
- 判分分 outcome 与轨迹两层：阈值折叠、违规 veto 终裁，不做简单加权平均。
- 可靠性看 pass^k（k 次全过才算数），不看"跑一次过了"。
- 每个试次逐位记录六项前置事实（input_complete / agent_finished / integration_valid /
  render_valid / judge_finished / artifact_schema_ok），套件在 `verdict.requirements`
  里声明哪几项必须成立。该成立却没成立，这条试次就记 `cannot_judge` 并退出有效分母：
  判不了既不算通过，也不算做错。报告与复算可以逐条核对这六位。

## 安装

```bash
pip install aeval-harbor    # 发行名 aeval-harbor（PyPI 上 aeval 已被停更包占用）；CLI 与 import 名仍是 aeval
```

Python ≥ 3.12 · 变更见 [CHANGELOG](https://gitcode.com/boostkit/AgentEval/blob/master/CHANGELOG.md) ·
0.x 实验期：minor 版本可能有破坏性变更，都会在 CHANGELOG 里列出。想先试试？直接跑下一节的
离线演示，从源码运行、不用安装。

## 30 秒体验（离线，零外部依赖）

不需要 docker / e2b / 模型 API key。仓库自带一条完整的离线判分链路，从"采集完成"那一点
切入，之后每一环都是生产代码，没有一处 mock：

```bash
git clone https://gitcode.com/boostkit/AgentEval.git && cd aeval
uv run python examples/offline-chain/run_offline_chain.py   # 跑完自动在浏览器打开 dashboard
```

> 没有 [uv](https://docs.astral.sh/uv/)：先 `curl -LsSf https://astral.sh/uv/install.sh | sh`
> （Python ≥ 3.12 它也能代管）；或传统方式 `python -m venv .venv && .venv/bin/pip install -e .`
> 再 `.venv/bin/python examples/offline-chain/run_offline_chain.py`。
> 加 `--no-open` 跳过自动打开浏览器（SSH / CI 环境打不开时会退回打印路径）。

一次产出 4 个 run 的报告（Markdown）、4 份 dashboard（自包含 HTML）、1 份轨迹面板，以及
一份覆盖索引 `out/index.html`（逐套件核对：声明过的族和守卫是否都出现在报告里。本仓库
26 项全过）。其中会话套件那个 run 有 23 任务 × 3 试 = 69 条判定（pass@3 0.9998、
pass^3 0.7952、红线告警）。演示刻意覆盖了判分标准的每条路径：

| 路径 | 演示用例 |
|---|---|
| 干净通过 | hello-world 前两试 |
| 质量阈值折叠判 fail | hello-world 第三试（冗长 / 低遵循，reward=1 也翻车） |
| 反作弊 veto（守卫） | sqlite-db-truncate：agent 偷看验证器，reward=1 但 ForbiddenAccess ⇒ 终判 fail |
| 安全族 veto | openssl-selfsigned-cert：agent 把自己的私钥 cat 回会话，reward=1 但泄露密钥材料 ⇒ 终判 fail |
| 真实失败 | openssl-selfsigned-cert（证书 CN 写错，reward=0） |
| 基础设施故障排除 | 显式排除记录，不进有效分母 |

## 面板导览

跑完上面 30 秒体验，`examples/offline-chain/out/` 里就有全部成品。三类产物都是
**自包含单文件 HTML**（零外部资源，发给别人就能打开）：

| 产物 | 命令 | 看什么 |
|---|---|---|
| 运行面板 `dashboard-*.html` | `aeval dashboard` | 一个 run 的全局视图 |
| 轨迹面板 `trajectory-*.html` | `aeval trajectory` | 单个任务的执行下钻 |
| 覆盖索引 `index.html` | 演示脚本生成 | 声明过的东西是否都出现在报告里 |

一条共同约定：**面板是第二渲染器，不是第二数据源**。它和 `aeval report` 用同一份聚合结果
（同一份 `RunSummary`、同一判分器版本、同一个有效分母），所以两边的数字一致。

### 运行面板 · `aeval dashboard`

一个 run 的全局视图：pass@k / pass^k、综合评分、红线状态、判定分布与排除明细、
token / 时长 / 工具调用统计。

```bash
aeval dashboard --store examples/offline-chain/out/aeval-intel-1/store.sqlite3 \
    run-aeval-intel-1 > dash.html
```

demo 对四个 run 各出一份，数字都来自该 run 的密封 store：

| 面板 | run（套件 · 契约） | 内容 |
|---|---|---|
| `dashboard-aeval-intel.html` | aeval-intel 0.6.0 · intel | 旗舰：69 条判定（pass@3 0.9998 · pass^3 0.7952 · 综合 0.9407 · 红线告警）；4 个族共 16 项声明指标；demo 跑完自动打开的就是它 |
| `dashboard-tbench-intel.html` | tbench-intel 0.4.0 · intel | 同一批 tbench 任务换成扩展契约：族指标 4+2 项、守卫 2+9 项；阈值折叠 + veto ⇒ 4 通过 / 4 失败；排除率 11.1% 超标，顶部写明"分数不可单独作为可信信号" |
| `dashboard-tbench-offline.html` | tbench-pilot 0.7.0 · offline | 基线契约：outcome 层 + 轨迹守卫 9 项。2 个失败试次（基线判错 1 + 守卫 veto 1），就是 pass@3 1.0000 与 pass^3 0.3571 的全部落差 |
| `dashboard-sbench-offline.html` | sbench-pilot 0.7.0 · offline | 服务自查 89 例（outcome-only，pass@1 0.7753 · 综合 0.7795 · 红线告警）；12 个族只分组、不声明指标 |

两种契约的差别在判分层：**offline** 到 outcome 层为止（外加声明式轨迹守卫）；**intel**
再叠会话质量 4 项与安全 2 项（阈值折叠 + veto）。表里后两行是同一批 tbench 任务的两个 run，
天然可对照：同样的 9 条试次、同样的基线，防线从"看轨迹"升到"看会话"。

#### 怎么读

面板从上到下分这些部分（雷达、四象限、热力图、短板是同一份数据的四个视图）：

| 节 | 一句话 |
|---|---|
| 顶部结论条 | 综合评分、整体通过率、红线状态；只算**有效分母**，红线告警就标红，不因总分高而消掉 |
| 判定分布 / 排除明细 | 四类计数；排除的不算通过也不算失败，排除率 > 5% 时顶部警告 |
| 轨迹采集 | 只统计密封证据里的 token / 时长 / 工具调用，缺字段就标注 |
| 能力雷达 · 四象限 · 热力图 · 短板 | 同一份"维度达标度"的四个视图；轴不足 3 个时雷达改画条形 |
| 按类别结果 | 一族一行；「判定分工」说明这族的失败是本族判的，还是别的层翻的 |
| 终判来源分解 | 每个失败试次：谁判的（基线 / 族 / 守卫）+ 用什么机制（判定 / 阈值折叠 / veto）|
| 终判结果 | 逐任务的 试次 / 有效 / 通过 / 失败 / pass^k；单任务 run 不重复出这张表 |
| 逐项指标 | 唯一带指标的节，形态随契约变，见下 |

「逐项指标」有三种形态，四个 demo 各占一种：

- **有族指标**（aeval-intel、tbench-intel）：每个族一张指标表，按声明顺序列出 判定 ok /
  降级 / 违规 / 跳过 / 跳过原因；每个任务再跟一行"这个任务判了哪几项 · 跳过 N 项"。
- **指标都在守卫行**（tbench-offline）：族只分组，9 项轨迹指标全在守卫表里，一行一项。
- **没有指标级判定**（sbench-offline）：这一节写明原因（套件没声明指标、判定由官方基线
  给出），并指向「终判结果」和「按类别结果」。

这一节的两条写作规则，也是整套面板的原则：

1. **同一件事只写一遍**：基线与「终判结果」同值的行不再复列；声明清单不另抄一行（指标表
   本身就是按声明顺序排的）；任务行只写判过的指标，被跳过的名字不重复列。
2. **没有的东西要说清为什么没有**：声明了却没判到的指标显示 0 计数；整个 run 没有指标级
   判定时这一节直说原因——**它永远不会静默消失**。

指标以 `中文(id)` 展示（如 `歧义澄清(clarification)`），机器 id 始终同时在场。显示名只影响
渲染，不进密封声明、不改 run 身份，所以改名之后 `aeval recompute` 仍然逐字节一致。

[![intel 运行面板](docs/images/dashboard-aeval-intel.png)](https://gitcode.com/boostkit/AgentEval/blob/master/docs/images/dashboard-aeval-intel.png)

*intel 运行面板（旗舰；demo 自动打开）*

[![offline 运行面板](docs/images/dashboard-sbench-offline.png)](https://gitcode.com/boostkit/AgentEval/blob/master/docs/images/dashboard-sbench-offline.png)

*offline 运行面板（sbench 服务自查 89 例，outcome-only）*

### 轨迹面板 · `aeval trajectory`

单个 task 的下钻视图：加载全部密封试次，逐份 transcript 对 sha256 校验；主视图是
**五轨联动执行视图**——执行、工具、Token、输入 Token 与上下文压力五条轨道共享横轴
（纵向轨道只作对比，不代表并发），各试次按自己的起点对齐相对执行时间；记忆基底的轮次
（fork base）在执行轨上画成虚线。点任一标记同步高亮同一步，右侧检查器展示消息、工具观测
与 turn 判分理由；轨迹太密时自动按密度分桶，可以继续下钻。

```bash
# 单任务 → stdout（重定向保存）
aeval trajectory --store examples/offline-chain/out/aeval-intel-1/store.sqlite3 \
    run-aeval-intel-1 memory.tenant_isolation > task.html

# 批量：run 下每个 task 各一份自包含面板 → <dir>/<task>.html
aeval trajectory --store examples/offline-chain/out/aeval-intel-1/store.sqlite3 \
    run-aeval-intel-1 --out /tmp/aeval-trajectory/
```

两条可选参数：`--context-window N` 由调用方声明上下文窗口（tokens），优先级高于证据自己
携带的窗口（provider 声明经 request / context 事件密封进会话）；两者不一致或缺失时如实
标注，不臆造。`--suites-dir` 指向套件根以装载 turn 指标，不给的话逐轮打分退化成结构切面。

它的展示逻辑也只看三条：

- **只画密封证据**：每份 transcript 先按采集清单对 sha256，缺失或损坏就标注，不补画。
- **不重新判分**：turn 判分复用轨迹判分的同一套指标对象（`turn_metrics(task_id)`），
  面板里没有第二套判分逻辑。
- **缺参数就标注**：上下文窗口按上面的顺序取，取不到就写明缺，不编一个占用率出来。

demo 用一个代表性任务（fork 记忆基底 + 一个坏试次 + 工具调用）走真实 CLI 产出
`out/trajectory-memory.tenant_isolation.html`：3 试 × 5 步，标记只有 34 个，适合先看懂结构。

下面这张是真实 run 的同一视图：`run-tbench-pilot-docker` · `sqlite-db-truncate`，3 个试次
36/36/26 步、44 次工具调用、239 个标记。轨迹一密，五轨联动才有用——哪一步慢、哪次调用
没有观测、token 在哪一轮冲高，都在图上看得到。（截图来自真实 run，仓库内不可复现。）

[![轨迹面板](docs/images/trajectory-panel.png)](https://gitcode.com/boostkit/AgentEval/blob/master/docs/images/trajectory-panel.png)

*任务轨迹面板（五轨联动执行视图 + 详情检查器）· 图为真实 run 的 `sqlite-db-truncate`，3 试 × 98 步*

### 覆盖索引 · `out/index.html`

一行一个套件，把两端摆在一起对照：一端是密封声明（`manifest.dimension_model` 里的族 /
守卫 / 基线），另一端是该套件自己的报告文本。缺一项落点就直接报错退出，不是打个警告。
本仓库 26 项全过。

| 列 | 含义 |
|---|---|
| 试次 / 有效 / 通过 | 与面板同源；有效 = 去掉 `cannot_judge` 与 `infra_invalid` |
| 族 / 守卫 | 该套件声明的族数（含只分组的族）与守卫数 |
| 逐项指标 | 报告里真正出现的指标行数（族块 + 守卫）；没有指标级判定的 run 就是 0，报告里会写明原因 |
| 落点 | 判定族的落点在「逐项指标」，只分组族的落点在「按类别结果」，两者都算有落点 |

一句话分工：**运行面板回答"这个 run 判成了什么"，覆盖索引回答"声明过的东西是不是都出现
了"**。两者都只做渲染和核对，都不重新判分。

## 为什么用 aeval

- **确定性判分，不是 LLM 拍脑袋**：证据门逐产物 sha256 校验，不通过就整条拒绝；判分器
  按 `graders/outcome.py@v7` 这样的版本引用并自证身份（`GRADER_ID` 非空、入口是协程、
  `LAYER` 与声明层一致）；outcome 与轨迹分层，阈值折叠、veto 终裁。六项前置事实是真正的
  门：套件声明哪几项必须成立，没成立就 `cannot_judge` 并退出有效分母。
- **防篡改证据链**：采集清单绑定运行时锁、逐产物 sha256；`recompute` 独立复算密封
  bundle；`rejudge` 在前置条件不满足时拒绝执行，而不是悄悄重判。
- **pass^k 可靠性**：k 次尝试次次通过才算数，报告同时给 pass@k 与 pass^k。
- **可复现**：虚拟时钟（`clock: virtual_offset`）、环境基线探针（baselines）、
  observables 声明式采集。
- **agent 中立**：`new-agent` 脚手架 + `conformance` 一致性测试，已带 DSH、deepagent、
  OpenAI-ACP 三个适配器。接入新 agent 先过一致性测试再进评测。
- **评测框架自己先被评测**：`aeval selftest` 做故障注入，每类该拦的失败都必须被拦下。

## CLI 一览

| 命令 | 作用 |
|---|---|
| `aeval list` | 列出发现的套件 |
| `aeval check` | (套件, agent) 配对可行性；只读，什么都不建 |
| `aeval run` | 校验 + 组合 Harbor job，并委托 Harbor 执行 |
| `aeval explain` | 渲染套件的只读组合视图（产物，不是输入） |
| `aeval probe` / `job` | 验证 agent 声明 / 为声明的 agent 组合 job |
| `aeval agents` / `new-agent` / `conformance` | 列出适配器 / 脚手架新 agent / 一致性验证 |
| `aeval import` / `export` | harbor-task 格式套件双向转换 |
| `aeval report` / `dashboard` / `trajectory` | Markdown 报告 / 运行面板 / 单任务轨迹面板 |
| `aeval recompute` | 独立复算密封 bundle（篡改即报错） |
| `aeval rejudge` | 受保护的重判入口：前置不满足即拒绝 |
| `aeval selftest manifest` / `isolation` | 故障注入自检（必须全部拦截） |

## 套件（suite）

一个套件就是一个目录：`suite.yaml` + datasets + jobs + graders + tasks。核心是判分契约：

```yaml
schema_version: 2
id: refund-policy
version: 1.4.0
baselines:                                    # 环境基线探针：跑前先验证环境
  - { id: orders_seeded, probe: "db:SELECT count(*) FROM orders", equals: 120 }
clock: { mode: virtual_offset, epoch: "2026-09-16T00:00:00+08:00" }   # 可复现时间
observables:                                  # 声明式采集
  - { name: order_status, type: string, source: "db:orders.status where id=$order_id" }
verdict:
  requirements: [input_complete, agent_finished, integration_valid,
                 render_valid, judge_finished, artifact_schema_ok]
  graders:
    default: { impl: "graders/outcome.py@v7", layer: outcome }   # grader 版本化
metrics:
  - { id: reliability, kind: pass_pow_k, k: 5 }                  # pass^k
provenance: { source: authored-internally, license: MIT }
```

[![套件 / 族 / 任务 / run 的装配关系](docs/images/suite-task-run-family.png)](https://gitcode.com/boostkit/AgentEval/blob/master/docs/images/suite-task-run-family.png)

*装配关系（上图）里的几条要点：*

- `suite.yaml` 声明官方基线（`verdict.graders.default`，恰好一个 outcome 判分器）、守卫
  （`verdict.guards`）和激活的指标族（`families/<族键>.yaml`）。
- 每项判分指标只在 `verdict.metric_catalog` 定义一次（层 / 严重度 / 角色），族与守卫按
  `id` 引用它；逐任务的输入写在族里。
- 族的适用范围由指标的输入派生（`metrics[].inputs` 里的任务键），`scope` 只放显示名和
  角色，所以没有单独的"用例"清单。
- 族用 `task_id` 引用 Harbor 的任务（`tasks/<task_id>/` 冻结，aeval 不改一个字节）。
- 一次 `run` 把套件身份和维度模型封进 manifest；每条试次按任务装配「基线 + 守卫 + 声明
  它的族引擎」，族引擎的判定汇总成族行。
- 一份报告只聚合同一个套件：跨套件聚合被 CLI 拒绝；同套件的多个 run 只在可比时合成一行
  （同一把锁、同 agent、同执行参数的重复跑），换引擎 / agent / 参数就各自成行。
- 族行附带「判定分工」（本族决定 / 他层决定）；「终判来源分解」逐试次说明每个失败是谁判的
  （基线 / 族阈值 / 族 veto / 守卫 veto）。基线与守卫不出族行，只折叠成试次终判。

仓库自带套件（`aeval list`）：

| 套件 | 内容 |
|---|---|
| `tbench-pilot` | terminal-bench 试点（含镜像摘要锁定与来源说明） |
| `aeval-intel` | 会话智能 10 任务 + 跨会话记忆 13 任务（含红线注入：回显、密钥泄露、跨租户越权等） |
| `sbench-pilot` | 服务自查 12 类 89 例（outcome-only 契约） |
| `deepagent-hello` / `deepagent-budget` / `e2e-hello` | deepagent 与端到端冒烟 |
| `_base` | 共享基座（继承源） |

服务类用例来自共享用例库 [`cases/`](https://gitcode.com/boostkit/AgentEval/blob/master/cases/README.md)：
12 个服务类别，套件在自己的 `cases.yaml` 里声明要注入哪些——用例事实与判分契约解耦。

## 接入新 agent

```bash
aeval new-agent --id my-agent     # 生成适配器骨架 + 声明（reader 故意 NotImplementedError）
# …实现你的 transcript reader…
aeval conformance --agent agents/my-agent.yaml   # 一致性验证：跳过的检查如实报告为 skipped，绝不谎报 passed
aeval check --suite <suite> --agent my-agent
```

## 使用者指南

三份自足的指南，从零讲清三件事（不需要先读源码或设计文档）：

| 指南 | 回答什么 |
|---|---|
| [写一个评测套件](https://gitcode.com/boostkit/AgentEval/blob/master/docs/guides/writing-a-suite.md) | 套件目录结构、最小 `suite.yaml`、全字段参考、继承与合并规则、哪些事实不许写进套件、校验与运行命令、常见错误 |
| [接入一个新的 agent](https://gitcode.com/boostkit/AgentEval/blob/master/docs/guides/adding-an-agent.md) | 两条接入路径（零代码 / 写适配器类）、五步走、声明与实现必须一致、预算声明、模型协议、运行时镜像、三条红线、排错表 |
| [指标语义与判分规则](https://gitcode.com/boostkit/AgentEval/blob/master/docs/guides/metric-semantics.md) | 报告里每个状态词和每个数字的确切含义：判分分层、六项前置事实门（未成立即 `cannot_judge`）、全部轨迹指标的公式与阈值、折叠与 veto、pass@k 与 pass^k、怎么读报告 |

## 成熟度（诚实说明）

- ✅ **离线判分链路**：任何机器可复现（见 30 秒体验），判分 / 密封 / 存储 / 报告全部是生产代码。
- ✅ **静态组合与验证**：`check` / `probe` / `job` / `explain` / `conformance` / `selftest`。
- 🚧 **真实沙箱全链路**：目标环境是 Linux 主机 + 自托管 ARM e2b 沙箱。这条链路已在真机上
  跑通并封存：真实 provider 凭据下，沙箱里的真实 agent 完成多步推理与工具调用，运行
  `exit 0`、`sealed: 1 trial(s) recorded, 34 file(s) attested, recompute passed`，
  终态判分产出计分 verdict（`score 1.0 / status pass`）。离线链路不依赖这套环境。
  ⚠️ 真机验证是在我们自己的拓扑上做的（自托管 e2b 集群 + 内网宿主机），换环境需要按你
  自己的网络与凭据重新验证。
- 版本号 0.3.0，接口可能变化；`harbor` / `e2b` 依赖有精确锁定（原因见
  [pyproject.toml](https://gitcode.com/boostkit/AgentEval/blob/master/pyproject.toml) 注释）。

## 仓库结构

```
src/aeval/
├── cli.py            # 16 个顶层命令 + selftest 故障注入组
├── suite_loader/     # 套件加载 · 继承 · 组合 · 校验 · 转换
├── verdict/          # requirements 门 · grader 加载 · 轨迹指标 · 折叠 · veto
├── hooks/            # 采集器 · 证据门（verify_evidence_bundle）
├── bundle/           # 密封 · attestation · recompute
├── store/            # TrialStore（SQLite）· 产物管理
├── metrics/          # pass^k · report · dashboard · 轨迹面板
├── agents/           # 适配器契约 · conformance · dsh/deepagent/ACP/testing
└── control/          # 控制面引导 · broker · flavors
suites/               # 评测套件（见上表）
cases/                # 共享服务用例库
control/              # aeval-control：agent 中立的宿主控制面（Node）
examples/offline-chain/  # 30 秒离线演示
```

## 相关项目

- **dsh-eval-control**：
  DSH 形态的宿主侧控制插件（实验变量注入、网关租约预算、fork 血统、bundle descriptor）。
  `aeval run` 的 DSH flavor 通过 `deploy_control_stack` 部署它；官方 session reader 从
  `node_modules/dsh-eval-control` 或 `AEVAL_DSH_SESSION_READER` 环境变量发现。
- **`control/`（aeval-control）**：从 dsh-eval-control 抽出的 agent 中立控制面
  （host broker + gateway lease），各 agent 的控制栈从这里组合部署产物。

## 开发

```bash
uv sync
.venv/bin/pytest                 # 单元 + 集成（含 hypothesis 性质测试）
.venv/bin/aeval selftest manifest   # 故障注入自检：必须全部拦截
.venv/bin/aeval selftest isolation
```

集成测试里的 DSH bridge 用例需要 PATH 上有 Node（`^22.19 || >=24`，dsh-eval-control 的
运行时）；缺失时这些用例会以 `node runtime not found` 失败。

## 发布

```bash
uv build && uvx twine check dist/*   # hatchling 锁 1.27.x——1.28+ 产出 PyPI 尚不识别的 Metadata 2.5，见 pyproject 注释
uv publish dist/*                    # 需要 PyPI token（如 UV_PUBLISH_TOKEN 环境变量）
```

## License

Apache-2.0，见 [LICENSE](https://gitcode.com/boostkit/AgentEval/blob/master/LICENSE)。
