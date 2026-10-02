# 《轻眸识刻》多端统一 API 接口协议

> **规范性质**：跨仓系统级网络接口硬性契约  
> **适用范围**：`TrainPlatform`（服务端实现）与 `WebClient`、`MobileClient`、`WxClient`（客户端调用）

---

## 一、 协议基础规范

1. **传输协议**：统一采用 HTTPS / HTTP 1.1 及以上，文本编码为 UTF-8；
2. **报文格式**：请求体与响应体默认采用 `application/json`；多媒体文件上传采用 `multipart/form-data`；
3. **接口幂等与时钟**：时间字段统一采用 ISO 8601 格式字符串（如 `2026-10-02T12:00:00Z`）；
4. **统一响应结构**：
   ```json
   {
     "code": 200,
     "message": "success",
     "data": {},
     "request_id": "req-uuid4-trace"
   }
   ```

---

## 二、 核心端点详细定义

### 1. 创建识别会话
- **端点**：`POST /api/v1/sessions`
- **说明**：针对一件食品开启新的查验会话，支持后续追加多张不同视角的特写图片；
- **请求体**（可选）：
  ```json
  {
    "client_type": "web",
    "notes": "超市货架查验"
  }
  ```
- **成功响应**：
  ```json
  {
    "code": 200,
    "message": "success",
    "data": {
      "session_id": "sess_8f9a2b1c",
      "created_at": "2026-10-02T12:00:00Z",
      "status": "AWAITING_INPUT"
    }
  }
  ```

### 2. 上传包装图像并执行增量推理
- **端点**：`POST /api/v1/sessions/{session_id}/images`
- **说明**：向指定会话追加一张包装图片，服务端完成前置质量过滤、触发推理并增量更新会话证据；
- **Content-Type**：`multipart/form-data`
- **表单字段**：
  - `file`: 二进制图像文件（JPEG / PNG，大小限制 <= 15MB）；
  - `image_role`（可选）: `macro_date`（微距打码）、`packaging_full`（全景配料表）、`barcode`（条形码）；
  - `auxiliary_barcode`（可选）: 客户端前端已预解析出的 EAN-13 字符串。
- **成功响应**：
  ```json
  {
    "code": 200,
    "message": "success",
    "data": {
      "image_id": "img_001",
      "observations": [
        {
          "type": "date_bbox",
          "bbox": [120, 340, 280, 410],
          "confidence": 0.942,
          "transcription": "20240518"
        }
      ],
      "session_decision": {
        "marked_status": "WITHIN_MARKED_PERIOD",
        "review_status": "EVIDENCE_COMPLETE",
        "mfg_date": "2024-05-18",
        "exp_date": "2025-05-17",
        "missing_fields": [],
        "guidance_tip": null
      }
    }
  }
  ```

### 3. 获取会话完整证据与标示状态
- **端点**：`GET /api/v1/sessions/{session_id}`
- **说明**：获取会话当前的完整时间线、所有已上传图片、机器观察值与综合判定结论；
- **成功响应**：
  ```json
  {
    "code": 200,
    "message": "success",
    "data": {
      "session_id": "sess_8f9a2b1c",
      "images_count": 2,
      "accumulated_evidences": [
        {"type": "MFG", "value": "2024-05-18", "source": "img_001"},
        {"type": "SHELF_LIFE", "value": "12个月", "source": "img_002"}
      ],
      "decision": {
        "marked_status": "WITHIN_MARKED_PERIOD",
        "review_status": "EVIDENCE_COMPLETE",
        "mfg_date": "2024-05-18",
        "exp_date": "2025-05-17",
        "storage_condition": "常温保存"
      }
    }
  }
  ```

### 4. 提交人工纠偏
- **端点**：`POST /api/v1/sessions/{session_id}/correction`
- **说明**：用户对识别错误进行在线修正，服务端保存难例并重新计算标示状态；
- **请求体**：
  ```json
  {
    "image_id": "img_001",
    "corrected_field": "mfg_date",
    "corrected_value": "2024-05-18",
    "correction_reason": "点阵断笔将8误识为0"
  }
  ```
- **成功响应**：
  ```json
  {
    "code": 200,
    "message": "correction accepted and archived for active learning",
    "data": {
      "updated_decision": {
        "marked_status": "WITHIN_MARKED_PERIOD"
      }
    }
  }
  ```

### 5. 查询最新模型元数据
- **端点**：`GET /api/v1/models/latest`
- **说明**：移动端客户端用于检查并拉取最新发布的轻量模型资产与哈希校验和；
- **成功响应**：
  ```json
  {
    "code": 200,
    "message": "success",
    "data": {
      "model_version": "v1.2",
      "detection_onnx_url": "https://assets.qingmou.org/models/v1.2/det.onnx",
      "detection_onnx_sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
      "recognition_onnx_url": "https://assets.qingmou.org/models/v1.2/rec.onnx",
      "recognition_onnx_sha256": "ca978112ca1bbdcafac231b39a23dc4da786eff8147c4e72b9807785afee48bb"
    }
  }
  ```

---

## 三、 统一业务状态与错误码

| 错误码 | 标识符 | 含义与处置指引 |
| :--- | :--- | :--- |
| `200` | `SUCCESS` | 请求处理成功 |
| `4001` | `IMAGE_INVALID` | 上传文件非支持格式或文件损坏 |
| `4002` | `IMAGE_BLURRY` | 图像严重模糊（拉普拉斯方差过低），提示用户重新对焦 |
| `4003` | `IMAGE_OVEREXPOSED` | 打码区域严重反光过曝死白，提示调整角度避开光源 |
| `4004` | `SESSION_NOT_FOUND` | 会话 ID 不存在或已过期销毁 |
| `4005` | `EVIDENCE_CONFLICT` | 包装多处标示存在不可调和冲突，建议人工复查 |
| `5001` | `INFERENCE_TIMEOUT` | GPU 推理节点响应超时，已优雅降级进入重试队列 |
