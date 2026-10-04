# 轻眸识刻设计与文档中枢

> **项目名称**：轻眸识刻——基于小目标驱动的食品包装日期智能识别系统  
> **组织归属**：`QingMouFoodDate` GitHub 协作组织  
> **本仓定位**：需求、架构、候选跨仓契约与质量标准的文档中枢
> **设计状态**：八仓职责推荐方案已批准，正式记录于 [ADR-001](decisions/ADR-001-REPOSITORY_BOUNDARIES.md)
> **实现状态**：M0 待实现；本文不声明功能已上线、指标已达成或生产契约已冻结

---

## 一、 权威依据

项目核心目标以立项申报书为准，答辩材料用于说明技术主线，演进论证材料仅作为远期规划参考。C0 是核心最终目标；C1 为增强、C2 为实验、C3 为储备，均不作为 M0 交付承诺。设计批准、功能实现和验收通过是不同事实，不能互相替代。

---

## 二、 稳定产品定位

系统围绕真实图像采集、日期区域定位、字符识别、包装标示状态推导和人工纠偏建立边界，只推导包装标示状态，不判断食品绝对安全或是否适宜食用。P2 检测头与受限字典 OCR 仅为研究候选，不是既定生产实现。

保留现有八仓：`.github`、`ProjectPRD`、`TrainPlatform`、`ModelTrain`、`InferPlatform`、`WebClient`、`MobileClient`、`WxClient`。TrainPlatform 管理离线数据与模型发布治理，ModelTrain 执行训练评测导出且只读冻结数据，InferPlatform 管理在线业务与内部 runtime。MobileClient、WxClient 暂停实现，不合并、不删除、不改名，也不新增 Data、Registry、Gateway、Runtime 仓。

首个里程碑 M0 是待实现的 C0 单图切片：Web 上传单图 -> 检测与 OCR Observation -> Evidence -> 日期 Decision -> 基础展示。多图会话、完整展示与纠偏审核保留为后续 C0 目标；MobileClient、WxClient、端侧推理和训练平台完整实现不属于 M0。详见 [需求规格](REQUIREMENTS.md) 与 [路线图](ROADMAP.md)。

---

## 三、 中枢文档导航

建议按以下顺序阅读：

1. `README.md`：了解项目定位、权威依据和文档边界；
2. `REQUIREMENTS.md`：了解产品范围、能力分级和非目标；
3. `ARCHITECTURE.md`：了解仓库职责、系统分层和核心数据流；
4. `decisions/`：了解已批准的仓库职责设计决议；
5. `contracts/`：了解待评审、待实现的跨仓契约；
6. `RESEARCH.md`：了解算法基线、中性研究问题和评估指标；
7. `standards/`：了解数据质量和工程协作要求；
8. `ROADMAP.md`：查看 M0 切片、后续阶段和验收门禁。

跨仓协作先对齐需求与 ADR，再评审契约。现有契约中的强制措辞、接口示例、模型形状和数值不代表生产冻结、已支持功能或实测结果；冲突应在后续契约修订中解决，不能据此声称当前实现完成。未提供可追溯验收链接的需求只标为 `Planned` 或 `Proposed`。

| 文档定位 | 文档路径 | 核心内容概述 |
| :--- | :--- | :--- |
| **产品与架构** | [`REQUIREMENTS.md`](REQUIREMENTS.md)、[`ARCHITECTURE.md`](ARCHITECTURE.md) | 产品边界、系统分层、仓库职责和非目标 |
| **设计决议** | [ADR-001](decisions/ADR-001-REPOSITORY_BOUNDARIES.md) | 八仓保留、职责划分与迁移顺序 |
| **跨仓契约** | [`contracts/`](contracts/) | 候选 API、数据模型、日期规则、模型与训练约束 |
| **研发与推进** | [`RESEARCH.md`](RESEARCH.md)、[`ROADMAP.md`](ROADMAP.md) | 科研假设、评估指标和内部推进参考 |
| **质量与协作** | [`standards/`](standards/) | 数据质量、命名、提交和工程协作规范 |

---

## 四、 仓库目录架构导航

```text
ProjectPRD/
├── README.md                      # 项目定位、权威来源与文档导航
├── REQUIREMENTS.md                # 产品层单一事实来源：C0-C3短规格需求池与非目标
├── ARCHITECTURE.md                # 八仓职责、在线离线边界与部署候选
├── RESEARCH.md                    # 基线、H1-H8中性研究问题与指标口径
├── ROADMAP.md                     # 时间与交付推进表：阶段里程碑、依赖关系与准入条件
├── decisions/                     # 已批准的设计决议与迁移边界
├── contracts/                     # API、数据、日期、模型、训练与机器契约候选
└── standards/                     # 系统质量标准与工程协作规范目录
    ├── DATA_QUALITY.md            # 数据真实性、标注质量与质量门禁
    └── ENGINEERING.md             # 命名、分支、提交、文档与测试规范
```

---

## 五、 工程协作原则

1. 仓库职责按 ADR 执行，具体生产字段、规则和接口须在契约评审后冻结；
2. 模型 `register / validate / release` 由 TrainPlatform 治理，`activate / rollback` 由 InferPlatform 控制、内部 runtime 执行；
3. 业务纠偏在 InferPlatform 接受后立即重算，训练资格由 TrainPlatform 独立审核，不自动更新模型；
4. 命名、分支、中文 emoji 提交、文档校验和质量门禁遵循 `standards/`；
5. 迁移按文档 -> 代码 -> 测试 -> CI 推进，不自动重写历史。本仓只承担文档与契约校验，不假装提供可运行服务。

## 六、候选契约导航

| 文档 | 范围 | 当前说明 |
| :--- | :--- | :--- |
| [API](contracts/API.md) | 在线业务 API、会话、权限、幂等与错误处理 | `api.v1-candidate`，M0 待实现 |
| [DATA](contracts/DATA.md) | Observation、Evidence、Decision、Correction 及领域枚举 | 候选 DTO 与语义约束，尚未冻结 |
| [DATE_RULES](contracts/DATE_RULES.md) | 日历运算、期限边界、临期、冲突与历史重放 | `date-rules.v1` 候选，尚无已验收日期引擎 |
| [MODEL](contracts/MODEL.md) | 模型包、元数据、兼容性、发布与切换约束 | `model-bundle.v1` 候选，示例不是实测 |
| [TRAINING](contracts/TRAINING.md) | 冻结数据、训练执行、评测登记与发布治理 | `training.v1` 候选，尚无已实现平台或入口 |

候选机器契约已经纳入本仓，并与人工契约同步评审和版本化：

- [OpenAPI 3.1](contracts/openapi.json)；
- [observation.v1.schema.json](contracts/schemas/observation.v1.schema.json)；
- [evidence.v1.schema.json](contracts/schemas/evidence.v1.schema.json)；
- [decision.v1.schema.json](contracts/schemas/decision.v1.schema.json)；
- [correction.v1.schema.json](contracts/schemas/correction.v1.schema.json)。

文件存在、JSON/Schema 结构校验通过或文档引用有效，都不等于生产契约冻结、服务可运行或模型效果达标。只有对应实现、契约、权限、日期规则、真实模型输入输出及端到端测试通过并保留审核记录后，才能冻结相关版本。

## 七、公开文档与访问权限

本仓内容按可公开文档边界编写，但仓库是否公开、成员访问权和分支保护以实际平台配置为准，本文不声明已配置。公开设计文档不授予图片、训练资产、模型包或在线会话的访问权限；这些资源须分别按来源许可、用户授权和最小权限控制。公开示例不包含私有路径、真实用户信息、秘密凭据或未授权素材。
