# edu docker-compose

راه‌اندازی Elasticsearch با SSL (TLS) فعال قرار داده‌ام.
این فایل شامل مراحل کلیدی زیر است:

تولید خودکار گواهی‌های SSL با elasticsearch-certutil

فعال‌سازی TLS برای Transport و HTTP

یک نود Elasticsearch با یوزر و پسورد

ذخیره گواهی‌ها در یک volume دائمی

بعد از آن داکر کامپوز اجرا می کنی

---

🔧 مراحل اجرا
1️⃣ ساخت پوشه برای گواهی‌ها
```
mkdir certs
```
2️⃣ اجرا کردن docker compose
```
docker compose up -d
```
🔍 تست SSL
تست HTTPS با اعتبارنامه:
```
curl -k -u elastic:StrongPass123! https://localhost:9200
```
انتظار خروجی:
```
{
  "name" : "es-ssl",
  "cluster_name" : "ssl-cluster",
  "version" : {...}
}
```

---


TLS/SSL در Elasticsearch — مفاهیم کلیدی

Elasticsearch دو مسیر ترافیک دارد:

Transport layer (پورت 9300) — برای ارتباط بین نودها (cluster internal). باید TLS فعال باشد در محیط‌های تولید (برای جلوگیری از شنود و Spoofing).

HTTP layer (پورت 9200) — برای REST API (کلاینت‌ها، Kibana، curl و غیره). با TLS فعال می‌شود که همان HTTPS است.

از نسخه‌ی Elasticsearch 8.x به بعد، security (و TLS برای transport و HTTP) به‌صورت پیش‌فرض فعال یا بسیار ساده‌تر قابل فعال‌سازی است. (در 7.x یا قبل‌تر باید xpack.security.enabled: true را صریحاً قرار دهی).

برای TLS نیاز داری گواهی + کلید برای نود (یا keystore/truststore بسته به روش).

۳) چطور SSL/TLS را برای Elasticsearch کانفیگ کنیم — روش کلی (مفید برای سرور یا داکر)

در ادامه یک مسیر عملی و پی‌درپی می‌آورم؛ دو روش تولید گواهی هم پوشش می‌دهم: با elasticsearch-certutil (راحت و مخصوص ES) و با OpenSSL (عمومی).

روش A — تولید گواهی با elasticsearch-certutil (پیشنهاد برای تست/محیط داخلی)

داخل سروری که تصویر ES دارد یا محیطی با همان نسخه‌ی Elasticsearch یک کانتینر موقتی اجرا کن:
```
docker run --rm -it -v $(pwd)/certs:/certs docker.elastic.co/elasticsearch/elasticsearch:8.14.0 bash
```
# یا اگر روی خود سرور نصب است، در مسیر bin/elasticsearch-certutil


ساختن CA:
```
elasticsearch-certutil ca --silent --pem --out /certs/ca.zip
unzip /certs/ca.zip -d /certs
```
# نتیجتاً /certs/ca/ca.crt و ca.key ساخته می‌شود


ساخت گواهی برای نود (اطمینان بده SAN شامل localhost هست اگر قرار است از localhost استفاده کنی):
```
elasticsearch-certutil cert --silent --pem \
  --ca-cert /certs/ca/ca.crt \
  --ca-key /certs/ca/ca.key \
  --name es-node \
  --dns localhost --ip 127.0.0.1 \
  --out /certs/es-node.zip

unzip /certs/es-node.zip -d /certs/es-node
```
# در خروجی es-node/، es-node.crt و es-node.key داریم


انتقال گواهی‌ها به مسیر config ES (مثلاً /etc/elasticsearch/certs یا در داکر /usr/share/elasticsearch/config/certs) و تنظیم مالکیت/پرمیژن.

روش B — تولید گواهی با OpenSSL (برای کسانی که کنترل کامل می‌خواهند)

