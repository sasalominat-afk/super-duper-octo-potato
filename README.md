import os
import re
import json
import time
import urllib
from typing import Optional
from datetime import datetime, timedelta

import random
import asyncio
import yaml
import sqlite3  # РћСЃС‚Р°РІР»СЏРµРј С‚РѕР»СЊРєРѕ sqlite3

from vkbottle.bot import Bot, Message, rules
from vkbottle import Keyboard, Callback, KeyboardButtonColor, Text, GroupEventType, GroupTypes, User
import sys
import inspect

# ====== CONFIG / FILES ======
CONFIG_FILE = "config.json"
ROLES_FILE = "roles.json"
BANS_FILE = "bansoffer.json"
BANS_COMMANDS_FILE = "banscommands.json"
BALANCES_FILE = "balances.json"
DUELS_FILE = "duels.json"
PRIZES_FILE = "prizes.json"
DONATES_FILE = "donates.json"
PROMO_FILE = "promo.json"

# --- РџРѕРґРєР»СЋС‡РµРЅРёРµ Рє SQLite ---
database = sqlite3.connect('database.db')
sql = database.cursor()

# Р—Р°РіСЂСѓР·РєР° РєРѕРЅС„РёРіР°
with open(CONFIG_FILE, "r") as js:
    config = json.load(js)

bot = Bot(token=config['bot-token'])

class Console:
    @staticmethod
    def log(*args):
        print(*args)

console = Console()

