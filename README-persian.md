[فارسی](README-persian.md) | [English](README.md)

# راه‌اندازی Telegraf و InfluxDB

ابزار **Telegraf** یک ایجنت (Agent) قدرتمند و مبتنی بر پلاگین است که برای جمع‌آوری و ارسال متریک‌ها استفاده میشه. این ابزار به صورت پیش‌فرض متریک‌های دقیق سیستم رو از ماشین‌های شما جمع‌آوری می‌کنه (مثل میزان مصرف CPU، رم، دیسک، شبکه، پراسس‌ها و... - دقیقاً مشابه کاری که Node Exporter انجام میده). البته نقطه قوت تلگراف اینه که صدها پلاگین داره و می‌تونه دیتا رو از دیتابیس‌ها، صف‌های پیام و APIهای خارجی هم بخونه.

از طرف دیگه، **InfluxDB** یک دیتابیس سری‌زمانی (Time-Series Database یا TSDB) بسیار محبوب هست که اختصاصاً برای ذخیره متریک‌هایی که تلگراف جمع‌آوری می‌کنه طراحی شده.

> ### نکته
> **اصلاً چرا داریم اینو یاد می‌گیریم؟**  
> ما توی ریپازیتوری‌های قبلی با **Grafana Alloy** به عنوان یک جمع‌آوری‌کننده تله‌متری مدرن آشنا شدیم و می‌تونیم ازش استفاده کنیم. با این حال، ترکیب Telegraf + InfluxDB (که بهش TICK stack هم میگن) در زیرساخت‌های قدیمی‌تر و شرکت‌های مختلف به‌شدت رایج و پراستفاده‌ست. به عنوان یک مهندس، مهمه که وقتی توی یک سیستم واقعی با این استک مواجه میشید، اونو بشناسید و بتونید باهاش کار کنید.

> ### نکته (آینده)
> در معماری‌های مدرن مانیتورینگ، InfluxDB رفته رفته داره جاشو به ابزارهایی مثل **OpenTelemetry** و دیتابیس‌های مقیاس‌پذیری مثل Prometheus/Mimir میده. ما در آینده حتماً در مورد OpenTelemetry توی یک ریپازیتوری جداگانه صحبت خواهیم کرد.

---

## نصب Telegraf به‌صورت سرویس systemd (باینری)

> ### نکته (مانیتورینگ چندین سرور)
> اگه شما چندین ماشین یا سرور مختلف دارید، بهترین کار اینه که روی **تک‌تک اون‌ها** ایجنت تلگراف رو نصب کنید (برای سادگی و درگیر نشدن با داکر، نصب باینری روی ماشین‌ها پیشنهاد میشه) و تنظیمشون کنید که همگی دیتای خودشون رو به یک سرور مرکزی که روش InfluxDB نصبه بفرستن!

اگر فقط می‌خواید ایجنت تلگراف رو روی یک سرور لینوکسی نصب کنید تا منابعش رو مانیتور کنه، می‌تونید اون رو به صورت مستقل (سرویس systemd) راه‌اندازی کنید.

### ۱. دانلود و نصب فایل باینری

برای دانلود فایل باینری، دستورات زیر را اجرا کنید:

```bash
# نکته‌ مهم
# اگر معماری سیستمی که Telegraf قرار است روی آن نصب شود amd64 نیست دستورات زیر به درستی برای شما کار نخواهند کرد.
# برای مثال اگر معماری شما arm64 است باید تمامی  `amd64` ها را با `arm64` در کامند‌های زیر جایگزین کنید:

VERSION=$(curl -s https://api.github.com/repos/influxdata/telegraf/releases/latest | grep '"tag_name"' | cut -d'"' -f4 | sed 's/v//')
wget -O telegraf.tar.gz https://dl.influxdata.com/telegraf/releases/telegraf-${VERSION}_linux_amd64.tar.gz
tar xvfz telegraf.tar.gz
```

فایل استخراج شده را به مسیر اجرایی سیستم منتقل کنید:
```bash
sudo mv telegraf-${VERSION}/usr/bin/telegraf /usr/local/bin/telegraf
rm -rf telegraf-${VERSION} telegraf.tar.gz
```

برای امنیت بیشتر، یک یوزر سیستمی اختصاصی و دایرکتوری‌های لازم رو بسازید:
```bash
sudo useradd --no-create-home --shell /usr/sbin/nologin telegraf
sudo mkdir -p /etc/telegraf
```

### ۲. ساخت فایل کانفیگ

فایل `/etc/telegraf/telegraf.conf` را با کانفیگ زیر ایجاد کنید:
```toml
[agent]
  interval = "10s"
  round_interval = true
  metric_batch_size = 1000
  metric_buffer_limit = 10000
  collection_jitter = "0s"
  flush_interval = "10s"
  flush_jitter = "0s"
  precision = "0s"

# بخش OUTPUT: دیتاها کجا ارسال بشن؟ (به InfluxDB نسخه 2)
[[outputs.influxdb_v2]]
  urls = ["http://localhost:8086"]
  token = "${INFLUX_TOKEN}"
  organization = "my-org"
  bucket = "my-bucket"

# بخش INPUTS: چه دیتاهایی از سرور جمع‌آوری بشن؟
[[inputs.cpu]]
  percpu = true
  totalcpu = true
  collect_cpu_time = false
  report_active = false
[[inputs.disk]]
  ignore_fs = ["tmpfs", "devtmpfs", "devfs", "iso9660", "overlay", "aufs", "squashfs"]
[[inputs.mem]]
[[inputs.net]]
[[inputs.system]]
```

