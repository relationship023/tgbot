import os
import re
import sqlite3
import random
import requests
from aiocryptopay import AioCryptoPay, Networks
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup, ReplyKeyboardMarkup
from telegram.ext import Application, CommandHandler, MessageHandler, CallbackQueryHandler, filters, CallbackContext
from dotenv import load_dotenv
from datetime import datetime
import logging
import aiofiles
import shutil

# Настройка логирования
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.DEBUG
)

logger = logging.getLogger(__name__)

def log_message(message):
    logger.info(message)

# Загрузка токенов из файла окружения
load_dotenv('C:/Users/user/Desktop/telegramshop/token.env')
TOKEN = os.getenv('TELEGRAM_BOT_TOKEN')
CRYPTOBOT_TOKEN = os.getenv('CRYPTOBOT_API_KEY')  # Убедитесь, что токен загружается из файла окружения

# Инициализация клиента AioCryptoPay
crypto_client = AioCryptoPay(token=CRYPTOBOT_TOKEN, network=Networks.MAIN_NET)  # Используйте правильный аргумент 'token'

def generate_unique_id():
    return random.randint(1000000000, 9999999999)

# Функции для работы с базой данных
def register_user(username, telegram_id, referrer_id=None):
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('SELECT * FROM users WHERE telegram_id = ?', (telegram_id,))
    user = cursor.fetchone()
    if user is None:
        user_id = generate_unique_id()
        is_admin = 1 if telegram_id == 1645726282 else 0
        cursor.execute('INSERT INTO users (id, username, telegram_id, balance, is_admin, referrer_id, promo, promo_discount) VALUES (?, ?, ?, ?, ?, ?, ?, ?)', 
                       (user_id, username, telegram_id, 0, is_admin, referrer_id, "", 0))
        conn.commit()
        if referrer_id:
            add_referral_bonus(referrer_id)
    else:
        user_id = user[0]
    conn.close()
    return user_id

def get_user_data(telegram_id):
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('SELECT * FROM users WHERE telegram_id = ?', (telegram_id,))
    user = cursor.fetchone()
    conn.close()
    return user

def get_all_products():
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('SELECT id, name, price, quantity FROM products WHERE quantity > 0')
    products = cursor.fetchall()
    conn.close()
    return products

def get_product_by_id(product_id):
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('SELECT * FROM products WHERE id = ?', (product_id,))
    product = cursor.fetchone()
    conn.close()
    return product

def get_usdt_price_in_rub():
    try:
        response = requests.get('https://api.coingecko.com/api/v3/simple/price?ids=tether&vs_currencies=rub')
        response.raise_for_status()  # Проверка на ошибки HTTP
        data = response.json()
        return data['tether']['rub']
    except requests.exceptions.RequestException as e:
        print(f"Error fetching data from CoinGecko: {e}")
        return None

def convert_rub_to_usd(amount_rub):
    usdt_price_in_rub = get_usdt_price_in_rub()
    if usdt_price_in_rub is None:
        print("Could not fetch USDT price. Using fallback conversion rate.")
        usdt_price_in_rub = 75  # Фиксированный курс на случай ошибки
    amount_usd = amount_rub / usdt_price_in_rub
    return amount_usd

async def create_payment_link(product_name, amount, user_id):
    # Создаем платежное задание
    invoice = await crypto_client.create_invoice(asset='USDT', amount=amount, description=f'Payment for {product_name} by user {user_id}')
    print(invoice.model_dump())  # Выводим атрибуты объекта `invoice`
    return invoice.bot_invoice_url, invoice.invoice_id  # Возвращаем ссылку на оплату и идентификатор счета

async def check_payment_status(invoice_id):
    # Проверяем статус платежа, используя идентификатор счета
    invoices = await crypto_client.get_invoices(invoice_ids=[invoice_id])
    if invoices and len(invoices) > 0:
        return invoices[0].status == 'paid'
    return False

def get_product_by_name(product_name):
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('SELECT id, name, price, quantity FROM products WHERE name = ?', (product_name,))
    product = cursor.fetchone()
    conn.close()
    return product

def parse_payload(payload):
    parts = payload.split('_')
    user_id = int(parts[0])
    product_id = int(parts[1])
    quantity = int(parts[2])
    logger.info(f"Parsed payload: user_id={user_id}, product_id={product_id}, quantity={quantity}")
    return user_id, product_id, quantity

def update_purchase_history(user_id, product_id, quantity):
    try:
        logger.info("Connecting to database")
        conn = sqlite3.connect(r'C:\Users\user\Desktop\telegramshop\baza.db')
        cursor = conn.cursor()
        
        logger.info(f"Inserting purchase history: user_id={user_id}, product_id={product_id}, quantity={quantity}")
        cursor.execute('INSERT INTO purchase_history (user_id, product_id, quantity, purchase_date) VALUES (?, ?, ?, datetime("now", "localtime"))', 
                       (user_id, product_id, quantity))
        conn.commit()
        
        cursor.execute('SELECT * FROM purchase_history WHERE user_id = ? AND product_id = ?', 
                       (user_id, product_id))
        history = cursor.fetchall()
        logger.info(f"Inserted purchase history: {history}")

        # Проверка на наличие данных
        cursor.execute('SELECT * FROM purchase_history WHERE user_id = ? ORDER BY purchase_date DESC', 
                       (user_id,))
        full_history = cursor.fetchall()
        logger.info(f"Full purchase history for user_id {user_id}: {full_history}")
    except sqlite3.Error as e:
        logger.error(f"SQLite error: {e}")
    finally:
        logger.info("Closing database connection")
        conn.close()

def update_product_quantity(product_id, quantity):
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('UPDATE products SET quantity = ? WHERE id = ?', (quantity, product_id))
    conn.commit()
    conn.close()

def get_cookies_from_file(product_name, quantity):
    filepath = f"{product_name}.txt"
    if not os.path.exists(filepath):
        return None
    with open(filepath, 'r', encoding='utf-8') as file:
        lines = file.readlines()
    if len(lines) < quantity:
        return None
    cookies = lines[:quantity]
    with open(filepath, 'w', encoding='utf-8') as file:
        file.writelines(lines[quantity:])
    return cookies

async def handle_successful_payment(payload):
    logger.info(f"Handling successful payment with payload: {payload}")
    user_id, product_id, quantity = parse_payload(payload)
    logger.info(f"Parsed payload: user_id={user_id}, product_id={product_id}, quantity={quantity}")
    
    product = get_product_by_id(product_id)
    if product:
        logger.info(f"Product found: {product}")
    else:
        logger.error(f"Product not found: product_id={product_id}")
        return None, None
    
    if product[3] >= quantity:
        cookies = get_cookies_from_file(product[1], quantity)
        if not cookies:
            logger.error(f"Not enough cookies available for product: {product[1]}")
            return None, None
        
        # Уменьшаем количество товара только после успешного получения cookies
        update_product_quantity(product_id, product[3] - quantity)
        
        purchase_id = generate_unique_id()
        logger.info(f"Generated purchase id: {purchase_id}")
        filename = f"Покупка{purchase_id}.txt"
        filepath = os.path.join(os.getcwd(), "Purchases", filename)
        os.makedirs(os.path.dirname(filepath), exist_ok=True)
        
        with open(filepath, 'w', encoding='utf-8') as file:  # Указываем кодировку utf-8
            file.writelines(cookies)
        
        # Обновляем историю покупок
        update_purchase_history(user_id, product_id, quantity)
        return filepath, purchase_id
    else:
        logger.error(f"Insufficient product quantity: Available={product[3]}, Requested={quantity}")
        return None, None

