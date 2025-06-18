# Speakr

[![License: AGPL v3](https://img.shields.io/badge/License-AGPL_v3-blue.svg)](https://www.gnu.org/licenses/agpl-3.0) [![Docker Build and Publish](https://github.com/murtaza-nasir/speakr/actions/workflows/docker-publish.yml/badge.svg)](https://github.com/murtaza-nasir/speakr/actions/workflows/docker-publish.yml) 

This project is dual-licensed. See the [License](#license) section for details.

Speakr is a personal, self-hosted web application designed for transcribing audio recordings (like meetings), generating concise summaries and titles, and interacting with the content through a chat interface. Keep all your meeting notes and insights securely on your own server.

## Screenshots
![image](static/img/speakr1.png)
![image](static/img/speakr2.png)
![image](static/img/speakr3.png)
![image](static/img/speakr4.png)

## Features

**Core Functionality:**

* **Audio Upload:** Upload audio files (MP3, WAV, M4A, etc.) via drag-and-drop or file selection.
* **Background Processing:** Transcription and summarization happen in the background without blocking the UI.
* **Multilingual Support:** User-configurable languages for both audio transcription (input) and AI-generated content like summaries and chat (output).
* **Transcription:** Uses OpenAI-compatible Speech-to-Text (STT) APIs (configurable, e.g., self-hosted Whisper). Transcription language can be set per user.
* **AI Summarization & Titling:** Generates concise titles and summaries using configurable LLMs via OpenAI-compatible APIs (like OpenRouter). Features an improved default summarization prompt, and users can provide their own custom prompt. Output language can be set per user.
* **Interactive Chat:** Ask questions and interact with the transcription content using an AI model. Incorporates user's professional context (name, title, company if provided by the user) for more relevant responses. Chat output language can be set per user.
* **Search, Inbox & Highlight:** For highlighting and easy processing
* **Metadata Editing:** Edit titles, participants, meeting dates, summaries, and notes associated with recordings.

**User Features:**

* **Authentication:** Secure user registration and login system.
* **Account Management:** Users can change their passwords and manage preferences on their Account page, including:
    * Setting transcription and output languages.
    * Defining a custom summarization prompt.
    * Adding personal/professional information (name, title, company) to enhance chat context and AI interactions.
* **Recording Gallery:** View, manage, and access all personal recordings.
* **Dark Mode:** Switch between light and dark themes.

**Admin Features:**

* **Admin Dashboard:** Central place for administration tasks (`/admin`).
* **User Management:** Add, edit, delete users, and grant/revoke admin privileges.
* **System Statistics:** View overall usage statistics (total users, recordings, storage, etc.).

## Technology Stack

* **Backend:** Python 3, Flask
* **Database:** SQLAlchemy ORM with SQLite (default)
* **WSGI Server:** Gunicorn
* **AI Integration:** OpenAI Python library (compatible with various endpoints like OpenAI, OpenRouter, or local servers)
* **Authentication:** Flask-Login, Flask-Bcrypt, Flask-WTF
* **Frontend:** Jinja2 Templates, Tailwind CSS, Vue.js (for interactive components), Font Awesome
* **Deployment:** 
  * Bash script (`setup.sh`) for Linux/Systemd environments
  * Docker with automated builds via GitHub Actions


# 🎙️ Speakr + LocalAI Whisper 实时语音转写系统部署指南

本项目集成了 [Speakr](https://github.com/murtaza-nasir/speakr) 和 [LocalAI](https://github.com/go-skynet/LocalAI)，实现了基于 Whisper 的本地离线音频转写功能，支持中文和英文录音的转写，并提供网页端界面。

---

## 📦 项目结构

 ├── docker-compose.yml
 ├── models/
 │      └── whisper/
 │               └── ggml-model.bin
 │               └── model.yaml
 ├── uploads/         # Speakr 用户上传内容存储路径
 └── instance/        # Speakr 数据库与配置文件路径

## 🚀 快速启动

### 1. 下载模型文件
```yaml

修改 `models/whisper/model.yaml` 为：


name: whisper-medium
backend: whisper
parameters:
  model: whisper/ggml-model.bin

usage: |
    ## example audio file
    wget --quiet --show-progress -O gb1.ogg https://upload.wikimedia.org/wikipedia/commons/1/1f/George_W_Bush_Columbia_FINAL.ogg
    curl http://localhost:8080/v1/audio/transcriptions -H "Content-Type: multipart/form-data" -F file="@$PWD/test.wav" -F model="whisper-medium"

download_files:
- filename: "ggml-model.bin"
  sha256: "a31f25a435c4f4c6577ccd883479273e5851372fc87ca7af9e663f47997f09ab"
  uri: "https://hf-mirror.com/mosesdaudu/whisper-medium-GGML/resolve/main/ggml-model.bin?download=true"
  
```
然后运行下载命令：

```bash
mkdir -p models/whisper
cd models/whisper
wget -O ggml-model.bin "https://hf-mirror.com/mosesdaudu/whisper-medium-GGML/resolve/main/ggml-model.bin?download=true"
```
### 2. 启动服务
```bash
docker network create speakr_shared_network  # 如果未创建过网络
docker compose up -d
```

🚨 如果使用 GPU，请确保你本机支持 NVIDIA Container Toolkit 和 CUDA 驱动。



### 3.🧾 `docker-compose.yml` 配置


```yaml
version: "3.9"
services:
  speakr:
    image: learnedmachine/speakr:latest
    container_name: speakr
    restart: always
    ports:
      - "8899:8899"
    volumes:
      - ./uploads:/data/uploads
      - ./instance:/data/instance
    environment:
      - SQLALCHEMY_DATABASE_URI=sqlite:////data/instance/transcriptions.db
      - UPLOAD_FOLDER=/data/uploads
      - TEXT_MODEL_BASE_URL=http://100.64.0.16:11434/v1  # 确保从speakr容器中能通过 curl http://100.64.0.16:11434/v1/models 获取到模型列表
      - TEXT_MODEL_API_KEY=111111
      - TEXT_MODEL_NAME=qwen2.5:14b
      - TRANSCRIPTION_BASE_URL=http://whisper_asr_container:8080/v1
      - TRANSCRIPTION_API_KEY=111111
      - WHISPER_MODEL=whisper-medium  # whisper-large-v3 这里和 models/whisper/model.yaml 文件中的name保持一致
      - ALLOW_REGISTRATION=false
      - ADMIN_USERNAME=admin
      - ADMIN_EMAIL=admin@alt.org
      - ADMIN_PASSWORD=11111111
      - MAX_CONTENT_LENGTH=2048 # 2048MB
    networks:
      - speakr_shared_network

  whisper_asr_container:
    image: localai/localai:latest-aio-gpu-nvidia-cuda-12
    container_name: whisper_asr_container
    ports:
      - "6001:8080"
    volumes:
      - ./models:/build/models
    environment:
      - MODELS_DIR=/build/models
      - THREADS=4
      - ENABLE_WHISPER=true
      - UPLOAD_LIMIT=2147483648 # 268435456字节=256MB
      - MAX_UPLOAD_SIZE=2147483648 # 2048MB
    deploy:
      resources:
       limits:
          memory: 4G  # 确保有足够内存处理大文件
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    runtime: nvidia
    networks:
      - speakr_shared_network

networks:
  speakr_shared_network:
    external: true
```

###  4.📺 使用方式
1、打开浏览器访问 http://127.0.0.1:8899

2、登录账户：admin@alt.org / 11111111

3、点击 + New Recording 上传音频文件

4、成功后页面右侧将显示自动生成的文本

###  5.🧪 测试音频转写 API
```bash
curl http://localhost:6001/v1/audio/transcriptions \
     -H "Content-Type: multipart/form-data" \
     -F file="@$PWD/test.wav" \
     -F model="whisper-medium"
```
```bash
在localai的portainer  bash中测试成功,说明whisper本身的语音转写服务没问题
curl http://localhost:8080/v1/audio/transcriptions -H "Content-Type: multipart/form-data" -F file="@$PWD/test.wav" -F model="whisper-medium"
```

## ✅ 依赖要求

- Docker >= 20.10
- 若使用 GPU：NVIDIA Container Toolkit
- 若使用 CPU：镜像改为 `localai/localai:latest-aio-cpu`

## 🙌 鸣谢

- [Speakr](https://github.com/murtaza-nasir/speakr)
- [LocalAI](https://github.com/go-skynet/LocalAI)
- [Whisper](https://github.com/openai/whisper)
