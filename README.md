# qa-junior-vacancy-bot
Telegram бот для поиска вакансий QA Engineer Junior
# Импортируем нужные библиотеки
from selenium import webdriver  # Импортируем Selenium для управления браузером (открывать сайты, кликать)
from selenium.webdriver.chrome.options import Options  # Для настройки Chrome (например, скрытый режим)
from selenium.webdriver.common.by import By  # Для поиска элементов на странице (кнопки, тексты)
from selenium.webdriver.chrome.service import Service  # Чтобы указать путь к драйверу Chrome
from bs4 import BeautifulSoup  # Для разбора HTML (вытаскивать данные со страницы)
import time  # Для пауз (например, ждём загрузку страницы)
import json  # Для работы с JSON-файлами (храним историю вакансий)
import os  # Для работы с файлами (проверка существования, чтение/запись)
from datetime import datetime  # Для получения текущей даты и времени

# === НАСТРОЙКИ ===
SEARCH_URL = "https://hh.ru/search/vacancy?text=qa+engineer+junior&area=1&order_by=publication_time"  # Ссылка на поиск вакансий
CHROMEDRIVER_PATH = "/path/to/chromedriver"  # Путь к chromedriver (замените на свой!)
SEEN_FILE = "seen_vacancies.json"  # Файл для хранения просмотренных вакансий
OUTPUT_FILE = "vacancies.txt"  # Файл для сохранения новых вакансий


# === ФУНКЦИЯ: загрузка уже просмотренных ID вакансий из файла ===
def load_seen_ids():
    try:  # Пробуем выполнить код (на случай ошибок)
        if os.path.exists(SEEN_FILE):  # Если файл существует
            with open(SEEN_FILE, "r", encoding="utf-8") as f:  # Открываем файл на чтение
                return json.load(f)  # Загружаем данные из JSON
    except json.JSONDecodeError:  # Если файл битый (например, пустой)
        return []  # Возвращаем пустой список
    return []  # Если файла нет, тоже возвращаем пустой список


# === ФУНКЦИЯ: сохранить новые ID в файл ===
def save_seen_ids(seen_ids):
    with open(SEEN_FILE, "w", encoding="utf-8") as f:  # Открываем файл на запись
        json.dump(seen_ids, f, ensure_ascii=False, indent=2)  # Сохраняем данные в JSON с отступами


# === ФУНКЦИЯ: получить HTML страницы с помощью Selenium ===
def fetch_page_source():
    # Настраиваем Chrome (без интерфейса, скрываем автоматизацию)
    options = Options()  # Создаём настройки браузера
    options.add_argument("--headless")  # Запуск без окна браузера (в фоне)
    options.add_argument("--disable-blink-features=AutomationControlled")  # Скрываем, что это бот
    options.add_argument("--no-sandbox")  # Для работы на Linux-серверах
    options.add_argument("--disable-dev-shm-usage")  # Чтобы не было ошибок памяти

    service = Service(CHROMEDRIVER_PATH)  # Указываем путь к драйверу
    driver = webdriver.Chrome(service=service, options=options)  # Запускаем Chrome

    try:  # Пробуем выполнить
        driver.get(SEARCH_URL)  # Открываем страницу с вакансиями
        time.sleep(5)  # Ждём 5 секунд (страница загрузится)
        return driver.page_source  # Возвращаем HTML-код страницы
    finally:  # Выполнится в любом случае (даже если ошибка)
        driver.quit()  # Закрываем браузер