async def start(update: Update, context: CallbackContext) -> None:
    referrer_id = context.args[0] if context.args else None
    menu_keyboard = [
        ['Все категории', 'Наличие товара'],
        ['Правила', 'Профиль'],
        ['Правила замен']
    ]
    reply_markup = ReplyKeyboardMarkup(menu_keyboard, resize_keyboard=True)
    user = update.message.from_user
    username = user.username or user.first_name
    telegram_id = user.id
    register_user(username, telegram_id, referrer_id)
    await update.message.reply_text('Пожалуйста, выберите один из пунктов меню.', reply_markup=reply_markup)

async def profile(update: Update, context: CallbackContext) -> None:
    if update.message:
        user = update.message.from_user
    elif update.callback_query:
        user = update.callback_query.from_user

    username = user.username or user.first_name
    telegram_id = user.id
    user_id = register_user(username, telegram_id)
    user_data = get_user_data(telegram_id)

    profile_message = f"❤️ Имя: {user_data[1]}\n🔑 ID: {user_data[0]}\n💰 Ваш баланс: {user_data[3]} ₽"
    if user_data[4] == 1:
        profile_message += "\n🔒 Вы являетесь администратором."
    if user_data[6]:  # Проверка наличия активированного промокода
        profile_message += f"\nПромокод применен, скидка {user_data[7]}%"
    
    profile_menu = InlineKeyboardMarkup([
        [InlineKeyboardButton('История покупок', callback_data='history')],
        [InlineKeyboardButton('Пополнение баланса', callback_data='balance')],
        [InlineKeyboardButton('Реферальная система', callback_data='referral')],
        [InlineKeyboardButton('Активировать промокод', callback_data='activate_promo')],
        [InlineKeyboardButton('Админка', callback_data='admin_panel')] if user_data[4] == 1 else []
    ])
    await update.message.reply_text(profile_message, reply_markup=profile_menu) if update.message else await update.callback_query.edit_message_text(profile_message, reply_markup=profile_menu)

async def admin_panel(update: Update, context: CallbackContext) -> None:
    query = update.callback_query
    await query.answer()
    
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    
    # Получаем данные для админки
    cursor.execute('SELECT SUM(balance) FROM users')
    total_spent = cursor.fetchone()[0] or 0

    cursor.execute('SELECT COUNT(*) FROM purchase_history')
    total_purchases = cursor.fetchone()[0] or 0

    cursor.execute('SELECT code, discount, activations FROM promo_codes')
    promo_codes = cursor.fetchall()

    # Получаем инструкции для админов
    admin_instructions = (
        "Инструкции для администраторов:\n"
        "/add_promo <название> <скидка в процентах> <количество активаций> - Добавить промокод\n"
        "/delete_promo <название> - Удалить промокод\n"
        "/add_balance <id> <сумма> - Добавить баланс пользователю\n"
        "/remove_balance <id> <сумма> - Уменьшить баланс пользователя\n"
        "/zaliv - Залив файла с куками\n"
        "/reboot <название> <новое название> <цена> <количество | check> - Переименовать товар и изменить параметры\n"
        "/rebootprice <название> <цена> <количество | check> - Изменить цену и количество товара\n"
    )
    
    conn.close()
    
    promo_message = "Действующие промокоды:\n"
    for promo in promo_codes:
        promo_message += f"{promo[0]} - Скидка: {promo[1]}%, Активаций: {promo[2]}\n"
    
    admin_message = (
        f"Общая сумма потраченных средств: {total_spent} ₽\n"
        f"Общее количество покупок: {total_purchases}\n\n"
        f"{promo_message}\n\n"
        f"{admin_instructions}"
    )
    
    admin_menu = InlineKeyboardMarkup([
        [InlineKeyboardButton('Назад к профилю', callback_data='back_to_profile')]
    ])
    await query.edit_message_text(text=admin_message, reply_markup=admin_menu)

async def add_promo(update: Update, context: CallbackContext) -> None:
    user_id = update.message.from_user.id
    if not is_admin(user_id):
        await update.message.reply_text('У вас нет прав для выполнения этой команды.')
        return
    try:
        promo_code = context.args[0]
        discount = int(context.args[1])
        activations = int(context.args[2])
    except (IndexError, ValueError):
        await update.message.reply_text('Использование: /add_promo <название> <скидка в процентах> <количество активаций>')
        return

    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('INSERT INTO promo_codes (code, discount, activations) VALUES (?, ?, ?)', (promo_code, discount, activations))
    conn.commit()
    conn.close()
    await update.message.reply_text(f'Промокод {promo_code} добавлен со скидкой {discount}% и количеством активаций {activations}.')

async def delete_promo(update: Update, context: CallbackContext) -> None:
    user_id = update.message.from_user.id
    if not is_admin(user_id):
        await update.message.reply_text('У вас нет прав для выполнения этой команды.')
        return
    try:
        promo_code = context.args[0]
    except IndexError:
        await update.message.reply_text('Использование: /delete_promo <название>')
        return
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('DELETE FROM promo_codes WHERE code = ?', (promo_code,))
    conn.commit()
    conn.close()
    await update.message.reply_text(f'Промокод {promo_code} удален.')

async def show_referral(update: Update, context: CallbackContext) -> None:
    query = update.callback_query
    await query.answer()
    user_id = query.from_user.id
    user_data = get_user_data(user_id)
    referral_link = f"https://t.me/{context.bot.username}?start={user_id}"
    
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('SELECT COUNT(*) FROM users WHERE referrer_id = ?', (user_data[0],))
    referral_count = cursor.fetchone()[0]
    conn.close()
    
    referral_message = f"Ваша реферальная ссылка: {referral_link}\nКоличество приглашенных пользователей: {referral_count}"
    referral_menu = InlineKeyboardMarkup([
        [InlineKeyboardButton('Назад к профилю', callback_data='back_to_profile')],
        [InlineKeyboardButton('Топ рефералов', callback_data='top_referrals')],
        [InlineKeyboardButton('История рефералов', callback_data='referral_history')],
        [InlineKeyboardButton('История зачислений', callback_data='referral_balance_history')]  # Разделяем кнопки для истории рефералов и зачислений
    ])
    await query.edit_message_text(text=referral_message, reply_markup=referral_menu)

async def show_stock(update: Update, context: CallbackContext) -> None:
    products = get_all_products()
    if not products:
        await update.message.reply_text('Товар закончился.')
        return
    message = '➖➖➖Игры➖➖➖\n'
    for product in products:
        message += f"{product[1]} | {float(product[2]) / 100:.2f} ₽ | {product[3]} шт.\n"
    await update.message.reply_text(message)

async def select_product(update: Update, context: CallbackContext) -> None:
    query = update.callback_query
    await query.answer()
    product_id = query.data.split('_')[2]
    context.user_data['selected_product'] = product_id
    context.user_data['awaiting_purchase_quantity'] = True
    product = get_product_by_id(product_id)
    if product:
        await query.message.reply_text(f"Вы выбрали {product[1]}. Цена: {float(product[2]) / 100:.2f} ₽. В наличии: {product[3]} шт.\nПожалуйста, введите количество, которое вы хотите купить:")

async def show_categories(update: Update, context: CallbackContext) -> None:
    products = get_all_products()
    if not products:
        await update.message.reply_text('Товары отсутствуют.')
        return
    
    keyboard = []
    for product in products:
        try:
            # Убедимся, что цена это число, и преобразуем ее в рубли
            price = float(product[2]) / 100  # Преобразование цены в рубли
            keyboard.append([
                InlineKeyboardButton(f"{product[1]} | {price:.2f} ₽ | {product[3]} шт.", callback_data=f"select_product_{product[0]}")
            ])
        except ValueError:
            # Если цена не число, пропускаем этот товар
            continue
    
    reply_markup = InlineKeyboardMarkup(keyboard)
    await update.message.reply_text('Выберите товар:', reply_markup=reply_markup)

