---
name: jimeng-telegram-seedance2.0
category: devops
description: Dreamina/Seedance 视频生成全流程自动化技能（WSL → Windows 环境，Telegram 交付）
version: 2.0
created: 2026-04-16
verified: 2026-04-17（含多模态音频/视频支持，生产验证成功）

---

# 🎬 Dreamina/Seedance 视频生成全流程（WSL → Windows → Telegram）

## ⚡ 快速启动清单（每次加载此 skill 后先执行）

1. **确认环境**：运行 `uname -a` 确认 WSL2
2. **确认路径**：运行 `ls /mnt/c/Users/YOUR_USERNAME/.local/bin/dreamina.exe` 确认 dreamina.exe 存在
3. **确认白名单**：所有文件操作仅限 `/mnt/j/WORKSPACE/`

---

## 你的身份和环境

你是一个运行在 **WSL2（Ubuntu）** 环境下的 AI Agent。
你的核心任务：接收用户的视频生成指令，按照本文档规则，完整自动地完成从参数确认到视频交付的全流程。

### 📍 关键路径信息（必须记住）

| 项目 | 值 |
|------|-----|
| **运行环境** | WSL2 (Ubuntu) |
| **Windows 宿主用户** | YOUR_USERNAME |
| **白名单目录 (WSL)** | `/mnt/j/WORKSPACE/` |
| **白名单目录 (Windows)** | `J:/WORKSPACE/` |
| **dreamina.exe 路径** | `/mnt/c/Users/YOUR_USERNAME/.local/bin/dreamina.exe` |
| **任务状态文件** | `/mnt/j/WORKSPACE/workflow/results/pending_tasks.json` |
| **视频保存目录** | `/mnt/j/WORKSPACE/video/` |

### 📍 路径转换规则（必须严格遵守）

```python
def wsl_to_win(p): return p.replace("/mnt/j/", "J:/")
def win_to_wsl(p): return p.replace("J:/", "/mnt/j/")
```

- **Python 文件操作**：使用 WSL 路径 `/mnt/j/...`
- **dreamina.exe 所有参数（--image, --audio, --video）**：必须用 Windows 路径 `J:/...`
- **所有路径必须用正斜杠 `/`**，不能用反斜杠 `\`

---

## 📋 完整工作流程（严格按顺序执行）

### Step 1: 接收指令 → 解析参数

从用户指令中提取：
- **提示词**: 保持中文，不翻译
- **参考图片**: 按顺序记录（第几张是什么角色/背景）
- **音频参考**（可选）: 声音设定文件路径
- **视频参考**（可选）: 视频参考文件路径
- **时长**: 默认 5 秒
- **比例**: 默认 16:9
- **模型版本**: 参照下方对照表

### Step 2: 出示提交预览单（⚠️ 必须做，不能跳过）

格式如下：
```
━━━━━━━━━━━━━━━━━━━━
📋 提交预览单
━━━━━━━━━━━━━━━━━━━━
模型：Seedance 2.0 VIP
时长：10秒
比例：16:9
提示词：[用户提供的中文描述]

素材路径：
- 图片1（第几张）：/mnt/j/WORKSPACE/up/人物设定/XXX.jpg
- 图片2（第几张）：/mnt/j/WORKSPACE/up/人物设定/XXX.jpg
- 背景：/mnt/j/WORKSPACE/up/背景设定/XXX.jpg

执行链路：预处理 → 提交 → 轮询 → 下载 → Telegram 发送
━━━━━━━━━━━━━━━━━━━━
确认无误请回复「确认提交」
```

### Step 3: 等待用户回复「确认提交」

收到「确认提交」后，**立即自动执行 Step 4~Step 9，全程不再提问**。

---

## 🔧 Step 4: 预处理素材

将所有图片压缩/复制到 `/mnt/j/WORKSPACE/temp_assets/`

**规则：**
- 文件大小 > 5MB 或非 JPG → 压缩处理
- 压缩参数：最长边 1280px，JPEG quality=85
- 输出命名：`asset_0.jpg`, `asset_1.jpg`, `asset_2.jpg`...
- 人物图在前，背景图在后

**Python 压缩代码示例：**
```python
from PIL import Image, os

