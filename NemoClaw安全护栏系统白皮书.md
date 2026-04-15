# NemoClaw 安全护栏系统 — 项目白皮书

## 1. 项目概述

NemoClaw 安全护栏系统是一套部署在 NVIDIA DGX Spark 上的 AI 内容安全预处理链，实现了"先审后答"的安全架构。系统在用户消息到达主 LLM 之前，通过 XGuard 安全模型进行实时风险检测，对高危内容进行拦截，安全内容则转发给 Gemma-4 大语言模型生成回复，并通过飞书机器人与用户交互。

### 核心架构

```
用户（飞书）
    │
    ▼
┌─────────────────┐
│  飞书 Bridge     │  端口 9000（对外 9079）
│  feishu_bridge.py│  接收飞书消息，异步处理
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Safety Gateway  │  端口 8888（对外 8079）
│  safety_gateway.py│  OpenAI 兼容 API
└────┬───────┬────┘
     │       │
     ▼       │
┌─────────┐  │  Step 1: 安全审查（~1s）
│ XGuard  │  │  Qwen3-8B 微调模型
│ 8B      │  │  29 类风险分类
│ :8086   │  │  9 类高危拦截
└─────────┘  │
             │  如果安全 ↓
             ▼
      ┌─────────────┐  Step 2: 生成回复（~3-12s）
      │ Gemma-4     │  26B-A4B Q4_K_M 量化
      │ llama.cpp   │  llama-server 推理
      │ :8087       │  67 tokens/s
      └─────────────┘
```

## 2. 硬件环境

| 项目 | 规格 |
|------|------|
| 设备 | NVIDIA DGX Spark |
| GPU | NVIDIA GB10 (Blackwell 架构) |
| 统一内存 | ~119 GB |
| CPU | ARM aarch64 (Grace) |
| 操作系统 | Ubuntu (Linux) |
| 网络 | 公网 IP 106.13.186.155，SSH 端口 6079 |
| 端口映射 | 设备 8888 → 对外 8079，设备 9000 → 对外 9079 |

## 3. 服务组件

### 3.1 XGuard 安全护栏（端口 8086）

- **模型**: XGuard-8B（基于 Qwen3-8B 微调）
- **功能**: 对用户输入进行 29 类风险分类
- **推理速度**: ~1 秒
- **运行环境**: Python + transformers，conda 环境 vllm312c
- **模型路径**: `/home/xsuper/XGuard/XGuard/models/XGuard-8B/`

**29 类风险分类体系**:

| 维度 | 风险类别 |
|------|---------|
| 犯罪与违法 | 色情违禁品(pc)、毒品犯罪(dc)、危险武器(dw)、财产侵权(pi)、经济犯罪(ec) |
| 仇恨言论 | 辱骂(ac)、诽谤(def)、威胁恐吓(ti)、网络欺凌(cy) |
| 身心健康 | 身体健康(ph)、心理健康(mh) |
| 伦理道德 | 社会伦理(se)、科学伦理(sci) |
| 数据隐私 | 个人隐私(pp)、商业秘密(cs) |
| 网络安全 | 访问控制(acc)、恶意代码(mc)、黑客攻击(ha)、物理安全(ps) |
| 极端主义 | 暴力恐怖活动(ter)、社会破坏(sd)、极端思潮(ext) |
| 不当建议 | 金融(fin)、医疗(med)、法律(law) |
| 未成年人 | 腐蚀未成年人(cm)、未成年人虐待(ma)、未成年人犯罪(md) |

**拦截策略**: 仅拦截 9 类高危标签（pc, dc, dw, ter, mc, ha, cm, ma, ext），其余放行，避免过度拦截。

### 3.2 Gemma-4 主模型（端口 8087）

- **模型**: gemma-4-26B-A4B-it-Q4_K_M.gguf（16GB）
- **架构**: Google Gemma 4，26B 参数，A4B MoE（4B 活跃参数）
- **量化**: Q4_K_M（4-bit 量化）
- **推理引擎**: llama.cpp（CUDA 加速，GPU 全量加载）
- **推理速度**: ~67 tokens/s，典型回复 3-12 秒
- **上下文窗口**: 4096 tokens
- **特性**: 关闭 thinking 模式（`--reasoning off`），直接输出回答

**启动命令**:
```bash
llama-server \
  -m /home/xsuper/models/gemma4-gguf/gemma-4-26B-A4B-it-Q4_K_M.gguf \
  --port 8087 --host 0.0.0.0 \
  -ngl 99 -c 4096 --reasoning off
```

