# دستورات Ansible — پروژه fg-crw

راهنمای اجرای بخش‌های مختلف playbook با `--tags` و `--skip-tags`.

**سرور هدف:** `fg-crw-01`  
**Playbook:** `fg-crw.yml`  
**Inventory:** `inventory/hosts`

---

## دستور پایه

```bash
cd /data/Project/fg-crw

# اجرای همه بخش‌ها
ansible-playbook fg-crw.yml --limit fg-crw-01

# فقط یک بخش (با tag)
ansible-playbook fg-crw.yml --limit fg-crw-01 --tags <TAG>

# همه به‌جز یک بخش
ansible-playbook fg-crw.yml --limit fg-crw-01 --skip-tags <TAG>

# چند بخش با هم
ansible-playbook fg-crw.yml --limit fg-crw-01 --tags "docker,app-config"
```

---

## جدول Tagها و دستورات

| بخش | Tag | دستور |
|-----|-----|-------|
| **همه** | — | `ansible-playbook fg-crw.yml --limit fg-crw-01` |
| **نصب پایه (همه installation)** | `installation` | `ansible-playbook fg-crw.yml --limit fg-crw-01 --tags installation` |
| نصب Zabbix repo | `zabbix` | `ansible-playbook fg-crw.yml --limit fg-crw-01 --tags zabbix` |
| نصب پکیج‌ها | `packages` | `ansible-playbook fg-crw.yml --limit fg-crw-01 --tags packages` |
| آپگرید سیستم | `upgrade` | `ansible-playbook fg-crw.yml --limit fg-crw-01 --tags upgrade` |
| ریبوت در صورت نیاز | `reboot` | `ansible-playbook fg-crw.yml --limit fg-crw-01 --tags reboot` |
| **تنظیمات OS (همه os-config)** | `os-config` | `ansible-playbook fg-crw.yml --limit fg-crw-01 --tags os-config` |
| WireGuard | `wireguard` | `ansible-playbook fg-crw.yml --limit fg-crw-01 --tags wireguard` |
| Zabbix agent | `zabbix` | `ansible-playbook fg-crw.yml --limit fg-crw-01 --tags zabbix` |
| کاربران | `users` | `ansible-playbook fg-crw.yml --limit fg-crw-01 --tags users` |
| PAM / faillock | `pam` | `ansible-playbook fg-crw.yml --limit fg-crw-01 --tags pam` |
| Hostname | `hostname` | `ansible-playbook fg-crw.yml --limit fg-crw-01 --tags hostname` |
| SSH | `ssh` | `ansible-playbook fg-crw.yml --limit fg-crw-01 --tags ssh` |
| Cron | `cron` | `ansible-playbook fg-crw.yml --limit fg-crw-01 --tags cron` |
| امنیت (unattended-upgrades) | `security` | `ansible-playbook fg-crw.yml --limit fg-crw-01 --tags security` |
| **فایروال iptables** | `firewall` | `ansible-playbook fg-crw.yml --limit fg-crw-01 --tags firewall` |
| Backup تنظیمات | `backup` | `ansible-playbook fg-crw.yml --limit fg-crw-01 --tags backup` |
| **نصب Docker** | `docker` | `ansible-playbook fg-crw.yml --limit fg-crw-01 --tags docker` |
| **اپلیکیشن (همه app)** | `app` | `ansible-playbook fg-crw.yml --limit fg-crw-01 --tags app` |
| فقط deploy فایل‌های compose | `app-config` | `ansible-playbook fg-crw.yml --limit fg-crw-01 --tags app-config` |
| build و start کانتینرها | `app-build` | `ansible-playbook fg-crw.yml --limit fg-crw-01 --tags app-build` |

---

## سناریوهای رایج

### اولین deploy کامل
```bash
ansible-playbook fg-crw.yml --limit fg-crw-01
```

### فقط به‌روزرسانی docker-compose (بدون rebuild)
```bash
ansible-playbook fg-crw.yml --limit fg-crw-01 --tags app-config
```

### فقط build و اجرای کانتینرها
```bash
ansible-playbook fg-crw.yml --limit fg-crw-01 --tags app-build
```

### deploy اپلیکیشن (config + build)
```bash
ansible-playbook fg-crw.yml --limit fg-crw-01 --tags "app-config,app-build"
# یا کوتاه‌تر:
ansible-playbook fg-crw.yml --limit fg-crw-01 --tags app
```

### فقط فایروال
```bash
ansible-playbook fg-crw.yml --limit fg-crw-01 --tags firewall
```

### همه به‌جز فایروال و reboot
```bash
ansible-playbook fg-crw.yml --limit fg-crw-01 --skip-tags "firewall,reboot"
```

### فقط زیرساخت OS (بدون Docker و App)
```bash
ansible-playbook fg-crw.yml --limit fg-crw-01 --tags "installation,os-config"
```

### Docker + App (بدون لمس OS)
```bash
ansible-playbook fg-crw.yml --limit fg-crw-01 --tags "docker,app"
```

### Dry-run (بدون اعمال تغییر)
```bash
ansible-playbook fg-crw.yml --limit fg-crw-01 --tags app --check --diff
```

### مشاهده لیست taskها بدون اجرا
```bash
ansible-playbook fg-crw.yml --limit fg-crw-01 --tags app --list-tasks
```

---

## بررسی health کانتینرها (بعد از deploy)

```bash
# روی سرور
cd /srv/docker-compose/fg-ins-crw   # یا app_deploy_dir شما
docker compose ps
docker inspect --format='{{.State.Health.Status}}' session-manager api kafka zookeeper
```

| سرویس | healthcheck |
|--------|-------------|
| zookeeper | `ruok` → `imok` روی پورت 2181 |
| kafka | `kafka-broker-api-versions` روی localhost:9092 |
| session-manager | HTTP `GET /health` روی پورت 8001 |
| api | HTTP `GET /health` روی پورت 8000 |

اگر endpoint سلامت متفاوت است، در `inventory/host_vars/fg-crw-01.yml` تنظیم کنید:

```yaml
app_api_health_path: /health
app_session_manager_health_path: /health
```

---

## نکات

- tag `base` روی roleهای `installation` و `os-config` هم هست (alias برای اجرای گروهی).
- `zabbix` هم در installation (repo) و هم در os-config (agent config) استفاده می‌شود — با `--tags zabbix` هر دو اجرا می‌شوند.
- قبل از `firewall` یک session SSH backup باز نگه دارید ([راهنمای iptables](iptables.md)).
- برای deploy فقط compose بدون build: `--tags app-config`
