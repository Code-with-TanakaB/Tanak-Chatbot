# Tanak-Chatbot — Technical System Report

**Project:** Cloud Document Storage (CDS) IT & Sales Assistant  
**File:** `index.html` (single-file application)  
**Date:** 23 February 2026  
**Author:** GitHub Copilot  

---

## 1. Overview

Tanak-Chatbot is a fully self-contained, client-side chatbot built as a single HTML file with no external dependencies or backend services. It serves as both an **IT Support Assistant** and a **Sales & Upgrade Assistant** for Cloud Document Storage (CDS), targeting professional sectors such as Medical and Legal.

The entire application — layout, styles, logic, and data — lives inside `index.html` and runs entirely in the user's browser.

---

## 2. Architecture

```
index.html
├── <style>        — All CSS (dark UI theme, component styles)
├── <body>         — HTML structure (chat shell, header, messages, chips, input)
└── <script>       — All JavaScript (state, i18n, NLP classifier, FSM, DOM logic)
```

There is no build step, no npm packages, no server, and no API calls. The chatbot is fully offline-capable once the file is loaded.

---

## 3. UI Structure

### 3.1 Chat Shell (`#chat-shell`)
A fixed-size, centered container (`max-width: 820px`, `height: 90vh`, capped at `860px`) built with CSS Flexbox. It has three stacked regions:

| Region | Element | Purpose |
|---|---|---|
| Header | `#chat-header` | Branding, bot name, language indicator, live status dot |
| Messages | `#messages` | Scrollable conversation history |
| Chips | `#chip-row` | Quick-reply suggestion buttons |
| Input | `#input-area` | Auto-resizing textarea + send button |

### 3.2 Message Bubbles (`.msg-row`)
Each message is a flex row containing:
- `.msg-avatar` — emoji icon (☁️ for bot, 👤 for user)
- `.bubble` — the message content

Bot bubbles align left; user bubbles align right via `flex-direction: row-reverse`. Both animate in via a `fadeUp` CSS keyframe (opacity 0→1, translateY 8px→0 over 250ms).

### 3.3 Typing Indicator (`#typing-indicator`)
Three animated dots (`.dot`) using a `bounce` CSS keyframe with staggered `animation-delay` values (0s, 0.2s, 0.4s). It is shown before the bot "responds" and hidden when the reply renders.

### 3.4 Quick-Reply Chips (`#chip-row`)
Pill-shaped buttons dynamically injected by JavaScript based on conversation context. Clicking a chip auto-fills the textarea and calls `sendMessage()`.

### 3.5 Special Blocks
- **Ticket Block** (`.ticket-block`): Monospaced card rendered inside a bot bubble when a support ticket is created.
- **Escalation Banner** (`.escalation-banner`): Red-tinted alert block rendered for Tier 3 issues.
- **Severity Tags** (`.tag-low`, `.tag-medium`, `.tag-high`): Coloured pill badges inside ticket blocks.

---

## 4. JavaScript Engine

### 4.1 Application State (`state` object)

```js
const state = {
  lang: 'en',        // Active language: 'en' | 'es' | 'fr'
  step: 'idle',      // FSM step (see Section 4.4)
  ticketData: {},    // Accumulates ticket field values during FSM
  salesData: {},     // Reserved for sales enquiry flow
};
```

### 4.2 Internationalisation (`i18n` object)
All user-facing strings are stored in a nested object keyed by language code (`en`, `es`, `fr`). Each language block contains:

| Key | Content |
|---|---|
| `greeting` | Opening welcome message |
| `chips_greeting` | Initial quick-reply chip labels |
| `ask_name` | Ticket flow prompt 1 |
| `ask_email` | Ticket flow prompt 2 |
| `ask_describe` | Ticket flow prompt 3 |
| `ticket_intro` | Confirmation header |
| `ticket_followup` | Post-ticket message |
| `escalate_msg` | Tier 3 escalation message |
| `only_it` | Off-topic refusal message |
| `unknown` | Fallback / clarification request |
| `chips_common` | Post-response quick-reply chip labels |

Strings are retrieved via the helper `t(key)`, which reads from `i18n[state.lang]`.

### 4.3 Language Detection (`detectLang`)
Called on every user message. Uses regex to detect French or Spanish keywords; defaults to English. If the detected language differs from `state.lang`, the language is switched immediately and all subsequent bot replies use the new language.

**French signals:** `bonjour`, `vous`, `réseau`, `problème`, `recommencer`, etc.  
**Spanish signals:** `hola`, `contraseña`, `computadora`, `problema`, `empezar`, etc.

### 4.4 Finite State Machine (FSM) — Ticket Collection
The `state.step` variable drives a linear multi-turn conversation flow for ticket creation:

```
idle
 │  (user says "create ticket" / chips)
 ▼
ticket_name   ← bot asks for full name
 │  (user provides name)
 ▼
ticket_email  ← bot asks for email
 │  (user provides email)
 ▼
ticket_describe ← bot asks to describe the issue
 │  (user describes issue)
 ▼
idle          ← bot renders ticket block + confirmation
```

At each step, the next user message is captured as the relevant field (`ticketData.name`, `.email`, `.description`). The FSM blocks all other processing while active.

### 4.5 Topic Classifier (`TOPICS` array)
An ordered array of topic objects. Each topic has:

| Property | Description |
|---|---|
| `id` | Unique identifier (`password`, `vpn`, `slow`, `browser`, `security`, `hardware`) |
| `level` | Support tier: `1` (low), `2` (mid), `3` (escalate) |
| `rx` | Regular expression tested against the user's raw input |
| `en` / `es` / `fr` | Localised title and step-by-step solution array |

