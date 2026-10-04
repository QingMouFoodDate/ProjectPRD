# 《轻眸识刻》包装标示日期规则

> **候选版本**：`date-rules.v1`，不是已实现或已验收的日期引擎。
>
> **依赖**：[DATA.md](DATA.md) 中的 `evidence.v1` 与 `decision.v1`。
>
> **冻结门槛**：真实实现及契约、规则、解析与模型输入输出测试通过，保存配置版本和审核依据后冻结；本文件测试向量仅为候选预期，不代表项目测试已通过。

## 一、判定对象与可靠输入

只比较包装标示日期，不推断食品安全、实际存储状态或开封后的可食用期限。原始 Observation 不因标准化、纠偏或日期推导而改变。

- 输入必须是完整不可变 Evidence，校验 schema_version、真实来源、会话归属、evidence_hash 及版本化解析 / 置信策略后才计算。
- 只有满足真实来源、合法字段语义、合法解析及对应版本可靠性策略的事实，才能 `eligible_for_decision: true`。此标志不能由客户端自填；候选置信阈值未宣称已经校准。
- MANUAL 事实必须来自合法存档 Correction，confidence 为 null，不模拟“模型置信度 1”。修正采纳须按版本化策略记录，新旧事实与 Evidence 不可静默覆盖。
- MFG 表示生产日期；EXP 和 BEST_BEFORE 的包装原始语义必须保留。BEST_BEFORE 可投影至 exp_date 做标示期限比较，但不能重述为食品安全保证或法定绝对截止时间。
- 只有一个无类型日期串时不能默认当成 MFG / EXP；必须有包装语义上下文或合法人工字段级修正。

## 二、观察时间与合法日期

1. 默认时区 `Asia/Shanghai`；Decision.timezone 写入实际使用的有效 IANA 时区名，时区选择与时区数据库版本必须可追溯。
2. `observed_at` 由服务端在本次业务观察时生成，为带显式偏移的 ISO 8601 / RFC 3339 时间；允许 Z，不允许无偏移或未知偏移 -00:00。偏移和目标时区不是同一个字段：先解释输入瞬时，再转换到 timezone。
3. 将该瞬时转换到 timezone 后取本地日期 `observed_date`；不是简单截取字符串前十位，也不采用客户端日期、EXIF 日期或重放当天的系统时钟。
4. mfg_date、exp_date 为公历真实完整日期，范围 0001-01-01 至 9999-12-31，严格检验年月日存在性；拒绝二位年、部分日期、无效月日和无法消歧的日期类型。
5. 日历计算和自然日差使用日期类型，而不是固定 24 小时秒数；夏令时切换不增加或减少自然日。超出支持年份或运算溢出为无效输入，不静默截断。
6. generated_at 记录生成时刻，不参与 remaining_days 或历史判定；原 observed_at 必须与原 Decision 一并保留。

## 三、保质期与日历加法

保质期始终是 `{value: 正整数, unit: DAY / MONTH / YEAR}`，不能把 MONTH / YEAR 预先转换为固定天数。

| unit | 候选算法 | 示例 |
| :--- | :--- | :--- |
| DAY | `derived_exp_date = mfg_date + value` 个自然日，**不减 1** | 2024-05-18 + 1 DAY -> 2024-05-19 |
| MONTH | 原日期一次加 value 个日历月；目标月没有原日号则截到目标月最后一天 | 2024-01-31 + 1 MONTH -> 2024-02-29 |
| YEAR | 一次加 `12 * value` 个日历月，同样执行目标月末截断 | 2024-02-29 + 1 YEAR -> 2025-02-28 |

- 本候选规则明确**不是“生产日计第 1 天”的包含式计数**。2024-05-18 + 12 MONTH -> 2025-05-18，不能生成 2025-05-17。
- 月加法只对最终目标月截断，不能重复执行单月加法累计日号损失：2024-01-31 + 2 MONTH -> 2024-03-31，不是 2024-03-29。
- 闰年按公历规则：被 4 整除，且不被 100 整除或被 400 整除。2024 为闰年，2100 不是闰年。
- 同一标示单位保留原值；计算得到的自然日长度是该日期区间的结果，不能回写为原始保质期事实或离线标签。
- 若包装明确采用不同的包含式口径、条件期限或其他单位，当前候选 v1 不自动猜测；记录警告并要求复核，升级规则及真实测试后再接入。

## 四、必要证据选择与一致性

### 1. 可靠性前置条件

先确定本次判定所需事实，再评估冲突、合法性和可靠性，不能先算一个乐观状态再用警告掩盖问题。

