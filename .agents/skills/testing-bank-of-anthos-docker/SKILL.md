---
name: testing-bank-of-anthos-docker
description: Run the full Bank of Anthos stack (this repo) end-to-end on a single box with plain Docker — no Kubernetes — to test one locally-built Java service against the unmodified upstream services. Covers image pulls, JWT key extraction and token minting, the k8s-hostname requirement, ledger-db append-only rules, tracing-enabled boots, and Java 8-vs-Java 21 parity diffing.
---

# End-to-end testing of Bank of Anthos services without a cluster

This repo is a fork of Google's Bank of Anthos pinned to release `v0.5.3`.
`skaffold.yaml` builds images with **jib** (there is no Dockerfile in
`src/balancereader`). You do **not** need a Kubernetes cluster to run the whole
app: plain `docker run` on a user-defined bridge network reproduces the k8s
wiring, because the services address each other by DNS name and the
`*_API_ADDR` env vars can point at container names.

## 1. Pull the unmodified upstream images

All upstream service images pull anonymously (no GCP auth needed):

```
gcr.io/bank-of-anthos-ci/{frontend,userservice,contacts,transactionhistory,ledgerwriter,balancereader,accounts-db,ledger-db}:v0.5.3
```

Pull the image of the service you are changing too — it is the single best
oracle for "behavior must be identical" claims (see §6).

## 2. Extract the demo JWT keypair

`extras/jwt/jwt-secret.yaml` holds base64 `jwtRS256.key` (private) and
`jwtRS256.key.pub` (public). Decode both to files. Services need
`PUB_KEY_PATH`; `userservice` also needs `PRIV_KEY_PATH`. Having the **private**
key locally is what lets you mint your own tokens for authorization tests.

## 3. Bring the stack up (order matters)

1. `ledger-db` and `accounts-db` first, with `USE_DEMO_DATA=True` so the demo
   ledger/users are seeded; wait ~25s for initdb to finish.
2. `userservice`, `contacts`, `transactionhistory`, `ledgerwriter`,
   `balancereader`, then `frontend`.
3. Point the frontend at container names:
   `TRANSACTIONS_API_ADDR=ledgerwriter:8080`, `BALANCES_API_ADDR=balancereader:8080`,
   `HISTORY_API_ADDR=transactionhistory:8080`, `CONTACTS_API_ADDR=contacts:8080`,
   `USERSERVICE_API_ADDR=userservice:8080`.
4. Ledger services need `LOCAL_ROUTING_NUM=883745000` to match the seeded data,
   plus `SPRING_DATASOURCE_URL/USERNAME/PASSWORD`
   (`jdbc:postgresql://ledger-db:5432/postgresdb`, `admin` / `password`).
5. Demo login is `testuser` / `password`; set `DEFAULT_USERNAME`/`DEFAULT_PASSWORD`
   on the frontend to prefill it.

### MANDATORY: fake a Kubernetes pod hostname

The Java services derive Stackdriver resource labels with
`podName.substring(0, podName.indexOf("-"))` on `$HOSTNAME`
(`BalanceReaderApplication.resourceLabels()`). Docker's default hostname has no
`-`, so `indexOf` returns `-1` and the container dies at startup with
`StringIndexOutOfBoundsException`. Always run each Java service with
`--hostname <service>-0` and `-e NAMESPACE=default`.

Note this happens even with `ENABLE_METRICS=false` (the flag gates
`enabled()`, not bean creation), and it reproduces on the **unmodified** Java 8
images — so it is upstream behavior, not a regression from your change. Work
around it; don't "fix" it inside a behavior-preserving PR.

## 4. Minting JWTs for authorization tests

Use PyJWT + the decoded private key. Claims the services check: `acct` (must
equal the requested account id) and standard `exp`. Cases worth covering:
valid/matching, valid-but-different-account, garbage `abc.def.ghi`, expired,
signed with a *foreign* RSA key, unsigned `alg=none`, and the RSA→HMAC
algorithm-confusion token (`alg=HS256` HMAC-signed with the RSA **public** key).

