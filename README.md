
import os
import sqlite3
import logging
from telegram import Update
from telegram.ext import (
    Application, CommandHandler, MessageHandler,
    ContextTypes, filters
)

TOKEN = os.getenv("BOT_TOKEN")
ADMIN_IDS = [
    int(x) for x in os.getenv("ADMIN_IDS", "").split(",") if x.strip()
]

logging.basicConfig(level=logging.INFO)

db = sqlite3.connect("movies.db", check_same_thread=False)
db.execute("""
CREATE TABLE IF NOT EXISTS movies (
    code TEXT PRIMARY KEY,
    file_id TEXT NOT NULL
)
""")
db.commit()


def is_admin(user_id):
    return user_id in ADMIN_IDS


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "Assalomu alaykum! 🎬\n\n"
        "Kino ko‘rish uchun kino kodini yuboring.\n"
        "Masalan: 101"
    )


async def add(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_admin(update.effective_user.id):
        return
    await update.message.reply_text(
        "Kino videosini yuboring va izohiga kino kodini yozing."
    )


async def save_movie(message):
    code = (message.caption or "").strip().split()[0] if message.caption else ""
    if not code:
        return False

    media = message.video or message.document
    if not media:
        return False

    db.execute(
        "INSERT OR REPLACE INTO movies(code, file_id) VALUES(?, ?)",
        (code, media.file_id)
    )
    db.commit()
    return code


async def receive_movie(update: Update, context: ContextTypes.DEFAULT_TYPE):
    message = update.effective_message
    user = update.effective_user

    if not user or not is_admin(user.id):
        return

    code = await save_movie(message)
    if code:
        await message.reply_text(f"✅ {code} kodi bilan kino saqlandi!")
    else:
        await message.reply_text("Kino izohiga kod yozing. Masalan: 101")


async def channel_movie(update: Update, context: ContextTypes.DEFAULT_TYPE):
    message = update.channel_post
    if message:
        code = await save_movie(message)
        if code:
            logging.info("Kanal orqali kino saqlandi: %s", code)


async def get_movie(update: Update, context: ContextTypes.DEFAULT_TYPE):
    code = update.message.text.strip()

    row = db.execute(
        "SELECT file_id FROM movies WHERE code = ?", (code,)
    ).fetchone()

    if row:
        await update.message.rep)ly_video(
            video=row[0],
            caption="🎬 Marhamat, kinoingiz!"
        )
    else:
        await update.message.reply_text(
            "❌ Bunday kino kodi topilmadi."
        )


def main():
    if not TOKEN:
        raise RuntimeError("BOT_TOKEN sozlanmagan!")

    app = Application.builder().token(TOKEN).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("add", add))
    app.add_handler(MessageHandler(
        filters.ChatType.PRIVATE & (filters.VIDEO | filters.Document.VIDEO),
        receive_movie
    ))
    app.add_handler(MessageHandler(
        filters.ChatType.CHANNEL & (filters.VIDEO | filters.Document.VIDEO),
        channel_movie
    ))
    app.add_handler(MessageHandler(
        filters.ChatType.PRIVATE & filters.TEXT & ~filters.COMMAND,
        get_movie
    ))

    app.run_polling()


if __name__ == "__main__":
    main()
    
