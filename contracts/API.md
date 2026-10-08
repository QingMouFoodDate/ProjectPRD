# 轻眸识刻在线 API 候选契约

> **状态**：`api.v1-candidate`，由 InferPlatform 实现后提供；当前 M0 待实现。
> **事实源**：[DATA](DATA.md)、[DATE_RULES](DATE_RULES.md)、[MODEL](MODEL.md)、[职责决议](../decisions/ADR-001-REPOSITORY_BOUNDARIES.md)。
> **机器定义**：[openapi.json](openapi.json)，OpenAPI 3.1（引用四份候选 JSON Schema）。

## 一、范围与基础约定

| 项目 | 候选约定 |
| :--- | :--- |
| 客户端接入 | WebClient 优先；MobileClient、WxClient 为后续规划，统一访问 InferPlatform 业务 API |
| 传输协议 | 生产使用 HTTPS；本地开发使用 HTTP；字符集 UTF-8 |
| 请求格式 | JSON；图片上传使用 multipart/form-data |
| 时间格式 | 带明确时区偏移的 RFC 3339；日期计算使用服务端 observed_at，不以客户端时钟替代 |
| 成功响应 | HTTP 200，响应结构 `{code: 200, message, data, request_id}` |
| 失败响应 | HTTP 4xx/5xx，响应结构 `{code, message, data: null, request_id}`；不回传内部敏感信息 |
| 数据传输对象 | 包含完整的 observation.v1 / evidence.v1 / decision.v1 安全 DTO |
| 架构分层 | 业务层负责编排、鉴权、Evidence 和 Decision；内部 Runtime 仅生成 Observation |

## 二、匿名凭证与资源归属

1. 创建会话返回服务端签发的短期不透明 `session_token`，令牌不得由 session_id 推导，无需登录或 SSO。
2. 后续会话查询、上传和纠偏携带 `Authorization: Bearer <session_token>`。
3. 会话、图片、Observation、Evidence、Decision 和 Correction 绑定创建时的匿名主体；路径与正文中的 image_id 须校验归属。
4. 令牌有效期不晚于会话有效期，TTL 与清理策略由配置显式定义。
5. 缺少或无效凭证返回 401；跨主体越权返回 403；会话过期返回 410。
6. 创建响应及其幂等重放返回同一令牌；GET、日志、图片 URL 与分享卡片严禁回传令牌。
7. 客户端安全存储凭证，不置于 URL 或日志；丢失凭证不重新签发，须重建会话。
8. `/models/latest` 为 C3 预留，须另行配置读取授权；会话令牌不自动获得模型管理权限。

## 三、请求幂等与并发

创建、上传、纠偏均须携带 `Idempotency-Key`。新操作生成至少 128 位随机熵的键；网络重试复用原键，不重新生成。

| 情形 | 候选处理 |
| :--- | :--- |
| 同键、同请求 | 返回首次处理的原结果与原实体 ID，不重复执行推理或纠偏 |
| 同键、不同请求 | 返回 HTTP 409，不复用原结果，亦不执行新请求 |
| 原请求仍在处理 | 返回 HTTP 409 并提供 Retry-After，复用原键稍后重试 |
| 创建会话操作 | 键作用域为创建操作，校验请求指纹以防重放 |
| 已有会话写入 | 键作用域限定为已鉴权主体 + session_id + 端点 |
| 文件请求指纹 | 校验上传文件实际内容 SHA-256 及辅助字段，同键不可替换文件 |
| 记录有效期 | 幂等记录覆盖会话有效期与约定重试窗口，过期操作返回明确提示 |

并发同键通过原子占用确保仅执行一次；不同上传/纠偏按会话顺序处理，迟到推理不得覆盖新结果。

## 四、端点与实施阶段

| 端点 | 阶段 | 设计职责 |
| :--- | :--- | :--- |
| `POST /api/v1/sessions` | M0 | 创建会话、签发匿名凭证 |
| `POST /api/v1/sessions/{session_id}/images` | M0 单图；C0 多图 | 接收图片、调用 Runtime、更新完整证据与决策 |
| `GET /api/v1/sessions/{session_id}` | M0 | 查询会话、图片、完整观察值、当前 Evidence 与 Decision |
| `POST /api/v1/sessions/{session_id}/correction` | C0 阶段 | 字段级纠偏并即时更新业务；离线审核独立 |
| `GET /api/v1/models/latest` | C3 预留 | 查询授权适配目标平台的 released 模型包元数据 |

