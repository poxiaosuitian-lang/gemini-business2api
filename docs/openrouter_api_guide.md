# Runloop/OpenRouter 视频 & 图片生成完整指南

> **给新对话的提示词**：请读取以下文档，然后我可以直接让你生成视频或图片。你不需要我有额外的上下文，按照文档中的步骤操作即可。

---

## 〇、Key 管理（自动创建 + 复用）

### 环境变量

环境中已预配置：

| 变量 | 说明 | 示例值 |
|------|------|--------|
| `OPENROUTER_API_KEY` | 网关 API Key（`gws_` 前缀，通过 Runloop 网关认证） | `gws_2VYXpzn2...` |
| `OPENROUTER_BASE_URL` | 网关地址 | `https://gateway.runloop.ai` |

### 自动化 Key 检查与创建流程

每次新对话开始时，按以下步骤处理 Key：

**Step 1: 检查环境变量是否有可用 Key**

```bash
# 检查环境变量中是否已有可用的 key
if [ -n "$OPENROUTER_API_KEY" ]; then
  echo "已找到 OPENROUTER_API_KEY（前缀: ${OPENROUTER_API_KEY:0:10}...）"
else
  echo "OPENROUTER_API_KEY 未设置，需要创建"
fi
```

**Step 2: 验证 Key 是否有效（调一个轻量接口测试）**

```bash
# 用 /health 或 /api/v1/models 验证 key 是否可用
RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" \
  "https://gateway.runloop.ai/health" \
  -H "Authorization: Bearer $OPENROUTER_API_KEY")

if [ "$RESPONSE" = "200" ]; then
  echo "Key 验证成功 ✅"
else
  echo "Key 验证失败 ❌ (HTTP $RESPONSE)，需要重新获取"
fi
```

**Step 3: 如果 Key 不存在或无效，自动创建**

```bash
# 方式 A：如果环境中已有 Runloop 的 session token（如 gws_ 开头的自动注入）
# 直接用环境变量中的 token 作为 key 使用，无需额外创建

# 方式 B：通过 Runloop MCP 创建新 devbox/workspace key
export RUNLOOP_API_KEY="$OPENROUTER_API_KEY"
npx @runloop/rl-cli gateway-config list 2>/dev/null

# 方式 C：手动指引 — 如果以上都失败，告诉用户：
echo "请访问以下地址创建 API Key："
echo "  https://openrouter.ai/settings/keys"
echo "创建后将 key 设置为环境变量："
echo "  export OPENROUTER_API_KEY=sk-or-xxxxx"
echo ""
echo "或使用 Runloop 网关 Key（如有 workspace session）："
echo "  export OPENROUTER_API_KEY=gws_xxxxx"
```

**Step 4: 缓存到本地文件（复用）**

```bash
# 将有效的 key 缓存到本地文件，避免每次手动设置
KEY_CACHE_FILE="$HOME/.openrouter_cache"

save_key() {
  echo "$OPENROUTER_API_KEY" > "$KEY_CACHE_FILE"
  chmod 600 "$KEY_CACHE_FILE"
  echo "Key 已缓存到 $KEY_CACHE_FILE"
}

load_cached_key() {
  if [ -f "$KEY_CACHE_FILE" ]; then
    export OPENROUTER_API_KEY=$(cat "$KEY_CACHE_FILE")
    echo "已从缓存加载 Key ✅"
    return 0
  fi
  return 1
}

# 完整流程：环境变量 → 缓存文件 → 用户创建
setup_key() {
  # 1. 先尝试环境变量
  if [ -n "$OPENROUTER_API_KEY" ]; then
    RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" \
      "https://gateway.runloop.ai/health" \
      -H "Authorization: Bearer $OPENROUTER_API_KEY")
    if [ "$RESPONSE" = "200" ]; then
      echo "使用环境变量中的 Key ✅"
      return 0
    fi
  fi
  
  # 2. 尝试缓存
  if load_cached_key; then
    RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" \
      "https://gateway.runloop.ai/health" \
      -H "Authorization: Bearer $OPENROUTER_API_KEY")
    if [ "$RESPONSE" = "200" ]; then
      echo "使用缓存的 Key ✅"
      return 0
    fi
  fi
  
  # 3. 都没有 → 打印指引
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "未找到可用 Key，请按以下步骤创建："
  echo ""
  echo "1. 访问 https://openrouter.ai/settings/keys"
  echo "2. 点击 'Create Key'"
  echo "3. 复制生成的 key"
  echo "4. 在终端执行：export OPENROUTER_API_KEY=你的key"
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  return 1
}
```

### Key 类型说明

