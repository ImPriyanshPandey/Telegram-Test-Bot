# Project Context — Telegram Bot

This file is a running summary meant to be pasted into a new AI chat (or kept in a project folder) so context doesn't have to be re-explained from scratch.

**User background:** Beginner-to-intermediate with Python (has completed a separate Spotify automation project). Using Windows. Prefers step-by-step explanations of *why* each piece of code does what it does, not just code to paste. Wants installation instructions spelled out, not assumed.

---

## Project: Telegram Multi-Feature Bot (IN PROGRESS — core features working, payment system hardened)

**Goal:** A Telegram bot with three core features (ticket system, group management, searchable link library), expanded with a manual, screenshot-verified UPI payment/unlock system for premium library content.

### Environment
- OS: Windows
- Language: Python
- Library: `python-telegram-bot` (v22.x, async/await style — `pip install python-telegram-bot --upgrade`)
- Also installed: `openpyxl` (for reading the Excel-based library)
- **Newly added: `qrcode[pil]`** (for generating UPI payment QR codes) — `pip install qrcode[pil] --upgrade`
- Project folder: `C:\Users\priya\telegram-bot`
- Bot created via `@BotFather` on Telegram; token obtained via `/newbot`
- User's own Telegram ID obtained via `@userinfobot`, used as the first entry in `ADMIN_IDS`

### Hosting decision
- Currently runs locally on the user's Windows laptop for development/testing (same pattern as the Spotify project — `python bot.py`, keep the window open, `run_polling()`).
- **Recommended next step when ready to go live 24/7: Railway** (simple git-based deploy, no Linux/server management needed). Oracle Cloud "Always Free" was discussed as a genuinely-free-forever alternative but requires Linux/SSH skills not yet covered — deferred for later. Render was discussed but its free tier doesn't suit an always-polling bot well.

### Files in the project folder
- **`bot.py`** — the main bot script, organized into labeled sections (see structure below). Currently the working, tested version, with a fully rebuilt payment/security layer.
- **`config.py`** — holds all secrets/config, kept out of GitHub via `.gitignore`:
  ```python
  BOT_TOKEN = "..."          # from BotFather
  ADMIN_IDS = [123456789]    # list, designed to support multiple admins
  UPI_ID = "yourname@bank"   # for the payment feature
  UPI_NAME = "Priya"
  ```
- **`library.xlsx`** — the searchable link library. Three columns: **Keyword**, **Link**, **Price** (blank/0 = free, any number = locked behind payment). Currently placeholder data (Google, Wikipedia, GitHub, etc. + two example paid entries) — user plans to swap in real data later.
- **`known_groups.json`** — auto-generated/updated at runtime; tracks every group the bot has been added to (chat ID, title, invite link).
- **`purchases.json`** — auto-generated; tracks which user IDs have unlocked which paid library keywords. The **only** file `grant_purchase()` writes to, and `grant_purchase()` is only ever called from the admin-gated `confirm_payment()` — this is the single source of truth for "who owns what."
- **`pending_payments.json`** — auto-generated; tracks in-flight payment claims awaiting admin confirmation. **Also now holds the sequential request-ID counter** (reserved key `"_next_id"`) — merged in deliberately so allocating a new ID and saving the new request happen in one atomic write instead of two files that could drift out of sync.
- ~~`request_counter.json`~~ — **removed.** Was a separate file for ID generation; folded into `pending_payments.json` instead (see above). Safe to delete if it still exists from earlier testing.

