import asyncio
import logging
import aiosqlite
from aiogram import Bot, Dispatcher, F, Router
from aiogram.filters import CommandStart, Command
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.types import (
    Message, CallbackQuery, InlineKeyboardMarkup, InlineKeyboardButton
)

# --- НАСТРОЙКИ ---
BOT_TOKEN = "9025563353:AAFpgVvaqnecAFUE44S2cjpxgT-v8hM85Ok"
ADMIN_ID = 8789111923  # Твой Telegram ID (число)
SUPPORT_USERNAME = "@DanyaMzx"

PRICE_PER_REF_STARS = 25
# Курс звезд к валютам (примерный для расчета):
STAR_TO_UAH = 0.8
STAR_TO_RUB = 1.8
STAR_TO_USD = 0.02

# --- ИНИЦИАЛИЗАЦИЯ БД ---
async def init_db():
    async with aiosqlite.connect("shop.db") as db:
        # Таблица пользователей
        await db.execute("""
            CREATE TABLE IF NOT EXISTS users (
                user_id INTEGER PRIMARY KEY,
                balance_uah REAL DEFAULT 0.0,
                balance_rub REAL DEFAULT 0.0,
                balance_usd REAL DEFAULT 0.0,
                currency TEXT DEFAULT 'UAH',
                joined_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        
        # Таблица заказов рефералов
        await db.execute("""
            CREATE TABLE IF NOT EXISTS orders (
                order_id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER,
                target_bot TEXT,
                quantity INTEGER,
                total_stars INTEGER,
                status TEXT DEFAULT 'pending',
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        
        # Таблица отзывов
        await db.execute("""
            CREATE TABLE IF NOT EXISTS reviews (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER,
                text TEXT,
                status TEXT DEFAULT 'pending',
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        await db.commit()

# --- FSM (СОСТОЯНИЯ) ---
class OrderState(StatesGroup):
    waiting_for_bot_name = State()
    waiting_for_quantity = State()

class ReviewState(StatesGroup):
    waiting_for_text = State()

class AdminReplyState(StatesGroup):
    waiting_for_reply = State()
    target_user_id = None

# --- КЛАВИАТУРЫ ---
def main_keyboard():
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="👤 Профиль", callback_data="profile")],
        [InlineKeyboardButton(text="💳 Пополнить баланс", callback_data="deposit")],
        [InlineKeyboardButton(text="🤝 Купить рефералов", callback_data="buy_refs")],
        [InlineKeyboardButton(text="⭐ Отзывы", callback_data="reviews")],
        [InlineKeyboardButton(text="💬 Поддержка", url=f"https://t.me/{SUPPORT_USERNAME.replace('@', '')}")]
    ])

def currency_keyboard():
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🇺🇦 UAH", callback_data="set_curr_UAH"),
         InlineKeyboardButton(text="🇷🇺 RUB", callback_data="set_curr_RUB"),
         InlineKeyboardButton(text="🇺🇸 USD", callback_data="set_curr_USD")],
        [InlineKeyboardButton(text="⬅️ Назад", callback_data="back_to_main")]
    ])

# --- ХЕНДЛЕРЫ ---
router = Router()

@router.message(CommandStart())
async def cmd_start(message: Message):
    async with aiosqlite.connect("shop.db") as db:
        await db.execute(
            "INSERT OR IGNORE INTO users (user_id) VALUES (?)", (message.from_user.id,)
        )
        await db.commit()
    await message.answer("👋 **Добро пожаловать в сервис по накрутке рефералов!**\nВыберите нужный раздел из меню ниже:", 
                         reply_markup=main_keyboard(), parse_mode="Markdown")

# --- РАЗДЕЛ: ПРОФИЛЬ ---
@router.callback_query(F.data == "profile")
async def show_profile(call: CallbackQuery):
    async with aiosqlite.connect("shop.db") as db:
        async with db.execute("SELECT balance_uah, balance_rub, balance_usd, currency FROM users WHERE user_id = ?", (call.from_user.id,)) as cursor:
            row = await cursor.fetchone()
            b_uah, b_rub, b_usd, curr = row if row else (0.0, 0.0, 0.0, 'UAH')

    text = (
        f"👤 **Ваш Профиль**\n\n"
        f"🆔 ID: `{call.from_user.id}`\n"
        f"🌐 Выбранная валюта: **{curr}**\n\n"
        f"💰 **Балансы:**\n"
        f"├ 🇺🇦 UAH: `{b_uah:.2f} ₴`\n"
        f"├ 🇷🇺 RUB: `{b_rub:.2f} ₽`\n"
        f"└ 🇺🇸 USD: `{b_usd:.2f} $`"
    )
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="💱 Сменить основную валюту", callback_data="change_curr")],
        [InlineKeyboardButton(text="⬅️ В главное меню", callback_data="back_to_main")]
    ])
    await call.message.edit_text(text, reply_markup=kb, parse_mode="Markdown")

@router.callback_query(F.data == "change_curr")
async def change_curr(call: CallbackQuery):
    await call.message.edit_text("Выберите валюту по умолчанию:", reply_markup=currency_keyboard())

@router.callback_query(F.data.startswith("set_curr_"))
async def set_currency(call: CallbackQuery):
    new_curr = call.data.split("_")[2]
    async with aiosqlite.connect("shop.db") as db:
        await db.execute("UPDATE users SET currency = ? WHERE user_id = ?", (new_curr, call.from_user.id))
        await db.commit()
    await call.answer(f"Основная валюта изменена на {new_curr}")
    await show_profile(call)

# --- РАЗДЕЛ: ПОПОЛНЕНИЕ ---
@router.callback_query(F.data == "deposit")
async def deposit_menu(call: CallbackQuery):
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="💳 Банковская карта", callback_data="pay_card")],
        [InlineKeyboardButton(text="💎 Крипта / CryptoBot", callback_data="pay_crypto")],
        [InlineKeyboardButton(text="⭐ Telegram Stars", callback_data="pay_stars")],
        [InlineKeyboardButton(text="⬅️ Назад", callback_data="back_to_main")]
    ])
    await call.message.edit_text("💳 **Выберите способ пополнения баланса:**", reply_markup=kb, parse_mode="Markdown")

@router.callback_query(F.data.in_(["pay_card", "pay_crypto", "pay_stars"]))
async def process_pay_method(call: CallbackQuery):
    await call.message.edit_text(
        f"ℹ️ Для пополнения счета этим способом, пожалуйста, напишите администратору: {SUPPORT_USERNAME}",
        reply_markup=InlineKeyboardMarkup(inline_keyboard=[[InlineKeyboardButton(text="⬅️ Назад", callback_data="deposit")]])
    )

# --- РАЗДЕЛ: КУПИТЬ РЕФЕРАЛОВ ---
@router.callback_query(F.data == "buy_refs")
async def start_buy_refs(call: CallbackQuery, state: FSMContext):
    await state.set_state(OrderState.waiting_for_bot_name)
    await call.message.edit_text(
        "🤝 **Для какого бота вам нужны рефералы?**\n"
        "Отправьте имя бота (например, `@example_bot`) или вашу реферальную ссылку:",
        reply_markup=InlineKeyboardMarkup(inline_keyboard=[[InlineKeyboardButton(text="❌ Отмена", callback_data="back_to_main")]])
    )

@router.message(OrderState.waiting_for_bot_name)
async def process_bot_name(message: Message, state: FSMContext):
    await state.update_data(target_bot=message.text)
    await state.set_state(OrderState.waiting_for_quantity)
    
    cost_min = 5 * PRICE_PER_REF_STARS
    cost_max = 100 * PRICE_PER_REF_STARS
    
    await message.answer(
        f"✅ Бот принят: `{message.text}`\n\n"
        f"📊 **Укажите количество рефералов (от 5 до 100):**\n"
        f"💰 Цена за 1 реферала: **{PRICE_PER_REF_STARS} Stars** (~{PRICE_PER_REF_STARS * STAR_TO_UAH:.1f} UAH / ~{PRICE_PER_REF_STARS * STAR_TO_USD:.2f} $)\n"
        f"Мин. заказ: 5 реф ({cost_min} Stars) | Макс. заказ: 100 реф ({cost_max} Stars)",
        parse_mode="Markdown"
    )

@router.message(OrderState.waiting_for_quantity)
async def process_quantity(message: Message, state: FSMContext):
    if not message.text.isdigit():
        await message.answer("⚠️ Пожалуйста, введите число от 5 до 100:")
        return
    
    qty = int(message.text)
    if qty < 5 or qty > 100:
        await message.answer("⚠️ Количество должно быть строго от 5 до 100!")
        return

    data = await state.get_data()
    target_bot = data.get("target_bot")
    total_stars = qty * PRICE_PER_REF_STARS
    
    total_uah = total_stars * STAR_TO_UAH
    total_usd = total_stars * STAR_TO_USD
    
    await state.clear()
    
    # Сохраняем заказ в базу данных
    async with aiosqlite.connect("shop.db") as db:
        await db.execute(
            "INSERT INTO orders (user_id, target_bot, quantity, total_stars) VALUES (?, ?, ?, ?)",
            (message.from_user.id, target_bot, qty, total_stars)
        )
        await db.commit()
    
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="✅ Подтвердить и оплатить", callback_data=f"confirm_order_{qty}_{total_stars}")],
        [InlineKeyboardButton(text="❌ Отмена", callback_data="back_to_main")]
    ])
    
    await message.answer(
        f"🛒 **Подтверждение заказа:**\n\n"
        f"🤖 Целевой бот: `{target_bot}`\n"
        f"👥 Количество: **{qty} рефералов**\n"
        f"💳 К оплате: **{total_stars} ⭐ Stars** (или `{total_uah:.2f} UAH` / `{total_usd:.2f} $`)",
        reply_markup=kb, parse_mode="Markdown"
    )

@router.callback_query(F.data.startswith("confirm_order_"))
async def confirm_order(call: CallbackQuery):
    _, _, qty, total_stars = call.data.split("_")
    
    await call.message.edit_text(
        "✅ **Заказ успешно сформирован и сохранен в базе!**\n"
        "Администратор свяжется с вами для запуска работы.",
        reply_markup=main_keyboard(), parse_mode="Markdown"
    )
    
    # Уведомление в админку
    bot = call.bot
    await bot.send_message(
        ADMIN_ID,
        f"📥 **НОВЫЙ ЗАКАЗ!**\n"
        f"От пользователя: @{call.from_user.username} (ID: `{call.from_user.id}`)\n"
        f"Количество: {qty} рефералов\n"
        f"Сумма: {total_stars} Stars"
    )

# --- РАЗДЕЛ: ОТЗЫВЫ И МОДЕРАЦИЯ ---
@router.callback_query(F.data == "reviews")
async def reviews_menu(call: CallbackQuery):
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="✍️ Оставить отзыв", callback_data="add_review")],
        [InlineKeyboardButton(text="⬅️ Назад", callback_data="back_to_main")]
    ])
    await call.message.edit_text(
        "⭐ **Раздел отзывов**\nЗдесь вы можете оставить отзыв о работе нашего сервиса. После модерации он будет опубликован!",
        reply_markup=kb, parse_mode="Markdown"
    )

@router.callback_query(F.data == "add_review")
async def start_add_review(call: CallbackQuery, state: FSMContext):
    await state.set_state(ReviewState.waiting_for_text)
    await call.message.edit_text("Напишите ваш отзыв в сообщении ниже:")

@router.message(ReviewState.waiting_for_text)
async def process_review_text(message: Message, state: FSMContext):
    async with aiosqlite.connect("shop.db") as db:
        cursor = await db.execute(
            "INSERT INTO reviews (user_id, text) VALUES (?, ?)", (message.from_user.id, message.text)
        )
        review_id = cursor.lastrowid
        await db.commit()
        
    await state.clear()
    await message.answer("Спасибо! Ваш отзыв отправлен на модерацию.", reply_markup=main_keyboard())
    
    # Отправка отзывов модераторам
    kb_mod = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="✅ Одобрить", callback_data=f"approve_rev_{review_id}"),
         InlineKeyboardButton(text="❌ Отклонить", callback_data=f"reject_rev_{review_id}")],
        [InlineKeyboardButton(text="💬 Ответить клиенту", callback_data=f"reply_user_{message.from_user.id}")]
    ])
    
    bot = message.bot
    await bot.send_message(
        ADMIN_ID,
        f"📝 **Новый отзыв №{review_id} на модерации!**\n"
        f"От: @{message.from_user.username} (ID: `{message.from_user.id}`)\n\n"
        f"Текст:\n{message.text}",
        reply_markup=kb_mod, parse_mode="Markdown"
    )

# --- ДЕЙСТВИЯ МОДЕРАТОРА ---
@router.callback_query(F.data.startswith("approve_rev_"))
async def approve_review(call: CallbackQuery):
    rev_id = call.data.split("_")[2]
    async with aiosqlite.connect("shop.db") as db:
        await db.execute("UPDATE reviews SET status = 'approved' WHERE id = ?", (rev_id,))
        await db.commit()
    await call.message.edit_text(call.message.text + "\n\n✅ **ОДОБРЕНО**")

@router.callback_query(F.data.startswith("reject_rev_"))
async def reject_review(call: CallbackQuery):
    rev_id = call.data.split("_")[2]
    async with aiosqlite.connect("shop.db") as db:
        await db.execute("UPDATE reviews SET status = 'rejected' WHERE id = ?", (rev_id,))
        await db.commit()
    await call.message.edit_text(call.message.text + "\n\n❌ **ОТКЛОНЕНО**")

@router.callback_query(F.data.startswith("reply_user_"))
async def start_reply_user(call: CallbackQuery, state: FSMContext):
    target_id = int(call.data.split("_")[2])
    AdminReplyState.target_user_id = target_id
    await state.set_state(AdminReplyState.waiting_for_reply)
    await call.message.answer(f"Введите ответное сообщение для пользователя `ID: {target_id}`:")

@router.message(AdminReplyState.waiting_for_reply)
async def send_reply_to_user(message: Message, state: FSMContext):
    target_id = AdminReplyState.target_user_id
    bot = message.bot
    try:
        await bot.send_message(target_id, f"💬 **Сообщение от поддержки/модерации:**\n\n{message.text}", parse_mode="Markdown")
        await message.answer("✅ Сообщение успешно отправлено клиенту!")
    except Exception as e:
        await message.answer(f"❌ Ошибка отправки: {e}")
    await state.clear()

# --- ВОЗВРАТ В МЕНЮ ---
@router.callback_query(F.data == "back_to_main")
async def back_to_main(call: CallbackQuery, state: FSMContext):
    await state.clear()
    await call.message.edit_text("👋 **Главное меню**", reply_markup=main_keyboard(), parse_mode="Markdown")

# --- ЗАПУСК ---
async def main():
    await init_db()
    bot = Bot(token=BOT_TOKEN)
    dp = Dispatcher(storage=MemoryStorage())
    dp.include_router(router)
    await dp.start_polling(bot)

if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)
    asyncio.run(main())
