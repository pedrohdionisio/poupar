> 🌎 **English** · [Português (Brasil)](README.md)

# Poupar

Grocery spending tracker that turns a **photo of a receipt** into a structured purchase, with price
history per product and spending by category.

This is the project's **hub** repository: it holds no code, it just gathers the repositories that
make up Poupar and explains how they fit together.

---

## Repositories

| Repository | What it is | Stack |
| --- | --- | --- |
| [**poupar-app**](https://github.com/pedrohdionisio/poupar-app) | Mobile app — sign in, receipts, statistics, receipt scan, manual purchase and merchants. | React Native 0.86 + Expo 57, strict TypeScript, NativeWind, TanStack Query |
| [**poupar-api**](https://github.com/pedrohdionisio/poupar-api) | Serverless API — auth, purchases, scans, products and aggregations. | AWS Lambda + API Gateway HTTP v2, DynamoDB single-table, Cognito, S3, SQS, OpenAI |

Each repository ships its own full README (architecture, conventions, environment variables,
deployment) in both Portuguese and English.

---

## How the pieces fit together

```
┌──────────────────┐    HTTPS + JWT (Cognito)    ┌──────────────────────────┐
│   poupar-app     │ ──────────────────────────▶ │       poupar-api         │
│  React Native    │ ◀────────────────────────── │  Lambda + API Gateway    │
└────────┬─────────┘                             └────────────┬─────────────┘
         │                                                    │
         │ direct upload (presigned POST)                     │
         ▼                                                    ▼
   ┌───────────┐   ObjectCreated   ┌───────────┐      ┌────────────────┐
   │  S3       │ ────────────────▶ │  SQS      │ ───▶ │  DynamoDB      │
   │  uploads  │                   │  scans    │      │  single-table  │
   └───────────┘                   └─────┬─────┘      └────────────────┘
                                         │
                                         ▼
                                   ┌───────────┐
                                   │  OpenAI   │  receipt extraction
                                   └───────────┘
```

The app **never** talks to S3, SQS or DynamoDB on its own. The single exception is the photo upload,
which goes straight to S3 using a signature issued by the API.

---

## Receipt scan flow, end to end

The path that crosses both repositories:

| # | Where | What happens |
| --- | --- | --- |
| 1 | app | The user picks the merchant and photographs the receipt (camera or gallery; HEIC is transcoded to JPEG on iOS). |
| 2 | app → api | `POST /scans` returns a `scanId` and a presigned POST valid for 5 min. |
| 3 | app → S3 | Direct multipart upload through a separate axios instance with no `Authorization` — S3 rejects that header. |
| 4 | api | `ObjectCreated` under the `scans/` prefix publishes to SQS; the `processScan` lambda extracts the items via OpenAI and stores the draft. |
| 5 | app → api | Polling `GET /scans/{scanId}` every 2s, capped at 180s: `PENDING → PROCESSING → AWAITING_REVIEW` (or `FAILED`). |
| 6 | app → api | The user reviews the draft and confirms: `POST /scans/{scanId}/confirm` creates the purchase, the receipt, the products and the price points, and updates the category spend. |

---

## Running the whole thing

Order matters: **the app has no backend mock**, so the API must be deployed first.

### 1. API

Requires Node 22+, pnpm 11+, AWS credentials, an authenticated Serverless Framework v4, an OpenAI
API key and a domain verified in SES.

```bash
git clone git@github.com:pedrohdionisio/poupar-api.git
cd poupar-api
pnpm install
cp .env.example .env      # fill it in before moving on
serverless deploy --stage <your-stage>
```

Take note of the API Gateway URL the deployment prints.

### 2. App

Requires Node 20+, yarn 1.22, and Xcode 16+ and/or Android Studio. The project uses
`expo-dev-client` — Expo Go **cannot** run this app.

```bash
git clone git@github.com:pedrohdionisio/poupar-app.git
cd poupar-app
yarn install
cp .env.example .env      # EXPO_PUBLIC_API_URL = the URL from the previous step
yarn ios                  # or: yarn android
```

Once the dev client is installed, day-to-day work is just `yarn start`.

---

## Shared conventions

The two repositories are independent, but they follow the same spirit:

- **Strict TypeScript**, no `any` and no `as` to silence the compiler.
- **Dependencies point one way only** — Clean Architecture in the API
  (`main → application → entities`), three layers in the app (`presentation → data → shared`).
- **Money as integer cents and quantity in thousandths** across the API contract; conversion to
  currency and units lives only in the app's mappers.
- **Comments explain why**, never restate the code.
- **Named exports** by default; Biome handling lint, formatting and import ordering.
- Each repository carries its own AI tooling under `.claude/` (rules, skills and subagents).

---

## License

`poupar-api` is ISC. `poupar-app` is private, with no distribution license defined.
