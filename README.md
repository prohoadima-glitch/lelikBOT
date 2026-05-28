[lelikBot.py](https://github.com/user-attachments/files/28358472/lelikBot.py)
import logging
import os
import random
import asyncio
from datetime import datetime, date, timedelta
from typing import Optional

import httpx
from telegram import Update, ChatMemberUpdated
from telegram.ext import (
    Application, CommandHandler, MessageHandler,
    filters, ContextTypes, ChatMemberHandler
)
from database import Database

# ─────────────────────────────────────────────
# Конфигурация
# ─────────────────────────────────────────────

logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

db = Database()

BOT_TOKEN   = os.getenv("8856328110:AAFl-DUAUm239bAXMl7CT8qpwAW0Fp8r4Bk")
AI_API_KEY  = os.getenv("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")
AI_API_URL  = "https://openrouter.ai/api/v1/chat/completions"
AI_MODEL    = "anthropic/claude-3-haiku"

SYSTEM_PROMPT = """Ты — Memory Emperor, цифровой архив воспоминаний человека.
Ты смесь личного ассистента, цифрового дневника и режиссёра воспоминаний.
Отвечай коротко (1-3 предложения), атмосферно, кинематографично.
Стиль: минимализм, загадочность, цифровая ностальгия.
НЕ используй эмодзи в каждом предложении. НЕ будь корпоративным.
Язык: русский. Обращайся на "ты"."""

# ─────────────────────────────────────────────
# AI — вызов OpenRouter
# ─────────────────────────────────────────────

async def ask_ai(prompt: str, system: str = SYSTEM_PROMPT) -> str:
    """Отправить запрос к AI через OpenRouter."""
    try:
        async with httpx.AsyncClient(timeout=20) as client:
            resp = await client.post(
                AI_API_URL,
                headers={
                    "Authorization": f"Bearer {AI_API_KEY}",
                    "Content-Type": "application/json",
                },
                json={
                    "model": AI_MODEL,
                    "messages": [
                        {"role": "system", "content": system},
                        {"role": "user",   "content": prompt}
                    ],
                    "max_tokens": 200,
                    "temperature": 0.85,
                }
            )
            data = resp.json()
            return data["choices"][0]["message"]["content"].strip()
    except Exception as e:
        logger.error(f"AI error: {e}")
        return random.choice([
            "Этот момент сохранён навсегда.",
            "Что-то в этом кадре останется с тобой.",
            "Ты снова сохраняешь что-то важное.",
            "Время идёт. Ты запомнил это.",
        ])

# ─────────────────────────────────────────────
# Вспомогательные функции
# ─────────────────────────────────────────────

def get_user_display(user) -> str:
    name = user.first_name or ""
    if user.last_name:
        name += f" {user.last_name}"
    return name.strip() or user.username or str(user.id)

async def is_admin(update: Update, context: ContextTypes.DEFAULT_TYPE, user_id: int = None) -> bool:
    uid = user_id or update.effective_user.id
    chat_id = update.effective_chat.id
    try:
        member = await context.bot.get_chat_member(chat_id, uid)
        return member.status in ("administrator", "creator")
    except Exception:
        return False

# ─────────────────────────────────────────────
# Приветствие
# ─────────────────────────────────────────────

WELCOME_TEXT = """*Memory Emperor* — твой цифровой архив воспоминаний.

Я не просто сохраняю фото. Я хранею часть твоей жизни.

─────────────────────
*Что я умею:*

`!запомнить` — сохрани фото или видео с этой подписью. Я проанализирую момент и скажу что-то важное.

`!моменты` — посмотреть свои воспоминания

`!капсула [дата]` — запечатай момент до будущего.
Пример: отправь фото с подписью `!капсула 2027-01-01`

`!стрик` — сколько дней подряд ты сохраняешь моменты

`!статистика` — активность в чате

`!помощь` — все команды

─────────────────────
Начни прямо сейчас — отправь фото с подписью `!запомнить`."""

async def cmd_start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.effective_message.reply_text(WELCOME_TEXT, parse_mode="Markdown")

async def on_chat_member_update(update: Update, context: ContextTypes.DEFAULT_TYPE):
    result: ChatMemberUpdated = update.my_chat_member
    if not result:
        return
    old_status = result.old_chat_member.status
    new_status = result.new_chat_member.status
    if old_status in ("kicked", "left") and new_status in ("member", "administrator"):
        await context.bot.send_message(
            chat_id=result.chat.id,
            text=WELCOME_TEXT,
            parse_mode="Markdown"
        )

# ─────────────────────────────────────────────
# Запись всех сообщений + стрик
# ─────────────────────────────────────────────

async def track_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    msg  = update.effective_message
    user = update.effective_user
    chat = update.effective_chat
    if not msg or not user:
        return

    text    = msg.text or msg.caption or ""
    tl      = text.lower()

    # Маршрутизация команд (работают и в группах и в личке)
    if "!запомнить" in tl and not tl.startswith("!капсула"):
        await save_moment(update, context)
        return
    if tl.startswith("!капсула"):
        await cmd_capsule(update, context)
        return
    if tl.startswith("!моменты"):
        await cmd_moments(update, context)
        return
    if tl.startswith("!стрик"):
        await cmd_streak(update, context)
        return
    if tl.startswith("!статистика"):
        await cmd_stats(update, context)
        return
    if tl.startswith("!помощь"):
        await cmd_help(update, context)
        return

    # Сохранить сообщение в базу
    msg_type = "text"
    file_id  = None
    if msg.photo:
        msg_type, file_id = "photo", msg.photo[-1].file_id
    elif msg.video:
        msg_type, file_id = "video", msg.video.file_id
    elif msg.voice:
        msg_type, file_id = "voice", msg.voice.file_id
    elif msg.audio:
        msg_type, file_id = "audio", msg.audio.file_id
    elif msg.document:
        msg_type, file_id = "document", msg.document.file_id
    elif msg.sticker:
        msg_type, file_id = "sticker", msg.sticker.file_id

    db.save_message(
        chat_id=chat.id, message_id=msg.message_id,
        user_id=user.id, username=user.username or "",
        display_name=get_user_display(user),
        msg_type=msg_type, text=text, file_id=file_id,
        timestamp=datetime.now().isoformat()
    )

# ─────────────────────────────────────────────
# !запомнить — сохранить момент с AI-анализом
# ─────────────────────────────────────────────

async def save_moment(update: Update, context: ContextTypes.DEFAULT_TYPE):
    msg  = update.effective_message
    user = update.effective_user
    chat = update.effective_chat

    file_id    = None
    media_type = None

    if msg.photo:
        file_id, media_type = msg.photo[-1].file_id, "photo"
    elif msg.video:
        file_id, media_type = msg.video.file_id, "video"

    # Получить описание
    raw_caption = (msg.caption or msg.text or "")
    description = raw_caption.replace("!запомнить", "").replace("!Запомнить", "").strip()

    # Стрик
    streak = db.update_streak(user.id, chat.id)

    # Статистика для AI
    total_moments = db.get_moments_count(user.id)
    hour          = datetime.now().hour
    time_of_day   = "ночь" if hour < 6 else "утро" if hour < 12 else "день" if hour < 18 else "вечер"

    # Запрос к AI
    ai_prompt = f"""Пользователь сохраняет {"фото" if media_type == "photo" else "видео" if media_type else "момент"}.
Время суток: {time_of_day}. Всего моментов в архиве: {total_moments + 1}. Стрик: {streak} дней.
{"Описание: " + description if description else "Без описания."}
Напиши одну атмосферную фразу-реакцию на этот момент (1-2 предложения)."""

    ai_comment = await ask_ai(ai_prompt)

    # Сохранить если есть медиа
    if file_id:
        db.save_moment(
            user_id=user.id, chat_id=chat.id,
            username=user.username or "",
            display_name=get_user_display(user),
            file_id=file_id, media_type=media_type,
            description=description, ai_comment=ai_comment,
            timestamp=datetime.now().isoformat()
        )

    # Ответ
    streak_line = f"\n_{streak} {'день' if streak == 1 else 'дня' if 2 <= streak <= 4 else 'дней'} подряд._" if streak > 1 else ""
    reply = f"{ai_comment}{streak_line}"

    if not file_id:
        reply = f"Текст сохранён в памяти.\n\n{ai_comment}{streak_line}"

    await msg.reply_text(reply, parse_mode="Markdown")

# ─────────────────────────────────────────────
# !капсула — капсула времени
# ─────────────────────────────────────────────

async def cmd_capsule(update: Update, context: ContextTypes.DEFAULT_TYPE):
    msg  = update.effective_message
    user = update.effective_user
    chat = update.effective_chat

    raw  = (msg.caption or msg.text or "").strip()
    # Формат: !капсула YYYY-MM-DD [описание]
    parts = raw.split(None, 2)  # ['!капсула', 'дата', 'описание?']

    if len(parts) < 2:
        await msg.reply_text(
            "Формат: отправь фото/видео с подписью\n"
            "`!капсула 2027-06-01`\n\n"
            "Я запечатаю этот момент и открою его в указанную дату.",
            parse_mode="Markdown"
        )
        return

    date_str = parts[1]
    extra    = parts[2] if len(parts) > 2 else ""

    try:
        open_date = date.fromisoformat(date_str)
    except ValueError:
        await msg.reply_text(
            "Неверный формат даты. Используй `YYYY-MM-DD`\nНапример: `!капсула 2027-01-01`",
            parse_mode="Markdown"
        )
        return

    if open_date <= date.today():
        await msg.reply_text("Дата должна быть в будущем.", parse_mode="Markdown")
        return

    file_id    = None
    media_type = None
    if msg.photo:
        file_id, media_type = msg.photo[-1].file_id, "photo"
    elif msg.video:
        file_id, media_type = msg.video.file_id, "video"

    open_at = datetime.combine(open_date, datetime.min.time()).isoformat()

    db.save_capsule(
        user_id=user.id, chat_id=chat.id,
        file_id=file_id, media_type=media_type,
        text=extra, open_at=open_at,
        timestamp=datetime.now().isoformat()
    )

    days_left = (open_date - date.today()).days
    ai_comment = await ask_ai(
        f"Пользователь запечатал воспоминание в капсулу времени до {date_str} ({days_left} дней). "
        f"Напиши одну атмосферную фразу про ожидание и время."
    )

    await msg.reply_text(
        f"🔒 *Запечатано до {date_str}*\n\n{ai_comment}\n\n_{days_left} дней до открытия._",
        parse_mode="Markdown"
    )

# ─────────────────────────────────────────────
# !моменты — просмотр воспоминаний
# ─────────────────────────────────────────────

async def cmd_moments(update: Update, context: ContextTypes.DEFAULT_TYPE):
    msg  = update.effective_message
    user = update.effective_user
    text = (msg.text or "").strip()
    parts = text.split()

    # Определить чьи моменты
    if len(parts) >= 2 and parts[1].startswith("@"):
        if not await is_admin(update, context):
            await msg.reply_text("Только администраторы могут смотреть моменты других.")
            return
        target_data = db.get_user_by_username(parts[1][1:])
        if not target_data:
            await msg.reply_text(f"Пользователь {parts[1]} не найден.")
            return
        target_id      = target_data["user_id"]
        target_display = target_data["display_name"]
        show_own       = False
    else:
        target_id      = user.id
        target_display = get_user_display(user)
        show_own       = True

    moments = db.get_moments(target_id)

    if not moments:
        await msg.reply_text(
            "Архив пуст.\n\nОтправь фото с подписью `!запомнить` — и я начну хранить твою жизнь.",
            parse_mode="Markdown"
        )
        return

    header_prompt = (
        f"У пользователя {len(moments)} сохранённых моментов. "
        f"Напиши одну короткую атмосферную фразу-вступление к его архиву воспоминаний."
    )
    header_ai = await ask_ai(header_prompt)

    await msg.reply_text(
        f"*Архив воспоминаний* — {len(moments)} моментов\n\n_{header_ai}_",
        parse_mode="Markdown"
    )

    for i, moment in enumerate(moments, 1):
        desc      = moment.get("description") or ""
        ai_txt    = moment.get("ai_comment") or ""
        date_str  = moment["timestamp"][:10]
        caption   = f"*#{i}* · {date_str}"
        if desc:
            caption += f"\n{desc}"
        if ai_txt:
            caption += f"\n\n_{ai_txt}_"

        try:
            if moment["media_type"] == "photo":
                await context.bot.send_photo(
                    chat_id=msg.chat_id, photo=moment["file_id"],
                    caption=caption, parse_mode="Markdown"
                )
            elif moment["media_type"] == "video":
                await context.bot.send_video(
                    chat_id=msg.chat_id, video=moment["file_id"],
                    caption=caption, parse_mode="Markdown"
                )
        except Exception as e:
            logger.error(f"Момент #{i}: {e}")
            await msg.reply_text(f"Момент #{i} недоступен — файл устарел.")
        await asyncio.sleep(0.3)

# ─────────────────────────────────────────────
# !стрик
# ─────────────────────────────────────────────

async def cmd_streak(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user   = update.effective_user
    streak = db.get_streak(user.id)

    if streak == 0:
        await update.effective_message.reply_text(
            "Твой стрик ещё не начался.\n\nОтправь первый момент — и отсчёт начнётся.",
            parse_mode="Markdown"
        )
        return

    ai_comment = await ask_ai(
        f"Пользователь сохраняет моменты уже {streak} дней подряд. "
        f"Напиши одну короткую мотивирующую или атмосферную фразу про это."
    )
    await update.effective_message.reply_text(
        f"*{streak} {'день' if streak == 1 else 'дня' if 2 <= streak <= 4 else 'дней'} подряд*\n\n_{ai_comment}_",
        parse_mode="Markdown"
    )

# ─────────────────────────────────────────────
# !статистика
# ─────────────────────────────────────────────

async def cmd_stats(update: Update, context: ContextTypes.DEFAULT_TYPE):
    chat  = update.effective_chat
    msg   = update.effective_message
    user  = update.effective_user

    stats = db.get_chat_stats(chat.id)
    total = db.get_message_count(chat.id)
    user_count = db.get_user_message_count(user.id, chat.id)
    months_active = db.get_user_months_active(user.id, chat.id)

    text = f"*Статистика архива*\n\nСообщений записано: *{total}*\n"
    if months_active:
        text += f"Ты активен уже *{months_active}* мес.\n"
    if user_count:
        text += f"Твоих сообщений: *{user_count}*\n"

    if stats:
        text += "\n*Топ участников:*\n"
        for i, row in enumerate(stats[:10], 1):
            text += f"{i}. {row['display_name']} — {row['count']}\n"

    await msg.reply_text(text, parse_mode="Markdown")

# ─────────────────────────────────────────────
# !помощь
# ─────────────────────────────────────────────

async def cmd_help(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = """*Memory Emperor — команды*

`!запомнить` — сохрани фото/видео с этой подписью
`!моменты` — открыть свой архив
`!моменты @username` — архив другого _(только админы)_
`!капсула YYYY-MM-DD` — запечатать момент до даты
`!стрик` — дней подряд
`!статистика` — активность чата
`!помощь` — это сообщение"""
    await update.effective_message.reply_text(text, parse_mode="Markdown")

# ─────────────────────────────────────────────
# Фоновые задачи
# ─────────────────────────────────────────────

async def job_capsules(context):
    """Открывать капсулы времени в нужный момент."""
    capsules = db.get_due_capsules()
    for cap in capsules:
        try:
            ai_text = await ask_ai(
                f"Капсула времени открыта. Прошло время с момента запечатывания. "
                f"Напиши атмосферную фразу про возвращение к воспоминанию."
            )
            header = f"🔓 *Капсула открыта*\n\n_{ai_text}_"
            chat_id = cap["chat_id"]
            user_id = cap["user_id"]

            if cap["file_id"] and cap["media_type"] == "photo":
                await context.bot.send_photo(
                    chat_id=chat_id, photo=cap["file_id"],
                    caption=header, parse_mode="Markdown"
                )
            elif cap["file_id"] and cap["media_type"] == "video":
                await context.bot.send_video(
                    chat_id=chat_id, video=cap["file_id"],
                    caption=header, parse_mode="Markdown"
                )
            else:
                msg_body = header
                if cap["text"]:
                    msg_body += f"\n\n{cap['text']}"
                await context.bot.send_message(
                    chat_id=chat_id, text=msg_body, parse_mode="Markdown"
                )
            db.mark_capsule_opened(cap["id"])
        except Exception as e:
            logger.error(f"Capsule {cap['id']} error: {e}")

async def job_ghost_memory(context):
    """Ghost Memory — случайно возвращать старые моменты раз в день."""
    users = db.get_all_users_with_moments()
    for u in users:
        # Только ~20% пользователей получают ghost memory в этот день
        if random.random() > 0.2:
            continue
        try:
            moment = db.get_random_old_moment(u["user_id"])
            if not moment:
                continue

            days_ago = (datetime.now() - datetime.fromisoformat(moment["timestamp"])).days
            ai_text  = await ask_ai(
                f"Бот случайно возвращает старое воспоминание пользователю. "
                f"Оно было сохранено {days_ago} дней назад. "
                f"Напиши одну таинственную короткую фразу — как будто прошлое само нашло тебя."
            )
            caption = f"_{ai_text}_\n\n{days_ago} дней назад."

            if moment["media_type"] == "photo":
                await context.bot.send_photo(
                    chat_id=u["chat_id"], photo=moment["file_id"],
                    caption=caption, parse_mode="Markdown"
                )
            elif moment["media_type"] == "video":
                await context.bot.send_video(
                    chat_id=u["chat_id"], video=moment["file_id"],
                    caption=caption, parse_mode="Markdown"
                )
        except Exception as e:
            logger.error(f"Ghost memory error for {u['user_id']}: {e}")

async def job_month_movie(context):
    """Month Movie — в начале месяца присылать итоги прошлого."""
    today = date.today()
    if today.day != 1:
        return

    prev_month = (today.replace(day=1) - timedelta(days=1))
    year, month = prev_month.year, prev_month.month

    users = db.get_all_users_with_moments()
    for u in users:
        try:
            moments = db.get_monthly_moments(u["user_id"], year, month)
            if not moments:
                continue

            ai_text = await ask_ai(
                f"Прошёл месяц. У пользователя было {len(moments)} сохранённых моментов. "
                f"Напиши атмосферное вступление к месячному обзору его жизни (1-2 предложения)."
            )

            month_names = {1:"Январь",2:"Февраль",3:"Март",4:"Апрель",
                           5:"Май",6:"Июнь",7:"Июль",8:"Август",
                           9:"Сентябрь",10:"Октябрь",11:"Ноябрь",12:"Декабрь"}
            header = f"*{month_names[month]} {year}* — твой месяц в воспоминаниях\n\n_{ai_text}_"

            await context.bot.send_message(
                chat_id=u["chat_id"], text=header, parse_mode="Markdown"
            )
            # Отправить до 5 лучших моментов
            for moment in moments[:5]:
                try:
                    cap = f"_{moment.get('ai_comment', '')}_" if moment.get("ai_comment") else ""
                    if moment["media_type"] == "photo":
                        await context.bot.send_photo(
                            chat_id=u["chat_id"], photo=moment["file_id"],
                            caption=cap, parse_mode="Markdown"
                        )
                    elif moment["media_type"] == "video":
                        await context.bot.send_video(
                            chat_id=u["chat_id"], video=moment["file_id"],
                            caption=cap, parse_mode="Markdown"
                        )
                    await asyncio.sleep(0.5)
                except Exception:
                    pass
        except Exception as e:
            logger.error(f"Month movie error for {u['user_id']}: {e}")

# ─────────────────────────────────────────────
# Запуск
# ─────────────────────────────────────────────

def main():
    app = Application.builder().token(BOT_TOKEN).build()

    # /start
    app.add_handler(CommandHandler("start", cmd_start))

    # Добавление в группу
    app.add_handler(ChatMemberHandler(on_chat_member_update, ChatMemberHandler.MY_CHAT_MEMBER))

    # Все сообщения (группы + личка)
    app.add_handler(MessageHandler(
        ~filters.COMMAND,
        track_message
    ))

    # Фоновые задачи
    jq = app.job_queue
    jq.run_repeating(job_capsules,    interval=60,   first=10)       # каждую минуту
    jq.run_repeating(job_ghost_memory, interval=86400, first=3600)   # раз в сутки
    jq.run_repeating(job_month_movie,  interval=86400, first=7200)   # раз в сутки

    logger.info("Memory Emperor запущен.")
    app.run_polling(allowed_updates=Update.ALL_TYPES)

if __name__ == "__main__":
    main()
