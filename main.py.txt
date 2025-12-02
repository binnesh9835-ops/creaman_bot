# main.py
import os
import re
import time
import sqlite3
import logging
from datetime import datetime
from functools import wraps

from telegram import (
    Update,
    InlineKeyboardButton,
    InlineKeyboardMarkup,
    ParseMode,
)
from telegram.ext import (
    ApplicationBuilder,
    ContextTypes,
    CommandHandler,
    MessageHandler,
    CallbackQueryHandler,
    filters,
)

# Load config (expects env vars; config.py optional)
try:
    from config import BOT_TOKEN, MAIN_CHANNEL_ID, ADMIN_GROUP_ID, ADMIN_USER_IDS, MIN_FOLLOWERS_DEFAULT
except Exception:
    BOT_TOKEN = os.getenv("8584891759:AAGds400yEwDwk8LrqwiXLVyB5LxaTdMkrE")
    MAIN_CHANNEL_ID = int(os.getenv("MAIN_CHANNEL_ID", "-1002863809955"))
    ADMIN_GROUP_ID = int(os.getenv("ADMIN_GROUP_ID", "-4992382090"))
    ADMIN_USER_IDS = [int(x) for x in os.getenv("ADMIN_USER_IDS", "5119859581").split(",") if x.strip()]
    MIN_FOLLOWERS_DEFAULT = int(os.getenv("MIN_FOLLOWERS_DEFAULT", "1000"))

if not BOT_TOKEN:
    raise RuntimeError("BOT_TOKEN not set in environment or config.py")

# Logging
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO
)
logger = logging.getLogger(__name__)

# DB (SQLite for start)
DB_PATH = "creaman.db"
conn = sqlite3.connect(DB_PATH, check_same_thread=False)
cur = conn.cursor()

def init_db():
    cur.execute('''CREATE TABLE IF NOT EXISTS users (
        user_id INTEGER PRIMARY KEY,
        tg_username TEXT,
        role TEXT,
        name TEXT,
        instagram_username TEXT,
        followers INTEGER,
        page_link TEXT,
        upi TEXT,
        bank_details TEXT,
        manager_name TEXT,
        contact_number TEXT,
        weekly_target INTEGER,
        registered_on TEXT
    )''')
    cur.execute('''CREATE TABLE IF NOT EXISTS admin_settings (
        key TEXT PRIMARY KEY,
        value TEXT
    )''')
    cur.execute('''CREATE TABLE IF NOT EXISTS weeks (
        week_id INTEGER PRIMARY KEY,
        start_ts INTEGER,
        end_ts INTEGER,
        status TEXT
    )''')
    cur.execute('''CREATE TABLE IF NOT EXISTS views (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        week_id INTEGER,
        username TEXT,
        owner_user_id INTEGER,
        views INTEGER,
        paid INTEGER DEFAULT 0,
        payment_amount REAL DEFAULT 0,
        paid_on INTEGER,
        note TEXT
    )''')
    cur.execute('''CREATE TABLE IF NOT EXISTS forwarded_map (
        admin_message_id INTEGER PRIMARY KEY,
        orig_chat_id INTEGER,
        orig_message_id INTEGER,
        orig_user_id INTEGER
    )''')
    conn.commit()

init_db()

# admin helper decorator
def admin_only(func):
    @wraps(func)
    async def wrapper(update: Update, context: ContextTypes.DEFAULT_TYPE, *args, **kwargs):
        user = update.effective_user
        if not user or user.id not in ADMIN_USER_IDS:
            if update.message:
                await update.message.reply_text("❌ You are not authorized to run this command.")
            elif update.callback_query:
                await update.callback_query.answer("Unauthorized")
            return
        return await func(update, context, *args, **kwargs)
    return wrapper

# admin settings helpers
def set_setting(key, value):
    cur.execute("INSERT OR REPLACE INTO admin_settings(key, value) VALUES (?, ?)", (key, str(value)))
    conn.commit()

def get_setting(key, default=None):
    cur.execute("SELECT value FROM admin_settings WHERE key=?", (key,))
    r = cur.fetchone()
    if r:
        return r[0]
    return default

