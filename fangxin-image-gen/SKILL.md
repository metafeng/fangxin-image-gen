---
name: fangxin-image-gen
description: AI image generation and image editing via fangxinapi.com using gpt-image-2. Supports text-to-image, image-to-image, multi-image composite edits, masks, PNG/JPEG output, and quality/background controls. Use when the user wants to generate images, edit uploaded images, merge multiple reference photos, or create illustrations. Triggered by phrases like "画一张", "生成图片", "改图", "合成照片", "draw", "generate image", "edit image", "image to image".
---

# Fangxin Image Generation

v1.3.0

## Configuration

脚本通过环境变量读取配置，支持两个变量：

- `FANGXIN_API_KEY` — Bearer token（必填，skill 不内置默认值）
- `FANGXIN_MODEL` — 模型名（默认 `gpt-image-2`）

`scripts/generate.py` 启动时会自动从下面的 `.env` 文件加载 KEY=VALUE 到 `os.environ`（已经在 shell 中 `export` 的值优先，不会被覆盖）：

```
~/.claude/skills/fangxin-image-gen/.env
```

`.env` 内容示例：

```
FANGXIN_API_KEY=sk-xxxxxxxx
# 可选：指定模型
# FANGXIN_MODEL=gpt-image-2
```

### 🔑 第一次配置 API Key

如果脚本报 `FANGXIN_API_KEY is not set` 或检测到该变量为空，请按下面的步骤引导用户：

1. 前往 **放心 API** 官网注册并获取 API Key：https://fangxinapi.com
2. 在控制台创建应用 / 复制 key（格式通常是 `sk-xxxxxxxx`）。**优先选用 AZ 分组下的 key，出图稳定性明显更好**；其他分组可能出现超时或连接中断。
3. 把 key 写入 `~/.claude/skills/fangxin-image-gen/.env`，建议同时把权限收紧到 600：

```bash
mkdir -p ~/.claude/skills/fangxin-image-gen
[ -f ~/.claude/skills/fangxin-image-gen/.env ] || touch ~/.claude/skills/fangxin-image-gen/.env
chmod 600 ~/.claude/skills/fangxin-image-gen/.env
# 然后用任意编辑器写入：FANGXIN_API_KEY=sk-xxxxxxxx
```

写入后无需重启 shell，下一次调用 `generate.py` 就会自动加载。

### 🛟 用户把 key 粘贴到聊天里时怎么办

如果用户在对话里直接发了一段形如 `sk-xxxxxxxx` 的 key（无论是否带前后缀），**不要把它原样回显或长期保存在对话里**。按下面的流程处理：

1. 立刻把这个 key 写入 `.env`，覆盖（或新增）`FANGXIN_API_KEY` 这一行；如果文件不存在就创建，并 `chmod 600`。可以用一段 shell 命令完成，例如：

```bash
ENV_FILE=~/.claude/skills/fangxin-image-gen/.env
mkdir -p "$(dirname "$ENV_FILE")"
touch "$ENV_FILE"
chmod 600 "$ENV_FILE"
# 用 awk 替换或追加 FANGXIN_API_KEY 这一行（KEY 通过环境变量传入，避免出现在命令行）
FANGXIN_API_KEY="<paste-here>" awk -v k="FANGXIN_API_KEY" -v v="$FANGXIN_API_KEY" '
  BEGIN{found=0}
  $0 ~ "^"k"=" {print k"="v; found=1; next}
  {print}
  END{if(!found) print k"="v}
' "$ENV_FILE" > "$ENV_FILE.tmp" && mv "$ENV_FILE.tmp" "$ENV_FILE"
```

2. 写入成功后，用一句话告诉用户：「已经把 key 保存到 `~/.claude/skills/fangxin-image-gen/.env`，下次直接用就行；以后不要把 key 贴在聊天里，可能被日志或其他工具收录。」
3. 不要在回复里复述完整的 key（最多展示前 4 位 + `...` + 后 4 位用于确认）。
4. 然后继续执行用户原本的图片生成请求。

## ⏳ 出图耗时提醒

fangxinapi.com 的 `gpt-image-2` 在高质量 / 2K 、4K 尺寸下单张耗时可能达到 30～120 秒，多图编辑会更久。调用脚本时：

- 不要提前中断；脚本已设 `--max-time 420` 并内置重试，耐心等到脚本返回即可。
- 遇到 `Empty reply from server` / `Connection reset by peer` / `SSL_ERROR_SYSCALL` 之类报错是服务端问题，脚本会自动重试，不要主动改参数抢跑。
- 如果用户表示「怎么还没出」，先告诉他处于正常等待（预计还需 · · · 秒），而不是重启任务。
- 如果反复超时且当前不是 AZ 分组的 key，可以提醒用户去控制台切换到 AZ 分组的 key 后重试。

## Supported Sizes

- `1024x1024` — 正方形
- `1536x1024` — 横版
- `1024x1536` — 竖版
- `2048x2048` — 2K 正方形
- `2048x1152` — 2K 横版
- `3840x2160` — 4K 横版
- `2160x3840` — 4K 竖版
- `auto` — 让模型自行决定

## Usage

### Text to image

```bash
python3 ~/.claude/skills/fangxin-image-gen/scripts/generate.py \
  --prompt "your prompt here" \
  [--model gpt-image-2] \
  [--size 1024x1024] \
  [--n 1] \
  [--quality auto] \
  [--background auto] \
  [--output-format png] \
  [--output-compression 100] \
  [--moderation auto] \
  [--outdir /tmp/fangxin-image-gen-output] \
  [--retries 3]
```

