---
name: anti-seo-researcher
description: >
  反SEO反水军深度消费决策研究工具（V5 通用版）。当用户想要购买某个商品或做消费决策时，使用此 Skill。
  V5 核心：在V3品类自适应架构基础上，新增电商真实评论层+社交评论区层，补全最高价值数据源。
allowed-tools:
  - execute_command
  - read_file
  - write_to_file
  - web_search
  - web_fetch
---

# 反SEO深度消费决策研究员（V5 通用版）

> 详细规则、评分细则和品类示例参见 `references/SKILL_REFERENCE.md`

## 架构概要

**AI 品类自适应 → AI 多层搜索（论坛帖+电商评论+社交评论区） → 脚本评分 → AI 语义分析 → 动态多维评分 → 报告**

- **品类层**：AI 生成 `category_profile` JSON（评估维度/权重/痛点词/安全风险/平台权重/电商评论搜索策略）
- **搜索层**（三层数据源）：
  - **L1 电商评论层**（最高优先级）：间接搜索京东/淘宝/拼多多的追评、差评汇总、评论截图帖
  - **L2 社交评论区层**（次高优先级）：搜索小红书/知乎的"种草后拔草"评论区反馈
  - **L3 论坛帖层**（传统层）：AI 用 `web_search` + `site:` 定向搜索社区帖子
- **评分层**：`credibility_scorer.py`（正则粗筛+品类信号注入+数据源层级加权）→ `ai_credibility_analyzer.py`（灰色地带30-85分AI精判）
- **多维评分**：`brand_scorer.py`（维度/权重来自 profile，安全封顶品类自适应）
- **报告层**：`generate_report.py`（动态表头+数据源分布统计，来自 profile 维度定义）

## V5 核心升级：数据源分层策略

**问题**：V3及之前版本过度依赖搜索引擎可索引的"帖子型"内容（知乎回答、什么值得买文章、论坛帖子），而这些内容的软广比例高。电商平台（京东/淘宝/拼多多）的真实购买评论和社交平台（小红书/知乎）的评论区反馈虽然信息密度更高、软广成本更高，但因为是动态加载内容无法被搜索引擎直接索引，导致这些高价值数据源被系统性忽略。

**V5 解决方案**：通过间接搜索策略（搜索"评论搬运帖"、"追评汇总"、"差评合集"等二手内容）获取电商评论和评论区数据。

| 数据源层级 | 来源 | 核心价值 | 可信度基础权重 |
|-----------|------|---------|-------------|
| L1 电商追评 | 京东追评/淘宝追评/拼多多评论（间接） | 真金白银的购买者、长期使用追评 | 0.85 |
| L2 评论区 | 小红书评论区/知乎评论区（间接） | 种草→拔草链条上的真实反馈 | 0.75 |
| L3 论坛帖 | V2EX/Chiphell/NGA/贴吧等 | 发烧友深度体验、横向对比 | 沿用 platform_relevance |
| L4 独立帖 | 知乎回答/什么值得买文章/B站视频 | 系统性横评框架 | 沿用 platform_relevance |

**关键约束**：电商评论层和评论区层的搜索占比应不低于总搜索量的30%。

## 工作流（7步）

### 第1步：交互式需求确认与解析

**核心原则**：调研一旦启动就不可中断（耗时且消耗大量token），因此必须在启动前确认需求。宁可多问一次，不可调研完发现方向错了。

**强制确认项（缺失则必须向用户询问）：**
- **目标品类**：用户想买什么？——不允许假设品类
- **预算范围**：价格上下限是多少？——不允许使用默认预算
- **核心使用场景或痛点**：主要用途是什么？最在意什么？——不允许假设需求

**条件确认项（当品类存在以下情况时主动询问）：**
- 版本/渠道差异显著时（如奶粉国行vs海淘、手机国行vs水货）→ 确认版本偏好
- 品牌偏好明显时（用户提到了某品牌或某品牌大类）→ 确认是否限定品牌
- 用户需求含糊可能导致调研方向偏差时 → 用选择题形式确认

**确认方式**：使用简短的选择题或开放式问题，最多3个问题，让用户几个字就能回答。

**确认完毕后输出 `task_config`**（供后续步骤引用）：

```json
{
  "category": "品类名称",
  "budget_min": 2500,
  "budget_max": 3500,
  "core_scenario": "打游戏",
  "pain_points": ["散热", "帧率稳定"],
  "excluded_brands": [],
  "preferred_brands": [],
  "variant_preference": "国行",
  "special_requirements": []
}
```

**直接跳过确认的条件**（全部满足才可跳过）：
1. 品类明确（如"推荐一款3000元的游戏手机"）
2. 预算明确（有具体数字或明确范围）
3. 使用场景明确（如"打游戏"、"给宝宝喝"）

### 第1.5步：AI 品类自适应分析

**在搜索前**必须生成 `category_profile` JSON，严格遵循以下 Schema：

