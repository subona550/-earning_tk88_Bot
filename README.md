https://subona550.github.io/-earning_tk88_Bot/
# Creating GitHub-ready ZIP for earning_tk88_bot with Monetag ad integration.
import os, zipfile, textwrap, json, pathlib
base = "/mnt/data/earning_tk88_bot_package"
os.makedirs(base, exist_ok=True)

# Main bot script with Monetag integration (updated)
main_py = r'''
#!/usr/bin/env python3
"""
earning_tk88_bot.py

Single-file Telegram earning bot with Monetag ad integration (prototype).

Configuration:
- TELEGRAM_TOKEN : bot token
- ADMIN_IDS : space-separated admin numeric IDs (optional)
- AD_URL : publicly hosted ad page URL (see ad_page.html in repo; host it on GitHub Pages)
- AD_REWARD : integer reward per ad (optional)
- REFERRAL_BONUS : referral bonus (optional)
- MIN_WITHDRAW : minimum withdraw amount (optional)

Notes:
- Host ad_page.html (included) on a public URL (e.g., GitHub Pages).
- The ad page includes the Monetag script; after watching the ad, user clicks "Claim in Telegram"
  which deep-links back to the bot: t.me/<bot_username>?start=claimad_<token>
- When the bot receives /start claimad_<token>, it will credit the user who opened the link.
- This is a prototype. Add proper anti-fraud controls and KYC before real payouts.
"""

import os
import sqlite3
import logging
import secrets
from datetime import datetime, timedelta

from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    ApplicationBuilder,
    ContextTypes,
    CommandHandler,
    MessageHandler,
    CallbackQueryHandler,
    filters,
)

# ------------------ CONFIG ------------------
TOKEN = os.getenv("TELEGRAM_TOKEN") or "PASTE_YOUR_BOT_TOKEN_HERE"
DB_PATH = os.getenv("EARNING_DB") or "earning_tk88.db"
BOT_NAME = os.getenv("BOT_USERNAME") or "@earning_tk88_Bot"
ADMIN_IDS = set(map(int, os.getenv("ADMIN_IDS", "").split())) if os.getenv("ADMIN_IDS") else {123456789}
AD_URL = os.getenv("AD_URL") or "https://yourdomain.github.io/ad_page.html"
AD_REWARD = int(os.getenv("AD_REWARD", "5"))
REFERRAL_BONUS = int(os.getenv("REFERRAL_BONUS", "10"))
MIN_WITHDRAW = int(os.getenv("MIN_WITHDRAW", "100"))
# ---------------------------------------------

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# ---------- Database helpers ----------
def init_db(path: str = DB_PATH):
    conn = sqlite3.connect(path, check_same_thread=False)
    cur = conn.cursor()
    cur.execute("""
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY,
            telegram_id INTEGER UNIQUE,
            username TEXT,
            balance INTEGER DEFAULT 0,
            referrer_code TEXT,
            own_ref_code TEXT UNIQUE,
            referrals_count INTEGER DEFAULT 0,
            created_at TEXT
        )
    """)
    cur.execute("""
        CREATE TABLE IF NOT EXISTS transactions (
            id INTEGER PRIMARY KEY,
            user_id INTEGER,
            type TEXT,
            amount INTEGER,
            note TEXT,
            created_at TEXT
        )
    """)
    cur.execute("""
        CREATE TABLE IF NOT EXISTS withdraws (
            id INTEGER PRIMARY KEY,
            user_id INTEGER,
            amount INTEGER,
            status TEXT DEFAULT 'pending',
            created_at TEXT
        )
    """)
    conn.commit()
    return conn

DB = init_db()

def now_iso():
    return datetime.utcnow().isoformat()

def get_user_by_telegram(tg_id: int):
    cur = DB.cursor()
    cur.execute("SELECT * FROM users WHERE telegram_id = ?", (tg_id,))
    return cur.fetchone()

def get_user_by_refcode(code: str):
    cur = DB.cursor()
    cur.execute("SELECT * FROM users WHERE own_ref_code = ?", (code,))
    return cur.fetchone()

def create_user(tg_id: int, username: str=None, refcode: str=None):
    cur = DB.cursor()
    own_code = secrets.token_urlsafe(6)
    created = now_iso()
    cur.execute(
        "INSERT OR IGNORE INTO users (telegram_id, username, balance, referrer_code, own_ref_code, created_at) VALUES (?, ?, ?, ?, ?, ?)",
        (tg_id, username or "", 0, refcode, own_code, created),
    )
    DB.commit()
    # handle referral bonus
    if refcode:
        cur.execute("SELECT id FROM users WHERE own_ref_code = ?", (refcode,))
        ref = cur.fetchone()
        if ref:
            ref_id = ref[0]
            cur.execute("UPDATE users SET balance = balance + ?, referrals_count = referrals_count + 1 WHERE id = ?", (REFERRAL_BONUS, ref_id))
            cur.execute("INSERT INTO transactions (user_id, type, amount, note, created_at) VALUES (?, 'referral_bonus', ?, ?, ?)", (ref_id, REFERRAL_BONUS, f'referral of {tg_id}', now_iso()))
            DB.commit()
    cur.execute("SELECT * FROM users WHERE telegram_id = ?", (tg_id,))
    return cur.fetchone()

def change_balance(user_id: int, amount: int, ttype: str, note: str = ""):
    cur = DB.cursor()
    cur.execute("UPDATE users SET balance = balance + ? WHERE id = ?", (amount, user_id))
    cur.execute("INSERT INTO transactions (user_id, type, amount, note, created_at) VALUES (?, ?, ?, ?, ?)", (user_id, ttype, amount, note, now_iso()))
    DB.commit()

# ---------- Bot handlers ----------
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    args = context.args
    tg_user = update.effective_user
    # handle claim from ad: /start claimad_<token>
    if args and args[0].startswith("claimad_"):
        # credit the opener
        user = get_user_by_telegram(tg_user.id)
        if not user:
            create_user(tg_user.id, tg_user.username)
            user = get_user_by_telegram(tg_user.id)
        # prevent rapid double-claim: check last ad_reward transaction in last 2 minutes
        cur = DB.cursor()
        cur.execute("SELECT created_at FROM transactions WHERE user_id = ? AND type = 'ad_reward' ORDER BY id DESC LIMIT 1", (user[0],))
        row = cur.fetchone()
        allow = True
        if row:
            last_time = datetime.fromisoformat(row[0])
            if datetime.utcnow() - last_time < timedelta(seconds=120):
                allow = False
        if allow:
            change_balance(user[0], AD_REWARD, "ad_reward", f"Monetag ad claim")
            await update.message.reply_text(f"ধন্যবাদ! তুমি অ্যাড দেখার জন্য {AD_REWARD} পেয়েছো। তোমার নতুন balance দেখতে /balance লিখো।")
        else:
            await update.message.reply_text("দুঃখিত, তুমি সাম্প্রতিকভাবে অ্যাড ক্লেইম করেছো — একটু অপেক্ষা করো।")
        return

    refcode = args[0] if args else None
    existing = get_user_by_telegram(tg_user.id)
    if not existing:
        create_user(tg_user.id, tg_user.username, refcode)
    user = get_user_by_telegram(tg_user.id)
    own_ref = user[5]  # own_ref_code
    balance = user[3]
    text = (
        f"স্বাগতম {tg_user.first_name}!\n\n"
        f"তোমার balance: {balance}\n"
        f"তোমার referral link: t.me/{context.bot.username}?start={own_ref}\n\n"
        "কমান্ডস:\n"
        "/balance - ব্যালান্স দেখো\n"
        "/watchad - অ্যাড দেখে আয় কর\n"
        "/refer - তোমার referral link নাও\n"
        "/withdraw <amount> - উইথড্র করারের অনুরোধ\n"
    )
    await update.message.reply_text(text)

async def balance_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = get_user_by_telegram(update.effective_user.id)
    if not user:
        create_user(update.effective_user.id, update.effective_user.username)
        user = get_user_by_telegram(update.effective_user.id)
    bal = user[3]
    await update.message.reply_text(f"তোমার balance: {bal}")

async def refer_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = get_user_by_telegram(update.effective_user.id)
    if not user:
        create_user(update.effective_user.id, update.effective_user.username)
        user = get_user_by_telegram(update.effective_user.id)
    own = user[5]
    await update.message.reply_text(f"তোমার referral link: t.me/{context.bot.username}?start={own}")

async def watchad_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = get_user_by_telegram(update.effective_user.id)
    if not user:
        create_user(update.effective_user.id, update.effective_user.username)
        user = get_user_by_telegram(update.effective_user.id)
    own = user[5]
    # generate URL to AD page with token param
    ad_url = f"{AD_URL}?token={own}"
    kb = InlineKeyboardMarkup.from_button(InlineKeyboardButton("Watch monetag ad (open)", url=ad_url))
    await update.message.reply_text("নিচের বাটনে চাপ দিয়ে অ্যাড দেখো — দেখার পরে \"Claim in Telegram\" চাপো।", reply_markup=kb)

async def withdraw_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = get_user_by_telegram(update.effective_user.id)
    if not user:
        create_user(update.effective_user.id, update.effective_user.username)
        user = get_user_by_telegram(update.effective_user.id)
    args = context.args
    if not args:
        await update.message.reply_text("ব্যবহার: /withdraw <amount> — উদাহরণ: /withdraw 150")
        return
    try:
        amount = int(args[0])
    except ValueError:
        await update.message.reply_text("অনুগ্রহ করে সংখ্যা লিখুন।")
        return
    if amount < MIN_WITHDRAW:
        await update.message.reply_text(f"কমপক্ষে {MIN_WITHDRAW} হওয়ার পরই উইথড্র করতে পারবে।")
        return
    if user[3] < amount:
        await update.message.reply_text("তোমার ব্যালান্স পর্যাপ্ত নয়।")
        return
    cur = DB.cursor()
    cur.execute("INSERT INTO withdraws (user_id, amount, status, created_at) VALUES (?, ?, 'pending', ?)", (user[0], amount, now_iso()))
    DB.commit()
    await update.message.reply_text("তোমার উইথড্র রিকোয়েস্ট জমা হয়েছে — অ্যাডমিন অনুমোদন করলেই পেমেন্ট হবে।")
    for aid in ADMIN_IDS:
        try:
            await context.bot.send_message(aid, f"নতুন withdraw request: user {user[1]} amount {amount}")
        except Exception:
            logger.exception("Failed to notify admin")

async def admin_withdraws(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id not in ADMIN_IDS:
        await update.message.reply_text("এডমিন কমান্ড।")
        return
    cur = DB.cursor()
    cur.execute("SELECT w.id, u.telegram_id, u.username, w.amount, w.created_at FROM withdraws w JOIN users u ON w.user_id = u.id WHERE w.status = 'pending'")
    rows = cur.fetchall()
    if not rows:
        await update.message.reply_text("কোনো pending withdraw নেই।")
        return
    text = "Pending withdraws:\n"
    for r in rows:
        wid, tgid, uname, amt, created = r
        text += f"id:{wid} user:{uname or tgid} amount:{amt} at:{created}\n"
    await update.message.reply_text(text)

async def admin_approve(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id not in ADMIN_IDS:
        await update.message.reply_text("এডমিন কমান্ড।")
        return
    args = context.args
    if len(args) < 2:
        await update.message.reply_text("ব্যবহার: /approve <withdraw_id> <approve|decline>")
        return
    wid = int(args[0])
    action = args[1].lower()
    cur = DB.cursor()
    cur.execute("SELECT user_id, amount, status FROM withdraws WHERE id = ?", (wid,))
    row = cur.fetchone()
    if not row:
        await update.message.reply_text("Invalid withdraw id")
        return
    user_id, amount, status = row
    if status != 'pending':
        await update.message.reply_text("Already processed")
        return
    if action == 'approve':
        cur.execute("UPDATE users SET balance = balance - ? WHERE id = ?", (amount, user_id))
        cur.execute("UPDATE withdraws SET status = 'approved' WHERE id = ?", (wid,))
        cur.execute("INSERT INTO transactions (user_id, type, amount, note, created_at) VALUES (?, 'withdraw', ?, ?, ?)", (user_id, -amount, f'withdraw id {wid}', now_iso()))
        DB.commit()
        await update.message.reply_text(f"Withdraw {wid} approved")
    else:
        cur.execute("UPDATE withdraws SET status = 'declined' WHERE id = ?", (wid,))
        DB.commit()
        await update.message.reply_text(f"Withdraw {wid} declined")

async def stats_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id not in ADMIN_IDS:
        await update.message.reply_text("এডমিন কমান্ড।")
        return
    cur = DB.cursor()
    cur.execute("SELECT COUNT(*), SUM(balance) FROM users")
    total_users, total_bal = cur.fetchone()
    cur.execute("SELECT COUNT(*) FROM withdraws WHERE status = 'pending'")
    pending = cur.fetchone()[0]
    await update.message.reply_text(f"Users: {total_users}\nTotal balance: {total_bal or 0}\nPending withdraws: {pending}")

async def unknown(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("আমি বুঝিনি — কমান্ড ব্যবহার করো। /help দেখো।")

def main():
    if TOKEN == "PASTE_YOUR_BOT_TOKEN_HERE":
        print("Please set TELEGRAM_TOKEN environment variable or paste your token in the script.")
        return
    app = ApplicationBuilder().token(TOKEN).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("balance", balance_cmd))
    app.add_handler(CommandHandler("refer", refer_cmd))
    app.add_handler(CommandHandler("watchad", watchad_cmd))
    app.add_handler(CommandHandler("withdraw", withdraw_cmd))

    # admin
    app.add_handler(CommandHandler("pending_withdraws", admin_withdraws))
    app.add_handler(CommandHandler("approve", admin_approve))
    app.add_handler(CommandHandler("stats", stats_cmd))

    app.add_handler(MessageHandler(filters.COMMAND, unknown))

    logger.info("starting bot...")
    app.run_polling()

if __name__ == "__main__":
    main()
'''

