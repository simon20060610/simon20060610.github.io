---
title: Toy2.0：基于 Raspberry Pi 的桌宠项目记录
date: 2026-06-19
categories:
  - DevLog
tags:
  - Raspberry Pi
  - PyQt5
  - Computer Vision
  - LLM
  - TTS
  - Desktop Companion
toc: true
comments: false
---

Toy2.0 是一个运行在 Raspberry Pi / Linux 桌面环境上的桌宠项目。它通过摄像头判断使用者是否在座位前，结合课表、时间段和状态提醒生成维尔汀（Vertin）风格的短句，并通过 TTS 播放语音。

这个项目的目标不是做一个单纯的提醒工具，而是尝试把“桌面陪伴感”和真实环境感知结合起来：设备能看到我是否在座位前，能知道现在大概是什么时间段，也能基于课表和运行状态给出更贴近场景的反馈。

<!-- more -->

## 项目概览

Toy2.0 当前主要由几部分组成：

- Raspberry Pi 5 作为运行设备
- Raspberry Pi Camera Module 或可被 Picamera2 / libcamera 访问的摄像头
- PyQt5 负责图形界面
- OpenCV / MediaPipe 负责视觉检测
- DeepSeek LLM 负责生成回复文本
- MiniMax TTS 负责语音合成
- `.ics` 课表文件提供日程信息

背景图片使用《重返未来：1999》官网背景动图，声音基于维尔汀在游戏中的语音录屏内容使用 MiniMax 训练而成。

## 安全说明

仓库中不应保存真实 API Key、私人课表、运行记忆或生成音频。以下内容都应该只保存在本地环境中：

- `.env`
- `TOY/data/`
- `reply.mp3`
- 私人 `.ics` 课表文件
- 任何真实 API Key

公开仓库只保留示例配置和格式说明。这样项目可以公开展示，也不会把私人配置和运行数据一起暴露出去。

## 硬件设备

项目当前使用或预期支持的硬件如下：

- Raspberry Pi 5
- Raspberry Pi Camera Module，或可被 Picamera2 / libcamera 访问的摄像头
- 扬声器、耳机或 HDMI / USB 音频输出设备
- 可显示 PyQt5 图形界面的屏幕
- 可访问 DeepSeek LLM 和 MiniMax TTS 的网络环境

## 运行环境

推荐系统与 Python 环境：

```bash
Raspberry Pi OS
conda Python 3.11
系统 Python 3.13 + python3-picamera2
```

项目主程序运行在 conda 环境 `toy_env` 中。由于 Raspberry Pi OS 的 `picamera2/libcamera` 通常绑定系统 Python ABI，conda Python 3.11 可能无法直接 `import Picamera2`。

为了解决这个问题，项目内置了外部相机 helper：

```text
conda Python 3.11 主程序
-> 如果无法 import Picamera2
-> 启动 /usr/bin/python3 TOY/picamera_capture.py
-> 系统 Python 负责采集相机帧
-> 原始 RGB 帧通过 stdout 传回主程序
```

正常启动时可能看到类似日志：

```text
[Camera] Picamera2 is not importable in this Python...
[Camera] External Picamera2 helper started.
[Vision] Using MediaPipe Face Detection
```

## 安装流程

创建并进入 conda 环境：

```bash
conda create -n toy_env python=3.11 -y
conda activate toy_env
cd ~/Toy2.0
```

安装 Python 依赖：

```bash
python -m pip install -r requirements-toy-env.txt
```

当前 `requirements-toy-env.txt` 包含：

```text
PyQt5
flask
ics
mediapipe
openai
opencv-contrib-python-headless==4.11.0.86
pygame
requests
```

检查依赖：

```bash
python tools/check_deps.py
```

如果缺包，可以自动安装缺失的 pip 包：

```bash
python tools/check_deps.py --install
```

安装 Raspberry Pi 系统相机包：

```bash
sudo apt install -y python3-picamera2
```

注意：`python3-picamera2` 是系统 Python 包，不一定能被 conda 直接导入。项目会在 conda 无法导入 Picamera2 时自动使用 `/usr/bin/python3 TOY/picamera_capture.py` 采集相机画面。

## 配置

复制示例配置：

```bash
cp .env.example .env
```

然后在 `.env` 中填写自己的密钥和本地路径：

```text
DEEPSEEK_API_KEY=
DEEPSEEK_BASE_URL=https://api.deepseek.com
DEEPSEEK_MODEL=deepseek-v4-flash

MINIMAX_GROUP_ID=
MINIMAX_API_KEY=
MINIMAX_TTS_MODEL=speech-01-turbo
MINIMAX_VOICE_ID=Vertin_clone

COMPANION_NAME=companion
SCHEDULE_FILE=schedule.ics
DATA_DIR=data
REPLY_AUDIO_FILE=reply.mp3
```

课表可以放在 `TOY/schedule.ics`，也可以通过 `SCHEDULE_FILE` 指向其他本地 `.ics` 文件。仓库中提供了 `TOY/example_schedule.ics` 作为格式参考。

## 使用说明

正常启动：

```bash
conda activate toy_env
cd ~/Toy2.0
python TOY/main.py
```

默认全屏：

```text
START_FULLSCREEN=1
```

临时使用小窗口调试：

```bash
START_FULLSCREEN=0 python TOY/main.py
```

切换视觉后端：

