---
name: marvin-6211-analysis
description: >-
  6+2+1+1个股深度分析。输出完整研报：Executive Summary、行业、商业模式、管理层、
  财务、估值、同业对比、投资逻辑、风险、技术面（含量化回测）、未来重大事件。
  触发词："分析"、"深度分析"、"研报"、"估值"、"目标价"、"怎么样"、"能不能买"、
  "值不值得买"、"6+2"、"全面分析"。仅支持单只股票。
metadata:
  version: 0.6.0
  author: MarvinLi
license: MIT
---

# 6+2+1+1 个股深度分析

系统性个股深度分析框架：6（行业+商业模式+管理层+财务+估值+同业对比）+ 2（投资逻辑+风险）+ 1（未来事件）+ 1（技术面）。

**输入：** 单只股票代码（如 US.AAPL, HK.00700, SH.600519）
**输出：** 中文结构化深度分析报告，含具体估值目标价

---

## 前置条件

- Futu OpenD 已启动并运行于 `127.0.0.1:11111`
- Python 3 可用，已安装 `futu-api`, `pandas`, `numpy`
- 如果 OpenD 未连接，提示用户执行 `/install-futu-opend`

---

## 输入解析

用户提供标的后，按以下规则标准化代码：

| 市场 | 格式 | 示例 |
|------|------|------|
| 美股 | `US.XXXX` | US.AAPL, US.NVDA, US.TSLA |
| 港股 | `HK.XXXXX` | HK.00700, HK.09988 |
| 沪市 | `SH.XXXXXX` | SH.600519 |
| 深市 | `SZ.XXXXXX` | SZ.000001 |

如果用户只给了名称（如"苹果"、"Apple"），先用 `get_company_profile.py` 或常识确定正确代码。

---

## API 数据解析容错规范

基于实际运行经验，各 API 返回数据有以下已知注意事项：

### 通用规则
- 所有脚本用 `--json` 参数调用时，输出为 JSON 格式
- 过滤 stderr 噪音：管道加 `2>&1 | grep -v "^\[" | grep -v "^2026"`
- **Python 解释器选择**：Windows 的 App Execution Alias 会注册一个假的 `python3` 命令（exit code 49），因此必须用以下逻辑选择解释器：
  ```bash
  # 验证 python3 是否真正可用（而非 Windows Store 重定向器）
  if python3 --version >/dev/null 2>&1; then PYTHON_BIN=python3; else PYTHON_BIN=python; fi
  ```

### 各接口数据结构注意

| 接口 | 注意事项 |
|------|----------|
| `get_snapshot.py` | 返回 `data` 字段是 **list**，需取 `data[0]`；仅含价格/OHLC/成交量，无 PE/PB/市值 |
| `get_valuation_detail.py` | 返回可能是字符串而非 dict，需先 `json.loads()` 再处理；PE/PB 值在 list 中按时间排列 |
| `get_kline.py` | K线数据日期字段 key 为 `time`（非 `time_key`）；先取5条检查 keys |
| `get_financials_statements.py` | 返回嵌套结构，外层 `data` 内含按报告期排列的 list |
| `get_company_profile.py` | 返回单个 dict，直接取字段 |
| `get_owner_plate.py` | 返回 plate list，每项含 `code`, `name`, `plate_type` |

### 估值数据获取策略

由于 `get_snapshot.py` 不含 PE/PB/市值等字段，需从以下来源组合获取：
- **当前PE/PB**: 从 `get_valuation_detail.py` 返回的时间序列取最新值
- **市值**: 从 `get_snapshot.py` 的 `last_price` × 总股本（总股本从财务数据获取）
- **EPS/BPS**: 从 `get_financials_statements.py` (statement_type=4 关键指标) 获取

---

## 完整分析工作流

按以下模块顺序执行分析。每个模块先获取数据，再进行分析和输出。

**报告顶部固定输出 Executive Summary（在所有模块分析完成后生成）。**

---

### Executive Summary（报告顶部）

在完成所有9个模块分析后，在报告最顶部生成精简摘要：

**输出格式：**
```
# {公司名} ({代码}) — 6+2+1+1 深度分析报告

## Executive Summary

| 项目 | 结论 |
|------|------|
| 综合评级 | {强烈推荐/推荐/中性/谨慎/回避} |
| 概率加权目标价 | ${XXX.XX}（当前 ${XXX.XX}，空间 {+/-XX%}） |
| 估值区间 | 保守 ${XXX} / 常规 ${XXX} / 激进 ${XXX} |
| 投资类型 | {高增长/稳定分红/周期反转/价值低估/投机} |
| 核心逻辑（一句话）| {最核心的买入/回避理由} |
| 主要风险（一句话）| {最大的单一风险} |
| 技术面信号 | {看多/看空/中性} — {当前位置简述} |
| 建议仓位 | {核心持仓5-15%/卫星配置2-5%/观察/回避} |

📅 生成时间: {YYYY-MM-DD HH:MM}
💰 当前价格: ${price} | 市值: ${market_cap}B

---
```

**评级标准：**
- 强烈推荐: 上行空间 >30% 且 风险可控 且 技术面不空
- 推荐: 上行空间 15-30% 且 基本面健康
- 中性: 上行空间 0-15% 或 多空因素均衡
- 谨慎: 下行风险 >15% 或 重大不确定性
- 回避: 基本面恶化 或 估值严重高估 或 破产风险

---

### 模块一：行业分析

**数据获取：**
```bash
if python3 --version >/dev/null 2>&1; then PYTHON_BIN=python3; else PYTHON_BIN=python; fi
SCRIPTS="$HOME/.claude/skills/futuapi/scripts/quote"

# 公司概况（含行业描述）
"$PYTHON_BIN" "$SCRIPTS/get_company_profile.py" {CODE} --json

# 所属板块
"$PYTHON_BIN" "$SCRIPTS/get_owner_plate.py" {CODE} --json
```

**分析要点：**
- 识别公司所属行业及子行业
- 判断行业生命周期阶段（萌芽/成长/成熟/衰退）
- 行业整体趋势（增长/稳定/萎缩）
- 竞争格局（寡头/分散/新进入者威胁）
- 基于公司描述和行业知识综合判断

**输出格式：**
```
## 一、行业分析

▸ 所属行业: {行业名称}
▸ 子行业/板块: {细分领域}
▸ 行业生命周期: {萌芽/成长/成熟/衰退} — {判断依据}
▸ 行业趋势: {描述趋势方向及驱动因素}
▸ 竞争格局: {描述主要竞争者及格局}
▸ 数据来源: get_company_profile, get_owner_plate
```

---

### 模块二：商业模式

**数据获取：**
```bash
# 营收构成（最近年报）
"$PYTHON_BIN" "$SCRIPTS/get_financials_revenue_breakdown.py" {CODE} --financial-type 7 --json
```

**分析要点：**
- 按业务/产品分类的营收占比及同比增长
- 按地区分类的营收占比（如有）
- 定性分析每个业务：What（做什么）、Who（服务谁）、Why（为何有竞争力）
- 各业务间协同效应分析

**输出格式：**
```
## 二、商业模式

▸ 主营业务构成:
| 业务/产品 | 营收(亿) | 占比 | 同比增长 |
|-----------|----------|------|----------|
| ... | ... | ...% | ...% |

▸ 地区构成（如有）:
| 地区 | 营收(亿) | 占比 |
|------|----------|------|
| ... | ... | ...% |

▸ 定性分析:
- {业务1}: What: ... | Who: ... | 竞争力: ...
- {业务2}: What: ... | Who: ... | 竞争力: ...

▸ 业务协同效应: {描述各业务之间如何相互促进}
▸ 数据来源: get_financials_revenue_breakdown
```

---

### 模块三：管理层

