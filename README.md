# fangxin-image-gen

面向 Claude Code、Codex、OpenClaw、Hermes 等 agent 的中文图片生成 Skill，基于 [放心 API](https://fangxinapi.com) 的 `gpt-image-2`。一句话：**让 AI 帮你画图、改图、合图，配置 5 分钟搞定。**

当前版本：**v1.6.1**

支持的玩法：

- 用一段文字直接出一张图
- 拿一张图改风格、改背景、改细节
- 把两张甚至多张图合成一张（比如两个人合一张合影）
- 参考一张图片 URL，不用先下到本地
- 多 key 自动切换：上游一闹脾气立刻换下一个 key 重试，不打断流程

## 仓库长什么样

仓库根目录就是 Skill 本身，clone 下来直接放进 agent 的 skills 目录就能用：

```text
fangxin-image-gen/          # 仓库根 = Skill 本体
├── README.md               # 你正在看的这个（GitHub 文档）
├── SKILL.md                # 给 AI 看的说明书（Skill 入口）
├── agents/openai.yaml      # Agent 元信息
├── scripts/generate.py     # 实际干活的脚本
├── fangxin-image-gen.zip   # 打包好的版本，方便离线分发
└── .gitignore              # 已忽略 .env / .DS_Store / __pycache__
```

## 三步用起来

### 第 1 步：拿一个放心 API 的 key

1. 打开 https://fangxinapi.com 注册 / 登录。
2. 在控制台创建 API Key（一串以 `sk-` 开头的字符）。
3. **建议优先使用 `AZ` 分组下面的 key**，出图稳定性明显更好；其他分组容易超时或者中途断连。
4. 想要更稳？再多领几把不同分组的 key 当备份，下一步用得上。

### 第 2 步：把 key 写进 `.env`

Skill 启动时会自动加载安装目录下的 `.env`，**不需要改 `~/.zshrc`，也不需要重启终端**：

```bash
SKILL_ROOT=/path/to/your/fangxin-image-gen   # 替换成你装到的位置
mkdir -p "$SKILL_ROOT"
touch "$SKILL_ROOT/.env"
chmod 600 "$SKILL_ROOT/.env"
```

最少写一行（替换成你自己的 key）：

```
FANGXIN_API_KEY=sk-xxxxxxxx
```

**想要主备自动切换**？再加几行：

```
FANGXIN_API_KEY=sk-primary
FANGXIN_API_KEY1=sk-backup-1
FANGXIN_API_KEY2=sk-backup-2
```

脚本会按 `FANGXIN_API_KEY` → `FANGXIN_API_KEY1` → `FANGXIN_API_KEY2` 顺序尝试，遇到 401/403/429/5xx/超时/连接中断时自动切到下一个，整个过程对你无感。

> 不小心把 key 贴在对话里？没关系，Skill 内置一段流程：AI 会自动帮你写到 `.env`、把权限收紧到 600，并提醒你以后别再贴。**但还是建议从一开始就在编辑器里写，别经过聊天**。

### 第 3 步：装 Skill

仓库根目录就是 Skill，直接 clone 到对应 agent 的 skills 目录：

```bash
# Claude Code
git clone https://github.com/metafeng/fangxin-image-gen.git \
  ~/.claude/skills/fangxin-image-gen

# Codex / OpenClaw / Hermes 等：clone 到对方 skill 扫描的目录即可
```

关键点只有一个：**`SKILL.md`、`agents/`、`scripts/` 这三个要保持同级**。

以后想更新只要：

```bash
cd /path/to/your/fangxin-image-gen
git pull
```

装好后重启一下 agent，让它读到新 Skill。

## 怎么用

装好之后直接用大白话描述你想要什么，AI 会自己挑接口、配参数。下面是一些示例。

### 文生图（只有文字）

> 「画一张极简风格的红色纸飞机海报，奶油色背景」
> 「做一张科技感很强的产品发布会主视觉」
> 「给我一张适合公众号封面的插画」

### 改图（你给一张图）

> 「把这张图改成商业海报风格」
> 「保持人物不变，背景换成摄影棚」
> 「保留原构图，整体改成时尚杂志风」

### 合图（你给两张或多张图）

> 「把这两个人合成一张自然的合影」
> 「把第一张的脸放到第二张的场景里」
> 「参考这几张产品图，生成一张新的品牌主视觉」

### 给 URL 也行

> 「用这个链接里的图片重生成一版海报：https://...」
> 「把这两个 URL 的人物合成合影」

Skill 会先把远程图片下载到 `/tmp`，调用完接口立刻删掉，**不会在你的服务器上堆图**。

## 出图要等多久？

简单说：**有点慢，请耐心**。

- 单张图常规耗时 30～120 秒，2K / 4K 或高质量参数可能更久
- 多图编辑还会再久一些
- 脚本设了 7 分钟超时和自动重试，**不要中途取消**
- 看到 `Empty reply from server` / `Connection reset by peer` 这种是上游问题，脚本会**自动切到下一个备用 key 重试**，不用慌
- 如果只配了一个 key 还反复超时，去控制台拿个 `AZ` 分组的 key 加进 `.env` 当备份

## 输出在哪里？

默认目录是 **当前工作目录下的** `./tmp/fangxin-image-gen-output/`，不是系统根目录的 `/tmp`，这样图就出在你正在干活的项目里、抬眼就能看到。

脚本输出的路径会被解析成绝对路径，例如：

```text
[1/1] /Users/me/proj/tmp/fangxin-image-gen-output/image_1745654321_1.png
```

直接复制粘贴就能在文件管理器或编辑器里打开。想换位置：传 `--outdir /absolute/path` 即可。

## 进阶：多 agent 共享同一份配置

如果你在多个 agent（Claude / Codex / OpenClaw 同时装了这个 skill）里想共用一套 key，不想每个安装目录都写一份 `.env`，可以指一个共享文件：

```bash
export FANGXIN_ENV_FILE=/Users/me/.config/fangxin/shared.env
```

读取顺序是：

1. 当前 shell 已 `export` 的环境变量（最高优先级）
2. `FANGXIN_ENV_FILE` 指定的 `.env`
3. 当前 skill 安装目录下的 `.env`

`.env` 只补齐缺失变量，**不会覆盖** shell 里已经 `export` 的值。

## 支持的尺寸

| 尺寸 | 说明 |
|------|------|
| `1024x1024` | 正方形 |
| `1536x1024` | 横版 |
| `1024x1536` | 竖版 |
| `2048x2048` | 2K 正方形 |
| `2048x1152` | 2K 横版 |
| `3840x2160` | 4K 横版 |
| `2160x3840` | 4K 竖版 |
| `auto` | 让模型自己决定 |

## 主要参数（AI 一般会自动填，自己跑脚本时才用得到）

- `--prompt`：提示词（必填）
- `--image`：参考图，可重复多次传多张
- `--mask`：局部编辑用的遮罩图
- `--input-fidelity`：`high`（更像原图）/ `low`（更自由）
- `--size`：尺寸，见上表
- `--quality`：`auto` / `high` / `medium` / `low`
- `--output-format`：`png` / `jpeg`（上游不支持 webp）
- `--outdir`：图片输出目录，默认 `./tmp/fangxin-image-gen-output`（**相对于你当前的工作目录**）

直接命令行跑：

```bash
python3 /path/to/your/fangxin-image-gen/scripts/generate.py \
  --prompt "一只在月亮上钓鱼的猫，水彩风" \
  --size 1024x1536 \
  --quality high
```

## 复用以前生成的图

不需要重新上传原图，只要你还有下面任一引用就行：

- 原始图片 URL
- 上次生成的本地文件路径（默认在当前工作目录的 `tmp/fangxin-image-gen-output/` 下面）
- 你自己业务系统里存的图片地址

直接把这些当 `--image` 参数传进去就 OK。

## 一些"踩过的坑"提醒

- 上游 `gpt-image-2` 返回的是 `b64_json`，**不是 URL**，脚本会自己解码保存
- 不接受 `response_format` 参数，传了会报错
- 不接受 `webp` 输出，只能 `png` 或 `jpeg`
- 编辑接口对"直接传远程 URL"兼容性差，所以脚本会先下载到 `/tmp` 再上传
- 用的是 `curl --http1.1`，因为 HTTP/2 在这家上游偶尔断流
- key 优先选 `AZ` 分组，其他分组稳定性差一截

## 安全约定

- `.env` 已经加进 `.gitignore`，不会被误推到 GitHub
- 建议把 `.env` 权限设成 `600`，只让自己读写
- 别把 key 贴在 Issue、PR、聊天截图、shell history 里
- 脚本日志只会显示脱敏 key（`sk-abcd...WXYZ`），不会泄露完整值
- 如果不小心泄露了，立刻去放心 API 控制台撤销并换一个，再更新到 `.env`

## 反馈

发现 bug 或者想加新能力，欢迎在 [Issues](https://github.com/metafeng/fangxin-image-gen/issues) 里提。