| 前缀 | 类型 | 用途 |
|------|------|------|
| `gws_` | Runloop Gateway Workspace | 自动注入，通过网关代理所有 API |
| `sk-or-` | OpenRouter 标准 Key | 直接调用 openrouter.ai |
| Management Key | 管理 API Key | 仅用于 `/api/v1/keys` 管理接口，不能调生成 API |

**注意**：
- 通过网关（`gateway.runloop.ai`）调用时用 `gws_` 格式的 key
- 直接调 `openrouter.ai` 时用 `sk-or-` 格式
- Management Key 是单独的，不能混用

---

## 一、API 基础信息

```bash
GATEWAY="https://gateway.runloop.ai"
AUTH="Authorization: Bearer $OPENROUTER_API_KEY"
```

**关键规则**：
- 所有 API 必须带 `/api/v1` 前缀（不是 `/v1`！直接调 `/v1/...` 会返回 HTML 错误页）
- 认证头：`Authorization: Bearer $OPENROUTER_API_KEY`

---

## 二、视频生成（16 个模型）

### 2.1 查询可用模型

```bash
curl -s "$GATEWAY/api/v1/videos/models" -H "$AUTH" | python3 -m json.tool
```

### 2.2 模型清单

| # | Model ID | 厂商 | 分辨率 | 比例 | 时长(秒) | 首/尾帧 | 音频 | Seed | 参考价($/s或/token) |
|---|----------|------|--------|------|----------|---------|------|------|---------------------|
| 1 | `google/veo-3.1` | Google | 720p-4K | 16:9/9:16 | 4/6/8 | ✅/✅ | ✅ | ✅ | $0.20-0.60/s |
| 2 | `google/veo-3.1-fast` | Google | 720p-4K | 16:9/9:16 | 4/6/8 | ✅/✅ | ✅ | ✅ | $0.08-0.30/s |
| 3 | `google/veo-3.1-lite` | Google | 720p/1080p | 16:9/9:16 | 4/6/8 | ✅/✅ | ✅ | ✅ | $0.03-0.08/s |
| 4 | `openai/sora-2-pro` | OpenAI | 720p/1080p | 16:9/9:16 | 4-20 | ❌/❌ | ✅ | ❌ | $0.30-0.50/s |
| 5 | `bytedance/seedance-2.0` | ByteDance | 480p-4K | 7种 | 4-15 | ✅/✅ | ✅ | ✅ | $0.007/token |
| 6 | `bytedance/seedance-2.0-fast` | ByteDance | 480p/720p | 7种 | 4-15 | ✅/✅ | ✅ | ✅ | $0.0056/token |
| 7 | `bytedance/seedance-1-5-pro` | ByteDance | 480p-1080p | 7种 | 4-12 | ✅/✅ | ✅ | ✅ | $0.0024/token |
| 8 | `kwaivgi/kling-v3.0-pro` | 快手 Kling | 720p | 16:9/9:16/1:1 | 3-15 | ✅/✅ | ✅ | ❌ | $0.112-0.168/s |
| 9 | `kwaivgi/kling-v3.0-std` | 快手 Kling | 720p | 16:9/9:16/1:1 | 3-15 | ✅/✅ | ✅ | ❌ | $0.084-0.126/s |
| 10 | `kwaivgi/kling-video-o1` | 快手 Kling | 720p | 16:9/9:16/1:1 | 5/10 | ✅/✅ | ✅ | ❌ | $0.112/s |
| 11 | `alibaba/happyhorse-1.1` | 阿里 | 720p/1080p | 7种 | 3-15 | ✅/❌ | ❌ | ✅ | $0.099-0.128/s |
| 12 | `alibaba/happyhorse-1.0` | 阿里 | 720p/1080p | 7种 | 3-15 | ✅/❌ | ❌ | ✅ | $0.099-0.169/s |
| 13 | `alibaba/wan-2.7` | 阿里万相 | 720p/1080p | 5种 | 2-10 | ✅/✅ | ✅ | ✅ | $0.10/s |
| 14 | `alibaba/wan-2.6` | 阿里万相 | 720p/1080p | 16:9/9:16 | 5/10 | ✅/❌ | ✅ | ✅ | $0.04-0.15/s |
| 15 | `minimax/hailuo-2.3` | 海螺 | 1080p | 16:9 | 6/10 | ✅/❌ | ❌ | ❌ | $0.082/s |
| 16 | `x-ai/grok-imagine-video` | xAI Grok | 480p/720p | 7种 | 1-15 | ✅/❌ | ❌ | ❌ | $0.05-0.07/s |

**比例完整列表**：`16:9` / `9:16` / `1:1` / `4:3` / `3:4` / `3:2` / `2:3` / `21:9` / `9:21`

### 2.3 提交视频生成（异步任务）

