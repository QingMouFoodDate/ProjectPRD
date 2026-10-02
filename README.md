# 《轻眸识刻》设计与文档中枢 (ProjectDocs)

> **项目名称**：轻眸识刻——基于小目标驱动的食品包装日期智能识别系统  
> **组织归属**：`QingMouFoodDate` GitHub 协作组织  
> **本仓定位**：**全域设计与文档中枢**（系统架构设计、跨仓接口协议、数据质量标准与需求规格库）  

---

## 一、 30 秒开工直达索引 (按技术任务查阅)

团队协同以任务方向为导向，不设僵化的人岗限制。组员在推进具体任务时，只需查阅对应的核心文档即可快速开工：

| 你当前要推进的任务方向 | 核心目标与交付物 | 重点查阅这 2~3 篇文档 |
| :--- | :--- | :--- |
| **算法训练与模型调优** | 针对食品包装反光与折角，训练小目标检测与微型字符 OCR，实现点阵纠错矩阵，导出标准 ONNX 模型 | 1. [算法流水线设计](architecture/inference_pipeline.md)<br>2. [模型交付契约](contracts/model_contracts.md)<br>3. [数据质量标准](standards/数据采集审核与质量控制规范.md) |
| **Web 演示前台研发** | 打造核心 Web 原型系统（拖拽/上传图片、Canvas 检测框高亮、保质期倒计时卡片、错例纠偏回流） | 1. [统一 API 契约协议](contracts/api_specs.md)<br>2. [WebClient 蓝图](../WebClient/README.md) |
| **业务后端支持** | 跑通轻量 RESTful API，接收图片并调用模型推理，结合保质期计算剩余天数，存盘难例样本 | 1. [统一 API 契约协议](contracts/api_specs.md)<br>2. [系统整体拓扑](architecture/system_overview.md) |
| **数据采集与清洗标注** | 实地拍摄真实食品包装、按照生活实拍容差标准审核样本、运行自动化门禁脚本确保数据集纯正 | 1. [数据采集与质检规范](standards/数据采集审核与质量控制规范.md)<br>2. [数据集暂存库指引](../data/README.md) |
| **答辩材料与申报梳理** | 凝练创新点（小目标、纠错矩阵、决策树）、准备大创立项/中期/结题材料、竞赛申报与软著整理 | 1. [项目立项论证 (docs/)](docs/BRD.md)<br>2. [核心范围界定](docs/SCOPE.md)<br>3. [根目录归档材料](../reports/) |

---

## 二、 组织 6 大仓库矩阵全景

整个轻眸识刻系统按技术栈与运行环境解耦为 6 个核心仓库，杜绝单仓臃肿与代码冲突：

| 仓库名称 (GitHub) | 中文定位 | 技术栈属性 | 核心职责 |
| :--- | :--- | :--- | :--- |
| **`ProjectDocs`**<br>*(即 ProjectPRD)* | **设计与文档中枢** | Markdown / Docs | **系统架构设计、接口契约、质量规范、阶段计划与需求规格库**。严禁存放业务代码。 |
| **[`TrainPlatform`](https://github.com/QingMouFoodDate/TrainPlatform)** | **业务后端与平台** | Python / Go / DB | **多端统一 API 服务（预测/上传/纠偏）、样本资产管理、轻量单机调度**。 |
| **[`ModelTrain`](https://github.com/QingMouFoodDate/ModelTrain)** | **算法计算核心** | PyTorch / Python | **小目标检测、微型字符 OCR、点阵双向纠错矩阵、ONNX 轻量化导出与评测**。 |
| **[`WebClient`](https://github.com/QingMouFoodDate/WebClient)** | **Web 核心原型系统** | 前端框架不限 (Vue/React/原生) | **大创官方预期成果 3：在线图片拖拽识别、Canvas 目标框、倒计时卡片、答辩演示大屏**。 |
| **[`MobileClient`](https://github.com/QingMouFoodDate/MobileClient)** | **移动端采图 App** | React Native / Flutter / Android | **面向商超现场的高清实拍工具、防抖防反光引导、包装属性初标、离线队列**。 |
| **[`WxClient`](https://github.com/QingMouFoodDate/WxClient)** | **微信小程序端** | 微信原生 / Taro / UniApp | **面向普通消费者的极简“拍照查生产日期与是否过期”、倒计时提醒、即用即走**。 |

---

## 三、 仓库结构与文档导航

```text
ProjectPRD/
├── README.md                      # [当前文档] 组织全貌、6大仓库矩阵、按任务30秒直达导航
├── REPOS.md                       # 仓库地图：各仓职责边界、跨仓协作契约、并行推进路线
├── PLAN.md                        # 大创阶段演进计划与极简任务池
│
├── contracts/                     # 【跨仓契约中枢】(所有代码实现唯一公认标准，接口先行)
│   ├── api_specs.md               # 多端统一 RESTful API 契约协议 (/ocr/predict 等)
│   ├── model_contracts.md         # 算法模型轻量化交付规约 (ONNX 输入输出张量规格与字典)
│   └── git_workflow.md            # 多仓库分支管理 (Git Flow) 与 Commit 提交规范
│
├── standards/                     # 【质量与规范底线】
│   ├── 数据采集审核与质量控制规范.md  # 4 项生活环境容差 + 6 大底线红线 + 自动化门禁说明
│   └── 项目开发与工程协作原则.md      # 单机轻量优先、技术选型松绑、防过度设计与核验准则
│
├── architecture/                  # 【系统架构与算法设计】
│   ├── system_overview.md         # 系统总体架构设计、多维协同拓扑与数据流向
│   └── inference_pipeline.md      # 小目标检测驱动的端到端日期识别与推导流水线
│
├── docs/                          # 【项目论证与战略背景归档】(大创申报、答辩背景储备)
│   ├── BRD.md                     # 商业与社会价值论证
│   ├── MRD.md                     # 市场与用户需求剖析
│   ├── SCOPE.md                   # 功能层级界定 (T1 MVP / T2 工程基准 / T3 远期拓展)
│   ├── PRD.md                     # 原始需求总索引
│   └── FeatureList.md             # 完整功能全貌总表
│
├── PRD/                           # 【功能详细规格库】(44 份工业级规格，分级降噪查阅)
│   ├── README.md                  # 分级阅读指南 (T1 核心必读 vs T2/T3 远景储备)
│   └── 01_数据采集/ ~ 09_系统管理/ # 详细模块化需求文档
│
├── template/                      # 文档编写模版 (BRD / MRD / PRD) 与编写规范指引
└── workflow/                      # 工作流状态定义
```

---

## 四、 敏捷推进与团队协作原则

1. **架构契约先行，实现细节自由发挥**：
   - 团队协作以 `contracts/` 目录下的 **API 接口契约** 与 **模型交付规格** 为唯一共同依据；
   - 只要契约输入输出对齐，各端负责同学可自由选用最擅长的语言与技术框架，拒绝过度干涉。
2. **聚焦轻量闭环，优先单机敏捷验证**：
   - 项目初期全力保障核心链路跑通，优先以 `TrainPlatform` 提供轻量 RESTful 服务，配合 `WebClient` 与 `ModelTrain` 跑通“传图 ➔ 圈框识别 ➔ 算过期日”的最小可用闭环（MVP）。
3. **数据纯正与质量一票否决**：
   - 全体组员在采图与审核时严格遵循 `standards/数据采集审核与质量控制规范.md`；
   - 坚决杜绝任何算法伪造标签、手绘红圈标记与拼图大板粗审，严守数据纯洁性底线。