def compress_image(wsl_path, output_name):
    img = Image.open(wsl_path).convert("RGB")
    if max(img.width, img.height) > 1280:
        ratio = 1280 / max(img.width, img.height)
        img = img.resize((int(img.width*ratio), int(img.height*ratio)), Image.LANCZOS)
    out_path = f"/mnt/j/WORKSPACE/temp_assets/{output_name}"
    img.save(out_path, "JPEG", quality=85)
    size_kb = os.path.getsize(out_path) // 1024
    print(f"✅ {wsl_path} → {out_path} ({size_kb}KB)")

# 使用示例：
compress_image("/mnt/j/WORKSPACE/up/人物设定/角色示例2.png", "asset_0.jpg")
```

⚠️ **注意：**
- Python 文件操作使用 WSL 路径 `/mnt/j/...`
- 输出目录创建：`mkdir -p /mnt/j/WORKSPACE/temp_assets`

---

## 🎵 Step 4.5: 音频参考文件处理（如需要）

### 可用声音设定目录

```
/mnt/j/WORKSPACE/up/声音设定/
- 角色语音1.wav (1.2MB)
- 角色语音2.wav (1.0MB)
- 角色语音3.wav (766KB)
- 角色语音4.wav (1.0MB)
- 角色语音5.wav (584KB) ✅ 已验证可用（6.08秒）
```

### 音频时长验证（必须执行！）

**约束条件：**
- `--audio` 最多传 **3 个文件**（用多个 --audio 标志）
- 每个音频时长必须在 **2~15 秒之间**
- 路径必须用 Windows 格式 J:/...
- 指向 /up/ 目录的原始文件

**验证命令：**
```bash
ffprobe -i "/mnt/j/WORKSPACE/up/声音设定/角色语音5.wav" \
  -show_entries format=duration \
  -v quiet -of csv="p=0"
# 返回：6.08（在 2~15 秒范围内，✅ 可用）
```

如果时长 < 2 秒或 > 15 秒 → **不能使用该音频**，需告知用户。

### --audio 参数用法

```bash
--audio "J:/WORKSPACE/up/声音设定/角色语音5.wav" \
--audio "J:/WORKSPACE/up/声音设定/角色语音1.wav"
```

---

## 🎬 Step 4.6: 视频参考文件处理（如需要）

### --video 参数用法

**约束条件：**
- `--video` 最多传 **3 个文件**（用多个 --video 标志）
- 路径必须用 Windows 格式 J:/...

```bash
--video "J:/WORKSPACE/video/reference_video.mp4" \
--video "J:/WORKSPACE/video/another_reference.mp4"
```

---

## 🚀 Step 5: 提交任务（⚠️ 关键步骤）

### ✅ 正确调用方式：直接调用 dreamina.exe

**基础用法（仅图片参考）：**
```bash
/mnt/c/Users/YOUR_USERNAME/.local/bin/dreamina.exe multimodal2video \
  --prompt "用户描述的场景" \
  --image "J:/WORKSPACE/up/人物设定/角色示例1设定图_compressed.jpg" \
  --image "J:/WORKSPACE/up/人物设定/角色示例2_compressed.jpg" \
  --image "J:/WORKSPACE/up/背景设定/场景示例1_compressed.jpg" \
  --duration 5 \
  --ratio "16:9" \
  --model_version "seedance2.0fast"
```

**多模态用法（图片 + 音频参考）：**
```bash
/mnt/c/Users/YOUR_USERNAME/.local/bin/dreamina.exe multimodal2video \
  --prompt "用户描述的场景" \
  --image "J:/WORKSPACE/up/cut02/image02-05.jpg" \
  --audio "J:/WORKSPACE/up/声音设定/角色语音5.wav" \
  --duration 4 \
  --ratio "21:9" \
  --model_version "seedance2.0fast" \
  --video_resolution 720p
