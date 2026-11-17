# OODA 执行环：把方法论变成可派工的步骤

本文件与 [route map](route-map.md) 的分工：
route-map 回答**先看哪里**（顺序问题），本文件回答**每一步做到什么程度算完**（完成判据问题）。
两者一起用；只读其中一个会退化成"照着清单划勾"。

环的六格与 SKILL.md 的 Phase 对应关系固定，不随项目变化：

| OODA 格 | 对应 Phase | 时间槽 | 唯一产物 |
|---|---|---|---|
| Purpose / Success | — | 2 min | 一句话：这次要证明什么、什么算证完 |
| Align / Ask | — | 1 min | 授权边界 + 可动范围（能不能起服务、能不能发包） |
| Observe / Diagnose | P0 + P1 | 15 min | 事实卡 + 候选清单（带 path:line） |
| Decide / Execute | P2 + P3 | 12 min | 排序后的路线 + 每条的降级理由 |
| Verify / Close | P4 + P5 | 5 min | 证据等级 + 负对照 + 报告 |
| 回写 | — | 2 min | 新规则 / 新闸门 / 新案例 |

---

## 1. Purpose / Success —— 先写"什么算证完"

不写这一格，后面必然滑向"读了很多代码，说不出结论"。

三种任务类型，成功标准完全不同，**先选类型再动手**：

| 类型 | 触发说法 | 成功标准 | 允许的终态 |
|---|---|---|---|
| **定向挖洞** | "这个版本有没有 RCE" | 给出 ≥1 条 L2，或给出"全部候选被闸门 X 拦住" | L3 / L2 / 有理由的 negative |
| **能力标定** | "测一下这套方法准不准" | 候选排序中真漏洞落在 Top-N | 命中率数字，不需要 L3 |
| **复现校验** | "CVE-xxxx 真的能打吗" | 读出补丁 diff，机制对得上，本机能打通 | L3 或 BLOCKED |

写下来的形式（一句话，必须可证伪）：

> 本次目标：在 `<commit>` 的 1Panel 里证明 `POST /settings/backup/backup` 的 `DetailName`
> 能否注入到 `remote.go:137` 的 `bash -c` 命令行。证完的标准是 L2（payload 字符逐跳存活）
> 或明确的闸门位置。**不追求 L3**，因为没有该版本的隔离构建环境。

## 2. Align / Ask —— 三个必问，问完才动

1. **授权**：目标是你自己的还是第三方的。第三方实例只做静态审计，不发包。
2. **可动范围**：能不能编译？能不能起服务？能不能装依赖？——这决定天花板是 L3 还是 L2。
3. **部署形态**：Dockerfile 是 `FROM scratch` 吗？服务账号是谁？——决定 E1 类结论是否成立。

第 3 条最容易被跳过，也最容易让结论作废：distroless 镜像里没有 `sh`，
源码层 100% 成立的 E1，运行时直接失败。

## 3. Observe / Diagnose —— 两个产物，缺一不可

**产物 A：事实卡**（5 分钟，全部可核实）

```
path:        /opt/corpus2/1Panel-dev_1Panel
commit:      <git rev-parse HEAD 或解包时间>
go.mod:      go 1.xx（决定 os.Root 等新 API 是否可用）
framework:   gin（router/*.go 集中注册）
deploy:      Dockerfile → 基础镜像 / 运行用户
entry:       HTTP 127.0.0.1:xxxxx/gin 路由数 N
skip:        vendor/ 无、*_test.go 已跳过
```

**产物 B：候选清单**（10 分钟）

```bash
python3 scripts/scan_sinks.py <path> --out scans/x.md --json scans/x.json
rg -n 'sh", "-c|/c", |CommandContext' --type go          # 兜底 1
rg -n 'hooks/|authorized_keys|crontab|plugin\.Open' --type go  # 兜底 2
```

**诊断判据**：候选数决定下一步走哪条分支——
- `> 30` → 必须先做 route-map §2–§3 的排序，直接通链会超时
- `5–30` → 按 Tier 排序后全查
- `< 5` → 全查，且要怀疑扫描器漏了（换 `--include-vendor`，或项目用了非标准框架）

## 4. Decide / Execute —— 每条候选只允许三种裁决

不要有第四种状态。"再看看"是拖延，不是裁决。

| 裁决 | 条件 | 记录形式 |
|---|---|---|
| **GO** | Tier S/A，入口未授权，闸门清单未命中 | `F1 | E1 | path:line | 路由 | 参数名` |
| **HOLD** | 有入口但闸门未查完 | `F2 | HOLD | 缺：DetailName 白名单未查` |
| **KILL** | 命中任一闸门且无法绕过 | `F0 | KILL | argv 分开传 @ hook.go:12` |

预算纪律：**S+A 全查上限 150 分钟**。超了就停，把剩余候选写进 Coverage gaps，
不要用"还剩两条就查完了"骗自己——那条通常是 Tier C。

## 5. Verify / Close —— 负对照是唯一硬门

| 检查 | 不通过的表现 |
|---|---|
| 负对照：payload 中和后不触发 | 中和了还触发 → 环境噪声，不是漏洞 |
| 字符存活逐跳记录 | 只说"应该能过" → 退回 L1 |
| 每个断言有 `path:line` | 引用了但没 `sed` 回读 → 可能记错文件 |
| Rejected 一节非空 | 空 → 说明没查过闸门，只查了 sink |

## 6. 回写 —— 只有一条判据

**这次学到的东西，能不能让下一次少走一步。**

- 能 → 加规则 / 加闸门 / 加案例，并配一条会失败的自测断言
- 不能（比如"我记住了一个 CVE 编号"）→ 不写

## 7. 环的失败模式（按出现频率排序）

| 失败 | 表象 | 根因 | 修法 |
|---|---|---|---|
| 跳过 Orient | 扫完 586 条，一条都没通链 | 没定成功标准，不知道停在哪 | 先写 Purpose |
| 按命中数排序 | 一直在读模板字符串 | 命中数 ≠ 可利用性（实测 S+A 仅 5.1%） | 用 Tier |
| 把 L1 当结论 | 报告写"存在命令注入" | 没过闸门 | 记住 L1 只能写"疑似可达" |
| 局部搜索下否定结论 | 说"没有 token 预算闸门" | 只在子目录搜过 | 在全包根搜，并记下命令 |
| 不写 Rejected | 报告只有确认项 | 读者无法区分"查过"和"没想到" | 强制一节 |
