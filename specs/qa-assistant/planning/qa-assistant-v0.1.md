# QA Assistant v0.1 MVP Implementation

## 目标

`QA Assistant v0.1` 的目标是让测试同学选择一个代码仓库，上传一份需求文档，点击生成后获得一批可编辑、可导出、可追溯的结构化测试用例。

MVP 只验证三件事：

- 能否读懂需求文档中的功能点、规则、异常、边界和角色。
- 能否在代码仓库中定位和需求相关的模块、文件和实现线索。
- 生成的测试用例是否足够具体，测试同学愿意在此基础上修改并使用。

不在 v0.1 中追求完整测试管理平台、多人协作、权限系统、自动执行测试或复杂知识库。

## 用户流程

```text
选择本地或远程仓库
  -> 上传需求文档
  -> 解析需求并选择本次生成范围
  -> 分析仓库结构和相关代码
  -> 生成结构化测试用例
  -> 人工编辑、删除、补充
  -> 导出 xlsx/csv
  -> 保存历史生成记录
```

## MVP 范围

包含：

- 本地仓库选择。
- 远程仓库克隆或拉取到本地工作目录。
- 需求文档上传，支持 `md`、`docx`、`pdf`、`txt`。
- 文本型 PDF 解析。
- AI 自动分析需求和代码。
- 结构化测试用例表格生成。
- 用例人工编辑、删除、补充。
- `xlsx` 和 `csv` 导出。
- 历史生成记录保存。
- 每批用例的质量报告。

暂不包含：

- Jira、禅道、TestRail 同步。
- 多人协作。
- 权限系统。
- 自动执行测试。
- 全仓库深度向量索引。
- 复杂知识库管理。
- 二进制扫描型 PDF OCR。

## 技术形态

优先采用 Electron 桌面端，因为主要用户是公司内测试同学，桌面端可以直接访问本地仓库和本地文档，降低上传、安全合规和权限申请成本。

```text
Electron / Web 客户端
  -> Node.js 本地后端
  -> OpenCode runtime / server
  -> QA Testcase Generator Agent
  -> 自定义工具
  -> 仓库代码 / 需求文档 / SQLite / Excel 导出
```

v0.1 的工程推进顺序应当是：

1. 先做样例、schema 和质量标准。
2. 再做命令行原型验证生成质量。
3. 然后接入 OpenCode runtime 和专用 QA Agent。
4. 最后做 Electron 客户端。

## 二开与官方同步策略

这个项目属于基于 OpenCode 的二次开发。长期维护目标不是“改出一个独立分叉”，而是在尽量少修改官方源码的前提下，增加 QA Assistant 能力。这样后续同步官方 `dev` 分支时，冲突范围可控，合并成本可预估。

核心原则：

- 官方源码尽量保持原样，二开能力优先放在独立 package、插件、配置、Agent 和工具中。
- 必须修改官方代码时，只改稳定扩展点或适配层，不在业务代码中散落产品逻辑。
- 文档、样例、prompt、schema、导出模板和桌面端页面要和官方文档隔离。
- 每个侵入式改动都要能说明原因、影响范围、回退方式和是否可上游化。
- 同步官方源码要变成固定流程，而不是等冲突积累后一次性处理。

### 仓库和分支策略

建议保留两个远程：

```bash
git remote add upstream <official-opencode-repo>
git remote add origin <internal-fork-repo>
```

分支约定：

```text
upstream/dev
  官方默认分支，只用于拉取和对比

dev
  内部分叉的集成分支，定期同步 upstream/dev

qa/v0.1
  QA Assistant MVP 开发分支

qa/sync-YYYYMMDD
  每次同步官方源码的临时合并分支
```

同步节奏：

- 高频开发期每周同步一次 `upstream/dev`。
- 发布前必须同步一次 `upstream/dev` 并完成回归验证。
- 官方有安全修复、runtime、plugin、session、desktop 相关重大变更时立即同步。

推荐同步流程：

```bash
git fetch upstream
git checkout dev
git pull origin dev
git checkout -b qa/sync-YYYYMMDD
git merge upstream/dev
```

冲突解决后：