**Matching logic:** Topics are iterated in order. The first matching `rx` wins. Level 3 topics trigger escalation; levels 1 and 2 render step-by-step guidance.

| Topic | Level | Example trigger phrases |
|---|---|---|
| Password Reset | 1 | "forgot password", "contraseña", "mot de passe" |
| VPN / Network | 2 | "vpn", "wifi", "dns", "internet" |
| Slow System | 1 | "slow", "lag", "freeze", "lento", "mémoire" |
| Browser Issues | 1 | "chrome", "cache", "cookie", "navigateur" |
| Security | 3 | "hack", "virus", "malware", "phishing", "breach" |
| Hardware | 3 | "blue screen", "hard drive", "crash", "data loss" |

### 4.6 Response Pipeline (`processInput`)
Every user message passes through the following ordered checks:

```
1. Detect language switch
2. Restart / reset command?       → reset state, show greeting
3. Active FSM step?               → capture field, advance FSM
4. Explicit ticket request?       → start FSM at ticket_name
5. "It worked" confirmation?      → close with success message
6. "Still not working"?           → offer to create ticket
7. TOPICS regex match?
     Level 3 → render escalation banner
     Level 1/2 → render step-by-step solution
8. Non-IT topic filter?           → polite refusal
9. Fallback                       → ask for clarification / offer ticket
```

### 4.7 DOM Helpers

| Function | Purpose |
|---|---|
| `addMessage(who, html)` | Inserts a new message row before the typing indicator |
| `botReply(html, chips, delay)` | Shows typing indicator, then after `delay`ms hides it and renders the reply |
| `setChips(labels)` | Rebuilds the chip row with new buttons |
| `showTyping()` / `hideTyping()` | Toggles the animated typing indicator |
| `scrollBottom()` | Keeps the message list scrolled to the latest message |
| `buildTicketHTML(data)` | Returns the `.ticket-block` HTML string from `ticketData` |
| `generateTicketId()` | Returns a random 6-digit string (100000–999999) |
| `renderMarkdown(text)` | Converts `**bold**` and `` `code` `` to HTML equivalents |

### 4.8 Send Handler
- Triggered by the send button click or `Enter` key (Shift+Enter inserts a newline).
- Sanitises user input (`<` → `&lt;`) before rendering to prevent XSS.
- Clears the textarea and resets its height after sending.

### 4.9 Bootstrap
On `DOMContentLoaded`, a 600ms `setTimeout` fires the English greeting and sets the initial chips, preceded by a brief typing indicator to simulate a natural response delay.

---

## 5. Support Tiers

| Tier | Level | Behaviour |
|---|---|---|
| **Level 1 — Low** | `1` | Bot provides self-service step-by-step resolution guide |
| **Level 2 — Mid** | `2` | Bot provides more technical instructions (e.g., DNS commands) |
| **Level 3 — High** | `3` | Immediate escalation to Tier 3 Senior Engineering; auto-ticket reference generated |

---

## 6. Ticket System

Tickets are **simulated** entirely in the browser. No data is sent to any server. The ticket flow collects:

- **Full Name**
- **Email Address**
- **Issue Description**
- **Severity** (defaults to `Medium`; `High` reserved for escalation paths)
- **Ticket ID** (auto-generated 6-digit random number)

The rendered ticket block is styled in monospace font inside a dark card to visually distinguish it from regular chat bubbles.

---

## 7. Multilingual Support

| Language | Code | Trigger keywords (examples) |
|---|---|---|
| English | `en` | Default fallback |
| Spanish | `es` | hola, contraseña, computadora, problema |
| French | `fr` | bonjour, réseau, mot de passe, problème |

Language is re-detected on **every message**, enabling seamless mid-conversation switching. All bot replies — including chip labels — are rendered in the detected language.

---

## 8. Security & Compliance Notes

- **No backend, no data storage** — all conversation data exists only in browser memory and is lost on page close.
- **No real passwords or sensitive data** are ever requested or stored.
- **XSS prevention** — user input is HTML-escaped before insertion into the DOM.
- **Data privacy** — Tanak-Chatbot does not transmit any user data externally.
- The ticket system is a **UI simulation**; in a production deployment, the `buildTicketHTML` function would be replaced with an API call to a real ticketing backend (e.g., Jira, Zendesk, ServiceNow).

---

## 9. Known Limitations & Future Improvements

| Limitation | Suggested Improvement |
|---|---|
| No real ticket backend | Integrate with a REST API (Zendesk / Jira) |
| Sales flow `salesData` state is reserved but not fully implemented | Build full Sales FSM (package selection → name → field → usage → confirmation) |
| Language detection is keyword-based | Replace with `Intl` or a lightweight ML model for better accuracy |
| No conversation history persistence | Add `localStorage` to save and restore chat sessions |
| Single-file architecture limits scalability | Migrate to a React or Vue component-based structure for larger teams |
| Ticket IDs are random and not sequential | Connect to a real ID-generation endpoint |

---

## 10. File Summary

| Aspect | Detail |
|---|---|
| **File** | `index.html` |
| **Lines** | ~705 |
| **Dependencies** | None (zero external libraries) |
| **Languages** | HTML5, CSS3, Vanilla JavaScript (ES6+) |
| **Browser support** | All modern browsers (Chrome, Firefox, Edge, Safari) |
| **Deployment** | Open the file in any browser — no server required |
| **Repository** | `Code-with-TanakaB/Tanak-Chatbot` on GitHub |
