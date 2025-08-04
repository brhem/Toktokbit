import os, json, time, pickle
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.chrome.service import Service
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, ContextTypes, filters

# إعداد
BOT_TOKEN = "8175697399:AAGClAMq4VV6Uw7gjdhuS3r20dKrO8beyn8"
COOKIES_DIR = "cookies"
ACCOUNTS_FILE = "accounts.json"

# ⬛ إنشاء المتصفح
def create_driver():
    options = Options()
    options.add_argument("--start-maximized")
    service = Service("chromedriver")  # تأكد أن chromedriver بجانب الملف
    return webdriver.Chrome(service=service, options=options)

# ⬛ تحميل الحسابات
def load_accounts():
    if not os.path.exists(ACCOUNTS_FILE):
        return []
    with open(ACCOUNTS_FILE, "r") as f:
        return json.load(f)

# ⬛ حفظ الحسابات
def save_accounts(accounts):
    with open(ACCOUNTS_FILE, "w") as f:
        json.dump(accounts, f)

# ⬛ حفظ الكوكيز
def save_cookies(driver, username):
    os.makedirs(COOKIES_DIR, exist_ok=True)
    with open(f"{COOKIES_DIR}/{username}.pkl", "wb") as f:
        pickle.dump(driver.get_cookies(), f)

# ⬛ تحميل الكوكيز
def load_cookies(driver, username):
    path = f"{COOKIES_DIR}/{username}.pkl"
    if os.path.exists(path):
        driver.get("https://www.tiktok.com/")
        with open(path, "rb") as f:
            cookies = pickle.load(f)
        for cookie in cookies:
            try:
                driver.add_cookie(cookie)
            except:
                pass
        driver.refresh()
        time.sleep(3)
        return True
    return False

# ⬛ تنفيذ متابعة أو إلغاء متابعة
def do_action(driver, target, follow=True):
    driver.get(f"https://www.tiktok.com/@{target}")
    time.sleep(5)
    try:
        if follow:
            btn = driver.find_element("xpath", '//button[contains(text(),"Follow")]')
        else:
            btn = driver.find_element("xpath", '//button[contains(text(),"Following")]')
        btn.click()
        time.sleep(2)
        return f"✅ {'تمت متابعة' if follow else 'تم إلغاء متابعة'} @{target}"
    except Exception as e:
        return f"❌ فشل: {str(e)}"

# ⬛ تسجيل الدخول تلقائيًا
def login_with_credentials(driver, username, password):
    driver.get("https://www.tiktok.com/login/phone-or-email/email")
    time.sleep(5)
    try:
        user_input = driver.find_element("name", "email")
        user_input.send_keys(username)
        time.sleep(1)

        pass_input = driver.find_element("name", "password")
        pass_input.send_keys(password)
        time.sleep(1)

        login_button = driver.find_element("xpath", '//button[@type="submit"]')
        login_button.click()

        time.sleep(10)
        if "login" not in driver.current_url:
            return True
        else:
            return False
    except Exception as e:
        print("خطأ أثناء تسجيل الدخول:", e)
        return False

# 🟩 أمر /start
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "🤖 مرحباً بك!\n\n"
        "الأوامر:\n"
        "- تسجيل @username ← تسجيل دخول يدوي\n"
        "- دخول username|password ← تسجيل تلقائي\n"
        "- تابع @user ← متابعة\n"
        "- الغِ @user ← إلغاء متابعة\n"
        "- عدد ← عدد الحسابات\n"
        "- حذف @username ← حذف حساب"
    )

# 🟩 التعامل مع الرسائل
async def handle(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text.strip()
    accounts = load_accounts()

    if text.startswith("تسجيل @"):
        username = text.replace("تسجيل @", "").strip()
        if any(a["username"] == username for a in accounts):
            await update.message.reply_text("⚠️ الحساب موجود مسبقًا.")
            return
        driver = create_driver()
        driver.get("https://www.tiktok.com/login")
        await update.message.reply_text(f"🔐 افتح المتصفح وسجّل الدخول يدويًا للحساب {username} ثم اضغط Enter في الطرفية.")
        input("🖱️ اضغط Enter بعد تسجيل الدخول...")
        save_cookies(driver, username)
        driver.quit()
        accounts.append({"username": username})
        save_accounts(accounts)
        await update.message.reply_text(f"✅ تم حفظ الحساب {username}.")

    elif text.startswith("دخول "):
        try:
            creds = text.replace("دخول ", "").strip()
            username, password = creds.split("|")
            driver = create_driver()
            ok = login_with_credentials(driver, username, password)
            if ok:
                save_cookies(driver, username)
                accounts.append({"username": username})
                save_accounts(accounts)
                await update.message.reply_text(f"✅ تم تسجيل الدخول وحفظ الحساب {username}")
            else:
                await update.message.reply_text("❌ فشل في تسجيل الدخول، تحقق من كلمة السر أو وجود حماية.")
            driver.quit()
        except Exception as e:
            await update.message.reply_text(f"❌ خطأ: {str(e)}")

    elif text.startswith("تابع @") or text.startswith("الغِ @"):
        follow = text.startswith("تابع")
        target = text.replace("تابع @", "").replace("الغِ @", "").strip()
        results = []
        for acc in accounts:
            driver = create_driver()
            if not load_cookies(driver, acc["username"]):
                results.append(f"⚠️ {acc['username']} غير مسجل دخول")
                driver.quit()
                continue
            result = do_action(driver, target, follow=follow)
            results.append(f"{acc['username']}: {result}")
            driver.quit()
        await update.message.reply_text("\n".join(results))

    elif text == "عدد":
        await update.message.reply_text(f"📊 عدد الحسابات: {len(accounts)}")

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

# ✅ تشغيل البوت
def main():
    app = ApplicationBuilder().token(BOT_TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle))
    app.run_polling()

if __name__ == "__main__":
    main()