# 轻眸识刻在线 API 候选契约

> **状态**：`api.v2-candidate`，由 InferPlatform 实现后提供；当前 M0 待实现，候选契约不代表服务或联调已完成。
> **事实源**：[DATA.md](DATA.md)、[日期推导规则.md](日期推导规则.md)、[MODEL.md](MODEL.md)、[系统架构.md](../系统架构.md)。
> **机器定义**：[openapi.json](openapi.json)，OpenAPI 3.1（引用四份候选 JSON Schema）。

## 一、范围与基础约定

| 项目 | 候选约定 |
| :--- | :--- |
| 客户端接入 | WebClient 优先；MobileClient、WxClient 为后续规划，统一访问 InferPlatform 业务 API |
| 传输协议 | 生产使用 HTTPS；本地开发使用 HTTP；字符集 UTF-8 |
| 请求格式 | JSON；图片上传使用 multipart/form-data |
| 时间格式 | 带明确时区偏移的 RFC 3339；日期计算使用服务端 observed_at，不以客户端或 EXIF 时钟替代 |
| 成功响应 | HTTP 200，响应结构 `{code: 200, message, data, request_id}` |
| 失败响应 | HTTP 4xx/5xx，响应结构 `{code, message, data: null, request_id}`；不回传内部敏感信息 |
| 数据传输对象 | 包含完整的 observation.v1 / evidence.v1 / decision.v2 安全 DTO |
| 架构分层 | 业务层负责编排、鉴权、Evidence 和 Decision；内部 Runtime 仅生成 Observation |
| 缓存控制 | 创建响应及其重放、GET 会话与受控图片资源使用 `Cache-Control: no-store` |

完整字段、可空性、坐标与 Evidence JCS 哈希规则见 [DATA.md](DATA.md) 及对应 Schema；`required` 字段允许 null 仍须出现。MODEL / MANUAL 来源与冲突 fact_ids 须引用本会话真实记录；`eligible_for_decision` 按版本化策略判定，Decision 的 `used_fact_ids` 仅引用对应 Evidence 内可采纳的真实事实，必要事实的未解决冲突不得忽略。结构示例不作为训练数据或实测指标。

## 二、匿名凭证与资源归属

1. 创建会话无需已有令牌，返回服务端签发的不可预测、至少 128 位随机熵的短期不透明 `session_token`；不得由 session_id 推导，无需登录或 SSO。
2. 后续会话查询、上传和纠偏携带 `Authorization: Bearer <session_token>`。
3. 会话、图片、Observation、Evidence、Decision 和 Correction 绑定创建时的匿名主体；路径与正文中的 image_id 须校验归属。
4. 令牌有效期不晚于会话有效期；会话 TTL、图片与审计保留期、幂等重试窗口和清理策略由版本化配置显式定义，关闭浏览器不代表服务端已删除。
5. 缺少、伪造或单独失效凭证返回 401；有效主体跨归属访问返回 403；能验证归属且会话已过期时返回 410，无法验证归属时不披露资源过期状态。
6. 创建成功及同键同请求的合法重放经 HTTPS 返回原令牌；GET、证据、日志、图片 URL 与分享卡片不返回或记录令牌。
7. 客户端安全存储凭证，不置于 URL 或日志；丢失凭证不重新签发，须重建会话。
8. `/models/latest` 为 C3 预留，须另行配置读取授权；会话令牌不自动获得模型读取、下载或管理权限。

## 三、请求幂等与并发

创建、上传、纠偏均须携带 `Idempotency-Key`。新操作生成至少 128 位随机熵的键；网络重试复用原键，不重新生成。

| 情形 | 候选处理 |
| :--- | :--- |
| 同键、同请求 | 返回首次处理的原结果与原实体 ID，不重复创建、推理或纠偏，保留原 Decision 的 observed_at、generated_at |
| 同键、不同请求 | 返回 HTTP 409，不复用原结果，亦不执行新请求 |
| 原请求仍在处理 | 返回 HTTP 409 并提供 Retry-After，复用原键稍后重试 |
| 创建会话操作 | 键作用域为创建端点；键也承载领取原令牌的能力，须保密、限流并校验请求指纹 |
| 已有会话写入 | 键作用域限定为已鉴权主体 + session_id + 端点；幂等键不替代 Bearer 授权 |
| 文件请求指纹 | 绑定规范化业务字段；上传另绑定实际文件字节 SHA-256、image_role、auxiliary_barcode，同键不可替换内容 |
| 记录有效期 | 幂等记录至少覆盖会话有效期与约定重试窗口；过期返回明确错误，旧键不当作新操作或自动重建会话 |

