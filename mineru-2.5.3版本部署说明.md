# MinerU 2.5.3 版本部署说明

<div align="center">
<p align="center">
  <img src="docs/images/MinerU-logo.png" width="300px" style="vertical-align:middle;">
</p>
</div>

## 文档概述

本文档详细介绍 MinerU 2.5.3 版本的安装部署流程，适用于 Linux、Windows 和 macOS 系统。MinerU 是一款将 PDF 转化为机器可读格式（如 markdown、JSON）的高效工具，支持多种部署方式和使用场景。

**版本信息**：MinerU 2.5.3（发布日期：2025/09/20）

## 项目简介

MinerU 是一款开源的文档解析工具，专注于将 PDF 文档转换为结构化的机器可读格式。项目具有以下核心特性：

### 主要功能

- 🔧 **智能内容提取**：删除页眉、页脚、脚注、页码等元素，确保语义连贯
- 📖 **阅读顺序优化**：输出符合人类阅读顺序的文本，适用于单栏、多栏及复杂排版
- 🏗️ **结构保持**：保留原文档的结构，包括标题、段落、列表等
- 🖼️ **多媒体处理**：提取图像、图片描述、表格、表格标题及脚注
- 🧮 **公式识别**：自动识别并转换文档中的公式为 LaTeX 格式
- 📊 **表格解析**：自动识别并转换文档中的表格为 HTML 格式
- 🔍 **智能检测**：自动检测扫描版 PDF 和乱码 PDF，并启用 OCR 功能
- 🌐 **多语言支持**：OCR 支持 84 种语言的检测与识别
- 📝 **多格式输出**：支持 Markdown、JSON、中间格式等多种输出格式
- 🎨 **可视化结果**：支持 layout 可视化、span 可视化等
- ⚡ **加速支持**：支持纯 CPU 环境运行，也支持 GPU(CUDA)/NPU(CANN)/MPS 加速

### 版本 2.5.3 更新内容

- **依赖优化**：依赖版本范围调整，使得 Turing 及更早架构显卡可以使用 vLLM 加速推理 MinerU2.5 模型
- **兼容性增强**：pipeline 后端对 torch 2.8.0 的兼容性修复
- **性能优化**：降低 vLLM 异步后端默认的并发数，减少服务端压力，避免高压导致的链接关闭问题

## 系统要求

### 硬件要求

| 组件                  | pipeline                                              | vlm-transformers             | vlm-vllm                     |
| --------------------- | ----------------------------------------------------- | ---------------------------- | ---------------------------- |
| **操作系统**    | Linux / Windows / macOS                               | Linux / Windows              | Linux / Windows (via WSL2)   |
| **CPU推理支持** | ✅                                                    | ❌                           | ❌                           |
| **GPU要求**     | Turing 及以后架构，6GB+ 显存``或 Apple Silicon | Turing 及以后架构，8GB+ 显存 | Turing 及以后架构，8GB+ 显存 |
| **内存要求**    | 最低 16GB+，推荐 32GB+                                | 最低 16GB+，推荐 32GB+       | 最低 16GB+，推荐 32GB+       |
| **磁盘空间**    | 20GB+，推荐使用 SSD                                   | 20GB+，推荐使用 SSD          | 20GB+，推荐使用 SSD          |

#### GPU 架构详细说明

| GPU 架构                | 计算能力 (CC) | 代表显卡型号                                | Docker 镜像版本   | 备注                      |
| ----------------------- | ------------- | ------------------------------------------- | ----------------- | ------------------------- |
| **Ampere 及以上** | ≥ 8.0        | RTX 30/40 系列、A100、H100、L40S 等         | v0.10.1.1（默认） | ✅ 直接支持               |
| **Turing**        | 7.5           | RTX 20 系列、GTX 16 系列、T4、Quadro RTX 等 | v0.10.2（需修改） | ⚠️ 需修改 Dockerfile    |
| **Volta**         | 7.0           | V100、Titan V                               | v0.10.2（需修改） | ⚠️ 需修改 Dockerfile    |
| **Pascal 及更早** | < 7.0         | P100、GTX 10 系列等                         | 不支持 vLLM       | ❌ 建议使用 pipeline 后端 |

> **💡 如何查看显卡架构**
>
> ```bash
> # 方法 1：使用 nvidia-smi
> nvidia-smi --query-gpu=name,compute_cap --format=csv
>
> # 方法 2：查看 NVIDIA 官方页面
> # https://developer.nvidia.com/cuda-gpus
> ```