نمونه خیلی خلاصه:
```
# 1. CA key & cert
openssl genpkey -algorithm RSA -out ca.key -pkeyopt rsa_keygen_bits:4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt -subj "/CN=MyLocalCA"
```
# 2. سرور key & CSR
```
openssl genpkey -algorithm RSA -out es.key -pkeyopt rsa_keygen_bits:2048
openssl req -new -key es.key -out es.csr -subj "/CN=localhost"

# 3. ایجاد فایل ext برای SAN
cat > san.ext <<EOF
subjectAltName = DNS:localhost,IP:127.0.0.1
EOF

# 4. امضای CSR با CA
openssl x509 -req -in es.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out es.crt -days 365 -sha256 -extfile san.ext

```
توجه: SAN باید شامل همه‌ی نام‌ها/آی‌پی‌هایی باشد که client با آن‌ها وصل می‌شود (مثلاً myhost.example.com).

۵) کانفیگ elasticsearch.yml

فایل elasticsearch.yml را ویرایش کن (یا مقادیر محیطی در داکر) — مثال ساده:
```
# Security basics
xpack.security.enabled: true

# HTTP layer (REST/Client)
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.key: /usr/share/elasticsearch/config/certs/es.key
xpack.security.http.ssl.certificate: /usr/share/elasticsearch/config/certs/es.crt
xpack.security.http.ssl.certificate_authorities: [ "/usr/share/elasticsearch/config/certs/ca.crt" ]

# Transport layer (node-to-node)
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.key: /usr/share/elasticsearch/config/certs/es.key
xpack.security.transport.ssl.certificate: /usr/share/elasticsearch/config/certs/es.crt
xpack.security.transport.ssl.certificate_authorities: [ "/usr/share/elasticsearch/config/certs/ca.crt" ]

# Optional network binding (برای تست محلی)
network.host: 0.0.0.0
http.port: 9200
```

نکته‌ها:

اگر از keystore/java keystore استفاده می‌کنی روش‌ها متفاوت است؛ اما مسیرهای PEM بالا برای نصب مستقیم در اکثر توزیع‌ها کار می‌کنند.

برای multi-node باید هر نود گواهی‌ای داشته باشد که SAN مربوط به hostname آن را داشته باشد یا از wildcard استفاده شود.

۴) آیا localhost:9200 برای SSL کافی است؟

از نظر اتصال محلی (local testing): بله — اگر گواهی سرور شامل localhost (یا 127.0.0.1) در SAN باشد، می‌توانی از https://localhost:9200 استفاده کنی و TLS برقرار خواهد شد.

نکته مهم در مورد گواهی‌های self-signed یا خود-CA:

اگر از CA داخلی یا self-signed استفاده می‌کنی، کلاینت (مثلاً curl یا مرورگر یا Kibana) باید CA root را به لیست CAهای مورد اعتماد خود اضافه کند تا خطای certificate trust نداشته باشی.

در تست می‌توانی از curl -k یا curl --insecure استفاده کنی که تست را آسان می‌کند ولی برای production این کار غیر‌امن است.

برای دسترسی از دیگر ماشین‌ها:

گواهی باید SAN شامل نام دامنه یا آدرس IP ای باشد که کلاینت از آن استفاده می‌کند.

اگر گواهی فقط localhost دارد، از دیگر ماشین‌ها که با myserver.example.com وصل شوند، خطای mismatch خواهید دید.

۵) چگونه بفهمم Elasticsearch با SSL فعال شده (چک‌لیست تست و فرمان‌ها)

در ادامه چند روش عملی برای اطمینان:

۱) تست ساده با curl

اگر CA را به کلاینت اضافه نکردی (self-signed) و فقط می‌خواهی ببینی TLS دارد:
```
curl -k -u elastic:YOURPASSWORD https://localhost:9200/
```

اگر CA را نصب کردی:
```
curl --cacert /path/to/ca.crt -u elastic:YOURPASSWORD https://localhost:9200/
```

اگر SSL فعال نباشد، اتصال HTTPS شکست می‌خورد یا به HTTP redirect خواهد شد. اگر موفق شود، خروجی JSON وضعیت کلاستر را می‌گیری:
```
{
  "name" : "es-node",
  "cluster_name" : "cluster",
  "cluster_uuid" : "...",
  "version" : { ... },
  "tagline" : "You Know, for Search"
}
```
۲) دیدن certificate با openssl s_client

این فرمان نشان می‌دهد TLS handshake انجام شده و گواهی برگشت داده می‌شود:
```
openssl s_client -connect localhost:9200 -showcerts
```

اگر TLS فعال باشد، خروجی شامل certificate chain و جزئیات CN/SAN خواهد بود.

