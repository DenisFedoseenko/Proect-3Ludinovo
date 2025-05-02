import logging
import json
from telegram import Update, InlineKeyboardMarkup, InlineKeyboardButton
from telegram.ext import (
    ApplicationBuilder,
    CommandHandler,
    MessageHandler,
    CallbackQueryHandler,
    filters,
    ContextTypes,
    ConversationHandler,
)

# Настройка логирования
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)
logger = logging.getLogger(__name__)

# Загрузка профессий из JSON-файла
with open('professions.json', 'r', encoding='utf-8') as f:
    professions = json.load(f)

GENDER, AGE, PROFESSION = range(3)


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Привет! Какой у вас пол? (мужской/женский)")
    return GENDER


async def gender(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_gender = update.message.text.lower()
    if user_gender not in ['мужской', 'женский']:
        await update.message.reply_text("Пожалуйста, укажите пол: мужской или женский.")
        return GENDER

    context.user_data['gender'] = user_gender
    await update.message.reply_text("Сколько вам лет?")
    return AGE


async def age(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user_age = int(update.message.text)
        context.user_data['age'] = user_age

        age_group = 'teen' if user_age < 18 else 'adult'
        user_gender = 'male' if context.user_data['gender'] == 'мужской' else 'female'
        job_options = professions[user_gender][age_group]

        keyboard = [
            [InlineKeyboardButton(job['title'], callback_data=job['title'])] for job in job_options
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)

        response = "Вот несколько профессий для вас:"
        await update.message.reply_text(response, reply_markup=reply_markup)

        return PROFESSION

    except ValueError:
        await update.message.reply_text("Пожалуйста, введите ваш возраст числом.")
        return AGE


async def choose_profession(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    selected_profession = query.data
    user_gender = 'male' if context.user_data['gender'] == 'мужской' else 'female'
    age_group = 'teen' if context.user_data['age'] < 18 else 'adult'
    job_options = professions[user_gender][age_group]

    for job in job_options:
        if selected_profession == job['title']:
            await query.message.reply_text(
                f"Поздравляю! Вы выбрали профессию: {selected_profession}. {job['description']}")
            return ConversationHandler.END

    await query.message.reply_text("Пожалуйста, выберите одну из предложенных профессий.")


def main():
    application = ApplicationBuilder().token("7730060034:AAHytxfkTy1HgLmfQOluVROj2b_HHGXpqro").build()

    conv_handler = ConversationHandler(
        entry_points=[CommandHandler("start", start)],
        states={
            GENDER: [MessageHandler(filters.TEXT & ~filters.COMMAND, gender)],
            AGE: [MessageHandler(filters.TEXT & ~filters.COMMAND, age)],
            PROFESSION: [
                CallbackQueryHandler(choose_profession),  # Обработчик нажатий на инлайн-кнопки
            ],
        },
        fallbacks=[],
    )

    application.add_handler(conv_handler)

    application.run_polling()


if __name__ == '__main__':
    main()
