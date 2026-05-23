# case-test - 需求文档到测试用例自动生成项目

## 项目概述

本项目是一个基于 Claude Code + 多个 Skill 协作的自动化测试用例生成流水线。输入需求文档（PDF/DOCX），经过 5 个阶段的处理，最终输出结构化的 Excel/XMind 测试用例文档。

## 核心原则

- **所有对话回复必须用中文**
- **处理任务过程中不直接写代码**，优先使用工具（Skill、Agent、Bash）完成任务
- **每个阶段完成后必须停止**，向用户展示输出产物并等待确认，不得自动进入下一阶段
- **实时更新任务清单**，每个阶段开始和完成时都要更新 `task_list.md`
- **生成子 Agent 时实时输出状态**，让用户了解每个子 Agent 的任务和进度
- **可并行处理的章节使用子 Agent 并发执行**，提升处理效率

## 技术栈

- Python 3.12（由 `.python-version` 管理）
- Skill 系统：6 个本地 Skill（详见 `.claude/skills/`）
- 项目管理工具：uv/pip

## 五阶段流水线

项目工作流依次执行以下 5 个阶段：

### 阶段 1：文档解析（mineru）

使用 `mineru` Skill 将 `docx/` 目录下的需求文档（PDF/DOCX）转换为 Markdown 格式，输出到 `output/{项目名称}/02_extracted/`。

- 调用方式：Skill 工具，skill 参数为 `mineru`
- 输入：`docx/{需求文档}.pdf` 或 `docx/{需求文档}.docx`
- 输出：`output/{项目名称}/02_extracted/{文档名称}.md` 及 `images/` 目录
- 策略：mineru所需求的token在.env中

### 阶段 2：需求拆分（requirement-splitter）

使用 `requirement-splitter` Skill 对 Markdown 需求文档按章节拆分，提取功能需求单元，生成结构化 JSON。

- 调用方式：Skill 工具，skill 参数为 `requirement-splitter`
- 输入：`output/{项目名称}/02_extracted/{文档名称}.md`
- 输出：`output/{项目名称}/03_requirements/` 下按章节目录组织的 `requirements.json`
- **重要**：拆分时启动多个子 Agent 并行处理不同章节

### 阶段 3：测试点提取（testpoint-extractor）

使用 `testpoint-extractor` Skill 读取需求 JSON 和关联图片，提取测试点。

- 调用方式：Skill 工具，skill 参数为 `testpoint-extractor`
- 输入：阶段 2 输出的 `requirements.json`
- 输出：`output/{项目名称}/04_testpoints/` 下按章节目录组织的 `testpoints.json`
- **重要**：每个需求作为一个独立单元，将需求描述+图片一起给子 Agent 提取测试点
- **重要**：一个需求一个需求地提取，可并发调用子 Agent

### 阶段 4：测试点评审（testpoint-reviewer）

使用 `testpoint-reviewer` Skill 对提取的测试点进行质量评审，输出评审报告。

- 调用方式：Skill 工具，skill 参数为 `testpoint-reviewer`
- 输入：阶段 3 输出的 `testpoints.json` + 阶段 2 输出的 `requirements.json`
- 输出：`output/{项目名称}/05_testpoint_review/` 下按章节目录组织的 `review_report.json`
- 评审维度：覆盖率分析、测试点质量、遗漏检测、一致性检查

### 阶段 5：测试点改进与用例导出（testcase-reports）

使用 `testcase-reports` Skill 完成两个子步骤：**先根据评审意见改进测试点**，再导出为测试用例文档。

- 调用方式：Skill 工具，skill 参数为 `testcase-reports`
- 输入：阶段 3 的 `testpoints.json` + 阶段 4 的 `review_report.json`（两者都必填）
- 步骤 1：根据 `review_report.json` 中的问题和建议，逐一修正测试点，生成 `improved_testpoints.json`
- 步骤 2：基于改进后的测试点导出 Excel/XMind 格式的测试用例
- 输出：
  - `output/{项目名称}/05_testpoint_review/{章节}/improved_testpoints.json`
  - `output/{项目名称}/06_testcase_export/` 下的 `.xlsx` 文件

## 最终输出目录结构

```
output/
├── {项目名称}/
│   ├── task_list.md              # 任务清单（实时更新）
│   ├── 01_original/              # 原始文档
│   │   └── {文档名称}.pdf
│   ├── 02_extracted/             # 阶段1输出
│   │   ├── {文档名称}.md
│   │   ├── images/
│   ├── 03_requirements/          # 阶段2输出
│   │   ├── 01_第X章_xxx/
│   │   │   ├── requirements.json
│   │   │   ├── requirements_detail.json
│   │   │   └── images/
│   │   └── README.md
│   ├── 04_testpoints/           # 阶段3输出
│   │   ├── 01_第X章_xxx/
│   │   │   ├── testpoints.json
│   │   │   └── README.md
│   │   ├── testpoints_summary.json
│   │   └── README.md
│   ├── 05_testpoint_review/     # 阶段4-5输出（评审报告 + 改进后测试点）
│   │   ├── 01_第X章_xxx/
│   │   │   ├── review_report.json          # 阶段4：评审报告
│   │   │   └── improved_testpoints.json    # 阶段5：根据评审改进后的测试点
│   │   ├── review_summary.json
│   │   └── README.md
│   └── 06_testcase_export/      # 阶段5输出
│       ├── {项目名称}_测试用例_{日期}.xlsx
│       └── README.md
└── README.md                    # 项目说明
```

## Skill 清单

| Skill 名称 | 用途 | 阶段 |
|---|---|---|
| `mineru` | 文档解析（PDF/DOCX → Markdown） | 阶段 1 |
| `requirement-splitter` | 需求拆分（Markdown → JSON） | 阶段 2 |
| `testpoint-extractor` | 测试点提取（需求 JSON → 测试点 JSON） | 阶段 3 |
| `testpoint-reviewer` | 测试点评审（质量检查，生成评审报告） | 阶段 4 |
| `testcase-reports` | 测试点改进（根据评审修正）+ 用例导出（JSON → Excel/XMind） | 阶段 5 |
| `xlsx` | Excel 文件创建和编辑的底层工具 | 辅助 |

## 命令速记

启动完整流水线时，按以下步骤逐一执行，**每完成一步必须停下来等待用户确认**：

```
1. 请使用 mineru 将 docx/{需求文档} 转为 Markdown
   → 完成后展示产物，等待用户确认

2. 请使用 requirement-splitter 拆分需求文档
   → 完成后展示产物，等待用户确认

3. 请使用 testpoint-extractor 提取测试点
   → 完成后展示产物，等待用户确认

4. 请使用 testpoint-reviewer 评审测试点（生成评审报告）
   → 完成后展示产物，等待用户确认

5. 请使用 testcase-reports 根据评审意见改进测试点并导出测试用例
   → 完成后展示产物
```

> **注意**：绝不可在用户未确认当前阶段产物的情况下自动执行下一阶段。

## 注意事项

- **每个阶段完成后必须停下来**，向用户展示当前阶段的输出产物，等待用户确认后再继续下一阶段
- 同一阶段内的不同章节可以并行处理（使用子 Agent）
- 处理过程中遇到错误不要跳过，先尝试修复
- 所有输出路径使用正斜杠 `/`，确保跨平台兼容
- 在 Windows 下使用 bash 执行命令，注意路径转义