**数据获取：**
```bash
# 高管信息
"$PYTHON_BIN" "$SCRIPTS/get_company_executives.py" {CODE} --json

# 内部人持仓（仅US）
"$PYTHON_BIN" "$SCRIPTS/get_insider_holder_list.py" {CODE} --json

# 内部人交易（仅US）
"$PYTHON_BIN" "$SCRIPTS/get_insider_trade_list.py" {CODE} --num 20 --json
```

**分析要点：**
- 核心高管（CEO/CFO/CTO等）的背景：教育经历、从业经历、重要成就
- 管理层总持股比例及变动趋势
- 近期内部人买卖行为（信号解读）
- 薪酬结构是否与股东利益一致

**输出格式：**
```
## 三、管理层分析

▸ 核心管理层:
| 姓名 | 职位 | 背景概述 | 持股数 | 年薪 |
|------|------|----------|--------|------|
| ... | CEO | {教育+从业经历简述} | ... | ... |

▸ 内部人总持股比例: {X}%
▸ 近期内部人交易:
| 人员 | 方向 | 股数 | 金额 | 日期 |
|------|------|------|------|------|
| ... | 买入/卖出 | ... | ... | ... |

▸ 管理层评估: {综合评价领导力、战略执行力、利益一致性}
▸ 数据来源: get_company_executives, get_insider_holder_list, get_insider_trade_list
```

注：港股/A股无内部人交易数据，该部分标注"该市场暂不提供此数据"。

---

### 模块四：财务分析

**数据获取：**
```bash
# 利润表（最近5年年报）
"$PYTHON_BIN" "$SCRIPTS/get_financials_statements.py" {CODE} --statement-type 1 --financial-type 7 --num 5 --json

# 资产负债表（最近5年年报）
"$PYTHON_BIN" "$SCRIPTS/get_financials_statements.py" {CODE} --statement-type 2 --financial-type 7 --num 5 --json

# 现金流量表（最近5年年报）
"$PYTHON_BIN" "$SCRIPTS/get_financials_statements.py" {CODE} --statement-type 3 --financial-type 7 --num 5 --json

# 关键财务指标（最近5年）
"$PYTHON_BIN" "$SCRIPTS/get_financials_statements.py" {CODE} --statement-type 4 --financial-type 7 --num 5 --json
```

**分析要点：**

**A. 差异化指标（根据行业选择）：**
- SaaS/软件: ARR、NDR、Gross Margin、Rule of 40
- 零售/消费: 同店销售增速、库存周转天数
- 汽车/制造: 交付量、ASP、单车毛利
- 银行/金融: 净息差NIM、不良率NPL、资本充足率CET1
- 互联网券商/金融科技: 客户资产AUM、用户数增速、ARPU、付费用户转化率、交易佣金率
- 周期/资源: 产能利用率、单位成本
- 平台型: GMV、Take Rate、MAU/DAU比、用户获取成本CAC、LTV/CAC
- 通用: 营收增速、运营杠杆

**B. 通用财务健康：**
- 盈利能力: 毛利率、营业利润率、净利率、ROE趋势
- 成长性: 营收增速、净利增速、EPS增速（最近3年CAGR）
- 负债: 资产负债率、流动比率、利息覆盖倍数
- 现金流: 经营现金流/净利润比率、自由现金流趋势

**C. 破产风险评估（简化 Ohlson O-Score）：**
```
O-Score 核心因子:
- 资产负债率 (Total Liabilities / Total Assets)
- 净利润/总资产 (ROA)
- 营运资本/总资产
- 流动负债/流动资产
- 净利润是否连续两年为负

简化评估:
- 资产负债率 > 70% 且 利息覆盖 < 2 → 高风险
- 资产负债率 50-70% 且 现金流为正 → 需关注
- 资产负债率 < 50% 且 现金流充裕 → 安全
```

**输出格式：**
```
## 四、财务分析

### A. 差异化指标（{行业}特有）
| 指标 | 最新值 | YoY变化 | 行业水平 |
|------|--------|---------|----------|
| {指标1} | ... | ... | ... |

### B. 通用财务健康

▸ 盈利能力:
| 指标 | FY-4 | FY-3 | FY-2 | FY-1 | FY(最新) | 趋势 |
|------|------|------|------|------|----------|------|
| 毛利率 | | | | | | ↑/↓/→ |
| 营业利润率 | | | | | | |
| 净利率 | | | | | | |
| ROE | | | | | | |

▸ 成长性:
| 指标 | 最新YoY | 3年CAGR |
|------|---------|---------|
| 营收增速 | ...% | ...% |
| 净利增速 | ...% | ...% |
| EPS增速 | ...% | ...% |

▸ 负债与偿债能力:
| 指标 | 当前值 | 安全阈值 | 评估 |
|------|--------|----------|------|
| 资产负债率 | ...% | <60% | ✅/⚠️/❌ |
| 流动比率 | ... | >1.5 | |
| 利息覆盖倍数 | ... | >3 | |

▸ 破产风险评估: {安全 ✅ / 需关注 ⚠️ / 高风险 ❌}
  理由: {基于上述指标的判断}

### D. 股东回报分析

▸ 分红:
  - 当前股息率(TTM): X.X%（年化DPS / 当前股价）
  - 近3年股息增长CAGR: X.X%
  - 分红覆盖倍数: X.Xx（EPS / DPS，>1.5为安全，<1为不可持续）
  - 分红政策: {固定比例/逐年递增/不规则/未分红}

▸ 回购:
  - 近12个月回购金额: $XX亿（占市值 X.X%）
  - 回购趋势: {加速/稳定/减少/无回购}

▸ 综合股东回报率: X.X%（股息率 + 回购yield）
  评估: {高回报(>5%)/中等(2-5%)/低(<2%)/无回报}

注: 数据来自 get_corporate_actions_dividends 和 get_corporate_actions_buybacks（在批次5获取）。

▸ 数据来源: get_financials_statements (statement_type 1-4), get_corporate_actions_dividends, get_corporate_actions_buybacks
```

---

### 模块五：估值分析

**数据获取：**
```bash
# 当前行情快照
"$PYTHON_BIN" "$SCRIPTS/get_snapshot.py" {CODE} --json

# PE 5年历史
"$PYTHON_BIN" "$SCRIPTS/get_valuation_detail.py" {CODE} --valuation-type 1 --interval-type 6 --json

# PB 5年历史
"$PYTHON_BIN" "$SCRIPTS/get_valuation_detail.py" {CODE} --valuation-type 2 --interval-type 6 --json

# PS 5年历史（亏损公司备用）
"$PYTHON_BIN" "$SCRIPTS/get_valuation_detail.py" {CODE} --valuation-type 3 --interval-type 6 --json

# 所属板块代码（用于行业对比）— 已在模块一获取，复用数据
# 取得板块代码后获取行业估值排名（PE）
"$PYTHON_BIN" "$SCRIPTS/get_valuation_plate_stock_list.py" {PLATE_CODE} --valuation-type 1 --num 30 --json

# 行业PB排名
"$PYTHON_BIN" "$SCRIPTS/get_valuation_plate_stock_list.py" {PLATE_CODE} --valuation-type 2 --num 30 --json

# 分析师共识
"$PYTHON_BIN" "$SCRIPTS/get_research_analyst_consensus.py" {CODE} --json

# 晨星报告（公允价值）
"$PYTHON_BIN" "$SCRIPTS/get_research_morningstar_report.py" {CODE} --json
```

**A. 直接估值 — DCF两阶段模型（含正常化FCF）：**

