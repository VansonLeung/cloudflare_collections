# Insights: Free Image Generation API

- Reviewed: 2026-10-29.
- Reference revision: `3d20626`.
- Scope: source review and local checks with a mocked AI binding.
- Assessment: useful learning example. Needs fixes before public use.

## 1. Project overview

| Item | Finding |
| --- | --- |
| Purpose | Generate an image from a text prompt. |
| Runtime | Cloudflare Workers. |
| Inference | Cloudflare Workers AI through `env.AI.run()`. |
| Active model | `@cf/stabilityai/stable-diffusion-xl-base-1.0`. |
| Authentication | One shared bearer key from `env.API_KEY`. |
| Interface | One `POST /` endpoint. |
| Frontend | One standalone HTML file. |
| Dependencies | No package manifest or external frontend library. |
| Persistence | None. Images stay in the browser until downloaded or discarded. |
| Deployment | Manual dashboard setup in the README. |

## 2. Files studied

| File | Role | Key observation |
| --- | --- | --- |
| [README.md](../../references/free-image-generation-api/README.md) | Setup and usage. | Overstates the free AI request allowance. |
| [worker.js](../../references/free-image-generation-api/worker.js) | Authentication, routing, inference, and responses. | Small enough to understand end to end. |
| [frontend_demo.html](../../references/free-image-generation-api/frontend_demo.html) | Prompt input, gallery, and downloads. | Needs a real API URL and browser integration fixes. |

- No Wrangler configuration is included.
- No automated tests or CI configuration are included.
- The README states MIT. No standalone license file is included.

## 3. Request flow

1. The user enters a prompt and an API key.
2. The browser sends JSON with a bearer header.
3. The Worker checks the key first.
4. The Worker accepts only `POST /`.
5. The Worker reads `prompt` from the JSON body.
6. Workers AI runs the fixed model.
7. The Worker returns the model result as an image response.
8. The browser creates a blob URL and a download card.

- Each accepted request makes one AI call.
- The Worker forwards only `prompt`.
- Model choice, dimensions, seed, and step count are not exposed.
- The gallery has no server history or reload recovery.

## 4. Current API behavior

| Request or condition | Status | Response or effect |
| --- | --- | --- |
| Valid key and prompt at `POST /` | `200` | Model output. Header says `image/jpeg`. |
| Missing or wrong bearer key | `401` | JSON: `Unauthorized`. |
| Valid key with another method or path | `405` | JSON: `Not allowed`. |
| Missing or empty prompt | `400` | JSON: `Prompt is required`. |
| Whitespace or a truthy non-string prompt | Model-dependent | Passes local validation and reaches AI. |
| Invalid JSON or JSON `null` | `500` | Generic failure plus exception details. |
| Missing AI binding or thrown inference error | `500` | Generic failure plus exception details. |
| Browser preflight without authorization | `401` | No CORS headers. Cross-origin use fails. |

- Authentication runs before route checks.
- Unknown paths return `405`, rather than `404`.
- JSON content type is not enforced by the Worker.
- The output header is hardcoded. Actual image encoding was not verified.

## 5. Platform claims checked

- Platform details below were checked on 2026-10-29.

