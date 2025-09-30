# MinerU 2.5.3 V100 部署指南

<div align="center">
<p align="center">
  <img src="docs/images/MinerU-logo.png" width="300px" style="vertical-align:middle;">
</p>
</div>

## 📋 部署概述

本文档专为 **Tesla V100** GPU的高性能服务器设计，采用国内镜像源优化部署方案，确保在V100架构下的最佳性能表现。

### 🎯 硬件配置

- **服务器地址**: 192.168.237.11
- **GPU型号**: Tesla V100 × 2 (双GPU配置，默认使用GPU1)
- **架构**: Volta (Compute Capability 7.0)
- **显存**: 32GB × 2 = 64GB 总显存
- **系统内存**: 125GB+ 推荐
- **优化配置**: 支持双GPU负载均衡

## ⚠️ 重要说明

V100 属于 **Volta 架构**（计算能力 7.0），需要使用特定版本的 vLLM 镜像：

- **标准镜像**: `vllm/vllm-openai:v0.10.1.1` (不支持V100)
- **V100兼容镜像**: `docker.m.daocloud.io/vllm/vllm-openai:v0.10.2` ✅

## 🚀 快速部署

### 步骤一：克隆项目并准备 Dockerfile

```bash
# 克隆MinerU项目到服务器路径
git clone https://github.com/opendatalab/MinerU.git -b release-2.5.3 /mnt/data/online/MinerU-2.5.3
cd /mnt/data/online/MinerU-2.5.3

# 进入Docker配置目录
cd docker/china

# 备份原文件
cp Dockerfile Dockerfile.backup
```

### 步骤二：修改 Dockerfile（V100专用）

编辑 `Dockerfile` 文件，进行以下修改：

```dockerfile
# V100专用配置 - 使用国内DaoCloud镜像 v0.10.2版本
# 适用于Volta架构GPU (Compute Capability 7.0)
FROM docker.m.daocloud.io/vllm/vllm-openai:v0.10.2

# 注释掉不兼容的镜像版本
# FROM docker.m.daocloud.io/vllm/vllm-openai:v0.10.1.1

# Install libgl for opencv support & Noto fonts for Chinese characters
RUN apt-get update && \
    apt-get install -y \
        fonts-noto-core \
        fonts-noto-cjk \
        fontconfig \
        libgl1 && \
    fc-cache -fv && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Install mineru latest with China mirror
RUN python3 -m pip install -U 'mineru[core]' -i https://mirrors.aliyun.com/pypi/simple --break-system-packages && \
    python3 -m pip cache purge

# Download models using ModelScope (China mirror)
RUN /bin/bash -c "mineru-models-download -s modelscope -m all"

# Set the entry point to activate the virtual environment and run the command line tool
ENTRYPOINT ["/bin/bash", "-c", "export MINERU_MODEL_SOURCE=local && exec \"$@\"", "--"]
```

### 步骤三：构建 V100 专用镜像

```bash
# 在 /mnt/data/online/MinerU-2.5.3/docker/china 目录下构建镜像
docker build -t mineru:2.5.3-v100 .

# 验证镜像构建成功
docker images | grep mineru:2.5.3-v100
```

## 📁 目录结构说明

```bash
/mnt/data/online/MinerU-2.5.3/           # 主项目目录
├── docker/                              # Docker配置文件
│   └── china/                           # 国内镜像版本
│       ├── Dockerfile                   # V100专用Dockerfile  
│       └── Dockerfile.backup            # 原始备份
└── workspace/                           # 运行时工作空间
    └── output/                          # 📤 输出目录
```

## ⚡ Tesla V100 双GPU优化启动

### 准备工作目录

```bash
# 在当前目录创建工作空间(/mnt/data/online/MinerU-2.5.3/)
mkdir -p mineru-workspace/output
```

### 优化启动命令（适用于2×V100-32GB + 125GB内存服务器）

