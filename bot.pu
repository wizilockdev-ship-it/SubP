# ============================================================
#  Sub_P BOT — единый файл
# ============================================================

import asyncio
import logging
import os
from datetime import datetime

import asyncpg
import aiohttp
from aiohttp import web

from aiogram import Bot, Dispatcher, F
from aiogram.filters import CommandStart, Command
from aiogram.types import Message, CallbackQuery, InlineKeyboardMarkup, InlineKeyboardButton
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup

# ==================== CONFIG ====================
BOT_TOKEN        = "8646399919:AAGgCc5C6NINuc7vpSzfqrtxvgGpMr7lUpU"
DATABASE_URL     = "postgresql://neondb_owner:npg_nzGKijX4ftZ9@ep-purple-lab-za2zqjl5-pooler.c-2.eu-west-2.aws.neon.tech/neondb?sslmode=require&channel_binding=require"

CHANNEL_USERNAME = "@sub_p_work"
CHANNEL_URL      = "https://t.me/sub_p_work"
BOT_LINK         = "https://t.me/sub_p_bot"

# ID группы задаётся автоматически через /setgroup, но можно и вписать руками:
ADMIN_CHAT_ID    = 0                      # сюда запишется после /setgroup
GROUP_LINK       = "https://t.me/kdskajshsjshshsjalog"

PIARFLOW_API_KEY = "_BCTydDgwlcia7o4gr8aySmIkJ30GYro"
PIARFLOW_BASE    = "https://piarflow.com/v1"

REWARD_PER_SUB   = 1000
REFERRAL_REWARD  = 3500
COOLDOWN_MINUTES = 30
PRICE_PER_SUB    = 250
MIN_SUBS         = 10
# ================================================

logging.basicConfig(level=logging.INFO, format="%(asctime)s [%(levelname)s] %(message)s")
log = logging.getLogger("sub_p")

bot = Bot(BOT_TOKEN, default=DefaultBotProperties(parse_mode=ParseMode.HTML))
dp = Dispatcher()

# ==================== БАЗА ДАННЫХ ====================
pool: asyncpg.Pool = None

async def init_db():
    global pool
    pool = await asyncpg.create_pool(DATABASE_URL, min_size=1, max_size=5)
    async with pool.acquire() as conn:
        await conn.execute("""
            CREATE TABLE IF NOT EXISTS users (
                user_id BIGINT PRIMARY KEY,
                username TEXT,
                first_name TEXT,
                balance BIGINT DEFAULT 0,
                referrer_id BIGINT,
                referral_rewarded BOOLEAN DEFAULT FALSE,
                last_earn_at TIMESTAMP,
                created_at TIMESTAMP DEFAULT NOW()
            );
        """)
        await conn.execute("""
            CREATE TABLE IF NOT EXISTS earn_tasks (
                id SERIAL PRIMARY KEY,
                user_id BIGINT NOT NULL,
                link TEXT NOT NULL,
                status TEXT DEFAULT 'pending',
                created_at TIMESTAMP DEFAULT NOW()
            );
        """)
        await conn.execute("""
            CREATE TABLE IF NOT EXISTS growth_orders (
                id SERIAL PRIMARY KEY,
                user_id BIGINT NOT NULL,
                channel_link TEXT NOT NULL,
                quantity INT NOT NULL,
                price_per_sub INT NOT NULL,
                total_cost BIGINT NOT NULL,
                status TEXT DEFAULT 'pending_payment',
                created_at TIMESTAMP DEFAULT NOW()
            );
        """)
        await conn.execute("""
            CREATE TABLE IF NOT EXISTS settings (
                key TEXT PRIMARY KEY,
                value TEXT
            );
        """)
    log.info("DB initialized")

async def get_setting(key: str):
    async with pool.acquire() as conn:
        return await conn.fetchval("SELECT value FROM settings WHERE key=$1", key)

async def set_setting(key: str, value: str):
    async with pool.acquire() as conn:
        await conn.execute(
            "INSERT INTO settings (key,value) VALUES ($1,$2) "
            "ON CONFLICT (key) DO UPDATE SET value=EXCLUDED.value", key, value)

async def get_or_create_user(user_id, username, first_name, referrer_id=None):
    async with pool.acquire() as conn:
        row = await conn.fetchrow("SELECT * FROM users WHERE user_id=$1", user_id)
        if not row:
            await conn.execute(
                "INSERT INTO users (user_id, username, first_name, referrer_id) VALUES ($1,$2,$3,$4)",
                user_id, username, first_name, referrer_id)
            row = await conn.fetchrow("SELECT * FROM users WHERE user_id=$1", user_id)
        return row