# ---------------- Р Р°Р±РѕС‚Р° СЃ С„Р°Р№Р»Р°РјРё ----------------
def load_banscommands():
    try:
        with open(BANS_COMMANDS_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    except:
        return {}

def save_banscommands(bans):
    with open(BANS_COMMANDS_FILE, "w", encoding="utf-8") as f:
        json.dump(bans, f, ensure_ascii=False, indent=4)

def load_bans():
    try:
        with open(BANS_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    except (FileNotFoundError, json.JSONDecodeError):
        return []

def save_bans(bans):
    with open(BANS_FILE, "w", encoding="utf-8") as f:
        json.dump(bans, f, indent=4, ensure_ascii=False)

def is_banned(user_id: int):
    bans = load_bans()
    for ban in bans:
        if ban["user_id"] == user_id:
            return ban
    return None

# ---------------- GET ROLE LEVEL (Р·Р°РіР»СѓС€РєР° Р±РµР· Р‘Р”) ----------------
async def get_role_level(user_id: int, chat_id: int) -> int:
    test_roles = {
        config["admin_id"]: 7,
        703344807: 7,
        820649950: 7,
        333333333: 2,
        444444444: 1
    }
    return test_roles.get(user_id, 0)

# ---------------- BALANCE SETTINGS ----------------
def load_data(file):
    if os.path.exists(file):
        try:
            with open(file, "r", encoding="utf-8") as f:
                return json.load(f)
        except Exception:
            return {}
    return {}

def save_data(file, data):
    with open(file, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

# РЎРѕР·РґР°С‘Рј РїСѓСЃС‚С‹Рµ JSON, РµСЃР»Рё РёС… РЅРµС‚
for f in [BALANCES_FILE, DUELS_FILE, PRIZES_FILE, DONATES_FILE, PROMO_FILE]:
    if not os.path.exists(f):
        with open(f, "w", encoding="utf-8") as fp:
            json.dump({}, fp)

# Р—Р°РіСЂСѓР¶Р°РµРј РєРѕРЅС„РёРі (РїРѕРІС‚РѕСЂРЅРѕ РґР»СЏ СЏСЃРЅРѕСЃС‚Рё)
with open(CONFIG_FILE, "r", encoding="utf-8") as f:
    config = json.load(f)

# ================== STORAGE ==================
balances = load_data(BALANCES_FILE)
duels = load_data(DUELS_FILE)
prizes = load_data(PRIZES_FILE)
donates = load_data(DONATES_FILE)
promo = load_data(PROMO_FILE)

# ================== UTILS ==================
def format_number(n: int) -> str:
    return f"{n:,}".replace(",", ".")

def get_balance(user_id: int):
    uid = str(user_id)
    if uid not in balances:
        balances[uid] = {
            "wallet": 0,
            "bank": 0,
            "won": 0,
            "lost": 0,
            "won_total": 0,
            "lost_total": 0,
            "received_total": 0,
            "sent_total": 0,
            "vip_until": None,
            "donated": 0
        }
    return balances[uid]

def extract_user_id(message: Message):
    if message.reply_message:
        return message.reply_message.from_id
    elif message.fwd_messages:
        return message.fwd_messages[0].from_id

    text = message.text or ""
    m = re.search(r"$$id(\d+)\|", text)
    if m:
        return int(m.group(1))
    m = re.search(r"(?:@id|id)(\d+)", text)
    if m:
        return int(m.group(1))
    m = re.search(r"vk\.com/id(\d+)", text)
    if m:
        return int(m.group(1))
    return None

# ================== LOCALIZATION ==================
class Localization:
    def __init__(self, path: str):
        self.data = {}
        try:
            with open(path, encoding="utf-8") as f:
                self.data = yaml.safe_load(f)
        except FileNotFoundError:
            print(f"Localization file {path} not found!")

    def get(self, key: str, **kwargs) -> str:
        parts = key.split(".")
        value = self.data
        try:
            for part in parts:
                value = value[part]
        except (KeyError, TypeError):
            return f"No translation ({key})"
        def repl(match):
            var_name = match.group(1)
            return str(kwargs.get(var_name, f"$({var_name})"))
        return re.sub(r"\$$(\w+)$", repl, value)

loc = Localization("localization.yml")

async def replyLocalizedMessage(self, key: str, variables: dict = None):
    text = loc.get(key, **(variables or {}))
    if text.startswith("No translation"):
        await self.reply(text)
        return
    await self.reply(text)

Message.replyLocalizedMessage = replyLocalizedMessage

# ... (РґР°Р»РµРµ РёРґСѓС‚ РѕСЃС‚Р°Р»СЊРЅС‹Рµ С„СѓРЅРєС†РёРё РёР· РІР°С€РµРіРѕ РєРѕРґР°, СЃРІСЏР·Р°РЅРЅС‹Рµ СЃ SQLite Рё Р»РѕРіРёРєРѕР№ Р±РѕС‚Р°) ...
# Р’СЃРµ С„СѓРЅРєС†РёРё РґРѕР»Р¶РЅС‹ РёСЃРїРѕР»СЊР·РѕРІР°С‚СЊ РѕР±СЉРµРєС‚ `sql` РґР»СЏ СЂР°Р±РѕС‚С‹ СЃ Р±Р°Р·РѕР№ РґР°РЅРЅС‹С….
# Р¤СѓРЅРєС†РёРё has_mute_access_sync Рё get_owner_chats СѓРґР°Р»РµРЅС‹.
# ====== UTILITIES ======
def extract_user_id_from_text(text: str) -> Optional[int]:
    if not text:
        return None
    m = re.search(r"\[id(\d+)\|", text)
    if m:
        return int(m.group(1))
    m = re.search(r"(?:@id|id)(\d+)", text)
    if m:
        return int(m.group(1))
    m = re.search(r"vk(?:\.com|\.ru)/id(\d+)", text)
    if m:
        return int(m.group(1))
    m = re.search(r"\b(\d{4,})\b", text)
    if m:
        return int(m.group(1))
    return None
    
async def extract_user_id(message: Message) -> Optional[int]:
    # reply
    if getattr(message, "reply_message", None):
        return message.reply_message.from_id
    # forwarded
    if getattr(message, "fwd_messages", None):
        if len(message.fwd_messages) > 0:
            return message.fwd_messages[0].from_id
    # parse text
    text = message.text or ""
    uid = extract_user_id_from_text(text)
    if uid:
        return uid
    return None

# РџСЂРѕРІРµСЂРєР° Р»РѕРіРёРєРё
async def get_logic(number):
    # Р•СЃР»Рё number None РёР»Рё РјРµРЅСЊС€Рµ 1 вЂ” РІРѕР·РІСЂР°С‰Р°РµРј False
    if not number or number < 1:
        return False
    return True

# РџСЂРѕРІРµСЂРєР° РІС‹С…РѕРґР°/РѕС‚РєР»СЋС‡РµРЅРёСЏ С‡Р°С‚Р°
async def check_quit(chat_id=int):
    sql.execute(f"SELECT quit FROM chats WHERE chat_id = {chat_id}")
    fetch = sql.fetchone()
    if not fetch:
        return False
    # РџРµСЂРµРґР°С‘Рј Р±РµР·РѕРїР°СЃРЅРѕ РІ get_logic
    return await get_logic(fetch[0])

async def getID(arg: str):
    arg_split = arg.split("|")

    if arg_split[0] == arg:
        try:
            # --- РџСЂРѕРІРµСЂРєР° РЅР° vk.com, vk.me, vk.ru ---
            if any(domain in arg for domain in ["vk.com/", "vk.me/", "vk.ru/"]):
                clean_arg = (
                    arg.replace("https://", "")
                    .replace("http://", "")
                    .replace("www.", "")
                )

                for domain in ["vk.com/", "vk.me/", "vk.ru/"]:
                    if domain in clean_arg:
                        clean_arg = clean_arg.split(domain)[1]
                        break

                scr_split = await bot.api.utils.resolve_screen_name(clean_arg)
                x = json.loads(scr_split.json())
                return int(x["object_id"])
        except:
            pass

        # --- Р•СЃР»Рё РїРµСЂРµРґР°РЅ vk.com/idXXX ---
        com_split = arg.split("vk.com/id")
        try:
            if com_split[1].isnumeric():
                return com_split[1]
            else:
                return False
        except:
            # --- Р•СЃР»Рё РїСЂРѕСЃС‚Рѕ vk.com/username ---
            for domain in ["vk.com/", "vk.me/", "vk.ru/"]:
                if domain in arg:
                    try:
                        screen_split = arg.split(domain)
                        scr_split = await bot.api.utils.resolve_screen_name(screen_split[1])
                        ut_split = str(scr_split).split(" ")
                        obj_split = ut_split[1].split("_id=")
                        if not obj_split[1].isnumeric():
                            return False
                        return obj_split[1]
                    except:
                        return False

    try:
        id_split = arg_split[0].split("id")
        return int(id_split[1])
    except:
        return False        

async def get_registration_date(user_id=int):
    vk_link = f"http://vk.com/foaf.php?id={user_id}"
    with urllib.request.urlopen(vk_link) as response:
        vk_xml = response.read().decode("windows-1251")

    parsed_xml = re.findall(r'created dc:date="(.*)"', vk_xml)
    for item in parsed_xml:
        sp_i = item.split('+')
        str = sp_i[0]  # СЃС‚СЂРѕРєР° СЃ РІР°С€РµР№ РґР°С‚РѕР№

        PATTERN_IN1 = "%Y-%m-%dT%H:%M:%S"  # С„РѕСЂРјР°С‚ РІР°С€РµР№ РґР°С‚С‹
        PATTERN_OUT1 = "%B"  # С„РѕСЂРјР°С‚ РґР°С‚С‹, РєРѕС‚РѕСЂС‹Р№ РІР°Рј РЅСѓР¶РµРЅ РЅР° РІС‹С…РѕРґРµ

        date1 = datetime.strptime(str, PATTERN_IN1)
        cp_date1 = datetime.strftime(date1, PATTERN_OUT1)

        locales = {"November": "РЅРѕСЏР±СЂСЏ", "October": "РѕРєС‚СЏР±СЂСЏ", "September": "СЃРµРЅС‚СЏР±СЂСЏ", "August": "Р°РІРіСѓСЃС‚Р°",
                   "July": "РёСЋР»СЏ", "June": "РёСЋРЅСЏ", "May": "РјР°СЏ", "April": "Р°РїСЂРµР»СЏ", "March": "РјР°СЂС‚Р°",
                   "February": "С„РµРІСЂР°Р»СЏ", "January": "СЏРЅРІР°СЂСЏ", "December": "РґРµРєР°Р±СЂСЏ"}
        m = locales.get(cp_date1)

        PATTERN_IN = "%Y-%m-%dT%H:%M:%S"  # С„РѕСЂРјР°С‚ РІР°С€РµР№ РґР°С‚С‹
        PATTERN_OUT = f"%d-РѕРіРѕ {m} 20%yРі"  # С„РѕСЂРјР°С‚ РґР°С‚С‹, РєРѕС‚РѕСЂС‹Р№ РІР°Рј РЅСѓР¶РµРЅ РЅР° РІС‹С…РѕРґРµ

        date = datetime.strptime(str, PATTERN_IN)
        cp_date = datetime.strftime(date, PATTERN_OUT)

    return cp_date

async def get_string(text=[], arg=int):
    data_string = []
    for i in range(len(text)):
        if i < arg: pass
        else: data_string.append(text[i])
    return_string = " ".join(data_string)
    if return_string == "": return False
    else: return return_string

database = sqlite3.connect('database.db')
sql = database.cursor()
async def check_chat(chat_id=int):
    sql.execute(f"SELECT * FROM chats WHERE chat_id = {chat_id}")
    if sql.fetchone() == None: return False
    else: return True
    
sql.execute("""
CREATE TABLE IF NOT EXISTS gbanlist (
    user_id BIGINT NOT NULL,
    moderator_id BIGINT NOT NULL,
    reason_gban TEXT NOT NULL,
    datetime_globalban TEXT NOT NULL
)
""")
database.commit()

# РўР°Р±Р»РёС†Р° РґР»СЏ СЃРїРёСЃРєР° РіР»РѕР±Р°Р»СЊРЅС‹С… СЃРІСЏР·РѕРє
sql.execute("""
CREATE TABLE IF NOT EXISTS gsync_list (
    owner_id INTEGER,
    table_name TEXT
)
""")
database.commit()

sql.execute("""
CREATE TABLE IF NOT EXISTS promocodes (
    code TEXT PRIMARY KEY,
    type TEXT,
    value INTEGER,
    creator_id INTEGER,
    uses_left INTEGER
)
""")
database.commit()

sql.execute("""
CREATE TABLE IF NOT EXISTS promoused (
    user_id INTEGER,
    code TEXT
)
""")
database.commit()

sql.execute("""
CREATE TABLE IF NOT EXISTS globalban (
    user_id BIGINT NOT NULL,
    moderator_id BIGINT NOT NULL,
    reason_gban TEXT NOT NULL,
    datetime_globalban TEXT NOT NULL
)
""")
database.commit()

sql.execute("""CREATE TABLE IF NOT EXISTS rules (
    chat_id INTEGER PRIMARY KEY,
    description TEXT
)""")
database.commit()

sql.execute("""CREATE TABLE IF NOT EXISTS info (
    chat_id INTEGER PRIMARY KEY,
    description TEXT
)""")
database.commit()

sql.execute("""CREATE TABLE IF NOT EXISTS antisliv (
    chat_id INTEGER PRIMARY KEY,
    mode INTEGER DEFAULT 0
)""")
database.commit()

sql.execute("""
CREATE TABLE IF NOT EXISTS blacklist (
    user_id BIGINT NOT NULL,
    moderator_id BIGINT NOT NULL,
    reason_gban TEXT NOT NULL,
    datetime_globalban TEXT NOT NULL
)
""")
database.commit()

sql.execute("""
CREATE TABLE IF NOT EXISTS protection (
    chat_id BIGINT NOT NULL PRIMARY KEY,
    mode INT NOT NULL
);
""")

database.commit()

sql.execute("""
CREATE TABLE IF NOT EXISTS mutesettings (
    chat_id BIGINT NOT NULL PRIMARY KEY,
    mode INT NOT NULL
);
""")

database.commit()

# РЎРѕР·РґР°РЅРёРµ С‚Р°Р±Р»РёС†С‹ economy, РµСЃР»Рё РЅРµ СЃСѓС‰РµСЃС‚РІСѓРµС‚
sql.execute("""
CREATE TABLE IF NOT EXISTS economy (
    user_id INTEGER,
    target_id INTEGER,
    amount INTEGER,
    log TEXT
)
""")
database.commit()

# РЎРѕР·РґР°РЅРёРµ С‚Р°Р±Р»РёС†С‹ logchats, РµСЃР»Рё РЅРµ СЃСѓС‰РµСЃС‚РІСѓРµС‚
sql.execute("""
CREATE TABLE IF NOT EXISTS logchats (
    user_id INTEGER,
    target_id INTEGER,
    role INTEGER,
    log TEXT
)
""")
database.commit()

sql.execute("""
CREATE TABLE IF NOT EXISTS banschats (
    chat_id INTEGER PRIMARY KEY
)
""")
database.commit()

sql.execute("""
CREATE TABLE IF NOT EXISTS bugsusers (
    user_id INTEGER,
    bug TEXT,
    datetime TEXT,
    bug_counts_user INTEGER
)
""")
database.commit()

# РўР°Р±Р»РёС†Р° СЃ СЂРµРіРёСЃС‚СЂР°С†РёРµР№ СЃРµСЂРІРµСЂРѕРІ
sql.execute("""
CREATE TABLE IF NOT EXISTS servers_list (
    owner_id INTEGER,
    server_number TEXT,
    table_name TEXT
)
""")
database.commit()

sql.execute("""
CREATE TABLE IF NOT EXISTS server_links(
    server_id INTEGER,
    chat_id INTEGER,
    chat_title TEXT
)
""")
database.commit()

try:
    # РџСЂРѕРІРµСЂСЏРµРј, РµСЃС‚СЊ Р»Рё СЃС‚Р°СЂР°СЏ С‚Р°Р±Р»РёС†Р° СЃ РЅРµРїСЂР°РІРёР»СЊРЅС‹РјРё РєРѕР»РѕРЅРєР°РјРё
    sql.execute("PRAGMA table_info(ban_words)")
    columns = [col[1] for col in sql.fetchall()]

    # Р•СЃР»Рё РЅСѓР¶РЅС‹С… РєРѕР»РѕРЅРѕРє РЅРµС‚ вЂ” РїРµСЂРµСЃРѕР·РґР°С‘Рј С‚Р°Р±Р»РёС†Сѓ
    if "word" not in columns or "creator_id" not in columns or "time" not in columns:
        print("[INIT] РџРµСЂРµСЃРѕР·РґР°РЅРёРµ С‚Р°Р±Р»РёС†С‹ ban_words...")
        sql.execute("DROP TABLE IF EXISTS ban_words")
        sql.execute("""
        CREATE TABLE IF NOT EXISTS ban_words (
            word TEXT NOT NULL,
            creator_id INTEGER NOT NULL,
            time TEXT NOT NULL
        )
        """)
        database.commit()
        print("[INIT] РўР°Р±Р»РёС†Р° ban_words СѓСЃРїРµС€РЅРѕ РїРµСЂРµСЃРѕР·РґР°РЅР°.")
except Exception as e:
    print(f"[INIT] РћС€РёР±РєР° РїСЂРё РїСЂРѕРІРµСЂРєРµ С‚Р°Р±Р»РёС†С‹ ban_words: {e}")    

async def new_chat(chat_id: int, peer_id: int, owner_id: int, chat_type: str = "def"):
    # РџСЂРѕРІРµСЂСЏРµРј, РєР°РєРёРµ РєРѕР»РѕРЅРєРё СЂРµР°Р»СЊРЅРѕ РµСЃС‚СЊ
    sql.execute("PRAGMA table_info(chats)")
    columns = [col[1] for col in sql.fetchall()]

    # Р¤РѕСЂРјРёСЂСѓРµРј СЃРїРёСЃРѕРє РєРѕР»РѕРЅРѕРє Рё Р·РЅР°С‡РµРЅРёР№ РґР»СЏ INSERT
    insert_columns = ["chat_id", "peer_id", "owner_id"]
    insert_values = [chat_id, peer_id, owner_id]

    if "welcome_msg" in columns:
        insert_columns.append("welcome_msg")
        insert_values.append("Р”РѕР±СЂРѕ РїРѕР¶Р°Р»РѕРІР°С‚СЊ, СѓРІР°Р¶Р°РµРјС‹Р№ %i РїРѕР»СЊР·РѕРІР°С‚РµР»СЊ!")

    if "type" in columns:
        insert_columns.append("type")
        insert_values.append(chat_type)

    sql.execute(f"INSERT INTO chats ({', '.join(insert_columns)}) VALUES ({', '.join(['?']*len(insert_values))})", insert_values)

    # РЎРѕР·РґР°С‘Рј РѕСЃС‚Р°Р»СЊРЅС‹Рµ С‚Р°Р±Р»РёС†С‹ РґР»СЏ С‡Р°С‚Р°
    sql.execute(f"CREATE TABLE IF NOT EXISTS permissions_{chat_id} (user_id BIGINT, level BIGINT);")
    sql.execute(f"CREATE TABLE IF NOT EXISTS nicks_{chat_id} (user_id BIGINT, nick TEXT);")
    sql.execute(f"CREATE TABLE IF NOT EXISTS banwords_{chat_id} (banword TEXT);")
    sql.execute(f"CREATE TABLE IF NOT EXISTS warns_{chat_id} (user_id BIGINT, count BIGINT, moder BIGINT, reason TEXT, date BIGINT, date_string TEXT);")
    sql.execute(f"CREATE TABLE IF NOT EXISTS mutes_{chat_id} (user_id BIGINT, moder TEXT, reason TEXT, date BIGINT, date_string TEXT, time BIGINT);")
    sql.execute(f"CREATE TABLE IF NOT EXISTS bans_{chat_id} (user_id BIGINT, moder BIGINT, reason TEXT, date BIGINT, date_string TEXT);")
    sql.execute(f"CREATE TABLE IF NOT EXISTS messages_{chat_id} (user_id BIGINT, date BIGINT, date_string TEXT, message_id BIGINT, cmid BIGINT);")
    sql.execute(f"CREATE TABLE IF NOT EXISTS warnhistory_{chat_id} (user_id BIGINT, count BIGINT, moder BIGINT, reason TEXT, date BIGINT, date_string TEXT);")
    sql.execute(f"CREATE TABLE IF NOT EXISTS punishments_{chat_id} (user_id BIGINT, date TEXT);")

    database.commit()
      
async def get_role(user_id = int, chat_id = int):
    sql.execute(f"SELECT level FROM global_managers WHERE user_id = {user_id}")
    fetch = sql.fetchone()
    try:
        if fetch[0] == 2: return 0
        if fetch[0] == 3: return 0
        if fetch[0] == 4: return 0
        if fetch[0] == 5: return 0        
        if fetch[0] == 6: return 0
        if fetch[0] == 7: return 0
    except:
        sql.execute(f"SELECT owner_id FROM chats WHERE chat_id = {chat_id}")
        if sql.fetchall()[0][0] == user_id: return 7
        sql.execute(f"SELECT level FROM permissions_{chat_id} WHERE user_id = {user_id}")
        fetch = sql.fetchone()
        if fetch == None: return 0
        else: return fetch[0]

async def get_warns(user_id=int, chat_id=int):
    sql.execute(f"SELECT count FROM warns_{chat_id} WHERE user_id = {user_id}")
    fetch = sql.fetchone()
    if fetch == None: return 0
    else: return fetch[0]

# === РџСЂРѕРІРµСЂРєР°, Рє РєР°РєРѕР№ СЃРІСЏР·РєРµ РїСЂРёРЅР°РґР»РµР¶РёС‚ С‡Р°С‚ ===
async def get_gsync_chats(chat_id):
    sql.execute("SELECT owner_id, table_name FROM gsync_list")
    gsyncs = sql.fetchall()

    for owner_id, table_name in gsyncs:
        try:
            sql.execute(f"SELECT chat_id FROM {table_name} WHERE chat_id = ?", (chat_id,))
            if sql.fetchone():
                sql.execute(f"SELECT chat_id FROM {table_name}")
                chats = sql.fetchall()
                return [c[0] for c in chats]
        except:
            continue
    return None

# === РџРѕР»СѓС‡РµРЅРёРµ СЃРІСЏР·РєРё РїРѕ С‡Р°С‚Сѓ (РґР»СЏ info) ===
async def get_gsync_table(chat_id):
    sql.execute("SELECT owner_id, table_name FROM gsync_list")
    gsyncs = sql.fetchall()

    for owner_id, table_name in gsyncs:
        try:
            sql.execute(f"SELECT chat_id FROM {table_name} WHERE chat_id = ?", (chat_id,))
            if sql.fetchone():
                return {"owner": owner_id, "table": table_name}
        except:
            continue
    return None    

async def get_user_name(user_id: int, chat_id: int | None = None) -> str:
    # РЎРЅР°С‡Р°Р»Р° РїСЂРѕРІРµСЂСЏРµРј РЅРёРє РІ Р±Р°Р·Рµ, С‚РѕР»СЊРєРѕ РµСЃР»Рё chat_id Р·Р°РґР°РЅ
    if chat_id is not None:
        try:
            sql.execute(f"SELECT nick FROM nicks_{chat_id} WHERE user_id = ?", (user_id,))
            fetch = sql.fetchone()
            if fetch and fetch[0]:
                return fetch[0]
        except:
            pass  # РќР° СЃР»СѓС‡Р°Р№, РµСЃР»Рё С‚Р°Р±Р»РёС†С‹ РЅРµС‚

    # Р•СЃР»Рё РЅРёРєР° РЅРµС‚ РёР»Рё chat_id РЅРµ Р·Р°РґР°РЅ, РїС‹С‚Р°РµРјСЃСЏ РїРѕР»СѓС‡РёС‚СЊ РёРјСЏ Рё С„Р°РјРёР»РёСЋ С‡РµСЂРµР· API
    try:
        info = await bot.api.users.get(user_ids=user_id)
        if info and len(info) > 0:
            return f"{info[0].first_name} {info[0].last_name}"
    except:
        pass

    # Р•СЃР»Рё РЅРёС‡РµРіРѕ РЅРµ РїРѕР»СѓС‡РёР»РѕСЃСЊ, РІРѕР·РІСЂР°С‰Р°РµРј ID
    return str(user_id)
    
# Р¤СѓРЅРєС†РёСЏ РѕС‡РёСЃС‚РєРё РІР°СЂРЅРѕРІ
async def clear_all_warns(chat_id: int) -> int:
    # РџСЂРѕРІРµСЂСЏРµРј, РµСЃС‚СЊ Р»Рё Р·Р°РїРёСЃРё
    sql.execute(f"SELECT DISTINCT user_id FROM warns_{chat_id}")
    users = sql.fetchall()

    if not users:
        return 0  # РЅРёС‡РµРіРѕ РЅРµС‚

    count = len(users)

    # РЈРґР°Р»СЏРµРј РІСЃРµ РІР°СЂРЅС‹
    sql.execute(f"DELETE FROM warns_{chat_id}")
    database.commit()

    return count
    
async def is_nick(user_id=int, chat_id=int):
    sql.execute(f"SELECT nick FROM nicks_{chat_id} WHERE user_id = {user_id}")
    if sql.fetchone() == None: return False
    else: return True

async def setnick(user_id=int, chat_id=int, nick=str):
    sql.execute(f"SELECT nick FROM nicks_{chat_id} WHERE user_id = {user_id}")
    if sql.fetchone() == None:
        sql.execute(f"INSERT INTO nicks_{chat_id} VALUES (?, ?)", (user_id, nick))
        database.commit()
    else:
        sql.execute(f"UPDATE nicks_{chat_id} SET nick = ? WHERE user_id = ?", (nick, user_id))
        database.commit()

async def rnick(user_id=int, chat_id=int):
    sql.execute(f"DELETE FROM nicks_{chat_id} WHERE user_id = {user_id}")
    database.commit()

async def get_acc(chat_id=int, nick=str):
    sql.execute(f"SELECT user_id FROM nicks_{chat_id} WHERE nick = '{nick}'")
    fetch = sql.fetchone()
    if fetch == None: return False
    else: return fetch[0]

async def get_nick(user_id=int, chat_id=int):
    sql.execute(f"SELECT nick FROM nicks_{chat_id} WHERE user_id = {user_id}")
    fetch = sql.fetchone()
    if fetch == None: return False
    else: return fetch[0]

async def log_economy(user_id=None, target_id=None, amount=None, log=None):
    try:
        sql.execute(
            "INSERT INTO economy (user_id, target_id, amount, log) VALUES (?, ?, ?, ?)",
            (user_id, target_id, amount, log)
        )
        database.commit()
        print(f"[ECONOMY LOG] {user_id} -> {target_id} | {amount} | {log}")
    except Exception as e:
        print(f"[ECONOMY LOG ERROR] {e}")       
        
async def chats_log(user_id=None, target_id=None, role=None, log=None):
    try:
        sql.execute(
            "INSERT INTO logchats (user_id, target_id, role, log) VALUES (?, ?, ?, ?)",
            (user_id, target_id, role, log)
        )
        database.commit()
        print(f"[CHATS LOG] {user_id} -> {target_id} | {role} | {log}")
    except Exception as e:
        print(f"[CHATS LOG ERROR] {e}")       

async def nlist(chat_id: int, page: int):
    sql.execute(f"SELECT * FROM nicks_{chat_id}")
    fetch = sql.fetchall()
    if not fetch:
        return []

    nicks = []
    gi = 0
    with open("config.json", "r") as json_file:
        open_file = json.load(json_file)
    max_nicks = open_file.get('nicks_max', 20)

    start = (page - 1) * max_nicks
    end = page * max_nicks

    for i in fetch:
        if gi < start:
            gi += 1
            continue
        if gi >= end:
            break

        info = await bot.api.users.get(user_ids=i[0])
        if info and len(info) > 0:
            name = f"{info[0].first_name} {info[0].last_name}"
        else:
            name = "РћС€РёР±РєР°"

        nicks.append(f"{gi+1}. @id{i[0]} ({name}) -- {i[1]}")
        gi += 1

    return nicks 

async def nonick(chat_id=int, page=int):
    sql.execute(f"SELECT * FROM nicks_{chat_id}")
    fetch = sql.fetchall()
    nicks = []
    for i in fetch:
        nicks.append(i[0])

    gi = 0
    nonick = []
    with open("config.json", "r") as json_file:
        open_file = json.load(json_file)
    max_nonick = open_file['nonick_max']
    users = await bot.api.messages.get_conversation_members(peer_id=2000000000+chat_id)
    users = json.loads(users.json())
    for i in users["profiles"]:
        if not i['id'] in nicks:
            gi = gi + 1
            if page*max_nonick >= gi and page*max_nonick-max_nonick < gi:
                nonick.append(f"{gi}) @id{i['id']} ({i['first_name']} {i['last_name']})")

    return nonick

async def warn(chat_id=int, user_id=int, moder=int, reason=str):
    actualy_warns = await get_warns(user_id, chat_id)
    date = time.time()
    cd = str(datetime.now()).split('.')
    date_string = cd[0]
    sql.execute(f"INSERT INTO warnhistory_{chat_id} VALUES (?, {actualy_warns+1}, ?, ?, {date}, '{date_string}')",(user_id, moder, reason))
    database.commit()
    if actualy_warns < 1:
        sql.execute(f"INSERT INTO warns_{chat_id} VALUES (?, 1, ?, ?, {date}, '{date_string}')", (user_id, moder, reason))
        database.commit()
        return 1
    else:
        sql.execute(f"UPDATE warns_{chat_id} SET user_id = ?, count = ?, moder = ?, reason = ?, date = {date}, date_string = '{date_string}' WHERE user_id = {user_id}", (user_id, actualy_warns+1, moder, reason))
        database.commit()
        return actualy_warns+1

async def clear_warns(chat_id=int, user_id=int):
    sql.execute(f"DELETE FROM warns_{chat_id} WHERE user_id = {user_id}")
    database.commit()

async def unwarn(chat_id=int, user_id=int):
    warns = await get_warns(user_id, chat_id)
    if warns < 2: await clear_warns(chat_id, user_id)
    else:
        sql.execute(f"UPDATE warns_{chat_id} SET count = {warns-1} WHERE user_id = {user_id}")
        database.commit()

    return warns-1

async def gwarn(user_id=int, chat_id=int):
    sql.execute(f"SELECT * FROM warns_{chat_id} WHERE user_id = {user_id}")
    fetch = sql.fetchone()
    if fetch == None: return False
    else:
        return {
            'count': fetch[1],
            'moder': fetch[2],
            'reason': fetch[3],
            'time': fetch[5]
        }

async def warnhistory(user_id=int, chat_id=int):
    sql.execute(f"SELECT * FROM warnhistory_{chat_id} WHERE user_id = {user_id}")
    fetch = sql.fetchall()
    warnhistory_mass = []
    gi = 0
    if fetch == None: return False
    else:
        for i in fetch:
            gi = gi + 1
            warnhistory_mass.append(f"{gi}) @id{i[2]} (РњРѕРґРµСЂР°С‚РѕСЂ) | {i[3]} | {i[5]}")

    return warnhistory_mass

async def warnlist(chat_id=int):
    sql.execute(f"SELECT * FROM warns_{chat_id}")
    fetch = sql.fetchall()
    warns = []
    gi = 0
    for i in fetch:
        gi = gi + 1
        warns.append(f"{gi}) @id{i[0]} (РџРѕР»СЊР·РѕРІР°С‚РµР»СЊ) | {i[3]} | @id{i[2]} (РњРѕРґРµСЂР°С‚РѕСЂ) | {i[1]}/3 | {i[5]}")

    if fetch == None: return False
    return warns

async def staff(chat_id: int):
    # ==== Р›РѕРєР°Р»СЊРЅС‹Рµ РїСЂР°РІР° РёР· С‡Р°С‚Р° ====
    sql.execute(f"SELECT * FROM permissions_{chat_id}")
    fetch = sql.fetchall()
    moders = []
    stmoders = []
    admins = []
    stadmins = []
    zamspecadm = []
    specadm = []
    testers = []

    if fetch:
        for i in fetch:
            level = i[1]
            user_id = i[0]
            if level == 1: moders.append(f'@id{user_id} ({await get_user_name(user_id, chat_id)})')
            elif level == 2: stmoders.append(f'@id{user_id} ({await get_user_name(user_id, chat_id)})')
            elif level == 3: admins.append(f'@id{user_id} ({await get_user_name(user_id, chat_id)})')
            elif level == 4: stadmins.append(f'@id{user_id} ({await get_user_name(user_id, chat_id)})')
            elif level == 5: zamspecadm.append(f'@id{user_id} ({await get_user_name(user_id, chat_id)})')
            elif level == 6: specadm.append(f'@id{user_id} ({await get_user_name(user_id, chat_id)})')
            elif level == 12: testers.append(f'@id{user_id} ({await get_user_name(user_id, chat_id)})')

    # ==== Р“Р»РѕР±Р°Р»СЊРЅС‹Рµ РїСЂР°РІР° ====
    sql.execute("SELECT user_id, level FROM global_managers WHERE level IN (2,3,4,5,6,7)")
    global_fetch = sql.fetchall()
    zamruk = []
    oszamruk = []
    ruk = []
    dev = []
    zamglt = []
    glt = []

    for user_id, level in global_fetch:
        if level == 2: zamruk.append(f'@id{user_id} ({await get_user_name(user_id, None)})')
        elif level == 3: oszamruk.append(f'@id{user_id} ({await get_user_name(user_id, None)})')
        elif level == 4: ruk.append(f'@id{user_id} ({await get_user_name(user_id, None)})')
        elif level == 5: dev.append(f'@id{user_id} ({await get_user_name(user_id, None)})')
        elif level == 6: zamglt.append(f'@id{user_id} ({await get_user_name(user_id, None)})')
        elif level == 7: glt.append(f'@id{user_id} ({await get_user_name(user_id, None)})')

    return {
        'moders': moders,
        'stmoders': stmoders,
        'admins': admins,
        'stadmins': stadmins,
        'zamspecadm': zamspecadm,
        'specadm': specadm,
        'testers': testers,
        'zamruk': zamruk,
        'oszamruk': oszamruk,
        'ruk': ruk,
        'dev': dev,
        'zamglt': zamglt,
        'glt': glt
    }    

async def add_mute(user_id=int, chat_id=int, moder=int, reason=str, mute_time=int):
    cd = str(datetime.now()).split('.')
    date_string = cd[0]
    sql.execute(f"INSERT INTO mutes_{chat_id} VALUES (?, ?, ?, ?, ?, ?)", (user_id, moder, reason, time.time(), date_string, mute_time))
    database.commit()

async def get_mute(user_id=int, chat_id=int):
    await checkMute(chat_id, user_id)

    sql.execute(f"SELECT * FROM mutes_{chat_id} WHERE user_id = {user_id}")
    fetch = sql.fetchone()

    if fetch == None: return False
    else:
        return {
            'moder': fetch[1],
            'reason': fetch[2],
            'date': fetch[4],
            'time': fetch[5]
        }

async def unmute(user_id=int, chat_id=int):
    sql.execute(f"DELETE FROM mutes_{chat_id} WHERE user_id = {user_id}")
    database.commit()

async def mutelist(chat_id=int):
    sql.execute(f"SELECT * FROM mutes_{chat_id}")
    fetch = sql.fetchall()
    mutes = []
    if fetch==None: return False
    else:
        for i in fetch:
            if not await checkMute(chat_id, i[0]):
                do_time = datetime.fromisoformat(i[4]) + timedelta(minutes=i[5])
                mute_time = str(do_time).split('.')[0]
                try:
                    int(i[1])
                    mutes.append(f"@id{i[0]} (РџРѕР»СЊР·РѕРІР°С‚РµР»СЊ) | {i[2]} | @id{i[1]} (РјРѕРґРµСЂР°С‚РѕСЂ) | Р”Рѕ: {mute_time}")
                except: mutes.append(f"@id{i[0]} (РџРѕР»СЊР·РѕРІР°С‚РµР»СЊ) | {i[2]} | Р‘РѕС‚ | Р”Рѕ: {mute_time}")

    return mutes

async def checkMute(chat_id=int, user_id=int):
    sql.execute(f"SELECT * FROM mutes_{chat_id} WHERE user_id = {user_id}")
    fetch = sql.fetchone()
    if not fetch == None:
        do_time = datetime.fromisoformat(fetch[4]) + timedelta(minutes=fetch[5])
        if datetime.now() > do_time:
            sql.execute(f"DELETE FROM mutes_{chat_id} WHERE user_id = {user_id}")
            database.commit()
            return True
        else: return False
    return False

async def get_banwords(chat_id=int):
    sql.execute(f"SELECT * FROM banwords_{chat_id}")
    banwords = []
    fetch = sql.fetchall()
    for i in fetch:
        banwords.append(i[0])

    return banwords

async def clear(user_id=int, chat_id=int, group_id=int, peer_id=int):
    sql.execute(f"SELECT cmid FROM messages_{chat_id} WHERE user_id = {user_id}")
    fetch = sql.fetchall()
    cmids = []
    gi = 0
    for i in fetch:
        gi = gi + 1
        if gi <= 199:
            cmids.append(i[0])
    try: await bot.api.messages.delete(group_id=group_id, peer_id=peer_id, delete_for_all=True, cmids=cmids)
    except: pass

    sql.execute(f"DELETE FROM messages_{chat_id} WHERE user_id = {user_id}")
    database.commit()

async def new_message(user_id=int, message_id=int, cmid=int, chat_id=int):
    cd = str(datetime.now()).split('.')
    date_string = cd[0]
    sql.execute(f"INSERT INTO messages_{chat_id} VALUES (?, ?, ?, ?, ?)", (user_id, time.time(), date_string, message_id, cmid))
    database.commit()

async def add_money(user_id, amount):
    balances = load_data(BALANCES_FILE)
    bal = balances.get(str(user_id), get_balance(user_id))
    bal["wallet"] += amount
    balances[str(user_id)] = bal
    save_data(BALANCES_FILE, balances)
    await log_economy(user_id=user_id, target_id=None, amount=amount, log=f"РїРѕР»СѓС‡РёР»(+Р°) {amount}$ С‡РµСЂРµР· РїСЂРѕРјРѕРєРѕРґ")
    return True

async def give_vip(user_id, days):
    balances = load_data(BALANCES_FILE)
    bal = balances.get(str(user_id), get_balance(user_id))

    now = datetime.now()
    if bal.get("vip_until"):
        try:
            until = datetime.fromisoformat(bal["vip_until"])
            if until > now:
                bal["vip_until"] = (until + timedelta(days=days)).isoformat()
            else:
                bal["vip_until"] = (now + timedelta(days=days)).isoformat()
        except:
            bal["vip_until"] = (now + timedelta(days=days)).isoformat()
    else:
        bal["vip_until"] = (now + timedelta(days=days)).isoformat()

    balances[str(user_id)] = bal
    save_data(BALANCES_FILE, balances)
    await log_economy(user_id=user_id, target_id=None, amount=None, log=f"РїРѕР»СѓС‡РёР»(+Р°) VIP РЅР° {days} РґРЅРµР№ С‡РµСЂРµР· РїСЂРѕРјРѕРєРѕРґ")
    return True    

# --- Р¤СѓРЅРєС†РёСЏ РїСЂРѕРІРµСЂРєРё Р±Р°РЅР° С‚РѕР»СЊРєРѕ РІ РѕРґРЅРѕРј С‡Р°С‚Рµ ---
async def checkban(user_id: int, chat_id: int):
    try:
        sql.execute(f"SELECT * FROM bans_{chat_id} WHERE user_id = ?", (user_id,))
        fetch = sql.fetchone()
        if not fetch:
            return False
        return {
            'moder': fetch[1],
            'reason': fetch[2],
            'date': fetch[4]
        }
    except:
        return False  # РµСЃР»Рё С‚Р°Р±Р»РёС†С‹ РЅРµС‚   
        
async def checkban_all(user_id: int):
    sql.execute("SELECT chat_id, title FROM chats")
    chats_list = sql.fetchall()

    all_bans = []
    count_bans = 0

    i = 1
    for c in chats_list:
        chat_id_check, chat_title = c
        table_name = f"bans_{chat_id_check}"
        try:
            sql.execute(f"SELECT moderator_id, reason, date FROM {table_name} WHERE user_id = ?", (user_id,))
            user_bans = sql.fetchall()
            for ub in user_bans:
                mod_id, reason, date = ub
                all_bans.append(f"{i}) {chat_title} | @id{mod_id} (РњРѕРґРµСЂР°С‚РѕСЂ) | {reason} | {date} РњРЎРљ (UTC+3)")
                i += 1
                count_bans += 1
        except:
            continue  # РµСЃР»Рё С‚Р°Р±Р»РёС†С‹ РЅРµС‚, РїСЂРѕРїСѓСЃРєР°РµРј

    return count_bans, all_bans        

# --- Р¤СѓРЅРєС†РёСЏ РґРѕР±Р°РІР»РµРЅРёСЏ/РѕР±РЅРѕРІР»РµРЅРёСЏ Р±Р°РЅР° ---
async def ban(user_id: int, moder: int, chat_id: int, reason: str):
    # РџСЂРѕРІРµСЂСЏРµРј, РµСЃС‚СЊ Р»Рё СѓР¶Рµ Р±Р°РЅ
    sql.execute(f"SELECT user_id FROM bans_{chat_id} WHERE user_id = ?", (user_id,))
    fetch = sql.fetchone()

    # РўРµРєСѓС‰РµРµ РІСЂРµРјСЏ РІ С„РѕСЂРјР°С‚Рµ YYYY-MM-DD HH:MM:SS
    date_string = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

    if fetch is None:
        # Р”РѕР±Р°РІР»СЏРµРј РЅРѕРІРѕРіРѕ Р·Р°Р±Р°РЅРµРЅРЅРѕРіРѕ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ
        sql.execute(
            f"INSERT INTO bans_{chat_id} (user_id, moder, reason, date) VALUES (?, ?, ?, ?)",
            (user_id, moder, reason, date_string)
        )
        database.commit()
    else:
        # РћР±РЅРѕРІР»СЏРµРј РґР°РЅРЅС‹Рµ, РµСЃР»Рё РїРѕР»СЊР·РѕРІР°С‚РµР»СЊ СѓР¶Рµ РІ Р±Р°РЅРµ
        sql.execute(
            f"UPDATE bans_{chat_id} SET moder = ?, reason = ?, date = ? WHERE user_id = ?",
            (moder, reason, date_string, user_id)
        )
        database.commit()
        
async def unban(user_id=int, chat_id=int):
    sql.execute(f"DELETE FROM bans_{chat_id} WHERE user_id = {user_id}")
    database.commit()

async def globalrole(user_id: int, level: int):
    """
    Р’С‹РґР°С‘С‚ РёР»Рё РѕР±РЅРѕРІР»СЏРµС‚ РіР»РѕР±Р°Р»СЊРЅСѓСЋ СЂРѕР»СЊ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ РІ С‚Р°Р±Р»РёС†Рµ global_managers.

    level:
        0 - СѓРґР°Р»РµРЅРёРµ СЂРѕР»Рё
        8 - zamruk
        9 - oszamruk
        10 - ruk
        11 - dev
    """
    # РџСЂРѕРІРµСЂСЏРµРј РµСЃС‚СЊ Р»Рё Р·Р°РїРёСЃСЊ
    sql.execute("SELECT user_id FROM global_managers WHERE user_id = ?", (user_id,))
    fetch = sql.fetchone()

    if fetch is None:
        if level != 0:
            sql.execute("INSERT INTO global_managers (user_id, level) VALUES (?, ?)", (user_id, level))
    else:
        if level == 0:
            sql.execute("DELETE FROM global_managers WHERE user_id = ?", (user_id,))
        else:
            sql.execute("UPDATE global_managers SET level = ? WHERE user_id = ?", (level, user_id))

    database.commit()    

async def roleG(user_id=int, chat_id=int, role=int):
    sql.execute(f"SElECT user_id FROM permissions_{chat_id} WHERE user_id = {user_id}")
    fetch = sql.fetchone()
    if fetch == None:
        if role == 0: sql.execute(f"DELETE FROM permissions_{chat_id} WHERE user_id = {user_id}")
        else: sql.execute(f"INSERT INTO permissions_{chat_id} VALUES (?, ?)", (user_id, role))
    else:
        if role == 0: sql.execute(f"DELETE FROM permissions_{chat_id} WHERE user_id = {user_id}")
        else: sql.execute(f"UPDATE permissions_{chat_id} SET level = ? WHERE user_id = ?", (role, user_id))

    database.commit()

async def banlist(chat_id=int):
    sql.execute(f"SELECT * FROM bans_{chat_id}")
    fetch = sql.fetchall()
    banlist = []
    for i in fetch:
        banlist.append(f"@id{i[0]} (РџРѕР»СЊР·РѕРІР°С‚РµР»СЊ) | {i[2]} | @id{i[1]} (РњРѕРґРµСЂР°С‚РѕСЂ) | {i[4]}")

    return banlist

async def quiet(chat_id=int):
    sql.execute(f"SELECT silence FROM chats WHERE chat_id = {chat_id}")
    result = sql.fetchone()[0]
    if not await get_logic(result):
        sql.execute(f"UPDATE chats SET silence = 1 WHERE chat_id = {chat_id}")
        database.commit()
        return True
    else:
        sql.execute(f"UPDATE chats SET silence = 0 WHERE chat_id = {chat_id}")
        database.commit()
        return False

async def get_pull_chats(chat_id=int):
    sql.execute(f"SELECT owner_id, in_pull FROM chats WHERE chat_id = {chat_id}")
    fetch = sql.fetchone()
    if fetch == None: return False
    if not await get_logic(fetch[1]): return False
    sql.execute(f"SELECT chat_id FROM chats WHERE owner_id = ? AND in_pull = ?", (fetch[0], fetch[1]))
    result = []
    fetch2 = sql.fetchall()
    for i in fetch2:
        result.append(i[0])

    return result

async def get_pull_id(chat_id=int):
    sql.execute(f"SELECT in_pull FROM chats WHERE chat_id = {chat_id}")
    fetch = sql.fetchone()
    return fetch[0]

async def rnickall(chat_id=int):
    sql.execute(f"DELETE FROM nicks_{chat_id}")
    database.commit()    

async def banwords(slovo=str, delete=bool, chat_id=int):
    if delete:
        sql.execute(f"DELETE FROM banwords_{chat_id} WHERE banword = ?", (slovo, ))
        database.commit()
    else:
        sql.execute(f"SELECT * FROM banwords_{chat_id} WHERE banword = ?", (slovo, ))
        fetch = sql.fetchone()
        if fetch == None:
            sql.execute(f"INSERT INTO banwords_{chat_id} VALUES (?)", (slovo,))
            database.commit()

async def get_filter(chat_id=int):
    sql.execute(f"SELECT filter FROM chats WHERE chat_id = {chat_id}")
    fetch = sql.fetchone()
    return await get_logic(fetch[0])

async def set_filter(chat_id=int, value=int):
    sql.execute("UPDATE chats SET filter = ? WHERE chat_id = ?", (value, chat_id))
    database.commit()

async def get_antiflood(chat_id=int):
    sql.execute(f"SELECT antiflood FROM chats WHERE chat_id = {chat_id}")
    fetch = sql.fetchone()
    return await get_logic(fetch[0])

async def set_antiflood(chat_id=int, value=int):
    sql.execute("UPDATE chats SET antiflood = ? WHERE chat_id = ?", (value, chat_id))
    database.commit()

async def get_spam(user_id=int, chat_id=int):
    sql.execute(f"SELECT date_string FROM messages_{chat_id}  WHERE user_id = {user_id} ORDER BY date_string DESC LIMIT 3")
    fetch = sql.fetchall()
    list_messages = []
    for i in fetch:
        list_messages.append(datetime.fromisoformat(i[0]))
    try: list_messages = list_messages[:3]
    except: return False

    if list_messages[0] - list_messages[2] < timedelta(seconds=2): return True
    else: return False

async def set_welcome(chat_id=int, text=int):
    sql.execute(f"UPDATE chats SET welcome_text = ? WHERE chat_id = ?", (text, chat_id))
    database.commit()

async def get_welcome(chat_id=int):
    sql.execute("SELECT welcome_text FROM chats WHERE chat_id = ?", (chat_id, ))
    fetch = sql.fetchone()
    if str(fetch[0]).lower().strip() == "off": return False
    else: return str(fetch[0])

async def invite_kick(chat_id=int, change=None):
    sql.execute("SELECT invite_kick FROM chats WHERE chat_id = ?", (chat_id, ))
    fetch = sql.fetchone()
    if not change == None:
        if await get_logic(fetch[0]):
            sql.execute("UPDATE chats SET invite_kick = 0 WHERE chat_id = ?", (chat_id, ))
            database.commit()
            return False
        else:
            sql.execute("UPDATE chats SET invite_kick = 1 WHERE chat_id = ?", (chat_id,))
            database.commit()
            return True
    else:
        return await get_logic(fetch[0])

async def leave_kick(chat_id=int, change=None):
    sql.execute("SELECT leave_kick FROM chats WHERE chat_id = ?", (chat_id,))
    fetch = sql.fetchone()
    if fetch == None: return False
    if change == None: return await get_logic(fetch[0])
    if await get_logic(fetch[0]):
        sql.execute("UPDATE chats SET leave_kick = 0 WHERE chat_id = ?", (chat_id,))
        database.commit()
        return False
    else:
        sql.execute("UPDATE chats SET leave_kick = 1 WHERE chat_id = ?", (chat_id,))
        database.commit()
        return True

async def get_server_chats(chat_id):
    """
    РћРїСЂРµРґРµР»СЏРµС‚, Рє РєР°РєРѕРјСѓ СЃРµСЂРІРµСЂСѓ РїСЂРёРЅР°РґР»РµР¶РёС‚ С‡Р°С‚, Рё РІРѕР·РІСЂР°С‰Р°РµС‚ СЃРїРёСЃРѕРє РІСЃРµС… chat_id РёР· СЌС‚РѕРіРѕ СЃРµСЂРІРµСЂР°.
    """
    sql.execute("SELECT owner_id, server_number, table_name FROM servers_list")
    servers = sql.fetchall()

    for owner_id, server_number, table_name in servers:
        try:
            sql.execute(f"SELECT chat_id FROM {table_name} WHERE chat_id = ?", (chat_id,))
            if sql.fetchone():
                sql.execute(f"SELECT chat_id FROM {table_name}")
                chats = sql.fetchall()
                return [c[0] for c in chats]
        except:
            continue
    return None    

async def get_current_server(chat_id):
    """
    Р’РѕР·РІСЂР°С‰Р°РµС‚ РЅРѕРјРµСЂ СЃРµСЂРІРµСЂР°, Рє РєРѕС‚РѕСЂРѕРјСѓ РїСЂРёРІСЏР·Р°РЅ РґР°РЅРЅС‹Р№ chat_id, РёР»Рё None, РµСЃР»Рё РЅРµ РїСЂРёРІСЏР·Р°РЅ.
    """
    sql.execute("SELECT owner_id, server_number, table_name FROM servers_list")
    servers = sql.fetchall()

    for owner_id, server_number, table_name in servers:
        try:
            sql.execute(f"SELECT chat_id FROM {table_name} WHERE chat_id = ?", (chat_id,))
            if sql.fetchone():
                return server_number  # РІРѕР·РІСЂР°С‰Р°РµРј С‚РѕР»СЊРєРѕ РЅРѕРјРµСЂ СЃРµСЂРІРµСЂР°
        except Exception as e:
            print(f"[get_current_server] РћС€РёР±РєР° РїСЂРё РїСЂРѕРІРµСЂРєРµ С‚Р°Р±Р»РёС†С‹ {table_name}: {e}")
            continue
    return None    

async def message_stats(user_id=int, chat_id=int):
    try:
        sql.execute(f"SELECT date_string FROM messages_{chat_id} WHERE user_id = ?", (user_id, ))
        fetch_all = sql.fetchall()
        sql.execute(f"SELECT date_string FROM messages_{chat_id} WHERE user_id = ? ORDER BY date_string DESC LIMIT 1", (user_id,))
        fetch_last = sql.fetchone()
        last = fetch_last[0]
        return {
            'count': len(fetch_all),
            'last': last
        }
    except: return {
        'count': 0,
        'last': 0
    }

async def set_pull(chat_id=int, value=int):
    sql.execute(f"UPDATE chats SET in_pull = ? WHERE chat_id = ?", (value, chat_id))
    database.commit()

async def get_all_peerids():
    sql.execute("SELECT peer_id FROM chats")
    fetch = sql.fetchall()
    peer_ids = []
    for i in fetch:
        peer_ids.append(i[0])

    return peer_ids

async def add_punishment(chat_id=int, user_id=int):
    cd = str(datetime.now()).split('.')
    date_string = cd[0]
    sql.execute(f"INSERT INTO punishments_{chat_id} VALUES (?, ?)", (user_id, date_string))
    database.commit()

async def get_sliv(user_id=int, chat_id=int):
    sql.execute(f"SELECT date FROM punishments_{chat_id}  WHERE user_id = {user_id} ORDER BY date DESC LIMIT 3")
    fetch = sql.fetchall()
    list_messages = []
    for i in fetch:
        list_messages.append(datetime.fromisoformat(i[0]))
    try: list_messages = list_messages[:3]
    except: return False

    if list_messages[0] - list_messages[2] < timedelta(seconds=6): return True
    else: return False

async def get_ServerChat(chat_id: int):
    try:
        # РџРѕР»СѓС‡Р°РµРј id СЃРµСЂРІРµСЂР°, Рє РєРѕС‚РѕСЂРѕРјСѓ РїСЂРёРІСЏР·Р°РЅ chat_id
        sql.execute("SELECT server FROM server_links WHERE chat_id = ?", (chat_id,))
        result = sql.fetchone()
        if not result:
            return None

        server_id = result[0]

        # РџРѕР»СѓС‡Р°РµРј РІСЃРµ chat_id, РїСЂРёРІСЏР·Р°РЅРЅС‹Рµ Рє СЌС‚РѕРјСѓ СЃРµСЂРІРµСЂСѓ
        sql.execute("SELECT chat_id FROM server_links WHERE server = ?", (server_id,))
        chats = [row[0] for row in sql.fetchall()]

        return {
            "server": server_id,
            "chats": chats
        }
    except Exception as e:
        print(f"[SERVER] РћС€РёР±РєР° РїСЂРё РїРѕР»СѓС‡РµРЅРёРё СЃРµСЂРІРµСЂР°: {e}")
        return None        

async def staff_zov(chat_id=int):
    sql.execute(f"SElECT user_id FROM permissions_{chat_id}")
    fetch = sql.fetchall()
    staff_zov_str = []
    for i in fetch:
        staff_zov_str.append(f"@id{i[0]} (вљњпёЏ)")

    return ''.join(staff_zov_str)

async def delete_message(group_id=int, peer_id=int, cmid=int):
    try: await bot.api.messages.delete(group_id=group_id, peer_id=peer_id, delete_for_all=True, cmids=cmid)
    except: pass

# РџРѕР»СѓС‡РёС‚СЊ С‚РµРєСѓС‰РµРµ СЃРѕСЃС‚РѕСЏРЅРёРµ Р°РЅС‚РёСЃР»РёРІР° (0 вЂ” РІС‹РєР», 1 вЂ” РІРєР»)
async def get_antisliv(chat_id):
    sql.execute("SELECT mode FROM antisliv WHERE chat_id = ?", (chat_id,))
    data = sql.fetchone()
    return data[0] if data else 0

# РЈСЃС‚Р°РЅРѕРІРёС‚СЊ РЅРѕРІРѕРµ СЃРѕСЃС‚РѕСЏРЅРёРµ Р°РЅС‚РёСЃР»РёРІР°
async def antisliv_mode(chat_id, mode):
    sql.execute("INSERT OR REPLACE INTO antisliv (chat_id, mode) VALUES (?, ?)", (chat_id, mode))
    database.commit()

async def set_onwer(user=int, chat=int):
    sql.execute("UPDATE chats SET owner_id = ? WHERE chat_id = ?", (user, chat))
    database.commit()

async def equals_roles(user_id_sender: int, user_id_two: int, chat_id: int, message):
    sender_role = await get_role(user_id_sender, chat_id)
    target_role = await get_role(user_id_two, chat_id)

    # РџСЂРѕРІРµСЂРєР°: РµСЃР»Рё РїРѕР»СЊР·РѕРІР°С‚РµР»СЊ РїС‹С‚Р°РµС‚СЃСЏ РїСЂРёРјРµРЅРёС‚СЊ РєРѕРјР°РЅРґСѓ РЅР° СѓС‡Р°СЃС‚РЅРёРєР° СЃ Р±РѕР»РµРµ РІС‹СЃРѕРєРёРј СЂР°РЅРіРѕРј
    if sender_role < 7 and sender_role < target_role:
        await roleG(user_id_sender, chat_id, 0)
        await message.reply(
            f"вќ—пёЏ РЈСЂРѕРІРµРЅСЊ РїСЂР°РІ @id{user_id_sender} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ) Р±С‹Р» СЃРЅСЏС‚ "
            f"РёР·-Р·Р° РїРѕРїС‹С‚РєРё РёСЃРїРѕР»СЊР·РѕРІР°РЅРёСЏ РєРѕРјР°РЅРґС‹ РЅР° СѓС‡Р°СЃС‚РЅРёРєР° СЃ Р±РѕР»РµРµ РІС‹СЃРѕРєРёРј СЂР°РЅРіРѕРј!"
        )
        return 0

    # Р•СЃР»Рё РІСЃС‘ РЅРѕСЂРјР°Р»СЊРЅРѕ вЂ” РІРѕР·РІСЂР°С‰Р°РµРј СЃС‚Р°РЅРґР°СЂС‚РЅС‹Рµ Р·РЅР°С‡РµРЅРёСЏ
    if sender_role > target_role:
        return 2
    elif sender_role == target_role:
        return 1
    else:
        return 0       
  
chat_types = {
    "def": "РѕР±С‰РёРµ Р±РµСЃРµРґС‹",
    "ext": "СЂР°СЃС€РёСЂРµРЅРЅР°СЏ Р±РµСЃРµРґР°",
    "pl": "Р±РµСЃРµРґР° РёРіСЂРѕРєРѕРІ",
    "hel": "Р±РµСЃРµРґР° С…РµР»РїРµСЂРѕРІ",
    "ld": "Р±РµСЃРµРґР° Р»РёРґРµСЂРѕРІ",
    "adm": "Р±РµСЃРµРґР° Р°РґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂРѕРІ",
    "mod": "Р±РµСЃРµРґР° РјРѕРґРµСЂР°С‚РѕСЂРѕРІ",
    "tex": "Р±РµСЃРµРґР° С‚РµС…РѕРІ",
    "test": "Р±РµСЃРµРґР° С‚РµСЃС‚РµСЂРѕРІ",
    "med": "Р±РµСЃРµРґР° РјРµРґРёР°-РїР°СЂС‚РЅС‘СЂРѕРІ",
    "ruk": "Р±РµСЃРµРґР° СЂСѓРєРѕРІРѕРґСЃС‚РІР°",
    "users": "Р±РµСЃРµРґР° РїРѕР»СЊР·РѕРІР°С‚РµР»РµР№"
}

@bot.on.chat_message(rules.ChatActionRule("chat_kick_user"))
async def user_leave(message: Message) -> None:
    user_id = message.from_id
    chat_id = message.chat_id
    if not await check_chat(chat_id): return True
    if not message.action.member_id == message.from_id: return True
    if await leave_kick(chat_id):
        try: await bot.api.messages.remove_chat_user(chat_id, user_id)
        except: pass
        await message.answer(f"@id{user_id} ({await get_user_name(user_id, chat_id)}), РІС‹С€РµР»(-Р»Р°) РёР· Р±РµСЃРµРґС‹", disable_mentions=1)
    else:
        keyboard = (
            Keyboard(inline=True)
            .add(Callback("РСЃРєР»СЋС‡РёС‚СЊ", {"command": "kick", "user": user_id, "chatId": chat_id}), color=KeyboardButtonColor.NEGATIVE)
        )
        await message.answer(f"@id{user_id} ({await get_user_name(user_id, chat_id)}), РІС‹С€РµР»(-Р»Р°) РёР· Р±РµСЃРµРґС‹", disable_mentions=1, keyboard=keyboard)

@bot.on.chat_message(rules.ChatActionRule("chat_invite_user"))
async def user_joined(message: Message) -> None:
    invited_user = message.action.member_id
    user_id = message.from_id
    chat_id = message.chat_id

    # РµСЃР»Рё С‡Р°С‚ РЅРµ РІ Р±Р°Р·Рµ вЂ” РёРіРЅРѕСЂРёСЂСѓРµРј
    if not await check_chat(chat_id):
        return True
        
    async def _safe_first_name(uid: int) -> str:
        try:
            resp = await bot.api.users.get(uid)
            if resp and len(resp) > 0:
                return resp[0].first_name
        except Exception:
            pass
        return str(uid)

    try:
        # Р‘РѕС‚ РґРѕР±Р°РІР»РµРЅ
        if invited_user == -232890128:
            await message.answer(
                "Р‘РѕС‚ РґРѕР±Р°РІР»РµРЅ РІ Р±РµСЃРµРґСѓ, РІС‹РґР°Р№С‚Рµ РјРЅРµ Р°РґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂР°, Р° Р·Р°С‚РµРј РІРІРµРґРёС‚Рµ /sync РґР»СЏ СЃРёРЅС…СЂРѕРЅРёР·Р°С†РёРё СЃ Р±Р°Р·РѕР№ РґР°РЅРЅС‹С…!\n\n"
                "РўР°РєР¶Рµ СЃ РїРѕРјРѕС‰СЊСЋ /type Р’С‹ РјРѕР¶РµС‚Рµ РІС‹Р±СЂР°С‚СЊ С‚РёРї Р±РµСЃРµРґС‹!"
            )
            return True
        
        # ==== рџ”№ РџСЂРѕРІРµСЂРєР° Р·Р°С‰РёС‚С‹ РѕС‚ СЃС‚РѕСЂРѕРЅРЅРёС… СЃРѕРѕР±С‰РµСЃС‚РІ ====
        sql.execute("SELECT * FROM protection WHERE chat_id = ? AND mode = 1", (chat_id,))
        prot = sql.fetchone()
        if prot:
            if invited_user < 0:  # СЃРѕРѕР±С‰РµСЃС‚РІРѕ
                try:
                    await bot.api.messages.remove_chat_user(chat_id, invited_user)
                except:
                    pass
                await message.answer(
                    f"@id{user_id} ({await get_user_name(user_id, chat_id)}) РґРѕР±Р°РІРёР» СЃРѕРѕР±С‰РµСЃС‚РІРѕ, СЌС‚Рѕ Р·Р°РїСЂРµС‰РµРЅРѕ РІ РЅР°СЃС‚СЂРѕР№РєР°С… РґР°РЅРЅРѕРіРѕ С‡Р°С‚Р°!\n\n"
                    f"Р’С‹РєР»СЋС‡РёС‚СЊ РјРѕР¶РЅРѕ: В«/Р·Р°С‰РёС‚Р°В»",
                    disable_mentions=1
                )
                return True

        # ==== рџ”№ РџСЂРѕРІРµСЂРєР° РіР»РѕР±Р°Р»СЊРЅРѕРіРѕ Р±Р°РЅР° ====
        sql.execute("SELECT * FROM gbanlist WHERE user_id = ?", (invited_user,))
        globalban = sql.fetchone()
        if globalban:
            try:
                await bot.api.messages.remove_chat_user(chat_id, invited_user)
            except:
                pass

            first = await _safe_first_name(invited_user)
            await message.answer(
                f"@id{invited_user} ({await get_user_name(invited_user, chat_id)}) РёРјРµРµС‚ РіР»РѕР±Р°Р»СЊРЅСѓСЋ Р±Р»РѕРєРёСЂРѕРІРєСѓ!\n\n"
                f"@id{globalban[1]} (РњРѕРґРµСЂР°С‚РѕСЂ) | {globalban[2]} | {globalban[3]}",
                disable_mentions=1
            )
            return True
            
        # ==== рџ”№ РџСЂРѕРІРµСЂРєР° РіР»РѕР±Р°Р»СЊРЅРѕРіРѕ Р±Р°РЅР° ====
        sql.execute("SELECT * FROM globalban WHERE user_id = ?", (invited_user,))
        globalban = sql.fetchone()
        if globalban:
            try:
                await bot.api.messages.remove_chat_user(chat_id, invited_user)
            except:
                pass

            first = await _safe_first_name(invited_user)
            await message.answer(
                f"@id{invited_user} ({await get_user_name(invited_user, chat_id)}), РёРјРµРµС‚ РѕР±С‰СѓСЋ Р±Р»РѕРєРёСЂРѕРІРєСѓ РІРѕ РІСЃРµС… Р±РµСЃРµРґР°С…!\n\n"
                f"@id{globalban[1]} (РњРѕРґРµСЂР°С‚РѕСЂ) | {globalban[2]} | {globalban[3]}",
                disable_mentions=1
            )
            return True            

        # ==== РџРѕР»СЊР·РѕРІР°С‚РµР»СЊ РІРѕС€С‘Р» СЃР°Рј ====
        if user_id == invited_user:
            checkban_str = await checkban(invited_user, chat_id)
            if checkban_str:
                try:
                    await bot.api.messages.remove_chat_user(chat_id, invited_user)
                except:
                    pass

                first = await _safe_first_name(invited_user)
                keyboard = (
                    Keyboard(inline=True)
                    .add(Callback("РЎРЅСЏС‚СЊ Р±Р°РЅ", payload=""), color=KeyboardButtonColor.POSITIVE)
                )
                await message.answer(
                    f"@id{invited_user} ({await get_user_name(invited_user, chat_id)}) Р·Р°Р±Р»РѕРєРёСЂРѕРІР°РЅ(-Р°) РІ СЌС‚РѕР№ Р±РµСЃРµРґРµ!\n\n"
                    f"РРЅС„РѕСЂРјР°С†РёСЏ Рѕ Р±Р»РѕРєРёСЂРѕРІРєРµ:\n@id{checkban_str['moder']} (РњРѕРґРµСЂР°С‚РѕСЂ) | {checkban_str['reason']} | {checkban_str['date']}",
                    disable_mentions=1,
                    keyboard=keyboard
                )
                return True

            welcome = await get_welcome(chat_id)
            if welcome:
                first = await _safe_first_name(invited_user)
                inviter_first = await _safe_first_name(user_id)
                welcome = welcome.replace('%u', f'@id{invited_user}')
                welcome = welcome.replace('%n', f'@id{invited_user} ({await get_user_name(invited_user, chat_id)})')
                welcome = welcome.replace('%i', f'@id{user_id}')
                welcome = welcome.replace('%p', f'@id{user_id} ({await get_user_name(user_id, chat_id)})')
                await message.answer(welcome)
                return True

        # ==== РљС‚Рѕ-С‚Рѕ РїСЂРёРіР»Р°СЃРёР» РґСЂСѓРіРѕРіРѕ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ ====
        if await get_role(user_id, chat_id) < 1 and await invite_kick(chat_id):
            try:
                await bot.api.messages.remove_chat_user(chat_id, invited_user)
            except:
                pass
            return True

        checkban_str = await checkban(invited_user, chat_id)
        if checkban_str:
            try:
                await bot.api.messages.remove_chat_user(chat_id, invited_user)
            except:
                pass

            first = await _safe_first_name(invited_user)
            keyboard = (
                Keyboard(inline=True)
                .add(Callback("РЎРЅСЏС‚СЊ Р±Р°РЅ", payload=""), color=KeyboardButtonColor.POSITIVE)
            )
            await message.answer(
                f"@id{invited_user} ({await get_user_name(invited_user, chat_id)}) Р·Р°Р±Р»РѕРєРёСЂРѕРІР°РЅ(-Р°) РІ СЌС‚РѕР№ Р±РµСЃРµРґРµ!\n\n"
                f"РРЅС„РѕСЂРјР°С†РёСЏ Рѕ Р±Р»РѕРєРёСЂРѕРІРєРµ:\n@id{checkban_str['moder']} (РњРѕРґРµСЂР°С‚РѕСЂ) | {checkban_str['reason']} | {checkban_str['date']}",
                disable_mentions=1,
                keyboard=keyboard
            )
            return True

        welcome = await get_welcome(chat_id)
        if welcome:
            first = await _safe_first_name(invited_user)
            inviter_first = await _safe_first_name(user_id)
            welcome = welcome.replace('%u', f'@id{invited_user}')
            welcome = welcome.replace('%n', f'@id{invited_user} ({await get_user_name(invited_user, chat_id)})')
            welcome = welcome.replace('%i', f'@id{user_id}')
            welcome = welcome.replace('%p', f'@id{user_id} ({await get_user_name(user_id, chat_id)})')
            await message.answer(welcome)
            return True

    except Exception as e:
        print(f"[user_joined] РћС€РёР±РєР°: {e}")
        return True        

@bot.on.raw_event(GroupEventType.MESSAGE_EVENT, dataclass=GroupTypes.MessageEvent)
async def handlers(message: GroupTypes.MessageEvent):
    payload = message.object.payload or {}
    command = str(payload.get("command", "")).lower()
    user_id = message.object.user_id
    chat_id = payload.get("chatId")

    # Р›РѕРі РґР»СЏ РєР°Р¶РґРѕР№ РєРЅРѕРїРєРё
    log_cmd = payload.get("log") or "РЅРµС‚ Р»РѕРіР°"
    print(f"{user_id} РёСЃРїРѕР»СЊР·РѕРІР°Р» РєРЅРѕРїРєСѓ {command}. Р’Рљ РІС‹РґР°Р»Рѕ: {log_cmd}")
    if command == "nicksminus":
        if await get_role(user_id, chat_id) < 1:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!"})
            )
            return True
        page = payload.get("page")
        if page < 2:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "Р­С‚Рѕ РїРµСЂРІР°СЏ СЃС‚СЂР°РЅРёС†Р°!"})
            )
            return True

        keyboard = (
            Keyboard(inline=True)
            .add(Callback("вЏЄ", {"command": "nicksMinus", "page": page - 1, "chatId": chat_id}),
                 color=KeyboardButtonColor.NEGATIVE)
            .add(Callback("Р‘РµР· РЅРёРєРѕРІ", {"command": "nonicks", "chatId": chat_id}), color=KeyboardButtonColor.PRIMARY)
            .add(Callback("вЏ©", {"command": "nicksPlus", "page": page - 1, "chatId": chat_id}),
                 color=KeyboardButtonColor.POSITIVE)
        )
        await delete_message(message.group_id, message.object.peer_id, message.object.conversation_message_id)
        nicks_str = '\n'.join(await nlist(chat_id, page-1))
        await bot.api.messages.send(peer_id=2000000000 + chat_id, message=f"РџРѕР»СЊР·РѕРІР°С‚РµР»Рё СЃ РЅРёРєРѕРј [{page-1} СЃС‚СЂР°РЅРёС†Р°]:\n{nicks_str}\n\nРџРѕР»СЊР·РѕРІР°С‚РµР»Рё Р±РµР· РЅРёРєРѕРІ: В«/nonickВ»", disable_mentions=1, random_id=0, keyboard=keyboard)

    if command == "nicksplus":
        if await get_role(user_id, chat_id) < 1:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!"})
            )
            return True

        page = payload.get("page")

        nicks = await nlist(chat_id, page + 1)
        if len(nicks) < 1:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "Р­С‚Рѕ РїРѕСЃР»РµРґРЅСЏСЏ СЃС‚СЂР°РЅРёС†Р°!"})
            )
            return True

        keyboard = (
            Keyboard(inline=True)
            .add(Callback("вЏЄ", {"command": "nicksMinus", "page": page+1, "chatId": chat_id}),
                 color=KeyboardButtonColor.NEGATIVE)
            .add(Callback("Р‘РµР· РЅРёРєРѕРІ", {"command": "nonicks", "chatId": chat_id}), color=KeyboardButtonColor.PRIMARY)
            .add(Callback("вЏ©", {"command": "nicksPlus", "page": page+1, "chatId": chat_id}),
                 color=KeyboardButtonColor.POSITIVE)
        )
        await delete_message(message.group_id, message.object.peer_id, message.object.conversation_message_id)
        nicks_str = '\n'.join(nicks)
        await bot.api.messages.send(peer_id=2000000000 + chat_id,message=f"РџРѕР»СЊР·РѕРІР°С‚РµР»Рё СЃ РЅРёРєРѕРј [{page + 1} СЃС‚СЂР°РЅРёС†Р°]:\n{nicks_str}\n\nРџРѕР»СЊР·РѕРІР°С‚РµР»Рё Р±РµР· РЅРёРєРѕРІ: В«/nonickВ»",disable_mentions=1, random_id=0, keyboard=keyboard)

    if command == "chatsminus":
        if await get_role(user_id, chat_id) < 10:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!"})
            )
            return True

        page = payload.get("page")
        if page < 2:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "Р­С‚Рѕ РїРµСЂРІР°СЏ СЃС‚СЂР°РЅРёС†Р°!"})
            )
            return True

        sql.execute("SELECT chat_id, owner_id FROM chats ORDER BY chat_id ASC")
        all_rows = sql.fetchall()
        total = len(all_rows)
        per_page = 5
        max_page = (total + per_page - 1) // per_page

        async def get_chats_page(page: int):
            start = (page - 1) * per_page
            end = start + per_page
            selected = all_rows[start:end]
            formatted = []
            for idx, (chat_id_row, owner_id) in enumerate(selected, start=start + 1):
                rel_id = 2000000000 + chat_id_row
                try:
                    resp = await bot.api.messages.get_conversations_by_id(peer_ids=rel_id)
                    if resp.items:
                        chat_title = resp.items[0].chat_settings.title or "Р‘РµР· РЅР°Р·РІР°РЅРёСЏ"
                    else:
                        chat_title = "Р‘РµР· РЅР°Р·РІР°РЅРёСЏ"
                except:
                    chat_title = "РћС€РёР±РєР° РїРѕР»СѓС‡РµРЅРёСЏ РЅР°Р·РІР°РЅРёСЏ"

                try:
                    link_resp = await bot.api.messages.get_invite_link(peer_id=rel_id, reset=0)
                    chat_link = link_resp.link
                except:
                    chat_link = "РќРµ СѓРґР°Р»РѕСЃСЊ РїРѕР»СѓС‡РёС‚СЊ"

                try:
                    owner_info = await bot.api.users.get(user_ids=owner_id)
                    owner_name = f"{owner_info[0].first_name} {owner_info[0].last_name}"
                except:
                    owner_name = "РќРµ СѓРґР°Р»РѕСЃСЊ РїРѕР»СѓС‡РёС‚СЊ РёРјСЏ"

                formatted.append(
                    f"{idx}. рџ’¬ Р‘РµСЃРµРґР° в„–{chat_id_row}\n"
                    f"рџ“› РќР°Р·РІР°РЅРёРµ: {chat_title}\n"
                    f"рџ‘‘ Р’Р»Р°РґРµР»РµС†: @id{owner_id} ({owner_name})\n"
                    f"рџ”— РЎСЃС‹Р»РєР°: {chat_link}\n"
                )
            return formatted

        new_page = page - 1
        chats = await get_chats_page(new_page)
        chats_text = "\n".join(chats)
        if not chats_text:
            chats_text = "Р‘РµСЃРµРґС‹ РѕС‚СЃСѓС‚СЃС‚РІСѓСЋС‚!"

        keyboard = (
            Keyboard(inline=True)
            .add(Callback("вЏЄ", {"command": "chatsMinus", "page": new_page}), color=KeyboardButtonColor.NEGATIVE)
            .add(Callback("вЏ©", {"command": "chatsPlus", "page": new_page}), color=KeyboardButtonColor.POSITIVE)
        )

        await delete_message(message.group_id, message.object.peer_id, message.object.conversation_message_id)
        await bot.api.messages.send(
            peer_id=message.object.peer_id,
            message=f"РЎРїРёСЃРѕРє Р·Р°СЂРµРіРёСЃС‚СЂРёСЂРѕРІР°РЅРЅС‹С… Р±РµСЃРµРґ [{new_page} СЃС‚СЂР°РЅРёС†Р° РёР· {max_page}]:\n\n{chats_text}\nрџ“Љ Р’СЃРµРіРѕ Р±РµСЃРµРґ: {total}",
            disable_mentions=1, random_id=0, keyboard=keyboard
        )
        return True


    if command == "chatsplus":
        if await get_role(user_id, chat_id) < 10:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!"})
            )
            return True

        page = payload.get("page")

        sql.execute("SELECT chat_id, owner_id FROM chats ORDER BY chat_id ASC")
        all_rows = sql.fetchall()
        total = len(all_rows)
        per_page = 5
        max_page = (total + per_page - 1) // per_page

        async def get_chats_page(page: int):
            start = (page - 1) * per_page
            end = start + per_page
            selected = all_rows[start:end]
            formatted = []
            for idx, (chat_id_row, owner_id) in enumerate(selected, start=start + 1):
                rel_id = 2000000000 + chat_id_row
                try:
                    resp = await bot.api.messages.get_conversations_by_id(peer_ids=rel_id)
                    if resp.items:
                        chat_title = resp.items[0].chat_settings.title or "Р‘РµР· РЅР°Р·РІР°РЅРёСЏ"
                    else:
                        chat_title = "Р‘РµР· РЅР°Р·РІР°РЅРёСЏ"
                except:
                    chat_title = "РћС€РёР±РєР° РїРѕР»СѓС‡РµРЅРёСЏ РЅР°Р·РІР°РЅРёСЏ"

                try:
                    link_resp = await bot.api.messages.get_invite_link(peer_id=rel_id, reset=0)
                    chat_link = link_resp.link
                except:
                    chat_link = "РќРµ СѓРґР°Р»РѕСЃСЊ РїРѕР»СѓС‡РёС‚СЊ"

                try:
                    owner_info = await bot.api.users.get(user_ids=owner_id)
                    owner_name = f"{owner_info[0].first_name} {owner_info[0].last_name}"
                except:
                    owner_name = "РќРµ СѓРґР°Р»РѕСЃСЊ РїРѕР»СѓС‡РёС‚СЊ РёРјСЏ"

                formatted.append(
                    f"{idx}. рџ’¬ Р‘РµСЃРµРґР° в„–{chat_id_row}\n"
                    f"рџ“› РќР°Р·РІР°РЅРёРµ: {chat_title}\n"
                    f"рџ‘‘ Р’Р»Р°РґРµР»РµС†: @id{owner_id} ({owner_name})\n"
                    f"рџ”— РЎСЃС‹Р»РєР°: {chat_link}\n"
                )
            return formatted

        new_page = page + 1
        chats = await get_chats_page(new_page)
        if len(chats) < 1:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "Р­С‚Рѕ РїРѕСЃР»РµРґРЅСЏСЏ СЃС‚СЂР°РЅРёС†Р°!"})
            )
            return True

        chats_text = "\n".join(chats)
        keyboard = (
            Keyboard(inline=True)
            .add(Callback("вЏЄ", {"command": "chatsMinus", "page": new_page}), color=KeyboardButtonColor.NEGATIVE)
            .add(Callback("вЏ©", {"command": "chatsPlus", "page": new_page}), color=KeyboardButtonColor.POSITIVE)
        )

        await delete_message(message.group_id, message.object.peer_id, message.object.conversation_message_id)
        await bot.api.messages.send(
            peer_id=message.object.peer_id,
            message=f"РЎРїРёСЃРѕРє Р·Р°СЂРµРіРёСЃС‚СЂРёСЂРѕРІР°РЅРЅС‹С… Р±РµСЃРµРґ [{new_page} СЃС‚СЂР°РЅРёС†Р° РёР· {max_page}]:\n\n{chats_text}\nрџ“Љ Р’СЃРµРіРѕ Р±РµСЃРµРґ: {total}",
            disable_mentions=1, random_id=0, keyboard=keyboard
        )
        return True
        
    if command == "nonicks":
        if await get_role(user_id, chat_id) < 1:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!"})
            )
            return True

        nonicks = await nonick(chat_id, 1)
        nonick_list = '\n'.join(nonicks)
        if nonick_list == "": nonick_list = "РџРѕР»СЊР·РѕРІР°С‚РµР»Рё Р±РµР· РЅРёРєРѕРІ РѕС‚СЃСѓС‚СЃС‚РІСѓСЋС‚!"

        keyboard = (
            Keyboard(inline=True)
            .add(Callback("вЏЄ", {"command": "nonickMinus", "page": 1, "chatId": chat_id}),
                 color=KeyboardButtonColor.NEGATIVE)
            .add(Callback("РЎ РЅРёРєР°РјРё", {"command": "nicks", "chatId": chat_id}), color=KeyboardButtonColor.PRIMARY)
            .add(Callback("вЏ©", {"command": "nonickPlus", "page": 1, "chatId": chat_id}),
                 color=KeyboardButtonColor.POSITIVE)
        )

        await delete_message(message.group_id, message.object.peer_id, message.object.conversation_message_id)
        await bot.api.messages.send(peer_id=2000000000+chat_id, message=f"РџРѕР»СЊР·РѕРІР°С‚РµР»Рё Р±РµР· РЅРёРєРѕРІ [1]:\n{nonick_list}\n\nРџРѕР»СЊР·РѕРІР°С‚РµР»Рё СЃ РЅРёРєР°РјРё: В«/nlistВ»", disable_mentions=1, random_id=0 ,keyboard=keyboard)

    if command == "nicks":
        if await get_role(user_id, chat_id) < 1:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!"})
            )
            return True

        nicks = await nlist(chat_id, 1)
        nick_list = '\n'.join(nicks)
        if nick_list == "": nick_list = "РќРёРєРё РѕС‚СЃСѓС‚СЃС‚РІСѓСЋС‚!"

        keyboard = (
            Keyboard(inline=True)
            .add(Callback("вЏЄ", {"command": "nicksMinus", "page": 1, "chatId": chat_id}),
                 color=KeyboardButtonColor.NEGATIVE)
            .add(Callback("Р‘РµР· РЅРёРєРѕРІ", {"command": "nonicks", "chatId": chat_id}), color=KeyboardButtonColor.PRIMARY)
            .add(Callback("вЏ©", {"command": "nicksPlus", "page": 1, "chatId": chat_id}),
                 color=KeyboardButtonColor.POSITIVE)
        )

        await delete_message(message.group_id, message.object.peer_id, message.object.conversation_message_id)
        await bot.api.messages.send(peer_id=2000000000+chat_id, message=f"РџРѕР»СЊР·РѕРІР°С‚РµР»Рё СЃ РЅРёРєРѕРј [1 СЃС‚СЂР°РЅРёС†Р°]:\n{nick_list}\n\nРџРѕР»СЊР·РѕРІР°С‚РµР»Рё Р±РµР· РЅРёРєРѕРІ: В«/nonickВ»",
                            disable_mentions=1, keyboard=keyboard, random_id=0)

    if command == "nonickminus":
        if await get_role(user_id, chat_id) < 1:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!"})
            )
            return True

        page = payload.get("page")
        if page < 2:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "Р­С‚Рѕ РїРµСЂРІР°СЏ СЃС‚СЂР°РЅРёС†Р°!"})
            )
            return True

        nonicks = await nonick(chat_id, 1)
        nonick_list = '\n'.join(nonicks)
        if nonick_list == "": nonick_list = "РџРѕР»СЊР·РѕРІР°С‚РµР»Рё Р±РµР· РЅРёРєРѕРІ РѕС‚СЃСѓС‚СЃС‚РІСѓСЋС‚!"

        keyboard = (
            Keyboard(inline=True)
            .add(Callback("вЏЄ", {"command": "nonickMinus", "page": page+1, "chatId": chat_id}),
                 color=KeyboardButtonColor.NEGATIVE)
            .add(Callback("РЎ РЅРёРєР°РјРё", {"command": "nicks", "chatId": chat_id}), color=KeyboardButtonColor.PRIMARY)
            .add(Callback("вЏ©", {"command": "nonickPlus", "page": page+1, "chatId": chat_id}),
                 color=KeyboardButtonColor.POSITIVE)
        )

        await delete_message(message.group_id, message.object.peer_id, message.object.conversation_message_id)
        await bot.api.messages.send(peer_id=2000000000 + chat_id, message=f"РџРѕР»СЊР·РѕРІР°С‚РµР»Рё Р±РµР· РЅРёРєРѕРІ [{page-1}]:\n{nonick_list}\n\nРџРѕР»СЊР·РѕРІР°С‚РµР»Рё СЃ РЅРёРєР°РјРё: В«/nlistВ»", disable_mentions=1, random_id=0, keyboard=keyboard)

    if command == "nonickplus":
        if await get_role(user_id, chat_id) < 1:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!"})
            )
            return True
        page = payload.get("page")
        nonicks = await nonick(chat_id, page+1)
        if len(nonicks) < 1:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "Р­С‚Рѕ РїРѕСЃР»РµРґРЅСЏСЏ СЃС‚СЂР°РЅРёС†Р°!"})
            )
            return True

        nonicks_str = '\n'.join(nonicks)
        await delete_message(message.group_id, message.object.peer_id, message.object.conversation_message_id)
        await bot.api.messages.send(peer_id=2000000000 + chat_id,
                                    message=f"РџРѕР»СЊР·РѕРІР°С‚РµР»Рё Р±РµР· РЅРёРєРѕРІ [{page + 1}]:\n{nonicks_str}\n\nРџРѕР»СЊР·РѕРІР°С‚РµР»Рё СЃ РЅРёРєР°РјРё: В«/nlistВ»",
                                    disable_mentions=1, random_id=0, keyboard=keyboard)

    if command == "clear":
        if await get_role(user_id, chat_id) < 1:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!"})
            )
            return True

        user = payload.get("user")
        await clear(user, chat_id, message.group_id, 2000000000+chat_id)
        x = await bot.api.messages.get_by_conversation_message_id(peer_id=2000000000+chat_id, conversation_message_ids=message.object.conversation_message_id, group_id=message.group_id)
        x = json.loads(x.json())['items'][0]['text']
        await bot.api.messages.edit(peer_id=2000000000 + chat_id, message=x, conversation_message_id=message.object.conversation_message_id, keyboard=None)
        await bot.api.messages.send(peer_id=2000000000 + chat_id, message=f"@id{user_id} ({await get_user_name(user_id, chat_id)}) РѕС‡РёСЃС‚РёР»(-Р°) СЃРѕРѕР±С‰РµРЅРёСЏ", disable_mentions=1, random_id=0)

    if command == "unwarn":
        if await get_role(user_id, chat_id) < 1:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!"})
            )
            return True

        user = payload.get("user")
        if await equals_roles(user_id, user, chat_id, message) < 2:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "Р’С‹ РЅРµ РјРѕР¶РµС‚Рµ СЃРЅСЏС‚СЊ РїСЂРµРґ РґР°РЅРЅРѕРјСѓ РїРѕР»СЊР·РѕРІР°С‚РµР»СЋ!"})
            )
            return True

        await unwarn(chat_id, user)
        x = await bot.api.messages.get_by_conversation_message_id(peer_id=2000000000 + chat_id,conversation_message_ids=message.object.conversation_message_id,group_id=message.group_id)
        x = json.loads(x.json())['items'][0]['text']
        await bot.api.messages.edit(peer_id=2000000000 + chat_id, message=x, conversation_message_id=message.object.conversation_message_id, keyboard=None)
        await bot.api.messages.send(peer_id=2000000000 + chat_id, message=f"@id{user_id} ({await get_user_name(user_id, chat_id)}) СЃРЅСЏР»(-Р°) РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёРµ @id{user} ({await get_user_name(user, chat_id)})", disable_mentions=1, random_id=0)

    if command == 'stats':
        if await get_role(user_id, chat_id) < 1:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!"})
            )
            return True

        user = payload.get("user")
        reg_data = await get_registration_date(user)
        info = await bot.api.users.get(user)
        role = await get_role(user, chat_id)
        warns = await get_warns(user, chat_id)
        if await is_nick(user_id, chat_id):
            nick = await get_user_name(user, chat_id)
        else:
            nick = "РќРµС‚"
        messages = await message_stats(user_id, chat_id)

        roles = {0: "РџРѕР»СЊР·РѕРІР°С‚РµР»СЊ", 1: "РњРѕРґРµСЂР°С‚РѕСЂ", 2: "РЎС‚Р°СЂС€РёР№ РњРѕРґРµСЂР°С‚РѕСЂ", 3: "РђРґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂ",
                 4: "РЎС‚Р°СЂС€РёР№ РђРґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂ", 5: "Р’Р»Р°РґРµР»РµС† Р±РµСЃРµРґС‹", 6: "РњРµРЅРµРґР¶РµСЂ Р±РѕС‚Р°"}

        x = await bot.api.messages.get_by_conversation_message_id(peer_id=2000000000 + chat_id,
                                                                  conversation_message_ids=message.object.conversation_message_id,
                                                                  group_id=message.group_id)
        x = json.loads(x.json())['items'][0]['text']
        await bot.api.messages.edit(peer_id=2000000000 + chat_id, message=x,conversation_message_id=message.object.conversation_message_id, keyboard=None)
        await bot.api.messages.send(peer_id=2000000000 + chat_id, message=f"@id{user_id} ({await get_user_name(user_id, chat_id)}), СЃС‚Р°С‚РёСЃС‚РёРєР° @id{user} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ):\nРРјСЏ Рё С„Р°РјРёР»РёСЏ: {info[0].first_name} {info[0].last_name}\nР”Р°С‚Р° СЂРµРіРёСЃС‚СЂР°С†РёРё: {reg_data}\nРќРёРє: {nick}\nР РѕР»СЊ: {roles.get(role)}\nР’СЃРµРіРѕ РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёР№: {warns}/3\nР’СЃРµРіРѕ СЃРѕРѕР±С‰РµРЅРёР№: {messages['count']}\nРџРѕСЃР»РµРґРЅРµРµ СЃРѕРѕР±С‰РµРЅРёРµ: {messages['last']}", disable_mentions=1, random_id=0)

    if command == "activewarns":
        if await get_role(user_id, chat_id) < 1:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!"})
            )
            return True

        user = payload.get("user")
        warns = await gwarn(user, chat_id)
        string_info = str
        if not warns: string_info = "РђРєС‚РёРІРЅС‹С… РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёР№ РЅРµС‚!"
        else: string_info = f"@id{warns['moder']} (РњРѕРґРµСЂР°С‚РѕСЂ) | {warns['reason']} | {warns['count']}/3 | {warns['time']}"

        keyboard = (
            Keyboard(inline=True)
            .add(Callback("РСЃС‚РѕСЂРёСЏ РІСЃРµС… РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёР№", {"command": "warnhistory", "user": user, "chatId": chat_id}),
                 color=KeyboardButtonColor.PRIMARY)
        )

        x = await bot.api.messages.get_by_conversation_message_id(peer_id=2000000000 + chat_id,
                                                                  conversation_message_ids=message.object.conversation_message_id,
                                                                  group_id=message.group_id)
        x = json.loads(x.json())['items'][0]['text']
        await bot.api.messages.edit(peer_id=2000000000 + chat_id, message=x,
                                    conversation_message_id=message.object.conversation_message_id, keyboard=None)
        await bot.api.messages.send(peer_id=2000000000 + chat_id, message=f"@id{user_id} ({await get_user_name(user_id, chat_id)}), РёРЅС„РѕСЂРјР°С†РёСЏ Рѕ Р°РєС‚РёРІРЅС‹С… РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёСЏС… @id{user} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ):\n{string_info}", disable_mentions=1, keyboard=keyboard, random_id=0)

    if command == "warnhistory":
        if await get_role(user_id, chat_id) < 1:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!"})
            )
            return True

        user = payload.get("user")

        warnhistory_mass = await warnhistory(user, chat_id)
        if not warnhistory_mass:wh_string = "РџСЂРµРґСѓРїСЂРµР¶РґРµРЅРёР№ РЅРµ Р±С‹Р»Рѕ!"
        else:wh_string = '\n'.join(warnhistory_mass)

        x = await bot.api.messages.get_by_conversation_message_id(peer_id=2000000000 + chat_id,
                                                                  conversation_message_ids=message.object.conversation_message_id,
                                                                  group_id=message.group_id)
        x = json.loads(x.json())['items'][0]['text']
        await bot.api.messages.edit(peer_id=2000000000 + chat_id, message=x,
                                    conversation_message_id=message.object.conversation_message_id, keyboard=None)
        await bot.api.messages.send(peer_id=2000000000 + chat_id, message=f"РРЅС„РѕСЂРјР°С†РёСЏ Рѕ РІСЃРµС… РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёСЏС… @id{user} ({await get_user_name(user, chat_id)})\nРљРѕР»РёС‡РµСЃС‚РІРѕ РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёР№ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ: {await get_warns(user, chat_id)}\n\nРРЅС„РѕСЂРјР°С†РёСЏ Рѕ РїРѕСЃР»РµРґРЅРёС… 10 РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёР№ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ:\n{wh_string}",disable_mentions=1, random_id=0)

    if command == "unmute":
        if await get_role(user_id, chat_id) < 1:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!"})
            )
            return True

        user = payload.get("user")

        if await get_role(user_id, chat_id) <= await get_role(user, chat_id):
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!"})
            )
            return True

        await unmute(user, chat_id)
        x = await bot.api.messages.get_by_conversation_message_id(peer_id=2000000000 + chat_id,
                                                                  conversation_message_ids=message.object.conversation_message_id,
                                                                  group_id=message.group_id)
        x = json.loads(x.json())['items'][0]['text']
        await bot.api.messages.edit(peer_id=2000000000 + chat_id, message=x,
                                    conversation_message_id=message.object.conversation_message_id, keyboard=None)
        await bot.api.messages.send(peer_id=2000000000 + chat_id,
                                    message=f"@id{user_id} ({await get_user_name(user_id, chat_id)}) СЂР°Р·РјСѓС‚РёР»(-Р°) @id{user} ({await get_user_name(user, chat_id)})",
                                    disable_mentions=1, random_id=0)

    if command == "unban":
        if await get_role(user_id, chat_id) < 2:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!"})
            )
            return True

        user = payload.get("user")
        if await equals_roles(user_id, user, chat_id, message) < 2:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps(
                    {"type": "show_snackbar", "text": "Р’С‹ РЅРµ РјРѕР¶РµС‚Рµ СЃРЅСЏС‚СЊ Р±Р°РЅ РґР°РЅРЅРѕРјСѓ РїРѕР»СЊР·РѕРІР°С‚РµР»СЋ!"})
            )
            return True

        await unban(user, chat_id)
        x = await bot.api.messages.get_by_conversation_message_id(peer_id=2000000000 + chat_id,
                                                                  conversation_message_ids=message.object.conversation_message_id,
                                                                  group_id=message.group_id)
        x = json.loads(x.json())['items'][0]['text']
        await bot.api.messages.edit(peer_id=2000000000 + chat_id, message=x,
                                    conversation_message_id=message.object.conversation_message_id, keyboard=None)
        await bot.api.messages.send(peer_id=2000000000 + chat_id,
                                    message=f"@id{user_id} ({await get_user_name(user_id, chat_id)}) СЂР°Р·Р±Р»РѕРєРёСЂРѕРІР°Р»(-Р°) @id{user} ({await get_user_name(user, chat_id)})",
                                    disable_mentions=1, random_id=0)

    if command == "kick":
        if await get_role(user_id, chat_id) < 1:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!"})
            )
            return True

        user = payload.get("user")
        if await equals_roles(user_id, user, chat_id, message) < 2:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps(
                    {"type": "show_snackbar", "text": "Р’С‹ РЅРµ РјРѕР¶РµС‚Рµ РєРёРєРЅСѓС‚СЊ РґР°РЅРЅРѕРіРѕ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ!"})
            )
            return True

        try: await bot.api.messages.remove_chat_user(chat_id, user)
        except: pass

        x = await bot.api.messages.get_by_conversation_message_id(peer_id=2000000000 + chat_id,
                                                                  conversation_message_ids=message.object.conversation_message_id,
                                                                  group_id=message.group_id)
        x = json.loads(x.json())['items'][0]['text']
        await bot.api.messages.edit(peer_id=2000000000 + chat_id, message=x,
                                    conversation_message_id=message.object.conversation_message_id, keyboard=None)
        await bot.api.messages.send(peer_id=2000000000 + chat_id,
                                    message=f"@id{user_id} ({await get_user_name(user_id, chat_id)}) РєРёРєРЅСѓР»(-Р°) @id{user} ({await get_user_name(user, chat_id)})",
                                    disable_mentions=1, random_id=0)

    if command == "approve_form" or command == "reject_form":
        # РџРѕР»СѓС‡Р°РµРј chat_id РёР· peer_id, РµСЃР»Рё РЅСѓР¶РЅРѕ
        chat_id = message.object.peer_id
        if chat_id > 2000000000:  # Р±РµСЃРµРґР°
            chat_id -= 2000000000

        # РџСЂРѕРІРµСЂРєР° РїСЂР°РІ
        if await get_role(user_id, chat_id) < 8:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!"})
            )
            return True

        # РџРѕР»СѓС‡Р°РµРј РґР°РЅРЅС‹Рµ РёР· payload Р±РµР·РѕРїР°СЃРЅРѕ
        target = payload.get("target")
        sender = payload.get("sender")
        reason = payload.get("reason", "РќРµ СѓРєР°Р·Р°РЅРѕ")

        if not target or not sender:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "РћС€РёР±РєР°: РЅРµС‚ РґР°РЅРЅС‹С… РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ"})
            )
            return True

        # Р РµРґР°РєС‚РёСЂСѓРµРј РїСЂРµРґС‹РґСѓС‰РµРµ СЃРѕРѕР±С‰РµРЅРёРµ Р±РµР· РєРЅРѕРїРѕРє
        x_resp = await bot.api.messages.get_by_conversation_message_id(
            peer_id=message.object.peer_id,
            conversation_message_ids=message.object.conversation_message_id,
            group_id=message.group_id
        )
        items = json.loads(x_resp.json()).get('items', [])
        if not items:
            return True
        x_text = items[0]['text']

        await bot.api.messages.edit(
            peer_id=message.object.peer_id,
            message=x_text,
            conversation_message_id=message.object.conversation_message_id,
            keyboard=None
        )

        # Р’С‹РїРѕР»РЅСЏРµРј approve РёР»Рё reject
        if command == "approve_form":
            sql.execute(
                "INSERT INTO gbanlist (user_id, moderator_id, reason_gban, datetime_globalban) VALUES (?, ?, ?, ?)",
                (target, user_id, f"{reason} | By form | @id{sender} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЊ)",
                 datetime.now().strftime("%d.%m.%Y %H:%M"))
            )
            database.commit()

            await bot.api.messages.send(
                peer_id=message.object.peer_id,
                message=f"@id{user_id} ({await get_user_name(user_id, chat_id)}) РѕРґРѕР±СЂРёР» С„РѕСЂРјСѓ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ @id{sender} ({await get_user_name(sender, chat_id)})",
                disable_mentions=1,
                random_id=0
            )
        else:
            await bot.api.messages.send(
                peer_id=message.object.peer_id,
                message=f"@id{user_id} ({await get_user_name(user_id, chat_id)}) РѕС‚РєР»РѕРЅРёР» С„РѕСЂРјСѓ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ @id{sender} ({await get_user_name(sender, chat_id)})",
                disable_mentions=1,
                random_id=0
            )

        return True

    if command == "banwordsminus":
        if await get_role(user_id, chat_id) < 10:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id, peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!"})
            )
            return True

        page = payload.get("page")
        if page < 2:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id, peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "Р­С‚Рѕ РїРµСЂРІР°СЏ СЃС‚СЂР°РЅРёС†Р°!"})
            )
            return True

        sql.execute("SELECT word, creator_id, time FROM ban_words ORDER BY time DESC")
        rows = sql.fetchall()
        total = len(rows)
        per_page = 5
        max_page = (total + per_page - 1) // per_page

        async def get_words_page(page: int):
            start = (page - 1) * per_page
            end = start + per_page
            formatted = []
            for i, (word, creator, tm) in enumerate(rows[start:end], start=start + 1):
                try:
                    info = await bot.api.users.get(user_ids=creator)
                    creator_name = f"{info[0].first_name} {info[0].last_name}"
                except:
                    creator_name = "РќРµ СѓРґР°Р»РѕСЃСЊ РїРѕР»СѓС‡РёС‚СЊ РёРјСЏ"
                formatted.append(f"{i}. {word} | @id{creator} ({creator_name}) | Р’СЂРµРјСЏ: {tm}")
            return formatted

        new_page = page - 1
        words = await get_words_page(new_page)
        words_text = "\n\n".join(words)

        keyboard = (
            Keyboard(inline=True)
            .add(Callback("вЏЄ", {"command": "banwordsMinus", "page": new_page}), color=KeyboardButtonColor.NEGATIVE)
            .add(Callback("вЏ©", {"command": "banwordsPlus", "page": new_page}), color=KeyboardButtonColor.POSITIVE)
        )

        await delete_message(message.group_id, message.object.peer_id, message.object.conversation_message_id)
        await bot.api.messages.send(
            peer_id=message.object.peer_id,
            message=f"Р—Р°РїСЂРµС‰С‘РЅРЅС‹Рµ СЃР»РѕРІР° (РЎС‚СЂР°РЅРёС†Р°: {new_page}):\n\n{words_text}\n\nР’СЃРµРіРѕ Р·Р°РїСЂРµС‰РµРЅРЅС‹С… СЃР»РѕРІ: {total}",
            disable_mentions=1, random_id=0, keyboard=keyboard
        )
        return True


    if command == "banwordsplus":
        if await get_role(user_id, chat_id) < 10:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id, peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!"})
            )
            return True

        page = payload.get("page")

        sql.execute("SELECT word, creator_id, time FROM ban_words ORDER BY time DESC")
        rows = sql.fetchall()
        total = len(rows)
        per_page = 5
        max_page = (total + per_page - 1) // per_page

        async def get_words_page(page: int):
            start = (page - 1) * per_page
            end = start + per_page
            formatted = []
            for i, (word, creator, tm) in enumerate(rows[start:end], start=start + 1):
                try:
                    info = await bot.api.users.get(user_ids=creator)
                    creator_name = f"{info[0].first_name} {info[0].last_name}"
                except:
                    creator_name = "РќРµ СѓРґР°Р»РѕСЃСЊ РїРѕР»СѓС‡РёС‚СЊ РёРјСЏ"
                formatted.append(f"{i}. {word} | @id{creator} ({creator_name}) | Р’СЂРµРјСЏ: {tm}")
            return formatted

        new_page = page + 1
        words = await get_words_page(new_page)
        if len(words) < 1:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id, peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "Р­С‚Рѕ РїРѕСЃР»РµРґРЅСЏСЏ СЃС‚СЂР°РЅРёС†Р°!"})
            )
            return True

        words_text = "\n\n".join(words)
        keyboard = (
            Keyboard(inline=True)
            .add(Callback("вЏЄ", {"command": "banwordsMinus", "page": new_page}), color=KeyboardButtonColor.NEGATIVE)
            .add(Callback("вЏ©", {"command": "banwordsPlus", "page": new_page}), color=KeyboardButtonColor.POSITIVE)
        )

        await delete_message(message.group_id, message.object.peer_id, message.object.conversation_message_id)
        await bot.api.messages.send(
            peer_id=message.object.peer_id,
            message=f"Р—Р°РїСЂРµС‰С‘РЅРЅС‹Рµ СЃР»РѕРІР° (РЎС‚СЂР°РЅРёС†Р° {new_page}):\n\n{words_text}\n\nР’СЃРµРіРѕ Р·Р°РїСЂРµС‰РµРЅРЅС‹С… СЃР»РѕРІ: {total}",
            disable_mentions=1, random_id=0, keyboard=keyboard
        )
        return True        
        
    if command == "join_duel":
        try:
            # Р Р°Р·Р±РѕСЂ payload
            data = {}
            if message.object.payload:
                try:
                    if isinstance(message.object.payload, str):
                        data = json.loads(message.object.payload)
                    elif isinstance(message.object.payload, dict):
                        data = message.object.payload
                    else:
                        print(f"[join_duel] payload РЅРµРёР·РІРµСЃС‚РЅРѕРіРѕ С‚РёРїР°: {type(message.object.payload)}")
                except Exception as e:
                    print(f"[join_duel] РћС€РёР±РєР° РїР°СЂСЃРёРЅРіР° payload: {e}")

            peer = str(data.get("peer")) if data else None
            print(f"[join_duel] peer РёР· payload: {peer}")

            if not peer or peer not in duels:
                print(f"[join_duel] Р”СѓСЌР»СЊ РЅРµРґРѕСЃС‚СѓРїРЅР°: РєР»СЋС‡ '{peer}' РЅРµ РЅР°Р№РґРµРЅ РІ duels. "
                      f"РўРµРєСѓС‰РёРµ РєР»СЋС‡Рё: {list(duels.keys())}")
                await bot.api.messages.send_message_event_answer(
                    event_id=message.object.event_id,
                    peer_id=message.object.peer_id,
                    user_id=message.object.user_id,
                    event_data=json.dumps({"type": "show_snackbar", "text": "вљ”пёЏ Р”СѓСЌР»СЊ РЅРµРґРѕСЃС‚СѓРїРЅР°"})
                )
                return True

            duel = duels[peer]
            print(f"[join_duel] РќР°Р№РґРµРЅР° РґСѓСЌР»СЊ: {duel}")

            author = duel["author"]
            stake = duel["stake"]
            user_id = message.object.user_id

            if user_id == author:
                print("[join_duel] РРіСЂРѕРє РїС‹С‚Р°РµС‚СЃСЏ РІСЃС‚СѓРїРёС‚СЊ РІ СЃРІРѕСЋ Р¶Рµ РґСѓСЌР»СЊ!")
                await bot.api.messages.send_message_event_answer(
                    event_id=message.object.event_id,
                    peer_id=message.object.peer_id,
                    user_id=user_id,
                    event_data=json.dumps({"type": "show_snackbar", "text": "РўС‹ РЅРµ РјРѕР¶РµС€СЊ РІСЃС‚СѓРїРёС‚СЊ РІ СЃРІРѕСЋ Р¶Рµ РґСѓСЌР»СЊ!"})
                )
                return True

            # Р—Р°РіСЂСѓР¶Р°РµРј Р±Р°Р»Р°РЅСЃ
            balances = load_data(BALANCES_FILE)
            joiner = balances.get(str(user_id), get_balance(user_id))
            if joiner["wallet"] < stake:
                print(f"[join_duel] РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РјРѕРЅРµС‚ Сѓ {user_id}: {joiner['wallet']} < {stake}")
                await bot.api.messages.send_message_event_answer(
                    event_id=message.object.event_id,
                    peer_id=message.object.peer_id,
                    user_id=user_id,
                    event_data=json.dumps({"type": "show_snackbar", "text": "РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РјРѕРЅРµС‚!"})
                )
                return True

            # РћРїСЂРµРґРµР»СЏРµРј РїРѕР±РµРґРёС‚РµР»СЏ
            winner = random.choice([author, user_id])
            loser = user_id if winner == author else author
            print(f"[join_duel] РџРѕР±РµРґРёС‚РµР»СЊ: {winner}, РџСЂРѕРёРіСЂР°РІС€РёР№: {loser}")

            w_bal = balances.get(str(winner), get_balance(winner))
            l_bal = balances.get(str(loser), get_balance(loser))

            w_bal["wallet"] += stake
            w_bal["won"] += 1
            w_bal["won_total"] += stake

            l_bal["wallet"] -= stake
            l_bal["lost"] += 1
            l_bal["lost_total"] += stake

            balances[str(winner)] = w_bal
            balances[str(loser)] = l_bal
            save_data(BALANCES_FILE, balances)
            print("[join_duel] Р‘Р°Р»Р°РЅСЃС‹ РѕР±РЅРѕРІР»РµРЅС‹ Рё СЃРѕС…СЂР°РЅРµРЅС‹")

            # РџРѕР»СѓС‡Р°РµРј РёРјРµРЅР°
            try:
                w_info = await bot.api.users.get(user_ids=winner)
                l_info = await bot.api.users.get(user_ids=loser)
                w_name = f"{w_info[0].first_name} {w_info[0].last_name}"
                l_name = f"{l_info[0].first_name} {l_info[0].last_name}"
            except Exception as e:
                print(f"[join_duel] РћС€РёР±РєР° РїРѕР»СѓС‡РµРЅРёСЏ РёРјС‘РЅ: {e}")
                w_name = str(winner)
                l_name = str(loser)

            # РЈР±РёСЂР°РµРј РєРЅРѕРїРєРё СЃ РёСЃС…РѕРґРЅРѕРіРѕ СЃРѕРѕР±С‰РµРЅРёСЏ
            try:
                x_resp = await bot.api.messages.get_by_conversation_message_id(
                    peer_id=message.object.peer_id,
                    conversation_message_ids=duel["message_id"],
                    group_id=message.group_id
                )
                items = json.loads(x_resp.json()).get('items', [])
                if items:
                    x_text = items[0]['text']
                    await bot.api.messages.edit(
                        peer_id=message.object.peer_id,
                        message=x_text,
                        conversation_message_id=duel["message_id"],
                        keyboard=None
                    )
                    print("[join_duel] РљРЅРѕРїРєРё СѓСЃРїРµС€РЅРѕ СѓР±СЂР°РЅС‹")
            except Exception as e:
                print(f"[join_duel] РћС€РёР±РєР° РїСЂРё СѓРґР°Р»РµРЅРёРё РєРЅРѕРїРѕРє: {e}")

            # РћС‚РїСЂР°РІР»СЏРµРј СЂРµР·СѓР»СЊС‚Р°С‚
            await bot.api.messages.send(
                peer_id=message.object.peer_id,
                message=(
                    f"рџ’Ґ Р”СѓСЌР»СЊ Р·Р°РІРµСЂС€РµРЅР°!\n\n"
                    f"[id{winner}|{w_name}] vs [id{loser}|{l_name}]\n"
                    f"рџ‘‘ РџРѕР±РµРґРёС‚РµР»СЊ: [id{winner}|{w_name}]\n\n"
                    f"рџ’° РћРЅ Р·Р°Р±РёСЂР°РµС‚ {format_number(stake)}$"
                ),
                random_id=0
            )
            print("[join_duel] Р РµР·СѓР»СЊС‚Р°С‚ РѕС‚РїСЂР°РІР»РµРЅ")

            duels.pop(peer, None)
            save_data(DUELS_FILE, duels)
            print(f"[join_duel] Р”СѓСЌР»СЊ {peer} СѓРґР°Р»РµРЅР° РёР· СЃРїРёСЃРєР°")
            return True

        except Exception as e:
            print(f"[join_duel] РћР±С‰Р°СЏ РѕС€РёР±РєР°: {e}")
            return True
                           
    if command == "getban":
        target_user = payload.get("getban")
        if not target_user:
            return True

        # РџСЂРѕРІРµСЂСЏРµРј СЂРѕР»СЊ С‚РѕРіРѕ, РєС‚Рѕ РЅР°Р¶Р°Р» РєРЅРѕРїРєСѓ
        role = await get_role(user_id, chat_id)
        if role < 2:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({
                    "type": "show_snackbar",
                    "text": "РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ РґР»СЏ РїСЂРѕСЃРјРѕС‚СЂР° РёРЅС„РѕСЂРјР°С†РёРё Рѕ Р±Р»РѕРєРёСЂРѕРІРєР°С…!"
                })
            )
            return True

        # РЈРґР°Р»СЏРµРј СЃС‚Р°СЂРѕРµ СЃРѕРѕР±С‰РµРЅРёРµ
        try:
            await bot.api.messages.delete(
                group_id=message.group_id,
                peer_id=message.object.peer_id,
                cmids=message.object.conversation_message_id,
                delete_for_all=True
            )
        except:
            pass

        # РћС‚РїСЂР°РІР»СЏРµРј /getban
        await on_chat_message(
            Message(
                text=f"/getban {target_user}",
                from_id=message.object.user_id,
                peer_id=message.object.peer_id,
                chat_id=message.object.peer_id - 2000000000,
                group_id=message.group_id,
                object=message.object,
                random_id=0
            )
        )
        return True        

        if command == "kick_blacklisted":
            # РџСЂРѕРІРµСЂРєР° РїСЂР°РІ вЂ” РµСЃР»Рё РјРµРЅСЊС€Рµ 7, РїРѕРєР°Р·С‹РІР°РµРј snackbar
            if await get_role(user_id, chat_id) < 7:
                try:
                    await bot.api.messages.send_message_event_answer(
                        event_id=message.object.event_id,
                        peer_id=message.object.peer_id,
                        user_id=message.object.user_id,
                        event_data=json.dumps({
                            "type": "show_snackbar",
                            "text": "РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!"
                        })
                    )
                except:
                    pass
                return True

            # РџРѕР»СѓС‡Р°РµРј РїРѕР»СЊР·РѕРІР°С‚РµР»РµР№ РёР· blacklist
            sql.execute("SELECT user_id FROM blacklist")
            blacklisted = sql.fetchall()
            if not blacklisted:
                try:
                    await bot.api.messages.edit(
                        peer_id=message.peer_id,
                        conversation_message_id=message.conversation_message_id,
                        message="РќРµ СѓРґР°Р»РѕСЃСЊ РёСЃРєР»СЋС‡РёС‚СЊ РЅРё РѕРґРЅРѕРіРѕ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ РёР· Р§РЎР‘.",
                        keyboard=None
                    )
                except:
                    pass
                return True

            kicked_users = ""
            i = 1
            for user_ban in blacklisted:
                user_ban_id = user_ban[0]
                try:
                    await bot.api.messages.remove_chat_user(chat_id=chat_id, member_id=user_ban_id)
                    kicked_users += f"{i}. @id{user_ban_id} ({await get_user_name(user_ban_id, chat_id)})\n"
                    i += 1
                except:
                    pass  # РµСЃР»Рё РЅРµ СѓРґР°Р»РѕСЃСЊ РєРёРєРЅСѓС‚СЊ вЂ” РїСЂРѕРїСѓСЃРєР°РµРј

            # РЈР±РёСЂР°РµРј РєРЅРѕРїРєСѓ РёР· РёСЃС…РѕРґРЅРѕРіРѕ СЃРѕРѕР±С‰РµРЅРёСЏ
            try:
                await bot.api.messages.edit(
                    peer_id=message.peer_id,
                    conversation_message_id=message.conversation_message_id,
                    message="РЈРґР°Р»РµРЅРёРµ РїРѕР»СЊР·РѕРІР°С‚РµР»РµР№ РІ Р§РЎР‘, Р·Р°РІРµСЂС€РµРЅРѕ...",
                    keyboard=None
                )
            except:
                pass

            # РћС‚РїСЂР°РІР»СЏРµРј РѕС‚С‡С‘С‚, РµСЃР»Рё РєРѕРіРѕ-С‚Рѕ СЂРµР°Р»СЊРЅРѕ РёСЃРєР»СЋС‡РёР»Рё
            if kicked_users:
                await bot.api.messages.send(
                    peer_id=message.peer_id,
                    random_id=0,
                    message=(
                        f"@id{user_id} ({await get_user_name(user_id, chat_id)}), "
                        f"РёСЃРєР»СЋС‡РёР»(-Р°) РїРѕР»СЊР·РѕРІР°С‚РµР»РµР№ РІ Р§РЎР‘:\n\n{kicked_users}"
                    ),
                    disable_mentions=1
                )
            else:
                await bot.api.messages.send(
                    peer_id=message.peer_id,
                    random_id=0,
                    message="РќРµ СѓРґР°Р»РѕСЃСЊ РёСЃРєР»СЋС‡РёС‚СЊ РЅРё РѕРґРЅРѕРіРѕ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ РёР· Р§РЎР‘.",
                    disable_mentions=1
                )

            return True            

    if command == "infoidminus":
        page = payload.get("page")
        target = payload.get("user")

        if await get_role(user_id, chat_id) < 10:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!"})
            )
            return True

        if page < 2:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "Р­С‚Рѕ РїРµСЂРІР°СЏ СЃС‚СЂР°РЅРёС†Р°!"})
            )
            return True

        sql.execute("SELECT chat_id FROM chats WHERE owner_id = ?", (target,))
        user_chats = sql.fetchall()
        per_page = 5
        start = (page - 2) * per_page
        end = start + per_page
        page_chats = user_chats[start:end]

        all_chats = []
        for idx, (chat_id_val,) in enumerate(page_chats, start=1):
            try:
                peer_id = 2000000000 + chat_id_val
                info = await bot.api.messages.get_conversations_by_id(peer_ids=peer_id)
                if info.items:
                    chat_title = info.items[0].chat_settings.title
                else:
                    chat_title = "Р‘РµР· РЅР°Р·РІР°РЅРёСЏ"
                link = (await bot.api.messages.get_invite_link(peer_id=peer_id, reset=0)).link
            except:
                chat_title = "РќРµ СѓРґР°Р»РѕСЃСЊ РїРѕР»СѓС‡РёС‚СЊ"
                link = "РќРµ СѓРґР°Р»РѕСЃСЊ РїРѕР»СѓС‡РёС‚СЊ"

            all_chats.append(f"{idx}. {chat_title} | рџ†”: {chat_id_val} | рџ”— РЎСЃС‹Р»РєР°: {link}")

        keyboard = (
            Keyboard(inline=True)
            .add(Callback("РќР°Р·Р°Рґ", {"command": "infoidMinus", "page": page - 1, "user": target}), color=KeyboardButtonColor.NEGATIVE)
            .add(Callback("Р’РїРµСЂС‘Рґ", {"command": "infoidPlus", "page": page - 1, "user": target}), color=KeyboardButtonColor.POSITIVE)
        )

        await delete_message(message.group_id, message.object.peer_id, message.object.conversation_message_id)
        all_chats_text = "\n".join(all_chats)
        await bot.api.messages.send(
            peer_id=message.object.peer_id,
            message=f"вќ— РЎРїРёСЃРѕРє Р±РµСЃРµРґ @id{target} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ):\n(РЎС‚СЂР°РЅРёС†Р°: {page - 1})\n\n{all_chats_text}\n\nрџ—ЁпёЏ Р’СЃРµРіРѕ Р±РµСЃРµРґ Сѓ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ: {idx}",
            random_id=0,
            disable_mentions=1,
            keyboard=keyboard
        )
        
    if command == "infoidplus":
        page = payload.get("page")
        target = payload.get("user")

        if await get_role(user_id, chat_id) < 10:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!"})
            )
            return True

        sql.execute("SELECT chat_id FROM chats WHERE owner_id = ?", (target,))
        user_chats = sql.fetchall()
        per_page = 5
        total_pages = (len(user_chats) + per_page - 1) // per_page

        if page >= total_pages:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "Р­С‚Рѕ РїРѕСЃР»РµРґРЅСЏСЏ СЃС‚СЂР°РЅРёС†Р°!"})
            )
            return True

        start = page * per_page
        end = start + per_page
        page_chats = user_chats[start:end]

        all_chats = []
        for idx, (chat_id_val,) in enumerate(page_chats, start=1):
            try:
                peer_id = 2000000000 + chat_id_val
                info = await bot.api.messages.get_conversations_by_id(peer_ids=peer_id)
                if info.items:
                    chat_title = info.items[0].chat_settings.title
                else:
                    chat_title = "Р‘РµР· РЅР°Р·РІР°РЅРёСЏ"
                link = (await bot.api.messages.get_invite_link(peer_id=peer_id, reset=0)).link
            except:
                chat_title = "РќРµ СѓРґР°Р»РѕСЃСЊ РїРѕР»СѓС‡РёС‚СЊ"
                link = "РќРµ СѓРґР°Р»РѕСЃСЊ РїРѕР»СѓС‡РёС‚СЊ"

            all_chats.append(f"{idx}. {chat_title} | рџ†”: {chat_id_val} | рџ”— РЎСЃС‹Р»РєР°: {link}")

        keyboard = (
            Keyboard(inline=True)
            .add(Callback("РќР°Р·Р°Рґ", {"command": "infoidMinus", "page": page + 1, "user": target}), color=KeyboardButtonColor.NEGATIVE)
            .add(Callback("Р’РїРµСЂС‘Рґ", {"command": "infoidPlus", "page": page + 1, "user": target}), color=KeyboardButtonColor.POSITIVE)
        )

        await delete_message(message.group_id, message.object.peer_id, message.object.conversation_message_id)
        all_chats_text = "\n".join(all_chats)
        await bot.api.messages.send(
            peer_id=message.object.peer_id,
            message=f"вќ— РЎРїРёСЃРѕРє Р±РµСЃРµРґ @id{target} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ):\n(РЎС‚СЂР°РЅРёС†Р°: {page + 1})\n\n{all_chats_text}\n\nР’СЃРµРіРѕ Р±РµСЃРµРґ: {idx}",
            random_id=0,
            disable_mentions=1,
            keyboard=keyboard
        )        
              
    if command == "alt":
        if await get_role(user_id, chat_id) < 1:
            await bot.api.messages.send_message_event_answer(
                event_id=message.object.event_id,
                peer_id=message.object.peer_id,
                user_id=message.object.user_id,
                event_data=json.dumps({"type": "show_snackbar", "text": "РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!"})
            )
            return True

        commands_levels = {
            1: [
                '\nРљРѕРјР°РЅРґС‹ РјРѕРґРµСЂР°С‚РѕСЂРѕРІ:',
                '/setnick вЂ” snick, nick, addnick, РЅРёРє, СЃРµС‚РЅРёРє, Р°РґРґРЅРёРє',
                '/removenick вЂ”  removenick, clearnick, cnick, СЂРЅРёРє, СѓРґР°Р»РёС‚СЊРЅРёРє, СЃРЅСЏС‚СЊРЅРёРє',
                '/getnick вЂ” gnick, РіРЅРёРє, РіРµС‚РЅРёРє',
                '/getacc вЂ” acc, РіРµС‚Р°РєРє, Р°РєРєР°СѓРЅС‚, account',
                '/nlist вЂ” РЅРёРєРё, РІСЃРµРЅРёРєРё, nlist, nickslist, nicklist, nicks',
                '/nonick вЂ” nonicks, nonicklist, nolist, nnlist, Р±РµР·РЅРёРєРѕРІ, РЅРѕРЅРёРєСЃ',
                '/kick вЂ” РєРёРє, РёСЃРєР»СЋС‡РёС‚СЊ',
                '/warn вЂ” РїСЂРµРґ, РІР°СЂРЅ, pred, РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёРµ',
                '/unwarn вЂ” СѓРЅРІР°СЂРЅ, Р°РЅРІР°СЂРЅ, СЃРЅСЏС‚СЊРїСЂРµРґ, РјРёРЅСѓСЃРїСЂРµРґ',
                '/getwarn вЂ” gwarn, getwarns, РіРµС‚РІР°СЂРЅ, РіРІР°СЂРЅ',
                '/warnhistory вЂ” historywarns, whistory, РёСЃС‚РѕСЂРёСЏРІР°СЂРЅРѕРІ, РёСЃС‚РѕСЂРёСЏРїСЂРµРґРѕРІ',
                '/warnlist вЂ” warns, wlist, РІР°СЂРЅС‹, РІР°СЂРЅР»РёСЃС‚',
                '/staff вЂ” СЃС‚Р°С„С„',
                '/reg вЂ” registration, regdate, СЂРµРі, СЂРµРіРёСЃС‚СЂР°С†РёСЏ, РґР°С‚Р°СЂРµРіРёСЃС‚СЂР°С†РёРё',
                '/mute вЂ” РјСѓС‚, РјСЊСЋС‚, РјСѓС‚Рµ, addmute',
                '/unmute вЂ” СЃРЅСЏС‚СЊРјСѓС‚, Р°РЅРјСѓС‚, СѓРЅРјСѓС‚, СЃРЅСЏС‚СЊРјСѓС‚',
                '/alt вЂ” Р°Р»СЊС‚, Р°Р»СЊС‚РµСЂРЅР°С‚РёРІРЅС‹Рµ',
                '/getmute -- gmute, РіРјСѓС‚, РіРµС‚РјСѓС‚, С‡РµРєРјСѓС‚',
                '/mutelist -- mutes, РјСѓС‚С‹, РјСѓС‚Р»РёСЃС‚',
                '/clear -- С‡РёСЃС‚РєР°, РѕС‡РёСЃС‚РёС‚СЊ, РѕС‡РёСЃС‚РєР°',
                '/getban -- С‡РµРєР±Р°РЅ, РіРµС‚Р±Р°РЅ, checkban',
                '/delete -- СѓРґР°Р»РёС‚СЊ',
                '/chatid -- С‡Р°С‚Р°Р№РґРё, Р°Р№РґРёС‡Р°С‚Р°'
            ],
            2: [
                '\nРљРѕРјР°РЅРґС‹ СЃС‚Р°СЂС€РёС… РјРѕРґРµСЂР°С‚РѕСЂРѕРІ:',
                '/ban вЂ” Р±Р°РЅ, Р±Р»РѕРєРёСЂРѕРІРєР°',
                '/unban -- СѓРЅР±Р°РЅ, СЃРЅСЏС‚СЊР±Р°РЅ',
                '/addmoder -- moder',
                '/removerole -- rrole, СЃРЅСЏС‚СЊСЂРѕР»СЊ',
                '/zov - Р·РѕРІ, РІС‹Р·РѕРІ',
                '/online - ozov, РѕР·РѕРІ',
                '/onlinelist - olist, РѕР»РёСЃС‚',
                '/banlist - bans, Р±Р°РЅР»РёСЃС‚, Р±Р°РЅС‹',
                '/inactive - ilist, inactive',
                '/masskick - mkick'
            ],
            3: [
                '\nРљРѕРјР°РЅРґС‹ Р°РґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂРѕРІ:',
                '/quiet -- silence, С‚РёС€РёРЅР°',
                '/skick -- СЃРєРёРє, СЃРЅСЏС‚',
                '/sban -- СЃР±Р°РЅ',
                '/sunban вЂ” СЃСѓРЅР±Р°РЅ, СЃР°РЅР±Р°РЅ',
                '/addsenmoder вЂ” senmoder',
                '/rnickall -- allrnick, arnick, mrnick',
                '/sremovenick -- srnick',
                '/szov -- serverzov, СЃР·РѕРІ',
                '/srole -- none',
                '/ssetnick -- ssnick, СЃСЃРЅРёРє'
            ],
            4: [
                '\nРљРѕРјР°РЅРґС‹ СЃС‚Р°СЂС€РёС… Р°РґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂРѕРІ:',
                '/addadmin -- admin',
                '/serverinfo -- СЃРµСЂРІРµСЂРёРЅС„Рѕ',
                '/filter -- none',
                '/sremoverole -- srrole',
                '/bug -- Р±Р°Рі',
                '/report -- СЂРµРї, rep, Р¶Р°Р»РѕР±Р°'
            ],
            5: [
                '\nРљРѕРјР°РЅРґС‹ Р·Р°Рј. СЃРїРµС†. Р°РґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂРѕРІ:',
                '/addsenadmin -- senadm, addsenadm, senadmin',
                '/sync -- СЃРёРЅС…СЂРѕРЅРёР·Р°С†РёСЏ, СЃСѓРЅСЃ, СЃРёРЅС…СЂРѕРЅРєР°',
                '/pin -- Р·Р°РєСЂРµРїРёС‚СЊ, РїРёРЅ',
                '/unpin -- РѕС‚РєСЂРµРїРёС‚СЊ, СѓРЅРїРёРЅ',
                '/deleteall -- СѓРґР°Р»РёС‚СЊРІСЃРµ',
                '/gsinfo -- none',
                '/gsrnick -- none',
                '/gssnick -- none',
                '/gskick -- none',
                '/gsban -- none',
                '/gsunban -- none'
            ],
            6: [
                '\nРљРѕРјР°РЅРґС‹ СЃРїРµС†. Р°РґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂРѕРІ:',
                '/addzsa -- zsa, Р·СЃР°',
                '/server -- СЃРµСЂРІРµСЂ',
                '/settings -- РЅР°СЃС‚СЂРѕР№РєРё',
                '/clearwarn -- РѕС‡РёСЃС‚РёС‚СЊРІР°СЂРЅС‹',
                '/title -- none',
                '/antisliv -- Р°РЅС‚РёСЃР»РёРІ'
            ],
            7: [
                '\nРЎРїРёСЃРѕРє РєРѕРјР°РЅРґ РІР»Р°РґРµР»СЊС†Р° Р±РµСЃРµРґС‹',
                '/addsa -- sa, СЃР°, spec, specadm',
                '/antiflood -- af',
                '/welcometext -- welcome, wtext',
                '/invite -- none',
                '/leave -- none',
                '/editowner -- owner',
                '/Р·Р°С‰РёС‚Р° -- protection',
                '/settingsmute -- РЅР°СЃС‚СЂРѕР№РєРёРјСѓС‚Р°',
                '/setinfo -- СѓСЃС‚Р°РЅРѕРІРёС‚СЊРёРЅС„Рѕ',
                '/setrules -- СѓСЃС‚Р°РЅРѕРІРёС‚СЊРїСЂР°РІРёР»Р°',
                '/type -- С‚РёРї',
                '/gsync -- РїСЂРёРІСЏР·РєР°',
                '/gunsync -- СѓРґР°Р»РёС‚СЊРїСЂРёРІСЏР·РєСѓ'
            ]
        }

        user_role = await get_role(user_id, chat_id)

        commands = []
        for i in commands_levels.keys():
            if i <= user_role:
                for b in commands_levels[i]:
                    commands.append(b)

        level_commands = '\n'.join(commands)

        await bot.api.messages.edit(peer_id=2000000000 + chat_id, message=f"РђР»СЊС‚РµСЂРЅР°С‚РёРІРЅС‹Рµ РєРѕРјР°РЅРґС‹\n\n{level_commands}",
                                    conversation_message_id=message.object.conversation_message_id, keyboard=None)
                                                                       