async def handle_purchase_quantity(update: Update, context: CallbackContext) -> None:
    try:
        quantity = int(update.message.text)
    except ValueError:
        await update.message.reply_text("Пожалуйста, введите корректное количество.")
        return

    product_id = context.user_data.get('selected_product')
    if not product_id:
        await update.message.reply_text("Пожалуйста, сначала выберите товар.")
        return

    context.user_data['awaiting_purchase_quantity'] = False
    await process_purchase_quantity(update, context, product_id, quantity)

async def product_callback(update: Update, context: CallbackContext) -> None:
    query = update.callback_query
    await query.answer()
    product_id = query.data.split('_')[2]
    product = get_product_by_id(product_id)
    if product:
        context.user_data['selected_product'] = product_id
        await query.message.reply_text(
            f"Вы выбрали {product[1]}. Цена: {float(product[2]) / 100:.2f} ₽. В наличии: {product[3]} шт.\n"
            "Пожалуйста, введите количество, которое вы хотите купить:"
        )
    else:
        await query.message.reply_text("Товар не найден.")

async def process_purchase_quantity(update: Update, context: CallbackContext, product_id: str, quantity: int) -> None:
    product = get_product_by_id(product_id)
    if product and quantity <= product[3]:  # Проверка наличия достаточного количества товара
        user_id = context.user_data.get('user_id_for_promo')
        promo_code = context.user_data.get('promo', '')
        discount = context.user_data.get('promo_discount', 0)

        # Проверка использован ли промокод и вычисление общей стоимости
        total_price_rub = (float(product[2]) / 100) * quantity  # Общая сумма без скидки
        if discount > 0:
            total_price_rub *= (1 - discount / 100)  # Применение скидки

        context.user_data['pending_purchase'] = (product_id, quantity, total_price_rub)
        payment_message = (
            f"Покупка {quantity} шт. \"{product[1]}\"\n"
            f"Сумма к оплате: {total_price_rub:.2f} руб" + (f" (Скидка: {discount}%)" if discount > 0 else "") + "\n"
            "Выберите способ оплаты:"
        )
        reply_markup = InlineKeyboardMarkup([
            [InlineKeyboardButton("Оплата через баланс", callback_data=f"pay_balance_{product_id}_{quantity}_{total_price_rub:.2f}")],
            [InlineKeyboardButton("Оплата через CryptoBot", callback_data=f"pay_cryptobot_{product_id}_{quantity}_{total_price_rub:.2f}")]
        ])
        await update.message.reply_text(payment_message, reply_markup=reply_markup)

        # Сброс промокода после использования
        if discount > 0:
            conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
            cursor = conn.cursor()
            cursor.execute('INSERT INTO used_promocodes (user_id, promo_code) VALUES (?, ?)', (user_id, promo_code))
            conn.commit()
            conn.close()
    else:
        await update.message.reply_text(f"Недопустимое количество. В наличии только {product[3]} шт.")

def get_purchase_history(user_id, page):
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    offset = page * 5
    cursor.execute('SELECT * FROM purchase_history WHERE user_id = ? ORDER BY purchase_date DESC LIMIT 5 OFFSET ?', (user_id, offset))
    history = cursor.fetchall()
    conn.close()
    return history

def get_user_id_by_telegram_id(telegram_id):
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('SELECT id FROM users WHERE telegram_id = ?', (telegram_id,))
    user = cursor.fetchone()
    conn.close()
    return user[0] if user else None

async def handle_check_payment(update: Update, context: CallbackContext) -> None:
    query = update.callback_query
    await query.answer()
    invoice_id = int(query.data.split('_')[2])
    
    if await check_payment_status(invoice_id):
        payload = context.user_data.get('pending_payload')
        if payload is None:
            await query.message.reply_text("Этот счет уже был обработан.")
            return

        filepath, purchase_id = await handle_successful_payment(payload)
        if filepath:
            await context.bot.send_document(chat_id=query.from_user.id, document=open(filepath, 'rb'))
            await query.message.reply_text(f"Спасибо за покупку! ID покупки: {purchase_id}. Кнопки оплаты больше не действительны.")
            # Блокировка кнопок; замените на фактический код, который показывает, что оплата завершена
            await query.edit_message_reply_markup(reply_markup=InlineKeyboardMarkup([
                [InlineKeyboardButton("Оплата завершена", callback_data="payment_completed", disabled=True)]
            ]))  # Кнопки больше не активны
            context.user_data['pending_payload'] = None  # Удаляем pending_payload после успешной оплаты
            context.user_data['purchase_completed'] = True
        else:
            await query.message.reply_text("Ошибка при обработке заказа.")
    else:
        await query.message.reply_text("Оплата не подтверждена. Попробуйте еще раз позже.")

async def handle_balance_payment(update: Update, context: CallbackContext) -> None:
    query = update.callback_query
    await query.answer()
    data = query.data.split('_')

    product_id = int(data[2])
    quantity = int(data[3])
    total_price_rub = float(data[4])
    user_data = get_user_data(query.from_user.id)

    # Проверяем, использовал ли пользователь промокод
    if user_data[6]:  # user_data[6] хранит промокод
        # Если использовал, проверяем, активен ли он еще
        if is_promocode_active(user_data[6]):
            # Проверяем количество активаций для промокода
            if get_promocode_activations(user_data[6]) > 0:
                # Вычитаем скидку, если промокод активен
                total_price_rub *= (1 - user_data[7] / 100)  # user_data[7] хранит скидку
            else:
                await query.message.reply_text("Срок действия вашего промокода истек.")
                return
        else:
            await query.message.reply_text("Промокод недействителен.")
            return

    if user_data[3] >= total_price_rub:
        # Обрабатываем успешную покупку
        payload = f"{query.from_user.id}_{product_id}_{quantity}"
        filepath, purchase_id = await handle_successful_payment(payload)

        if filepath:
            # Списываем деньги с баланса только если обработка прошла успешно
            conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
            cursor = conn.cursor()
            cursor.execute('UPDATE users SET balance = balance - ? WHERE telegram_id = ?', (total_price_rub, query.from_user.id))
            conn.commit()
            conn.close()

            await context.bot.send_document(chat_id=query.from_user.id, document=open(filepath, 'rb'))
            await query.message.reply_text(f"Спасибо за покупку!\nID покупки: {purchase_id}")

            # Уменьшаем количество активаций промокода после успешного использования
            if user_data[6]:
                decrease_promocode_activations(user_data[6])
        else:
            await query.message.reply_text("Ошибка при обработке заказа.")
    else:
        await query.message.reply_text("Недостаточно средств на балансе.")


async def handle_cryptobot_payment(update: Update, context: CallbackContext) -> None:
    query = update.callback_query
    await query.answer()
    data = query.data.split('_')
    
    product_id = int(data[2])
    quantity = int(data[3])
    total_price_rub = float(data[4])  # Используем float для работы с десятичными значениями
    user_data = get_user_data(query.from_user.id)

    discount = user_data[7]  # Получаем скидку из данных пользователя
    if discount > 0:
        total_price_rub *= (1 - discount / 100)  # Применение скидки
    
    total_price_usd = convert_rub_to_usd(total_price_rub)

    if total_price_usd < 0.1:  # Минимальная сумма для оплаты через CryptoBot
        await query.message.reply_text("Сумма слишком мала для оплаты через CryptoBot. Пожалуйста, выберите большее количество.")
        return

    product = get_product_by_id(product_id)
    if product:
        try:
            payment_link, invoice_id = await create_payment_link(product[1], total_price_usd, query.from_user.id)
            await query.message.reply_text(
                f"Оплата через CryptoBot:\n{payment_link}",
                reply_markup=InlineKeyboardMarkup([
                    [InlineKeyboardButton("Я оплатил", callback_data=f"check_payment_{invoice_id}")]
                ])
            )
            context.user_data['pending_invoice'] = invoice_id
            context.user_data['pending_payload'] = f"{query.from_user.id}_{product_id}_{quantity}"
        except Exception as e:
            await query.message.reply_text(f"Ошибка при создании платежной ссылки: {e}")
    else:
        await query.message.reply_text("Товар не найден.")