```bash
bun typecheck
# 从对应 package 目录运行必要测试，不能从 repo root 跑测试
git diff --check
```

确认通过后再合回内部 `dev`。如果官方历史较线性且团队熟悉 rebase，也可以对短生命周期 feature 分支使用 rebase；长期集成分支优先 merge，保留同步节点，方便追踪冲突来源。

### 文档隔离

二开文档不要混入官方已有说明，也不要改写官方 README 来承载内部产品说明。

建议目录：

```text
specs/qa-assistant/
  planning/
    qa-assistant-v0.1.md
  architecture/
    architecture.md
  operations/
    sync-upstream.md
    release-checklist.md
  standards/
    testcase-field-spec.md
    quality-rubric.md
  prompts/
    prompt-rules.md
examples/qa-assistant/
  requirements/
  expected-cases/
```

文档规则：

- 官方文档只在必要处增加短链接，不直接嵌入大段二开方案。
- 内部方案、样例、质量标准和排期放在 `specs/qa-assistant/` 的分类子目录下。
- 每次官方同步后，在 `operations/sync-upstream.md` 记录官方 commit、冲突文件、解决方式和后续风险。
- prompt、schema、Excel 模板和灰度反馈都视为二开资产，避免放入官方通用目录。
- 若未来准备向官方贡献通用能力，再单独抽取为小 PR，不把内部业务文档一并提交。

### 代码架构隔离

优先级从高到低：

1. 独立 package。
2. OpenCode 插件或 Agent 工具。
3. 通过已有 server/runtime 扩展点注册。
4. 很薄的 adapter 接入层。
5. 修改官方核心代码。

建议结构：

```text
packages/qa-assistant/
  src/
    agent/
    cli/
    desktop/
    export/
    requirement/
    repo/
    schema/
    storage/
  README.md
  package.json

packages/opencode/
  src/
    plugin/
      qa-assistant-adapter.ts   # 仅在确实需要接入官方 runtime 时新增薄适配层
```

架构边界：

- `packages/qa-assistant` 拥有需求解析、schema、测试用例生成、导出、历史记录和 QA Agent prompt。
- OpenCode 官方 package 只提供 runtime、tool registry、session、model 调用、文件读取等基础能力。
- 桌面端 UI 如果必须进入现有 Electron app，应按页面或路由集中接入，不改散落组件。
- 所有 QA 专属配置使用独立 namespace，例如 `qa_assistant`，避免污染官方配置结构。
- 数据库表使用独立前缀或独立 SQLite 文件，避免和官方 session、event、project 表耦合。

依赖方向：

```text
qa-assistant -> opencode public API / plugin API / server API
opencode core     -X-> qa-assistant business logic
```

如果发现必须让 OpenCode core 反向依赖 QA 模块，说明扩展点设计不够好，应优先增加一个通用扩展点，而不是直接引入 QA 业务代码。

### 最小侵入接入点

允许的官方代码改动类型：

- 注册一个插件入口。
- 暴露缺失但通用的 runtime API。
- 增加桌面端路由挂载点。
- 增加通用配置读取能力。
- 增加通用工具注册或权限声明能力。

不建议的改动类型：

- 在 session runner 中硬编码 QA 流程。
- 在通用文件读取工具里加入测试用例业务规则。
- 修改官方消息结构来适配 QA 输出。
- 在官方 README 或通用文档中写内部业务说明。
- 直接改官方已有 UI 组件满足单一 QA 页面需求。

每个官方代码改动都需要在提交说明或变更记录里标记：

```text
Why: 为什么必须改官方代码
Scope: 影响哪些 package 和 API
Fallback: 如何关闭或回退
Upstreamable: 是否可能拆成官方 PR
```

### 配置和功能开关

QA 功能默认应可关闭。

建议配置：

```json
{
  "qa_assistant": {
    "enabled": true,
    "storage_path": "~/.opencode/qa-assistant",
    "max_code_files": 80,
    "max_file_bytes": 200000,
    "default_depth": "standard"
  }
}
```

要求：

- 不启用 QA 功能时，官方 OpenCode 行为应保持一致。
- 配置读取失败不能影响 OpenCode 主流程启动。
- Agent 和工具权限独立声明。
- 远程仓库 clone、文档读取和导出路径必须有明确用户动作触发。

