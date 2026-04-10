# Hysteria2 + WARP: установка на чистый сервер

> **Предварительные условия:**
> - Сервер без nginx и других сервисов, занимающих порт 443
> - Домен с A-записью, указывающей на IP сервера

---

## Часть 1: Hysteria2

### 1. Обновить систему

```bash
apt update && apt upgrade -y
```

### 2. Открыть порты

```bash
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 443/udp
ufw --force enable
```

### 3. Установить Hysteria2

```bash
bash <(curl -fsSL https://get.hy2.sh/)
```

### 4. Записать конфиг

Замени `ТВОЙ_ДОМЕН`, `ТВОЙ_EMAIL`, `ИМЯ` и `ПАРОЛЬ` на свои значения:

```bash
cat > /etc/hysteria/config.yaml <<'EOF'
listen: 0.0.0.0:443

acme:
  type: http
  domains:
    - ТВОЙ_ДОМЕН
  email: ТВОЙ_EMAIL

auth:
  type: userpass
  userpass:
    ИМЯ: ПАРОЛЬ

masquerade:
  type: proxy
  proxy:
    url: https://www.bing.com
    rewriteHost: true
EOF
```

> Hysteria2 автоматически получит SSL-сертификат через Let's Encrypt при первом запуске.

### 5. Запустить и проверить

```bash
systemctl enable --now hysteria-server
systemctl status hysteria-server
ss -ulnp | grep 443
```

> Команда `ss` должна показать строку с `hysteria` и `0.0.0.0:443`.

---

### Клиентские конфиги

**Ссылка для Hiddify / HAPP / v2rayNG:**

```
hy2://ИМЯ:ПАРОЛЬ@ТВОЙ_ДОМЕН:443?sni=ТВОЙ_ДОМЕН#HY2
```

**Конфиг для Stash (iOS):**

```yaml
proxies:
  - name: HY2
    type: hysteria2
    server: ТВОЙ_ДОМЕН
    port: 443
    auth: "ИМЯ:ПАРОЛЬ"
    fast-open: true
    sni: ТВОЙ_ДОМЕН
    skip-cert-verify: false
    up-speed: 50
    down-speed: 200

proxy-groups:
  - name: Proxy
    type: select
    proxies:
      - HY2
      - DIRECT

rules:
  - MATCH,Proxy
```

> **Особенности Stash:**
> - Поле называется `auth`, а не `password`. Формат: `username:password`
> - `obfs` в Stash не поддерживается

---

> ✅ **На этом этапе Hysteria2 полностью работает. Часть 2 с WARP — опциональное улучшение.**

---

## Часть 2: WARP (опционально)

> Добавляется поверх рабочего Hysteria2. Клиентские конфиги менять не нужно.  
> После подключения на `ip.me` будет отображаться IP Cloudflare (`104.x.x.x` или `162.x.x.x`).

### 1. Установить и подключить WARP

```bash
curl -fsSL https://pkg.cloudflareclient.com/pubkey.gpg \
  | gpg --dearmor -o /usr/share/keyrings/cloudflare-warp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/cloudflare-warp-archive-keyring.gpg] \
  https://pkg.cloudflareclient.com/ $(lsb_release -cs) main" \
  | tee /etc/apt/sources.list.d/cloudflare-client.list

apt update && apt install cloudflare-warp -y

warp-cli --accept-tos registration new
warp-cli --accept-tos mode proxy
warp-cli --accept-tos connect
warp-cli status

systemctl enable warp-svc
```

> Должно отобразиться: `Status update: Connected` и `Network: healthy`

### 2. Обновить конфиг Hysteria2

```bash
cat > /etc/hysteria/config.yaml <<'EOF'
listen: 0.0.0.0:443

acme:
  type: http
  domains:
    - ТВОЙ_ДОМЕН
  email: ТВОЙ_EMAIL

auth:
  type: userpass
  userpass:
    ИМЯ: ПАРОЛЬ

outbounds:
  - name: warp
    type: socks5
    socks5:
      addr: 127.0.0.1:40000

masquerade:
  type: proxy
  proxy:
    url: https://www.bing.com
    rewriteHost: true
EOF
```

### 3. Настроить запуск после WARP

Создай override-файл, чтобы Hysteria2 стартовал только после запуска WARP:

```bash
mkdir -p /etc/systemd/system/hysteria-server.service.d

cat > /etc/systemd/system/hysteria-server.service.d/override.conf <<'EOF'
[Unit]
After=warp-svc.service
Requires=warp-svc.service
EOF
```

Применить изменения и перезапустить:

```bash
systemctl daemon-reload
systemctl restart hysteria-server
systemctl status hysteria-server
```

## Часть 3 — Два профиля: с WARP и без (опционально)
 
> Порт `:443` — без WARP, порт `:4443` — с WARP.
 
```bash
ufw allow 4443/udp
```
 
**Конфиг без WARP** → `/etc/hysteria/config-direct.yaml`:
 
```bash
cat > /etc/hysteria/config-direct.yaml <<'EOF'
listen: 0.0.0.0:443
 
acme:
  type: http
  domains:
    - ТВОЙ_ДОМЕН
  email: ТВОЙ_EMAIL
 
auth:
  type: userpass
  userpass:
    ИМЯ: ПАРОЛЬ
 
masquerade:
  type: proxy
  proxy:
    url: https://www.bing.com
    rewriteHost: true
EOF
```
 
**Конфиг с WARP** → `/etc/hysteria/config.yaml` — порт `4443`:
 
```bash
cat > /etc/hysteria/config.yaml <<'EOF'
listen: 0.0.0.0:4443
 
acme:
  type: http
  domains:
    - ТВОЙ_ДОМЕН
  email: ТВОЙ_EMAIL
 
auth:
  type: userpass
  userpass:
    ИМЯ: ПАРОЛЬ
 
outbounds:
  - name: warp
    type: socks5
    socks5:
      addr: 127.0.0.1:40000
 
masquerade:
  type: proxy
  proxy:
    url: https://www.bing.com
    rewriteHost: true
EOF
```
 
Создать второй systemd-сервис:
 
```bash
cat > /etc/systemd/system/hysteria-direct.service <<'EOF'
[Unit]
Description=Hysteria2 Direct
After=network-online.target
 
[Service]
Type=simple
ExecStart=/usr/local/bin/hysteria server --config /etc/hysteria/config-direct.yaml
Restart=on-failure
 
[Install]
WantedBy=multi-user.target
EOF
 
systemctl daemon-reload
systemctl enable --now hysteria-server
systemctl enable --now hysteria-direct
```
 
**Клиентские ссылки:**
```
hy2://ИМЯ:ПАРОЛЬ@ТВОЙ_ДОМЕН:443?sni=ТВОЙ_ДОМЕН#HY2-direct
hy2://ИМЯ:ПАРОЛЬ@ТВОЙ_ДОМЕН:4443?sni=ТВОЙ_ДОМЕН#HY2-warp
```
 
---
 
## Часть 4 — Несколько пользователей
 
Добавь нужные пары `логин: пароль` в блок `userpass` любого конфига:
 
```yaml
auth:
  type: userpass
  userpass:
    user1: pass1
    user2: pass2
    user3: pass3
```
 
```bash
systemctl restart hysteria-server
systemctl restart hysteria-direct   # если запущен
```
 
