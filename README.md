import os
import html
import logging

from telegram import (
    Update,
    InlineKeyboardButton,
    InlineKeyboardMarkup,
    ReplyKeyboardMarkup,
    ReplyKeyboardRemove,
    KeyboardButton,
)
from telegram.ext import (
    Application,
    CommandHandler,
    CallbackQueryHandler,
    MessageHandler,
    ContextTypes,
    filters,
)

logging.basicConfig(level=logging.INFO)

# Token va guruh ID'sini hostingdagi Environment Variables'dan oladi
TOKEN = os.environ["8504183165:AAGd6qBFuOHlVcFAlkL9u0KrkI5pfARli6U"]
GROUP_ID = int(os.environ["1004399029782"])


# =========================
# O'ZBEKISTON SHAHARLARI
# =========================

UZBEKISTAN_CITIES = [
    "Toshkent",
    "Samarqand",
    "Buxoro",
    "Farg'ona",
    "Andijon",
    "Namangan",
    "Qo'qon",
    "Marg'ilon",
    "Navoiy",
    "Qarshi",
    "Termiz",
    "Jizzax",
    "Urganch",
    "Nukus",
]


# =========================
# ROSSIYA SHAHARLARI
# =========================

RUSSIA_CITIES = [
    "Moskva",
    "Sankt-Peterburg",
    "Kazan",
    "Yekaterinburg",
    "Novosibirsk",
    "Samara",
    "Ufa",
    "Perm",
    "Omsk",
    "Chelyabinsk",
    "Volgograd",
    "Rostov-na-Donu",
    "Krasnodar",
    "Voronej",
    "Saratov",
    "Tyumen",
]


# =========================
# DAVLAT TUGMALARI
# =========================

def country_keyboard(prefix):
    buttons = [
        InlineKeyboardButton(
            "🇺🇿 O'zbekiston",
            callback_data=f"{prefix}:uz"
        ),
        InlineKeyboardButton(
            "🇷🇺 Rossiya",
            callback_data=f"{prefix}:ru"
        ),
    ]

    return InlineKeyboardMarkup([buttons])


# =========================
# SHAHAR TUGMALARI
# =========================

def city_keyboard(prefix, cities, exclude=None):
    buttons = [
        InlineKeyboardButton(
            city,
            callback_data=f"{prefix}:{city}"
        )
        for city in cities
        if city != exclude
    ]

    rows = [
        buttons[i:i + 2]
        for i in range(0, len(buttons), 2)
    ]

    return InlineKeyboardMarkup(rows)


# =========================
# START
# =========================

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data.clear()

    await update.message.reply_text(
        "Ipak Cargo botiga xush kelibsiz! 🚚\n\n"
        "📍 Yuk qayerdan jo'natiladi?\n\n"
        "Davlatni tanlang:",
        reply_markup=country_keyboard("from_country")
    )


# =========================
# QAYERDAN — DAVLAT
# =========================

async def choose_from_country(
    update: Update,
    context: ContextTypes.DEFAULT_TYPE
):
    query = update.callback_query
    await query.answer()

    country = query.data.split(":", 1)[1]

    if country == "uz":
        cities = UZBEKISTAN_CITIES
        country_name = "🇺🇿 O'zbekiston"
    else:
        cities = RUSSIA_CITIES
        country_name = "🇷🇺 Rossiya"

    context.user_data["from_country"] = country

    await query.edit_message_text(
        f"{country_name}\n\n"
        "📍 Shahardan birini tanlang:",
        reply_markup=city_keyboard("from", cities)
    )


# =========================
# QAYERDAN — SHAHAR
# =========================

async def choose_from(
    update: Update,
    context: ContextTypes.DEFAULT_TYPE
):
    query = update.callback_query
    await query.answer()

    city = query.data.split(":", 1)[1]

    context.user_data["from"] = city

    await query.edit_message_text(
        f"📍 Qayerdan: {city}\n\n"
        "🏁 Qayerga yuboriladi?\n\n"
        "Davlatni tanlang:",
        reply_markup=country_keyboard("to_country")
    )


# =========================
# QAYERGA — DAVLAT
# =========================