دسترسی‌ها را تنظیم کنید:
```bash
sudo chown -R telegraf:telegraf /etc/telegraf
```

### ۳. ساخت فایل سرویس systemd

ابتدا فایل `/etc/telegraf/telegraf.env` را برای قرار دادن توکن امنیتی بسازید:
```env
INFLUX_TOKEN=your-super-secret-token
```

فایل `/etc/systemd/system/telegraf.service` را با محتوای زیر بسازید:
```ini
[Unit]
Description=Telegraf
Wants=network-online.target
After=network-online.target

[Service]
User=telegraf
Group=telegraf
EnvironmentFile=/etc/telegraf/telegraf.env
ExecStart=/usr/local/bin/telegraf --config /etc/telegraf/telegraf.conf

Restart=always

[Install]
WantedBy=multi-user.target
```

در نهایت سرویس را فعال و اجرا کنید:
```bash
sudo systemctl daemon-reload
sudo systemctl enable telegraf
sudo systemctl start telegraf
```

---

## راه‌اندازی InfluxDB و Telegraf با Docker Compose

معمولاً دیتابیس (InfluxDB) از طریق داکر راه‌اندازی میشه. در اینجا یک فایل `docker-compose.yml` داریم که هم دیتابیس InfluxDB (نسخه 2) و هم ایجنت تلگراف رو به صورت همزمان بالا میاره.

```yaml
services:
  influxdb:
    image: influxdb:2.7
    container_name: influxdb
    restart: unless-stopped
    ports:
      - "8086:8086"
    volumes:
      - influxdb-data:/var/lib/influxdb2
      - influxdb-config:/etc/influxdb2
    environment:
      # این متغیرها برای ستاپ اولیه دیتابیس استفاده میشن
      - DOCKER_INFLUXDB_INIT_MODE=setup
      - DOCKER_INFLUXDB_INIT_USERNAME=admin
      - DOCKER_INFLUXDB_INIT_PASSWORD=secure_password
      - DOCKER_INFLUXDB_INIT_ORG=my-org
      - DOCKER_INFLUXDB_INIT_BUCKET=my-bucket
      - DOCKER_INFLUXDB_INIT_ADMIN_TOKEN=my-super-secret-auth-token

  telegraf:
    image: telegraf:latest
    container_name: telegraf
    restart: unless-stopped
    volumes:
      - ./telegraf.conf:/etc/telegraf/telegraf.conf:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro # برای خوندن متریک‌های داکر
    environment:
      - INFLUX_TOKEN=my-super-secret-auth-token

volumes:
  influxdb-data:
  influxdb-config:
```

کانتینرها را اجرا کنید:
```bash
docker compose up -d
```

---

## توی خود InfluxDB چه کارهایی می‌تونیم بکنیم؟

دیتابیس InfluxDB فقط یک محل ذخیره اطلاعات نیست! اگر توی مرورگرتون آدرس `http://{IP_ADDRESS}:8086` رو باز کنید و با یوزر پسوردی که دادید لاگین کنید، به **رابط کاربری وب (Data Explorer)** دسترسی پیدا می‌کنید.

داخل این پنل می‌تونید:
۱. **دیتا رو اکسپلور کنید (Explore Data)**: به صورت بصری روی باکت‌ها بگردید و متریک‌ها رو مشاهده کنید.
۲. **دشبورد بسازید**: خود InfluxDB سیستم دشبوردسازی داخلی داره! (هرچند گرافانا همیشه انتخاب بهتریه).
۳. **زبان Flux**: با استفاده از زبان قدرتمند `Flux` یا `InfluxQL` روی دیتاها کوئری‌های پیچیده بزنید.
۴. **ایجاد تسک (Tasks)**: می‌تونید کارهای پس‌زمینه (Background Jobs) تعریف کنید تا مثلاً دیتای قدیمی رو فشرده (Downsample) کنن که حجم دیسک پر نشه، و یا حتی تنظیم کنید که بهتون آلرت بده!

---

## اتصال InfluxDB به Grafana (از طریق رابط کاربری)

برای اینکه متریک‌های جمع‌آوری شده توسط تلگراف رو توی گرافانا ببینید، باید InfluxDB رو به عنوان Data Source گرافانا معرفی کنید. این کار به صورت گرافیکی توی پنل گرافانا به این شکل انجام میشه:

۱. گرافانا رو توی مرورگر باز کنید (`http://{IP_ADDRESS}:3000`).
۲. از منوی سمت چپ برید روی تب **Connections** و **Data Sources** رو انتخاب کنید.
۳. روی **Add data source** کلیک کنید و **InfluxDB** رو جستجو کنید.
۴. توی تنظیمات:
   - در بخش **Query Language**، حتماً گزینه **Flux** رو انتخاب کنید.
   - در بخش **URL**، آدرس `http://influxdb:8086` رو وارد کنید (یا IP سرور رو بنویسید).
۵. بیاید پایین به بخش **InfluxDB Details** و موارد زیر رو پر کنید:
   - **Organization**: `my-org`
   - **Token**: همون توکنی که تو داکر کمپوز مشخص کردید (`my-super-secret-auth-token`)
   - **Default Bucket**: `my-bucket`
۶. در نهایت روی **Save & Test** کلیک کنید. 

تموم شد! حالا گرافانا به InfluxDB وصله. می‌تونید دشبوردهای آماده (مثل [Telegraf System Dashboard](https://grafana.com/grafana/dashboards/928-telegraf-system-dashboard/)) رو ایمپورت کنید و از مانیتورینگ سیستم‌تون لذت ببرید!