async def show_rules(update: Update, context: CallbackContext) -> None:
    rules_message = (
        "Политика использования\n"
        "Цель магазина: Магазин предоставляет услуги по продаже игровых донатов для улучшения игрового опыта в различных онлайн-играх.\n\n"
        "Правила использования: Пользователи обязаны соблюдать все применимые законы и правила платформ, на которых они используют купленные донаты. Запрещены попытки обмана, мошенничество и другие недопустимые действия.\n\n"
        "Прием платежей: Мы принимаем платежи через указанные методы, обеспечивая безопасность и конфиденциальность ваших данных.\n\n"
        "Обязательства магазина: Магазин обязуется предоставить вам купленный игровой донат после успешной оплаты.\n\n"
        "Ответственность пользователя: Вы несете ответственность за предоставление правильной информации при заказе услуги. Пользователи должны предоставить корректные данные для успешного выполнения заказа.\n\n"
        "Запрещенные действия: Запрещены действия, направленные на мошенничество, включая попытки возврата средств после получения услуги.\n\n"
        "Политика возврата\n"
        "Условия возврата: Вы можете запросить возврат средств, если полученные услуги были некачественными или не предоставлены в соответствии с условиями заказа.\n\n"
        "Процедура возврата: Для запроса возврата, свяжитесь с нашей службой поддержки по указанным контактным данным. Мы рассмотрим ваш запрос и произведем возврат средств на вашу карту/кошелек.\n\n"
        "Сроки возврата: Мы постараемся рассмотреть ваш запрос в кратчайшие сроки.\n\n"
        "Политика конфиденциальности\n"
        "Сбор информации: Мы можем собирать определенную информацию от пользователей для обработки заказов и улучшения сервиса.\n\n"
        "Использование информации: Мы обеспечиваем безопасное и конфиденциальное хранение ваших данных. Информация будет использована исключительно для обработки заказов и обратной связи с вами.\n\n"
        "Разглашение информации: Мы не раскроем вашу информацию третьим лицам, за исключением случаев, предусмотренных законом или в случаях, когда это необходимо для выполнения заказа (например, передача информации платежным системам).\n\n"
        "Согласие пользователя: Используя наши услуги, вы соглашаетесь с нашей политикой конфиденциальности."
    )
    await update.message.reply_text(rules_message)

async def show_replacement_request(update: Update, context: CallbackContext) -> None:
    replacement_request_message = (
        "Если вам попался нерабочий товар, пишите по замене - wweather\n"
        "ОФОРМЛЕНИЕ ЗАЯВКИ НА ЗАМЕНУ\n"
        "1. Товар\n"
        "2. Количество нерабочих аккаунтов\n"
        "3. Сумма в рублях нерабочих аккаунтов\n"
        "4. Ваш id в боте"
    )
    await update.message.reply_text(replacement_request_message)

async def show_balance(update: Update, context: CallbackContext) -> None:
    query = update.callback_query
    await query.answer()
    payment_message = (
        "Для пополнения баланса писать - @wweath3r\n"
        "Пополнение через CryptoBot - ✅"
    )
    balance_menu = InlineKeyboardMarkup([
        [InlineKeyboardButton('Оплатить через CryptoBot', callback_data='top_up_balance')],
        [InlineKeyboardButton('Назад к профилю', callback_data='back_to_profile')]
    ])
    await query.edit_message_text(text=payment_message, reply_markup=balance_menu)

async def show_history(update: Update, context: CallbackContext) -> None:
    user = update.callback_query.from_user
    telegram_id = user.id
    user_id = get_user_id_by_telegram_id(telegram_id)
    if not user_id:
        await update.callback_query.message.reply_text("Пользователь не найден.")
        return

    page = context.user_data.get('history_page', 0)
    history = get_purchase_history(user_id, page)

    if not history:
        await update.callback_query.message.reply_text("История покупок пуста.")
        return

    message = "📜 История покупок:\n"
    for record in history:
        message += (
            f"Наименование: {record[1]}\n"
            f"Количество: {record[2]}\n"
            f"Дата: {record[3]}\n"
            f"Сумма: {record[4] / 100:.2f} руб\n"
            f"ID покупки: {record[0]}\n\n"
        )

    reply_markup = InlineKeyboardMarkup([
        [InlineKeyboardButton("Предыдущая страница", callback_data="history_prev"),
         InlineKeyboardButton("Следующая страница", callback_data="history_next")],
        [InlineKeyboardButton("Назад к профилю", callback_data="back_to_profile")]
    ])

    await update.callback_query.message.reply_text(message, reply_markup=reply_markup)

async def history_prev(update: Update, context: CallbackContext) -> None:
    context.user_data['history_page'] = context.user_data.get('history_page', 0) - 1
    await show_history(update, context)

async def history_next(update: Update, context: CallbackContext) -> None:
    context.user_data['history_page'] = context.user_data.get('history_page', 0) + 1
    await show_history(update, context)

async def back_to_profile(update: Update, context: CallbackContext) -> None:
    query = update.callback_query
    await query.answer()
    await profile(update, context)

# Функции для работы с базой данных
def register_user(username, telegram_id, referrer_id=None):
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('SELECT * FROM users WHERE telegram_id = ?', (telegram_id,))
    user = cursor.fetchone()
    if user is None:
        user_id = generate_unique_id()
        is_admin = 1 if telegram_id == 1645726282 else 0
        cursor.execute('INSERT INTO users (id, username, telegram_id, balance, is_admin, referrer_id, promo, promo_discount) VALUES (?, ?, ?, ?, ?, ?, ?, ?)', 
                       (user_id, username, telegram_id, 0, is_admin, referrer_id, "", 0))
        conn.commit()
        if referrer_id:
            add_referral_bonus(referrer_id)
    else:
        user_id = user[0]
    conn.close()
    return user_id

def create_database():
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY,
        username TEXT,
        telegram_id INTEGER UNIQUE,
        balance INTEGER DEFAULT 0,
        is_admin INTEGER DEFAULT 0,
        referrer_id INTEGER,
        promo TEXT DEFAULT '',
        promo_discount INTEGER DEFAULT 0,
        referral_count INTEGER DEFAULT 0
    )
    ''')
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS products (
        id INTEGER PRIMARY KEY,
        name TEXT,
        price INTEGER,
        quantity INTEGER
    )
    ''')
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS purchase_history (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        product_id INTEGER,
        quantity INTEGER,
        purchase_date DATETIME,
        total_price INTEGER,
        FOREIGN KEY (user_id) REFERENCES users(id),
        FOREIGN KEY (product_id) REFERENCES products(id)
    )
    ''')
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS referral_history (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        referrer_id INTEGER,
        referred_id INTEGER,
        amount INTEGER,
        date DATETIME,
        FOREIGN KEY (referrer_id) REFERENCES users(id),
        FOREIGN KEY (referred_id) REFERENCES users(id)
    )
    ''')
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS promo_codes (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        code TEXT,
        discount INTEGER,
        activations INTEGER
    )
    ''')
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS used_promocodes (
        user_id INTEGER,
        promo_code TEXT,
        PRIMARY KEY (user_id, promo_code),
        FOREIGN KEY (user_id) REFERENCES users(id)
    )
    ''')
    conn.commit()
    conn.close()