```

**多模态用法（图片 + 视频参考）：**
```bash
/mnt/c/Users/YOUR_USERNAME/.local/bin/dreamina.exe multimodal2video \
  --prompt "用户描述的场景" \
  --image "J:/WORKSPACE/up/人物设定/角色示例1日常_compressed.jpg" \
  --video "J:/WORKSPACE/video/reference_video.mp4" \
  --duration 5 \
  --ratio "16:9" \
  --model_version "seedance2.0fast"
```

**全组合（图片 + 音频 + 视频）：**
```bash
/mnt/c/Users/YOUR_USERNAME/.local/bin/dreamina.exe multimodal2video \
  --prompt "用户描述的场景" \
  --image "J:/WORKSPACE/up/人物设定/XXX_compressed.jpg" \
  --audio "J:/WORKSPACE/up/声音设定/XXX.wav" \
  --video "J:/WORKSPACE/video/reference.mp4" \
  --duration 5 \
  --ratio "16:9" \
  --model_version "seedance2.0fast"
```

### ⚠️ 关键规则（必须牢记）：

| 项目 | 正确做法 | 错误做法 |
|------|---------|----------|
| **子命令** | `multimodal2video` (提交) / `query_result` (查询) | `check`, `query`, `status` (会失败) |
| **--image 路径格式** | Windows 格式 `J:/...`，指向 `/up/` 原始文件 | `/mnt/j/...` 或 `temp_assets` 副本 |
| **--audio 路径格式** | Windows 格式 `J:/...`，指向 `/up/声音设定/` 原始文件 | `/mnt/j/...` |
| **--video 路径格式** | Windows 格式 `J:/...` | `/mnt/j/...` |
| **路径斜杠** | 正斜杠 `/` | 反斜杠 `\` |
| **素材来源** | 原始路径 `/up/` 目录 | temp_assets 副本（会上传失败） |
| **模型参数名** | `--model_version` | `--model` (dreamina.exe 用 --model_version) |

### ✅ 成功返回示例：

```json
{
  "submit_id": "<SUBMIT_ID>",
  "gen_status": "querying",
  "credit_count": 8,
  "queue_info": {
    "queue_idx": 2,
    "queue_status": "Queueing"
  }
}
```

**记录 `submit_id`，进入 Step 6。**

### ⚠️ gen_status 判断规则：

| gen_status | fail_reason | 含义 | 操作 |
|------------|-------------|------|------|
| `querying` / `processing` / `waiting` | - | 排队或生成中 | **继续轮询**，正常 |
| `success` | - | 完成 | 进入 Step 7 |
| `fail` | **有值** | 真正失败 | 汇报用户错误 |
| `fail` | **无此字段** | 初始状态（不是真失败） | **继续轮询**！ |

---

## 📊 Step 6: 注册任务到 pending_tasks.json

```python
import json, shutil
from datetime import datetime

PENDING = "/mnt/j/WORKSPACE/workflow/results/pending_tasks.json"
submit_id = "<SUBMIT_ID>"  # 替换为实际值

shutil.copy2(PENDING, f"{PENDING}.bak_{datetime.now().strftime('%Y%m%d_%H%M%S')}")
with open(PENDING, "r") as f:
    tasks = json.load(f)

tasks[submit_id] = {
    "status": "pending",
    "video_url": None,
    "local_path": None,
    "save_dir": "/mnt/j/WORKSPACE/video",
    "notified": False,
    "created_at": datetime.now().isoformat(),
    "updated_at": datetime.now().isoformat(),
    "last_check_at": None,
    "error_msg": None
}
with open(PENDING, "w") as f:
    json.dump(tasks, f, indent=2, ensure_ascii=False)

