# Hướng dẫn Deploy Website lên AWS Lightsail + Cloudflare

> **Mục đích:** Tài liệu nội bộ cho team dev — hướng dẫn deploy một dự án website (static hoặc app) lên AWS Lightsail, gắn domain trên Cloudflare, và cấu hình HTTPS chuẩn production.
>
> **Đối tượng:** Dev có kiến thức cơ bản về Linux/SSH/DNS.
>
> **Phiên bản đầu tiên:** 2026-05-08 — viết sau khi deploy `compassscribe.com` (project `twiilio-verify`).

---

## Mục lục

1. [Tổng quan kiến trúc](#1-tổng-quan-kiến-trúc)
2. [Tài khoản và thông tin cần có](#2-tài-khoản-và-thông-tin-cần-có)
3. [Thiết lập server lần đầu (one-time)](#3-thiết-lập-server-lần-đầu-one-time)
4. [Quy trình deploy dự án mới (chuẩn)](#4-quy-trình-deploy-dự-án-mới-chuẩn)
5. [Cấu hình Cloudflare](#5-cấu-hình-cloudflare)
6. [Verify và Troubleshoot](#6-verify-và-troubleshoot)
7. [Bảo trì (Maintenance)](#7-bảo-trì-maintenance)
8. [Phụ lục: Bảo mật và Best practices](#8-phụ-lục-bảo-mật-và-best-practices)
9. [Tham chiếu nhanh (Cheat sheet)](#9-tham-chiếu-nhanh-cheat-sheet)

---

## 1. Tổng quan kiến trúc

### 1.1 Sơ đồ

```
       [Người dùng truy cập compassscribe.com]
                         │
                         ▼
        ┌─────────────────────────────────┐
        │  CLOUDFLARE (lớp ngoài cùng)    │
        │  • DNS: dịch domain → IP        │
        │  • Proxy (CDN): cache + che IP  │
        │  • WAF: chặn tấn công           │
        │  • SSL edge: HTTPS với user     │
        └─────────────────────────────────┘
                         │ (HTTPS đến origin)
                         ▼
        ┌─────────────────────────────────┐
        │  AWS LIGHTSAIL (origin server)  │
        │  • Static IP                    │
        │  • Firewall: chỉ mở 80/443/22   │
        │  • Apache: nhận request HTTP(S) │
        │  • Let's Encrypt: cert HTTPS    │
        │  • Virtual Hosts: nhiều domain  │
        │    cùng 1 server                │
        └─────────────────────────────────┘
                         │
                         ▼
        ┌─────────────────────────────────┐
        │  /var/www/                      │
        │  ├── compassscribe/  ← project1 │
        │  ├── project2/                  │
        │  └── ...                        │
        └─────────────────────────────────┘
```

### 1.2 Thành phần

#### Cloudflare (lớp DNS + CDN)

Có 2 vai trò khác nhau:

- **DNS only (cloud xám):** chỉ trả về IP khi browser hỏi. Traffic đi thẳng tới origin.
- **Proxied (cloud cam):** traffic đi qua Cloudflare trước khi tới origin. Có CDN cache, WAF, ẩn IP origin.

**SSL/TLS modes — phải hiểu rõ:**

| Mode | User ↔ CF | CF ↔ Origin | Khi nào dùng |
|------|-----------|-------------|--------------|
| Off | HTTP | HTTP | Không bao giờ |
| Flexible | HTTPS | **HTTP** | Không an toàn — gây redirect loop nếu origin force HTTPS |
| Full | HTTPS | HTTPS (cert nào cũng được) | OK cho dev |
| **Full (strict)** | HTTPS | HTTPS (cert phải valid) | **Chuẩn production** |

> **Quy ước team:** mọi domain production đặt SSL/TLS mode = **Full (strict)**.

#### AWS Lightsail (Origin server)

- **Static IP:** IP cố định, miễn phí khi attach instance đang chạy. **Bắt buộc** vì IP động sẽ đổi khi reboot.
- **Firewall (Networking tab):** chỉ mở port 22 (SSH), 80 (HTTP), 443 (HTTPS).
- **Apache Virtual Hosts:** một server chạy được nhiều domain — Apache đọc header `Host:` để biết phục vụ site nào.

#### SSL Certificate

3 lựa chọn:

| Loại | Đặc điểm | Khuyến nghị |
|------|----------|-------------|
| **Let's Encrypt** | Miễn phí, auto-renew 90 ngày, valid mọi browser, hoạt động độc lập với CF | ✅ **Dùng** |
| Cloudflare Origin Cert | Miễn phí, valid 15 năm, chỉ hoạt động khi qua CF proxy | Dùng khi không muốn quản lý renewal |
| Self-signed | Miễn phí, browser cảnh báo "không an toàn" | ❌ Không production |

> **Quy ước team:** dùng **Let's Encrypt** qua certbot.

#### Cấu trúc thư mục

```
/var/www/<project-name>/        ← document root
├── index.html (hoặc public/)
├── privacy/index.html
└── ...
```

---

## 2. Tài khoản và thông tin cần có

Trước khi bắt đầu, dev cần access vào:

| Hạng mục | Ghi chú |
|----------|---------|
| AWS Lightsail Console | https://lightsail.aws.amazon.com |
| Cloudflare Dashboard | https://dash.cloudflare.com |
| SSH key của Lightsail instance | File `.pem` từ Lightsail console hoặc setup ssh-key qua `~/.ssh/authorized_keys` |
| GitHub access | Để clone repo source code |
| Email admin (cho Let's Encrypt) | VD: `admin@compass247.vn` |

---

## 3. Thiết lập server lần đầu (one-time)

> Phần này chỉ làm **một lần khi tạo instance Lightsail mới**. Nếu instance đã setup sẵn (như instance hiện tại), bỏ qua phần này.

### 3.1 Tạo instance Lightsail

1. Lightsail Console → **Create instance**
2. **Region:** chọn gần khách hàng nhất (vd. `Singapore` cho khách Việt Nam, `Oregon` cho khách Mỹ)
3. **Platform:** Linux/Unix
4. **Blueprint:** OS Only → Ubuntu 22.04 LTS (hoặc mới hơn)
5. **Plan:** chọn theo nhu cầu (tối thiểu $5/tháng — 1GB RAM cho static site)
6. **Identifier:** đặt tên rõ ràng, vd. `web-prod-01`

### 3.2 Gắn Static IP

1. Lightsail Console → tab **Networking** (thanh menu trên cùng)
2. **Create static IP**
3. **Region:** trùng region instance
4. **Attach to instance:** chọn instance vừa tạo
5. **Identify:** đặt tên, vd. `web-prod-01-ip`
6. **Create**
7. **Lưu lại IP này** — dùng cho mọi DNS record sau này

⚠️ Sau khi attach Static IP, IP cũ của instance sẽ bị thay. Nếu đang SSH bằng IP cũ thì phải SSH lại bằng IP mới.

### 3.3 Cấu hình firewall Lightsail

Vào instance → tab **Networking** → mục **IPv4 Firewall**:

| Application | Protocol | Port | Source |
|-------------|----------|------|--------|
| SSH | TCP | 22 | Restrict (chỉ IP văn phòng/VPN) |
| HTTP | TCP | 80 | Any IPv4 |
| HTTPS | TCP | 443 | Any IPv4 |

> **An toàn:** giới hạn SSH chỉ cho IP cố định nếu có thể. Tuyệt đối không mở port khác trừ khi thật cần.

### 3.4 Cài đặt phần mềm cơ bản trên server

SSH vào server rồi chạy:

```bash
# Update OS
sudo apt update && sudo apt upgrade -y

# Cài Apache
sudo apt install -y apache2

# Enable các module Apache cần thiết
sudo a2enmod ssl rewrite headers

# Cài certbot (Let's Encrypt) — bản snap mới nhất
sudo snap install --classic certbot
sudo ln -sf /snap/bin/certbot /usr/bin/certbot

# Cài Git (để clone source)
sudo apt install -y git

# Cài Fail2ban (chống brute-force SSH)
sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban

# Reload Apache để apply modules
sudo systemctl reload apache2
sudo systemctl enable apache2
```

### 3.5 Cấu hình Apache mặc định (force HTTPS toàn server)

File `/etc/apache2/sites-available/000-default.conf` nên có:

```apache
<VirtualHost *:80>
    RewriteEngine On
    RewriteCond %{HTTPS} off
    RewriteRule ^(.*)$ https://%{HTTP_HOST}$1 [R=301,L]

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

→ Mọi request HTTP đến server (không phân biệt domain) sẽ tự redirect sang HTTPS.

### 3.6 Cấu hình Cloudflare API token (tùy chọn — cho DNS-01 challenge)

Trong tài liệu này dùng HTTP-01 challenge (đơn giản), nên bước này có thể bỏ qua. Cần khi:

- Wildcard cert (vd. `*.example.com`) — bắt buộc DNS-01
- Domain không thể truy cập từ Internet (private network)

---

## 4. Quy trình deploy dự án mới (chuẩn)

> Đây là quy trình re-use cho **mỗi domain mới**. Giả sử server đã setup xong (mục 3).
>
> Ví dụ trong tài liệu này: deploy `newdomain.com` cho project `my-project`.

### 4.1 Checklist trước khi deploy

- [ ] Đã có repo source code trên GitHub
- [ ] Đã mua domain (đặt trên Cloudflare hoặc transfer về Cloudflare)
- [ ] Có quyền SSH vào server
- [ ] Có access Cloudflare dashboard

### 4.2 Bước 1 — Cloudflare: thêm DNS records (tắt proxy)

Vào https://dash.cloudflare.com → chọn domain → **DNS** → **Records** → **Add record**:

| Type | Name | Content (IPv4) | Proxy status | TTL |
|------|------|----------------|--------------|-----|
| A | `@` (apex) | `<Static IP>` | **DNS only (cloud xám)** | Auto |
| A | `www` | `<Static IP>` | **DNS only (cloud xám)** | Auto |

⚠️ **Bắt buộc tắt proxy lúc này** — vì certbot HTTP-01 challenge cần verify domain bằng cách truy cập trực tiếp origin. Nếu bật proxy, request sẽ bị CF chặn → certbot fail.

Đợi 1-5 phút cho DNS lan truyền, verify:

```bash
dig newdomain.com +short        # phải ra Static IP
dig newdomain.com @1.1.1.1 +short
```

### 4.3 Bước 2 — Server: clone code và setup document root

SSH vào server:

```bash
# Tạo thư mục project (quy ước: /var/www/<project-name>/)
sudo mkdir -p /var/www/my-project
sudo chown -R $USER:$USER /var/www/my-project

# Clone source
cd /var/www
sudo rm -rf my-project   # nếu cần clean
sudo git clone https://github.com/compass247/my-project.git
# Hoặc nếu private repo:
# sudo git clone git@github.com:compass247/my-project.git

# Đổi quyền cho Apache đọc được
sudo chown -R www-data:www-data /var/www/my-project
sudo chmod -R 755 /var/www/my-project
```

### 4.4 Bước 3 — Server: tạo Apache virtual host

Tạo file `/etc/apache2/sites-available/newdomain.com.conf`:

```apache
<VirtualHost *:80>
    ServerName newdomain.com
    ServerAlias www.newdomain.com

    DocumentRoot /var/www/my-project

    <Directory /var/www/my-project>
        Options -Indexes +FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/newdomain_error.log
    CustomLog ${APACHE_LOG_DIR}/newdomain_access.log combined
</VirtualHost>
```

> **Lưu ý:** chỉ tạo block `*:80` — certbot sẽ tự động tạo block `*:443` (file `<domain>-le-ssl.conf`).

Test config và enable:

```bash
sudo apache2ctl configtest                          # phải ra "Syntax OK"
sudo a2ensite newdomain.com.conf
sudo systemctl reload apache2
```

Test local (chưa qua DNS):

```bash
curl -H "Host: newdomain.com" http://localhost/ -I  # phải ra 200 hoặc 301
```

### 4.5 Bước 4 — Server: lấy SSL cert với Let's Encrypt

```bash
sudo certbot --apache \
    -d newdomain.com -d www.newdomain.com \
    --non-interactive \
    --agree-tos \
    --email admin@compass247.vn \
    --redirect
```

**Giải thích flag:**

- `--apache`: dùng plugin Apache, tự động sửa vhost
- `-d ...`: liệt kê domain cần cover (cả apex và www)
- `--non-interactive`: chạy không hỏi
- `--agree-tos`: đồng ý điều khoản LE
- `--email`: dùng để nhận thông báo expire
- `--redirect`: tự thêm rule HTTP → HTTPS vào vhost

**Kết quả mong đợi:**

```
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/newdomain.com/fullchain.pem
Successfully deployed certificate for newdomain.com
```

Certbot sẽ:

- Tạo file mới: `/etc/apache2/sites-available/newdomain.com-le-ssl.conf` (vhost HTTPS)
- Sửa file `newdomain.com.conf` (vhost HTTP) thêm redirect → HTTPS
- Reload Apache tự động
- Đăng ký auto-renew qua systemd timer

### 4.6 Bước 5 — Verify từ Internet (chưa qua CF proxy)

```bash
# Từ máy local hoặc bất kỳ máy nào ngoài server:
curl -I http://newdomain.com         # phải ra 301 → https://...
curl -I https://newdomain.com        # phải ra 200
curl -sI https://newdomain.com | grep -i 'server'   # ra "Server: Apache/..."
```

Mở browser truy cập `https://newdomain.com` — phải hiện đúng trang, có khóa xanh (HTTPS valid).

### 4.7 Bước 6 — Cloudflare: bật proxy + cấu hình SSL

Sau khi xác nhận site live qua HTTPS, vào Cloudflare dashboard:

#### 6.1 Bật proxy (cloud cam)

DNS tab → click cloud xám của 2 record (`@` và `www`) → đổi sang **cam (Proxied)**.

#### 6.2 SSL/TLS mode

**SSL/TLS** → **Overview** → chọn **Full (strict)**.

⚠️ **Tuyệt đối không chọn Flexible** — sẽ gây **redirect loop**:
- User → CF: HTTPS
- CF → Origin: HTTP
- Origin Apache thấy HTTP → 301 về HTTPS
- CF nhận 301 → tiếp tục gửi HTTP → loop

#### 6.3 Always Use HTTPS

**SSL/TLS** → **Edge Certificates** → bật **Always Use HTTPS**.

Tác dụng: CF redirect HTTP → HTTPS ngay tại edge, nhanh hơn redirect ở origin.

#### 6.4 Min TLS Version

**SSL/TLS** → **Edge Certificates** → **Minimum TLS Version** → chọn **1.2**.

Lý do: chặn các client cũ dùng TLS 1.0/1.1 (đã bị deprecated, không an toàn).

#### 6.5 (Tùy chọn) HSTS

**SSL/TLS** → **Edge Certificates** → **HTTP Strict Transport Security (HSTS)** → Enable.

Settings khuyến nghị (chỉ bật khi chắc chắn site stable, vì HSTS một khi user đã cache thì rất khó rút lại):

- Max Age: `6 months` (tăng dần lên 1 year sau khi ổn định)
- Include subdomains: tùy nhu cầu
- Preload: chỉ bật khi muốn submit lên HSTS preload list

#### 6.6 (Khuyến nghị) Cache HTML ở edge

**Caching** → **Cache Rules** → **Create rule**:

- Rule name: `Cache static HTML`
- When: Hostname equals `newdomain.com`
- Then: 
    - Cache Eligibility: **Eligible for cache**
    - Edge TTL: **2 hours** (hoặc dài hơn nếu nội dung ít đổi)
    - Browser TTL: **1 hour**

### 4.8 Bước 7 — Verify cuối cùng (qua CF proxy)

```bash
# DNS giờ phải ra IP của Cloudflare (104.x hoặc 172.x), không phải Static IP
dig newdomain.com @1.1.1.1 +short

# Header phải có cf-ray và server: cloudflare
curl -sI https://newdomain.com | grep -iE 'server|cf-ray'

# Test redirect không bị loop
curl -sIL http://newdomain.com --max-redirs 5 -o /dev/null -w "%{http_code} | redirects=%{num_redirects}\n"
```

✅ Pass khi: DNS → CF IPs, header có `cf-ray`, không bị loop, browser hiện khóa xanh.

---

## 5. Cấu hình Cloudflare

### 5.1 Setting áp dụng cho mọi domain (set 1 lần ở account level hoặc mỗi domain)

| Setting | Vị trí | Giá trị | Lý do |
|---------|--------|---------|-------|
| SSL/TLS mode | SSL/TLS → Overview | **Full (strict)** | Bảo mật end-to-end |
| Always Use HTTPS | SSL/TLS → Edge Certificates | ON | Force HTTPS ở edge |
| Min TLS Version | SSL/TLS → Edge Certificates | **1.2** | Block TLS cũ |
| Auto Minify | Speed → Optimization | (Cloudflare đã deprecate, bỏ qua) | — |
| Brotli | Speed → Optimization | ON | Nén tốt hơn gzip |
| Email Address Obfuscation | Scrape Shield | ON (mặc định) | Chống bot scrape email |
| Hotlink Protection | Scrape Shield | OFF (trừ khi cần) | Tránh chặn nhầm CDN/embed |

### 5.2 Page Rules / Cache Rules đặc biệt

- Static HTML: cache edge 2h-1day, tùy domain
- Files có hash trong tên (vd. `app.[hash].js`): cache 1 year (immutable)
- API endpoints: bypass cache

---

## 6. Verify và Troubleshoot

### 6.1 Lệnh verify chuẩn

```bash
# DNS
dig <domain> +short
dig <domain> @1.1.1.1 +short

# HTTP/HTTPS
curl -I http://<domain>
curl -I https://<domain>

# Headers + cert
curl -sI https://<domain>
echo | openssl s_client -servername <domain> -connect <domain>:443 2>/dev/null \
  | openssl x509 -noout -issuer -subject -dates

# Redirect chain
curl -sIL http://<domain> --max-redirs 5 -o /dev/null \
  -w "Final: %{http_code} | Redirects: %{num_redirects} | URL: %{url_effective}\n"

# Test trực tiếp origin (bypass CF) — qua Host header
curl -sI -H "Host: <domain>" --resolve <domain>:443:<STATIC_IP> https://<domain>
```

### 6.2 Lỗi thường gặp

#### "Too many redirects" (ERR_TOO_MANY_REDIRECTS)

**Nguyên nhân:** SSL/TLS mode = Flexible nhưng origin force HTTPS.
**Fix:** Cloudflare → SSL/TLS → Overview → đổi sang **Full (strict)**.

#### Certbot "Failed authorization" (HTTP-01 challenge fail)

**Nguyên nhân thường gặp:**
1. DNS chưa trỏ đúng → `dig <domain> +short` để check
2. Cloudflare proxy đang BẬT (cloud cam) → tắt thành cloud xám trước
3. Firewall Lightsail chưa mở port 80
4. Apache không listen port 80 → `sudo ss -tlnp | grep :80`

#### "Site can't be reached" sau khi bật proxy CF

**Nguyên nhân:** SSL/TLS mode = Off, hoặc origin cert không valid.
**Fix:** đảm bảo có Let's Encrypt cert valid trên origin + SSL mode = Full (strict).

#### Cert origin expired

**Nguyên nhân:** auto-renew lỗi (vd. domain bỏ proxy CF không đúng cách rồi quên bật lại, hoặc port 80 bị block).
**Fix:**
```bash
sudo certbot renew --dry-run    # test renewal
sudo certbot renew              # renew thật nếu dry-run pass
sudo systemctl status snap.certbot.renew.timer   # check timer
```

#### Apache không reload sau khi sửa vhost

```bash
sudo apache2ctl configtest      # check syntax error trước
sudo systemctl reload apache2
sudo journalctl -u apache2 -n 50  # xem log nếu fail
```

### 6.3 Log files cần biết

```
/var/log/apache2/<domain>_access.log    # request log của vhost cụ thể
/var/log/apache2/<domain>_error.log     # error log của vhost
/var/log/apache2/access.log             # default access (catch-all)
/var/log/apache2/error.log              # default error
/var/log/letsencrypt/letsencrypt.log    # log của certbot
/var/log/auth.log                       # SSH/sudo log
/var/log/fail2ban.log                   # fail2ban activity
```

Xem realtime:
```bash
sudo tail -f /var/log/apache2/<domain>_access.log
```

---

## 7. Bảo trì (Maintenance)

### 7.1 Auto-renew SSL

Certbot tự đăng ký systemd timer khi cài. Verify:

```bash
sudo systemctl list-timers | grep certbot
sudo certbot renew --dry-run
```

Cert renew tự động khi còn ≤30 ngày. Không cần làm gì thêm.

### 7.2 Update OS định kỳ

```bash
# Hàng tuần / hàng tháng
sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y

# Reboot nếu kernel update
sudo reboot
```

> **Cảnh báo:** trước khi reboot, đảm bảo Static IP đã attach (không bị mất IP sau reboot).

### 7.3 Backup

**Tối thiểu cần backup:**

| Item | Vị trí | Cách backup |
|------|--------|-------------|
| Source code | `/var/www/*` | Đã có trên GitHub (đủ) |
| Apache config | `/etc/apache2/sites-available/*.conf` | Lưu vào repo riêng `infra/` hoặc snapshot Lightsail |
| SSL certs | `/etc/letsencrypt/` | Có thể re-issue, nhưng nên backup để tránh rate limit của LE |
| App data (database, uploads) | tùy app | Cron job dump → S3 |

**Snapshot Lightsail:**

- Lightsail Console → instance → tab **Snapshots** → **Create snapshot**
- Khuyến nghị: tạo trước mọi thay đổi lớn (upgrade OS, đổi config quan trọng)
- Lightsail có Auto-snapshot daily — bật trong tab **Snapshots** → **Auto snapshots**

### 7.4 Monitor

**Tối thiểu cần:**

- Lightsail Metrics (CPU, RAM, network) — tab **Metrics** trong instance
- Cloudflare Analytics — Dashboard → Analytics
- Uptime monitor: UptimeRobot (free) hoặc Better Uptime, ping mỗi 5 phút

### 7.5 Cập nhật code (deploy lại)

Nếu code đã trên GitHub:

```bash
cd /var/www/my-project
sudo git pull origin main
sudo chown -R www-data:www-data .
# Static site: xong. App cần build/restart thì làm tiếp.
```

> **Nâng cao:** setup CI/CD (GitHub Actions → SSH deploy) để tự động pull khi push lên `main`. Xem mục 8.4.

---

## 8. Phụ lục: Bảo mật và Best practices

### 8.1 SSH hardening

Edit `/etc/ssh/sshd_config`:

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

Restart: `sudo systemctl restart ssh`

### 8.2 Ẩn version Apache

Edit `/etc/apache2/conf-enabled/security.conf`:

```
ServerTokens Prod
ServerSignature Off
```

Reload Apache.

### 8.3 Headers bảo mật

Thêm vào vhost (hoặc file riêng `/etc/apache2/conf-enabled/security-headers.conf`):

```apache
<IfModule mod_headers.c>
    Header always set X-Content-Type-Options "nosniff"
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set Referrer-Policy "strict-origin-when-cross-origin"
    Header always set Permissions-Policy "geolocation=(), camera=(), microphone=()"
</IfModule>
```

### 8.4 Auto-deploy (GitHub Actions → SSH)

Tạo `.github/workflows/deploy.yml` trong repo:

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            cd /var/www/my-project
            git pull origin main
            sudo chown -R www-data:www-data .
            # Nếu có build step:
            # npm ci && npm run build
            # sudo systemctl reload apache2
```

GitHub repo → Settings → Secrets:
- `SSH_HOST`: Static IP
- `SSH_USER`: `ubuntu` (hoặc user khác)
- `SSH_KEY`: nội dung private key (file `.pem` từ Lightsail)

### 8.5 Rate limiting với Cloudflare

Free plan có 1 rate limit rule miễn phí. Setup cho endpoint nhạy cảm (login, API):

CF Dashboard → Security → WAF → Rate limiting rules → Create.

---

## 9. Tham chiếu nhanh (Cheat sheet)

### 9.1 Deploy domain mới — checklist 1 phút

```
□ DNS: thêm A record (cloud xám), trỏ Static IP
□ Server: clone code → /var/www/<project>/
□ Server: tạo /etc/apache2/sites-available/<domain>.conf
□ Server: a2ensite + reload apache2
□ Server: certbot --apache -d <domain> -d www.<domain> --redirect
□ Verify HTTPS từ ngoài
□ CF: bật proxy (cloud cam)
□ CF: SSL/TLS = Full (strict)
□ CF: Always Use HTTPS = ON
□ CF: Min TLS = 1.2
□ Verify lại lần cuối qua proxy
```

### 9.2 Lệnh hay dùng

```bash
# Apache
sudo apache2ctl -S                              # list vhosts
sudo apache2ctl configtest                      # test syntax
sudo systemctl reload apache2                   # apply config
sudo a2ensite <domain>.conf                     # enable site
sudo a2dissite <domain>.conf                    # disable site

# Certbot
sudo certbot certificates                       # list certs
sudo certbot renew --dry-run                    # test renewal
sudo certbot delete --cert-name <domain>        # xóa cert

# Logs
sudo tail -f /var/log/apache2/<domain>_access.log
sudo journalctl -u apache2 -f

# DNS / network
dig <domain> +short
dig <domain> @1.1.1.1 +short
curl -sI https://<domain>
ss -tlnp                                        # ports đang listen
```

### 9.3 Đường dẫn quan trọng

```
/var/www/<project>/                                  # source code
/etc/apache2/sites-available/<domain>.conf           # vhost HTTP
/etc/apache2/sites-available/<domain>-le-ssl.conf    # vhost HTTPS (do certbot tạo)
/etc/apache2/sites-enabled/                          # symlinks tới sites-available
/etc/letsencrypt/live/<domain>/                      # SSL certs
/var/log/apache2/                                    # logs
```

### 9.4 Khi cần thêm domain vào instance hiện tại

```bash
# 1. Cloudflare: thêm A record (cloud xám trước)

# 2. SSH vào server
sudo mkdir -p /var/www/new-project
cd /var/www && sudo git clone <repo-url> new-project
sudo chown -R www-data:www-data /var/www/new-project

# 3. Copy template vhost từ domain cũ
sudo cp /etc/apache2/sites-available/compassscribe.com.conf \
        /etc/apache2/sites-available/newdomain.com.conf

# 4. Sửa file mới: đổi ServerName, ServerAlias, DocumentRoot, log filename
sudo nano /etc/apache2/sites-available/newdomain.com.conf

# 5. Enable + test + reload
sudo apache2ctl configtest && \
    sudo a2ensite newdomain.com.conf && \
    sudo systemctl reload apache2

# 6. Lấy SSL
sudo certbot --apache -d newdomain.com -d www.newdomain.com \
    --non-interactive --agree-tos --email admin@compass247.vn --redirect

# 7. Cloudflare: bật proxy (cloud cam) + verify
```

---

## Liên hệ và lịch sử cập nhật

| Ngày | Nội dung | Người |
|------|----------|-------|
| 2026-05-08 | Phiên bản đầu tiên — sau khi deploy `compassscribe.com` | Claude + admin@compass247.vn |

> Khi update tài liệu, ghi vào bảng trên để team theo dõi.
