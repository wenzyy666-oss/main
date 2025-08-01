# main
Вльв
import logging
from aiogram import Bot, Dispatcher, types, F
from aiogram.filters import Command
from aiogram.types import (
    Message,
    CallbackQuery,
    ReplyKeyboardMarkup,
    KeyboardButton,
    InlineKeyboardMarkup,
    InlineKeyboardButton
)
from aiogram.utils.keyboard import ReplyKeyboardBuilder, InlineKeyboardBuilder
from aiogram.exceptions import TelegramBadRequest

# Настройки
logging.basicConfig(level=logging.INFO)
BOT_TOKEN = "8407559236:AAFqzDqei9ptY06AtpuRkcgE5blJbk0AFhs"  # Замените на реальный токен
ADMIN_ID = 7275349993     # Ваш ID
ADMIN_USERNAME = "@Wenzyy69"
GOLD_RATE = 0.69
MIN_PAYMENT = 50
FIXED_FEE = 15
MARKET_COMMISSION = 0.3

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()

# Хранение данных
user_requests = {}
admin_pending_messages = {}
request_counter = 1
is_active = True
active_users = set()
user_states = {}  # Для отслеживания состояния пользователей

class UserState:
    CALCULATING = "calculating"
    BUYING = "buying"
    NONE = "none"
    WAITING_SKIN = "waiting_skin"  # Новое состояние для ожидания скрина скина

# Главное меню
def main_menu():
    builder = ReplyKeyboardBuilder()
    builder.row(
        KeyboardButton(text="🌟 Купить голду"),
        KeyboardButton(text="🔢 Калькулятор")
    )
    builder.row(
        KeyboardButton(text="📖 Помощь")
    )
    return builder.as_markup(resize_keyboard=True)

# Кнопки для админа
def admin_buttons(request_id):
    builder = InlineKeyboardBuilder()
    builder.add(
        InlineKeyboardButton(text="✅ Принять", callback_data=f"accept_{request_id}"),
        InlineKeyboardButton(text="❌ Отклонить", callback_data=f"reject_{request_id}")
    )
    return builder.as_markup()

# Кнопка для реквизитов
def details_button(request_id):
    builder = InlineKeyboardBuilder()
    builder.add(InlineKeyboardButton(
        text="📝 Отправить реквизиты", 
        callback_data=f"details_{request_id}")
    )
    return builder.as_markup()

# Сброс состояния пользователя
def reset_user_state(user_id):
    user_states[user_id] = UserState.NONE
    if user_id in user_requests:
        for req_id, req in list(user_requests.items()):
            if req['user_id'] == user_id and req['status'] in ['waiting', 'waiting_payment']:
                user_requests.pop(req_id)

# Проверка активности
async def check_active(message: Message):
    global is_active, active_users
    
    if not is_active and message.from_user.id not in active_users:
        await message.answer("⛔ Бот находится на техническом перерыве. Попробуйте позже.")
        return False
    
    if message.from_user.id not in active_users:
        active_users.add(message.from_user.id)
    
    return True

# Команда /start
@dp.message(Command("start"))
async def cmd_start(message: Message):
    if not await check_active(message):
        return
    
    reset_user_state(message.from_user.id)
    await message.answer(
        "🔫 Добро пожаловать в @alyans_shop_bot!\n"
        f"💰 Курс: 1 голда = {GOLD_RATE}₽\n"
        f"⚠ Минимальная покупка: {MIN_PAYMENT}₽\n\n"
        "Выберите действие:",
        reply_markup=main_menu()
    )

# Обработка кнопки "Купить голду"
@dp.message(F.text == "🌟 Купить голду")
async def buy_gold(message: Message):
    if not await check_active(message):
        return
    
    reset_user_state(message.from_user.id)
    user_states[message.from_user.id] = UserState.BUYING
    await message.answer(
        f"💵 Введите сумму в рублях (от {MIN_PAYMENT}₽):",
        reply_markup=types.ReplyKeyboardRemove()
    )

