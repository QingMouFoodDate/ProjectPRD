# 《轻眸识刻》多端统一 RESTful API 接口规约

> **契约性质**：跨仓库多端协同抽象协议（WebClient / WxClient / MobileClient ↔ TrainPlatform）  
> **基准命名空间**：`/api/v1`  
> **设计原则**：规范交互接口类型与语义职责，不提前锁死内部业务字段细节，为各端实现保留演进弹性

---

## 一、 核心接口分类矩阵

全系统交互基于**识别会话 (`RecognitionSession`)** 模型展开，划分为 7 大抽象接口类别：

| 接口分类 | 典型端点路径 | HTTP 方法 | 调用方与接收方 | 核心职责说明 |
| :--- | :--- | :--- | :--- | :--- |
| **1. 识别会话生命周期** | `/sessions`<br/>`/sessions/{id}` | `POST`<br/>`GET / DELETE` | 客户端 ➔ TrainPlatform | 初始化单件商品的跨图识别会话，查询会话累积的证据链与当前判定状态，或主动关闭会话。 |
| **2. 图像多分支感知** | `/sessions/{id}/images` | `POST` | 客户端 ➔ TrainPlatform ➔ Runtime | 向指定会话上传包装照片（附带拍摄部位标注：瓶盖/背标/侧面等），后端执行多分支感知并追加机器观察值。 |
| **3. 辅助证据录入** | `/sessions/{id}/evidence` | `POST` | 客户端 ➔ TrainPlatform | 提交经客户端解码的商品条码数字，或用户明确确认的保质期天数，即时触发会话状态重新推算。 |
| **4. 纠错与难例上报** | `/sessions/{id}/correction` | `POST` | 客户端 ➔ TrainPlatform | 提交用户对本次会话结果的人工校正记录，自动关联该会话下全量原始图片归档至待审难例库。 |
| **5. 样本资产管理** | `/samples`<br/>`/samples/batch-delete` | `GET`<br/>`POST` | 管理前端 ➔ TrainPlatform | 集中样本资产的多维检索、图库浏览、在线标注框保存与低质重复样本批量剔除。 |
| **6. 算法作业与日志流** | `/training/jobs`<br/>`/training/jobs/{id}/logs` | `POST`<br/>`GET (SSE)` | 管理前端 ➔ TrainPlatform ➔ ModelTrain | 远程下发模型训练作业与超参数配置，通过流式响应实时回传训练过程中的指标变化趋势。 |
| **7. 端侧轻量模型下发** | `/models/latest` | `GET` | MobileClient ➔ TrainPlatform | 移动端检查本地离线模型版本哈希，获取通过评测验证的轻量化量化模型包与元数据。 |

---

## 二、 核心交互语义详述

### 1. 创建识别会话
- **操作语义**：用户开始对一件新食品进行查验时，初始化一个唯一的会话标识；
- **初始状态**：`assessment_status = NOT_ASSESSABLE`, `review_status = INCOMPLETE_EVIDENCE`；
- **返回值概念**：包含生成的 `session_id`、创建时间戳与初始空证据链。

### 2. 追加包装图像与多分支感知
- **操作语义**：用户拍摄或上传包装某部位图片（如瓶盖打码特写或背标配料表）；
- **参数概念**：图片二进制流、拍摄部位提示（可选：`cap`、`back_label`、`barcode`、`overall`）；
- **感知返回概念**：
  - 本张图片在原图上的检测定位边界框（归一化坐标 `[x_min, y_min, x_max, y_max]`）；
  - 提取的机器观察值文本与置信度；
  - 会话当前最新的综合二维状态、已知字段字典与靶向缺失要素提示。

### 3. 补充辅助凭据
- **操作语义**：前端扫码识别出 EAN-13 条码，或用户在手动补录抽屉中确认了保质期时长；
- **参数概念**：凭证类型（`BARCODE_69` 或 `USER_SHELF_LIFE`）、凭证数值、来源标记；
- **处理逻辑**：系统将该凭据记为 L3 或 L5 证据，驱动状态决策引擎即时重算。

### 4. 人工纠偏回流
- **操作语义**：算法识别有偏差时，用户提交正确日期；
- **参数概念**：纠正后的生产日期、到期日、修正说明文本；
- **处理逻辑**：将会话全量图片与人工校准值绑定，打标入库，供算法进行错误类型聚类与主动学习。

---

## 三、 标准响应状态模型

所有业务响应均采用标准外层封装，状态判定结果严格遵循二维正交解耦规约：

- **`code`**：业务状态码（`200` 成功，`400` 参数语义错误，`500` 内部服务异常）；
- **`msg`**：状态摘要说明；
- **`data`**：
  - **`assessment_status`**：`WITHIN_MARKED_PERIOD`、`NEAR_MARKED_END`、`PAST_MARKED_END`、`NOT_ASSESSABLE`；
  - **`review_status`**：`NONE`、`INCOMPLETE_EVIDENCE`、`EVIDENCE_CONFLICT`、`LOW_CONFIDENCE`；
  - **`calculated_details`**：包含推算出的基准生产日期、标示到期日与倒计时天数；
  - **`evidence_chain`**：累积的多源证据列表；
  - **`missing_fields`**：当前仍缺失的关键字段清单（如 `["SHELF_LIFE_SPAN"]`）；
  - **`warnings`**：储存温区约束与包装开封时效等提示文案。
