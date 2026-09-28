# lcsc-bom-sourcing

> 把 BOM 变成可直接下单的采购核对表：国内现货、人民币阶梯价、参数比对结论、风险评估。

一个 Agent Skill。给它一份 BOM（或几行硬件参数），它逐个料号查立创商城，筛出有货的候选，给出**符合 / 风险 / 信息不足**的结论，输出一份可直接交给采购的 Excel。

> 关键词：立创商城 · LCSC · szlcsc · 电子物料 · 元器件选型 · BOM 核对 · 替代料 · 现货库存 · 阶梯价 · 采购清单

## 它解决什么问题

手工核 BOM 的痛苦在于三件事挤在一起：

1. **查了才知道没货**——十几个型号逐个点进详情页，一半没现货，白查。
2. **价格口径不确定**——页面给的是阶梯价，100 片和 5000 片单价差 30% 以上，报给采购得先算。
3. **参数对不对说不清**——型号看着一样，额定电流、温度范围、容差可能不满足，靠肉眼容易漏。

这个技能把三件事一次做完：**搜索定位 → 详情页实抓 → 过滤缺货 → 按规格判定 → 排序输出**。

## 输入输出

**输入**（每行至少一组硬件参数，列顺序随意，也接受 Excel 或截图）：

| 字段 | 必要性 | 作用 |
|---|---|---|
| **硬件参数** | **必选** | 需求本体，缺了会先问你 |
| 物料编号 | 可选 | 给了就用作 sheet 名；没给会自动命名并标注 |
| 品牌 | 可选 | 选型偏好，给了就优先在该品牌内挑候选 |

```
物料编号（可省）    硬件参数                          品牌（可省）
DEMO-R-0603-10K    0603 10kΩ ±1% 100mW             不限
（不填也行）        0603 100nF X7R 50V               风华
DEMO-L-3012-10U    10µH ±20% ≥920mA SMD 3×3        （不填也行）
```

**输出**：一份工作簿，第 1 张汇总页给结论，之后每个物料一张 sheet、5 个候选横向铺开：

| 属性 | ★ 候选 1 | 候选 2 | 候选 3 |
|---|---|---|---|
| 型号 (MPN) | … | … | … |
| 现货库存 | … | … | … |
| 整盘档单价 | … | … | … |
| 匹配结论 | 符合 | 风险 | 信息不足 |

完整版式见 [`references/output-layout.md`](references/output-layout.md)，示例见 [`examples/`](examples/)。

## 安装

**环境要求**：承载它的 Agent 需具备**网页抓取能力**（能访问 `szlcsc.com`），并有一个可运行 Python 的执行环境。生成 Excel 需要 `openpyxl`（第三方库）；没有时输出降级为 Markdown 表格。无 API key 要求。

### 方式一：作为 Agent Skill（推荐）

把仓库克隆/复制到任一支持 SKILL.md 标准的工具的技能目录下即可，无需改代码：

```bash
git clone <repo-url>
```

| 工具 | 用户级目录 | 项目级目录 |
|---|---|---|
| WorkBuddy | `~/.workbuddy/skills/` | `<repo>/.workbuddy/skills/` |
| Claude Code | `~/.claude/skills/` | `<repo>/.claude/skills/` |
| Codex | `~/.agents/skills/`（官方；部分版本亦识别 `~/.codex/skills/`） | `<repo>/.agents/skills/` |
| Cursor | `~/.cursor/skills/` | `<repo>/.cursor/skills/` |
| CodeBuddy | `~/.codebuddy/skills/` | `<repo>/.codebuddy/skills/` |

