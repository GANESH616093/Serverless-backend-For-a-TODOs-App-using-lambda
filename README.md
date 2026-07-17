# TODOs — Serverless Backend

A microservice-style TODOs API built entirely on managed AWS services —
no servers to patch, scale, or pay for while idle.

## Architecture

```
                         ┌─────────────────────┐
   Browser / Frontend ──▶│    API Gateway       │
                         │  (REST API, public)  │
                         └──────────┬───────────┘
                                    │
                    ┌───────────────┼────────────────┐
                    │               │                 │
             POST /auth/*     Lambda Authorizer   /todos/*
             (public)         (validates JWT)     (protected)
                    │               │                 │
                    ▼               ▼                 ▼
              register.js /    Allow/Deny      create/list/get/
              login.js          IAM policy      update/delete.js
                    │                                  │
                    └───────────────┬──────────────────┘
                                    ▼
                         DynamoDB (Users, Todos)
```

- **API Gateway** is the single public entry point — the "front door" of
  the microservice architecture. Every route maps 1:1 to one small,
  independently-deployable Lambda function.
- **Lambda Authorizer** (`src/authorizer/authorizer.js`) is a separate
  function that API Gateway calls *before* any `/todos/*` route. It checks
  the JWT in the `Authorization` header and returns an IAM policy telling
  API Gateway whether to let the request through — the todos handlers
  never see a request from someone who isn't authenticated.
- **DynamoDB** stores `Users` (keyed by email) and `Todos` (keyed by
  `userId` + `todoId`, so a Query naturally returns only that user's
  todos).
- Everything scales to zero — you pay per request, not per idle hour.

## Prerequisites

- Node.js 18+
- An AWS account + AWS CLI configured (`aws configure`) with a user that
  has permission to create Lambda, API Gateway, DynamoDB and IAM
  resources
- `npm install -g serverless` (Serverless Framework v3)

## Deploy — this is how you get your live host link

```bash
cd todos-serverless-backend
npm install
export JWT_SECRET="pick-a-long-random-string-here"
npx serverless deploy
```

The deploy takes 1–3 minutes. When it finishes, Serverless Framework
prints an **endpoints** block like:

```
endpoints:
  POST - https://abc123xyz.execute-api.us-east-1.amazonaws.com/dev/auth/register
  POST - https://abc123xyz.execute-api.us-east-1.amazonaws.com/dev/auth/login
  POST - https://abc123xyz.execute-api.us-east-1.amazonaws.com/dev/todos
  GET  - https://abc123xyz.execute-api.us-east-1.amazonaws.com/dev/todos
  ...
```

`https://abc123xyz.execute-api.us-east-1.amazonaws.com/dev` **is your
live host link.** Paste it into the frontend's "API URL" field and the
web app is fully working, hosted, and shareable.

To tear everything down and stop being billed: `npx serverless remove`.

## API reference

| Method | Path             | Auth | Body                                |
|--------|------------------|------|--------------------------------------|
| POST   | /auth/register   | none | `{ name, email, password }`         |
| POST   | /auth/login      | none | `{ email, password }`               |
| POST   | /todos           | JWT  | `{ title, description? }`           |
| GET    | /todos           | JWT  | —                                    |
| GET    | /todos/{id}      | JWT  | —                                    |
| PUT    | /todos/{id}      | JWT  | any of `{ title, description, completed }` |
| DELETE | /todos/{id}      | JWT  | —                                    |

Authenticated requests send `Authorization: Bearer <token>` using the
token returned by register/login.

## Hosting the frontend (get a link for the web app itself)

The API deploy above gives you a live *backend* URL. To get a public
*website* link too, put `index.html` on S3 as a static site.

```bash
# 1. Create a uniquely-named bucket
aws s3 mb s3://relay-todos-app-yourname

# 2. Enable static website hosting on it
aws s3 website s3://relay-todos-app-yourname --index-document index.html

# 3. Allow public reads (required for a public static site)
aws s3api put-bucket-policy --bucket relay-todos-app-yourname --policy '{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicReadGetObject",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::relay-todos-app-yourname/*"
  }]
}'

# 4. Upload the frontend
aws s3 cp index.html s3://relay-todos-app-yourname/index.html



Open that URL, paste your API Gateway base URL into the "API BASE URL"
field once, and it's saved in the browser from then on. (For HTTPS + a
custom domain, front the bucket with CloudFront — optional for a
learning project, but the standard next step for a production one.)

Netlify or Vercel are equally valid one-command alternatives if you'd
rather skip the S3/CloudFront setup entirely — just drag-and-drop
`index.html` into their dashboard.

## Local development (optional)

`npm run dev` runs the API on `http://localhost:4000` via
`serverless-offline`, no AWS deploy needed for iterating on handler logic
(DynamoDB calls still need real AWS credentials, or swap in
`dynamodb-local`).

## Why a Lambda Authorizer instead of checking the token in every handler?

Centralizing auth in one function means:
- The five todos handlers stay focused only on todo logic
- Auth logic changes (rotate the JWT secret, add scopes, swap to OAuth)
  happen in one place
- API Gateway caches the authorizer's decision (`resultTtlInSeconds`),
  so you're not re-verifying a JWT on every single request if you raise
  that number
- It's the standard pattern for securing a REST API Gateway in a
  microservice architecture — each downstream service trusts the gateway
  layer instead of reimplementing auth
