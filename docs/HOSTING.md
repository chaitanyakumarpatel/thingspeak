# ThingSpeak Hosting Instructions

Detailed, step-by-step instructions for hosting this ThingSpeak application
(Rails 4.0.5 / Ruby 2.1, MySQL, Redis + Resque) on either:

- **Option A – a cloud VPS** (recommended for anything you want to keep online), or
- **Option B – a home machine** (fine for tinkering, sensor projects on your LAN).

Everything below assumes a Debian/Ubuntu base (Ubuntu 14.04/16.04/18.04 behave
closest to what this app was written for). Notes for other distros are inline.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Prerequisites & Sizing](#2-prerequisites--sizing)
3. [Option A: Deploying on a VPS](#3-option-a-deploying-on-a-vps)
4. [Option B: Deploying on a Home Machine](#4-option-b-deploying-on-a-home-machine)
5. [Install System Dependencies](#5-install-system-dependencies)
6. [Install Ruby 2.1 with RVM](#6-install-ruby-21-with-rvm)
7. [Install & Configure MySQL](#7-install--configure-mysql)
8. [Install & Configure Redis](#8-install--configure-redis)
9. [Get the Application Code](#9-get-the-application-code)
10. [Configure the Application](#10-configure-the-application)
11. [Database Setup](#11-database-setup)
12. [Precompile Assets](#12-precompile-assets)
13. [Run the App: Puma + Resque Workers + Clockwork](#13-run-the-app-puma--resque-workers--clockwork)
14. [Reverse Proxy with Nginx](#14-reverse-proxy-with-nginx)
15. [HTTPS with Let's Encrypt (VPS)](#15-https-with-lets-encrypt-vps)
16. [Process Management with systemd](#16-process-management-with-systemd)
17. [Firewall & Security Hardening](#17-firewall--security-hardening)
18. [Backups & Restore](#18-backups--restore)
19. [Upgrades & Rolling Updates](#19-upgrades--rolling-updates)
20. [Monitoring & Logs](#20-monitoring--logs)
21. [Troubleshooting](#21-troubleshooting)
22. [Appendix: Quick Reference Cheatsheet](#22-appendix-quick-reference-cheatsheet)

---

## 1. Architecture Overview

ThingSpeak is not a single process. A production deployment runs four things:

```
                    ┌──────────────────────────────────────────────┐
 Internet / LAN ──▶ │ nginx (port 80/443)  – reverse proxy, static │
                    └───────────────┬──────────────────────────────┘
                                    │ proxy_pass http://127.0.0.1:3000
                    ┌───────────────▼──────────────┐
                    │ puma (Rails app server)      │  ← serves web UI + API
                    └───────┬──────────┬───────────┘
                            │          │
              ┌─────────────▼──┐   ┌───▼──────────────────┐
              │ MySQL          │   │ Redis                │
              │ (channels,     │   │ (Resque job queue,   │
              │  data fields)  │   │  caching)            │
              └────────────────┘   └───▲──────────────────┘
                                       │
                    ┌──────────────────┴───────────────┐
                    │ resque workers  +  clockwork     │
                    │ (background jobs, cleanup,       │
                    │  channel expiry, tweets, emails) │
                    └──────────────────────────────────┘
```

| Component | Purpose | Required? |
|---|---|---|
| **Puma** (or any Rack server) | Serves the Rails app / HTTP API | Yes |
| **MySQL** | Primary datastore (`config/database.yml`) | Yes |
| **Redis** | Backs Resque background jobs | Yes |
| **Resque workers** | Process queued jobs (feeds, cleanup, notifications) | Yes for full function |
| **Clockwork** | Scheduled/recurring tasks (see `config/application.rb`, which drops `app/admin` paths when `$clockwork == true`) | Recommended |
| **Nginx** | Reverse proxy, static assets, TLS termination | Strongly recommended |

---

## 2. Prerequisites & Sizing

### Minimum hardware

| Deployment | CPU | RAM | Disk | Notes |
|---|---|---|---|---|
| Home / lab | 1 core (Pi 3+/x86) | 1 GB | 8 GB | Run with 1 worker, swap enabled |
| Small VPS | 1–2 vCPU | 2 GB | 20 GB SSD | Comfortable for personal use |
| Busy public instance | 2–4 vCPU | 4 GB+ | 40 GB+ SSD | Add swap; scale Resque workers |

Ruby 2.1 + Rails 4 + MySQL + Redis + several Resque workers fits in ~1 GB RAM,
but **add at least 1–2 GB of swap** on smaller machines (see §5.4).

### What you need before starting

- A machine running **Ubuntu 14.04–18.04** (or Debian 8/9; adapt `apt` names as needed).
- Root or sudo access.
- For a VPS: a domain name pointed at the server's IP (for HTTPS). Optional for home use.
- Basic comfort with SSH and a text editor.

---

## 3. Option A: Deploying on a VPS

### 3.1 Choose a provider & create the instance

Any KVM-based VPS works (Hetzner Cloud, DigitalOcean, Vultr, Linode/Akamai, AWS Lightsail…).
Pick the smallest **2 GB RAM** plan if you can; 1 GB works with swap. Create the
droplet/instance with **Ubuntu 18.04 LTS** (or 16.04 for closest compatibility
with Ruby 2.1 system tooling).

> Why not the newest Ubuntu? This app pins Ruby 2.1 and Rails 4.0.5. Newer
> distros ship OpenSSL 3 / newer compilers that make building Ruby 2.1 painful.
> RVM will still build 2.1 on 18.04 reliably. If you must use a newer distro,
> expect to patch OpenSSL flags when compiling Ruby.

### 3.2 First login & base setup

```bash
ssh root@YOUR_SERVER_IP

# update packages
apt-get update && apt-get -y upgrade

# create a deploy user (never run the app as root)
adduser deploy
usermod -aG sudo deploy

# copy your SSH key to the new user
rsync --archive --chown=deploy:deploy ~/.ssh /home/deploy   # run from YOUR laptop
```

Disable password SSH login (edit `/etc/ssh/sshd_config`:
`PasswordAuthentication no`, then `systemctl restart sshd`).

### 3.3 DNS

At your registrar, create an **A record**:

```
thingspeak.example.com.   A   YOUR_SERVER_IP
```

Wait for propagation: `dig +short thingspeak.example.com` should return your IP.

Then continue at [§5 Install System Dependencies](#5-install-system-dependencies).

---

## 4. Option B: Deploying on a Home Machine

Good choices: an old laptop, an Intel NUC, or a Raspberry Pi 3B+/4 (ARM builds
of Ruby/native gems work but take longer to compile — budget 30–60 min for
`bundle install`).

### 4.1 OS & local network

1. Install Ubuntu Server (or Desktop + disable GUI to save RAM).
2. Set a **static LAN IP or DHCP reservation** on your router, e.g. `192.168.1.50`,
   so devices/sensors always find the same address.
3. Enable SSH so you can manage it headless:
   ```bash
   sudo apt-get install -y openssh-server
   ```

### 4.2 Decide your exposure level

| Mode | How | When |
|---|---|---|
| **LAN only** (recommended default) | Nothing else to do; app reachable at `http://192.168.1.50:3000` (or via nginx on port 80) | Sensors & dashboards inside your home |
| **Internet via tunnel** | Cloudflare Tunnel, Tailscale Funnel, ngrok, or SSH reverse tunnel to a VPS | No open ports on your router; easiest safe option |
| **Port forwarding** | Router: forward WAN 8443 → `192.168.1.50:443`; use a dynamic-DNS hostname | You control the router and accept the risk |

If you port-forward, **you must use HTTPS** (Let's Encrypt supports DNS-01
challenges for machines behind NAT — see §15) and keep the machine patched.
Never expose plain HTTP write APIs to the internet: API keys travel in URLs.

### 4.3 Power & reliability tips

- Disable suspend/hibernate: `sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target`
- Put the machine on a UPS if possible; MySQL corrupts more easily on power loss.
- Consider auto-login disabled and `unattended-upgrades` installed:
  `sudo apt-get install -y unattended-upgrades && sudo dpkg-reconfigure -plow unattended-upgrades`

Then continue at [§5](#5-install-system-dependencies).

---

## 5. Install System Dependencies

As your `deploy` (or home) user with sudo:

### 5.1 Build tools & libraries

```bash
sudo apt-get update
sudo apt-get -y install \
  build-essential git curl wget \
  mysql-server mysql-client libmysqlclient-dev \
  redis-server \
  libxml2-dev libxslt1-dev libssl-dev libreadline-dev zlib1g-dev \
  libyaml-dev libffi-dev libgdbm-dev \
  nginx \
  nodejs        # optional helper JS runtime; therubyracer covers asset compilation too
```

(Old README used `libxslt-dev`; on newer Ubuntu the package is `libxslt1-dev`.)

### 5.2 Verify services are up

```bash
sudo systemctl status mysql redis-server nginx --no-pager
redis-cli ping        # → PONG
mysql --version
```

### 5.3 Start them on boot

```bash
sudo systemctl enable mysql redis-server nginx
```

### 5.4 Add swap (machines ≤ 2 GB RAM)

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

## 6. Install Ruby 2.1 with RVM

The repo pins `ruby-2.1.0` via `.ruby-version`. Use RVM (as the upstream README does):

```bash
\curl -sSL https://get.rvm.io | bash -s stable
source ~/.rvm/scripts/rvm

rvm install 2.1
rvm use 2.1 --default
ruby -v        # ruby 2.1.x
gem -v
```

> **Common build failure:** on newer systems Ruby 2.1 fails against OpenSSL 1.1+/3.x.
> Workaround: install an older OpenSSL via RVM (`rvm pkg install openssl`) and
> `rvm install 2.1 --with-openssl-dir=$HOME/.rvm/usr`. Alternatively, many users
> successfully bump to Ruby 2.4–2.7 with this codebase by also updating the
> `nokogiri`/`mysql2`/`therubyracer` versions in the Gemfile — test thoroughly.

Install Bundler (the Gemfile.lock is old-format; Bundler 1.x is safest):

```bash
gem install bundler -v '~> 1.17'
```

---

## 7. Install & Configure MySQL

Already installed in §5. Now secure it and create the app database user.

```bash
sudo mysql_secure_installation     # set root password, remove anon users/test db
sudo mysql -u root -p
```

Inside the MySQL shell — these credentials match `config/database.yml.example`
(`username: thing`, `password: "speak"`). **Change the password for real
deployments** and mirror the change in `database.yml`:

```sql
CREATE USER 'thing'@'localhost' IDENTIFIED BY 'speak';
GRANT ALL PRIVILEGES ON thingspeak_production.* TO 'thing'@'localhost';
GRANT ALL PRIVILEGES ON thingspeak_development.* TO 'thing'@'localhost';
GRANT ALL PRIVILEGES ON thingspeak_test.* TO 'thing'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Useful tuning for small machines — append to `/etc/mysql/conf.d/thingspeak.cnf`:

```ini
[mysqld]
innodb_buffer_pool_size = 256M   # ~50% of RAM on a dedicated DB box
max_connections         = 100
bind-address            = 127.0.0.1   # never expose MySQL to the network
```

Then `sudo systemctl restart mysql`.

---

## 8. Install & Configure Redis

Installed in §5. Defaults are fine for a single-host deployment, but harden:

`/etc/redis/redis.conf`:

```conf
bind 127.0.0.1 ::1
requirepass YOUR_REDIS_PASSWORD
appendonly yes          # persist queued jobs across restarts
maxmemory 128mb
maxmemory-policy noeviction
```

```bash
sudo systemctl restart redis-server
redis-cli -a YOUR_REDIS_PASSWORD ping    # → PONG
```

If you set `requirepass`, export it for the app (used by Resque/`dalli`):

```bash
export REDIS_URL=redis://:YOUR_REDIS_PASSWORD@127.0.0.1:6379/0
```

Add that line to `~/.bashrc` **and** to the systemd units in §16.

---

## 9. Get the Application Code

```bash
sudo mkdir -p /opt/apps && sudo chown $USER /opt/apps
cd /opt/apps
git clone https://github.com/iobridge/thingspeak.git thingspeak
cd thingspeak
```

(For a private fork, clone your own URL instead.)

---

## 10. Configure the Application

### 10.1 Database config

```bash
cp config/database.yml.example config/database.yml
$EDITOR config/database.yml
```

Set the `production:` block to your real MySQL credentials from §7:

```yaml
production:
  adapter: mysql2
  encoding: utf8
  reconnect: true
  database: thingspeak_production
  pool: 5
  username: thing
  password: "YOUR_MYSQL_PASSWORD"
  socket: /var/run/mysqld/mysqld.sock
```

### 10.2 Secret token

`config/initializers/secret_token.rb` ships with a placeholder. Generate a fresh one:

```bash
rake secret        # prints a long hex string
```

Paste it as the `secret_key_base` value. **Never commit this file.**

### 10.3 Email (optional but recommended)

Edit `config/environment.rb` — it already contains a Gmail SMTP template
(`:address => 'smtp.gmail.com'`). Set `:domain`, `:user_name`, and `:password`
to your own account (use an app password). Without this, sign-up/notification
mail silently fails but the app still runs.

### 10.4 Application text / name

`config/locales/en.yml` — set `application_name` and any wording you like.
Indentation must be spaces, not tabs.

### 10.5 Install gems

```bash
bundle install
```

This compiles native gems (`mysql2`, `nokogiri`, `therubyracer`, `kgio`) —
expect several minutes on a Pi/small VPS.

---

## 11. Database Setup

```bash
RAILS_ENV=production rake db:create
RAILS_ENV=production rake db:schema:load     # initial schema
# (on upgrades use: RAILS_ENV=production rake db:migrate)
```

### Create the admin user

```bash
RAILS_ENV=production rails console
```

```ruby
AdminUser.create!(email: 'you@example.com',
                  password: 'STRONG_PASSWORD',
                  password_confirmation: 'STRONG_PASSWORD')
exit
```

The admin panel lives at `http://YOUR_DOMAIN/admin` (ActiveAdmin).

---

## 12. Precompile Assets

```bash
RAILS_ENV=production rake assets:precompile
```

Output goes to `public/assets`. `non-stupid-digest-assets` keeps undigested
copies, so nginx static serving works without extra rules. Re-run this after
every code update that touches assets.

---

## 13. Run the App: Puma + Resque Workers + Clockwork

⚠️ The Gemfile puts `puma` in the `:development` group. For production installs
either move it out of the group, or install with:

```bash
bundle install --without test development
gem list puma   # confirm present; if not, add `gem 'puma'` outside groups and re-bundle
```

### 13.1 Smoke-test manually first

Terminal 1 — web app:
```bash
cd /opt/apps/thingspeak
RAILS_ENV=production bundle exec puma -C config/puma.rb -t 2:4 -b tcp://127.0.0.1:3000
```
(No `config/puma.rb` in-repo? Plain `bundle exec puma -b tcp://127.0.0.1:3000` is fine.)

Terminal 2 — background jobs:
```bash
RAILS_ENV=production COUNT=1 QUEUE=* bundle exec rake resque:work
```

Terminal 3 — scheduler (this repo does not ship a `clock.rb`; create one at the
repo root listing the recurring tasks you need, e.g. feed refresh/cleanup):
```bash
# clock.rb (example)
require 'clockwork'
module Clockwork
  handler { |job| Thingspeak::Application.initialize! unless defined?(Rails) && Rails.application }
  every(5.minutes, 'refresh.feeds') { Feed.all.each(&:trigger!) }
end
```
```bash
RAILS_ENV=production bundle exec clockwork clock.rb
```

Browse to `http://127.0.0.1:3000/` — you should see the ThingSpeak homepage.
Once all three work by hand, wire them into systemd (§16) for production.

### 13.2 Worker sizing

One Resque worker handles feeds/cleanup fine for personal use. Add workers
(`COUNT=2..4`) if channels update heavily. Each Rails process costs ~150–300 MB.

---

## 14. Reverse Proxy with Nginx

Create `/etc/nginx/sites-available/thingspeak`:

```nginx
upstream thingspeak {
    server 127.0.0.1:3000;
}

server {
    listen 80;
    server_name thingspeak.example.com;   # or your LAN IP, e.g. 192.168.1.50

    root /opt/apps/thingspeak/public;

    access_log /var/log/nginx/thingspeak.access.log;
    error_log  /var/log/nginx/thingspeak.error.log;

    client_max_body_size 10M;

    # serve precompiled assets directly
    location /assets {
        try_files $uri @thingspeak;
        expires 30d;
        add_header Cache-Control "public";
    }

    location / {
        proxy_set_header Host              $http_host;
        proxy_set_header X-Real-IP           $remote_addr;
        proxy_set_header X-Forwarded-For     $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto   $scheme;
        proxy_redirect off;
        proxy_pass http://thingspeak;
    }
}
```

Enable and reload:

```bash
sudo ln -s /etc/nginx/sites-available/thingspeak /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

Test: `curl -I http://thingspeak.example.com/` should return `HTTP/1.1 200`.

---

## 15. HTTPS with Let's Encrypt (VPS)

Home machines behind NAT: use DNS-01 (e.g. `certbot-dns-cloudflare`) or a
Cloudflare Tunnel instead of HTTP-01.

```bash
sudo apt-get -y install certbot python3-certbot-nginx
sudo certbot --nginx -d thingspeak.example.com
sudo certbot renew --dry-run
```

Certbot edits the nginx site to add TLS and redirect port 80→443. Renewals run
automatically via systemd timer (`systemctl list-timers | grep certbot`).

After enabling HTTPS, in Rails ensure `config/environments/production.rb` has
`config.force_ssl = true` if you want HSTS-style redirects at the app layer too.

---

## 16. Process Management with systemd

Run everything as the `deploy` user. Adjust `Environment=` lines to your paths
and secrets.

### 16.1 Web app — `/etc/systemd/system/thingspeak-puma.service`

```ini
[Unit]
Description=ThingSpeak Puma
After=network.target mysql.service redis-server.service

[Service]
Type=notify
User=deploy
WorkingDirectory=/opt/apps/thingspeak
Environment=RUBY_HOME=/home/deploy/.rvm/rubies/ruby-2.1.0
Environment=PATH=/home/deploy/.rvm/gems/ruby-2.1.0/bin:/home/deploy/.rvm/rubies/ruby-2.1.0/bin:/usr/local/bin:/usr/bin:/bin
Environment=RAILS_ENV=production
Environment=REDIS_URL=redis://:YOUR_REDIS_PASSWORD@127.0.0.1:6379/0
ExecStart=/bin/bash -lc 'cd /opt/apps/thingspeak && exec bundle exec puma -t 2:4 -b tcp://127.0.0.1:3000'
Restart=always
RestartSec=5
StandardOutput=append:/var/log/thingspeak/puma.log
StandardError=append:/var/log/thingspeak/puma.err.log

[Install]
WantedBy=multi-user.target
```

### 16.2 Resque worker — `/etc/systemd/system/thingspeak-resque@.service`

```ini
[Unit]
Description=ThingSpeak Resque worker %i
After=network.target redis-server.service mysql.service

[Service]
Type=simple
User=deploy
WorkingDirectory=/opt/apps/thingspeak
Environment=PATH=/home/deploy/.rvm/gems/ruby-2.1.0/bin:/home/deploy/.rvm/rubies/ruby-2.1.0/bin:/usr/local/bin:/usr/bin:/bin
Environment=RAILS_ENV=production
Environment=BACKGROUND=yes
Environment=QUEUE=*
Environment=COUNT=1
Environment=REDIS_URL=redis://:YOUR_REDIS_PASSWORD@127.0.0.1:6379/0
ExecStart=/bin/bash -lc 'cd /opt/apps/thingspeak && exec bundle exec rake resque:work'
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 16.3 Clockwork — `/etc/systemd/system/thingspeak-clockwork.service`

Same skeleton as the worker, but:

```ini
Description=ThingSpeak Clockwork scheduler
ExecStart=/bin/bash -lc 'cd /opt/apps/thingspeak && exec bundle exec clockwork clock.rb'
Environment=CLOCKWORK=true
```

### 16.4 Enable everything

```bash
sudo mkdir -p /var/log/thingspeak && sudo chown deploy /var/log/thingspeak
sudo systemctl daemon-reload
sudo systemctl enable --now thingspeak-puma
sudo systemctl enable --now thingspeak-resque@1 thingspeak-resque@2
sudo systemctl enable --now thingspeak-clockwork
systemctl status thingspeak-* --no-pager
```

---

## 17. Firewall & Security Hardening

### UFW (works on both VPS and home)

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
# LAN-only home box: restrict web to your subnet instead of 80/443 everywhere:
#   sudo ufw allow from 192.168.1.0/24 to any port 80,443 proto tcp
sudo ufw enable
```

**Never** open 3306 (MySQL), 6379 (Redis), or 3000 (Puma) — they bind to
127.0.0.1 and are reached through nginx only.

### Checklist

- [ ] App runs as non-root `deploy` user.
- [ ] `config/database.yml`, `secret_token.rb` mode `600`, not in git.
- [ ] MySQL root + `thing` passwords strong; Redis `requirepass` set.
- [ ] SSH key-only auth; `fail2ban` installed (`sudo apt-get -y install fail2ban`).
- [ ] Admin panel (`/admin`) protected by strong AdminUser credentials.
- [ ] TLS on; force HTTPS in Rails/nginx.
- [ ] Unattended security updates enabled (esp. home machines).

---

## 18. Backups & Restore

### 18.1 Dump script — `/opt/apps/thingspeak/script/backup.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail
DEST=/var/backups/thingspeak
STAMP=$(date +%F_%H%M)
mkdir -p "$DEST"
mysqldump --single-transaction -u thing -p"YOUR_MYSQL_PASSWORD" \
  thingspeak_production | gzip > "$DEST/db_$STAMP.sql.gz"
tar czf "$DEST/files_$STAMP.tar.gz" -C /opt/apps/thingspeak public/uploads config/database.yml
find "$DEST" -type f -mtime +14 -delete    # keep 14 days
```

```bash
chmod +x script/backup.sh
sudo crontab -u deploy -e
# nightly at 02:30
30 2 * * * /opt/apps/thingspeak/script/backup.sh >> /var/log/thingspeak/backup.log 2>&1
```

Copy backups off-box (VPS: `rclone` to S3/B2; home: rsync to NAS or cloud).
A backup that lives only on the same disk is not a backup.

### 18.2 Restore

```bash
gunzip -c db_YYYY-MM-DD_HHMM.sql.gz | mysql -u thing -p thingspeak_production
tar xzf files_*.tar.gz -C /opt/apps/thingspeak
```

---

## 19. Upgrades & Rolling Updates

```bash
cd /opt/apps/thingspeak
sudo systemctl stop thingspeak-puma thingspeak-resque@* thingspeak-clockwork
git pull origin master
bundle install
RAILS_ENV=production rake db:migrate
RAILS_ENV=production rake assets:precompile
sudo systemctl start thingspeak-puma thingspeak-resque@1 thingspeak-resque@2 thingspeak-clockwork
```

Take a §18 backup **before** every upgrade. Tag your known-good commit
(`git tag -a prod-2026-11-22 -m ok`) so you can `git checkout` back quickly.

---

## 20. Monitoring & Logs

| What | Where / how |
|---|---|
| Rails logs | `log/production.log` (tail: `tail -f log/production.log`) |
| Puma / nginx | `/var/log/thingspeak/puma.log`, `/var/log/nginx/thingspeak.*.log` |
| systemd state | `systemctl status thingspeak-puma`, `journalctl -u thingspeak-puma -e` |
| Resque queue depth | Resque web UI (mount `Resque::Server` in routes for admins) or `redis-cli llen resque:queue:*` |
| Health probe | `curl -fsS https://thingspeak.example.com/channels/public.json >/dev/null && echo OK` via cron/UptimeRobot |
| Resource use | `htop`, `df -h`, `free -h` |

Rotate Rails logs with `/etc/logrotate.d/thingspeak`:

```
/opt/apps/thingspeak/log/*.log {
    daily
    rotate 14
    compress
    missingok
    delaycompress
    copytruncate
}
```

The repo also includes `lib/server_stats.sh` for quick stats collection.

---

## 21. Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `Can't connect to local MySQL server through socket` | Wrong `socket:` path in `database.yml`; check `mysqld --socket`, or MySQL not running. |
| `Access denied for user 'thing'@'localhost'` | Password mismatch between MySQL grants and `database.yml`. |
| 502 Bad Gateway from nginx | Puma not running (`systemctl status thingspeak-puma`); check puma.err.log. |
| `bundle install` fails on `therubyracer`/`libv8` | Old gem vs new compiler; try `gem install libv8 -v 3.16.14.3` first, or use a source build flag; on ARM, expect long compile. |
| Ruby 2.1 won't compile on Ubuntu 22.04+ | Use Ubuntu 18.04/20.04, or `rvm pkg install openssl` and pass `--with-openssl-dir`. |
| Jobs queue but never run | Resque worker down or wrong Redis DB/password; check `REDIS_URL` matches between app env and workers. |
| Feeds/charts blank after deploy | Forgot `rake assets:precompile`; hard-refresh browser cache. |
| Timezone weirdness in charts | Set correct timezone per user/channel; `tzinfo-data` gem is included for platforms without zoneinfo. |
| Out of memory / OOM-killed puma | Reduce `-t` threads and worker count; add swap (§5.4). |
| ActiveAdmin breaks under clockwork | Expected — `config/application.rb` removes admin paths when `$clockwork == true`; run clockwork as its own process only. |

---

## 22. Appendix: Quick Reference Cheatsheet

```bash
# --- bring up from scratch (Debian/Ubuntu) ---
apt-get update && apt-get -y install build-essential git curl mysql-server \
  mysql-client libmysqlclient-dev redis-server libxml2-dev libxslt1-dev \
  libssl-dev libyaml-dev nginx
curl -sSL https://get.rvm.io | bash -s stable && source ~/.rvm/scripts/rvm
rvm install 2.1 && rvm use 2.1 --default
gem install bundler -v '~> 1.17'

git clone https://github.com/iobridge/thingspeak.git /opt/apps/thingspeak
cd /opt/apps/thingspeak
cp config/database.yml.example config/database.yml   # EDIT credentials
bundle install
RAILS_ENV=production rake db:create db:schema:load assets:precompile
RAILS_ENV=production rails console  # AdminUser.create!(...)

# --- day-to-day ---
sudo systemctl restart thingspeak-puma thingspeak-resque@1 thingspeak-clockwork
tail -f log/production.log
redis-cli info clients
mysqldump -u thing -p thingspeak_production | gzip > backup.sql.gz
```

---

*Document created 2026-11-22 for the iobridge/thingspeak Rails 4 codebase.
Adjust paths, users, and versions to your environment; always test restores
from your backups.*