```
所需数据（从模块四的财务数据中提取）:
- FCF = 经营性现金流 - 资本支出（从现金流量表）
- 营收增速（从利润表 YoY）
- 分析师共识增速（从 analyst_consensus）
- 总股本、现金及等价物、总负债（从资产负债表和快照）

=== 正常化FCF逻辑（解决高Capex周期低估问题）===

问题: 处于重大投资周期的公司（如AI基建、新产能建设），当年FCF被异常压低，
直接用于DCF会严重低估公司价值。

正常化判断条件（满足任一即触发）:
1. 当年Capex/营收 > 历史5年均值的1.5倍
2. 当年Capex同比增长 > 50%
3. 当年FCF为负但经营现金流为正且增长

正常化排除条件（满足任一则不做正常化，直接用原始FCF或标注不适用）:
1. 近3年营收增速均 < 10% 且 Capex持续3年高于历史均值
   （说明高Capex可能是结构性上升而非周期性投入）
2. 同行业可比公司的 Capex/营收 水平与本公司相当
   （说明这是行业常态，非公司特有的投资周期）
3. 管理层未在财报/业绩会中表示Capex将在2-3年内回归正常水平
   （缺乏Capex回落的依据）

正常化方法（通过判断条件且不被排除时执行）:
- 正常化Capex = 历史5年Capex/营收均值 × 当年营收
- 正常化FCF = 当年经营现金流 - 正常化Capex
- 在报告中明确标注: "⚠️ 已对FCF进行正常化处理（公司处于{原因}投资周期）"
- 标注正常化假设前提: "假设Capex将于{N}年后回归{X}%历史均值水平"
- 同时展示: 未正常化DCF目标价 vs 正常化DCF目标价

若无法正常化（经营现金流也为负，或被排除条件命中）:
- 标注"DCF不适用（经营现金流为负/Capex为结构性高位）"
- 改用PS估值

计算步骤:
1. 基础FCF = 正常化FCF（如触发条件）或 最近完整财年FCF
   - 若 FCF < 0 且无法正常化：标注"DCF不适用"，跳过DCF
   
2. Stage 1 增速 (Year 1-3):
   growth_stage1 = min(分析师共识增速, 历史3年营收CAGR, 30%)
   - 若无分析师数据，使用历史3年CAGR
   
3. Stage 1 增速 (Year 4-5):
   growth_y4 = growth_stage1 × 0.7
   growth_y5 = growth_stage1 × 0.5

4. 永续增长率:
   - US: g_terminal = 2.5%
   - HK: g_terminal = 2.5%
   - A股: g_terminal = 3.0%

5. WACC:
   - 无风险利率 Rf（⚠️ 假设基于2025年中利率环境，若10Y国债收益率偏离超50bp需手动调整）:
     US: 4.3% | HK: 3.8% | A股: 2.5%
   - 股权风险溢价 ERP:
     US: 5.5% | HK: 6.5% | A股: 7.0%
   - Beta 计算（从3年日K线回归）:
     Beta = Cov(r_stock, r_benchmark) / Var(r_benchmark)
     基准: US→SPY | HK→恒生指数 | A股→沪深300
     使用750日日收益率回归计算，若数据不足或R²<0.1则用行业默认值:
     科技/成长: 1.2 | 金融: 1.0 | 消费: 0.9 | 公用事业: 0.6 | 周期: 1.3
   - Cost of Equity = Rf + Beta × ERP
   - Cost of Debt = 利息支出 / 有息负债（从财务数据推算）
     若无法计算，按信用分档: 大盘蓝筹3.5% | 中等5.0% | 高杠杆/小盘6.5%
   - Tax Rate = 所得税/税前利润（从利润表推算），若无则取21%(US)/16.5%(HK)/25%(A股)
   - D/(D+E) = 有息负债 / (有息负债 + 市值)
   - WACC = CoE × E/(D+E) + CoD × (1-T) × D/(D+E)

6. 逐年折现:
   FCF_1 = FCF_0 × (1 + growth_stage1)
   FCF_2 = FCF_1 × (1 + growth_stage1)
   FCF_3 = FCF_2 × (1 + growth_stage1)
   FCF_4 = FCF_3 × (1 + growth_y4)
   FCF_5 = FCF_4 × (1 + growth_y5)
   PV_stage1 = Σ FCF_n / (1+WACC)^n

7. 终值:
   Terminal_Value = FCF_5 × (1 + g_terminal) / (WACC - g_terminal)
   PV_terminal = Terminal_Value / (1+WACC)^5

8. 企业价值与目标价:
   Enterprise_Value = PV_stage1 + PV_terminal
   Equity_Value = EV + Cash - Total_Debt
   DCF_Target_Price = Equity_Value / Total_Shares
```

**B. 间接估值（根据标的类型选择）：**

```
选择规则（扩展分类）:
- EPS > 0 且 盈利稳定 → PE估值
- 金融/重资产/周期股 → PB估值
- EPS < 0 (亏损) → PS估值
- 金融科技/互联网券商（如FUTU、TIGR）→ PE为主 + PB辅助（轻资产但有AUM）
- 平台型公司（如美团、拼多多）→ PS + PE（如已盈利）

PE估值（含动态PE上限）:

  === 动态PE上限计算 ===
  PE_cap = min(max(sector_floor, 3yr_NI_CAGR × sector_multiplier), historical_PE_7yr + 2σ, industry_ceiling)

  行业分档参数:
  | 行业分类 | PE底线(floor) | CAGR乘数 | PE天花板(ceiling) |
  |----------|---------------|----------|-------------------|
  | SaaS/软件 | 25x | 1.2 | 60x |
  | 高增长金融科技 | 20x | 1.0 | 50x |
  | 消费/平台 | 18x | 1.0 | 50x |
  | 稳定科技 | 15x | 0.8 | 40x |
  | 金融/工业 | 10x | 0.7 | 30x |
  | 公用事业/REITs | 12x | 0.5 | 22x |

  注: historical_PE 使用7-10年回溯期（覆盖完整周期），IPO不足7年的用全部可用数据
  注: 若EPS增速<5%，增速隐含PE权重降至0.15，行业均值PE权重升至0.45

  加权PE = 5年历史均值PE × 0.4 + 行业均值PE × 0.3 + (3yr_NI_CAGR × sector_multiplier) × 0.3
  合理PE = max(sector_floor, min(PE_cap, 加权PE))
  PE_Target = EPS × 合理PE  (EPS选取见下方"盈利基数分层")

PB估值:
  合理PB = (5年历史均值PB + 行业均值PB) / 2
  合理PB = max(0.5, min(8, 合理PB))  # 下限0.5x，上限8x
  PB_Target = BPS × 合理PB

PS估值:
  合理PS = (5年历史均值PS + 行业均值PS) / 2
  合理PS = max(0.5, min(20, 合理PS))  # 下限0.5x，上限20x
  PS_Target = 每股营收 × 合理PS
```

**C. 盈利基数处理流水线（三版本估值核心）：**