# ad_page.html with Monetag script provided by user
ad_html = r'''
<!doctype html>
<html lang="bn">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Monetag Ad — Claim</title>
</head>
<body style="font-family: Arial, Helvetica, sans-serif; max-width:700px; margin:20px auto; padding:20px; text-align:center;">
  <h2>Monetag Ad</h2>
  <p>নিচের বিজ্ঞাপনটি দেখুন। দেখার পরে "Claim in Telegram" চাপুন।</p>

  <!-- Monetag ad script inserted below -->
  <div id="ad_container" style="margin:20px 0;">
    <script src='//libtl.com/sdk.js' data-zone='9978627' data-sdk='show_9978627'></script>
  </div>

  <p style="color:#666; font-size:14px;">দয়া করে সম্পূর্ণরূপে অ্যাডটি দেখুন — তারপর বাটনে চাপুন।</p>

  <!-- The claim button deep-links back to Telegram bot.
       Replace YOUR_BOT_USERNAME with your bot's username or let users click the button which opens Telegram.
       The `token` query param will be appended by the bot when generating the URL. -->
  <a id="claim_btn" href="#" style="display:inline-block; padding:12px 18px; border-radius:8px; background:#2ca5e0; color:white; text-decoration:none;">Claim in Telegram</a>

  <script>
    // parse token param and set Telegram deep link
    function getParam(name){
      const params = new URLSearchParams(window.location.search);
      return params.get(name);
    }
    const token = getParam('token') || 'none';
    const bot = 'YOUR_BOT_USERNAME'; // replace with actual bot username or instruct user to edit this file
    const deep = 'https://t.me/' + bot + '?start=claimad_' + encodeURIComponent(token);
    document.getElementById('claim_btn').href = deep;
  </script>

  <hr style="margin-top:30px; opacity:0.6;">
  <p style="font-size:12px; color:#999;">Prototype ad page. Host this file on GitHub Pages and set AD_URL in the bot config.</p>
</body>
</html>
'''

