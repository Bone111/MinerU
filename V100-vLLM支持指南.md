# MinerU 2.5.3 V100 vLLM 支持指南

## 📋 概述

本文档详细说明了Tesla V100 GPU对MinerU 2.5.3版本中vLLM支持的情况，包括兼容性要求、部署方案、优化配置和最佳实践。

### 🎯 关键信息

- **GPU型号**: Tesla V100-SXM2-32GB
- **架构**: Volta (Compute Capability 7.0)
- **vLLM支持**: ✅ 完全支持，需要特定版本
- **推荐镜像**: `docker.m.daocloud.io/vllm/vllm-openai:v0.10.2`

## ✅ **V100 vLLM支持状态**

### **兼容性矩阵**

| 镜像版本 | V100支持 | 架构要求 | 说明 |
|----------|----------|----------|------|
| `vllm/vllm-openai:v0.10.1.1` | ❌ **不支持** | CC≥8.0 (Ampere+) | 仅支持RTX 30/40系列、A100等新架构 |
| `vllm/vllm-openai:v0.10.2` | ✅ **支持** | CC≥7.0 (Volta+) | **V100推荐版本** |
| `docker.m.daocloud.io/vllm/vllm-openai:v0.10.2` | ✅ **支持** | CC≥7.0 (Volta+) | **国内镜像，推荐** |

### **架构分级说明**

| GPU架构 | Compute Capability | 代表GPU | vLLM版本要求 | 状态 |
|---------|-------------------|---------|-------------|------|
| **Ampere** | 8.0, 8.6, 8.7 | RTX 30/40系列, A100, H100 | v0.10.1.1+ | ✅ 完全支持 |
| **Turing** | 7.5 | RTX 20系列, GTX 16系列, T4 | v0.10.2 | ✅ 需修改配置 |
| **Volta** | 7.0 | **V100**, Titan V | v0.10.2 | ✅ **需修改配置** |
| **Pascal及以下** | <7.0 | P100, GTX 10系列 | 不支持 | ❌ 建议pipeline后端 |

## 🚀 部署方案

### **方案一：构建V100专用镜像（推荐）**

#### 1. 准备Dockerfile

```dockerfile
# V100专用配置 - 使用vLLM v0.10.2版本
# 适用于Volta架构GPU (Compute Capability 7.0)
FROM docker.m.daocloud.io/vllm/vllm-openai:v0.10.2

# 注释掉不兼容的镜像版本（仅支持Ampere架构）
# FROM docker.m.daocloud.io/vllm/vllm-openai:v0.10.1.1

# 安装系统依赖和中文字体支持
RUN apt-get update && \
    apt-get install -y \
        fonts-noto-core \
        fonts-noto-cjk \
        fontconfig \
        libgl1 \
        curl \
        htop \
        nvtop \
        vim && \
    fc-cache -fv && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# 安装MinerU（使用国内镜像源加速）
RUN python3 -m pip install -U 'mineru[core]' \
    -i https://mirrors.aliyun.com/pypi/simple \
    --break-system-packages && \
    python3 -m pip cache purge

# 下载模型文件（使用ModelScope国内镜像）
RUN /bin/bash -c "mineru-models-download -s modelscope -m all"

# 设置V100优化的环境变量
ENV CUDA_VISIBLE_DEVICES=0,1
ENV OMP_NUM_THREADS=16  
ENV PYTORCH_CUDA_ALLOC_CONF=max_split_size_mb:512,expandable_segments:True
ENV VLLM_WORKER_MULTIPROC_METHOD=spawn

# 创建工作目录
WORKDIR /app
RUN mkdir -p /app/output /app/logs /app/models /app/input

# 设置入口点
ENTRYPOINT ["/bin/bash", "-c", "export MINERU_MODEL_SOURCE=local && exec \"$@\"", "--"]
```

#### 2. 构建镜像

```bash
# 在MinerU项目目录下
cd /path/to/MinerU/docker/china

# 替换Dockerfile内容
cp Dockerfile Dockerfile.backup
# 将上述Dockerfile内容写入Dockerfile文件

# 构建V100专用镜像
docker build -t mineru-v100:2.5.3 .

# 验证镜像构建成功
docker images | grep mineru-v100:2.5.3
```

### **方案二：直接使用预构建镜像**

如果已有V100兼容的预构建镜像：

```bash
# 拉取V100兼容镜像
docker pull docker.m.daocloud.io/vllm/vllm-openai:v0.10.2

# 或使用官方镜像
docker pull vllm/vllm-openai:v0.10.2
```

## ⚡ V100 vLLM服务启动

### **单GPU配置（推荐）**

