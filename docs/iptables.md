# راهنمای iptables — پروژه fg-crw

این سند توضیح می‌دهد فایروال سرور چگونه کار می‌کند، چرا هنگام اجرای Ansible ممکن است SSH قطع شود، و در صورت قفل شدن چطور دسترسی را برگردانید.

---

## مشکل قبلی: چرا SSH قطع می‌شد؟

در نسخه قبلی `roles/os-config/tasks/iptables.yml` قوانین **یکی‌یکی** و در **taskهای جدا** اعمال می‌شدند:

```text
1. iptables -F              ← همه قوانین پاک می‌شود
2. iptables -P INPUT DROP  ← ورودی پیش‌فرض = DROP (همه بسته‌ها رد)
3. ... بعداً قانون SSH اضافه می‌شد
```

بین مرحله ۲ و مرحله «Allow SSH» یک پنجره زمانی وجود داشت که:

- policy روی `DROP` بود
- هنوز قانون `ACCEPT` برای پورت SSH (`5566`) وجود نداشت
- session فعلی Ansible/SSH ممکن بود قطع شود

اگر Ansible وسط کار fail شود یا timeout بخورد، سرور با policy `DROP` و بدون قانون SSH باقی می‌ماند و از راه SSH دیگر وصل نمی‌شوید.

### راه‌حل اعمال‌شده

الان همه قوانین در **یک فایل** (`/etc/iptables/rules.v4`) جمع می‌شوند و با **یک دستور** اعمال می‌شوند:

```bash
iptables-restore /etc/iptables/rules.v4
```

این کار **اتمیک** است: یا همه قوانین با هم اعمال می‌شوند، یا هیچ‌کدام. قانون SSH همیشه **قبل از** policy نهایی `DROP` در فایل قرار دارد.

---

## قوانین فعلی سرور fg-crw-01

| ترافیک | پروتکل | پورت | منبع | توضیح |
|--------|--------|------|------|-------|
| SSH | TCP | `5566` | همه | دسترسی مدیریت |
| WireGuard | UDP | `23693` | همه | تونل VPN داخلی |
| Zabbix Agent | TCP | `10050` | `172.20.1.1` | مانیتورینگ |
| Loopback | — | — | `lo` | ترافیک داخلی سیستم |
| Established | — | — | — | اتصال‌های باز (SSH فعال) |

**پیش‌فرض INPUT:** `DROP` — یعنی هر چیز دیگری رد می‌شود.

**مهم:** فقط chain `INPUT` (ورودی به خود سرور) محدود می‌شود. chainهای `FORWARD` و `OUTPUT` و قوانین Docker **دست‌نخورده** می‌مانند تا:
- سرور به اینترنت دسترسی داشته باشد (`OUTPUT ACCEPT`)
- کانتینرها بتوانند `pip install` و DNS انجام دهند (`FORWARD` توسط Docker مدیریت می‌شود)

---

## چرا iptables اینترنت/Docker را قطع می‌کرد؟

نسخه قبلی از `iptables-restore` با `FORWARD DROP` استفاده می‌کرد. این دو مشکل ایجاد می‌کرد:

1. **`iptables-restore` کل جدول filter را پاک می‌کند** — شامل chainهای `DOCKER`, `DOCKER-USER`, `DOCKER-FORWARD` که Docker خودش می‌سازد.
2. **`FORWARD DROP`** — ترافیک خروجی کانتینرها (build، pip، DNS به `pypi.org`) از chain FORWARD رد می‌شود و drop می‌خورد.

علامت در Docker build:
```text
Temporary failure in name resolution
Failed to establish a new connection ... /simple/python-dotenv/
```

### راه‌حل فعلی

Ansible **فقط chain INPUT** را rebuild می‌کند و FORWARD/OUTPUT را دست نمی‌زند:

```bash
iptables -F INPUT          # فقط INPUT
# ... قوانین ACCEPT ...
iptables -P INPUT DROP     # فقط policy ورودی
iptables-save > /etc/iptables/rules.v4   # ذخیره کل وضعیت (شامل قوانین Docker)
```