### 软件要求

- **Python 版本**：3.10 - 3.13
- **操作系统**：
  - Linux：Ubuntu 18.04+, CentOS 8+, 或其他主流发行版
  - Windows：Windows 10/11（推荐使用 WSL2 以获得更好的兼容性）
  - macOS：10.15+ (支持 Intel 和 Apple Silicon)

> **⚠️ 环境支持说明**
>
> 为确保项目稳定性和可靠性，我们主要在特定软硬件环境进行优化测试。在非主线环境中，由于配置多样性和第三方依赖兼容性问题，无法保证 100% 可用性。建议先查阅 FAQ 文档，大多数问题已有对应解决方案。

## 安装部署

### 方式一：使用 pip/uv 安装（推荐）

#### 1. 更新包管理器

```bash
# 更新 pip
pip install --upgrade pip

# 安装 uv（更快的包管理器）
pip install uv
```

#### 2. 安装 MinerU

```bash
# 安装核心版本（推荐）
uv pip install -U "mineru[core]"

# 如果在中国大陆，可使用镜像源加速
uv pip install -U "mineru[core]" -i https://mirrors.aliyun.com/pypi/simple
```

> **💡 安装选项说明**
>
> - `mineru[core]`：包含除 vLLM 加速外的所有核心功能，兼容 Windows/Linux/macOS
> - `mineru[all]`：包含所有功能，包括 vLLM 加速
> - `mineru[vlm]`：仅 VLM 相关功能
> - `mineru[pipeline]`：仅 pipeline 后端
> - `mineru[api]`：API 服务功能
> - `mineru[gradio]`：Web UI 功能

### 方式二：源码安装

```bash
# 克隆代码仓库
git clone https://github.com/opendatalab/MinerU.git
cd MinerU

# 安装依赖
uv pip install -e .[core]

# 如果在中国大陆，使用镜像源
uv pip install -e .[core] -i https://mirrors.aliyun.com/pypi/simple
```

### 方式三：Docker 部署