```bash
# 创建工作目录
mkdir -p ~/mineru-v100-workspace/{output,logs,models}
cd ~/mineru-v100-workspace

# 启动V100 vLLM服务器
docker run -d --restart=unless-stopped \
  --name mineru-vllm-v100 \
  --gpus '"device=1"' \
  -p 30000:30000 \
  -v $(pwd)/output:/app/output \
  -v $(pwd)/logs:/app/logs \
  -v $(pwd)/models:/app/models \
  --memory=24g \
  --cpus=8 \
  --shm-size=8g \
  -e CUDA_VISIBLE_DEVICES=1 \
  -e VLLM_WORKER_MULTIPROC_METHOD=spawn \
  -e PYTORCH_CUDA_ALLOC_CONF=max_split_size_mb:512,expandable_segments:True \
  -e LOG_LEVEL=info \
  mineru-v100:2.5.3 \
  mineru-vllm-server \
    --host 0.0.0.0 \
    --port 30000 \
    --gpu-memory-utilization 0.7 \
    --max-model-len 8192 \
    --trust-remote-code
```

### **双GPU配置（高性能）**

```bash
# Tesla V100双GPU配置
docker run -d --restart=unless-stopped \
  --name mineru-vllm-v100-dual \
  --gpus all \
  -p 30000:30000 \
  -v $(pwd)/output:/app/output \
  -v $(pwd)/logs:/app/logs \
  -v $(pwd)/models:/app/models \
  --memory=48g \
  --cpus=16 \
  --shm-size=16g \
  -e CUDA_VISIBLE_DEVICES=0,1 \
  -e VLLM_WORKER_MULTIPROC_METHOD=spawn \
  -e PYTORCH_CUDA_ALLOC_CONF=max_split_size_mb:1024,expandable_segments:True \
  mineru-v100:2.5.3 \
  mineru-vllm-server \
    --host 0.0.0.0 \
    --port 30000 \
    --gpu-memory-utilization 0.8 \
    --tensor-parallel-size 2 \
    --max-model-len 16384 \
    --trust-remote-code
```

### **API服务配置（连接vLLM）**

```bash
# 启动API服务（使用vLLM后端）
docker run -d --restart=unless-stopped \
  --name mineru-api-v100 \
  -p 8000:8000 \
  -v $(pwd)/output:/app/output \
  -v $(pwd)/logs:/app/logs \
  --memory=8g \
  --cpus=4 \
  -e WORKERS=4 \
  -e LOG_LEVEL=info \
  mineru-v100:2.5.3 \
  mineru-api --host 0.0.0.0 --port 8000 --workers 4
```

## ⚙️ V100优化配置

### **关键参数说明**

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `--gpu-memory-utilization` | 0.6-0.8 | GPU显存使用率，V100建议0.7 |
| `--tensor-parallel-size` | 1或2 | 双V100时可设为2 |
| `--max-model-len` | 8192-16384 | 最大序列长度 |
| `--trust-remote-code` | 必需 | 信任远程代码执行 |
| `VLLM_WORKER_MULTIPROC_METHOD` | spawn | V100多进程方法 |
| `PYTORCH_CUDA_ALLOC_CONF` | max_split_size_mb:512 | CUDA内存分配优化 |

### **性能调优建议**

```bash
# 单V100优化配置
--gpu-memory-utilization 0.7 \
--max-model-len 8192 \
--swap-space 4 \
--disable-log-requests \
--trust-remote-code

# 双V100优化配置  
--gpu-memory-utilization 0.8 \
--tensor-parallel-size 2 \
--max-model-len 16384 \
--pipeline-parallel-size 1 \
--trust-remote-code
```

## 📊 性能对比

### **不同后端性能表现**

| 后端类型 | V100单卡性能 | V100双卡性能 | 特点 |
|----------|--------------|--------------|------|
| **Pipeline** | 基准性能 | - | 稳定兼容，CPU+GPU混合 |
| **VLM-transformers** | 1.5-2x基准 | - | 纯GPU推理，内存占用大 |
| **VLM-vLLM** | **3-5x基准** | **5-8x基准** | **高性能推理加速** ✨ |

### **预期处理时间（33页文档）**

| 模式 | 单V100 | 双V100 | CPU模式对比 |
|------|--------|--------|-------------|
| **布局分析** | ~30秒 | ~20秒 | ~180秒 |
| **OCR识别** | ~60秒 | ~40秒 | ~300秒 |
| **总处理时间** | **~2-3分钟** | **~1.5-2分钟** | ~10-15分钟 |

## 🔧 服务管理

### **健康检查**

```bash
# 检查vLLM服务状态
curl http://localhost:30000/health

# 检查可用模型
curl http://localhost:30000/v1/models

# 检查GPU使用情况
nvidia-smi

# 查看容器日志
docker logs -f mineru-vllm-v100
```

### **服务控制**

```bash
# 启动服务
docker start mineru-vllm-v100

# 停止服务
docker stop mineru-vllm-v100

# 重启服务
docker restart mineru-vllm-v100

# 查看服务状态
docker ps | grep mineru-vllm-v100
```

### **资源监控**

