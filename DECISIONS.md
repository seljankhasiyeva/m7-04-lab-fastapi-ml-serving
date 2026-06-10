# API Design Decisions

## Versioning

We chose path-based versioning (`/v1/...`) over header-based versioning because it is visible and cacheable: partner engineers can see the version in logs, gateways, and CDN keys without inspecting headers, which simplifies routing and debugging across multiple major versions. It also makes breaking changes explicit and easy to run side-by-side (e.g. `/v1/predict` and `/v2/predict` on the same server) without conditional logic based on header values.

## Batch ordering and partial failures

When `predict-batch` is called with 32 items and one image is corrupt, the response returns HTTP 200 with a `results` array containing 32 entries, each keyed by the caller-supplied `id` rather than relying on array position. The corrupt item gets `status: "error"` with a populated `error` object (code `invalid_image_format`), while the other 31 items get `status: "ok"` with their `labels`. We chose this "always 200, per-item status" approach because batch processing is independent per image, and a single bad input shouldn't force the caller to retry 31 successful classifications or parse a top-level error to find which item failed.

## Async lifecycle

A job moves through `queued → running → done` on success, or `queued → running → failed` if processing errors out (e.g. corrupt image, model error). `GET /v1/predictions/{job_id}` returns the current status at any point, with `labels` and `model_version` populated only when `status` is `done`, and `error` populated only when `status` is `failed`. Results (and the job record itself) are retained for 24 hours after reaching a terminal state (`done` or `failed`); after that, `GET` on the `job_id` returns `404` with the shared `Error` schema.