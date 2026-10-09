---
name: personal-financial-analysis-skill
agent_created: true
description: "个人资产负债表财务体检与投资诊断。当用户提供个人资产/负债/持仓表格（xlsx/csv/截图/文字描述）并希望系统分析、诊断财务健康度、给出理财投资建议时触发，产出深色高端商务风的单文件 HTML 诊断报告。触发词：资产负债表、财务体检、财务诊断、理财建议、投资建议、资产配置分析、我的资产、净资产分析、持仓分析、负债分析、家庭财务分析、personal finance diagnosis、net worth analysis、portfolio checkup、asset allocation review"
---

# 💰 Personal Financial Analysis Skill — 个人财务体检与投资诊断

把一份个人「资产 & 负债」记账表，变成一份能真正改变行动的 **单文件 HTML 诊断报告**。

## Purpose

用户的记账表通常只有「存量」视角：有什么、值多少。本技能补上缺失的三层：
1. **结构诊断** —— 配置是否合理、集中度是否危险、有没有同质化重叠
2. **效率诊断** —— 资产增长到底来自储蓄还是投资、被动收入进度如何
3. **数据质量诊断** —— 表里的公式错误、口径变更、记账断层（这一层最容易被忽略，却最常导致用户对自己的财富产生错误认知）

最后输出**按优先级排序、可直接执行**的行动方案。

## When to Use

- 用户提供资产负债 xlsx / csv / 截图 / 文字清单，要求「分析」「诊断」「体检」「给理财建议」
- 用户问「我的资产配置合理吗」「我该怎么调整」「我的钱放得对不对」
- 用户要求生成个人财务 HTML 报告

**不适用**：单只股票/基金分析（用 `stock-analysis`）、财经概念科普（用 `xiaobai_finance_skill`）、上市公司财报分析。

## Skill Base Directory

```
~/.workbuddy/skills/personal-financial-analysis-skill/
├── SKILL.md
├── references/
│   ├── data-extraction.md        # 第 1 步：读表与字段识别（含 edsdk 调用范式）
│   ├── diagnostic-framework.md   # 第 2 步：指标体系 + 六维评分 + 问题库（24 条）
│   └── advice-playbook.md        # 第 3 步：建议生成规则 + 目标配置模型
├── assets/
│   └── report-skeleton.html      # 第 4 步：HTML 骨架（深色商务风 + Chart.js）
└── scripts/
    └── dump_sheets.py            # xlsx 全表转 CSV（openpyxl 降级通道，缓存可能陈旧）
```

---

## 执行流程（4 步，严格按序）

### Step 1 · 读数据（必须读真实数据，禁止编造）

**首选通道**：本地 Office MCP editor_sdk（**唯一可靠**——它会实时重算公式，而磁盘上的缓存常是陈旧的）

```bash
SK="/Applications/WorkBuddy.app/Contents/Resources/app.asar.unpacked/resources/plugins/workbuddy-builtin/skills/tencent-local-office-edit"
cd "$SK"
python3 edsdk.py call open_file file_path="<绝对路径>"
python3 edsdk.py call sheet_get_sheet_info file_id="<绝对路径>"
# 逐 sheet 取 CSV（end_row 取 used_row_count-1，end_col 取 used_col_count-1）
python3 edsdk.py call sheet_get_cell_data file_id="<绝对路径>" \
  sheet_id=000001 start_row=0 start_col=0 end_row=61 end_col=14 return_csv=true
```

**降级通道**：`python3 scripts/dump_sheets.py <xlsx路径> --list`
⚠️ openpyxl 只读缓存不计算，实测这类文件缓存大面积陈旧（见「坑 6」）。**仅在 editor_sdk 完全不可用时使用，且必须向用户声明数据可能不准。**

读取要求：
- **多 sheet 必须全读**。个人记账表普遍按月分 sheet，历史趋势是报告里最有价值的部分。
- **各月列数可能不同**（用户会逐月加列）。不要用同一套列索引硬解析，按表头文字定位。
- 读完立刻做**交叉校验**：各分类市值之和是否等于「总计」行。不等就说明有隐藏行或解析错位，必须查清再继续。

详细字段识别规则 → `references/data-extraction.md`

### Step 2 · 算指标 + 打分

必算的 12 个指标、六维评分规则、24 条问题检测库 → `references/diagnostic-framework.md`

**三条铁律：**
1. **全口径与分口径必须分开说**。房产/车产的浮亏经常把金融资产的浮盈全额吃掉，只报「金融资产 +X%」是误导。
2. **必须主动找表里的错误**。实测用户表格的高频错误：收益率公式符号错（亏损显示为正收益）、`#DIV/0!` 批量残留、口径中途变更未追溯历史、记账断层。这些要单独成段提示。
3. **被动收入占比是全局最重要的单一指标**。它决定了用户离「资产替工资干活」有多远。注意很多表把债基/货基收益计入「资产浮动」而没确认为「被动收入」，会使该指标严重低估——必须识别并重算真实值。

### Step 3 · 生成建议

