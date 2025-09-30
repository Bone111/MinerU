# MinerU 2.5.3 使用文档

## 简介

MinerU 是一个专业的 PDF 文档转 Markdown 工具，支持文本、表格、公式、图像的智能识别和提取。2.5.3 版本提供了完整的 Docker 化部署方案和多种后端支持。

## 工作目录结构说明

MinerU 2.5.3 版本采用统一的工作目录管理，推荐的目录结构如下：

```
./mineru-workspace/                    # 主工作目录
├── output/                           # MinerU API服务输出
│   ├── document1.md                  # 解析后的Markdown文件
│   ├── document1_content_list.json   # 内容列表
│   └── images/                       # 提取的图片
├── logs/                             # MinerU API服务日志
│   ├── mineru.log                    # 应用日志
│   └── error.log                     # 错误日志
├── vlm-output/                       # VLM服务输出  
│   └── vlm_results/                  # VLM处理结果
├── vlm-logs/                         # VLM服务日志
│   ├── vlm.log                       # VLM应用日志
│   └── vlm_error.log                 # VLM错误日志
├── models/                           # AI模型文件
│   ├── layout_model/                 # 布局检测模型
│   ├── ocr_model/                    # OCR模型
│   └── vlm_model/                    # 视觉语言模型
└── pdfs/                             # 原始PDF文件（可选）
```

### 🎯 目录说明

