# 《轻眸识刻》核心数据结构与实体模型

> **状态**：全部 DTO、JSON Schema 与日期规则均为候选契约，不代表当前实现、联调或验收完成。
>
> **冻结门槛**：真实 Runtime / InferPlatform / 客户端实现、契约测试、日期规则测试及真实模型输入输出与交付兼容性测试全部通过，并保留版本与审核记录后，才能冻结对应版本。仅编写文档、Schema 或通过示例结构校验不构成冻结。
>
> **适用范围**：离线样本资产、在线会话、Runtime 观察值与 InferPlatform 业务证据和决策。

## 一、事实主线与契约索引

`Image -> Runtime -> Observation -> InferPlatform 业务解析与聚合 -> Evidence -> 日期规则 -> Decision`。

- Runtime 只生成原始 `Observation`，不生成业务 `Evidence`、`Decision` 或食品安全结论。
- 原始图像与机器观察值不可被人工修正、业务解析或历史重算覆盖。重复推理产生新 `observation_id`，保留旧观察值及其真实模型版本。
- 人工修正保存独立 `Correction`，业务层新增 `MANUAL` 事实，形成新的不可变 `Evidence` 和 `Decision`；旧事实、来源与历史决策保持可追溯。
- 派生到期日是决策中的日期投影，不是 OCR 观察值，也不伪造 MODEL 来源事实。
- 所有状态只表达包装标示期限，不推断食品可食用性、腐败情况或存储条件已实际满足。

| 候选契约 | 版本 | 机器结构定义 |
| :--- | :--- | :--- |
| Observation | `observation.v1` | [observation.v1.schema.json](schemas/observation.v1.schema.json) |
| Evidence | `evidence.v1` | [evidence.v1.schema.json](schemas/evidence.v1.schema.json) |
| Decision | `decision.v1` | [decision.v1.schema.json](schemas/decision.v1.schema.json) |
| Correction 请求 | `correction.v1` | [correction.v1.schema.json](schemas/correction.v1.schema.json) |
| 日期规则 | `date-rules.v1` | [DATE_RULES.md](DATE_RULES.md) |

相关边界依据：[架构](../ARCHITECTURE.md)、[需求池](../REQUIREMENTS.md)、[数据质量](../standards/DATA_QUALITY.md)。[API.md](API.md) 负责端点与响应封装，[MODEL.md](MODEL.md) 负责模型包与加载契约；这些文档须由各自负责人对齐本候选字段，旧示例别名不构成 v1 的额外兼容能力。

### 1. 通用结构约束

四份 Schema 使用 JSON Schema Draft 2020-12，均为新增契约而非运行代码；每个有固定字段的对象均明确 `additionalProperties: false`。`required` 字段必须出现，允许 `null` 不等于允许省略；数组允许为空，除非明确要求非空。未知字段、旧 `evidence_chain` / `calculated_days` 等别名不能静默接入 v1。

- ID 与版本标识为非空字符串；ID 唯一性、实体存在性及会话归属还须业务校验。
- 日期为真实公历完整 `YYYY-MM-DD`，范围 `0001-01-01` 至 `9999-12-31`；正则外形不能代替日历有效性校验。
- 时间为带显式偏移的 ISO 8601 / RFC 3339 字符串，例如 `2025-05-18T09:00:00+08:00`；`Z` 表示已知 UTC 偏移。无偏移时间与未知偏移 `-00:00` 不可作为决策时钟。
- 验证器必须开启 `date` / `date-time` 的 format assertion；Draft 2020-12 默认仅注解 format 的验证器不能宣称已验证日期合法。若用 Python jsonschema 做独立校验，须具备 `jsonschema[format]` 所需依赖并确认 FormatChecker 已注册这两种格式；仅构造 FormatChecker 不保证可用。日期、时间和哈希字符串不接受尾随换行或空白。
- 数值必须为有限数；拒绝 NaN、Infinity、-Infinity 以及非标准 JSON 常量。所有置信度为 `0..1` 或字段定义允许的 `null`，不能把“未计算”填成 0 或 1。
- 语义校验覆盖 Schema 无法表达的跨字段、跨实体和跨版本约束。结构通过不等于事实可靠、引用正确或哈希验证成功。

## 二、离线 SampleAsset

