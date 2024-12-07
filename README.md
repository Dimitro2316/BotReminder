import telebot
import schedule
import time
import datetime
import logging

logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

TOKEN = "7707474857:AAH6OIGYNzHbMX9HK3pcr74PNcLtb8A8XU0"

bot = telebot.TeleBot(TOKEN)

reminders = {}


def send_reminder(user_id, reminder_text):
    try:
        bot.send_message(user_id, f"Напоминание: {reminder_text}")
    except telebot.apihelper.ApiException as e:
        logging.error(f"Ошибка отправки напоминания пользователю {user_id}: {e}")


def schedule_reminders():
    for user_id, user_reminders in reminders.items():
        for reminder_id, reminder_data in user_reminders.items():
            time_str = reminder_data['time']
            reminder_text = reminder_data['text']
            try:
                schedule.every().day.at(time_str).do(send_reminder, user_id, reminder_id, reminder_text)
            except ValueError as e:
                logging.error(f"Ошибка парсинга времени для напоминания {reminder_id}: {e}")


def run_scheduler():
    while True:
        schedule.run_pending()
        time.sleep(1)


@bot.message_handler(commands=['start'])
def start_command(message):
    bot.reply_to(message, "Привет! Я бот для напоминаний. Используйте /add, /list, /delete.")


@bot.message_handler(commands=['add'])
def add_reminder(message):
    msg = bot.reply_to(message, "Введите время напоминания в формате ЧЧ:ММ и "
                                "текст напоминания через пробел (например, 14:30 Пойти в магазин).")
    bot.register_next_step_handler(msg, process_add_reminder)


def process_add_reminder(message):
    try:
        time_str, reminder_text = message.text.split(maxsplit=1)
        datetime.datetime.strptime(time_str, "%H:%M")
        user_id = message.chat.id
        reminder_id = len(reminders.get(user_id, {}))
        if user_id not in reminders:
            reminders[user_id] = {}
        reminders[user_id][reminder_id] = {'time': time_str, 'text': reminder_text}
        bot.reply_to(message, f"Напоминание добавлено! Время: {time_str}, Текст: {reminder_text}")
        schedule_reminders()
    except ValueError:
        bot.reply_to(message, "Неверный формат. Используйте ЧЧ:ММ Текст.")
    except Exception as e:
        logging.error(f"Ошибка при добавлении напоминания: {e}")
        bot.reply_to(message, "Ошибка при добавлении напоминания.")


@bot.message_handler(commands=['list'])
def list_reminders(message):
    user_id = message.chat.id
    user_reminders = reminders.get(user_id, {})
    if user_reminders:
        reminder_list = "\n".join(
            f"{reminder_id + 1}. Время: {reminder_data['time']}, Текст: {reminder_data['text']}"
            for reminder_id, reminder_data in user_reminders.items()
        )
        bot.reply_to(message, f"Ваши напоминания:\n{reminder_list}")
    else:
        bot.reply_to(message, "У вас нет напоминаний.")


@bot.message_handler(commands=['delete'])
def delete_reminder(message):
    msg = bot.reply_to(message, "Введите номер напоминания для удаления:")
    bot.register_next_step_handler(msg, process_delete_reminder)


def process_delete_reminder(message):
    try:
        reminder_index = int(message.text) - 1
        user_id = message.chat.id
        user_reminders = reminders.get(user_id, {})
        if reminder_index in user_reminders:
            del user_reminders[reminder_index]
            bot.reply_to(message, "Напоминание удалено!")
            schedule_reminders()
        else:
            bot.reply_to(message, "Напоминание с таким номером не найдено.")
    except ValueError:
        bot.reply_to(message, "Неверный формат номера.")
    except Exception as e:
        logging.error(f"Ошибка удаления напоминания: {e}")
        bot.reply_to(message, "Ошибка при удалении напоминания.")


if __name__ == "__main__":
    schedule_reminders()
    import threading
    thread = threading.Thread(target=run_scheduler)
    thread.start()
    bot.infinity_polling()