### 合并冲突控制

重点降低三类冲突：

- 文档冲突：通过独立目录解决。
- UI 冲突：通过独立路由、独立页面和少量导航挂载解决。
- runtime 冲突：通过插件和 adapter 解决。

冲突高风险区域：

```text
packages/opencode/src/session/
packages/opencode/src/server/
packages/opencode/src/plugin/
packages/desktop/
packages/app/
README*.md
```

控制措施：

- 不在高风险目录里放 QA 业务逻辑。
- 对必须修改的文件建立“补丁清单”，同步前先阅读官方变化。
- 保持 adapter 文件短小，避免和官方大文件频繁同改。
- 对官方核心文件的修改尽量集中在少数 commit 中，方便 cherry-pick、revert 和复盘。
- 同步前运行 `git diff upstream/dev...dev --stat` 查看二开补丁面是否扩大。

### 上游同步检查清单

每次同步官方源码时执行：

- 拉取 `upstream/dev`。
- 创建 `qa/sync-YYYYMMDD` 临时分支。
- 合并官方代码并解决冲突。
- 检查 `packages/qa-assistant` 是否仍能编译。
- 检查 OpenCode 主流程是否不受 QA 功能影响。
- 从 package 目录运行 `bun typecheck`。
- 运行 CLI 冒烟用例：示例需求文档 + 示例仓库 + 导出 xlsx/csv。
- 打开桌面端检查项目页、需求页、生成页、用例页是否可进入。
- 更新 `specs/qa-assistant/operations/sync-upstream.md`。
- 把冲突原因沉淀为架构调整事项，减少下一次同类冲突。

同步记录模板：

```md
# Sync upstream YYYY-MM-DD

- Upstream base: <commit>
- Internal base: <commit>
- Merge branch: qa/sync-YYYYMMDD
- Conflict files:
  - path: reason / resolution
- Verification:
  - bun typecheck: pass/fail
  - cli smoke: pass/fail
  - desktop smoke: pass/fail
- Follow-ups:
  - ...
```

### 可上游化策略

如果二开过程中发现某些能力具有通用价值，应拆成小而独立的上游贡献：

- 通用工具注册能力。
- 通用 Agent 配置能力。
- 通用桌面端路由扩展点。
- 通用文件导出能力。
- 通用文档解析接口。

上游 PR 不包含公司内测试流程、私有 prompt、样例文档或业务字段。这样既能减少长期维护补丁，也不会把内部产品逻辑推给官方项目。

## 标准输出 Schema

所有生成结果必须先通过 schema 校验，再进入表格编辑和导出。

```ts
type TestCase = {
  id: string
  module: string
  title: string
  type: "functional" | "boundary" | "exception" | "permission" | "integration" | "regression"
  priority: "P0" | "P1" | "P2" | "P3"
  preconditions: string[]
  testData: string[]
  steps: string[]
  expectedResults: string[]
  requirementRefs: string[]
  codeRefs: {
    file: string
    reason: string
  }[]
  riskPoint: string
  confidence: number
}
```

建议同时维护 `testcase.schema.json`，由 CLI、Agent 工具、桌面客户端和导出模块共同使用。

字段规范：

- `id`：批次内唯一，建议格式为 `TC-001`。
- `module`：产品或代码模块名称，避免填泛化的“系统”“功能模块”。
- `title`：可直接放入测试管理工具的用例标题。
- `type`：测试类型，必须从枚举中选择。
- `priority`：业务影响和风险优先级，`P0` 最高。
- `preconditions`：执行前置条件，不能写成步骤。
- `testData`：测试数据、账号、配置、输入样例。
- `steps`：可执行操作步骤，要求清晰、按顺序。
- `expectedResults`：每个关键步骤或最终结果的预期。
- `requirementRefs`：需求原文片段或章节编号，必须可追溯。
- `codeRefs`：相关代码文件和引用原因，不允许编造不存在的路径。
- `riskPoint`：该用例覆盖的业务风险、技术风险或遗漏风险。
- `confidence`：0 到 1 的置信度，低于 0.6 时需要在质量报告中标记。

## 数据模型