```bash
# 实时GPU监控
watch -n 1 nvidia-smi

# 容器资源使用
docker stats mineru-vllm-v100

# 详细GPU信息
nvidia-smi --query-gpu=name,compute_cap,memory.total,memory.used,utilization.gpu,temperature.gpu --format=csv
```

## 🧪 使用示例

### **基础API调用**

```bash
# 使用vLLM后端解析文档
curl -X POST "http://localhost:8000/file_parse" \
  -H "Content-Type: multipart/form-data" \
  -F "files=@document.pdf" \
  -F "backend=vlm" \
  -F "server_url=http://localhost:30000/v1" \
  -F "lang_list=ch" \
  -F "return_md=true" \
  --max-time 300
```

### **Python客户端**

```python
import requests

def parse_with_vllm(pdf_path, server_url="http://localhost:8000"):
    """使用V100 vLLM后端解析PDF"""
    
    with open(pdf_path, 'rb') as f:
        files = {'files': f}
        data = {
            'backend': 'vlm',
            'server_url': 'http://localhost:30000/v1',
            'lang_list': 'ch',
            'return_md': 'true',
            'formula_enable': 'true',
            'table_enable': 'true'
        }
        
        response = requests.post(
            f"{server_url}/file_parse", 
            files=files, 
            data=data,
            timeout=300
        )
    
    if response.status_code == 200:
        result = response.json()
        print("✅ 解析成功")
        return result
    else:
        print(f"❌ 解析失败: {response.status_code}")
        print(response.text)
        return None

# 使用示例
result = parse_with_vllm("test.pdf")
```

## 🚨 故障排除

### **常见问题及解决方案**

#### 1. **CUDA架构不兼容错误**

```
error: no kernel image is available for execution on the device
```

**解决方案**：
```bash
# 确认使用v0.10.2镜像
docker inspect mineru-vllm-v100 | grep Image

# 重建容器使用正确镜像
docker stop mineru-vllm-v100
docker rm mineru-vllm-v100
# 使用v0.10.2镜像重新启动
```

#### 2. **vLLM服务启动失败**

```bash
# 检查日志
docker logs mineru-vllm-v100

# 常见原因及解决方案
# - GPU内存不足：降低gpu-memory-utilization
# - 模型加载失败：检查模型文件完整性
# - 端口冲突：更换端口
```

#### 3. **性能不如预期**

```bash
# 检查GPU使用率
nvidia-smi

# 优化建议
# 1. 增加gpu-memory-utilization到0.8
# 2. 使用双GPU tensor-parallel-size=2
# 3. 调整max-model-len适应文档长度
```

#### 4. **内存溢出（OOM）**

```bash
# 解决方案
# 1. 降低gpu-memory-utilization
# 2. 减少max-model-len
# 3. 启用swap-space
# 4. 重启容器清理显存

docker restart mineru-vllm-v100
```

### **性能优化检查清单**

- [ ] 确认使用v0.10.2镜像版本
- [ ] GPU内存利用率设置在0.6-0.8之间
- [ ] 双GPU时启用tensor-parallel-size=2
- [ ] 设置合适的max-model-len
- [ ] 启用CUDA内存优化环境变量
- [ ] 监控GPU温度避免过热降频

## 📋 最佳实践

### **生产环境部署建议**

1. **资源配置**
   - 单V100：24GB内存，8CPU核心
   - 双V100：48GB内存，16CPU核心
   - SSD存储用于模型和输出文件

2. **监控告警**
   ```bash
   # 设置GPU温度监控
   watch -n 30 "nvidia-smi --query-gpu=temperature.gpu --format=csv,noheader,nounits"
   
   # 设置内存使用监控
   docker stats --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}"
   ```

3. **定期维护**
   ```bash
   # 每日重启服务（可选）
   docker restart mineru-vllm-v100
   
   # 清理输出文件
   find ~/mineru-v100-workspace/output -mtime +7 -delete
   
   # 更新镜像
   docker pull docker.m.daocloud.io/vllm/vllm-openai:v0.10.2
   ```

## 🎯 结论

Tesla V100完全支持MinerU 2.5.3的vLLM加速功能，关键在于：

1. ✅ **使用正确的镜像版本**: `v0.10.2`而非`v0.10.1.1`
2. ✅ **优化配置参数**: 特别是GPU内存利用率和并行设置
3. ✅ **双GPU配置**: 可获得5-8倍性能提升
4. ✅ **持续监控**: 确保服务稳定高效运行

相比CPU模式，V100 vLLM可以提供：
- **5-10倍处理速度提升**
- **更好的资源利用率**
- **支持更大文档和批量处理**

**强烈建议从CPU模式切换到V100 vLLM部署！**

---

**文档版本**: v1.0  
**更新日期**: 2024年9月24日  
**适用版本**: MinerU 2.5.3  
**硬件要求**: Tesla V100 (Volta架构, CC 7.0)

