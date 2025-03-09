import asyncio
import aiohttp
import os
import logging
import platform
import time
import ssl
from dotenv import load_dotenv
from telegram import Update, ReplyKeyboardMarkup, KeyboardButton
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes, JobQueue

# Завантажуємо змінні середовища
load_dotenv(dotenv_path="C:\\Users\\konto\\Desktop\\Become_Riht bot\\token.env")
TELEGRAM_TOKEN = os.getenv("TELEGRAM_TOKEN")

if not TELEGRAM_TOKEN:
    raise ValueError("❌ Токен бота не знайдено. Перевірте файл .env!")

logging.basicConfig(format="%(asctime)s - %(name)s - %(levelname)s - %(message)s", level=logging.INFO)
logger = logging.getLogger(__name__)

if platform.system() == "Windows":
    asyncio.set_event_loop_policy(asyncio.WindowsSelectorEventLoopPolicy())

# 📌 Функція для створення **меню кнопок** (тепер гарантовано працює!)
def get_main_menu():
    keyboard = [
        [KeyboardButton("📈 Старт моніторингу"), KeyboardButton("🛑 Стоп моніторингу")],
        [KeyboardButton("⚙️ Встановити поріг"), KeyboardButton("ℹ️ Статус")],
        [KeyboardButton("📋 Активні користувачі")]  # ✅ Додаємо кнопку "Активні користувачі"
    ]
    return ReplyKeyboardMarkup(keyboard, resize_keyboard=True, one_time_keyboard=False)

# 📌 Функція для **показу списку активних користувачів**
async def show_active_users(update: Update, context: ContextTypes.DEFAULT_TYPE):
    active_users = context.bot_data.get("active_users", {})

    if active_users:
        users_list = "\n".join([f"🔹 {name} ({user_id})" for user_id, name in active_users.items()])
        await update.message.reply_text(f"📋 *Список активних користувачів:*\n{users_list}", parse_mode="Markdown")
    else:
        await update.message.reply_text("ℹ️ Наразі немає активних користувачів.")

# 📢 **Обробка натискання кнопок**
async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text

    if text == "📈 Старт моніторингу":
        await monitor(update, context)
    elif text == "🛑 Стоп моніторингу":
        await stop(update, context)
    elif text == "⚙️ Встановити поріг":
        await update.message.reply_text("⚠️ Введіть команду: `/set_threshold 5` (де 5 – це %)")
    elif text == "ℹ️ Статус":
        await update.message.reply_text("📊 Бот працює та моніторить ринок")
    elif text == "📋 Активні користувачі":
        await show_active_users(update, context)  # ✅ Викликаємо список активних користувачів

# 📌 **Команда запуску моніторингу**
async def monitor(update: Update, context: ContextTypes.DEFAULT_TYPE):
    chat_id = update.message.chat_id
    user_name = update.message.from_user.full_name  

    active_users = context.bot_data.setdefault("active_users", {})
    active_users[chat_id] = user_name  

    logger.info(f"📢 Новий активний користувач: {user_name} ({chat_id})")

    # ✅ Вивід списку активних користувачів після підключення нового
    users_list = "\n".join([f"🔹 {name} ({user_id})" for user_id, name in active_users.items()])
    await update.message.reply_text(
        f"✅ Моніторинг запущено!\n\n📋 *Список активних користувачів:*\n{users_list}",
        parse_mode="Markdown",
        reply_markup=get_main_menu()
    )

# 📌 **Команда зупинки моніторингу**
async def stop(update: Update, context: ContextTypes.DEFAULT_TYPE):
    chat_id = update.message.chat_id
    active_users = context.bot_data.get("active_users", {})

    if chat_id in active_users:
        del active_users[chat_id]  
        logger.info(f"🚫 Користувач {chat_id} вимкнув моніторинг.")

    await update.message.reply_text("🚫 Моніторинг зупинено.", reply_markup=get_main_menu())

# 🔄 **Головна функція**
def main():
    app = Application.builder().token(TELEGRAM_TOKEN).build()
    
    app.add_handler(CommandHandler("monitor", monitor))
    app.add_handler(CommandHandler("stop", stop))
    app.add_handler(CommandHandler("active_users", show_active_users))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, button_handler))  # ✅ Додаємо обробку кнопок

    if "active_users" not in app.bot_data:
        app.bot_data["active_users"] = {}

    logger.info(f"✅ Бот запущено! Активні користувачі: {len(app.bot_data['active_users'])}")
    
    app.run_polling()

if __name__ == "__main__":
    main()
