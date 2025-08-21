# EchoSight

EchoSight is an open source FastAPI server that streams audio and image input to a multimodal model and returns low‑latency speech and text responses. It combines model streaming, VAD preprocessing, and SSE/WebSocket APIs to support realtime camera + microphone assistants similar in behavior to Google's Project Astra or GPT‑4o realtime vision, but fully open source.

Core files
- `model_server.py` — main FastAPI app and the `StreamManager` implementation.
- `vad_utils.py` — Silero VAD wrapper and helpers (`VadOptions`, `get_speech_timestamps`, `run_vad`, `SileroVADModel`).
- `silero_vad.onnx` — ONNX VAD model used by `vad_utils.py` (if included in repo).
- `pyproject.toml` — project metadata and dependencies.

Quick overview
- Ingest audio (base64 / raw wav), optional image frames, and structured messages.
- Apply VAD (Silero) to trim audio before model consumption (`run_vad` in `vad_utils.py`).
- Prefill model context and stream generation via `StreamManager.prefill` and `StreamManager.generate` in `model_server.py`.
- Expose HTTP and WebSocket endpoints for streaming, completions, options and feedback.

Table of contents
- Features
- Requirements
- Install & Run (PowerShell)
- API / Usage
- Internals & important symbols
- Customization
- Enable "GPT‑OSS-120B for all clients"
- Troubleshooting
- License & Notes

## Features
- Real‑time streaming API (HTTP SSE + WebSocket).
- Voice activity detection using Silero VAD (`VadOptions`, `get_speech_timestamps`, `run_vad`).
- Model streaming / TTS integration via Hugging Face style model loading in `model_server.py`.
- Session, logging and feedback hooks (logs saved under `./log_data/<port>/`).

