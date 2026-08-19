# 墨西哥玩具零售经营与库存配置分析

**Mexico Toy Retail Sales & Inventory Allocation Analysis**

## 项目简介

本项目基于 Mexico Toy Sales 公开练习数据，使用 **Python、Pandas、Jupyter Notebook 与 Power BI / PBIP**，搭建了一套从零售经营诊断、商品优先级识别，到库存健康评估和库存配置决策支持的完整分析流程。

项目共处理 **829,262 条销售记录、35 个商品和 50 家门店**。观察期累计销售额为 **14.44M**，毛利润为 **4.01M**，整体毛利率为 **27.79%**。

库存分析进一步识别出 **77 个活跃需求缺货 Store-SKU** 和 **289 个高风险库存组合**。在以最近 90 天销售速度为需求基准、Receiver 补至 14 天库存覆盖、Donor 保留 45 天库存覆盖的 Base Scenario 下，**7,622 件风险需求中理论可由现有跨门店库存匹配 1,426 件，覆盖 18.71% 的需求量和 23.21% 的毛利润机会，剩余 6,196 件进入补货优先级**。

本项目中的库存配置结果均为规则型情景模拟，用于支持调拨候选和补货优先级判断，不代表真实调拨执行结果或最优库存方案。

---

## 业务问题

项目主要回答以下三个问题：

1. **当前销售和盈利表现怎么样？**  
   从销售额、销量、毛利润、毛利率和月度环比等指标监控整体经营表现，并定位观察期内值得进一步分析的变化。

2. **哪些商品和库存风险应该优先处理？**  
   基于毛利润贡献进行 ABC / Pareto 商品分级，再结合近期销售速度和库存覆盖天数识别缺货、低库存、过量库存和近期无销量库存。

3. **现有库存能否重新配置以缓解短缺？**  
   检查同一 SKU 在不同门店之间是否存在缺货与库存过剩并存的情况，并通过规则型调拨情景估算内部库存能够覆盖多少需求，剩余需求再进入补货优先级。

---

## 分析流程

整个项目按六个业务阶段展开：

```text
业务目标与数据基础
        ↓
销售与盈利表现诊断
        ↓
商品优先级
        ↓
库存健康与风险评估
        ↓
库存配置决策支持
        ↓
情景验证与业务建议
```

---

## 1. 业务目标与数据基础

首先确认四张核心数据表的粒度和关联关系：

- `sales`：一条销售记录
- `products`：一个商品
- `stores`：一家门店
- `inventory`：一个实际记录的 Store-SKU 库存组合

主要进行：

- 日期和金额字段清洗
- 缺失值与重复值检查
- 主键 / 候选键验证
- 孤立外键检查
- `many_to_one` 关联粒度验证
- Store-SKU 库存组合完整性检查

理论上：

```text
50 stores × 35 products = 1,750 Store-SKU
```

实际 `inventory` 只有：

```text
1,593 Store-SKU
```

因此存在 **157 个缺失库存记录组合**。

进一步检查发现：

- 41 个缺失组合历史上产生过销量
- 23 个缺失组合最近 90 天仍有销量

这些记录统一标记为：

> **Inventory Unknown / Inventory Data Exception**

数据缺失不等于库存为 0，因此不将其纳入缺货判断。

---

## 2. 销售与盈利表现诊断

销售事实表计算：

```text
Sales = Units × Product_Price

COGS = Units × Product_Cost

Gross Profit = Sales - COGS

Gross Margin = Gross Profit / Sales
```

### 核心经营结果

| 指标 | 结果 |
|---|---:|
| Sales Records | 829,262 |
| Total Units | 1,090,565 |
| Total Sales | 14,444,572.35 |
| Gross Profit | 4,014,029.00 |
| Gross Margin | 27.79% |

金额均使用**源数据提供的金额单位**。

由于原始数据缺少可信的 currency 字段，本项目不对 MXN 或 USD 作确定性声明。

### 月度变化分析

按月监控：

- Sales
- Units
- Gross Profit
- Gross Margin
- Sales MoM
- Gross Profit MoM

观察期内最大的销售额月度下降发生在：

> **2018-08 vs 2018-07：Sales MoM = -20.22%**

销售额由约 **828,348.86** 降至 **660,877.07**，净减少约 **167,471.79**。