async def get_balance(user_id):
    async with pool.acquire() as conn:
        v = await conn.fetchval("SELECT balance FROM users WHERE user_id=$1", user_id)
        return v or 0

async def add_balance(user_id, amount):
    async with pool.acquire() as conn:
        await conn.execute("UPDATE users SET balance = balance + $1 WHERE user_id=$2", amount, user_id)

async def get_cooldown_left(user_id):
    async with pool.acquire() as conn:
        last = await conn.fetchval("SELECT last_earn_at FROM users WHERE user_id=$1", user_id)
    if not last:
        return 0
    delta = datetime.utcnow() - last.replace(tzinfo=None)
    return max(0, COOLDOWN_MINUTES * 60 - int(delta.total_seconds()))

async def set_last_earn(user_id):
    async with pool.acquire() as conn:
        await conn.execute("UPDATE users SET last_earn_at=NOW() WHERE user_id=$1", user_id)

async def clear_pending_tasks(user_id):
    async with pool.acquire() as conn:
        await conn.execute("DELETE FROM earn_tasks WHERE user_id=$1 AND status='pending'", user_id)

async def save_tasks(user_id, links):
    await clear_pending_tasks(user_id)
    async with pool.acquire() as conn:
        for link in links:
            await conn.execute("INSERT INTO earn_tasks (user_id, link) VALUES ($1,$2)", user_id, link)

async def get_next_task(user_id):
    async with pool.acquire() as conn:
        return await conn.fetchrow(
            "SELECT * FROM earn_tasks WHERE user_id=$1 AND status='pending' ORDER BY id LIMIT 1", user_id)

async def mark_task_done(task_id):
    async with pool.acquire() as conn:
        await conn.execute("UPDATE earn_tasks SET status='done' WHERE id=$1", task_id)

async def count_pending(user_id):
    async with pool.acquire() as conn:
        return await conn.fetchval(
            "SELECT COUNT(*) FROM earn_tasks WHERE user_id=$1 AND status='pending'", user_id) or 0

async def try_reward_referrer(user_id):
    async with pool.acquire() as conn:
        user = await conn.fetchrow("SELECT referrer_id, referral_rewarded FROM users WHERE user_id=$1", user_id)
        if not user or user["referral_rewarded"] or not user["referrer_id"]:
            return False
        done = await conn.fetchval(
            "SELECT COUNT(*) FROM earn_tasks WHERE user_id=$1 AND status='done'", user_id)
        if done > 0:
            await conn.execute("UPDATE users SET referral_rewarded=TRUE WHERE user_id=$1", user_id)
            await conn.execute("UPDATE users SET balance = balance + $1 WHERE user_id=$2",
                               REFERRAL_REWARD, user["referrer_id"])
            return True
        return False

async def get_stats():
    async with pool.acquire() as conn:
        total = await conn.fetchval("SELECT COUNT(*) FROM users") or 0
        today = await conn.fetchval("SELECT COUNT(*) FROM users WHERE created_at::date = CURRENT_DATE") or 0
        start = await conn.fetchval("SELECT MIN(created_at) FROM users")
        return total, today, start

async def create_growth_order(user_id, channel_link, quantity, price_per_sub):
    total = quantity * price_per_sub
    async with pool.acquire() as conn:
        return await conn.fetchval(
            "INSERT INTO growth_orders (user_id, channel_link, quantity, price_per_sub, total_cost) "
            "VALUES ($1,$2,$3,$4,$5) RETURNING id",
            user_id, channel_link, quantity, price_per_sub, total)

# ==================== PIARFLOW API ====================
PF_HEADERS = {
    "Authorization": f"Bearer {PIARFLOW_API_KEY}",
    "Content-Type": "application/json",
}

async def pf_get_sponsors(user_id, chat_id, max_sponsors=10):
    async with aiohttp.ClientSession(timeout=aiohttp.ClientTimeout(total=15)) as s:
        async with s.post(f"{PIARFLOW_BASE}/sponsors",
                          json={"user_id": user_id, "chat_id": chat_id, "max_sponsors": max_sponsors},
                          headers=PF_HEADERS) as r:
            data = await r.json(content_type=None)
            if r.status >= 400:
                raise RuntimeError(data.get("message") or f"PiarFlow {r.status}")
            return data.get("sponsors") or []

async def pf_check_sponsor(user_id, link):
    async with aiohttp.ClientSession(timeout=aiohttp.ClientTimeout(total=15)) as s:
        async with s.post(f"{PIARFLOW_BASE}/sponsors/check",
                          json={"user_id": user_id, "links": [link]},
                          headers=PF_HEADERS) as r:
            data = await r.json(content_type=None)
            if r.status >= 400:
                raise RuntimeError(data.get("message") or f"PiarFlow {r.status}")
            sponsors = data.get("sponsors") or []
            return bool(sponsors) and sponsors[0].get("status") == "subscribed"