- **output/**: MinerU API服务的解析结果存储目录
- **logs/**: MinerU API服务的运行日志目录
- **vlm-output/**: VLM服务的处理结果存储目录
- **vlm-logs/**: VLM服务的运行日志目录
- **models/**: 所有AI模型文件的统一存储目录
- **pdfs/**: 可选的PDF源文件存储目录

## 快速部署

### 方法一：使用 Docker（推荐）

#### 国际版本

```bash
git clone https://github.com/opendatalab/MinerU.git -b release-2.5.3
cd MinerU/docker/global
docker build -t mineru:2.5.3 .
```

#### 国内版本（加速镜像）

```bash
git clone https://github.com/opendatalab/MinerU.git -b release-2.5.3
cd MinerU/docker/china
docker build -t mineru:2.5.3-v100 .
```

### 方法二：源码安装

```bash
# 创建虚拟环境
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# 安装 MinerU
pip install 'mineru[core]==2.5.3'

# 下载模型
mineru-models-download -s huggingface -m all  # 国际
# 或
mineru-models-download -s modelscope -m all   # 国内
```

## 服务启动

### API 服务（推荐）

#### 基础启动

```bash
# 创建MinerU工作目录结构
mkdir -p ./mineru-workspace/output ./mineru-workspace/logs

# 启动 MinerU API 服务
docker run --rm -it --gpus all -p 8000:8000 \
  --name mineru-api \
  -v $(pwd)/mineru-workspace/output:/app/output \
  -v $(pwd)/mineru-workspace/logs:/app/logs \
  mineru:2.5.3-v100 \
  mineru-api --host 0.0.0.0 --port 8000

# 访问 API 文档
# http://localhost:8000/docs
```

#### 生产环境启动

##### 通用配置

```bash
# 创建MinerU工作目录结构
mkdir -p ./mineru-workspace/output ./mineru-workspace/logs

# 后台运行，自动重启，限制资源
docker run -d --restart=unless-stopped \
  --name mineru-api-prod \
  --gpus all \
  -p 8000:8000 \
  -v $(pwd)/mineru-workspace/output:/app/output \
  -v $(pwd)/mineru-workspace/logs:/app/logs \
  --memory=8g \
  --cpus=4 \
  -e WORKERS=4 \
  -e LOG_LEVEL=info \
  mineru:2.5.3-v100 \
  mineru-api --host 0.0.0.0 --port 8000 --workers 4
```

##### T4显卡优化配置（15G显存 + 60G内存）

```bash
# 创建MinerU工作目录结构
mkdir -p ./mineru-workspace/output ./mineru-workspace/logs

# T4显卡优化启动（推荐配置）
docker run -d --restart=unless-stopped \
  --name mineru-api-prod \
  --gpus all \
  -p 8000:8000 \
  -v $(pwd)/mineru-workspace/output:/app/output \
  -v $(pwd)/mineru-workspace/logs:/app/logs \
  --memory=24g \
  --cpus=8 \
  --shm-size=4g \
  -e WORKERS=6 \
  -e LOG_LEVEL=info \
  -e PYTORCH_CUDA_ALLOC_CONF=max_split_size_mb:512,expandable_segments:True \
  -e CUDA_VISIBLE_DEVICES=0 \
  mineru:2.5.3-v100 \
  mineru-api --host 0.0.0.0 --port 8000 --workers 6
```

##### 稳定性优先配置（保守配置）

```bash
# 创建MinerU工作目录结构
mkdir -p ./mineru-workspace/output ./mineru-workspace/logs

# 稳定性优先，适合长期运行
docker run -d --restart=unless-stopped \
  --name mineru-api-prod \
  --gpus all \
  -p 8000:8000 \
  -v $(pwd)/mineru-workspace/output:/app/output \
  -v $(pwd)/mineru-workspace/logs:/app/logs \
  --memory=16g \
  --cpus=6 \
  --shm-size=2g \
  -e WORKERS=4 \
  -e LOG_LEVEL=info \
  -e PYTORCH_CUDA_ALLOC_CONF=max_split_size_mb:256 \
  mineru:2.5.3-v100 \
  mineru-api --host 0.0.0.0 --port 8000 --workers 4
```

##### Tesla V100双GPU优化配置（高性能服务器）

```bash
# 创建MinerU工作目录结构
mkdir -p ./mineru-workspace/output ./mineru-workspace/logs ./mineru-workspace/models

# Tesla V100双GPU优化启动（适用于2×V100-32GB + 125GB内存服务器）
# 推荐在专门的工作目录中运行，例如：/home/username/工作目录
docker run -d --restart=unless-stopped \
  --name mineru-api-v100 \
  --gpus all \
  -p 8000:8000 \
  -v $(pwd)/mineru-workspace/output:/app/output \
  -v $(pwd)/mineru-workspace/logs:/app/logs \
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

# 使用GPU 1（因为GPU 0可能已有其他任务占用内存）
# 如果需要使用两个GPU进行负载均衡，可以移除CUDA_VISIBLE_DEVICES限制
```

##### Tesla V100双GPU负载均衡配置

```bash
# 创建MinerU工作目录结构
mkdir -p ./mineru-workspace/output ./mineru-workspace/logs ./mineru-workspace/models

# V100双GPU负载均衡（如果GPU 0的任务可以共享）
docker run -d --restart=unless-stopped \
  --name mineru-api-v100-dual \
  --gpus all \
  -p 8000:8000 \
  -v $(pwd)/mineru-workspace/output:/app/output \
  -v $(pwd)/mineru-workspace/logs:/app/logs \
  --memory=64g \
  --cpus=20 \
  --shm-size=16g \
  -e WORKERS=10 \
  -e LOG_LEVEL=info \
  -e PYTORCH_CUDA_ALLOC_CONF=max_split_size_mb:1024,expandable_segments:True \
  -e OMP_NUM_THREADS=20 \
  -e MKL_NUM_THREADS=20 \
  mineru:2.5.3-v100 \
  mineru-api --host 0.0.0.0 --port 8000 --workers 10
```

### VLM 服务（视觉语言模型）

#### 标准配置

```bash
# 创建VLM工作目录结构
mkdir -p ./mineru-workspace/vlm-output ./mineru-workspace/vlm-logs ./mineru-workspace/models

# 启动 vLLM 服务（OpenAI 兼容）
docker run --rm -it --gpus all -p 30000:30000 \
  --name mineru-vlm \
  --shm-size=16g \
  -v $(pwd)/mineru-workspace/models:/app/models \
  -v $(pwd)/mineru-workspace/vlm-output:/app/output \
  -v $(pwd)/mineru-workspace/vlm-logs:/app/logs \
  mineru:2.5.3-v100 \
  mineru-vllm-server --host 0.0.0.0 --port 30000
```

#### 高性能配置

```bash
# 创建VLM工作目录结构
mkdir -p ./mineru-workspace/vlm-output ./mineru-workspace/vlm-logs ./mineru-workspace/models

# 高性能配置（多 GPU、大内存）
docker run --rm -it --gpus all -p 30000:30000 \
  --name mineru-vlm-hp \
  --shm-size=32g \
  -v $(pwd)/mineru-workspace/models:/app/models \
  -v $(pwd)/mineru-workspace/vlm-output:/app/output \
  -v $(pwd)/mineru-workspace/vlm-logs:/app/logs \
  -e CUDA_VISIBLE_DEVICES=0,1 \
  mineru:2.5.3-v100 \
  mineru-vllm-server \
    --host 0.0.0.0 \
    --port 30000 \
    --gpu-memory-utilization 0.9 \
    --max-model-len 32768 \
    --tensor-parallel-size 2
```

#### 低资源配置

```bash
# 创建VLM工作目录结构
mkdir -p ./mineru-workspace/vlm-output ./mineru-workspace/vlm-logs ./mineru-workspace/models

# 低资源配置（单 GPU、节省内存）
docker run --rm -it --gpus '"device=0"' -p 30000:30000 \
  --name mineru-vlm-lite \
  --shm-size=8g \
  -v $(pwd)/mineru-workspace/models:/app/models \
  -v $(pwd)/mineru-workspace/vlm-output:/app/output \
  -v $(pwd)/mineru-workspace/vlm-logs:/app/logs \
  mineru:2.5.3-v100 \
  mineru-vllm-server \
    --host 0.0.0.0 \
    --port 30000 \
    --gpu-memory-utilization 0.6 \
    --max-model-len 8192
```

#### Tesla V100双GPU配置（推荐用于您的服务器）

```bash
# 创建VLM工作目录结构
mkdir -p ./mineru-workspace/vlm-output ./mineru-workspace/vlm-logs ./mineru-workspace/models

# V100单GPU配置（使用GPU 1，避免与现有任务冲突）
docker run -d --restart=unless-stopped \
  --name mineru-vlm-v100-single \
  --gpus '"device=1"' \
  -p 30000:30000 \
  --shm-size=16g \
  -v $(pwd)/mineru-workspace/models:/app/models \
  -v $(pwd)/mineru-workspace/vlm-output:/app/output \
  -v $(pwd)/mineru-workspace/vlm-logs:/app/logs \
  --memory=32g \
  -e CUDA_VISIBLE_DEVICES=1 \
  mineru:2.5.3-v100 \
  mineru-vllm-server \
    --host 0.0.0.0 \
    --port 30000 \
    --gpu-memory-utilization 0.85 \
    --max-model-len 32768

# V100双GPU配置（如果可以与GPU 0的任务共存）
docker run -d --restart=unless-stopped \
  --name mineru-vlm-v100-dual \
  --gpus all \
  -p 30001:30000 \
  --shm-size=32g \
  -v $(pwd)/mineru-workspace/models:/app/models \
  -v $(pwd)/mineru-workspace/vlm-output:/app/output \
  -v $(pwd)/mineru-workspace/vlm-logs:/app/logs \
  --memory=48g \
  mineru:2.5.3-v100 \
  mineru-vllm-server \
    --host 0.0.0.0 \
    --port 30000 \
    --gpu-memory-utilization 0.7 \
    --max-model-len 65536 \
    --tensor-parallel-size 2
```

### Gradio Web 界面

#### 标准启动

```bash
# 创建MinerU工作目录结构
mkdir -p ./mineru-workspace/output ./mineru-workspace/logs

# 启动 Web 界面
docker run --rm -it --gpus all -p 8000:8000 \
  --name mineru-gradio \
  -v $(pwd)/mineru-workspace/output:/app/output \
  -v $(pwd)/mineru-workspace/logs:/app/logs \
  mineru:2.5.3-v100 \
  mineru-gradio --server-name 0.0.0.0 --server-port 8000
```

#### 公网访问配置

```bash
# 创建MinerU工作目录结构
mkdir -p ./mineru-workspace/output ./mineru-workspace/logs

# 启用公网访问和认证
docker run --rm -it --gpus all -p 8000:8000 \
  --name mineru-gradio-public \
  -v $(pwd)/mineru-workspace/output:/app/output \
  -v $(pwd)/mineru-workspace/logs:/app/logs \
  -e GRADIO_SERVER_NAME=0.0.0.0 \
  -e GRADIO_AUTH=username:password \
  mineru:2.5.3-v100 \
  mineru-gradio \
    --server-name 0.0.0.0 \
    --server-port 8000 \
    --share \
    --auth username:password
```

## API 使用

### 文件解析 API

#### 基础用法

```bash
curl -X POST "http://localhost:8000/file_parse" \
  -H "Content-Type: multipart/form-data" \
  -F "files=@/path/to/your/file.pdf" \
  -F "backend=pipeline" \
  -F "parse_method=auto" \
  -F "lang_list=ch" \
  -F "return_md=true" \
  -F "response_format_zip=false"
```

#### 获取 ZIP 压缩结果

```bash
curl -X POST "http://localhost:8000/file_parse" \
  -H "Content-Type: multipart/form-data" \
  -F "files=@/path/to/your/file.pdf" \
  -F "backend=pipeline" \
  -F "return_md=true" \
  -F "return_images=true" \
  -F "response_format_zip=true" \
  --output result.zip
```

#### 批量处理多个文件

```bash
curl -X POST "http://localhost:8000/file_parse" \
  -H "Content-Type: multipart/form-data" \
  -F "files=@/path/to/file1.pdf" \
  -F "files=@/path/to/file2.pdf" \
  -F "files=@/path/to/file3.pdf" \
  -F "backend=pipeline" \
  -F "lang_list=ch" \
  -F "lang_list=en" \
  -F "lang_list=ch" \
  -F "return_md=true"
```

#### 使用 VLM 后端

```bash
curl -X POST "http://localhost:8000/file_parse" \
  -H "Content-Type: multipart/form-data" \
  -F "files=@/path/to/your/file.pdf" \
  -F "backend=vlm" \
  -F "server_url=http://localhost:30000/v1" \
  -F "lang_list=ch" \
  -F "return_md=true"
```

### VLM 服务 API

#### 文本对话

```bash
curl http://localhost:30000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-xxx" \
  -d '{
    "model": "auto",
    "messages": [{"role": "user", "content": "你好，请介绍一下你的功能"}],
    "max_tokens": 200
  }'
```

#### 图像理解

```bash
curl http://localhost:30000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-xxx" \
  -d '{
    "model": "auto",
    "messages": [{
      "role": "user",
      "content": [
        {"type": "text", "text": "请分析这张图片中的文字和表格"},
        {"type": "image_url", "image_url": {"url": "data:image/jpeg;base64,/9j/4AAQ..."}}
      ]
    }],
    "max_tokens": 500
  }'
```

#### Python SDK 调用

```python
from openai import OpenAI

# 初始化客户端
client = OpenAI(
    base_url="http://localhost:30000/v1", 
    api_key="sk-xxx"
)

# 文本对话
response = client.chat.completions.create(
    model="auto",
    messages=[{"role": "user", "content": "你好，做个自我介绍"}],
    max_tokens=200
)
print(response.choices[0].message.content)

# 图像理解
response = client.chat.completions.create(
    model="auto",
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": "描述这张图片"},
            {"type": "image_url", "image_url": {"url": "https://example.com/image.jpg"}}
        ]
    }],
    max_tokens=500
)
print(response.choices[0].message.content)
```

## 参数说明

### file_parse API 参数

| 参数                    | 类型     | 默认值     | 说明                              |
| ----------------------- | -------- | ---------- | --------------------------------- |
| `files`               | File[]   | 必需       | 上传的文件列表（PDF/图片）        |
| `backend`             | string   | "pipeline" | 后端类型："pipeline" 或 "vlm"     |
| `parse_method`        | string   | "auto"     | 解析方法："auto", "ocr", "txt"    |
| `lang_list`           | string[] | ["ch"]     | 语言列表：ch, en, ja, ko 等       |
| `formula_enable`      | bool     | true       | 是否启用公式识别                  |
| `table_enable`        | bool     | true       | 是否启用表格识别                  |
| `return_md`           | bool     | true       | 是否返回 Markdown 格式            |
| `return_images`       | bool     | false      | 是否返回提取的图片                |
| `return_middle_json`  | bool     | false      | 是否返回中间 JSON 结果            |
| `return_model_output` | bool     | false      | 是否返回模型原始输出              |
| `return_content_list` | bool     | false      | 是否返回内容列表                  |
| `response_format_zip` | bool     | false      | 是否以 ZIP 格式返回               |
| `start_page_id`       | int      | 0          | 起始页码                          |
| `end_page_id`         | int      | 99999      | 结束页码                          |
| `server_url`          | string   | null       | VLM 服务器 URL（使用 vlm 后端时） |

### VLM 服务参数

| 参数                         | 说明                                                 |
| ---------------------------- | ---------------------------------------------------- |
| `--port`                   | 服务端口（默认：30000）                              |
| `--host`                   | 绑定地址（默认：127.0.0.1，Docker 中建议用 0.0.0.0） |
| `--model`                  | 指定模型路径（默认自动下载）                         |
| `--gpu-memory-utilization` | GPU 内存利用率（默认：0.5）                          |

## 命令行工具

### 基础使用

```bash
# 解析单个文件
mineru /path/to/file.pdf

# 指定输出目录
mineru /path/to/file.pdf --output-dir ./output

# 批量处理
mineru /path/to/files/*.pdf

# 指定语言
mineru /path/to/file.pdf --lang ch

# 使用 VLM 后端
mineru /path/to/file.pdf --backend vlm --server-url http://localhost:30000/v1
```

### 高级选项

```bash
# 禁用公式识别
mineru /path/to/file.pdf --no-formula

# 禁用表格识别  
mineru /path/to/file.pdf --no-table

# 指定页面范围
mineru /path/to/file.pdf --start-page 1 --end-page 10

# 输出额外格式
mineru /path/to/file.pdf --dump-images --dump-middle-json
```

## 支持的文件格式

- **PDF**: .pdf
- **Word文档**: .doc, .docx
- **PowerPoint**: .ppt, .pptx
- **图片**: .jpg, .jpeg, .png, .bmp, .tiff, .webp

## 支持的语言

- `ch`: 中文
- `en`: 英文
- `ja`: 日语
- `ko`: 韩语
- `ar`: 阿拉伯语
- `hi`: 印地语
- `de`: 德语
- `fr`: 法语
- `ru`: 俄语

## 日志管理和监控

### 查看容器日志

#### 实时查看日志

```bash
# 查看API服务日志（实时）
docker logs -f mineru-api-prod

# 查看最近100行日志
docker logs --tail 100 mineru-api-prod

# 查看特定时间段的日志
docker logs --since "2024-01-01T00:00:00" mineru-api-prod

# 查看VLM服务日志
docker logs -f mineru-vlm
```

#### 查看挂载的日志文件

```bash
# 查看应用日志文件
tail -f ./logs/mineru.log
tail -f ./logs/error.log
tail -f ./logs/access.log

# 查看所有日志文件
ls -la ./logs/

# 搜索错误日志
grep -i error ./logs/*.log
grep -i "out of memory" ./logs/*.log
```

#### 进入容器查看

```bash
# 进入容器
docker exec -it mineru-api-prod /bin/bash

# 在容器内查看日志
cd /app/logs
ls -la
tail -f *.log

# 查看系统资源使用
top
free -h
nvidia-smi
```

### 日志清理和管理

#### 清理旧日志

```bash
# 查看日志文件大小
du -sh ./logs/*

# 删除7天前的日志
find ./logs -name "*.log" -mtime +7 -delete

# 压缩旧日志文件
find ./logs -name "*.log" -mtime +3 -exec gzip {} \;

# 清理Docker容器日志（谨慎使用）
docker logs --details mineru-api-prod 2>/dev/null | wc -l  # 查看日志行数
# 如果日志过大，可以重启容器来清理
docker restart mineru-api-prod
```

### 监控和诊断

#### 容器状态监控

```bash
# 查看容器状态
docker ps --filter name=mineru-

# 查看容器资源使用情况
docker stats mineru-api-prod
docker stats --no-stream --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}" mineru-api-prod

# 查看GPU使用情况
nvidia-smi
watch -n 1 nvidia-smi  # 每秒刷新
```

#### 容器健康检查

```bash
# 检查容器进程
docker exec mineru-api-prod ps aux

# 检查容器内存使用
docker exec mineru-api-prod free -h

# 检查容器磁盘使用
docker exec mineru-api-prod df -h

# 检查API服务是否正常
curl -s http://localhost:8000/docs
curl -s http://localhost:8000/health  # 如果有健康检查端点

# 测试API功能
curl -X POST "http://localhost:8000/file_parse" \
  -H "Content-Type: multipart/form-data" \
  -F "files=@test.pdf" \
  -F "backend=pipeline" \
  -F "parse_method=auto"
```

#### 性能分析

```bash
# 查看处理队列和任务状态
curl -s http://localhost:8000/status  # 如果API提供状态端点

# 监控处理时间
time curl -X POST "http://localhost:8000/file_parse" \
  -H "Content-Type: multipart/form-data" \
  -F "files=@test.pdf" \
  -F "backend=pipeline"

# 查看错误统计
grep -c "ERROR" ./logs/*.log
grep -c "CUDA out of memory" ./logs/*.log
```

### 日志配置优化

#### 日志轮转配置

```bash
# 创建logrotate配置文件
sudo tee /etc/logrotate.d/mineru << EOF
/path/to/your/logs/*.log {
    daily
    missingok
    rotate 30
    compress
    notifempty
    create 0644 root root
    postrotate
        docker kill -s USR1 mineru-api-prod 2>/dev/null || true
    endscript
}
EOF

# 手动执行日志轮转
sudo logrotate -f /etc/logrotate.d/mineru
```

#### Docker日志配置

```bash
# 启动时限制日志大小（添加到docker run命令）
--log-opt max-size=100m \
--log-opt max-file=5 \
```

## 故障排除

### 常见问题

#### 1. onnxruntime 安装失败

**问题**：`onnxruntime>=1.17.1 has no wheels with a matching platform tag`

**解决方案**：

- 使用 Docker 部署（推荐）
- 或升级系统 glibc 到 2.27+
- 或修改依赖版本为 onnxruntime==1.17.0

#### 2. GPU 内存不足

**现象**：`CUDA out of memory`

**解决方案**：

```bash
# 停止其他 GPU 进程
docker ps
docker stop <other_containers>

# 降低内存使用
mineru-vllm-server --gpu-memory-utilization 0.3

# 或使用环境变量
export PYTORCH_CUDA_ALLOC_CONF=max_split_size_mb:512
```

#### 3. 端口冲突

**问题**：`Address already in use`

**解决方案**：

```bash
# 查看端口占用
netstat -tlnp | grep :8000

# 更换端口
docker run ... -p 8080:8000 ...
```

#### 4. 模型下载失败

**问题**：网络超时或下载中断

**解决方案**：

```bash
# 使用国内镜像
mineru-models-download -s modelscope -m all

# 或手动下载后指定路径
mineru --model-path /path/to/models
```

### 性能优化

#### GPU 优化

```bash
# 设置合适的 batch size
export MINERU_BATCH_SIZE=4

# 启用混合精度
export MINERU_FP16=true

# 优化内存分配
export PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True
```

#### CPU 优化

```bash
# 设置线程数
export OMP_NUM_THREADS=8
export MKL_NUM_THREADS=8

# 禁用 GPU（仅使用 CPU）
export CUDA_VISIBLE_DEVICES=""
```

## 开发集成

### Python API

```python
import asyncio
from mineru.cli.common import aio_do_parse

async def parse_pdf():
    await aio_do_parse(
        output_dir="./output",
        pdf_file_names=["document"],
        pdf_bytes_list=[pdf_bytes],
        p_lang_list=["ch"],
        backend="pipeline",
        parse_method="auto",
        formula_enable=True,
        table_enable=True,
        f_dump_md=True
    )

# 运行
asyncio.run(parse_pdf())
```

### Docker Compose

```yaml
version: '3.8'
services:
  mineru-api:
    image: mineru:2.5.3-v100
    ports:
      - "8000:8000"
    command: mineru-api --host 0.0.0.0 --port 8000
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
  
  mineru-vllm:
    image: mineru:2.5.3-v100
    ports:
      - "30000:30000"
    command: mineru-vllm-server --host 0.0.0.0
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
```

## 版本信息

- **MinerU 版本**: 2.5.3
- **基础镜像**: vllm/vllm-openai:v0.10.1.1
- **Python 版本**: 3.10+
- **CUDA 支持**: 是
- **多语言支持**: 是

## 快速启动脚本

### 一键启动脚本

创建 `start-mineru.sh` 脚本文件：

```bash
#!/bin/bash

# MinerU 2.5.3 一键启动脚本

set -e

# 配置参数
DOCKER_IMAGE="mineru:2.5.3-v100"
API_PORT=8000
VLM_PORT=30000
WORKSPACE_DIR="$(pwd)/mineru-workspace"
API_OUTPUT_DIR="$WORKSPACE_DIR/output"
API_LOGS_DIR="$WORKSPACE_DIR/logs"
VLM_OUTPUT_DIR="$WORKSPACE_DIR/vlm-output"
VLM_LOGS_DIR="$WORKSPACE_DIR/vlm-logs"
MODELS_DIR="$WORKSPACE_DIR/models"

# 创建完整的工作目录结构
echo "创建MinerU工作目录结构在: $WORKSPACE_DIR"
mkdir -p "$API_OUTPUT_DIR" "$API_LOGS_DIR" "$VLM_OUTPUT_DIR" "$VLM_LOGS_DIR" "$MODELS_DIR"

# 检查 Docker 镜像是否存在
if ! docker images | grep -q "$DOCKER_IMAGE"; then
    echo "Docker 镜像 $DOCKER_IMAGE 不存在，请先构建镜像"
    exit 1
fi

# 停止已存在的容器
echo "停止已存在的容器..."
docker stop mineru-api mineru-vlm 2>/dev/null || true
docker rm mineru-api mineru-vlm 2>/dev/null || true

# 启动服务
echo "启动 MinerU 服务..."

# 启动 API 服务
echo "启动 MinerU API 服务 (端口: $API_PORT)..."
docker run -d --restart=unless-stopped \
  --name mineru-api \
  --gpus all \
  -p $API_PORT:8000 \
  -v "$API_OUTPUT_DIR:/app/output" \
  -v "$API_LOGS_DIR:/app/logs" \
  --memory=8g \
  --cpus=4 \
  "$DOCKER_IMAGE" \
  mineru-api --host 0.0.0.0 --port 8000 --workers 4

# 启动 VLM 服务
echo "启动 VLM 服务 (端口: $VLM_PORT)..."
docker run -d --restart=unless-stopped \
  --name mineru-vlm \
  --gpus all \
  -p $VLM_PORT:30000 \
  --shm-size=16g \
  -v "$MODELS_DIR:/app/models" \
  -v "$VLM_OUTPUT_DIR:/app/output" \
  -v "$VLM_LOGS_DIR:/app/logs" \
  "$DOCKER_IMAGE" \
  mineru-vllm-server --host 0.0.0.0 --port 30000

echo "服务启动完成！"
echo "API 服务: http://localhost:$API_PORT/docs"
echo "VLM 服务: http://localhost:$VLM_PORT/v1"
echo "检查服务状态: docker ps"
```

使用方法：

```bash
chmod +x start-mineru.sh
./start-mineru.sh
```

### Tesla V100服务器专用启动脚本

创建 `start-mineru-v100.sh` 脚本文件（适用于您的服务器配置）：

```bash
#!/bin/bash

# MinerU 2.5.3 Tesla V100双GPU服务器专用启动脚本

set -e

# 配置参数
DOCKER_IMAGE="mineru:2.5.3-v100"
API_PORT=8000
VLM_PORT=30000
BASE_DIR="$(pwd)"
WORKSPACE_DIR="$BASE_DIR/mineru-workspace"
API_OUTPUT_DIR="$WORKSPACE_DIR/output"
API_LOGS_DIR="$WORKSPACE_DIR/logs"
VLM_OUTPUT_DIR="$WORKSPACE_DIR/vlm-output"
VLM_LOGS_DIR="$WORKSPACE_DIR/vlm-logs"
MODELS_DIR="$WORKSPACE_DIR/models"

# 显示工作目录信息
echo "当前基础目录: $BASE_DIR"
echo "MinerU工作目录: $WORKSPACE_DIR"
echo "API输出目录: $API_OUTPUT_DIR"
echo "API日志目录: $API_LOGS_DIR"
echo "VLM输出目录: $VLM_OUTPUT_DIR"
echo "VLM日志目录: $VLM_LOGS_DIR"
echo "模型目录: $MODELS_DIR"
echo ""

# 检查GPU状态
echo "检查GPU状态..."
nvidia-smi

# 创建完整的工作目录结构
echo ""
echo "创建MinerU工作目录结构..."
mkdir -p "$API_OUTPUT_DIR" "$API_LOGS_DIR" "$VLM_OUTPUT_DIR" "$VLM_LOGS_DIR" "$MODELS_DIR"

# 检查 Docker 镜像是否存在
if ! docker images | grep -q "$DOCKER_IMAGE"; then
    echo "Docker 镜像 $DOCKER_IMAGE 不存在，请先构建镜像"
    exit 1
fi

# 停止已存在的容器
echo "停止已存在的容器..."
docker stop mineru-api-v100 mineru-vlm-v100 2>/dev/null || true
docker rm mineru-api-v100 mineru-vlm-v100 2>/dev/null || true

# 启动服务
echo "启动 MinerU V100 服务..."

# 启动 API 服务（使用GPU 1，避免与GPU 0现有任务冲突）
echo "启动 MinerU API 服务 (端口: $API_PORT, 使用GPU 1)..."
docker run -d --restart=unless-stopped \
  --name mineru-api-v100 \
  --gpus all \
  -p $API_PORT:8000 \
  -v "$API_OUTPUT_DIR:/app/output" \
  -v "$API_LOGS_DIR:/app/logs" \
  --memory=48g \
  --cpus=16 \
  --shm-size=8g \
  -e WORKERS=8 \
  -e LOG_LEVEL=info \
  -e PYTORCH_CUDA_ALLOC_CONF=max_split_size_mb:1024,expandable_segments:True \
  -e CUDA_VISIBLE_DEVICES=1 \
  -e OMP_NUM_THREADS=16 \
  -e MKL_NUM_THREADS=16 \
  "$DOCKER_IMAGE" \
  mineru-api --host 0.0.0.0 --port 8000 --workers 8

# 启动 VLM 服务（使用GPU 1）
echo "启动 VLM 服务 (端口: $VLM_PORT, 使用GPU 1)..."
docker run -d --restart=unless-stopped \
  --name mineru-vlm-v100 \
  --gpus '"device=1"' \
  -p $VLM_PORT:30000 \
  --shm-size=16g \
  -v "$MODELS_DIR:/app/models" \
  -v "$VLM_OUTPUT_DIR:/app/output" \
  -v "$VLM_LOGS_DIR:/app/logs" \
  --memory=32g \
  -e CUDA_VISIBLE_DEVICES=1 \
  "$DOCKER_IMAGE" \
  mineru-vllm-server \
    --host 0.0.0.0 \
    --port 30000 \
    --gpu-memory-utilization 0.85 \
    --max-model-len 32768

echo "V100服务器专用配置启动完成！"
echo "API 服务: http://localhost:$API_PORT/docs"
echo "VLM 服务: http://localhost:$VLM_PORT/v1"
echo ""
echo "当前GPU使用情况："
nvidia-smi
echo ""
echo "检查服务状态: docker ps --filter name=mineru-"
echo "查看API日志: docker logs -f mineru-api-v100"
echo "查看VLM日志: docker logs -f mineru-vlm-v100"
```

使用方法：

```bash
chmod +x start-mineru-v100.sh
./start-mineru-v100.sh
```

### 服务管理脚本

创建 `manage-mineru.sh` 脚本文件：

```bash
#!/bin/bash

# MinerU 服务管理脚本

case "$1" in
  start)
    echo "启动 MinerU 服务..."
    ./start-mineru.sh
    ;;
  stop)
    echo "停止 MinerU 服务..."
    docker stop mineru-api mineru-vlm mineru-gradio 2>/dev/null || true
    echo "服务已停止"
    ;;
  restart)
    echo "重启 MinerU 服务..."
    $0 stop
    sleep 2
    $0 start
    ;;
  status)
    echo "MinerU 服务状态："
    docker ps --filter name=mineru- --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
    ;;
  logs)
    if [ -z "$2" ]; then
      echo "用法: $0 logs [api|vlm|gradio]"
      exit 1
    fi
    docker logs -f "mineru-$2"
    ;;
  clean)
    echo "清理 MinerU 容器和卷..."
    docker stop $(docker ps -q --filter name=mineru-) 2>/dev/null || true
    docker rm $(docker ps -aq --filter name=mineru-) 2>/dev/null || true
    echo "清理完成"
    ;;
  *)
    echo "用法: $0 {start|stop|restart|status|logs|clean}"
    echo "  start   - 启动所有服务"
    echo "  stop    - 停止所有服务"
    echo "  restart - 重启所有服务"
    echo "  status  - 查看服务状态"
    echo "  logs    - 查看服务日志 (需指定服务名: api/vlm/gradio)"
    echo "  clean   - 清理所有容器"
    exit 1
    ;;
esac
```

使用方法：

```bash
chmod +x manage-mineru.sh
./manage-mineru.sh start    # 启动服务
./manage-mineru.sh status   # 查看状态
./manage-mineru.sh logs api # 查看 API 日志
./manage-mineru.sh stop     # 停止服务
```

### 健康检查脚本

创建 `health-check.sh` 脚本文件：

```bash
#!/bin/bash

# MinerU 健康检查脚本

API_URL="http://localhost:8000"
VLM_URL="http://localhost:30000"

echo "检查 MinerU 服务健康状态..."

# 检查 API 服务
echo -n "API 服务: "
if curl -s "$API_URL/health" >/dev/null 2>&1; then
    echo "✅ 正常"
else
    echo "❌ 异常"
fi

# 检查 VLM 服务
echo -n "VLM 服务: "
if curl -s "$VLM_URL/health" >/dev/null 2>&1; then
    echo "✅ 正常"
else
    echo "❌ 异常"
fi

# 检查容器状态
echo -e "\n容器状态："
docker ps --filter name=mineru- --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# 检查资源使用
echo -e "\n资源使用："
docker stats --no-stream --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}" $(docker ps -q --filter name=mineru-)
```

使用方法：

```bash
chmod +x health-check.sh
./health-check.sh
```

## 针对您服务器的推荐配置

根据您的服务器配置（2×Tesla V100-32GB GPU + 125GB内存），以下是最佳实践：

### 🎯 推荐的启动命令

由于您的GPU 0已被占用（22GB/32GB），建议使用GPU 1运行MinerU：

#### 1. API服务启动（推荐）

```bash
# 创建MinerU工作目录结构  
mkdir -p ./mineru-workspace/output ./mineru-workspace/logs ./mineru-workspace/models

docker run -d --restart=unless-stopped \
  --name mineru-api-v100 \
  --gpus all \
  -p 8000:8000 \
  -v $(pwd)/mineru-workspace/output:/app/output \
  -v $(pwd)/mineru-workspace/logs:/app/logs \
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

#### 2. VLM服务启动

```bash
docker run -d --restart=unless-stopped \
  --name mineru-vlm-v100 \
  --gpus '"device=1"' \
  -p 30000:30000 \
  --shm-size=16g \
  -v $(pwd)/mineru-workspace/models:/app/models \
  -v $(pwd)/mineru-workspace/vlm-output:/app/output \
  -v $(pwd)/mineru-workspace/vlm-logs:/app/logs \
  --memory=32g \
  -e CUDA_VISIBLE_DEVICES=1 \
  mineru:2.5.3-v100 \
  mineru-vllm-server \
    --host 0.0.0.0 \
    --port 30000 \
    --gpu-memory-utilization 0.85 \
    --max-model-len 32768
```

### ⚡ 配置要点

- **目录结构**：统一使用 `mineru-workspace/` 作为主工作目录
- **服务分离**：
  - MinerU API → `output/` 和 `logs/`
  - VLM服务 → `vlm-output/` 和 `vlm-logs/`
- **GPU使用**：专用GPU 1（避免与GPU 0冲突）
- **内存分配**：API服务48GB + VLM服务32GB，充分利用您的125GB大内存
- **并发处理**：8个worker进程，匹配V100的强大性能
- **内存优化**：使用大容量shared memory和expandable segments

### 📊 性能监控

启动服务后，使用以下命令监控：

```bash
# 查看GPU使用情况
nvidia-smi

# 查看容器状态
docker ps --filter name=mineru-

# 查看资源使用
docker stats mineru-api-v100 mineru-vlm-v100

# 查看工作目录结构
ls -la ./mineru-workspace/
tree ./mineru-workspace/

# 查看MinerU API日志
docker logs -f mineru-api-v100
tail -f ./mineru-workspace/logs/*.log

# 查看VLM服务日志
docker logs -f mineru-vlm-v100
tail -f ./mineru-workspace/vlm-logs/*.log

# 查看输出结果
ls -la ./mineru-workspace/output/        # MinerU解析结果
ls -la ./mineru-workspace/vlm-output/    # VLM处理结果
```

### 🔧 故障排除

如果遇到内存不足：

```bash
# 降低内存使用
docker update --memory=32g mineru-api-v100
```

如果需要使用双GPU：

```bash
# 移除CUDA_VISIBLE_DEVICES限制
-e CUDA_VISIBLE_DEVICES=0,1 \
```

## 更多资源

- **官方网站**: https://mineru.net/
- **文档**: https://opendatalab.github.io/MinerU/
- **GitHub**: https://github.com/opendatalab/MinerU
- **问题反馈**: https://github.com/opendatalab/MinerU/issues

---

*最后更新：2025年9月25日*