```
=== Step 1: 一次性收益过滤 → 输出 Cleaned NI ===

目的: 排除非经常性损益对估值基数的污染

过滤规则:
- 计算 gap = |NI - Operating_Income| / |NI|
- 若 gap > 20% 且 该差距较上年扩大 >15pp → 标记为含一次性项目
- 标记后: Cleaned_NI = Operating_Income（用于所有下游计算）
- 未标记: Cleaned_NI = NI（原值透传）
- 金融类公司（银行/券商/保险）豁免此过滤（利息收入属于经营性质）
- 输出标注: "⚠️ 已剔除非经常性损益 ${X}亿，使用经营利润作为估值基础"

注: Cleaned_NI 作为后续所有步骤的输入，包括周期触发判断和盈利基数选取

=== Step 2: 周期性评分触发（替代简单50%门槛）===

使用评分制判断是否处于周期峰值:

cyclicality_score = 0
if Cleaned_NI_YoY_growth > 50%: score += 2
if Cleaned_NI_YoY_growth > 30%: score += 1
if net_profit_margin > 1σ above 5yr_avg: score += 2
if net_profit_margin > 0.5σ above 5yr_avg: score += 1
if revenue_growth < NI_growth × 0.5: score += 1  (利润率扩张驱动，非收入驱动)
if industry_is_known_cyclical: score += 1  (券商/半导体/大宗/广告)

触发阈值: score >= 3 → 触发周期性修正
score = 2 → 部分修正（下方说明）

=== Step 2b: 前瞻性覆盖规则（防止误判结构性增长）===

即使触发周期修正，若满足以下条件则暂停正常化:
- 分析师NTM净利共识 > 当前FY净利（即市场预期盈利继续上升）
- 此时标注: "⚠️ 分析师前瞻预期高于当前峰值，暂停周期正常化，按常规流程估值"
- 回退到非触发场景的盈利基数分层

=== Step 3: 盈利基数选取 ===

【情况A: 周期触发 (score >= 3) 且未被前瞻规则暂停】

行业分档正常化系数:
| 行业周期性 | 代表行业 | 正常化系数 |
|-----------|----------|-----------|
| 高周期 | 券商、大宗商品、半导体设备 | 60-65% of peak |
| 中周期 | 消费可选、广告、金融科技 | 70-75% of peak |
| 低周期 | SaaS、医疗、公用事业 | 80-85% of peak |

三版本盈利基数:
- 保守: 3年平均 Cleaned_NI
- 常规: peak × 行业正常化系数
- 激进: 最新FY Cleaned_NI（实际值）

【情况B: 部分修正 (score = 2)】
- 保守: min(latest_FY, TTM) Cleaned_NI
- 常规: average(latest_FY, TTM) Cleaned_NI
- 激进: max(latest_FY, TTM) × 1.1
（介于完全触发和未触发之间的混合方案）

【情况C: 未触发 (score <= 1)】

盈利基数分层（加季节性守卫）:
- 先计算季节性系数: CoV = 过去8个季度NI的变异系数
- 若 CoV > 0.3（强季节性）: 不使用单季度年化，改用 max(FY, TTM)

三版本盈利基数:
- 保守: min(latest_FY, TTM) Cleaned_NI
- 常规: latest_FY Cleaned_NI
- 激进:
  - CoV <= 0.3: annualized_latest_quarter × 4（但不超常规版×1.5）
  - CoV > 0.3: max(latest_FY, TTM) Cleaned_NI

硬性约束: 激进版盈利基数 ≤ 常规版 × 1.5（防止极端偏离）

=== Step 4: 三版本增速与WACC假设 ===

| 版本 | 概率权重 | DCF增速 | WACC | PE倍数 |
|------|----------|---------|------|--------|
| 保守 | 30% | consensus × 0.7 | base + 1.5% | 历史均值PE 或 行业下沿 |
| 常规 | 50% | min(consensus, hist_CAGR, 30%) | base | 加权PE（上方公式） |
| 激进 | 20% | consensus × 1.2 (不超30%) | base - 0.5% | 动态PE_cap |

注: 概率权重为固定值（V1），未来版本可根据分析师分歧度/VIX动态调整

=== Step 5: 三版本综合目标价 ===

每个版本分别计算 DCF + 间接估值 + 分析师共识的加权值:

根据标的类型加权:
- 高成长股 (PE>25, 营收增速>25%): DCF×40% + PE×35% + 分析师共识×25%
- 稳定成长股 (PE 15-25, 营收增速10-25%): DCF×45% + PE×35% + 分析师共识×20%
- 价值股 (PE<15, 稳定分红): DCF×30% + PE×40% + 分析师共识×30%
- 金融科技/互联网券商: DCF×30% + PE×40% + 分析师共识×30%
- 传统金融股: DCF×15% + PB×55% + 分析师共识×30%
- 平台型（已盈利）: DCF×35% + PE×30% + PS×15% + 分析师共识×20%
- 亏损股: PS×50% + 分析师共识×50%

分析师共识权重调整（基于覆盖充分度）:
- 覆盖分析师 >= 10人: 使用上述完整权重
- 覆盖分析师 5-9人: 共识权重减半，差额平分给DCF和间接估值
- 覆盖分析师 < 5人: 共识权重降至5%，差额平分给DCF和间接估值
- 覆盖分析师 = 0人: 不使用分析师共识，仅用DCF+间接估值（按原比例归一化）
（覆盖人数从 get_research_analyst_consensus 返回数据中获取）

最终输出:
- 保守目标价 = 保守版加权结果
- 常规目标价 = 常规版加权结果
- 激进目标价 = 激进版加权结果
- 概率加权目标价 = 保守×30% + 常规×50% + 激进×20%

价差约束: 若 激进/保守 > 2.5:1，标注"⚠️ 估值分歧度较大，建议重点参考常规版"

=== Step 6: Sanity Check 层 ===

最终输出前执行合理性校验:
- 若 |概率加权目标价 - 当前价| / 当前价 > 40%
  且 |概率加权目标价 - 分析师共识| / 分析师共识 > 30%
  → 标注: "⚠️ 低置信度估值 — 模型输出与市场定价及分析师共识偏离较大，建议结合定性判断"
- 若 DCF终值占总企业价值 > 65%
  → 标注: "⚠️ DCF终值占比过高({X}%)，目标价对永续假设高度敏感"
```

**输出格式：**
```
## 五、估值分析

### A. 直接估值 — DCF模型

{若触发正常化:}
⚠️ **FCF正常化处理**: 公司处于{原因}投资周期，当年Capex/营收={X}%（历史均值{Y}%），
已将Capex正常化至历史均值水平计算。

▸ 关键假设:
| 参数 | 数值 | 来源/依据 |
|------|------|-----------|
| 基础FCF | $XXX亿 | FY{年度}现金流量表{是否正常化} |
| Stage 1增速(Y1-3) | XX% | min(分析师共识/历史CAGR/30%) |
| Stage 1增速(Y4-5) | XX%→XX% | 衰减至0.7×/0.5× |
| 永续增长率 | X.X% | {市场}长期GDP增速参考 |
| 无风险利率 | X.X% | {市场}10Y国债收益率 |
| Beta | X.X | 行业平均/K线回归 |
| WACC | X.X% | Rf + Beta×ERP加权 |

▸ DCF计算:
| 年份 | 预测FCF | 折现因子 | 现值 |
|------|---------|----------|------|
| Y1 | $XX亿 | 0.XX | $XX亿 |
| Y2 | $XX亿 | 0.XX | $XX亿 |
| Y3 | $XX亿 | 0.XX | $XX亿 |
| Y4 | $XX亿 | 0.XX | $XX亿 |
| Y5 | $XX亿 | 0.XX | $XX亿 |
| 终值 | $XXXX亿 | 0.XX | $XXX亿 |

  企业价值 = $XXX亿
  + 现金 $XX亿 - 负债 $XX亿
  = 股权价值 $XXX亿
  ÷ 总股本 XX亿股

  ⭐ **DCF目标价: $XXX.XX**
  {若正常化: ⭐ 未正常化DCF目标价: $XXX.XX（仅供参考）}

▸ 敏感性分析A（WACC vs 永续增长率）:
| WACC \ 永续增长率 | g-2% | g-1% | g(base) | g+1% | g+2% |
|-------------------|------|------|---------|------|------|
| WACC-1% | $XX | $XX | $XX | $XX | $XX |
| WACC(base) | $XX | $XX | **$XX** | $XX | $XX |
| WACC+1% | $XX | $XX | $XX | $XX | $XX |

▸ 敏感性分析B（WACC vs Stage1增速）:
| WACC \ Stage1增速 | s-10% | s-5% | s(base) | s+5% | s+10% |
|-------------------|-------|------|---------|------|-------|
| WACC-1% | $XX | $XX | $XX | $XX | $XX |
| WACC(base) | $XX | $XX | **$XX** | $XX | $XX |
| WACC+1% | $XX | $XX | $XX | $XX | $XX |

注: 表B帮助理解"如果高增速假设偏乐观/保守，对目标价的影响幅度"。

### B. 间接估值 — {PE/PB/PS}法

▸ 标的分类: {高成长/稳定成长/价值/金融科技/传统金融/平台型/亏损}
▸ 选择理由: {为什么选择这种估值方法}
▸ 当前{PE/PB/PS}(TTM): {X.X}
▸ 5年历史均值: {X.X}
▸ 历史分位: {XX}%（当前位于5年{低/中/高}位）
▸ 行业平均: {X.X}（来自板块排名数据）
▸ 行业排名: 第{X}名/共{N}家（{分位}%）
▸ 合理{PE/PB/PS}: {X.X}（计算过程: ...）

  ⭐ **间接估值目标价: $XXX.XX**
  (= {EPS/BPS/每股营收} × 合理{PE/PB/PS})

### C. 三版本综合估值

▸ 盈利基数处理:
  - 一次性过滤: {已剔除/未触发} {若剔除: "剔除非经常性损益$X亿"}
  - 周期性评分: {X}分 → {触发/部分修正/未触发}
  - 前瞻覆盖: {NTM共识 vs 峰值的关系，是否暂停正常化}

▸ 三版本盈利基数:
| 版本 | 盈利基数 | 来源 |
|------|----------|------|
| 保守 | $XX亿 | {3年均值/min(FY,TTM)/...} |
| 常规 | $XX亿 | {峰值×系数/FY/avg(FY,TTM)} |
| 激进 | $XX亿 | {实际FY/max(FY,TTM)/季度年化} |

▸ 三版本估值结果:

| 版本 | 概率 | DCF目标 | PE目标 | 综合目标价 | vs当前价 |
|------|------|---------|--------|-----------|----------|
| 保守 | 30% | $XXX | $XXX | **$XXX** | +/-XX% |
| 常规 | 50% | $XXX | $XXX | **$XXX** | +/-XX% |
| 激进 | 20% | $XXX | $XXX | **$XXX** | +/-XX% |

⭐ **概率加权目标价: $XXX.XX**（当前 $XXX.XX，空间 +/-XX%）

▸ 各版本核心假设:
- 保守: {一句话说明假设场景}
- 常规: {一句话说明假设场景}
- 激进: {一句话说明假设场景}

▸ Sanity Check:
{若触发: ⚠️ 低置信度/终值占比过高/价差过大 等标注}
{若未触发: ✅ 通过合理性校验}

▸ 交叉验证:
| 参考来源 | 价格 | 本报告偏差 |
|----------|------|-----------|
| 晨星公允价值 | $XXX | +/-XX% |
| 分析师共识(n=X) | $XXX | +/-XX% |

▸ 晨星公允价值（如有）: $XXX — {与本报告目标价的偏离度}
▸ 数据来源: get_financials_statements, get_valuation_detail, get_valuation_plate_stock_list, get_research_analyst_consensus, get_research_morningstar_report
```