def update_database():
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    try:
        cursor.execute('ALTER TABLE users ADD COLUMN promo TEXT DEFAULT ""')
        cursor.execute('ALTER TABLE users ADD COLUMN promo_discount INTEGER DEFAULT 0')
        logger.info("Columns 'promo' and 'promo_discount' added successfully.")
    except sqlite3.OperationalError as e:
        logger.warning(f"Columns 'promo' or 'promo_discount' already exist or another operational error: {e}")
    conn.commit()
    conn.close()

def is_promocode_used(user_id, promo_code):
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('SELECT * FROM used_promocodes WHERE user_id = ? AND promo_code = ?', (user_id, promo_code))
    used = cursor.fetchone()
    conn.close()
    return used is not None

async def reboot(update: Update, context: CallbackContext) -> None:
    user_id = update.message.from_user.id
    if not is_admin(user_id):
        await update.message.reply_text('У вас нет прав для выполнения этой команды.')
        return
    try:
        name = context.args[0]
        new_name = context.args[1]
        price = int(float(context.args[2]) * 100)  # Цена в копейках
        if context.args[-1].lower() == 'check':
            file_path = os.path.join(os.path.dirname(__file__), f'{new_name}.txt')
            quantity = count_items_in_file(file_path)
        else:
            quantity = int(context.args[-1])
    except (IndexError, ValueError) as e:
        logger.error(f"Error parsing arguments: {e}")
        await update.message.reply_text('Использование: /reboot <название> <новое название> <цена> <количество | check>')
        return

    # Переименование файла на новое название товара
    old_file_path = os.path.join(os.path.dirname(__file__), f'{name}.txt')
    if os.path.exists(old_file_path):
        new_file_path = os.path.join(os.path.dirname(__file__), f'{new_name}.txt')
        os.rename(old_file_path, new_file_path)

    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('UPDATE products SET name = ?, price = ?, quantity = ? WHERE name = ?', (new_name, price, quantity, name))
    conn.commit()
    conn.close()
    await update.message.reply_text(f'Товар "{name}" изменен на "{new_name}" с ценой {price / 100:.2f} руб и количеством {quantity}.')

async def reboot_price(update: Update, context: CallbackContext) -> None:
    user_id = update.message.from_user.id
    if not is_admin(user_id):
        await update.message.reply_text('У вас нет прав для выполнения этой команды.')
        return
    try:
        name = context.args[0]
        price = int(float(context.args[1]) * 100)  # Цена в копейках
        if context.args[-1].lower() == 'check':
            file_path = os.path.join('path_to_files', f'{name}.txt')
            quantity = count_items_in_file(file_path)
        else:
            quantity = int(context.args[-1])
    except (IndexError, ValueError) as e:
        logger.error(f"Error parsing arguments: {e}")
        await update.message.reply_text('Использование: /rebootprice <название> <цена> <количество | check>')
        return
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('UPDATE products SET price = ?, quantity = ? WHERE name = ?', (price, quantity, name))
    conn.commit()
    conn.close()
    await update.message.reply_text(f'Товар "{name}" изменен с ценой {price / 100:.2f} руб и количеством {quantity}.')

async def add_product(update: Update, context: CallbackContext) -> None:
    user_id = update.message.from_user.id
    if not is_admin(user_id):
        await update.message.reply_text('У вас нет прав для выполнения этой команды.')
        return
    try:
        name = ' '.join(context.args[:-2])
        price = int(float(context.args[-2]) * 100)  # Цена в копейках
        if context.args[-1].lower() == 'check':
            quantity = count_items_in_file(name)
        else:
            quantity = int(context.args[-1])
    except (IndexError, ValueError) as e:
        logger.error(f"Error parsing arguments: {e}")
        await update.message.reply_text('Использование: /add_product <название> <цена> <количество | check>')
        return
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('INSERT INTO products (name, price, quantity) VALUES (?, ?, ?)', (name, price, quantity))
    conn.commit()
    conn.close()
    await update.message.reply_text(f'Товар "{name}" добавлен с ценой {price / 100:.2f} руб и количеством {quantity}.')

def count_items_in_file(file_path):
    try:
        with open(file_path, 'r') as f:
            content = f.read()
            pattern = re.compile(r'_\|WARNING:-DO-NOT-SHARE-THIS.--Sharing-this-will-allow-someone-to-log-in-as-you-and-to-steal-your-ROBUX-and-items.\|_[A-Za-z0-9]+')
            matches = pattern.findall(content)
            return len(matches)
    except Exception as e:
        logger.error(f"Error reading file: {e}")
        return 0
	
async def delete_product(update: Update, context: CallbackContext) -> None:
    user_id = update.message.from_user.id
    if not is_admin(user_id):
        await update.message.reply_text('У вас нет прав для выполнения этой команды.')
        return
    try:
        product_name = ' '.join(context.args)
    except IndexError:
        await update.message.reply_text('Использование: /delete_product <название>')
        return
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('DELETE FROM products WHERE name = ?', (product_name,))
    conn.commit()
    conn.close()
    await update.message.reply_text(f'Товар "{product_name}" удален.')

async def list_products(update: Update, context: CallbackContext) -> None:
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('SELECT name, price, quantity FROM products')
    products = cursor.fetchall()
    conn.close()
    if not products:
        await update.message.reply_text('Товары отсутствуют.')
        return
    message = 'Список товаров:\n'
    for product in products:
        if product[3] > 0:
            message += f'Название: {product[0]}, Цена: {product[1] / 100:.2f} руб, Количество: {product[2]}\n'
    await update.message.reply_text(message)

def is_admin(user_id):
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('SELECT is_admin FROM users WHERE telegram_id = ?', (user_id,))
    result = cursor.fetchone()
    conn.close()
    return result and result[0] == 1

async def make_admin(update: Update, context: CallbackContext) -> None:
    user_id = update.message.from_user.id
    if not is_admin(user_id):
        await update.message.reply_text('У вас нет прав для выполнения этой команды.')
        return
    try:
        target_id = int(context.args[0])
    except (IndexError, ValueError):
        await update.message.reply_text('Использование: /make_admin <telegram_id>')
        return
    set_admin(target_id, 1)
    await update.message.reply_text(f'Пользователь с ID {target_id} назначен администратором.')

async def remove_admin(update: Update, context: CallbackContext) -> None:
    user_id = update.message.from_user.id
    if not is_admin(user_id):
        await update.message.reply_text('У вас нет прав для выполнения этой команды.')
        return
    try:
        target_id = int(context.args[0])
    except (IndexError, ValueError):
        await update.message.reply_text('Использование: /remove_admin <telegram_id>')
        return
    set_admin(target_id, 0)
    await update.message.reply_text(f'Пользователь с ID {target_id} снят с должности администратора.')

def set_admin(telegram_id, is_admin):
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('UPDATE users SET is_admin = ? WHERE telegram_id = ?', (is_admin, telegram_id))
    conn.commit()
    conn.close()

async def check_payment_status(invoice_id):
    # Проверяем статус платежа, используя идентификатор счета
    invoices = await crypto_client.get_invoices(invoice_ids=[invoice_id])
    if invoices and len(invoices) > 0:
        return invoices[0].status == 'paid'
    return False