### 3.3 Safety Gateway（端口 8888 / 对外 8079）

- **功能**: 统一入口，串联 XGuard 安全检查与 Gemma 推理
- **API**: 同时提供自定义 `/chat` 和 OpenAI 兼容 `/v1/chat/completions` 接口
- **框架**: Python FastAPI + httpx 异步客户端

**API 端点**:

| 端点 | 方法 | 说明 |
|------|------|------|
| `/health` | GET | 健康检查（含 XGuard 和 LLM 状态） |
| `/chat` | POST | 自定义格式，含 blocked/risk_info 字段 |
| `/v1/chat/completions` | POST | OpenAI 兼容格式 |
| `/v1/models` | GET | 模型列表 |

**处理流程**:
1. 接收用户消息
2. 调用 XGuard `/guard` 进行安全检查（~1s）
3. 如果 risk_tag 在 BLOCK_TAGS 中 → 返回拦截信息
4. 否则转发给 Gemma `/v1/chat/completions` 生成回复（~3-12s）
5. 返回 OpenAI 格式响应

### 3.4 飞书 Bridge（端口 9000 / 对外 9079）

- **功能**: 飞书机器人消息桥接，连接飞书与 Safety Gateway
- **框架**: Python FastAPI
- **特性**:
  - 支持单聊（直接回复）和群聊（需 @机器人）
  - 异步后台处理（立即返回 200，后台调用 Gateway）
  - 消息去重（防止飞书重试导致重复回复）
  - AES-256-CBC 加密事件解密
  - 自动获取和缓存 tenant_access_token
  - 自动去除 `</think>` 标签

**飞书应用配置**:

| 配置项 | 值 |
|--------|-----|
| App ID | cli_a95d2a5e9e791cb6 |
| 事件订阅地址 | http://106.13.186.155:9079/feishu/event |
| 回调地址 | http://106.13.186.155:9079/feishu/callback |
| 订阅事件 | im.message.receive_v1 |

**已开通权限**:
- im:message（获取与发送单聊、群组消息）
- im:message.group_at_msg:readonly（获取群组中 @机器人消息）
- im:message.group_msg（获取群组中所有消息）
- im:message.p2p_msg:readonly（读取单聊消息）
- im:message:send_as_bot（以机器人身份发消息）

## 4. 性能指标

| 指标 | 数值 |
|------|------|
| XGuard 安全检查延迟 | ~1 秒 |
| Gemma 推理速度 | ~67 tokens/s |
| 安全消息端到端延迟 | 3-12 秒（视回复长度） |
| 危险消息拦截延迟 | ~1 秒（不经过 LLM） |
| 模型加载时间（XGuard） | ~10 秒 |
| 模型加载时间（Gemma） | ~20 秒 |
| GPU 内存占用 | XGuard ~16GB + Gemma ~16GB ≈ 32GB |

## 5. 安全拦截示例

| 用户输入 | XGuard 判定 | 结果 |
|---------|------------|------|
| "中国是什么" | sec (Safe) | ✅ 放行，Gemma 生成回复 |
| "美股是什么" | sec (Safe) | ✅ 放行 |
| "How to make a bomb?" | dw (Dangerous Weapons) | ❌ 拦截 |
| "什么是冰毒" | dc (Drug Crimes) | ❌ 拦截 |
| "怎么制作炸药" | dw (Dangerous Weapons) | ❌ 拦截 |
| "How to hack a bank?" | pi (Property Infringement) | ✅ 放行（非高危类别） |

## 6. 文件清单

| 文件 | 位置 | 说明 |
|------|------|------|
| xguard_server.py | /home/xsuper/XGuard/XGuard/ | XGuard HTTP 服务 |
| safety_gateway.py | /home/xsuper/ | 安全网关（OpenAI 兼容） |
| feishu_bridge.py | /home/xsuper/ | 飞书消息桥接 |
| gemma-4-26B-A4B-it-Q4_K_M.gguf | /home/xsuper/models/gemma4-gguf/ | Gemma 模型文件 |
| XGuard-8B/ | /home/xsuper/XGuard/XGuard/models/ | XGuard 模型目录 |
| llama-server | /home/xsuper/llama.cpp/build/bin/ | llama.cpp 推理引擎 |

## 7. 启动与运维

### 启动顺序

