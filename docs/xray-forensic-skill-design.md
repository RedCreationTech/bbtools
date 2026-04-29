# XRay Forensic Skill 设计文档

## 1. 背景

当前仓库里已经有两段可复用能力：

1. `xray/`
   负责对指定 Git 仓库和时间窗口生成离线 HTML 报告，以及配套的 `data.json`、`meta.json`。
2. 历史上存在一套独立的模板目录
   曾负责基于 `data.json` 生成 4 份 markdown 分析报告模板；现已由 `xray-skills/xray-forensic-report/` 取代。

现状问题不在于分析引擎缺失，而在于能力是分散的：

- `xray` 是 CLI，不是 skill
- 模板目录更像“示例工程”，还不是可直接复用的 skill 资产
- 4 份模板当前带有明显的 `MaterialPlat` 示例痕迹，泛化不足

本次目标是把“生成 HTML 报告 + 生成 4 份分析报告”收敛成一个可复用 skill。

## 2. 目标与非目标

### 2.1 目标

为指定目录下的 Git 项目，在指定时间段内完成一条完整流水线：

1. 生成离线 HTML 报告
2. 产出结构化 `data.json`
3. 基于同一份数据生成 4 份模板化分析报告
4. 让 Codex 可以通过一个 skill 直接触发整条流程

### 2.2 非目标

- 不重写 `xray` 指标计算逻辑
- 不解析 HTML 页面反向提取数据
- 本期不做 PDF 导出、在线服务、消息推送
- 本期不追求“完全自动写完所有洞察”，保留人工判断插槽

## 3. 设计原则

### 3.1 复用现有 `xray`，不复制分析引擎

HTML 报告和 `data.json` 的生成能力已经在 `xray/src/xray/cli.clj` 里闭环，skill 应该只负责参数约束、调用编排和结果组织。

### 3.2 报告生成基于 `data.json`，不是基于 HTML DOM

用户描述是“基于这个报告”，但技术上应以 HTML 报告输出目录中的 `data.json` 作为统一数据契约：

- HTML 只是面向人的可视化投影
- `data.json` 才是稳定 API
- 模板脚本基于 `data.json` 可测试、可复用、可扩展

### 3.3 skill 保持轻，复杂逻辑下沉到脚本

`SKILL.md` 只描述触发条件、参数约定和执行流程。
参数解析、路径校验、命令调用、模板填充等具体工作放进脚本，避免 skill 文本过重。

### 3.4 模板必须去示例化

当前 4 个模板更像“MaterialPlat 样例报告”，不是通用模板。提炼 skill 时必须把硬编码业务名、路径、结论、人物替换为：

- 占位变量
- 自动填充段落
- 明确的人工补充标记

## 4. 推荐方案

推荐新增一个 repo 内 skill，暂定名：`xray-forensic-report`。

原因：

- 不直接改动现有全局 skill `xray-local-report`，避免影响已有使用方式
- 新 skill 的职责更完整，不只是“产出 HTML”，而是“产出完整法医分析包”
- 仓库内先落地，验证后再决定是否安装到全局 skills

## 5. skill 对外能力

### 5.1 输入

必填：

- `repo`: 本地 Git 仓库绝对路径
- `since`: 开始日期，格式 `YYYY-MM-DD`
- `until`: 结束日期，格式 `YYYY-MM-DD`

选填：

- `path`: 仓库子目录
- `branch`: 分支名
- `config`: xray 配置文件路径
- `out`: 输出目录
- `topN`: 报告默认 TopN
- `include-raw`: 是否保留 raw commits，默认 `true`
- `repo-name`: 报告展示名
- `ai-analysis`: 是否启用 AI 模式附加分析

### 5.2 输出

推荐输出目录：

```text
<repo>/target/xray-forensic-report-<timestamp>/
├── index.html
├── data.json
├── meta.json
├── assets/
└── reports/
    ├── report-forensic-analysis.md
    ├── report-management-summary.md
    ├── report-technical-plan.md
    └── report-refactoring-guide.md
```

其中：

- `index.html` 是可直接打开的离线可视化报告
- `data.json` 是模板生成和后续扩展的统一输入
- `reports/` 是 4 份最终 markdown 报告

## 6. skill 目录结构

推荐在仓库内新增：

```text
xray-skills/xray-forensic-report/
├── SKILL.md
├── agents/
│   └── openai.yaml
├── scripts/
│   ├── run_forensic_pipeline.py
│   └── fill_templates.py
├── assets/
│   └── templates/
│       ├── template-forensic-analysis.md
│       ├── template-management-summary.md
│       ├── template-technical-plan.md
│       └── template-refactoring-guide.md
└── references/
    └── data-schema.md
```

说明：

- `run_forensic_pipeline.py` 负责编排整条流程
- `fill_templates.py` 负责把 `data.json` 转成 4 份 markdown
- `assets/templates/` 放泛化后的模板
- `references/data-schema.md` 只保留模板依赖的数据字段说明

## 7. 执行流程

### 7.1 skill 触发流程

