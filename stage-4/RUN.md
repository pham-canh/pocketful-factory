# RUN.md — stage-4 (stage-3 carried forward, plus refunds and batch corrections)

Build and start, exactly as a judge would: from a clean container, nothing manual.

```bash
docker build -t pocketful-stage4 ./stage-4
docker run --rm -p 8080:8080 -e PORT=8080 pocketful-stage4
```

Verify:

```bash
curl -s http://127.0.0.1:8080/health
# -> {"status":"ok"}
```

Without Docker:

```bash
cd stage-4 && pip install -r requirements.txt && PORT=8080 python3 -m src.app
```

## Seed the accounts, then sign in

A fresh service holds no users: the specification seeds state through
`POST /_test/reset`, and nothing is seeded at container start. Seed the published
fixture, then sign in as `ada@example.com` with the password `correct horse`:

```bash
curl -s -X POST http://127.0.0.1:8080/_test/reset -H 'Content-Type: application/json' -d '{
  "currency": "EUR",
  "minor_units": 2,
  "users": [
    { "id": "u_ada", "email": "ada@example.com", "password": "correct horse",
      "display_name": "Ada", "handle": "ada", "balance": 10000 },
    { "id": "u_bob", "email": "bob@example.com", "password": "correct horse",
      "display_name": "Bob", "handle": "bob", "balance": 2500 }
  ],
  "payments": [
    { "id": "p_1", "from_user_id": "u_ada", "to_user_id": "u_bob",
      "amount": 500, "note": "coffee", "visibility": "public" }
  ],
  "requests": [
    { "id": "rq_1", "requester_id": "u_bob", "payer_id": "u_ada",
      "amount": 1200, "note": "taxi", "status": "pending" }
  ]
}'
# -> 204, and no body
```

After that, <http://127.0.0.1:8080/> shows Ada with a balance of 100.00 EUR, one paid
"coffee" in the feed and one pending "taxi" request. `POST /auth/signup` also works if
you would rather make your own account.

The service binds `0.0.0.0` and reads `PORT`, default `8080`. Dependencies are installed at
build time; nothing reaches the network at run time.

The browser routes serve HTML when the request carries `Accept: text/html`, and JSON otherwise.

Added in this stage: `POST /payments/{id}/refunds`, an idempotent write the receiver alone may
make; `POST /correction-batches`, an operator's idempotent write that is aware of settlements and
applies all of its members or none; and `GET /payments/{id}/refunds`.

## Or open the face on it

`../deploy/streamlit/streamlit_app.py` is the judge-facing page for this stage. It is not a
second implementation: it starts `src/app.py` — the exact module the factory produced — as a
subprocess, and every button on it is one HTTP call against that service.

```bash
pip install -r ../deploy/streamlit/requirements.txt
streamlit run ../deploy/streamlit/streamlit_app.py
```

It starts the service itself on port 8080, so run it before anything else already holds that
port. The capture in `docs/demo.mp4` at the top of the repository is this page: reset answering
204, Ada signed in at 100.00 EUR, an overdraft refused with the balance unchanged, then the same
idempotency key fired twice answering 201 and then 200.

`deploy/build_demo_repo.sh` assembles a separate deployable repo from that page and this
folder, because Streamlit Community Cloud wants `streamlit_app.py` at the root of the repo it
deploys and refuses to keep one copy of `app.py` in two places.

Every stage-1, stage-2 and stage-3 check still passes against this folder.