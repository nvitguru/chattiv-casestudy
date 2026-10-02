<p align="center">
  <img src="assets/og-image.png" alt="Chattiv" width="820">
</p>

<h1 align="center">Chattiv: Case Study</h1>

<p align="center">
  <b>Live chat for small business websites, with an AI assistant that knows when to step aside.</b><br>
  Laravel 13 · PHP 8.3 · MySQL · Redis + Horizon · Reverb · Claude · Stripe · Vue 3 · Preact
</p>

---

Most website chat tools fail small businesses in one of two ways: the chat goes unanswered, or a bot traps the
visitor in a loop with no way to reach a person. Chattiv is built around the opposite promise. People answer by
default. The AI assistant is optional, answers only from the business's own content, always says it is an AI, and
hands the conversation to a human the moment one is needed, with a summary so the visitor never repeats themselves.

This repository is a **portfolio case study**: a public overview of the product, architecture and engineering. The
full application source is private and available to reviewers on request.

> **Status:** live in production at [chattiv.com](https://chattiv.com), running chat on its own site and on
> several other SaaS products in the same portfolio.

---

## The product

<p align="center">
  <img src="assets/home.png" alt="Chattiv marketing site" width="900">
</p>

| | |
|---|---|
| **Chat widget** | One script tag. About 13 KB, isolated in a Shadow DOM, loads after the host page is idle. Brand color, greeting, offline form, file sharing, ratings, transcript by email. |
| **Team inbox** | Real-time inbox with views (waiting, mine, AI, unread), take over and hand back, private notes, canned replies, live visitor details, light and dark mode. Installable as a phone app with push alerts. |
| **Ava, the AI assistant** | Opt-in per website: off, human first with AI backup, AI after hours, or AI first. Learns from the site, FAQ answers and documents, and from the questions she could not answer. |
| **Agent API** | Businesses can connect their own AI. It receives events by signed webhook or WebSocket and replies over REST, under exactly the same handoff rules. |
| **Billing** | Free, Starter and Pro plans with Stripe, plus AI chat packs that never expire and an optional auto top-up with a monthly cap. |
| **Help center** | One set of articles shown in the app, on the public site, and used by Ava to answer customers' "how do I" questions. |

<p align="center">
  <img src="assets/ava.png" alt="Ava, the AI assistant" width="900">
</p>

---

## Architecture

```mermaid
flowchart LR
    subgraph Visitor["Visitor's browser"]
        W["Chat widget<br/>(Preact, Shadow DOM)"]
    end
    subgraph Team["Team"]
        I["Inbox<br/>(Inertia + Vue 3, PWA)"]
    end
    subgraph App["Laravel 13"]
        WAPI["Widget API<br/>origin allowlist"]
        CS["ConversationService<br/>single owner + version guard"]
        AAPI["Agent API /v1<br/>hashed keys"]
        Q["Horizon queues<br/>default · low"]
    end
    R["Reverb<br/>(WebSocket)"]
    DB[("MySQL<br/>FULLTEXT knowledge")]
    C["Claude<br/>(structured output,<br/>prompt caching)"]
    EXT["Customer's own AI"]
    S["Stripe"]
    P["Web push · Email"]

    W -- REST --> WAPI --> CS
    I -- REST --> CS
    CS <--> DB
    CS -- events --> R
    R -- live updates --> W
    R -- live updates --> I
    CS -- jobs --> Q
    Q -- "AvaRespond" --> C
    Q -- "signed webhooks" --> EXT
    EXT -- replies --> AAPI --> CS
    Q --> P
    S -- webhooks --> App
```

One Laravel application serves the marketing site, the app, the Agent API and the widget API on separate hosts.
Laravel Reverb pushes every change to the inbox and the widget in real time. Horizon runs two queues so a long website
crawl can never delay a chat reply.

---

## Engineering highlights

### 1. One owner per conversation, enforced on the server

The core rule of the product is that the AI and a person never talk over each other. Every conversation has exactly
one owner (`ai`, `waiting`, `human`, `team`), and every transition goes through one service that bumps a version
number. A reply from the AI that arrives after a person took over is rejected rather than shown.

```php
// Simplified: a transition only succeeds against the version the caller saw.
$updated = Conversation::whereKey($c->id)
    ->where('owner_version', $c->owner_version)
    ->update(['owner' => $to, 'owner_version' => $c->owner_version + 1]);

if (! $updated) {
    throw new OwnershipConflict();   // someone else moved first
}
```

The handoff triggers are deliberately conservative: the visitor asks for a person (an always-visible button, or plain
words like "real person"), mentions a topic the business marked as human-only, gets two answers the AI could not
ground in the business's content, or the AI errors or misses a 30-second deadline. While the visitor waits, the AI
keeps helping, clearly labeled, and the team receives a short summary with a mood tag.

### 2. An AI that is honest by construction

Ava runs on Claude with **structured output**: every turn returns a JSON object (`reply`, `grounded`, `handoff`,
`summary`, `mood`, `contact`) rather than free text, so the application, not the model, decides what happens next.

- **Grounding.** Each turn retrieves the most relevant passages from that website's own knowledge (crawled pages, FAQ
  answers, documents) with MySQL FULLTEXT. The model must flag answers it could not support; two ungrounded answers
  hand the chat to a person.