# ==================== FSM ====================
class GrowthStates(StatesGroup):
    waiting_link = State()
    waiting_quantity = State()

# ==================== КЛАВИАТУРЫ ====================
def subscribe_kb():
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="📢 Подписаться", url=CHANNEL_URL)],
        [InlineKeyboardButton(text="✅ Проверить", callback_data="check_sub")],
    ])

def main_menu_kb():
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="👤 Профиль", callback_data="profile")],
        [InlineKeyboardButton(text="💰 Заработать SP", callback_data="earn")],
        [InlineKeyboardButton(text="🚀 Бесплатная накрутка", callback_data="growth")],
        [InlineKeyboardButton(text="📊 Статистика", callback_data="stats")],
    ])

def earn_task_kb(link):
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="📢 Подписаться", url=link)],
        [InlineKeyboardButton(text="✅ Проверить", callback_data="earn_check")],
    ])

def cancel_kb():
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="❌ Отменить", callback_data="cancel_growth")],
    ])

def profile_kb(user_id):
    ref_link = f"{BOT_LINK}?start=ref_{user_id}"
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🔗 Моя реферальная ссылка", url=ref_link)],
        [InlineKeyboardButton(text="🏠 В меню", callback_data="main_menu")],
    ])

def stats_kb():
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="📢 Наш Telegram канал", url=CHANNEL_URL)],
        [InlineKeyboardButton(text="🏠 В меню", callback_data="main_menu")],
    ])

# ==================== ПРОВЕРКА ПОДПИСКИ ====================
async def is_subscribed(user_id):
    try:
        m = await bot.get_chat_member(CHANNEL_USERNAME, user_id)
        return m.status in ("member", "administrator", "creator")
    except Exception as e:
        log.warning(f"sub check error: {e}")
        return False

# ==================== /start ====================
@dp.message(CommandStart())
async def cmd_start(msg: Message):
    args = msg.text.split(maxsplit=1)
    referrer_id = None
    if len(args) > 1 and args[1].startswith("ref_"):
        try:
            referrer_id = int(args[1].replace("ref_", ""))
            if referrer_id == msg.from_user.id:
                referrer_id = None
        except ValueError:
            pass
    await get_or_create_user(msg.from_user.id, msg.from_user.username or "", msg.from_user.first_name or "", referrer_id)
    await msg.answer(
        "👋 <b>Добро пожаловать в Sub_P!</b>\n\n"
        "Твой бот бесплатной накрутки.\n\n"
        "Для использования подпишись на наш канал 👇",
        reply_markup=subscribe_kb())

@dp.callback_query(F.data == "check_sub")
async def check_sub(cb: CallbackQuery):
    if await is_subscribed(cb.from_user.id):
        await cb.message.edit_text(
            "✅ Подписка подтверждена!\n\nДобро пожаловать в <b>Главное меню</b>.",
            reply_markup=main_menu_kb())
    else:
        await cb.answer("❌ Ты ещё не подписался на канал!", show_alert=True)

@dp.callback_query(F.data == "main_menu")
async def back_to_menu(cb: CallbackQuery):
    await cb.message.edit_text("🏠 <b>Главное меню</b>\n\nВыбери действие:", reply_markup=main_menu_kb())

# ==================== КОМАНДА УСТАНОВКИ ГРУППЫ ====================
@dp.message(Command("setgroup"))
async def set_group(msg: Message):
    """Выполнить в группе, куда бот добавлен админом."""
    if msg.chat.type in ("group", "supergroup"):
        await set_setting("admin_chat_id", str(msg.chat.id))
        await msg.answer(
            f"✅ Группа для уведомлений установлена!\n"
            f"ID: <code>{msg.chat.id}</code>\n"
            f"Теперь все заказы будут приходить сюда.")

async def get_admin_chat_id():
    """Получить ID группы: из БД, иначе из конфига."""
    if ADMIN_CHAT_ID:
        return ADMIN_CHAT_ID
    val = await get_setting("admin_chat_id")
    if val:
        return int(val)
    return None

