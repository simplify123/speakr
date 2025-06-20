# Speakr

## Screenshots
![image](https://raw.githubusercontent.com/murtaza-nasir/speakr/master/static/img/hero-shot.png)
![image](https://raw.githubusercontent.com/murtaza-nasir/speakr/master/static/img/multilingual-support.png)

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

## 在 LocalAI 的实现中，Whisper 转录功能是通过集成 whisper.cpp 来实现的，并不是直接使用 WhisperX 作为其引擎，所以不支持多个说话人分离。若需要该功能，则需要部署image: onerahmet/openai-whisper-asr-webservice:latest-gpu
# 🎙️ Speakr + LocalAI Whisper 本地离线语音转写系统部署指南

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

## final result
If you follow all my steps correctly,you should see
![SunnyCapturer2025-06-18_18-30-43.png](https://img.picui.cn/free/2025/06/19/6854038790256.png)
![SunnyCapturer2025-06-19_20-32-18.png](https://img.picui.cn/free/2025/06/19/685403882035c.png)

## ✅ 依赖要求

- Docker >= 20.10
- 若使用 GPU：NVIDIA Container Toolkit
- 若使用 CPU：镜像改为 `localai/localai:latest-aio-cpu`

## 另一个版本：支持人物角色分离，对于中文，推荐使用large-v3模型，准确率更高
```yml
services:
  whisper-asr-webservice6002:
    image: onerahmet/openai-whisper-asr-webservice:latest
    container_name: whisper-asr-webservice6002
    ports:
      - "6002:9000"
    volumes:
      - ./huggingface/hub:/root/.cache/huggingface/hub
    environment:
      - ASR_MODEL=large-v3  # 可选 large-v3、medium、distil-large-v3
      - ASR_COMPUTE_TYPE=int8
      - ASR_ENGINE=whisperx
      - HF_TOKEN=hf_your_huggingface_token_here
      - HF_ENDPOINT=https://hf-mirror.com   # https://hf-mirror.com
      - HF_HOME=/root/.cache/huggingface/
    deploy:
      resources:
        limits:
          memory: 4G
    restart: unless-stopped
    networks:
      - speakr-network

  app:
    image: learnedmachine/speakr:latest
    container_name: speakr8890
    restart: unless-stopped
    ports:
      - "8890:8899"
    environment:
      - TEXT_MODEL_BASE_URL=http://100.64.0.16:11434/v1
      - TEXT_MODEL_API_KEY=111111
      - TEXT_MODEL_NAME=qwen2.5:14b
      - USE_ASR_ENDPOINT=true

      - ASR_BASE_URL=http://100.64.0.16:6002
      - ASR_ENCODE=true
      - ASR_TASK=transcribe
      - ASR_DIARIZE=true
      - ASR_MIN_SPEAKERS=1
      - ASR_MAX_SPEAKERS=5

      - ALLOW_REGISTRATION=false
      - SUMMARY_MAX_TOKENS=8000
      - CHAT_MAX_TOKENS=5000
      - ADMIN_USERNAME=admin
      - ADMIN_EMAIL=admin@alt.org
      - ADMIN_PASSWORD=11111111

      - SQLALCHEMY_DATABASE_URI=sqlite:////data/instance/transcriptions.db
      - UPLOAD_FOLDER=/data/uploads

      # 以下三个变量用于提高默认的上传附件大小
      - MAX_CONTENT_LENGTH=2048 # 2048MB
      - UPLOAD_LIMIT=2147483648 # 字节
      - MAX_UPLOAD_SIZE=2147483648
    volumes:
      - ./uploads:/data/uploads
      - ./instance:/data/instance
    depends_on:
      - whisper-asr-webservice6002
    networks:
      - speakr-network

networks:
  speakr-network:
    driver: bridge
```

### 如果你按照我的代码正确完成了部署，应当可以看到以下结果
![SunnyCapturer2025-06-20_18-43-50.png](https://img.picui.cn/free/2025/06/20/68553b749867e.png)

## 🙌 鸣谢

- [Speakr](https://github.com/murtaza-nasir/speakr)
- [LocalAI](https://github.com/go-skynet/LocalAI)
- [Whisper](https://github.com/openai/whisper)