**هرگز** این کار را نکنید (Docker و اینترنت کانتینر را می‌شکند):
```bash
iptables-restore /etc/iptables/rules.v4   # ❌ اگر فایل قوانین Docker نداشته باشد
```

### بازیابی سریع اگر Docker build / pip الان fail می‌شود

روی سرور (SSH یا Console):

```bash
# 1. FORWARD را باز کنید (اگر DROP شده)
sudo iptables -P FORWARD ACCEPT

# 2. Docker قوانین iptables خودش را دوباره بسازد
sudo systemctl restart docker

# 3. تست DNS از داخل یک کانتینر
sudo docker run --rm alpine ping -c 2 pypi.org

# 4. دوباره build
cd /srv/docker-compose/fg-ins-crw   # یا app_deploy_dir شما
sudo docker compose build --no-cache session-manager
```

سپس Ansible را با نسخه جدید اجرا کنید:
```bash
ansible-playbook fg-crw.yml -K --tags firewall --limit fg-crw-01
ansible-playbook fg-crw.yml -K --tags app --limit fg-crw-01
```

---

## فایل‌های مرتبط در پروژه

| فایل | نقش |
|------|-----|
| `roles/os-config/tasks/iptables.yml` | taskهای Ansible |
| `roles/os-config/templates/iptables/rules.v4.j2` | قالب قوانین |
| `/etc/iptables/rules.v4` (روی سرور) | قوانین ذخیره‌شده |
| `inventory/host_vars/fg-crw-01.yml` | `ssh_port`, `wg_interface_listen_port`, `zabbix_server_ip` |

---

## اجرای Ansible

```bash
# فقط فایروال
ansible-playbook fg-crw.yml --tags firewall --limit fg-crw-01

# همه تنظیمات به‌جز فایروال
ansible-playbook fg-crw.yml --skip-tags firewall --limit fg-crw-01
```

**توصیه:** اول بار firewall را جداگانه تست کنید و یک session SSH دوم باز نگه دارید.

---

## دستورات مفید روی سرور

### مشاهده قوانین

```bash
# خلاصه
sudo iptables -L -n -v

# با شماره خط (برای حذف)
sudo iptables -L INPUT -n --line-numbers

# ذخیره فعلی
sudo iptables-save

# مقایسه با فایل persistent
sudo cat /etc/iptables/rules.v4
```

### اعمال دستی قوانین از فایل

```bash
# ❌ خطرناک — ممکن است قوانین Docker را پاک کند
# sudo iptables-restore /etc/iptables/rules.v4

# ✅ فقط INPUT را rebuild کنید (مثل Ansible)
sudo iptables -F INPUT
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 5566 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 23693 -j ACCEPT
sudo iptables -A INPUT -p tcp -s 172.20.1.1/32 --dport 10050 -j ACCEPT
sudo iptables -P INPUT DROP
sudo iptables-save | sudo tee /etc/iptables/rules.v4
```

### تست موقت با rollback (پیشنهاد قبل از تغییر دستی)

```bash
# 120 ثانیه فرصت دارید؛ اگر تأیید نکنید، قوانین قبلی برمی‌گردد
sudo iptables-apply --timeout 120 /etc/iptables/rules.v4
# سپس در همان session:
sudo iptables-apply --confirm
```

### ذخیره قوانین فعلی

```bash
sudo iptables-save | sudo tee /etc/iptables/rules.v4
sudo systemctl restart netfilter-persistent
```

---

## 🚨 بازیابی اضطراری — SSH قطع شد

### روش ۱: Console سرور (بهترین راه)

از پنل هاست (Hetzner، OVH، Proxmox، …) وارد **Console / VNC / Rescue** شوید و این دستورات را بزنید:

```bash
# باز کردن کامل فایروال (موقت)
sudo iptables -F
sudo iptables -X
sudo iptables -P INPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo iptables -P OUTPUT ACCEPT
```