@bot.on.chat_message()
async def on_chat_message(message: Message):
    bot_identifiers = ['!', '+', '/']

    user_id = message.from_id
    chat_id = message.chat_id
    peer_id = message.peer_id
    arguments = message.text.split(' ')
    arguments_lower = message.text.lower().split(' ')

    # --- РџСЂРѕРІРµСЂРєР° РЅР° Р±Р°РЅ С‡Р°С‚Р° РґРѕ РІСЃРµРіРѕ РѕСЃС‚Р°Р»СЊРЅРѕРіРѕ ---
    sql.execute("SELECT chat_id FROM banschats WHERE chat_id = ?", (chat_id,))
    if sql.fetchone():
        await message.reply("Р’Р»Р°РґРµР»РµС† Р±РµСЃРµРґС‹, РЅРµ С‡Р»РµРЅ СѓР¶Рµ BLACK MANAGER! РЇ РЅРµ Р±СѓРґСѓ Р·РґРµСЃСЊ СЂР°Р±РѕС‚Р°С‚СЊ.")
        return True

    # --- РџСЂРѕРІРµСЂРєР°, Р·Р°СЂРµРіРёСЃС‚СЂРёСЂРѕРІР°РЅ Р»Рё С‡Р°С‚ ---
    is_registered = await check_chat(chat_id)

    # --- РџСЂРѕРІРµСЂРєР° РЅР° Р·Р°РїСЂРµС‰С‘РЅРЅС‹Рµ СЃР»РѕРІР° ---
    if is_registered and await get_role(user_id, chat_id) <= 0:
        try:
            sql.execute("SELECT word FROM ban_words")
            banned_words = [row[0].lower() for row in sql.fetchall()]
            text_lower = message.text.lower()
            for word in banned_words:
                if word in text_lower:
                    admin = "blackrussiamanagerbot"
                    reason = "РќР°РїРёСЃР°РЅРёРµ Р·Р°РїСЂРµС‰РµРЅРЅС‹С… СЃР»РѕРІ"
                    mute_time = 30

                    await add_mute(user_id, chat_id, admin, reason, mute_time)

                    keyboard = (
                        Keyboard(inline=True)
                        .add(Callback("РЎРЅСЏС‚СЊ РјСѓС‚", {"command": "unmute", "user": user_id, "chatId": chat_id}), color=KeyboardButtonColor.POSITIVE)
                    )

                    await message.reply(
                        f"вќ— @id{user_id} (РџРѕР»СЊР·РѕРІР°С‚РµР»СЊ), РІС‹ РїРѕР»СѓС‡РёР»Рё РјСѓС‚ РЅР° 30 РјРёРЅСѓС‚ Р·Р° РЅР°РїРёСЃР°РЅРёРµ Р·Р°РїСЂРµС‰РµРЅРЅС‹С… СЃР»РѕРІ!",
                        disable_mentions=1,
                        keyboard=keyboard
                    )

                    await bot.api.messages.delete(
                        group_id=message.group_id,
                        peer_id=message.peer_id,
                        delete_for_all=True,
                        cmids=message.conversation_message_id
                    )
                    return True
        except Exception as e:
            print(f"[BANWORDS] РћС€РёР±РєР° РїСЂРѕРІРµСЂРєРё СЃР»РѕРІ: {e}")            

    # --- РџСЂРѕРІРµСЂРєР° РјСѓС‚Р° Рё СЂРµР°РєС†РёРё РІ Р·Р°РІРёСЃРёРјРѕСЃС‚Рё РѕС‚ РЅР°СЃС‚СЂРѕРµРє (С‚РѕР»СЊРєРѕ РµСЃР»Рё С‡Р°С‚ Р°РєС‚РёРІРёСЂРѕРІР°РЅ) ---
    if is_registered and await get_mute(user_id, chat_id) and not await checkMute(chat_id, user_id):
        sql.execute("SELECT mode FROM mutesettings WHERE chat_id = ?", (chat_id,))
        mode_data = sql.fetchone()
        mode = mode_data[0] if mode_data else 0

        warns = await get_warns(user_id, chat_id)

        if mode == 1:
            if warns < 3:
                bot_name = "blackrussiamanagerbot"
                reason = "РќР°РїРёСЃР°РЅРёРµ СЃР»РѕРІ РІ РјСѓС‚Рµ"
                await warn(chat_id, user_id, bot_name, reason)
                await message.reply(
                    f"Р’ РґР°РЅРЅРѕРј С‡Р°С‚Рµ Р·Р°РїСЂРµС‰РµРЅРѕ РѕС‚РїСЂР°РІР»СЏС‚СЊ СЃРѕРѕР±С‰РµРЅРёСЏ РІРѕ РІСЂРµРјСЏ РјСѓС‚Р°.\n"
                    f"@id{user_id} ({await get_user_name(user_id, chat_id)}), РІР°Рј РІС‹РґР°РЅРѕ РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёРµ В«{warns}/3В»\n\n"
                    f"РџСЂРё РґРѕСЃС‚РёР¶РµРЅРёРё 3/3 РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёР№ РІС‹ Р±СѓРґРµС‚Рµ РёСЃРєР»СЋС‡РµРЅС‹.",
                    disable_mentions=1
                )
                await bot.api.messages.delete(
                    group_id=message.group_id,
                    peer_id=message.peer_id,
                    delete_for_all=True,
                    cmids=message.conversation_message_id
                )
            else:
                try:
                    await bot.api.messages.remove_chat_user(chat_id, user_id)
                    await message.reply(
                        f"@id{user_id} ({await get_user_name(user_id, chat_id)}) Р±С‹Р» РёСЃРєР»СЋС‡РµРЅ Р·Р° РїСЂРµРІС‹С€РµРЅРёРµ Р»РёРјРёС‚Р° РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёР№!",
                        disable_mentions=1
                    )
                    await clear_warns(chat_id, user_id)
                except:
                    await message.reply(
                        f"РќРµ СѓРґР°Р»РѕСЃСЊ РёСЃРєР»СЋС‡РёС‚СЊ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ @id{user_id}. Р’РѕР·РјРѕР¶РЅРѕ, РЅРµС‚ РїСЂР°РІ Р°РґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂР°.",
                        disable_mentions=1
                    )
        else:
            await bot.api.messages.delete(
                group_id=message.group_id,
                peer_id=message.peer_id,
                delete_for_all=True,
                cmids=message.conversation_message_id
            )

    # --- РџСЂРѕРІРµСЂРєР° РЅР° РЅР°Р»РёС‡РёРµ Р·Р°Р±Р»РѕРєРёСЂРѕРІР°РЅРЅС‹С… РїРѕР»СЊР·РѕРІР°С‚РµР»РµР№ (С‚РѕР»СЊРєРѕ РµСЃР»Рё С‡Р°С‚ Р°РєС‚РёРІРёСЂРѕРІР°РЅ) ---
    if is_registered:
        sql.execute("SELECT user_id, moderator_id, reason_gban FROM blacklist")
        blacklisted = sql.fetchall()

        if any(user_id == b[0] for b in blacklisted):
            users = ""
            i = 1
            for user_ban in blacklisted:
                user_ban_id, moderator, reason = user_ban
                users += f"\n{i}. @id{user_ban_id} ({await get_user_name(user_ban_id, chat_id)}) | " \
                         f"@id{moderator} (РњРѕРґРµСЂР°С‚РѕСЂ) | РџСЂРёС‡РёРЅР°: {reason}\n"
                i += 1

            chat_info = await bot.api.messages.get_conversations_by_id(peer_ids=message.peer_id)
            chat_title = chat_info.items[0].chat_settings.title if chat_info.items else "РќРµРёР·РІРµСЃС‚РЅР°СЏ Р±РµСЃРµРґР°"

            keyboard = (
                Keyboard(inline=True)
                .add(Callback("РСЃРєР»СЋС‡РёС‚СЊ РІСЃРµС… Р·Р°Р±Р»РѕРєРёСЂРѕРІР°РЅРЅС‹С…", {"command": "kick_blacklisted", "chatId": chat_id}),
                     color=KeyboardButtonColor.NEGATIVE)
            )

            await message.reply(
                f"Р’ С‡Р°С‚Рµ В«{chat_title}В» РЅР°С…РѕРґСЏС‚СЃСЏ Р·Р°Р±Р»РѕРєРёСЂРѕРІР°РЅРЅС‹Рµ РїРѕР»СЊР·РѕРІР°С‚РµР»Рё.\n\n"
                f"вќ— | РЎРїРёСЃРѕРє РІСЃРµС… РїРѕР»СЊР·РѕРІР°С‚РµР»РµР№ РІ С‡РµСЂРЅРѕРј СЃРїРёСЃРєРµ Р±РѕС‚Р°:\n{users}\n\n"
                f"Р РµРєРѕРјРµРЅРґСѓРµРј РёСЃРєР»СЋС‡РёС‚СЊ РїРѕР»СЊР·РѕРІР°С‚РµР»РµР№ РёР· Р±РµСЃРµРґС‹, С‚Р°Рє РєР°Рє РѕРЅРё РЅР°СЂСѓС€РёР»Рё РїСЂР°РІРёР»Р° РёСЃРїРѕР»СЊР·РѕРІР°РЅРёСЏ Р±РѕС‚Р°.",
                disable_mentions=1,
                keyboard=keyboard
            )
            return True

    # --- РўРµРїРµСЂСЊ РѕР±СЂР°Р±Р°С‚С‹РІР°РµРј РєРѕРјР°РЅРґС‹ (РєРѕРјР°РЅРґС‹ РґРѕСЃС‚СѓРїРЅС‹ РІСЃРµРіРґР°) ---
    try:
        command_identifier = arguments[0].strip()[0]
        command = arguments_lower[0][1:]
    except:
        command_identifier = " "
        command = " "

    if command_identifier in bot_identifiers:
        try:
            test_admin = await bot.api.messages.get_conversation_members(peer_id=message.peer_id)
        except:
            await message.reply("РћР¶РёРґР°СЋ РІС‹РґР°С‡Рё Р·РІС‘Р·РґРѕС‡РєРё С‡С‚РѕР±С‹ РЅР°С‡Р°С‚СЊ СЂР°Р±РѕС‚Сѓ СЃ С‡Р°С‚РѕРј!", disable_mentions=1)
            return True

        # --- Р•СЃР»Рё С‡Р°С‚ РЅРµ Р°РєС‚РёРІРёСЂРѕРІР°РЅ, СЂР°Р·СЂРµС€Р°РµРј С‚РѕР»СЊРєРѕ /start ---
        if not is_registered and command not in ['start', 'СЃС‚Р°СЂС‚', 'Р°РєС‚РёРІРёСЂРѕРІР°С‚СЊ']:
            await message.reply("вљ пёЏ РЎРЅР°С‡Р°Р»Р° Р°РєС‚РёРІРёСЂСѓР№С‚Рµ С‡Р°С‚ РїСЂРё РїРѕРјРѕС‰Рё РєРѕРјР°РЅРґС‹ /start", disable_mentions=1)
            return True

        # ==== РџСЂРѕРІРµСЂРєР° РіР»РѕР±Р°Р»СЊРЅРѕРіРѕ Р±Р°РЅР° ====
        if is_registered:
            sql.execute("SELECT * FROM gbanlist WHERE user_id = ?", (user_id,))
            check_global = sql.fetchone()
            if check_global:
                moderator_id = check_global[1]
                reason_gban = check_global[2]
                datetime_globalban = check_global[3]

                try:
                    resp = await bot.api.users.get(user_ids=user_id)
                    full_name = f"{resp[0].first_name} {resp[0].last_name}"
                except:
                    full_name = str(user_id)

                keyboard = (
                    Keyboard(inline=True)
                    .add(Callback("РЎРЅСЏС‚СЊ Р±Р°РЅ", {}), color=KeyboardButtonColor.POSITIVE)
                )

                await message.reply(
                    f"@id{user_id} ({full_name}) Р·Р°Р±Р»РѕРєРёСЂРѕРІР°РЅ(-Р°) РІ Р±РµСЃРµРґР°С… РёРіСЂРѕРєРѕРІ!\n\n"
                    f"РРЅС„РѕСЂРјР°С†РёСЏ Рѕ Р±Р»РѕРєРёСЂРѕРІРєРµ:\n@id{moderator_id} (РњРѕРґРµСЂР°С‚РѕСЂ) | {reason_gban} | {datetime_globalban}",
                    disable_mentions=1,
                    keyboard=keyboard
                )
                await bot.api.messages.remove_chat_user(chat_id, user_id)
                return True
                
        # ==== РџСЂРѕРІРµСЂРєР° РіР»РѕР±Р°Р»СЊРЅРѕРіРѕ Р±Р°РЅР° ====
        if is_registered:
            sql.execute("SELECT * FROM globalban WHERE user_id = ?", (user_id,))
            check_global = sql.fetchone()
            if check_global:
                moderator_id = check_global[1]
                reason_gban = check_global[2]
                datetime_globalban = check_global[3]

                try:
                    resp = await bot.api.users.get(user_ids=user_id)
                    full_name = f"{resp[0].first_name} {resp[0].last_name}"
                except:
                    full_name = str(user_id)

                keyboard = (
                    Keyboard(inline=True)
                    .add(Callback("РЎРЅСЏС‚СЊ Р±Р°РЅ", {}), color=KeyboardButtonColor.POSITIVE)
                )

                await message.reply(
                    f"@id{user_id} ({full_name}) Р·Р°Р±Р»РѕРєРёСЂРѕРІР°РЅ(-Р°) РІРѕ РІСЃРµС… Р±РµСЃРµРґР°С…!\n\n"
                    f"РРЅС„РѕСЂРјР°С†РёСЏ Рѕ Р±Р»РѕРєРёСЂРѕРІРєРµ:\n@id{moderator_id} (РњРѕРґРµСЂР°С‚РѕСЂ) | {reason_gban} | {datetime_globalban}",
                    disable_mentions=1,
                    keyboard=keyboard
                )
                await bot.api.messages.remove_chat_user(chat_id, user_id)
                return True                
                                        
        if command in ['start', 'СЃС‚Р°СЂС‚', 'Р°РєС‚РёРІРёСЂРѕРІР°С‚СЊ']:
            if await check_chat(chat_id):
                await message.reply("Р‘РѕС‚ Р±С‹Р» СЂР°РЅРµРµ Р°РєС‚РёРІРёСЂРѕРІР°РЅ РІ РґР°РЅРЅРѕР№ Р±РµСЃРµРґРµ!", disable_mentions=1)
                return True
            await new_chat(chat_id, peer_id, user_id)
            await message.reply("Р‘РµСЃРµРґР° СѓСЃРїРµС€РЅРѕ Р·Р°РЅРµСЃРµРЅР° РІ Р±Р°Р·Сѓ РґР°РЅРЅС‹С… Р±РѕС‚Р°!\n\nРСЃРїРѕР»СЊР·СѓР№С‚Рµ В«/helpВ» РґР»СЏ РѕР·РЅР°РєРѕРјР»РµРЅРёСЏ СЃРїРёСЃРєР° РєРѕРјР°РЅРґ!", disable_mentions=1)
            return True  

        # ---------------- FORM ----------------
        if command in ["form", "С„РѕСЂРјР°"]:
            if chat_id != 9:
                await message.reply(
                    "вќ— РљРѕРјР°РЅРґР° РґРѕСЃС‚СѓРїРЅР° С‚РѕР»СЊРєРѕ [https://vk.me/join/Am_qZQ/ppZ90u1wU6Zrd5w0vJKGFKpN1M0M=|РІ С„РѕСЂРјР°С… РЅР° Р±Р»РѕРєРёСЂРѕРІРєСѓ]"
                )
                return True

            # РћРїСЂРµРґРµР»СЏРµРј target
            target = None
            reason = "РќРµ СѓРєР°Р·Р°РЅРѕ"
            if message.reply_message:
                target = message.reply_message.from_id
                if len(arguments) > 1:
                    reason = await get_string(arguments, 1)
            elif len(arguments) > 1 and await getID(arguments[1]):
                target = await getID(arguments[1])
                if len(arguments) > 2:
                    reason = await get_string(arguments, 2)
            else:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ С‡РµСЂРµР· СЂРµРїР»Р°Р№ РёР»Рё ID!")
                return True

            if await equals_roles(user_id, user, chat_id, message) < 2:
                await message.reply("Р’С‹ РЅРµ РјРѕР¶РµС‚Рµ РїРѕРґР°С‚СЊ С„РѕСЂРјСѓ РЅР° РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ РІС‹С€Рµ РІР°СЃ СЂР°РЅРіРѕРј!", disable_mentions=1)
                return True

            sender_name = await get_user_name(user_id, chat_id)
            target_name = await get_user_name(target, chat_id)
            name = datetime.now().strftime("%I:%M:%S %p")

            # РљР»Р°РІРёР°С‚СѓСЂР° СЃ РєРЅРѕРїРєР°РјРё
            keyboard = (
                Keyboard(inline=True)
                .add(
                    Callback(
                        "РћРґРѕР±СЂРёС‚СЊ",
                        {"command": "approve_form", "target": target, "sender": user_id, "reason": reason},
                    ),
                    color=KeyboardButtonColor.POSITIVE,
                )
                .add(
                    Callback(
                        "РћС‚РєР°Р·Р°С‚СЊ",
                        {"command": "reject_form", "target": target, "sender": user_id, "reason": reason},
                    ),
                    color=KeyboardButtonColor.NEGATIVE,
                )
            )

            # РћС‚РїСЂР°РІР»СЏРµРј СЃРѕРѕР±С‰РµРЅРёРµ РїСЂСЏРјРѕ РІ С‡Р°С‚, РѕС‚РєСѓРґР° РїСЂРёС€Р»Р° РєРѕРјР°РЅРґР°
            await message.reply(
                (
                    f"рџ“Њ | Р¤РѕСЂРјР° РЅР° В«/gbanplВ»:\n"
                    f"1. РџРѕР»СЊР·РѕРІР°С‚РµР»СЊ: @id{user_id} ({sender_name})\n"
                    f"2. РќР°СЂСѓС€РёС‚РµР»СЊ: @id{target} ({target_name})\n"
                    f"3. РџСЂРёС‡РёРЅР°: {reason}\n"
                    f"4. Р”Р°С‚Р° РїРѕРґР°С‡Рё С„РѕСЂРјС‹: {name} РњРЎРљ (UTC+3)"
                ),
                keyboard=keyboard,
            )
            return True            

        if command in ['id', 'РёРґ', 'getid', 'РіРµС‚РёРґ', 'РїРѕР»СѓС‡РёС‚СЊРёРґ', 'giveid']:
            user = int
            if message.reply_message:
                user = message.reply_message.from_id
            elif len(arguments) >= 2 and await getID(arguments[1]):
                user = await getID(arguments[1])
            else:
                user = user_id
            if user < 0:
                await message.reply(f"РћСЂРёРіРёРЅР°Р»СЊРЅР°СЏ СЃСЃС‹Р»РєР° [club{abs(user)}|СЃРѕРѕР±С‰РµСЃС‚РІР°]:\nhttps://vk.com/club{abs(user)}",disable_mentions=1)
                return True
            await message.reply(f"РћСЂРёРіРёРЅР°Р»СЊРЅР°СЏ СЃСЃС‹Р»РєР° @id{user} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ):\nhttps://vk.com/id{user}", disable_mentions=1)

        if message.reply_message and message.reply_message.from_id < 0:
            return True
            
        if command in ['РјРёРЅРµС‚', 'РѕС‚СЃРѕСЃ', 'РѕС‚СЃРѕСЃР°С‚СЊ', 'minet', 'СЃРѕСЃР°С‚СЊ']:
            if message.reply_message:
                user = message.reply_message.from_id
            elif len(arguments) >= 2 and await getID(arguments[1]):
                user = await getID(arguments[1])
            else:
                user = user_id

            # РџРѕР»СѓС‡Р°РµРј РёРјСЏ С†РµР»Рё
            try:
                info = await bot.api.users.get(user_ids=user)
                name_target = f"{info[0].first_name} {info[0].last_name}"
            except:
                if user < 0:
                    name_target = f"@club{abs(user)} (РќРµ СѓРґР°Р»РѕСЃСЊ РїРѕР»СѓС‡РёС‚СЊ РёРјСЏ)"
                else:
                    name_target = f"@id{user} (РќРµ СѓРґР°Р»РѕСЃСЊ РїРѕР»СѓС‡РёС‚СЊ РёРјСЏ)"

            # РџРѕР»СѓС‡Р°РµРј РёРјСЏ РёРЅРёС†РёР°С‚РѕСЂР°
            try:
                info = await bot.api.users.get(user_ids=user_id)
                name = f"{info[0].first_name} {info[0].last_name}"
            except:
                name = f"@id{user_id} (РќРµ СѓРґР°Р»РѕСЃСЊ РїРѕР»СѓС‡РёС‚СЊ РёРјСЏ)"

            if user < 0:
                await message.reply(
                    f"рџ”ћ | @id{user_id} ({name}) РѕС‚СЃРѕСЃР°Р»(-Р°) Сѓ @club{abs(user)} ({name_target})",
                    disable_mentions=1
                )
            else:
                await message.reply(
                    f"рџ”ћ | @id{user_id} ({name}) РѕС‚СЃРѕСЃР°Р»(-Р°) Сѓ @id{user} ({name_target})",
                    disable_mentions=1
                )
            return True
      
        if command in ['С‚СЂР°С…РЅСѓС‚СЊ', 'СЃРµРєСЃ', 'seks', 'С‚СЂР°С…', 'trax']:
            if message.reply_message:
                user = message.reply_message.from_id
            elif len(arguments) >= 2 and await getID(arguments[1]):
                user = await getID(arguments[1])
            else:
                user = user_id

            # РџРѕР»СѓС‡Р°РµРј РёРјСЏ С†РµР»Рё
            try:
                info = await bot.api.users.get(user_ids=user)
                name_target = f"{info[0].first_name} {info[0].last_name}"
            except:
                if user < 0:
                    name_target = f"@club{abs(user)} (РќРµ СѓРґР°Р»РѕСЃСЊ РїРѕР»СѓС‡РёС‚СЊ РёРјСЏ)"
                else:
                    name_target = f"@id{user} (РќРµ СѓРґР°Р»РѕСЃСЊ РїРѕР»СѓС‡РёС‚СЊ РёРјСЏ)"

            # РџРѕР»СѓС‡Р°РµРј РёРјСЏ РёРЅРёС†РёР°С‚РѕСЂР°
            try:
                info = await bot.api.users.get(user_ids=user_id)
                name = f"{info[0].first_name} {info[0].last_name}"
            except:
                name = f"@id{user_id} (РќРµ СѓРґР°Р»РѕСЃСЊ РїРѕР»СѓС‡РёС‚СЊ РёРјСЏ)"

            if user < 0:
                await message.reply(
                    f"рџ”ћ | @id{user_id} ({name}) РїСЂРёРЅСѓРґРёР»(-Р°) Рє РёРЅС‚РёРјСѓ @club{abs(user)} ({name_target})",
                    disable_mentions=1
                )
            else:
                await message.reply(
                    f"рџ”ћ | @id{user_id} ({name}) РїСЂРёРЅСѓРґРёР»(-Р°) Рє РёРЅС‚РёРјСѓ @id{user} ({name_target})",
                    disable_mentions=1
                )
            return True      

        # ---------------- OFFER ----------------
        if command in ["offer", "РїСЂРµРґР»РѕР¶РµРЅРёРµ"]:
            try:
                user_info = await bot.api.users.get(user_ids=user_id)
                full_name = f"{user_info[0].first_name} {user_info[0].last_name}"
            except:
                full_name = f"id{user_id} (РћС€РёР±РєР°)"

            args = message.text.split(maxsplit=1)
            if len(arguments) < 2 or len(args[1]) < 5:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РїСЂРµРґР»РѕР¶РµРЅРёРµ РїРѕ СѓР»СѓС‡С€РµРЅРёСЋ!")
                return

            offer = args[1]

            ADMIN_ID = 860294414

            await bot.api.messages.send(
                peer_id=2000000017,
                random_id=0,
                message=(
                    f"в­ђ | РџСЂРµРґР»РѕР¶РµРЅРёРµ РїРѕ СѓР»СѓС‡С€РµРЅРёСЋ Р±РѕС‚Р°:\n"
                    f"1. РџРѕР»СЊР·РѕРІР°С‚РµР»СЊ: [id{user_id}|{full_name}]\n"
                    f"2. РџСЂРµРґР»РѕР¶РµРЅРёРµ РїРѕ СѓР»СѓС‡С€РµРЅРёСЋ: {offer}\n"
                    f"3. Р”Р°С‚Р° РїРѕРґР°С‡Рё СѓР»СѓС‡С€РµРЅРёСЏ: NULL"
                )
            )
            
            await chats_log(user_id=user_id, target_id=None, role=None, log=f"РїРѕРґР°Р»(-Р°) РїСЂРµРґР»РѕР¶РµРЅРёРµ РїРѕ СѓР»СѓС‡С€РµРЅРёСЋ СЃ СЃРѕРґРµСЂР¶Р°РЅРёРµРј: В«{offer}В»")            
            await message.reply("РЎРїР°СЃРёР±Рѕ Р·Р° РїСЂРµРґР»РѕР¶РµРЅРёРµ РїРѕ СѓР»СѓС‡С€РµРЅРёСЋ Р±РѕС‚Р°! РњС‹ РѕР±СЏР·Р°С‚РµР»СЊРЅРѕ СЂР°СЃСЃРјРѕС‚СЂРёРј РІР°С€Рµ РїСЂРµРґР»РѕР¶РµРЅРёРµ.")
            return

        if command in ['Р»РѕРіСЌРєРѕРЅРѕРјРёРєРё', 'logeco', 'logeconomy', 'Р»РѕРіРёСЌРєРѕ']:
            if await get_role(user_id, chat_id) < 8:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return True

            target = None
            if message.reply_message:
                target = message.reply_message.from_id
            elif len(arguments) >= 2 and await getID(arguments[1]):
                target = await getID(arguments[1])

            if target:
                # --- Р›РѕРіРё РєРѕРЅРєСЂРµС‚РЅРѕРіРѕ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ ---
                sql.execute("SELECT * FROM economy WHERE user_id = ? ORDER BY rowid DESC LIMIT 9999999999999", (target,))
                logs = sql.fetchall()

                if not logs:
                    await message.reply(f"РЈ @id{target} ({await get_user_name(target, chat_id)}) РѕС‚СЃСѓС‚СЃС‚РІСѓСЋС‚ Р·Р°РїРёСЃРё РІ Р»РѕРіР°С… СЌРєРѕРЅРѕРјРёРєРё.", disable_mentions=1)
                    return True

                economy_text = ""
                i = 0
                for entry in logs:
                    i += 1
                    u_id, t_id, amount, log_text = entry

                    try:
                        u_info = await bot.api.users.get(user_ids=u_id)
                        u_name = f"{u_info[0].first_name} {u_info[0].last_name}"
                    except:
                        u_name = str(u_id)

                    if t_id:
                        try:
                            t_info = await bot.api.users.get(user_ids=t_id)
                            t_name = f"{t_info[0].first_name} {t_info[0].last_name}"
                            t_display = f"@id{t_id} ({t_name})"
                        except:
                            t_display = f"@id{t_id}"
                    else:
                        t_display = "None"

                    a_display = f"{format_number(amount)}$" if amount else "None"
                    l_display = log_text if log_text else "вЂ”"

                    economy_text += f"{i}. @id{u_id} ({u_name}) | РљРѕРјСѓ: {t_display} | РЎРєРѕР»СЊРєРѕ: {a_display} | Р›РѕРі: {l_display}\n\n"

                await message.reply(
                    f"РЎРїРёСЃРѕРє РґРµР№СЃС‚РІРёР№ СЃ СЌРєРѕРЅРѕРјРёРєРѕР№ @id{target} ({await get_user_name(target, chat_id)}):\n\n{economy_text}",
                    disable_mentions=1
                )
                return True

            else:
                # --- РћР±С‰РёРµ Р»РѕРіРё СЌРєРѕРЅРѕРјРёРєРё ---
                sql.execute("SELECT * FROM economy ORDER BY rowid DESC LIMIT 9999999999999")
                logs = sql.fetchall()

                if not logs:
                    await message.reply(f"Р›РѕРіРё СЌРєРѕРЅРѕРјРёРєРё РѕС‚СЃСѓС‚СЃС‚РІСѓСЋС‚!", disable_mentions=1)
                    return True

                economy_text = ""
                i = 0
                for entry in logs:
                    i += 1
                    u_id, t_id, amount, log_text = entry

                    try:
                        u_info = await bot.api.users.get(user_ids=u_id)
                        u_name = f"{u_info[0].first_name} {u_info[0].last_name}"
                    except:
                        u_name = str(u_id)

                    if t_id:
                        try:
                            t_info = await bot.api.users.get(user_ids=t_id)
                            t_name = f"{t_info[0].first_name} {t_info[0].last_name}"
                            t_display = f"@id{t_id} ({t_name})"
                        except:
                            t_display = f"@id{t_id}"
                    else:
                        t_display = "None"

                    a_display = f"{format_number(amount)}$" if amount else "None"
                    l_display = log_text if log_text else "вЂ”"

                    economy_text += f"{i}. @id{u_id} ({u_name}) | РљРѕРјСѓ: {t_display} | РЎРєРѕР»СЊРєРѕ: {a_display} | Р›РѕРі: {l_display}\n\n"

                await message.reply(
                    f"@id{user_id} ({await get_user_name(user_id, chat_id)}), Р»РѕРіРёСЂРѕРІР°РЅРёРµ РѕР±С‰РµР№ СЌРєРѕРЅРѕРјРёРєРё Р±РѕС‚Р°:\n\n{economy_text}",
                    disable_mentions=1
                )
                return True

        # === Р”РѕР±Р°РІР»РµРЅРёРµ РІ Р§С‘СЂРЅС‹Р№ СЃРїРёСЃРѕРє ===
        if command in ['addblack', 'Р±Р»РµРєР»РёСЃС‚', 'С‡СЃ', 'blackadd', 'addch']:
            if await get_role(user_id, chat_id) < 10:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return True

            if chat_id == 23:
                await message.reply("Р”Р°РЅРЅР°СЏ Р±РµСЃРµРґР° РїСЂРѕРІРѕРґРёС‚СЃСЏ РІ СЃРїРµС†РёР°Р»РёР·РёСЂРѕРІР°РЅРЅРѕРј С‡Р°С‚Рµ, РєРѕС‚РѕСЂС‹Р№ РїСЂРµРґРЅР°Р·РЅР°С‡РµРЅ РёСЃРєР»СЋС‡РёС‚РµР»СЊРЅРѕ РґР»СЏ С‚РµСЃС‚РёСЂРѕРІС‰РёРєРѕРІ Р±РѕС‚Р°.\n\nР’ СЂР°РјРєР°С… РґР°РЅРЅРѕРіРѕ РѕР±СЃСѓР¶РґРµРЅРёСЏ РЅРµ РґРѕРїСѓСЃРєР°РµС‚СЃСЏ РёСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ РєРѕРјР°РЅРґ, РЅРµ РѕС‚РЅРѕСЃСЏС‰РёС…СЃСЏ Рє СЂР°Р±РѕС‚Рµ РїРѕ С‚РµСЃС‚РёСЂРѕРІР°РЅРёСЋ РёР»Рё С„СѓРЅРєС†РёРѕРЅРёСЂРѕРІР°РЅРёСЋ СЃРёСЃС‚РµРјС‹ РІ С†РµР»РѕРј.", disable_mentions=1)
                return True

            # РћРїСЂРµРґРµР»СЏРµРј РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ
            target = int
            arg = 0
            if message.reply_message:
                target = message.reply_message.from_id
                arg = 1
            elif message.fwd_messages and message.fwd_messages[0].from_id > 0:
                target = message.fwd_messages[0].from_id
                arg = 1
            elif len(arguments) >= 2 and await getID(arguments[1]):
                target = await getID(arguments[1])
                arg = 2
            else:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ!", disable_mentions=1)
                return True

            # РџСЂРѕРІРµСЂРєР° вЂ” РЅРµ РІ Р§РЎ Р»Рё СѓР¶Рµ
            sql.execute("SELECT * FROM blacklist WHERE user_id = ?", (target,))
            if sql.fetchone():
                await message.reply("Р”Р°РЅРЅС‹Р№ РїРѕР»СЊР·РѕРІР°С‚РµР»СЊ СѓР¶Рµ РЅР°С…РѕРґРёС‚СЃСЏ РІ С‡РµСЂРЅРѕРј СЃРїРёСЃРєРµ Р±РѕС‚Р°!", disable_mentions=1)
                return True

            if await equals_roles(user_id, target, chat_id, message) < 2:
                await message.reply("Р’С‹ РЅРµ РјРѕР¶РµС‚Рµ РґРѕР±Р°РІРёС‚СЊ РґР°РЅРЅРѕРіРѕ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ РІ Р§РЎ!", disable_mentions=1)
                return True

            reason = await get_string(arguments, arg)
            if not reason:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РїСЂРёС‡РёРЅСѓ Р±Р»РѕРєРёСЂРѕРІРєРё!", disable_mentions=1)
                return True

            date_now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

            sql.execute("INSERT INTO blacklist (user_id, moderator_id, reason_gban, datetime_globalban) VALUES (?, ?, ?, ?)",
                        (target, user_id, reason, date_now))
            database.commit()

            await message.reply(f"@id{user_id} ({await get_user_name(user_id, chat_id)}) РґРѕР±Р°РІРёР» @id{target} ({await get_user_name(target, chat_id)}) РІ С‡РµСЂРЅС‹Р№ СЃРїРёСЃРѕРє Р±РѕС‚Р°", disable_mentions=1)
            await chats_log(user_id=user_id, target_id=target, role=None, log=f"РґРѕР±Р°РІРёР» @id{target} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ) РІ Р§С‘СЂРЅС‹Р№ СЃРїРёСЃРѕРє. РџСЂРёС‡РёРЅР°: {reason}")            
            return True


        # === РЈРґР°Р»РµРЅРёРµ РёР· Р§С‘СЂРЅРѕРіРѕ СЃРїРёСЃРєР° ===
        if command in ['unblack', 'СѓР±СЂР°С‚СЊС‡СЃ', 'blackdel', 'unch']:
            if await get_role(user_id, chat_id) < 10:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return True

            if chat_id == 23:
                await message.reply("Р”Р°РЅРЅР°СЏ Р±РµСЃРµРґР° РїСЂРѕРІРѕРґРёС‚СЃСЏ РІ СЃРїРµС†РёР°Р»РёР·РёСЂРѕРІР°РЅРЅРѕРј С‡Р°С‚Рµ, РєРѕС‚РѕСЂС‹Р№ РїСЂРµРґРЅР°Р·РЅР°С‡РµРЅ РёСЃРєР»СЋС‡РёС‚РµР»СЊРЅРѕ РґР»СЏ С‚РµСЃС‚РёСЂРѕРІС‰РёРєРѕРІ Р±РѕС‚Р°.\n\nР’ СЂР°РјРєР°С… РґР°РЅРЅРѕРіРѕ РѕР±СЃСѓР¶РґРµРЅРёСЏ РЅРµ РґРѕРїСѓСЃРєР°РµС‚СЃСЏ РёСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ РєРѕРјР°РЅРґ, РЅРµ РѕС‚РЅРѕСЃСЏС‰РёС…СЃСЏ Рє СЂР°Р±РѕС‚Рµ РїРѕ С‚РµСЃС‚РёСЂРѕРІР°РЅРёСЋ РёР»Рё С„СѓРЅРєС†РёРѕРЅРёСЂРѕРІР°РЅРёСЋ СЃРёСЃС‚РµРјС‹ РІ С†РµР»РѕРј.", disable_mentions=1)
                return True

            target = int
            if message.reply_message:
                target = message.reply_message.from_id
            elif message.fwd_messages and message.fwd_messages[0].from_id > 0:
                target = message.fwd_messages[0].from_id
            elif len(arguments) >= 2 and await getID(arguments[1]):
                target = await getID(arguments[1])
            else:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ!", disable_mentions=1)
                return True

            sql.execute("SELECT * FROM blacklist WHERE user_id = ?", (target,))
            if not sql.fetchone():
                await message.reply("Р”Р°РЅРЅС‹Р№ РїРѕР»СЊР·РѕРІР°С‚РµР»СЊ РЅРµ РЅР°С…РѕРґРёС‚СЃСЏ РІ С‡РµСЂРЅРѕРј СЃРїРёСЃРєРµ Р±РѕС‚Р°!", disable_mentions=1)
                return True

            sql.execute("DELETE FROM blacklist WHERE user_id = ?", (target,))
            database.commit()

            await message.reply(f"@id{user_id} ({await get_user_name(user_id, chat_id)}) СѓРґР°Р»РёР» @id{target} ({await get_user_name(target, chat_id)}) РёР· С‡РµСЂРЅРѕРіРѕ СЃРїРёСЃРєР° Р±РѕС‚Р°!", disable_mentions=1)
            await chats_log(user_id=user_id, target_id=target, role=None, log=f"СѓРґР°Р»РёР» @id{target} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ) РёР· Р§С‘СЂРЅРѕРіРѕ СЃРїРёСЃРєР°")            
            return True           
                
        if command in ['Р»РѕРіРёРѕР±С‰РёРµ', 'logs', 'logsmoders', 'Р»РѕРіРё']:
            if await get_role(user_id, chat_id) < 8:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return True

            target = None
            if message.reply_message:
                target = message.reply_message.from_id
            elif len(arguments) >= 2 and await getID(arguments[1]):
                target = await getID(arguments[1])

            if target:
                # --- Р›РѕРіРё РєРѕРЅРєСЂРµС‚РЅРѕРіРѕ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ ---
                sql.execute("SELECT * FROM logchats WHERE user_id = ? ORDER BY rowid DESC LIMIT 9999999999999", (target,))
                logs = sql.fetchall()

                if not logs:
                    await message.reply(f"РЈ @id{target} ({await get_user_name(target, chat_id)}) РѕС‚СЃСѓС‚СЃС‚РІСѓСЋС‚ Р·Р°РїРёСЃРё РІ Р»РѕРіР°С… РјРѕРґРµСЂР°С†РёРё.", disable_mentions=1)
                    return True

                economy_text = ""
                i = 0
                for entry in logs:
                    i += 1
                    u_id, t_id, amount, log_text = entry

                    try:
                        u_info = await bot.api.users.get(user_ids=u_id)
                        u_name = f"{u_info[0].first_name} {u_info[0].last_name}"
                    except:
                        u_name = str(u_id)

                    if t_id:
                        try:
                            t_info = await bot.api.users.get(user_ids=t_id)
                            t_name = f"{t_info[0].first_name} {t_info[0].last_name}"
                            t_display = f"@id{t_id} ({t_name})"
                        except:
                            t_display = f"@id{t_id}"
                    else:
                        t_display = "None"

                    a_display = f"{format_number(amount)}$" if amount else "None"
                    l_display = log_text if log_text else "вЂ”"

                    economy_text += f"{i}. @id{u_id} ({u_name}) | РљРѕРјСѓ: {t_display} | Р РѕР»СЊ: {a_display} | Р›РѕРі: {l_display}\n\n"

                await message.reply(
                    f"РЎРїРёСЃРѕРє РґРµР№СЃС‚РІРёР№ СЃ РґРµР№СЃС‚РІРёСЏРјРё РјРѕРґРµСЂР°С‚РѕСЂРѕРІ @id{target} ({await get_user_name(target, chat_id)}):\n\n{economy_text}",
                    disable_mentions=1
                )
                return True

            else:
                # --- РћР±С‰РёРµ Р»РѕРіРё СЌРєРѕРЅРѕРјРёРєРё ---
                sql.execute("SELECT * FROM logchats ORDER BY rowid DESC LIMIT 9999999999999")
                logs = sql.fetchall()

                if not logs:
                    await message.reply(f"Р›РѕРіРё СЃ РґРµР№СЃС‚РІРёСЏРјРё РјРѕРґРµСЂР°С‚РѕСЂРѕРІ РѕС‚СЃСѓС‚СЃС‚РІСѓСЋС‚!", disable_mentions=1)
                    return True

                economy_text = ""
                i = 0
                for entry in logs:
                    i += 1
                    u_id, t_id, amount, log_text = entry

                    try:
                        u_info = await bot.api.users.get(user_ids=u_id)
                        u_name = f"{u_info[0].first_name} {u_info[0].last_name}"
                    except:
                        u_name = str(u_id)

                    if t_id:
                        try:
                            t_info = await bot.api.users.get(user_ids=t_id)
                            t_name = f"{t_info[0].first_name} {t_info[0].last_name}"
                            t_display = f"@id{t_id} ({t_name})"
                        except:
                            t_display = f"@id{t_id}"
                    else:
                        t_display = "None"

                    a_display = f"{format_number(amount)}$" if amount else "None"
                    l_display = log_text if log_text else "вЂ”"

                    economy_text += f"{i}. @id{u_id} ({u_name}) | РљРѕРјСѓ: {t_display} | Р РѕР»СЊ: {a_display} | Р›РѕРі: {l_display}\n\n"

                await message.reply(
                    f"@id{user_id} ({await get_user_name(user_id, chat_id)}), Р»РѕРіРёСЂРѕРІР°РЅРёРµ РѕР±С‰РёС… РґРµР№СЃС‚РІРёР№ РјРѕРґРµСЂР°С‚РѕСЂРѕРІ:\n\n{economy_text}",
                    disable_mentions=1
                )
                return True
                            
        if command in ["casino", "РєР°Р·РёРЅРѕ"]:
            if len(arguments) < 1:
                await message.reply("рџЋ° РЈРєР°Р¶Рё СЃСѓРјРјСѓ СЃС‚Р°РІРєРё: /РєР°Р·РёРЅРѕ 10000")
                return

            try:
                stake = int(arguments[-1])
            except:
                await message.reply("РЎС‚Р°РІРєР° РґРѕР»Р¶РЅР° Р±С‹С‚СЊ С‡РёСЃР»РѕРј!")
                return

            if stake < 100:
                await message.reply("РњРёРЅРёРјР°Р»СЊРЅР°СЏ СЃС‚Р°РІРєР° РґРѕР»Р¶РЅР° Р±С‹С‚СЊ вЂ” 10$")
                return

            balances = load_data(BALANCES_FILE)
            bal = balances.get(str(user_id), get_balance(user_id))

            if bal["wallet"] < stake:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ СЃСЂРµРґСЃС‚РІ РґР»СЏ СЃС‚Р°РІРєРё!")
                return

            # Р­РјРѕРґР·Рё СЂСѓР»РµС‚РєРё
            emojis = ["рџ’Ћ", "рџЌ’", "рџЌЂ", "рџЄ™", "рџ””", "рџЌ‹", "рџ’°", "в­ђпёЏ", "рџ”Ґ", "рџЋІ"]

            # Р“РµРЅРµСЂР°С†РёСЏ СЃР»СѓС‡Р°Р№РЅС‹С… С‚СЂС‘С… СЌРјРѕРґР·Рё
            result = random.choices(emojis, k=3)

            # РџСЂРѕРІРµСЂРєР° РЅР° РґР¶РµРєРїРѕС‚
            jackpot = False
            if result[0] == result[1] == result[2]:
                jackpot = True

            # РџРѕРґСЃС‡РёС‚С‹РІР°РµРј Р±РѕРЅСѓСЃС‹
            multiplier = 0.0
            bonuses = {
                "рџ’Ћ": 0.3,  # 30%
                "рџЄ™": 0.1,  # 10%
                "рџ””": 0.5   # 50%
            }

            triggered = []
            for emoji, bonus in bonuses.items():
                if emoji in result:
                    multiplier += bonus
                    triggered.append(emoji)

            # Р‘Р°Р·РѕРІС‹Р№ РІС‹РёРіСЂС‹С€ / РїСЂРѕРёРіСЂС‹С€
            if multiplier == 0 and not jackpot:
                # РџСЂРѕРёРіСЂС‹С€
                bal["wallet"] -= stake
                balances[str(user_id)] = bal
                save_data(BALANCES_FILE, balances)

                await message.reply(
                    f"рџЋ° Р’С‹ СЃС‹РіСЂР°Р»Рё РЅР° СЃС‚Р°РІРєСѓ В«{format_number(stake)}В»\n"
                    f"Р РµР·СѓР»СЊС‚Р°С‚: {' '.join(result)}\n\n"
                    f"вќЊ РќРµ РІС‹РїР°Р»Рё рџ’Ћ, рџЄ™ РёР»Рё рџ”” вЂ” РІС‹ РїСЂРѕРёРіСЂР°Р»Рё!"
                )
                return
            else:
                win_amount = stake

                if multiplier > 0:
                    win_amount = int(stake * (1 + multiplier))

                # Р•СЃР»Рё РґР¶РµРєРїРѕС‚ вЂ” СѓС‚СЂРѕРёС‚СЊ РІС‹РёРіСЂС‹С€
                if jackpot:
                    win_amount = int(win_amount * 3)

                profit = win_amount - stake
                bal["wallet"] -= stake
                bal["wallet"] += win_amount
                balances[str(user_id)] = bal
                save_data(BALANCES_FILE, balances)
                await log_economy(user_id=user_id, target_id=None, amount=stake, log=f"СЃС‹РіСЂР°Р»(-Р°) РІ В«РљР°Р·РёРЅРѕВ» РЅР° {stake}$")

                emoji_str = ", ".join(triggered) if triggered else "РЅРµС‚"
                jackpot_text = ""
                if jackpot:
                    jackpot_text = f"\n\nвќ— JECKPOT! 3 РѕРґРёРЅР°РєРѕРІС‹С… {result[0]} рџ”Ґрџ”Ґрџ”Ґ"

                await message.reply(
                    f"рџЋ° Р’С‹ СЃС‹РіСЂР°Р»Рё РЅР° СЃС‚Р°РІРєСѓ В«{format_number(stake)}В»\n"
                    f"Р РµР·СѓР»СЊС‚Р°С‚: {' '.join(result)}{jackpot_text}\n\n"
                    f"Р’С‹РїР°Р»Рё: {emoji_str}\n"
                    f"рџ“€ РћР±С‰РёР№ Р±РѕРЅСѓСЃ: +{int(multiplier * 100)}%\n"
                    f"рџ’° Р’С‹РёРіСЂС‹С€: {format_number(win_amount)}$ (РїСЂРёР±С‹Р»СЊ: {format_number(profit)}$)"
                )
                return            
            
        # ---------------- BUG ----------------
        if command in ["bug", "Р±Р°Рі"]:
            if await get_role(user_id, chat_id) < 4:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!")
                return True
        	
            try:
                user_info = await bot.api.users.get(user_ids=user_id)
                full_name = f"{user_info[0].first_name} {user_info[0].last_name}"
            except:
                full_name = f"id{user_id} (РћС€РёР±РєР°)"

            args = message.text.split(maxsplit=1)
            if len(arguments) < 2 or len(args[1]) < 5:
                await message.reply("РЎР»РёС€РєРѕРј РєРѕСЂРѕС‚РєРёР№ Р±Р°Рі!")
                return

            offer = args[1]

            ADMIN_ID = 860294414

            await bot.api.messages.send(
                peer_id=2000000017,
                random_id=0,
                message=(
                    f"рџ‘ѕ | Р‘Р°Рі-С‚СЂРµРєРµСЂ:\n"
                    f"1. РџРѕР»СЊР·РѕРІР°С‚РµР»СЊ: [id{user_id}|{full_name}]\n"
                    f"2. РЎРѕРґРµСЂР¶РёРјРѕРµ Р±Р°РіР°: {offer}\n"
                    f"3. Р”Р°С‚Р° РїРѕРґР°С‡Рё Р±Р°РіР°: NULL"
                )
            )
            
            await chats_log(user_id=user_id, target_id=None, role=None, log=f"РїРѕРґР°Р»(-Р°) Р±Р°Рі-СЂРµРїРѕСЂС‚ СЃ СЃРѕРґРµСЂР¶Р°РЅРёРµРј: В«{offer}В»")            
            await message.reply("Р’Р°С€ Р±Р°Рі СЂРµРїРѕСЂС‚ Р±С‹Р» РѕС‚РїСЂР°РІР»РµРЅ СЂР°Р·СЂР°Р±РѕС‚С‡РёРєСѓ!")
            return            

        if command in ['stats', 'СЃС‚Р°С‚Р°', 'СЃС‚Р°С‚РёСЃС‚РёРєР°', 'stata', 'statistic']:
                # РћРїСЂРµРґРµР»СЏРµРј РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ РґР»СЏ РїРѕРєР°Р·Р° СЃС‚Р°С‚РёСЃС‚РёРєРё
                user = int
                if message.reply_message:
                    user = message.reply_message.from_id
                elif len(arguments) >= 2 and await getID(arguments[1]):
                    user = await getID(arguments[1])
                else:
                    user = user_id

                if user < 0:
                    await message.reply("РќРµР»СЊР·СЏ РІР·Р°РёРјРѕРґРµР№СЃС‚РІРѕРІР°С‚СЊ СЃ СЃРѕРѕР±С‰РµСЃС‚РІРѕРј!")
                    return True

                reg_data = "-"  # РІРјРµСЃС‚Рѕ РґР°С‚С‹ СЂРµРіРёСЃС‚СЂР°С†РёРё
                role = await get_role(user, chat_id)
                warns = await get_warns(user, chat_id)

                # РџРѕР»СѓС‡Р°РµРј РЅРёРє
                if await is_nick(user, chat_id):
                    nick = await get_user_name(user, chat_id)
                else:
                    nick = "РќРµС‚"

                # РџРѕР»СѓС‡Р°РµРј РёРјСЏ Рё С„Р°РјРёР»РёСЋ С‡РµСЂРµР· VK
                try:
                    info = await bot.api.users.get(user_ids=user)
                    name = f"{info[0].first_name} {info[0].last_name}"
                except:
                    name = f"@id{user} (РќРµ СѓРґР°Р»РѕСЃСЊ РїРѕР»СѓС‡РёС‚СЊ РёРјСЏ)"

                messages = await message_stats(user, chat_id)

                # РџСЂРѕРІРµСЂРєР° РіР»РѕР±Р°Р»СЊРЅРѕРіРѕ Р±Р°РЅР°
                sql.execute("SELECT * FROM gbanlist WHERE user_id = ?", (user,))
                gban = sql.fetchone()
                gban_status = "Р”Р°" if gban else "РќРµС‚"

                # РџСЂРѕРІРµСЂРєР° РіР»РѕР±Р°Р»СЊРЅРѕРіРѕ Р±Р°РЅР° 2
                sql.execute("SELECT * FROM globalban WHERE user_id = ?", (user,))
                gban2 = sql.fetchone()
                globalban = "Р”Р°" if gban2 else "РќРµС‚"

                # РџСЂРѕРІРµСЂСЏРµРј, РµСЃС‚СЊ Р»Рё РјСѓС‚
                sql.execute(f"SELECT * FROM mutes_{chat_id} WHERE user_id = ?", (user,))
                mute = sql.fetchone()
                mute_status = "Р”Р°" if mute else "РќРµС‚"

                # --- РџСЂРѕРІРµСЂРєР° Р±Р°РЅРѕРІ РІРѕ РІСЃРµС… С‡Р°С‚Р°С… ---
                sql.execute("SELECT chat_id FROM chats")
                chats_list = sql.fetchall()
                bans = ""
                bans_count = 0
                i = 1
                for c in chats_list:
                    chat_id_check = c[0]
                    try:
                        sql.execute(f"SELECT moder, reason, date FROM bans_{chat_id_check} WHERE user_id = ?", (user,))
                        user_bans = sql.fetchall()
                        if user_bans:
                            bans_count += len(user_bans)
                            for ub in user_bans:
                                mod, reason, date = ub
                                bans += f"{i}) @id{mod} (РњРѕРґРµСЂР°С‚РѕСЂ) | {reason} | {date} РњРЎРљ (UTC+3)\n"
                                i += 1
                    except:
                        continue  # РµСЃР»Рё С‚Р°Р±Р»РёС†С‹ РЅРµС‚, РїСЂРѕРїСѓСЃРєР°РµРј

                roles = {
                    0: "РџРѕР»СЊР·РѕРІР°С‚РµР»СЊ",
                    1: "РњРѕРґРµСЂР°С‚РѕСЂ",
                    2: "РЎС‚Р°СЂС€РёР№ РјРѕРґРµСЂР°С‚РѕСЂ",
                    3: "РђРґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂ",
                    4: "РЎС‚Р°СЂС€РёР№ Р°РґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂ",
                    5: "Р—Р°Рј. СЃРїРµС† Р°РґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂР°",
                    6: "РЎРїРµС† Р°РґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂ",
                    7: "Р’Р»Р°РґРµР»РµС† Р±РµСЃРµРґС‹",
                    8: "Р—Р°РјРµСЃС‚РёС‚РµР»СЊ СЂСѓРєРѕРІРѕРґРёС‚РµР»СЏ",
                    9: "РћСЃРЅРѕРІРЅРѕР№ Р·Р°Рј. СЂСѓРєРѕРІРѕРґРёС‚РµР»СЏ",
                    10: "РЎРїРµС†РёР°Р»СЊРЅС‹Р№ СЂСѓРєРѕРІРѕРґРёС‚РµР»СЊ",
                    11: "Р Р°Р·СЂР°Р±РѕС‚С‡РёРє Р±РѕС‚Р°",
                    12: "рџ‘ѕ РўРµСЃС‚РёСЂРѕРІС‰РёРє Р±РѕС‚Р°",
                    13: "рџ‘ѕ Р—Р°Рј. РіР»Р°РІРЅРѕРіРѕ С‚РµСЃС‚РёСЂРѕРІС‰РёРєР° Р±РѕС‚Р°",
                    14: "рџ‘ѕ Р“Р»Р°РІРЅС‹Р№ С‚РµСЃС‚РёСЂРѕРІС‰РёРє Р±РѕС‚Р°"
                }

                # РЎРѕР·РґР°С‘Рј РєР»Р°РІРёР°С‚СѓСЂСѓ С‚РѕР»СЊРєРѕ РµСЃР»Рё СЂРѕР»СЊ > 1
                keyboard = None
                if await get_role(user_id, chat_id) > 1:
                    keyboard = Keyboard(inline=True)
                    keyboard.add(
                        Callback("Р’СЃРµ РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёСЏ", {"command": "activeWarns", "user": user, "chatId": chat_id}),
                        color=KeyboardButtonColor.PRIMARY
                    )
                    keyboard.add(
                        Callback("РРЅС„РѕСЂРјР°С†РёСЏ Рѕ Р±Р»РѕРєРёСЂРѕРІРєР°С…", {"command": "getban", "user": user, "chatId": chat_id}),
                        color=KeyboardButtonColor.PRIMARY
                    )

                await message.reply(
                    f"РРЅС„РѕСЂРјР°С†РёСЏ Рѕ @id{user} (РїРѕР»СЊР·РѕРІР°С‚РµР»Рµ):\n"
                    f"Р РѕР»СЊ: {roles.get(role)}\n"
                    f"Р‘Р»РѕРєРёСЂРѕРІРѕРє: {bans_count}\n"
                    f"РћР±С‰Р°СЏ Р±Р»РѕРєРёСЂРѕРІРєР° РІ С‡Р°С‚Р°С…: {globalban}\n"
                    f"РћР±С‰Р°СЏ Р±Р»РѕРєРёСЂРѕРІРєР° РІ Р±РµСЃРµРґР°С… РёРіСЂРѕРєРѕРІ: {gban_status}\n"
                    f"РђРєС‚РёРІРЅС‹Рµ РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёСЏ: {warns}\n"
                    f"Р‘Р»РѕРєРёСЂРѕРІРєР° С‡Р°С‚Р°: {mute_status}\n"
                    f"РќРёРє: {nick}\n"
                    f"Р’СЃРµРіРѕ СЃРѕРѕР±С‰РµРЅРёР№: {messages['count']}\n"
                    f"РџРѕСЃР»РµРґРЅРµРµ СЃРѕРѕР±С‰РµРЅРёРµ: {messages['last']}",
                    disable_mentions=1,
                    keyboard=keyboard
                )
                return True
                           
        if command in ["banid", "Р±Р°РЅС‡Р°С‚Р°"]:
            if await get_role(user_id, chat_id) < 10:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!")
                return True

            if chat_id == 23:
                await message.reply("Р”Р°РЅРЅР°СЏ Р±РµСЃРµРґР° РїСЂРѕРІРѕРґРёС‚СЃСЏ РІ СЃРїРµС†РёР°Р»РёР·РёСЂРѕРІР°РЅРЅРѕРј С‡Р°С‚Рµ, РєРѕС‚РѕСЂС‹Р№ РїСЂРµРґРЅР°Р·РЅР°С‡РµРЅ РёСЃРєР»СЋС‡РёС‚РµР»СЊРЅРѕ РґР»СЏ С‚РµСЃС‚РёСЂРѕРІС‰РёРєРѕРІ Р±РѕС‚Р°.\n\nР’ СЂР°РјРєР°С… РґР°РЅРЅРѕРіРѕ РѕР±СЃСѓР¶РґРµРЅРёСЏ РЅРµ РґРѕРїСѓСЃРєР°РµС‚СЃСЏ РёСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ РєРѕРјР°РЅРґ, РЅРµ РѕС‚РЅРѕСЃСЏС‰РёС…СЃСЏ Рє СЂР°Р±РѕС‚Рµ РїРѕ С‚РµСЃС‚РёСЂРѕРІР°РЅРёСЋ РёР»Рё С„СѓРЅРєС†РёРѕРЅРёСЂРѕРІР°РЅРёСЋ СЃРёСЃС‚РµРјС‹ РІ С†РµР»РѕРј.", disable_mentions=1)
                return True

            if len(arguments) < 2:
                await message.reply("РЈРєР°Р¶РёС‚Рµ С‡Р°С‚!")
                return True

            try:
                target_chat = int(arguments[1])
            except:
                await message.reply("РЈРєР°Р¶РёС‚Рµ С‡Р°С‚!")
                return True

            sql.execute("SELECT chat_id FROM banschats WHERE chat_id = ?", (target_chat,))
            if sql.fetchone():
                await message.reply("Р‘РµСЃРµРґР° СѓР¶Рµ РЅР°С…РѕРґРёС‚СЃСЏ РІ Р±Р»РѕРєРёСЂРѕРІРєРµ!")
                return True

            sql.execute("INSERT INTO banschats (chat_id) VALUES (?)", (target_chat,))
            database.commit()
            
            target_peer = 2000000000 + target_chat
            await bot.api.messages.send(
                peer_id=target_peer,
                random_id=0,
                message=(
                    f"Р’Р»Р°РґРµР»РµС† Р±РµСЃРµРґС‹ вЂ” РЅРµ С‡Р»РµРЅ, СѓР¶Рµ BLACK MANAGER! РЇ РЅРµ Р±СѓРґСѓ Р·РґРµСЃСЊ СЂР°Р±РѕС‚Р°С‚СЊ."
                )
            )

            await message.reply(f"@id{user_id} ({await get_user_name(user_id, chat_id)}) Р·Р°Р±Р»РѕРєРёСЂРѕРІР°Р»(-Р°) Р±РµСЃРµРґСѓ в„–В«{target_chat}В»")
            return True

        if command in ["unbanid", "СЂР°Р·Р±Р°РЅС‡Р°С‚Р°"]:
            if await get_role(user_id, chat_id) < 10:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!")
                return True

            if chat_id == 23:
                await message.reply("Р”Р°РЅРЅР°СЏ Р±РµСЃРµРґР° РїСЂРѕРІРѕРґРёС‚СЃСЏ РІ СЃРїРµС†РёР°Р»РёР·РёСЂРѕРІР°РЅРЅРѕРј С‡Р°С‚Рµ, РєРѕС‚РѕСЂС‹Р№ РїСЂРµРґРЅР°Р·РЅР°С‡РµРЅ РёСЃРєР»СЋС‡РёС‚РµР»СЊРЅРѕ РґР»СЏ С‚РµСЃС‚РёСЂРѕРІС‰РёРєРѕРІ Р±РѕС‚Р°.\n\nР’ СЂР°РјРєР°С… РґР°РЅРЅРѕРіРѕ РѕР±СЃСѓР¶РґРµРЅРёСЏ РЅРµ РґРѕРїСѓСЃРєР°РµС‚СЃСЏ РёСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ РєРѕРјР°РЅРґ, РЅРµ РѕС‚РЅРѕСЃСЏС‰РёС…СЃСЏ Рє СЂР°Р±РѕС‚Рµ РїРѕ С‚РµСЃС‚РёСЂРѕРІР°РЅРёСЋ РёР»Рё С„СѓРЅРєС†РёРѕРЅРёСЂРѕРІР°РЅРёСЋ СЃРёСЃС‚РµРјС‹ РІ С†РµР»РѕРј.", disable_mentions=1)
                return True

            if len(arguments) < 2:
                await message.reply("РЈРєР°Р¶РёС‚Рµ С‡Р°С‚!")
                return True

            try:
                target_chat = int(arguments[-1])
            except:
                await message.reply("РЈРєР°Р¶РёС‚Рµ С‡Р°С‚!")
                return True

            sql.execute("SELECT chat_id FROM banschats WHERE chat_id = ?", (target_chat,))
            if not sql.fetchone():
                await message.reply("Р‘РµСЃРµРґР° Рё С‚Р°Рє РЅР°С…РѕРґРёС‚СЃСЏ РІ Р±Р»РѕРєРёСЂРѕРІРєРµ!")
                return True

            sql.execute("DELETE FROM banschats WHERE chat_id = ?", (target_chat,))
            database.commit()
            
            target_peer = 2000000000 + target_chat
            await bot.api.messages.send(
                peer_id=target_peer,
                random_id=0,
                message=(
                    f"Р§Р°С‚ СЂР°Р·Р±Р»РѕРєРёСЂРѕРІР°РЅ РІ Р±РѕС‚Рµ!"
                )
            )

            await message.reply(f"@id{user_id} ({await get_user_name(user_id, chat_id)}) СЂР°Р·Р±Р»РѕРєРёСЂРѕРІР°Р»(-Р°) Р±РµСЃРµРґСѓ в„–В«{target_chat}В»")
            return True

        if command in ['statstester', 'С‚РµСЃС‚РµСЂСЃС‚Р°С‚Р°', 'С‚РµСЃС‚СЃС‚Р°С‚Р°']:
            # РџСЂРѕРІРµСЂРєР°: РґРѕСЃС‚СѓРїРЅР° С‚РѕР»СЊРєРѕ РІ С‡Р°С‚Рµ С‚РµСЃС‚РµСЂРѕРІ
            if chat_id != 23:
                await message.reply("Р”Р°РЅРЅР°СЏ РєРѕРјР°РЅРґР° РґРѕСЃС‚СѓРїРЅР° С‚РѕР»СЊРєРѕ РІ С‚РµСЃС‚РѕРІРѕРј С‡Р°С‚Рµ!", disable_mentions=1)
                return True

            # РџСЂРѕРІРµСЂРєР° СЂРѕР»Рё вЂ” С‚РѕР»СЊРєРѕ РґР»СЏ С‚РµСЃС‚РµСЂРѕРІ Рё РІС‹С€Рµ
            if await get_role(user_id, chat_id) < 12:
                await message.reply("Р’С‹ РЅРµ СЏРІР»СЏРµС‚РµСЃСЊ С‚РµСЃС‚РёСЂРѕРІС‰РёРєРѕРј Р±РѕС‚Р°!", disable_mentions=1)
                return True

            # РћРїСЂРµРґРµР»СЏРµРј РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ РґР»СЏ РїСЂРѕСЃРјРѕС‚СЂР°
            if message.reply_message:
                target = message.reply_message.from_id
            elif message.fwd_messages and message.fwd_messages[0].from_id > 0:
                target = message.fwd_messages[0].from_id
            else:
                target = user_id

            if target < 0:
                await message.reply("РќРµР»СЊР·СЏ РїРѕР»СѓС‡РёС‚СЊ РёРЅС„РѕСЂРјР°С†РёСЋ Рѕ СЃРѕРѕР±С‰РµСЃС‚РІРµ!", disable_mentions=1)
                return True

            # РџСЂРѕРІРµСЂРєР° СЂРѕР»Рё вЂ” С‚РѕР»СЊРєРѕ РґР»СЏ С‚РµСЃС‚РµСЂРѕРІ Рё РІС‹С€Рµ
            if await get_role(target, chat_id) < 12:
                await message.reply("рџ”№РЈРєР°Р·Р°РЅРЅС‹Р№ РїРѕР»СЊР·РѕРІР°С‚РµР»СЊ РЅРµ С‚РµСЃС‚РёСЂРѕРІС‰РёРє, СЃС‚Р°С‚РёСЃС‚РёРєР° РЅРµРІРѕР·РјРѕР¶РЅР° Рє СЂР°СЃСЃРјРѕС‚СЂРµРЅРёСЋ!", disable_mentions=1)
                return True

            # РџРѕР»СѓС‡Р°РµРј СЂРѕР»СЊ
            role = await get_role(target, chat_id)

            # РџСЂРѕРІРµСЂРєР° РіР»РѕР±Р°Р»СЊРЅРѕРіРѕ Р±Р°РЅР°
            sql.execute("SELECT * FROM gbanlist WHERE user_id = ?", (target,))
            gban = sql.fetchone()
            gban_status = "Р”Р°" if gban else "РќРµС‚"

            # РџРѕР»СѓС‡Р°РµРј РєРѕР»РёС‡РµСЃС‚РІРѕ Р±Р°РіРѕРІ
            sql.execute("SELECT COUNT(*) FROM bugsusers WHERE user_id = ?", (target,))
            bug_count = sql.fetchone()[0] or 0

            # РџРѕР»СѓС‡Р°РµРј РёРјСЏ Рё С„Р°РјРёР»РёСЋ
            try:
                info = await bot.api.users.get(user_ids=target)
                name = f"{info[0].first_name} {info[0].last_name}"
            except:
                name = f"@id{target} (РќРµ СѓРґР°Р»РѕСЃСЊ РїРѕР»СѓС‡РёС‚СЊ РёРјСЏ)"

            # Р’СЃРµ СЂРѕР»Рё
            roles = {
                0: "РџРѕР»СЊР·РѕРІР°С‚РµР»СЊ",
                1: "РњРѕРґРµСЂР°С‚РѕСЂ",
                2: "РЎС‚Р°СЂС€РёР№ РјРѕРґРµСЂР°С‚РѕСЂ",
                3: "РђРґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂ",
                4: "РЎС‚Р°СЂС€РёР№ Р°РґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂ",
                5: "Р—Р°Рј. СЃРїРµС† Р°РґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂР°",
                6: "РЎРїРµС† Р°РґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂ",
                7: "Р’Р»Р°РґРµР»РµС† Р±РµСЃРµРґС‹",
                8: "Р—Р°Рј. СЂСѓРєРѕРІРѕРґРёС‚РµР»СЏ",
                9: "РћСЃРЅРѕРІРЅРѕР№ Р·Р°Рј. СЂСѓРєРѕРІРѕРґРёС‚РµР»СЏ",
                10: "РЎРїРµС†РёР°Р»СЊРЅС‹Р№ СЂСѓРєРѕРІРѕРґРёС‚РµР»СЊ",
                11: "Р Р°Р·СЂР°Р±РѕС‚С‡РёРє Р±РѕС‚Р°",
                12: "рџ‘ѕ РўРµСЃС‚РёСЂРѕРІС‰РёРє Р±РѕС‚Р° рџ‘ѕ",
                13: "рџ‘ѕ Р—Р°Рј. РіР»Р°РІРЅРѕРіРѕ С‚РµСЃС‚РёСЂРѕРІС‰РёРєР° рџ‘ѕ",
                14: "рџ‘ѕ Р“Р»Р°РІРЅС‹Р№ С‚РµСЃС‚РёСЂРѕРІС‰РёРє рџ‘ѕ",
            }

            await message.reply(
                f"рџ‘ѕ РРЅС„РѕСЂРјР°С†РёСЏ Рѕ @id{target} ({name}):\n\n"
                f"рџ”№ Р РѕР»СЊ: {roles.get(role, 'РќРµРёР·РІРµСЃС‚РЅРѕ')}\n"
                f"рџ”№ Р“Р»РѕР±Р°Р»СЊРЅР°СЏ Р±Р»РѕРєРёСЂРѕРІРєР°: {gban_status}\n"
                f"рџ”№ Р’СЃРµРіРѕ РїРѕРґР°РЅРѕ Р±Р°РіРѕРІ: {bug_count}\n\n"
                f"рџ§© Р’С‹ С‚РµСЃС‚РёСЂРѕРІС‰РёРє, СЃРїР°СЃРёР±Рѕ Р·Р° Р±РѕР»СЊС€РѕР№ РІРєР»Р°Рґ РІ СЂР°Р·РІРёС‚РёРµ СЃРёСЃС‚РµРјС‹!",
                disable_mentions=1
            )
            return True            

        # === /bugcommand вЂ” РѕС‚РїСЂР°РІРєР° Р±Р°РіР° ===
        if command in ['bugcommand', 'Р±Р°РіРєРѕРјР°РЅРґР°', 'Р±Р°РіРєРјРґ', 'bugcmd', 'bagcmd']:
            # РџСЂРѕРІРµСЂРєР°, С‡С‚Рѕ РєРѕРјР°РЅРґР° С‚РѕР»СЊРєРѕ РІ С‡Р°С‚Рµ ID 23
            if chat_id != 23:
                await message.reply("Р”Р°РЅРЅР°СЏ РєРѕРјР°РЅРґР° РґРѕСЃС‚СѓРїРЅР° С‚РѕР»СЊРєРѕ РІ РѕС„РёС†РёР°Р»СЊРЅРѕРј С‚РµСЃС‚РѕРІРѕРј С‡Р°С‚Рµ Р±РѕС‚Р°!", disable_mentions=1)
                return True

            # РџСЂРѕРІРµСЂРєР° СЂРѕР»Рё
            if await get_role(user_id, chat_id) < 12:
                await message.reply("Р’С‹ РЅРµ СЏРІР»СЏРµС‚РµСЃСЊ С‚РµСЃС‚РёСЂРѕРІС‰РёРєРѕРј Р±РѕС‚Р°!", disable_mentions=1)
                return True

            # РџСЂРѕРІРµСЂСЏРµРј РЅР°Р»РёС‡РёРµ С‚РµРєСЃС‚Р° Р±Р°РіР°
            bug_text = await get_string(arguments, 1)
            if not bug_text or len(bug_text) < 5:
                await message.reply("вљ пёЏ РЈРєР°Р¶РёС‚Рµ РѕРїРёСЃР°РЅРёРµ Р±Р°РіР° (РјРёРЅРёРјСѓРј 5 СЃРёРјРІРѕР»РѕРІ).", disable_mentions=1)
                return True

            # РџРѕР»СѓС‡Р°РµРј С‚РµРєСѓС‰РµРµ РєРѕР»РёС‡РµСЃС‚РІРѕ Р±Р°РіРѕРІ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ
            sql.execute("SELECT COUNT(*) FROM bugsusers WHERE user_id = ?", (user_id,))
            bug_count = sql.fetchone()[0]

            # Р¤РѕСЂРјРёСЂСѓРµРј РґР°С‚Сѓ/РІСЂРµРјСЏ
            vremya = datetime.now().strftime("%d/%m/%Y %I:%M:%S %p")

            # Р”РѕР±Р°РІР»СЏРµРј Р·Р°РїРёСЃСЊ
            sql.execute("INSERT INTO bugsusers (user_id, bug, datetime, bug_counts_user) VALUES (?, ?, ?, ?)",
                        (user_id, bug_text, vremya, bug_count + 1))
            database.commit()

            # РћС‚РїСЂР°РІР»СЏРµРј СѓРІРµРґРѕРјР»РµРЅРёРµ СЂР°Р·СЂР°Р±РѕС‚С‡РёРєСѓ (РЅР°РїСЂРёРјРµСЂ, id = 123456789)
            dev_id = 860294414  # <-- СЃСЋРґР° РІРїРёС€Рё СЃРІРѕР№ VK ID
            await bot.api.messages.send(
                peer_id=dev_id,
                random_id=0,
                message=f"рџ‘ѕ | РќРѕРІС‹Р№ Р±Р°Рі-СЂРµРїРѕСЂС‚ РєРѕРјР°РЅРґС‹ РѕС‚ @id{user_id} ({await get_user_name(user_id, chat_id)}):\n\n{bug_text}\n\nрџ•’ {vremya}"
            )

            await message.reply(
                f"@id{user_id} ({await get_user_name(user_id, chat_id)}), Р’Р°С€ Р±Р°Рі РїСЂРёРЅСЏС‚!\n\n"
                f"Р’СЂРµРјСЏ РїРѕРґР°С‡Рё Р±Р°РіР°: {vremya}\n"
                f"РЎРѕРґРµСЂР¶Р°РЅРёРµ Р±Р°РіР° вЂ” {bug_text}\n"
                f"Р’С‹ РѕС‚РїСЂР°РІРёР»Рё СѓР¶Рµ вЂ” {bug_count + 1} Р±Р°Рі(РѕРІ).",
                disable_mentions=1
            )
            return True


        # === /buglist вЂ” СЃРїРёСЃРѕРє РІСЃРµС… Р±Р°РіРѕРІ ===
        if command in ['buglist', 'Р±Р°РіР»РёСЃС‚', 'Р±Р°РіРё']:
            if chat_id != 23:
                await message.reply("Р”Р°РЅРЅР°СЏ РєРѕРјР°РЅРґР° РґРѕСЃС‚СѓРїРЅР° С‚РѕР»СЊРєРѕ РІ С‚РµСЃС‚РѕРІРѕРј С‡Р°С‚Рµ!", disable_mentions=1)
                return True

            if await get_role(user_id, chat_id) < 12:
                await message.reply("РЈ РІР°СЃ РЅРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ РґР»СЏ РїСЂРѕСЃРјРѕС‚СЂР° СЃРїРёСЃРєР° Р±Р°РіРѕРІ!", disable_mentions=1)
                return True

            # РџРѕР»СѓС‡Р°РµРј РІСЃРµ Р±Р°РіРё РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ
            sql.execute("SELECT datetime, bug, bug_counts_user FROM bugsusers WHERE user_id = ?", (user_id,))
            user_bugs = sql.fetchall()

            if not user_bugs:
                await message.reply("РЈ РІР°СЃ РїРѕРєР° РЅРµС‚ РїРѕРґРґР°РЅС‹С… Р±Р°РіРѕРІ!", disable_mentions=1)
                return True

            # Р¤РѕСЂРјРёСЂСѓРµРј СЃРїРёСЃРѕРє
            bugs_text = ""
            for i, (vremya, bug, count) in enumerate(user_bugs, start=1):
                bugs_text += f"{i}) Р’СЂРµРјСЏ: {vremya} || Р‘Р°Рі: {bug}\n"

            total_bugs = user_bugs[-1][2]  # Р±РµСЂС‘Рј РїРѕСЃР»РµРґРЅРµРµ Р·РЅР°С‡РµРЅРёРµ СЃС‡С‘С‚С‡РёРєР°

            await message.reply(
                f"вќ— | РЎРїРёСЃРѕРє РІР°С€РёС… РїРѕРґР°РЅРЅС‹С… Р±Р°РіРѕРІ:\n\n{bugs_text}\n\nР’СЃРµРіРѕ Р±Р°РіРѕРІ РїРѕРґР°РЅРѕ: {total_bugs}",
                disable_mentions=1
            )
            return True            
            
        if command in ["clearchat", "СѓРґР°Р»РёС‚СЊС‡Р°С‚"]:
            if await get_role(user_id, chat_id) < 10:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!")
                return True

            if chat_id == 23:
                await message.reply("Р”Р°РЅРЅР°СЏ Р±РµСЃРµРґР° РїСЂРѕРІРѕРґРёС‚СЃСЏ РІ СЃРїРµС†РёР°Р»РёР·РёСЂРѕРІР°РЅРЅРѕРј С‡Р°С‚Рµ, РєРѕС‚РѕСЂС‹Р№ РїСЂРµРґРЅР°Р·РЅР°С‡РµРЅ РёСЃРєР»СЋС‡РёС‚РµР»СЊРЅРѕ РґР»СЏ С‚РµСЃС‚РёСЂРѕРІС‰РёРєРѕРІ Р±РѕС‚Р°.\n\nР’ СЂР°РјРєР°С… РґР°РЅРЅРѕРіРѕ РѕР±СЃСѓР¶РґРµРЅРёСЏ РЅРµ РґРѕРїСѓСЃРєР°РµС‚СЃСЏ РёСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ РєРѕРјР°РЅРґ, РЅРµ РѕС‚РЅРѕСЃСЏС‰РёС…СЃСЏ Рє СЂР°Р±РѕС‚Рµ РїРѕ С‚РµСЃС‚РёСЂРѕРІР°РЅРёСЋ РёР»Рё С„СѓРЅРєС†РёРѕРЅРёСЂРѕРІР°РЅРёСЋ СЃРёСЃС‚РµРјС‹ РІ С†РµР»РѕРј.", disable_mentions=1)
                return True

            if len(arguments) < 2:
                await message.reply("РЈРєР°Р¶РёС‚Рµ С‡Р°С‚!")
                return True

            try:
                target_chat = int(arguments[-1])
            except:
                await message.reply("РЈРєР°Р¶РёС‚Рµ С‡Р°С‚!")
                return True
                
            target_peer = 2000000000 + target_chat
            await bot.api.messages.send(
                peer_id=target_peer,
                random_id=0,
                message=(
                    f"Р§Р°С‚ СѓРґР°Р»РµРЅ РёР· Р±Р°Р·С‹ РґР°РЅРЅС‹С… Р±РѕС‚Р°! Р Р°Р±РѕС‚Р° Р±РѕС‚Р° РІ С‡Р°С‚Рµ РїСЂРµРєСЂР°С‰РµРЅР°."
                )
            )

            sql.execute("DELETE FROM chats WHERE chat_id = ?", (target_chat,))
            database.commit()

            await message.reply(f"@id{user_id} ({await get_user_name(user_id, chat_id)}) СѓРґР°Р»РёР»(-Р°) Р±РµСЃРµРґСѓ в„–В«{target_chat}В»")
            return True
                        
        if command in ['help', 'РїРѕРјРѕС‰СЊ', 'С…РµР»Рї', 'РєРѕРјР°РЅРґС‹', 'commands']:
            commands_levels = {
                0: [
                    'РљРѕРјР°РЅРґС‹ РїРѕР»СЊР·РѕРІР°С‚РµР»РµР№:',
                    '/info -- РѕС„РёС†Р°Р»СЊРЅС‹Рµ СЂРµСЃСѓСЂСЃС‹ РїСЂРѕРµРєС‚Р°',
                    '/РїСЂР°РІРёР»Р° вЂ” РїСЂР°РІРёР»Р° С‡Р°С‚Р° СѓСЃС‚Р°РЅРѕРІР»РµРЅРЅС‹Рµ РІР»Р°РґРµР»СЊС†РµРј Р±РµСЃРµРґС‹',
                    '/РїСЂР°РІРёР»Р°Р±РѕС‚Р° вЂ” РїСЂР°РІРёР»Р° РёСЃРїРѕР»СЊР·РѕРІР°РЅРёСЏ Р±РѕС‚Р°',
                    '/infobot вЂ” РѕС„РёС†Р°Р»СЊРЅС‹Рµ СЂРµСЃСѓСЂСЃС‹ Р±РѕС‚Р°',                    
                    '/stats -- РёРЅС„РѕСЂРјР°С†РёСЏ Рѕ РїРѕР»СЊР·РѕРІР°С‚РµР»Рµ',
                    '/getid -- СѓР·РЅР°С‚СЊ РѕСЂРёРіРёРЅР°Р»СЊРЅС‹Р№ ID РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ РІ Р’Рљ',
                    '/q -- РІС‹С…РѕРґ РёР· С‚РµРєСѓС‰РµР№ Р±РµСЃРµРґС‹',
                    '/other -- РґСЂСѓРіРёРµ РєРѕРјР°РЅРґС‹ (РёРіСЂРѕРІС‹Рµ, СЂРї РєРѕРјР°РЅРґС‹)'
                ],
                1: [
                    '\nРљРѕРјР°РЅРґС‹ РјРѕРґРµСЂР°С‚РѕСЂРѕРІ:',
                    '/setnick вЂ” СЃРјРµРЅРёС‚СЊ РЅРёРє Сѓ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ',
                    '/removenick вЂ” РѕС‡РёСЃС‚РёС‚СЊ РЅРёРє Сѓ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ',
                    '/getnick вЂ” РїСЂРѕРІРµСЂРёС‚СЊ РЅРёРє РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ',
                    '/getacc вЂ” СѓР·РЅР°С‚СЊ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ РїРѕ РЅРёРєСѓ',
                    '/nlist вЂ” РїСЂРѕСЃРјРѕС‚СЂРµС‚СЊ РЅРёРєРё РїРѕР»СЊР·РѕРІР°С‚РµР»РµР№',
                    '/nonick вЂ” РїРѕР»СЊР·РѕРІР°С‚РµР»Рё Р±РµР· РЅРёРєРѕРІ',
                    '/kick вЂ” РёСЃРєР»СЋС‡РёС‚СЊ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ РёР· Р±РµСЃРµРґС‹',
                    '/warn вЂ” РІС‹РґР°С‚СЊ РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёРµ РїРѕР»СЊР·РѕРІР°С‚РµР»СЋ',
                    '/unwarn вЂ” СЃРЅСЏС‚СЊ РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёРµ РїРѕР»СЊР·РѕРІР°С‚РµР»СЋ',
                    '/getwarn вЂ” РёРЅС„РѕСЂРјР°С†РёСЏ Рѕ Р°РєС‚РёРІРЅС‹С… РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёСЏС… РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ',
                    '/warnhistory вЂ” РёРЅС„РѕСЂРјР°С†РёСЏ Рѕ РІСЃРµС… РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёСЏС… РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ',
                    '/warnlist вЂ” СЃРїРёСЃРѕРє РїРѕР»СЊР·РѕРІР°С‚РµР»РµР№ СЃ РІР°СЂРЅРѕРј',
                    '/staff вЂ” РїРѕР»СЊР·РѕРІР°С‚РµР»Рё СЃ СЂРѕР»СЏРјРё',
                    '/mute вЂ” Р·Р°РјСѓС‚РёС‚СЊ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ',
                    '/unmute вЂ” СЂР°Р·РјСѓС‚РёС‚СЊ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ',
                    '/alt вЂ” СѓР·РЅР°С‚СЊ Р°Р»СЊС‚РµСЂРЅР°С‚РёРІРЅС‹Рµ РєРѕРјР°РЅРґС‹',
                    '/getmute -- РёРЅС„РѕСЂРјР°С†РёСЏ Рѕ РјСѓС‚Рµ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ',
                    '/mutelist -- СЃРїРёСЃРѕРє РїРѕР»СЊР·РѕРІР°С‚РµР»РµР№ СЃ РјСѓС‚РѕРј',
                    '/clear -- РѕС‡РёСЃС‚РёС‚СЊ СЃРѕРѕР±С‰РµРЅРёСЏ',
                    '/getban -- РёРЅС„РѕСЂРјР°С†РёСЏ Рѕ Р±Р°РЅР°С… РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ',
                    '/delete -- СѓРґР°Р»РёС‚СЊ СЃРѕРѕР±С‰РµРЅРёРµ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ',
                    '/chatid -- СѓР·РЅР°С‚СЊ РѕСЂРёРіРёРЅР°Р»СЊРЅС‹Р№ Р°Р№РґРё С‡Р°С‚Р° РІ Р±РѕС‚Рµ'                    
                ],
                2: [
                    '\nРљРѕРјР°РЅРґС‹ СЃС‚Р°СЂС€РёС… РјРѕРґРµСЂР°С‚РѕСЂРѕРІ:',
                    '/ban вЂ” Р·Р°Р±Р»РѕРєРёСЂРѕРІР°С‚СЊ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ РІ Р±РµСЃРµРґРµ',
                    '/unban -- СЂР°Р·Р±Р»РѕРєРёСЂРѕРІР°С‚СЊ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ РІ Р±РµСЃРµРґРµ',
                    '/addmoder -- РІС‹РґР°С‚СЊ РїРѕР»СЊР·РѕРІР°С‚РµР»СЋ РјРѕРґРµСЂР°С‚РѕСЂР°',
                    '/removerole -- Р·Р°Р±СЂР°С‚СЊ СЂРѕР»СЊ Сѓ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ',
                    '/zov -- СѓРїРѕРјСЏРЅСѓС‚СЊ РІСЃРµС… РїРѕР»СЊР·РѕРІР°С‚РµР»РµР№',
                    '/online -- СѓРїРѕРјСЏРЅСѓС‚СЊ РїРѕР»СЊР·РѕРІР°С‚РµР»РµР№ РѕРЅР»Р°Р№РЅ',
                    '/onlinelist вЂ” РїРѕСЃРјРѕС‚СЂРµС‚СЊ РїРѕР»СЊР·РѕРІР°С‚РµР»РµР№ РІ РѕРЅР»Р°Р№РЅ',
                    '/banlist -- РїРѕСЃРјРѕС‚СЂРµС‚СЊ Р·Р°Р±Р»РѕРєРёСЂРѕРІР°РЅРЅС‹С…',
                    '/inactivelist -- СЃРїРёСЃРѕРє РЅРµР°РєС‚РёРІРЅС‹С… РїРѕР»СЊР·РѕРІР°С‚РµР»РµР№',
                    '/masskick -- РёСЃРєР»СЋС‡РёС‚СЊ РЅРµСЃРєРѕР»СЊРєРёС… РїРѕР»СЊР·РѕРІР°С‚РµР»РµР№'
                ],
                3: [
                    '\nРЎРїРёСЃРѕРє РєРѕРјР°РЅРґ Р°РґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂРѕРІ:',
                    '/quiet -- Р’РєР»СЋС‡РёС‚СЊ РІС‹РєР»СЋС‡РёС‚СЊ СЂРµР¶РёРј С‚РёС€РёРЅС‹',
                    '/skick -- РёСЃРєР»СЋС‡РёС‚СЊ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ СЃ Р±РµСЃРµРґ СЃРµС‚РєРё',
                    '/sban -- Р·Р°Р±Р»РѕРєРёСЂРѕРІР°С‚СЊ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ РІ СЃРµС‚РєРµ Р±РµСЃРµРґ',
                    '/sunban вЂ” СЂР°Р·Р±Р°РЅРёС‚СЊ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ РІ СЃРµС‚РєРµ Р±РµСЃРµРґ',
                    '/addsenmoder вЂ” РІС‹РґР°С‚СЊ РїСЂР°РІР° СЃС‚Р°СЂС€РµРіРѕ РјРѕРґРµСЂР°С‚РѕСЂР°',
                    '/rnickall -- РѕС‡РёСЃС‚РёС‚СЊ РІСЃРµ РЅРёРєРё РІ Р±РµСЃРµРґРµ',
                    '/sremovenick -- РѕС‡РёСЃС‚РёС‚СЊ РЅРёРє Сѓ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ РІ СЃРµС‚РєРµ Р±РµСЃРµРґ',
                    '/szov -- РІС‹Р·РѕРІ СѓС‡Р°СЃС‚РЅРёРєРѕРІ Р±РµСЃРµРґ СЃРµС‚РєРё',
                    '/srole -- РІС‹РґР°С‚СЊ РїСЂР°РІР° РІ СЃРµС‚РєРµ Р±РµСЃРµРґ'
                ],
                4: [
                    '\nРЎРїРёСЃРѕРє РєРѕРјР°РЅРґ СЃС‚Р°СЂС€РёС… Р°РґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂРѕРІ:',
                    '/addadmin -- РІС‹РґР°С‚СЊ РїСЂР°РІР° Р°РґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂР°',
                    '/serverinfo -- РёРЅС„РѕСЂРјР°С†РёСЏ Рѕ СЃРµСЂРІРµСЂРµ',
                    '/filter -- С„РёР»СЊС‚СЂ Р·Р°РїСЂРµС‰РµРЅРЅС‹С… СЃР»РѕРІ',
                    '/sremoverole -- Р·Р°Р±СЂР°С‚СЊ СЂРѕР»СЊ Сѓ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ РІ СЃРµС‚РєРµ Р±РµСЃРµРґ',
                    '/ssetnick -- СѓСЃС‚Р°РЅРѕРІРёС‚СЊ РЅРёРє РІ СЃРµС‚РєРµ Р±РµСЃРµРґ',
                    '/bug -- РѕС‚РїСЂР°РІРёС‚СЊ Р±Р°Рі-С‚СЂРµРєРµСЂ СЂР°Р·СЂР°Р±РѕС‚С‡РёРєСѓ Р±РѕС‚Р°',
                    '/report -- Р¶Р°Р»РѕР±Р° РЅР° РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ'                   
                ],
                5: [
                    '\nРЎРїРёСЃРѕРє РєРѕРјР°РЅРґ Р·Р°Рј. СЃРїРµС† Р°РґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂР°:',
                    '/addsenadmin -- РІС‹РґР°С‚СЊ РїСЂР°РІР° СЃС‚Р°СЂС€РµРіРѕ Р°РґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂР°',
                    '/sync -- СЃРёРЅС…СЂРѕРЅРёР·Р°С†РёСЏ СЃ Р±Р°Р·РѕР№ РґР°РЅРЅС‹С…',
                    '/pin -- Р·Р°РєСЂРµРїРёС‚СЊ СЃРѕРѕР±С‰РµРЅРёРµ',
                    '/unpin -- РѕС‚РєСЂРµРїРёС‚СЊ СЃРѕРѕР±С‰РµРЅРёРµ',
                    '/deleteall -- СѓРґР°Р»РёС‚СЊ РїРѕСЃР»РµРґРЅРёРµ 200 СЃРѕРѕР±С‰РµРЅРёР№ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ',
                    '/gsinfo -- РёРЅС„РѕСЂРјР°С†РёСЏ Рѕ РіР»РѕР±Р°Р»СЊРЅРѕР№ РїСЂРёРІСЏР·РєРµ',
                    '/gsrnick -- РѕС‡РёСЃС‚РёС‚СЊ РЅРёРє Сѓ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ РІ Р±РµСЃРµРґР°С… РїСЂРёРІСЏР·РєРё',
                    '/gssnick -- РїРѕСЃС‚Р°РІРёС‚СЊ РЅРёРє РїРѕР»СЊР·РѕРІР°С‚РµР»СЋ РІ Р±РµСЃРµРґР°С… РїСЂРёРІСЏР·РєРё',
                    '/gskick -- РёСЃРєР»СЋС‡РёС‚СЊ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ СЃ Р±РµСЃРµРґ РїСЂРёРІСЏР·РєРё',
                    '/gsban -- Р·Р°Р±Р»РѕРєРёСЂРѕРІР°С‚СЊ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ РІ Р±РµСЃРµРґР°С… РїСЂРёРІСЏР·РєРё',
                    '/gsunban -- СЂР°Р·Р±Р°РЅРёС‚СЊ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ РІ Р±РµСЃРµРґР°С… РїСЂРёРІСЏР·РєРё'                    
                ],                
                6: [
                    '\nРЎРїРёСЃРѕРє РєРѕРјР°РЅРґ СЃРїРµС†. Р°РґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂР°:',
                    '/addzsa -- РІС‹РґР°С‚СЊ РїСЂР°РІР° Р·Р°Рј. СЃРїРµС†. Р°РґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂР°',
                    '/server -- РїСЂРёРІСЏР·Р°С‚СЊ Р±РµСЃРµРґСѓ Рє СЃРµСЂРІРµСЂСѓ',
                    '/settings -- РїРѕРєР°Р·Р°С‚СЊ РЅР°СЃС‚СЂРѕР№РєРё Р±РµСЃРµРґС‹',
                    '/clearwarn -- СЃРЅСЏС‚СЊ РІР°СЂРЅС‹ РІСЃРµРј РїРѕР»СЊР·РѕРІР°С‚РµР»СЏРј',
                    '/title -- РёР·РјРµРЅРёС‚СЊ РЅР°Р·РІР°РЅРёРµ Р±РµСЃРµРґС‹',
                    '/antisliv -- РІРєР»СЋС‡РёС‚СЊ СЃРёСЃС‚РµРјСѓ Р°РЅС‚РёСЃР»РёРІР° РІ Р±РµСЃРµРґРµ'
                ],                
                7: [
                    '\nРЎРїРёСЃРѕРє РєРѕРјР°РЅРґ РІР»Р°РґРµР»СЊС†Р° Р±РµСЃРµРґС‹:',
                    '/addsa -- РІС‹РґР°С‚СЊ РїСЂР°РІР° СЃРїРµС†. Р°РґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂР°',
                    '/antiflood -- СЂРµР¶РёРј Р·Р°С‰РёС‚С‹ РѕС‚ СЃРїР°РјР°',
                    '/welcometext -- С‚РµРєСЃС‚ РїСЂРёРІРµС‚СЃС‚РІРёСЏ',
                    '/invite -- СЃРёСЃС‚РµРјР° РґРѕР±Р°РІР»РµРЅРёСЏ РїРѕР»СЊР·РѕРІР°С‚РµР»РµР№ С‚РѕР»СЊРєРѕ РјРѕРґРµСЂР°С‚РѕСЂР°РјРё',
                    '/leave -- СЃРёСЃС‚РµРјР° РёСЃРєР»СЋС‡РµРЅРёСЏ РїРѕР»СЊР·РѕРІР°С‚РµР»РµР№ РїСЂРё РІС‹С…РѕРґРµ',
                    '/editowner -- РїРµСЂРµРґР°С‚СЊ РїСЂР°РІР° РІР»Р°РґРµР»СЊС†Р° Р±РµСЃРµРґС‹',
                    '/masskick -- РёСЃРєР»СЋС‡РёС‚СЊ СѓС‡Р°СЃС‚РЅРёРєРѕРІ Р±РµР· СЂРѕР»РµР№',
                    '/Р·Р°С‰РёС‚Р° -- Р·Р°С‰РёС‚Р° РѕС‚ СЃС‚РѕСЂРѕРЅРЅРёС… СЃРѕРѕР±С‰РµСЃС‚РІ',
                    '/settingsmute -- РІРєР»СЋС‡РёС‚СЊ РІС‹РґР°С‡Сѓ РІР°СЂРЅРѕРІ Р·Р° РЅР°РїРёСЃР°РЅРёРµ СЃРѕРѕР±С‰РµРЅРёР№ РІ РјСѓС‚Рµ',
                    '/setinfo -- СѓСЃС‚Р°РЅРѕРІРёС‚СЊ РёРЅС„РѕСЂРјР°С†РёСЋ Рѕ РѕС„РёС†РёР°Р»СЊРЅС‹С… СЂРµСЃСѓСЂСЃР°С… РїСЂРѕРµРєС‚Р° РІ В«/infoВ»',
                    '/setrules -- СѓСЃС‚Р°РЅРѕРІРёС‚СЊ РїСЂР°РІРёР»Р° Р±РµСЃРµРґС‹ РІ В«/rulesВ»',
                    '/type вЂ“ РёР·РјРµРЅРёС‚СЊ С‚РёРї Р±РµСЃРµРґС‹',
                    '/gsync -- РїРѕСЃС‚Р°РІРёС‚СЊ РіР»РѕР±Р°Р»СЊРЅСѓСЋ СЃРёРЅС…СЂРѕРЅРёР·Р°С†РёСЋ Р±РµСЃРµРґ',
                    '/gunsync вЂ“ РѕС‚РєР»СЋС‡РёС‚СЊ РіР»РѕР±Р°Р»СЊРЅСѓСЋ СЃРёРЅС…СЂРѕРЅРёР·Р°С†РёСЋ Р±РµСЃРµРґ'                   
                ]               
            }

            user_role = await get_role(user_id, chat_id)

            if user_role > 1:
                keyboard = (
                    Keyboard(inline=True)
                    .add(Callback("РђР»СЊС‚РµСЂРЅР°С‚РёРІРЅС‹Рµ РєРѕРјР°РЅРґС‹", {"command": "alt", "chatId": chat_id}), color=KeyboardButtonColor.PRIMARY)
                )
            else:
                keyboard = None

            commands = []
            for i in commands_levels.keys():
                if i <= user_role:
                    for b in commands_levels[i]:
                        commands.append(b)

            level_commands = '\n'.join(commands)

            await message.reply(f"{level_commands}", disable_mentions=1, keyboard=keyboard)
            await chats_log(user_id=user_id, target_id=None, role=None, log=f"РїРѕСЃРјРѕС‚СЂРµР»(-Р°) СЃРїРёСЃРѕРє РґРѕСЃС‚СѓРїРЅС‹С… РєРѕРјР°РЅРґ")            

        if command in ['snick', 'setnick', 'nick', 'addnick', 'РЅРёРє', 'СЃРµС‚РЅРёРє', 'Р°РґРґРЅРёРє']:
            if await get_role(user_id, chat_id) < 1:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!")
                return True

            user = int
            arg = 0
            if message.reply_message:
                user = message.reply_message.from_id
                arg = 1
            elif message.fwd_messages and message.fwd_messages[0].from_id > 0:
                user = message.fwd_messages[0].from_id
                arg = 1
            elif len(arguments) >= 2 and await getID(arguments[1]):
                user = await getID(arguments[1])
                arg = 2
            else:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ!", disable_mentions=1)
                return True

            if await equals_roles(user_id, user, chat_id, message) == 0:
                await message.reply("Р’С‹ РЅРµ РјРѕР¶РµС‚Рµ СѓСЃС‚Р°РЅРѕРІРёС‚СЊ РЅРёРє РґР°РЅРЅРѕРјСѓ РїРѕР»СЊР·РѕРІР°С‚РµР»СЋ!", disable_mentions=1)
                return True

            new_nick = await get_string(arguments, arg)
            if not new_nick:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РЅРёРє РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ!", disable_mentions=1)
                return True
            else: await setnick(user, chat_id, new_nick)

            await message.reply(f"@id{user_id} ({await get_user_name(user_id, chat_id)}) СѓСЃС‚Р°РЅРѕРІРёР» РЅРѕРІРѕРµ РёРјСЏ @id{user} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЋ)!\nРќРѕРІС‹Р№ РЅРёРє: {new_nick}", disable_mentions=1)
            await chats_log(user_id=user_id, target_id=user, role=None, log=f"СѓСЃС‚Р°РЅРѕРІРёР»(-Р°) РЅРѕРІС‹Р№ РЅРёРє @id{user} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЋ). РќРѕРІС‹Р№ РЅРёРє: {new_nick}")                       

        if command in ['rnick', 'removenick', 'clearnick', 'cnick', 'СЂРЅРёРє', 'СѓРґР°Р»РёС‚СЊРЅРёРє', 'СЃРЅСЏС‚СЊРЅРёРє']:
            if await get_role(user_id, chat_id) < 1:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return True

            user = int
            if message.reply_message: user = message.reply_message.from_id
            elif message.fwd_messages and message.fwd_messages[0].from_id > 0:
                user = message.fwd_messages[0].from_id
            elif len(arguments) >= 2 and await getID(arguments[1]): user = await getID(arguments[1])
            else:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ!", disable_mentions=1)
                return True

            if await equals_roles(user_id, user, chat_id, message) == 0:
                await message.reply("Р’С‹ РЅРµ РјРѕР¶РµС‚Рµ СѓРґР°Р»РёС‚СЊ РЅРёРє РґР°РЅРЅРѕРјСѓ РїРѕР»СЊР·РѕРІР°С‚РµР»СЋ!", disable_mentions=1)
                return True

            await rnick(user, chat_id)
            await message.reply(f"@id{user_id} ({await get_user_name(user_id, chat_id)}) СѓР±СЂР°Р»(-Р°) РЅРёРє Сѓ @id{user} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ)!", disable_mentions=1)
            await chats_log(user_id=user_id, target_id=user, role=None, log=f"СѓРґР°Р»РёР»(-Р°) СЃС‚Р°СЂС‹Р№ РЅРёРє @id{user} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЋ)")            

        if command in ['type', 'С‚РёРї']:
            if await get_role(user_id, chat_id) < 7:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return True

            # РїРѕР»СѓС‡Р°РµРј Р°СЂРіСѓРјРµРЅС‚ (РЅРѕРІС‹Р№ С‚РёРї)
            if len(arguments) < 2:
                # С‚РёРї РЅРµ СѓРєР°Р·Р°РЅ, РїРѕРєР°Р·С‹РІР°РµРј С‚РµРєСѓС‰РёР№ С‚РёРї
                sql.execute(f"SELECT type FROM chats WHERE chat_id = {chat_id}")
                current_type = sql.fetchone()
                if current_type:
                    type_value = current_type[0]
                    await message.reply(
                        f"Р‘РµСЃРµРґР° РёРјРµРµС‚ С‚РёРї: {chat_types.get(type_value, type_value)}\n\n"
                        "Р’СЃРµ С‚РёРїС‹ Р±РµСЃРµРґ:\n" +
                        "\n".join([f"{k} -- {v}" for k, v in chat_types.items()]),
                        disable_mentions=1
                    )
                return True

            new_type = arguments[1].lower()

            # РїСЂРѕРІРµСЂРєР° РЅР° РІР°Р»РёРґРЅРѕСЃС‚СЊ
            if new_type not in chat_types:
                await message.reply(
                    "РќРµРІРµСЂРЅС‹Р№ С‚РёРї Р±РµСЃРµРґС‹, С‚РёРїС‹:\n" +
                    "\n".join([f"{k} -- {v}" for k, v in chat_types.items()]),
                    disable_mentions=1
                )
                return True

            # СѓСЃС‚Р°РЅР°РІР»РёРІР°РµРј РЅРѕРІС‹Р№ С‚РёРї
            sql.execute(f"UPDATE chats SET type = ? WHERE chat_id = ?", (new_type, chat_id))
            database.commit()

            await message.reply(f"Р’С‹ СѓСЃС‚Р°РЅРѕРІРёР»Рё С‚РёРї Р±РµСЃРµРґС‹: {chat_types[new_type]}", disable_mentions=1)
            await chats_log(user_id=user_id, target_id=None, role=None, log=f"СѓСЃС‚Р°РЅРѕРІРёР»(-Р°) РЅРѕРІС‹Р№ С‚РёРї Р±РµСЃРµРґС‹. РќРѕРІС‹Р№ С‚РёРї: {chat_types[new_type]}")            
            
        if command in ["settings", "РЅР°СЃС‚СЂРѕР№РєРё"]:
            if await get_role(user_id, chat_id) < 6:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return

            # РџРѕР»СѓС‡Р°РµРј РІР»Р°РґРµР»СЊС†Р° С‡Р°С‚Р° С‡РµСЂРµР· VK API
            x = await bot.api.messages.get_conversations_by_id(
                peer_ids=peer_id,
                extended=1,
                fields='chat_settings',
                group_id=message.group_id
            )
            x = json.loads(x.json())
            chat_owner = None
            chat_title = None
            for i in x['items']:
                chat_owner = int(i["chat_settings"]["owner_id"])
                chat_title = i["chat_settings"]["title"]

            # РџРѕР»СѓС‡Р°РµРј РґР°РЅРЅС‹Рµ РёР· Р±Р°Р·С‹ РїРѕ chat_id
            sql.execute(f"SELECT type, in_pull, filter, leave_kick, invite_kick, antiflood FROM chats WHERE chat_id = {chat_id}")
            row = sql.fetchone()
            if row:
                type_value = chat_types.get(row[0], row[0])
                server = await get_current_server(chat_id)
                filter_text = "Р’РєР»СЋС‡РµРЅРѕ" if row[2] == 1 else "Р’С‹РєР»СЋС‡РµРЅРѕ"
                leave_text = "Р’РєР»СЋС‡РµРЅРѕ" if row[3] == 1 else "Р’С‹РєР»СЋС‡РµРЅРѕ"
                invite_text = "Р’РєР»СЋС‡РµРЅРѕ" if row[4] == 1 else "Р’С‹РєР»СЋС‡РµРЅРѕ"
                antiflood_text = "Р’РєР»СЋС‡РµРЅРѕ" if row[5] == 1 else "Р’С‹РєР»СЋС‡РµРЅРѕ"
            else:
                type_value = "РћР±С‰РёРµ Р±РµСЃРµРґС‹"
                server = "0"
                filter_text = "Р’С‹РєР»СЋС‡РµРЅРѕ"
                leave_text = "Р’С‹РєР»СЋС‡РµРЅРѕ"
                invite_text = "Р’С‹РєР»СЋС‡РµРЅРѕ"
                antiflood_text = "Р’С‹РєР»СЋС‡РµРЅРѕ"

            await chats_log(user_id=user_id, target_id=None, role=None, log=f"РїРѕСЃРјРѕС‚СЂРµР»(-Р°) С‚РµРєСѓС‰РёРµ РЅР°СЃС‚СЂРѕР№РєРё Р±РµСЃРµРґС‹")            
            await message.reply(
                f"РќР°Р·РІР°РЅРёРµ С‡Р°С‚Р°: {chat_title}\n"
                f"Р’Р»Р°РґРµР»РµС† С‡Р°С‚Р°: @id{chat_owner} ({await get_user_name(chat_owner, chat_id)})\n"
                f"РўРёРї Р±РµСЃРµРґС‹: {type_value}\n"
                f"РЎРµСЂРІРµСЂ: {server}\n"
                f"ID С‡Р°С‚Р°: {chat_id}\n"
                f"Р¤РёР»СЊС‚СЂ: {filter_text}\n"
                f"РСЃРєР»СЋС‡РµРЅРёРµ РїСЂРё РІС‹С…РѕРґРµ: {leave_text}\n"
                f"РџСЂРёРіР»Р°С€РµРЅРёРµ РѕС‚ РјРѕРґРµСЂР°С‚РѕСЂР° +: {invite_text}\n"
                f"РђРЅС‚Рё-С„Р»СѓРґ: {antiflood_text}"
            )
            return            

        if command in ['gsrnick', 'РіСЂРЅРёРє']:
            if await get_role(user_id, chat_id) < 5:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return True

            gsync_chats = await get_gsync_chats(chat_id)
            if not gsync_chats:
                await message.reply("Р‘РµСЃРµРґР° РЅРµ РїСЂРёРІСЏР·Р°РЅР° Рє РіР»РѕР±Р°Р»СЊРЅРѕР№ СЃРІСЏР·РєРµ!", disable_mentions=1)
                return True

            user = int
            if message.reply_message:
                user = message.reply_message.from_id
            elif message.fwd_messages and message.fwd_messages[0].from_id > 0:
                user = message.fwd_messages[0].from_id
            elif len(arguments) >= 2 and await getID(arguments[1]):
                user = await getID(arguments[1])
            else:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ!", disable_mentions=1)
                return True

            if await equals_roles(user_id, user, chat_id, message) == 0:
                await message.reply("Р’С‹ РЅРµ РјРѕР¶РµС‚Рµ СЃРЅСЏС‚СЊ РЅРёРє Сѓ РґР°РЅРЅРѕРіРѕ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ!", disable_mentions=1)
                return True

            for i in gsync_chats:
                try:
                    await rnick(user, i)
                except:
                    continue

            await message.reply(f"@id{user_id} ({await get_user_name(user_id, chat_id)}) СѓР±СЂР°Р» РЅРёРє Сѓ @id{user} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ) РІРѕ РІСЃРµС… Р±РµСЃРµРґР°С… РіР»РѕР±Р°Р»СЊРЅРѕР№ СЃРІСЏР·РєРё.", disable_mentions=1)
            await chats_log(user_id=user_id, target_id=user, role=None, log=f"СЃРЅСЏР» РЅРёРє @id{user} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЋ) РІРѕ РІСЃРµС… Р±РµСЃРµРґР°С… РіР»РѕР±Р°Р»СЊРЅРѕР№ СЃРІСЏР·РєРё")
            return True
            
        if command in ['gssnick', 'РіСЃСЃРЅРёРє']:
            if await get_role(user_id, chat_id) < 5:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return True

            gsync_chats = await get_gsync_chats(chat_id)
            if not gsync_chats:
                await message.reply("Р‘РµСЃРµРґР° РЅРµ РїСЂРёРІСЏР·Р°РЅР° Рє РіР»РѕР±Р°Р»СЊРЅРѕР№ СЃРІСЏР·РєРµ!", disable_mentions=1)
                return True

            user = int
            arg = 0
            if message.reply_message:
                user = message.reply_message.from_id
                arg = 1
            elif message.fwd_messages and message.fwd_messages[0].from_id > 0:
                user = message.fwd_messages[0].from_id
                arg = 1
            elif len(arguments) >= 2 and await getID(arguments[1]):
                user = await getID(arguments[1])
                arg = 2
            else:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ!", disable_mentions=1)
                return True

            if await equals_roles(user_id, user, chat_id, message) == 0:
                await message.reply("Р’С‹ РЅРµ РјРѕР¶РµС‚Рµ СѓСЃС‚Р°РЅРѕРІРёС‚СЊ РЅРёРє РґР°РЅРЅРѕРјСѓ РїРѕР»СЊР·РѕРІР°С‚РµР»СЋ!", disable_mentions=1)
                return True

            new_nick = await get_string(arguments, arg)
            if not new_nick:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РЅРёРє!", disable_mentions=1)
                return True

            for i in gsync_chats:
                try:
                    await setnick(user, i, new_nick)
                except:
                    continue

            await message.reply(f"@id{user_id} ({await get_user_name(user_id, chat_id)}) СѓСЃС‚Р°РЅРѕРІРёР» РЅРёРє @id{user} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЋ) РІРѕ РІСЃРµС… Р±РµСЃРµРґР°С… РіР»РѕР±Р°Р»СЊРЅРѕР№ СЃРІСЏР·РєРё.\nРќРѕРІС‹Р№ РЅРёРє: {new_nick}", disable_mentions=1)
            await chats_log(user_id=user_id, target_id=user, role=None, log=f"СѓСЃС‚Р°РЅРѕРІРёР» РЅРёРє {new_nick} @id{user} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЋ) РІРѕ РІСЃРµС… Р±РµСЃРµРґР°С… РіР»РѕР±Р°Р»СЊРЅРѕР№ СЃРІСЏР·РєРё")
            return True

        if command in ['gskick', 'РіСЃРєРёРє']:
            if await get_role(user_id, chat_id) < 5:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return True

            gsync_chats = await get_gsync_chats(chat_id)
            if not gsync_chats:
                await message.reply("Р‘РµСЃРµРґР° РЅРµ РїСЂРёРІСЏР·Р°РЅР° Рє РіР»РѕР±Р°Р»СЊРЅРѕР№ СЃРІСЏР·РєРµ!", disable_mentions=1)
                return True

            user = int
            reason = None
            if message.reply_message:
                user = message.reply_message.from_id
            elif message.fwd_messages and message.fwd_messages[0].from_id > 0:
                user = message.fwd_messages[0].from_id
            elif len(arguments) >= 2 and await getID(arguments[1]):
                user = await getID(arguments[1])
            else:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ!", disable_mentions=1)
                return True

            if await equals_roles(user_id, user, chat_id, message) < 2:
                await message.reply("Р’С‹ РЅРµ РјРѕР¶РµС‚Рµ РёСЃРєР»СЋС‡РёС‚СЊ РґР°РЅРЅРѕРіРѕ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ!", disable_mentions=1)
                return True

            for i in gsync_chats:
                try:
                    await bot.api.messages.remove_chat_user(i, user)
                    msg = f"@id{user_id} ({await get_user_name(user_id, chat_id)}) РёСЃРєР»СЋС‡РёР» @id{user} ({await get_user_name(user, chat_id)}) РІ Р±РµСЃРµРґР°С… РіР»РѕР±Р°Р»СЊРЅРѕР№ СЃРІСЏР·РєРё!"
                    if reason:
                        msg += f"\nРџСЂРёС‡РёРЅР°: {reason}"
                    await bot.api.messages.send(peer_id=2000000000 + i, message=msg, disable_mentions=1, random_id=0)
                except:
                    continue

            await message.reply(f"@id{user_id} ({await get_user_name(user_id, chat_id)}) РёСЃРєР»СЋС‡РёР» @id{user} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ) РёР· РІСЃРµС… Р±РµСЃРµРґ РіР»РѕР±Р°Р»СЊРЅРѕР№ СЃРІСЏР·РєРё.", disable_mentions=1)
            await chats_log(user_id=user_id, target_id=user, role=None, log=f"РёСЃРєР»СЋС‡РёР» @id{user} РёР· РІСЃРµС… Р±РµСЃРµРґ РіР»РѕР±Р°Р»СЊРЅРѕР№ СЃРІСЏР·РєРё")
            return True

        if command in ['gsban', 'РіСЃР±Р°РЅ']:
            if await get_role(user_id, chat_id) < 5:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return True

            gsync_chats = await get_gsync_chats(chat_id)
            if not gsync_chats:
                await message.reply("Р‘РµСЃРµРґР° РЅРµ РїСЂРёРІСЏР·Р°РЅР° Рє РіР»РѕР±Р°Р»СЊРЅРѕР№ СЃРІСЏР·РєРµ!", disable_mentions=1)
                return True

            user = int
            arg = 0
            if message.reply_message:
                user = message.reply_message.from_id
                arg = 1
            elif message.fwd_messages and message.fwd_messages[0].from_id > 0:
                user = message.fwd_messages[0].from_id
                arg = 1
            elif len(arguments) >= 2 and await getID(arguments[1]):
                user = await getID(arguments[1])
                arg = 2
            else:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ!", disable_mentions=1)
                return True

            if await equals_roles(user_id, user, chat_id, message) < 2:
                await message.reply("Р’С‹ РЅРµ РјРѕР¶РµС‚Рµ Р·Р°Р±Р»РѕРєРёСЂРѕРІР°С‚СЊ РґР°РЅРЅРѕРіРѕ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ!", disable_mentions=1)
                return True

            reason = await get_string(arguments, arg)
            if not reason:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РїСЂРёС‡РёРЅСѓ Р±Р»РѕРєРёСЂРѕРІРєРё!", disable_mentions=1)
                return True

            for i in gsync_chats:
                try:
                    await ban(user, user_id, i, reason)
                    await bot.api.messages.remove_chat_user(i, user)
                    msg = f"@id{user_id} ({await get_user_name(user_id, chat_id)}) РёСЃРєР»СЋС‡РёР» @id{user} ({await get_user_name(user, chat_id)}) РІ Р±РµСЃРµРґР°С… РіР»РѕР±Р°Р»СЊРЅРѕР№ СЃРІСЏР·РєРё!"
                    if reason:
                        msg += f"\nРџСЂРёС‡РёРЅР°: {reason}"
                    await bot.api.messages.send(peer_id=2000000000 + i, message=msg, disable_mentions=1, random_id=0)
                except:
                    continue

            await message.reply(f"@id{user_id} ({await get_user_name(user_id, chat_id)}) Р·Р°Р±Р»РѕРєРёСЂРѕРІР°Р» @id{user} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ) РІРѕ РІСЃРµС… Р±РµСЃРµРґР°С… РіР»РѕР±Р°Р»СЊРЅРѕР№ СЃРІСЏР·РєРё.\nРџСЂРёС‡РёРЅР°: {reason}", disable_mentions=1)
            await chats_log(user_id=user_id, target_id=user, role=None, log=f"Р·Р°Р±Р»РѕРєРёСЂРѕРІР°Р» @id{user} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ) РІРѕ РІСЃРµС… Р±РµСЃРµРґР°С… РіР»РѕР±Р°Р»СЊРЅРѕР№ СЃРІСЏР·РєРё. РџСЂРёС‡РёРЅР°: {reason}")
            return True            
            
        if command in ['gsunban', 'РіСЃСѓРЅР±Р°РЅ']:
            if await get_role(user_id, chat_id) < 5:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return True

            gsync_chats = await get_gsync_chats(chat_id)
            if not gsync_chats:
                await message.reply("Р‘РµСЃРµРґР° РЅРµ РїСЂРёРІСЏР·Р°РЅР° Рє РіР»РѕР±Р°Р»СЊРЅРѕР№ СЃРІСЏР·РєРµ!", disable_mentions=1)
                return True

            user = int
            if message.reply_message:
                user = message.reply_message.from_id
            elif message.fwd_messages and message.fwd_messages[0].from_id > 0:
                user = message.fwd_messages[0].from_id
            elif len(arguments) >= 2 and await getID(arguments[1]):
                user = await getID(arguments[1])
            else:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ!", disable_mentions=1)
                return True

            if await equals_roles(user_id, user, chat_id, message) == 0:
                await message.reply("Р’С‹ РЅРµ РјРѕР¶РµС‚Рµ СЂР°Р·Р±Р°РЅРёС‚СЊ РґР°РЅРЅРѕРіРѕ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ!", disable_mentions=1)
                return True

            for i in gsync_chats:
                try:
                    await unban(user, i)
                except:
                    continue

            await message.reply(f"@id{user_id} ({await get_user_name(user_id, chat_id)}) СЃРЅСЏР» Р±Р»РѕРєРёСЂРѕРІРєСѓ СЃ @id{user} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ) РІРѕ РІСЃРµС… Р±РµСЃРµРґР°С… РіР»РѕР±Р°Р»СЊРЅРѕР№ СЃРІСЏР·РєРё.", disable_mentions=1)
            await chats_log(user_id=user_id, target_id=user, role=None, log=f"СЂР°Р·Р±Р»РѕРєРёСЂРѕРІР°Р» @id{user} РІРѕ РІСЃРµС… Р±РµСЃРµРґР°С… РіР»РѕР±Р°Р»СЊРЅРѕР№ СЃРІСЏР·РєРё")
            return True
            
        if command in ['getacc', 'acc', 'РіРµС‚Р°РєРє', 'Р°РєРєР°СѓРЅС‚', 'account']:
            if await get_role(user_id, chat_id) < 1:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return True

            nick = await get_string(arguments, 1)
            if not nick:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РЅРёРє!", disable_mentions=1)
                return True

            nick_result = await get_acc(chat_id, nick)

            if not nick_result: await message.reply(f"РќРёРє {nick} РЅРёРєРѕРјСѓ РЅРµ РїСЂРёРЅР°РґР»РµР¶РёС‚!", disable_mentions=1)
            else:
                info = await bot.api.users.get(nick_result)
                await message.reply(f"РќРёРє {nick} РїСЂРёРЅР°РґР»РµР¶РёС‚ @id{nick_result} ({info[0].first_name} {info[0].last_name})", disable_mentions=1)
                await chats_log(user_id=user_id, target_id=None, role=None, log=f"РїРѕСЃРјРѕС‚СЂРµР»(-a) РєРѕРјСѓ РїСЂРёРЅР°РґР»РµР¶РёС‚ РќРёРєРќРµР№Рј В«{nick}В»")            

        if command in ['getnick', 'gnick', 'РіРЅРёРє', 'РіРµС‚РЅРёРє']:
            if await get_role(user_id, chat_id) < 1:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return True

            user = 0
            if message.reply_message: user = message.reply_message.from_id
            elif message.fwd_messages and message.fwd_messages[0].from_id > 0:
                user = message.fwd_messages[0].from_id
            elif len(arguments) >= 2 and await getID(arguments[1]): user = await getID(arguments[1])
            else:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ!", disable_mentions=1)
                return True

            nick = await get_nick(user, chat_id)
            if not nick: await message.reply(f"РЈ РґР°РЅРЅРѕРіРѕ @id{user} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ) РЅРµС‚ РЅРёРєР°!", disable_mentions=1)
            else: await message.reply(f"РќРёРє РґР°РЅРЅРѕРіРѕ @id{user} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ): {nick}", disable_mentions=1)
            await chats_log(user_id=user_id, target_id=user, role=None, log=f"РїРѕСЃРјРѕС‚СЂРµР»(-Р°) С‚РµРєСѓС‰РµРµ РёРјСЏ @id{user} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ). РўРµРєСѓС‰РёР№ РЅРёРє: В«{nick}В»")            

        if command in ['РЅРёРєР»РёСЃС‚', 'РЅРёРєРё', 'РІСЃРµРЅРёРєРё', 'nlist', 'nickslist', 'nicklist', 'nicks']:
            if await get_role(user_id, chat_id) < 1:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return True

            nicks = await nlist(chat_id, 1)
            nick_list = '\n'.join(nicks)
            if nick_list == "": nick_list = "РќРёРєРё РѕС‚СЃСѓС‚СЃС‚РІСѓСЋС‚!"

            keyboard = (
                Keyboard(inline=True)
                .add(Callback("вЏЄ", {"command": "nicksMinus", "page": 1, "chatId": chat_id}), color=KeyboardButtonColor.NEGATIVE)
                .add(Callback("Р‘РµР· РЅРёРєРѕРІ", {"command": "nonicks", "chatId": chat_id}), color=KeyboardButtonColor.PRIMARY)
                .add(Callback("вЏ©", {"command": "nicksPlus", "page": 1, "chatId": chat_id}), color=KeyboardButtonColor.POSITIVE)
            )

            await message.reply(f"РџРѕР»СЊР·РѕРІР°С‚РµР»Рё СЃ РЅРёРєРѕРј [1 СЃС‚СЂР°РЅРёС†Р°]:\n{nick_list}\n\nРџРѕР»СЊР·РѕРІР°С‚РµР»Рё Р±РµР· РЅРёРєРѕРІ: В«/nonickВ»", disable_mentions=1, keyboard=keyboard)
            await chats_log(user_id=user_id, target_id=None, role=None, log=f"РїРѕСЃРјРѕС‚СЂРµР»(-Р°) РїРѕР»СЊР·РѕРІР°С‚РµР»РµР№ СЃ РЅРёРєРѕРј")            

        if command in ['nonick', 'nonicks', 'nonicklist', 'nolist', 'nnlist', 'Р±РµР·РЅРёРєРѕРІ', 'РЅРѕРЅРёРєСЃ']:
            if await get_role(user_id, chat_id) < 1:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return True

            nonicks = await nonick(chat_id, 1)
            nonick_list = '\n'.join(nonicks)
            if nonick_list == "": nonick_list = "РџРѕР»СЊР·РѕРІР°С‚РµР»Рё Р±РµР· РЅРёРєРѕРІ РѕС‚СЃСѓС‚СЃС‚РІСѓСЋС‚!"

            keyboard = (
                Keyboard(inline=True)
                .add(Callback("вЏЄ", {"command": "nonickMinus", "page": 1, "chatId": chat_id}), color=KeyboardButtonColor.NEGATIVE)
                .add(Callback("РЎ РЅРёРєР°РјРё", {"command": "nicks", "chatId": chat_id}), color=KeyboardButtonColor.PRIMARY)
                .add(Callback("вЏ©", {"command": "nonickPlus", "page": 1, "chatId": chat_id}),
                     color=KeyboardButtonColor.POSITIVE)
            )

            await message.reply(f"РџРѕР»СЊР·РѕРІР°С‚РµР»Рё Р±РµР· РЅРёРєРѕРІ [1]:\n{nonick_list}\n\nРџРѕР»СЊР·РѕРІР°С‚РµР»Рё СЃ РЅРёРєР°РјРё: В«/nlistВ»", disable_mentions=1, keyboard=keyboard)
            await chats_log(user_id=user_id, target_id=None, role=None, log=f"РїРѕСЃРјРѕС‚СЂРµР»(-Р°) РїРѕР»СЊР·РѕРІР°С‚РµР»РµР№ Р±РµР· РЅРёРєРѕРІ")            

        if command in ['kick', 'РєРёРє', 'РёСЃРєР»СЋС‡РёС‚СЊ']:
            if await get_role(user_id, chat_id) < 1:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return True

            user = int
            arg = 0
            if message.reply_message:
                user = message.reply_message.from_id
                arg = 1
            elif message.fwd_messages and message.fwd_messages[0].from_id > 0:
                user = message.fwd_messages[0].from_id
                arg = 1
            elif len(arguments) >= 2 and await getID(arguments[1]):
                user = await getID(arguments[1])
                arg = 2
            else:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ!", disable_mentions=1)
                return True

            if chat_id == 23:
                await message.reply("Р”Р°РЅРЅР°СЏ Р±РµСЃРµРґР° РїСЂРѕРІРѕРґРёС‚СЃСЏ РІ СЃРїРµС†РёР°Р»РёР·РёСЂРѕРІР°РЅРЅРѕРј С‡Р°С‚Рµ, РєРѕС‚РѕСЂС‹Р№ РїСЂРµРґРЅР°Р·РЅР°С‡РµРЅ РёСЃРєР»СЋС‡РёС‚РµР»СЊРЅРѕ РґР»СЏ С‚РµСЃС‚РёСЂРѕРІС‰РёРєРѕРІ Р±РѕС‚Р°.\n\nР’ СЂР°РјРєР°С… РґР°РЅРЅРѕРіРѕ РѕР±СЃСѓР¶РґРµРЅРёСЏ РЅРµ РґРѕРїСѓСЃРєР°РµС‚СЃСЏ РёСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ РєРѕРјР°РЅРґ, РЅРµ РѕС‚РЅРѕСЃСЏС‰РёС…СЃСЏ Рє СЂР°Р±РѕС‚Рµ РїРѕ С‚РµСЃС‚РёСЂРѕРІР°РЅРёСЋ РёР»Рё С„СѓРЅРєС†РёРѕРЅРёСЂРѕРІР°РЅРёСЋ СЃРёСЃС‚РµРјС‹ РІ С†РµР»РѕРј.", disable_mentions=1)
                return True

            if await equals_roles(user_id, user, chat_id, message) < 2:
                await message.reply("Р’С‹ РЅРµ РјРѕР¶РµС‚Рµ РёСЃРєР»СЋС‡РёС‚СЊ РґР°РЅРЅРѕРіРѕ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ!", disable_mentions=1)
                return True

            reason = await get_string(arguments, arg)

            try: await bot.api.messages.remove_chat_user(chat_id, user)
            except:
                await message.reply(f"РќРµ СѓРґР°РµС‚СЃСЏ РёСЃРєР»СЋС‡РёС‚СЊ РґР°РЅРЅРѕРіРѕ @id{user} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ)! РќРµРѕР±С…РѕРґРёРјРѕ Р·Р°Р±СЂР°С‚СЊ Сѓ РЅРµРіРѕ Р·РІРµР·РґСѓ.", disable_mentions=1)
                return True

            keyboard = (
                Keyboard(inline=True)
                .add(Callback("РћС‡РёСЃС‚РёС‚СЊ", {"command": "clear", "chatId": chat_id, "user": user}), color=KeyboardButtonColor.NEGATIVE)
            )

            if not reason: await message.reply(f"@id{user_id} ({await get_user_name(user_id, chat_id)}) РєРёРєРЅСѓР»(-Р°) @id{user} ({await get_user_name(user, chat_id)})", disable_mentions=1, keyboard=keyboard)
            else: await message.reply(f"@id{user_id} ({await get_user_name(user_id, chat_id)}) РєРёРєРЅСѓР»(-Р°) @id{user} ({await get_user_name(user, chat_id)})\nРџСЂРёС‡РёРЅР°: {reason}", disable_mentions=1, keyboard=keyboard)
            await chats_log(user_id=user_id, target_id=user, role=None, log=f"РёСЃРєР»СЋС‡РёР»(-Р°) @id{user} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ) РёР· Р±РµСЃРµРґС‹")            

            await add_punishment(chat_id, user_id)
            if await get_sliv(user_id, chat_id) and await get_role(user_id, chat_id) < 5:
                await roleG(user_id, chat_id, 0)
                await message.reply(
                    f"вќ—пёЏ РЈСЂРѕРІРµРЅСЊ РїСЂР°РІ @id{user_id} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ) Р±С‹Р» СЃРЅСЏС‚ РёР·-Р·Р° РїРѕРґРѕР·СЂРµРЅРёР№ РІ СЃР»РёРІРµ Р±РµСЃРµРґС‹\n\n{await staff_zov(chat_id)}")

        if command in ['warn', 'РїСЂРµРґ', 'РІР°СЂРЅ', 'pred', 'РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёРµ']:
            if await get_role(user_id, chat_id) < 1:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return True

            user = int
            arg = 0
            if message.reply_message:
                user = message.reply_message.from_id
                arg = 1
            elif message.fwd_messages and message.fwd_messages[0].from_id > 0:
                user = message.fwd_messages[0].from_id
                arg = 1
            elif len(arguments) >= 2 and await getID(arguments[1]):
                user = await getID(arguments[1])
                arg = 2
            else:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ!", disable_mentions=1)
                return True

            if await equals_roles(user_id, user, chat_id, message) < 2:
                await message.reply("Р’С‹ РЅРµ РјРѕР¶РµС‚Рµ РІС‹РґР°С‚СЊ РїСЂРµРґ РґР°РЅРЅРѕРјСѓ РїРѕР»СЊР·РѕРІР°С‚РµР»СЋ!", disable_mentions=1)
                return True

            reason = await get_string(arguments, arg)
            if not reason:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РїСЂРёС‡РёРЅСѓ РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёСЏ!")
                return True

            warns = await warn(chat_id, user, user_id, reason)
            if warns < 3:
                keyboard = (
                    Keyboard(inline=True)
                    .add(Callback("РЎРЅСЏС‚СЊ РІР°СЂРЅ", {"command": "unwarn", "user": user, "chatId": chat_id}), color=KeyboardButtonColor.POSITIVE)
                    .add(Callback("РћС‡РёСЃС‚РёС‚СЊ", {"command": "clear", "chatId": chat_id, "user": user}), color=KeyboardButtonColor.NEGATIVE)
                )
                await message.reply(f"@id{user_id} ({await get_user_name(user_id, chat_id)}) РІС‹РґР°Р»(-Р°) РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёРµ @id{user} ({await get_user_name(user, chat_id)})\nРџСЂРёС‡РёРЅР°: {reason}\nРљРѕР»РёС‡РµСЃС‚РІРѕ РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёР№: {warns}", disable_mentions=1, keyboard=keyboard)
            else:
                keyboard = (
                    Keyboard(inline=True)
                    .add(Callback("РћС‡РёСЃС‚РёС‚СЊ", {"command": "clear", "chatId": chat_id, "user": user}),color=KeyboardButtonColor.NEGATIVE)
                )
                await message.answer(f"@id{user_id} ({await get_user_name(user_id, chat_id)}) РІС‹РґР°Р»(-Р°) РїРѕСЃР»РµРґРЅРµРµ РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёРµ @id{user} ({await get_user_name(user, chat_id)}) (3/3)\nРџСЂРёС‡РёРЅР°: {reason}\n@id{user} (РџРѕР»СЊР·РѕРІР°С‚РµР»СЊ) Р±С‹Р» РёСЃРєР»СЋС‡РµРЅ Р·Р° Р±РѕР»СЊС€РѕРµ РєРѕР»РёС‡РµСЃС‚РІРѕ РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёР№!",disable_mentions=1, keyboard=keyboard)
                try: await bot.api.messages.remove_chat_user(user)
                except: pass
                await clear_warns(chat_id, user)

            await add_punishment(chat_id, user_id)
            await chats_log(user_id=user_id, target_id=user, role=None, log=f"РІС‹РґР°Р»(-Р°) РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёРµ @id{user} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЋ). РџСЂРёС‡РёРЅР°: {reason}, РС‚РѕРіРѕ Сѓ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ: {warns}/3")            
            if await get_sliv(user_id, chat_id) and await get_role(user_id, chat_id) < 5:
                await roleG(user_id, chat_id, 0)
                await message.reply(
                    f"вќ—пёЏ РЈСЂРѕРІРµРЅСЊ РїСЂР°РІ @id{user_id} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ) Р±С‹Р» СЃРЅСЏС‚ РёР·-Р·Р° РїРѕРґРѕР·СЂРµРЅРёР№ РІ СЃР»РёРІРµ Р±РµСЃРµРґС‹\n\n{await staff_zov(chat_id)}")

        if command in ['unwarn', 'СѓРЅРІР°СЂРЅ', 'Р°РЅРІР°СЂРЅ', 'СЃРЅСЏС‚СЊРїСЂРµРґ', 'РјРёРЅСѓСЃРїСЂРµРґ']:
            if await get_role(user_id, chat_id) < 1:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return True

            user = int
            if message.reply_message: user = message.reply_message.from_id
            elif len(arguments) >= 2 and await getID(arguments[1]): user = await getID(arguments[1])
            else:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ!", disable_mentions=1)
                return True

            if await equals_roles(user_id, user, chat_id, message) < 2:
                await message.reply("Р’С‹ РЅРµ РјРѕР¶РµС‚Рµ СЃРЅСЏС‚СЊ РїСЂРµРґ РґР°РЅРЅРѕРјСѓ РїРѕР»СЊР·РѕРІР°С‚РµР»СЋ!", disable_mentions=1)
                return True

            if await get_warns(user, chat_id) < 1:
                await message.reply("РЈ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ РЅРµС‚ РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёР№!")
                return True

            warns = await unwarn(chat_id, user)
            await chats_log(user_id=user_id, target_id=user, role=None, log=f"СЃРЅСЏР»(-Р°) РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёРµ @id{user} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЋ)")            
            await message.reply(f"@id{user_id} ({await get_user_name(user_id, chat_id)}) СЃРЅСЏР»(-Р°) РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёРµ @id{user} ({await get_user_name(user, chat_id)})\nРљРѕР»РёС‡РµСЃС‚РІРѕ РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёР№: {warns}", disable_mentions=1)

        # --- /rules ---
        if command in ['rules', 'РїСЂР°РІРёР»Р°', 'РїСЂР°РІРёР»Р°С‡Р°С‚Р°']:
            sql.execute("SELECT description FROM rules WHERE chat_id = ?", (chat_id,))
            rules_text = sql.fetchone()

            if not rules_text:
                await message.reply("Р’ СЌС‚РѕРј С‡Р°С‚Рµ РµС‰С‘ РЅРµ СѓСЃС‚Р°РЅРѕРІР»РµРЅС‹ РїСЂР°РІРёР»Р°!\n\nРЈСЃС‚Р°РЅРѕРІРёС‚СЊ РЅРѕРІС‹Рµ РїСЂР°РІРёР»Р° РјРѕР¶РµС‚ РІР»Р°РґРµР»РµС† Р±РµСЃРµРґС‹ РєРѕРјР°РЅРґРѕР№: В«/setrulesВ»", disable_mentions=1)
                return True

            await message.reply(f"{rules_text[0]}", disable_mentions=1)
            return True

        # --- /setrules ---
        if command in ['setrules', 'СѓСЃС‚Р°РЅРѕРІРёС‚СЊРїСЂР°РІРёР»Р°']:
            if await get_role(user_id, chat_id) < 7:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return True

            if len(arguments) < 2:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РЅРѕРІС‹Рµ РїСЂР°РІРёР»Р° Р±РµСЃРµРґС‹!", disable_mentions=1)
                return True

            text = " ".join(arguments[1:])
            sql.execute("INSERT OR REPLACE INTO rules (chat_id, description) VALUES (?, ?)", (chat_id, text))
            database.commit()

            await message.reply(f"@id{user_id} ({await get_user_name(user_id, chat_id)}) СѓСЃС‚Р°РЅРѕРІРёР»(-Р°) РЅРѕРІС‹Рµ РїСЂР°РІРёР»Р° РІ Р±РµСЃРµРґСѓ В«/rulesВ»:\n\n{text}", disable_mentions=1)
            return True

        if command in ['infoid', 'РёРЅС„РѕР°Р№РґРё', 'С‡Р°С‚С‹РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ', 'РёРЅС„РѕРёРґ']:
                if await get_role(user_id, chat_id) < 10:
                        await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                        return True

                if len(arguments) < 2:
                        await message.reply("РЈРєР°Р¶РёС‚Рµ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ!", disable_mentions=1)
                        return True

                target = await getID(arguments[1])
                if not target:
                        await message.reply("РќРµ СѓРґР°Р»РѕСЃСЊ РѕРїСЂРµРґРµР»РёС‚СЊ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ.", disable_mentions=1)
                        return True

                sql.execute("SELECT chat_id FROM chats WHERE owner_id = ?", (target,))
                user_chats = sql.fetchall()
                if not user_chats:
                        await message.reply("РЈ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ РЅРµС‚ Р·Р°СЂРµРіРёСЃС‚СЂРёСЂРѕРІР°РЅРЅС‹С… Р±РµСЃРµРґ.", disable_mentions=1)
                        return True

                # Р‘РµСЂРµРј РїРµСЂРІСѓСЋ СЃС‚СЂР°РЅРёС†Сѓ
                page = 1
                per_page = 5
                total_pages = (len(user_chats) + per_page - 1) // per_page
                start = (page - 1) * per_page
                end = start + per_page
                page_chats = user_chats[start:end]

                all_chats = []
                for idx, (chat_id_val,) in enumerate(page_chats, start=1):
                        try:
                                peer_id = 2000000000 + chat_id_val
                                info = await bot.api.messages.get_conversations_by_id(peer_ids=peer_id)
                                if info.items:
                                        chat_title = info.items[0].chat_settings.title
                                else:
                                        chat_title = "Р‘РµР· РЅР°Р·РІР°РЅРёСЏ"
                                link = (await bot.api.messages.get_invite_link(peer_id=peer_id, reset=0)).link
                        except:
                                chat_title = "РќРµ СѓРґР°Р»РѕСЃСЊ РїРѕР»СѓС‡РёС‚СЊ"
                                link = "РќРµ СѓРґР°Р»РѕСЃСЊ РїРѕР»СѓС‡РёС‚СЊ"

                        all_chats.append(f"{idx}. {chat_title} | рџ†”: {chat_id_val} | рџ”— РЎСЃС‹Р»РєР°: {link}")

                all_chats_text = "\n".join(all_chats)
                keyboard = (
                    Keyboard(inline=True)
                    .add(Callback("РќР°Р·Р°Рґ", {"command": "infoidMinus", "page": 1, "user": target}), color=KeyboardButtonColor.NEGATIVE)
                    .add(Callback("Р’РїРµСЂС‘Рґ", {"command": "infoidPlus", "page": 1, "user": target}), color=KeyboardButtonColor.POSITIVE)
                )

                await message.reply(
                        f"вќ— РЎРїРёСЃРѕРє Р±РµСЃРµРґ @id{target} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ):\n(РЎС‚СЂР°РЅРёС†Р°: 1)\n\n{all_chats_text}\n\nрџ—ЁпёЏ Р’СЃРµРіРѕ Р±РµСЃРµРґ Сѓ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ: {idx}",
                        disable_mentions=1,
                        keyboard=keyboard
                )
                return True                

        if command in ['banwords', 'Р·Р°РїСЂРµС‰РµРЅРЅС‹РµСЃР»РѕРІР°', 'banwordlist']:
                if await get_role(user_id, chat_id) < 10:
                        await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                        return True

                sql.execute("SELECT word, creator_id, time FROM ban_words ORDER BY time DESC")
                rows = sql.fetchall()
                if not rows:
                        await message.reply("Р—Р°РїСЂРµС‰С‘РЅРЅС‹Рµ СЃР»РѕРІР° РѕС‚СЃСѓС‚СЃС‚РІСѓСЋС‚!", disable_mentions=1)
                        return True

                total = len(rows)
                per_page = 5
                max_page = (total + per_page - 1) // per_page

                async def get_words_page(page: int):
                        start = (page - 1) * per_page
                        end = start + per_page
                        formatted = []
                        for i, (word, creator, tm) in enumerate(rows[start:end], start=start + 1):
                                try:
                                        info = await bot.api.users.get(user_ids=creator)
                                        creator_name = f"{info[0].first_name} {info[0].last_name}"
                                except:
                                        creator_name = "РќРµ СѓРґР°Р»РѕСЃСЊ РїРѕР»СѓС‡РёС‚СЊ РёРјСЏ"
                                formatted.append(f"{i}. {word} | @id{creator} ({creator_name}) | Р’СЂРµРјСЏ: {tm}")
                        return formatted

                page = 1
                page_data = await get_words_page(page)
                page_text = "\n\n".join(page_data)

                keyboard = (
                        Keyboard(inline=True)
                        .add(Callback("вЏЄ", {"command": "banwordsMinus", "page": 1}), color=KeyboardButtonColor.NEGATIVE)
                        .add(Callback("вЏ©", {"command": "banwordsPlus", "page": 1}), color=KeyboardButtonColor.POSITIVE)
                )

                await message.reply(
                        f"Р—Р°РїСЂРµС‰С‘РЅРЅС‹Рµ СЃР»РѕРІР° (РЎС‚СЂР°РЅРёС†Р° 1):\n\n{page_text}\n\nР’СЃРµРіРѕ Р·Р°РїСЂРµС‰РµРЅРЅС‹С… СЃР»РѕРІ: {total}",
                        disable_mentions=1, keyboard=keyboard
                )
                return True
                
        if command in ['addbanwords', 'addword', 'banword']:
                if await get_role(user_id, chat_id) < 10:
                        await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                        return True
                if len(arguments) < 2:
                        await message.reply("РџСЂРёРјРµСЂ: /addbanwords С‚РµРєСЃС‚")
                        return True

                word = arguments[1].lower()
                time_now = datetime.now().strftime("%I:%M %p")

                sql.execute("SELECT word FROM ban_words WHERE word = ?", (word,))
                if sql.fetchone():
                        await message.reply("РЎР»РѕРІРѕ СѓР¶Рµ РЅР°С…РѕРґРёС‚СЊСЃСЏ РІ СЃРїРёСЃРєРµ Р·Р°РїСЂРµС‰РµРЅРЅС‹С… СЃР»РѕРІ!")
                        return True

                sql.execute("INSERT INTO ban_words (word, creator_id, time) VALUES (?, ?, ?)", (word, user_id, time_now))
                database.commit()

                await message.reply(f"@id{user_id} ({await get_user_name(user_id, chat_id)}) РґРѕР±Р°РІРёР»(-Р°) СЃР»РѕРІРѕ В«{word}В» РІ СЃРїРёСЃРѕРє Р·Р°РїСЂРµС‰РµРЅРЅС‹С… СЃР»РѕРІ!")
                return True

        if command in ['removebanwords', 'unword', 'unbanword']:
                if await get_role(user_id, chat_id) < 10:
                        await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                        return True
                if len(arguments) < 2:
                        await message.reply("РџСЂРёРјРµСЂ: /removebanwords С‚РµРєСЃС‚")
                        return True

                word = arguments[1].lower()
                sql.execute("SELECT word FROM ban_words WHERE word = ?", (word,))
                if not sql.fetchone():
                        await message.reply("РЎР»РѕРІРѕ РѕС‚СЃСѓС‚СЃС‚РІСѓРµС‚ РІ СЃРїРёСЃРєРµ Р·Р°РїСЂРµС‰РµРЅРЅС‹С… СЃР»РѕРІ!")
                        return True

                sql.execute("DELETE FROM ban_words WHERE word = ?", (word,))
                database.commit()

                await message.reply(f"@id{user_id} ({await get_user_name(user_id, chat_id)}) СѓРґР°Р»РёР»(-Р°) СЃР»РѕРІРѕ В«{word}В» РёР· СЃРїРёСЃРєР° Р·Р°РїСЂРµС‰РµРЅРЅС‹С… СЃР»РѕРІ!")
                return True
                
        # --- /info ---
        if command in ['info', 'РёРЅС„Рѕ', 'РёРЅС„РѕСЂРјР°С†РёСЏ']:
            sql.execute("SELECT description FROM info WHERE chat_id = ?", (chat_id,))
            info_text = sql.fetchone()

            if not info_text:
                await message.reply("Р’ СЌС‚РѕРј С‡Р°С‚Рµ РµС‰С‘ РЅРµ СѓСЃС‚Р°РЅРѕРІР»РµРЅР° РёРЅС„РѕСЂРјР°С†РёСЏ!\n\nРЈСЃС‚Р°РЅРѕРІРёС‚СЊ РЅРѕРІСѓСЋ РёРЅС„РѕСЂРјР°С†РёСЋ РјРѕР¶РµС‚ РІР»Р°РґРµР»РµС† Р±РµСЃРµРґС‹ РєРѕРјР°РЅРґРѕР№: В«/setinfoВ»", disable_mentions=1)
                return True

            await message.reply(f"{info_text[0]}", disable_mentions=1)
            return True

        if command in ['other', 'РґСЂСѓРіРёРµ', 'РґСЂСѓРіРёРµРєРјРґ', 'РёРіСЂРѕРІС‹РµРєРјРґ']:
            await message.reply(
                "/РїСЂРёР· вЂ” РїРѕР»СѓС‡РёС‚СЊ РµР¶РµРґРЅРµРІРЅС‹Р№ Р±РѕРЅСѓСЃ\n"
                "/Р±Р°Р»Р°РЅСЃ вЂ” РїРѕСЃРјРѕС‚СЂРµС‚СЊ СЃРІРѕР№ Р±Р°Р»Р°РЅСЃ\n"
                "/РґСѓСЌР»СЊ вЂ” СЃС‹РіСЂР°С‚СЊ РґСѓСЌР»СЊ\n"
                "/РїРµСЂРµРґР°С‚СЊ вЂ” РїРµСЂРµРґР°С‚СЊ РјРѕРЅРµС‚С‹ РґСЂСѓРіРѕРјСѓ РїРѕР»СЊР·РѕРІР°С‚РµР»СЋ\n"
                "/С‚РѕРї вЂ” С‚РѕРї СЃР°РјС‹С… Р±РѕРіР°С‚С‹С… РїРѕР»СЊР·РѕРІР°С‚РµР»РµР№\n"
                "/РїРѕР»РѕР¶РёС‚СЊ вЂ” РїРѕР»РѕР¶РёС‚СЊ РґРµРЅСЊРіРё РІ Р±Р°РЅРє\n"
                "/СЃРЅСЏС‚СЊ вЂ” СЃРЅСЏС‚СЊ РґРµРЅСЊРіРё СЃ Р±Р°РЅРєР°\n"
                "/Р±Р»Р°РіРѕ вЂ” РѕС‚РїСЂР°РІРёС‚СЊ РјРѕРЅРµС‚С‹ РІ Р±Р»Р°РіРѕС‚РІРѕСЂРёС‚РµР»СЊРЅРѕСЃС‚СЊ\n"
                "/С‚РѕРїР±Р»Р°РіРѕ вЂ” С‚РѕРї РѕС‚РїСЂР°РІРёС‚РµР»РµР№ РјРѕРЅРµС‚ РІ Р±Р»Р°РіРѕС‚РІРѕСЂРёС‚РµР»СЊРЅРѕСЃС‚СЊ\n"
                "/buyvip вЂ” РєСѓРїРёС‚СЊ РІРёРї СЃС‚Р°С‚СѓСЃ\n"
                "/РїСЂРѕРјРѕ вЂ” РїРѕР»СѓС‡РёС‚СЊ Р±РѕРЅСѓСЃ\n"
                "/РѕС‚РєСЂС‹С‚СЊРґРµРїРѕР·РёС‚ вЂ” РѕС‚РєСЂС‹С‚СЊ РґРµРїРѕР·РёС‚ (РґР»СЏ РІРёРї)\n"
                "/Р·Р°РєСЂС‹С‚СЊРґРµРїРѕР·РёС‚ вЂ” Р·Р°РєСЂС‹С‚СЊ РґРµРїРѕР·РёС‚ (РґР»СЏ РІРёРї)\n"
                "/form вЂ” РїРѕРґР°С‚СЊ С„РѕСЂРјСѓ РЅР° Р±Р°РЅ (С‚РѕР»СЊРєРѕ РІ РѕРїСЂРµРґРµР»РµРЅРЅРѕРј С‡Р°С‚Рµ)\n"
                "/offer вЂ” РїСЂРµРґР»РѕР¶РµРЅРёРµ РїРѕ СѓР»СѓС‡С€РµРЅРёСЋ Р±РѕС‚Р°\n"
                "/РєР°Р·РёРЅРѕ вЂ” РёРіСЂР° РІ РєР°Р·РёРЅРѕ РЅР° СЃС‚Р°РІРєСѓ\n"
                "/promo вЂ” Р°РєС‚РёРІРёСЂРѕРІР°С‚СЊ РѕРїСЂРµРґРµР»РµРЅРЅС‹Р№ РїСЂРѕРјРѕ-РєРѕРґ\n"
                "/promolist вЂ” СЃРїРёСЃРѕРє Р°РєС‚РёРІРёСЂРѕРІР°РЅРЅС‹С… РїСЂРѕРјРѕ-РєРѕРґРѕРІ"
            )
            return True            
            
        # --- /setinfo ---
        if command in ['setinfo', 'СѓСЃС‚Р°РЅРѕРІРёС‚СЊРёРЅС„Рѕ']:
            if await get_role(user_id, chat_id) < 7:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return True

            if len(arguments) < 2:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РЅРѕРІСѓСЋ РёРЅС„РѕСЂРјР°С†РёСЋ РІ Р±РµСЃРµРґРµ;", disable_mentions=1)
                return True

            text = " ".join(arguments[1:])
            sql.execute("INSERT OR REPLACE INTO info (chat_id, description) VALUES (?, ?)", (chat_id, text))
            database.commit()

            await message.reply(f"@id{user_id} ({await get_user_name(user_id, chat_id)}) СѓСЃС‚Р°РЅРѕРІРёР»(-Р°) РЅРѕРІСѓСЋ РёРЅС„РѕСЂРјР°С†РёСЋ РІ Р±РµСЃРµРґСѓ В«/infoВ»:\n\n{text}", disable_mentions=1)
            return True

        if command in ['antisliv', 'Р°РЅС‚РёСЃР»РёРІ']:
            if await get_role(user_id, chat_id) < 6:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return True

            # РџРѕР»СѓС‡Р°РµРј С‚РµРєСѓС‰РµРµ СЃРѕСЃС‚РѕСЏРЅРёРµ Р°РЅС‚РёСЃР»РёРІР°
            current_mode = await get_antisliv(chat_id)
            new_mode = 0 if current_mode == 1 else 1

            # РћР±РЅРѕРІР»СЏРµРј СЃРѕСЃС‚РѕСЏРЅРёРµ
            await antisliv_mode(chat_id, new_mode)

            # РџРѕР»СѓС‡Р°РµРј РёРјСЏ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ, РєС‚Рѕ РёР·РјРµРЅРёР» СЂРµР¶РёРј
            user_name = await get_user_name(user_id, chat_id)

            # Р¤РѕСЂРјРёСЂСѓРµРј С‚РµРєСЃС‚ СЃС‚Р°С‚СѓСЃР°
            if new_mode == 1:
                text = f"@id{user_id} ({user_name}) РІРєР»СЋС‡РёР»(-Р°) СЃРёСЃС‚РµРјСѓ Р°РЅС‚РёСЃР»РёРІР°!"
            else:
                text = f"@id{user_id} ({user_name}) РІС‹РєР»СЋС‡РёР»(-Р°) СЃРёСЃС‚РµРјСѓ Р°РЅС‚РёСЃР»РёРІР°!"

            await message.reply(text, disable_mentions=1)
            return True            
            
        if command in ['clearwarn', 'РѕС‡РёСЃС‚РёС‚СЊРІР°СЂРЅС‹']:
            if await get_role(user_id, chat_id) < 6:  # РґРѕСЃС‚СѓРї СЃ 6 СЂР°РЅРіР°
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return True

            count = await clear_all_warns(chat_id)

            if count == 0:
                await message.reply("Р’ РґР°РЅРЅРѕР№ Р±РµСЃРµРґРµ РЅРµС‚ РїРѕР»СЊР·РѕРІР°С‚РµР»РµР№ СЃ РЅР°РєР°Р·Р°РЅРёСЏРјРё", disable_mentions=1)
            else:
                await message.reply(f"РЈРґР°Р»РµРЅС‹ РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёСЏ Сѓ {count} РїРѕР»СЊР·РѕРІР°С‚РµР»РµР№!", disable_mentions=1)
                await chats_log(user_id=user_id, target_id=None, role=None, log=f"РѕС‡РёСЃС‚РёР»(-Р°) РІР°СЂРЅС‹ Сѓ {count} РїРѕР»СЊР·РѕРІР°С‚РµР»РµР№")            

            return True
            
        if command in ['getwarn', 'gwarn', 'getwarns', 'РіРµС‚РІР°СЂРЅ', 'РіРІР°СЂРЅ']:
            if await get_role(user_id, chat_id) < 1:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return True

            user = int
            if message.reply_message: user = message.reply_message.from_id
            elif message.fwd_messages and message.fwd_messages[0].from_id > 0:
                user = message.fwd_messages[0].from_id
            elif len(arguments) >= 2 and await getID(arguments[1]): user = await getID(arguments[1])
            else:
                await message.reply("Р’С‹ РЅРµ СѓРєР°Р·Р°Р»Рё @РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ!", disable_mentions=1)
                return True

            warns = await gwarn(user, chat_id)
            string_info = str
            if not warns: string_info = "РђРєС‚РёРІРЅС‹С… РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёР№ РЅРµС‚!"
            else: string_info = f"@id{warns['moder']} (РњРѕРґРµСЂР°С‚РѕСЂ) | {warns['reason']} | {warns['count']}/3 | {warns['time']}"

            keyboard = (
                Keyboard(inline=True)
                .add(Callback("РСЃС‚РѕСЂРёСЏ РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёР№", {"command": "warnhistory", "user": user, "chatId": chat_id}), color=KeyboardButtonColor.PRIMARY)
            )

            await message.answer(f"@id{user_id} ({await get_user_name(user_id, chat_id)}), РёРЅС„РѕСЂРјР°С†РёСЏ Рѕ Р°РєС‚РёРІРЅС‹С… РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёСЏС… @id{user} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ):\n{string_info}", disable_mentions=1, keyboard=keyboard)
            await chats_log(user_id=user_id, target_id=user, role=None, log=f"РїРѕСЃРјРѕС‚СЂРµР»(-Р°) Р°РєС‚РёРІРЅС‹Рµ РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёСЏ @id{user} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ)")            

        if command in ['warnhistory', 'historywarns', 'whistory', 'РёСЃС‚РѕСЂРёСЏРІР°СЂРЅРѕРІ', 'РёСЃС‚РѕСЂРёСЏРїСЂРµРґРѕРІ']:
            if await get_role(user_id, chat_id) < 1:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return True

            user = int
            if message.reply_message: user = message.reply_message.from_id
            elif message.fwd_messages and message.fwd_messages[0].from_id > 0:
                user = message.fwd_messages[0].from_id
            elif len(arguments) >= 2 and await getID(arguments[1]): user = await getID(arguments[1])
            else:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ!", disable_mentions=1)
                return True

            warnhistory_mass = await warnhistory(user, chat_id)
            if not warnhistory_mass: wh_string = "РџСЂРµРґСѓРїСЂРµР¶РґРµРЅРёР№ РЅРµ Р±С‹Р»Рѕ!"
            else: wh_string = '\n'.join(warnhistory_mass)

            keyboard = (
                Keyboard(inline=True)
                .add(Callback("РђРєС‚РёРІРЅС‹Рµ РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёСЏ", {"command": "activeWarns", "user": user, "chatId": chat_id}), color=KeyboardButtonColor.PRIMARY)
                .add(Callback("Р’СЃСЏ РёРЅС„РѕСЂРјР°С†РёСЏ", {"command": "stats", "user": user, "chatId": chat_id}),color=KeyboardButtonColor.PRIMARY)
            )

            await message.reply(f"РРЅС„РѕСЂРјР°С†РёСЏ Рѕ РІСЃРµС… РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёСЏС… @id{user} ({await get_user_name(user, chat_id)})\nРљРѕР»РёС‡РµСЃС‚РІРѕ РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёР№ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ: {await get_warns(user, chat_id)}\n\nРРЅС„РѕСЂРјР°С†РёСЏ Рѕ РїРѕСЃР»РµРґРЅРёС… 10 РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёР№ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ:\n{wh_string}", disable_mentions=1, keyboard=keyboard)
            await chats_log(user_id=user_id, target_id=user, role=None, log=f"РїРѕСЃРјРѕС‚СЂРµР»(-Р°) РІСЃРµ РїСЂРµРґСѓРїСЂРµР¶РґРµРЅРёСЏ @id{user} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ)")            

