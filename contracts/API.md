# 轻眸识刻在线 API 候选契约

> **状态**：`api.v1-candidate`，由 InferPlatform 实现后提供；当前 M0 待实现。文档和机器契约不证明服务可运行。
> **事实源**：[DATA](DATA.md)、[DATE_RULES](DATE_RULES.md)、[MODEL](MODEL.md)、[职责决议](../decisions/ADR-001-REPOSITORY_BOUNDARIES.md)。
> **机器定义**：[openapi.json](openapi.json)，OpenAPI 3.1；其中引用四份候选 JSON Schema。实施、契约、权限、日期和真实模型测试通过后才能冻结。

## 一、范围与基础约定

| 项目 | 候选约定 |
| :--- | :--- |
| 客户端接入 | WebClient 优先；MobileClient、WxClient 为后续规划，只访问 InferPlatform 业务 API |
| 传输 | 生产使用 HTTPS；本地开发可使用 HTTP；UTF-8 |
| 请求 | JSON；图片上传使用 multipart/form-data |
| 时间 | 带明确偏移的 RFC 3339；日期计算使用服务端 observed_at 和 DATE_RULES，不以客户端或 EXIF 时钟替代 |
| 成功 | HTTP 200，响应 `{code: 200, message, data, request_id}` |
| 失败 | 合适的 HTTP 4xx/5xx，响应 `{code, message, data: null, request_id}`；不回传秘密、内部路径或原始敏感文本 |
| DTO | 完整、安全的 observation.v1 / evidence.v1 / decision.v1；不是数据库对象，也不是未声明摘要 |
| 分层 | 业务层编排、鉴权、Evidence 和 Decision；内部 Runtime 只生成 Observation，不对客户端开放 |

OpenAPI 的示例全部是合成结构校验输入，不来自实拍、模型运行或部署；不得加入训练集或当作项目指标。对象类型、可空性、来源、坐标和证据哈希的约束以 DATA 为准；Schema 结构通过不能替代语义或完整性校验。

## 二、匿名凭证与资源归属

1. 创建会话返回服务端生成的不可预测、短期不透明 `session_token`。令牌不得由 session_id 推导，不要求企业登录或 SSO。
2. 后续会话查询、上传和纠偏携带 `Authorization: Bearer <session_token>`。创建操作本身不要求已有会话令牌。
3. 会话、图片、Observation、Evidence、Decision 和 Correction 绑定创建时的匿名主体；知道 ID 不等于拥有访问权。路径和正文中的 image_id 均须校验归属。
4. 令牌有效期不晚于会话有效期，具体 TTL 和保留策略由不可变部署配置显式记录，不能默认永久。关闭浏览器不承诺服务端已删除；用户界面须说明保留与清理策略。
5. 缺少、伪造或单独失效的凭证返回 401；跨主体访问返回 403。能验证归属但会话已过期时返回 410；不能验证归属时不披露他人资源状态。
6. 创建响应及其合法幂等重放返回同一令牌；GET、日志、图片 URL、分享卡片和证据对象不返回令牌。创建响应使用 `Cache-Control: no-store`。
7. 客户端按平台安全能力最小化保存凭证，不放 URL、公开分享或日志；丢失凭证不凭 session_id 重新签发。用户只能重新创建会话或使用后续正式定义的恢复机制。
8. `/models/latest` 为 C3 预留，须另有模型元数据读取授权；M0 会话令牌不自动获得下载或模型管理权限。

## 三、请求幂等与并发

创建、上传、纠偏均必须携带 `Idempotency-Key`。每次新业务操作生成不可预测、至少 128 位随机熵的键；网络重试复用同一键，而不是重新生成。

| 情形 | 候选处理 |
| :--- | :--- |
| 同键、同请求 | 返回首次处理的原结果与原实体 ID，不重复创建、推理、纠偏或按新时间生成 Decision |
| 同键、不同请求 | HTTP 409；不复用原结果，也不执行新请求 |
| 原请求仍在处理 | HTTP 409 并提供 Retry-After；复用原键稍后重试，不启动第二任务 |
| 创建会话 | 键作用域为创建操作；创建键也是领取创建结果的临时凭证，必须保密、限流并校验原请求指纹 |
| 已有会话写入 | 键作用域为已鉴权主体 + session_id + 端点，不接受他人键作为授权 |
| 文件请求指纹 | 校验解码后的实际文件和上传内容 SHA-256、角色及辅助字段；相同键不能替换文件 |
| 保留与过期 | 幂等记录至少覆盖会话有效期和约定重试窗口；已过期操作返回明确过期错误，新操作须新建键 |

