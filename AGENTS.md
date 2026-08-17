# AGENTS.md — Railway 3x-ui Bot

راهنمای زمینه برای دستیارهای هوشمند/ای‌‌های کدینگ که روی این ریپو کار می‌کنند.

## نمای کلی پروژه

ربات تلگرام (Python) که ساخت زیرساخت 3x-ui **چند-ریجن روی Railway** را کاملاً خودکار
می‌کند: ساخت پروژه + ۴ سرویس، اتصال نودها، اینباند VLESS+Reality، TCP proxy با
روتیت، و تحویل لینک اتصال. چند اکانت Railway به‌طور هم‌زمان مدیریت می‌شود.

## پشته‌ی تکنولوژی

- **زبان:** Python 3.12
- **فریم‌ورک ربات:** `python-telegram-bot` v21 (async)
- **حافظه:** ماژول-گلوبال `USERS` + پایش روی ولوم `/data/state.json`
- **API Railway:** GraphQL (backboard.railway.com/graphql/v2) با urllib (بدون SDK)
- **رمزنگاری (اختیاری):** `cryptography` (Fernet) — `STATE_SECRET`
- **تست:** pytest (غیروابسته، با مونکی‌پچ `bot.gql` / `deploy.gql`)
- **CI:** GitHub Actions (pytest روی push/PR)

## ساختار پوشه

```
railway-bot/
├── bot.py                  # کل منطق ربات (هندلرها، state، استایل UI)
├── scripts/
│   ├── deploy.py            # ساخت پروژه + ۴ سرویس + دامنه + ولوم
│   ├── xui-node-connector.py# اتصال نودهای راه‌دور به پنل مرکزی (xui-nl)
│   ├── xui-reality-inbound.py# اینباند VLESS+Reality (کلید مشترک)
│   ├── xui-tcp-proxy-setup.py# TCP proxy + روتیت به دامنه خوب + Host ها
│   ├── xui-link-maker.py    # ساخت لینک‌های VLESS از SERVERS_JSON
│   ├── xui-client-create.py # ساخت کلاینت + لینک ساب
│   └── rotate-proxies.py    # چرخش TCP proxy (هوشمند/رندوم)
├── tests/test_bot.py        # تست‌های واحد
└── Dockerfile
```

## قراردادها و نکات مهم

1. **ربات مونولیت در یک فایل `bot.py`** (~۱۸۰۰ خط) — عمداً تک‌فایلی نگه داشته شده.
   اگر بخشی را جدا کردی، `from telegram...` و توابع کمکی مشترک (`card`, `esc`,
   `bar`, `gql`, `get_acc`) را نشکن. اسکریپت‌ها به‌صورت پروسه‌ی جدا و async صدا
   زده می‌شوند؛ قرارداد خروجی آن‌ها:
   - `PROXY_RESULT: <region>=<host>:<port>` (برای چرخش)
   - `SUB_LINK_HTTPS=...` و `UUID=...` (برای ساخت کلاینت)
   - `✅ دامنه: https://...` (برای دامنه‌های سرویس)
2. **State machine:** کلیدهای `awaiting_*` (acc_name / acc_token / client_name /
   uuid) علامتی برای دریافت پیام بعدی کاربراند و **هنگام مصرف پاک می‌شوند** —
   وقتی تغییری در این‌ها دادی، در همان مصرف `save_state()` را صدا بزن.
3. **callback_data تلگرام حداکثر ۶۴ بایت** — اسم‌های فارسی باید با کلید کوتاه
   (`a1,a2,…` در `user['cb_names']`) نگاشت شوند؛ مستقیم اسم در callback نگذار.
4. **ویسازی مستمر:** همه‌ی ورودی کاربر و خروجی اسکریپت موقع نمایش باید از
   `esc()` رد شود (HTML parse). هرگز متن خام کاربر را در پیام HTML قالب‌بندی نکن.
5. احراز هویت با `@require_auth` (دکوراتور) یا `is_allowed()` + `q.answer` برای
   callbackها. هندلرهای جدید هم باید همین گارد را داشته باشند.
6. **سقف Railway:** اکانت‌های جدید ۲۵ ساخت سرویس در روز. `find_or_create_project`
   و `deploy.create_service` باید پروژه/سرویس ناقص قبلی را بازیافت کنند — این منطق را
   در هنگام ساخت، دور نزن.
7. **توکن‌ها:** هرگز در لاگ چاپ نکن. هنگام ذخیره روی دیسک، اگر `_CIPHER` فعال بود
   توکن از طریق `_enc_state` رمزنگاری می‌شود — آزمون `state.json` را که می‌نویسی
   با ذهنیت رمزنگاری (کلید `token_e`) بنویس.

## اجرا

```bash
pip install -r requirements.txt
BOT_TOKEN=xxx python bot.py            # محلی
pytest tests/ -q                       # تست‌ها
```

متغیر الزامی فقط `BOT_TOKEN` است؛ بقیه (ولوم `/data`, ALLOWED_UID و…) اختیاری‌اند.