# === ФУНКЦИЯ: поиск свежих вакансий и запись в файл ===
def find_new_vacancies():
    html = fetch_page_source()  # Получаем HTML страницы
    soup = BeautifulSoup(html, "html.parser")  # Создаём "суп" для поиска данных
    seen_ids = load_seen_ids()  # Загружаем просмотренные вакансии
    new_seen_ids = seen_ids.copy()  # Копируем список (чтобы не менять исходный)
    found_vacancies = []  # Сюда будем складывать новые вакансии

    vacancies = soup.find_all('div', class_='serp-item')  # Ищем все блоки вакансий

    for index, vacancy in enumerate(vacancies, 1):  # Проходим по каждой вакансии (index - номер)
        title_tag = vacancy.find('a', class_='serp-item__title')  # Ищем заголовок
        date_tag = vacancy.find('span', {'data-qa': 'vacancy-serp__vacancy-date'})  # Ищем дату
        desc = vacancy.get_text()  # Получаем весь текст вакансии

        if not (title_tag and date_tag):  # Если нет заголовка или даты - пропускаем
            continue

        title = title_tag.text.strip()  # Очищаем заголовок от лишних пробелов
        url = title_tag['href'].split('?')[0]  # Берём ссылку и убираем параметры
        date_text = date_tag.text.lower()  # Переводим дату в нижний регистр ("Сегодня" -> "сегодня")

        # Проверяем, что вакансия свежая и ещё не была в списке
        if ("сегодня" in date_text or "час" in date_text) and url not in seen_ids:
            desc_lower = desc.lower()  # Переводим описание в нижний регистр
            # Проверяем, что есть "удалёнка" и зарплата
            if ("удал" in desc_lower or "remote" in desc_lower) and ("от" in desc_lower or "₽" in desc):
                # Форматируем вакансию и добавляем в список
                found_vacancies.append(f"{index}. {title}\n{url}\nДата: {date_text}\n")
                new_seen_ids.append(url)  # Добавляем ссылку в просмотренные

    # Если нашли новые вакансии - сохраняем в файл
    if found_vacancies:
        with open(OUTPUT_FILE, "a", encoding="utf-8") as f:  # Открываем файл для добавления
            f.write(f"\n🔍 Найдено вакансий: {len(found_vacancies)} — {datetime.now()}\n")  # Заголовок
            f.write('\n'.join(found_vacancies) + '\n')  # Записываем вакансии

    save_seen_ids(new_seen_ids)  # Сохраняем обновлённый список просмотренных
    return found_vacancies  # Возвращаем найденные вакансии


# ----------------- Telegram-бот -----------------
from telegram import Update, Bot, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Updater, CommandHandler, CallbackContext, CallbackQueryHandler

# === НАСТРОЙКИ ТЕЛЕГРАМ ===
TOKEN = "ВАШ_ТОКЕН_ОТ_BOTFATHER"  # Токен бота (получить у @BotFather)
USER_ID = 123456789  # Ваш ID в Telegram (можно узнать у @userinfobot)


# === КНОПКИ ДЛЯ БОТА ===
def start(update: Update, context: CallbackContext):
    # Создаём кнопку "Найти вакансии"
    keyboard = [[InlineKeyboardButton("🔍 Найти вакансии", callback_data="run_search")]]
    # Отправляем сообщение с кнопкой
    update.message.reply_text("Привет! Я бот для поиска вакансий QA Junior 👋",
                              reply_markup=InlineKeyboardMarkup(keyboard))


# === ОБРАБОТЧИК НАЖАТИЯ НА КНОПКУ ===
def button(update: Update, context: CallbackContext):
    query = update.callback_query  # Получаем информацию о нажатии
    query.answer()  # Подтверждаем нажатие

    if query.data == "run_search":  # Если нажали на кнопку "Найти вакансии"
        results = find_new_vacancies()  # Ищем вакансии
        chat_id = query.message.chat_id  # ID чата (откуда пришло нажатие)

        if results:  # Если вакансии найдены
            msg = "\n\n".join(results)  # Объединяем вакансии в одну строку
            context.bot.send_message(chat_id=chat_id, text=f"📬 Найдено вакансий:\n\n{msg}")
        else:  # Если вакансий нет
            context.bot.send_message(chat_id=chat_id, text="Новых вакансий не найдено.")

        # Отправляем файл с вакансиями (если он есть)
        if os.path.exists(OUTPUT_FILE):
            context.bot.send_document(chat_id=chat_id, document=open(OUTPUT_FILE, "rb"))


# === ЗАПУСК БОТА ===
def run_bot():
    updater = Updater(TOKEN)  # Создаём бота
    dp = updater.dispatcher  # Диспетчер команд

    # Добавляем обработчики:
    dp.add_handler(CommandHandler("start", start))  # Обработчик команды /start
    dp.add_handler(CallbackQueryHandler(button))  # Обработчик нажатий на кнопки

    updater.start_polling()  # Запускаем бота
    print("Бот запущен.")  # Сообщение в консоль
    updater.idle()  # Бот работает до остановки


# === ТЕСТОВЫЙ ЗАПУСК ===
if __name__ == "__main__":
    if "run_bot" in globals():  # Если функция run_bot существует
        run_bot()  # Запускаем бота
    else:  # Иначе тестируем поиск вакансий
        vacancies = find_new_vacancies()  # Ищем вакансии
        if vacancies:  # Если есть результаты
            print("Найдено новых вакансий:")
            for v in vacancies:  # Выводим каждую вакансию
                print(v)
        else:  # Если нет результатов
            print("Новых вакансий не найдено.")