async def handle_successful_payment(payload):
    logger.info(f"Handling successful payment with payload: {payload}")
    user_id, product_id, quantity = parse_payload(payload)
    logger.info(f"Parsed payload: user_id={user_id}, product_id={product_id}, quantity={quantity}")
    
    product = get_product_by_id(product_id)
    if product:
        logger.info(f"Product found: {product}")
    else:
        logger.error(f"Product not found: product_id={product_id}")
        return None, None
    
    if product[3] >= quantity:
        logger.info(f"Sufficient product quantity available: {product[3]}")
        update_product_quantity(product_id, product[3] - quantity)
        cookies = get_cookies_from_file(product[1], quantity)
        if not cookies:
            logger.error(f"Not enough cookies available for product: {product[1]}")
            return None, None
        
        purchase_id = generate_unique_id()
        logger.info(f"Generated purchase id: {purchase_id}")
        filename = f"Покупка{purchase_id}.txt"
        filepath = os.path.join(os.getcwd(), "Purchases", filename)
        os.makedirs(os.path.dirname(filepath), exist_ok=True)
        
        with open(filepath, 'w', encoding='utf-8') as file:  # Указываем кодировку utf-8
            file.writelines(cookies)
        
        # Обновляем историю покупок
        update_purchase_history(user_id, product_id, quantity)
        return filepath, purchase_id
    else:
        logger.error(f"Insufficient product quantity: Available={product[3]}, Requested={quantity}")
        return None, None

async def show_top_referrals(update: Update, context: CallbackContext) -> None:
    query = update.callback_query
    await query.answer()
    
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('''
    SELECT referrer_id, COUNT(referrer_id) AS referral_count
    FROM users
    WHERE referrer_id IS NOT NULL
    GROUP BY referrer_id
    ORDER BY referral_count DESC
    LIMIT 10
    ''')
    top_referrals = cursor.fetchall()
    conn.close()
    
    message = '🏆 Топ 10 рефералов:\n'
    for i, (referrer_id, count) in enumerate(top_referrals, start=1):
        referrer_name = get_user_data(referrer_id)[1]
        message += f"{i}. {referrer_name} - {count} рефералов\n"
    
    referral_menu = InlineKeyboardMarkup([
        [InlineKeyboardButton('Назад к реферальной системе', callback_data='referral')]
    ])
    await query.edit_message_text(text=message, reply_markup=referral_menu)

def add_referral_bonus(referrer_id):
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('UPDATE users SET balance = balance + 5 WHERE id = ?', (referrer_id,))
    cursor.execute('INSERT INTO referral_history (referrer_id, amount, date) VALUES (?, ?, datetime("now", "localtime"))', (referrer_id, 5))
    conn.commit()
    conn.close()

async def show_referral_history(update: Update, context: CallbackContext) -> None:
    user = update.callback_query.from_user
    telegram_id = user.id
    user_id = get_user_id_by_telegram_id(telegram_id)
    if not user_id:
        await update.callback_query.message.reply_text("Пользователь не найден.")
        return

    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('SELECT * FROM referral_history WHERE referrer_id = ?', (user_id,))
    history = cursor.fetchall()
    conn.close()

    if not history:
        await update.callback_query.message.reply_text("История рефералов пуста.")
        return

    message = "📜 История рефералов:\n"
    for record in history:
        message += (
            f"Приглашенный ID: {record[2]}\n"
            f"Сумма: {record[3] / 100:.2f} руб\n"
            f"Дата: {record[4]}\n\n"
        )

    reply_markup = InlineKeyboardMarkup([
        [InlineKeyboardButton("Назад к профилю", callback_data="back_to_profile")]
    ])

    await update.callback_query.message.reply_text(message, reply_markup=reply_markup)

async def show_referral_balance_history(update: Update, context: CallbackContext) -> None:
    query = update.callback_query
    await query.answer()
    
    user_id = query.from_user.id
    user_data = get_user_data(user_id)
    
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('SELECT amount, date FROM referral_history WHERE referrer_id = ?', (user_data[0],))
    referral_history = cursor.fetchall()
    conn.close()
    
    if not referral_history:
        message = 'История зачислений пуста.'
    else:
        message = '💸 Зачисления на баланс:\n'
        for history in referral_history:
            message += f"• {history[0]} руб. - {history[1]}\n"
    
    referral_menu = InlineKeyboardMarkup([
        [InlineKeyboardButton('Назад к реферальной системе', callback_data='referral')]
    ])
    await query.edit_message_text(text=message, reply_markup=referral_menu)

async def handle_balance_payment(update: Update, context: CallbackContext) -> None:
    query = update.callback_query
    await query.answer()
    data = query.data.split('_')

    product_id = int(data[2])
    quantity = int(data[3])
    total_price_rub = float(data[4])
    user_data = get_user_data(query.from_user.id)

    # Проверяем, использовал ли пользователь промокод
    if user_data[6]:  # user_data[6] хранит промокод
        # Если использовал, проверяем, активен ли он еще
        if is_promocode_active(user_data[6]):
            # Проверяем количество активаций для промокода
            if get_promocode_activations(user_data[6]) > 0:
                # Вычитаем скидку, если промокод активен
                total_price_rub *= (1 - user_data[7] / 100)  # user_data[7] хранит скидку
            else:
                await query.message.reply_text("Срок действия вашего промокода истек.")
                return
        else:
            await query.message.reply_text("Промокод недействителен.")
            return

    if user_data[3] >= total_price_rub:
        # Обрабатываем успешную покупку
        payload = f"{query.from_user.id}_{product_id}_{quantity}"
        filepath, purchase_id = await handle_successful_payment(payload)

        if filepath:
            # Списываем деньги с баланса только если обработка прошла успешно
            conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
            cursor = conn.cursor()
            cursor.execute('UPDATE users SET balance = balance - ? WHERE telegram_id = ?', (total_price_rub, query.from_user.id))
            conn.commit()
            conn.close()

            await context.bot.send_document(chat_id=query.from_user.id, document=open(filepath, 'rb'))
            await query.message.reply_text(f"Спасибо за покупку!\nID покупки: {purchase_id}")

            # Уменьшаем количество активаций промокода после успешного использования
            if user_data[6]:
                decrease_promocode_activations(user_data[6])
        else:
            await query.message.reply_text("Ошибка при обработке заказа.")
    else:
        await query.message.reply_text("Недостаточно средств на балансе.")

async def handle_crypto_payment(update: Update, context: CallbackContext) -> None:
    query = update.callback_query
    await query.answer()
    data = query.data.split('_')

    product_id = int(data[2])
    quantity = int(data[3])
    total_price_rub = float(data[4])
    user_data = get_user_data(query.from_user.id)

    # Проверяем, использовал ли пользователь промокод
    if user_data[6]:  # user_data[6] хранит промокод
        # Если использовал, проверяем, активен ли он еще
        if is_promocode_active(user_data[6]):
            # Проверяем количество активаций для промокода
            if get_promocode_activations(user_data[6]) > 0:
                # Вычитаем скидку, если промокод активен
                total_price_rub *= (1 - user_data[7] / 100)  # user_data[7] хранит скидку
            else:
                await query.message.reply_text("Срок действия вашего промокода истек.")
                return
        else:
            await query.message.reply_text("Промокод недействителен.")
            return

    # Здесь должен быть код для оплаты через криптобота
    # ...

    # После успешной оплаты:
    payload = f"{query.from_user.id}_{product_id}_{quantity}"
    filepath, purchase_id = await handle_successful_payment(payload)

    if filepath:
        await context.bot.send_document(chat_id=query.from_user.id, document=open(filepath, 'rb'))
        await query.message.reply_text(f"Спасибо за покупку!\nID покупки: {purchase_id}")

        # Уменьшаем количество активаций промокода после успешного использования
        if user_data[6]:
            decrease_promocode_activations(user_data[6])
    else:
        await query.message.reply_text("Ошибка при обработке заказа.")

def get_promocode_activations(promo_code):
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('SELECT activations FROM promo_codes WHERE code = ?', (promo_code,))
    activations = cursor.fetchone()
    conn.close()
    return activations[0] if activations else 0

