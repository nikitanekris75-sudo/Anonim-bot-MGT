# anon_bot.py
# Анонимный чат с кнопками и JSON-хранилищем
# Работает в Pydroid 3

import json
import os
from telegram import Update, KeyboardButton, ReplyKeyboardMarkup
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes

# === НАСТРОЙКИ ===
BOT_TOKEN = "7506475009:AAEefz_DyXb5nTLaTajXMSTKIjdkfpM-Gn0"  # ← Замени на свой токен
ADMIN_ID = 8015206707  # ← Замени на свой Telegram ID (узнай у @userinfobot)
DATA_FILE = "user_sessions.json"  # Файл для сохранения пользователей
# =================

# --- Работа с JSON ---
def load_sessions():
    """Загружаем список пользователей из файла"""
    if os.path.exists(DATA_FILE):
        with open(DATA_FILE, "r", encoding="utf-8") as f:
            try:
                return json.load(f)  # { "123456": true, ... }
            except json.JSONDecodeError:
                return {}
    return {}

def save_sessions(data):
    """Сохраняем список пользователей"""
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

# --- Глобальные переменные ---
user_sessions = load_sessions()  # Пользователи, которые запускали бота
admin_message_to_user = {}       # Временное: { forwarded_msg_id: user_id }

# --- Клавиатура ---
def get_keyboard():
    keyboard = [
        [KeyboardButton("✉️ Написать анонимно")],
        [KeyboardButton("🔗 Моя ссылка"), KeyboardButton("❓ Помощь")]
    ]
    return ReplyKeyboardMarkup(keyboard, resize_keyboard=True, one_time_keyboard=False)

# --- Команда /start ---
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    user_name = update.effective_user.full_name
    bot_username = "Анонимный чат МГТ"  # ← Замени на юзернейм своего бота

    referral_link = f"https://t.me/{bot_username}?start={user_id}"

    if user_id == ADMIN_ID:
        await update.message.reply_text(
            f"🛡 Добро пожаловать, админ!\n"
            f"Ты получишь анонимные сообщения.\n"
            f"Чтобы ответить — просто ответь на пересланное сообщение.",
            reply_markup=get_keyboard()
        )
    else:
        # Добавляем пользователя, если его нет
        if str(user_id) not in user_sessions:
            user_sessions[str(user_id)] = True
            save_sessions(user_sessions)

        await update.message.reply_text(
            f"🛡 Привет, {user_name}! Готов передать твоё сообщение анонимно.",
            reply_markup=get_keyboard()
        )

# --- Обработка сообщений ---
async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    text = update.message.text

    # --- Админ пишет ---
    if user_id == ADMIN_ID:
        # Ответ на пересланное сообщение
        if update.message.reply_to_message:
            original_msg = update.message.reply_to_message
            msg_id = original_msg.message_id

            if msg_id in admin_message_to_user:
                target_user_id = admin_message_to_user[msg_id]

                try:
                    await context.bot.send_message(
                        chat_id=target_user_id,
                        text=f"📩 Админ ответил:\n\n{update.message.text}"
                    )
                    await update.message.reply_text("✅ Ответ отправлен пользователю.")
                except Exception:
                    await update.message.reply_text("❌ Не удалось отправить (пользователь заблокировал бота).")
        return

    # --- Пользователь: кнопки ---
    if text == "✉️ Написать анонимно":
        await update.message.reply_text("Напиши своё сообщение (текст, фото, голос и т.д.):")
        return

    elif text == "🔗 Моя ссылка":
        bot_username = "YourAnonBot"  # ← Замени
        referral_link = f"https://t.me/{bot_username}?start={user_id}"
        await update.message.reply_text(f"Вот твоя личная ссылка:\n{referral_link}")
        return

    elif text == "❓ Помощь":
        await update.message.reply_text(
            "🛡 Анонимный ящик\n"
            "1. Нажми 'Написать анонимно' и отправь сообщение.\n"
            "2. Админ получит его без твоего имени.\n"
            "3. Если он ответит — ты получишь уведомление.\n"
            "4. Твоя ссылка — чтобы друзья тоже могли написать."
        )
        return

    # --- Пересылка сообщения админу ---
    try:
        sent_message = await update.message.forward(chat_id=ADMIN_ID)
        admin_message_to_user[sent_message.message_id] = user_id
        await update.message.reply_text("✅ Сообщение отправлено анонимно!")
    except Exception as e:
        await update.message.reply_text("❌ Не удалось отправить сообщение.")
        print("Ошибка при пересылке:", e)

# --- Запуск бота ---
def main():
    global admin_message_to_user
    print("🔄 Загружаем данные...")
    admin_message_to_user = {}  # Очищаем при перезапуске

    print("🤖 Запускаем бота...")
    app = Application.builder().token(BOT_TOKEN).build()

    # Обработчики
    app.add_handler(CommandHandler("start", start))

    # Текст (кроме команд)
    app.add_handler(MessageHandler(
        filters.TEXT & ~filters.COMMAND,
        handle_message
    ))

    # Медиа: фото, голос, видео, аудио
    app.add_handler(MessageHandler(
        filters.PHOTO | filters.VOICE | filters.VIDEO | filters.AUDIO,
        handle_message
    ))

    print("✅ Бот запущен! Готов к работе.")
    app.run_polling()

# --- Запуск ---
if __name__ == "__main__":
    main()