`SampleAsset` 是离线样本清单与受控资产管理记录，一行对应一个真实包装样本，**不等同于在线 `Image`**。保留来源、使用许可、原图审阅、标注与审核、数据划分和同源分组记录；未经审核的在线图片或纠偏记录不能自动成为训练标签或基准样本。

| 字段或分组 | 候选含义与审核要求 |
| :--- | :--- |
| `id`, `file_name`, `sha256`, `phash` | 真实资产身份、文件名、真实文件哈希与感知去重信息 |
| `source` | 来源类别、采集时间、来源地址或采集记录引用；不编造实拍来源 |
| `license` | 使用许可状态与许可依据引用；许可未知不能视为已授权 |
| `annotation` | 标注版本、状态、审核人和审核时间；保留框、原始文字及 MFG / EXP / BEST_BEFORE 语义，未看清的值为 null |
| `text_raw`, `date_type`, `standard_date` | 肉眼可见文字、标示类型、审核后的标准日期；标准日期不是从缺失文字猜出的标签 |
| `shelf_life_raw`, `shelf_life` | 包装原文及 `{value: 正整数, unit: DAY / MONTH / YEAR}` 或 null；“12个月”保留 MONTH，不能写成固定 365 天 |
| `printing_tech`, `package_type`, `difficulty` | 真实工艺、载体与审核后的难度分层，不代表模型已识别这些属性 |
| `split`, `group_key` | 划分及同产品、同包装实例、同次采集等同源分组键；同 group_key 不可跨训练、验证、测试集合泄漏 |
| `review`, `is_benchmark` | 审核结论及依据；只有真实质量、许可、标注和划分审核通过后才允许 `is_benchmark: true` |

以下仅为离线候选字段示意，尖括号值是占位，不是已采集样本、真实哈希或已批准许可；本次不新增 SampleAsset Schema，不修改实际清单：

```json
{
  "id": "00001",
  "file_name": "00001.jpg",
  "sha256": "<真实文件SHA-256>",
  "phash": "<真实感知哈希>",
  "source": {"kind": "<采集来源>", "source_url": null, "acquired_at": null, "record_ref": "<来源记录>"},
  "license": {"status": "PENDING_REVIEW", "record_ref": null},
  "annotation": {"version": "<标注版本>", "status": "PENDING_REVIEW", "reviewer": null, "reviewed_at": null},
  "text_raw": null,
  "date_type": null,
  "standard_date": null,
  "shelf_life_raw": null,
  "shelf_life": null,
  "printing_tech": null,
  "package_type": null,
  "difficulty": null,
  "split": null,
  "group_key": "<真实同源分组键>",
  "review": {"status": "PENDING_REVIEW", "record_ref": null},
  "is_benchmark": false
}
```

工艺枚举保留：`inkjet_dot_matrix`、`inkjet`、`laser_engraving`、`emboss`、`label_thermal`、`print`。载体枚举保留：`plastic_bag`、`can`、`bottle`、`bottle_cap`、`box`、`tetra_pak`、`plastic_tray`。其中 `bottle_cap` 对应瓶盖或封口塞。

## 三、Observation 候选契约

由 Runtime 生成，完整保存本次前向计算与后处理得到的原始观察值。

| 必需字段 | 类型与约束 |
| :--- | :--- |
| `schema_version` | 固定 `observation.v1` |
| `observation_id` | 非空字符串，本次观察值唯一 ID |
| `image_id` | 非空字符串，引用原始在线 Image |
| `model_version` | 非空真实模型包版本标识，不能以 `released` 等 model_status 代替版本 |
| `generated_at` | 带偏移时间，记录 Runtime 生成时刻 |
| `image_size` | 必需 `width`, `height`，均为正整数，表示原图像素宽高 |
| `quality_passed` | boolean 或 null；null 表示未提供质量判断，不等于质量通过 |
| `detections` | 检测项数组，可为空；空数组不代表存在日期事实 |

每个 `detections[]` 必须包含：

| 必需字段 | 类型与约束 |
| :--- | :--- |
| `box_id` | 非空字符串，同一 Observation 内唯一 |
| `category` | 固定 `date_area` |
| `bbox` | 恰好四个非负有限数 `[x1, y1, x2, y2]`，单位为原图像素，可含小数 |
| `bbox_format` | 固定 `xyxy` |
| `coordinate_space` | 固定 `original_image` |
| `confidence` | 检测置信度，有限数 `0..1` |
| `transcription` | 字符串或 null，保留原始 OCR 输出 |
| `text_confidence` | OCR 置信度，有限数 `0..1` 或 null |