SQLite 保存项目、需求文档、生成批次、中间结果和测试用例。

```text
project
  id
  name
  repo_type              local | remote
  repo_path
  remote_url
  branch
  last_analyzed_at
  created_at
  updated_at

requirement_document
  id
  project_id
  file_name
  file_type
  file_path
  content_hash
  parsed_text_path
  parsed_structure_json
  created_at

generation_run
  id
  project_id
  requirement_document_id
  status                  pending | running | succeeded | failed
  depth                   quick | standard | deep
  selected_types_json
  selected_requirement_refs_json
  requirement_analysis_json
  code_analysis_json
  strategy_json
  review_report_json
  error_message
  started_at
  finished_at
  created_at

test_case
  id
  generation_run_id
  case_no
  module
  title
  type
  priority
  preconditions_json
  test_data_json
  steps_json
  expected_results_json
  requirement_refs_json
  code_refs_json
  risk_point
  confidence
  is_deleted
  created_at
  updated_at
```

历史记录以 `generation_run` 为单位查看。测试同学编辑过的用例直接更新 `test_case`，原始 AI 输出可以保存在 run 目录中用于排查。

## 文件与目录建议

具体目录可随目标仓库结构调整，但 v0.1 建议保持边界清楚：

```text
qa-assistant/
  cli/
    qa-gen.ts
  src/
    schema/
      testcase.schema.json
    requirement/
      parse.ts
      summarize.ts
    repo/
      clone.ts
      tree.ts
      search.ts
      read.ts
    agent/
      prompt.ts
      pipeline.ts
      tools.ts
    export/
      csv.ts
      xlsx.ts
    storage/
      db.ts
      migrations/
    desktop/
      pages/
        project/
        requirement/
        generate/
        cases/
  examples/
    requirements/
    expected-cases/
  docs/
    testcase-field-spec.md
    quality-rubric.md
```

如果直接集成进 OpenCode 现有仓库，应优先遵守现有 package 拆分，避免为了 MVP 新增过深抽象。

作为二开项目，优先新增独立的 `packages/qa-assistant` 或同等隔离目录。只有 runtime 注册、桌面入口、配置挂载这类必要接入点可以进入官方既有 package，并且要保持适配层薄、集中、可回退。

## 第 1 步：样例、Schema 和质量标准

目标：在写 UI 之前建立“什么是好用例”的标准。

输入：

- 3 到 5 份真实需求文档。
- 2 到 3 个目标仓库模块。
- 20 到 50 条人工整理的理想测试用例。

交付物：

- `testcase.schema.json`。
- `测试用例字段规范`。
- `优秀用例样例集`。
- `生成质量评估标准`。

任务清单：

- 收集真实需求文档，并脱敏。
- 按模块挑选相关代码目录。
- 人工编写理想测试用例。
- 标注每条用例引用的需求片段和代码文件。
- 总结好用例规则，例如标题具体、步骤可执行、预期可验证。
- 编写 JSON Schema，并准备通过和失败样例。

验收标准：

- 每条样例用例都能映射到需求原文。
- 至少 60% 样例用例能映射到一个或多个代码文件。
- schema 能拦截缺字段、错误枚举、错误数组类型和非法置信度。
- 质量评估标准能被测试同学理解并用于打分。

## 第 2 步：命令行原型

目标：先验证生成质量，不投入复杂 UI。

命令：

```bash
qa-gen \
  --repo /path/to/repo \
  --requirement ./需求文档.docx \
  --output ./testcases.xlsx
```

可选参数：

```bash
qa-gen \
  --repo /path/to/repo \
  --requirement ./需求文档.docx \
  --output ./testcases.xlsx \
  --types functional,boundary,exception \
  --depth standard \
  --save-run ./runs/run-2026-06-04
```

内部流程：

1. 解析需求文档为纯文本和章节结构。
2. 生成需求摘要、功能点、业务规则、验收点、异常和边界。
3. 扫描仓库目录结构，排除依赖、构建产物和大文件。
4. 让 AI 判断相关目录、模块和候选文件。
5. 使用关键词和文件路径读取相关代码片段。
6. 生成测试策略。
7. 生成测试用例 JSON。
8. 使用 schema 校验。
9. 审查、去重、补边界和补异常。
10. 导出 Excel 和 CSV。
11. 保存中间结果。