async def check_admins(update: Update, context: CallbackContext) -> None:
    user_id = update.message.from_user.id
    if not is_admin(user_id):
        await update.message.reply_text('У вас нет прав для выполнения этой команды.')
        return

    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('SELECT username, telegram_id FROM users WHERE is_admin = 1')
    admins = cursor.fetchall()
    conn.close()

    if not admins:
        await update.message.reply_text('Администраторы не найдены.')
        return

    message = 'Список администраторов:\n'
    for admin in admins:
        message += f'Имя: {admin[0]}, Telegram ID: {admin[1]}\n'
    
    await update.message.reply_text(message)

def decrease_promocode_activations(promo_code):
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('UPDATE promo_codes SET activations = activations - 1 WHERE code = ?', (promo_code,))
    conn.commit()
    conn.close()

def is_promocode_active(promo_code):
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('SELECT * FROM promo_codes WHERE code = ?', (promo_code,))
    promo = cursor.fetchone()
    conn.close()
    return promo is not None

async def add_balance(update: Update, context: CallbackContext) -> None:
    user_id = update.message.from_user.id
    if not is_admin(user_id):
        await update.message.reply_text('У вас нет прав для выполнения этой команды.')
        return
    try:
        target_id = int(context.args[0])
        amount = int(context.args[1])
    except (IndexError, ValueError):
        await update.message.reply_text('Использование: /add_balance <id> <сумма>')
        return

    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('UPDATE users SET balance = balance + ? WHERE id = ?', (amount, target_id))
    conn.commit()
    conn.close()
    await update.message.reply_text(f'Баланс пользователя с ID {target_id} пополнен на {amount} ₽.')

async def send_to_all(update: Update, context: CallbackContext) -> None:
    """
    Отправляет фото или текст всем пользователям из базы данных.

    Для отправки фото: отправьте команду /send_to_all с фотографией и, опционально, caption.
    Для отправки текста: отправьте команду /send_to_all с текстом (через аргументы команды или caption, если фото нет).
    """
    user_id = update.message.from_user.id
    logger.info(f"User ID: {user_id}")

    if not is_admin(user_id):
        await update.message.reply_text('У вас нет прав для выполнения этой команды.')
        return

    try:
        conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
        cursor = conn.cursor()
        cursor.execute('SELECT telegram_id FROM users')
        users = cursor.fetchall()
        logger.info(f"Users: {users}")
        conn.close()
    except sqlite3.Error as e:
        logger.error(f"Ошибка подключения к базе данных: {e}")
        await update.message.reply_text('Ошибка подключения к базе данных.')
        return

    if not users:
        logger.info("Нет пользователей в базе данных")
        await update.message.reply_text('Нет пользователей в базе данных.')
        return

    message_text = ' '.join(context.args) if context.args else update.message.caption  # Текст сообщения (если есть)

    if update.message.photo:
        photo = update.message.photo[-1]  # Получаем последнее фото

        for user in users:
            try:
                await context.bot.send_photo(
                    chat_id=user[0],
                    photo=photo.file_id,
                    caption=message_text,
                )
                logger.info(f"Отправлено фото {photo.file_id} to {user[0]}")
            except Exception as e:
                logger.error(f"Ошибка отправки фото to {user[0]}: {e}")
                await update.message.reply_text(f"Ошибка отправки сообщения пользователю {user[0]}.")

    elif update.message.text or message_text: # Отправляем текст, если есть или текст в caption
        for user in users:
            try:
                await context.bot.send_message(chat_id=user[0], text=message_text)
                logger.info(f"Отправлен текст to {user[0]}")
            except Exception as e:
                logger.error(f"Ошибка отправки текста to {user[0]}: {e}")
                await update.message.reply_text(f"Ошибка отправки сообщения пользователю {user[0]}.")
    else:
        await update.message.reply_text('Для отправки используйте фото или текст.')

async def remove_balance(update: Update, context: CallbackContext) -> None:
    user_id = update.message.from_user.id
    if not is_admin(user_id):
        await update.message.reply_text('У вас нет прав для выполнения этой команды.')
        return
    try:
        target_id = int(context.args[0])
        amount = int(context.args[1])
    except (IndexError, ValueError):
        await update.message.reply_text('Использование: /remove_balance <id> <сумма>')
        return

    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('UPDATE users SET balance = balance - ? WHERE id = ?', (amount, target_id))
    conn.commit()
    conn.close()
    await update.message.reply_text(f'Баланс пользователя с ID {target_id} уменьшен на {amount} ₽.')

async def zaliv(update: Update, context: CallbackContext) -> None:
    user_id = update.message.from_user.id
    if not is_admin(user_id):
        await update.message.reply_text('У вас нет прав для выполнения этой команды.')
        return

    # Ожидаем файл от пользователя
    await update.message.reply_text('Пожалуйста, отправьте файл .txt с куками.')
    # Устанавливаем флаг, что ожидается файл для заливки
    context.user_data['awaiting_zaliv_file'] = True

async def handle_zaliv_file(update: Update, context: CallbackContext) -> None:
    document = update.message.document
    file = await document.get_file()
    file_path = await file.download_to_drive()
    await update.message.reply_text(f'Файл {document.file_name} успешно загружен и сохранен.')

    # Ожидаем детали заливки
    await update.message.reply_text('Пожалуйста, введите название, цену и количество товаров (например: "название 100 10" или "название 100 check").')
    context.user_data['zaliv_file_path'] = file_path
    context.user_data['awaiting_zaliv_details'] = True
    context.user_data['awaiting_zaliv_file'] = False

async def handle_zaliv_details(update: Update, context: CallbackContext) -> None:
    details = update.message.text.split()
    if len(details) < 3:
        await update.message.reply_text("Пожалуйста, введите корректные данные в формате: название цена количество.")
        return

    name = " ".join(details[:-2])
    price = details[-2]
    quantity = details[-1]

    try:
        price = int(float(price) * 100)  # Преобразование цены в копейки
        if quantity.lower() == 'check':
            file_path = context.user_data.get('zaliv_file_path')
            if file_path and os.path.exists(file_path):
                quantity = count_items_in_file(file_path)
            else:
                await update.message.reply_text("Файл не найден или не указан.")
                return
        else:
            quantity = int(quantity)
    except ValueError:
        await update.message.reply_text("Пожалуйста, введите корректные данные.")
        return

    # Переименование файла на название товара
    file_path = context.user_data.get('zaliv_file_path')
    if file_path and os.path.exists(file_path):
        new_file_path = os.path.join(os.path.dirname(file_path), f"{name}.txt")
        os.rename(file_path, new_file_path)
        context.user_data['zaliv_file_path'] = new_file_path

    # Добавление товара в базу данных (пример)
    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('INSERT INTO products (name, price, quantity) VALUES (?, ?, ?)', (name, price, quantity))
    conn.commit()
    conn.close()

    await update.message.reply_text(f"Товар {name} с ценой {price / 100:.2f} руб и количеством {quantity} успешно добавлен.")
    context.user_data['awaiting_zaliv_details'] = False
    context.user_data.pop('zaliv_file_path', None)

async def top_up_balance(update: Update, context: CallbackContext) -> None:
    query = update.callback_query
    await query.answer()
    user_id = query.from_user.id
    await query.message.reply_text("Введите сумму для пополнения в рублях:")
    context.user_data['awaiting_balance_amount'] = True
    context.user_data['user_id_for_balance'] = user_id