- **A learning loop without training.** Ungrounded questions are collected per website. The owner types an answer
  once, it becomes knowledge, and it is used on the very next message.
- **Locked disclosure.** The AI badge and "I'm an AI assistant" behavior are outside the business's control.
- **Cost control.** The stable part of the prompt (persona, rules, business profile) is prompt-cached; only the
  transcript and retrieved knowledge change per turn. Usage is metered per account per month with a hard cap.

### 3. A widget that is a good guest

The widget is the only code Chattiv runs on someone else's website, so it is built to be invisible until wanted:

- Preact and TypeScript bundled by esbuild into one file of about 13 KB gzipped, loaded only after the host page has
  finished and the browser is idle.
- Rendered inside a Shadow DOM so the host's CSS cannot break it and it cannot break the host.
- A minimal hand-written Pusher-protocol client for real-time updates instead of a client library.
- Every request is checked against the website's allowed domains, so a copied snippet does not work elsewhere.

### 4. Bring your own AI, same rules

The Agent API lets a business plug in any AI. Keys are stored hashed and shown once. Webhooks are signed with
HMAC-SHA256 over the timestamp and body, retried with backoff, and logged with response codes so developers can
debug without asking support. External agents get a 30-second reply deadline, after which the chat falls back to
the team automatically.

```text
X-Chattiv-Signature: sha256=<hex HMAC-SHA256(secret, "{timestamp}.{raw body}")>
```

Webhook targets and crawl URLs are validated against private and reserved networks to prevent server-side request
forgery.

### 5. Never miss a chat

Chattiv treats "nobody answered" as a design problem, not an edge case:

- Web push (VAPID) to an installable PWA, so alerts reach phones even when the inbox is closed, plus an in-app
  doorbell sound for the moments that matter.
- A configurable wait window. If nobody answers in time, the AI covers (when enabled) or the visitor is offered an
  email follow-up, and the team's reply is emailed to them, branded as the business.
- Time zones handled end to end: stored in UTC, shown in the reader's own zone, with business hours evaluated in
  each website's zone.

### 6. Billing that cannot surprise anyone

Stripe through Laravel Cashier, attached to the account rather than the user so a team shares one subscription.
AI usage is the only metered cost, so it has a hard monthly cap. Extra AI chat packs never expire, and optional auto
top-up runs only within a monthly spending limit the owner sets, pausing itself if a payment fails.

---

## More of the product

<p align="center">
  <img src="assets/pricing.png" alt="Pricing" width="900">
</p>

<p align="center">
  <img src="assets/help.png" alt="Help center" width="900">
</p>

<p align="center">
  <img src="assets/api-docs.png" alt="Agent API documentation" width="900">
</p>

---

## Quality and safety

- A feature test suite covering ownership and handoff, AI escalation, the offline path, widget origin checks, the
  Agent API and webhook signing, plan limits, billing and top-ups, emails, time zones and the help center.
- The test bootstrap refuses to run against anything but an in-memory database, so a test run can never touch
  production data.
- Accessibility and performance by default: keyboard support in the widget, reduced-motion support, and no work on
  the host page until it has loaded.

## Tech stack

**Backend:** PHP 8.3, Laravel 13, MySQL 8, Redis, Horizon, Reverb, Cashier (Stripe), Fortify (2FA and passkeys),
Anthropic PHP SDK (Claude).
**Frontend:** Inertia 3, Vue 3, TypeScript, Tailwind CSS. **Widget:** Preact, TypeScript, esbuild.

---

<p align="center">
  Chattiv is a product of SocialPoints Media LLC. Source available to reviewers on request.
</p>