- 必要事实存在未解决冲突、非法日期 / 运算、可靠生产日期晚于 observed_date，或非正生产到期区间 -> `UNDETERMINED`。
- 必要字段只有 `eligible_for_decision: false` 或无法证明可靠的候选 -> `UNDETERMINED / LOW_CONFIDENCE`，不直接使用最高置信候选。
- 必要证据不存在 -> `UNDETERMINED / INFORMATION_MISSING`，保留缺项与补充建议。
- 语法非法日期不能塞入合法 Evidence.fact.value；保留原 Observation，拒绝该解析事实并记录 warnings。若它是唯一必要候选，缺少合法输入，输出 UNDETERMINED / INFORMATION_MISSING；合法但不可靠候选走 LOW_CONFIDENCE。输入 DTO 本身不合法则拒绝契约输入，不能伪造一份“成功校验”的决策。
- 可靠候选之间的冲突按有记录的版本化聚合 / 修正策略解决；未解决前不取第一条、最后一条或最大分数。

### 2. 直接与派生到期日

1. 有可靠直接 EXP 时可使用它；不能强制同时补齐 MFG 和 shelf_life 才判断已超期或 30 天内临期。
2. 同时有可靠 MFG + shelf_life 时，必须推导 EXP，并与可靠直接 EXP 比较。日期不同就是必要冲突，`UNDETERMINED / EVIDENCE_CONFLICT`，即使两者暂时落在同一 marked_status 也不能忽略。
3. 无直接 EXP，但有可靠 MFG + shelf_life 时，使用 derived_exp_date，snapshot.exp_date_is_derived = true，used_fact_ids 引用这两个真实来源事实，不制造 EXP Observation / MODEL fact。
4. 有可靠 MFG + 直接 EXP 时，即使没有 shelf_life 也可计算 total_lifetime_days；不把缺少 shelf_life 当成固定缺项。
5. 直接与派生 EXP 一致时，snapshot 采用直接 EXP，exp_date_is_derived = false；used_fact_ids 保留直接 EXP 与参与一致性检查的真实 MFG / shelf_life。
6. 日期字段没有可靠选择结果时，相应 snapshot 槽位用 null，完整 Evidence 保留所有候选与冲突，不能以 snapshot 隐藏未解决冲突。

### 3. 必要性与复核范围

必要性由结果依赖决定：EXP 是直接路线的必要事实；MFG + shelf_life 是派生路线的必要事实；MFG 是剩余超过 30 天时计算寿命比例的必要事实。已提供的可靠 MFG 还必须检查未来生产与生产到期先后；已有可靠 MFG + shelf_life 的直接 / 派生一致性检查不可跳过。

如果可靠直接 EXP 已经足以判断 PAST 或 30 天内 NEAR，缺少 MFG / shelf_life 或仅有不参与计算的不可靠补充候选，不应强制 LOW_CONFIDENCE / INFORMATION_MISSING。未解决冲突若影响必要日期选择或必须进行的一致性检查，仍先返回 UNDETERMINED。非必要 storage_condition 的缺失、低置信或冲突只产生 warnings，不把日期决策降为冲突。

Decision.review_status 优先级固定：

| 优先级 | review_status | 适用情形 |
| :--- | :--- | :--- |
| 1 | EVIDENCE_CONFLICT | 必要事实未解决冲突，未来生产、EXP <= MFG 等时间一致性错误 |
| 2 | LOW_CONFIDENCE | 没有上述冲突，但必要字段只有不可靠候选 |
| 3 | INFORMATION_MISSING | 没有上述问题，但缺少必要合法事实或比例输入 |
| 4 | EVIDENCE_COMPLETE | 本次日期判定所需事实足够可靠且一致 |

复核检查只针对影响日期判定的必要事实，不要求每个包装字段齐全。混合问题保留全部相关 warnings / missing_fields，但 review_status 只取最高优先级。

## 五、标示期限判定顺序

定义日期差，不用秒数：

- `remaining_days = exp_date - observed_date`，为带符号的整数自然日差。
- 有可靠 MFG + EXP 时，`total_lifetime_days = exp_date - mfg_date`，必须为正整数。
- 没有 MFG 时，不虚构 total_lifetime_days；有保质期文字但无生产日期也不能自动证明当前总寿命。

严格按以下顺序执行：

| 顺序 | 条件 | marked_status |
| :--- | :--- | :--- |
| 1 | 存在必要冲突、无效 / 未来生产、低置信或缺少本次必需输入 | UNDETERMINED |
| 2 | 可靠 EXP 且 remaining_days < 0 | PAST_MARKED_PERIOD |
| 3 | remaining_days >= 0 且 remaining_days <= 30 | NEAR_EXPIRY |
| 4 | remaining_days > 30 且无可靠 MFG，不能计算比例 | UNDETERMINED |
| 5 | 有正 total_lifetime_days 且 remaining_days * 10 <= total_lifetime_days | NEAR_EXPIRY |
| 6 | 其余可靠、完整且一致的输入 | WITHIN_MARKED_PERIOD |