### 1. 创建会话

可选请求字段：`client_type`（web / mobile / wx，默认 web）；`notes`（可选备注）。

响应示例：

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

表单字段 `file` 必需；接收可解码的 JPEG / PNG，文件上限 15 MB。可选 `image_role`（macro_date / packaging_full / barcode）与 `auxiliary_barcode`（13 位字符串）。

校验文件类型、大小、解码尺寸与像素上限。成功 data 包含 `image_id`、完整 `observation`、`evidence` 与 `decision`。真实无检出返回空 detections，日期证据不足产生 UNDETERMINED。

M0 单会话仅接收单张图片，追加第二张返回 409。

### 3. 查询会话

成功 data 返回 RecognitionSession 完整 DTO：`session_id`、`created_at`、`status`、`images`、`observations`、`evidence`、`decision`。未产出结果时对应字段为 null 或空数组，不得省略必需字段充当摘要。

图片列表提供受控 `asset_url`，不暴露存储路径。GET 返回已存结果，不以查询时钟静默重算历史决策。

### 4. 字段级纠偏

请求遵循 `correction.v1` Schema；支持 mfg_date、exp_date、shelf_life、storage_condition 四类字段。

```json
{
  "schema_version": "correction.v1",
  "image_id": "img_example",
  "corrected_field": "shelf_life",
  "corrected_value": {"value": 12, "unit": "MONTH"},
  "correction_reason": "人工核对包装保质期说明"
}
```

成功 data 包含 `correction_id`、`decision_effect: APPLIED_TO_SESSION`、`training_review_status: PENDING_REVIEW`、新版 `evidence` 与 `decision`。合法修正新增 MANUAL 事实并触发决策重算。

### 5. 端侧模型元数据

C3 候选查询参数 `target_platform`。返回 MODEL 中的 model_version、bundle_version、哈希、兼容信息及受控下载清单。下载后必须成套校验文件。

## 五、内部 Runtime 候选边界

内部请求采用受控通信，不暴露于公网：

端点 `POST /internal/v1/inference/recognize`：请求含 `request_id`、`image_id`、受控 `image_uri` 与 `model_version`；响应含完整 `observation.v1`。

Runtime 校验模型包、哈希与张量，执行检测与 OCR；业务层校验 DTO 后派生 Evidence 与 Decision。

## 六、错误语义与生命周期

| HTTP 状态码 | 业务 code | 标识与语义 |
| :--- | :--- | :--- |
| 400 | 400 / 4001 | 请求结构无效 / IMAGE_INVALID |
| 422 | 4002 / 4003 | IMAGE_BLURRY / IMAGE_OVEREXPOSED（质量门禁拒绝） |
| 401 / 403 | 401 / 403 | 凭证无效 / 资源越权访问 |
| 404 | 4004 / 404 | SESSION_NOT_FOUND / 资源不存在 |
| 409 | 409 | 幂等冲突、正在处理或单图会话超量 |
| 410 | 410 | 会话或操作已过期 |
| 413 / 429 | 413 / 429 | 文件超限 / 请求限流 |
| 504 | 5001 | INFERENCE_TIMEOUT |
| 503 | 5002 | INFERENCE_UNAVAILABLE（无可用模型） |
| 502 | 5003 | INFERENCE_OUTPUT_INVALID（输出不合规） |
| 500 | 500 | 未预期服务异常 |

证据冲突属正常业务结果：返回 HTTP 200，Decision 为 UNDETERMINED / EVIDENCE_CONFLICT。

## 七、迁移与冻结检查

旧候选字段不再作为新 DTO 别名，统一绑定四份版本化 Schema 与 date-rules.v1。

冻结前核验项：OpenAPI 语法、响应结构、幂等重试、并发控制、跨会话越权、凭据过期、解码限制、真实 Runtime 映射、Evidence JCS 哈希、纠偏原子更新与历史重放一致性。
