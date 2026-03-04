import asyncio
import logging
import sqlite3
from datetime import datetime
from aiogram import Bot, Dispatcher, types
from aiogram.filters import Command
from aiogram.types import Message

# ---------- НАСТРОЙКИ ----------
BOT_TOKEN = "8699496900:AAEYrhKB4xKx7jGx0WEheFGR-s0bt3ylCY4"
MAIN_ADMIN_ID = 8324426263  # Ваш Telegram ID

logging.basicConfig(level=logging.INFO)
bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()

# ---------- ХРАНЕНИЕ ДАННЫХ В ПАМЯТИ ----------
# Словарь: (admin_chat_id, message_id) -> user_id
# Нужен, чтобы находить пользователя по reply админа
user_message_map = {}

# Множество пользователей, от которых уже было отправлено первое сообщение
first_message_sent = set()

# ---------- БАЗА ДАННЫХ ----------
def init_db():
    conn = sqlite3.connect('caesar_bot.db')
    cur = conn.cursor()
    cur.execute('''
        CREATE TABLE IF NOT EXISTS users (
            user_id INTEGER PRIMARY KEY,
            username TEXT,
            first_name TEXT,
            last_name TEXT,
            first_seen TIMESTAMP,
            last_active TIMESTAMP,
            is_blocked INTEGER DEFAULT 0
        )
    ''')
    cur.execute('''
        CREATE TABLE IF NOT EXISTS admins (
            user_id INTEGER PRIMARY KEY,
            username TEXT,
            added_by INTEGER,
            added_at TIMESTAMP,
            is_main_admin INTEGER DEFAULT 0
        )
    ''')
    conn.commit()
    cur.execute('INSERT OR IGNORE INTO admins (user_id, added_by, added_at, is_main_admin) VALUES (?, ?, ?, ?)',
                (MAIN_ADMIN_ID, MAIN_ADMIN_ID, datetime.now(), 1))
    conn.commit()
    conn.close()

def get_all_admins():
    conn = sqlite3.connect('caesar_bot.db')
    cur = conn.cursor()
    cur.execute('SELECT user_id FROM admins')
    admins = [row[0] for row in cur.fetchall()]
    conn.close()
    return admins

def is_admin(user_id: int) -> bool:
    return user_id in get_all_admins()

def is_main_admin(user_id: int) -> bool:
    conn = sqlite3.connect('caesar_bot.db')
    cur = conn.cursor()
    cur.execute('SELECT is_main_admin FROM admins WHERE user_id = ?', (user_id,))
    result = cur.fetchone()
    conn.close()
    return result and result[0] == 1

def add_admin(user_id: int, username: str, added_by: int):
    conn = sqlite3.connect('caesar_bot.db')
    cur = conn.cursor()
    cur.execute('INSERT OR REPLACE INTO admins (user_id, username, added_by, added_at, is_main_admin) VALUES (?, ?, ?, ?, ?)',
                (user_id, username, added_by, datetime.now(), 0))
    conn.commit()
    conn.close()

def remove_admin(user_id: int) -> bool:
    if user_id == MAIN_ADMIN_ID:
        return False
    conn = sqlite3.connect('caesar_bot.db')
    cur = conn.cursor()
    cur.execute('DELETE FROM admins WHERE user_id = ? AND is_main_admin = 0', (user_id,))
    deleted = cur.rowcount > 0
    conn.commit()
    conn.close()
    return deleted

def save_user(message: Message):
    user = message.from_user
    conn = sqlite3.connect('caesar_bot.db')
    cur = conn.cursor()
    cur.execute('''
        INSERT OR REPLACE INTO users 
        (user_id, username, first_name, last_name, last_active)
        VALUES (?, ?, ?, ?, ?)
    ''', (
        user.id,
        user.username,
        user.first_name,
        user.last_name,
        datetime.now()
    ))
    conn.commit()
    conn.close()

def is_user_blocked(user_id: int) -> bool:
    conn = sqlite3.connect('caesar_bot.db')
    cur = conn.cursor()
    cur.execute('SELECT is_blocked FROM users WHERE user_id = ?', (user_id,))
    result = cur.fetchone()
    conn.close()
    return result and result[0] == 1

