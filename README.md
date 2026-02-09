# MiniCPM-o Omni Docker

MiniCPM-o 端到端语音对话系统的完整部署方案。基于 llama.cpp-omni C++ 推理后端，提供 Web 前端 + 后端 + 实时通信的一体化部署。

支持**单工**（一问一答）和**双工**（实时打断）两种对话模式。

## 架构概览

```
┌──────────┐     ┌──────────┐     ┌──────────────┐     ┌─────────────────┐
│ Frontend │────▶│ Backend  │────▶│  LiveKit RTC  │────▶│  C++ 推理服务    │
│ :3000    │     │ :18021   │     │  :7880        │     │  :9060          │
└──────────┘     └────┬─────┘     └──────────────┘     └────────┬────────┘
                      │                                         │
                 ┌────▼─────┐                          ┌────────▼────────┐
                 │  Redis   │                          │  llama-server   │
                 │  :6379   │                          │  :19060 (内部)   │
                 └──────────┘                          └─────────────────┘
```

| 组件 | 说明 | 运行方式 |
|------|------|----------|
| Frontend | Web 界面 | Docker |
| Backend | 业务后端，管理会话与服务注册 | Docker |
| LiveKit | WebRTC 实时音视频通信 | Docker |
| Redis | 服务注册与缓存 | Docker |
| C++ 推理服务 | MiniCPM-o 模型推理（Python HTTP 封装 + llama-server） | 原生进程 |

## 前置条件

- **Docker Desktop** (macOS) 或 Docker Engine (Linux)
- **llama.cpp-omni** 已编译（含 `build/bin/llama-server`）
- **GGUF 模型文件**（LLM、音频、视觉、TTS、Token2Wav）
- **Python 3.10+**

## 快速开始

### 一键部署（推荐）

```bash
./deploy_all.sh \
    --cpp-dir /path/to/llama.cpp-omni \
    --model-dir /path/to/gguf
```

脚本将自动完成：加载 Docker 镜像、配置 LiveKit IP、启动所有容器、安装 Python 依赖、启动推理服务、注册到后端。

使用双工模式：

```bash
./deploy_all.sh \
    --cpp-dir /path/to/llama.cpp-omni \
    --model-dir /path/to/gguf \
    --duplex
```

### 部署参数

| 参数 | 说明 | 必须 | 默认值 |
|------|------|------|--------|
| `--cpp-dir PATH` | llama.cpp-omni 编译后的根目录 | 是 | - |
| `--model-dir PATH` | GGUF 模型目录 | 是 | - |
| `--python PATH` | Python 解释器路径 | 否 | 自动检测 |
| `--simplex` | 单工模式（一问一答） | 否 | 默认 |
| `--duplex` | 双工模式（实时打断） | 否 | - |
| `--port PORT` | 推理服务端口 | 否 | 9060 |

### 手动分步部署

如需更细粒度的控制，参考 [DEPLOY.md](DEPLOY.md) 中的分步指南。

## 项目结构

```
.
├── deploy_all.sh                          # 一键部署脚本
├── docker-compose.yml                     # Docker 服务编排
├── nginx.conf                             # 前端 Nginx 配置
├── DEPLOY.md                              # 详细部署指南
├── cpp_server/
│   ├── minicpmo_cpp_http_server.py        # C++ 推理服务 Python 封装
│   ├── requirements.txt                   # Python 依赖
│   └── README.md                          # 推理服务详细文档
└── omini_backend_code/
    └── config/
        └── livekit.yaml                   # LiveKit 服务配置
```

## 服务端口

| 服务 | 端口 | 说明 |
|------|------|------|
| Frontend | 3000 | Web UI |
| Backend | 18021 / 18022 | 后端 API |
| LiveKit | 7880 / 7881 / 7882 | 实时通信 |
| Redis | 6379 | 缓存 |
| 推理服务 | 9060 | Python HTTP API |
| 健康检查 | 9061 | 独立健康检查（推理期间可用） |
| llama-server | 19060 | C++ 内部服务 |

## 支持平台

| 平台 | GPU 加速 | 状态 |
|------|----------|------|
| macOS (Apple Silicon) | Metal | 已测试 |
| Linux (NVIDIA GPU) | CUDA | 支持 |

## 模型文件

```
<MODEL_DIR>/
├── MiniCPM-o-4_5-Q4_K_M.gguf          # LLM 主模型 (~5GB)
├── audio/                              # 音频编码器
├── vision/                             # 视觉编码器
├── tts/                                # TTS 模型
└── token2wav-gguf/                     # Token2Wav 模型
```

## 常用命令

```bash
# 查看服务状态
docker compose ps

# 查看日志
docker compose logs -f

# 停止所有 Docker 服务
docker compose down

# 检查推理服务健康
curl http://localhost:9060/health

# 查看推理进程
ps aux | grep minicpmo

# 停止推理服务
pkill -f "minicpmo_cpp_http_server"
```

## 文档

- [DEPLOY.md](DEPLOY.md) — 完整部署指南（含故障排查）
- [cpp_server/README.md](cpp_server/README.md) — C++ 推理服务详细文档与 API 参考