print(f"✅ 任务已注册：{submit_id}")
```

---

## 📊 Step 7: 轮询生成状态（每 20 秒查一次）

### ✅ 正确查询命令：

```bash
/mnt/c/Users/YOUR_USERNAME/.local/bin/dreamina.exe query_result --submit_id=<SUBMIT_ID>
```

⚠️ **注意：子命令是 `query_result`，不是 `check` / `query` / `status`（这些都会失败）**

### 📊 状态判断逻辑（同 Step 5）：

| gen_status | queue_status | 含义 | 操作 |
|------------|--------------|------|------|
| `querying` | `Queueing` | 排队中 | **继续轮询**，正常 |
| `querying` | `Generating` | 正在生成 | **继续轮询**，正常 |
| `success` | - | 完成 | 进入 Step 8 |
| `fail` + `fail_reason` | - | 真正失败 | 汇报用户错误 |
| `fail` + 无 `fail_reason` | - | 初始状态 | **继续轮询**，不是真失败！ |

### ⚠️ 重要规则：

- **看到 `gen_status=fail` 不要立即报失败！**
- 没有 `fail_reason` 字段的 `fail` 是初始状态，继续轮询
- 每 20 秒查询一次，直到 `success` 或真正失败
- queue_idx 降到 0 但 gen_status 仍是 querying → 正常，表示已出队进入生成中

---

## 📥 Step 8: 下载视频

从 query_result 的返回中提取 `video_url`：

```python
video_url = resp["result_json"]["videos"][0]["video_url"]
```

用 Python requests 下载（⚠️ **不要用 dreamina.exe download**）：

```python
import requests, os

submit_id = "<SUBMIT_ID>"
video_url = "<VIDEO_URL>"  # 从 query_result 提取
out_path = f"/mnt/j/WORKSPACE/video/{submit_id}.mp4"

r = requests.get(video_url, stream=True, timeout=120)
r.raise_for_status()
with open(out_path, "wb") as f:
    for chunk in r.iter_content(chunk_size=8192):
        f.write(chunk)

file_size_mb = os.path.getsize(out_path) / (1024 * 1024)
print(f"✅ 下载完成！文件大小：{file_size_mb:.2f} MB")
```

---

## 📤 Step 9: 发送给用户（⚠️ 最高优先级规则）

### ✅ 正确顺序（必须严格遵守）：

1. **先用工具更新 pending_tasks.json**（status=completed, notified=true）
2. **最后一步才输出 `MEDIA:` 路径，作为 final_response 的一部分**

### ⚠️ 绝对禁止事项：

| 禁止行为 | 后果 |
|---------|------|
| `MEDIA:` 后面还有工具调用 | gateway 把后续消息当 final_response，视频不会发送！ |
| 在 `MEDIA:` 后输出任何文字 | 违反规则，可能导致视频无法送达 |

### ✅ 正确格式（最后一条回复的唯一内容）：

```
🎉 视频生成完毕，发送给你：

MEDIA:/mnt/j/WORKSPACE/video/<SUBMIT_ID>.mp4
```

**注意：**
- `MEDIA:` 必须是大写
- 路径前不能有空格或其他文字（除了换行和表情符号）
- **`MEDIA:` 必须是最后一条回复的唯一内容，绝对不能在其后调用任何工具**

---

## 🧠 模型版本对照表（必须记住）

| 用户说 | 传给 `--model_version` 的值 |
|--------|----------------------------|
| **2.0 Fast** / Fast / 快速 | `seedance2.0fast` |
| **2.0 VIP** / VIP | `seedance2.0vip` |
| **Fast VIP** / VIP Fast / 2.0 VIP Fast | `seedance2.0fastvip` |
| **2.0** / 标准 | `seedance2.0` |
| **1.5 Pro** / Pro | `seedance1.5pro` |

### ⚠️ 规则：
- 用户没有明确说 VIP，就用非 VIP 版本
- 有歧义时在预览单里列出解析结果让用户确认

---

## 📌 提示词字数检查规则（⚠️ 必须执行）

在提交前必须检查提示词字符数：
- **上限**: 2000 字符
- **如果超过**: 自动截断，并向用户展示修订版供确认

```python
prompt = "用户的中文提示词..."
print(f"提示词字数：{len(prompt)} 字符（上限 2000）")
if len(prompt) > 2000:
    revised = prompt[:2000] + "... (已截断)"
    print(f"修订版：{revised}")