并发同键通过原子占用确保仅执行一次，实体、幂等结果与当前证据引用一致提交；创建结果受控保存，凭证不进入普通缓存或明文日志。不同上传/纠偏按会话顺序处理，迟到推理不得覆盖新结果；超时重试须确认原执行状态，不能在旧推理仍运行时盲目重启。

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

表单字段 `file` 必需；接收可实际解码的 JPEG / PNG，文件上限 15 MB（15000000 字节）。可选 `image_role`（macro_date / packaging_full / barcode）与 `auxiliary_barcode`（13 位数字字符串，仅作辅助输入）。

校验媒体类型与内容一致性、大小、解码宽高及配置明确的像素与处理资源上限，使用隔离存储键并防止路径穿越；质量拒绝提示补拍，阈值须经真实样本校准。成功 data 必需包含 `image_id`、完整 `observation`、`evidence` 与 `decision`。真实无检出返回合法空 detections，日期证据不足产生 UNDETERMINED；Runtime 故障、超时或非法输出按错误表返回，不伪造成功空对象。

M0 单会话仅接收单张图片：新键追加第二张返回 409，原上传同键重放仍返回原成功结果，先识别合法重放再检查图片数量。M0 候选同步返回最终结果；后续异步协议另行定义，不强制异步队列或进度推送。

### 3. 查询会话

成功 data 返回 RecognitionSession 完整 DTO：`session_id`、`created_at`、`status`、`images`、`observations`、`evidence`、`decision`。未产出结果时列表为空，evidence / decision 为 null；存在的对象均遵循 [DATA.md](DATA.md)，不得省略必需字段或用 Decision 的日期投影替代完整 Evidence。

图片列表保留 DATA 定义的完整 Image 字段，`asset_url` 为受控地址或 null，不暴露存储路径；授权与有效期覆盖原图及后续局部资源，图像变体尚未在 v1 定义。GET 不回传令牌，返回已存结果，不以查询时钟静默重算历史决策；普通重新查验可新建会话，显式重算接口另定契约。

会话保留 DATA 的完整八状态；M0 使用 AWAITING_INPUT、PROCESSING、COMPLETED、NEEDS_REVIEW、FAILED、EXPIRED，RECEIVING_IMAGES、PARTIAL_RESULT 留待后续。必要日期事实缺失、冲突或低置信时进入 NEEDS_REVIEW，非必要存储条件问题只产生提示，技术失败进入 FAILED。

### 4. 字段级纠偏

请求遵循 `correction.v1` Schema；支持 mfg_date（生产日期）、exp_date（直接标示到期日）、shelf_life（正整数 value 与 DAY / MONTH / YEAR unit）、storage_condition（非空存储说明）四类语义字段，以及 start_mode（起算模式纠偏：COUNT_FROM_NEXT_DAY / COUNT_FROM_MFG_DAY）。日期须为完整且日历合法的 YYYY-MM-DD；拒绝未知字段与 null 修正值。语义字段纠偏写入 Evidence.facts；start_mode 纠偏派生新 CalculationProfile 触发 Decision 重算。

```json
{
  "schema_version": "correction.v1",
  "image_id": "img_example",
  "corrected_field": "shelf_life",
  "corrected_value": {"value": 12, "unit": "MONTH"},
  "correction_reason": "人工核对包装保质期说明"
}
```

成功 data 包含 `correction_id`、`decision_effect: APPLIED_TO_SESSION`、`training_review_status: PENDING_REVIEW`、完整新版 `evidence` 与 `decision`。合法修正新增 MANUAL 事实：confidence 为 null，source_correction_id 引用合法存档；相关 Observation 与检测框可保留真实引用或为 null；生成新 evidence_id / decision_id，保留原图、Observation、原事实及旧 Evidence / Decision，并记录版本化采纳策略和新旧引用。修正存档、新证据/决策、会话当前引用、幂等结果及待审记录原子一致保存后，才返回已应用。

业务更新不等待训练审核；TrainPlatform 独立审核来源、许可、质量、标注与分组，通过后才获训练候选资格，不自动入训练或发布。原始 OCR 文字与 bbox 修正仍属 C0 需求，correction.v1 未定义，须升级契约和测试后接入。

### 5. 端侧模型元数据

