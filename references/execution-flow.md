# 执行流程全图（mermaid）

本文件是 `SKILL.md` 的**图形化等价物**。文字版看 SKILL.md，想知道"这一步为什么排在这里"看本文件。

所有图都是 mermaid 源码，可直接粘进支持 mermaid 的渲染器（GitHub、Typora、VS Code、飞书文档）。
图中的 `E1–E6 / X` 是六相编号，`L0–L3` 是证据等级，`T0–T4` 是时间预算槽位，与
[route map](route-map.md) §4 的三十五分钟路线一一对应。

---

## 图 1 — 总流程：OODA 六步 × Phase 0–5

一条任务从进来到出报告只走一遍这条线。**虚线是回写**，它让下一次跑得更快。

```mermaid
flowchart TB
    IN(["接到任务：一个 Go 项目路径"]) --> G0{"有可读源码？"}
    G0 -- 无 --> BLOCK["BLOCKED: no source<br/>只交付方法与清单，不交付结论"]
    G0 -- 有 --> O1

    subgraph OODA["OODA 环（跑一轮约 35 分钟）"]
        direction TB

        O1["Orient 定向 · Phase 0 · T0 5min<br/>go.mod 版本 / 框架 / 部署形态 / 入口面 / 部署用户"]
        O2["Observe 观察 · Phase 1 · T2 10min<br/>scan_sinks.py 出候选 + 3 条 rg 兜底"]
        O3["Decide 决策 · route-map §2–§3<br/>入口形态 → 优先相 → Tier S/A/B/C/X"]
        O4["Act 行动 · Phase 2–3 · T3+T4 10min<br/>通链：入口→参数→变换链→闸门→payload"]
        O5{"Verify 验证 · Phase 4<br/>负对照是否成立？"}
        O6["Close 收尾 · Phase 5<br/>报告：Finding + Rejected + Coverage gaps"]

        O1 --> O2 --> O3 --> O4 --> O5
        O5 -- "负对照也触发 → 假复现" --> REJ["降级并记入 Rejected<br/>FALSE-POSITIVE: 原因 @ path:line"]
        O5 -- "payload 通 / 负对照不通" --> O6
        REJ --> O6
    end

    O6 --> FB["回写资产<br/>新规则 / 新闸门 / 新案例"]
    FB -.->|下次同一形态 5 秒命中| O2

    classDef stop fill:#FCEBEB,stroke:#A32D2D,color:#501313
    classDef act fill:#E6F1FB,stroke:#185FA5,color:#042C53
    classDef gate fill:#FAEEDA,stroke:#854F0B,color:#412402
    class BLOCK stop
    class O4,O6 act
    class O5 gate
```

**读法**：`Orient → Observe` 之间**不能跳**。跳过定向直接扫，会出现 route-map §0 记录的那种事故——
Gitea 1.4.0 的 LFS handler 全都写在源码里，但 git 版本不够时 `LFS.StartServer = false`，
路由根本不注册，所有请求 404，白猜三轮。

---

## 图 2 — 判定树：一条 sink 命中该走哪一相

扫描器只回答"这里有危险调用"，本图回答"它属于哪一相、用什么标准证伪"。
**这是全流程里唯一必须逐条走完的分支**，走错相会导致整条链的分析结论作废。