> 想在多个工具间共用同一份文件，用**目录联接**指向同一个源目录，而不是复制多份：
>
> ```powershell
> # Windows（无需管理员权限）
> New-Item -ItemType Junction -Path "$HOME\.claude\skills\lcsc-bom-sourcing" `
>                            -Target "$HOME\.workbuddy\skills\lcsc-bom-sourcing"
> ```
>
> ```bash
> # macOS / Linux
> ln -s ~/.workbuddy/skills/lcsc-bom-sourcing ~/.claude/skills/lcsc-bom-sourcing
> ```

装好后，显式调用或直接用自然语言描述任务（触发词见下节）。

> **注意**：这是一个**流程型技能**——没有可独立运行的脚本，所有抓取、比对、出表动作由 Agent 按 `SKILL.md` 里的流程执行。脱离 Agent（比如只想在命令行跑一下）是跑不起来的。另需宿主具备**网络访问能力**；部分工具（如 Codex）默认沙箱禁用网络，需先开启才能抓取站点。

### 跨工具兼容性

本技能依据开放 Agent Skill 标准编写，`SKILL.md` 的 frontmatter **只使用公共字段**（`name` + `description`），未使用任何宿主私有字段，因此在下列工具中均可直接加载：

| 工具 | 兼容性 | 说明 |
|---|---|---|
| Claude Code | ✅ | `name` 与目录名一致、`description` 远低于 1024 字符上限；如需减少授权弹窗，可自行在 frontmatter 追加 `allowed-tools` |
| Codex | ✅ | 同一套标准。Codex 另支持可选的 `agents/openai.yaml`（显示名、图标、MCP 依赖），仅 Codex 读取，其他工具忽略；不加不影响使用 |
| Cursor / CodeBuddy / WorkBuddy | ✅ | 按上表目录放置即可 |

差异只在**可选元数据字段**（各宿主自有），而不在正文结构——正文是给 Agent 读的自然语言流程，与宿主无关。

## 使用

**① BOM 表（最常用）**

```
按这个 BOM 查立创库存和价格：

DEMO-R-0603-10K   0603 10kΩ ±1% 100mW
DEMO-C-0603-100N  0603 100nF X7R 50V

采购量按最小包装整盘算，现货优先、单价最优。
```

（物料编号和品牌都可以省——只给参数也能跑，只是 sheet 名会用自动生成的名称。）

**② 单个料号**

```
查一下 C25804 的立创现货和阶梯价，再给 4 个同规格替代
```

**③ 原理图截图**

```
这是电源部分的原理图，帮我核对这几个料号选得对不对
```

截图只能读符号与标注值；图上没有的参数会标成「信息不足」，不会猜。

触发词：`查立创库存`、`立创商城选型`、`帮我核一下 BOM`，或直接贴料号 / 截图。

## 目录结构

```
lcsc-bom-sourcing/
├── SKILL.md                    # Agent 技能入口：输入规则、六步流程、排序口径、输出格式
├── README.md / README.en.md    # 中文 / 英文说明
├── CHANGELOG.md                # 版本变更记录
├── CONTRIBUTING.md             # 贡献指南（禁止提交真实内部数据）
├── NOTICE                      # 商标与非官方声明（未附开源许可证）
├── references/
│   ├── site-notes.md           # 站点结构、URL 规则、反爬兜底、库存字段口径
│   ├── pitfalls.md             # 已踩过的坑（含实测案例）
│   └── output-layout.md        # 工作簿版式规范（汇总页 + 每物料一 sheet）
└── examples/
    ├── input-bom.md            # 三种输入格式示例
    ├── output-sample.md        # 输出示例（Markdown 形态）
    └── output-sample.xlsx      # 输出示例（Excel 形态，虚构数据）