首次领取令牌必须经 HTTPS；创建幂等结果须受控保存，不在普通缓存或日志存明文凭证。并发同键通过原子占用保证只执行一次；持久化实体、幂等结果与当前证据引用须一致更新。不同上传/纠偏按会话受控顺序重聚合，迟到推理不能覆盖较新的结果。

## 四、端点与实施阶段

| 端点 | 阶段 | 设计职责 |
| :--- | :--- | :--- |
| `POST /api/v1/sessions` | M0 | 创建会话、返回匿名凭证 |
| `POST /api/v1/sessions/{session_id}/images` | M0 单图；C0 多图 | 接收一张图、调用 Runtime、更新完整证据与决策 |
| `GET /api/v1/sessions/{session_id}` | M0 | 查询会话、图片、完整观察值、当前 Evidence / Decision |
| `POST /api/v1/sessions/{session_id}/correction` | C0 后续 | 字段级纠偏立即更新业务；训练审核独立 |
| `GET /api/v1/models/latest` | C3 预留 | 查询授权、适配目标平台的 released 包元数据，不表示线上 active |

### 1. 创建会话

可选请求字段：`client_type` 为 web / mobile / wx，默认 web；`notes` 为可选备注，不应包含不必要个人信息。

以下仅为创建响应结构示例，令牌不是可用秘密：

```json
{
  "code": 200,
  "message": "success",
  "request_id": "req_example_create",
  "data": {
    "session_id": "sess_example",
    "session_token": "st_example_not_a_real_token",
    "created_at": "2025-01-01T12:00:00+08:00",
    "status": "AWAITING_INPUT"
  }
}
```

### 2. 上传图片

表单 `file` 必需；仅接收可实际解码的 JPEG / PNG，候选上限 15 MB，即 15,000,000 字节。`image_role` 可选 macro_date / packaging_full / barcode；`auxiliary_barcode` 可选 13 位字符串，仅作辅助输入，不证明条码感知已实现。

必须校验媒体类型、内容、大小、解码尺寸与可配置像素/资源上限，隔离存储键并防止路径穿越。前置质量拒绝应提示补拍；质量阈值须经真实样本验证，不以 README 中的示例分数代替。失败文件不得进入有效推理。

成功 data 固定包含 `image_id`、完整 `observation`、完整 `evidence`、完整 `decision`。三对象采用 DATA 的候选 Schema；真实无检出可返回合法空 detections，日期证据不足产生 UNDETERMINED，而 Runtime 故障不能伪造成功空对象。

M0 一个会话只接收一张图；新键追加第二张返回 409，同键重试原上传仍返回原结果。C0 多图实施后，每次追加重新聚合会话证据。M0 候选同步返回最终结果，不承诺已存在异步队列或进度推送；后续异步协议需单独定义。

### 3. 查询会话

成功 data 为 DATA 定义的 RecognitionSession 视图：`session_id`、`created_at`、`status`、`images`、`observations`、`evidence`、`decision`。未形成结果时 Evidence / Decision 为 null，列表为空；一旦存在对象必须是完整 DTO，不能省必需字段当作摘要。

图片列表只提供受控 asset_url 或 null，不暴露物理存储路径。授权和有效期必须覆盖原图与后续局部资源；校正图、增强图、OCR 输入图等变体尚未在 v1 定义，客户端不得假装已有对应字段。

GET 返回已存结果，不以查询当天时钟静默重算历史 Decision。M0 的普通重新查验可创建新会话；后续显式重算接口需另定契约。会话状态保留 DATA 八枚举；M0 采用 AWAITING_INPUT、PROCESSING、COMPLETED、NEEDS_REVIEW、FAILED、EXPIRED 子集。信息缺失、冲突或低置信走 NEEDS_REVIEW，技术失败走 FAILED，不能混同。

### 4. 字段级纠偏

请求直接采用 `correction.v1` Schema；允许 mfg_date、exp_date、shelf_life、storage_condition。日期为完整合法 ISO 日期，保质期保留 value/unit；不允许未知 corrected_field、null 修正值或直接覆盖 Observation。

```json
{
  "schema_version": "correction.v1",
  "image_id": "img_example",
  "corrected_field": "shelf_life",
  "corrected_value": {"value": 12, "unit": "MONTH"},
  "correction_reason": "人工核对包装保质期说明"
}
```

成功 data 包含 `correction_id`、`decision_effect: APPLIED_TO_SESSION`、`training_review_status: PENDING_REVIEW`、完整新 `evidence` 与 `decision`。只有修正存档及新证据/决策一致提交成功，才返回业务已应用；失败不显示成功。