建议必须满足：
- **按"确定性收益"排序，不按金额排序**。还一笔年化 15% 的信用卡债 > 任何固收配置。这是最常被用户自己忽略的排序。
- **每条带金额、带动作、带收益测算**。写「清零花呗 ¥3,085.87，年化省息约 15%」，不写「建议优化负债结构」。
- **分三阶段**：0-1 月（确定性收益）/ 1-3 月（结构优化）/ 3-12 月（长期机制）。
- **必查三个缺口**：保险（记账表 90% 没有）、高股息现金流资产、黄金/另类。

规则库与目标配置模型 → `references/advice-playbook.md`

### Step 4 · 产出 HTML

复制 `assets/report-skeleton.html`，替换数据区与文案区。

**视觉规范（硬性）：**

| 项 | 值 |
|---|---|
| 背景 | `#0B0D18` + 双径向渐变光斑 |
| 主色 | 金 `#C5A569`，文字米白 `#F0EBE0` |
| 面板 | `#141829` / `#1A1F35`，描边 `#2A3150` |
| **涨/正** | 红 `#E8464B`（中国市场惯例） |
| **跌/负** | 绿 `#19B37A` |
| 字体 | 微软雅黑 / PingFang SC |
| 图表 | Chart.js 4.x CDN，必须带 `window.Chart` 存在性兜底 |

**必含 7 张图**：评分环形、六维雷达（当前 vs 健康基准）、大类配置环形、风险等级横向条、三年趋势折线（资产/负债/累计收益三轴）、月度盈亏柱状（红涨绿跌）、当前 vs 目标配置对比柱状。

**必含 7 个章节**：核心指标速览 → 健康度评分 → 资产结构解剖 → 负债结构解剖 → 趋势分析 → 问题清单（P0/P1/P2）→ 行动方案（三阶段）。

### Step 5 · 收尾

1. `node --check` 校验内联 JS（见下方「已知坑」）
2. `present_files` 打开报告
3. 文字回复里给出 3 个最重要的发现 + 行动顺序，不要复述整份报告

---

## ⚠️ 已知坑（踩过，必看）

### 坑 1：内联 JS 数组里的中文直引号会炸
把问题清单、持仓名写进 JS 数组时，中文内容里的 ASCII 直引号 `"` 会提前结束字符串。

**对策**：中文引号统一用 `「」`。写完必须校验：
```bash
python3 - <<'EOF' > /tmp/chk.js
import io,re
s=io.open('<报告路径>',encoding='utf-8').read()
io.open('/dev/stdout','w',encoding='utf-8').write(re.findall(r'<script>(.*?)</script>',s,re.S)[-1])
EOF
node --check /tmp/chk.js && echo "JS OK"
```

### 坑 2：CSS 里打错色值不报错，静默失效
`#5B6party`、`#0F1striking` 这类手滑不会报错，只会渲染成默认色。所有颜色一律用 `var(--*)` 或 6 位 hex，写完 grep 一遍 `#[0-9A-Fa-f]{6}` 之外的 `#` 开头 token。

### 坑 3：各月 sheet 列数不一致
用户会逐月加列（如后来才加「上月价」「月总收益」）。老 sheet 的「总计」行是 `成本,最新价,总收益,收益率`，新 sheet 是 `成本,上月价,最新价,上月收益,总收益,收益率,...`。**按表头定位，不按列号。**

### 坑 4：负债口径中途变更
房贷从「本息总额」改为「本金」记账时，负债总额会单月骤降几十万。**这不是还款，不能当成财富跃升。** 必须在趋势图加断点标注。

### 坑 5：房产/车产收益率公式符号错
`最新价` 列留空时公式按 0 算，导致「总收益 = 成本」「收益率 = +100%」。这是高频错误，必须单独提示并用最后一次有效估值重算。
实测根因往往比"符号错"更糟：公式写成 `=成本 − 上月价`，**列引用和方向双错**。要用 `include_formula=true` 把公式原文取出来看，不要只看结果猜。

### 坑 6：openpyxl 读到的数字可能是陈旧缓存（最危险）
个人记账表常由 WPS/非 Excel 工具保存，**写入公式但不更新计算缓存**。实测同一文件：
- 总成本 openpyxl 读到 856,332，editor_sdk 实时算出 **863,168**
- 某货基成本 openpyxl 读到 42,053，实际 **16,208**（差 2.6 倍）

且成本列常是反推公式 `=最新价 − 总收益`，**没有独立原始值**，缓存错了无法自救。
**对策：editor_sdk 是唯一可靠通道。** 若被迫用 openpyxl，只能信任手工录入的数值列，并在报告里声明数据可能不准。详见 `references/data-extraction.md` 第 5 节。

### 坑 7：同表内求和范围不一致
实测 `E47=SUM(E3:E46)` 但 `G47=SUM(G3:G43)`——最新价列漏算 3 行。漏掉的行恰好空仓时总数看起来是对的，**一旦有持仓就静默少算**。必须解析各小计公式的 `SUM(X{a}:X{b})` 区间并与真实数据行比对。

---

## 输出文件命名

`financial-health-report-<YYYYMM>.html`（英文小写连字符，避免中文路径转义问题；如需部署到 any1 线上同样适用）

## 免责声明（必须放在报告 footer）

> 本报告为基于你所提供数据的结构化分析与一般性财务规划建议，不构成任何具体证券、基金或保险产品的投资推荐。所有数字（尤其房产估值、贷款利率、月度支出）请以实际对账单复核为准；涉及提前还款的操作请先向银行确认违约金条款。