def block_user(user_id: int):
    conn = sqlite3.connect('caesar_bot.db')
    cur = conn.cursor()
    cur.execute('UPDATE users SET is_blocked = 1 WHERE user_id = ?', (user_id,))
    conn.commit()
    conn.close()

def unblock_user(user_id: int):
    conn = sqlite3.connect('caesar_bot.db')
    cur = conn.cursor()
    cur.execute('UPDATE users SET is_blocked = 0 WHERE user_id = ?', (user_id,))
    conn.commit()
    conn.close()

# ---------- КОМАНДЫ ПОЛЬЗОВАТЕЛЕЙ ----------
@dp.message(Command("start"))
async def cmd_start(message: Message):
    save_user(message)
    await message.answer("Вы можете написать сообщение, и администратор ответит вам.")

@dp.message(Command("help"))
async def cmd_help(message: Message):
    await message.answer("Отправьте сообщение – администратор ответит.")

# ---------- АДМИНСКИЕ КОМАНДЫ ----------
@dp.message(Command("admins"))
async def cmd_admins(message: Message):
    if not is_admin(message.from_user.id):
        return
    admin_ids = get_all_admins()
    if not admin_ids:
        await message.answer("Нет админов.")
        return
    lines = []
    for admin_id in admin_ids:
        try:
            chat = await bot.get_chat(admin_id)
            name = f"@{chat.username}" if chat.username else chat.first_name
            lines.append(f"• {name}" + (" (главный)" if admin_id == MAIN_ADMIN_ID else ""))
        except:
            lines.append(f"• Админ {admin_id}")
    await message.answer("Администраторы:\n" + "\n".join(lines))

@dp.message(Command("stats"))
async def cmd_stats(message: Message):
    if not is_admin(message.from_user.id):
        return
    conn = sqlite3.connect('caesar_bot.db')
    cur = conn.cursor()
    cur.execute('SELECT COUNT(*) FROM users')
    total = cur.fetchone()[0]
    cur.execute('SELECT COUNT(*) FROM users WHERE is_blocked = 1')
    blocked = cur.fetchone()[0]
    conn.close()
    await message.answer(f"Пользователей: {total}\nЗаблокировано: {blocked}")

@dp.message(Command("ban"))
async def cmd_ban(message: Message):
    if not is_admin(message.from_user.id):
        return
    try:
        user_id = int(message.text.split()[1])
        block_user(user_id)
        await message.answer(f"Заблокирован {user_id}.")
        try:
            await bot.send_message(user_id, "❌ Вы заблокированы.")
        except:
            pass
    except:
        await message.answer("Использование: /ban 123456789")

@dp.message(Command("unban"))
async def cmd_unban(message: Message):
    if not is_admin(message.from_user.id):
        return
    try:
        user_id = int(message.text.split()[1])
        unblock_user(user_id)
        await message.answer(f"Разблокирован {user_id}.")
    except:
        await message.answer("Использование: /unban 123456789")

@dp.message(Command("add_admin"))
async def cmd_add_admin(message: Message):
    if not is_main_admin(message.from_user.id):
        await message.answer("Только главный админ может добавлять.")
        return
    try:
        user_id = int(message.text.split()[1])
        try:
            chat = await bot.get_chat(user_id)
            username = chat.username
        except:
            username = None
        add_admin(user_id, username, message.from_user.id)
        await message.answer(f"Добавлен админ {user_id}.")
        try:
            await bot.send_message(user_id, "Вам выдали админку , команды - ( /ban id , /unban id ) ")
        except:
            pass
    except:
        await message.answer("Использование: /add_admin 123456789")

@dp.message(Command("remove_admin"))
async def cmd_remove_admin(message: Message):
    if not is_main_admin(message.from_user.id):
        await message.answer("Только главный админ может удалять.")
        return
    try:
        user_id = int(message.text.split()[1])
        if remove_admin(user_id):
            await message.answer(f"Удалён админ {user_id}.")
        else:
            await message.answer("Не удалось удалить.")
    except:
        await message.answer("Использование: /remove_admin 123456789")