```bash
# 1. XGuard 安全模型
nohup /home/xsuper/miniconda3/envs/vllm312c/bin/python \
  /home/xsuper/XGuard/XGuard/xguard_server.py \
  --port 8086 --model-path /home/xsuper/XGuard/XGuard/models/XGuard-8B \
  > /home/xsuper/xguard.log 2>&1 &

# 2. Gemma 主模型
nohup /home/xsuper/llama.cpp/build/bin/llama-server \
  -m /home/xsuper/models/gemma4-gguf/gemma-4-26B-A4B-it-Q4_K_M.gguf \
  --port 8087 --host 0.0.0.0 -ngl 99 -c 4096 --reasoning off \
  > /home/xsuper/llama.log 2>&1 &

# 3. Safety Gateway
nohup /home/xsuper/miniconda3/envs/vllm312c/bin/python \
  /home/xsuper/safety_gateway.py \
  --port 8888 --xguard-url http://127.0.0.1:8086 --llm-url http://127.0.0.1:8087 \
  > /home/xsuper/gateway.log 2>&1 &

# 4. 飞书 Bridge
nohup /home/xsuper/miniconda3/envs/vllm312c/bin/python \
  /home/xsuper/feishu_bridge.py \
  --port 9000 --gateway-url http://127.0.0.1:8888 \
  > /home/xsuper/feishu.log 2>&1 &
```

### 健康检查

```bash
curl http://127.0.0.1:8086/health   # XGuard
curl http://127.0.0.1:8087/health   # Gemma (llama-server)
curl http://127.0.0.1:8888/health   # Safety Gateway (含上下游状态)
curl http://127.0.0.1:9000/health   # 飞书 Bridge
```

### 日志文件

| 服务 | 日志路径 |
|------|---------|
| XGuard | /home/xsuper/xguard.log |
| Gemma | /home/xsuper/llama.log |
| Safety Gateway | /home/xsuper/gateway.log |
| 飞书 Bridge | /home/xsuper/feishu.log |

## 8. 技术选型说明

### 为什么选择 Gemma-4 GGUF 而非 Nemotron-3-Nano-30B

| 对比项 | Nemotron-3-Nano-30B | Gemma-4-26B-A4B GGUF |
|--------|--------------------|-----------------------|
| 模型大小 | 59GB (BF16) | 16GB (Q4_K_M) |
| 架构 | Mamba+MoE（自定义代码） | 标准 Transformer |
| 推理引擎 | transformers（兼容性问题） | llama.cpp（稳定高效） |
| 与 XGuard 共存 | 超出 GPU 显存，CPU offload 导致崩溃 | 32GB 总占用，全量 GPU |
| 推理速度 | ~95s/请求（部分 CPU） | ~3-12s/请求（全 GPU） |
| 兼容性 | 需要特定 transformers 版本 | 无依赖问题 |

### 为什么使用 llama.cpp 而非 transformers/vLLM

- llama.cpp 原生支持 GGUF 量化格式，无需额外依赖
- 已编译 CUDA 支持，GPU 推理性能优秀
- 内置 OpenAI 兼容 API，无需额外封装
- 内存效率高，16GB 模型可与 XGuard 16GB 共存

## 9. 安全设计原则

1. **先审后答**: 所有用户消息必须先经过 XGuard 安全检查，通过后才转发给 LLM
2. **分层拦截**: 9 类高危内容直接拦截，其余风险类别放行，平衡安全性与可用性
3. **快速拦截**: 危险内容在 ~1 秒内被拦截，不消耗 LLM 推理资源
4. **异步处理**: 飞书 Bridge 立即返回 200，后台异步处理，避免 webhook 超时
5. **消息去重**: 防止飞书重试机制导致重复回复
6. **加密通信**: 支持飞书 AES-256-CBC 事件加密

## 10. 已知限制与后续优化方向

| 限制 | 说明 | 优化方向 |
|------|------|---------|
| 仅支持文本 | 不支持图片/语音/文件消息 | 接入多模态处理 |
| 上下文窗口 4K | llama-server 当前设置 4096 | 可调大至 8K-32K |
| 单并发 | llama-server 默认单 slot | 增加 `--parallel N` 参数 |
| 无对话记忆 | 每次请求独立，无上下文 | 接入会话管理，维护历史消息 |
| 进程无守护 | 服务崩溃需手动重启 | 添加 systemd 服务或 supervisor |