---

### 模块六：同业对比

**数据来源：** 复用模块五中 `get_valuation_plate_stock_list` 的行业排名数据，取同行业3-5家可比公司。

**可比公司选择逻辑：**
- 从行业板块排名中，选取市值相近或业务模式相似的3-5家公司
- 优先选择投资者常做对比的公司（如 FUTU vs TIGR，AAPL vs MSFT）
- 如果板块数据中可比公司不足，基于行业知识补充（标注"基于行业知识"）

**对比维度：**

| 维度 | 指标 |
|------|------|
| 规模 | 市值、营收 |
| 成长 | 营收增速、净利增速 |
| 盈利 | 毛利率、净利率、ROE |
| 估值 | PE、PB、PS |
| 效率 | 行业特有指标（如券商用ARPU/AUM，SaaS用NDR） |

**输出格式：**
```
## 六、同业对比

▸ 可比公司选择: {选择逻辑说明}

| 指标 | {本公司} | {可比1} | {可比2} | {可比3} | 行业均值 |
|------|----------|---------|---------|---------|----------|
| 市值($B) | | | | | |
| 营收增速 | | | | | |
| 净利率 | | | | | |
| ROE | | | | | |
| PE(TTM) | | | | | |
| PB | | | | | |

▸ 相对估值定位:
- vs 行业均值: PE {溢价/折价} {X}%，PB {溢价/折价} {X}%
- 溢价/折价是否合理: {判断，基于成长性差异、市场地位等}

▸ 核心差异化优势: {相对同业的独特竞争力}
▸ 数据来源: get_valuation_plate_stock_list, 行业知识
```

---

### 模块七：投资逻辑 (Bull Case)

**数据来源：** 综合前六个模块的分析结果 + 晨星报告中的 bull_say

**分析要点：**
- 提炼3-5个核心买入理由（基于前述分析的客观数据）
- 判断投资类型：高增长 / 爆发潜力 / 稳定分红 / 周期反转 / 价值低估
- 建议投资组合定位：核心持仓 / 卫星配置 / 投机仓位

**输出格式：**
```
## 七、投资逻辑 (Bull Case)

▸ 核心买入理由:
1. {理由1 — 基于具体数据}
2. {理由2 — 基于具体数据}
3. {理由3 — 基于具体数据}

▸ 投资类型: {类型}
▸ 组合定位建议: {定位} — {理由}
▸ 晨星多头观点: {bull_say内容，如有}
```

---

### 模块八：风险分析 (Bear Case)

**数据获取：**
```bash
# 股东变动（看机构减持）
"$PYTHON_BIN" "$SCRIPTS/get_shareholders_holding_changes.py" {CODE} --filter-type 0 --num 20 --json

# 股东概况（机构持仓占比）
"$PYTHON_BIN" "$SCRIPTS/get_shareholders_overview.py" {CODE} --json

# 卖空数据（仅US）
"$PYTHON_BIN" "$SCRIPTS/get_short_interest.py" {CODE} --json 2>/dev/null
```

**分析要点：**
- 从前述分析中提炼3-5个主要风险
- 引用晨星报告中的 bear_say
- 卖空数据解读（仅美股）
- 机构持仓结构及近期变动
- 管理层潜在隐忧

**输出格式：**
```
## 八、风险分析 (Bear Case)

▸ 主要风险:
1. {风险1 — 具体描述及可能影响}
2. {风险2 — 具体描述及可能影响}
3. {风险3 — 具体描述及可能影响}

▸ 晨星空头观点: {bear_say内容，如有}

▸ 机构持仓分析:
  - 机构持股占比: {X}%
  - 近期变动趋势: {增持/减持/稳定}
  - 重要机构动向: {如有显著增减持}

▸ 卖空数据（仅美股）:
  - 空头持仓比: {X}%
  - 回补天数: {X}天
  - 信号解读: {正常(<5%)/偏高(5-10%)/极端(>10%)}

▸ 管理层隐忧: {如有}
```

---

### 模块九：技术面分析

**数据获取：**
```bash
# 120日日K线
"$PYTHON_BIN" "$SCRIPTS/get_kline.py" {CODE} --ktype 1d --num 120 --json

# 52周周K线
"$PYTHON_BIN" "$SCRIPTS/get_kline.py" {CODE} --ktype 1w --num 52 --json

# 技术异常信号（30天）
TECH_SCRIPTS="$HOME/.claude/skills/futu-technical-anomaly/scripts"
"$PYTHON_BIN" "$TECH_SCRIPTS/handle_technical_anomaly.py" {CODE} --time-range 30 --json

# 3年日K线（用于量化回测 + Beta计算）
"$PYTHON_BIN" "$SCRIPTS/get_kline.py" {CODE} --ktype 1d --num 750 --json

# 市场基准3年日K线（用于Beta回归计算）
# US → US.SPY | HK → HK.800000(恒指ETF) | A股 → SH.000300(沪深300)
"$PYTHON_BIN" "$SCRIPTS/get_kline.py" {BENCHMARK_CODE} --ktype 1d --num 750 --json
```