# README
readme = r'''
# earning_tk88_bot (Premium) — Monetag-enabled

A prototype single-file Telegram earning bot with Monetag ad integration.

## What's included
- `earning_tk88_bot.py` — main bot script.
- `ad_page.html` — simple ad page you can host (contains Monetag script).
- `requirements.txt` — Python dependencies.
- `README.md` — this file.

## How it works (quick)
1. Host `ad_page.html` on a public URL (GitHub Pages recommended).
2. Edit `ad_page.html` and replace `YOUR_BOT_USERNAME` with your bot's username (without @).
3. Set the public URL as `AD_URL` environment variable (or edit the script).
   Example: `https://yourusername.github.io/ad_page.html`
4. Run the bot. When a user runs `/watchad`, bot sends a link to the ad page:
   `https://.../ad_page.html?token=<user_ref_code>`
5. After the user watches the monetag ad there, they click **Claim in Telegram** which
   opens the bot with `?start=claimad_<token>` parameter and the bot credits the opener.

## Setup
1. Create bot with @BotFather and get TELEGRAM_TOKEN.
2. (Optional) Set ADMIN_IDS env var to space-separated admin numeric IDs.
3. Host `ad_page.html` somewhere public (GitHub Pages).
4. Set env vars and run:
```bash
export TELEGRAM_TOKEN="123:ABC..."
export AD_URL="https://yourusername.github.io/ad_page.html"
export ADMIN_IDS="123456789"
python earning_tk88_bot.py
