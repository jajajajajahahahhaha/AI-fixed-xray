# 🔧 رفع مشکل هسته Xray

## مشکل
پس از build جدید Xray-core (نسخه v1.8+ و بالاتر — در این ریپو v26.3.27)، هسته Xray دیگر پروتکل قدیمی **mtproto** را داخل خودش پشتیبانی نمی‌کند. در فایل `panel_db.json`، کاربر `admin` با تنظیم `is_proxy_type: true, proxy_protocol: "mtproto"` تعریف شده بود.

نتیجه: هر بار `sync_xray_core()` یک `mtproto inbound` و `mtproto outbound` به کانفیگ اضافه می‌کرد. Xray در startup این خطا را می‌داد:

```
Failed to start: main: failed to load config files:
infra/conf: failed to build inbound config with tag tg-in >
infra/conf: failed to load inbound detour config for protocol mtproto >
infra/conf: unknown config id: mtproto
```

و اصلاً بالا نمی‌آمد. `xray_live` همیشه `False` بود.

## راه‌حل
در تابع `sync_xray_core()` (فایل `analytics_worker.py` — حدود خط 850):

- متغیر `has_mtproto_proxy` **همیشه** به `False` مقداردهی می‌شود
- در نتیجه بلاک `if has_mtproto_proxy:` که inbound و outbound `mtproto` را اضافه می‌کرد، دیگر اجرا نمی‌شود
- کاربران mtproto **در UI و DB دست‌نخورده باقی می‌مانند** — لینک ساب اشتراکشان همچنان تولید می‌شود (چون آن مسیر مستقل از Xray است)
- فقط دیگر یک inbound mtproto در Xray ثبت نمی‌شود، پس کانفیگ Xray معتبر می‌شود و هسته سالم استارت می‌شود

## بهبود ثانویه
قبل از استارت Xray، دستور `xray -test` اجرا می‌شود. اگر کانفیگ invalid بود، خطای دقیق در لاگ و کانال تلگرام گزارش می‌شود. این کمک می‌کند در آینده اگر پروتکل دیگری از Xray حذف شد، سریع تشخیص داده شود.

## UI
**هیچ تغییری** در UI ایجاد نشده. تمام تب‌ها، فرم‌ها و ویژگی‌های پنل دقیقاً مثل قبل هستند.

## تست
- ✅ `xray -config config.json -test` → `Configuration OK`
- ✅ فرآیند xray روی PID مشخص روی پورت 8085 (VLESS/WS) و 8089 (SOCKS) گوش می‌دهد
- ✅ API `/api/stats` مقدار `xray_live: True` برمی‌گرداند
- ✅ Watchdog بعد از kill دستی، در کمتر از 8 ثانیه هسته را خودکار احیا می‌کند