# GAME
        # ---------------- Р‘РђР›РђРќРЎ ----------------
        if command in ["Р±Р°Р»Р°РЅСЃ"]:
            target = await extract_user_id(message)
            if not target:
                target = user_id

            # Р—Р°РіСЂСѓР¶Р°РµРј Р°РєС‚СѓР°Р»СЊРЅС‹Рµ РґР°РЅРЅС‹Рµ РёР· С„Р°Р№Р»Р°
            balances = load_data(BALANCES_FILE)
            if str(target) not in balances:
                balances[str(target)] = get_balance(target)  # СЃРѕР·РґР°С‘Рј Р·Р°РїРёСЃСЊ, РµСЃР»Рё РµС‘ РЅРµС‚
            bal = balances[str(target)]

            now = datetime.now()

            # РџРѕР»СѓС‡Р°РµРј РёРјСЏ РІ СЂРѕРґРёС‚РµР»СЊРЅРѕРј РїР°РґРµР¶Рµ
            try:
                info = await bot.api.users.get(user_ids=target, name_case="gen")
                name = f"{info[0].first_name} {info[0].last_name}"
                mention = f"РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ [id{target}|{name}]"
            except:
                mention = f"[id{target}|id{target}]"

            # РџСЂРѕРІРµСЂРєР° РЅР° VIP
            vip_until = bal.get("vip_until")
            if vip_until:
                try:
                    vip_end = datetime.fromisoformat(vip_until)
                    if vip_end > now:
                        is_vip = True
                        delta = vip_end - now
                        days, seconds = delta.days, delta.seconds
                        hours, minutes = divmod(seconds // 60, 60)
                        vip_status = "VIP"
                        vip_time = f"вЏі Р”Рѕ РѕРєРѕРЅС‡Р°РЅРёСЏ СЃС‚Р°С‚СѓСЃР°: {days}Рґ {hours}С‡ {minutes}Рј"
                        transfer_limit = 500_000
                    else:
                        is_vip = False
                        vip_status = "РћС‚СЃСѓС‚СЃС‚РІСѓРµС‚"
                        vip_time = "вЏі РћС‚СЃСѓС‚СЃС‚РІСѓРµС‚"
                        transfer_limit = 100_000
                except:
                    is_vip = False
                    vip_status = "РћС‚СЃСѓС‚СЃС‚РІСѓРµС‚"
                    vip_time = "вЏі РћС‚СЃСѓС‚СЃС‚РІСѓРµС‚"
                    transfer_limit = 100_000
            else:
                is_vip = False
                vip_status = "РћС‚СЃСѓС‚СЃС‚РІСѓРµС‚"
                vip_time = "вЏі РћС‚СЃСѓС‚СЃС‚РІСѓРµС‚"
                transfer_limit = 100_000

            # Р›РёРјРёС‚ РїРµСЂРµРІРѕРґРѕРІ
            today = now.date().isoformat()
            spent_today = bal.get("transfers_today", {}).get(today, 0)
            remaining_limit = max(0, transfer_limit - spent_today)

            # Р”РµРїРѕР·РёС‚
            deposit_text = ""
            deposit_amount = bal.get("deposit_amount", 0)
            deposit_until = bal.get("deposit_until")
            deposit_percent = bal.get("deposit_percent", 0)
            if deposit_amount > 1 and deposit_until:
                try:
                    end_time = datetime.fromisoformat(deposit_until)
                    if now < end_time:
                        delta = end_time - now
                        days, seconds = delta.days, delta.seconds
                        hours, minutes = divmod(seconds // 60, 60)
                        deposit_text = (
                            f"\nрџ’ё Р”РµРїРѕР·РёС‚: {format_number(deposit_amount)}$ "
                            f"РЅР° {days} РґРЅ. "
                            f"РїРѕРґ {deposit_percent}%"
                            f"\nвЏі Р”Рѕ РІС‹РІРѕРґР°: {days}Рґ {hours}С‡ {minutes}Рј"
                        )
                    else:
                        deposit_text = (
                            f"\nрџ’ё Р”РµРїРѕР·РёС‚: {format_number(deposit_amount)}$ "
                            f"РїРѕРґ {deposit_percent}%"
                            f"\nвЏі Р”Рѕ РІС‹РІРѕРґР°: РјРѕР¶РЅРѕ Р·Р°Р±РёСЂР°С‚СЊ!"
                        )
                except:
                    pass

            await message.reply(
                f"рџ’° РЈ {mention} {format_number(bal['wallet'])}$\n"
                f"рџЏ› РЎС‡РµС‚ РІ Р±Р°РЅРєРµ: {format_number(bal['bank'])}$\n"
                f"рџЏ† Р”СѓСЌР»РµР№ РІС‹РёРіСЂР°РЅРѕ: {bal['won']}\n"
                f"рџ’” Р”СѓСЌР»РµР№ РїСЂРѕРёРіСЂР°РЅРѕ: {bal['lost']}\n"
                f"рџЋ‰ Р’СЃРµРіРѕ РІС‹РёРіСЂР°РЅРѕ: {format_number(bal['won_total'])}$\n"
                f"рџ’° Р’СЃРµРіРѕ РїСЂРѕРёРіСЂР°РЅРѕ: {format_number(bal['lost_total'])}$\n"
                f"рџ“¤ РћС‚РїСЂР°РІР»РµРЅРѕ РїРµСЂРµРІРѕРґР°РјРё: {format_number(bal['sent_total'])}$\n"
                f"рџ“Ґ РџРѕР»СѓС‡РµРЅРѕ РїРµСЂРµРІРѕРґР°РјРё: {format_number(bal['received_total'])}$\n"
                f"в­ђ РЎС‚Р°С‚СѓСЃ: {vip_status}\n"
                f"{vip_time}\n"
                f"рџ”„ РћСЃС‚Р°С‚РѕРє Р»РёРјРёС‚Р° РЅР° СЃРµРіРѕРґРЅСЏ: {format_number(remaining_limit)}$ / {format_number(transfer_limit)}$"
                f"{deposit_text}"
            )
            return            
          
        # ---------------- GIVEALL / Р РђР—Р”РђР§Рђ ----------------
        if command in ["giveall", "СЂР°Р·РґР°С‡Р°"]:
            # СЂР°Р·СЂРµС€С‘РЅРЅС‹Р№ Р’Рљ ID Р°РґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂР°
            role = await get_role(user_id, chat_id)
            if role < 11:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!")
                return

            if chat_id == 23:
                await message.reply("Р”Р°РЅРЅР°СЏ Р±РµСЃРµРґР° РїСЂРѕРІРѕРґРёС‚СЃСЏ РІ СЃРїРµС†РёР°Р»РёР·РёСЂРѕРІР°РЅРЅРѕРј С‡Р°С‚Рµ, РєРѕС‚РѕСЂС‹Р№ РїСЂРµРґРЅР°Р·РЅР°С‡РµРЅ РёСЃРєР»СЋС‡РёС‚РµР»СЊРЅРѕ РґР»СЏ С‚РµСЃС‚РёСЂРѕРІС‰РёРєРѕРІ Р±РѕС‚Р°.\n\nР’ СЂР°РјРєР°С… РґР°РЅРЅРѕРіРѕ РѕР±СЃСѓР¶РґРµРЅРёСЏ РЅРµ РґРѕРїСѓСЃРєР°РµС‚СЃСЏ РёСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ РєРѕРјР°РЅРґ, РЅРµ РѕС‚РЅРѕСЃСЏС‰РёС…СЃСЏ Рє СЂР°Р±РѕС‚Рµ РїРѕ С‚РµСЃС‚РёСЂРѕРІР°РЅРёСЋ РёР»Рё С„СѓРЅРєС†РёРѕРЅРёСЂРѕРІР°РЅРёСЋ СЃРёСЃС‚РµРјС‹ РІ С†РµР»РѕРј.", disable_mentions=1)
                return True

            if len(arguments) < 1:
                await message.reply("рџ’° РџСЂРёРјРµСЂ: /СЂР°Р·РґР°С‡Р° 1000")
                return

            try:
                amount = int(arguments[-1])
                if amount <= 0:
                    raise ValueError()
            except:
                await message.reply("РЈРєР°Р¶РёС‚Рµ СЃСѓРјРјСѓ С‡РёСЃР»РѕРј!")
                return

            # Р·Р°РіСЂСѓР¶Р°РµРј Р±Р°Р»Р°РЅСЃС‹
            balances = load_data(BALANCES_FILE)

            all_users_text = ""
            for i, (uid, bal) in enumerate(balances.items(), start=1):
                # РѕР±РЅРѕРІР»СЏРµРј РєРѕС€РµР»С‘Рє
                bal["wallet"] += amount

                # РїРѕР»СѓС‡Р°РµРј РёРјСЏ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ
                try:
                    info = await bot.api.users.get(user_ids=uid)
                    full_name = f"{info[0].first_name} {info[0].last_name}"
                except:
                    full_name = f"РћС€РёР±РєР°"

                all_users_text += f"{i}. [id{uid}|{full_name}] | рџ’° РќРѕРІС‹Р№ Р±Р°Р»Р°РЅСЃ: {format_number(bal['wallet'])}\n"

            # СЃРѕС…СЂР°РЅСЏРµРј РѕР±РЅРѕРІР»С‘РЅРЅС‹Рµ Р±Р°Р»Р°РЅСЃС‹
            save_data(BALANCES_FILE, balances)
            await log_economy(user_id=uid, target_id=None, amount=amount, log=f"РїСЂРѕРёР·РІРµР»(-Р°) СЂР°Р·РґР°С‡Сѓ РЅР° {amount}$")            

            # С„РѕСЂРјРёСЂСѓРµРј СЃРѕРѕР±С‰РµРЅРёРµ
            admin_name = f"@id{user_id}"  # РёР»Рё РјРѕР¶РЅРѕ РїРѕР»СѓС‡РёС‚СЊ РїРѕР»РЅРѕРµ РёРјСЏ Р°РґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂР°
            await message.reply(
                f"Р Р°Р·РґР°С‡Р° РЅР° В«{format_number(amount)}$В» Р±С‹Р»Р° СѓСЃРїРµС€РЅРѕ РїСЂРѕРёР·РІРµРґРµРЅР° {admin_name} (Р°РґРјРёРЅРёСЃС‚СЂР°С‚РѕСЂРѕРј Р±РѕС‚Р°), РјРѕРЅРµС‚С‹ РїРѕР»СѓС‡РёР»Рё:\n\n{all_users_text}"
            )
            return            

        if command in ['say', 'СЃРѕРѕР±С‰РµРЅРёРµ']:
            if await get_role(user_id, chat_id) < 10:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return True

            if len(arguments) < 2:
                await message.reply("РЈРєР°Р¶РёС‚Рµ Р°Р№РґРё Р±РµСЃРµРґС‹!")
                return True

            # РџР°СЂСЃРёРј target_chat РёР· РїРµСЂРІРѕРіРѕ Р°СЂРіСѓРјРµРЅС‚Р°
            try:
                target_chat = int(arguments[1])
            except ValueError:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РєРѕРЅРєСЂРµС‚РЅС‹Р№ Р°Р№РґРё Р±РµСЃРµРґС‹!")
                return True

            # РџСЂРѕРІРµСЂРєР°: РµСЃР»Рё СЌС‚Рѕ Р±РµСЃРµРґР°, РїСЂРёР±Р°РІР»СЏРµРј 2000000000
            if target_chat > 0:
                target_peer = 2000000000 + target_chat
            else:
                target_peer = target_chat

            # РўРµРєСЃС‚ СЃРѕРѕР±С‰РµРЅРёСЏ вЂ” РІСЃС‘ РїРѕСЃР»Рµ РїРµСЂРІРѕРіРѕ Р°СЂРіСѓРјРµРЅС‚Р°
            text = " ".join(arguments[2:])
            if not text.strip():
                await message.reply("РЈРєР°Р¶РёС‚Рµ С‚РµРєСЃС‚ СЃРѕРѕР±С‰РµРЅРёСЏ!")
                return True

            try:
                await bot.api.messages.send(
                    peer_id=target_peer,
                    message=text,
                    random_id=0
                )
                await message.reply(f"РЎРѕРѕР±С‰РµРЅРёРµ СѓСЃРїРµС€РЅРѕ РѕС‚РїСЂР°РІР»РµРЅРѕ РІ С‡Р°С‚ ID {target_chat}.")
                await chats_log(user_id=user_id, target_id=None, role=None, log=f"РѕС‚РїСЂР°РІРёР»(-Р°) СЃРѕРѕР±С‰РµРЅРёРµ РІ С‡Р°С‚ В«{target_chat}В» РЎРѕРѕР±С‰РµРЅРёРµ: {text}")            
            except Exception as e:
                await message.reply(f"РџСЂРѕРёР·РѕС€Р»Р° РѕС€РёР±РєР° РїСЂРё РѕС‚РїСЂР°РІРєРµ: {e}")
                print(f"[say command] РћС€РёР±РєР° РѕС‚РїСЂР°РІРєРё РІ С‡Р°С‚ {target_chat}: {e}")
            return True
            
        # ---------------- GIVE ----------------
        if command in ["give", "РІС‹РґР°С‚СЊ"]:
            role = await get_role(user_id, chat_id)
            if role < 10:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!")
                return

            if chat_id == 23:
                await message.reply("Р”Р°РЅРЅР°СЏ Р±РµСЃРµРґР° РїСЂРѕРІРѕРґРёС‚СЃСЏ РІ СЃРїРµС†РёР°Р»РёР·РёСЂРѕРІР°РЅРЅРѕРј С‡Р°С‚Рµ, РєРѕС‚РѕСЂС‹Р№ РїСЂРµРґРЅР°Р·РЅР°С‡РµРЅ РёСЃРєР»СЋС‡РёС‚РµР»СЊРЅРѕ РґР»СЏ С‚РµСЃС‚РёСЂРѕРІС‰РёРєРѕРІ Р±РѕС‚Р°.\n\nР’ СЂР°РјРєР°С… РґР°РЅРЅРѕРіРѕ РѕР±СЃСѓР¶РґРµРЅРёСЏ РЅРµ РґРѕРїСѓСЃРєР°РµС‚СЃСЏ РёСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ РєРѕРјР°РЅРґ, РЅРµ РѕС‚РЅРѕСЃСЏС‰РёС…СЃСЏ Рє СЂР°Р±РѕС‚Рµ РїРѕ С‚РµСЃС‚РёСЂРѕРІР°РЅРёСЋ РёР»Рё С„СѓРЅРєС†РёРѕРЅРёСЂРѕРІР°РЅРёСЋ СЃРёСЃС‚РµРјС‹ РІ С†РµР»РѕРј.", disable_mentions=1)
                return True

            target = await extract_user_id(message)
            if not target:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ!")
                return

            if len(arguments) < 1:
                await message.reply("РЎСѓРјРјР° РґРѕР»Р¶РЅР° Р±С‹С‚СЊ С‡РёСЃР»РѕРј.")
                return

            try:
                amount = int(arguments[-1])
            except:
                await message.reply("РЎСѓРјРјР° РґРѕР»Р¶РЅР° Р±С‹С‚СЊ С‡РёСЃР»РѕРј.")
                return

            # РїРѕР»СѓС‡Р°РµРј Р±Р°Р»Р°РЅСЃ Рё РѕР±РЅРѕРІР»СЏРµРј
            balances = load_data(BALANCES_FILE)
            bal = balances.get(str(target), get_balance(target))
            bal["wallet"] += amount
            balances[str(target)] = bal
            await log_economy(user_id=user_id, target_id=target, amount=amount, log=f"РІС‹РґР°Р»(-Р°) {amount}$ РїРѕР»СЊР·РѕРІР°С‚РµР»СЋ {target}")          
            save_data(BALANCES_FILE, balances)

            try:
                s_info = await bot.api.users.get(user_ids=user_id)
                r_info = await bot.api.users.get(user_ids=target)
                s_name = f"{s_info[0].first_name} {s_info[0].last_name}"
                r_name = f"{r_info[0].first_name} {r_info[0].last_name}"
            except:
                s_name = str(user_id)
                r_name = str(target)

            await message.reply(
                f"[id{user_id}|{s_name}] РІС‹РґР°Р»(-Р°) В«{format_number(amount)}$В» РїРѕР»СЊР·РѕРІР°С‚РµР»СЋ [id{target}|{r_name}]"
            )
            return

        if command in ['getban', 'С‡РµРєР±Р°РЅ', 'РіРµС‚Р±Р°РЅ', 'checkban']:
            if await get_role(user_id, chat_id) < 1:
                await message.reply("РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїСЂР°РІ!", disable_mentions=1)
                return True

            # РџРѕР»СѓС‡Р°РµРј С†РµР»СЊ
            target = None
            if message.reply_message:
                target = message.reply_message.from_id
            elif message.fwd_messages and message.fwd_messages[0].from_id > 0:
                target = message.fwd_messages[0].from_id
            elif len(arguments) >= 2 and await getID(arguments[1]):
                target = await getID(arguments[1])
            else:
                await message.reply("РЈРєР°Р¶РёС‚Рµ РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ!", disable_mentions=1)
                return True

            # --- РџСЂРѕРІРµСЂРєР° РіР»РѕР±Р°Р»СЊРЅС‹С… Р±Р°РЅРѕРІ ---
            sql.execute("SELECT * FROM gbanlist WHERE user_id = ?", (target,))
            gbanlist = sql.fetchone()

            sql.execute("SELECT * FROM globalban WHERE user_id = ?", (target,))
            globalban = sql.fetchone()

            globalbans_chats = ""
            if globalban and gbanlist:
                gbanchats = f"@id{globalban[1]} (РњРѕРґРµСЂР°С‚РѕСЂ) | {globalban[2]} | {globalban[3]} РњРЎРљ (UTC+3)"
                gban_str = f"@id{gbanlist[1]} (РњРѕРґРµСЂР°С‚РѕСЂ) | {gbanlist[2]} | {gbanlist[3]} РњРЎРљ (UTC+3)"
                globalbans_chats = f"РРЅС„РѕСЂРјР°С†РёСЏ РѕР± РѕР±С‰РµР№ Р±Р»РѕРєРёСЂРѕРІРєРµ РІ Р±РµСЃРµРґР°С…:\n{gbanchats}\n\nРРЅС„РѕСЂРјР°С†РёСЏ РѕР± Р±Р»РѕРєРёСЂРѕРІРєРµ РІ Р±РµСЃРµРґР°С… РёРіСЂРѕРєРѕРІ:\n{gban_str}"
            elif globalban:
                gbanchats = f"@id{globalban[1]} (РњРѕРґРµСЂР°С‚РѕСЂ) | {globalban[2]} | {globalban[3]} РњРЎРљ (UTC+3)"
                globalbans_chats = f"РРЅС„РѕСЂРјР°С†РёСЏ РѕР± РѕР±С‰РµР№ Р±Р»РѕРєРёСЂРѕРІРєРµ РІ Р±РµСЃРµРґР°С…:\n{gbanchats}"
            elif gbanlist:
                gban_str = f"@id{gbanlist[1]} (РњРѕРґРµСЂР°С‚РѕСЂ) | {gbanlist[2]} | {gbanlist[3]} РњРЎРљ (UTC+3)"
                globalbans_chats = f"РРЅС„РѕСЂРјР°С†РёСЏ РѕР± Р±Р»РѕРєРёСЂРѕРІРєРµ РІ Р±РµСЃРµРґР°С… РёРіСЂРѕРєРѕРІ:\n{gban_str}"
            else:
                globalbans_chats = "Р‘Р»РѕРєРёСЂРѕРІРєР° РІРѕ РІСЃРµС… Р±РµСЃРµРґР°С… вЂ” РѕС‚СЃСѓС‚СЃС‚РІСѓРµС‚\nР‘Р»РѕРєРёСЂРѕРІРєР° РІ Р±РµСЃРµРґР°С… РёРіСЂРѕРєРѕРІ вЂ” РѕС‚СЃСѓС‚СЃС‚РІСѓРµС‚"

            # --- РџСЂРѕРІРµСЂРєР° Р±Р°РЅРѕРІ РІРѕ РІСЃРµС… С‡Р°С‚Р°С… ---
            sql.execute("SELECT chat_id FROM chats")
            chats_list = sql.fetchall()
            bans = ""
            count_bans = 0
            i = 1
            for c in chats_list:
                chat_id_check = c[0]
                try:
                    sql.execute(f"SELECT moder, reason, date FROM bans_{chat_id_check} WHERE user_id = ?", (target,))
                    user_bans = sql.fetchall()
                    if user_bans:
                        # РџРѕР»СѓС‡Р°РµРј РЅР°Р·РІР°РЅРёРµ Р±РµСЃРµРґС‹
                        rel_id = 2000000000 + chat_id_check
                        try:
                            resp = await bot.api.messages.get_conversations_by_id(peer_ids=rel_id)
                            if resp.items:
                                chat_title = resp.items[0].chat_settings.title or "Р‘РµР· РЅР°Р·РІР°РЅРёСЏ"
                            else:
                                chat_title = "Р‘РµР· РЅР°Р·РІР°РЅРёСЏ"
                        except:
                            chat_title = "РћС€РёР±РєР° РїРѕР»СѓС‡РµРЅРёСЏ РЅР°Р·РІР°РЅРёСЏ"

                        count_bans += 1
                        for ub in user_bans:
                            mod, reason, date = ub
                            bans += f"{i}) {chat_title} | @id{mod} (РњРѕРґРµСЂР°С‚РѕСЂ) | {reason} | {date} РњРЎРљ (UTC+3)\n"
                            i += 1
                except:
                    continue  # РµСЃР»Рё С‚Р°Р±Р»РёС†С‹ РЅРµС‚, РїСЂРѕРїСѓСЃРєР°РµРј
                                       
            if count_bans == 0:
                bans_chats = "Р‘Р»РѕРєРёСЂРѕРІРєРё РІ Р±РµСЃРµРґР°С… РѕС‚СЃСѓС‚СЃС‚РІСѓСЋС‚"
            else:
                bans_chats = f"РљРѕР»РёС‡РµСЃС‚РІРѕ Р±РµСЃРµРґ, РІ РєРѕС‚РѕСЂС‹С… Р·Р°Р±Р»РѕРєРёСЂРѕРІР°РЅ РїРѕР»СЊР·РѕРІР°С‚РµР»СЊ: {count_bans}\nРРЅС„РѕСЂРјР°С†РёСЏ Рѕ Р±Р°РЅР°С… РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ:\n{bans}"

            # --- РС‚РѕРіРѕРІРѕРµ СЃРѕРѕР±С‰РµРЅРёРµ ---
            await message.reply(
                f"РРЅС„РѕСЂРјР°С†РёСЏ Рѕ Р±Р»РѕРєРёСЂРѕРІРєР°С… @id{target} (РџРѕР»СЊР·РѕРІР°С‚РµР»СЊ)\n\n"
                f"{globalbans_chats}\n\n"
                f"{bans_chats}",
                disable_mentions=1
            )

            await chats_log(
                user_id=user_id,
                target_id=target,
                role=None,
                log=f"РїРѕСЃРјРѕС‚СЂРµР»(-Р°) СЃРїРёСЃРѕРє Р±Р»РѕРєРёСЂРѕРІРѕРє @id{target} (РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ)"
            )
            return True
                        
        # ---------------- RESETMONEY ----------------
        if command in ["resetmoney", "Р°РЅСѓР»РёСЂРѕРІР°С‚СЊ", "Р