### `bot.py` structure (as of latest version, organized into labeled sections)
1. **Imports**
2. **Constants** — conversation stage labels, filenames
3. **Data storage helpers** — load/save for groups, library (with price), purchases, pending payments. All saves now go through `_atomic_write_json()` (see Security section below).
4. **Basic commands** — `/start`, `/menu` (shows a button grid: Raise a Ticket / My Groups / Library / **My Purchases**)
5. **Ticket system** — multi-step `ConversationHandler` flow asking Name/Username → Age → Contact (optional) → Email (optional) → Description, then sends a formatted summary to every admin in `ADMIN_IDS`. Triggered by `/ticket` or the menu button.
6. **Group management** — auto-tracks every group the bot is added to via `ChatMemberHandler`. `/mygroups` (or menu button) shows buttons: "Leave" for groups the user is in, "Join" (invite link) for ones they're not. Leaving uses ban+immediate unban (clean removal, not a punitive ban). **Important limitation understood and designed around:** a bot cannot make a user join a group automatically — Telegram requires the user to click an invite link themselves; the bot also needs to be an **admin** in a group to remove other users from it.
7. **Library search** — `/library` (or menu button) offers **Random** or **Word Based** search. Word-based matches on the *first letter* of the keyword (`.startswith()`), not "contains." Results/picks are shown as buttons — free/already-purchased items open the direct link, locked paid items route into the payment flow. **`/purchases`** (or menu button) now shows a filtered list of only the items *this specific user* has actually paid for and been confirmed on, reusing the same `has_purchased()` check as everywhere else so there's no separate, driftable list.
8. **Payments** (manual UPI, screenshot-verified, no gateway) — see below, fully rebuilt from the original button-only version.
9. **Main/handler registration** — `Application.builder()...run_polling()`. **Handler order matters** in this library (first match wins, no fallthrough): the ticket `ConversationHandler` must be registered *before* the generic library letter-search text handler, or typed ticket answers get silently swallowed. The photo handler (payment proof) is registered before the generic text catcher too, though order between them doesn't strictly matter since they match different message types.

