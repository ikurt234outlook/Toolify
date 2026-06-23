# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Toolify is a Python-based middleware proxy that injects OpenAI-compatible function calling capabilities into LLMs that don't natively support it. It acts as an intermediary between client applications and upstream LLM APIs, parsing XML-formatted tool calls from model responses and converting them to the standard OpenAI `tool_calls` format.

## Architecture

**Single-File Application**: The entire application logic resides in `main.py` (~2750 lines), a deliberately monolithic design for simplicity and deployment ease.

**Core Components**:
- **TokenCounter**: Token counting using `tiktoken` with support for multiple model encodings
- **StreamingFunctionCallDetector**: Real-time detection and parsing of function calls in streaming responses
- **Function Call Parser**: XML-based parsing of tool calls with validation against tool schemas
- **Message Preprocessor**: Converts `role=tool` messages to user-formatted text and handles role conversions

**Request Flow**:
1. Client sends `/v1/chat/completions` request with tools
2. Toolify injects system prompt instructing the model to output tool calls in XML format
3. Request is proxied to configured upstream service
4. Response is parsed for XML tool calls (triggered by unique signal)
5. Tool calls are converted to OpenAI format and returned to client

**Routing**: Model-based routing to different upstream services (e.g., OpenAI, Google Gemini). Supports model aliases (`gemini-2.5:gemini-2.5-pro`) for random selection among variants.

## Development Commands

### Running the Server

**Python directly**:
```bash
python main.py
```

**Docker Compose** (recommended for deployment):
```bash
docker-compose up -d
```

The server runs on `http://localhost:8000` by default.

### Configuration

1. Copy the example config:
```bash
cp config.example.yaml config.yaml
```

2. Edit `config.yaml` to configure:
   - Upstream services (API keys, base URLs, supported models)
   - Client authentication keys (`allowed_keys`)
   - Feature toggles (logging, function calling, passthrough modes)

### Dependencies

Install with:
```bash
pip install -r requirements.txt
```

Key dependencies: FastAPI, uvicorn, httpx, Pydantic, PyYAML, tiktoken.

## Key Implementation Details

**Function Calling Injection**:
- Tool definitions are injected via system prompt using `generate_function_prompt()`
- Models output tool calls in XML format with a unique trigger signal
- `parse_function_calls_xml()` extracts and validates tool calls
- Schema validation ensures parameter types match tool definitions

**Streaming Support**:
- `StreamingFunctionCallDetector` parses chunks in real-time
- Handles partial XML chunks and buffers incomplete function calls
- Emits properly formatted `tool_calls` delta chunks

**Message Preprocessing** (`preprocess_messages()`):
- Converts `role=tool` messages to user messages with formatted context
- Converts assistant `tool_calls` back to XML for upstream context
- Optionally converts `role=developer` to `role=system`

**Error Handling**:
- Automatic retry for function call parsing failures (`enable_fc_error_retry`)
- Retry for upstream connection errors with exponential backoff
- Comprehensive error responses matching OpenAI API format

**Token Counting**:
- Uses tiktoken with model-specific encodings
- Falls back to `o200k_base` for unknown models
- Provides accurate usage statistics in responses

## Configuration Notes

**Model Passthrough**: When `features.model_passthrough: true`, all requests route to the service named `openai`, ignoring model-based routing.

**Key Passthrough**: When `features.key_passthrough: true`, client-provided API keys are forwarded upstream instead of using configured keys.

**Custom Prompts**: `features.prompt_template` allows customizing the function calling instruction prompt. Must include `{tools_list}` and `{trigger_signal}` placeholders.

**Logging Levels**: DEBUG, INFO, WARNING, ERROR, CRITICAL, or DISABLED.

## Testing

No automated tests are present in this repository. Manual testing involves:
1. Starting the server with a valid `config.yaml`
2. Making OpenAI SDK calls to `http://localhost:8000/v1`
3. Verifying function calling behavior with different models