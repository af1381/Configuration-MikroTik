# README — راه‌اندازی APT از Nexus (`packages.hesaba.co`)

این سند نحوه‌ی راه‌اندازی کلاینت‌های Ubuntu برای استفاده از Nexus به‌عنوان منبع APT را توضیح می‌دهد. سناریو هدف: **Ubuntu 22.04 (Jammy)**. اگر 24.04 (Noble) دارید، در پیوست انتهای فایل راهنما آمده است.

---

## 1) پیش‌نیازهای Nexus (سمت سرور)

1. وارد **Administration → Security → Anonymous Access** شوید و تیک **Allow anonymous users to access the server** را **بردارید**.
2. در **Security → Roles** یک Role فقط-خواندنی بسازید (مثلاً `apt-readers`) و برای هر ریپویی که می‌خواهید در دسترس باشد، دسترسی‌های زیر را بدهید:
   - `nx-repository-view-apt-apt-ubuntu-jammy-browse`
   - `nx-repository-view-apt-apt-ubuntu-jammy-read`
   - `nx-repository-view-apt-apt-ubuntu-jammy-security-browse`
   - `nx-repository-view-apt-apt-ubuntu-jammy-security-read`
   > الگو: `nx-repository-view-<format>-<repository>-browse|read`
3. در **Security → Users** یک کاربر مخصوص کلاینت‌ها بسازید (مثلاً: `aptclient`) و Role بالا را به او بدهید.  
   **توصیه**: اگر امکانش هست از **User Token** استفاده کنید (Security → User Tokens) تا به‌جای پسورد واقعی، توکن بدهید.

---

## 2) پیکربندی کلاینت Ubuntu 22.04 (Jammy)

### 2.1) احراز هویت APT (ایمن و تمیز)
فایل اختصاصی احراز هویت را بسازید تا نام کاربری/رمز در URL دیده نشود.
```bash
sudo install -m 0700 -d /etc/apt/auth.conf.d
sudo tee /etc/apt/auth.conf.d/hesaba.conf >/dev/null <<'EOF'
machine packages.hesaba.co
login aptclient
password "YOUR_TOKEN_OR_PASSWORD"
EOF
sudo chown root:root /etc/apt/auth.conf.d/hesaba.conf
sudo chmod 600 /etc/apt/auth.conf.d/hesaba.conf
```

> اگر پورت اختصاصی دارید، هاست را به‌صورت `packages.hesaba.co:PORT` بنویسید.

### 2.2) تعریف منابع APT (Jammy)
**توجه:** ریپوی `apt-ubuntu-jammy` همه‌ی «کامپوننت‌های توزیع» `jammy`, `jammy-updates`, `jammy-backports` را پوشش می‌دهد.  
`apt-ubuntu-jammy-security` مخصوص `jammy-security` است.

```bash
# بکاپ از سورس‌های فعلی (اختیاری)
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
# اختیاری: برای تست، سورس‌های پیش‌فرض Ubuntu را موقتاً comment کن
sudo sed -i 's/^\s*deb /# &/g' /etc/apt/sources.list

sudo tee /etc/apt/sources.list.d/hesaba-jammy.list >/dev/null <<'EOF'
deb [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] https://packages.hesaba.co/repository/apt-ubuntu-jammy jammy main restricted universe multiverse
deb [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] https://packages.hesaba.co/repository/apt-ubuntu-jammy jammy-updates main restricted universe multiverse
deb [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] https://packages.hesaba.co/repository/apt-ubuntu-jammy jammy-backports main restricted universe multiverse
deb [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] https://packages.hesaba.co/repository/apt-ubuntu-jammy-security jammy-security main restricted universe multiverse
EOF
```

> چرا `signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg`؟ چون Nexus این مخازن را **پروکسی** می‌کند و بسته‌ها با کلید رسمی Ubuntu امضا شده‌اند؛ نیازی به کلید جدید نیست.

### 2.3) به‌روزرسانی و تست
```bash
sudo apt clean
sudo apt update

# نصب یک پکیج کوچک برای تست
sudo apt install -y jq

# بررسی اینکه منبع واقعاً از Nexus است
apt-cache policy jq | sed -n '1,15p'
```

