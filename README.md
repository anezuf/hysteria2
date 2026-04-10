# Hysteria2 + WARP: Установка на чистый сервер

> Гайд для сервера без nginx и других сервисов.  
> Домен должен иметь A-запись на IP сервера.

---

## Часть 1: Hysteria2

**1. Обновить систему:**

```bash
apt update && apt upgrade -y
```

**2. Открыть порты:**

```bash
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 443/udp
ufw --force enable
```

**3. Установить Hysteria2:**

```bash
bash <(curl -fsSL https://get.hy2.sh/)
```

**4. Записать конфиг:**

Замени `ТВО_ДОМЕН`, `ТВО_EMAIL`, `ИМЯ` и `ПАРОЛЬ`:

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

Hysteria2 сам получит SSL-сертификат через Let's Encrypt при первом запуске.

**5. Запустить:**

```bash
systemctl enable --now hysteria-server
systemctl status hysteria-server
ss -ulnp | grep 443
```

`ss` должен показать строку с `hysteria` и `0.0.0.0:443`.

---

### Клиентские конфиги

Ссылка (Hiddify / HAPP / v2rayNG):
```
hy2://ИМЯ:ПАРОЛЬ@ТВО_ДОМЕН:443?sni=ТВО_ДОМЕН#HY2
```

Stash (iOS):
```yaml
proxies:
  - name: HY2
    type: hysteria2
    server: ТВО_ДОМЕН
    port: 443
    auth: "ИМЯ:ПАРОЛЬ"
    fast-open: true
    sni: ТВО_ДОМЕН
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

> В Stash поле называется `auth`, не `password`. Формат: `username:password`.  
> `obfs` в Stash не поддерживается.

✅ **На этом этапе Hysteria2 полностью работает. WARP — опциональное улучшение.**

---

---

## Часть 2: WARP (опционально)

> Добавляется поверх рабочего Hysteria2. Клиентские конфиги не меняются.  
> После подключения на `ip.me` будет показан IP Cloudflare (`104.x.x.x` или `162.x.x.x`).

**1. Установить и подключить:**

```bash
curl -fsSL https://pkg.cloudflareclient.com/pubkey.gpg | gpg --dearmor -o /usr/share/keyrings/cloudflare-warp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/cloudflare-warp-archive-keyring.gpg] https://pkg.cloudflareclient.com/ $(lsb_release -cs) main" | tee /etc/apt/sources.list.d/cloudflare-client.list
apt update && apt install cloudflare-warp -y
warp-cli --accept-tos registration new
warp-cli --accept-tos mode proxy
warp-cli --accept-tos connect
warp-cli status
systemctl enable warp-svc
```

Должно показать: `Status update: Connected` / `Network: healthy`

**2. Обновить конфиг Hysteria2:**

```bash
cat > /etc/hysteria/config.yaml <<'EOF'
listen: 0.0.0.0:443

acme:
  type: http
  domains:
    - ТВО_ДОМЕН
  email: ТВО_EMAIL

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

**3. Настроить запуск после WARP:**

Создай override-файл для hysteria-server:

```bash
mkdir -p /etc/systemd/system/hysteria-server.service.d
cat > /etc/systemd/system/hysteria-server.service.d/override.conf <<'EOF'
[Unit]
After=warp-svc.service
Requires=warp-svc.service
EOF
```

Применить изменения:

```bash
systemctl daemon-reload
systemctl restart hysteria-server
systemctl status hysteria-server
```
