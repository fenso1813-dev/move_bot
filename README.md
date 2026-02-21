Vip_kanal, [22/02 2026 02:09]
import asyncio
import sqlite3
import os
from aiogram import Bot, Dispatcher, types, F
from aiogram.filters import Command, CommandStart
from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton

# -------------------------
# TOKEN va ADMIN ID
# -------------------------
BOT_TOKEN = "8525438393:AAG9OWgGSmepQzUloKaNaRo4qQWelvlv5o8"
ADMIN_ID = 7331921638

# -------------------------
# Click API ma’lumotlarini environment variables orqali olamiz
# -------------------------
CLICK_MERCHANT_ID = os.getenv("CLICK_MERCHANT_ID")
CLICK_SECRET_KEY = os.getenv("CLICK_SECRET_KEY")

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()

# -------------------------
# DATABASE
# -------------------------
if not os.path.exists("movies.db"):
    conn = sqlite3.connect("movies.db")
    c = conn.cursor()
    c.execute("""
    CREATE TABLE movies(
        code TEXT,
        episode INTEGER,
        title TEXT,
        file_id TEXT,
        is_vip INTEGER,
        vip_price INTEGER,
        external_url TEXT,
        PRIMARY KEY(code, episode)
    )
    """)
    c.execute("""
    CREATE TABLE vip_users(
        user_id INTEGER PRIMARY KEY,
        paid INTEGER DEFAULT 0
    )
    """)
    conn.commit()
    conn.close()

# -------------------------
# DATABASE FUNKSIYALARI
# -------------------------
def add_movie(code, episode, title, file_id, is_vip=0, vip_price=0, external_url=None):
    conn = sqlite3.connect("movies.db")
    c = conn.cursor()
    c.execute("INSERT OR REPLACE INTO movies VALUES (?, ?, ?, ?, ?, ?, ?)",
              (code, episode, title, file_id, is_vip, vip_price, external_url))
    conn.commit()
    conn.close()

def get_episodes(code):
    conn = sqlite3.connect("movies.db")
    c = conn.cursor()
    c.execute("SELECT episode, title, file_id, is_vip, vip_price, external_url FROM movies WHERE code=?", (code,))
    res = c.fetchall()
    conn.close()
    return sorted(res, key=lambda x: x[0])

def is_vip(user_id):
    conn = sqlite3.connect("movies.db")
    c = conn.cursor()
    c.execute("SELECT paid FROM vip_users WHERE user_id=?", (user_id,))
    res = c.fetchone()
    conn.close()
    return res and res[0]==1

def set_vip(user_id):
    conn = sqlite3.connect("movies.db")
    c = conn.cursor()
    c.execute("INSERT OR REPLACE INTO vip_users(user_id, paid) VALUES (?,1)", (user_id,))
    conn.commit()
    conn.close()

# -------------------------
# START COMMAND
# -------------------------
@dp.message(CommandStart())
async def start(msg: types.Message):
    kb = types.ReplyKeyboardMarkup(resize_keyboard=True)
    kb.add("Qidiruv")
    if msg.from_user.id == ADMIN_ID:
        kb.add("/admin")
        kb.add("⚙ Click sozlamalari")
    await msg.answer("Kino kodini kiriting yoki 'Qidiruv' tugmasini bosing", reply_markup=kb)

# -------------------------
# ADMIN PANEL
# -------------------------
@dp.message(Command("admin"))
async def admin(msg: types.Message):
    if msg.from_user.id != ADMIN_ID:
        return
    kb = types.ReplyKeyboardMarkup(resize_keyboard=True)
    kb.add("🎬 Oddiy kino qo‘shish", "⭐ VIP kino qo‘shish")
    kb.add("➕ Epizod qo‘shish", "📄 Ro‘yxat")
    kb.add("⚙ Click sozlamalari")
    await msg.answer("Admin panel:", reply_markup=kb)

# -------------------------
# ADD MOVIE
# -------------------------
@dp.message(F.text=="🎬 Oddiy kino qo‘shish")
async def add_normal(msg: types.Message):
    if msg.from_user.id != ADMIN_ID: return
    await msg.answer("Format: kod | episode | nomi | tashqi_havola(optional)")
    dp["mode"]="normal"

@dp.message(F.text=="⭐ VIP kino qo‘shish")
async def add_vip(msg: types.Message):
    if msg.from_user.id != ADMIN_ID: return
    await msg.answer("Format: kod | episode | nomi | narx | tashqi_havola(optional)")
    dp["mode"]="vip"

