# 《轻眸识刻》设计与文档中枢 (ProjectDocs)

> **项目名称**：轻眸识刻——基于小目标驱动的食品包装日期智能识别系统  
> **组织归属**：`QingMouFoodDate` GitHub 协作组织  
> **本仓定位**：**全域设计与文档中枢**（系统架构设计、产品需求规格 PRD、多端接口协议与质量交付规范）  

---

## 一、 项目全景与 6 大仓库矩阵

轻眸识刻致力于攻克工业包装与生活实拍场景下食品生产日期的**小尺度、反光褶皱、点阵喷码断笔、复杂背景干扰**等痛点。为保证团队多人高效协作，系统按技术栈与运行环境解耦为 6 个核心仓库：

| 仓库名称 (GitHub) | 中文定位 | 技术栈属性 | 核心职责 |
| :--- | :--- | :--- | :--- |
| **`ProjectDocs`**<br>*(即 ProjectPRD)* | **设计与文档中枢** | Markdown / Docs | **系统架构设计、产品需求规格 (PRD)、多端统一接口契约、模型交付标准、数据质量宪法**。 |
| **[`TrainPlatform`](https://github.com/QingMouFoodDate/TrainPlatform)** | **业务后端与训练平台** | Python / Go / DB | **多端统一 API 服务（预测/上传/纠偏）、样本资产与元数据管理、轻量训练任务调度、模型版本看板**。 |
| **[`ModelTrain`](https://github.com/QingMouFoodDate/ModelTrain)** | **算法计算核心** | PyTorch / Python | **小目标日期检测网络、包装日期微型 OCR、图像自适应增强算子、模型轻量化导出 (ONNX) 与评测**。 |
| **[`WebClient`](https://github.com/QingMouFoodDate/WebClient)** | **Web 客户端 (在线识别与评测)** | Vue3 / React + Vite | **面向桌面端与全平台的食品包装日期在线拖拽识别、目标框/OCR/到期倒计时可视化、难例纠错标记、答辩演示大屏**。 |
| **[`MobileClient`](https://github.com/QingMouFoodDate/MobileClient)** | **移动端客户端 (实地采图 App)** | React Native / Flutter | **面向超市/现场的高清实地采图工具、防抖防反光引导、包装类型与保质期初标、离线队列、端侧模型快速初验**。 |
| **[`WxClient`](https://github.com/QingMouFoodDate/WxClient)** | **微信小程序端 (C 端轻量查日期)** | 微信原生 / Taro | **面向普通消费者的极简“拍照查生产日期与是否过期”、保质期推导倒计时提醒、即用即走**。 |

---

## 二、 仓库结构与文档导航

```text
ProjectDocs/ (ProjectPRD)
├── README.md                      # [当前文档] 组织全貌、6大仓库矩阵、顶层导航
├── REPOS.md                       # 仓库地图：各仓职责边界、跨仓协作契约、并行推进路线
│
├── architecture/                  # 【系统架构设计中枢】
│   ├── system_overview.md         # 系统总体架构设计、多维协同拓扑与数据流向
│   └── inference_pipeline.md      # 小目标检测驱动的端到端日期识别与推导流水线
│
├── contracts/                     # 【多端跨仓协议与接口规约】（团队协作事实标准）
│   ├── api_specs.md               # 多端统一 RESTful API 契约协议 (/ocr/predict 等)
│   ├── model_contracts.md         # 算法模型轻量化交付规约 (ONNX 输入输出张量规格)
│   └── git_workflow.md            # 多仓库分支管理 (Git Flow) 与 Commit 提交规范
│
├── standards/                     # 【工程质量宪法与行为规范】
│   └── 数据采集审核与质量控制规范.md  # 4 项生活环境容差 + 6 大底线红线 + 事故复盘合订本
│
├── BRD.md                         # 商业需求文档 (战略层)
├── MRD.md                         # 市场需求文档 (战术层)
├── PRD.md                         # 产品需求文档索引 (产品层)
├── SCOPE.md                       # 范围界定：T1 内核 / T2 工程约束 / T3 外端
├── PLAN.md                        # 产品形态与排期规划
├── FeatureList.md                 # 完整功能清单总表
│
├── PRD/                           # 【详细产品需求功能规格库】(44 份模块化需求文档)
│   ├── 01_数据采集/               # SOP 定义、任务派发、移动端拍摄
│   ├── 02_数据导入/               # 本地上传、标准数据集格式 (VOC/YOLO)、视频抽帧
│   ├── 03_数据管理/               # 存储桶、数据集与版本、检索与筛选
│   ├── 04_标注中心/               # 标注器交互、任务派单、质检返工、难例沉淀
│   ├── 05_训练平台/               # 算力资源、训练任务调度、模型版本管理
│   ├── 06_流水线/                 # 算子编排、执行与重跑
│   ├── 07_部署推理/               # 服务发布、在线试跑、端侧导出
│   ├── 08_端接入/                 # 多端接入规范、Open API
│   └── 09_系统管理/               # 用户权限、审计日志、监控看板
│
├── template/                      # 文档编写模版 (BRD / MRD / PRD)
└── workflow/                      # 工作流定义
```

---

## 三、 团队协作与敏捷推进原则

1. **架构契约先行，实现细节自由发挥**：
   - 团队协作以 `contracts/` 目录下的 **API 接口契约** 与 **模型交付规格** 为唯一共同依据；
   - 在契约达成共识后，各端（前端/后端/算法/移动端）负责人可自主选择并发挥最擅长的技术实现，拒绝“保姆式”微观管控。
2. **聚焦轻量闭环，优先单机敏捷验证**：
   - 项目起步阶段避免过度引入复杂的集群编排与繁杂分布式运维；
   - 优先以 `TrainPlatform` 提供轻量 RESTful 服务，配合 `WebClient` 与 `ModelTrain` 快速跑通“传图 ➔ 识别 ➔ 圈框展示 ➔ 算过期日”的最小可用闭环（MVP）。
3. **数据纯正与质量一票否决**：
   - 全体组员在采图与审核时严格遵循 `standards/数据采集审核与质量控制规范.md`；
   - 坚决杜绝任何算法伪造标签、手绘红圈标记与拼图大板粗审，严守数据纯洁性底线。

---

## 四、 快速查阅指引

- **想了解系统全局怎么串联？** ➔ 请查阅 [`architecture/system_overview.md`](architecture/system_overview.md)
- **想了解算法模型怎么跑、怎么算过期日？** ➔ 请查阅 [`architecture/inference_pipeline.md`](architecture/inference_pipeline.md)
- **前后端如何调接口？** ➔ 请查阅 [`contracts/api_specs.md`](contracts/api_specs.md)
- **算法模型怎么导出给后端或手机？** ➔ 请查阅 [`contracts/model_contracts.md`](contracts/model_contracts.md)
- **采图与标注有什么要求？** ➔ 请查阅 [`standards/数据采集审核与质量控制规范.md`](standards/数据采集审核与质量控制规范.md)
- **查看详细的系统功能蓝图？** ➔ 请查阅 [`PRD.md`](PRD.md) 与 [`PRD/`](PRD/) 目录。
