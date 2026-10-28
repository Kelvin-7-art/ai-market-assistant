# Deploy Market AI and Vision Pipeline on Render

Status: prepared source bundle, NOT deployed. No paid resource has been created.

## What is included

Your supplied source projects remain modular. Render builds Dockerfile.market
and Dockerfile.vision from the repository root. render.yaml creates:

- market-ai: public Streamlit web service, Frankfurt, 2c-4g (4 GB).
- market-vision-api: private API service, Frankfurt, 2c-8g (8 GB), 20 GB disk.

These are PAID compute plans plus paid disk storage and any Groq usage.
Review Render's displayed total before applying the Blueprint. Memory needs
have not been measured; 8 GB does not guarantee the full analyzer stack fits.
Do not select free instances expecting this complete ML workload to fit.
https://render.com/pricing
https://render.com/docs/compute-plans

## Deploy steps

1. Extract this ZIP. Create a PRIVATE GitHub repository. Upload the contents
   so render.yaml, both Dockerfiles, market_ai/ and vision_pipeline/ are at
   the repository root. Do not upload the ZIP itself or an enclosing folder.
2. In Render, choose New > Blueprint, connect that repository and select
   render.yaml. Keep both services in the same region/workspace.
3. Fill the secret fields listed below. Review plans and charges before
   creating services. Use newly rotated keys; never reuse exposed keys.
4. Apply the Blueprint. Inspect build logs for BOTH services. The Docker
   builds install CPU PyTorch and run pip check. Dependency resolution is
   not locked: this bundle still needs its first full Linux build validation.
5. Open the market-ai onrender.com URL and enter APP_PASSWORD. The backend
   is intentionally private; it does not have a public browser URL.
6. Open Chat and re-check pipeline. Upload ONE small chart. Expect initial
   downloads/model loading to be slow. Check backend logs and RAM graphs.
   Test follow-ups, JSON chat and one model run before wider use.
7. After successful builds and smoke tests, pin resolved package versions
   for reproducibility. Avoid changing all model libraries together casually.

## Environment values

Backend market-vision-api:
- GROQ_API_KEY: your actual rotated Groq key.
- GROQ_VISION_MODEL: qwen/qwen3.8-27b (verify access in your Groq account).

Frontend market-ai:
- GROQ_API_KEY: Groq key for image-chat explanations.
- GROQ_PRIMARY_API_KEY: primary Chat AI key; can be the same as above.
- GROQ_SECONDARY_API_KEY: secondary key; can use the same key initially.
- GROQ_MODEL: qwen/qwen3.8-27b.
- APP_PASSWORD: a long, unique access password; do not leave it empty.
- LLM_PROVIDER: groq_primary (already set by Blueprint).
- VISION_PIPELINE_HOSTPORT: automatically injected from the private backend.
- VISION_PIPELINE_TIMEOUT: 600 seconds (already set).

Optional: set HF_TOKEN on the backend if needed for downloads; set OpenAI
or Anthropic keys on the frontend only if you select those providers.
Do NOT set VISION_PIPELINE_URL to localhost on Render. It overrides the
injected address. Leave it unset when using this Blueprint.

## Changes from the uploaded projects

- Added deployment-only requirements-render.txt files; kept original
  requirements files for comparison. Added dependencies imported by the code.
- Added Linux libraries for OpenCV, barcode readers and LightGBM.
- CPU torch/torchvision are installed from the PyTorch CPU wheel index.
- Frontend now supports env URL/private hostport and configurable timeout.
- /health no longer loads models. It reports liveness only, NOT model readiness.
- Analyzer imports and construction happen on first analysis. Full /analyze
  inference runs in a thread, serialized per process, keeping health responsive.
- Kept one backend worker to avoid duplicate model stacks.
- Chart model configurable via GROQ_VISION_MODEL; bounded Groq timeout/retries.
- Failed image analyses are no longer reused from session cache.
- Public Streamlit access is password-gated when APP_PASSWORD is set.
- Removed developer caches, environments, model weights and secrets files
  from the deploy bundle. Runtime Hugging Face/Torch/EasyOCR caches use /var/data.
  Other libraries may download to their own locations; not every model cache
  is guaranteed persistent. Never set HF_HUB_OFFLINE on a fresh deployment.

## Limits and troubleshooting

The entire analyzer stack remains enabled: it can be slow and memory-heavy.
A healthy HTTP process does not prove all models loaded. 600-second client
wait does not eliminate Render/network timeouts or CPU constraints. Use low
concurrency. For a public multi-user production workload, add a durable job
queue, per-user accounts, quotas and per-analyzer resource controls first.
The shared-password gate is a small private-demo gate, not full account auth.

If builds fail, save the first dependency error and surrounding lines.
If runtime exits, inspect memory metrics and exit code before resizing.
If /health works but /analyze fails, inspect model download/analyzer logs.
Repeated retries can queue work: do not repeatedly submit a timed-out image.
The /compare and /analyze/region endpoints retain their original implementation
and require separate workload validation; no readiness guarantee is claimed.

Validation performed: Python syntax, Blueprint YAML and Dockerfile paths;
see VALIDATION.txt for backend smoke-test status. Full Docker builds,
model downloads/inference, browser workflows and paid deployment not tested.

## Optional local Docker build checks

From this folder with Docker running:

    docker build -f Dockerfile.market -t market-ai .
    docker build -f Dockerfile.vision -t market-vision .

No keys are needed at build time. Do not bake keys into either image.

Official references:
https://render.com/docs/blueprint-spec
https://render.com/docs/private-network
https://render.com/docs/health-checks
https://console.groq.com/docs/vision