## Requirements
- Python 3.12+ (this repository's `pyproject.toml` and `uv.lock` require Python >=3.12).
- Common required packages (examples):
  - fastapi, uvicorn, transformers, torch, onnxruntime, librosa, soundfile, pillow, aiofiles
- GPU recommended for model inference but CPU will work for small tests.

This repo includes `pyproject.toml` and a lockfile `uv.lock` with pinned dependency versions. If you use `astral-uv` as your environment / dependency manager, prefer installing from `pyproject.toml` and `uv.lock` so the exact, tested dependency set is used.

Note: use the versions pinned in `pyproject.toml` and `uv.lock` when available.

## Install & Run (PowerShell)
1. Create and activate a virtual environment:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

2. Install dependencies.

Preferred (astral-uv): this repository includes `pyproject.toml` and `uv.lock`. Use `astral-uv` to install the locked dependency set so you get the exact, tested versions.

For Linux use:
```bash
# install astral-uv (one-time)
curl -LsSf https://astral.sh/uv/install.sh | sh
```
For Windows use:
```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Sync Dependencies:
```powershell
# sync/install dependencies from pyproject.toml + uv.lock
astral-uv sync
```

If you cannot use `astral-uv`, produce a pinned requirements file from `uv.lock` (or use your preferred tool) and install with pip as a fallback.

3. Start the server (default port 32550). Replace the model id as needed:

```powershell
uv run model_server.py
```

The `--model` argument points the server at a pretrained model or local path. See "Enable GPT‑OSS-120B" below for a note about changing the default.

## API / Usage
The server exposes both HTTP and WebSocket endpoints. The handlers are implemented in `model_server.py`.

Endpoints
- POST `/stream` and `/api/v1/stream`
  - Used for uploading audio/image/message batches for processing. Accepts JSON messages which can include `input_audio` (base64 wav), `image` (base64), and control / metadata.
- WebSocket `/ws/stream` and `/ws/api/v1/stream`
  - Real‑time bi‑directional transport for audio frames and control messages.
- POST `/completions` and `/api/v1/completions`
  - Returns an SSE stream (text/event-stream) produced by `generate_sse_response` and `StreamManager.generate`.
- POST `/stop` and `/api/v1/stop`
  - Signal server to stop generation for the current session.
- POST `/feedback` and `/api/v1/feedback`
  - Submit feedback; stored to log/feedback files in the server log directory.
- POST `/init_options` and `/api/v1/init_options`
  - Upload custom audio or options used for system initialization.
- GET `/health` and `/api/v1/health`
  - Health check endpoint.

Example: start a streaming completion (SSE) via curl (PowerShell):

```powershell
# Example single request (replace JSON body with your content)
curl -H "uid: user-123" -X POST "http://localhost:32550/completions" -H "Content-Type: application/json" -d '{"messages":[{"role":"user","content":[{"type":"input_audio","input_audio":{"data":"<base64>","format":"wav"}}]}]}'
```

Responses for streaming endpoints are produced by the `StreamManager` and usually contain incremental text tokens and base64 audio chunks.

## Internals & important symbols
- `StreamManager` (in `model_server.py`) — central session manager responsible for
  - `prefill`: buffering incoming audio/image, running VAD, and pre‑conditioning model context.
  - `generate`: driving the streaming generation loop and yielding streaming events.
  - `upload_customized_audio` and `update_customized_options`: accept and store custom prompt audio and options.
- `vad_utils.py` — contains
  - `VadOptions`: configuration for Silero VAD thresholds and durations.
  - `get_speech_timestamps`, `run_vad`: helpers to split long audio into speech chunks.
  - `SileroVADModel`: wrapper to run the ONNX VAD model.
- `silero_vad.onnx` — the ONNX model artifact used by `SileroVADModel` (ensure it is present alongside the project if you use VAD).

## Customization
- Change the runtime model by passing `--model` when starting `model_server.py`.
- Tweak VAD behavior by editing default values in `VadOptions` in `vad_utils.py` (threshold, min/max durations, padding).
- Add or change logging/file paths inside `StreamManager.sys_prompt_init` to fit your retention and storage policies.

## Enable "GPT‑OSS-120B for all clients"
EchoSight reads the model to serve from the `--model` command line argument in `model_server.py`. To use a different model (for example `openai/gpt-oss-120b`) as the canonical server model for all clients, do one of the following:

1. Start the server with the desired model id (simple, server‑wide):

```powershell
uv run python .\model_server.py --model openai/gpt-oss-120b
```

2. If you need a runtime feature flag (so the model can be switched without restarting):
  - Add an environment variable or config value (for example, `MODEL_ID`) and read it during startup in `model_server.py`.
  - Use that value when constructing `StreamManager` and the underlying model loader.

3. If your target model is not a Hugging Face style model or uses a different API (OpenAI-hosted), update or adapt the model-loading and streaming code in `StreamManager` to call the appropriate SDK or client (e.g., OpenAI streaming API). The current server expects a model object exposing the methods used in `StreamManager` (such as `streaming_prefill` / `streaming_generate` or your project's equivalent).

Notes on compatibility: ensure the chosen model supports the streaming/prefill API patterns used by `StreamManager`, or adapt the manager to your model's APIs.

## Troubleshooting
- Missing `uid` header: many endpoints expect a `uid` header. Add `-H "uid: <some-id>"` to requests.
- ONNX VAD issues: install `onnxruntime` and verify `silero_vad.onnx` is present.
- Model load failures: check that `--model` points at a compatible model and that required model files and dependencies are available.

## Security & privacy
- Audio frames, logs, and uploaded customized audio may be written to disk by default. Review and secure the `log_data` directory in `StreamManager` and ensure proper access controls.
- If exposing the server publicly, gate it behind a reverse proxy and add authentication and rate limiting.

## Contributing
- This project is intended as a starting point for building realtime, multimodal assistant experiences. Improvements can include:
  - A stronger auth layer for client connections.
  - A pluggable model adapter interface to support different streaming model backends.
  - A small web UI client for camera/mic streaming and playback.

## License & Notes
- This README describes code present in `model_server.py` and `vad_utils.py` in this repository. Review third‑party model licensing (Hugging Face models, ONNX artifacts, and TTS assets) before production use.