```bash
curl -s -X POST "$GATEWAY/api/v1/videos" \
  -H "$AUTH" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "bytedance/seedance-2.0",
    "prompt": "Two anime warriors fighting with swords on a mountain cliff at sunset, dramatic action",
    "resolution": "720p",
    "aspect_ratio": "16:9",
    "duration": 15,
    "generate_audio": true
  }'
```

**返回**：
```json
{
  "id": "9mYwB7O4mw0kmIibglTO",
  "polling_url": "https://openrouter.ai/api/v1/videos/9mYwB7O4mw0kmIibglTO",
  "status": "pending"
}
```

### 2.4 完整请求体

```json
{
  "model": "string (必填)",
  "prompt": "string (必填，避免版权 IP 名称)",
  "resolution": "480p | 720p | 1080p | 1K | 2K | 4K",
  "aspect_ratio": "16:9 | 9:16 | 1:1 | 4:3 | 3:4 | 3:2 | 2:3 | 21:9 | 9:21",
  "size": "WIDTHxHEIGHT (如 1280x720，与 resolution+aspect_ratio 二选一)",
  "duration": "整数(秒)",
  "generate_audio": true,
  "seed": 12345,
  "frame_images": [
    {"type": "image_url", "image_url": {"url": "https://..."}, "frame_type": "first_frame"},
    {"type": "image_url", "image_url": {"url": "https://..."}, "frame_type": "last_frame"}
  ],
  "input_references": [
    {"type": "image_url", "image_url": {"url": "https://..."}},
    {"type": "audio_url", "audio_url": {"url": "https://..."}},
    {"type": "video_url", "video_url": {"url": "https://..."}}
  ],
  "callback_url": "https://your-webhook-url",
  "provider": {"options": {"byteplus": {"watermark": false}}}
}
```

**注意**：
- `input_references` 中 `audio_url` / `video_url` 仅 Seedance 2.0（`byteplus` 后端）支持
- `frame_images` 需要公网可访问的图片 URL
- prompt 不能包含知名 IP 名称（会触发版权审核失败）

### 2.5 轮询任务状态

```bash
# 每 15-20 秒查询一次，最长等 10 分钟
JOB_ID="9mYwB7O4mw0kmIibglTO"

curl -s "$GATEWAY/api/v1/videos/$JOB_ID" -H "$AUTH"
```

**状态流转**：`pending` → `in_progress` → `completed` | `failed` | `cancelled` | `expired`

**完成返回**：
```json
{
  "id": "...",
  "generation_id": "gen-vid-...",
  "status": "completed",
  "unsigned_urls": ["https://openrouter.ai/api/v1/videos/JOB_ID/content?index=0"],
  "usage": {"cost": 2.268, "is_byok": false}
}
```

### 2.6 下载视频

```bash
# 通过网关下载（替换域名为 gateway.runloop.ai）
curl -s -L "$GATEWAY/api/v1/videos/$JOB_ID/content?index=0" \
  -H "$AUTH" -o output.mp4
```

### 2.7 完整 Python 调用模板

```python
import requests, time, json, os

GATEWAY = os.environ.get("OPENROUTER_BASE_URL", "https://gateway.runloop.ai")
AUTH = {"Authorization": f"Bearer {os.environ['OPENROUTER_API_KEY']}",
        "Content-Type": "application/json"}

def generate_video(model, prompt, resolution="720p", aspect_ratio="16:9",
                   duration=10, generate_audio=True, **kwargs):
    """提交视频生成任务并等待完成"""
    
    # 1. 提交
    payload = {
        "model": model, "prompt": prompt,
        "resolution": resolution, "aspect_ratio": aspect_ratio,
        "duration": duration, "generate_audio": generate_audio, **kwargs
    }
    resp = requests.post(f"{GATEWAY}/api/v1/videos", headers=AUTH, json=payload)
    job = resp.json()
    job_id = job["id"]
    print(f"[提交] job_id={job_id}, status={job['status']}")
    
    # 2. 轮询（最多 10 分钟）
    for i in range(30):
        time.sleep(20)
        r = requests.get(f"{GATEWAY}/api/v1/videos/{job_id}", headers=AUTH).json()
        status = r.get("status", "unknown")
        print(f"  [{i+1:2d}] {status}")
        if status in ("completed", "failed", "cancelled", "expired"):
            return r
    
    return {"status": "timeout", "id": job_id}

def download_video(result, output_path):
    """从结果中下载视频"""
    urls = result.get("unsigned_urls", [])
    if not urls:
        print(f"无视频 URL, error: {result.get('error', 'N/A')}")
        return False
    url = urls[0].replace("https://openrouter.ai", GATEWAY)
    resp = requests.get(url, headers=AUTH, stream=True)
    with open(output_path, "wb") as f:
        for chunk in resp.iter_content(8192):
            f.write(chunk)
    size = os.path.getsize(output_path) / (1024*1024)
    print(f"[下载] {output_path} ({size:.1f} MB)")
    return True

# ===== 使用示例 =====
result = generate_video(
    model="bytedance/seedance-2.0",
    prompt="两个武士在日落山崖上用剑激烈搏斗，日漫风格...",
    resolution="720p", aspect_ratio="16:9", duration=15
)

if result.get("status") == "completed":
    download_video(result, "output.mp4")
    print(f"费用: ${result['usage']['cost']}")
else:
    print(f"失败: {result.get('error', result.get('status', 'timeout'))}")
```