顺序 4 属于顺序 1 的必要输入判断的条件分支；不能在不知道比例的情况下先落入“在期内”。因此：

- EXP-only 且已超期 -> PAST_MARKED_PERIOD；EXP-only 且 0..30 天 -> NEAR_EXPIRY。没有其他必要问题时 review_status = EVIDENCE_COMPLETE，missing_fields = []。
- EXP-only 且剩余 > 30 天 -> UNDETERMINED / INFORMATION_MISSING，`missing_fields = ["mfg_date"]`，不得默认 WITHIN_MARKED_PERIOD。若 MFG 只有不可靠候选，则复核优先 LOW_CONFIDENCE。
- 有 MFG 和 EXP 时，不强制 shelf_life；有 MFG 和 shelf_life 时，不强制直接 EXP。
- 临期逻辑为 `remaining_days <= 30 OR remaining_days * 10 <= total_lifetime_days`，PAST 与 UNDETERMINED 前置检查优先。
- 到期当天 remaining_days = 0 -> NEAR_EXPIRY，次日 remaining_days = -1 -> PAST_MARKED_PERIOD。
- 比例比较只用整数乘法，禁止除以 0、浮点百分比四舍五入或把 10% 向上取整；总寿命 <= 0 先视为无效区间。
- 未找到可靠 EXP 且不能推导时，missing_fields 标明缺少的真实可补充输入；补可靠直接 EXP 与补齐 MFG + shelf_life 是替代路线，不暗示全部字段都必须齐全。

## 六、候选日期解析矩阵

下列仅是解析格式候选，须由真实解析器逐格式正负测试和真实 OCR / 字典输出验证后冻结，**不表示当前已经支持**，也不据此宣称支持 12+ 种格式。

| 格式候选 | 合法示意 | 标准化候选 | 冻结前检查 |
| :--- | :--- | :--- | :--- |
| YYYY-MM-DD | 2024-05-18 | 2024-05-18 | 完整四位年与双位月日、日历合法性 |
| YYYY/MM/DD | 2024/05/18 | 2024-05-18 | 分隔符一致，不接受部分串 |
| YYYY.MM.DD | 2024.05.18 | 2024-05-18 | 不把批号或小数误作日期 |
| YYYYMMDD | 20240518 | 2024-05-18 | 恰好八位且必须有可证实类型语义 |
| YYYY年M月D日 | 2024年5月18日 | 2024-05-18 | 月日一或两位、字典及真实 OCR 能力验证 |

正负测试还必须覆盖：二位年如 24/05/18、缺日如 2024-05、非法 2023-02-29、混合分隔符、前后多余数字、同串多个日期、无类型单日期串、LOT / 批号歧义，以及 MFG / EXP / BEST_BEFORE 上下文。

当前候选不能从二位年补世纪、从部分日期补月末、从无类型串推定 MFG，也不能无证据“纠正”原 OCR 数字。未解析成功保留原始观察值，以 UNDETERMINED 和补充建议表达；字段级 Correction 可提供合法明确输入，但不表示已支持原 OCR 文字 / 框修正。

## 七、存储与开封提示

“冷藏保存”“阴凉处保存”“开封后 48 小时内饮用”等只进入存储事实或 warnings / guidance，不自动改写生产日、保质期单位或到期日；没有真实开封时间、存储履约证据及独立规则时，不计算新的食品安全期限。storage_condition 冲突不覆盖可靠日期；提示只能要求核对包装与实际条件，不能保证可食用。

## 八、版本与历史重放

- 规则、解析 / 置信配置及相关时区数据库变更须登记新版本，保留旧实现、配置及审核依据，不让 date-rules.v1 在原名下静默改变结果。
- 历史重放固定：完整不可变 Evidence 与实际 evidence_hash、原 observed_at、原 timezone、原 date-rules.v1 实现与配置、Evidence.parser_version 对应解析 / 置信策略及来源修正记录。snapshot 单独不足以重放。
- 重放先重新验证 Evidence 的 JCS 哈希与实体引用，再核对选中事实、派生日期、used_fact_ids 和二维状态；哈希不一致、原版本不可获得或原始证据缺失则拒绝作为可信历史重放，不能退回使用当前配置或仅用 snapshot。
- 重放可重新生成时间或独立核验记录，但不得覆盖原 Decision；原观察日期不随当前系统时钟变化。
- 使用新的观察日重新查验，必须产生新 decision_id / observed_at / generated_at，即使引用同一 Evidence 也不是历史重放，更不能覆盖旧决策。
- 新规则评估旧证据必须明确记录新规则版本与新决策；当前 decision.v1 Schema 固定 date-rules.v1，新规则接入须同步补充 / 升级决策契约及测试。