```

---

## 🚨 常见错误与处理方式

### ❌ 错误 1: `gen_status=fail`，没有 `fail_reason`

**原因**: 初始状态不是真失败  
**处理**: **继续轮询**，不要立即报失败

### ❌ 错误 2: `--image` 路径使用 `/mnt/j/...`

**原因**: dreamina.exe 是 Windows 程序，只认 `J:/` 格式  
**处理**: 所有 `--image`, `--audio`, `--video` 参数必须用 **Windows 格式 `J:/...`**

### ❌ 错误 3: 使用 temp_assets 副本做 --image

**原因**: dreamina.exe 上传该路径下的文件会失败（no file upload）  
**处理**: 必须用 `/up/` 目录的原始路径

### ❌ 错误 4: `--image` 路径使用反斜杠 `J:\`

**原因**: dreamina.exe 接受正斜杠 `J:/`，不接受反斜杠  
**处理**: 统一使用正斜杠 `J:/...`

### ❌ 错误 5: 使用了错误的子命令 (`check`, `query`, `status`)

**原因**: 这些子命令不存在  
**处理**: **只使用 `multimodal2video` (提交) / `query_result` (查询)**

### ❌ 错误 6: MEDIA: 后面还有工具调用

**原因**: gateway 把后续消息当 final_response，视频不会发送  
**处理**: **MEDIA: 必须是最后一条回复的唯一内容**

### ❌ 错误 7: 用 dreamina.exe download 子命令下载

**原因**: 该命令不可靠  
**处理**: 使用 `requests.get(video_url, stream=True)` 下载

### ❌ 错误 8: Python subprocess 调 python 脚本

**原因**: WSL 下 python 映射到 Windows Python，路径解析全失败  
**处理**: 直接在同一进程内执行 Python 代码，不要用 subprocess

### ❌ 错误 9: 音频时长超出 2~15 秒限制

**原因**: dreamina.exe 拒绝超长或过短的音频  
**处理**: 提交前用 `ffprobe` 验证时长，不在范围内的音频不能使用

---

## 📋 完整成功案例参考（2026-04-17 多模态成功）

```bash
# Step 1: 验证音频时长
ffprobe -i "/mnt/j/WORKSPACE/up/声音设定/角色语音5.wav" \
  -show_entries format=duration -v quiet -of csv="p=0"
# 返回：6.08 ✅

# Step 2: 提交（图片 + 音频）
/mnt/c/Users/YOUR_USERNAME/.local/bin/dreamina.exe multimodal2video \
  --prompt "京都动画/新海诚风格，咖啡厅内景。女主坐在靠窗位置，阳光透过玻璃洒在她身上..." \
  --image "J:/WORKSPACE/up/cut02/image02-05.jpg" \
  --audio "J:/WORKSPACE/up/声音设定/角色语音5.wav" \
  --duration 4 \
  --ratio "21:9" \
  --model_version "seedance2.0fast" \
  --video_resolution 720p

# Step 3: 轮询（约64分钟）
/mnt/c/Users/YOUR_USERNAME/.local/bin/dreamina.exe query_result --submit_id=<SUBMIT_ID>

# Step 4: 下载
# video_url = resp["result_json"]["videos"][0]["video_url"]
# requests.get(video_url, stream=True) → /mnt/j/WORKSPACE/video/<SUBMIT_ID>.mp4 (3502KB)

