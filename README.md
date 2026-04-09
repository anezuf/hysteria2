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

> ⚠️ Используй только `printf` — nano и heredoc вносят скрытые символы которые ломают парсер.

Замени `ТВО_ДОМЕН`, `ТВО_EMAIL`, `ИМЯ` и `ПАРОЛЬ`:

```bash
printf 'listen: 0.0.0.0:443\n\nacme:\n  type: http\n  domains:\n    - ТВО_ДОМЕН\n  email: ТВО_EMAIL\n\nauth:\n  type: userpass\n  userpass:\n    ИМЯ: ПАРОЛЬ\n\nmasquerade:\n  type: proxy\n  proxy:\n    url: https://www.bing.com\n    rewriteHost: true\n' > /etc/hysteria/config.yaml
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
printf 'listen: 0.0.0.0:443\n\nacme:\n  type: http\n  domains:\n    - ТВО_ДОМЕН\n  email: ТВО_EMAIL\n\nauth:\n  type: userpass\n  userpass:\n    ИМЯ: ПАРОЛЬ\n\noutbounds:\n  - name: warp\n    type: socks5\n    socks5:\n      addr: 127.0.0.1:40000\n\nmasquerade:\n  type: proxy\n  proxy:\n    url: https://www.bing.com\n    rewriteHost: true\n' > /etc/hysteria/config.yaml
```

**3. Настроить запуск после WARP:**

```bash
systemctl edit hysteria-server
```

Вставить:

```ini
[Unit]
After=warp-svc.service
Requires=warp-svc.service
```

Сохранить (`Ctrl+X` → `Y` → `Enter`):

```bash
systemctl daemon-reload
systemctl restart hysteria-server
systemctl status hysteria-server
```