### Image edit / image-to-image / multi-image merge

```bash
python3 ~/.claude/skills/fangxin-image-gen/scripts/generate.py \
  --prompt "combine both people into one photo" \
  --image /path/to/photo-1.png \
  --image /path/to/photo-2.png \
  [--mask /path/to/mask.png] \
  [--input-fidelity high] \
  [--model gpt-image-2] \
  [--size 1024x1024] \
  [--quality high] \
  [--output-format png] \
  [--outdir /tmp/fangxin-image-gen-output] \
  [--retries 3]
```

`--image` accepts either local file paths or remote URLs. Repeat `--image` to pass multiple references in one request.

When a URL is provided, the script downloads it to a temporary file under `/tmp`, calls the Fangxin edit endpoint with file uploads, then deletes the temporary file after the request completes. This avoids keeping long-lived copies on your server while still working around Fangxin's lack of direct URL support for edits.

## Parameters

| Param | Default | Notes |
|-------|---------|-------|
| `--prompt` | required | Image description or edit instruction |
| `--model` | `gpt-image-2` | Default and preferred model on this provider |
| `--image` | none | Reference image path or URL. URLs are downloaded to temp files automatically |
| `--mask` | none | Optional mask path or URL for edit mode |
| `--input-fidelity` | `high` | Edit-mode fidelity to the source images: `high` or `low` |
| `--size` | `auto` | Output size: `auto`, `1024x1024`, `1536x1024`, `1024x1536`, `2048x2048`, `2048x1152`, `3840x2160`, `2160x3840` |
| `--n` | `1` | Number of output images |
| `--quality` | `auto` | Quality level: `auto`, `high`, `medium`, `low` |
| `--background` | `auto` | Background: `auto`, `transparent`, `opaque` |
| `--output-format` | `png` | Supported on this provider: `png`, `jpeg` |
| `--output-compression` | `100` | Compression level 0-100 |
| `--moderation` | `auto` | Content moderation: `auto`, `low` |
| `--style` | none | Only for `dall-e-3`; not supported for `gpt-image-2` |
| `--user` | none | Optional end-user identifier |
| `--outdir` | `/tmp/fangxin-image-gen-output` | Directory where decoded image files are saved |
| `--retries` | `3` | Retry count when the provider closes the connection or drops TLS |

## Request Routing

The script now routes automatically based on inputs:

- No `--image`: uses `POST /v1/images/generations`
- One or more `--image`: uses `POST /v1/images/edits`
- `--mask` is only meaningful in edit mode

This gives one unified skill for the full image workflow.

## Tested Provider Differences

Compared with the standard OpenAI image API behavior, this provider differs in a few important ways:

- `response_format` is **not** accepted.
- Successful `gpt-image-2` responses return `data[].b64_json`, not image URLs.
- `output_format=webp` is **not** supported by the provider validation. Only `png` and `jpeg` are accepted.
- `style` is rejected as an unknown parameter for `gpt-image-2`.
- The script uses `curl --http1.1` because direct `urllib` requests were observed to disconnect on this endpoint.
- For image edits, this provider expects multipart form uploads and accepts repeated `image` fields for multi-image requests.
- Direct remote URL references were not accepted reliably by the provider, so the script converts URLs into temporary local files before upload.
- `images[]` and `images[0]` style multipart fields were tested and did **not** work on this provider.

## Model Priority

Always use `gpt-image-2` unless the user explicitly requests a different model by name.

## Output Format

The script prints a summary line followed by each saved image path:

```text
mode=edit | model=gpt-image-2 | inputs=2 | size=1024x1024 | quality=high | format=png | n=1 | time=42.1s

[1/1] /tmp/fangxin-image-gen-output/image_....png
      revised_prompt: ...
```

After running, present results to the user like this:

```text
模式: edit | 模型: gpt-image-2 | 输入图: 2 | 尺寸: 1024x1024 | 质量: high | 格式: png | 耗时: 42.1s

生成文件: /tmp/fangxin-image-gen-output/image_....png
```

If `revised_prompt` is present, show it as a note so the user knows the API adjusted their prompt.

## Workflow

1. Check `FANGXIN_API_KEY`：脚本启动时已自动加载 `~/.claude/skills/fangxin-image-gen/.env`。如果仍为空，按上文「🔑 第一次配置 API Key」引导用户；如果用户已经把 key 直接粘贴进对话，按「🛟 用户把 key 粘贴到聊天里时怎么办」自动写入 `.env` 后再继续。
2. If the user only gives text, run generation mode.
3. If the user provides one or more images, run edit mode with repeated `--image` flags.
4. If an image input is a URL, download it to a temp file first and clean it up after the request.
5. Build the command from the user's request.
6. Run the script and capture stdout/stderr.
7. Parse the summary line and each `[i/n] path` line from stdout.
8. Present a clean result block with metadata plus saved file paths.
9. If the script exits non-zero, show the stderr error to the user.

## Reusing Old Images

If a user wants to regenerate from an older image, they do **not** need to upload it manually again as long as they still have one of these:

- The original image URL
- The saved local file path from a previous generation result
- A durable object-storage URL you issued for the image

Recommended practice:

- For transient processing: pass image URLs and let the script download to `/tmp` and clean up automatically
- For repeatable workflows: store the original source URLs or the final generated image URLs/paths in your application database, then feed them back into `--image` later
