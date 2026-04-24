# fangxin-image-gen-skill

一个面向 Codex / Claude Skills 的中文图片生成 Skill，基于放心API `fangxinapi.com` 的图片接口封装，统一支持：

- 文生图
- 以图生图
- 多图参考合成
- 合影合成
- 图片 URL 输入
- 临时文件下载与自动清理

仓库内容：

- `fangxin-image-gen/`：Skill 目录
- `fangxin-image-gen.zip`：可直接分发的打包文件

## 功能说明

这个 Skill 做了两条能力路由：

- 只提供文字：自动走 `POST /v1/images/generations`
- 提供一张或多张图片：自动走 `POST /v1/images/edits`

已经实测通过的能力：

- `gpt-image-2` 文生图
- 单图编辑
- 多图同时上传参考
- 两张图片合成一张合影

## 适用场景

- 画一张海报、插画、封面
- 把一张原图改成另一种风格
- 两张人物图合成一张合影
- 多张参考图融合成一张新图
- 用户只提供图片 URL，不希望把图片长期存到服务器本地

## 如何安装

你有两种安装方式。

### 方式一：从 GitHub 直接安装

如果你的环境支持 `skill-installer`，推荐直接用这个 GitHub 路径安装：

```bash
scripts/install-skill-from-github.py \
  --url https://github.com/metafeng/fangxin-image-gen-skill/tree/main/fangxin-image-gen
```

或者使用 repo/path 形式：

```bash
scripts/install-skill-from-github.py \
  --repo metafeng/fangxin-image-gen-skill \
  --path fangxin-image-gen
```

安装完成后，重启 Codex / Claude 以加载新 Skill。

### 方式二：手动安装

把仓库里的 `fangxin-image-gen/` 放到你的 Skills 目录中即可。

如果你是直接使用打包文件，可以解压 `fangxin-image-gen.zip`，得到：

```text
fangxin-image-gen/
  SKILL.md
  agents/openai.yaml
  scripts/generate.py
```

## 如何配置 API Key

这个 Skill 不再内置默认 key，必须由你自己提供：

```bash
/update-config set FANGXIN_API_KEY=<your-key>
```

可选模型配置：

```bash
/update-config set FANGXIN_MODEL=gpt-image-2
```

## API Key 从哪里获取

按当前这套接入方式，你需要在 **放心API** 后台获取自己的 key。

官网：

- https://fangxinapi.com

使用时建议按当前实测方式确认这几项：

1. 在 放心API 网站后台创建你自己的 API Key
2. 使用或选择 `AZ` 分组
3. 确认可调用的图片模型是 `gpt-image-2`

也就是说，当前推荐的组合是：

- 分组：`AZ`
- 模型：`gpt-image-2`

说明：

- 这里写的是当前这套 Skill 的实测接入方式
- 如果放心API后台以后改了分组名、模型可用性或价格策略，需要以后台实际情况为准

## 支持的尺寸

- `1024x1024`：正方形
- `1536x1024`：横版
- `1024x1536`：竖版
- `2048x2048`：2K 正方形
- `2048x1152`：2K 横版
- `3840x2160`：4K 横版
- `2160x3840`：4K 竖版
- `auto`：由模型自动决定

## 使用案例

### 1. 文生图

如果你只是想直接出一张图，不需要上传任何参考图片，只要告诉它你想画什么就可以。

例如，你可以直接用自然语言描述：

- 画一张极简风格的红色纸飞机海报，背景是奶油色
- 做一张科技感很强的产品发布会主视觉
- 生成一张适合文章封面的插画

这类请求会自动走文生图接口。

### 2. 单图编辑

如果你已经有一张图片，想在原图基础上修改风格、重绘细节、提升质感，直接把这张图作为参考图传进去即可。

例如：

- 把这张图改成更干净的商业海报风格
- 保持人物不变，把背景换成摄影棚
- 保留原构图，整体改成高级时尚杂志风格

这类请求会自动走图片编辑接口。

### 3. 多图合成 / 合影合成

如果你有两张或多张参考图，希望把它们合成到一张图里，也可以直接用自然语言描述目标结果。

例如：

- 把两张图中的人物合成一张自然真实的双人合影
- 把第一个人的脸和第二张图的场景融合在一起
- 参考多张产品图，生成一张新的品牌主视觉

已经实测通过的场景包括：

- 双人合影合成
- 多图参考融合
- 一张主图加多张风格参考图

### 4. 直接传图片 URL

如果你不想把图片长期保存在服务器本地，可以直接给图片 URL。

例如：

- 用这个链接里的图片重新生成一版海报
- 把这两个 URL 对应的人物合成一张合影
- 参考这个 OSS 链接里的图片做风格迁移

对调用方来说，使用体验仍然是“直接传 URL”；Skill 内部会自动下载临时文件、请求完成后再删除。

## URL 输入的处理逻辑

放心API的编辑接口对“直接远程 URL 作为编辑输入”兼容性不稳定，所以这个 Skill 做了一层兼容：

- 如果传的是本地文件：直接上传
- 如果传的是图片 URL：先下载到 `/tmp` 临时文件
- 调用编辑接口
- 请求结束后自动删除临时文件

这样做的好处：

- 外部调用仍然可以只传 URL
- 服务器不会长期囤积图片
- 不需要把所有用户图片永久保存到本地磁盘

## 如何复用旧图片

如果用户以后想基于旧图重新生成，不一定要重新上传原文件。

只要你还保留下面任意一种引用，就可以继续复用：

- 原始图片 URL
- 上一次生成结果的本地文件路径
- 你自己系统里保存的历史图片路径

推荐做法：

- 如果是一次性处理：传 URL，让脚本自动下载到临时文件并清理
- 如果需要以后重复使用：在你的业务系统里保存原图 URL 或历史结果路径

## 关键参数

- `--prompt`：提示词，必填
- `--image`：参考图，可以重复传多次
- `--mask`：局部编辑用遮罩图
- `--input-fidelity`：编辑时对原图的保真程度，可选 `high` / `low`
- `--size`：出图尺寸
- `--quality`：出图质量，可选 `auto` / `high` / `medium` / `low`
- `--output-format`：输出格式，当前建议 `png` 或 `jpeg`
- `--outdir`：输出目录

## 当前接入结论

基于当前实测，`fangxinapi.com` 这条链路建议这样使用：

- 模型：`gpt-image-2`
- 文生图：可用
- 单图编辑：可用
- 多图参考：可用
- 合影合成：可用
- URL 输入：通过“临时下载后上传”的兼容层可用

## 仓库结构

```text
fangxin-image-gen-skill/
  README.md
  fangxin-image-gen.zip
  fangxin-image-gen/
    SKILL.md
    agents/openai.yaml
    scripts/generate.py
```

其中用于直接安装的 Skill 路径是：

```text
https://github.com/metafeng/fangxin-image-gen-skill/tree/main/fangxin-image-gen
```

## 主文件说明

- `fangxin-image-gen/SKILL.md`：Skill 定义与使用说明
- `fangxin-image-gen/scripts/generate.py`：统一入口脚本
- `fangxin-image-gen/agents/openai.yaml`：Agent 元信息

## 注意事项

- 这个 Skill 不自带默认 API key
- 没有配置 `FANGXIN_API_KEY` 时会直接报错
- 放心API上游偶尔会出现 TLS 中断或超时，脚本里已经做了基础重试
- 某些尺寸或模型策略如果后续在放心API后台变化，需要按实际可用情况调整
