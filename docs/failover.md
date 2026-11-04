# راهنمای Failover — scheduler و WireGuard

<div dir="rtl">

وقتی ارتباط WireGuard با شرکت **قطع** شود، دو worker پایتون (`main_post.py` و `main_story.py`) به‌صورت خودکار اجرا می‌شوند.  
وقتی تونل **وصل** شود، workerها متوقف و پورت‌های VPN برای `api` و `session-manager` فعال می‌شوند.

</div>

---

## معماری

<div dir="rtl">

| حالت | تونل WG | scheduler | docker |
|------|---------|-----------|--------|
| عادی (شرکت وصل) | UP | `main_post` / `main_story` **متوقف** | `api` + `session-manager` روی IP وایرگارد (`172.20.1.2`) |
| failover (شرکت قطع) | DOWN | **در حال اجرا** | `api` روی `127.0.0.1:8000`، Kafka روی `127.0.0.1:9093` |

کانتینرهای `zookeeper`، `kafka`، `api`، `session-manager` همیشه بالا می‌مانند تا scheduler به Mongo، Kafka و API دسترسی داشته باشد.

</div>

---

## مسیرها

<div dir="ltr">

| مسیر | توضیح |
|------|--------|
| `/srv/docker-compose/fg-ins-crw/scheduler/` | کدهای `main_post.py`، `main_story.py`، `requirements.txt` |
| `/srv/docker-compose/fg-ins-crw/scheduler/scheduler.env` | متغیرهای محیطی (Ansible) |
| `/srv/docker-compose/fg-ins-crw/scheduler/.venv/` | virtualenv پایتون |
| `/usr/local/bin/wg-failover-controller` | اسکریپت مانیتور تونل |
| `/etc/systemd/system/wg-failover.service` | سرویس مانیتور |
| `/etc/systemd/system/failover-scheduler@.service` | واحد workerها |

</div>

---

## متغیرهای محیطی scheduler

<div dir="ltr">

```env
CONFIG_SCHEDULER_MONGODB_HOST=127.0.0.1
CONFIG_SCHEDULER_MONGODB_PORT=27017
CONFIG_SCHEDULER_MONGODB_TABLE_NAME=instagram
CONFIG_SCHEDULER_MONGODB_COLLECTION_NAME=source
CONFIG_SCHEDULER_FETCHER_API_URL=http://127.0.0.1:8000/api/v1
CONFIG_SCHEDULER_BOOTSTRAP_SERVERS=127.0.0.1:9093
```

</div>

<div dir="rtl">

نام متغیرها باید با `scheduler/config/config.py` یکی باشد (`CONFIG_SCHEDULER_*`).  
مقادیر پیش‌فرض در `roles/failover/defaults/main.yml` است؛ در `inventory/host_vars/fg-crw-01.yml` قابل override هستند.

**نکته:** `MongoClient(f"{HOST}:{PORT}")` است — پس `HOST` باید فقط IP/hostname باشد (مثلاً `127.0.0.1`)، نه `mongodb://...`.

</div>

---

## deploy با Ansible

<div dir="ltr">

```bash
# فقط failover
ansible-playbook fg-crw.yml --limit fg-crw-01 --tags failover

# app + failover
ansible-playbook fg-crw.yml --limit fg-crw-01 --tags "app,failover"
```

</div>

---

## بررسی روی سرور

<div dir="ltr">

```bash
# وضعیت مانیتور
systemctl status wg-failover.service
journalctl -u wg-failover.service -f

# وضعیت workerها
systemctl status failover-scheduler@main_post.service
systemctl status failover-scheduler@main_story.service

# وضعیت تونل
wg show wg1
ping -c 2 172.20.1.1

# وضعیت کانتینرها
cd /srv/docker-compose/fg-ins-crw
docker compose ps
```

</div>

---

## تشخیص قطع تونل

<div dir="rtl">

مانیتور هر ۱۵ ثانیه بررسی می‌کند:

1. interface `wg1` در حالت UP باشد
2. آخرین WireGuard handshake کمتر از ۱۲۰ ثانیه پیش باشد
3. ping به `172.20.1.1` (سرور شرکت) موفق باشد

برای جلوگیری از نوسان (flapping)، ۳ بار متوالی fail → حالت DOWN، ۲ بار success → حالت UP.

`PostUp` / `PreDown` در WireGuard برای واکنش سریع‌تر به قطع/وصل فیزیکی interface استفاده می‌شود.

</div>

---

## عیب‌یابی

<div dir="rtl">

| مشکل | بررسی |
|------|--------|
| worker اجرا نمی‌شود | `journalctl -u wg-failover` — آیا حالت `down` ثبت شده؟ |
| خطای pip / venv | وجود `scheduler/requirements.txt` روی سرور |
| Kafka وصل نمی‌شود | `127.0.0.1:9093` — listener مخصوص host |
| API در failover | `curl http://127.0.0.1:8000` |
| Mongo | `mongosh mongodb://127.0.0.1:27017` روی هاست |
| `wg-quick` با `PostUp`/`Configuration parsing error` | هوک‌ها باید زیر `[Interface]` باشند نه بعد از `[Peer]` — دوباره `--tags failover` بزنید |
| `wg-quick@wg1` start fail با هوک‌های درست | `PostUp` نباید `systemctl` همزمان بزند (deadlock). اسکریپت notify باید background + `exit 0` باشد. بازیابی دستی: `wg-quick down wg1; ip link del wg1; wg-quick up wg1` |
| `ModuleNotFoundError` در worker | پکیج در `scheduler/requirements.txt` نیست — اضافه کنید و `pip install -r requirements.txt` داخل `.venv` |
| `wg-failover` با status=129 | سیگنال HUP موقع `sleep` با `set -e` — کنترلر باید `sleep \|\| true` داشته باشد |
| تونل UP ولی state=`down` | چک interface نباید فقط `state UP` باشد؛ WG معمولاً `state UNKNOWN` می‌دهد |

</div>
