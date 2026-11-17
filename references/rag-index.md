# RAG 索引层：1672 行怎么在 5 秒内缩到 200 行

## 0. 为什么需要这一层

实测（2026-11-13，本机统计，`references/` 共 10 份）：

| 指标 | 数值 |
|---|---|
| 总字符 / 总字节 | 55,934 字符（UTF-8 下 81,645 字节） |
| **全量估算 token** | **约 20,209**（中文按 1.5 字符/token，英文按 4） |
| chunk 数（按 `##`/`###` 切） | **114** |
| 单 chunk 平均 | 约 129 个检索词元 |
| `SKILL.md` 常驻部分 | 229 行 / 约 2.1k token |

把 20k token 全塞进上下文，等于在开始做事之前先烧掉一次完整分析的预算；
全部不读，就等于只剩六相表格，遇到具体项目仍然靠猜。所以中间必须有一层**只加载命中的那 3–5 段**。

这一层不是"向量库"。语料是同一个人写的技术文档，术语高度集中，
**符号索引（指纹 + BM25）比 embedding 更准也更快**，而且零依赖、能离线跑。

---

## 1. 索引单元：一个 chunk 是什么

切分规则：遇到 `##` 或 `###` 标题就开新 chunk，记录 `file` / `heading` / `start` / `end`。

```python
c = dict(file="case-library.md", heading="A7. Gitea 1.4.0 LFS Oid 路径穿越…",
         start=86, end=112, level=3)
```

`start`/`end` 是**行号**，不是字符偏移。这样检索结果能直接喂给 `Read(offset=86, limit=27)`，
不需要再读整份文件。这是本层能省 token 的全部原因。

切的粒度选 `##`/`###` 而不是 `#`：`#` 级是整个文件（case-library 一整份 179 行），
`####` 级会把「判据 1 / 判据 2」切散，检索命中判据 2 却丢了判据 1 的上下文。
`###` 级平均 174 token，正好是一段完整论点。

---

## 2. 两条检索通道，职责不重叠

### 通道 A：指纹唤醒（确定性，不参与打分）

输入项目目录，抽 6 类事实，按固定表映射到必读 chunk。**不是"相关"，是"必读"**。

| 指纹 | 检测方式 | 唤醒 |
|---|---|---|
| `gin` / `echo` / `chi` / `macaron` / `fiber` / `beego` | `go.mod` + 抽样 `.go` 里的框架符号 | route-map §1 框架指纹表 |
| 存在 `router/` `routers/` `routes/` | 目录名 | route-map §1 |
| 裸 `net/http` 且无其他框架 | 有 `http.HandleFunc` 且无框架依赖 | route-map §1 + 「优先看这些端点」 |
| Dockerfile 含 `FROM scratch` / `distroless` | 读 Dockerfile | route-map §5 闸门（无 shell → E1 打折） |
| 扫描器命中 `X-PATHTRAV-*` | `--hits` 参数 | sink-catalog 该规则 **+ case-library A7（先看写侧闸门）** |
| 扫描器命中 `E1–E6` | `--hits` 参数 | 对应相的 six-paths 节 |

**为什么 `X-PATHTRAV-*` 要强制带出 A7**：Gitea 那一轮的教训——路径穿越看着能到 RCE，
但写侧有 `sha256(content)==oid` 互斥闸门。不带 A7 就会把"任意文件读"写成 RCE。
这条映射是**防错**用的，不是防漏。

### 通道 B：BM25 关键词（概率性，补足 A 没想到的）

- 分词：英文按 `[A-Za-z_]\w{1,}`，中文按 **2-gram**（`路径穿越` → `路径`/`径穿`/`穿越`）。
  所以中英混查、同义词歪写都能召回，不需要维护词表。
- heading 命中查询词时得分 ×2：标题是人工写的主题标签，比正文更可信。
- `len < 5` 的纯标题节直接跳过（否则会命中只有两行的分节标题）。

实测三条：

