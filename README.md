# 轻眸识刻设计与文档中枢

> 面向全域系统规范、需求池、跨仓技术契约与质量标准的单一事实来源
>
> **工程状态**：八仓职责架构已批准 · 正式载入 ADR-001 · M0 核心原型推进中

ProjectPRD 作为轻眸识刻全域工程的“设计与规范中枢”，统筹管理产品需求池（REQUIREMENTS）、技术架构设计（ARCHITECTURE）、跨仓通信与数据契约（contracts）、科研实验规范（RESEARCH）以及全周期推进路线（ROADMAP）。全域代码仓库的接口实现、数据校验与开发行为均以此仓为唯一事实来源（SSOT）。

---

## 一、权威依据与目标分级

项目核心承诺严格以立项申报书为最高准绳，答辩材料阐明技术主线，论证演进材料作为远期技术储备：

- **C0 核心目标**：单图与多图会话流、日期小目标检测、微型字符 OCR、标示状态推导与人工纠偏闭环，为系统最终验收基准；
- **C1 增强扩展**：复杂包装形变自适应、局部图像增强与条码弱辅助对齐；
- **C2/C3 远期储备**：移动端传感器防抖辅助、适老化大字关怀模式与端侧轻量化推理探索；
- **M0 优先验证**：作为 C0 的单图主链路切片，优先拉通端到端识别与验证闭环。

---

## 二、产品稳定定位与法定边界

系统聚焦于食品包装日期的智能视觉识别与语义推导，产品核心边界法定限定为**包装标示状态推导**：

1. **法定对象受限**：系统仅对包装上客观印刷/喷印的文本内容进行检测识别与日历规则计算，严禁向用户直接断言食品是否绝对可食用或卫生安全结论；
2. **稳定八仓矩阵**：全系统严格划分为 8 个职责明确的独立代码仓库，不随意新增、合并或更名仓库；
3. **在线离线严格解耦**：InferPlatform 专职负责在线 API 与实时推理，TrainPlatform 专职负责样本资产治理与训练调度，ModelTrain 专职负责算法网络实现。

---

## 三、中枢文档阅读指引

建议研发成员与代码审查者按以下推荐顺序深入阅读：

1. [`README.md`](README.md)：系统全貌、权威依据与文档导航；
2. [`REQUIREMENTS.md`](REQUIREMENTS.md)：C0~C3 产品需求池与明确非目标；
3. [`ARCHITECTURE.md`](ARCHITECTURE.md)：八仓物理拓扑、在线/离线解耦与数据流向模型；
4. [`decisions/ADR-001-REPOSITORY_BOUNDARIES.md`](decisions/ADR-001-REPOSITORY_BOUNDARIES.md)：八仓职责划分与边界决策记录；
5. [`contracts/`](contracts/)：跨仓通信 API、数据模型、法定日期规则与模型交付规约；
6. [`RESEARCH.md`](RESEARCH.md)：算法研究基线、H1~H8 中性消融假设与评测口径；
7. [`standards/`](standards/)：真实数据质量门禁与全域工程协作规范；
8. [`ROADMAP.md`](ROADMAP.md)：M0 准入准出标准与各阶段演进路线。

---

## 四、仓库目录架构导航

```text
ProjectPRD/
├── contracts/                     # 跨仓接口与模型格式契约目录
│   └── schemas/                  # 机器校验数据结构定义目录
├── decisions/                     # 架构决策记录目录
├── standards/                     # 研发质量与数据规范约束目录
├── README.md                      # 中枢定位、权威依据与文档导航
├── REQUIREMENTS.md                # 产品层单一事实来源短规格需求池
├── ARCHITECTURE.md                # 八仓职责、在线离线边界与部署候选
├── RESEARCH.md                    # 算法基线与中性消融实验设计指南
└── ROADMAP.md                     # 阶段里程碑推进与准入准出规约
```

---

## 五、全域候选契约导航

各工程仓之间的交互通信与数据交换强制绑定以下候选契约，禁止私自定制数据传输格式：

| 契约文档 | 覆盖领域与核心约束 | 状态说明 |
| :--- | :--- | :--- |
| [contracts/API.md](contracts/API.md) | 在线 RESTful API、识别会话、鉴权令牌、幂等性与错误码 | 候选标准，M0 优先落地 |
| [contracts/DATA.md](contracts/DATA.md) | 机器 Observation、聚合 Evidence、决策 Decision 与 Correction | 候选标准，底层统一数据格式 |
| [contracts/DATE_RULES.md](contracts/DATE_RULES.md) | 公历日历规则、不减一天准则、冲突消歧与多状态推导 | 候选标准，语义推导引擎基准 |
| [contracts/MODEL.md](contracts/MODEL.md) | 标准模型包文件清单、张量维度、字典、预后处理与 smoke 用例 | 候选标准，模型统一交付格式 |
| [contracts/TRAINING.md](contracts/TRAINING.md) | 冻结数据集只读规范、单机调度输入输出与状态机迁移 | 候选标准，离线实验统一约束 |

全系统同时维护具备强类型约束的机器可读契约：
- [OpenAPI 3.1 规范定义](contracts/openapi.json)
- [Observation 机器观察值 Schema](contracts/schemas/observation.v1.schema.json)
- [Evidence 结构化证据 Schema](contracts/schemas/evidence.v1.schema.json)
- [Decision 标示决策 Schema](contracts/schemas/decision.v1.schema.json)
- [Correction 字段纠偏 Schema](contracts/schemas/correction.v1.schema.json)

---

## 六、工程协作与质量原则

- **架构决议至上**：所有跨仓代码实现必须严格对齐 ADR-001 边界；
- **模型生命周期隔离**：TrainPlatform 负责治理留痕（`register / validate / release`），InferPlatform 负责在线加载与安全回滚（`activate / rollback`）；
- **数据流闭环纪律**：用户纠偏在在线层即时更新会话决策；晋升为离线训练样本必须经由 TrainPlatform 严格质量审核，杜绝未经审核的标签污染；
- **代码提交规范**：遵循 Conventional Commits 格式 `<emoji> <type>: <简明中文改动>`；
- **资产与代码隔离**：大容量图像与模型文件绝不入 Git 仓库，敏感配置通过受控环境变量注入。