```json
{
  "category": "<品类名>",
  "category_type": "<food|durable_goods|electronics|personal_care|service|other>",
  "evaluation_dimensions": [
    {"name":"维度名","weight":0.25,"description":"描述","key_parameters":["参数1"],"data_sources":["来源"]}
  ],
  "pain_point_keywords": {"safety":[],"quality":[],"experience":[],"trust":[]},
  "safety_risk_types": {"critical":[],"high":[],"medium":[],"low":[]},
  "platform_relevance": {"v2ex":0.9,"chiphell":0.9,"nga":0.85,"smzdm":0.65,"zhihu":0.55,"tieba":0.75,"xiaohongshu":0.3,"douban":0.8},
  "category_positive_signals": [{"pattern_description":"描述","regex_hint":"正则","score":15,"label":"标签"}],
  "has_variant_issue": false,
  "variant_types": [],
  "variant_search_keywords": [],
  "non_commercial_indicators": [],
  "commercial_bias_sources": [],
  "ecommerce_search_strategy": {
    "enabled": true,
    "primary_platforms": ["京东", "淘宝"],
    "search_templates": {
      "review_aggregation": ["[商品] 京东评论 追评 真实", "[商品] 淘宝评价 追评 差评"],
      "negative_reviews": ["[商品] 京东差评 一星 退货", "[商品] 淘宝差评 避坑 后悔"],
      "long_term_reviews": ["[商品] 追评 半年 一年 使用感受", "[商品] 京东追评 长期使用"]
    },
    "high_value_indicators": ["追评", "用了几个月后", "补充评价", "再来更新"],
    "low_value_indicators": ["默认好评", "好评返现", "此用户未填写评价"]
  },
  "comment_section_strategy": {
    "enabled": true,
    "primary_platforms": ["小红书", "知乎"],
    "search_templates": {
      "debunk_feedback": ["[商品] 小红书 评论区 真实", "[商品] 种草 拔草 后悔"],
      "experience_sharing": ["[商品] 买了 翻车 评论", "[商品] 知乎 评论区 实际体验"]
    },
    "high_value_indicators": ["我也买了", "同款翻车", "用了之后发现", "评论区才是真相"],
    "low_value_indicators": ["求链接", "已入手", "好种草"]
  }
}
```

**约束**：维度3-6个，单维度权重≤0.4，权重之和=1.0。平台权重根据品类动态调整。`ecommerce_search_strategy` 和 `comment_section_strategy` 的 `search_templates` 中的 `[商品]` 会在搜索时被替换为具体商品型号。

### 第2步：多层数据源搜索（含电商评论层+评论区层+结果自适应）

**V5 搜索三层架构**：按数据源层级从高到低依次搜索，确保高价值数据源优先覆盖。

#### 第2a层：电商评论间接搜索（L1，占比≥15%）

电商平台的评论区是动态加载内容，搜索引擎无法直接索引。通过以下间接策略获取：

**策略1：评论搬运/汇总帖搜索**
```
web_search("[商品型号] 京东评论 追评 真实 差评")
web_search("[商品型号] 淘宝评价 真实评价 买家秀")
web_search("[商品型号] 拼多多评价 差评 退货")
```

**策略2：追评专项搜索（高价值）**
```
web_search("[商品型号] 追评 用了半年 一年 长期")
web_search("[商品型号] 京东追评 后悔 问题 坏了")
```

**策略3：电商差评汇总搜索**
```
web_search("[商品型号] 一星差评 退货原因 避坑")
web_search("[商品型号] 差评 最多的问题 真实反馈")
```

搜索关键词模板从 `category_profile.ecommerce_search_strategy.search_templates` 读取。
搜到的电商评论来源内容标注 `source_layer: "L1_ecommerce"`, `base_weight: 0.85`。

#### 第2b层：社交评论区间接搜索（L2，占比≥15%）

小红书/知乎等平台的帖子本身可能是软广，但评论区经常出现真实的"拔草"反馈。

**策略1：种草→拔草链搜索**
```
web_search("[商品型号] 小红书 评论区 真实 翻车")
web_search("[商品型号] 种草 买了 后悔 拔草")
```

**策略2：知乎评论区反馈搜索**
```
web_search("[商品型号] 知乎 评论区 真实体验 反驳")
web_search("[商品型号] 买了才知道 实际体验 和博主说的不一样")
```

搜索关键词模板从 `category_profile.comment_section_strategy.search_templates` 读取。
搜到的评论区来源内容标注 `source_layer: "L2_comment_section"`, `base_weight: 0.75`。

#### 第2c层：论坛帖+独立帖搜索（L3/L4，传统层，占比≤70%）

使用 `web_search` + `site:` 定向搜索。**搜索均衡策略：40%中性 + 20%正面 + 40%负面**。

**双时间窗**：每组搜索拼接当前年份（即时窗口）+ 不限年份（历史窗口）。

**平台优先级**：按 `platform_relevance` 权重排序，权重<0.2的跳过。

**搜索关键词构成**：
- 中性：`[商品] [品类] 评测 对比 [key_parameters]`
- 正面：`[商品] [品类] 长期使用 满意 推荐`
- 负面：`[商品] [品类] [pain_point_keywords.quality/experience/trust]`

