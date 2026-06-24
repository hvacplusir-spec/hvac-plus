import logging
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, ContextTypes
import sqlite3
import os

TOKEN = "YOUR_BOT_TOKEN"

logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)

def get_user_count(user_id):
    conn = sqlite3.connect('users.db')
    cursor = conn.cursor()
    cursor.execute('SELECT download_count FROM users WHERE user_id = ?', (user_id,))
    result = cursor.fetchone()
    conn.close()
    if result:
        return result[0]
    else:
        conn = sqlite3.connect('users.db')
        cursor = conn.cursor()
        cursor.execute('INSERT INTO users (user_id, download_count) VALUES (?, 0)', (user_id,))
        conn.commit()
        conn.close()
        return 0

def increment_user_count(user_id):
    conn = sqlite3.connect('users.db')
    cursor = conn.cursor()
    cursor.execute('UPDATE users SET download_count = download_count + 1 WHERE user_id = ?', (user_id,))
    conn.commit()
    conn.close()

def get_all_files():
    conn = sqlite3.connect('users.db')
    cursor = conn.cursor()
    cursor.execute('SELECT id, title, file_link FROM files')
    result = cursor.fetchall()
    conn.close()
    return result

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    count = get_user_count(user_id)
    text = f"""سلام {update.effective_user.first_name} 👋
تعداد دانلودهای باقی‌مانده: {2 - count} از ۲ دانلود رایگان
برای مشاهده لیست فایل‌ها، دکمه زیر را بزنید:"""
    keyboard = [[InlineKeyboardButton("📁 مشاهده لیست فایل‌ها", callback_data="show_files")]]
    reply_markup = InlineKeyboardMarkup(keyboard)
    await update.message.reply_text(text, reply_markup=reply_markup)

async def show_files(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    files = get_all_files()
    if not files:
        await query.edit_message_text("📭 در حال حاضر هیچ فایلی موجود نیست.")
        return
    keyboard = []
    for file_id, title, link in files:
        keyboard.append([InlineKeyboardButton(title, callback_data=f"download_{file_id}")])
    keyboard.append([InlineKeyboardButton("🔙 بازگشت", callback_data="back")])
    reply_markup = InlineKeyboardMarkup(keyboard)
    await query.edit_message_text("📚 لیست فایل‌های موجود:\n\nبرای دانلود، روی عنوان فایل کلیک کنید:", reply_markup=reply_markup)

async def download_file(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    user_id = update.effective_user.id
    file_id = int(query.data.split('_')[1])
    conn = sqlite3.connect('users.db')
    cursor = conn.cursor()
    cursor.execute('SELECT title, file_link FROM files WHERE id = ?', (file_id,))
    file_info = cursor.fetchone()
    conn.close()
    if not file_info:
        await query.edit_message_text("❌ فایل یافت نشد.")
        return
    title, file_link = file_info
    count = get_user_count(user_id)
    if count < 2:
        increment_user_count(user_id)
        await query.edit_message_text(f"✅ دانلود فایل «{title}» با موفقیت انجام شد.\n\nتعداد دانلودهای باقی‌مانده: {2 - (count + 1)} از ۲")
        await context.bot.send_message(chat_id=user_id, text=f"🔗 لینک دانلود:\n{file_link}")
    else:
        await query.edit_message_text(f"⚠️ شما از سهمیه رایگان خود (۲ فایل) استفاده کرده‌اید.\n\n💰 برای دانلود فایل‌های بیشتر، باید اشتراک ماهانه ۱۰۰,۰۰۰ تومان پرداخت کنید.\n\n🔗 لینک پرداخت: (لینک درگاه پرداخت را اینجا قرار دهید)")

async def back(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    await start(update, context)

def main():
    application = Application.builder().token(TOKEN).build()
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CallbackQueryHandler(show_files, pattern="^show_files$"))
    application.add_handler(CallbackQueryHandler(download_file, pattern="^download_"))
    application.add_handler(CallbackQueryHandler(back, pattern="^back$"))
    print("🤖 ربات در حال اجراست...")
    application.run_polling()

if __name__ == '__main__':
    main()