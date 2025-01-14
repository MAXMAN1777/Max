import telebot
from telebot.types import InlineKeyboardMarkup, InlineKeyboardButton
from datetime import datetime, time
import threading
import time as tm  # Kutish uchun time moduli

# Bot token va ma'lumotlar
TOKEN = '8003666068:AAEyRRGIEFJAmiJM4fJSehTKcenYl_qZx9I'
bot = telebot.TeleBot(TOKEN)

ADMIN_ID = 5151316980  # Admin ID
CHANNEL_URL = 'https://t.me/+Di0aG9oCLSJiY2Zi'  # Maxfiy kanal linki
ALLOWED_USERS = []  # Ruxsat berilgan foydalanuvchilar ro‘yxati
BLOCKED_USERS = []  # Bloklangan foydalanuvchilar ro'yxati

# Kechasi avtomatik boshqaruv o'zgaruvchisi
NIGHT_MODE = False


# Vaqtni tekshirish funksiyasi
def check_time():
    global NIGHT_MODE
    current_time = datetime.now().time()
    if time(0, 0) <= current_time <= time(6, 0):  # 00:00 dan 06:00 gacha
        NIGHT_MODE = True
    else:
        NIGHT_MODE = False
    threading.Timer(60, check_time).start()  # Har 1 daqiqada vaqtni tekshiradi


# /start komandasi
@bot.message_handler(commands=['start'])
def start(message):
    user_id = message.chat.id
    username = message.from_user.username or "No username"

    # Foydalanuvchi bloklangan bo'lsa
    if user_id in BLOCKED_USERS:
        bot.send_message(user_id, "❌ Siz bloklangansiz. Adminga murojaat qiling.")
        return

    # Tungi rejim
    if NIGHT_MODE:
        if user_id not in ALLOWED_USERS:
            ALLOWED_USERS.append(user_id)
            bot.send_message(ADMIN_ID, f"Tungi rejim: Yangi foydalanuvchi so'rov yubordi:\nUsername: @{username}\nID: {user_id}")
            bot.send_message(user_id, "✅ Tungi rejimda avtomatik ruxsat beriladi. Iltimos, 1 daqiqa kuting...")
            tm.sleep(60)  # 1 daqiqa kutadi
            bot.send_message(
                user_id,
                "✅ Tabriklaymiz! Siz tasdiqlandingiz. Maxfiy kanalga kirish uchun quyidagi tugmani bosing:",
                reply_markup=InlineKeyboardMarkup().add(
                    InlineKeyboardButton("Kanalga kirish", url=CHANNEL_URL)
                )
            )
        return

    # Kunduzgi rejim
    markup = InlineKeyboardMarkup()
    markup.add(
        InlineKeyboardButton("✅ Ruxsat berish", callback_data=f"approve_{user_id}"),
        InlineKeyboardButton("❌ Bloklash", callback_data=f"block_{user_id}")
    )

    bot.send_message(
        ADMIN_ID,
        f"Yangi foydalanuvchi so'rov yubordi:\nUsername: @{username}\nID: {user_id}",
        reply_markup=markup
    )
    bot.send_message(user_id, "So‘rovingiz adminga yuborildi. Javobni kuting.")


# Callback ma'lumotlarini qayta ishlash
@bot.callback_query_handler(func=lambda call: True)
def handle_callback(call):
    action, user_id = call.data.split('_')
    user_id = int(user_id)

    if action == 'approve':
        if user_id not in ALLOWED_USERS:
            ALLOWED_USERS.append(user_id)
            bot.send_message(
                user_id,
                "✅ Tabriklaymiz! Siz tasdiqlandingiz. Maxfiy kanalga kirish uchun quyidagi tugmani bosing:",
                reply_markup=InlineKeyboardMarkup().add(
                    InlineKeyboardButton("Kanalga kirish", url=CHANNEL_URL)
                )
            )
        bot.send_message(ADMIN_ID, f"Foydalanuvchi {user_id} ga ruxsat berildi.")

    elif action == 'block':
        if user_id not in BLOCKED_USERS:
            BLOCKED_USERS.append(user_id)
        bot.send_message(user_id, "❌ Siz bloklangansiz.")
        bot.send_message(ADMIN_ID, f"Foydalanuvchi {user_id} bloklandi.")

    elif action == 'unblock':
        if user_id in BLOCKED_USERS:
            BLOCKED_USERS.remove(user_id)
        bot.send_message(user_id, "✅ Siz blokdan chiqarildingiz.")
        bot.send_message(ADMIN_ID, f"Foydalanuvchi {user_id} blokdan chiqarildi.")


# Adminga foydalanuvchi ro'yxatini ko'rsatish
@bot.message_handler(commands=['users'])
def show_users(message):
    if message.chat.id == ADMIN_ID:
        users_list = "\n".join([f"ID: {user_id}" for user_id in ALLOWED_USERS])
        blocked_list = "\n".join([f"ID: {user_id}" for user_id in BLOCKED_USERS])

        if not users_list:
            users_list = "Hech kim foydalanmayapti."
        if not blocked_list:
            blocked_list = "Bloklangan foydalanuvchilar yo'q."

        markup = InlineKeyboardMarkup()
        for user_id in ALLOWED_USERS:
            markup.add(
                InlineKeyboardButton(f"❌ Bloklash: {user_id}", callback_data=f"block_{user_id}")
            )
        for user_id in BLOCKED_USERS:
            markup.add(
                InlineKeyboardButton(f"✅ Blokdan chiqarish: {user_id}", callback_data=f"unblock_{user_id}")
            )
        bot.send_message(
            ADMIN_ID,
            f"Botdan foydalanayotganlar:\n{users_list}\n\nBloklanganlar:\n{blocked_list}",
            reply_markup=markup
        )


# Ruxsat etilmagan foydalanuvchilarni cheklash
@bot.message_handler(func=lambda message: message.chat.id not in ALLOWED_USERS)
def restrict_unapproved_users(message):
    bot.send_message(message.chat.id, "❌ Sizga botdan foydalanishga ruxsat berilmagan.")


# Botni ishga tushirish
print("Bot ishlayapti...")
check_time()  # Vaqt tekshirishni boshlash
bot.infinity_polling()

