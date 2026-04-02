# CogVideoX-3 视频生成

智谱 CogVideoX-3 文生视频 / 图生视频 / 首尾帧生成。

## API

Base URL: `https://open.bigmodel.cn/api/paas/v4`

### 1. 提交生成任务

```
POST /videos/generations
Authorization: Bearer {ZHIPU_API_KEY}
Content-Type: application/json
```

**请求体：**
```json
{
  "model": "cogvideox-3",
  "prompt": "描述视频内容的文本",
  "image_url": "可选，图生视频的输入图片URL",
  "first_frame_image_url": "可选，首帧图片URL",
  "last_frame_image_url": "可选，尾帧图片URL",
  "duration": "可选，视频时长，默认5，可选5/10",
  "resolution": "可选，分辨率，默认1080p，可选480p/720p/1080p",
  "aspect_ratio": "可选，宽高比，默认16:9，可选16:9/9:16/1:1"
}
```

**响应：**
```json
{
  "id": "task_id_xxx",
  "model": "cogvideox-3",
  "task_status": "PROCESSING",
  "request_id": "req_xxx"
}
```

### 2. 查询任务结果

```
GET /videos/generations/{task_id}
Authorization: Bearer {ZHIPU_API_KEY}
```

**响应（处理中）：**
```json
{
  "id": "task_id_xxx",
  "model": "cogvideox-3",
  "task_status": "PROCESSING"
}
```

**响应（成功）：**
```json
{
  "id": "task_id_xxx",
  "model": "cogvideox-3",
  "task_status": "SUCCESS",
  "video_result": [
    {
      "url": "https://...",
      "cover_image_url": "https://..."
    }
  ]
}
```

### task_status 状态
- `PROCESSING` — 处理中
- `SUCCESS` — 成功
- `FAIL` — 失败

## 使用方式

当用户要求生成视频时：

1. 用 exec 调用 `curl` 提交任务（API Key 从 zai auth profile 获取）
2. 告诉用户已提交，预计 1-5 分钟完成
3. 设置 cron 或用 exec sleep 轮询（每 30 秒查一次）
4. 查询成功后，下载视频到 workspace 并通知用户

### 提交任务示例
```bash
curl -s -X POST 'https://open.bigmodel.cn/api/paas/v4/videos/generations' \
  -H "Authorization: Bearer $ZHIPU_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"model":"cogvideox-3","prompt":"一只猫在雪地里奔跑","duration":"5","resolution":"1080p"}'
```

### 查询任务示例
```bash
curl -s 'https://open.bigmodel.cn/api/paas/v4/videos/generations/{task_id}' \
  -H "Authorization: Bearer $ZHIPU_API_KEY"
```

### 获取 API Key

从 OpenClaw 的 zai auth profile 获取：
```bash
# 读取配置中的 API key（redacted，需要从环境变量或 keychain 获取实际值）
# 智谱 API key 格式: xxx.yyy
# 使用 open.bigmodel.cn 的同一 key
```

注意：视频生成 API 使用的是智谱开放平台的 API Key，与对话 API 共享同一个 key。

## 通知

视频生成完成后，通过飞书通知用户（参考 TOOLS.md 中的飞书配置）。