合法修正形成 MANUAL 事实并按版本化策略处理原机器候选，保留采纳依据和新旧引用，不能无记录“最后写入胜出”。框与原 OCR 文字修正保留 C0 需求，但当前 correction.v1 未定义，须补充/升级契约和测试后接入。

InferPlatform 收集待审核记录；TrainPlatform 独立审核来源、许可、质量、标注及分组，通过后才取得训练候选资格。PENDING_REVIEW 不等于已归档为训练候选，也不触发训练或模型发布。

### 5. 端侧模型元数据

C3 候选查询参数 `target_platform` 用于选择兼容发布产物。返回 MODEL 中的 model_version、bundle_version、model_status、完整包与 manifest 哈希、兼容信息及受控文件下载清单；下载后核验完整包与文件哈希。模型文件、字典、标签和预后处理必须成套，不能只下载两个 ONNX。

latest 只表示满足授权和平台约束的 released 版本，不等于 Infer 的 active；无可用版本返回 404。设备支持、量化收益及端侧运行仍待验证，不能因元数据存在就宣称已适配。端侧候选 Observation 提交和端云分流未在本协议定义。

## 五、内部 Runtime 候选边界

内部请求采用受控调用，不复用客户端匿名令牌，部署网络只允许业务层；同进程实现可使用等价的函数边界，不强制 HTTP 微服务。

候选端点 `POST /internal/v1/inference/recognize`：请求含 `request_id`、`image_id`、受控 `image_uri`、明确 `model_version`；响应含 `request_id` 和完整 `observation.v1`。image_id、原图尺寸及实际加载版本必须一致，客户端不能提供任意 image_uri 触发内部取址。

Runtime 校验包、哈希、兼容性及张量，执行检测与局部 OCR；业务层验证 DTO、来源和坐标后生成 Evidence / Decision。超时、不可用、加载失败、非法输出均传播为明确错误；是否可重试取决于同一任务的幂等和运行状态，不能盲目重复推理。健康检查、超时/取消、内部授权及错误传播须在实现与测试中验证，当前不证明已存在运行时。

## 六、错误语义与生命周期

| HTTP | code | 标识或语义 |
| :--- | :--- | :--- |
| 400 | 400 / 4001 | 请求结构无效 / IMAGE_INVALID |
| 422 | 4002 / 4003 | IMAGE_BLURRY / IMAGE_OVEREXPOSED，候选质量门禁明确拒绝 |
| 401 / 403 | 401 / 403 | 凭证无效 / 资源越权 |
| 404 | 4004 或 404 | SESSION_NOT_FOUND / 其他资源不存在 |
| 409 | 409 | 幂等内容不一致、仍在处理、状态冲突或 M0 图片数量限制 |
| 410 | 410 | 验证归属后的会话或操作过期 |
| 413 / 429 | 413 / 429 | 文件过大 / 请求限流；限流可提供 Retry-After |
| 504 | 5001 | INFERENCE_TIMEOUT |
| 503 | 5002 | INFERENCE_UNAVAILABLE，包含尚无可用 active 模型 |
| 502 | 5003 | INFERENCE_OUTPUT_INVALID 或版本/结构不合法 |
| 500 | 500 | 未预期服务异常，保留 request_id |

证据冲突是正常业务结果：合法推理后返回 HTTP 200，Decision 为 UNDETERMINED / EVIDENCE_CONFLICT，不用旧 4005 错误中断证据展示。一次失败不生成冒充成功的 Observation/Evidence/Decision；既有成功历史保留，不用旧结果伪装本次成功。

会话到期、图片保留、审计保留与删除保护须配置、记录并测试；不能静默删除仍被纠偏审核、冻结版本或历史重放引用的资产。访问控制、TTL、幂等响应保存和资源清理尚未实现，文档约束不等于已落实安全能力。

## 七、迁移与冻结检查

旧候选的 session_decision、updated_decision、accumulated_evidences、evidence_chain、source、shelf_life_days 和 date-rules-v1 不再作为新 DTO 别名；采用四份版本化 Schema 及 date-rules.v1。当前没有已验证生产消费者，不能虚构兼容保证；实施仓固定引用契约版本并记录升级影响。

冻结前至少核验：OpenAPI及Schema引用、每种响应/错误、同键重试及并发、跨会话访问、令牌泄露与过期、解码/像素资源限制、真实 Runtime 输出、Evidence JCS 完整性、纠偏原子更新、日期边界/历史重放、活动模型版本和端到端样例。结构校验通过不等于这些实现测试已经通过。