# init defaults if not set
if get_setting('creator_price_per_million') is None:
    set_setting('creator_price_per_million', '600')
if get_setting('manager_price_per_million') is None:
    set_setting('manager_price_per_million', '700')
if get_setting('min_followers') is None:
    set_setting('min_followers', str(MIN_FOLLOWERS_DEFAULT))
if get_setting('logo_file_id') is None:
    set_setting('logo_file_id', '')
if get_setting('bio_text') is None:
    set_setting('bio_text', '')
if get_setting('rules_text') is None:
    set_setting('rules_text', '')

# week helpers
def current_week():
    cur.execute("SELECT week_id, start_ts, end_ts, status FROM weeks WHERE status='active' ORDER BY week_id DESC LIMIT 1")
    r = cur.fetchone()
    if r:
        return {'week_id': r[0], 'start_ts': r[1], 'end_ts': r[2], 'status': r[3]}
    return None

def create_week(week_id:int):
    now = int(time.time())
    cur.execute("INSERT INTO weeks(week_id, start_ts, end_ts, status) VALUES (?, ?, ?, ?)", (week_id, now, 0, 'active'))
    conn.commit()

def end_week(week_id:int):
    end_ts = int(time.time())
    cur.execute("UPDATE weeks SET end_ts=?, status='ended' WHERE week_id=?", (end_ts, week_id))
    conn.commit()

def archive_week(week_id:int):
    cur.execute("UPDATE weeks SET status='cleared' WHERE week_id=?", (week_id,))
    conn.commit()

# parsing admin view lines
views_pattern = re.compile(r'^\s*(@?[A-Za-z0-9_+-]+)\s*-\s*([0-9,]+)\s*$')

# --------- Handlers ----------
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    keyboard = [
        [InlineKeyboardButton("Join Channel", url=f"https://t.me/{str(MAIN_CHANNEL_ID).lstrip('-100')}")],
        [InlineKeyboardButton("Continue", callback_data="continue")]
    ]
    text = ("🌐 *Welcome to CreaManNetwork*\n"
            "A structured, premium workspace for creators & managers who value clarity, performance, and consistent earnings.\n\n"
            "Before we set up your panel, you must complete a quick verification to keep the system clean, secure, and spam-free.")
    await update.message.reply_text(text, reply_markup=InlineKeyboardMarkup(keyboard), parse_mode=ParseMode.MARKDOWN)

async def button_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    if query.data == "continue":
        user = query.from_user
        try:
            # verify membership
            member = await context.bot.get_chat_member(chat_id=MAIN_CHANNEL_ID, user_id=user.id)
            keyboard = [
                [InlineKeyboardButton("I am a Creator", callback_data="role_creator")],
                [InlineKeyboardButton("I am a Manager", callback_data="role_manager")],
                [InlineKeyboardButton("Switch Role Later", callback_data="role_later")]
            ]
            text = "✅ Verification Successful.\nPlease choose your role to continue:\n1) Creator\n2) Manager\n\nYou can change this role later."
            await query.edit_message_text(text, reply_markup=InlineKeyboardMarkup(keyboard))
        except Exception:
            await query.edit_message_text("⚠️ Please join the official channel first and then press Continue. (Bot must be admin in the channel to auto-verify.)")

# role selection
async def role_creator_start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    context.user_data['role'] = 'creator'
    text = ("Great. You have selected: *Creator* ✔️\n\n"
            "To set up your creator panel, please submit the following details one by one.\n\n"
            "1) *Instagram username* (eg. @username)\n2) *Follower count* (numbers only)\n3) *Instagram page link* (public link)\n4) *UPI ID or Bank details* (for payouts)\n\nPlease send your Instagram username now.")
    await query.edit_message_text(text, parse_mode=ParseMode.MARKDOWN)

async def role_manager_start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    context.user_data['role'] = 'manager'
    text = ("Great. You have selected: *Manager* ✔️\n\n"
            "To set up your manager profile, please submit the following details one by one.\n\n"
            "1) *Agency / Media Name*\n2) *Contact number*\n3) *Agency Instagram page link*\n4) *UPI ID or Bank details*\n5) *Weekly views target* (numbers, e.g. 50000000)\n\nPlease send your Agency / Media Name now.")
    await query.edit_message_text(text, parse_mode=ParseMode.MARKDOWN)