| Reference claim | Verified detail | Implication |
| --- | --- | --- |
| 100,000 free AI calls per day. | Workers Free includes 100,000 Worker requests per day. [Workers pricing](https://developers.cloudflare.com/workers/platform/pricing/). | HTTP request allowance does not guarantee this many images. |
| Workers AI has the same request allowance. | General AI allocation is 10,000 neurons per day. Paid overage is $0.011 per 1,000 neurons. [AI pricing](https://developers.cloudflare.com/workers-ai/platform/pricing/). | Budget AI usage separately. |
| The active model supports free generation. | Its page lists beta status and $0.00 per step. [SDXL model](https://developers.cloudflare.com/workers-ai/models/stable-diffusion-xl-base-1.0/). | This listing still does not guarantee 100,000 successful generations per day. |
| Models can be changed in the code. | FLUX uses `@cf/black-forest-labs/flux-1-schnell`. It returns a base64 `image` field. [FLUX model](https://developers.cloudflare.com/workers-ai/models/flux-1-schnell/). | Fix the commented model ID. Add output conversion when switching. |

- The commented ID `@cf/blackforestlabs/ux-1-schnell` is incorrect.
- Image-to-image and inpainting options need different input handling.
- A model change requires checking its input schema and response format.
- Free-tier capacity depends on the chosen model and platform limits.

## 6. Main gaps and fixes

- P0: fix before exposing the service.
- P1: fix before relying on browser use or regular operation.
- P2: improve repeatability and maintenance.

| Priority | Finding | Evidence | Suggested fix |
| --- | --- | --- | --- |
| P0 | Missing API key can bypass the intended protection. | `worker.js`, lines 3–9. An unset key matches `Bearer undefined`. | Reject requests if the configured key is missing or empty. Store it as a Worker secret. |
| P0 | Prompt content enters `innerHTML`. | `frontend_demo.html`, lines 154–158. HTML can execute after a successful response. | Create DOM nodes. Set prompt text with `textContent`. |
| P1 | Cross-origin browser requests fail. | No `OPTIONS` handler or CORS response headers. | Serve UI and API on one origin, or support an explicit origin allowlist and preflight. |
| P1 | Validation accepts unsuitable prompts. | `worker.js`, line 20 checks only truthiness. | Require a string. Trim it. Enforce prompt and body size limits. |
| P1 | Bad JSON becomes a server error. | Parsing and inference share one catch block. | Return `400` for malformed JSON. Use separate upstream error handling. |
| P1 | Error responses expose exception messages. | `worker.js`, line 40 returns `err.message`. | Return a stable error code. Log diagnostic details privately. |
| P1 | One key controls all usage. No rate limit exists. | One bearer comparison. No quota checks. | Keep shared credentials with trusted callers. Add caller limits and usage caps. |
| P1 | Image type and model output are assumed. | `worker.js`, lines 31–38. | Adapt each supported model output. Set the matching content type and extension. |
| P1 | Repeated clicks can start parallel generations. | Generate button stays enabled. | Disable it while pending. Add a bounded request timeout. |
| P2 | Blob URLs accumulate. | `frontend_demo.html`, line 149 creates URLs without revocation. | Revoke URLs when cards are removed. Bound gallery size. |
| P2 | Browser errors lose useful context. | Every non-success response becomes `API Error`. | Read safe API error codes. Preserve the prompt after failure. |
| P2 | Deployment depends on dashboard steps. | No checked-in deployment configuration. | Add Wrangler configuration, an AI binding, and documented secret setup. |

## 7. Lessons for this collection

| Keep | Improve when adapting |
| --- | --- |
| Small Worker entry point. | Separate configuration, validation, and inference errors. |
| Direct Workers AI binding. | Use a model adapter for output differences. |
| Binary image response. | Verify image encoding and match response headers. |
| Simple prompt-to-gallery workflow. | Handle pending, success, failure, and cleanup states. |
| Minimal infrastructure. | Add storage only when persistent history is needed. |

- Start with one verified text-to-image model.
- Keep prompt validation on the server.
- Use same-origin hosting for the first browser integration.
- Use caller authentication suitable for the intended audience.
- Track generation counts, latency, failures, and AI usage.
- Add R2 only if saved images or shareable links are required.
- Add D1 only if searchable generation metadata is required.
- Add a queue only if background jobs or retries are required.

## 8. Verification and limits

- Ran 12 local checks against the unchanged Worker.
- Used Node's Request and Response APIs.
- Mocked `env.AI.run()`. No Cloudflare inference was performed.
- All checks confirmed the behavior listed below.

| Check | Observed result |
| --- | --- |
| Valid prompt | `200`. AI called once. |
| Missing authorization | `401`. AI not called. |
| Wrong path with valid key | `405`. AI not called. |
| GET with valid key | `405`. AI not called. |
| Unauthenticated OPTIONS preflight | `401`. No CORS allow-origin header. |
| Missing prompt | `400`. AI not called. |
| Whitespace prompt | Reached AI. Mock returned `200`. |
| Numeric prompt | Reached AI. Mock returned `200`. |
| Malformed JSON | `500`. AI not called. |
| JSON `null` | `500`. AI not called. |
| Unset key with `Bearer undefined` | Authentication passed. AI called. |
| Missing AI binding | `500`. |

- Frontend findings come from static inspection.
- Browser execution and the HTML injection were not tested live.
- Live image quality, format, latency, and capacity remain unverified.
- Model availability and account-specific limits need a deployment smoke test.