必须语义校验 `0 <= x1 < x2 <= width` 且 `0 <= y1 < y2 <= height`；此处采用像素边界坐标，右、下边界可等于 width / height。Runtime 必须将缩放、letterbox、裁剪等坐标正确映射回原图，不能把模型输入坐标标记成 `original_image`。宽高、image_id 与实际受控原图一致，模型版本须可追溯到实际加载的模型包；不修改模型发布状态规则。

### 1. 明确可选的观察字段

以下字段均不在必需字段列表中，也不强制任何 C1 能力作为 C0 前置条件：

| 可选字段 | 提供时的候选结构 |
| :--- | :--- |
| `quality_metrics` | 对象；`blurriness_score` 为非负有限数、`overexposure_ratio` 为 `0..1`，两者均可省略，不宣称已实现质量阈值 |
| `global_texts` | 数组；每项必需 `transcription` 字符串、`confidence` 为 `0..1` 或 null；不宣称全局 OCR 已交付 |
| `barcode` | 对象；必需 `type`, `value` 非空字符串及 `confidence` 为 `0..1` 或 null；条码不能自动填补或覆盖包装日期 |
| `detections[].character_confidences` | 可选 `0..1` 有限数数组；字符对齐方式须依真实解码器及版本测试，不臆造逐字符置信度 |

可选字段未生成时省略，不擅自增加 Schema 未定义的指标。若提供对象或数组，必须满足对应子字段约束。

## 四、Evidence 候选契约

由 InferPlatform 业务层从一张或多张不可变 Observation 及合法 Correction 提取、聚合事实；不由 Runtime 直接生成。

| 必需字段 | 类型与约束 |
| :--- | :--- |
| `schema_version` | 固定 `evidence.v1` |
| `evidence_id`, `session_id` | 非空字符串，证据版本及会话引用 |
| `generated_at` | 带偏移生成时间 |
| `parser_version` | 非空解析版本；必须关联不可变解析规则、置信策略及配置清单，不能只记录一个不含策略映射的标签 |
| `facts` | 事实数组，可为空；`fact_id` 在此证据内唯一 |
| `conflicts` | 冲突数组，可为空 |
| `missing_fields` | 非空字符串的去重数组，可为空，描述聚合阶段发现的缺项 |
| `evidence_hash` | `sha256:` 加 64 位小写十六进制；算法见下节 |

每个 `facts[]` 必须包含以下全部字段，允许 null 的字段也不能省略：

| 必需字段 | 类型与约束 |
| :--- | :--- |
| `fact_id` | 非空字符串，真实来源事实标识 |
| `field` | `mfg_date` / `exp_date` / `shelf_life` / `storage_condition` |
| `value` | 日期字段为真实完整 ISO 日期字符串；保质期为必需 `{value: 正整数, unit: DAY / MONTH / YEAR}`；存储条件为非空且非纯空白文本 |
| `raw_text` | 原始证据文本或 null；不能用标准化结果覆盖 Observation 原文 |
| `source_type` | `MODEL` / `MANUAL` |
| `source_image_id` | 非空字符串，必须引用会话内真实 Image |
| `source_observation_id` | 非空字符串或 null；MODEL 必须非空并引用真实 Observation |
| `source_box_id` | 非空字符串或 null；非空时须定位到所引用 Observation 的真实检测框；全局文本可为 null |
| `source_correction_id` | 非空字符串或 null；MANUAL 必须非空并引用服务端存档的合法 Correction；MODEL 为 null |
| `confidence` | 有限数 `0..1` 或 null；MANUAL 固定 null，不能伪造为 1 |
| `eligible_for_decision` | boolean，由版本化解析与置信策略及合法人工修正共同决定，不由客户端自行授予可靠性 |

每个 `conflicts[]` 必需 `field` 为上述四枚举之一、`fact_ids` 为至少一个非空真实 fact_id 的去重数组、`reason` 为非空且非纯空白字符串。派生 EXP 与直接 EXP 不一致时，冲突引用真实 MFG、shelf_life、直接 EXP 的 fact_id，不造派生 Observation 或来源事实。

