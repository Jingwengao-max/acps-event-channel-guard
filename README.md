# 智能体三维事件通信风险检测器

Three-Dimensional Agent Event Communication Guard

## 1. 模块用途

本模块是面向 ACPs 智能体互联场景的只读旁路安全审计原型，用于发现传统身份认证和权限控制难以识别的异常通信行为。

它重点回答：

> 一个智能体即使拥有合法 AIC、合法调用权限和有效委托令牌，其跨任务通信行为是否仍然异常？

模块从三个维度分析任务事件：

- **Storage**：异常载荷大小、长 Base64 高熵数据、重复载荷；
- **Timing**：过度规则的调用间隔、稳定长短交替模式；
- **Behavioral**：状态倒退、终态恢复、机械式事件切换。

 文件说明

| 文件 | 用途 |
|---|---|
| acps_cluster/covert_event_guard.py | 核心检测器 |
| acps_cluster/tests/test_covert_event_guard.py | 自动化测试 |
| acps_cluster/examples/covert_guard_normal.jsonl.example | 正常事件样例 |
| acps_cluster/examples/covert_guard_suspicious.jsonl.example | 构造异常样例 |
| docs/covert-event-guard.md | 详细使用与规则说明 |
| evidence/unit-tests.txt | 单元测试结果 |
| evidence/real-peer-timing.txt | 真实历史通信回放结果 |
| evidence/positive-control.txt | 构造阳性对照结果 |
| evidence/sha256.txt | 核心代码哈希 |

evidence 目录仅在交付包包含验证证据时存在。

## 4. 环境要求

- Python 3.10 或更高版本；
- 仅使用 Python 标准库；
- 无需安装额外依赖；
- 可独立测试，也可放入 AIC-agent-connect 仓库运行。

建议使用 -B，避免生成 .pyc 文件。

## 5. 快速验证

在包的根目录运行：

    python3 -B -m unittest discover \
      -s acps_cluster/tests \
      -p 'test_covert_event_guard.py' \
      -v

预期结果：

    Ran 12 tests
    OK

正常样例：

    python3 -B -m acps_cluster.covert_event_guard \
      --input acps_cluster/examples/covert_guard_normal.jsonl.example

预期：

    Risk: LOW (0/100)

异常样例：

    python3 -B -m acps_cluster.covert_event_guard \
      --input acps_cluster/examples/covert_guard_suspicious.jsonl.example

预期：

    Risk: HIGH (73/100)

## 6. 检测真实任务

检测单个任务：

    python3 -B -m acps_cluster.covert_event_guard \
      --db /path/to/acps/aip-tasks.sqlite3 \
      --task-ref task_xxxxxxxxxxxx

跨任务 Timing 检测：

    python3 -B -m acps_cluster.covert_event_guard \
      --db /path/to/acps/aip-tasks.sqlite3 \
      --peer-timing \
      --limit 1000

输出 JSON：

    python3 -B -m acps_cluster.covert_event_guard \
      --db /path/to/acps/aip-tasks.sqlite3 \
      --peer-timing \
      --limit 1000 \
      --json

数据库使用 SQLite mode=ro 打开，检测器不会写入任务数据库。

## 7. Timing 规则

跨任务检测按以下键分组：

    direction + local_aic + remote_aic

规则边界：

- 相邻任务间隔超过 3600 秒时开启新会话；
- 少于 4 个有效间隔：INSUFFICIENT_DATA；
- 4–5 个有效间隔：只观察，不计分；
- 至少 6 个有效间隔：允许参与 Timing 评分；
- 时间间隔变异系数 CV 不超过 0.10：增加 20 分；
- 稳定长短交替模式：增加 25 分；
- 小于 1 秒的间隔：标记为 operational_burst，不增加安全风险分。

Peer Timing 风险分级：

- 0–19：LOW；
- 20–39：MEDIUM；
- 40–100：HIGH。

## 8. 当前验证结果

- 12 项自动化测试全部通过；
- 对 140 个真实历史任务进行了只读基线分析；
- 真实任务每个仅包含 3–5 个事件；
- 三个主要通信方分别有 84、30、11 次调用；
- 共形成 7 个真实可分析会话；
- 真实跨任务 Timing 检测未产生安全告警；
- 观察到的亚秒级并发被标记为 operational_burst，未计入风险；
- 构造的规则时间与交替时间组合被识别为 HIGH 45/100；
- 单任务三维异常样例被识别为 HIGH 73/100。

上述结果只能说明当前历史样本中的观测情况，不能表述为经过完整标注数据集验证的 0% 误报率或 100% 检出率。

## 9. 安全边界

当前模块：

- 不读取或输出 API Key、完整令牌和私钥；
- 不执行载荷中的代码；
- 不修改任务数据库；
- 不修改心跳或 Kafka 转发；
- 不自动阻断任务；
- 不自动撤销委托令牌；
- 不自动冻结协作域；
- 不将风险结果视为攻击定论。

检测结果用于异常筛查，生产处置必须结合任务语义与人工复核。

## 10. 推荐接入方式

建议合入 AIC-agent-connect 的 Supervisor 治理层，但首先以独立旁路服务运行：

    Supervisor任务数据库
              ↓
    定时只读检测
              ↓
       风险报告/告警
              ↓
         人工复核
              ↓
    必要时再联动PEP处置

第一阶段不建议将单次 HIGH 结果直接用于自动冻结、阻断或撤销授权。

## 11. 当前状态

当前版本属于经过真实数据回放验证的安全原型，适合：

- 代码审查；
- 历史任务审计；
- 答辩或演示；
- Supervisor 旁路集成验证。

尚未接入生产 Supervisor 主流程