```bash
# Tesla V100双GPU优化启动命令
docker run -d --restart=unless-stopped \
  --name mineru-api-v100 \
  --gpus all \
  -p 8000:8000 \
  -v $(pwd)/mineru-workspace/output:/vllm-workspace/output \
  --memory=48g \
  --cpus=16 \
  --shm-size=8g \
  -e WORKERS=8 \
  -e LOG_LEVEL=info \
  -e PYTORCH_CUDA_ALLOC_CONF=max_split_size_mb:1024,expandable_segments:True \
  -e CUDA_VISIBLE_DEVICES=1 \
  -e OMP_NUM_THREADS=16 \
  -e MKL_NUM_THREADS=16 \
  mineru:2.5.3-v100 \
  mineru-api --host 0.0.0.0 --port 8000 --workers 8
```

> **注意**: 使用GPU 1（因为GPU 0可能已有其他任务占用内存）。如果需要使用两个GPU进行负载均衡，可以移除 `CUDA_VISIBLE_DEVICES=1` 限制。

### 参数说明

| 参数                        | 值           | 说明                     |
| --------------------------- | ------------ | ------------------------ |
| `--gpus all`              | 使用所有GPU  | 允许访问所有GPU设备      |
| `--memory=48g`            | 48GB内存限制 | 限制容器最大内存使用     |
| `--cpus=16`               | 16个CPU核心  | 限制容器CPU使用核心数    |
| `--shm-size=8g`           | 8GB共享内存  | 适配V100显存和多进程需求 |
| `WORKERS=8`               | 8个工作进程  | 并行处理请求数量         |
| `CUDA_VISIBLE_DEVICES=1`  | 使用GPU 1    | 指定使用第二个GPU        |
| `PYTORCH_CUDA_ALLOC_CONF` | 内存分配优化 | 优化CUDA内存分配策略     |
| `OMP_NUM_THREADS=16`      | OpenMP线程数 | 优化CPU并行计算性能      |
| `--workers 8`             | API工作线程  | 提高API服务并发处理能力  |

## 🔧 服务管理

### 服务状态检查

```bash
# 查看运行状态
docker ps | grep mineru-api-v100

# 查看GPU使用情况
nvidia-smi

# 查看服务日志
docker logs mineru-api-v100 -f
```

### 服务停止/重启

```bash
# 停止服务
docker stop mineru-api-v100

# 重启服务
docker restart mineru-api-v100

# 删除容器
docker rm mineru-api-v100
```

## 🗑️ 输出文件清理

### 手动清理命令

```bash
# 清理3天前的所有输出文件
find ./mineru-workspace/output -type f -mtime +3 -delete

# 清理7天前的文件（推荐）
find ./mineru-workspace/output -type f -mtime +7 -delete

# 只保留最新50个Markdown文件
cd ./mineru-workspace/output
ls -t *.md 2>/dev/null | tail -n +51 | xargs -r rm

# 只保留最新100个文件（包括所有类型）
ls -t 2>/dev/null | tail -n +101 | xargs -r rm

# 清理大文件（大于50MB的文件）
find ./mineru-workspace/output -type f -size +50M -delete

# 清理空的images目录
find ./mineru-workspace/output -type d -name "images" -empty -delete

# 一键清理：删除7天前文件并只保留最新100个
find ./mineru-workspace/output -type f -mtime +7 -delete && \
cd ./mineru-workspace/output && \
ls -t *.md 2>/dev/null | tail -n +101 | xargs -r rm
```

### 查看存储使用情况

```bash
# 查看output目录大小
du -sh ./mineru-workspace/output

# 查看文件数量
find ./mineru-workspace/output -type f | wc -l

# 查看各类型文件数量
echo "Markdown文件: $(find ./mineru-workspace/output -name "*.md" | wc -l)"
echo "JSON文件: $(find ./mineru-workspace/output -name "*.json" | wc -l)"
echo "图片文件: $(find ./mineru-workspace/output -name "*.png" -o -name "*.jpg" | wc -l)"
```

### 定期清理建议