结构合法与 `eligible_for_decision: true` 不自动等价：来源必须真实、字段语义明确、解析合法，并按关联版本的置信策略评估。策略阈值尚属候选，**不得宣称已经校准**；未知或不合格必要候选不能冒充可靠证据。合法 MANUAL 可按明确的修正与冲突处置策略成为可靠事实，但不会抹去原 MODEL 候选或把机器置信度改成 1。历史策略及修正选择记录必须可追溯，不能无记录地“取最后一条”解决冲突。

MFG、EXP、BEST_BEFORE 的包装语义须保留于原始文字、Observation 与解析审核记录。当前 DTO 只有 `mfg_date` / `exp_date` 日期槽位，不新增虚构的语义字段；将 BEST_BEFORE 投影到 exp_date 只用于标示日期比较，不能把它改称食品安全截止日期。无类型的单日期串不能默认当成生产日期。

### 1. Evidence 哈希与不可变性

候选算法严格定义为：

1. 取得完整 Evidence 对象，仅移除顶层 `evidence_hash` 字段，不移除 generated_at、parser_version、来源、eligible、冲突或缺项，也不额外排序数组。
2. 按 [RFC 8785 JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785) 对这个对象规范化，得到标准 JCS UTF-8 字节。
3. 计算这些字节的 SHA-256，输出 `sha256:` + 64 位小写十六进制。

标准 JCS 包含规定的属性排序、字符串处理和 ECMAScript 数字序列化；`json.dumps(sort_keys=True)` 不是 JCS。必须拒绝重复对象键、非法 Unicode、非有限数字及不满足 JCS / I-JSON 表达要求的数据；实现须验证合规数字表达与测试向量，不能把任意精度数值静默改写成不同的证据。JCS 不做 Unicode 归一化，也不保证不同数组顺序得到相同哈希。

生成时计算并存档完整不可变 Evidence；使用、重放时重新计算并核对。Schema 只能检验哈希字符串外形，不能证明哈希真实有效。任何事实、来源、资格、冲突、时间或策略版本的变化都生成新的 evidence_id / evidence_hash，不覆盖历史。哈希是完整性校验，不替代授权或真实性审核。

本文件不提供伪造的完整 Evidence 哈希示例。第八节 Evidence / Decision JSON 是**缺少哈希的结构片段**，不是可通过整体 Schema 和完整性校验的对象；只有实际 JCS 实现生成并核验哈希后，才能构成完整证据与决策。

## 五、Decision 候选契约

由 InferPlatform 业务层依据所引用的完整 Evidence、固定规则及服务端原观察时刻生成。

| 必需字段 | 类型与约束 |
| :--- | :--- |
| `schema_version` | 固定 `decision.v1` |
| `decision_id`, `session_id` | 非空字符串 |
| `marked_status` | `WITHIN_MARKED_PERIOD` / `NEAR_EXPIRY` / `PAST_MARKED_PERIOD` / `UNDETERMINED` |
| `review_status` | `EVIDENCE_COMPLETE` / `INFORMATION_MISSING` / `EVIDENCE_CONFLICT` / `LOW_CONFIDENCE` |
| `observed_at` | 服务端产生、带偏移的业务观察时刻；不是图像 EXIF 时间或客户端本地时间 |
| `generated_at` | 带偏移的本次决策生成时刻；不能替代 observed_at |
| `timezone` | 非空有效 IANA 时区名，候选默认 `Asia/Shanghai` |
| `calculation_rule_version` | 固定 `date-rules.v1` |
| `evidence_schema_version` | 固定 `evidence.v1` |
| `evidence_id`, `evidence_hash` | 引用完整不可变 Evidence；hash 与所引用 Evidence 一致且重新校验有效 |
| `evidence_snapshot` | 选择后的日期输入与派生投影，子字段见下表 |
| `used_fact_ids` | 引用该 Evidence 的真实事实 ID 的去重数组，可为空；派生 EXP 必须包含真实 MFG 与 shelf_life fact_id |
| `missing_fields` | 本次日期判定所需缺项的去重字符串数组，可为空；不机械复制与本次结果无关的全部聚合缺项 |
| `warnings` | 字符串数组，可为空，保留无效日期、开封或存储等提示，不自动改日期 |
| `guidance` | 字符串或 null，提供补拍、补录或人工复核建议，不能断言食品安全 |

`evidence_snapshot` 的四个必需字段：

