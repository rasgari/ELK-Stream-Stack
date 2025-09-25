# elasticsearch


به‌صورت پیش‌فرض Elasticsearch از پورت 9200 استفاده می‌کند

تنظیم elasticsearch.yml برای اجازه دسترسی بیرونی
فایل تنظیمات معمولاً در مسیر زیره:

```
C:\Program Files\Elastic\Elasticsearch\config\elasticsearch.yml
```


======================================================================


باز کردن پورت در فایروال ویندوز
به‌صورت پیش‌فرض Elasticsearch از پورت 9200 استفاده می‌کند (و گاهی 9300 برای nodeها):

مراحل:
وارد Windows Defender Firewall بشو.

گزینه‌ی Advanced Settings را بزن.

در بخش Inbound Rules:

روی "New Rule" بزن.

گزینه‌ی Port را انتخاب کن.

پروتکل TCP و پورت 9200 را وارد کن.

Allow the connection را انتخاب کن.

نام مناسبی مثل "Elasticsearch 9200" بده.
اگه میخواید به بیرون شبکه راه داشته باشه تو elasticsearch.config اینها را از کامنت خارج کنید و فایروال هم غیرفعال کنید

```bash
network.host: 0.0.0.0
http.port: 9200
```


⚠️ هشدار امنیتی مهم
دسترسی مستقیم Elasticsearch به اینترنت خیلی خطرناکه چون هیچ‌گونه احراز هویتی نداره به‌صورت پیش‌فرض.

راهکارهای امنیتی:
حتماً از reverse proxy با authentication استفاده کن (مثل nginx + basic auth)

یا حداقل با فایروال فقط IPهایی خاص رو اجازه بده

بهتره اصلاً مستقیم به اینترنت ندیدش و از VPN یا SSH tunnel استفاده کنید.

======================================================================

از Services ویندوز → Elasticsearch → Restart

یا از CMD:

```
net stop elasticsearch
net start elasticsearch
```