بعد SSH را تست کنید:

```bash
ssh -p 5566 ansible@185.250.249.216
```

سپس Ansible را دوباره اجرا کنید تا قوانین درست deploy شوند:

```bash
ansible-playbook fg-crw.yml --tags firewall --limit fg-crw-01
```

### روش ۲: اصلاح فایل persistent از Console

اگر بعد از reboot هم SSH نمی‌آید، فایل ذخیره‌شده خراب است:

```bash
sudo nano /etc/iptables/rules.v4
```

محتوای موقت امن (همه چیز باز):

```
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
COMMIT
```

```bash
sudo iptables-restore /etc/iptables/rules.v4
```

### روش ۳: Rescue Mode

اگر Console در دسترس نیست:

1. سرور را در **Rescue Mode** بوت کنید
2. دیسک اصلی mount کنید
3. فایل `/etc/iptables/rules.v4` را خالی یا ACCEPT کنید
4. reboot به حالت عادی

### روش ۴: WireGuard (اگر تونل بالا باشد)

اگر WireGuard از قبل وصل است و سرویس‌ها روی `172.20.1.2` هستند:

```bash
ssh -p 5566 ansible@172.20.1.2
```

توجه: iptables فعلی SSH را روی **همه interfaceها** باز می‌کند، نه فقط IP عمومی؛ پس WireGuard معمولاً کمک می‌کند مگر اینکه session از همان IP عمومی قطع شده باشد.

---

## اضافه کردن قانون جدید

1. فایل `roles/os-config/templates/iptables/rules.v4.j2` را ویرایش کنید
2. قانون جدید را **بالای** `COMMIT` و **بعد از** قوانین ACCEPT موجود اضافه کنید
3. deploy کنید:

```bash
ansible-playbook fg-crw.yml --tags firewall --limit fg-crw-01
```

**مثال — اجازه HTTP فقط از IP مشخص:**

```
-A INPUT -p tcp -s 203.0.113.10/32 --dport 80 -j ACCEPT
```

**هرگز** policy `DROP` را قبل از قوانین ACCEPT اعمال نکنید.

---

## خطاهای رایج

| علامت | علت احتمالی | راه‌حل |
|-------|-------------|--------|
| SSH timeout | policy DROP بدون قانون SSH | Console → flush + ACCEPT |
| Ansible قطع وسط firewall | taskهای جدا (نسخه قدیم) | نسخه جدید — فقط INPUT rebuild |
| Docker build / pip / DNS fail | `FORWARD DROP` یا `iptables-restore` | فقط INPUT rebuild؛ از `iptables-restore` دستی پرهیز کنید |
| بعد از reboot SSH نمی‌آید | `/etc/iptables/rules.v4` اشتباه | Console → اصلاح فایل |
| پورت اشتباه | `ssh_port` در host_vars ≠ پورت واقعی sshd | `sshd_config` و `host_vars` را هماهنگ کنید |
| nftables فعال | تداخل با iptables | `sudo nft list ruleset` — فقط یکی را استفاده کنید |

---

## چک‌لیست قبل از deploy فایروال

- [ ] پورت SSH در `host_vars` با `sshd_config` یکی است (`5566`)
- [ ] یک session SSH backup باز است
- [ ] دسترسی Console پنل هاست را دارید
- [ ] WireGuard پورت UDP در فایروال باز است
- [ ] تست: `ansible-playbook fg-crw.yml --tags firewall --limit fg-crw-01`

---

## مرجع سریع

```bash
# وضعیت
sudo iptables -L -n -v
sudo iptables-save

# اعمال از فایل پروژه (روی سرور بعد از deploy)
sudo iptables-restore /etc/iptables/rules.v4

# اضطراری — باز کردن همه
sudo iptables -F && sudo iptables -P INPUT ACCEPT

# تست امن
sudo iptables-apply --timeout 120 /etc/iptables/rules.v4
```
