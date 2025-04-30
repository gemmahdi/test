import asyncio
import logging
import pytz
from datetime import datetime
from telegram import Update, ReplyKeyboardMarkup, KeyboardButton
from telegram.ext import (
    Application,
    CommandHandler,
    MessageHandler,
    filters,
    ContextTypes
)
from apscheduler.schedulers.asyncio import AsyncIOScheduler

# تنظیمات لاگ‌گیری
logging.basicConfig(
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    level=logging.INFO
)
logger = logging.getLogger(__name__)

# تنظیمات ربات
TOKEN = "8058230202:AAGKTXWWOgnRNgzaq10JL1vRZKuKlnOyC38"
TIMEZONE = pytz.timezone("Asia/Tehran")

# ساختار ذخیره داده‌ها
study_sessions = {}
active_timers = {}

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """مدیریت دستور /start"""
    keyboard = [
        [KeyboardButton("📖 شروع مطالعه جدید")],
        [KeyboardButton("⏸ توقف مطالعه"), KeyboardButton("🔄 ادامه مطالعه")],
        [KeyboardButton("📊 آمار مطالعه")]
    ]
    reply_markup = ReplyKeyboardMarkup(keyboard, resize_keyboard=True)
    
    await update.message.reply_text(
        "📚 ربات مدیریت زمان مطالعه\n\n"
        "برای شروع مطالعه جدید دکمه مربوطه را انتخاب کنید:",
        reply_markup=reply_markup
    )

async def handle_messages(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """پردازش پیام‌های متنی"""
    text = update.message.text
    user_id = update.message.from_user.id
    
    if text == "📖 شروع مطالعه جدید":
        await ask_subject_name(update)
    elif text == "⏸ توقف مطالعه":
        await pause_study(update, user_id)
    elif text == "🔄 ادامه مطالعه":
        await resume_study(update, user_id)
    elif text == "📊 آمار مطالعه":
        await show_stats(update, user_id)
    elif user_id in active_timers and active_timers[user_id].get('awaiting_subject'):
        await start_new_study_session(update, user_id)

async def ask_subject_name(update: Update) -> None:
    """درخواست نام درس از کاربر"""
    user_id = update.message.from_user.id
    active_timers[user_id] = {'awaiting_subject': True}
    await update.message.reply_text("📝 لطفاً نام درس را وارد کنید:")

async def start_new_study_session(update: Update, user_id: int) -> None:
    """شروع جلسه مطالعه جدید"""
    subject = update.message.text.strip()
    
    if user_id not in study_sessions:
        study_sessions[user_id] = {}
    
    if subject not in study_sessions[user_id]:
        study_sessions[user_id][subject] = {
            'total_seconds': 0,
            'sessions': []
        }
    
    study_sessions[user_id][subject]['sessions'].append({
        'start': datetime.now(TIMEZONE),
        'end': None,
        'paused': False
    })
    
    active_timers[user_id] = {
        'subject': subject,
        'session_index': len(study_sessions[user_id][subject]['sessions']) - 1
    }
    
    await update.message.reply_text(
        f"⏳ مطالعه درس '{subject}' شروع شد.\n"
        f"برای توقف موقت از دکمه ⏸ استفاده کنید."
    )

async def pause_study(update: Update, user_id: int) -> None:
    """توقف موقت مطالعه"""
    if user_id not in active_timers:
        await update.message.reply_text("⚠️ شما در حال حاضر مطالعه فعال ندارید.")
        return
    
    subject = active_timers[user_id]['subject']
    session_index = active_timers[user_id]['session_index']
    
    study_sessions[user_id][subject]['sessions'][session_index]['end'] = datetime.now(TIMEZONE)
    study_sessions[user_id][subject]['sessions'][session_index]['paused'] = True
    
    elapsed = (study_sessions[user_id][subject]['sessions'][session_index]['end'] - 
              study_sessions[user_id][subject]['sessions'][session_index]['start']).seconds
    
    study_sessions[user_id][subject]['total_seconds'] += elapsed
    
    del active_timers[user_id]
    
    await update.message.reply_text(
        f"⏸ مطالعه درس '{subject}' به مدت {elapsed//60} دقیقه متوقف شد.\n"
        "برای ادامه از دکمه 🔄 استفاده کنید."
    )

async def resume_study(update: Update, user_id: int) -> None:
    """ادامه مطالعه متوقف شده"""
    if user_id not in study_sessions:
        await update.message.reply_text("⚠️ شما هیچ مطالعه‌ای برای ادامه ندارید.")
        return
    
    # پیدا کردن آخرین جلسه متوقف شده
    last_paused = None
    for subject, data in study_sessions[user_id].items():
        for i, session in enumerate(data['sessions']):
            if session['paused'] and session['end'] is not None:
                last_paused = (subject, i)
    
    if not last_paused:
        await update.message.reply_text("⚠️ هیچ مطالعه متوقفی پیدا نشد.")
        return
    
    subject, session_index = last_paused
    
    # شروع جلسه جدید برای همان درس
    study_sessions[user_id][subject]['sessions'].append({
        'start': datetime.now(TIMEZONE),
        'end': None,
        'paused': False
    })
    
    active_timers[user_id] = {
        'subject': subject,
        'session_index': len(study_sessions[user_id][subject]['sessions']) - 1
    }
    
    await update.message.reply_text(
        f"🔄 مطالعه درس '{subject}' ادامه یافت.\n"
        f"زمان قبلی: {study_sessions[user_id][subject]['total_seconds']//60} دقیقه"
    )

async def show_stats(update: Update, user_id: int) -> None:
    """نمایش آمار مطالعه"""
    if user_id not in study_sessions or not study_sessions[user_id]:
        await update.message.reply_text("📊 شما هنوز مطالعه‌ای ثبت نکرده‌اید.")
        return
    
    message = "📊 آمار مطالعه شما:\n\n"
    for subject, data in study_sessions[user_id].items():
        total_minutes = data['total_seconds'] // 60
        message += f"📚 {subject}: {total_minutes} دقیقه ({len(data['sessions'])} جلسه)\n"
    
    await update.message.reply_text(message)

async def stop_study(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """توقف کامل مطالعه"""
    user_id = update.message.from_user.id
    if user_id not in active_timers:
        await update.message.reply_text("⚠️ شما در حال حاضر مطالعه فعال ندارید.")
        return
    
    subject = active_timers[user_id]['subject']
    session_index = active_timers[user_id]['session_index']
    
    study_sessions[user_id][subject]['sessions'][session_index]['end'] = datetime.now(TIMEZONE)
    elapsed = (study_sessions[user_id][subject]['sessions'][session_index]['end'] - 
              study_sessions[user_id][subject]['sessions'][session_index]['start']).seconds
    
    study_sessions[user_id][subject]['total_seconds'] += elapsed
    
    del active_timers[user_id]
    
    await update.message.reply_text(
        f"✅ مطالعه درس '{subject}' به پایان رسید.\n"
        f"زمان کل: {study_sessions[user_id][subject]['total_seconds']//60} دقیقه"
    )

def main() -> None:
    """اجرای اصلی ربات"""
    application = Application.builder().token(TOKEN).build()

    # ثبت هندلرها
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("stop", stop_study))
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_messages))
    
    # شروع ربات
    application.run_polling()

if __name__ == "__main__":
    main()
