
بعد از اجرا :
```
kubectl apply -f elasticsearch-statefulset.yaml
```

بررسی  وضعیت کلاستر
```
kubectl get pods -l app=elasticsearch
```

باید این‌ها رو ببینی:
```
elasticsearch-0   Running
elasticsearch-1   Running
elasticsearch-2   Running
```
تست کلاستر:
```
kubectl port-forward elasticsearch-0 9200:9200
```
```
curl http://localhost:9200/_cluster/health?pretty
```

خروجی سالم:
```
"number_of_nodes" : 3,
"status" : "green"
```
4️⃣ چرا StatefulSet نه Deployment؟ (نکته مصاحبه‌ای 🔥)
مورد	دلیل
نام ثابت Pod	elasticsearch-0/1/2
Persistent Volume	دیتا از بین نمی‌ره
Discovery	نودها همدیگه رو پیدا می‌کنن
ترتیب بالا آمدن	کنترل‌شده
5️⃣ منابع مورد نیاز پیشنهادی (حداقلی)
Resource	مقدار
CPU	1 core
RAM	2GB
Disk	10GB × 3

⚠️ اگر نود کم‌رم باشه، Elasticsearch بالا نمیاد