中间结果目录：

```text
runs/<run-id>/
  requirement.raw.txt
  requirement.structure.json
  requirement.analysis.json
  repo.tree.txt
  code.candidates.json
  code.snippets.json
  strategy.json
  testcases.raw.json
  testcases.validated.json
  review-report.json
  testcases.xlsx
  testcases.csv
```

验收标准：

- 对 `md`、`docx`、`txt` 能稳定解析文本。
- 文本型 PDF 能提取主体文本。
- 输出 JSON 通过 schema 校验。
- Excel 包含全部核心字段。
- 同一份输入重复运行时结构稳定，不出现不存在的代码路径。
- 生成失败时保留足够中间结果用于定位问题。

## 第 3 步：接入 OpenCode Agent

目标：把 OpenCode 作为 agent runtime，而不是 fork 它的界面。

专用 Agent：

```text
QA Testcase Generator Agent

职责：
- 根据需求文档和代码仓库生成测试用例。
- 必须输出符合 schema 的 JSON。
- 必须引用需求片段和相关代码文件。
- 不允许修改代码。
- 不允许编造不存在的文件路径。
- 低置信度场景必须标记，而不是假装确定。
```

工具：

```text
read_requirement_doc
  输入：文档路径
  输出：纯文本、章节结构、解析警告

repo_tree
  输入：仓库路径、忽略规则、深度
  输出：目录树、文件统计、候选模块

search_code
  输入：仓库路径、关键词、glob、数量限制
  输出：匹配文件、行号、上下文片段

read_code_files
  输入：仓库路径、文件路径列表、行数限制
  输出：文件内容片段、是否截断

export_testcases
  输入：测试用例 JSON、导出格式、目标路径
  输出：导出文件路径、行数、警告
```

Agent 约束：

- 所有工具读取必须限制在用户选择的仓库和需求文档目录内。
- Agent 只能读取和导出，不允许编辑目标仓库代码。
- `codeRefs.file` 必须来自 `repo_tree`、`search_code` 或 `read_code_files` 的真实返回。
- 输出前必须执行 schema 校验。
- 如果需求不足或代码定位失败，生成低置信度用例并在质量报告说明。

验收标准：

- Agent 能完成 CLI 同等流程。
- 工具调用过程可记录并复盘。
- 生成结果包含需求引用和代码引用。
- Agent 不会修改目标仓库文件。

## 第 4 步：多阶段生成流水线

目标：避免一次 prompt 直接生成所有用例，提高可排查性和稳定性。

```text
需求文档
  -> 需求解析
  -> 代码定位
  -> 测试策略规划
  -> 用例生成
  -> 用例审查
  -> 导出 Excel/CSV
```

### 需求解析

输出：

```json
{
  "summary": "...",
  "requirementPoints": [
    {
      "id": "REQ-001",
      "title": "...",
      "description": "...",
      "sourceRef": "第 2.1 节 / 原文片段",
      "roles": ["..."],
      "businessRules": ["..."],
      "acceptanceCriteria": ["..."],
      "boundaryConditions": ["..."],
      "exceptions": ["..."]
    }
  ]
}
```

### 代码定位

输出：

```json
{
  "modules": [
    {
      "name": "...",
      "reason": "...",
      "files": [
        {
          "file": "src/...",
          "reason": "接口入口 / 页面组件 / 服务逻辑 / 数据模型"
        }
      ]
    }
  ],
  "unresolvedRequirementPoints": ["REQ-..."]
}
```

### 测试策略规划

输出：

```json
{
  "coveragePlan": [
    {
      "requirementId": "REQ-001",
      "types": ["functional", "boundary", "exception"],
      "priorityHint": "P1",
      "riskPoints": ["..."]
    }
  ]
}
```

### 用例生成

按需求点分批生成，避免单次上下文过大。每个批次只读取与当前需求点相关的代码片段。

输出：

```json
{
  "testCases": []
}
```

### 用例审查

审查规则：