```
$ python3 scripts/kb_retrieve.py --q "路径穿越 任意文件写 谁执行 hook" --top 4
 1. A7. Gitea 1.4.0 LFS Oid 路径穿越                        score=32.08  case-library.md:86-112
 2. E4 写到哪里就会被执行                                    score=26.81  six-paths.md:123-162
 3. E4-HOOK-WRITE (critical)                                score=15.19  sink-catalog.md:111-116
 4. X-PATHTRAV-JOIN (high)                                  score=13.94  sink-catalog.md:176-185

$ python3 scripts/kb_retrieve.py --q "argument injection git upload-pack prefix" --top 3
 1. E5-GIT-OPT (high)                                       score=27.06  sink-catalog.md:39-44
 2. E5 参数注入（Go 里"没有 shell"不等于安全）                 score=8.79   six-paths.md:163-187
 3. E5-ARG-NONCONST (low)                                   score=8.42   sink-catalog.md:45-54
```

中文查询和英文查询各自命中了同一组概念的正确段落，没有互相污染。

---

## 3. 四层加载策略（决定什么时候读什么）

| 层 | 内容 | 体量 | 加载时机 | 失效表现 |
|---|---|---|---|---|
| **L0 常驻** | SKILL.md 的 §0 非协商 + §1 六相表 | ~600 token | 永远在 | 没有它连"该走哪一相"都不知道 |
| **L1 路由** | route-map §1 指纹表 + §2 入口→优先相 | ~800 token | 拿到项目后第一件事 | 跳过它 → 直接扫 586 条，按噪声排序 |
| **L2 详情** | 命中的 3–5 个 chunk（通道 A+B） | ~600–900 token | Phase 1 出候选之后 | 全读 → 18k，只读 L0 → 靠猜 |
| **L3 原文** | 具体案例全文 / 补丁 diff | 按需 | 只有要写 `L3-repro` 或对照 CVE 时 | 提前读 → 浪费；不读 → 编造机制 |

一次完整的 35 分钟流程，实际加载量约 **2.5k–3.5k token**，占全量的 15%–20%。

---

## 4. 唤醒预算（防"检索成瘾"）

检索本身也花时间。硬性上限：

| 项目规模 | 允许的检索次数 | 说明 |
|---|---|---|
| < 50 个 .go 文件 | 1 次 | 直接 L0+L1，不需要 L2 |
| 50–500 | 2–3 次 | 通道 A 一次 + 通道 B 两次 |
| > 500 | 3–4 次 | **不允许超过 4 次**；超了说明在拿检索代替思考 |

判断"检索够了"的标准：**当前候选能不能写出一行 `F1 | 相 | path:line | 入口 | 参数`**。
写得出就停，写不出说明缺的是通链工作，不是文档。

---

## 5. 怎么用（三条命令）

```bash
# 1. 拿到项目，先唤醒必读段
python3 scripts/kb_retrieve.py --project /opt/corpus2/gogs_gogs

# 2. 扫完出了候选，按命中规则补详情
python3 scripts/kb_retrieve.py --hits X-PATHTRAV-JOIN,E4-HOOK-WRITE

# 3. 卡住了，用自然语言问
python3 scripts/kb_retrieve.py --q "写进去但是被删了 没落盘"
```

输出里每行都带 `Read(file_path=..., offset=..., limit=...)`，可直接执行。
脚本零依赖（stdlib），主机 Python 3.6.8 实测可跑。

---

## 6. 这一层不解决的问题

- **不判断真伪**。检索只决定"读哪段"，读完之后是否命中闸门仍然要靠通链。
- **不做跨文档推理**。`A7 写侧被 sha256 挡住` + `E4 需要下游执行者` 这两条要人合并成
  "这条链打不通"，检索不会替你推。
- **不索引源码**。项目源码靠 scanner + `rg`，本层只管 references。

下一层可做但本次未做：把 `case-library` 的每条案例拆成
`形态 / 入口条件 / 闸门 / 修复提交` 四个结构化字段，用字段匹配替代全文 BM25。
那样同构迁移的召回会更准，代价是要人工标注 7 条案例。