### 2.4) اگر HTTPS با گواهی داخلی/خودامضا دارید
CA داخلی را به سیستم اضافه کنید تا خطای TLS نگیرید:
```bash
sudo cp YOUR_CA_CERT.crt /usr/local/share/ca-certificates/hesaba-ca.crt
sudo update-ca-certificates
```

---

## 3) عیب‌یابی سریع

- **401 Unauthorized**  
  - فایل `/etc/apt/auth.conf.d/hesaba.conf` وجود ندارد یا سطح دسترسی آن درست نیست (`chmod 600`).  
  - نام هاست در `machine` دقیقاً باید `packages.hesaba.co` (یا با پورت) باشد.
- **403 Forbidden**  
  - کاربر دارید ولی روی ریپوهای مورد نظر `browse/read` ندارید.
- **404 Not Found**  
  - مسیر/توزیع اشتباه: باید `.../repository/apt-ubuntu-jammy/dists/jammy/...` باشد.
- **TLS/Certificate**  
  - CA داخلی را نصب کنید (بخش 2.4).
- **NO_PUBKEY / امضای GPG**  
  - از `signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg` استفاده کرده‌اید؟ بسته‌ها با کلید رسمی Ubuntu امضا شده‌اند.

**تست مستقیم با curl:**
```bash
curl -I -u aptclient:YOUR_TOKEN_OR_PASSWORD \
  https://packages.hesaba.co/repository/apt-ubuntu-jammy/dists/jammy/InRelease
```

---

## 4) (اختیاری) Docker APT برای Jammy
اگر ریپوی `apt-docker-jammy` را هم در Nexus دارید و می‌خواهید از آن استفاده کنید (Docker CE):
```bash
# کلید رسمی Docker (برای امضای بسته‌های download.docker.com که توسط Nexus پروکسی می‌شود)
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

sudo tee /etc/apt/sources.list.d/hesaba-docker-jammy.list >/dev/null <<'EOF'
deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://packages.hesaba.co/repository/apt-docker-jammy jammy stable
EOF

sudo apt update
# مثال نصب
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

---

## پیوست A) Ubuntu 24.04 (Noble Numbat)
اگر میزبان شما 24.04 است، از ریپوهای `apt-ubuntu-noble` و `apt-ubuntu-noble-security` استفاده کنید و `jammy` را با `noble` جایگزین کنید:

```bash
sudo tee /etc/apt/sources.list.d/hesaba-noble.list >/dev/null <<'EOF'
deb [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] https://packages.hesaba.co/repository/apt-ubuntu-noble noble main restricted universe multiverse
deb [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] https://packages.hesaba.co/repository/apt-ubuntu-noble noble-updates main restricted universe multiverse
deb [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] https://packages.hesaba.co/repository/apt-ubuntu-noble noble-backports main restricted universe multiverse
deb [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] https://packages.hesaba.co/repository/apt-ubuntu-noble-security noble-security main restricted universe multiverse
EOF
sudo apt update
```

> **هشدار:** مخازن `noble` را با Ubuntu 22.04 استفاده نکنید (ناسازگاری وابستگی‌ها).

---

## پیوست B) روش جایگزین (کم‌امن‌تر) با قرار دادن یوزر/پس در URL
استفاده از این روش توصیه نمی‌شود (ممکن است در لاگ‌ها یا ابزارها آشکار شود)، ولی در صورت نیاز:
```bash
deb [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] \
https://USERNAME:PASSWORD@packages.hesaba.co/repository/apt-ubuntu-jammy jammy main
```

---

## خلاصهٔ سریع
1) Anonymous را در Nexus ببندید؛ نقش فقط-خواندنی بسازید؛ کاربر/توکن بسازید.  
2) در کلاینت: `auth.conf.d/hesaba.conf` را با `machine packages.hesaba.co` تنظیم کنید.  
3) سورس‌های `jammy` و `jammy-security` را به Nexus اشاره دهید.  
4) `apt update` و نصب تست.  اگر خطا داشتید، بخش عیب‌یابی را ببینید.