```mermaid
flowchart TD
    HIT(["sink 命中 L0"]) --> Q1{"命令行里有 shell 解释器？<br/>sh / bash / cmd / powershell"}
    Q1 -- 是 --> E1["E1 shell 拼接<br/>证伪：argv 分开传 + 目标平台无 shell"]
    Q1 -- 否 --> Q2{"模板文本来自用户？<br/>或 FuncMap 含 exec/readfile"}
    Q2 -- 是 --> E2["E2 模板求值<br/>证伪：字面量模板 + 纯函数 FuncMap"]
    Q2 -- 否 --> Q3{"写操作落在执行位置？<br/>hooks / cron.d / authorized_keys / webroot / plugin"}
    Q3 -- 是 --> E4{"写完之后谁执行它？"}
    Q4 -- 写不出来 --> E4D["降到 L0<br/>没有下游执行者 ≠ RCE"]
    Q3 -- 否 --> Q3b{"解压 / 还原 / checkout<br/>能控制落点？"}
    Q3b -- 是 --> E4
    Q3b -- 否 --> Q5{"外部命令的 argv 里有用户可控项？"}
    Q5 -- 是 --> E5["E5 参数注入<br/>证伪：前缀字面量 / 有 -- 分隔"]
    Q5 -- 否 --> Q6{"plugin.Open / syscall.Exec / 解释器 LOAD<br/>目标可控？"}
    Q6 -- 是 --> E6["E6 动态加载<br/>证伪：路径字面量或有签名校验"]
    Q6 -- 否 --> Q7{"只是解码？gob / yaml / cookie"}
    Q7 -- 是 --> E3["E3 反序列化<br/>解码本身永不执行 → 必须看解码结果的用途"]
    Q7 -- 否 --> X["X 前置原语<br/>路径穿越 / pprof / netrpc<br/>本身不是 RCE，是链的前半段"]

    E4 --> EXEC["写出执行者 + 触发时机<br/>否则不算"]
    E1 --> P3["进 Phase 3 注入性证明"]
    E2 --> P3
    E5 --> P3
    E6 --> P3
    E3 --> P3
    X --> CHAIN{"能接到 E4 或 E2 吗？"}
    CHAIN -- 不能 --> L0["停在 L0 / L1<br/>写明缺少的后半段"]
    CHAIN -- 能 --> P3

    classDef path fill:#EEEDFE,stroke:#534AB7,color:#26215C
    classDef bad fill:#FCEBEB,stroke:#A32D2D,color:#501313
    classDef ok fill:#E1F5EE,stroke:#0F6E56,color:#04342C
    class E1,E2,E3,E4,E5,E6,X path
    class E4D,L0 bad
    class EXEC,P3 ok
```

**这张图存在的理由**：`exec.Command` 不等于 E1。只有 shell 会把字符串变成命令行。
`exec.Command("cat", "/reports/"+name)` 是路径穿越，报成 RCE 会被一分钟打回。
反过来，Gitea 2026 的在野 RCE 全程没有 `exec`——它写了个 git hook，是纯 E4。

---

## 图 3 — 证据门：L0→L3 与九道闸门的拦截位置

九个闸门不是"注意事项"，它们是**流水线上的拦截器**。每撞上一个，等级必须当场降下来。

```mermaid
flowchart LR
    L0["L0 候选<br/>sink 命中"] --> GA{"闸门 1<br/>路由注册了吗？"}
    GA -- 未注册 --> STOP1["降 L0<br/>死代码"]
    GA -- 已注册 --> GB{"闸门 2<br/>认证态？"}
    GB -- "需登录 / 管理员" --> NOTE1["降级并标注<br/>认证后利用 ≠ 未授权"]
    GB -- "未授权" --> L1["L1 疑似可达<br/>入口 + 参数 + 变换链写清"]
    NOTE1 --> L1
    L1 --> GC{"闸门 3–6<br/>内容 / 长度 / 版本 / 权限"}
    GC -- 拦 --> STOP2["降 L1<br/>写明闸门位置 path:line"]
    GC -- 过 --> L2["L2 疑似漏洞<br/>payload 字符逐跳存活"]
    L2 --> GD{"闸门 7–9<br/>白名单 / 沙箱 / shell 缺失"}
    GD -- 拦 --> STOP2
    GD -- 过 --> REP["Phase 4 隔离环境复现"]
    REP --> GN{"负对照：<br/>payload 中和后是否不触发？"}
    GN -- 仍触发 --> FAKE["假复现<br/>不是漏洞，是环境噪声"]
    GN -- 不触发 --> L3["L3 已复现<br/>可写 confirmed"]

    classDef lv fill:#E6F1FB,stroke:#185FA5,color:#042C53
    classDef stop fill:#FCEBEB,stroke:#A32D2D,color:#501313
    classDef ok fill:#E1F5EE,stroke:#0F6E56,color:#04342C
    class L0,L1,L2 lv
    class STOP1,STOP2,FAKE stop
    class L3,REP ok
```