### Payment system — current design (rebuilt from the original button-only flow)
- Originally planned to use **Razorpay** via Telegram's Bot Payments API — pivoted away due to business KYC requirements and ~2%+GST fees, impractical for an unregistered individual project.
- **First manual-UPI version** used a `upi://pay?...` link inside an inline keyboard **URL button** — this turned out to be a hard Telegram API limitation: inline URL buttons only accept `http://`, `https://`, or `tg://` schemes, so every tap crashed with `telegram.error.BadRequest: unsupported url protocol`. Fixed by switching to a **QR code image** instead (see below) — any UPI app can scan it, no clickable link needed.
- **Current flow, end to end:**
  1. Paid library entries show as `🔒 Keyword (₹Price)` buttons.
  2. Tapping one first checks `has_purchased()` — **if already owned, the bot just replies with the link immediately and stops there.** This guards against duplicate payment requests (e.g. tapping an old 🔒 button still sitting in chat history from before it was confirmed), which matters because double-paying via UPI can cause real bank-side reconciliation issues.
  3. If not yet owned: generates a sequential, human-readable **Request ID** (`001`, `002`, `003`, ...) via `allocate_request_id()`, saves a new entry in `pending_payments.json` with status `"awaiting_proof"`, and generates a **QR code image** (via the `qrcode` library, in-memory with `io.BytesIO()`, no file written to disk) encoding the `upi://pay?...` deep link. Sends the QR photo with a caption showing the Request ID, amount, and UPI ID as plain text (manual-entry fallback).
  4. User pays via their own UPI app (scan or manual), then **sends a screenshot as proof, right in the chat.**
  5. `handle_payment_proof()` catches the photo. Tracks *all* of a user's currently-open payment requests in `context.user_data['awaiting_proof_ids']` (a list, not a single value) — if only one is open, auto-matches the screenshot to it; if more than one, checks the photo's caption against the open Request IDs and asks the user to resend with the right ID captioned if it doesn't match. This prevents a screenshot for one item being wrongly linked to another when a user is paying for multiple things at once.
  6. The screenshot is forwarded via `send_photo` to **all** admins (loops over `ADMIN_IDS`) with the image itself plus Confirm/Reject buttons in the caption — admin reviews the actual payment proof inline, no app-switching required.
  7. `confirm_payment()` / `reject_payment()` both re-verify `query.from_user.id in ADMIN_IDS` before doing anything (Telegram guarantees this ID can't be spoofed by another user). On Confirm: the request is `pop()`-ped from `pending_payments.json` **immediately**, before any further action — this closes a race condition where two admins tapping Confirm near-simultaneously could otherwise both process the same request. Then `grant_purchase()` writes the unlock to `purchases.json` and the real link is sent to the buyer. Reject notifies the buyer without unlocking anything.
- **Known/accepted limitation (inherent to a no-gateway system, not a bug):** the admin still has to visually judge that a screenshot is a genuine, matching payment before confirming — code can't verify a screenshot wasn't reused or altered. Everyone in `ADMIN_IDS` is fully trusted by design.
- **Possible future upgrade path (not started):** register as an individual/sole proprietor with Razorpay for automated payment verification, if/when volume justifies it. Stripe/PayPal payment links discussed as secondary options for later/international use.

### Security hardening done in this pass
- **Duplicate-payment guard:** already-purchased items short-circuit before a new payment request is ever created (see step 2 above) — prevents accidental double-charging.
- **Atomic file writes:** `_atomic_write_json()` writes to a `*.tmp` file then `os.replace()`s it into place — atomic on both Windows and Linux, so a crash or power loss mid-save can never leave a half-written/corrupted JSON file. Used by every save function (`save_groups`, `save_purchases`, `save_pending`).
- **Single source of truth for request IDs:** the counter lives inside `pending_payments.json` itself (not a separate file) specifically so that allocating an ID and saving the request it belongs to happen in one write — eliminates any drift risk between two files.
- **Admin-only unlock path verified end-to-end:** `grant_purchase()` is called from exactly one place (`confirm_payment()`), which is itself gated on `ADMIN_IDS` membership checked against Telegram's own (non-spoofable) `from_user.id`. No user-supplied text ever decides who gets unlocked — the buyer's ID is captured once, at request creation, from their own tap.
- **Race-condition close on double-confirm:** `pending.pop()` happens before any side effects in `confirm_payment`/`reject_payment`, so a second near-simultaneous admin tap sees "already handled" instead of double-processing.
- Bot runs with default `concurrent_updates=False` (not overridden anywhere), so `python-telegram-bot` processes updates one at a time — no true multi-request race condition within the bot process itself on top of the above.

### Known bugs fixed during development
1. **Handler order bug:** generic text handler (library letter search) was registered before the ticket `ConversationHandler`, causing ticket answers to be silently swallowed. Fixed by moving `ticket_conv` registration first.
2. **`PTBUserWarning` about `per_message=False`:** harmless warning caused by mixing a `CommandHandler` and a `CallbackQueryHandler` as entry points for the same `ConversationHandler`. Resolved by explicitly setting `per_message=False` (already the default, but stated explicitly to silence the warning).
3. **`upi://` URL button crash:** `telegram.error.BadRequest: unsupported url protocol` — inline keyboard URL buttons only accept `http(s)://` or `tg://`. Fixed by replacing the button with a QR code image of the same UPI deep link.

### Not yet built / discussed as future polish
- **Sequential ticket numbering** (currently tickets are identifiable only by the user's Telegram ID/name in the summary, not a clean "#14"-style counter). Note: the *payment* system now has sequential Request IDs (`001`, `002`, ...) — the same pattern could be reused for tickets.
- **Admin replying to tickets through the bot itself** (currently an admin would message the user manually, outside the bot's tracked flow).
- **GitHub upload for this project** — same pattern as the Spotify project (move secrets to `config.py` ✅ already done from the start this time, add `.gitignore` covering `config.py`, `known_groups.json`, `purchases.json`, `pending_payments.json`, `__pycache__/`, then `git init` → `add` → `commit` → `push`). Not yet actually performed for this repo.
- **Deploying to Railway** for 24/7 uptime — discussed as the next step once local testing is fully done, not yet started.
- **Real library data** — user plans to provide their actual keyword/link (and optionally price) list later; current data is placeholder (Google, Wikipedia, GitHub, etc. + 2 example paid items).

---

## How to use this file
Paste this whole file at the start of a new chat to skip re-explaining background. Update it as steps get completed — e.g., once real library data is in, once GitHub push is done, once Railway deployment is live, once ticket numbering is added, etc.