# Step 5: 发送
MEDIA:/mnt/j/WORKSPACE/video/<SUBMIT_ID>.mp4
```

**结果**: 成功生成并交付，视频大小 3502KB。

---

## 🎯 操作规范总结（必须遵守）

| # | 规则 |
|---|------|
| 1 | **提交前必须出示预览单**，等待「确认提交」 |
| 2 | 确认后**自动执行完整链路**，不再问任何问题 |
| 3 | `MEDIA:` 必须是**最后一条回复的唯一内容**，后面不能有任何工具调用 |
| 4 | `--image`, `--audio`, `--video` 路径必须用 **Windows 格式 `J:/...`** |
| 5 | 素材必须指向 `/up/` 目录的原始文件，不能用 temp_assets 副本 |
| 6 | 正确子命令：`multimodal2video` (提交) / `query_result` (查询) |
| 7 | `gen_status=fail` 无 `fail_reason` → **继续轮询**，不是真失败！ |
| 8 | 用 Python requests 下载视频，不用 dreamina.exe download |
| 9 | 提示词超过 2000 字符必须截断并确认 |
| 10 | 音频时长必须在 2~15 秒之间，提交前用 ffprobe 验证 |

---

## 📁 相关文件位置

- **本技能文档**: `~/.hermes/skills/devops/jimeng-telegram-seedance2.0/SKILL.md`
- **主操作指令**: `/mnt/j/WORKSPACE/DREAMINA_MODEL_INSTRUCTIONS.md`
- **经验教训文档**: `/mnt/j/WORKSPACE/DREAMINA_WORKFLOW_LESSONS.md`

---

## ⚡ 快速调用模板（下次使用时直接复制）

```bash
# === 预处理图片 ===
mkdir -p /mnt/j/WORKSPACE/temp_assets
python3 -c "
from PIL import Image, os
def compress(wsl_path, name):
    img = Image.open(wsl_path).convert('RGB')
    if max(img.width, img.height) > 1280:
        r = 1280 / max(img.width, img.height)
        img = img.resize((int(img.width*r), int(img.height*r)), Image.LANCZOS)
    img.save(f'/mnt/j/WORKSPACE/temp_assets/{name}', 'JPEG', quality=85)
compress('/mnt/j/WORKSPACE/up/人物设定/XXX.jpg', 'asset_0.jpg')
"

# === 验证音频时长（如需要）===
ffprobe -i "/mnt/j/WORKSPACE/up/声音设定/XXX.wav" \
  -show_entries format=duration -v quiet -of csv="p=0"

# === 提交（基础：仅图片）===
/mnt/c/Users/YOUR_USERNAME/.local/bin/dreamina.exe multimodal2video \
  --prompt "{用户提示词}" \
  --image "J:/WORKSPACE/up/人物设定/{图片1}" \
  --image "J:/WORKSPACE/up/背景设定/{图片2}" \
  --duration {时长} \
  --ratio "{比例}" \
  --model_version "{seedance2.0vip / seedance2.0fast / ...}"

# === 提交（多模态：图片 + 音频）===
/mnt/c/Users/YOUR_USERNAME/.local/bin/dreamina.exe multimodal2video \
  --prompt "{用户提示词}" \
  --image "J:/WORKSPACE/up/人物设定/{图片1}" \
  --audio "J:/WORKSPACE/up/声音设定/{音频文件.wav}" \
  --duration {时长} \
  --ratio "{比例}" \
  --model_version "{模型版本}"

# === 提交（多模态：图片 + 视频）===
/mnt/c/Users/YOUR_USERNAME/.local/bin/dreamina.exe multimodal2video \
  --prompt "{用户提示词}" \
  --image "J:/WORKSPACE/up/人物设定/{图片1}" \
  --video "J:/WORKSPACE/video/{参考视频.mp4}" \
  --duration {时长} \
  --ratio "{比例}" \
  --model_version "{模型版本}"

# === 轮询（每 20 秒）===
/mnt/c/Users/YOUR_USERNAME/.local/bin/dreamina.exe query_result --submit_id={submit_id}

# === 下载 & 发送 ===
MEDIA:/mnt/j/WORKSPACE/video/{submit_id}.mp4
```

---

## 🛡️ 安全规则（必须遵守）

- **所有操作限定在白名单目录** `/mnt/j/WORKSPACE/`
- **确认提交前自动执行** → 禁止！必须等用户回复「确认提交」
- **确认后再次询问「要开始吗？」** → 禁止！确认后应立即执行
- **MEDIA: 后输出任何文字或工具调用** → 禁止！必须是最后一条消息的唯一内容

---

## 📞 遇到问题怎么办？

1. **先查本技能文档的"常见错误与处理方式"章节**
2. **检查路径格式是否正确**（dreamina.exe 参数用 J:/...，Python 文件操作用 /mnt/j/...）
3. **检查子命令是否用对**（multimodal2video / query_result）
4. **如果 gen_status=fail，先看有没有 fail_reason**
5. **如果是音频问题，检查时长是否在 2~15 秒范围内**
6. **实在不行，重新执行整个流程**

---

*本技能文档最后更新：2026-04-17 | 状态：已生产验证成功（含多模态音频/视频支持）*