进一步进行 Contribution Analysis，用于判断这次下降主要由哪些品类、商品和门店贡献。

### Category 主要负向贡献

| Category | Delta Sales | 净下降贡献 |
|---|---:|---:|
| Toys | -61,223.47 | 36.56% |
| Art & Crafts | -51,567.35 | 30.79% |
| Sports & Outdoors | -30,167.58 | 18.01% |
| Games | -20,515.12 | 12.25% |
| Electronics | -3,998.27 | 2.39% |

其中：

> **Toys 与 Art & Crafts 合计贡献约 67.35% 的净下降。**

分析继续下钻到 Product 和 Store 层级，用于定位变化主要集中在哪些业务对象，但不将贡献变化解释为因果关系。

---

## 3. 商品优先级：ABC / Pareto

项目按照完整历史 **Gross Profit Contribution** 对商品进行 ABC 分级。

相比单纯按照销售额排序，毛利润更适合作为本项目的商品优先级基础，因为在库存资源有限时，高毛利润商品的缺货通常具有更高的经营关注价值。

分类规则：

- A：累计毛利润贡献 ≤ 80%
- B：累计毛利润贡献 80%–95%
- C：累计毛利润贡献 95%–100%

结果：

| ABC | 商品数 | 历史毛利润占比 |
|---|---:|---:|
| A | 15 | 79.65% |
| B | 9 | 14.79% |
| C | 11 | 5.56% |

即：

> **15 个 A 类商品贡献约 79.65% 的历史毛利润。**

后续库存风险不再只看“有没有缺货”，而是结合 ABC 判断哪些风险更值得优先处理。

---

## 4. 库存健康与风险评估

### 近期需求基准

以最大销售日期 **2018-09-30** 为终点，使用最近 90 个自然日：

```text
2018-07-03 → 2018-09-30
```

计算每个 Store-SKU：

```text
Recent_90D_Units

Avg_Daily_Demand_90D =
Recent_90D_Units / 90
```

这里的日均需求表示：

> **近期历史销售速度**

而不是需求预测。

### 库存覆盖天数

对于近期有销量的 Store-SKU：

```text
Days Cover =
Stock_On_Hand / Avg_Daily_Demand_90D
```

根据库存和近期需求，将 1,593 个实际库存组合划分为：

| 库存状态 | 定义 | Store-SKU |
|---|---|---:|
| Active Stockout | 库存 = 0，且最近 90 天有销量 | 77 |
| Critical | 0 < Days Cover ≤ 7 | 289 |
| Low Stock | 7 < Days Cover ≤ 14 | 267 |
| Healthy | 14 < Days Cover ≤ 60 | 661 |
| Overstock | Days Cover > 60 | 206 |
| Dormant | 有库存，但最近 90 天无销量 | 93 |

这些 **7 / 14 / 60 天阈值属于分析情景参数**，不代表企业真实安全库存或补货政策。

### Dormant Inventory

93 个近期无销量 Store-SKU 共包含：

- **1,349 units**
- 库存成本 **16,656.51**

这些组合被标记为：

> **Dormant / No Recent Sales Inventory**

不直接解释为永久滞销，因为可能仍受到季节性、低频需求或分析窗口的影响。

---

## 5. ABC × 库存风险优先级

将商品 ABC 与库存健康状态结合，建立分析优先级。

例如：

- A + Active Stockout → P1
- A + Critical → P1
- B + Active Stockout / Critical → P2
- A + Low Stock → P2
- C + Overstock / Dormant → Inventory Reduction Candidate

最终识别：

> **184 个 A 类 Active Stockout / Critical Store-SKU 构成最高优先级 P1。**

这些 P1 组合对应：

| 指标 | 结果 |
|---|---:|
| P1 Store-SKU | 184 |
| Need | 4,815 |
| GP Opportunity Exposure | 16,972.00 |
| Base Receiver GP Opportunity 占比 | 69.54% |

这说明库存风险并不是平均分布的，高价值商品集中贡献了较大部分潜在经营机会暴露。

---

## 6. 库存配置决策支持

### Receiver 定义

Base Scenario 的 Receiver 只包括：

- Active Stockout
- Critical

Receiver 的目标库存覆盖设置为：

```text
14 days
```

需求量计算：