- 去除标题、步骤和预期高度相似的重复用例。
- 检查每条用例是否有需求引用。
- 检查 `codeRefs.file` 是否真实存在。
- 检查边界、异常、权限和回归覆盖是否符合选择范围。
- 标记低置信度用例。
- 给出缺口报告。

输出：

```json
{
  "coverage": {
    "coveredRequirementPoints": 12,
    "totalRequirementPoints": 15
  },
  "missingBoundaryRequirementIds": ["REQ-..."],
  "missingExceptionRequirementIds": ["REQ-..."],
  "duplicateCaseIds": ["TC-..."],
  "lowConfidenceCaseIds": ["TC-..."],
  "warnings": ["..."]
}
```

## 第 5 步：Electron 客户端

目标：给测试同学一个最小可用的桌面操作界面。

### 项目页

能力：

- 选择本地仓库。
- 填写远程仓库地址并克隆到本地。
- 显示当前分支。
- 展示目录结构摘要。
- 显示最近分析时间。

验收标准：

- 用户能选择本地目录并保存为项目。
- 用户能打开已有项目历史。
- 仓库不存在、不是 Git 仓库或无读取权限时给出明确错误。

### 需求页

能力：

- 上传 `md`、`docx`、`pdf`、`txt`。
- 展示解析后的需求摘要和需求点列表。
- 支持勾选本次要生成的需求点。
- 展示解析警告，例如 PDF 无文本层。

验收标准：

- 用户能看懂 AI 提取出的需求点。
- 用户能排除本次不需要生成的需求点。
- 文档解析失败不会进入生成流程。

### 生成页

能力：

- 选择用例类型：功能、异常、边界、权限、集成、回归。
- 选择覆盖深度：快速、标准、深度。
- 点击生成。
- 展示进度：解析需求、分析代码、生成用例、审查用例、导出。
- 支持失败后查看错误和中间结果。

验收标准：

- 用户能知道当前运行到哪一步。
- 生成失败时不会丢失已完成的中间结果。
- 单次生成目标耗时控制在 3 到 8 分钟。

### 用例页

能力：

- 表格展示测试用例。
- 编辑字段。
- 删除和补充用例。
- 按模块、优先级、类型、置信度过滤。
- 查看需求引用和代码引用。
- 导出 `xlsx` 和 `csv`。
- 查看质量报告。

验收标准：

- 测试同学能直接在表格中完成二次编辑。
- Excel 格式能进入现有测试流程。
- 删除的用例在历史中保留软删除状态。

## Excel 和 CSV 导出

Excel 列建议：

```text
用例编号
模块
标题
类型
优先级
前置条件
测试数据
操作步骤
预期结果
需求引用
代码引用
风险点
置信度
```

格式规则：

- 数组字段使用换行展示。
- `codeRefs` 展示为 `file: reason`。
- 置信度低于 0.6 的行加浅色标记。
- 表头冻结。
- 自动筛选开启。
- 列宽根据内容设置上限，避免超宽。

CSV 规则：

- 使用 UTF-8。
- 数组字段用换行或分号连接。
- 换行、逗号和引号必须正确转义。

## 质量报告

每个生成批次都应生成质量报告。

示例：

```text
覆盖需求点：12 / 15
生成用例数：48
缺少边界用例：2 个需求点
缺少异常用例：1 个需求点
疑似重复用例：3 条
低置信度用例：5 条
无代码引用用例：6 条
```

质量评分建议：

```text
总分 = 需求覆盖分 * 0.35
     + 类型覆盖分 * 0.20
     + 可执行性分 * 0.20
     + 引用可信度分 * 0.15
     + 去重完整性分 * 0.10
```

评分只作为辅助，不应阻止测试同学编辑和导出。

## Prompt 设计原则

- 每阶段 prompt 只完成一个清晰任务。
- 明确要求输出 JSON，不夹杂解释文字。
- 输出前要求自检 schema 字段。
- 明确禁止编造代码路径。
- 对不确定信息使用低置信度和 warning。
- 在生成用例时给出优秀样例作为 few-shot。
- 对同一个需求点分别考虑正常、边界、异常、权限、集成和回归。

用例生成 prompt 必须包含：