async def choose_to_country(
    update: Update,
    context: ContextTypes.DEFAULT_TYPE
):
    query = update.callback_query
    await query.answer()

    if "from" not in context.user_data:
        await query.edit_message_text(
            "Iltimos, /start bosing."
        )
        return

    country = query.data.split(":", 1)[1]

    if country == "uz":
        cities = UZBEKISTAN_CITIES
        country_name = "🇺🇿 O'zbekiston"
    else:
        cities = RUSSIA_CITIES
        country_name = "🇷🇺 Rossiya"

    context.user_data["to_country"] = country

    from_city = context.user_data["from"]

    await query.edit_message_text(
        f"📍 Qayerdan: {from_city}\n"
        f"{country_name}\n\n"
        "🏁 Shahardan birini tanlang:",
        reply_markup=city_keyboard(
            "to",
            cities,
            exclude=from_city
        )
    )


# =========================
# QAYERGA — SHAHAR
# =========================

async def choose_to(
    update: Update,
    context: ContextTypes.DEFAULT_TYPE
):
    query = update.callback_query
    await query.answer()

    if "from" not in context.user_data:
        await query.edit_message_text(
            "Iltimos, /start bosing."
        )
        return

    city = query.data.split(":", 1)[1]

    context.user_data["to"] = city
    context.user_data["awaiting_phone"] = True

    await query.edit_message_text(
        f"📍 Qayerdan: {context.user_data['from']}\n"
        f"🏁 Qayerga: {city}"
    )

    keyboard = ReplyKeyboardMarkup(
        [
            [
                KeyboardButton(
                    "📞 Raqamni yuborish",
                    request_contact=True
                )
            ]
        ],
        resize_keyboard=True,
        one_time_keyboard=True,
    )

    await query.message.reply_text(
        "Siz bilan bog'lanishimiz uchun telefon "
        "raqamingizni yuboring.\n\n"
        "📞 Tugmani bosing yoki raqamni yozing:",
        reply_markup=keyboard,
    )


# =========================
# TELEFON RAQAMI
# =========================

async def get_phone(
    update: Update,
    context: ContextTypes.DEFAULT_TYPE
):
    if not context.user_data.get("awaiting_phone"):
        await update.message.reply_text(
            "Buyurtma berish uchun /start bosing."
        )
        return

    if update.message.contact:
        phone = update.message.contact.phone_number
    else:
        phone = update.message.text.strip()

    user = update.effective_user

    name = html.escape(user.full_name)

    username = (
        f"@{user.username}"
        if user.username
        else "yo'q"
    )

    from_city = context.user_data.get("from", "Noma'lum")
    to_city = context.user_data.get("to", "Noma'lum")

    text = (
        "🆕 <b>YANGI BUYURTMA</b>\n\n"
        f"📍 Qayerdan: <b>{html.escape(from_city)}</b>\n"
        f"🏁 Qayerga: <b>{html.escape(to_city)}</b>\n\n"
        f"👤 Mijoz: "
        f'<a href="tg://user?id={user.id}">{name}</a>\n'
        f"🔗 Username: {html.escape(username)}\n"
        f"📞 Telefon: <b>{html.escape(phone)}</b>"
    )

    await context.bot.send_message(
        chat_id=GROUP_ID,
        text=text,
        parse_mode="HTML"
    )

    await update.message.reply_text(
        "✅ Buyurtmangiz qabul qilindi!\n\n"
        "Tez orada siz bilan bog'lanamiz.",
        reply_markup=ReplyKeyboardRemove(),
    )

    context.user_data.clear()


# =========================
# BOTNI ISHGA TUSHIRISH
# =========================

def main():
    app = Application.builder().token(TOKEN).build()

    app.add_handler(
        CommandHandler("start", start)
    )

    app.add_handler(
        CommandHandler("buyurtma", start)
    )

    app.add_handler(
        CallbackQueryHandler(
            choose_from_country,
            pattern=r"^from_country:"
        )
    )

    app.add_handler(
        CallbackQueryHandler(
            choose_from,
            pattern=r"^from:"
        )
    )

    app.add_handler(
        CallbackQueryHandler(
            choose_to_country,
            pattern=r"^to_country:"
        )
    )

    app.add_handler(
        CallbackQueryHandler(
            choose_to,
            pattern=r"^to:"
        )
    )

    app.add_handler(
        MessageHandler(
            filters.CONTACT
            | (filters.TEXT & ~filters.COMMAND),
            get_phone
        )
    )

    app.run_polling()


if __name__ == "__main__":
    main()