# Обработка ввода суммы
@dp.message(F.text.replace('.', '').isdigit())
async def process_input(message: Message):
    global request_counter
    
    if not await check_active(message):
        return
    
    user_id = message.from_user.id
    current_state = user_states.get(user_id, UserState.NONE)
    
    try:
        amount = float(message.text)
        
        if current_state == UserState.BUYING:
            # Обработка покупки
            if amount < MIN_PAYMENT:
                await message.answer(f"Минимальная сумма {MIN_PAYMENT}₽", reply_markup=main_menu())
                return
            
            gold = amount / GOLD_RATE
            total = amount + FIXED_FEE
            request_id = request_counter
            request_counter += 1
            
            user_requests[request_id] = {
                'user_id': user_id,
                'username': message.from_user.username,
                'amount': amount,
                'gold': gold,
                'total': total,
                'status': 'waiting',
                'message_id': message.message_id
            }
            
            await bot.send_message(
                ADMIN_ID,
                f"🛒 Новая заявка #{request_id} от @{message.from_user.username}:\n"
                f"• Сумма: {amount}₽\n"
                f"• Голда: {gold:.2f}\n"
                f"• К оплате: {total}₽",
                reply_markup=details_button(request_id)
            )
            
            await message.answer(
                f"✅ Заявка #{request_id} принята!\nОжидайте реквизиты",
                reply_markup=main_menu()
            )
            user_states[user_id] = UserState.NONE
            
        elif current_state == UserState.CALCULATING:
            # Обработка калькулятора
            if amount < MIN_PAYMENT:
                await message.answer(f"Минимальная сумма {MIN_PAYMENT}₽", reply_markup=main_menu())
                return
            
            gold = amount / GOLD_RATE
            total = amount + FIXED_FEE
            await message.answer(
                f"📊 Результат:\n"
                f"• Сумма: {amount}₽\n"
                f"• Голда: {gold:.2f}\n"
                f"• К оплате: {total}₽ (включая комиссию {FIXED_FEE}₽)",
                reply_markup=main_menu()
            )
            user_states[user_id] = UserState.NONE
            
        else:
            await message.answer("Пожалуйста, выберите действие из меню", reply_markup=main_menu())
            
    except Exception as e:
        logging.error(f"Ошибка в process_input: {e}")
        await message.answer("Произошла ошибка, попробуйте позже", reply_markup=main_menu())

# Калькулятор
@dp.message(F.text == "🔢 Калькулятор")
async def calculator(message: Message):
    if not await check_active(message):
        return
    
    reset_user_state(message.from_user.id)
    user_states[message.from_user.id] = UserState.CALCULATING
    await message.answer(
        "🧮 Введите сумму в рублях для расчета:\n"
        "Пример: 100",
        reply_markup=types.ReplyKeyboardRemove()
    )

# Обработка кнопки "Отправить реквизиты"
@dp.callback_query(F.data.startswith("details_"))
async def send_details(callback: CallbackQuery):
    try:
        request_id = int(callback.data.split('_')[1])
        admin_pending_messages[callback.from_user.id] = request_id
        await callback.message.answer("Введите реквизиты для оплаты:")
        await callback.answer()
    except Exception as e:
        logging.error(f"Ошибка в send_details: {e}")
        await callback.answer("Произошла ошибка")

# Обработка реквизитов от админа
@dp.message(F.from_user.id == ADMIN_ID)
async def handle_admin_commands(message: Message):
    global is_active
    
    # Обработка команд активности
    if message.text == "-актив":
        is_active = False
        active_users.clear()
        await message.answer("⛔ Технический перерыв активирован. Бот отправил уведомления всем пользователям.")
        return
    elif message.text == "+актив":
        is_active = True
        await message.answer("✅ Бот снова активен для всех пользователей.")
        return
        
    # Обычная обработка реквизитов
    try:
        if message.from_user.id in admin_pending_messages:
            request_id = admin_pending_messages[message.from_user.id]
            request = user_requests.get(request_id)
            
            if request and request['status'] == 'waiting':
                await bot.send_message(
                    request['user_id'],
                    f"💳 Вы получили реквизиты:\n\n{message.text}\n\n"
                    "После оплаты отправьте скриншот перевода"
                )
                request['status'] = 'waiting_payment'
                del admin_pending_messages[message.from_user.id]
                await message.answer("✅ Реквизиты отправлены пользователю")
    except Exception as e:
        logging.error(f"Ошибка в handle_admin_commands: {e}")

