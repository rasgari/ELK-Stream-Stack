# solution ssl elastic

زمانی که سرور در حال اجراست این دستور برای راه اندازی ssl elastic انجام میدیم:

```
docker exec -it elastic1 elasticsearch-setup-passwords interactive
```

بعد از وارد کردن کامند مربوطه سوالات زیر پرسیده می شود:
```
enter password for elastic
reenter password for elastic
enter password for apm system
reenter password for apm system
enter password for kibana_system
reenter password for kibana_system
enter password for logstash_system
reenter password for logstash_system
enter password for beats_system
reenter password for beats_system
enter password for remot_monitoring_user
reenter password for remot_monitoring_user
```