| 必需字段 | 类型与约束 |
| :--- | :--- |
| `mfg_date` | 完整真实日期字符串或 null |
| `shelf_life` | `{value: 正整数, unit: DAY / MONTH / YEAR}` 或 null |
| `exp_date` | 选中的直接到期日或按规则派生的完整日期字符串，未选中则 null |
| `exp_date_is_derived` | boolean 或 null；直接 EXP 为 false，派生 EXP 为 true，无 exp_date 时为 null |

snapshot 不包含完整事实、来源、低置信候选或冲突，**不能替代完整 Evidence 进行重放**。派生标志为 true 时必须有真实 mfg_date 与 shelf_life 输入，used_fact_ids 指向真实来源事实。存在未解决必要冲突时不能把任意一条冲突事实表现成已可靠选定的日期，未选定投影用 null，原因留在完整 Evidence 与 warnings 中。

### 1. 二维状态语义

- `WITHIN_MARKED_PERIOD`：可靠输入下既未超期，也未触发临期条件。
- `NEAR_EXPIRY`：可靠输入下 remaining_days <= 30，或 remaining_days * 10 <= total_lifetime_days；到期当天仍属此状态。
- `PAST_MARKED_PERIOD`：可靠输入下 remaining_days < 0，即到期日次日起标示超期。
- `UNDETERMINED`：必要证据不可靠、无效、冲突或不足；不是“默认在期内”。
- `EVIDENCE_COMPLETE`：足以支持本次日期判定，不要求所有包装字段齐全；可靠 EXP-only 可判定已超期或 30 天内临期。
- `INFORMATION_MISSING`：缺少本次必要输入；EXP-only 且 remaining_days > 30 时必须缺 `mfg_date`，不能默认在期内。
- `EVIDENCE_CONFLICT`：本次必要日期证据存在未解决冲突或时间一致性错误。
- `LOW_CONFIDENCE`：必要候选只有不可靠输入；不能凭最高分或人工伪造分数给出确定标示状态。

复核优先级为冲突 > 低置信 > 缺失 > 完备，只检查影响本次日期判定的必要事实。非必要存储条件缺项或冲突只产生提示；完整处理顺序、月末规则、比例判定与候选测试向量见 [DATE_RULES.md](DATE_RULES.md)。

## 六、Correction 候选契约与最小实体

### 1. 字段级修正请求

| 必需字段 | 类型与约束 |
| :--- | :--- |
| `schema_version` | 固定 `correction.v1` |
| `image_id` | 非空字符串，须属于目标会话 |
| `corrected_field` | 仅 `mfg_date` / `exp_date` / `shelf_life` / `storage_condition` |
| `corrected_value` | 与对应 Evidence fact.value 类型一致：日期字符串、value/unit 对象或非空非纯空白存储文本；不允许 null |
| `correction_reason` | 非空且非纯空白字符串 |

服务端先验证会话权限、图片归属、字段值合法性与修正理由，再分配 correction_id 并不可变存档，新增 MANUAL fact，按版本化策略重新聚合并生成新 Evidence / Decision。修正的采纳与冲突处置须有记录；请求成功不表示自动成为训练标签、进入基准集或更新线上模型。

原始 OCR 文字修正与检测框修正仍保留在 `FR-C0-006` 的 C0 规划中，**当前 correction.v1 没有定义这些字段或请求形态**，不得宣称已支持；未来必须升级或补充契约及对应测试后接入。禁止直接覆盖 Observation.transcription / bbox。

### 2. 在线 Image 最小字段表

以下是供 API 负责人映射的候选实体，不在四份 Schema 中额外扩展请求或创建第五份 Schema。

| 必需字段 | 类型与候选语义 |
| :--- | :--- |
| `image_id`, `session_id` | 非空字符串，在线图片与归属会话 |
| `created_at` | 服务端带偏移创建时间 |
| `width`, `height` | 原图正整数宽高，与 Observation.image_size 一致 |
| `image_role` | `macro_date` / `packaging_full` / `barcode` 或 null；声明拍摄角色不证明已实现对应感知能力 |
| `asset_url` | 受控 HTTPS URL 或 null，须满足访问控制与有效期约束；不回传本地路径或公开永久存储地址 |

### 3. RecognitionSession 最小字段表