**分析要点：**
- 当前趋势判断（上升/下降/盘整）
- MA20/MA50/MA120 相对价格位置及排列状态
- 52周高低点及当前位置
- 支撑位/阻力位识别
- 近30天技术异常信号
- 成交量趋势

**量化策略回测：**

使用获取的3年日K线数据（750个交易日），在本地执行 Python 回测脚本。3年数据可覆盖至少一轮完整牛熊周期，提升策略评估可靠性。

策略选择规则：
- 计算标的 ATR/Price 比率判断波动率，ADX 判断趋势强度
- 高波动+强趋势(ADX>25) → 选趋势跟踪策略（双均线交叉、MACD信号、突破N日高点）
- 低波动+弱趋势(ADX<20) → 选均值回归策略（RSI超买超卖、布林带）
- 中等情况 → 混合策略

回测脚本模板（生成临时 Python 文件执行）：

```python
import json, sys
import pandas as pd
import numpy as np

# K线数据从文件读取
with open(sys.argv[1], 'r') as f:
    kline_data = json.load(f)

df = pd.DataFrame(kline_data)
# 注意: K线日期字段为 'time'（非 'time_key'）
df['close'] = df['close'].astype(float)
df['high'] = df['high'].astype(float)
df['low'] = df['low'].astype(float)
df['volume'] = df['volume'].astype(float)

initial_capital = 100000
commission = 0.001  # 0.1% 单边

# === 策略定义 ===

def strategy_ma_cross(df, short=20, long=60):
    """双均线交叉"""
    df = df.copy()
    df['ma_short'] = df['close'].rolling(short).mean()
    df['ma_long'] = df['close'].rolling(long).mean()
    signals = []
    position = 0
    for i in range(long, len(df)):
        if df['ma_short'].iloc[i] > df['ma_long'].iloc[i] and df['ma_short'].iloc[i-1] <= df['ma_long'].iloc[i-1]:
            if position == 0:
                signals.append(('buy', i))
                position = 1
        elif df['ma_short'].iloc[i] < df['ma_long'].iloc[i] and df['ma_short'].iloc[i-1] >= df['ma_long'].iloc[i-1]:
            if position == 1:
                signals.append(('sell', i))
                position = 0
    return signals

def strategy_macd(df):
    """MACD信号"""
    df = df.copy()
    ema12 = df['close'].ewm(span=12).mean()
    ema26 = df['close'].ewm(span=26).mean()
    macd = ema12 - ema26
    signal = macd.ewm(span=9).mean()
    signals = []
    position = 0
    for i in range(30, len(df)):
        if macd.iloc[i] > signal.iloc[i] and macd.iloc[i-1] <= signal.iloc[i-1]:
            if position == 0:
                signals.append(('buy', i))
                position = 1
        elif macd.iloc[i] < signal.iloc[i] and macd.iloc[i-1] >= signal.iloc[i-1]:
            if position == 1:
                signals.append(('sell', i))
                position = 0
    return signals

def strategy_rsi(df, period=14, oversold=30, overbought=70):
    """RSI超买超卖"""
    df = df.copy()
    delta = df['close'].diff()
    gain = (delta.where(delta > 0, 0)).rolling(window=period).mean()
    loss = (-delta.where(delta < 0, 0)).rolling(window=period).mean()
    rs = gain / loss
    rsi = 100 - (100 / (1 + rs))
    signals = []
    position = 0
    for i in range(period+1, len(df)):
        if rsi.iloc[i] < oversold and position == 0:
            signals.append(('buy', i))
            position = 1
        elif rsi.iloc[i] > overbought and position == 1:
            signals.append(('sell', i))
            position = 0
    return signals

def strategy_bollinger(df, period=20, std_dev=2):
    """布林带"""
    df = df.copy()
    ma = df['close'].rolling(period).mean()
    std = df['close'].rolling(period).std()
    upper = ma + std_dev * std
    lower = ma - std_dev * std
    signals = []
    position = 0
    for i in range(period, len(df)):
        if df['close'].iloc[i] < lower.iloc[i] and position == 0:
            signals.append(('buy', i))
            position = 1
        elif df['close'].iloc[i] > upper.iloc[i] and position == 1:
            signals.append(('sell', i))
            position = 0
    return signals

def strategy_breakout(df, entry_period=20, exit_period=10):
    """突破N日高点"""
    signals = []
    position = 0
    for i in range(entry_period, len(df)):
        high_n = df['high'].iloc[i-entry_period:i].max()
        low_n = df['low'].iloc[i-exit_period:i].min()
        if df['close'].iloc[i] > high_n and position == 0:
            signals.append(('buy', i))
            position = 1
        elif df['close'].iloc[i] < low_n and position == 1:
            signals.append(('sell', i))
            position = 0
    return signals

def strategy_ma_volume(df, ma_period=20, vol_mult=1.5):
    """均线+放量确认"""
    df = df.copy()
    ma = df['close'].rolling(ma_period).mean()
    vol_ma = df['volume'].rolling(ma_period).mean()
    signals = []
    position = 0
    for i in range(ma_period, len(df)):
        if (df['close'].iloc[i] > ma.iloc[i] and 
            df['close'].iloc[i-1] <= ma.iloc[i-1] and
            df['volume'].iloc[i] > vol_ma.iloc[i] * vol_mult and position == 0):
            signals.append(('buy', i))
            position = 1
        elif df['close'].iloc[i] < ma.iloc[i] and position == 1:
            signals.append(('sell', i))
            position = 0
    return signals

# === 回测引擎 ===

def backtest(df, signals, strategy_name, initial_capital=100000, commission=0.001, slippage=0.001):
    capital = initial_capital
    shares = 0
    trades = 0
    equity_curve = [initial_capital]
    trades_detail = []  # 交易明细记录
    
    for action, idx in signals:
        price = df['close'].iloc[idx]
        trade_date = df['time'].iloc[idx]
        if action == 'buy':
            exec_price = price * (1 + slippage)
            shares = int(capital * (1 - commission) / exec_price)
            capital -= shares * exec_price * (1 + commission)
            trades += 1
            trades_detail.append({
                'date': str(trade_date).split(' ')[0],
                'action': '买入',
                'price': round(exec_price, 2),
                'reason': ''  # 由调用方填充
            })
        elif action == 'sell' and shares > 0:
            exec_price = price * (1 - slippage)
            capital += shares * exec_price * (1 - commission)
            shares = 0
            trades += 1
            trades_detail.append({
                'date': str(trade_date).split(' ')[0],
                'action': '卖出',
                'price': round(exec_price, 2),
                'reason': ''
            })
        current_value = capital + shares * df['close'].iloc[idx]
        equity_curve.append(current_value)
    
    # 如果最后还有持仓，按最后价格平仓
    if shares > 0:
        capital += shares * df['close'].iloc[-1] * (1 - slippage) * (1 - commission)
    
    final_value = capital
    total_return = (final_value - initial_capital) / initial_capital * 100
    days = len(df)
    annual_return = ((final_value / initial_capital) ** (252 / days) - 1) * 100
    
    # 最大回撤
    equity_series = pd.Series(equity_curve)
    peak = equity_series.cummax()
    drawdown = (equity_series - peak) / peak
    max_drawdown = abs(drawdown.min()) * 100
    
    # 夏普比率（标准计算）
    daily_returns = equity_series.pct_change().dropna()
    annual_std = daily_returns.std() * np.sqrt(252)
    rf = 0.043  # 无风险利率，与DCF模型一致
    sharpe = (annual_return / 100 - rf) / annual_std if annual_std > 0 else 0
    
    # 信号充分性标记
    sufficient = trades >= 6  # 至少3次完整买卖（6笔交易）
    
    # 当前持仓状态
    current_position = 'holding' if shares > 0 else 'empty'
    last_buy_info = None
    if current_position == 'holding':
        # 找最后一次买入记录
        for t in reversed(trades_detail):
            if t['action'] == '买入':
                last_buy_info = {'date': t['date'], 'price': t['price']}
                break
    
    return {
        'total_return': round(total_return, 2),
        'annual_return': round(annual_return, 2),
        'max_drawdown': round(max_drawdown, 2),
        'sharpe': round(sharpe, 2),
        'trades': trades,
        'sufficient': sufficient,
        'trades_detail': trades_detail,
        'current_position': current_position,
        'last_buy_info': last_buy_info
    }

# === 主流程 ===

# 判断标的类型
atr = (df['high'] - df['low']).rolling(14).mean().iloc[-1]
atr_ratio = atr / df['close'].iloc[-1]

# 计算ADX（含互斥逻辑）
plus_dm = df['high'].diff()
minus_dm = -df['low'].diff()
# +DM/-DM 互斥：仅保留较大方向，另一方向归零
plus_dm[(plus_dm < minus_dm) | (plus_dm < 0)] = 0
minus_dm[(minus_dm < plus_dm) | (minus_dm < 0)] = 0
tr = pd.concat([df['high'] - df['low'], 
                abs(df['high'] - df['close'].shift(1)),
                abs(df['low'] - df['close'].shift(1))], axis=1).max(axis=1)
atr14 = tr.rolling(14).mean()
plus_di = 100 * plus_dm.rolling(14).mean() / atr14
minus_di = 100 * minus_dm.rolling(14).mean() / atr14
dx = 100 * abs(plus_di - minus_di) / (plus_di + minus_di + 1e-10)
adx = dx.rolling(14).mean().iloc[-1]

# 选择3个策略
if atr_ratio > 0.03 and adx > 25:  # 高波动+强趋势
    selected = [strategy_ma_cross, strategy_macd, strategy_breakout]
    names = ['双均线交叉(MA20/60)', 'MACD信号', '突破20日高点']
    type_desc = '高波动趋势型'
elif atr_ratio < 0.015 and adx < 20:  # 低波动+弱趋势
    selected = [strategy_rsi, strategy_bollinger, strategy_ma_volume]
    names = ['RSI超买超卖', '布林带', '均线+放量']
    type_desc = '低波动震荡型'
else:  # 中等
    selected = [strategy_ma_cross, strategy_macd, strategy_breakout]
    names = ['双均线交叉(MA20/60)', 'MACD信号', '突破20日高点']
    type_desc = '中等波动'

# Buy & Hold 基准
bnh_return = (df['close'].iloc[-1] - df['close'].iloc[0]) / df['close'].iloc[0] * 100
bnh_annual = ((df['close'].iloc[-1] / df['close'].iloc[0]) ** (252 / len(df)) - 1) * 100
# Buy & Hold 最大回撤
bnh_equity = df['close'] / df['close'].iloc[0] * initial_capital
bnh_peak = bnh_equity.cummax()
bnh_dd = abs(((bnh_equity - bnh_peak) / bnh_peak).min()) * 100
# Buy & Hold 夏普比率（标准计算）
bnh_daily_returns = (df['close'] / df['close'].shift(1) - 1).dropna()
bnh_annual_std = bnh_daily_returns.std() * np.sqrt(252)
bnh_sharpe = (bnh_annual / 100 - 0.043) / bnh_annual_std if bnh_annual_std > 0 else 0

# 策略信号依据描述（用于操作明细表的"信号依据"列）
signal_reasons = {
    '双均线交叉(MA20/60)': {'buy': 'MA20上穿MA60', 'sell': 'MA20下穿MA60'},
    'MACD信号': {'buy': 'MACD金叉(DIF上穿DEA)', 'sell': 'MACD死叉(DIF下穿DEA)'},
    '突破20日高点': {'buy': '突破20日最高价', 'sell': '跌破10日最低价'},
    'RSI超买超卖': {'buy': 'RSI<30超卖', 'sell': 'RSI>70超买'},
    '布林带': {'buy': '触及下轨', 'sell': '触及上轨'},
    '均线+放量': {'buy': '放量站上MA20', 'sell': '跌破MA20'}
}

# 回测每个策略
results = []
for func, name in zip(selected, names):
    try:
        signals = func(df)
        result = backtest(df, signals, name)
        result['name'] = name
        # 填充每笔交易的信号依据
        reasons = signal_reasons.get(name, {'buy': '买入信号', 'sell': '卖出信号'})
        for t in result.get('trades_detail', []):
            if t['action'] == '买入':
                t['reason'] = reasons['buy']
            else:
                t['reason'] = reasons['sell']
        results.append(result)
    except Exception as e:
        results.append({'name': name, 'total_return': 0, 'annual_return': 0, 
                       'max_drawdown': 0, 'sharpe': 0, 'trades': 0, 
                       'trades_detail': [], 'current_position': 'empty',
                       'error': str(e)})

# 输出JSON结果
output = {
    'start_date': str(df['time'].iloc[0]).split(' ')[0],
    'end_date': str(df['time'].iloc[-1]).split(' ')[0],
    'trading_days': len(df),
    'atr_ratio': round(atr_ratio * 100, 2),
    'adx': round(adx, 1),
    'type': type_desc,
    'buy_hold': {
        'total_return': round(bnh_return, 2),
        'annual_return': round(bnh_annual, 2),
        'max_drawdown': round(bnh_dd, 2),
        'sharpe': round(bnh_sharpe, 2)
    },
    'strategies': results
}
print(json.dumps(output, ensure_ascii=False))
```

