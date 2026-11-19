# Signature HQ Python SDK

Official Python client for the [Signature HQ](https://signaturehq.com.au) e-signature API v1.

- Python 3.9+, fully typed (`py.typed`), sync and async clients
- One runtime dependency: [httpx](https://www.python-httpx.org/)
- Safe retries with automatic idempotency keys, auto-paging, streamed downloads, webhook verification

## Install

Not yet published to PyPI. Once it is:

```bash
pip install signaturehq
```

## Quickstart

```python
import os
from signaturehq import SignatureHQ

shq = SignatureHQ(os.environ["SHQ_API_KEY"])  # "shq_..." live, "shq_test_..." test mode

envelope = shq.envelopes.create(title="Lease agreement", message="Please sign by Friday.")
document = shq.documents.upload(envelope.id, "lease.pdf")
signer = shq.recipients.add(envelope.id, name="Jane Citizen", email="jane@example.com", role="signer")

shq.fields.set(envelope.id, [
    {
        "document": document.id,
        "recipient": signer.id,
        "type": "signature",
        "page": 1,
        "x": 10, "y": 80, "width": 30, "height": 6,  # percentages of the page
        "required": True,
    },
])

shq.envelopes.send(envelope.id)

# Later (or when the envelope.sealed webhook arrives):
if shq.envelopes.retrieve(envelope.id).sealed:
    shq.envelopes.save_sealed_document(envelope.id, "lease-signed.pdf")
```

Use the client as a context manager (`with SignatureHQ(...) as shq:`) or call `shq.close()` when you are done.

### Async

`AsyncSignatureHQ` has the same methods (asyncio):

```python
import asyncio
import os

from signaturehq import AsyncSignatureHQ

async def main() -> None:
    async with AsyncSignatureHQ(os.environ["SHQ_API_KEY"]) as shq:
        envelope = await shq.envelopes.create(title="Lease agreement")
        async for item in shq.envelopes.all(status="sent"):
            print(item.id, item.status)

asyncio.run(main())
```

### Every endpoint

| Method                                                                                                          | Endpoint                                         |
| --------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| `envelopes.list(status=, per_page=, page=)`, `envelopes.all(...)`                                               | `GET /envelopes`                                 |
| `envelopes.create(title=, message=, signing_order=, expires_on=, reminder_interval_days=, append_certificate=)` | `POST /envelopes`                                |
| `envelopes.retrieve(id)`                                                                                        | `GET /envelopes/{id}`                            |
| `envelopes.update(id, ...)`                                                                                     | `PATCH /envelopes/{id}` (drafts)                 |
| `envelopes.delete(id)`                                                                                          | `DELETE /envelopes/{id}` (drafts)                |
| `envelopes.send(id)`                                                                                            | `POST /envelopes/{id}/send`                      |
| `envelopes.void(id, reason)`                                                                                    | `POST /envelopes/{id}/void`                      |
| `envelopes.download_sealed_document(id)`, `envelopes.save_sealed_document(id, path)`                            | `GET /envelopes/{id}/sealed-document`            |
| `documents.upload(envelope_id, file, filename=)`                                                                | `POST /envelopes/{id}/documents`                 |
| `recipients.add(envelope_id, name=, email=, role=, verification_method=, phone=)`                               | `POST /envelopes/{id}/recipients`                |
| `recipients.update(envelope_id, recipient_id, ...)`, `recipients.remove(...)`                                   | `PATCH`/`DELETE /envelopes/{id}/recipients/{id}` |
| `fields.set(envelope_id, fields)`                                                                               | `PUT /envelopes/{id}/fields`                     |
| `webhooks.list()`, `create(url=, events=)`, `retrieve(id)`, `update(id, ...)`, `delete(id)`                     | `/webhooks`                                      |
| `webhooks.rotate_secret(id)`, `webhooks.ping(id)`                                                               | `POST /webhooks/{id}/rotate-secret`, `/ping`     |
| `webhooks.deliveries(id)`, `webhooks.all_deliveries(id)`                                                        | `GET /webhooks/{id}/deliveries`                  |

Every method also takes `timeout=` (seconds) to override the client's timeout for that call.

Optional parameters you leave out are not sent. On `update`, `None` clears a value: `message=None`, `expires_on=None`
(no expiry), `reminder_interval_days=None` (no automatic reminders). `expires_on` is a `datetime.date` or
`"YYYY-MM-DD"` (end of that day, Australian Eastern time, at most 179 days ahead).

Recipients default to `verification_method="email_link"`. `email_otp` emails a one-time code; `sms_otp` texts one and
needs `phone` (E.164, or an Australian number such as `"0412 345 678"`).

Radio and dropdown fields take `options`, the list of labels to choose from. `fields.set` replaces the whole set:
fields you leave out are deleted, so pass `id` to keep an existing one.

`documents.upload` accepts a path (`str` or `pathlib.Path`), bytes, or a binary file object, up to 25 MB. The server
checks that the file name's extension matches the content, so pass `filename=` with bytes.

### Models

Responses are frozen dataclasses (`Envelope`, `Document`, `Recipient`, `Field`, `WebhookEndpoint`, `WebhookDelivery`,
`WebhookEvent`) with the API's field names. Timestamps are timezone-aware `datetime` objects. Every model keeps the full
JSON object in `.raw`, so fields added to the API later are there before the SDK models them.

## Configuration

```python
shq = SignatureHQ(
    api_key,
    base_url="https://app.signaturehq.com.au/api/v1",  # https only (http is accepted for localhost)
    timeout=60.0,         # seconds per attempt
    max_retries=2,        # 0 disables retries
    max_retry_after=60,   # longest Retry-After (seconds) waited for automatically
    app_info="acme-crm/2.1",  # appended to the User-Agent
    debug=print,          # request/response/retry events; Authorization redacted, never bodies
    http_client=None,     # your own httpx.Client (proxies, custom TLS); never closed by the SDK
)
```

`shq.mode` is `"test"` or `"live"` and `shq.is_test_key` is `True` for `shq_test_` keys.

The API key never appears in URLs, exception messages, debug events or `repr()`, and clients refuse to be pickled.
Redirects are never followed, so the key only ever goes to the configured host. Event hooks and custom transports on
an `http_client` you pass in see every request, including the Authorization header: keep them free of logging.

## Errors

Every exception derives from `SignatureHQError`.

| Exception                               | When                                                                       |
| --------------------------------------- | -------------------------------------------------------------------------- |
| `AuthenticationError`                   | 401: missing, invalid, expired or revoked key                              |
| `PermissionDeniedError`                 | 403: the key lacks the ability, or the user the permission                 |
| `NotFoundError`                         | 404: unknown, another organisation's, or of the other mode (test/live)     |
| `ConflictError`                         | 409: wrong state, e.g. the envelope was already sent                       |
| `IdempotencyConflictError`              | 409: the same key is still being processed (retry later with the same key) |
| `ValidationError`                       | 422: `err.field_errors("email")` lists one field's messages                |
| `IdempotencyKeyReusedError`             | 422: the key was used for a different request                              |
| `RateLimitError`                        | 429: `err.retry_after` in seconds                                          |
| `ServerError`                           | 5xx, or a response that is not the API's JSON                              |
| `InvalidRequestError`                   | 400, other 4xx, or an unexpected redirect                                  |
| `APIConnectionError`, `APITimeoutError` | no response (the request may or may not have reached the server)           |
| `InvalidArgumentError`                  | bad input caught before sending (invalid id, key, file too large)          |
| `WebhookSignatureError`                 | webhook verification failed                                                |

`APIError` subclasses carry `status`, `type`, `message`, `errors` and `headers`.

```python
from signaturehq import ConflictError, ValidationError

try:
    shq.recipients.add(envelope_id, name="Jane", email="not-an-email")
except ValidationError as err:
    print(err.field_errors("email"))
except ConflictError:
    print("The envelope was already sent.")
```

## Idempotency and retries

`envelopes.create` and `envelopes.send` always send an `Idempotency-Key`: a random UUID per call, or yours with
`idempotency_key=` (8 to 255 characters: letters, digits, `-`, `_`, `:`, `.`). Retries reuse the key and the exact
same body, so a retried create never makes a second draft.

If retries run out, the error carries the key (`err.idempotency_key`): retry later with that key to finish the same
operation. The server keeps the first response below 500 for 24 hours per key, including 4xx errors, so after you fix
a rejected request, send it with a new key (the default when you do not pass one).

Automatic retries (2 by default, exponential backoff doubling from 0.5 s up to 8 s, with jitter of up to half):

- network errors, timeouts and 5xx: only `GET`, `PUT`, `DELETE`, and create/send (keyed)
- 429: every request, honouring `Retry-After` up to `max_retry_after`
- 409 `idempotency_conflict`: keyed requests

Other requests (`update`, `void`, recipient add/update, uploads, webhook create/update, `rotate_secret`, `ping`) are
never retried once they may have reached the server. A retried `DELETE` that gets 404 after an earlier attempt failed
with a network error, timeout or 5xx is treated as success (the first attempt did the delete).

## Pagination

```python
page = shq.envelopes.list(status="completed", per_page=50)
for envelope in page:
    ...
next_page = page.next_page()  # None on the last page

for envelope in shq.envelopes.all(status="completed"):  # fetches pages as you go
    ...
```

`all()` skips items it already returned, since envelopes created while you page can push older ones onto the next
page.

## Sealed PDFs

```python
with shq.envelopes.download_sealed_document(envelope_id) as pdf:
    for chunk in pdf.iter_bytes():
        sink.write(chunk)

shq.envelopes.save_sealed_document(envelope_id, "signed.pdf")
```

`save_sealed_document` writes to a new temporary file next to the target (mode 0600) and renames it when complete. Both raise
`NotFoundError` until the envelope's `sealed` is true. While the body streams, the timeout applies to each network
read, not to the whole download.

## Webhooks

Verify every delivery against the **raw request body**, before parsing it:

```python
from signaturehq import Webhook, WebhookSignatureError

event = Webhook.construct_event(raw_body, signature_header, secret)  # raises WebhookSignatureError
ok = Webhook.verify_signature(raw_body, signature_header, secret)     # bool
```

It checks the HMAC-SHA256 of `"<t>.<raw body>"` in constant time and refuses timestamps more than 300 seconds away
(`tolerance=` to change). Passing parsed JSON raises `TypeError`. `secret` can be a list: after
`webhooks.rotate_secret` the old secret stops working at once, so accept both while you deploy the new one.

De-duplicate on `event.id` (also the `SHQ-Event-Id` header): retries and several endpoints share it. Payloads hold ids
and statuses only, so fetch details with the API.

Flask:

```python
@app.post("/shq-webhook")
def shq_webhook():
    try:
        event = Webhook.construct_event(request.get_data(), request.headers.get("SHQ-Signature"), SECRET)
    except WebhookSignatureError:
        abort(400)
    queue_for_processing(event)
    return "", 204
```

FastAPI:

```python
@app.post("/shq-webhook", status_code=204)
async def shq_webhook(request: Request) -> None:
    body = await request.body()
    try:
        event = Webhook.construct_event(body, request.headers.get("SHQ-Signature"), SECRET)
    except WebhookSignatureError:
        raise HTTPException(status_code=400)
    queue_for_processing(event)
```

Django:

```python
@csrf_exempt
@require_POST
def shq_webhook(request):
    try:
        event = Webhook.construct_event(request.body, request.headers.get("SHQ-Signature"), SECRET)
    except WebhookSignatureError:
        return HttpResponse(status=400)
    queue_for_processing(event)
    return HttpResponse(status=204)
```

Respond with a 2xx within 10 seconds and do the work asynchronously. `shq.webhooks.ping(endpoint_id)` sends a signed
`ping` test event (`event.type == "ping"`) to check your receiver; it shows in the delivery log.
`Webhook.signature_header(body, secret)` builds a valid header for your own tests.

## Test mode

Create a **Test** token in Settings > API tokens (it starts with `shq_test_`). A test key only sees and creates test
envelopes and test webhook endpoints (`livemode` is `False`); anything of the other mode is a `NotFoundError`.

Test envelopes run the real flow: **invitation emails are really sent**, recipients really sign, and the sealed PDF is
real but stamped "TEST - NOT LEGALLY BINDING". Use addresses you control. An organisation can send at most 50 test
envelopes, reaching at most 100 recipients, per rolling 24 hours.

## Development

```bash
python3 -m venv .venv && .venv/bin/pip install -e '.[dev]'
.venv/bin/pytest
.venv/bin/ruff check src tests integration && .venv/bin/ruff format --check src tests integration
.venv/bin/mypy
```

`integration/` runs against a real server; from the repository root inside DDEV:
`E2E_PORT=8035 E2E_DB=e2e_p node sdks/integration/run.mjs python`.