# ==================== ПРОФИЛЬ ====================
@dp.callback_query(F.data == "profile")
async def profile(cb: CallbackQuery):
    user = await get_or_create_user(cb.from_user.id, cb.from_user.username or "", cb.from_user.first_name or "")
    balance = await get_balance(cb.from_user.id)
    nick = f"@{user['username']}" if user["username"] else user["first_name"]
    ref_link = f"{BOT_LINK}?start=ref_{cb.from_user.id}"
    text = (
        "👤 <b>Твой профиль</b>\n\n"
        f"🆔 ID: <code>{user['user_id']}</code>\n"
        f"📝 Ник: {nick}\n"
        f"💰 Баланс: <b>{balance} SP</b>\n\n"
        f"🔗 <b>Реферальная ссылка:</b>\n<code>{ref_link}</code>\n\n"
        f"<i>За каждого реферала, который подпишется хотя бы на 1 канал из заданий, "
        f"ты получишь {REFERRAL_REWARD} SP.</i>"
    )
    await cb.message.edit_text(text, reply_markup=profile_kb(cb.from_user.id), disable_web_page_preview=True)

# ==================== ЗАРАБОТАТЬ SP ====================
async def show_next_task(cb: CallbackQuery):
    user_id = cb.from_user.id
    task = await get_next_task(user_id)
    if not task:
        await set_last_earn(user_id)
        await cb.message.edit_text(
            "🎉 <b>Каналы закончились!</b>\n\n"
            f"Приходите через {COOLDOWN_MINUTES} минут за новыми заданиями.",
            reply_markup=main_menu_kb())
        return
    remaining = await count_pending(user_id)
    await cb.message.edit_text(
        "💰 <b>Задание</b>\n\n"
        "Подпишись на канал ниже и нажми «Проверить».\n"
        f"Награда: <b>{REWARD_PER_SUB} SP</b>\n"
        f"Осталось заданий: {remaining}",
        reply_markup=earn_task_kb(task["link"]),
        disable_web_page_preview=True)

@dp.callback_query(F.data == "earn")
async def earn_start(cb: CallbackQuery):
    user_id = cb.from_user.id
    left = await get_cooldown_left(user_id)
    if left > 0:
        await cb.answer(f"⏳ Подожди ещё {left // 60}м {left % 60}с", show_alert=True)
        return
    if await count_pending(user_id) > 0:
        await show_next_task(cb)
        return
    await cb.answer("Загружаю задания…")
    try:
        sponsors = await pf_get_sponsors(user_id, cb.message.chat.id, max_sponsors=10)
    except Exception:
        log.exception("pf_get_sponsors")
        await cb.message.edit_text("⚠️ Не удалось получить задания. Попробуй позже.", reply_markup=main_menu_kb())
        return
    links = [s["link"] for s in sponsors if s.get("status") != "subscribed"]
    if not links:
        await set_last_earn(user_id)
        await cb.message.edit_text("🎉 <b>Каналы закончились!</b>\n\nПриходите через 30 минут.", reply_markup=main_menu_kb())
        return
    await clear_pending_tasks(user_id)
    await save_tasks(user_id, links)
    await show_next_task(cb)

@dp.callback_query(F.data == "earn_check")
async def earn_check(cb: CallbackQuery):
    user_id = cb.from_user.id
    task = await get_next_task(user_id)
    if not task:
        await cb.answer("Задание не найдено. Нажми «Заработать SP» заново.", show_alert=True)
        return
    await cb.answer("Проверяю…")
    try:
        ok = await pf_check_sponsor(user_id, task["link"])
    except Exception:
        log.exception("pf_check_sponsor")
        await cb.answer("Ошибка проверки. Попробуй ещё раз.", show_alert=True)
        return
    if not ok:
        await cb.answer("❌ Подписка не засчитана. Подпишись и попробуй снова.", show_alert=True)
        return
    await mark_task_done(task["id"])
    await add_balance(user_id, REWARD_PER_SUB)

    rewarded = await try_reward_referrer(user_id)
    if rewarded:
        u = await get_or_create_user(user_id, cb.from_user.username or "", cb.from_user.first_name or "")
        if u["referrer_id"]:
            try:
                await bot.send_message(
                    u["referrer_id"],
                    "🎉 <b>Реферал активирован!</b>\n\n"
                    f"Пользователь @{cb.from_user.username or cb.from_user.first_name} выполнил первое задание.\n"
                    f"Тебе начислено <b>{REFERRAL_REWARD} SP</b>!")
            except Exception:
                pass
    await show_next_task(cb)

# ==================== БЕСПЛАТНАЯ НАКРУТКА ====================
@dp.callback_query(F.data == "growth")
async def growth_start(cb: CallbackQuery, state: FSMContext):
    await state.set_state(GrowthStates.waiting_link)
    await cb.message.edit_text(
        "🚀 <b>Бесплатная накрутка</b>\n\n"
        "Накрутка производится на канал.\n"
        "Отправьте публичную ссылку или ссылку с заявками:\n\n"
        f"<i>Цена за одного подписчика: {PRICE_PER_SUB} SP. Минимум {MIN_SUBS}.</i>",
        reply_markup=cancel_kb())
    await cb.answer()

