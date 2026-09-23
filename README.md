# ad-maker

Backend skeleton for a mobile-first product-ad video maker: a customer uploads a product photo and a
one-line description, pays, and gets a short vertical video plus a caption. Everything external
(video generation, payments, storage, database) sits behind an interface, so the flow runs today
against mocks and costs nothing.

## Run it

```bash
npm install
npm test               # 52 tests
npm run typecheck
cp .env.example .env   # then export the vars, or set them inline
npm run dev
```

Try the whole flow (uses the mock providers; the "video" is a placeholder file, not playable):

```bash
# 1. upload a photo -> { imageKey }
curl -X POST localhost:3000/uploads -H 'content-type: image/png' --data-binary @photo.png

# 2. create an order -> price + status "awaiting_payment"
curl -X POST localhost:3000/orders -H 'content-type: application/json' -d '{
  "businessName":"Mama Njeri Fresh Produce",
  "description":"Fresh sukuma wiki and tomatoes delivered daily",
  "language":"sw","style":"clean","packageId":"basic",
  "phone":"0712345678","imageKey":"<imageKey from step 1>"}'

# 3. dev only: pretend the customer paid
curl -X POST localhost:3000/dev/orders/<id>/pay

# 4. poll until status is "ready", then open downloadUrl
curl localhost:3000/orders/<id>
```

## How it fits together

| Piece | Where | Real implementation still to write |
|---|---|---|
| Video provider (`estimate`, `submit`, `poll`, `download`) | `src/providers/video` | Higgsfield adapter written; not yet run against the real API |
| Payment provider (`initiate`) + `OrderService.confirmPayment` | `src/providers/payment`, `src/services/orders.ts` | M-Pesa Daraja STK push + authenticated callback |
| Storage | `src/storage` | S3/R2/GCS (local disk now) |
| Order repo | `src/db` | Postgres (`schema.sql` is ready; in-memory now) |
| Job queue + worker | `src/jobs` | Fine for one process; use a real queue for several |

Order states: `awaiting_payment -> paid -> generating -> ready | failed`.

Design choices worth knowing:

- **Pay first.** Nothing is generated until payment is confirmed.
- **Margin guardrail** (`services/pricing.ts`). Before taking money, and again before each job, the
  provider's cost estimate must fit within `MAX_COST_SHARE` of the price. Otherwise the tier is
  downgraded (never upgraded) or the order is refused. At run time, refusal means fail + refund.
- **Idempotent payment confirmation.** Callbacks are retried in the real world. `confirmPayment`
  uses a compare-and-set on status, so duplicates start exactly one job. It also checks the amount.
- **No double spend on restart.** The provider job id is saved right after submit; a restart resumes
  polling instead of submitting again (`recoverPending`).
- **Failed after payment => `refundDue`.** The customer is told; you refund manually for now.
- **Own copy of every video.** Provider files expire, so results are copied into your storage.
- **Public responses are minimal.** No phone number, payment reference, cost or provider info.

## Higgsfield adapter

`src/providers/video/higgsfield.ts` follows Higgsfield's documented flow: presigned image upload,
POST to the model endpoint, poll `/requests/{id}/status`, download the result. Turn it on with
`VIDEO_PROVIDER=higgsfield` plus `HIGGSFIELD_API_KEY_ID`, `HIGGSFIELD_API_KEY_SECRET` and
`HIGGSFIELD_TIERS` (see `.env.example`). There are deliberately no built-in prices: you must supply
`usdPerSecond` for each tier from Higgsfield's pricing page, and the margin guardrail uses them.

How it protects your balance:

- Higgsfield submissions have no idempotency key, so a POST is **never repeated** after a timeout,
  dropped connection or 5xx. Those orders fail with `refundDue` and are logged for reconciliation.
  Only definite refusals that start no job (400, 423, 503) are retried, up to `MAX_SUBMIT_ATTEMPTS`.
- Status checks, upload-URL creation and uploads are safe to repeat and are retried with backoff.
  A status outage never refunds a job that may still finish; the worker keeps polling until its deadline.
- Failed and NSFW jobs are documented as uncharged, so a failed job may be resubmitted; a moderation
  rejection is shown to the customer as `content_rejected`.
- An empty Higgsfield balance or bad credentials show the customer only `service_unavailable`
  and are logged at error level for you.
- Credentials go only to `api.higgsfield.ai`, never to the presigned upload host or the result CDN.

What is and is not verified:

- Tested against an in-memory fake built from Higgsfield's docs (19 tests). It has **not** been run
  against the real API.
- I could not retrieve the Seedance 2.5 image-to-video schema page. The request body
  (`image_url`, `prompt`, `duration`, `resolution`, `generate_audio`, `output_format`) is taken from
  the documented workflow rules and the sibling endpoints. Check it in the Higgsfield Playground and
  adjust `submit` if a field differs, before spending money.
- Image-to-video keeps the **input image's framing**, so a landscape photo gives a landscape video.
  A crop/pad-to-9:16 step before upload is still needed.
- `estimate` uses your configured rates. Higgsfield also documents an estimate endpoint that could
  replace it.

## Placeholders you must replace before charging anyone

- `PACKAGES` prices in `src/domain/catalog.ts` and `USD_TO_LOCAL` in config are placeholders.
  Set them from the provider's real cost estimates and your target margin.
- The Swahili call to action (`Agiza leo!`) and caption template are placeholders. Use an LLM step
  and have a native speaker review.
- Mock provider costs are illustrative, not Higgsfield's rates.
- **Check the unit economics first.** For reference only, fal.ai prices the same Seedance 2.5 model by
  tokens; by its published formula a 5-second vertical clip is roughly $1.03 at 480p and $2.31 at
  720p. Higgsfield's rates may differ, but at the placeholder KES 300 price and the default 35% cost
  cap the budget is only about $0.81, so expect the guardrail to refuse orders until you raise
  prices, shorten clips, or use cheaper models.

## Known gaps (do these before real users)

- **Content moderation.** Customer text and photos go straight into a generation prompt. Add a
  moderation step before `submit`, and check the provider's acceptable-use terms.
- **Rate limiting and abuse controls** on `/uploads` and `/orders`.
- **Order lookup is by unguessable UUID only.** Fine for a demo; add real customer auth later.
- **Single process.** Two instances would both try to resume the same jobs; add a row lease.
- **Kenya-only phone handling** (`lib/phone.ts`); add countries alongside their payment adapters.
- **Timeouts.** If a provider job times out we fail and refund, but it may still finish and be billed.
  Reconcile against the provider dashboard.

## Next steps

1. Crop/pad uploaded photos to 9:16 before submitting (image-to-video keeps the photo's framing).
2. Confirm real Higgsfield rates and the image-to-video request fields, then a first paid test clip.
3. M-Pesa Daraja adapter (sandbox first), including callback authentication.
4. Postgres repo using `src/db/schema.sql`.
5. Moderation, rate limiting, then a mobile-first front end.
