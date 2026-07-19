# Telegram Multi-Feature Bot

A Telegram bot with four core features: a support ticket system, group
join/leave management, a searchable link library, and a manual UPI
payment flow for unlocking paid library items - no payment gateway,
no fees, just a QR code and admin confirmation.

---
## Features

### 🎫 Ticket System
A step-by-step conversation collects a user's Name/Username, Age,
Contact (optional), Email (optional), and a description of their
request. Once submitted, a formatted summary is sent to every admin
listed in `ADMIN_IDS`. Triggered via `/ticket` or the **Raise a
Ticket** menu button. Users can cancel mid-flow with `/cancel`.

### 👥 Group Management
The bot automatically tracks every group it's added to (name, chat
ID, invite link) via Telegram's `ChatMemberHandler`. `/mygroups` (or
the **My Groups** menu button) shows each tracked group as a button:
- **Leave** if the user is currently in it (removes them via a clean
  ban+immediate-unban, not a punitive ban)
- **Join** (an invite link) if they're not

> **Note:** Telegram doesn't allow a bot to add a user to a group
> automatically - the user has to tap the invite link themselves.
> The bot also needs to be an **admin** in a group to remove other
> members from it.

### 📚 Library Search
A searchable link library backed by `library.xlsx` (columns:
**Keyword**, **Link**, **Price**). Accessible via `/library` or the
**Library** menu button, with two modes:
- **Random** - returns one random entry
- **Word Based** - matches keywords by their *first letter*
  (`.startswith()`, not "contains")

Each result is shown as a button. Free entries (blank/0 price) open
the link directly; paid entries show as locked (`🔒 Keyword (₹Price)`)
and route into the payment flow instead.

### 💳 My Purchases
`/purchases` (or the **My Purchases** menu button) lists every paid
item a user has already unlocked, matched live against the current
library so removed or renamed items don't show stale/broken links.

### 💰 Payments - Manual UPI, No Gateway
Paid library items are unlocked through a fully manual UPI flow
(no Razorpay/Stripe/etc., no transaction fees):

1. Tapping a locked (`🔒`) item generates a payment request and shows
   a scannable UPI QR code plus the admin's UPI ID.
2. The user pays via any UPI app (GPay, PhonePe, Paytm, etc.) and
   sends a screenshot of the payment as proof, right in the chat.
3. The screenshot is forwarded to **every** admin with **Confirm** /
   **Reject** buttons.
4. The admin manually verifies the payment actually arrived (in
   their own UPI app) before tapping Confirm.
5. Confirming permanently unlocks that item for that user and sends
   them the real link. Rejecting notifies the user without unlocking
   anything.

If a user has more than one payment open at once, they're asked to
add the Request ID as the screenshot's caption so the right payment
gets matched.

> **Known limitation:** nothing technically stops a user from tapping
> "I've Paid" without having actually paid - the system relies on an
> admin visually verifying the payment before confirming. This is an
> accepted tradeoff for small, personal-project scale.

> **Future Aspects:** Can be linked to payment providers/vendors to accept worldwide payments or from card, netbanking or even wallet. Apart from this, the payment feature can be automated once payment provider system is set up so that it verifies and accepts payments automatically, allowing fast and easy access to files without admin-verification.

---

## Requirements

- Python 3.10+
- A Telegram bot token (from [@BotFather](https://t.me/BotFather))
- Your own Telegram numeric ID (from [@userinfobot](https://t.me/userinfobot)), used as an admin ID
- A UPI ID to receive payments - UPI is operated in many country nowadays otherwise consider your PayPal for the same.

### Python packages

```bash
pip install python-telegram-bot --upgrade
pip install openpyxl
pip install qrcode
```

---

## Setup

1. **Clone the repo**
   ```bash
   git clone https://github.com/<your-username>/<your-repo>.git
   cd <your-repo>
   ```

2. **Install dependencies** (see above)
> **Note:** This is a native bot system so it will run of your own system. If you want to run it 24/7 then you will need server access. 

3. **Create `config.py`** in the project root (this file is
   git-ignored - never commit it):
   ```python
   BOT_TOKEN = "your-bot-token-from-botfather"
   ADMIN_IDS = [123456789]        # list - supports multiple admins.
   # Want to check for User ID to add as admin? Search '@userinfobot' to get it.
   UPI_ID = "yourname@bank"
   UPI_NAME = "Your Name"
   ```

4. **Create `library.xlsx`** in the project root with three columns
   in row 1: **Keyword**, **Link**, **Price**. Leave Price blank or
   `0` for free entries; any number locks the entry behind payment.

5. **Run the bot:**
   ```bash
   python bot.py
   ```
   Keep the terminal window open - the bot runs via long polling.
   Stop it with `Ctrl+C`.

---

## Project Structure

```
.
├── bot.py                    # Main bot script (see Section Layout below)
├── config.py                 # Secrets/config - NOT committed (see .gitignore)
├── library.xlsx               # Searchable link library - Keyword | Link | Price
├── known_groups.json          # Auto-generated - tracks groups the bot has joined
├── purchases.json             # Auto-generated - tracks unlocked paid items per user
├── pending_payments.json      # Auto-generated - tracks in-flight payment claims
└── .gitignore
```

`known_groups.json`, `purchases.json`, and `pending_payments.json`
are created automatically the first time they're needed - you don't
need to create them by hand.

### `bot.py` section layout
1. Imports
2. Constants (conversation stages, filenames)
3. Data storage helpers (load/save for groups, library, purchases, pending payments - writes are atomic via a temp-file-then-`os.replace()` pattern, so a crash mid-save can't corrupt any of the JSON files)
4. Basic commands - `/start`, `/menu`
5. Ticket system
6. Group management
7. Library search
8. Payments (manual UPI + admin confirmation)
9. Main / handler registration

---

## Commands

| Command | Description |
|---|---|
| `/start` | Confirms the bot is running |
| `/menu` | Shows the main button menu |
| `/ticket` | Starts a new support ticket |
| `/cancel` | Cancels an in-progress ticket |
| `/mygroups` | Shows groups you can join/leave |
| `/library` | Search the link library |
| `/purchases` | Shows items you've already unlocked |

---

## A Note on Handler Order

`python-telegram-bot` runs only the **first matching handler** per
update - there's no fallthrough. The ticket `ConversationHandler` is
registered *before* the generic library letter-search text handler
so that typed ticket answers aren't accidentally swallowed by the
library search logic. If you add new text handlers, register them
with this ordering in mind (see the comment above the final
`app.add_handler(...)` call in `bot.py`).

---

## Roadmap

- [ ] Sequential ticket numbering (e.g. `#014`) instead of identifying tickets only by user ID/name
- [ ] In-bot admin replies to tickets (currently admins reply to users manually, outside the bot)
- [ ] Swap in real library data (current `library.xlsx` is placeholder)
- [ ] Allow more data type (e.g. `photos`, `videos`, etc.) in library data (current `library.xlsx` holds https/http links and their price value `0 for free` & `above for paid`)
- [ ] Optional future upgrade: register for Razorpay (individual/sole proprietor) for automated payment verification instead of manual admin confirmation, if volume justifies it

---

## License

No license needed, this is open source and I took help from AI and other learning materials to make this a project.
