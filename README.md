# 🤖 Build Your Own AI Chatbot App Using Reverse Engineering — For Educational Purposes

> **Disclaimer**: This guide is created strictly for **educational and research purposes only**. Reverse engineering web services may violate Terms of Service of the respective platforms. Always respect platform policies, rate limits, and legal boundaries. The author assumes no responsibility for misuse of this information.

---

## 📖 Table of Contents

- [What Is This Guide About?](#-what-is-this-guide-about)
- [How Reverse-Engineered LLM APIs Work](#-how-reverse-engineered-llm-apis-work)
- [Architecture Deep-Dive](#-architecture-deep-dive)
- [Supported LLM Platforms & Adapters](#-supported-llm-platforms--adapters)
- [Understanding the Free One API Architecture](#-understanding-the-free-one-api-architecture)
- [Step-by-Step: Building Your Own Chatbot](#-step-by-step-building-your-own-chatbot)
- [Adapter Deep-Dive: How Each Reverse Engineering Method Works](#-adapter-deep-dive-how-each-reverse-engineering-method-works)
- [Advanced Techniques](#-advanced-techniques)
- [API Reference](#-api-reference)
- [Configuration Reference](#-configuration-reference)
- [Deployment Methods](#-deployment-methods)
- [Load Balancing & Channel Selection Algorithm](#-load-balancing--channel-selection-algorithm)
- [Watchdog & Heartbeat System](#-watchdog--heartbeat-system)
- [Security Considerations](#-security-considerations)
- [Troubleshooting](#-troubleshooting)
- [Legal & Ethical Considerations](#-legal--ethical-considerations)
- [Credits & References](#-credits--references)

---

## 🔍 What Is This Guide About?

This guide teaches you how to build your own AI chatbot application by leveraging reverse-engineered LLM (Large Language Model) libraries. The core idea is to create a **unified OpenAI-compatible API** that sits in front of multiple free, reverse-engineered LLM services, allowing any OpenAI-compatible client to use them transparently.

The reference implementation is **[Free One API](https://github.com/RockChinQ/free-one-api)** by RockChinQ — an open-source project (AGPL-3.0) that provides a standard OpenAI API format wrapper around multiple reverse-engineered LLM libraries. It acts as a **proxy/gateway** that translates OpenAI API requests into calls to various reverse-engineered LLM web interfaces.

### Key Concepts

- **Reverse Engineering LLM Libraries**: These are open-source Python libraries that intercept and replicate the web browser requests sent to LLM platforms (like ChatGPT, Claude, Gemini/Bard, etc.), allowing programmatic access without paying for official API keys.
- **Adapter Pattern**: Each reverse-engineered library is wrapped in an "adapter" that translates between the OpenAI API format and the library's native format.
- **Channel System**: Each configured adapter instance is called a "channel." Multiple channels can serve the same model, enabling load balancing and failover.
- **API Key Management**: The system generates its own API keys (prefixed `sk-foa`) that clients use to authenticate, completely abstracting the underlying reverse-engineered services.

---

## 🧠 How Reverse-Engineered LLM APIs Work

When you visit `chat.openai.com` in your browser, the frontend JavaScript sends HTTP requests to OpenAI's backend servers. These requests include authentication tokens (session cookies, access tokens) and conversation data. Reverse-engineered libraries replicate these HTTP requests programmatically:

```
┌─────────────┐     HTTP Requests      ┌──────────────────┐
│  Your Code   │ ──────────────────────► │  LLM Web Server  │
│  (via lib)   │ ◄────────────────────── │  (chat.openai.ai)│
└─────────────┘    HTML/JSON Response    └──────────────────┘
```

### Authentication Methods Used by Reverse Engineering Libraries

| Method | How It Works | Used By |
|--------|-------------|---------|
| **Access Token** | Extract from `/api/auth/session` endpoint | acheong08/ChatGPT |
| **Session Token** | Extract `__Secure-next-auth.session-token` cookie | Zai-Kun/reverse-engineered-chatgpt |
| **Cookie String** | Copy full Cookie header from browser DevTools | Claude-API, revTongYi |
| **Email/Password** | Login programmatically using credentials | hugchat |
| **`__Secure-1PSID`** | Extract specific cookie from browser | Bard-API |
| **No Auth** | Public endpoints that require no authentication | gpt4free |

### The Problem This Solves

Without a unified API, you'd need to:
1. Learn each library's unique API
2. Handle different request/response formats
3. Manage authentication for each service separately
4. Implement your own load balancing and failover

With Free One API, you get a single **OpenAI-compatible endpoint** that handles all of this automatically.

---

## 🏗 Architecture Deep-Dive

The system follows a layered architecture:

```
┌──────────────────────────────────────────────────────────────┐
│                     Client Application                        │
│          (Any OpenAI-compatible app: curl, Python SDK, etc.)  │
└───────────────────────────┬──────────────────────────────────┘
                            │ OpenAI API format
                            ▼
┌──────────────────────────────────────────────────────────────┐
│                    Forward Manager                             │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────┐ │
│  │ Auth Check   │  │ Channel      │  │ Response Formatting  │ │
│  │ (API Key)    │  │ Selection    │  │ (SSE/JSON)          │ │
│  └─────────────┘  └──────┬───────┘  └─────────────────────┘ │
└──────────────────────────┼───────────────────────────────────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ Channel 1│ │ Channel 2│ │ Channel N│
        │ Adapter  │ │ Adapter  │ │ Adapter  │
        └────┬─────┘ └────┬─────┘ └────┬─────┘
             ▼            ▼            ▼
     ┌───────────┐ ┌───────────┐ ┌───────────┐
     │ChatGPT Web│ │Claude Web │ │Bard/HF    │
     └───────────┘ └───────────┘ └───────────┘
```

### Core Components

| Component | File | Purpose |
|-----------|------|---------|
| **Application** | `impls/app.py` | Bootstraps everything: loads config, initializes DB, adapters, channels, keys, watchdog |
| **Forward Manager** | `impls/forward/mgr.py` | Handles incoming `/v1/chat/completions` requests, selects channel, formats response |
| **Channel Manager** | `impls/channel/mgr.py` | CRUD operations for channels + load-balancing selection algorithm |
| **Key Manager** | `impls/key/mgr.py` | Creates, lists, revokes API keys |
| **Adapter Registry** | `models/adapter/__init__.py` | Decorator-based adapter registration system |
| **Router Manager** | `impls/router/mgr.py` | Quart (async Flask) HTTP server routing |
| **Watchdog** | `impls/watchdog/wd.py` | Background task scheduler for heartbeat checks |
| **SQLite DB** | `impls/database/sqlite.py` | Persistent storage for channels, keys, logs |
| **Web UI** | `web/` | Vue.js SPA for managing channels and keys |

---

## 📋 Supported LLM Platforms & Adapters

| Adapter | Multi-Round | Stream | Function Call | Auth Required | Status |
|---------|:-----------:|:------:|:-------------:|:-------------:|:------:|
| **acheong08/ChatGPT** | ✅ | ✅ | ❌ | Access Token or Email/Password | ⚠️ Unmaintained |
| **Zai-Kun/reverse-engineered-chatgpt** | ✅ | ✅ | ❌ | Session Token | ⚠️ Unstable |
| **KoushikNavuluri/Claude-API** | ✅ | ❌ | ❌ | Cookie String | ⚠️ Non-stream only |
| **dsdanielpark/Bard-API** | ✅ | ❌ | ❌ | `__Secure-1PSID` Cookie | ⚠️ Non-stream only |
| **xtekky/gpt4free** | ✅ | ✅ | ❌ | None | ⚠️ Very unstable |
| **Soulter/hugging-chat-api** | ✅ | ✅ | ❌ | Email/Password | ✅ More stable |
| **xw5xr6/revTongYi** | ✅ | ✅ | ❌ | Cookie String | ⚠️ Chinese platform |

### Working Methods Ranked by Reliability

Based on code analysis and adapter architecture:

1. **🟢 HuggingChat (Soulter/hugging-chat-api)** — Most reliable. Uses proper login flow, supports streaming, and HuggingFace's interface is relatively stable. Requires a free HuggingFace account.
2. **🟡 GPT4Free (xtekky/gpt4free)** — No auth needed, auto-selects working providers. However, providers are extremely unstable and change frequently. The adapter automatically tests providers and selects working ones.
3. **🟡 Claude (KoushikNavuluri/Claude-API)** — Works but non-stream only. Requires cookie extraction from browser.
4. **🟠 ChatGPT via re_gpt (Zai-Kun/reverse-engineered-chatgpt)** — Newer ChatGPT adapter, uses session token. Requires reverse proxy.
5. **🔴 ChatGPT via acheong08/ChatGPT** — Original adapter, no longer maintained by upstream.
6. **🔴 Bard (dsdanielpark/Bard-API)** — Non-stream, Google frequently changes their interface.
7. **🔴 revTongYi** — Specific to Alibaba's QianWen platform, may require Chinese phone number.

---

## 🔧 Understanding the Free One API Architecture

### How Requests Flow Through the System

When a client sends a request to `/v1/chat/completions`:

1. **Authentication**: The `ForwardAPIGroup` checks the `Authorization: Bearer sk-foa...` header against the key database.
2. **Channel Selection**: The `ChannelManager.select_channel()` method:
   - Filters out disabled channels
   - Filters channels that support the requested path (`/v1/chat/completions`)
   - Filters channels that support the requested model name
   - Evaluates each remaining channel using a scoring algorithm
   - Selects the highest-scoring channel (random among ties)
3. **Model Mapping**: If the channel has a `model_mapping` config, the requested model name is replaced (e.g., `gpt-4` → actual model name).
4. **Query Execution**: The adapter's `query()` method is called, returning an async generator of `Response` objects.
5. **Response Formatting**: Based on whether `stream: true` or `stream: false`:
   - **Stream**: Returns Server-Sent Events (SSE) with `data: {...}` chunks
   - **Non-stream**: Accumulates all chunks and returns a single JSON response

### The Adapter Pattern

All adapters inherit from `LLMLibAdapter` and must implement:

```python
class LLMLibAdapter(metaclass=abc.ABCMeta):
    @classmethod
    def name(cls) -> str: ...           # e.g., "acheong08/ChatGPT"
    @classmethod
    def description(cls) -> str: ...    # Human-readable description
    def supported_models(cls) -> list: ...  # ["gpt-3.5-turbo", "gpt-4"]
    def function_call_supported(cls) -> bool: ...
    def stream_mode_supported(cls) -> bool: ...
    def multi_round_supported(cls) -> bool: ...
    @classmethod
    def config_comment(cls) -> str: ... # Help text for config
    @classmethod
    def supported_path(cls) -> str: ... # "/v1/chat/completions"
    async def test(cls) -> tuple: ...   # Returns (success: bool, error: str)
    async def query(cls, req) -> AsyncGenerator[Response, None]: ...
```

Adapters are registered using the `@adapter.llm_adapter` decorator:

```python
@adapter.llm_adapter
class MyCustomAdapter(llm.LLMLibAdapter):
    @classmethod
    def name(cls) -> str:
        return "my-org/my-adapter"
    # ... implement all abstract methods
```

---

## 🚀 Step-by-Step: Building Your Own Chatbot

### Prerequisites

- Docker (recommended) or Python 3.10+
- A free HuggingFace account (for the most stable adapter)
- Basic familiarity with REST APIs

### Step 1: Deploy Free One API

#### Option A: Docker (Recommended)

```bash
docker run -d \
  -p 3000:3000 \
  --restart always \
  --name free-one-api \
  -v ~/free-one-api/data:/app/data \
  rockchin/free-one-api
```

#### Option B: Docker Compose

```yaml
version: '3'
services:
  free-one-api:
    container_name: free-one-api
    image: rockchin/free-one-api
    ports:
      - "3000:3000"
    restart: always
    volumes:
      - ~/free-one-api/data:/app/data
```

```bash
docker compose up -d
```

#### Option C: Manual Installation

```bash
git clone https://github.com/RockChinQ/free-one-api.git
cd free-one-api

# Build the web UI
cd web && npm install && npm run build && cd ..

# Install Python dependencies
pip install -r requirements.txt

# Start the server
python main.py
```

### Step 2: Access the Admin Panel

Open `http://localhost:3000/` in your browser. Default password is `12345678`.

### Step 3: Create a Channel

1. Navigate to the **Channels** page
2. Click **Add Channel**
3. Enter a name (e.g., "My HuggingChat")
4. Select an adapter (e.g., `Soulter/hugging-chat-api`)
5. Fill in the adapter config:

```json
{
  "email": "your-huggingface-email@example.com",
  "passwd": "your-huggingface-password"
}
```

6. Click **Save** then **Test** the channel

### Step 4: Generate an API Key

1. Go to the **API Keys** page
2. Click **Create Key**
3. Give it a name (e.g., "my-chatbot-key")
4. Copy the generated key (format: `sk-foa...`)

### Step 5: Use the API

#### Using cURL

```bash
curl http://localhost:3000/v1/chat/completions \
  -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-foaYOUR_API_KEY_HERE" \
  -d '{
    "model": "gpt-3.5-turbo",
    "messages": [
      {
        "role": "system",
        "content": "You are a helpful assistant."
      },
      {
        "role": "user",
        "content": "Write a Python function to calculate fibonacci numbers!"
      }
    ],
    "stream": true
  }'
```

#### Using Python (OpenAI SDK)

```python
import openai

openai.api_base = "http://localhost:3000/v1"
openai.api_key = "sk-foaYOUR_API_KEY_HERE"

# Non-streaming
response = openai.ChatCompletion.create(
    model="gpt-3.5-turbo",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Hello! How are you?"}
    ],
    stream=False,
)
print(response.choices[0].message.content)

# Streaming
response = openai.ChatCompletion.create(
    model="gpt-3.5-turbo",
    messages=[
        {"role": "user", "content": "Write a poem about coding."}
    ],
    stream=True,
)
for chunk in response:
    content = chunk.choices[0].delta.get("content", "")
    print(content, end="", flush=True)
```

#### Using Node.js

```javascript
import OpenAI from 'openai';

const openai = new OpenAI({
  apiKey: 'sk-foaYOUR_API_KEY_HERE',
  baseURL: 'http://localhost:3000/v1',
});

async function chat() {
  const stream = await openai.chat.completions.create({
    model: 'gpt-3.5-turbo',
    messages: [{ role: 'user', content: 'Explain quantum computing simply.' }],
    stream: true,
  });

  for await (const chunk of stream) {
    process.stdout.write(chunk.choices[0]?.delta?.content || '');
  }
}

chat();
```

---

## 🔬 Adapter Deep-Dive: How Each Reverse Engineering Method Works

### 1. HuggingChat Adapter (`Soulter/hugging-chat-api`)

**How it works**: This adapter uses the `hugchat` Python library to programmatically interact with HuggingFace's chat interface at `huggingface.co/chat`. The library replicates the browser's HTTP requests to HuggingFace's chat API endpoints.

**Authentication**: Email/password login. The adapter caches cookies to `data/hugchatCookies` so subsequent logins don't require re-authentication.

**Stream Implementation**:
```python
# Creates a new conversation for each query
self.chatbot.change_conversation(self.chatbot.new_conversation())

# Streams tokens one at a time
for resp in self.chatbot.query(text=prompt, stream=True):
    yield response.Response(
        normal_message=resp['token'],  # Each token individually
        finish_reason=response.FinishReason.NULL,
    )

# Clean up conversation after response
self.chatbot.delete_conversation(self.chatbot.current_conversation)
```

**Internal prompt format**: Messages are flattened into a single prompt string:
```
system: You are a helpful assistant.
user: Hello!
assistant: 
```

### 2. GPT4Free Adapter (`xtekky/gpt4free`)

**How it works**: GPT4Free is a meta-library that aggregates dozens of free GPT providers. The adapter automatically tests all available providers and selects ones that actually work.

**Authentication**: None required. GPT4Free uses publicly accessible endpoints.

**Provider Selection Algorithm**:
```python
async def _select_provider(self):
    # Iterates through all providers in g4f.Provider.__all__
    for provider in providers:
        # Skips known-broken providers
        if provider in exclude:
            continue
        
        # Tests each provider with a simple prompt
        resp = await g4f.ChatCompletion.create_async(
            model="gpt-3.5-turbo",
            messages=[{"role": "user", "content": "Hi, My name is Rock."}],
            provider=provider
        )
        
        # Validates the response contains expected content
        if 'Rock' in resp and '<' not in resp:
            # Also tests streaming capability
            if provider.supports_stream:
                # ... test stream mode ...
```

**Important**: The `requests.get` function is temporarily monkey-patched during `g4f` import to bypass its version check:
```python
# Bypass g4f's version check that makes an HTTP request
old_get = requests.get
requests.get = repl  # Returns fake version response
import g4f
requests.get = old_get  # Restore original
```

### 3. ChatGPT via acheong08/ChatGPT

**How it works**: Uses the `revChatGPT` library (V1 AsyncChatbot) to interact with ChatGPT's web interface. Requires a reverse proxy to bypass Cloudflare protection.

**Authentication Methods**:
- Access token (obtained from `https://chat.openai.com/api/auth/session`)
- Email/password
- Optional: reverse proxy URL

**Reverse Proxy**: ChatGPT enforces Cloudflare protection. A reverse proxy is needed to route requests:
```python
self._chatbot = chatgpt.AsyncChatbot(
    config=self.cfg,
    base_url=self.reverse_proxy,  # Default: https://chatproxy.rockchin.top/api/
)
```

**Duplicate Response Handling**: The reverse proxy sometimes causes the model to repeat previous text. The adapter has auto-detection:
```python
# Skip if message matches any assistant message in the conversation
if any([message == msg for msg in assistant_messages]) and self.auto_ignore_duplicated:
    continue
```

### 4. ChatGPT via Zai-Kun/reverse-engineered-chatgpt

**How it works**: A newer ChatGPT reverse-engineering library using session tokens instead of access tokens.

**Authentication**: Uses the `__Secure-next-auth.session-token` cookie value.

**Key difference from acheong08**: Uses a synchronous `SyncChatGPT` client with context manager pattern:
```python
with self.chatbot as chatbot:
    conversation = chatbot.create_new_conversation()
    for message in conversation.chat(user_input=prompt):
        yield response.Response(normal_message=message["content"])
    chatbot.delete_conversation(conversation.conversation_id)
```

### 5. Claude Adapter (`KoushikNavuluri/Claude-API`)

**How it works**: Interacts with Claude's web interface at `claude.ai` using cookie-based authentication.

**Authentication**: Full cookie string from browser DevTools (F12 → Network → Copy Cookie header).

**Limitation**: Non-streaming only. Each query creates a new conversation, sends the prompt, gets the full response, and deletes the conversation.

```python
conversation_id = self.chatbot.create_new_chat()['uuid']
resp_text = self.chatbot.send_message(prompt, conversation_id)
self.chatbot.delete_conversation(conversation_id)
```

### 6. Bard Adapter (`dsdanielpark/Bard-API`)

**How it works**: Uses the `bardapi` library to access Google Bard (now Gemini).

**Authentication**: Requires the `__Secure-1PSID` cookie value from the browser.

**Limitation**: Non-streaming only. Single response returned all at once.

### 7. QianWen/TongYi Adapter (`xw5xr6/revTongYi`)

**How it works**: Reverse-engineers Alibaba's TongYi QianWen (通义千问) at `qianwen.aliyun.com`.

**Authentication**: Full cookie string from browser.

**Stream support**: Yes, uses `stream=True` parameter and iterates over streaming responses, tracking incremental content with `prev_text`.

---

## 🎓 Advanced Techniques

### Writing Your Own Custom Adapter

To add support for a new reverse-engineered LLM library, create a new adapter:

```python
# free_one_api/impls/adapter/my_adapter.py

import typing
import random
from ...models import adapter
from ...models.adapter import llm
from ...entities import request, response, exceptions
from ...models.channel import evaluation

@adapter.llm_adapter  # This decorator registers the adapter
class MyCustomAdapter(llm.LLMLibAdapter):
    
    @classmethod
    def name(cls) -> str:
        return "myorg/my-reverse-lib"
    
    @classmethod
    def description(cls) -> str:
        return "Custom adapter for my reverse-engineered LLM library."
    
    def supported_models(self) -> list[str]:
        return ["gpt-3.5-turbo", "gpt-4", "my-custom-model"]
    
    def function_call_supported(self) -> bool:
        return False
    
    def stream_mode_supported(self) -> bool:
        return True
    
    def multi_round_supported(self) -> bool:
        return True
    
    @classmethod
    def config_comment(cls) -> str:
        return """Provide your API token:
{
    "token": "your-api-token"
}"""
    
    @classmethod
    def supported_path(cls) -> str:
        return "/v1/chat/completions"
    
    def __init__(self, config: dict, eval: evaluation.AbsChannelEvaluation):
        self.config = config
        self.eval = eval
        # Initialize your reverse-engineered client here
        self.client = MyReverseLibClient(token=config.get("token"))
    
    async def test(self) -> typing.Union[bool, str]:
        try:
            # Send a simple test message
            resp = self.client.chat("Hello, respond 'hi' only.")
            return True, ""
        except Exception as e:
            return False, str(e)
    
    async def query(self, req: request.Request) -> typing.AsyncGenerator[response.Response, None]:
        # Build the prompt from messages
        prompt = ""
        for msg in req.messages:
            prompt += f"{msg['role']}: {msg['content']}\n"
        prompt += "assistant: "
        
        random_int = random.randint(0, 1000000000)
        
        # Stream the response
        for chunk in self.client.chat_stream(prompt):
            yield response.Response(
                id=random_int,
                finish_reason=response.FinishReason.NULL,
                normal_message=chunk,
                function_call=None
            )
        
        # Send final stop response
        yield response.Response(
            id=random_int,
            finish_reason=response.FinishReason.STOP,
            normal_message="",
            function_call=None
        )
```

Then register it in `impls/app.py`:

```python
from .adapter import my_adapter

adapter_config_mapping = {
    # ... existing adapters ...
    "myorg_my-reverse-lib": my_adapter.MyCustomAdapter,
}
```

### Model Mapping

You can map request model names to different model names on the channel level. For example, if a client requests `gpt-4` but your adapter only supports a different model name:

```json
{
    "model_mapping": {
        "gpt-4": "actual-model-name-on-platform",
        "gpt-3.5-turbo": "fallback-model"
    }
}
```

### Combining with one-api

Free One API can be combined with [songquanpeng/one-api](https://github.com/songquanpeng/one-api) (which handles paid official APIs). This gives you:
- Free reverse-engineered channels via Free One API
- Paid official channels via one-api
- Unified OpenAI format across both

### Building a Chatbot Frontend

Since the API is OpenAI-compatible, you can use any chatbot UI framework:

- **ChatGPT-Next-Web**: Set `OPENAI_API_BASE=http://localhost:3000/v1`
- **LobeChat**: Configure custom API endpoint
- **Open WebUI**: Set OpenAI API URL to your Free One API instance
- **Custom frontend**: Just use the OpenAI JavaScript/Python SDK

### Rate Limiting & Concurrency

The system handles concurrent requests through:
1. **Async architecture**: Built on Quart (async Flask) with Python asyncio
2. **Load balancing**: Distributes requests across multiple channels
3. **Watchdog**: Automatically disables channels that fail heartbeat checks

---

## 📡 API Reference

### Chat Completion Endpoint

```
POST /v1/chat/completions
```

**Request Body** (OpenAI-compatible):
```json
{
    "model": "gpt-3.5-turbo",
    "messages": [
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Hello!"}
    ],
    "stream": false,
    "functions": []
}
```

**Response (Non-stream)**:
```json
{
    "id": "chatcmpl-001ReGu4abcdefghijklmno",
    "object": "chat.completion",
    "created": 1700000000,
    "model": "gpt-3.5-turbo",
    "choices": [
        {
            "index": 0,
            "message": {
                "role": "assistant",
                "content": "Hello! How can I help you today?"
            },
            "finish_reason": "stop"
        }
    ],
    "usage": {
        "prompt_tokens": 20,
        "completion_tokens": 8,
        "total_tokens": 28
    }
}
```

**Response (Stream)** — Server-Sent Events:
```
data: {"id":"chatcmpl-...","object":"chat.completion.chunk","created":1700000000,"model":"gpt-3.5-turbo","choices":[{"index":0,"delta":{"content":"Hello"},"finish_reason":null}]}

data: {"id":"chatcmpl-...","object":"chat.completion.chunk","created":1700000000,"model":"gpt-3.5-turbo","choices":[{"index":0,"delta":{"content":"!"},"finish_reason":null}]}

data: {"id":"chatcmpl-...","object":"chat.completion.chunk","created":1700000000,"model":"gpt-3.5-turbo","choices":[{"index":0,"delta":{},"finish_reason":"stop"}]}

data: [DONE]
```

### Admin API Endpoints

All admin endpoints require `Authorization: Bearer <md5_of_admin_password>` header.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/channel/list` | List all channels |
| POST | `/api/channel/create` | Create a new channel |
| DELETE | `/api/channel/delete/<id>` | Delete a channel |
| GET | `/api/channel/details/<id>` | Get channel details |
| PUT | `/api/channel/update/<id>` | Update channel |
| POST | `/api/channel/enable/<id>` | Enable a channel |
| POST | `/api/channel/disable/<id>` | Disable a channel |
| POST | `/api/channel/test/<id>` | Test a channel (returns latency) |
| GET | `/api/adapter/list` | List available adapters |
| GET | `/api/adapter/details/<name>` | Get adapter details |
| GET | `/api/key/list` | List API keys |
| GET | `/api/key/raw/<id>` | Get raw API key value |
| POST | `/api/key/create` | Create new API key |
| DELETE | `/api/key/revoke/<id>` | Revoke an API key |
| GET | `/api/log/list?page=0&capacity=20` | List logs (paginated) |
| DELETE | `/api/log/delete?start=0&end=100` | Delete logs by range |
| GET | `/api/info/version` | Get version (no auth required) |

### Channel Data Format

```json
{
    "id": 1,
    "name": "My Channel",
    "adapter": {
        "type": "Soulter/hugging-chat-api",
        "config": {
            "email": "user@example.com",
            "passwd": "password123"
        }
    },
    "model_mapping": {
        "gpt-4": "meta-llama/Llama-2-70b-chat-hf"
    },
    "enabled": true,
    "latency": 1.23
}
```

---

## ⚙️ Configuration Reference

Configuration file: `data/config.yaml`

```yaml
# Database settings
database:
  type: sqlite                    # Only SQLite supported
  path: ./data/free_one_api.db    # Database file path

# Watchdog / heartbeat settings
watchdog:
  heartbeat:
    interval: 1800    # Heartbeat check interval in seconds (30 min)
    timeout: 300      # Single channel test timeout in seconds (5 min)
    fail_limit: 3     # Disable channel after this many consecutive failures

# Server settings
router:
  port: 3000          # Server listen port
  token: '12345678'   # Admin panel password

# Web frontend
web:
  frontend_path: ./web/dist/  # Path to Vue.js frontend build

# Logging
logging:
  debug: false        # Enable debug logging

# Random advertisement (appended to responses)
random_ad:
  enabled: false      # Enable/disable ads
  rate: 0.05          # Probability of ad appearing (5%)
  ad_list:
    - ' (Sponsored by Free One API)'

# Adapter-specific settings
adapters:
  acheong08_ChatGPT:
    reverse_proxy: https://chatproxy.rockchin.top/api/
    auto_ignore_duplicated: true  # Auto-filter repeated text
```

---

## 🐳 Deployment Methods

### Docker (Production)

```bash
docker run -d \
  -p 3000:3000 \
  --restart always \
  --name free-one-api \
  -v ~/free-one-api/data:/app/data \
  rockchin/free-one-api
```

### With Nginx Reverse Proxy

```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        
        # Required for SSE streaming
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 300s;
    }
}
```

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `DEBUG` | Enable debug logging (`true`/`false`) | `false` |
| `DUMP_SCORE_RECORDS` | Dump channel selection scores | `false` |

---

## ⚖️ Load Balancing & Channel Selection Algorithm

The system uses a sophisticated channel selection algorithm:

### Hard Filters (exclude channels)

1. **Disabled channels** — Channels that are manually disabled
2. **Path mismatch** — Channels that don't support the requested API path
3. **Model mismatch** — Channels that don't support the requested model name (considering `model_mapping`)

### Soft Scoring (rank channels)

After hard filtering, each remaining channel is scored by `ChannelEvaluation.evaluate()`:

```python
score = round(lastUseTime / 5) * 5 - using_amount * 5
```

Where:
- `lastUseTime`: Seconds since the channel was last used (higher = less recently used = preferred)
- `using_amount`: Number of currently active queries on the channel (penalty for busy channels)

This means:
- **Idle channels are preferred** (higher `lastUseTime` score)
- **Busy channels get penalized** (-5 per active query)
- The scoring ensures automatic load distribution

### Tie-Breaking

When multiple channels have the same highest score, one is selected randomly:

```python
random.seed(time.time())
return random.choice(max_score_channels)
```

---

## 🐕 Watchdog & Heartbeat System

The watchdog runs background tasks on a configurable interval (default: 30 minutes).

### Heartbeat Task

For each enabled channel:
1. Wait a random delay (0–30 seconds) to avoid thundering herd
2. Call the channel's `adapter.test()` method with a timeout
3. If the test fails, increment `fail_count`
4. If `fail_count >= fail_limit` (default: 3), **automatically disable the channel**
5. If the test succeeds, reset `fail_count` and update `latency`

```python
class HeartBeatTask(task.AbsTask):
    async def trigger(self):
        for chan in self.channel.channels:
            fail_count = await ch.heartbeat(timeout=self.cfg["timeout"])
            if fail_count >= self.cfg["fail_limit"]:
                await self.channel.disable_channel(ch.id)
```

---

## 🔒 Security Considerations

### Authentication Layers

1. **Admin Panel**: Protected by MD5-hashed password stored in config
2. **API Keys**: Generated keys (`sk-foa` + 45 random chars) validated on every request
3. **Channel Configs**: Stored in SQLite database (not exposed via API)

### Known Security Issues

- **MD5 for password hashing**: The admin password is hashed with MD5 (not cryptographically secure)
- **No HTTPS**: The server runs plain HTTP; use a reverse proxy (Nginx/Caddy) with TLS
- **No rate limiting**: No built-in rate limiting on the API
- **Token in config**: Admin token stored in plaintext in `config.yaml`

### Recommended Security Hardening

1. Change the default admin password immediately
2. Use HTTPS via a reverse proxy
3. Don't expose the admin panel publicly
4. Restrict API key access by IP if possible
5. Regularly rotate API keys
6. Monitor logs for abuse

---

## 🐛 Troubleshooting

### Common Issues

| Problem | Cause | Solution |
|---------|-------|---------|
| "No suitable channel found" | No enabled channels match the requested model | Add a channel supporting that model |
| 401 Unauthorized | Invalid API key | Check your `sk-foa...` key |
| Channel auto-disabled | Heartbeat check failed 3+ times | Fix the underlying auth/credential issue, then re-enable |
| Duplicate text in responses | Known issue with ChatGPT reverse proxy | Set `auto_ignore_duplicated: true` in config |
| "No provider available" (g4f) | All gpt4free providers are down | Try again later, providers are volatile |
| Stream not working | Adapter doesn't support streaming | Use `stream: false` or switch adapters |
| Cloudflare blocking | ChatGPT reverse proxy blocked | Set up your own reverse proxy |

### Debug Mode

Enable debug logging in `data/config.yaml`:
```yaml
logging:
  debug: true
```

Or set environment variable:
```bash
DEBUG=true python main.py
```

---

## ⚖️ Legal & Ethical Considerations

### Important Warnings

1. **Terms of Service**: Reverse engineering web interfaces typically violates the Terms of Service of the respective platforms. This includes OpenAI, Anthropic, Google, and others.
2. **Rate Limits**: Even if you can access these services programmatically, they have rate limits. Aggressive use may result in account bans.
3. **Data Privacy**: Your conversations are being processed by third-party services. Be cautious about sending sensitive information.
4. **Intellectual Property**: The responses generated by these models are subject to the respective platforms' terms regarding output ownership.
5. **Stability**: Reverse-engineered interfaces break frequently when platforms update their APIs. This is not suitable for production use.
6. **AGPL-3.0 License**: Free One API is licensed under AGPL-3.0, which requires you to make source code available if you host the service for others over a network.

### Ethical Use Guidelines

- Use for personal learning and research only
- Respect platform rate limits
- Do not use for commercial purposes without proper API subscriptions
- Consider supporting the platforms by purchasing official API access
- Do not redistribute access to others without proper authorization
- Be transparent about the limitations and risks

---

## 📚 Credits & References

### Project

- **Free One API** by [RockChinQ](https://github.com/RockChinQ) — [GitHub Repository](https://github.com/RockChinQ/free-one-api)
- License: GNU Affero General Public License v3.0 (AGPL-3.0)

### Reverse Engineering Libraries

| Library | Repository | Target |
|---------|-----------|--------|
| acheong08/ChatGPT | https://github.com/acheong08/ChatGPT | ChatGPT Web |
| Zai-Kun/reverse-engineered-chatgpt | https://github.com/Zai-Kun/reverse-engineered-chatgpt | ChatGPT Web |
| KoushikNavuluri/Claude-API | https://github.com/KoushikNavuluri/Claude-API | Claude Web |
| dsdanielpark/Bard-API | https://github.com/dsdanielpark/Bard-API | Google Bard |
| xtekky/gpt4free | https://github.com/xtekky/gpt4free | Multiple platforms |
| Soulter/hugging-chat-api | https://github.com/Soulter/hugging-chat-api | HuggingFace Chat |
| xw5xr6/revTongYi | https://github.com/xw5xr6/revTongYi | Alibaba QianWen |

### Related Projects

- [songquanpeng/one-api](https://github.com/songquanpeng/one-api) — OpenAI API management & distribution system (for official paid APIs)
- [flyingpot/chatgpt-proxy](https://github.com/flyingpot/chatgpt-proxy) — Reverse proxy for ChatGPT

### Tech Stack

| Component | Technology |
|-----------|-----------|
| Backend | Python 3.10, Quart (async Flask) |
| Frontend | Vue.js 3, Element Plus, Vite |
| Database | SQLite (via aiosqlite) |
| Server | Quart built-in async server |
| Container | Docker (python:3.10-slim-bullseye) |
| Token Counting | tiktoken (OpenAI's tokenizer) |

---

> **Remember**: This guide is for **educational purposes only**. Building production applications using reverse-engineered APIs is unreliable, potentially illegal, and unethical. If you need reliable LLM access, consider using official APIs from [OpenAI](https://platform.openai.com), [Anthropic](https://console.anthropic.com), [Google](https://ai.google.dev), or [HuggingFace](https://huggingface.co).
