فعال‌سازی SSL/TLS در Elasticsearch (به‌صورت کامل)

Elasticsearch از نسخه 7.0 به بعد قابلیت امنیت و SSL را به‌صورت داخلی دارد.
در نسخه 8.x به‌صورت پیش‌فرض فعال است فقط باید certificate بسازی.

⚡ بخش ۱ — فعال‌سازی SSL در Elasticsearch نسخه 8.x (پیشنهادی)

Elasticsearch 8.x به‌صورت پیش‌فرض security فعال است.
فقط کافیست certificate بسازی.

🔧 مرحله ۱ — تولید CA و Cert ها

در سرور Elasticsearch اجرا کن:
```
./bin/elasticsearch-certutil ca
./bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12
```

این دو فایل ساخته می‌شوند:
```
elastic-stack-ca.p12 → CA
```
```
elastic-certificates.p12 → certificate اصلی
```
فایل‌ها را در مسیر /etc/elasticsearch/certs قرار بده.

🔧 مرحله ۲ — ویرایش elasticsearch.yml

فایل:
```
/etc/elasticsearch/elasticsearch.yml
```

محتوای زیر را اضافه کن:
```
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: certs/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: certs/elastic-certificates.p12

xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: certs/elastic-certificates.p12
xpack.security.http.ssl.truststore.path: certs/elastic-certificates.p12
```
🔧 مرحله ۳ — ریستارت Elasticsearch
```
systemctl restart elasticsearch
```
🔧 مرحله ۴ — تست اتصال HTTPS
```
curl -k https://localhost:9200
```

اگر Username/Password خواست:
```
user: elastic
```
pass: موجود در /var/lib/elasticsearch/ یا هنگام نصب تولید شده

⚡ بخش ۲ — فعال‌سازی SSL در Elasticsearch نسخه 7.x

در نسخه 7 نیز باید certificate بسازی:

🔧 مرحله ۱ — ساخت CA
```
./bin/elasticsearch-certutil ca
```
🔧 مرحله ۲ — ساخت certificate برای Nodeها
```
./bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12
```
🔧 مرحله ۳ — فعال‌سازی SSL در elasticsearch.yml
```
xpack.security.enabled: true

xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: certs/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: certs/elastic-certificates.p12

xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: certs/elastic-certificates.p12
```
🔧 مرحله ۴ — ریستارت
```
systemctl restart elasticsearch
```
⚡ بخش ۳ — فعال‌سازی SSL برای Kibana

در Kibana (kibana.yml):
```
server.ssl.enabled: true
server.ssl.certificate: /etc/kibana/certs/server.crt
server.ssl.key: /etc/kibana/certs/server.key

elasticsearch.ssl.verificationMode: none
```

ریستارت:
```
systemctl restart kibana
```
⚡ بخش ۴ — نکات مهم امنیتی
🔐 امنیت HTTP

اگر HTTPS فعال نشود، یوزرنیم و پسورد به‌صورت plain ارسال می‌شود!

🔐 امنیت Cluster

Elasticsearch بدون SSL کشتی‌اش ترک می‌خورد؛ چون تمام Nodeها با هم بدون رمز ارتباط می‌گیرند.

🔐 امنیت Kibana

بدون SSL هر کسی ترافیک را Sniff کند، Session Token را می‌دزدد.