PyJWT deliberately refuses to build that last one
(`InvalidKeyError: ... should not be used as an HMAC secret`). If you shell-out
and ignore the error you will silently send an **empty** token and get a
meaningless "pass". Hand-assemble it instead: base64url header + payload,
`hmac.new(pubkey_bytes, signing_input, hashlib.sha256)`.

Pre-existing, non-regression behaviors to record but not fail on: a raw token
with **no** `Bearer ` prefix is accepted (200), and a missing `Authorization`
header yields **400** (`MissingRequestHeaderException`), not 401.

## 5. Ledger DB gotchas

- `TRANSACTIONS` is **append-only**: `src/ledger-db/initdb` installs
  `prevent_delete` / `prevent_update` rules (`DO INSTEAD NOTHING`). `DELETE`
  reports `DELETE 0` and silently changes nothing. To test the LedgerReader
  liveness path you must `DROP RULE prevent_delete ON transactions;` first, then
  delete the newest rows so `MAX(TRANSACTION_ID)` goes backwards — the reader
  logs `out of sync`, sets `alive=false`, and `/healthy` flips to 500
  (`Ledger reader not healthy`) while `/version` and `/ready` stay 200. Run this
  **last**; it poisons the ledger for other tests.
- Amounts are **cents**; the frontend divides by 100. `AMOUNT` is `integer` but
  balances are summed into a `bigint`, so an account's balance can legitimately
  exceed int32 — a good probe for native-query type regressions.
- Inserting rows directly into `TRANSACTIONS` is the way to exercise the
  background reader: the new balance should appear within `POLL_MS`.
- Cross-routing filtering: rows whose `TO_ROUTE`/`FROM_ROUTE` is not
  `LOCAL_ROUTING_NUM` must be ignored by `findBalance`.

## 6. Testing Spring Boot 2 -> 3 / Java 8 -> 21 upgrades

- **Best oracle:** run the upgraded image and the unmodified upstream image
  side-by-side on different host ports against the **same** database, then diff
  `body|http_code|content_type` for every endpoint and every auth case. Anything
  that differs is either a real regression or a Boot-3 default change you must
  explain. Known benign Boot 3 differences: the auto-generated error JSON drops
  the empty `"message"` field, and some malformed input that used to escape as a
  500 now maps to the handled 401/400 path.
- **Hibernate 6 native-query types** (`Long` vs `BigInteger`) silently corrupt
  money. Cross-check the two independent paths for the same account: warm the
  cache (native `findBalance`), insert a row so the Java-side reader callback
  does the arithmetic, then `docker restart` the service to force a cold recompute
  by the native query. Both must produce the identical value, including for a
  negative balance and one exceeding int32.
- **`ENABLE_TRACING=true` will crash the app** in any non-GCP environment
  (`stackdriverSender` -> `Application Default Credentials are not available`).
  This is identical on the unmodified Java 8 image, so it is not a regression.
  To actually test the tracing code path, point `GOOGLE_APPLICATION_CREDENTIALS`
  at a **dummy** service-account JSON. The private key must be **PKCS#8**
  (`openssl pkcs8 -topk8 -nocrypt`) — the repo's `jwtRS256.key` is PKCS#1 and
  yields `Invalid PKCS#8 data`. With a well-formed dummy key the app boots and
  serves traffic; no traces are exported and nothing needs network at startup.
- With tracing/metrics off, `Your default credentials were not found` warnings
  and Mockito's Java 21 self-attach warning are expected noise, not failures.

## Devin Secrets Needed

None. All images pull anonymously, and the JWT keypair ships in the repo under
`extras/jwt/`. Cloud Trace / Cloud Monitoring export cannot be validated without
real GCP credentials — report it as untested rather than faking it.
