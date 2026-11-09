# MongoDB — اتصال از کانتینرها

<div dir="rtl">

Mongo روی **هاست** اجرا می‌شود (`127.0.0.1:27017` برای ابزارهای محلی).  
`session-manager` داخل Docker است و به `127.0.0.1` داخل کانتینر وصل نمی‌شود؛ باید به **IP بریج Docker روی هاست** وصل شود:

</div>

<div dir="ltr">

```text
mongodb://172.17.0.1:27017
```

</div>

<div dir="rtl">

## پیش‌نیازها

1. **Mongo روی IP بریج هم listen کند** (نه فقط `127.0.0.1`)
2. **فایروال** ترافیک `docker0` / `br+` به پورت `27017` را اجازه دهد (در role فایروال اضافه شده)

---

## تنظیم bindIp مونگو

فایل معمولاً:

</div>

<div dir="ltr">

```text
/etc/mongod.conf
```

</div>

<div dir="rtl">

بخش شبکه:

</div>

<div dir="ltr">

```yaml
net:
  port: 27017
  bindIp: 127.0.0.1,172.17.0.1
```

</div>

<div dir="rtl">

سپس:

</div>

<div dir="ltr">

```bash
sudo systemctl restart mongod
ss -lntp | grep 27017
# باید 127.0.0.1:27017 و 172.17.0.1:27017 را ببینید
```

</div>

<div dir="rtl">

**امنیت:** پورت `27017` را روی IP عمومی (`0.0.0.0` / eth0) باز نکنید. فقط localhost + docker bridge کافی است. فایروال هم از اینترنت عمومی به `27017` ACCEPT ندارد.

---

## تست از داخل کانتینر

</div>

<div dir="ltr">

```bash
# از هاست
docker exec -it session-manager sh -c 'python -c "import socket; s=socket.create_connection((\"172.17.0.1\",27017),5); print(\"ok\"); s.close()"'

# لاگ session-manager
docker logs session-manager --tail 100
```

</div>

<div dir="rtl">

اگر هنوز fail است:

</div>

<div dir="ltr">

```bash
# فایروال: آیا قانون docker0 هست؟
sudo iptables -L INPUT -n -v | grep 27017

# gateway واقعی Docker
ip -4 addr show docker0
```

</div>

<div dir="rtl">

اگر IP `docker0` غیر از `172.17.0.1` بود، همان را در `app_mongo_uri` و `bindIp` بگذارید.

---

## import داده (روی هاست)

</div>

<div dir="ltr">

```bash
mongoimport --uri "mongodb://127.0.0.1:27017" --db instagram --collection account --file /data/account/instagram.account11.json --jsonArray

mongoimport --uri "mongodb://127.0.0.1:27017" --db instagram --collection local_source --file /data/account/instagram.local_source_11.json --jsonArray

# Create Collection anonymous (MobileProxy)
mongoimport --uri "mongodb://127.0.0.1:27017" --db instagram --collection anonymous --file /data/account/instagram.anonymous_11.json --jsonArray
```

</div>

<div dir="rtl">

## Ansible

بعد از اصلاح `mongod.conf`:

</div>

<div dir="ltr">

```bash
ansible-playbook fg-crw.yml --limit fg-crw-01 --tags firewall
ansible-playbook fg-crw.yml --limit fg-crw-01 --tags "app-config,app-build"
```

</div>