@dp.message(F.text=="➕ Epizod qo‘shish")
async def add_episode(msg: types.Message):
    if msg.from_user.id != ADMIN_ID: return
    await msg.answer("Format: kod | episode | nomi | VIP(0/1) | narx | tashqi_havola(optional) va faylni yuboring")
    dp["mode"]="episode"

Vip_kanal, [22/02 2026 02:09]
# -------------------------
# PARSE ADMIN INPUT
# -------------------------
@dp.message(F.text.regexp(r"^[A-Za-z0-9]+ \| \d+ \| .+"))
async def parse_admin(msg: types.Message):
    if msg.from_user.id != ADMIN_ID: return
    if "mode" not in dp: return
    parts = [p.strip() for p in msg.text.split("|")]
    code = parts[0]
    episode = int(parts[1])
    title = parts[2]
    vip = 0
    vip_price = 0
    external_url = None
    if dp["mode"]=="vip":
        try:
            vip_price = int(parts[3])
            vip = 1
            if len(parts)>4:
                external_url = parts[4]
        except:
            await msg.answer("VIP narx noto‘g‘ri kiritildi")
            return
    elif dp["mode"]=="episode":
        try:
            vip = int(parts[3])
            vip_price = int(parts[4])
            if len(parts)>5:
                external_url = parts[5]
        except:
            await msg.answer("Format noto‘g‘ri")
            return
    dp["pending"]={"code":code,"episode":episode,"title":title,"vip":vip,"vip_price":vip_price,"external_url":external_url}
    await msg.answer("Endi faylni yuboring (agar tashqi havola bo‘lsa fayl shart emas)")
    del dp["mode"]

# -------------------------
# RECEIVE FILE
# -------------------------
@dp.message(F.video | F.document)
async def receive_file(msg: types.Message):
    if msg.from_user.id != ADMIN_ID: return
    if "pending" not in dp: return
    file_id = None
    if msg.video or msg.document:
        file_id = msg.video.file_id if msg.video else msg.document.file_id
    data = dp["pending"]
    add_movie(data["code"], data["episode"], data["title"], file_id, data["vip"], data["vip_price"], data["external_url"])
    del dp["pending"]
    await msg.answer("Kino muvaffaqiyatli qo‘shildi!")

# -------------------------
# USER SEARCH
# -------------------------
@dp.message()
async def search(msg: types.Message):
    code = msg.text.strip().upper()
    episodes = get_episodes(code)
    if not episodes:
        return await msg.answer("Bunday kod topilmadi.")
    vip = any(e[3]==1 for e in episodes)
    if vip and not is_vip(msg.from_user.id):
        if not CLICK_MERCHANT_ID:
            return await msg.answer("Click API ma’lumotlari sozlanmagan. Admin bilan bog‘laning.")
        price = max(e[4] for e in episodes if e[3]==1)
        kb = InlineKeyboardMarkup().add(
            InlineKeyboardButton(text=f"💳 CLICK to‘lash ({price} so‘m)", url=f"https://my.click.uz/pay?merchant={CLICK_MERCHANT_ID}&amount={price}&account={msg.from_user.id}")
        )
        return await msg.answer(f"Bu VIP kino. Narxi: {price} so‘m. To‘lov uchun Click tugmasini bosing.", reply_markup=kb)
    kb = InlineKeyboardMarkup(row_width=3)
    for ep, t, f, v, p, url in episodes:
        if url:
            kb.insert(InlineKeyboardButton(text=str(ep), url=url))
        else:
            kb.insert(InlineKeyboardButton(text=str(ep), callback_data=f"watch_{code}_{ep}"))
    await msg.answer("Epizodni tanlang:", reply_markup=kb)

# -------------------------
# CALLBACK HANDLER
# -------------------------
@dp.callback_query()
async def cb_handler(call: types.CallbackQuery):
    data = call.data
    if data.startswith("watch_"):
        _, code, ep = data.split("_")
        ep = int(ep)
        episodes = get_episodes(code)
        for e in episodes:
            if e[0]==ep:
                if e[5]:  # external_url
                    await call.message.answer(f"Video havolasi: {e[5]}")
                elif e[2]:
                    await call.message.answer_video(e[2], caption=e[1])

# -------------------------
# RUN BOT
# -------------------------
async def main():
    await dp.start_polling(bot)

if name == "main":
    asyncio.run(main())
