---
name: youtube-transcribe
description: >
  Use when user wants to download YouTube subtitles or transcribe YouTube videos.
  Handles both scenarios: videos with existing subtitles and videos without subtitles
  (requires Whisper audio transcription). Includes workarounds for YouTube 403 errors.
triggers:
  - youtube/视频/字幕/转录/下载字幕/transcribe/subtitle
  - yt-dlp/whisper/音频转录
---

# YouTube 字幕下载与转录技能

## 功能概述

1. 下载 YouTube 视频的现有字幕（手动上传或自动生成）
2. 当视频无字幕时，下载音频并使用 Whisper 进行本地转录
3. 处理 YouTube 403 下载错误的解决方案

## 前置依赖

```bash
# 检查 yt-dlp 是否安装
yt-dlp --version

# 检查 whisper 是否安装
pip install openai-whisper

# 检查 ffmpeg 是否安装
ffmpeg -version
```

## 工作流程

### 第一步：尝试下载字幕

```bash
# 下载中英文字幕（不下载视频）
yt-dlp --write-sub --write-auto-sub --sub-lang "zh-Hans,zh,en" --skip-download -o "/tmp/%(id)s" "YouTube_URL"

# 查看可用字幕列表
yt-dlp --list-subs "YouTube_URL"
```

### 第二步：如无字幕，下载音频

```bash
# 方法1：直接下载音频（可能遇到403错误）
yt-dlp -x --audio-format mp3 -o "/tmp/%(id)s.%(ext)s" "YouTube_URL"

# 方法2：使用 web_embedded 客户端绕过403（推荐）
yt-dlp --remote-components ejs:github --extractor-args "youtube:player_client=web_embedded" -f 140 -o "/tmp/%(id)s.%(ext)s" "YouTube_URL"

# 方法3：指定音频格式ID下载
yt-dlp --list-formats "YouTube_URL"  # 查看可用格式
yt-dlp -f 140 -o "/tmp/%(id)s.%(ext)s" "YouTube_URL"  # 下载m4a音频
```

### 第三步：使用 Whisper 转录

```bash
# 基础转录（自动检测语言）
whisper /tmp/VIDEO_ID.m4a --model tiny --output_dir /tmp --output_format txt

# 指定中文转录（更快）
whisper /tmp/VIDEO_ID.m4a --model tiny --language zh --output_dir /tmp --output_format srt

# 使用更大模型获得更准确结果（耗时更长）
whisper /tmp/VIDEO_ID.m4a --model base --language zh --output_dir /tmp --output_format txt
```

### 第四步：文本校正（可选）

语音转录存在同音字错误，建议进行校正：
- 技术术语校正：RAG、向量、切片、权限、检索等
- 同音字修正：权线->权限、相量->向量、检所->检索等

## Whisper 模型选择

| 模型 | 大小 | 速度 | 准确度 |
|------|------|------|--------|
| tiny | 39MB | 最快 | 较低 |
| base | 74MB | 快 | 中等 |
| small | 244MB | 中等 | 较高 |
| medium | 769MB | 慢 | 高 |
| large | 1550MB | 最慢 | 最高 |

## 常见问题

### 403 Forbidden 错误

YouTube 反爬虫机制导致，解决方案：
1. 使用 `--remote-components ejs:github` 参数
2. 使用 `--extractor-args "youtube:player_client=web_embedded"`
3. 使用浏览器 cookies：`--cookies-from-browser chrome`

### 转录超时

Whisper 在 CPU 上转录较慢，可使用：
- 更小的模型（tiny）
- 指定语言减少检测时间
- 增加 bash 超时时间

### 输出文件格式

- `.txt`：纯文本转录
- `.srt`：带时间戳的字幕文件
- `.vtt`：WebVTT 字幕格式

## 输出位置

- 音频文件：`/tmp/VIDEO_ID.m4a`
- 转录文件：`/tmp/VIDEO_ID.txt` 或 `/tmp/VIDEO_ID.srt`
- 项目存储：`/data/youtube/项目名-转录稿.txt`

## 使用示例

```bash
# 完整流程示例
VIDEO_URL="https://www.youtube.com/watch?v=VIDEO_ID"

# 1. 尝试下载字幕
yt-dlp --write-sub --write-auto-sub --sub-lang "zh-Hans,zh,en" --skip-download -o "/tmp/%(id)s" "$VIDEO_URL"

# 2. 如果无字幕，下载音频
yt-dlp --remote-components ejs:github --extractor-args "youtube:player_client=web_embedded" -f 140 -o "/tmp/%(id)s.%(ext)s" "$VIDEO_URL"

# 3. 转录
whisper /tmp/VIDEO_ID.m4a --model tiny --language zh --output_dir /tmp --output_format txt

# 4. 查看结果
cat /tmp/VIDEO_ID.txt
```