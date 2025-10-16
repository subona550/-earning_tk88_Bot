import telebot
import json
import os

API_TOKEN = 'YOUR_BOT_TOKEN_HERE'
ADMIN_ID = 123456789  # আপনার নিজের Telegram ID বসান

bot = telebot.TeleBot(API_TOKEN)

DATA_FILE = 'data.json'

def load_data():
    if not os.path.exists(DATA_FILE):
        return {}
    with open(DATA_FILE, 'r') as f:
        return json.load(f)

def save_data(data):
    with open(DATA_FILE, 'w') as f:
        json.dump(data, f, indent=4)

users = load_data()

@bot.message_handler(commands=['start'])
def start(message):
    user_id = str(message.chat.id)
    args = message.text.split()

    if user_id not in users:
        users[user_id] = {"balance": 0, "referrals": [], "id": user_id}
        if len(args) > 1:
            referrer_id = args[1]
            if referrer_id != user_id and referrer_id in users:
                if user_id not in users[referrer_id]["referrals"]:
                    users[referrer_id]["referrals"].append(user_id)
                    users[referrer_id]["balance"] += 1

    save_data(users)

    markup = telebot.types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.row('▶️ Watch Ad', '💰 Balance')
    markup.row('📲 Referral Link', '💸 Withdraw')

    bot.send_message(message.chat.id, "👋 স্বাগতম! আমি আপনার রিওয়ার্ড বট!", reply_markup=markup)

@bot.message_handler(func=lambda msg: msg.text == '▶️ Watch Ad')
def watch_ad(message):
    user_id = str(message.chat.id)
    users[user_id]["balance"] += 2
    save_data(users)
    bot.send_message(message.chat.id, "✅ আপনি বিজ্ঞাপন দেখেছেন এবং 2 টাকা পেয়েছেন!")

@bot.message_handler(func=lambda msg: msg.text == '💰 Balance')
def balance(message):
    user_id = str(message.chat.id)
    user = users[user_id]
    bot.send_message(message.chat.id, f"💼 ব্যালেন্স: {user['balance']} টাকা\n👥 রেফারেল: {len(user['referrals'])} জন")

@bot.message_handler(func=lambda msg: msg.text == '📲 Referral Link')
def referral_link(message):
    user_id = str(message.chat.id)
    username = "YOUR_BOT_USERNAME"  # বটের username বসাতে হবে
    link = f"https://t.me/{username}?start={user_id}"
    bot.send_message(message.chat.id, f"🔗 আপনার রেফারেল লিংক:\n{link}")

@bot.message_handler(func=lambda msg: msg.text == '💸 Withdraw')
def withdraw(message):
    user_id = str(message.chat.id)
    user = users[user_id]
    if user['balance'] >= 10:
        user['balance'] -= 10
        save_data(users)
        bot.send_message(message.chat.id, "✅ আপনার উইথড্র রিকোয়েস্ট পাঠানো হয়েছে!")
        bot.send_message(ADMIN_ID, f"💸 Withdraw Request:\nUser ID: {user_id}\nAmount: 10 টাকা")
    else:
        bot.send_message(message.chat.id, "❌ আপনার ব্যালেন্স কম।")

bot.polling()import telebot
import json
import os

API_TOKEN = 'YOUR_BOT_TOKEN_HERE'
ADMIN_ID = 123456789  # আপনার নিজের Telegram ID বসান

bot = telebot.TeleBot(API_TOKEN)

DATA_FILE = 'data.json'

def load_data():
    if not os.path.exists(DATA_FILE):
        return {}
    with open(DATA_FILE, 'r') as f:
        return json.load(f)

def save_data(data):
    with open(DATA_FILE, 'w') as f:
        json.dump(data, f, indent=4)

users = load_data()

@bot.message_handler(commands=['start'])
def start(message):
    user_id = str(message.chat.id)
    args = message.text.split()

    if user_id not in users:
        users[user_id] = {"balance": 0, "referrals": [], "id": user_id}
        if len(args) > 1:
            referrer_id = args[1]
            if referrer_id != user_id and referrer_id in users:
                if user_id not in users[referrer_id]["referrals"]:
                    users[referrer_id]["referrals"].append(user_id)
                    users[referrer_id]["balance"] += 1

    save_data(users)

    markup = telebot.types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.row('▶️ Watch Ad', '💰 Balance')
    markup.row('📲 Referral Link', '💸 Withdraw')

    bot.send_message(message.chat.id, "👋 স্বাগতম! আমি আপনার রিওয়ার্ড বট!", reply_markup=markup)

@bot.message_handler(func=lambda msg: msg.text == '▶️ Watch Ad')
def watch_ad(message):
    user_id = str(message.chat.id)
    users[user_id]["balance"] += 2
    save_data(users)
    bot.send_message(message.chat.id, "✅ আপনি বিজ্ঞাপন দেখেছেন এবং 2 টাকা পেয়েছেন!")

@bot.message_handler(func=lambda msg: msg.text == '💰 Balance')
def balance(message):
    user_id = str(message.chat.id)
    user = users[user_id]
    bot.send_message(message.chat.id, f"💼 ব্যালেন্স: {user['balance']} টাকা\n👥 রেফারেল: {len(user['referrals'])} জন")

@bot.message_handler(func=lambda msg: msg.text == '📲 Referral Link')
def referral_link(message):
    user_id = str(message.chat.id)
    username = "YOUR_BOT_USERNAME"  # বটের username বসাতে হবে
    link = f"https://t.me/{username}?start={user_id}"
    bot.send_message(message.chat.id, f"🔗 আপনার রেফারেল লিংক:\n{link}")

@bot.message_handler(func=lambda msg: msg.text == '💸 Withdraw')
def withdraw(message):
    user_id = str(message.chat.id)
    user = users[user_id]
    if user['balance'] >= 10:
        user['balance'] -= 10
        save_data(users)
        bot.send_message(message.chat.id, "✅ আপনার উইথড্র রিকোয়েস্ট পাঠানো হয়েছে!")
        bot.send_message(ADMIN_ID, f"💸 Withdraw Request:\nUser ID: {user_id}\nAmount: 10 টাকা")
    else:
        bot.send_message(message.chat.id, "❌ আপনার ব্যালেন্স কম।")

bot.polling()
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