```text
Need_Qty =
max(
    ceil(
        Avg_Daily_Demand_90D × 14
        - Stock_On_Hand
    ),
    0
)
```

结果：

| 指标 | 结果 |
|---|---:|
| Receiver Store-SKU | 366 |
| Receiver Need | 7,622 |
| Sales Opportunity Exposure | 88,867.78 |
| GP Opportunity Exposure | 24,407.00 |

其中 Opportunity Exposure 仅用于衡量风险规模，不代表实际销售损失。

---

### Donor 定义

Base Scenario 要求 Donor 调出库存后仍至少保留：

```text
45 days
```

可释放库存计算：

```text
Excess_Qty =
max(
    floor(
        Stock_On_Hand
        - Avg_Daily_Demand_90D × 45
    ),
    0
)
```

近期无销量的 Dormant Inventory 不直接作为 Base Donor，以避免假设这些库存可以全部释放。

Base Scenario 下：

| 指标 | 结果 |
|---|---:|
| Effective Donor Store-SKU | 290 |
| Donor Excess | 3,331 |

---

## Same-SKU Cross-Store Imbalance

项目进一步检查：

> 同一个 SKU 是否在部分门店库存紧张，同时其他门店存在可以释放的库存。

结果显示：

> 所有 Active Stockout / Critical Receiver 所涉及的 SKU，都可以在其他门店找到至少一个有效 Donor。

但：

> **存在 Donor ≠ Donor 数量足够。**

不同 SKU 的 Need 和 Excess 在数量上并不完全匹配，因此库存问题同时包含：

1. 门店之间的库存配置错位
2. 部分 SKU 整体库存不足
3. Excess 集中在并非最紧缺的 SKU

内部调拨可以缓解第一类问题，但无法解决全部短缺。

---

## Base Scenario

Base 参数：

```text
Demand Window = 90D
Receiver Target = 14D
Donor Reserve = 45D
```

结果：

| 指标 | 结果 |
|---|---:|
| Receiver Store-SKU | 366 |
| Receiver Need | 7,622 |
| Donor Excess | 3,331 |
| Matched Units | 1,426 |
| Unit Coverage | 18.71% |
| GP Opportunity Coverage | 23.21% |
| Remaining Need | 6,196 |

核心结论：

> **7,622 件风险需求中，理论上有 1,426 件可通过现有跨门店库存重新配置覆盖，占需求量的 18.71%，对应 23.21% 的毛利润机会；剩余 6,196 件仍需要进入补货优先级。**

因此，本项目没有得出：

> “通过内部调拨即可解决库存问题。”

而是：

> **内部调拨可以缓解部分短缺，但不能替代补货。**

这也说明库存问题不仅是门店之间的配置问题，同时存在 SKU 层面的整体供应不足。

---

## 行动分层

基于 Receiver 级别的确定性调拨模拟，将库存风险进一步转化为行动建议。

调拨顺序为：

1. Active Stockout 优先于 Critical
2. 同一 SKU 内按照 GP Opportunity 从高到低处理
3. 优先匹配 Same-City Donor
4. 剩余需求再匹配 Cross-City Donor
5. 无法覆盖的需求进入 Replenishment Priority

最终结果：

| 行动类型 | Receiver 数 |
|---|---:|
| Same-City Reallocation Candidate | 40 |
| Cross-City-only Reallocation Candidate | 74 |
| Replenishment Priority | 252 |

进一步得到：

| 指标 | 结果 |
|---|---:|
| Receiver with Any Match | 114 |
| Fully Covered Receiver | 94 |
| Partially Matched Receiver | 20 |
| Receiver with Remaining Need | 272 |
| Remaining Replenishment Need | 6,196 |

需要注意：

> 一个 Receiver 即使获得部分调拨，也可能仍然存在 Remaining Need。

因此，“调拨候选”和“补货需求”并不意味着该 Receiver 的所有缺口都已经被解决。

---

## Same-City 保守约束

为了进一步测试调拨在简单地理约束下的可执行性，项目加入：

```text
Receiver Store_City = Donor Store_City
```

即：

> 优先检查同一城市内部是否可以完成 Same-SKU 调拨。

结果：

| 指标 | Same-City |
|---|---:|
| Matched Units | 342 |
| Unit Coverage | 4.49% |
| GP Opportunity Coverage | 5.97% |

