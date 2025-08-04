# Toktokbit
متابعة تيك تيك 
import os, json, time, pickle
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.chrome.service import Service
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, ContextTypes, filters

إعداد

BOT_TOKEN = "8233070429:AAHVBoHWSnbw6odpskMpvIyRWUJXRHv8OtE"
COOKIES_DIR = "cookies"
ACCOUNTS_FILE = "accounts.json"

⬛ إنشاء المتصفح

def create_driver():
options = Options()
options.add_argument("--start-maximized")
service = Service("chromedriver")
return webdriver.Chrome(service=service, options=options)

⬛ تحميل الحسابات

def load_accounts():
if not os.path.exists(ACCOUNTS_FILE):
return []
with open(ACCOUNTS_FILE, "r") as f:
return json.load(f)

⬛ حفظ الحسابات

def save_accounts(accounts):
with open(ACCOUNTS_FILE, "w") as f:
json.dump(accounts, f)

⬛ حفظ الكوكيز

def save_cookies(driver, username):
os.makedirs(COOKIES_DIR, exist_ok=True)
with open(f"{COOKIES_DIR}/{username}.pkl", "wb") as f:
pickle.dump(driver.get_cookies(), f)

⬛ تحميل الكوكيز

def load_cookies(driver, username):
path = f"{COOKIES_DIR}/{username}.pkl"
if os.path.exists(path):
driver.get("https://www.tiktok.com/")
with open(path, "rb") as f:
cookies = pickle.load(f)
for cookie in cookies:
driver.add_cookie(cookie)
driver.refresh()
return True
return False

⬛ تنفيذ متابعة أو إلغاء متابعة

def do_action(driver, target, follow=True):
driver.get(f"https://www.tiktok.com/@{target}")
time.sleep(5)
try:
if follow:
btn = driver.find_element("xpath", '//button[contains(text(),"Follow")]')
else:
btn = driver.find_element("xpath", '//button[contains(text(),"Following")]')
btn.click()
return f"✅ {'متابعة' if follow else 'إلغاء'} @{target}"
except Exception as e:
return f"❌ خطأ: {str(e)}"

🟩 بدء البوت

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
await update.message.reply_text("🤖 أرسل:\n- تسجيل @username\n- تابع @user\n- الغِ @user\n- عدد\n- حذف @username")

🟩 التعامل مع الرسائل

async def handle(update: Update, context: ContextTypes.DEFAULT_TYPE):
text = update.message.text.strip()
accounts = load_accounts()

# إضافة حساب  
if text.startswith("تسجيل @"):  
    username = text.replace("تسجيل @", "").strip()  
    if any(a["username"] == username for a in accounts):  
        await update.message.reply_text("⚠️ الحساب موجود مسبقًا.")  
        return  
    driver = create_driver()  
    driver.get("https://www.tiktok.com/login")  
    await update.message.reply_text(f"🔐 افتح المتصفح وسجّل الدخول يدويًا للحساب {username} ثم اضغط Enter هنا.")  
    input("✅ بعد تسجيل الدخول اضغط Enter...")  
    save_cookies(driver, username)  
    driver.quit()  
    accounts.append({"username": username})  
    save_accounts(accounts)  
    await update.message.reply_text(f"✅ تم حفظ الحساب {username}.")  

# متابعة أو إلغاء  
elif text.startswith("تابع @") or text.startswith("الغِ @"):  
    follow = text.startswith("تابع")  
    target = text.replace("تابع @", "").replace("الغِ @", "").strip()  
    results = []  
    for acc in accounts:  
        driver = create_driver()  
        if not load_cookies(driver, acc["username"]):  
            await update.message.reply_text(f"⚠️ الحساب {acc['username']} غير مسجّل دخول.")  
            driver.quit()  
            continue  
        result = do_action(driver, target, follow=follow)  
        results.append(f"{acc['username']}: {result}")  
        driver.quit()  
    await update.message.reply_text("\n".join(results))  

# عرض عدد الحسابات  
elif text == "عدد":  
    await update.message.reply_text(f"📊 عدد الحسابات: {len(accounts)}")  

# حذف حساب  
elif text.startswith("حذف @"):  
    username = text.replace("حذف @", "").strip()  
    accounts = [a for a in accounts if a["username"] != username]  
    save_accounts(accounts)  
    path = f"{COOKIES_DIR}/{username}.pkl"  
    if os.path.exists(path):  
        os.remove(path)  
    await update.message.reply_text(f"🗑️ تم حذف الحساب {username}")  

else:  
    await update.message.reply_text("❓ أمر غير معروف")

✅ تشغيل البوت

def main():
app = ApplicationBuilder().token(BOT_TOKEN).build()
app.add_handler(CommandHandler("start", start))
app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle))
app.run_polling()

if name == "main":
main()