- 当前需求点。
- 需求原文引用。
- 相关代码文件和片段。
- 已选择的用例类型。
- 覆盖深度。
- schema。
- 输出数量建议。
- 禁止项。

## 安全和边界

- 仓库读取路径必须限制在用户选择的 repo 根目录。
- 需求文档读取路径必须来自用户上传或选择。
- 远程仓库需要显式用户确认后 clone。
- 不保存模型密钥明文。
- 不把本地代码和需求上传到非授权服务。
- 导出的文件路径需要用户选择或确认。
- Agent 不允许修改目标仓库。

## 开发排期

```text
第 1 周：样例集、schema、CLI 原型
第 2 周：OpenCode agent、自定义工具、多阶段生成
第 3 周：Excel 导出、质量审查、历史记录
第 4 周：Electron 客户端 MVP
第 5 周：真实项目试用、prompt/rule 优化
第 6 周：稳定性、打包、内部分发
```

## 里程碑和验收

### M1：标准和样例完成

产出：

- schema。
- 字段规范。
- 样例集。
- 质量评估标准。

验收：

- 测试同学认可样例质量。
- schema 能覆盖核心字段约束。

### M2：CLI 跑通

产出：

- `qa-gen` 命令。
- 文档解析。
- 仓库扫描。
- AI 生成 JSON。
- Excel/CSV 导出。

验收：

- 至少 2 个真实模块能生成可审查用例。
- 输出可追溯到需求和代码。

### M3：Agent 和工具跑通

产出：

- QA Agent。
- 5 个自定义工具。
- 多阶段流水线。
- 中间结果持久化。

验收：

- Agent 运行过程可复盘。
- 代码引用真实存在。
- 失败时可定位到具体阶段。

### M4：桌面端 MVP

产出：

- 项目页。
- 需求页。
- 生成页。
- 用例页。
- 历史记录。

验收：

- 测试同学不需要命令行即可完成完整流程。
- 能编辑、删除、补充和导出用例。

### M5：灰度试用

产出：

- 真实项目试用报告。
- prompt 和规则优化记录。
- 缺陷清单。
- 下一阶段计划。

验收：

- 至少 2 到 3 个测试同学连续试用 1 周。
- 有真实用例采纳和修改数据。

## MVP 成功标准

- 70% 以上生成用例被测试同学认为“有参考价值”。
- 40% 以上用例可以轻微修改后直接使用。
- 每个需求点能追溯到原文片段。
- 关键用例能追溯到相关代码文件。
- 单次生成时间控制在 3 到 8 分钟。
- Excel 导出能直接进入现有测试流程。
- 灰度用户愿意继续在真实需求中试用。

## 主要风险和应对

| 风险 | 表现 | 应对 |
| --- | --- | --- |
| 需求解析泛化 | 输出很多空泛功能点 | 引入样例和章节结构，要求引用原文 |
| 代码定位不准 | 引用无关文件或漏关键文件 | 结合目录树、关键词搜索和文件读取，多阶段确认 |
| 用例不可执行 | 步骤抽象、预期不可验证 | 质量审查检查步骤和预期，样例约束输出风格 |
| 输出不符合 schema | UI 或导出失败 | 生成后强制 schema 校验，失败则修复或重试 |
| 生成耗时过长 | 用户等待超过 10 分钟 | 按需求点分批，限制读取文件数量，提供快速模式 |
| PDF 解析失败 | 扫描件无文本层 | v0.1 明确只支持文本型 PDF，并给出错误说明 |
| 测试同学不愿改 | 字段多、格式不合流程 | 灰度收集字段和 Excel 格式反馈，快速调整 |
| 官方同步冲突频繁 | 每次合并都大量改同一批文件 | 二开代码放入独立 package，官方代码只保留薄 adapter |
| 二开逻辑污染核心 | session、server、desktop 中散落 QA 业务判断 | 使用插件、Agent 工具、独立配置 namespace 和功能开关 |

## v0.2 候选方向

- Jira、禅道、TestRail 同步。
- 团队用例模板。
- 接口用例自动生成。
- 需求变更后的回归用例推荐。
- 代码 diff 驱动的测试影响分析。
- OCR 支持扫描 PDF。
- 组织级知识库和业务术语表。
- 多人协作和权限系统。