Same-City 只能覆盖较小部分需求，说明主要理论调拨空间来自跨城市库存配置。

但 `Store_City` 仍然只是一个简单地理代理，因为数据缺少：

- 门店间实际距离
- 运输成本
- 调拨时间
- 配送网络
- 运输能力

因此 Same-City 结果不能解释为真实物流可执行方案。

---

## 7. 情景验证

### 调拨策略情景

固定：

```text
Demand Window = 90D
Receiver Target = 14D
```

只改变 Donor Reserve。

| Scenario | Donor Reserve | Unit Coverage | GP Opportunity Coverage |
|---|---:|---:|---:|
| Conservative | 60D | 13.61% | 17.46% |
| Base | 45D | 18.71% | 23.21% |
| Aggressive | 30D | 26.96% | 33.19% |

结果表明：

> Donor 保留库存要求越低，可用于内部调拨的库存越多，覆盖率也越高。

但即使在 Aggressive Scenario 下：

> **内部调拨仍然只能覆盖部分需求。**

因此情景分析的作用不是寻找一个“最优参数”，而是验证不同库存政策假设下结论是否发生明显变化。

---

### Demand Window Sensitivity

固定：

```text
Receiver Target = 14D
Donor Reserve = 45D
```

分别使用：

- 30D demand
- 60D demand
- 90D demand

结果：

| Demand Window | Receiver Need | Matched Units | Unit Coverage | GP Opportunity Coverage |
|---|---:|---:|---:|---:|
| 30D | 8,898 | 1,872 | 21.04% | 22.18% |
| 60D | 7,249 | 1,695 | 23.38% | 27.40% |
| 90D | 7,622 | 1,426 | 18.71% | 23.21% |

虽然具体覆盖率会随着需求窗口变化，但核心业务结论没有发生反转：

> **存在 Same-SKU 跨门店库存错配，但内部调拨只能缓解部分库存风险，多数风险需求仍需进入补货优先级。**

项目最终选择 90D 作为 Base Scenario，是为了使用更稳定的近期历史需求窗口，而不是为了选择覆盖率最高的参数。

---

# Power BI Dashboard

最终 Power BI 项目包含三页 Dashboard。

## 01 经营表现

回答：

> **整体销售和盈利表现怎么样？**

主要包括：

- 销售额
- 销售数量
- 毛利润
- 毛利率
- 销售额环比
- 月度销售额与毛利润趋势
- 品类销售与毛利润表现
- 毛利润最高商品 Top 8
- 销售额最高门店 Top 8
- 2018 年 8 月主要销售变化提示

---

## 02 库存风险

回答：

> **当前库存风险集中在哪里，哪些高价值商品需要优先处理？**

主要包括：

- 库存成本
- Active Stockout
- Critical
- Dormant Inventory Cost
- Base GP Opportunity
- 库存健康状态分布
- ABC × Inventory Risk
- 高价值库存风险分布
- 高优先级 Store-SKU
- Inventory Exception

---

## 03 配置决策

回答：

> **现有库存能够缓解多少短缺，哪些需求仍需要补货？**

主要包括：

- Receiver Need
- Matched Units
- Unit Coverage
- GP Opportunity Coverage
- Remaining Need
- Action Distribution
- Matched vs Remaining Need
- Policy Scenario Coverage
- Demand Window Sensitivity
- Same-City Constraint
- Action Priority

---

# 分析方法

## 经营监控与诊断

- KPI / OSM Framework
- Monthly Trend
- Month-over-Month Analysis
- Contribution Analysis
- Category / Product / Store Segmentation

## 商品与库存优先级

- ABC / Pareto Analysis
- Recent Demand
- Days of Cover
- Inventory Health Classification
- ABC × Inventory Risk

## 决策支持

- Opportunity Exposure
- Same-SKU Cross-Store Imbalance
- Receiver / Donor Classification
- Reallocation Scenario Analysis
- Action Classification
- Policy Scenario Analysis
- Sensitivity Analysis

---

# 技术栈

- Python
- Pandas
- Jupyter Notebook
- Power BI / PBIP

本项目**不使用 SQL**。

同时没有为了增加方法数量而加入与当前业务问题不匹配的：