| 必需字段 | 类型与候选语义 |
| :--- | :--- |
| `session_id` | 非空字符串 |
| `created_at` | 服务端带偏移创建时间 |
| `status` | 下列完整八状态之一 |
| `images` | Image 数组，可为空 |
| `observations` | 完整 Observation 数组，可为空，不是仅含 bbox 的无版本摘要 |
| `evidence` | 当前完整 Evidence 或 null，历史版本另行不可变保留 |
| `decision` | 当前完整 Decision 或 null，历史版本另行不可变保留 |

创建成功响应在这个会话视图外额外提供 `session_token`，**仅在创建时返回一次**；GET 会话视图不回传令牌，不把令牌存入 Evidence / Decision。鉴权、有效期和响应封装由 API 契约负责；本表不宣称已有服务端实现。

### 4. Correction 存档最小字段表

| 必需字段 | 类型与候选语义 |
| :--- | :--- |
| `correction_id`, `session_id` | 服务端分配的非空修正 ID 与归属会话 |
| `created_at` | 服务端带偏移存档时间 |
| `schema_version`, `image_id`, `corrected_field`, `corrected_value`, `correction_reason` | 原样保留经过验证的 correction.v1 请求字段与合法类型 |

存档实体与请求 Schema 不同，不能把 correction_id / session_id / created_at 擅自加入当前请求。存档记录、原始 Observation、被修正事实及新旧证据版本之间须保留可审计关联，不改动原始观察值。

## 七、完整会话状态与 M0 候选子集

| 状态 | 候选语义 | M0 候选子集 |
| :--- | :--- | :--- |
| `AWAITING_INPUT` | 已创建，等待图片或有效输入 | 是 |
| `RECEIVING_IMAGES` | 显式接收图片中的中间状态 | 否 |
| `PROCESSING` | 推理、聚合或日期计算进行中 | 是 |
| `PARTIAL_RESULT` | 部分结果可用，尚有任务未结束 | 否 |
| `COMPLETED` | 本轮处理完成且无需必要证据复核 | 是 |
| `NEEDS_REVIEW` | 本轮处理完成但需补充或复核必要证据 | 是 |
| `FAILED` | 本轮技术处理失败，不表示食品标示超期 | 是 |
| `EXPIRED` | 会话已过有效期 | 否 |

完整枚举固定保留上述八种值；M0 此处仅提出 `AWAITING_INPUT`, `PROCESSING`, `COMPLETED`, `NEEDS_REVIEW`, `FAILED` 的最小候选映射，不声称已实现状态机。若 M0 API 实施会话到期策略，必须同步纳入 `EXPIRED` 并补状态转移测试，不能用 FAILED 或“不存在”静默替代到期状态。RECEIVING_IMAGES / PARTIAL_RESULT 不因存在于完整词表就成为同步原型的已支持能力；API、实现与测试冻结前须共同确认实际子集及转移规则。

## 八、内嵌结构样例与校验边界

以下合成数据只用于契约说明，不是模型实测、真实采集样本或已通过项目测试的结果。

### 1. Observation 最小完整结构样例

本块可作为 `observation.v1.schema.json` 的结构校验输入；数值在界内，不包含可选增强字段。合成模型版本和 image_id 仍需真实系统引用校验，不能据此证明 Runtime 已实现。

```json
{
  "schema_version": "observation.v1",
  "observation_id": "obs_example_001",
  "image_id": "img_example_001",
  "model_version": "det-rec-example.v1",
  "generated_at": "2024-05-18T09:00:00+08:00",
  "image_size": {"width": 640, "height": 480},
  "quality_passed": null,
  "detections": [
    {
      "box_id": "box_example_001",
      "category": "date_area",
      "bbox": [120, 240, 210, 310],
      "bbox_format": "xyxy",
      "coordinate_space": "original_image",
      "confidence": 0.935,
      "transcription": "MFG 20240518",
      "text_confidence": 0.912
    }
  ]
}
```

### 2. Correction 最小完整结构样例

以下两块分别可作为 `correction.v1.schema.json` 的结构校验输入；日期与保质期类型不可互换，人工修正没有伪造的 confidence 字段。

```json
{
  "schema_version": "correction.v1",
  "image_id": "img_example_001",
  "corrected_field": "mfg_date",
  "corrected_value": "2024-05-18",
  "correction_reason": "人工核对包装生产日期文字"
}
```