## 九、候选边界测试向量

以下 **36 个向量为候选预期，不是已通过项目测试**。除特殊说明，时区 Asia/Shanghai、观察日取对应服务端本地日期、输入均为真实来源且 eligible 的测试替身，所有日期单位按本规则。C = EVIDENCE_COMPLETE，M = INFORMATION_MISSING，X = EVIDENCE_CONFLICT，L = LOW_CONFIDENCE；这些缩写只用于表格，不是 DTO 枚举。

| ID | 输入与观察日 | 候选关键结果 | marked_status / review_status |
| :--- | :--- | :--- | :--- |
| R01 | MFG 2024-05-18，12 MONTH；观察 2024-05-18 | 派生 EXP 2025-05-18，不减一天 | WITHIN_MARKED_PERIOD / C |
| R02 | MFG 2024-01-31，1 MONTH；观察 2024-02-28 | EXP 2024-02-29，remaining = 1 | NEAR_EXPIRY / C |
| R03 | MFG 2023-01-31，1 MONTH；观察 2023-03-01 | EXP 2023-02-28，remaining = -1 | PAST_MARKED_PERIOD / C |
| R04 | MFG 2024-02-29，1 YEAR；观察 2025-02-28 | EXP 2025-02-28，remaining = 0 | NEAR_EXPIRY / C |
| R05 | MFG 2024-02-29，4 YEAR；观察 2028-03-01 | EXP 2028-02-29，remaining = -1 | PAST_MARKED_PERIOD / C |
| R06 | MFG 2024-05-18，1 DAY；观察 2024-05-18 | EXP 2024-05-19，不以生产日计第 1 天 | NEAR_EXPIRY / C |
| R07 | MFG 2024-05-18，1 YEAR 或 12 MONTH；观察 2024-05-18 | 两种输入都派生 EXP 2025-05-18 | WITHIN_MARKED_PERIOD / C |
| R08 | MFG 2024-01-31，2 MONTH；观察 2024-02-01 | EXP 2024-03-31，不连续截月末 | WITHIN_MARKED_PERIOD / C |
| R09 | MFG 2024-02-28，1 DAY；观察 2024-03-01 | EXP 2024-02-29，remaining = -1 | PAST_MARKED_PERIOD / C |
| R10 | MFG 2025-01-01，EXP 2025-05-18；观察 2025-04-18 | remaining = 30；无需 shelf_life | NEAR_EXPIRY / C |
| R11 | 同 R10；观察 2025-04-17 | remaining = 31，total = 137，310 > 137 | WITHIN_MARKED_PERIOD / C |
| R12 | MFG 2024-01-01，EXP 2025-02-04；观察 2024-12-26 | remaining = 40，total = 400，400 <= 400 | NEAR_EXPIRY / C |
| R13 | 同 R12；观察 2024-12-25 | remaining = 41，410 > 400；不四舍五入 | WITHIN_MARKED_PERIOD / C |
| R14 | 仅 EXP 2025-05-18；观察 2025-05-18 | 到期当天，remaining = 0，missing_fields = [] | NEAR_EXPIRY / C |
| R15 | 仅 EXP 2025-05-18；观察 2025-05-19 | 到期次日，remaining = -1 | PAST_MARKED_PERIOD / C |
| R16 | 仅 EXP 2025-05-18；观察 2025-04-17 | remaining = 31；missing_fields = [mfg_date] | UNDETERMINED / M |
| R17 | 仅 EXP 2025-05-18；观察 2025-04-18 | remaining = 30，不要求补 MFG 才判临期 | NEAR_EXPIRY / C |
| R18 | 仅 EXP 2025-05-18；观察 2025-04-17；唯一 MFG 候选不可靠 | 必要 MFG 只有低置信候选，不能默认在期内 | UNDETERMINED / L |
| R19 | MFG 2024-05-18，12 MONTH，直接 EXP 2025-05-17；观察 2024-06-01 | 派生 EXP 2025-05-18 与直接 EXP 不一致，引用三项真实事实 | UNDETERMINED / X |
| R20 | 同 R19，但直接 EXP 2025-05-18 | 一致，snapshot derived = false，used_fact_ids 保留三项 | WITHIN_MARKED_PERIOD / C |
| R21 | 两个未解决的可靠 EXP：2025-05-18 与 2025-05-19；观察 2025-05-18 | 不因同属临期而忽略日期冲突 | UNDETERMINED / X |
| R22 | MFG 2025-05-19，EXP 2026-05-19；观察 2025-05-18 | 未来生产，警告 FUTURE_MFG_DATE | UNDETERMINED / X |
| R23 | MFG 与 EXP 均为 2025-05-18；观察同日 | total = 0，拒绝除零与比例比较 | UNDETERMINED / X |
| R24 | MFG 2025-05-19，EXP 2025-05-18；观察 2025-05-20 | total = -1，无效区间，不能先判 PAST | UNDETERMINED / X |
| R25 | 唯一 EXP 候选 2025-05-18 不可靠；观察 2025-05-19 | 不能因数字看似已超期而绕过可靠性 | UNDETERMINED / L |
| R26 | R01 日期输入，另有 storage_condition 冲突或缺失 | 日期结果不降级，保留存储提示 | WITHIN_MARKED_PERIOD / C |
| R27 | 仅 EXP 2025-05-18；observed_at 2025-05-17T16:00:00Z | 上海本地 2025-05-18，不能按 UTC 日期截取 | NEAR_EXPIRY / C |
| R28 | 仅 EXP 2025-05-18；observed_at 2025-05-18T16:00:00Z | 上海本地 2025-05-19 | PAST_MARKED_PERIOD / C |
| R29 | timezone America/New_York，MFG 2024-03-09，2 DAY；observed_at 2024-03-10T23:30:00-04:00 | EXP 2024-03-11，跨 DST 按自然日，remaining = 1 | NEAR_EXPIRY / C |
| R30 | R14 的原完整 Evidence、hash、observed_at、timezone、原规则配置；在 2025-05-19 重放 | 仍使用原观察日 2025-05-18，不用当前日期 | NEAR_EXPIRY / C，原决策不覆盖 |
| R31 | 与 R14 同一 Evidence；新 observed_at 2025-05-19T09:00:00+08:00 | 新 Decision，不是重放；保留原 NEAR 决策 | PAST_MARKED_PERIOD / C |
| R32 | 修改 Evidence 某 fact / eligible / generated_at 但沿用旧 hash，或只有 snapshot | 完整性不通过或证据不足，拒绝可信重放 | 不生成冒充成功的历史 Decision |
| R33 | 唯一原文 MFG 24/05/18，无合法补充；观察 2024-05-18 | 二位年不补世纪，不产生合法日期事实 | UNDETERMINED / M |
| R34 | 唯一原文 MFG 2023-02-29，无合法补充；观察 2023-03-01 | 非闰年非法日期，保留原文与 INVALID_DATE 警告 | UNDETERMINED / M |
| R35 | 唯一无类型串 20240518 或部分日期 2024-05，无合法补充 | 不默认类型或补日；保留歧义提示 | UNDETERMINED / M |
| R36 | MFG 2099-01-31，1 MONTH，观察 2099-02-28；另测 MFG 2100-01-31，1 MONTH，观察 2100-02-28 | 两个目标二月均为 28 天，世纪年 2100 非闰年 | NEAR_EXPIRY / C，到期日按各自输入 |