C3 候选查询参数 `target_platform`，须独立读取授权。latest 仅表示获授权且兼容目标平台的 released 版本，不等于 InferPlatform 的 active；无可用版本返回 404。返回保留 model_version、bundle_version、model_status、manifest_sha256、bundle_sha256、target_platform、compatibility、files；文件项保留 path、size_bytes、sha256、download_url，清单覆盖 manifest、模型、字典、标签、预后处理配置、包含 Profile 的 metadata 及其余完整包文件。

下载后成套校验固定制品字节、manifest 原始字节及清单内全部文件；受控动态下载 URL 不参与包哈希。固定包字节的取得与下载机制，以及 compatibility 的 runtime、最低版本、opset、目标平台/精度 Profile、张量 shape 与轴含义，须在冻结前补齐机器定义并实测；端侧接入与收益另行验证。

## 五、内部 Runtime 候选边界

内部请求仅向业务层开放，不复用客户端匿名令牌；同进程可使用等价函数边界，分进程可采用以下受控候选端点，不暴露于公网：

端点 `POST /internal/v1/inference/recognize`：请求含 `request_id`、`image_id`、受控 `image_uri` 与明确 `model_version`；响应含原 `request_id` 与完整 `observation.v1`。image_id、实际加载版本、原图尺寸与物理像素坐标须一致，拒绝客户端任意 URI 取址。

Runtime 校验模型包、哈希、兼容性与张量，执行检测与 OCR；业务层校验 DTO、来源与坐标后派生 Evidence 与 Decision。内部授权、健康检查、超时/取消与错误传播须在实现中验证，加载失败不得转成空检测成功。

## 六、错误语义与生命周期

| HTTP 状态码 | 业务 code | 标识与语义 |
| :--- | :--- | :--- |
| 400 | 400 / 4001 | 请求结构无效 / IMAGE_INVALID |
| 422 | 4002 / 4003 | IMAGE_BLURRY / IMAGE_OVEREXPOSED（质量门禁拒绝） |
| 401 / 403 | 401 / 403 | 凭证无效 / 资源越权访问 |
| 404 | 4004 / 404 | SESSION_NOT_FOUND / 资源不存在 |
| 409 | 409 | 幂等冲突、正在处理或单图会话超量 |
| 410 | 410 | 验证归属后的会话或操作已过期 |
| 413 / 429 | 413 / 429 | 文件超限 / 请求限流 |
| 504 | 5001 | INFERENCE_TIMEOUT |
| 503 | 5002 | INFERENCE_UNAVAILABLE（无可用 active 模型或加载不可用） |
| 502 | 5003 | INFERENCE_OUTPUT_INVALID（结构、坐标或版本不合规） |
| 500 | 500 | 未预期服务异常 |

证据冲突属正常业务结果，返回 HTTP 200；必要日期冲突输出 UNDETERMINED / EVIDENCE_CONFLICT，非必要保存条件冲突仅提示，不用旧 4005 中断证据展示。技术错误保留类别、阶段和 request_id，不删除已有成功历史，也不以旧结果伪装本次成功；错误响应不泄露内部路径或敏感原文。清理前检查纠偏审核、冻结版本、模型血缘和历史重放引用，仍被引用的资产不得静默删除。

## 七、迁移与冻结检查

旧候选的 session_decision、updated_decision、accumulated_evidences、evidence_chain、source、shelf_life_days 和 date-rules-v1 不再作为新 DTO 别名，当前绑定 Observation/Evidence/Correction v1、Decision v2 与 date-rules.v2；实施仓固定契约版本并记录升级影响。

新候选统一返回 decision.v2，calculation_profile_id 由服务端保存并绑定实际证据、阈值、时区及起算依据；客户端不新增请求字段指定口径。路径仍为 /api/v1，路径编号与 DTO 版本分别管理；当前新协议定义 v2 新会话载荷。旧 decision.v1 与原规则保留用于显式历史适配和重放，读取适配协议在实现前另定，不能直接送入当前 v2 响应或同键重放成新版。固定天数临期默认候选30天；可靠直接 EXP 不因缺 MFG 阻断；起算未知时按双口径包络定理区分非临界安全区保守下界推导与临界窗口阻断。

冻结前核验项：OpenAPI 语法及 Schema 引用、完整响应与错误映射、幂等重试与并发、跨会话越权、凭据泄露/过期、解码限制、真实 Runtime 映射、Evidence JCS 哈希、纠偏原子更新、日期边界与历史重放、会话生命周期及实际模型版本。
