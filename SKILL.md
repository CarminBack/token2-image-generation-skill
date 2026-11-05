---
name: token2-image-generation
description: Generate images with the OpenAI-compatible gpt-image-2.5 endpoint at token2-api.mewinyou.shop. Use when the user asks for image generation or image editing through this provider, or when configuring Codex to use it.
compatibility: Requires network access and a TOKEN2_API_KEY environment variable. The provider must expose an OpenAI-compatible /v1/images endpoint.
---

# Token2 image generation

Use the user's Token2-compatible image service with these fixed defaults:

- API base: `https://token2-api.mewinyou.shop/v1`
- Model: `gpt-image-2.5`
- Generation endpoint: `POST https://token2-api.mewinyou.shop/v1/images/generations`
- Edit endpoint: `POST https://token2-api.mewinyou.shop/v1/images/edits`

## Configure credentials

Before making a paid or authenticated request, check whether `TOKEN2_API_KEY` is already present and non-empty. Never print, echo, log, commit, or put the key in a prompt, source file, skill file, URL, or generated report.

If it is missing, stop and ask the user to configure it locally through their shell or secret manager, then retry. Give this example without asking them to paste the secret into chat:

```sh
read -rsp 'Token2 API key: ' TOKEN2_API_KEY; export TOKEN2_API_KEY; echo
```

For a persistent local setup, recommend the user's OS secret manager or a local shell profile with restrictive permissions. Do not write `.env` files unless the user explicitly requests that and the file is confirmed ignored and mode `0600`.

## Request shape

Use the OpenAI Images API shape. Keep `n=1` unless the user asks for more, and pass the user's prompt verbatim except for necessary API validation. For a basic request:

```sh
curl --fail-with-body --silent --show-error \
  --connect-timeout 10 --max-time 110 \
  -H "Authorization: Bearer ${TOKEN2_API_KEY}" \
  -H 'Content-Type: application/json' \
  https://token2-api.mewinyou.shop/v1/images/generations \
  -d '{"model":"gpt-image-2.5","prompt":"<user prompt>","n":1}'
```

Treat the response as OpenAI-compatible JSON. Prefer `data[].b64_json`; if the service returns `data[].url`, download only an HTTPS URL and do not forward the authorization header to that image host. Validate the decoded bytes as PNG, JPEG, or WebP before presenting or saving them.

For an edit, use multipart form data with `model=gpt-image-2.5`, `prompt`, and the image file. Let `curl` create the multipart boundary; do not set `Content-Type: multipart/form-data` manually.

## Cloudflare's 120-second timeout

A client-side timeout cannot remove Cloudflare's request limit. Do not claim that a larger `--max-time`, a keep-alive header, chunked encoding, or a fake progress heartbeat bypasses it. A response that remains open past the proxy limit will still be terminated.

When an image request can take close to two minutes, use this decision order:

1. Check the provider's documented asynchronous job API. Submit the job, return immediately with its job ID, then poll a status endpoint with bounded backoff and download the result after completion. Preserve the job ID so an interrupted client can resume. Do not invent status paths; verify them from the provider's API response or documentation.
2. If the provider has no async API, route the long-running image endpoint around the Cloudflare proxy to a protected origin hostname or private network path. Keep the public hostname for short control requests and downloads. Restrict the direct origin with firewall rules, authentication, and TLS; do not expose an unprotected admin or origin port.
3. If the hostname must stay proxied, move the long-running work behind a queue/worker and have the HTTP request return a short-lived task ID. Poll or use a callback/webhook. Configure Cloudflare and the origin for the short submit/status/download requests.
4. Cloudflare Enterprise options may change some timeout limits, but they still do not turn a synchronous long-running image request into a reliable API. Confirm the account's actual limits before relying on them.

If the endpoint only exposes synchronous `/v1/images/generations` and no direct origin or async facility is available, report that the 120-second cutoff cannot be solved from Codex alone and return the request failure without blindly retrying a potentially duplicated paid generation.

## Failure handling

- `401` or `403`: report authentication or access failure without exposing the key.
- `404`: verify whether the service expects `/v1` and whether the model is listed by `/v1/models`; do not silently switch providers.
- `429`: respect `Retry-After` when present and retry only when the operation is known to be idempotent.
- `5xx`, connection reset, or Cloudflare `524`: preserve the request/job context, avoid blind duplicate submissions, and use the async/direct-origin path above.
- Do not send the secret to any diagnostic URL or include it in telemetry.