冻结前还需真实测试：Schema 拒绝 0 / 负数 / 小数保质期、非法单位与未知修正字段；格式矩阵全部正负例；解析非法或越界年份；未偏移时钟；MANUAL 引用不存在；conflicts / used_fact_ids 悬空；必要日期冲突 + 低置信 + 缺失的优先级；开封提示不改日期；规则配置不可获得时拒绝重放；JCS 真实标准向量及 Evidence 篡改检测。不能以本表存在或独立算术核对替代项目日期引擎与真实模型测试。

## 十、候选未决边界

- date-rules.v1 采用“不计生产日为第 1 天”的候选口径；若包装或法规明确要求其他计数口径，须人工复核并另定规则版本，不能静默减一天。
- 当前 DTO 没有独立 date_semantic、confidence_policy_version 或 timezone_database_version 字段。必须通过原始文字和不可变版本 / 配置清单保留相应信息；如果实现无法做到无歧义追溯，冻结前应补契约，而不是丢弃这些信息。
- 冲突解决、合法修正的优先选择、可靠性阈值和 M0 状态转移尚需真实实现与审核证据；此处不假定“人工永远覆盖机器”或“阈值已校准”。
- 原始 OCR 文字 / 框修正虽属 C0 规划，但 correction.v1 尚未定义；未来升级后才能接入，不能复用 corrected_field 的未知值暗中支持。