```

## 它会先问你什么

只在**口径没定**时问，不会每次都问：

| 什么时候 | 问什么 | 为什么必须问 |
|---|---|---|
| 没给硬件参数 | 阻值/容值/感值、封装、精度 | 参数是需求本体，没有它无法开工 |
| 没说采购量 | 单价按哪个数量档算 | 同颗电阻 100 片与 5000 片单价差可达 36% |
| 关键参数缺失 | 精度、温度范围、额定值 | 决定结论是「符合」还是「信息不足」 |
| 首次跑某类器件 | 是否要替代料、严格度 | 「严格 P2P」与「允许参数更优」候选完全不同 |
| 只给了型号或 C 编码 | 按该型号采购，还是按其规格另找 | 型号不是需求描述，猜错方向整表作废 |
| 物料编号重名但参数不同 | 合并还是编号写错 | 猜错比停下来问更糟 |

## 设计取舍

- **为什么不先查国际站**：立创国际站（lcsc.com）参数更全、替代料表更完整，但**不支持中国大陆收货地址**，其库存与价格不构成采购依据。所以它只用来补参数与发现候选，国内站是唯一的筛选与报价入口。
- **为什么用四级词典序而不是加权打分**：打分要先定权重，权重怎么定都主观，换个器件类型就不适用。词典序每一级都是客观事实，可复现、可复核。
- **为什么物料编号可选、参数必选**：编号只是内部体系的一个标签，与选型结果无关——你给了就照用，没给就自动命名并标注，不阻碍出表；但**不替你猜编号**，猜错比留空更糟。参数不同，它是需求本体，缺了整件事无从做起，所以缺失时一定先问。
- **品牌为什么是可选偏好而非硬条件**：指定品牌只改变筛选优先级（先在该品牌内挑），不改变参数门槛与缺货纪律——不因指定品牌就放宽容差或接受零库存。该品牌无货时补其他品牌，但会标注出来让你知情。
- **为什么缺货料号直接剔除**：避免出现「看起来很美的替代料最后买不到」。
- **为什么候选不足不凑数**：宁可如实列 1 款并标注单一供应商风险，也不放宽容差凑够 5 个。

## 可配置

| 项 | 默认 | 其他选项 |
|---|---|---|
| 采购数量口径 | 按最小包装整盘 | 逐行需求量 / 需求量下一档 |
| 每物料候选数 | 5 | 关键器件可提到 8–10 |
| 选型偏好 | 现货优先 + 单价最优 | 可叠加国产品牌优先、工业级/车规优先 |
| 品牌偏好 | 不限 | 指定品牌（优先该品牌，跨品牌会标注） |
| 替代料严格度 | 仅同封装同参数 | 严格 P2P / 允许参数更优 |
| 汇总页列 | 九列模板 | 可自定义 |

## 已知限制

- **价格与库存是抓取瞬间的快照**，每次输出都带时间戳，隔几天建议重抓。
- **建议每轮 8–10 个物料**；物料编号超过 15 个建议分批出工作簿。
- 目标站点有反爬限频。技能内置了源码级兜底抓取路径（见 [`references/site-notes.md`](references/site-notes.md)），但高频大量抓取仍可能被限。
- 冷门规格可能全市场只有 1 款有货——会如实列出并标注风险，不凑数。
- 结论基于公开商品页参数，**不替代工程判断**；关键设计请以原厂 datasheet 为准。

## 贡献

欢迎补充新的站点变化与踩坑记录——见 [CONTRIBUTING.md](CONTRIBUTING.md)。

## 免责声明

- 本项目为**非官方**工具，与立创商城、LCSC、嘉立创及文中出现的任何元器件厂商**无关联、无背书**。相关名称与商标均归其各自权利人所有。
- 本技能只读取公开商品页。使用者需自行遵守目标站点的服务条款与 `robots.txt`，并控制请求频率。
- 输出的价格与库存为抓取快照，**不构成报价**，请以下单时页面为准。

## 许可

本仓库**未附开源许可证**，保留所有权利：内容仅供查看，未经作者许可不得复制、修改或再分发；如需使用请开 issue 联系。

商标与非官方声明见 [NOTICE](NOTICE)。

---

## English

**lcsc-bom-sourcing** is an agent skill that turns a hardware BOM into a procurement-ready spreadsheet. For each line item it searches LCSC's China storefront (szlcsc.com), keeps only in-stock candidates, captures CNY ladder pricing and key parameters, judges whether each candidate **meets / risks / lacks information** against the requirement, and ranks candidates by a deterministic four-level dictionary order.

Design notes: the international site (lcsc.com) is used **only** to fill in missing parameters and to discover alternatives — it cannot ship to Mainland China, so it is never a sourcing basis. Out-of-stock parts are dropped rather than listed. See [SKILL.md](SKILL.md) and [references/](references/).

**Keywords:** LCSC · szlcsc · BOM · electronic components · component sourcing · hardware sourcing · procurement · stock availability · ladder pricing · alternative parts · agent skill
