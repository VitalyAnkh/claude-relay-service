# OpenAI Relay – Sora 2 Video Endpoint (Beta)

This document describes the new `/openai/v1/videos` endpoint that adapts ChatGPT Sora 2 usage quotas into the relay API. The goal is to let you script against the same internal video workflow that the ChatGPT web client uses, so you can drive Sora 2 renders from automation while still consuming your ChatGPT subscription minutes.

> ⚠️ **Important:** As of 8 October 2025 OpenAI has not released a public Sora 2 developer API. All automation relies on the ChatGPT consumer backend (`https://chatgpt.com/backend-api/videos`). This behaviour is subject to change without notice and may violate OpenAI’s consumer terms. Proceed only if you accept that risk.

## Endpoint summary

- Base path: `POST /openai/v1/videos` (alias: `POST /openai/videos`)
- Requires an API key with `openai` or `all` permissions.
- Supports two upstream account types:
  - **ChatGPT access-token accounts (`openai`)** – forwards to `chatgpt.com/backend-api/videos` using the stored access token, mirroring ChatGPT web requests.
  - **Official OpenAI Responses accounts (`openai-responses`)** – forwards to `https://api.openai.com/v1/videos` (or your custom `baseApi`) with the stored API key.
- The relay treats the body as an opaque JSON payload. You must supply the same shape that ChatGPT expects (prompt, duration, aspect ratio, etc.).

## Request workflow for ChatGPT subscriptions

1. Capture the JSON payload that ChatGPT sends when you manually start a Sora 2 render. The payload typically includes:
   - `prompt` – natural-language description of the scene.
   - `preset` or `quality` – e.g. `"standard"`.
   - `duration_seconds` – commonly 5, 10, or 20 seconds.
   - `aspect_ratio` – such as `"16:9"` or `"9:16"`.
   - Optional control parameters (camera hints, motion cues, seeds, etc.).
2. Call the relay endpoint with the captured payload:

```bash
curl https://your-relay.example.com/openai/v1/videos \
  -H "Authorization: Bearer <relay-api-key>" \
  -H "Content-Type: application/json" \
  -d '{
        "prompt": "A drone shot of surfers riding sunrise waves",
        "aspect_ratio": "16:9",
        "duration_seconds": 10,
        "quality": "standard"
      }'
```

3. The relay returns the upstream JSON response. For ChatGPT accounts this usually contains a job identifier you can poll through ChatGPT’s media status endpoints (not yet proxied by this service).

## Response handling

- **200–202:** Successful request. The body is the upstream JSON (job metadata, status, etc.).
- **429:** The selected account hit a Sora quota. The scheduler marks the account as rate limited using the `resets_in_seconds` hint (when present) and removes the sticky session mapping so another account can be chosen next time.
- **401 / 402:** Authentication or billing failure. The scheduler flags the account as unauthorised and evicts existing sticky sessions.
- **Other 4xx/5xx:** The relay returns the upstream error payload and logs the failure for troubleshooting.

## Sticky sessions and headers

- If you supply `session_id` in the request body or the `Session-Id`/`X-Session-Id` headers, the relay preserves sticky routing so assets for one project stay on the same account when possible.
- Headers such as `user-agent`, `oai-device-id`, and `oai-latency-trace-id` are forwarded when present, mirroring the ChatGPT browser behaviour.

## Limitations & caveats

- **No job polling proxy yet.** You must still use your own tooling (or future relay endpoints) to poll `chatgpt.com/backend-api/videos/<job_id>` or download finished assets.
- **Upstream schema can change.** Because ChatGPT’s Sora payload is undocumented, OpenAI may change field names or validation. Keep an eye on upstream changes and update your payload accordingly.
- **Official API support pending.** When OpenAI introduces a public `v1/videos` API, you can bind `openai-responses` accounts to the relay and the same route will forward to the official endpoint instead of the ChatGPT backend.
- **Terms of service.** Automating ChatGPT consumer endpoints may breach OpenAI’s user agreement. Evaluate and accept the compliance risk before enabling this feature.

## References

- Early community reverse-engineering reports confirming the `/backend-api/videos` workflow for Sora 2 (Trends Daily AI, September 2025).
- Public coverage noting that Sora 2 remains a ChatGPT-exclusive experience without an official developer API as of late September 2025 (Dev.to community analysis).