```json
{
  "schema_version": "correction.v1",
  "image_id": "img_example_001",
  "corrected_field": "shelf_life",
  "corrected_value": {"value": 12, "unit": "MONTH"},
  "correction_reason": "人工核对包装标示12个月"
}
```

### 3. Evidence 结构片段

本块刻意省略必需 `evidence_hash`，**不是完整可验证 Evidence**。fact 可单独用 `#/$defs/fact` 校验结构；合成引用与 eligible 值仅为规则输入示意，阈值没有因此获得校准。需要完整验证时，由标准 JCS 实现按第四节算法实际计算并填入哈希，不使用占位哈希。

```json
{
  "schema_version": "evidence.v1",
  "evidence_id": "ev_example_001",
  "session_id": "sess_example_001",
  "generated_at": "2024-05-18T09:00:01+08:00",
  "parser_version": "parser-policy-example.v1",
  "facts": [
    {
      "fact_id": "fact_example_mfg",
      "field": "mfg_date",
      "value": "2024-05-18",
      "raw_text": "MFG 20240518",
      "source_type": "MODEL",
      "source_image_id": "img_example_001",
      "source_observation_id": "obs_example_001",
      "source_box_id": "box_example_001",
      "source_correction_id": null,
      "confidence": 0.912,
      "eligible_for_decision": true
    },
    {
      "fact_id": "fact_example_life",
      "field": "shelf_life",
      "value": {"value": 12, "unit": "MONTH"},
      "raw_text": null,
      "source_type": "MANUAL",
      "source_image_id": "img_example_001",
      "source_observation_id": null,
      "source_box_id": null,
      "source_correction_id": "corr_example_001",
      "confidence": null,
      "eligible_for_decision": true
    }
  ],
  "conflicts": [],
  "missing_fields": []
}
```

### 4. Decision 结构片段

本块同样刻意省略必需 `evidence_hash`，**不是完整可验证 Decision**。snapshot 子对象可以单独按对应 Schema 属性校验；最终 hash 必须等于所引用完整 Evidence 的实际 JCS 哈希。派生到期日为 `2025-05-18`，不是减去一天的 `2025-05-17`。

```json
{
  "schema_version": "decision.v1",
  "decision_id": "dec_example_001",
  "session_id": "sess_example_001",
  "marked_status": "WITHIN_MARKED_PERIOD",
  "review_status": "EVIDENCE_COMPLETE",
  "observed_at": "2024-05-18T09:00:00+08:00",
  "generated_at": "2024-05-18T09:00:02+08:00",
  "timezone": "Asia/Shanghai",
  "calculation_rule_version": "date-rules.v1",
  "evidence_schema_version": "evidence.v1",
  "evidence_id": "ev_example_001",
  "evidence_snapshot": {
    "mfg_date": "2024-05-18",
    "shelf_life": {"value": 12, "unit": "MONTH"},
    "exp_date": "2025-05-18",
    "exp_date_is_derived": true
  },
  "used_fact_ids": ["fact_example_mfg", "fact_example_life"],
  "missing_fields": [],
  "warnings": [],
  "guidance": null
}
```

## 九、冻结前的真实校验清单

1. 四份 Schema 的 Draft 2020-12 自检，开启格式校验的正负结构用例；required、未知字段、类型联动及 nullable 行为与文档一致。
2. Runtime 使用真实图像及真实模型包验证原图映射、退化框与边界、非有限数、空检测、实际版本来源；不以合成框宣称检测或 OCR 效果。
3. InferPlatform 验证 MODEL / MANUAL 的来源、会话归属、合法修正、唯一 fact_id、冲突与 used_fact_ids 引用、版本化可靠性策略；置信阈值须有真实评测证据。
4. 使用标准 JCS 实现和 RFC 测试向量验证数字、Unicode、属性排序及哈希；禁止拿字符串外形或 sort_keys 代替完整性验证。
5. 用真实日期引擎运行 DATE_RULES 全部候选边界向量、解析格式正负测试、时区和历史重放测试；核对完整 Evidence、hash、snapshot、used_fact_ids 及二维状态一致。
6. API 与客户端实际联调验证会话实体、八状态与 M0 子集、仅创建返回令牌、受控图片访问及修正生命周期；真实模型输入输出与模型包兼容性测试通过后共同冻结，不凭文档宣称已经支持。
