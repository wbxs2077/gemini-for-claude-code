[中文](https://github.com/wbxs2077/gemini-for-claude-code/blob/main/README_CN.md) | [English](https://github.com/wbxs2077/gemini-for-claude-code/blob/main/README.md)

# Gemini API 智能高可用网关

本项目是一个为 Google Gemini API 设计的、生产级的智能网关。它旨在通过智能地管理多个API密钥、优雅地处理故障并提供实时状态监控，来为您的服务提供高可用性和高弹性。

## 核心功能

- **自动故障转移与轮换**: 当正在使用的API密钥因速率限制或其他错误而失败时，系统会自动切换到一个健康的备用密钥。
- **状态化密钥管理**: 根据错误类型，智能地禁用密钥：
    - **临时禁用**: 触发速率限制的密钥将被临时禁用一段时间，时长可配置（默认为24小时）。
    - **永久禁用**: 无效或已被吊销的密钥将被从密钥池中永久移除。
- **实时状态监控**: 特设的 `/v1/keys/status` API端点可以实时查看所有受管密钥的健康状况，为安全起见，密钥的敏感部分会被自动屏蔽。
- **差异化的处理逻辑**: 为不同请求类型实现各自优化的策略：
    - **非流式请求**: 在所有可用密钥之间循环重试一次请求，以最大化成功的概率。
    - **流式请求**: 在单个密钥上实现快速失败，以保证低延迟，并让客户端自行决定是否重试。
- **环境变量配置**: 所有的设置项都通过项目根目录下的 `.env` 文件进行管理，遵循十二因数应用（Twelve-Factor App）的最佳实践。

## 工作原理

本网关位于您的客户端应用和Gemini API之间。其核心智能在于 `ApiKeyManager` 类，它负责维护所有已提供的API密钥的状态。

1.  **接收请求**: 网关在 `/v1/messages` 端点接收到一个请求。
2.  **逻辑分叉**: 系统检查请求体中 `stream` 字段的值。
    -   **非流式路径**: 网关在一个循环中尝试请求，逐一使用所有可用的密钥。如果一个密钥因为速率限制或无效而失败，`ApiKeyManager` 会更新该密钥的状态，然后循环会继续使用下一个密钥。网关会返回第一个成功密钥的响应，或者在所有密钥都失败后返回 `503 Service Unavailable` 错误。
    -   **流式路径**: 网关仅选择一个可用密钥并发起流式请求。这种设计旨在当该密钥遇到问题时能够快速失败，让客户端能迅速收到错误并自行决定其重试策略。
3.  **状态管理**: `ApiKeyManager` 是整个系统的大脑。它根据API的实时反馈，在 `available`（可用）、`temporarily_disabled`（临时禁用）和 `permanently_disabled`（永久禁用）这三个池中移动密钥，确保只有健康的密钥处于轮换中。

## 快速上手

### 1. 环境要求
- Python 3.8 或更高版本

### 2. 安装与配置

**克隆仓库:**
```bash
git clone <repository_url>
cd gemini-for-claude-code
```

**创建并激活虚拟环境:**
```bash
python -m venv venv
source venv/bin/activate  # 在 Windows 上, 使用 `venv\Scripts\activate`
```

**安装依赖:**
```bash
pip install -r requirements.txt
```

**配置环境变量:**
在项目根目录创建一个 `.env` 文件，并填入您的配置。您可以复制 `.env.example` 文件作为模板。

```env
# 您的 Gemini API 密钥，用逗号分隔。为了实现故障转移，推荐至少提供两个。
GEMINI_API_KEY=your_gemini_api_key_1,your_gemini_api_key_2

# 可选: 当一个密钥达到速率限制后，临时禁用它的时长（分钟）。
KEY_COOLDOWN_MINUTES=1440 # 24 小时

# 可选: 设置日志级别。可选值为: "DEBUG", "INFO", "WARNING", "ERROR"。
LOG_LEVEL="INFO"
```

### 3. 运行服务

使用 Uvicorn 启动应用:
```bash
uvicorn server:app --host 0.0.0.0 --port 8082
```
现在，服务已在 `http://localhost:8082` 上运行。

## API 使用说明

### 发送消息

- **API端点**: `POST /v1/messages`
- **描述**: 使用一个可用的密钥，将请求转发给 Gemini API。此端点与 OpenAI/Anthropic 的 Messages API 格式兼容。

**示例 cURL (非流式):**
```bash
curl -X POST http://localhost:8082/v1/messages \
-H "Content-Type: application/json" \
-d '{
    "model": "gemini-1.5-pro-latest",
    "messages": [{"role": "user", "content": "你好，今天天气怎么样？"}],
    "stream": false
}'
```

### 获取密钥状态

- **API端点**: `GET /v1/keys/status`
- **描述**: 获取由本网关管理的所有API密钥的实时状态。

**示例 cURL:**
```bash
curl http://localhost:8082/v1/keys/status
```

**响应示例:**
```json
{
  "available_keys": [
    "AIzaSy...o_rA"
  ],
  "temporarily_disabled_keys": {
    "AIzaSy...t_bM": "2023-10-27T10:30:00Z"
  },
  "permanently_disabled_keys": [
    "AIzaSy...x_pQ"
  ],
  "summary": {
    "total": 3,
    "available": 1,
    "temporarily_disabled": 1,
    "permanently_disabled": 1
  }
}
```

## 架构局限性与未来工作

- **状态孤岛 (Island of State)**: 当前的实现将 `ApiKeyManager` 的状态保存在每个运行进程的内存中。这带来了两个挑战：
    1.  **服务重启后状态会丢失。**
    2.  **这阻碍了服务的水平扩展** (运行多个worker或节点)，因为每个进程都会有自己独立的状态。
    -   **解决方案**: 对于一个真正的生产级部署，应将状态外部化到一个共享的存储中，例如 **Redis**。

- **缺少认证机制**: 网关的API端点目前是开放的，不受保护。
    - **解决方案**: 生产环境部署时应实现一个认证层 (例如, 要求在请求头中包含一个 `X-API-Key`)，以保护网关免受未授权访问。

## 许可证

本项目基于 MIT 许可证授权。
