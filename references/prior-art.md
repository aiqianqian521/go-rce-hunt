# 同类项目与可借鉴设计（prior art）

写这个 skill 之前做过一轮 GitHub 实测（2026-11-13，GitHub REST API），结论记在这里，避免下次重新调研。

**完整清单**：`Go-RCE-Skill-GitHub-参考清单.md`（工作区，含全部 star/活跃度数据）。

---

## 一、最该读的一个：cloudflare/security-audit-skill

71k star，2026-09 更新。六阶段流水线，是"AI 审计"从"让模型看代码"变成"有状态流程"的范本。

```
1 Reconnaissance    → architecture.md + coverage-ledger.json
2 Coverage hunting  → 从账本单位分配隔离 hunter，coverage critic 找缺口
3 Candidate validation → 每个候选交给全新 verifier，任务是证伪
4 Structured output → findings.json 三态 + report-schema.json 校验
5 Record verification → 全新 agent 复核源码声明
6 Reporting         → 从已验证记录派生 REPORT.md
```

**四条可直接搬的规范**：

1. **三态输出，且中间态不给 severity**
   - `confirmed`：完整源码 trace + 有界观测结果
   - `needs_validation`：有精确的未决事实，**没有严重级别**
   - `rejected`：记录被证伪的候选（不是删掉）

   这比本 skill 的 L0–L3 更严格。L1/L2 仍能进正文，容易让读者把"疑似"读成"确认"。
   改进方向：L2 等价于 `needs_validation`，报告里不写 severity，只写"缺哪个事实"。

2. **对抗性验证**：检查 finding 的 agent 永远不是发现它的 agent。
   本 skill 目前没有这一步，L1→L2 容易自我确认。最低成本实现：升 L2 前强制换一轮对话独立重判，并记录分歧点。

3. **防御纵深缺失不算漏洞**：A 层已挡住攻击时，缺 B 层只是 hardening note。
   直接采用。写进报告的"Rejected with reason"。

4. **覆盖账本**：`coverage-ledger.json` 机械记录"哪个单元查没查"。
   本 skill 现在只有一段文字说明"未扫 vendor/"，无法证明查过了。这是最大的可审计性缺口。

**一条硬数据**：单次运行只能找到约一半漏洞，重复运行累加。所以"跑一遍就下结论"本身是方法错误——报告里必须写明运行次数。

---

## 二、Go 专用工具（先跑它们，再跑本 skill）

| 工具 | 覆盖 | 缺口（本 skill 的补位） |
|------|------|----------------------|
| `securego/gosec` | E1(G204)、E4 部分(G304) | 不懂 FuncMap，不懂"写入即执行"，不判可达性 |
| `golang/vuln` (govulncheck) | 依赖侧 CVE，基于调用图 | 不覆盖自有代码逻辑 |
| `praetorian-inc/gokart` | 污点分析 source→sink | 2024-01 后未更新，规则老化 |
| `golangci-lint` | 聚合 lint | 安全规则非主体 |

**结论**：现成工具在 E1 和依赖侧足够，**E2（模板 FuncMap）、E4（写入即执行）、E5（参数注入）是真空白**。这三相就是自建扫描器的理由。

---

## 三、最接近的竞品：protectai/vulnhuntr（2.8k）

零样本 LLM 漏洞发现，专攻 RCE 类。**思路与本 skill 相反**：

- vulnhuntr：先枚举入口（能接收用户输入的点）→ 从入口往下追到 sink
- 本 skill：先扫 sink → 从 sink 反向追调用者（scanner 输出 caller 线索）

两种顺序各有盲区：正向追会漏"间接入口"（定时任务、webhook、消息队列），反向追会漏"跨文件且调用点不明显"的路径。
**建议**：大项目先用正向（选入口少的模块），小项目用反向（sink 少）。

> 注意：该项目 2025-02 后未更新，规则可能不匹配新框架。

---

## 四、正统路线：GitHubSecurityLab/seclab-taskflow-agent（285）

YAML 驱动的多 Agent 编排 + CodeQL。代表另一种哲学：**让 LLM 指挥成熟分析引擎，而不是让 LLM 肉眼看代码**。

对本 skill 的启示：当 Go 工具链可用时，正确顺序是
`CodeQL/gosec 出结构化事实` → `LLM 做可达性与可利用性判断` → `人工验证`，
而不是让 LLM 直接从源码文本猜。我们现在的正则扫描是"没有 CodeQL 时的降级方案"，应如实这样标注。

---

## 五、训练语料

- `vulhub/vulhub`（21k）：Docker 化历史 CVE 靶场
- `sunblaze-ucb/cybergym`（1k）：真实漏洞分析任务集
- `uiuc-kang-lab/cve-bench`（310）：40 个 critical CVE 的利用评测

用法见 `case-library.md` D1 节（漏洞版本 / 修复版本双跑）。这是唯一能证明规则有效的硬指标。

---

## 六、情报源（引用前必查）

`google/osv.dev`、`github/advisory-database`、`golang/vuln`。

任何 CVE 编号进报告前，必须命中 NVD / cve.org / GHSA / OSV 之一。查不到就当不存在——内容农场在 Go 生态里编造的 CVE 编号（如 CVE-2024-34567）已经污染了大量搜索结果。
