# 墨西哥玩具零售经营与库存配置分析

Mexico Toy Retail Sales & Inventory Allocation Analysis

## 项目概览

本项目使用 Python、Pandas、Jupyter Notebook 和 Power BI/PBIP，将 Mexico Toy Sales 公开练习数据组织为一套从经营诊断到库存配置决策支持的可复现分析流程。项目不使用 SQL，也不将情景模拟包装为实际执行成果。

完整业务流程分为六个阶段：

1. 业务目标与数据基础
2. 销售与盈利表现诊断
3. 商品优先级
4. 库存健康与风险评估
5. 库存配置决策支持
6. 情景验证与业务建议

## 业务框架与分析方法

- KPI / OSM framework
- Monthly trend 与 MoM
- Contribution analysis
- ABC / Pareto 商品分级
- Inventory health classification
- Days of Cover
- Opportunity Exposure
- Same-SKU cross-store imbalance
- Reallocation scenario analysis
- Sensitivity analysis

## 已验证结果

### 经营表现

| 指标 | 结果 |
|---|---:|
| Total Sales | 14,444,572.35 |
| Total Units | 1,090,565 |
| Gross Profit | 4,014,029.00 |
| Gross Margin | 27.79% |
| 最大观察期月度销售下降 | 2018-08 vs 2018-07：-20.22% |

金额均按源数据提供的金额单位呈现；源文件没有可信的 currency 字段，因此本项目不作 MXN 或 USD 的确定性声明。

### ABC 商品优先级

| ABC | 商品数 | 历史毛利占比 |
|---|---:|---:|
| A | 15 | 79.65% |
| B | 9 | — |
| C | 11 | — |

### 库存健康

| 状态 | Store-SKU 数量 |
|---|---:|
| Active Stockout | 77 |
| Critical | 289 |
| Low Stock | 267 |
| Healthy | 661 |
| Overstock | 206 |
| Dormant | 93 |

Dormant inventory 包含 1,349 units，对应库存成本 16,656.51（源数据金额单位）。

### Base Scenario：配置决策支持

| 指标 | 结果 |
|---|---:|
| Receiver Store-SKU | 366 |
| Need | 7,622 |
| Matched Units | 1,426 |
| Unit Coverage | 18.71% |
| GP Opportunity Coverage | 23.21% |
| Remaining Need | 6,196 |

Same-City 情景匹配 342 units，Unit Coverage 为 4.49%，GP Opportunity Coverage 为 5.97%。动作分层包含 40 个 Same-City Candidate、74 个 Cross-City-only Candidate 和 252 个 Replenishment Priority。

### 数据完整性例外

- 157 个缺失 inventory records
- 其中 41 个组合存在历史销售
- 其中 23 个组合存在近期需求
- Inventory Unknown 不等于零库存，不参与零库存断言

## 最终分析输出

Notebook 运行后在 `output/` 生成七张 Power BI 输入表：

1. `sales_fact.csv`
2. `products.csv`
3. `stores.csv`
4. `inventory_fact.csv`
5. `inventory_actions.csv`
6. `inventory_exceptions.csv`
7. `scenario_summary.csv`

Power BI 项目包含三页 Dashboard：

1. `01 经营表现`
2. `02 库存风险`
3. `03 配置决策`

仓库暂不展示 Dashboard 截图：原有两张图片属于旧版两页结构，已从发布版本移除。三页正式截图可在后续从最终 PBIP 人工导出补充；当前 Power BI 源码完整保留。

## 项目结构

```text
mexico-toy-sales-inventory-analysis/
├── README.md
├── .gitignore
├── mexico_toy_sales_analysis.ipynb
├── data_sample/                 # 字段结构展示样例，不是完整分析数据
├── docs/
│   └── data_dictionary/         # 正式数据字典
└── powerbi/                     # 三页 PBIP 报表与语义模型源码
```

完整原始数据和生成结果不纳入 Git：

- `data/`：放置完整原始 CSV；Notebook 保持从该目录读取
- `output/`：由 Notebook 生成七张 CSV；其中 `sales_fact.csv` 含 829,262 行

## 复现步骤

1. Clone 本仓库。
2. 将四张完整原始 CSV 放入项目根目录的 `data/`。
3. 在项目根目录打开 `mexico_toy_sales_analysis.ipynb` 并执行 **Run All**。
4. 确认 `output/` 已生成上述七张 CSV。
5. 打开 `powerbi/mexico_toy_sales_inventory.pbip`。
6. 在 Power BI 中将参数 `DataRootPath` 修改一次，使其指向本机项目的 `output/` 目录，并保留末尾路径分隔符。
7. 刷新模型。

公开 PBIP 将 `DataRootPath` 设为示例占位值 `C:\path\to\03_Mexico_Toy_Sales\output\`，不会包含项目作者的用户名或桌面路径。PBIP/PBIR 不在此处伪造不可靠的相对路径；使用者需按第 6 步设置自己的路径。

`data_sample/` 仅展示字段结构，不能替代完整数据运行 Notebook 或刷新 Power BI。

## 分析边界

- Opportunity Exposure 是基于规则定义的机会暴露，不是实际销售损失。
- Covered GP Opportunity 是情景覆盖指标，不是实际挽回利润。
- Reallocation 结果是情景模拟，不是实际执行结果，也不是最优调拨算法。
- 90D demand 表示近期历史销售速度，不是需求预测。
- Days Cover 是分析指标，不是企业正式安全库存政策。
- Same-City 仅是简单地理代理，不代表真实运输网络。
- 数据不包含运输成本、距离、Lead Time、采购或补货记录。
- 库存表是缺少明确快照日期的单时点状态，不能据此分析库存历史趋势。
- 缺失 inventory record 的组合标记为 Inventory Unknown，不能视为零库存。
- 所有结论均为描述性分析或规则情景结果，不应解释为因果效果。

## 技术栈

- Python
- Pandas
- Jupyter Notebook
- Power BI / PBIP
