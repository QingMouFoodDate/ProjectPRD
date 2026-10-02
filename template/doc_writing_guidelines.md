# AGENTS.md - 轻眸识刻 产品文档

> AI Agent 操作指南 | 轻眸识刻 视觉模型训练基建

## Repository Overview

**Type**: Product Documentation (BRD / MRD / PRD)
**Product**: 轻眸识刻 - 视觉检测模型训练基建平台
**Language**: 中文文档, English technical terms
**Build/Lint/Test**: N/A (纯文档仓库，无代码)

---

## Document Structure

```
轻眸识刻/
├── BRD.md                    # 商业需求文档 (战略层)
├── MRD.md                    # 市场需求文档 (战术层)
├── PRD.md                    # 产品需求文档索引 (产品层)
├── FeatureList.md            # 功能总表
├── PRD/                      # 详细产品需求文档
│   ├── 01_数据接入/
│   ├── 02_数据管理/
│   ├── 03_标注中心/
│   ├── 04_训练平台/
│   ├── 05_流水线/
│   ├── 06_部署推理/
│   ├── 07_系统管理/
│   └── 08_端接入/
├── template/                 # 文档模板
└── workflow/                 # 工作流程定义
```

**信息流**: BRD → MRD → PRD (战略指导产品)

---

## Language Rules

| 场景 | 语言 |
| :--- | :--- |
| 与用户沟通 | 中文 |
| 文档内容 | 中文 |
| 内部思考 | English |
| 技术术语 | English (YOLO, ONNX, COCO, mAP, S3, RBAC, TensorRT, GPU, Docker) |

---

## Naming Conventions

| 类型 | 格式 | 示例 |
| :--- | :--- | :--- |
| BRD | `B0000+项目名` | `B0001_轻眸识刻` |
| MRD | `M0000+项目名` | `M0001_轻眸识刻` |
| PRD | `P0000+功能名` | `P0003_视频抽帧` |
| 模块目录 | `M00+模块名` | `01_数据接入` |
| 功能 | `F00+功能名` | `F02_压缩包导入` |

### PRD 编号分配（预分配区间，禁止跨区间占用）

| 模块目录 | 编号区间 | 职责 |
| :--- | :--- | :--- |
| `01_数据采集` | `P0001-P0019` | 采集作业链：项目、任务下发、现场拍摄、验收 |
| `02_数据导入` | `P0020-P0039` | 已有文件的导入：本地上传、压缩包、标准格式、抽帧、预检 |
| `03_数据管理` | `P0040-P0059` | 存储绑定、数据集与版本、检索、预览、统计、导出 |
| `04_标注中心` | `P0060-P0079` | 标注器、派单、质检、AI 预标注、字符转写 |
| `05_训练平台` | `P0080-P0099` | 算力节点、环境镜像、任务调度、指标、模型仓库、评估 |
| `06_流水线` | `P0100-P0109` | 十阶段算子化编排与执行记录 |
| `07_部署推理` | `P0110-P0119` | 模型服务、在线试跑、端侧导出、灰度回滚 |
| `08_端接入` | `P0120-P0129` | 微信小程序 / RN 客户端契约、开放 API |
| `09_系统管理` | `P0130-P0139` | 用户权限、审计日志、告警中心 |

---

## PRD 必备章节 (10 sections)

1. **全局说明** - 名词解释, 统一异常处理 (E-code)
2. **项目背景** - Context, Problem Statement, Goals, Risks
3. **价值模型** - 方案概述, 目标指标
4. **项目范围** - 功能结构图 + In/Out of Scope
5. **业务流程** - 核心流程 (Mermaid 必须)
6. **用户分析** - User Personas, Journey Map
7. **功能需求** - User Stories, 验收标准, 边界情况
8. **技术架构与数据模型** - 数据实体表, 状态机
9. **UI/UX 规格说明** - ASCII 线框图, 交互元素表
10. **非功能性需求** - Performance, Security, Compatibility, Scalability

---

## User Story Format

```markdown
### 用户故事

作为 [角色]，
我想要 [功能/行为]，
以便 [目的/价值]。

### 逻辑与业务规则

- **前置条件**: …
- **主流程**: 1. … 2. …
- **后置条件**: …

### 验收标准 (Gherkin)

```gherkin
场景: 场景名
  Given 前置条件
  When 动作
  Then 预期结果
```

### 边界情况与异常处理

| 场景 | 处理 |
| :--- | :--- |
| 网络中断 | … |
| 权限拒绝 | … |
| 空数据 | … |
```

---

## Diagram Requirements (MANDATORY)

**任何复杂逻辑必须使用 Mermaid**:

```mermaid
sequenceDiagram    # API/交互流程
flowchart TD       # 业务流程
stateDiagram-v2    # 状态机
erDiagram          # 数据关系
```

---

## Data Table Format

```markdown
| 字段名 | 类型 | 必填 | 说明 | 约束 |
| :--- | :--- | :--- | :--- | :--- |
| id | UUID | Y | 主键 | PK, auto-gen |
| status | Enum | Y | 状态 | DRAFT/ACTIVE/ARCHIVED |
```

---

## UI/UX Specification Format

```markdown
### 交互元素

| 元素 | 类型 | 状态 | 行为 |
| :--- | :--- | :--- | :--- |
| 提交按钮 | Button | Default/Hover/Active/Disabled/Loading | onClick: submit |

### 表单设计

| 字段 | 类型 | 校验规则 | 默认值 |
| :--- | :--- | :--- | :--- |
| 数据集名称 | Input | required, 1-64 chars | - |
```

---

## PRD Workflow (prd-workflow)

1. **读取上下文**: 先读 BRD.md 和 MRD.md
2. **探索同级文档**: 读取同一模块下的相关 PRD
3. **确认细节**: 提问直到所有问题决策完毕
4. **生成草稿**: 完整临时 artifact 供评论
5. **迭代修改**: 根据反馈多次修订
6. **发布**: 用户说 "发布草稿" 后存储为 Markdown

---

## Key Guidelines

### 主动预判 (Proactive)
用户说 "数据集导入" → 自动补全: 格式解析、类别映射、脏数据预检、重复检测、失败重试、幂等。

### 技术对齐 (Tech-Aligned)
- 包含关键数据结构和状态机逻辑
- 文档要让开发者具备必要信息

### 深度防御 (Edge Case Defense)
永远处理: 网络中断、权限拒绝、空数据、磁盘写满、大文件、并发写入。

### 结构化输出
Markdown 格式；复杂逻辑必须 Mermaid 图表。

---

## File Editing Rules

- **NEVER** 未经请求创建新 PRD 文件
- **ALWAYS** 先读 BRD.md 和 MRD.md
- **ALWAYS** 检查同模块的 sibling 文档
- **PREFER** 编辑已有文档而非新建
- **USE** 一致的标题层级 (# 标题, ## 章节)

---

## Thinking Framework

1. **Why (战略层)**: 价值是什么？
2. **What (架构层)**: 涉及哪些模块？数据流向？ER 关系？
3. **How (执行层)**: 交互逻辑、字段定义、校验规则、失败语义

---

## Common Mistakes to Avoid

1. 用英文写中文文档
2. 复杂流程缺少 Mermaid 图
3. 验收标准模糊 (必须 Given/When/Then)
4. 忽略边界和异常处理
5. 未读父级文档就创建 PRD
6. 命名规范不一致
7. 为不存在的规模做过度设计 (本产品面向单机 2~3 人团队)