# ---------- ПЕРЕСЫЛКА СООБЩЕНИЙ ----------
@dp.message()
async def handle_message(message: Message):
    # Пропускаем команды (они уже обработаны выше)
    if message.text and message.text.startswith('/'):
        return

    # Если пользователь заблокирован – уведомляем его
    if is_user_blocked(message.from_user.id):
        await message.answer("❌ Вы заблокированы.")
        return

    save_user(message)

    # Сообщение от обычного пользователя -> пересылаем админам
    if not is_admin(message.from_user.id):
        user_id = message.from_user.id

        # Определяем, нужно ли добавлять информацию о пользователе
        if user_id not in first_message_sent:
            # Первое сообщение – отправляем с заголовком
            header = (
                f"ID: {user_id}\n"
                f"Username: @{message.from_user.username}\n"
                f"Имя: {message.from_user.first_name}\n"
                f"Время: {datetime.now().strftime('%H:%M %d.%m.%Y')}\n"
                "━━━━━━━━━━━━"
            )
            first_message_sent.add(user_id)
        else:
            # Повторное сообщение – только контент
            header = None

        # Рассылаем всем админам
        for admin_id in get_all_admins():
            try:
                if header:
                    # Отправляем с заголовком
                    if message.text:
                        sent = await bot.send_message(admin_id, f"{header}\n\n{message.text}")
                    elif message.photo:
                        sent = await bot.send_photo(admin_id, message.photo[-1].file_id, caption=f"{header}\n\n{message.caption or ''}")
                    elif message.video:
                        sent = await bot.send_video(admin_id, message.video.file_id, caption=f"{header}\n\n{message.caption or ''}")
                    elif message.document:
                        sent = await bot.send_document(admin_id, message.document.file_id, caption=f"{header}\n\n{message.caption or ''}")
                    elif message.voice:
                        sent = await bot.send_voice(admin_id, message.voice.file_id, caption=header)
                    else:
                        sent = await bot.send_message(admin_id, f"{header}\n\n[неподдерживаемый тип]")
                else:
                    # Отправляем без заголовка
                    if message.text:
                        sent = await bot.send_message(admin_id, message.text)
                    elif message.photo:
                        sent = await bot.send_photo(admin_id, message.photo[-1].file_id, caption=message.caption or "")
                    elif message.video:
                        sent = await bot.send_video(admin_id, message.video.file_id, caption=message.caption or "")
                    elif message.document:
                        sent = await bot.send_document(admin_id, message.document.file_id, caption=message.caption or "")
                    elif message.voice:
                        sent = await bot.send_voice(admin_id, message.voice.file_id)
                    else:
                        sent = await bot.send_message(admin_id, "[неподдерживаемый тип]")

                # Запоминаем, что это сообщение (admin_id, message_id) принадлежит данному пользователю
                user_message_map[(admin_id, sent.message_id)] = user_id

            except Exception as e:
                logging.error(f"Ошибка отправки админу {admin_id}: {e}")

        # Пользователь не получает никакого подтверждения

    # Сообщение от админа – возможно, ответ пользователю
    else:
        # Если админ ответил на какое-то сообщение (reply)
        if message.reply_to_message:
            admin_id = message.chat.id
            replied_msg_id = message.reply_to_message.message_id

            # Ищем пользователя по ключу (admin_id, replied_msg_id)
            user_id = user_message_map.get((admin_id, replied_msg_id))

            if user_id:
                try:
                    # Отправляем ответ пользователю
                    if message.text:
                        await bot.send_message(user_id, message.text)
                    elif message.photo:
                        await bot.send_photo(user_id, message.photo[-1].file_id, caption=message.caption or "")
                    elif message.video:
                        await bot.send_video(user_id, message.video.file_id, caption=message.caption or "")
                    elif message.document:
                        await bot.send_document(user_id, message.document.file_id, caption=message.caption or "")
                    elif message.voice:
                        await bot.send_voice(user_id, message.voice.file_id)
                    else:
                        await bot.send_message(user_id, "[медиа]")
                    # Не отправляем админу никакого подтверждения
                except Exception as e:
                    # При ошибке тоже ничего не сообщаем (можно залогировать)
                    logging.error(f"Ошибка отправки ответа пользователю {user_id}: {e}")
            # else: если не нашли пользователя – тихо игнорируем
        # Если админ пишет без reply – игнорируем

# ---------- ЗАПУСК ----------
async def main():
    init_db()
    logging.info("Бот запущен")
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