# Обработка скрина оплаты
@dp.message(F.photo)
async def handle_payment_screenshot(message: Message):
    try:
        # Ищем заявку в статусе ожидания оплаты
        request = next(
            (r for r in user_requests.values() 
             if r['user_id'] == message.from_user.id and r['status'] == 'waiting_payment'),
            None
        )
        
        if request:
            request_id = list(user_requests.keys())[list(user_requests.values()).index(request)]
            await bot.send_photo(
                ADMIN_ID,
                message.photo[-1].file_id,
                caption=f"📸 Скрин оплаты по заявке #{request_id}\n"
                       f"От @{message.from_user.username}",
                reply_markup=admin_buttons(request_id)
            )
            await message.answer("Скрин оплаты отправлен на проверку. Ожидайте подтверждения.")
            return
        
        # Если это не скрин оплаты, проверяем, не скрин ли это скина
        await handle_skin_screenshot(message)
    except Exception as e:
        logging.error(f"Ошибка в handle_payment_screenshot: {e}")

# Обработка скрина скина (отдельная функция)
async def handle_skin_screenshot(message: Message):
    try:
        # Ищем заявку в статусе ожидания скина
        request = next(
            (r for r in user_requests.values() 
             if r['user_id'] == message.from_user.id and r['status'] == 'waiting_skin'),
            None
        )
        
        if request:
            request_id = list(user_requests.keys())[list(user_requests.values()).index(request)]
            gold_with_commission = request['gold'] * (1 + MARKET_COMMISSION)
            
            await bot.send_photo(
                ADMIN_ID,
                message.photo[-1].file_id,
                caption=f"🖼 Скрин скина по заявке #{request_id}\n"
                       f"От @{message.from_user.username}\n"
                       f"Должен быть выставлен за: {gold_with_commission:.2f} голды\n"
                       f"Проверьте паттерн и цену"
            )
            
            await message.answer("✅ Скрин скина успешно отправлен админу!")
            user_requests.pop(request_id)
    except Exception as e:
        logging.error(f"Ошибка в handle_skin_screenshot: {e}")

# Обработка решений админа
@dp.callback_query(F.data.startswith(("accept_", "reject_")))
async def handle_decision(callback: CallbackQuery):
    try:
        action, request_id = callback.data.split('_')
        request_id = int(request_id)
        request = user_requests.get(request_id)
        
        if not request:
            await callback.answer("Заявка не найдена")
            return
        
        if action == "accept":
            if request['status'] == 'waiting_payment':
                gold_with_commission = request['gold'] * (1 + MARKET_COMMISSION)
                await bot.send_message(
                    request['user_id'],
                    f"✅ Ваша заявка подтверждена!\n\n"
                    f"1. Выставьте Беретты Дамаскус за:\n"
                    f"💰 {gold_with_commission:.2f} голды (сумма +30% комиссии рынка)\n\n"
                    f"2. Отправьте скриншот скина с видимым:\n"
                    "- Паттерном\n"
                    "- Ценой\n"
                    "- Надписью 'Только мои запросы'"
                )
                request['status'] = 'waiting_skin'
                user_states[request['user_id']] = UserState.WAITING_SKIN
                try:
                    await callback.message.edit_text(
                        f"✅ Заявка #{request_id} подтверждена\n"
                        f"Ожидаем скрин скина от @{request['username']}"
                    )
                except TelegramBadRequest:
                    await callback.message.answer(
                        f"✅ Заявка #{request_id} подтверждена\n"
                        f"Ожидаем скрин скина от @{request['username']}"
                    )
            else:
                await callback.answer("Заявка уже обработана")
        else:
            await bot.send_message(
                request['user_id'],
                "❌ Ваша заявка отклонена\n"
                "Причина: оплата не подтверждена"
            )
            try:
                await callback.message.edit_text(f"❌ Заявка #{request_id} отклонена")
            except TelegramBadRequest:
                await callback.message.answer(f"❌ Заявка #{request_id} отклонена")
            user_requests.pop(request_id)
        
        await callback.answer()
    except Exception as e:
        logging.error(f"Ошибка в handle_decision: {e}")
        await callback.answer("Произошла ошибка")

# Помощь
@dp.message(F.text == "📖 Помощь")
async def help_command(message: Message):
    if not await check_active(message):
        return
    
    reset_user_state(message.from_user.id)
    await message.answer(
        f"По всем вопросам обращайтесь к {ADMIN_USERNAME}",
        reply_markup=main_menu()
    )

# Запуск бота
async def main():
    await dp.start_polling(bot)

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