```bash
# 创建清理脚本（推荐）
cat > ~/cleanup-mineru.sh << 'EOF'
#!/bin/bash
OUTPUT_DIR="./mineru-workspace/output"
KEEP_DAYS=3
MAX_FILES=100

echo "开始清理MinerU输出文件..."
echo "清理前: $(find "$OUTPUT_DIR" -type f | wc -l) 个文件"

# 删除3天前的文件
find "$OUTPUT_DIR" -type f -mtime +$KEEP_DAYS -delete

# 如果文件数还是太多，只保留最新100个
current_count=$(find "$OUTPUT_DIR" -name "*.md" | wc -l)
if [ "$current_count" -gt "$MAX_FILES" ]; then
    cd "$OUTPUT_DIR"
    ls -t *.md 2>/dev/null | tail -n +$((MAX_FILES+1)) | xargs -r rm
fi

# 清理空目录
find "$OUTPUT_DIR" -type d -empty -delete

echo "清理后: $(find "$OUTPUT_DIR" -type f | wc -l) 个文件"
echo "当前目录大小: $(du -sh "$OUTPUT_DIR" | cut -f1)"
EOF

# 使脚本可执行
chmod +x ~/cleanup-mineru.sh

# 运行清理
~/cleanup-mineru.sh
```

> **提示**: 建议定期运行清理脚本，避免output目录占用过多存储空间。可以根据实际需求调整保留天数和文件数量。

## 📡 API使用示例

### 基础API调用

```bash
# 健康检查
curl http://localhost:8000/health

# API文档
curl http://localhost:8000/docs

# 文件解析
curl -X POST "http://localhost:8000/parse" \
  -H "accept: application/json" \
  -H "Content-Type: multipart/form-data" \
  -F "file=@document.pdf"
```

### Python客户端示例

```python
import requests

# API服务地址
API_URL = "http://localhost:8000"

def parse_pdf(pdf_path):
    """解析PDF文件"""
    with open(pdf_path, 'rb') as f:
        files = {'file': f}
        response = requests.post(f"{API_URL}/parse", files=files)
  
    if response.status_code == 200:
        result = response.json()
        print(f"解析成功，输出文件: {result.get('output_file')}")
        return result
    else:
        print(f"解析失败: {response.status_code}")
        return None

# 使用示例
result = parse_pdf("document.pdf")
```

## ⚠️ 故障排除

### 常见问题及解决方案

#### 1. GPU不可用

```bash
# 检查GPU状态
nvidia-smi

# 检查容器内GPU可见性
docker exec mineru-api-v100 nvidia-smi

# 检查CUDA设备
docker exec mineru-api-v100 python -c "import torch; print(torch.cuda.device_count())"
```

#### 2. 路径权限问题

```bash
# 确保路径权限正确
sudo chown -R $(whoami):$(whoami) /mnt/data/online/MinerU-2.5.3/mineru-workspace/
chmod -R 755 /mnt/data/online/MinerU-2.5.3/mineru-workspace/
```

#### 3. 服务无法启动

```bash
# 查看详细错误日志
docker logs mineru-api-v100

# 检查端口占用
netstat -tlnp | grep 8000

# 重新启动服务
docker restart mineru-api-v100
```

### 支持信息

- **GPU配置**: Tesla V100 × 2 (默认使用GPU1，支持双GPU负载均衡)
- **架构**: Volta (CC 7.0)
- **部署版本**: MinerU 2.5.3 + vLLM v0.10.2
- **优化特性**: 多进程并行、内存优化、CPU多线程

### 联系方式

- **技术文档**: [MinerU官方文档](https://opendatalab.github.io/MinerU/)
- **GitHub Issues**: [提交问题](https://github.com/opendatalab/MinerU/issues)

---

**文档版本**: v2.5.3-V100-Simple | **更新日期**: 2024-09-25 | **适用硬件**: Tesla V100

> 🚀 本部署方案为V100 GPU简化配置，专注于API服务启动。如遇问题，请参考故障排除章节。