**V4 搜索结果自适应机制**：

搜索完成后，评估结果充分度（或使用脚本 `--adaptive` 参数）：

| 充分度 | 条件 | 处理策略 |
|--------|------|----------|
| 充分 | 总数≥30 且每商品≥5条 | 正常进入下一步 |
| 基本充分 | 总数≥15 且不足商品≤1 | 对不足商品补搜一轮 |
| 不足 | 总数<15 或多商品不足 | 去掉site:限制全网搜、扩大时间窗、增加平台 |
| 严重不足 | 总数<8 | 冷门品类模式：降低评分门槛，报告中注明数据量不足 |

过多时（>200条）：按可信度排序裁剪，每商品最多保留40条。

```bash
python scripts/platform_search.py "[查询]" --adaptive \
    --candidate-products "商品A,商品B,商品C" \
    --category-profile category_profile.json
```

### 第3步：内容抓取与分析

`web_fetch` 抓取有价值的帖子（包含使用时长、多人讨论、无营销词标题的帖子）。

### 第3.5步：品类参数结构化提取

根据 `evaluation_dimensions[].key_parameters` 从内容中提取各候选商品的结构化参数表。

### 第4步：动态深挖（含电商追评深挖）

对高频型号执行负面长尾搜索，关键词来自 `pain_point_keywords`。

**V5 新增：电商追评深挖**——对每个候选型号额外执行电商追评专项搜索：

```
web_search("[候选型号] 京东 追评 差评 一年后 质量")
web_search("[候选型号] 淘宝 追评 后悔 退货 售后")
web_search("[候选型号] 电商评价 差评合集 真实反馈")
```

```bash
python scripts/deep_dive_search.py --auto-extract results.json --days 730 \
    --category-profile category_profile.json \
    --ecommerce-dive
```

### 第4.5步：安全事件搜索

**对每个候选品牌**全网搜索安全事件。双层分级：通用层（召回/致死/下架）+ 品类层（`safety_risk_types`）。

```bash
python scripts/deep_dive_search.py "[品牌]" --days 365 \
    --category-profile category_profile.json
```

### 第5步：可信度评估

```bash
python scripts/credibility_scorer.py results.json --v2 --output scored.json --threshold 40 \
    --category-profile category_profile.json
```

**评分架构**：正则粗筛 → 品类信号注入(profile) → AI语义分析(灰色地带30-85分) → 加权融合

### 第5.5步：证据冲突仲裁

当同一商品在不同来源出现矛盾评价时：

```bash
python scripts/conflict_resolver.py scored.json \
    --category-profile category_profile.json \
    --output conflicts.json
```

仲裁规则：非商业来源 > 商业来源，长期反馈 > 短期反馈，高可信度 > 低可信度。

### 第6步：多维评分

```bash
python scripts/brand_scorer.py scored.json \
    --category-profile category_profile.json \
    --safety-results safety.json \
    --output scores.json
```

| 综合分 | 结论 |
|--------|------|
| ≥70 | 推荐 |
| 55-69 | 有条件推荐 |
| 40-54 | 谨慎选择 |
| <40 | 避坑 |

安全封顶：food(阈值30) > personal_care(25) > electronics(20) > durable_goods(15)

### 第7步：报告生成

```bash
python scripts/generate_report.py scored.json \
    --query "[品类]" --budget [预算] --pain-point "[痛点]" \
    --category-profile category_profile.json \
    --brand-scores scores.json \
    --output report.md
```

## 关键规则（不可跳过）

1. **先确认需求再搜索**——预算、品类、使用场景三者缺一不可，缺失则必须先问用户
2. **不假设用户需求**——用户说"推荐手机"但没说预算和用途时，不要自行假设，先问
3. **先生成 category_profile 再搜索**——它是后续所有步骤的基石
4. **搜两轮**——广撒网后，必须对高频型号做负面定向深挖
5. **搜索必须拼年份**——拼接当前年份+上一年
6. **负面占比≥40%**——主动搜负面，搜不到说明深度不够
7. **安全事件必须搜**——Top推荐品牌必须过安全事件筛查
8. **搜索结果自适应**——结果不足时扩大搜索范围，过多时按可信度裁剪
9. **知乎万赞要警惕**——高赞≠真实
10. **长期反馈优先**——"用了两年"比"刚买很好"有价值十倍
11. **交叉验证是核心**——商业评测说好+小众论坛抱怨→信后者
12. **profile是唯一品类知识源**——脚本硬编码仅作fallback
13. **【V5】电商评论层不可跳过**——每次调研必须执行电商追评间接搜索（L1层），占总搜索量≥15%
14. **【V5】评论区层不可跳过**——每次调研必须执行社交评论区搜索（L2层），占总搜索量≥15%
15. **【V5】追评>开箱**——电商追评（3个月+）的参考价值远高于开箱评测
16. **【V5】评论区>帖子正文**——种草帖的评论区真实反馈优先于帖子正文结论