**输出格式：**
```
## 九、技术面分析

**补充技术面的理由:** 6+2方法解决"值不值得买"，技术面解决"何时买"和"怎么买"。
通过趋势判断避免逆势操作，支撑/阻力位辅助设定买入区间，量化回测验证策略有效性。

▸ 当前趋势: {上升趋势 📈 / 下降趋势 📉 / 盘整 ↔️}
  判断依据: {MA排列状态、价格位置}

▸ 关键价位:
  - 52周高: $XXX | 52周低: $XXX
  - 当前位于52周区间的 {XX}% 位置
  - MA20: $XXX | MA50: $XXX | MA120: $XXX
  - 支撑位: $XXX（依据: ...）
  - 阻力位: $XXX（依据: ...）

▸ MA排列: {多头排列/空头排列/交织}
  MA20 {>/<} MA50 {>/<} MA120

▸ 近30天技术信号:
  {列出从 handle_technical_anomaly 获取的信号}

▸ 技术面综合判断: {看多/看空/中性} — {简述理由}

### 量化策略回测

▸ 回测区间: {start_date} ~ {end_date}（共{N}个交易日，约{N/252:.1f}年）
▸ 标的特征: 波动率(ATR/Price)={X.X}%, 趋势强度(ADX)={XX}
▸ 标的类型: {高波动趋势型/低波动震荡型/中等波动}
▸ 策略选择理由: {为什么选择这3个策略}
▸ 回测参数: 初始资金$100,000 | 佣金0.1% | 滑点0.1% | 无杠杆

| 策略 | 买入规则 | 卖出规则 | 总收益率 | 年化收益 | 最大回撤 | 夏普比率 | 交易次数 | 有效性 |
|------|----------|----------|----------|----------|----------|----------|----------|--------|
| Buy & Hold | 首日买入持有 | 不卖出 | XX% | XX% | XX% | X.XX | 1 | ✅ |
| {策略1} | {买入信号} | {卖出信号} | XX% | XX% | XX% | X.XX | XX | ✅/⚠️ |
| {策略2} | {买入信号} | {卖出信号} | XX% | XX% | XX% | X.XX | XX | ✅/⚠️ |
| {策略3} | {买入信号} | {卖出信号} | XX% | XX% | XX% | X.XX | XX | ✅/⚠️ |

买入/卖出规则从 signal_reasons 字典中读取，各策略对应规则：
- 双均线交叉(MA20/60): 买入=MA20上穿MA60 | 卖出=MA20下穿MA60
- MACD信号: 买入=MACD金叉(DIF上穿DEA) | 卖出=MACD死叉(DIF下穿DEA)
- 突破20日高点: 买入=收盘价突破20日最高价 | 卖出=收盘价跌破10日最低价
- RSI超买超卖: 买入=RSI<30超卖 | 卖出=RSI>70超买
- 布林带: 买入=价格触及下轨 | 卖出=价格触及上轨
- 均线+放量: 买入=放量(>1.5倍均量)站上MA20 | 卖出=跌破MA20

注: 有效性⚠️表示交易次数<6笔（不足3次完整买卖），统计意义不足，结论仅供参考。

#### {策略1} 操作明细

| # | 日期 | 操作 | 价格 | 信号依据 |
|---|------|------|------|----------|
| 1 | YYYY-MM-DD | 买入 | $XX.XX | {触发条件，如: MA20上穿MA60} |
| 2 | YYYY-MM-DD | 卖出 | $XX.XX | {触发条件，如: MA20下穿MA60} |
| ... | | | | |

当前状态: {空仓 / 持仓中(自YYYY-MM-DD买入于$XX.XX)}

#### {策略2} 操作明细
（同上格式）

#### {策略3} 操作明细
（同上格式）

▸ 推荐策略: **{最佳策略名}**（仅从有效性✅的策略中选取）
  理由: {夏普比率最高/收益回撤比最优/...}
  操作建议: {当前信号状态，如"当前MA20即将上穿MA60，关注金叉信号"}

▸ 回测局限性: 若回测区间内为单边行情，策略表现可能不具普适性。
▸ 数据来源: get_kline, handle_technical_anomaly
```