اگر اتصال برقرار نشود یا متن خطا نشان دهد no peer certificate یا handshake failed، یعنی TLS فعال نیست یا پورت اشتباه است.

۳) مطالعه لاگ‌های Elasticsearch

در لاگ‌ها (معمولاً /var/log/elasticsearch/elasticsearch.log یا در container logs) وقتی HTTP یا transport با TLS بالا می‌آید پیام‌هایی می‌بینید مانند:
```
HTTP server started [http://0.0.0.0:9200] — (اگر TLS غیرفعال)
```
یا پیامی درباره TLS/SSL initialization یا Transport SSL context initialized و مشابه آن در نسخه‌های مختلف.
لاگ دقیقا متن متفاوت دارد ولی در زمان بالا آمدن نود پیام مرتبط با Certificate loading یا enabling TLS خواهد آمد.

۴) بررسی پورت‌ها با netstat/ss
```
ss -tlnp | grep 9200
# یا
netstat -tlnp | grep 9200
```

این فقط نشان می‌دهد سرویس به پورت گوش می‌دهد؛ خودش TLS را اثبات نمی‌کند اما ترکیب با openssl s_client تأیید نهایی است.

۵) از طریق مرورگر یا Kibana

باز کردن https://localhost:9200 در مرورگر: اگر گواهی معتبر توسط مرورگر trust شده باشد، آدرس با قفل سبز/قفل نشان داده می‌شود؛ اگر self-signed باشد، مرورگر هشدار می‌دهد.

Kibana: برای استفاده Kibana از Elasticsearch با HTTPS باید kibana.yml را طوری تنظیم کنی که به https://...:9200 وصل شود و یا CA را trust کند.

۶) مثال‌های عملی — تست‌های رایج
مثال ۱ — تست با curl و CA
```
curl --cacert ./certs/ca.crt -u elastic:StrongPass123! https://localhost:9200/_cluster/health?pretty
```

اگر خروجی JSON با اطلاعات health برگشت، SSL فعال و اعتبار CA صحیح است.

مثال ۲ — تست با curl بدون اعتماد (برای چک سریع)
```
curl -k -u elastic:StrongPass123! https://localhost:9200/
# یا
curl -k https://localhost:9200/
```
مثال ۳ — گرفتن certificate با openssl
```
openssl s_client -connect localhost:9200 -servername localhost </dev/null 2>/dev/null | openssl x509 -noout -text
```

این اطلاعات SAN، validity و issuer را نشان می‌دهد.

۷) مشکلات رایج و رفع آن‌ها

خطای CN/SAN mismatch
راه‌حل: هنگام ساخت یا امضای گواهی، SAN را شامل نام/IP مورد استفاده قرار بده.

کلاینت خطای trust می‌دهد (self-signed)
راه‌حل: CA را به کلاینت اضافه کن یا از CA معتبر (Let's Encrypt / internal PKI) استفاده کن.

کلید با پسورد/رمزگذاری شده
اگر کلید خصوصی با passphrase باشد، Elasticsearch ممکن است نتواند آن را بدون وارد کردن رمز باز کند. برای تولید خودکار بهتر از PEM بدون passphrase یا keystore استفاده کن.

پرمیشن فایل‌ها
مطمئن شو فایل‌های key/crt فقط توسط کاربر elasticsearch قابل خواندن است (امنیت).

Transport mismatch (certs not trusted between nodes)
برای cluster، همه نودها باید CA یا certificate chain معتبری داشته باشند تا node-to-node TLS کار کند.

۸) خلاصه و چک‌لیست سریع برای راه‌اندازی کامل

تصمیم بگیر: از CA محلی (self) استفاده می‌کنی یا CA عمومی.

تولید CA و گواهی‌ سرور (مطابق SAN).

کپی فایل‌ها به /usr/share/elasticsearch/config/certs یا مسیر مناسب و تنظیم مالکیت.

ویرایش elasticsearch.yml برای فعال‌سازی xpack.security.http.ssl.* و xpack.security.transport.ssl.*.

ری‌استارت Elasticsearch.

تست با openssl s_client و curl --cacert و چک لاگ‌ها.

در صورت نیاز، CA را در کلاینت‌های دیگر (Kibana, Beats, Logstash) trust کن.

---


