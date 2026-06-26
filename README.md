# kllama

llama.cpp 外挂式管理器。Go 编写，支持模型下载、多实例管理、OpenAI 兼容 API 代理、Web 聊天/向量 UI、系统服务化部署。

**Repository:** [github.com/xiaoyaoking/kllama](https://github.com/xiaoyaoking/kllama)

## 为什么造轮子 
之前用了ollama 对比了编译版本的 llama.cpp 感觉性能和资源占用都有明显差异,所以想造个轮子,跑编译版的llama.cpp
为了管理方便才做了一个外挂式的管理器

## 功能

- 未编译时自动安装 git/cmake 等工具，并 clone + 编译 llama.cpp
- 模型下载：URL 直链（含 ModelScope API 链接）、自有 Registry、HuggingFace（`pull hf`）、ModelScope（`pull ms`）
- 断点续传，显示进度/速度/已用时间/剩余时间
- 扁平化命令：`kllama list/create/start/stop/restart/rm`
- 实例按需启动、闲时回收、常驻模式；支持 `start/stop/restart all` 批量操作
- `serve` 与 CLI 实例状态自动同步（`instances.json` 变更后代理可感知）
- Web UI：对话 / Embedding、实例状态提示（就绪 / 加载中 / 冷启动）
- 服务化部署（install / start / status）

## 快速开始

从 [GitHub Releases](https://github.com/xiaoyaoking/kllama/releases) 下载预编译程序，拉取 Qwen2.5-3B-Instruct 并创建实例。

### Linux / macOS

```bash
# 1. 下载 kllama（按平台选择对应文件，见 Releases 页）
curl -L -o kllama https://github.com/xiaoyaoking/kllama/releases/latest/download/kllama-linux-amd64
chmod +x kllama

# 2. 拉取 Qwen2.5-3B-Instruct（魔搭 Q4_0，约 2GB）
./kllama pull ms qwen/Qwen2.5-3B-Instruct-GGUF qwen2.5-3b-instruct-q4_0.gguf

# 3. 创建实例（有 GPU 时建议全卸载；-1 表示全部层上 GPU，纯 CPU 可去掉该参数）
./kllama create qwen25-3b qwen2.5-3b-instruct-q4_0.gguf --n-gpu-layers -1

# 4. 启动实例（首次会编译 llama.cpp，需 git/cmake，约数分钟）
./kllama start qwen25-3b

# 5. 启动 API 代理 + Web UI
./kllama serve
```

浏览器打开 [http://127.0.0.1:8080](http://127.0.0.1:8080)，选择 `qwen25-3b` 即可对话。

macOS 请将下载链接中的 `kllama-linux-amd64` 换为 `kllama-darwin-arm64` 或 `kllama-darwin-amd64`。

### Windows（PowerShell）

```powershell
# 1. 下载
Invoke-WebRequest -Uri "https://github.com/xiaoyaoking/kllama/releases/latest/download/kllama-windows-amd64.exe" -OutFile kllama.exe

# 2. 拉取模型
.\kllama.exe pull ms qwen/Qwen2.5-3B-Instruct-GGUF qwen2.5-3b-instruct-q4_0.gguf

# 3. 创建实例
.\kllama.exe create qwen25-3b qwen2.5-3b-instruct-q4_0.gguf --n-gpu-layers -1

# 4. 启动实例（首次会编译 llama.cpp）
.\kllama.exe start qwen25-3b

# 5. 启动服务（保持此窗口运行）
.\kllama.exe serve
```

> **说明：** kllama 是 llama.cpp 的管理外壳，核心推理由 `llama-server` 完成。`pull` / `create` 不依赖 llama.cpp；**首次 `start` 或 `serve` 会自动编译**（也可手动 `kllama setup`）。`create --start` 同样会在启动前触发编译。

## 编译

```bash
go build -o kllama ./cmd/kllama
```

## 命令

| 命令 | 说明 |
|------|------|
| `kllama pull <url>` | 从 URL 下载 GGUF（支持 `?FilePath=xxx.gguf` 等 query 参数） |
| `kllama pull hf <repo> [file]` | HuggingFace |
| `kllama pull ms <owner/name> [file]` | [ModelScope 魔搭](https://modelscope.cn/) |
| `kllama models` | 列出本地模型文件 |
| `kllama list` / `kllama ps` | 实例列表 / 运行中实例 |
| `kllama create <name> <model.gguf>` | 创建实例 |
| `kllama create ... --always-run` | 常驻实例（闲时不回收，serve 启动时自动拉起） |
| `kllama start/stop/restart <name\|all>` | 启停 / 重启单个或全部实例 |
| `kllama rm <name>` | 删除实例 |
| `kllama llama-flags` | 常用 llama-server 参数说明 |
| `kllama setup` | 编译安装 llama.cpp（`start`/`serve` 也会自动触发） |
| `kllama serve` | API 代理 + Web UI（默认开启） |
| `kllama serve --no-web-ui` | 仅 API |
| `kllama serve --install --start` | 系统服务 |

## 模型类型与 llama-server 参数

kllama 会根据 **GGUF 文件名** 自动识别模型类型，并注入对应的 `llama-server` 启动参数：

| 类型 | 识别规则（文件名含） | llama-server 自动参数 | API |
|------|----------------------|------------------------|-----|
| **对话 Chat** | 默认（Qwen、Llama 等） | 无额外参数 | `POST /v1/chat/completions` |
| **向量 Embedding** | `bge`、`embed`、`e5`、`gte`、`nomic` 等 | **`--embeddings`** | `POST /v1/embeddings` |

> 新版 llama.cpp 要求 Embedding 模型启动时必须加 `--embeddings`，否则返回  
> `This server does not support embeddings. Start it with --embeddings`  
> **kllama 会自动添加**，一般无需手动 `--arg`。

### 对话模型示例

```bash
kllama pull ms Qwen/Qwen2.5-0.5B-Instruct
kllama create qwen Qwen2.5-0.5B-Instruct-Q4_K_M.gguf --start --n-gpu-layers 99
kllama serve
# Web UI「对话」模式，或 curl /v1/chat/completions
```

### Embedding 模型示例（bge-m3）

```bash
kllama pull ms gpustack/bge-m3-GGUF bge-m3-FP16.gguf
kllama create bge-m3 bge-m3-FP16.gguf --always-run
# 内部等价于: llama-server --model ... --embeddings --host 127.0.0.1 --port ...

kllama stop bge-m3 && kllama start bge-m3   # 若旧实例未带 --embeddings，需重启

curl http://127.0.0.1:8080/v1/embeddings \
  -H "Authorization: Bearer sk-test" \
  -H "Content-Type: application/json" \
  -d '{"model":"bge-m3","input":"hello"}'
```

Web UI：选择 bge-m3 →「向量」模式；支持深/浅色主题切换；向量结果可一键复制完整数组。

### 手动覆盖参数

仍可通过 `--arg` 传入任意 llama-server 参数（完整兼容）：

```bash
kllama create bge-m3 bge-m3-FP16.gguf --arg --embeddings --arg --pooling --arg mean
```

## 创建实例常用参数

```bash
kllama create <name> <model.gguf> [选项...]
```

| 参数 | 说明 |
|------|------|
| `--start` | 创建后立即在后台启动 |
| `--always-run` | 常驻模式：闲时不回收；`serve` 启动时自动拉起；崩溃后重启 |
| `--n-gpu-layers <N>` | GPU 卸载层数，`-1` 表示全部 |
| `--ctx-size <N>` | 上下文长度（token），默认 4096 |
| `--threads <N>` | CPU 推理线程数 |
| `--threads-batch <N>` | 批处理线程数 |
| `--batch-size <N>` | 逻辑 batch 大小 |
| `--ubatch-size <N>` | 物理 batch 大小 |
| `--parallel <N>` | 并行 slot 数（多路并发） |
| `--main-gpu <N>` | 主 GPU 序号 |
| `--tensor-split <S>` | 多卡分配比例，如 `0.5,0.5` |
| `--flash-attn <mode>` | Flash Attention（`on` / `off` / `auto`） |
| `--cont-batching` | 启用 continuous batching |
| `--no-cont-batching` | 关闭 continuous batching |
| `--mlock` | 将模型锁定在内存，禁止换页 |
| `--no-mmap` | 禁用 mmap 加载模型 |
| `--arg <flag>` | 透传任意 llama-server 参数（可重复） |

**启动时机：** 满足以下任一条件会在创建后自动启动：`--start`、`--always-run`、或配置中 `instance.auto_start: true`。

**参数优先级（后者覆盖前者）：** 配置文件 `default_llama_args` → 创建时的 `--xxx` / `--arg` → 模型类型自动注入（如 Embedding 的 `--embeddings`）。

### 示例

```bash
# GPU 全卸载 + 8K 上下文，创建并启动
kllama create qwen model.gguf --start --n-gpu-layers -1 --ctx-size 8192

# 多卡 + Flash Attention
kllama create llama70 model.gguf --tensor-split 0.6,0.4 --flash-attn on --parallel 4

# 透传 llama-server 原生参数（与简写等价）
kllama create my model.gguf --arg --speculative --arg --model-draft --arg draft.gguf

# 查看常用 llama-server 参数列表
kllama llama-flags
```

## 配置 `~/.kllama/kllama.yaml`

```yaml
server:
  addr: "0.0.0.0:8080"          # 监听地址
  api_keys: ["sk-change-me"]
  idle_timeout_min: 10          # 空闲实例回收时间（分钟）
  recycle_interval_sec: 60      # 回收检查间隔（秒）
  web_ui: true                  # false 关闭 Web UI

paths:
  model_dir: ""                 # 留空则 ~/.kllama/models
  registry_url: ""              # 自有模型 Registry

instance:
  port_start: 18080             # 实例端口范围起点
  port_end: 19080
  log_dir: ""                   # 留空则 ~/.kllama/logs
  auto_start: false             # true 时 create 后默认启动（等同 --start）
  default_llama_args: []        # 见下文
```

### `default_llama_args`

全局附加到**每个** `llama-server` 进程的参数，适合机器级默认，避免每次 `create` 重复填写。

- **格式：** YAML 字符串数组，每项对应一个 argv 片段（与命令行一致）
- **生效时机：** 实例 `start` 时，在 `--model` / `--port` / `--host` 之后、实例自身参数之前追加
- **典型用途：** 统一 GPU 层数、上下文、线程数、Flash Attention 等

```yaml
instance:
  default_llama_args:
    - "--n-gpu-layers"
    - "99"
    - "--ctx-size"
    - "8192"
    - "--threads"
    - "8"
    - "--flash-attn"
    - "auto"
```

等价于每次启动时多传：`llama-server ... --n-gpu-layers 99 --ctx-size 8192 --threads 8 --flash-attn auto`

单个实例可用 `kllama create` 的 `--n-gpu-layers`、`--ctx-size` 或 `--arg` 覆盖全局默认值。Embedding 模型所需的 `--embeddings` 仍由 kllama 按文件名自动追加，一般无需写入 `default_llama_args`。

## 实例运行模式

| 模式 | 创建方式 | 闲时回收 | serve 启动时 |
|------|----------|----------|--------------|
| 默认 idle | `kllama create name model.gguf` | ✅ | ❌ 等 API 请求 |
| 常驻 always | `--always-run` | ❌ | ✅ 自动启动 |

### 实例状态（API / Web UI）

`GET /v1/models` 返回每个实例的运行状态，Web UI 发送消息时会据此显示不同提示：

| `state` | 含义 | Web UI 提示 |
|---------|------|-------------|
| `ready` | health 就绪，可直接推理 | 「模型推理中…」 |
| `starting` | 进程已起，模型仍在加载 | 「模型加载中，请稍候…」 |
| `stopped` | 未启动，需冷启动 | 「正在启动模型（冷启动约 1-2 分钟）…」 |

同时返回 `running`（等同 `state == ready`）、`starting` 布尔字段。CLI 执行 `restart all` 后，运行中的 `serve` 会自动重载 `instances.json` 并 adopt 新进程。

## 环境变量

| 变量 | 说明 |
|------|------|
| `KLLAMA_HOME` | 数据目录（默认 `~/.kllama`） |
| `KLLAMA_API_KEY` | API 鉴权 |
| `KLLAMA_REGISTRY_URL` | 自有模型 Registry |
| `MODELSCOPE_API_TOKEN` | ModelScope gated 模型 |

## 服务化

```bash
kllama serve --install --api-key your-key
kllama serve --start
kllama serve --status
```

## License

Copyright © 2026 [xiaoyaoking](https://github.com/xiaoyaoking). All rights reserved.

Source code: [https://github.com/xiaoyaoking/kllama](https://github.com/xiaoyaoking/kllama)
