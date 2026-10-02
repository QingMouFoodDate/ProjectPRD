# 《轻眸识刻》核心数据结构与实体模型

> **规范性质**：全系统通用数据模型与枚举值硬性规约  
> **适用范围**：数据治理、服务端业务、推理运行时与前端客户端交互

---

## 一、 样本资产元数据结构

用于离线样本暂存库（`manifest.jsonl`）与服务端资产管理，每行对应一个独立的真实包装样本：

```json
{
  "id": "00001",
  "file_name": "00001.jpg",
  "sha256": "5c1f2e8d4a...",
  "phash": "a3b8c9d2...",
  "text_raw": "20240518",
  "date_type": "MFG",
  "standard_date": "2024-05-18",
  "shelf_life_raw": "12个月",
  "shelf_life_days": 365,
  "printing_tech": "inkjet_dot_matrix",
  "package_type": "can",
  "difficulty": "medium",
  "is_benchmark": true
}
```

---

## 二、 运行时机器观察值结构

由推理运行时（Inference Runtime）前向计算产生，表示视觉模型从图像中直接抽取的客观检测框与字符识别结果：

```json
{
  "image_id": "img_9a8b7c",
  "quality_passed": true,
  "quality_metrics": {
    "blurriness_score": 142.5,
    "overexposure_ratio": 0.02
  },
  "detections": [
    {
      "box_id": "box_01",
      "category": "date_area",
      "bbox": [120, 240, 210, 310],
      "confidence": 0.935,
      "text": "20240518",
      "text_confidence": 0.912,
      "character_confidences": [0.95, 0.94, 0.92, 0.91, 0.88, 0.90, 0.91, 0.89]
    }
  ],
  "global_texts": [
    {
      "text": "保质期: 12个月",
      "confidence": 0.981
    }
  ],
  "barcode": {
    "type": "EAN_13",
    "value": "6901234567890",
    "confidence": 0.999
  }
}
```

---

## 三、 会话证据聚合与综合决策模型

由业务中枢服务结合日历进位规则推导，输出给各客户端直接呈现的结构化判定结果：

```json
{
  "session_id": "sess_8f9a2b1c",
  "marked_status": "WITHIN_MARKED_PERIOD",
  "review_status": "EVIDENCE_COMPLETE",
  "evidence_chain": {
    "mfg_date": {
      "value": "2024-05-18",
      "source_image_id": "img_001",
      "confidence": 0.912
    },
    "shelf_life": {
      "raw_text": "12个月",
      "calculated_days": 365,
      "source_image_id": "img_002"
    },
    "exp_date": {
      "value": "2025-05-17",
      "is_derived": true
    }
  },
  "missing_elements": [],
  "guidance": null,
  "warnings": [
    "包装标明开封后请于48小时内饮用完"
  ]
}
```

---

## 四、 核心枚举值字典

### 1. 标示期限状态
- `WITHIN_MARKED_PERIOD`：标示在期内（距离到期时间充足）；
- `APPROACHING_EXPIRY`：标示临期（距离到期日不足 30 天或总寿命 10%）；
- `PAST_MARKED_PERIOD`：标示已超期；
- `CANNOT_DETERMINE`：暂无法判定（缺少核心事实）。

### 2. 证据复核状态
- `EVIDENCE_COMPLETE`：证据完备（生产日期与保质期齐全且无冲突）；
- `INCOMPLETE_INFO`：信息缺失（需用户补拍或手动补录）；
- `CONFLICTING_EVIDENCE`：多处证据冲突（如不同部位打码日期不一致）；
- `LOW_CONFIDENCE`：低置信度（打码严重磨损，需人工目视确认）。

### 3. 打码工艺类型
- `inkjet_dot_matrix`：点阵喷墨打码；
- `inkjet`：连续喷墨打码；
- `laser_engraving`：激光雕刻刻印；
- `emboss`：凹凸钢印；
- `label_thermal`：热敏不干胶标签打印；
- `print`：包装预制印刷。

### 4. 包装载体类型
- `plastic_bag`：塑料软包装袋；
- `can`：易拉罐与金属罐头；
- `bottle`：玻璃或塑料瓶体；
- `bottle_cap`：瓶盖或封口塞；
- `box`：纸盒与硬纸箱；
- `tetra_pak`：利乐无菌包；
- `plastic_tray`：透明塑料托盒与冷鲜碗。