async def role_later(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    await query.edit_message_text("Role selection skipped. You can choose later using /start.")

# details collector
async def collect_details(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    uid = user.id
    role = context.user_data.get('role')
    text = (update.message.text or "").strip()
    step = context.user_data.get('step', 0)

    if role == 'creator':
        if step == 0:
            context.user_data['instagram_username'] = text
            context.user_data['step'] = 1
            await update.message.reply_text("Send your follower count (numbers only). Example: 1200")
            return
        if step == 1:
            try:
                followers = int(text.replace(',', '').strip())
            except:
                await update.message.reply_text("Please send follower count as number only. Example: 2400")
                return
            context.user_data['followers'] = followers
            minf = int(get_setting('min_followers', MIN_FOLLOWERS_DEFAULT))
            if followers < minf:
                await update.message.reply_text(f"⚠️ Minimum followers requirement: {minf}. Your profile may be rejected. Send page link (we will verify).")
            context.user_data['step'] = 2
            await update.message.reply_text("Send your Instagram page link (public profile). Example: https://instagram.com/yourpage")
            return
        if step == 2:
            context.user_data['page_link'] = text
            context.user_data['step'] = 3
            await update.message.reply_text("Send your UPI ID or Bank details for payouts. Example: example@upi  OR  Account Name | Account Number | IFSC | Bank Name")
            return
        if step == 3:
            upi_or_bank = text
            instagram_username = context.user_data.get('instagram_username')
            followers = context.user_data.get('followers')
            page_link = context.user_data.get('page_link')
            now = datetime.utcnow().isoformat()
            cur.execute('INSERT OR REPLACE INTO users(user_id, tg_username, role, instagram_username, followers, page_link, upi, registered_on) VALUES (?, ?, ?, ?, ?, ?, ?, ?)',
                        (uid, user.username or '', 'creator', instagram_username, followers, page_link, upi_or_bank, now))
            conn.commit()
            await update.message.reply_text("Thank you. Your details have been submitted. Please wait while we verify your profile. This usually takes a few seconds.")
            context.user_data.clear()
            # forward verification to admin group
            msg = (f"[VERIFICATION] Creator registration by @{user.username or user.id}\nInsta: {instagram_username}\n"
                   f"Followers: {followers}\nPage: {page_link}\nUPI/Bank: {upi_or_bank}")
            await context.bot.send_message(ADMIN_GROUP_ID, msg)
            return

    elif role == 'manager':
        if step == 0:
            context.user_data['agency_name'] = text
            context.user_data['step'] = 1
            await update.message.reply_text("Send your contact number as it is (e.g. 9876543210).")
            return
        if step == 1:
            context.user_data['contact_number'] = text
            context.user_data['step'] = 2
            await update.message.reply_text("Send your agency Instagram page link (public).")
            return
        if step == 2:
            context.user_data['agency_page_link'] = text
            context.user_data['step'] = 3
            await update.message.reply_text("Send your UPI ID or Bank details for payouts.")
            return
        if step == 3:
            context.user_data['upi_bank'] = text
            context.user_data['step'] = 4
            await update.message.reply_text("Send your weekly views target (numbers only). Example: 50000000")
            return
        if step == 4:
            try:
                target = int(text.replace(',', '').strip())
            except:
                await update.message.reply_text("Please send a numeric weekly target like 50000000")
                return
            agency_name = context.user_data.get('agency_name')
            contact_number = context.user_data.get('contact_number')
            page_link = context.user_data.get('agency_page_link')
            upi_bank = context.user_data.get('upi_bank')
            now = datetime.utcnow().isoformat()
            # storing manager in users table; role=manager
            cur.execute('INSERT OR REPLACE INTO users(user_id, tg_username, role, name, manager_name, contact_number, page_link, upi, weekly_target, registered_on) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)',
                        (uid, user.username or '', 'manager', agency_name, agency_name, contact_number, page_link, upi_bank, target, now))
            conn.commit()
            await update.message.reply_text("Thank you. Your details have been submitted. Please wait while admins verify your manager profile.")
            await context.bot.send_message(ADMIN_GROUP_ID, (f"[VERIFICATION] Manager registration by @{user.username or user.id}\nAgency: {agency_name}\n"
                                                            f"Contact: {contact_number}\nPage: {page_link}\nUPI/Bank: {upi_bank}\nWeekly target: {target}"))
            context.user_data.clear()
            return

    else:
        await update.message.reply_text("Please press Continue first and choose your role.")
        return

# ---------- Admin Commands ----------
@admin_only
async def setlogo_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Send the logo media (photo/video) now. Bot will store the first received file as logo.")

@admin_only
async def setbio_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = " ".join(context.args) if context.args else ""
    if not text:
        await update.message.reply_text("Usage: /setbio <bio text>")
        return
    set_setting('bio_text', text)
    await update.message.reply_text("✔️ Bio updated successfully.")

@admin_only
async def setrules_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = " ".join(context.args) if context.args else ""
    if not text:
        await update.message.reply_text("Usage: /setrules <rules text>")
        return
    set_setting('rules_text', text)
    await update.message.reply_text("✔️ Rules updated successfully.")

@admin_only
async def setfollowers_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.args:
        await update.message.reply_text("Usage: /setfollowers <number>")
        return
    try:
        num = int(context.args[0])
    except:
        await update.message.reply_text("Please provide numeric value.")
        return
    set_setting('min_followers', str(num))
    await update.message.reply_text(f"✔ Minimum followers updated to {num}.")

@admin_only
async def setprice_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if len(context.args) < 2:
        await update.message.reply_text("Usage: /setprice creators 600  OR  /setprice managers 700")
        return
    who = context.args[0].lower()
    try:
        val = int(context.args[1])
    except:
        await update.message.reply_text("Price must be integer like 600 (meaning ₹0.0006 per view).")
        return
    if who.startswith('creator'):
        set_setting('creator_price_per_million', str(val))
        await update.message.reply_text(f"✔ Creator price updated: {val} per million => rate per view = {val/1000000:.7f}")
    elif who.startswith('manager'):
        set_setting('manager_price_per_million', str(val))
        await update.message.reply_text(f"✔ Manager price updated: {val} per million => rate per view = {val/1000000:.7f}")
    else:
        await update.message.reply_text("First arg must be 'creators' or 'managers'.")

@admin_only
async def week_start_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.args:
        await update.message.reply_text("Usage: /weekstart <week_number>")
        return
    try:
        wk = int(context.args[0])
    except:
        await update.message.reply_text("Please provide week number like /weekstart 1")
        return
    create_week(wk)
    await context.bot.send_message(ADMIN_GROUP_ID, f"🟢 Week {wk} has started.")
    cur.execute("SELECT user_id FROM users")
    rows = cur.fetchall()
    for r in rows:
        try:
            await context.bot.send_message(r[0], f"🔔 Week {wk} has started. Your views will be counted this week (Mon-Fri).")
        except:
            pass
    await update.message.reply_text(f"Week {wk} started and notifications sent.")

@admin_only
async def week_end_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.args:
        await update.message.reply_text("Usage: /weekend <week_number>")
        return
    try:
        wk = int(context.args[0])
    except:
        await update.message.reply_text("Provide numeric week")
        return
    end_week(wk)
    await context.bot.send_message(ADMIN_GROUP_ID, f"🔴 Week {wk} has ended. No further view uploads will be accepted for this week.")
    await update.message.reply_text(f"Week {wk} ended.")

@admin_only
async def week_payment_details_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.args:
        await update.message.reply_text("Usage: /paymentdetails <week_number>")
        return
    wk = int(context.args[0])
    cur.execute("SELECT username, owner_user_id, views, paid, payment_amount FROM views WHERE week_id=? AND paid=0", (wk,))
    rows = cur.fetchall()
    if not rows:
        await update.message.reply_text(f"No unpaid entries for week {wk}.")
        return
    msg_lines = ["Unpaid entries for week %d:" % wk]
    for r in rows:
        uname, owner_id, v, paid, amt = r
        cur.execute("SELECT upi, page_link FROM users WHERE instagram_username=?", (uname,))
        u = cur.fetchone()
        upi = u[0] if u else "N/A"
        page = u[1] if u else "N/A"
        msg_lines.append(f"{uname}\nPage: {page}\nUPI: {upi}\nViews: {v}\nAmount: {amt:.2f}\n---")
    await update.message.reply_text("\n".join(msg_lines))

@admin_only
async def mark_paid_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if len(context.args) < 3:
        await update.message.reply_text("Usage: /markpaid <username> <amount> <week>")
        return
    uname = context.args[0].lstrip('@')
    try:
        amount = float(context.args[1])
        wk = int(context.args[2])
    except:
        await update.message.reply_text("Provide numeric amount and week.")
        return
    cur.execute("UPDATE views SET paid=1, payment_amount=?, paid_on=? WHERE week_id=? AND username=?", (amount, int(time.time()), wk, uname))
    conn.commit()
    await update.message.reply_text(f"Marked {uname} as paid for week {wk}.")
    cur.execute("SELECT user_id, upi FROM users WHERE instagram_username=?", (uname,))
    u = cur.fetchone()
    if u:
        uid = u[0]
        try:
            await context.bot.send_message(uid, f"💸 Your payment for Week {wk} of ₹{amount:.2f} has been completed. Please send feedback if you'd like.")
        except Exception:
            logger.exception("notify pay error")

@admin_only
async def handle_logo_upload_admin(update: Update, context: ContextTypes.DEFAULT_TYPE):
    # admin posted a media in admin group; bot saves first media from admins after /setlogo prompt
    if update.effective_chat.id != ADMIN_GROUP_ID:
        return
    f = None
    if update.message.photo:
        f = update.message.photo[-1].file_id
    elif update.message.video:
        f = update.message.video.file_id
    if f:
        set_setting('logo_file_id', f)
        await update.message.reply_text("✔ Logo saved.")
    else:
        await update.message.reply_text("Please send a photo or video as logo.")

@admin_only
async def broadcast_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = " ".join(context.args) if context.args else (update.message.reply_to_message.text if update.message.reply_to_message else "")
    if not text:
        await update.message.reply_text("Usage: /broadcast <message> (or reply to a message with /broadcast)")
        return
    cur.execute("SELECT user_id FROM users")
    rows = cur.fetchall()
    count = 0
    for r in rows:
        try:
            await context.bot.send_message(r[0], f"📢 Broadcast:\n\n{text}")
            count += 1
        except Exception:
            pass
    await update.message.reply_text(f"Broadcast sent to {count} users.")

@admin_only
async def week_clear_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.args:
        await update.message.reply_text("Usage: /weekclear <week_number>")
        return
    wk = int(context.args[0])
    archive_week(wk)
    await update.message.reply_text(f"Week {wk} cleared (archived).")

@admin_only
async def pending_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.args:
        await update.message.reply_text("Usage: /pending <week_number>")
        return
    wk = int(context.args[0])
    cur.execute("SELECT username, views, payment_amount FROM views WHERE week_id=? AND paid=0", (wk,))
    rows = cur.fetchall()
    if not rows:
        await update.message.reply_text("No pending payments.")
        return
    out = []
    for u,v,a in rows:
        cur.execute("SELECT upi, page_link FROM users WHERE instagram_username=?", (u,))
        r = cur.fetchone()
        upi = r[0] if r else "N/A"
        page = r[1] if r else "N/A"
        out.append(f"{u}\nPage: {page}\nUPI: {upi}\nViews: {v}\nAmount: {a:.2f}\n---")
    await update.message.reply_text("\n".join(out))

# forwarding mapping helper
def store_forward_mapping(admin_message_id, orig_chat_id, orig_message_id, orig_user_id):
    cur.execute("INSERT OR REPLACE INTO forwarded_map(admin_message_id, orig_chat_id, orig_message_id, orig_user_id) VALUES (?, ?, ?, ?)",
                (admin_message_id, orig_chat_id, orig_message_id, orig_user_id))
    conn.commit()

# when forwarding a user message to admin group
async def forward_to_admin_group(bot, chat_id, message_id, orig_user_id):
    try:
        fwd = await bot.forward_message(chat_id=ADMIN_GROUP_ID, from_chat_id=chat_id, message_id=message_id)
        store_forward_mapping(fwd.message_id, chat_id, message_id, orig_user_id)
    except Exception:
        logger.exception("forward fail")

# admin group message handler: parse view lines and relay replies
async def admin_group_message_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    # only run in admin group
    if update.effective_chat.id != ADMIN_GROUP_ID:
        return
    text = update.message.text or ""
    text = text.strip()
    # parse @username - views
    m = views_pattern.match(text)
    if m:
        name = m.group(1).lstrip('@')
        views_str = m.group(2).replace(',', '')
        try:
            views = int(views_str)
        except:
            await update.message.reply_text("Could not parse views number.")
            return
        wk = current_week()
        if not wk:
            await update.message.reply_text("No active week found. Use /weekstart first.")
            return
        week_id = wk['week_id']
        # insert views record
        # owner lookup
        cur.execute("SELECT user_id, role FROM users WHERE instagram_username=?", (name,))
        rr = cur.fetchone()
        owner_id = rr[0] if rr else None
        role = rr[1] if rr else None
        cur.execute("INSERT INTO views(week_id, username, owner_user_id, views, paid) VALUES (?, ?, ?, ?, 0)", (week_id, name, owner_id, views))
        conn.commit()
        # calc payment
        per_m = int(get_setting('manager_price_per_million')) if role=='manager' else int(get_setting('creator_price_per_million'))
        rate = per_m / 1000000.0
        amount = views * rate
        cur.execute("UPDATE views SET payment_amount=? WHERE week_id=? AND username=?", (amount, week_id, name))
        conn.commit()
        # notify owner
        if owner_id:
            try:
                await context.bot.send_message(owner_id, (f"📊 Weekly Views Update\n\nYour total views for this week: {views}\n"
                                                         f"Payable: ₹{amount:.2f}\n(Placed as PENDING)"))
            except Exception:
                logger.info("Could not notify owner")
        await update.message.reply_text(f"Recorded {name} -> {views} views, payable ₹{amount:.2f}")
        return

    # if admin replies to a forwarded message, relay back to original user
    if update.message.reply_to_message:
        replied = update.message.reply_to_message
        admin_msg_id = replied.message_id
        cur.execute("SELECT orig_chat_id, orig_message_id, orig_user_id FROM forwarded_map WHERE admin_message_id=?", (admin_msg_id,))
        mrow = cur.fetchone()
        if mrow:
            orig_chat_id, orig_msg_id, orig_user_id = mrow
            # relay admin's reply text to original user (do not reveal admin identity)
            text = update.message.text or ""
            if text:
                try:
                    await context.bot.send_message(orig_user_id, f"Response from review team:\n{text}")
                    await update.message.reply_text("Relayed to original user.")
                except Exception:
                    logger.exception("relay failed")
            return

# creator / manager private message handler
async def private_message_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    uid = user.id
    text = update.message.text or ""
    # find role
    cur.execute("SELECT role FROM users WHERE user_id=?", (uid,))
    r = cur.fetchone()
    role = r[0] if r else None
    # creator logic
    if role == 'creator':
        wd = datetime.utcnow().weekday()
        # weekend: block reel submissions
        if wd >= 5:
            if 'http' in text or 'reel' in text.lower():
                await update.message.reply_text("⚠️ Work Stopped (Weekend Notice)\nReel submissions are closed on Saturdays & Sundays. Please submit from Monday.")
                return
        # check views command
        if text.lower().strip() in ['check views', 'views', '/myviews', 'myviews']:
            wk = current_week()
            if wk and wk['status']=='active':
                await update.message.reply_text("📊 Views Counting in Progress\nYour weekly views are still being counted by our team. You will receive your final views tonight once counting is completed. Please wait patiently.")
            else:
                await update.message.reply_text("No active week found. Please wait for the next week.")
            return
        # else consider it a reel link / content -> forward to admin group
        if 'http' in text:
            await update.message.reply_text("Reel received. Forwarding to review team for approval.")
            await forward_to_admin_group(context.bot, update.message.chat_id, update.message.message_id, uid)
            return
    # manager logic
    if role == 'manager':
        wd = datetime.utcnow().weekday()
        if wd >= 5:
            # accept only sheets or views statement
            # simple heuristic: if message contains 'sheet' or document, treat as sheet
            if update.message.document or 'sheet' in (update.message.caption or "").lower() or 'views' in text.lower():
                await update.message.reply_text("Please confirm — Is this a *views sheet*? (yes/no)", parse_mode=ParseMode.MARKDOWN)
                await forward_to_admin_group(context.bot, update.message.chat_id, update.message.message_id, uid)
                return
            else:
                await update.message.reply_text("⚠️ Saturday Restriction\nReel links, logo reels, and uploads are not accepted on weekends. Only views sheets are allowed today.")
                return
    # unregistered user prompt
    await update.message.reply_text("If you want to join, press /start and verify. For support, contact admins.")

# feedback handler
async def feedback_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    uid = user.id
    text = (update.message.text or "").strip()
    lowered = text.lower()
    # a simple capture: if message contains 'feedback' or thanks or 'service' etc.
    if lowered.startswith('feedback:') or 'thank' in lowered or 'service' in lowered:
        cur.execute("SELECT instagram_username FROM users WHERE user_id=?", (uid,))
        r = cur.fetchone()
        page = r[0] if r else (user.username or str(uid))
        await context.bot.send_message(ADMIN_GROUP_ID, f"Feedback received\nPage: {page}\nFeedback: {text}")
        await update.message.reply_text("Thanks for your feedback! We appreciate it.")
        return

# error handler
async def error_handler(update: object, context: ContextTypes.DEFAULT_TYPE):
    logger.exception("Exception in update")

# ------------- Bot startup --------------
async def main():
    app = ApplicationBuilder().token(BOT_TOKEN).build()

    # Basic handlers
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CallbackQueryHandler(button_callback, pattern="^continue$"))
    app.add_handler(CallbackQueryHandler(role_creator_start, pattern="role_creator"))
    app.add_handler(CallbackQueryHandler(role_manager_start, pattern="role_manager"))
    app.add_handler(CallbackQueryHandler(role_later, pattern="role_later"))

    # collect details (text)
    app.add_handler(MessageHandler(filters.TEXT & filters.PRIVATE, collect_details))

    # admin commands (only admins will be allowed inside handlers by decorator)
    app.add_handler(CommandHandler("setlogo", setlogo_cmd))
    app.add_handler(CommandHandler("setbio", setbio_cmd))
    app.add_handler(CommandHandler("setrules", setrules_cmd))
    app.add_handler(CommandHandler("setfollowers", setfollowers_cmd))
    app.add_handler(CommandHandler("setprice", setprice_cmd))
    app.add_handler(CommandHandler("weekstart", week_start_cmd))
    app.add_handler(CommandHandler("weekend", week_end_cmd))
    app.add_handler(CommandHandler("paymentdetails", week_payment_details_cmd))
    app.add_handler(CommandHandler("markpaid", mark_paid_cmd))
    app.add_handler(CommandHandler("broadcast", broadcast_cmd))
    app.add_handler(CommandHandler("weekclear", week_clear_cmd))
    app.add_handler(CommandHandler("pending", pending_cmd))

    # admin group specific handlers
    app.add_handler(MessageHandler(filters.Chat(int(ADMIN_GROUP_ID)) & filters.TEXT, admin_group_message_handler))
    app.add_handler(MessageHandler(filters.Chat(int(ADMIN_GROUP_ID)) & (filters.PHOTO | filters.VIDEO), handle_logo_upload_admin))

    # private messages (creator/manager actions + feedback)
    app.add_handler(MessageHandler(filters.PRIVATE & filters.TEXT, private_message_handler))
    app.add_handler(MessageHandler(filters.PRIVATE & filters.TEXT, feedback_handler))

    app.add_error_handler(error_handler)

    logger.info("Starting bot...")
    await app.initialize()
    await app.start()
    await app.updater.start_polling()  # polling
    logger.info("Bot started. Press Ctrl+C to stop.")
    await app.idle()

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
