# 轻眸识刻工程协作与规范指南

> **适用范围**：现有八仓的文档、契约与开发协作
> **设计依据**：[ADR-001](../decisions/ADR-001-REPOSITORY_BOUNDARIES.md)
> **状态边界**：协作规范不代表功能已实现、权限已配置或生产契约已冻结

通用协作规则与领域契约分别管理。字段、枚举、模型形状和数据命名属于候选领域契约，变更须评估关联实现和测试；提交、评审、文档排版与真实验证属于协作要求。

## 一、命名与领域分类

### 1. 样本与资产

- 稳定样本 ID 与文件名通过清单关联，不以文件名推定来源或语义；
- 既有数字名（如 `00001.jpg`）可作为兼容示例，不强制为连续编号重命名历史资产；
- 原图保留原始格式、分辨率与完整性，训练裁剪与增强单独记录派生关系；
- `manifest.jsonl` 为候选图像清单名；bbox、OCR、语义、来源许可、group_key 与 split 须明确关联。

### 2. API 路由与字段

- 路由采用小写复数名词与短横线（如 `POST /api/v1/sessions`、`POST /api/v1/sessions/{session_id}/images`）；
- JSON 字段使用 `snake_case`（如 `session_id`、`image_id`、`mfg_date`、`exp_date`、`shelf_life`、`marked_status`）；派生字段须带来源与规则版本；
- API 接入属于 InferPlatform 模块，不独立设立 Gateway 仓。

### 3. 枚举按领域约定

枚举大小写按领域分类统一约定：

| 领域 | 形式 | 候选值与约束 |
| :--- | :--- | :--- |
| 标示期限 `marked_status` | 大写下划线 | `WITHIN_MARKED_PERIOD`、`NEAR_EXPIRY`、`PAST_MARKED_PERIOD`、`UNDETERMINED` |
| 证据复核 `review_status` | 大写下划线 | `EVIDENCE_COMPLETE`、`INFORMATION_MISSING`、`EVIDENCE_CONFLICT`、`LOW_CONFIDENCE` |
| 会话状态与业务错误 | 大写下划线 | 如 `AWAITING_INPUT`、`IMAGE_INVALID`、`INFERENCE_TIMEOUT` |
| 文本语义类型 | 大写缩写 | `MFG`、`EXP`、`SHELF_LIFE` |
| 打码工艺 | 小写下划线 | `inkjet_dot_matrix`、`inkjet`、`laser_engraving`、`emboss`、`label_thermal`、`print` |
| 包装载体 | 小写下划线 | `plastic_bag`、`can`、`bottle`、`bottle_cap`、`box`、`tetra_pak`、`plastic_tray` |
| 图片角色与客户端 | 小写词 | 如 `macro_date`、`packaging_full`、`barcode`、`web` |
| 模型治理状态 | 小写词 | `candidate`、`validated`、`released`、`deprecated` |
| 需求当前状态 | 首字母大写 | 无可追溯验收链接时只使用 `Planned` 或 `Proposed` |

### 4. 模型包与治理动作

候选资产名为 `qingmou_det.onnx`、`qingmou_rec.onnx`、`dict.txt`、`metadata.yaml`：

- ModelTrain 只读冻结数据，执行训练、评测与导出，提供血缘产物；
- TrainPlatform 治理 `register / validate / release`，记录数据版本、任务与发布门禁；
- InferPlatform 控制 `activate / rollback`，内部 runtime 校验兼容性并执行原子切换；
- 业务纠偏接受后即时重算，训练资格独立审核，不自动更新模型。

### 5. 文档命名

核心文档与契约使用大写文件名（如 `README.md`、`DATE_RULES.md`）。设计决议使用 `ADR-编号-主题.md`。机器契约保持声明路径。

各级 Markdown 标题严禁包含中英文括号。目录树仅展示根配置与目录/子目录层级，每行附简短对齐的中文功能注释，不展开代码细节。

## 二、分支与中文提交规范

### 1. 分支管理

- `main` 保持可验证：代码仓满足构建/测试要求，文档仓满足链接与契约一致性；
- 主干保护按平台配置执行，禁止强制推送覆盖历史；
- `feat/*`、`fix/*` 从主干建立，按切片评审并合入。

### 2. 提交信息

格式为 `<emoji> <type>: <简明中文改动>`，动宾短语，不以句号结尾：

| 前缀 | 含义 | 示例 |
| :--- | :--- | :--- |
| `🎉 init` | 项目初始化 | `🎉 init: 初始化仓库工程结构` |
| `✨ feat` | 新特性 | `✨ feat: 新增多图会话接口` |
| `🐞 fix` | 修复缺陷 | `🐞 fix: 修复闰年日期推导偏差` |
| `🎈 perf` | 性能与体验优化 | `🎈 perf: 优化模型推理耗时` |
| `🦄 refactor` | 代码重构 | `🦄 refactor: 调整业务平台模块分层` |
| `📃 docs` | 仅修改文档 | `📃 docs: 记录八仓职责与单图里程碑` |
| `🌈 style` | 仅格式缩进 | `🌈 style: 统一组件代码排版` |
| `🧪 test` | 增加或修改测试 | `🧪 test: 增加会话接口集成测试` |
| `🔧 build` | 依赖相关 | `🔧 build: 更新模型导出依赖` |
| `🐎 ci` | CI/CD 配置 | `🐎 ci: 调整持续集成检查流程` |
| `🐳 chore` | 构建或辅助工具 | `🐳 chore: 更新开发辅助脚本` |
| `↩ revert` | 版本回滚 | `↩ revert: 回滚会话接口变更` |

### 3. 提交前核验

1. 使用 `git status` 与 `git diff` 检查工作区，排除缓存、依赖和临时文件；
2. 运行与改动相关的检查命令，如实记录结果；
3. 原图、视频与模型权重严禁进入代码仓，使用受控存储和版本引用；
4. 权限按任务授权执行，不超范围修改文件。

## 三、文档状态与权限

- 客观陈述场景、边界、输入输出与验收，不写宣传口号；
- 性能与准确率无数据时标为未测量目标；示例不作为实测成绩；
- 严禁包含私有路径、未脱敏用户数据或凭证；
- 包装标示状态不等于食品安全结论，保留复核提示；
- 数学表达采用纯文本：映射用 `->`，乘除用 `*`、`/`，指数用 `^`。

## 四、契约与真实验证

契约按 API、DATA、DATE_RULES、MODEL、TRAINING 分工。机器契约包含 `contracts/openapi.json` 及 `contracts/schemas/` 下四份 Schema：

- 文档仓检查需求 ID、职责一致性、链接与 Markdown 结构；
- 实现仓分别验证 Observation、Evidence/Decision 规则、纠偏即时重算、分组防泄漏与模型激活隔离；
- M0 优先单机单图闭环；多图与完整纠偏审核不作为 M0 阻断门禁。

## 五、迁移与协作

1. 保留八仓原名，旧职责弃用；MobileClient、WxClient 暂缓实现；不新增其它仓库；
2. 按文档 -> 代码 -> 测试 -> CI 顺序迁移，保留版本映射与可验证记录；
3. 契约先评审再实现，模块间不直接耦合内部实现细节；
4. 先核验、后执行、再复检，不伪造门禁结果。