---

## 三、图片生成（8 个模型）

### 3.1 查询可用模型

```bash
curl -s "$GATEWAY/api/v1/models" -H "$AUTH" | python3 -c "
import json, sys
data = json.load(sys.stdin)
for m in data.get('data', []):
    arch = m.get('architecture', {})
    if 'image' in arch.get('output_modalities', []):
        print(f\"{m['id']:<45} {m.get('name',''):<45} ctx={m.get('context_length','')}\")
"
```

### 3.2 模型清单

| # | Model ID | 名称 | 输入 | 上下文 | 特点 |
|---|----------|------|------|--------|------|
| 1 | `google/gemini-2.5-flash-image` | Nano Banana | text,image | 32K | 最快最便宜 |
| 2 | `google/gemini-3.1-flash-image` | Nano Banana 2 | text,image | 131K | 新一代 Flash |
| 3 | `google/gemini-3.1-flash-image-preview` | Flash Preview | text,image | 131K | 预览版 |
| 4 | `google/gemini-3-pro-image` | Nano Banana Pro | text,image | 65K | 高质量 |
| 5 | `google/gemini-3-pro-image-preview` | Pro Preview | text,image | 65K | 预览版 |
| 6 | `openai/gpt-5-image` | GPT-5 Image | text,image,file | 400K | 最强，支持文件 |
| 7 | `openai/gpt-5-image-mini` | GPT-5 Image Mini | text,image,file | 400K | 快速版 |
| 8 | `openai/gpt-5.4-image-2` | GPT-5.4 Image 2 | text,image,file | 272K | 最新 |

### 3.3 通过 Chat Completions 调用

图片模型通过标准 Chat 接口调用，图片以 base64 或 URL 返回：

```bash
curl -s -X POST "$GATEWAY/api/v1/chat/completions" \
  -H "$AUTH" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "google/gemini-2.5-flash-image",
    "messages": [
      {"role": "user", "content": "Generate an image of a cute anime cat girl"}
    ]
  }'
```

**说明**：
- 结果在 `choices[0].message.content` 中，包含 base64 图片数据或 URL
- `gemini-*-image`：快速、成本低，适合日常生成
- `gpt-5-image`：质量最高，支持文件作为输入参考

---

## 四、音频生成（2 个模型，通过 Chat Completions）

| Model ID | 说明 | 价格 |
|----------|------|------|
| `google/lyria-3-pro-preview` | 完整歌曲 | $0.08/首 |
| `google/lyria-3-clip-preview` | 30秒片段 | $0.04/段 |

---

## 五、通用注意事项

### 必须遵守
1. **API 路径**：所有调用必须用 `$GATEWAY/api/v1/...`（不能省略 `/api`）
2. **认证**：`Authorization: Bearer $OPENROUTER_API_KEY`
3. **视频是异步**：POST 提交 → GET 轮询 → GET 下载，轮询间隔 15-20 秒

### 费用
- 视频：15 秒 720p 约 $1-3，4K 更贵
- 图片：按 token 计费，通常极便宜
- 音频：按首/段计费

### 安全
- **版权问题**：prompt 不能含知名 IP 名称（如 Dragon Ball Z、Naruto、Marvel 等），否则触发审核失败
- **通用描述**：用 "anime style warriors" 代替 "like Naruto"
- **费用控制**：每次调用后检查 `usage.cost`

### 快速排错
| 错误 | 原因 | 解决 |
|------|------|------|
| `missing_envelope` | 用了 `/v1/...` 而非 `/api/v1/...` | 加 `/api` 前缀 |
| `401 Missing Authentication` | Key 无效或过期 | 检查 `setup_key` 流程 |
| `404 Not Found` | 端点错误 | 确认路径拼写 |
| `403 Error 1010` | Cloudflare 拦截 | 检查 User-Agent / 请求格式 |
| 视频 `failed` + copyright | Prompt 含版权内容 | 用通用描述重新写 prompt |