---

### 模块十：未来3个月重大事件 (+1)

**数据获取：**
```bash
# 分红记录（推算下次分红日）
"$PYTHON_BIN" "$SCRIPTS/get_corporate_actions_dividends.py" {CODE} --json

# 回购记录
"$PYTHON_BIN" "$SCRIPTS/get_corporate_actions_buybacks.py" {CODE} --json

# 公司公告（仅官方公告，news_type=2）
curl -sG "https://ai-news-search.futunn.com/news_search" \
  --data-urlencode "keyword={COMPANY_NAME}" \
  --data-urlencode "size=20" \
  --data-urlencode "news_type=2" \
  --data-urlencode "sort_type=2" \
  --data-urlencode "language_id=0"
```

**事件推算逻辑：**

1. **财报日推算：**
   - 从最近发布的财报日期，按历史季度节奏推算下一次
   - 美股：通常每季度（Q1:4月, Q2:7月, Q3:10月, Q4:1月前后）
   - 港股：中报(8-9月), 年报(3-4月)
   - A股：一季报(4/30前), 中报(8/31前), 三季报(10/31前), 年报(4/30前)

2. **分红日推算：**
   - 分析历史分红频率和日期模式
   - 季度分红公司 → 推算下一季度除息日
   - 年度分红公司 → 推算下一年度除息日

3. **公告筛选：**
   - 从公告中提取未来日期相关事件
   - 筛选条件：含日期、含"计划"/"预计"/"拟"/"将于"等前瞻性词语

**输出格式：**
```
## 十、未来3个月重大事件 (+1)

数据来源仅限官方渠道（公司公告、监管文件、财报日历），不含媒体报道。

| 预计日期 | 事件 | 发生概率 | 来源 | 影响方向 | 对估值影响 |
|----------|------|----------|------|----------|------------|
| {日期}✓/≈ | {事件描述} | 高/中/低 | {来源} | 利好/利空/中性 | {影响描述} |

▸ 事件详细分析:

**{事件1}** ({日期})
- 发生概率: {高>80%/中50-80%/低<50%} — {理由，如"基于公司历史固定的季度财报发布节奏，确定性高"}
- 影响分析: {对估值的具体影响逻辑，如"若营收超分析师预期5%以上，基于历史反应模式，可能带来5-10%的估值上修"}

**{事件2}** ({日期})
- 发生概率: {高/中/低} — {理由}
- 影响分析: {影响逻辑}

▸ 数据来源: get_corporate_actions_dividends, get_corporate_actions_buybacks, Futu News API (news_type=2公告)
```

---

## 报告结尾

```
---

⚠️ **免责声明**
本报告基于富途OpenAPI公开数据自动生成，仅供学习研究参考，不构成任何投资建议。
投资有风险，入市需谨慎。报告中的估值模型基于假设，实际股价受多种不可预测因素影响。

📊 数据来源: 富途 OpenAPI (通过 OpenD 127.0.0.1:11111)
📐 分析框架: 6+2+1+1分析法（v0.6.0）
📅 报告生成时间: {当前时间}
```

---

## 执行节奏

为避免 API 限流（30秒内最多30次请求），按以下批次执行数据获取：

**批次1（基础信息，并行）：**
- get_company_profile
- get_owner_plate
- get_snapshot

**批次2（商业模式+管理层，并行）：**
- get_financials_revenue_breakdown
- get_company_executives
- get_insider_holder_list (US only)
- get_insider_trade_list (US only, --num 20)

**批次3（财务数据，间隔1秒）：**
- get_financials_statements type=1
- get_financials_statements type=2
- get_financials_statements type=3
- get_financials_statements type=4

**批次4（估值+研究，并行）：**
- get_valuation_detail (PE, valuation-type=1)
- get_valuation_detail (PB, valuation-type=2)
- get_valuation_detail (PS, valuation-type=3)
- get_research_analyst_consensus
- get_research_morningstar_report

**批次5（行业对比+股东，并行）：**
- get_valuation_plate_stock_list (PE排名, 用批次1的板块代码)
- get_valuation_plate_stock_list (PB排名)
- get_shareholders_overview
- get_shareholders_holding_changes
- get_corporate_actions_dividends
- get_corporate_actions_buybacks

**批次6（技术面+Beta基准，分两组各间隔1秒）：**
- 组A: get_kline {CODE} 120日日K、get_kline {CODE} 52周周K
- 组B: get_kline {CODE} 750日日K（回测+Beta用）、get_kline {BENCHMARK} 750日日K（Beta基准）
- handle_technical_anomaly

**批次7（事件）：**
- Futu News API (公告)

获取完所有数据后，按模块顺序分析并输出报告。最后回到顶部生成 Executive Summary。

---

## 错误处理

| 错误场景 | 处理方式 |
|----------|----------|
| OpenD 未连接 | 提示: "OpenD未运行，请先启动 OpenD 或执行 /install-futu-opend" |
| 某个API调用失败 | 该模块标注 "⚠️ 数据暂不可用"，继续其余模块 |
| FCF为负（可正常化） | 使用正常化FCF计算DCF，标注正常化原因 |
| FCF为负（不可正常化） | DCF标注"不适用（经营现金流为负）"，间接估值用PS法替代 |
| EPS为负 | PE法标注"不适用（亏损）"，用PS法 |
| 金融类公司 | 跳过DCF，PB为主要估值方法 |
| 内部人数据（非US） | 标注"该市场暂不提供此数据" |
| API限流 | 等待5秒后重试一次 |
| K线数据不足750天 | 回测使用可用数据，标注"回测周期: {实际天数}天（不足3年，结论参考性降低）" |
| snapshot无PE/PB字段 | 从valuation_detail最新值获取，或从财务数据计算 |
| valuation_detail返回字符串 | 先json.loads()解析再处理 |
| 板块排名数据不足 | 同业对比标注"可比公司数据有限"，基于行业知识补充 |
| API返回空数据(空list/null) | 标注"该数据暂无"，跳过依赖此数据的计算，不报错 |
| Beta回归R²<0.1 | 回归无效，改用行业默认Beta值并标注 |
| 分析师覆盖为0 | 综合目标价中去掉分析师共识，仅用DCF+间接估值 |
| 回测策略交易次数<6 | 该策略标注⚠️"信号不足"，不参与最佳策略推荐 |