> **⚠️ GPU 架构兼容性说明**
>
> MinerU 2.5.3 版本的 Docker 镜像需要根据您的显卡架构选择不同的 vLLM 版本：
>
> - **Ampere 架构及以上**（如 RTX 30/40 系列、A100、H100 等，CC >= 8.0）：使用默认配置
> - **Turing 及更早架构**（如 RTX 20 系列、GTX 16 系列、V100、P100 等，CC < 8.0）：需要修改 Dockerfile
>
> 您可以通过 [NVIDIA 官方页面](https://developer.nvidia.com/cuda-gpus) 查询显卡的计算能力 (Compute Capability)。

#### 1. 构建 Docker 镜像

**步骤 1：下载 Dockerfile**

```bash
# 下载 Dockerfile
wget https://gcore.jsdelivr.net/gh/opendatalab/MinerU@master/docker/china/Dockerfile
```

**步骤 2：根据显卡架构修改 Dockerfile（如果需要）**

如果您使用的是 **V100、T4、RTX 20 系列等 Turing 及更早架构**的显卡，需要修改 Dockerfile：

```bash
# 编辑 Dockerfile
nano Dockerfile

# 找到以下行：
FROM docker.m.daocloud.io/vllm/vllm-openai:v0.10.1.1

# 将其注释掉，并取消注释支持老架构的版本：
# FROM docker.m.daocloud.io/vllm/vllm-openai:v0.10.1.1
FROM docker.m.daocloud.io/vllm/vllm-openai:v0.10.2
```

**步骤 3：构建镜像**

```bash
# 构建镜像（Ampere 及以上架构）
docker build -t mineru-vllm:latest -f Dockerfile .

# 构建镜像（Turing 及更早架构，如 V100/T4）
docker build -t mineru-vllm:v100-t4 -f Dockerfile .
```

#### 2. 启动 Docker 容器

```bash
# 启动容器（支持 GPU）
docker run --gpus all \
  --shm-size 32g \
  -p 30000:30000 -p 7860:7860 -p 8000:8000 \
  --ipc=host \
  -it mineru-vllm:latest \
  /bin/bash
```

#### 3. 使用 Docker Compose（推荐）

```bash
# 下载 compose 文件
wget https://gcore.jsdelivr.net/gh/opendatalab/MinerU@master/docker/compose.yaml

# 启动 vLLM 服务
docker compose -f compose.yaml --profile vllm-server up -d

# 启动 Web API 服务
docker compose -f compose.yaml --profile api up -d

# 启动 Gradio WebUI 服务
docker compose -f compose.yaml --profile gradio up -d
```

## 模型下载

安装完成后，首次使用需要下载模型文件：

```bash
# 下载所有模型（推荐）
mineru-models-download

# 下载指定类型模型
mineru-models-download --model-type pipeline  # 仅下载 pipeline 模型
mineru-models-download --model-type vlm       # 仅下载 VLM 模型
```

模型文件约占 20GB 存储空间，请确保网络连接稳定。

## 使用方法

### 命令行使用

#### 基本用法

```bash
# 最简单的使用方式
mineru -p <input_pdf_path> -o <output_directory>

# 指定后端
mineru -p document.pdf -o ./output -b pipeline        # 使用 pipeline 后端
mineru -p document.pdf -o ./output -b vlm-transformers # 使用 VLM transformers 后端
mineru -p document.pdf -o ./output -b vlm-vllm        # 使用 VLM vLLM 后端
```

#### 高级参数

```bash
# 指定语言
mineru -p document.pdf -o ./output --lang auto        # 自动检测语言
mineru -p document.pdf -o ./output --lang zh          # 中文
mineru -p document.pdf -o ./output --lang en          # 英文

# 批量处理
mineru -p ./pdf_folder -o ./output_folder             # 处理文件夹中所有 PDF

# 自定义配置
mineru -p document.pdf -o ./output --config custom_config.json
```

### API 服务

#### 启动 API 服务

```bash
# 启动 FastAPI 服务
mineru-api

# 自定义端口和地址
mineru-api --host 0.0.0.0 --port 8000
```

#### API 使用示例

```python
import requests

# 上传文件并解析
files = {'file': open('document.pdf', 'rb')}
response = requests.post('http://localhost:8000/parse', files=files)
result = response.json()
```

### Web UI 使用

#### 启动 Gradio WebUI

```bash
# 启动 Web 界面
mineru-gradio

# 自定义端口
mineru-gradio --server-port 7860 --server-name 0.0.0.0
```

访问 `http://localhost:7860` 使用 Web 界面进行文档解析。

### vLLM 服务器模式

#### 启动 vLLM 服务器

```bash
# 启动 vLLM 服务器
mineru-vllm-server --port 30000 --model opendatalab/MinerU2.5-2509-1.2B

# 使用客户端连接
mineru -p document.pdf -o ./output -b vlm-http-client -u http://localhost:30000
```

## 配置说明

### 配置文件位置

- Linux/macOS: `~/.mineru/mineru.json`
- Windows: `%USERPROFILE%\.mineru\mineru.json`

### 基本配置示例

```json
{
  "backend": "pipeline",
  "model": {
    "layout_model": "doclayout_yolo",
    "ocr_model": "PaddleOCR",
    "formula_model": "unimernet"
  },
  "output": {
    "format": "markdown",
    "extract_images": true,
    "extract_tables": true
  },
  "performance": {
    "device": "auto",
    "batch_size": 1,
    "max_workers": 4
  }
}
```

### 高级配置选项

#### 公式识别配置

```json
{
  "formula": {
    "enabled": true,
    "delimiter": {
      "inline": ["$", "$"],
      "block": ["$$", "$$"]
    }
  }
}
```

#### OCR 配置

```json
{
  "ocr": {
    "language": "auto",
    "detection_model": "ch_ppocr_mobile_v2.0_det",
    "recognition_model": "ch_ppocr_mobile_v2.0_rec"
  }
}
```

## 性能优化

### 硬件加速配置

#### GPU 加速

```bash
# 检查 GPU 可用性
nvidia-smi

# 使用 GPU 加速
mineru -p document.pdf -o ./output --device cuda
```

#### MPS 加速（macOS Apple Silicon）

```bash
# 使用 Apple Silicon 加速
mineru -p document.pdf -o ./output --device mps
```

#### CPU 优化

```bash
# 设置 CPU 线程数
export OMP_NUM_THREADS=8
mineru -p document.pdf -o ./output --device cpu
```

### 内存优化

```bash
# 限制内存使用
export PYTORCH_MPS_HIGH_WATERMARK_RATIO=0.0  # macOS
export CUDA_VISIBLE_DEVICES=0                # Linux/Windows

# 批处理优化
mineru -p ./pdf_folder -o ./output --batch-size 2
```

## 故障排除

### 常见问题及解决方案

#### 1. 安装问题

```bash
# 依赖冲突
pip install --force-reinstall mineru[core]

# 权限问题（Linux/macOS）
pip install --user mineru[core]

# 网络问题（中国大陆）
pip install -i https://mirrors.aliyun.com/pypi/simple mineru[core]
```

#### 2. 模型下载问题

```bash
# 手动指定模型下载路径
export MINERU_MODEL_PATH=/path/to/models
mineru-models-download

# 使用代理下载
export https_proxy=http://proxy:port
mineru-models-download
```

#### 3. GPU 相关问题

**显卡架构不兼容**

```bash
# 检查显卡计算能力
nvidia-smi --query-gpu=name,compute_cap --format=csv

# 如果显示 Compute Capability < 8.0（如 V100 显示 7.0，T4 显示 7.5）
# 需要修改 Docker 镜像或使用兼容版本

# 对于 V100/T4 等老架构显卡，使用兼容镜像：
docker pull vllm/vllm-openai:v0.10.2
```

**CUDA 版本问题**

```bash
# 检查 CUDA 版本
nvcc --version
nvidia-smi

# 重新安装 PyTorch
pip uninstall torch torchvision
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu118
```

**vLLM 加速失败**

```bash
# 现象：无法使用 GPU 加速或出现 "unsupported compute capability" 错误
# 解决方案：
# 1. 确认显卡架构是否支持
nvidia-smi --query-gpu=compute_cap --format=csv,noheader,nounits

# 2. 如果 CC < 8.0，切换到兼容版本
# 修改配置使用 CPU 后端或更换 Docker 镜像

# 3. 检查 vLLM 版本兼容性
python -c "import vllm; print(vllm.__version__)"
```

#### 4. 内存不足

```bash
# 减少批处理大小
mineru -p document.pdf -o ./output --batch-size 1

# 使用 CPU 模式
mineru -p document.pdf -o ./output --device cpu
```

### 日志调试

```bash
# 启用详细日志
export MINERU_LOG_LEVEL=DEBUG
mineru -p document.pdf -o ./output --verbose

# 查看日志文件
tail -f ~/.mineru/logs/mineru.log
```

## 输出格式说明

### Markdown 格式

- 保留文档结构和格式
- 公式转换为 LaTeX 语法
- 表格转换为 Markdown 表格
- 图片以链接形式展示

### JSON 格式

```json
{
  "content_list": [
    {
      "type": "text",
      "content": "文本内容",
      "bbox": [x1, y1, x2, y2],
      "page": 1
    },
    {
      "type": "formula",
      "content": "LaTeX 公式",
      "bbox": [x1, y1, x2, y2],
      "page": 1
    }
  ]
}
```

### 中间格式（middle.json）

包含详细的解析中间结果，适用于二次开发。

## 版本升级

### 从旧版本升级

```bash
# 升级到最新版本
pip install --upgrade mineru[core]

# 清理旧模型文件（如需要）
rm -rf ~/.mineru/models
mineru-models-download
```

### 版本兼容性

- 2.5.x 版本向前兼容
- 配置文件格式可能需要更新
- 模型文件建议重新下载

## 社区支持

### 官方资源

- **官网**：https://mineru.net/
- **文档**：https://opendatalab.github.io/MinerU/
- **GitHub**：https://github.com/opendatalab/MinerU
- **在线体验**：
  - [官网在线版](https://mineru.net/OpenSourceTools/Extractor)
  - [HuggingFace](https://huggingface.co/spaces/opendatalab/MinerU)
  - [ModelScope](https://www.modelscope.cn/studios/OpenDataLab/MinerU)

### 获取帮助

- **GitHub Issues**：https://github.com/opendatalab/MinerU/issues
- **Discord 社区**：https://discord.gg/Tdedn9GTXq
- **微信群**：通过官网获取入群方式
- **FAQ 文档**：查看常见问题解答

### 贡献指南

欢迎提交 Issue、Pull Request 或参与社区讨论。详细贡献指南请参考项目 README。

## 许可证信息

MinerU 采用 AGPL-3.0 许可证。项目中部分模型基于 YOLO 训练，遵循 AGPL 协议，可能对某些商业使用场景构成限制。未来版本将探索更宽松的许可条款。

## 致谢

MinerU 项目得到了众多开源项目的支持，特别感谢：

- PDF-Extract-Kit
- DocLayout-YOLO
- UniMERNet
- PaddleOCR
- RapidTable
- 以及所有贡献者

---

**文档版本**：v2.5.3
**更新日期**：2025-09-25
**维护者**：OpenDataLab Team

> 本文档持续更新中，如有疑问或建议，欢迎通过官方渠道反馈。