@dp.message(GrowthStates.waiting_link)
async def growth_link(msg: Message, state: FSMContext):
    link = msg.text.strip()
    if not (link.startswith("https://t.me/") or link.startswith("@")):
        await msg.answer("❌ Отправьте корректную ссылку на канал (https://t.me/... или @канал).", reply_markup=cancel_kb())
        return
    await state.update_data(channel_link=link)
    await state.set_state(GrowthStates.waiting_quantity)
    await msg.answer(
        "✅ Ссылка принята.\n\n"
        f"💰 Цена за одного подписчика: <b>{PRICE_PER_SUB} SP</b>\n"
        f"📊 Минимум: <b>{MIN_SUBS}</b>\n\n"
        "Введите, сколько подписчиков нужно:",
        reply_markup=cancel_kb())

@dp.message(GrowthStates.waiting_quantity)
async def growth_quantity(msg: Message, state: FSMContext):
    try:
        qty = int(msg.text.strip())
    except ValueError:
        await msg.answer("❌ Введите число.", reply_markup=cancel_kb())
        return
    if qty < MIN_SUBS:
        await msg.answer(f"❌ Минимум {MIN_SUBS} подписчиков.", reply_markup=cancel_kb())
        return
    data = await state.get_data()
    channel_link = data["channel_link"]
    user_id = msg.from_user.id
    cost = qty * PRICE_PER_SUB
    balance = await get_balance(user_id)
    if balance < cost:
        await msg.answer(
            "❌ <b>Недостаточно SP!</b>\n\n"
            f"Нужно: {cost} SP\n"
            f"У тебя: {balance} SP\n\n"
            "Заработай SP в главном меню.",
            reply_markup=main_menu_kb())
        await state.clear()
        return
    await add_balance(user_id, -cost)
    order_id = await create_growth_order(user_id, channel_link, qty, PRICE_PER_SUB)
    await state.clear()

    admin_id = await get_admin_chat_id()
    if admin_id:
        try:
            await bot.send_message(
                admin_id,
                "🚀 <b>Новый заказ на накрутку!</b>\n\n"
                f"👤 Пользователь: {msg.from_user.full_name} (@{msg.from_user.username or 'нет'})\n"
                f"🆔 ID: <code>{user_id}</code>\n"
                f"📢 Канал: {channel_link}\n"
                f"👥 Подписчиков: <b>{qty}</b>\n"
                f"💰 Стоимость: <b>{cost} SP</b>\n"
                f"🆔 Заказ: #{order_id}")
        except Exception as e:
            log.error(f"admin notify error: {e}")
    else:
        log.warning("ADMIN_CHAT_ID не задан — выполни /setgroup в группе")

    await msg.answer(
        "✅ <b>Заказ принят!</b>\n\n"
        f"📢 Канал: {channel_link}\n"
        f"👥 Подписчиков: {qty}\n"
        f"💰 Стоимость: {cost} SP\n\n"
        "Мы свяжемся с вами после выполнения.",
        reply_markup=main_menu_kb())

@dp.callback_query(F.data == "cancel_growth")
async def cancel_growth(cb: CallbackQuery, state: FSMContext):
    await state.clear()
    await cb.message.edit_text("❌ Действие отменено.", reply_markup=main_menu_kb())

# ==================== СТАТИСТИКА ====================
@dp.callback_query(F.data == "stats")
async def stats(cb: CallbackQuery):
    total, today, start = await get_stats()
    start_str = start.strftime("%d.%m.%Y") if start else "—"
    await cb.message.edit_text(
        "📊 <b>Статистика бота</b>\n\n"
        f"👥 Пользователей за всё время: <b>{total}</b>\n"
        f"📅 Пользователей сегодня: <b>{today}</b>\n"
        f"🚀 Дата старта: <b>{start_str}</b>",
        reply_markup=stats_kb())

# ==================== HEALTH SERVER ====================
async def health_server():
    app = web.Application()
    app.router.add_get("/", lambda r: web.Response(text="Bot is running"))
    app.router.add_get("/health", lambda r: web.Response(text="OK"))
    runner = web.AppRunner(app)
    await runner.setup()
    port = int(os.environ.get("PORT", 8080))
    site = web.TCPSite(runner, "0.0.0.0", port)
    await site.start()
    log.info(f"Health server on port {port}")

# ==================== ЗАПУСК ====================
async def main():
    await init_db()
    asyncio.create_task(health_server())
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