- AARRR
- A/B Test
- Machine Learning
- Forecasting Model
- Linear Programming / Optimization Algorithm

分析方法均围绕：

> **经营诊断、库存风险识别和库存配置决策支持**

展开。

---

# 最终数据输出

运行 Notebook 后会在 `output/` 中生成七张 Power BI 输入表：

```text
sales_fact.csv
products.csv
stores.csv
inventory_fact.csv
inventory_actions.csv
inventory_exceptions.csv
scenario_summary.csv
```

主要粒度：

| 表 | 粒度 |
|---|---|
| `sales_fact.csv` | 一条销售记录 |
| `products.csv` | 一个商品 |
| `stores.csv` | 一家门店 |
| `inventory_fact.csv` | 一个实际 Store-SKU |
| `inventory_actions.csv` | 一个 Base Receiver Store-SKU |
| `inventory_exceptions.csv` | 一个缺失 inventory Store-SKU |
| `scenario_summary.csv` | 一个分析情景 |

其中：

- `sales_fact.csv`：829,262 行
- `products.csv`：35 行
- `stores.csv`：50 行
- `inventory_fact.csv`：1,593 行
- `inventory_actions.csv`：366 行
- `inventory_exceptions.csv`：157 行

---

# Power BI 数据模型

Power BI 使用星型模型组织主要数据。

主要关系包括：

```text
products 1 ─── * sales_fact
products 1 ─── * inventory_fact
products 1 ─── * inventory_actions
products 1 ─── * inventory_exceptions

stores 1 ─── * sales_fact
stores 1 ─── * inventory_fact
stores 1 ─── * inventory_actions
stores 1 ─── * inventory_exceptions

DateTable 1 ─── * sales_fact
```

`scenario_summary` 作为独立情景表使用，不与事实表建立强制关系。

模型避免：

- Fact-to-Fact Relationship
- Many-to-Many Relationship
- 不必要的双向筛选

Python 负责：

- 数据清洗
- ABC 分类
- 需求计算
- 库存健康分类
- Receiver / Donor 判断
- Opportunity Exposure
- 调拨模拟
- Scenario
- Sensitivity
- Inventory Exception

Power BI / DAX 主要负责：

- 指标聚合
- 筛选交互
- Dashboard 展示

---

# 项目结构

```text
mexico-toy-sales-inventory-analysis/
├── README.md
├── .gitignore
├── mexico_toy_sales_analysis.ipynb
│
├── data_sample/
│   └── 字段结构展示样例
│
├── docs/
│   └── data_dictionary/
│       └── 正式数据字典
│
└── powerbi/
    └── 三页 PBIP 报表与语义模型源码
```

完整原始数据与生成结果不提交至 Git：

```text
data/
output/
```

其中：

- `data/`：存放完整原始 CSV
- `output/`：由 Notebook Run All 后生成 Power BI 输入表
- `data_sample/`：仅用于展示字段结构，不能替代完整分析数据

---

# 如何复现

## 1. Clone 仓库

```bash
git clone https://github.com/CoursesGit/mexico-toy-sales-inventory-analysis.git
```

进入项目目录。

---

## 2. 准备完整原始数据

将四张完整原始 CSV 放入：

```text
data/
```

Notebook 使用完整数据进行分析。

`data_sample/` 仅用于字段结构展示，不能替代完整数据运行项目。

---

## 3. 运行 Notebook

打开：

```text
mexico_toy_sales_analysis.ipynb
```

执行：

```text
Run All
```

Notebook 将完成：

- 数据质量检查
- 经营指标计算
- 销售变化诊断
- ABC 分类
- 库存健康分类
- Receiver / Donor 分析
- Reallocation Scenario
- Sensitivity Analysis
- Inventory Exception 识别
- Power BI 输出表生成

---

## 4. 检查输出

运行完成后，确认：

```text
output/
```

目录已经生成：

```text
sales_fact.csv
products.csv
stores.csv
inventory_fact.csv
inventory_actions.csv
inventory_exceptions.csv
scenario_summary.csv
```

---

## 5. 打开 Power BI

打开：

```text
powerbi/mexico_toy_sales_inventory.pbip
```

---

## 6. 设置 DataRootPath

在 Power BI 中将参数：

```text
DataRootPath
```

设置为本机项目的：

