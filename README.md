[中文](https://github.com/wbxs2077/gemini-for-claude-code/blob/main/README_CN.md) | [English](https://github.com/wbxs2077/gemini-for-claude-code/blob/main/README.md)

# Gemini API Resilient & Intelligent Gateway

This project provides a robust, production-ready gateway to interact with the Google Gemini API. It's designed to offer high availability and resilience by intelligently managing multiple API keys, handling failures gracefully, and providing real-time observability.

## Core Features

- **Automatic Failover & Rotation**: Automatically switches to a healthy API key when the current one fails due to rate limits or other errors.
- **Stateful Key Management**: Intelligently disables keys based on the type of error:
    - **Temporary Deactivation**: Keys hitting rate limits are temporarily disabled for a configurable duration (defaulting to 24 hours).
    - **Permanent Deactivation**: Invalid or revoked keys are permanently removed from the pool.
- **Real-time Status Monitoring**: A dedicated `/v1/keys/status` endpoint provides a live view of the health of all managed API keys, with sensitive parts of the keys masked for security.
- **Distinct Handling Logic**: Implements separate, optimized strategies for:
    - **Non-Streaming Requests**: Retries a request across all available keys to maximize success probability.
    - **Streaming Requests**: Fails fast on a single key to ensure low latency and lets the client decide whether to retry.
- **Configuration via Environment Variables**: All settings are managed through a `.env` file, following the Twelve-Factor App methodology.

## How It Works

The gateway sits between your client applications and the Gemini API. Its core intelligence resides in the `ApiKeyManager` class, which maintains the state of all provided API keys.

1.  **Request Reception**: The gateway receives a request at `/v1/messages`.
2.  **Logic Fork**: It checks if the request is for `streaming`.
    -   **Non-Streaming Path**: The gateway attempts the request in a loop, trying each available key one by one. If a key fails due to a rate limit or invalidity, the `ApiKeyManager` updates its state, and the loop continues with the next key. It returns a successful response from the first key that works, or a `503 Service Unavailable` error if all keys fail.
    -   **Streaming Path**: The gateway picks one available key and initiates the stream. It's designed to fail fast if this key encounters an error, allowing the client to quickly receive the error and decide on its own retry strategy.
3.  **State Management**: The `ApiKeyManager` is the brain. It moves keys between `available`, `temporarily_disabled`, and `permanently_disabled` pools based on real-time API feedback, ensuring that only healthy keys are in rotation.

## Getting Started

### 1. Prerequisites
- Python 3.8+

### 2. Setup

**Clone the repository:**
```bash
git clone <repository_url>
cd gemini-for-claude-code
```

**Create a virtual environment:**
```bash
python -m venv venv
source venv/bin/activate  # On Windows, use `venv\Scripts\activate`
```

**Install dependencies:**
```bash
pip install -r requirements.txt
```

**Configure your environment:**
Create a `.env` file in the root of the project and add your configuration. You can copy the example file (`.env.example`) to get started.

```env
# Your Gemini API Keys, comma-separated. At least two are recommended for failover.
GEMINI_API_KEY=your_gemini_api_key_1,your_gemini_api_key_2

# Optional: The duration in minutes to temporarily disable a key after it hits a rate limit.
KEY_COOLDOWN_MINUTES=1440 # 24 hours

# Optional: Set the logging level. Options are: "DEBUG", "INFO", "WARNING", "ERROR".
LOG_LEVEL="INFO"
```

### 3. Running the Server

Start the application with Uvicorn:
```bash
uvicorn server:app --host 0.0.0.0 --port 8082
```
The server will now be running on `http://localhost:8082`.

## API Usage

### Send a Message

- **Endpoint**: `POST /v1/messages`
- **Description**: Forwards a request to the Gemini API using an available key. This endpoint is compatible with the OpenAI/Anthropic Messages API format.

**Example cURL (Non-Streaming):**
```bash
curl -X POST http://localhost:8082/v1/messages \
-H "Content-Type: application/json" \
-d '{
    "model": "gemini-1.5-pro-latest",
    "messages": [{"role": "user", "content": "Hello, what is the weather like today?"}],
    "stream": false
}'
```

### Get Key Status

- **Endpoint**: `GET /v1/keys/status`
- **Description**: Retrieves the real-time status of all API keys managed by the gateway.

**Example cURL:**
```bash
curl http://localhost:8082/v1/keys/status
```

**Example Response:**
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

## Architectural Limitations & Future Work

- **Island of State**: The current implementation holds the `ApiKeyManager` state in memory for each running process. This presents two challenges:
    1.  **State is lost on restart.**
    2.  **It prevents horizontal scaling** (running multiple workers or nodes), as each process would have its own independent state.
    -   **Solution**: For a true production-grade deployment, the state should be externalized to a shared store like **Redis**.

- **Missing Authentication**: The gateway endpoints are currently unsecured.
    - **Solution**: A production deployment should implement an authentication layer (e.g., requiring an `X-API-Key` header) to protect the gateway from unauthorized access.

## License

This project is licensed under the MIT License.