```bash
FACE_DETECT_BACKEND=mediapipe python TOY/main.py
FACE_DETECT_BACKEND=yunet python TOY/main.py
FACE_DETECT_BACKEND=haar python TOY/main.py
```

调整 MediaPipe 置信度：

```bash
FACE_DETECT_CONFIDENCE=0.75 python TOY/main.py
```

配置忽略区域，例如屏蔽墙上告示牌区域：

```bash
FACE_IGNORE_ZONES="0.65,0.10,0.25,0.30" python TOY/main.py
```

忽略区域格式：

```text
x,y,w,h
```

小于等于 `1.0` 的值会按画面比例解释，也可以写像素值。多个区域用分号分隔：

```bash
FACE_IGNORE_ZONES="0.65,0.10,0.25,0.30;20,30,180,120" python TOY/main.py
```

## 视觉方案

项目入口是：

```bash
python TOY/main.py
```

视觉线程位于：

```text
TOY/vision_thread.py
```

当前默认视觉后端：

```text
FACE_DETECT_BACKEND=mediapipe
```

整体链路：

```text
Picamera2 / 外部相机 helper
-> 获取 640x480 RGB 帧
-> 灰度/颜色预处理
-> MediaPipe Face Detection 检测人脸
-> 过滤异常框：尺寸比例、最大尺寸、忽略区域等
-> OpenCV 在画面上画识别框
-> 转成 QImage
-> PyQt5 QLabel 显示到左上角摄像头小窗
-> presence_score 做“人在/离开”的状态判断
```

备选后端仍保留：

```bash
FACE_DETECT_BACKEND=yunet python TOY/main.py
FACE_DETECT_BACKEND=haar python TOY/main.py
```

YuNet 使用模型：

```text
TOY/face_detection_yunet_2023mar.onnx
```

MediaPipe 不可用时会自动退回旧方案。YuNet 模型缺失时也会退回 Haar。

## OpenCV 说明

项目使用：

```text
opencv-contrib-python-headless==4.11.0.86
```

选择 headless 版本的原因：

```text
主程序不使用 cv2.imshow
画面显示由 PyQt5 完成
OpenCV 只负责图像处理、画框、DNN/YuNet
headless 可以避免 cv2 自带 Qt 插件和 PyQt5 冲突
```

检查 OpenCV：

```bash
python -c "import cv2; print(cv2.__version__)"
```

应输出类似：

```text
4.11.0
```

## 常用环境变量

窗口与显示：

```text
START_FULLSCREEN=1
WINDOW_WIDTH=320
WINDOW_HEIGHT=480
DRAWER_REFRESH_SECONDS=10
BACKGROUND_MEDIA_FILE=main_ui.gif
```

相机与颜色：

```text
CAMERA_FORMAT=RGB888
CAMERA_COLOR_ORDER=BGR
CAMERA_GRAY_WORLD_STRENGTH=0.65
CAMERA_COLOUR_GAINS=
```

人脸检测：

```text
FACE_DETECT_BACKEND=mediapipe
FACE_DETECT_CONFIDENCE=0.65
FACE_IGNORE_ZONES=
FACE_DETECT_MIN_SIZE=48
FACE_DETECT_MAX_SIZE_RATIO=0.75
FACE_DETECT_MIN_ASPECT=0.75
FACE_DETECT_MAX_ASPECT=1.35
```

人在 / 离开状态判断：

```text
AWAY_THRESHOLD_SECONDS=12.0
RETURN_TRIGGER_MIN_AWAY_SECONDS=20.0
PRESENCE_SCORE_MAX=10.0
PRESENCE_SEEN_GAIN=2.5
PRESENCE_MISSED_DECAY=0.45
PRESENCE_LEAVE_SCORE=0.5
```

久坐与睡眠提醒：

```text
SIT_LIMIT_SECONDS=3600
REMINDER_COOLDOWN_SECONDS=1800
SLEEP_REMINDER_FIRST_HOUR=22
SLEEP_REMINDER_FIRST_MINUTE=0
SLEEP_REMINDER_SECOND_HOUR=22
SLEEP_REMINDER_SECOND_MINUTE=30
```

## 故障排查

如果启动时报缺少依赖：

```bash
python tools/check_deps.py
python tools/check_deps.py --install
```

如果 Picamera2 在 conda 中无法导入，先确认系统包存在：

```bash
sudo apt install -y python3-picamera2
```

如果 OpenCV 和 PyQt5 出现 Qt 插件冲突，重新安装 headless 版本：

```bash
python -m pip uninstall -y opencv-python opencv-contrib-python opencv-python-headless opencv-contrib-python-headless
python -m pip install --no-cache-dir opencv-contrib-python-headless==4.11.0.86
```

如果 AI 或 TTS 无响应，需要检查 `.env` 中的 DeepSeek 和 MiniMax 配置，并确认设备联网情况。

## 小结

Toy2.0 目前更像是一个把多个技术点串起来的桌面陪伴实验：它涉及 Raspberry Pi 相机、PyQt5 桌面界面、OpenCV / MediaPipe 视觉检测、LLM 文本生成、TTS 语音输出和课表数据解析。

后续如果继续完善，我希望它不只是“能说话的桌宠”，而是能逐渐拥有更稳定的场景感知能力，并且在提醒、陪伴和日常记录之间找到一个自然的平衡。
