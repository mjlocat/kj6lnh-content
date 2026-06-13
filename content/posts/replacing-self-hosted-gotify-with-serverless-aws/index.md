---
title: "Replacing Self-Hosted Gotify with a Serverless AWS Backend"
date: 2026-06-12
lastmod: 2026-06-12
draft: false
description: "How I rebuilt the Gotify notification server as Go Lambdas, DynamoDB, and a single CloudFront distribution — keeping the stock Android app working unmodified, for pennies a month."
slug: "serverless-gotify"
tags:
  - aws
  - serverless
  - gotify
  - lambda
  - terraform
  - go
categories:
  - Computers
---

For a couple of years I ran [Gotify](https://gotify.net) on a small EC2 instance to get push
notifications from my own scripts and services to my phone. It's a great little
project: a single Go binary, a REST API to post messages, and an Android app that
holds a WebSocket open to receive them in real time. The problem was never Gotify —
it was the box it ran on. Patching, the occasional reboot, paying for an always-on
server to handle a few notifications a day. It felt like a lot of standing
infrastructure for something that's idle 99.99% of the time.

So I rebuilt the **server** as a serverless stack on AWS, with one hard rule: the
**stock Gotify Android app must keep working, unmodified**. No fork, no custom build —
just change the server URL in the app's settings and everything continues to work.

The result is [`serverless-notify`](https://github.com/mjlocat/serverless-notify): a
Gotify-wire-compatible backend built from Go Lambdas, DynamoDB, and a single CloudFront
distribution. It costs me effectively pennies a month, and there's no server to keep
alive.

## The one constraint that shaped everything

"Keep the stock app working" sounds like a small requirement. It turned out to be the
single most important architectural constraint in the whole project, because of one
detail in how the app connects.

The Gotify app derives its WebSocket URL from the same base URL you give it for REST.
Point it at `https://notify.example.com` and it will:

- make REST calls to `https://notify.example.com/message`, `/application`, etc., and
- open a WebSocket to `wss://notify.example.com/stream`.

Same host, same scheme family, two very different protocols. On AWS, the natural
homes for these are an **HTTP API** (for REST) and a **WebSocket API** (for `/stream`)
in API Gateway. But API Gateway will not let an HTTP API and a WebSocket API share a
custom domain. And the app gives me exactly one base URL to work with.

The fix is to put **CloudFront** in front of both and let it be the single public
hostname:

```
 Senders / Android app  ──HTTPS+WSS──▶  CloudFront (notify.example.com)
                                          │
                                          ├─ /stream*  ─▶ API Gateway WebSocket API ─▶ ws Lambda
                                          ├─ /image/*  ─▶ S3 (app icons)
                                          └─ default   ─▶ API Gateway HTTP API      ─▶ api Lambda
                                                                                         │
                              new message ──postToConnection()──▶ live WS connections    │
                                                                                         ▼
                                                          DynamoDB (single table) + SSM (credentials)
```

CloudFront routes by path: `/stream*` goes to the WebSocket API, `/image/*` to an S3
bucket of app icons, and everything else to the HTTP API.

There's one more wrinkle. The WebSocket API only answers on its stage path
(`/<stage>`), not on `/stream`. So a tiny **CloudFront Function** runs on the
WebSocket handshake and rewrites the URI:

```js
function handler(event) {
  var request = event.request;
  request.uri = "/production"; // the WS API stage
  return request;
}
```

That's the whole trick that makes `wss://notify.example.com/stream` reach a WebSocket
API living at a completely different execute-api hostname. With it in place, the app
can't tell it's not talking to a real Gotify server.

## Two Lambdas, one table

The compute is two small Go Lambdas on the `provided.al2023` ARM64 runtime:

- **`api`** — the entire Gotify REST surface (applications, clients, messages) plus the
  sender endpoint `POST /message`. When a new message arrives it also fans it out to
  every live WebSocket connection.
- **`ws`** — the WebSocket lifecycle: `$connect` (authenticate the client token, record
  the connection), `$disconnect` (forget it), and `$default`.

Everything persists in a **single DynamoDB table** using the classic single-table
pattern — applications, clients, messages, live connections, and an atomic counter all
share one table, distinguished by partition key:

```
Application  PK="APP"      SK=<padded id>
Client       PK="CLIENT"   SK=<padded id>
Message      PK="MSG"      SK=<padded id>
Connection   PK="CONN"     SK=<connectionId>
Counter      PK="COUNTER"  SK=<name>
```

A small but important detail: Gotify message IDs are **monotonic integers**, and the
app's pagination relies on it (each page is "messages with an ID less than the last one
I saw"). DynamoDB doesn't hand you an auto-increment column, so I use an atomic
`UpdateItem ADD` against a counter item to mint the next ID. Sort keys are
zero-padded so lexicographic ordering matches numeric ordering.

When the `api` Lambda accepts a message it calls `PostToConnection` on each stored
connection ID to push it down the open sockets. Dead connections come back as a
`GoneException`, which is my cue to prune them from the table.

## Wire compatibility is mostly about being boring

Most of "make the stock app happy" is just returning exactly the JSON shapes Gotify
returns. The app reads `GET /version` and gates behavior on it, so the server reports a
real-looking Gotify version. Tokens follow Gotify's format — a one-character type
prefix (`A` for application/sender tokens, `C` for client/receiver tokens) followed by
14 URL-safe random characters — so the app accepts them without complaint.

Auth is deliberately minimal because this is a **single-user** deployment: it's just
me. Senders present an application token; the app presents its client token for both
REST and the WebSocket. Management calls also accept HTTP Basic auth, with the one
username/password pair stored as an encrypted SecureString in SSM Parameter Store
(behind a dedicated KMS key) and loaded once at Lambda cold start.

## The bug that only the real app could find

Here's my favorite gotcha, because it's a perfect example of why "it passes curl" is
not the same as "it works."

After switching my phone to the new server, live notifications arrived perfectly over
the WebSocket. But when I opened the message list, it just spun forever — no history
ever loaded. The app logs had the answer:

```
java.lang.NullPointerException: getSince(...) must not be null
  at com.github.gotify.messages.provider.MessageStateHolder.newMessages(...)
```

The message list endpoint returns a paging envelope with a `since` cursor pointing at
the next page. My Go struct had it tagged like this:

```go
Since int64 `json:"since,omitempty"`
```

`omitempty` looks harmless. But on the **last** page — the common case where there's
nothing older to fetch — `Since` is `0`, so `omitempty` dropped the field from the JSON
entirely. The Gotify Android client treats `paging.since` as **non-nullable** and
crashes the instant it's missing. The WebSocket path never touches paging, which is
exactly why live delivery worked while the list was dead.

The fix was simply to remove the `omitempty` flag — always emit the field:

```go
Since int64 `json:"since"`
```

No amount of `curl`-ing the endpoint myself would have surfaced this. Only the real
client, with its strict deserialization, cared. It's a good reminder that
"wire-compatible" means compatible with the *consumer's* expectations, not just with a
spec you read once.

## App icons: a half-wired feature, finished

The other rough edge was application icons. The infrastructure was there — CloudFront
serves `/image/*` and `/static/*` from a private S3 bucket via Origin Access Control —
but two things were missing: the bucket had no default icon (so the placeholder
`404`/`403`'d), and the upload endpoint was a stub that accepted your image and quietly
threw it away.

Finishing it was straightforward: `POST /application/{id}/image` now parses the
multipart upload, sniffs the content type, stores the file in S3 under `image/`, and
points the application's `image` field at the new object. Terraform uploads a real
default icon so fresh applications have something to show. It's the same call the app's
own icon picker makes, so custom icons now "just work" from the phone.

## What I deliberately left out: the web UI

Gotify isn't just a server and a phone app — it also ships a built-in web interface. It's
a single-page React app the server hosts at the root, and it lets you do everything from a
browser: read your message history, create and delete applications and clients, copy
tokens, upload icons, change your password, and manage plugins. I haven't reimplemented
any of it, and that was a conscious decision rather than an oversight.

The reason is that my one hard rule was about the **Android app**, not the web client.
Every endpoint the phone needs is implemented and wire-compatible; the web UI leans on a
handful of endpoints the app never touches — things like `PUT /current/user/password`
and the whole `/plugin` surface. Building the web client would mean serving a bundle of
static assets *and* implementing that extra slice of the API, all to duplicate
functionality I can already get with a `curl` one-liner or a quick Postman request. For a
single-user deployment that's just me, the cost/benefit isn't there.

So administration is "API-first": I create an application with a `POST /application`, grab
the returned token, and start sending. There's no dashboard to log into, which honestly
suits how I use it — I provision a sender once and never think about it again. It does
mean two things worth being upfront about. First, if you point a browser at the root URL
you get an API 404, not a friendly login page. Second, anything that *only* exists in the
web UI — password changes, plugin configuration — simply isn't available here; the
credentials live in Terraform and SSM instead.

If I ever wanted the browser experience back, the path is clear: drop Gotify's prebuilt UI
assets into the S3 bucket, add a CloudFront behavior to serve them at the root, and fill in
the few missing endpoints. But until I actually miss it, it stays out — every feature I
don't build is a feature I don't have to secure, pay for, or maintain, which was the entire
point of going serverless in the first place.

## What it costs

This is the part that still makes me smile. One always-connected phone is roughly 44k
WebSocket connection-minutes a month, which is about a **cent**. Add some negligible
Lambda invocations, on-demand DynamoDB, and CloudFront, and the whole thing rounds to
pennies. Compared to an always-on EC2, the economics aren't close — and there's nothing
to patch.

## The honest limitations

It's not magic, and there are trade-offs worth naming:

- **API Gateway WebSocket connections have a 10-minute idle timeout and a hard 2-hour
  maximum.** The app reconnects automatically and back-fills anything it missed via
  `GET /message?since=…`, so nothing is lost, but there can be a brief delay across a
  reconnect. (This is also why connection records carry a TTL — dead sockets
  self-clean even if a `$disconnect` is missed.)
- **No true push.** Real push via FCM or UnifiedPush would fix both the reconnect
  window and battery usage, but it requires a modified app — which breaks my one rule.
  It's parked in the backlog.
- **Single-user by design.** This replaces *my* Gotify server. Multi-user would mean
  real auth, per-user data partitioning, and a lot more surface area.

For my use case — get notifications from my own services to my own phone, reliably,
cheaply, with no server to babysit — it hits exactly the mark I wanted.

## Wrapping up

The interesting parts of this project weren't the Lambdas or the DynamoDB schema; those
are well-trodden. The interesting parts were the seams: convincing a stock client that a
pile of managed AWS services is a single Gotify server. One hostname fronting two
incompatible API types. A CloudFront Function smoothing over a path mismatch. And a
`json:"...,omitempty"` tag that turned out to be the difference between "works" and
"spins forever."

The whole thing is open source (MIT) and Terraform-deployable end to end:
[**github.com/mjlocat/serverless-notify**](https://github.com/mjlocat/serverless-notify).
If you're self-hosting Gotify and tired of feeding an EC2 or VPS, it might be worth a look.