九道闸门的识别方法见 [route map](route-map.md) §5。实测数据说明为什么这张图重要：
Gitea 1.4.0 那条链在 L2 看着完美——未授权、参数直达、payload 字符全存活，
但闸门 3（内容闸门：先写 `.tmp` 再比对 `sha256(content)==oid`）把写侧彻底封死，
最终只能定级为"未授权任意文件读 + 任意目录创建"，**不是 RCE**。

---

## 图 4 — 候选排序：25 条规则 → 5 个 Tier → 通链预算

扫描器输出 25 条规则的命中，本图是**把它们压成可执行的预算分配**。
校准数据来自 10 个真实项目 / 760k LOC / 586 命中。

```mermaid
flowchart TB
    RAW["586 条原始命中<br/>密度 0.77 / 千行"] --> T{"按可执行性半径分 Tier"}

    T --> S["Tier S · 14 条 · 2.4%<br/>exec sh -c / syscall.Exec / plugin.Open 变量<br/>→ 全查，每条 5 分钟"]
    T --> A["Tier A · 16 条 · 2.7%<br/>exec git / tar + 用户参数 · template.Parse 用户串<br/>→ 全查，每条 5 分钟"]
    T --> B["Tier B · 87 条 · 14.8%<br/>WriteFile 落在执行目录<br/>→ 先问谁执行，答不出直接丢"]
    T --> C["Tier C · 327 条 · 55.8%<br/>gob / yaml / 反射<br/>→ 只在有明确下游时才看"]
    T --> X["Tier X · 141 条 · 24.1%<br/>filepath.Join 无前缀校验<br/>→ 只在要接 E4/E2 时才看"]

    S --> BUD["预算：S+A 全查 = 150 分钟上限<br/>B/C/X 只做定向抽查"]
    A --> BUD
    B --> DROP{"能写出执行者？"}
    C --> DROP
    X --> DROP
    DROP -- 不能 --> D2["L0 附录，不进正文"]
    DROP -- 能 --> BUD

    BUD --> REAL["实测：14 条 Tier S 里<br/>1 条值得通链 / 6 条待定 / 7 条当场否决<br/>→ 全量精确率约 0.5%–1%"]

    classDef s fill:#FCEBEB,stroke:#A32D2D,color:#501313
    classDef a fill:#FAEEDA,stroke:#854F0B,color:#412402
    classDef n fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A
    class S s
    class A a
    class B,C,X n
```

**核心结论**：命中数最多的三条规则（E2 模板 40%、X 路径 22%、E6 反射 15%）
恰好是最不可直接利用的三条。**按命中数排序 = 按噪声排序**，
所以 route-map 用 Tier 而不用 `--min-severity` 排序。

> **数值时效**：图中的 586 / 14 / 16 是 2026-11-13 首轮规则下的标定。同日盲测发现
> `E2-TPL-USERSTRING` 与 `E4-ZIPSLIP` 各有一处形变错误（详见
> `sink-catalog.md` 对应条的"历史噪声"），修正后四个项目合计减少 81 条 Tier A/B 命中。
> 结论方向不变（S+A 占比极低），但**具体条数需按当轮规则重跑，不要引用旧数**。

---

## 图 5 — RAG 唤醒：从任务输入到加载哪份 reference

skill 有 8 份 reference、2087 行，全读会挤爆上下文。本图是**检索层**：
每份 reference 由一个"指纹"唤醒，指纹来自项目本身，不来自用户的话。