```text
output/
```

目录。

公开 PBIP 使用示例占位路径：

```text
C:\path\to\03_Mexico_Toy_Sales\output\
```

不会包含项目作者的本机用户名或桌面路径。

设置完成后刷新模型，即可查看完整三页 Dashboard。

---

# 分析边界与限制

## 1. 需求口径

最近 90 天销量用于估算：

> **近期历史销售速度**

并不代表未来真实需求预测。

因此项目没有使用：

- 时间序列预测
- 机器学习需求预测
- 自动补货预测模型

---

## 2. 库存口径

`inventory` 数据是缺少明确快照日期的单时点库存状态。

因此无法分析：

- 历史库存趋势
- 持续缺货时间
- 实际补货周期
- 库存变化过程

Days Cover 和库存状态阈值均属于：

> **分析情景假设**

而不是企业正式安全库存政策。

---

## 3. 调拨口径

Reallocation 属于规则型情景模拟，不是：

- 实际调拨执行结果
- 最优物流方案
- 最优库存模型
- 数学规划优化结果

数据缺少：

- 运输距离
- 运输成本
- Lead Time
- 在途库存
- 采购记录
- 补货记录
- 门店最低展示库存
- 配送中心信息
- 真实物流网络约束

因此实际执行前仍需要结合真实供应链条件。

---

## 4. Same-City 口径

Same-City 仅使用：

```text
Store_City
```

作为简单地理约束。

它只能用于测试：

> 同一城市内部是否存在潜在 Same-SKU 库存重新配置空间。

不能直接解释为：

> 真实物流上一定能够完成该调拨。

---

## 5. Opportunity Exposure 口径

`Sales Opportunity Exposure` 和 `GP Opportunity Exposure` 用于衡量：

> 在当前分析规则下，库存风险对应的经营机会暴露规模。

它们不等于：

- 实际损失销售额
- 实际损失利润
- 已经发生的业务损失

---

## 6. Covered GP Opportunity 口径

`Covered GP Opportunity` 表示：

> 在当前调拨情景下理论被库存匹配覆盖的毛利润机会。

它不等于：

- 实际挽回利润
- 调拨实施后的利润提升
- 因果业务成果

---

## 7. Inventory Unknown

缺失 inventory record 的 Store-SKU 被标记为：

> **Inventory Unknown**

而不是：

> Stock = 0

因此这些记录：

- 不被直接计入 Active Stockout
- 不用于虚构库存量
- 单独进入 Inventory Exception 分析

---

# 核心结论

本项目最终得到四个主要结论：

1. **经营表现存在明显的结构性差异。**  
   全观察期销售额为 **14.44M**，毛利润为 **4.01M**，毛利率为 **27.79%**；2018 年 8 月出现观察期最大月度销售下降，环比 **-20.22%**。

2. **经营价值高度集中在少数商品。**  
   35 个商品中，**15 个 A 类商品贡献 79.65% 的历史毛利润**，因此库存风险处理不能只看缺货数量，还需要结合商品价值优先级。

3. **库存问题同时包含短缺与配置错位。**  
   当前识别出 **77 个 Active Stockout** 和 **289 个 Critical Store-SKU**，同时部分门店存在可释放库存，说明同一 SKU 在不同门店之间存在一定配置不均衡。

4. **内部调拨能够缓解风险，但无法替代补货。**  
   Base Scenario 下 **7,622 件风险需求中有 1,426 件可由现有跨门店库存理论匹配，需求覆盖率为 18.71%，毛利润机会覆盖率为 23.21%，剩余 6,196 件仍需进入补货优先级**。

因此，本项目的最终业务主线可以概括为：

> **经营诊断 → 商品优先级 → 库存风险识别 → 跨店库存错配分析 → 调拨候选识别 → 补货优先级判断 → 情景与敏感性验证**

---

# 项目定位

本项目定位为：

> **零售经营分析 + 库存配置决策支持**

而不是：

> 单纯的销售 Dashboard

也不是：

> 需求预测 / 供应链优化算法项目

项目重点在于将经营指标、商品价值、库存健康和跨门店库存配置连接起来，形成一条可以从：

> **发现问题**

继续推进到：

> **识别优先级**

再到：

> **支持调拨与补货决策**

的完整业务分析链路。