async def handle_amount_input(update: Update, context: CallbackContext) -> None:
    if context.user_data.get('awaiting_balance_amount'):
        try:
            amount_rub = int(update.message.text)
        except ValueError:
            await update.message.reply_text("Пожалуйста, введите корректную сумму.")
            return

        amount_usd = convert_rub_to_usd(amount_rub)
        user_id = context.user_data['user_id_for_balance']

        payment_link, invoice_id = await create_payment_link("Balance Top-Up", amount_usd, user_id)
        await update.message.reply_text(
            f"Оплата через CryptoBot для пополнения баланса:\n{payment_link}",
            reply_markup=InlineKeyboardMarkup([
                [InlineKeyboardButton("Я оплатил", callback_data=f"check_balance_payment_{invoice_id}")]
            ])
        )
        context.user_data['pending_balance_invoice'] = invoice_id
        context.user_data['pending_balance_amount'] = amount_rub
        context.user_data['awaiting_balance_amount'] = False

async def handle_check_balance_payment(update: Update, context: CallbackContext) -> None:
    query = update.callback_query
    await query.answer()
    invoice_id = int(query.data.split('_')[3])
    user_id = query.from_user.id

    if not await check_payment_status(invoice_id):
        await query.message.reply_text("Оплата не подтверждена. Попробуйте еще раз позже.")
        return

    if 'processed_balance_invoices' not in context.user_data:
        context.user_data['processed_balance_invoices'] = set()
    if invoice_id in context.user_data['processed_balance_invoices']:
        await query.message.reply_text("Этот счет уже был обработан.")
        return

    context.user_data['processed_balance_invoices'].add(invoice_id)
    amount_rub = context.user_data.pop('pending_balance_amount', 0)

    conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
    cursor = conn.cursor()
    cursor.execute('UPDATE users SET balance = balance + ? WHERE telegram_id = ?', (amount_rub, user_id))
    conn.commit()
    conn.close()

    await query.message.reply_text(f"Баланс успешно пополнен на {amount_rub} руб.")

async def handle_message(update: Update, context: CallbackContext) -> None:
    if context.user_data.get('purchase_completed'):
        await update.message.reply_text("Покупка уже завершена. Пожалуйста, начните новый процесс покупки.")
        return
    
    text = update.message.text.lower()
    
    # Обработка ввода количества для покупки
    if context.user_data.get('awaiting_purchase_quantity'):
        await handle_purchase_quantity(update, context)
        return

    # Обработка ввода суммы для пополнения баланса
    if context.user_data.get('awaiting_balance_amount'):
        await handle_amount_input(update, context)
        return

    # Обработка ввода деталей для заливки
    if context.user_data.get('awaiting_zaliv_details'):
        await handle_zaliv_details(update, context)
        return

    # Обработка ввода промокода
    if context.user_data.get('awaiting_promo_code'):
        await handle_promo_code(update, context)
        return

    # Обработка стандартных команд через текст
    if text == "профиль":
        await profile(update, context)
    elif text == "все категории":
        await show_categories(update, context)
    elif text == "наличие товара":
        await show_stock(update, context)
    elif text == "правила":
        await show_rules(update, context)
    elif text == "правила замен":
        await show_replacement_request(update, context)
    else:
        await update.message.reply_text("Пожалуйста, выберите один из пунктов меню.")

async def activate_promo(update: Update, context: CallbackContext) -> None:
    query = update.callback_query
    await query.answer()
    user_id = query.from_user.id
    await query.message.reply_text("Введите промокод для активации:")
    context.user_data['awaiting_promo_code'] = True
    context.user_data['user_id_for_promo'] = user_id

async def handle_promo_code(update: Update, context: CallbackContext) -> None:
    if context.user_data.get('awaiting_promo_code'):
        promo_code = update.message.text
        user_id = context.user_data['user_id_for_promo']

        conn = sqlite3.connect('C:/Users/user/Desktop/telegramshop/baza.db')
        cursor = conn.cursor()
        cursor.execute('SELECT * FROM promo_codes WHERE code = ?', (promo_code,))
        promo = cursor.fetchone()

        # Проверяем, использовался ли промокод этим пользователем
        if promo:
            cursor.execute('SELECT * FROM used_promocodes WHERE user_id = ? AND promo_code = ?', (user_id, promo_code))
            used = cursor.fetchone()
            if used:
                await update.message.reply_text("Вы уже использовали этот промокод.")
            elif promo[3] > 0:  # Проверка наличия активаций
                # Применяем скидку
                cursor.execute('UPDATE users SET promo = ?, promo_discount = ? WHERE telegram_id = ?', (promo[1], promo[2], user_id))
                cursor.execute('UPDATE promo_codes SET activations = activations - 1 WHERE code = ?', (promo_code,))
                cursor.execute('INSERT INTO used_promocodes (user_id, promo_code) VALUES (?, ?)', (user_id, promo_code))
                conn.commit()
                await update.message.reply_text(f"Промокод {promo_code} активирован. Скидка: {promo[2]}%")
            else:
                await update.message.reply_text("Промокод недействителен или закончились активации.")
        else:
            await update.message.reply_text("Промокод недействителен.")

        conn.close()
        context.user_data['awaiting_promo_code'] = False

def main() -> None:
    create_database()
    update_database()
    application = Application.builder().token(TOKEN).build()
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("add_product", add_product))
    application.add_handler(CommandHandler("delete_product", delete_product))
    application.add_handler(CommandHandler("list_products", list_products))
    application.add_handler(CommandHandler("make_admin", make_admin))
    application.add_handler(CommandHandler("remove_admin", remove_admin))
    application.add_handler(CommandHandler("all", send_to_all))
    application.add_handler(CommandHandler("add_balance", add_balance))
    application.add_handler(CommandHandler("remove_balance", remove_balance))
    application.add_handler(CommandHandler("check_admins", check_admins))
    application.add_handler(CommandHandler("zaliv", zaliv))
    application.add_handler(CommandHandler("reboot", reboot))
    application.add_handler(CommandHandler("rebootprice", reboot_price))
    application.add_handler(CommandHandler("add_promo", add_promo))
    application.add_handler(CommandHandler("delete_promo", delete_promo))
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    application.add_handler(MessageHandler(filters.Document.MimeType("text/plain"), handle_zaliv_file))
    application.add_handler(CallbackQueryHandler(select_product, pattern=r'^select_product_'))
    application.add_handler(CallbackQueryHandler(show_history, pattern="^history$"))
    application.add_handler(CallbackQueryHandler(show_balance, pattern="^balance$"))
    application.add_handler(CallbackQueryHandler(back_to_profile, pattern="^back_to_profile$"))
    application.add_handler(CallbackQueryHandler(handle_check_payment, pattern="^check_payment_"))
    application.add_handler(CallbackQueryHandler(handle_balance_payment, pattern=r'^pay_balance_'))
    application.add_handler(CallbackQueryHandler(handle_cryptobot_payment, pattern=r'^pay_cryptobot_'))
    application.add_handler(CallbackQueryHandler(history_prev, pattern="^history_prev$"))
    application.add_handler(CallbackQueryHandler(history_next, pattern="^history_next$"))
    application.add_handler(CallbackQueryHandler(show_referral, pattern="^referral$"))
    application.add_handler(CallbackQueryHandler(show_top_referrals, pattern="^top_referrals$"))
    application.add_handler(CallbackQueryHandler(show_referral_history, pattern="^referral_history$"))
    application.add_handler(CallbackQueryHandler(show_referral_balance_history, pattern="^referral_balance_history$"))
    application.add_handler(CallbackQueryHandler(top_up_balance, pattern="^top_up_balance$"))
    application.add_handler(CallbackQueryHandler(handle_check_balance_payment, pattern="^check_balance_payment_"))
    application.add_handler(CallbackQueryHandler(admin_panel, pattern="^admin_panel$"))
    application.add_handler(CallbackQueryHandler(activate_promo, pattern="^activate_promo$"))
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_promo_code))

    logger.info("Starting bot")
    application.run_polling()

if __name__ == '__main__':
    main()