1. 校验 `repo` 是否为绝对路径且是 Git 仓库
2. 解析时间范围和可选参数
3. 调用 `bb` 执行 `xray report`
4. 在输出目录下找到 `data.json`
5. 调用模板填充脚本生成 4 份 markdown 报告
6. 返回输出目录、HTML 路径、4 份报告路径

### 7.2 底层命令建议

skill 不直接手写复杂 `bb` 命令，统一走 Python 包装脚本。

包装脚本内部执行的核心命令保持为：

```bash
bb --config xray/bb.edn report \
  --repo <repo> \
  --since <since> \
  --until <until> \
  --out <out> \
  ...
```

## 8. 模板策略

### 8.1 当前模板存在的问题

旧版模板目录中的 4 个模板曾有这些问题：

- 标题和正文绑定到 `MaterialPlat`
- 结论段落写死了具体文件、作者和数值
- 不是“模板”，而是“带样例数据的成品文档”

如果直接作为 skill 资产使用，会导致换一个仓库后内容失真。

### 8.2 模板改造方向

每份模板分为三类内容：

1. 自动填充区
   适合直接从 `data.json` 提取的字段和表格
2. 半自动建议区
   由脚本给出候选结论、风险排行、重点对象
3. 人工判断区
   明确保留 `TODO` 或 `待补充` 标记

### 8.3 四类模板的职责

1. `forensic-analysis`
   面向 TL/架构师的完整演化分析
2. `management-summary`
   面向管理层的摘要和风险矩阵
3. `technical-plan`
   面向技术整改的计划和优先级
4. `refactoring-guide`
   面向开发执行的任务拆解手册

### 8.4 模板填充层级

推荐分两层：

- 第一层：硬数据自动填充
- 第二层：根据排名生成通用建议句式

例如：

- “最高风险文件”为 `risk[0]`
- “最高耦合配对”为 `coupling_pairs[0]`
- “核心作者”来自 `raw.commits` 聚合
- “建议优先拆分的文件”来自高风险且高复杂度交集

## 9. 脚本职责拆分

### 9.1 `run_forensic_pipeline.py`

职责：

- 参数校验
- 解析默认输出目录
- 调用 `xray report`
- 调用 `fill_templates.py`
- 汇总结果路径

不负责：

- 指标计算
- markdown 模板内容拼装细节

### 9.2 `fill_templates.py`

职责：

- 读取 `data.json`
- 计算通用变量和派生指标
- 填充 4 份模板
- 输出到 `reports/`

本次建议顺手升级当前脚本：

- `REPO_NAME` 默认从 `repo.root` 推导，而不是固定 `Your Repo`
- `ANALYSIS_PATH` 缺省显示为 `.`，而不是写死 `src`
- 对无 `raw` 的情况做降级处理
- 支持更多风险、耦合、ownership、staleness 变量
- 将输出目录统一为 `reports/`

## 10. 与现有全局 skill 的关系

当前全局已有 `xray-local-report`，它只负责：

- 给指定本地仓库生成 HTML 报告

本设计建议：

- 保留 `xray-local-report` 不动
- 在仓库里新增 `xray-forensic-report`
- 后续如果验证稳定，再考虑把两者合并成：
  - `xray-local-report`: 只做 HTML
  - `xray-forensic-report`: HTML + 4 份分析报告

这样职责更清晰，也减少兼容性风险。

## 11. 实施步骤

### Phase 1: skill 骨架

- 新建 `xray-skills/xray-forensic-report/SKILL.md`
- 新建 `agents/openai.yaml`
- 新建 `scripts/run_forensic_pipeline.py`

### Phase 2: 模板资产收拢

- 将现有 4 个模板迁移到 `assets/templates/`
- 去掉 `MaterialPlat` 等样例硬编码
- 明确自动区、半自动区、人工区

### Phase 3: 模板脚本升级

- 升级 `fill_templates.py`
- 补充变量映射和派生指标
- 统一输出目录结构

### Phase 4: 验证

- 选择一个真实仓库做 smoke test
- 验证 HTML 可打开
- 验证 4 份 markdown 可生成
- 验证无 `raw`、无 `path`、空数据窗口等边界情况

## 12. 风险与注意点

### 12.1 最大风险

不是 skill 框架本身，而是模板泛化不足。

如果模板不去示例化，skill 只会稳定地产出“错误但看起来像真的”报告。

### 12.2 数据口径风险

部分 narrative 依赖 `raw.commits`。如果用户关闭 `--include-raw`，则：

- 作者分布
- 提交计数
- 某些 AI 模式判断

需要降级或跳过。

### 12.3 叙事自动化边界

本期建议把“事实填充”和“结构化建议”自动化，而不是强行自动写满全部结论。
否则容易出现听起来流畅但证据不足的段落。

## 13. 推荐结论

推荐按下面的路线开工：

1. 新增 repo 内 skill `xray-forensic-report`
2. 复用现有 `xray` CLI，不动分析引擎
3. 以 `data.json` 为统一数据接口
4. 改造现有 4 个模板为真正可泛化的模板
5. 用一个编排脚本串起 HTML 报告与 markdown 报告生成

这个方案改动面集中、复用率高、验证成本低，也是当前仓库最稳妥的演进路径。
