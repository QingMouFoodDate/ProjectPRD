# 《轻眸识刻》多端统一 RESTful API 接口规约 (API Specifications)

> **契约性质**：跨仓库多端协同硬性规范（WebClient / WxClient / MobileClient ↔ TrainPlatform）  
> **基准路径**：`http(s)://<host>:<port>/api/v1`  
> **数据格式**：统一采用 `multipart/form-data`（上传）与 `application/json`（响应）  

---

## 📌 一、 核心接口清单

| 接口名称 | HTTP 方法 | 路径 | 适用调用方 | 说明 |
| :--- | :--- | :--- | :--- | :--- |
| **包装日期在线预测** | `POST` | `/ocr/predict` | `WebClient`, `WxClient`, `MobileClient` | 上传单张食品包装图，返回小目标检测框、OCR 识别文本与过期推导 |
| **错例与难例纠错回流** | `POST` | `/samples/feedback` | `WebClient`, `WxClient`, `MobileClient` | 用户/评测人员修正错误日期，自动沉淀至难例微调池 |
| **现场采集样本批量上传** | `POST` | `/samples/batch-upload` | `MobileClient` | 移动端实地采集样本及包装元数据入库 |
| **最新端侧模型拉取** | `GET` | `/models/latest` | `MobileClient`, `WxClient` | 检查并拉取最新轻量化 ONNX 模型文件及 SHA-256 校验码 |

---

## 📡 二、 接口详述

### 1. 包装日期在线预测 (`/ocr/predict`)

**请求方式**：`POST`  
**Content-Type**：`multipart/form-data`

#### 请求参数：
- `image` *(File, 必填)*: 待检测的食品包装照片（支持 JPG/PNG，单图 <= 10MB）；
- `shelf_life_days` *(Integer, 可选)*: 用户输入的保质期天数（如输入 180），若图片中只有生产日期时配合推算；
- `return_visual` *(Boolean, 可选, 默认 false)*: 是否返回绘制了检测候选框的 Base64 缩略图（便于 WebClient 直接渲染）。

#### 响应格式 (JSON)：
```json
{
  "code": 200,
  "msg": "success",
  "data": {
    "detection": {
      "bbox": [152, 280, 310, 345],
      "score": 0.958
    },
    "recognition": {
      "text": "20240915",
      "score": 0.932
    },
    "parsed": {
      "date_type": "MFG",
      "mfg_date": "2024-09-15",
      "exp_date": "2025-03-14"
    },
    "shelf_life_status": {
      "is_expired": false,
      "days_remaining": 163,
      "status_tag": "NORMAL",
      "status_text": "在保质期内，剩余 163 天"
    },
    "visual_base64": null
  }
}
```

---

### 2. 错例与难例纠偏回流 (`/samples/feedback`)

当模型识别错误或用户修改时，客户端发起纠偏上报，用于闭环自进化。

**请求方式**：`POST`  
**Content-Type**：`application/json`

#### 请求参数：
```json
{
  "request_id": "req_20261002_00891",
  "image_url": "/uploads/cache/temp_00891.jpg",
  "corrected_mfg": "2024-09-15",
  "corrected_exp": "2025-03-14",
  "corrected_text": "20240915 批号B2",
  "feedback_type": "OCR_ERROR",
  "notes": "点阵第5个数字0被误识为8"
}
```

#### 响应格式 (JSON)：
```json
{
  "code": 200,
  "msg": "feedback recorded successfully",
  "data": {
    "sample_id": "feedback_0042"
  }
}
```

---

### 3. 最新端侧模型拉取 (`/models/latest`)

移动端或小程序定期检查是否有更新的端侧量化模型。

**请求方式**：`GET`  
**查询参数**：
- `target_platform`: `onnx` / `ncnn` / `tflite`

#### 响应格式 (JSON)：
```json
{
  "code": 200,
  "msg": "success",
  "data": {
    "model_name": "qingmou_date_det_nano",
    "version": "v2.1.0",
    "format": "onnx",
    "download_url": "https://api.qingmou.internal/models/qingmou_det_v2.1.onnx",
    "sha256": "8a3f912c98d4...",
    "input_shape": [1, 3, 640, 640],
    "mean": [0.485, 0.456, 0.406],
    "std": [0.229, 0.224, 0.225]
  }
}
```