```mermaid
flowchart LR
    TASK(["任务输入"]) --> FP["抽指纹（不读全文）<br/>go.mod · 目录名 · 依赖 · 框架符号"]

    FP --> F1{"框架？"}
    F1 --> R1["route-map §1<br/>框架→路由文件→认证中间件"]

    FP --> F2{"sink 属于哪一相？"}
    F2 --> R2["six-paths<br/>该相的机制 + payload 形态 + 证伪条件"]

    F2 --> R3["sink-catalog<br/>这条规则什么时候是真、什么时候是噪声"]

    FP --> F3{"形态像已知案例？"}
    F3 --> R4["case-library<br/>A1–A7 的证据等级 + 重复出现的闸门"]

    FP --> F4{"链要跳几个函数？"}
    F4 --> R5["reachability<br/>接口分发 / 中间件顺序 / goroutine 捕获"]

    FP --> F5{"要不要下结论？"}
    F5 --> R6["verification-gates<br/>L0→L3 清单 + 负对照"]

    FP --> F6{"项目超过 30 个候选？"}
    F6 --> R7["route-map §2–§4<br/>优先相 → Tier → 35 分钟路线"]

    classDef in fill:#E1F5EE,stroke:#0F6E56,color:#04342C
    classDef ref fill:#EEEDFE,stroke:#534AB7,color:#26215C
    class TASK,FP in
    class R1,R2,R3,R4,R5,R6,R7 ref
```

唤醒规则（可机械执行，见 [rag-index](rag-index.md)）：
- 出现 `gin` / `echo` / `chi` / `macaron` 任一 → 必读 route-map §1
- 候选数 > 30 → 必读 route-map §2–§4，否则 Phase 2 会变成两天
- 准备写 "confirmed" → 必须过 verification-gates 的负对照
- 命中 `X-PATHTRAV-*` → 必读 case-library A7 的写侧闸门，否则会把"任意文件读"写成 RCE

---

## 图 6 — 回写闭环：一次复现如何变成下次的秒级命中

```mermaid
flowchart LR
    P["复现结论"] --> Q{"这次新学到的是什么？"}
    Q -- "新形态" --> R1["加规则<br/>scan_sinks.py RULES"]
    Q -- "新拦截" --> R2["加闸门<br/>route-map §5"]
    Q -- "新机制" --> R3["加案例<br/>case-library，标注 L3-repro"]
    Q -- "旧结论被推翻" --> R4["改写旧条目<br/>并写明与 writeup 的偏差"]

    R1 --> T["三件套断言<br/>正样本命中 + 负样本不命中 + 变异测试"]
    R2 --> T
    R3 --> T
    R4 --> T
    T --> RUN["test_scan_sinks.py 全过"]
    RUN --> NEXT["下次同形态<br/>规则先命中 → 人工只做通链"]

    classDef w fill:#EAF3DE,stroke:#3B6D11,color:#173404
    classDef ok fill:#E1F5EE,stroke:#0F6E56,color:#04342C
    class P,R1,R2,R3,R4 w
    class NEXT ok
```

判断"值不值得回写"的标准只有一条：**它能不能让下一次少走一步**。
Gitea 那一轮交了三条（git 版本闸门、sha256 写闸门、Content-Length 截断），这三条都能复用。
"我记住了一个 CVE 编号"不能复用，所以不进 case library。

---

## 与 SKILL.md 的对应关系

| 本文件 | SKILL.md | route-map |
|---|---|---|
| 图 1 | §3 Phase 0–5 | §4 T0–T4 |
| 图 2 | §1 六相表 + §2 五条反直觉 | §2 入口形态→优先相 |
| 图 3 | §4 证据等级 | §5 九道闸门 |
| 图 4 | §3 Phase 1 的排序提示 | §3 Tier 表 |
| 图 5 | §6 References | 全文 |
| 图 6 | §7 维护 | — |
