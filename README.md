<div align="center">

# 🔁 kfwl.1337.cx

**VPN Config Converter** — серверлесс-конвертер подписок, парсер прокси-линков, расшифровщик happ-крипт и YouTube-даунлоадер. Один HTML, один Node-сервер, один Rust-бинарь, один SSL-сертификат.

[![status](https://img.shields.io/badge/status-production-22c55e)]()
[![host](https://img.shields.io/badge/host-kfwl.1337.cx-7c3aed)]()
[![node](https://img.shields.io/badge/node-18%2B-339933)]()
[![nginx](https://img.shields.io/badge/nginx-LE_HTTPS-009639)]()
[![license](https://img.shields.io/badge/license-Proprietary-red)]()

</div>

---

## ⚠️ Дисклеймер

ПО **проприетарное**. Исходники в открытом доступе **не выкладываются** — этот документ описывает только архитектуру и публичный API сервиса. Любая публичная информация о ноде, ключе или подписке в примерах — **синтетика**, реальные конфиги в README не утекают.

Сервис предоставляется **as-is**, без SLA, для личного и ознакомительного использования. Использование сторонних подписок — на ответственности пользователя; их содержимое не сохраняется на сервере.

---

## 📋 Содержание

- [Что это](#-что-это)
- [Архитектура](#-архитектура)
- [Фичи фронта](#-фичи-фронта)
- [Публичный API](#-публичный-api)
  - [POST /api/fetch — Happ-крипт через crypto.happ.su](#post-apifetch--happ-крипт-через-cryptohappsu)
  - [GET /api/fetch — серверный прокси подписок](#get-apifetch--серверный-прокси-подписок)
  - [GET /api/challenge — ALTCHA challenge](#get-apichallenge--altcha-challenge)
  - [GET /api/dl — YouTube downloader](#get-apidl--youtube-downloader)
  - [POST /api/incy/happ-local — офлайн-расшифровка Crypt5](#post-apiincyhapp-local--офлайн-расшифровка-crypt5)
  - [GET /api/incy/health](#get-apiincyhealth)
- [Безопасность](#-безопасность)
- [Лимиты и таймауты](#-лимиты-и-таймауты)
- [Deploy](#-deploy)
- [FAQ](#-faq)

---

## 🎯 Что это

`kfwl.1337.cx` — однофайловый веб-конвертер для прокси/VPN-инфраструктуры. Решает четыре боли одним сайтом:

1. **Подписки не открываются в браузере** (CORS, отсутствие `User-Agent`, `x-hwid`) → серверный прокси `/api/fetch`.
2. **Happ Crypt-ссылки нечитаемы** (`happ://crypt5/...`) → две независимые расшифровки: онлайн (`happy-decoder.cc`) и офлайн (локальный Rust-бинарь).
3. **Нужно быстро перепарсить/переименовать/пересобрать линки** → парсер с экспортом в `Selector` / `URLTest` / Base64 / Subscription JSON.
4. **YouTube без рекламы и без VPN на клиенте** → `/api/dl` с серверным `yt-dlp` + `ffmpeg` (файл не хранится).

Всё это под одним SSL-доменом, без аккаунтов, без БД, без логов содержимого.

---

## 🏗 Архитектура

```
          Internet
             │
     ┌───────▼────────┐
     │ nginx :443 LE  │  kfwl.1337.cx (Let's Encrypt, HSTS by default config)
     └───┬────────┬───┘
         │        │
  /api/incy/*    /api/*  (все остальные — /api/challenge, /api/fetch, /api/dl)
         │        │
   ┌─────▼──┐  ┌──▼──────────────────────┐
   │incy-svc│  │ server.js (Node 18+, 0-dep)
   │:8002   │  │ :3000 / 127.0.0.1        │
   │Python  │  │  • /api/challenge ALTCHA │
   │+Rust   │  │  • /api/fetch (proxy+POST)
   │ bin    │  │  • /api/dl   (yt-dlp+ffmpeg)
   └────────┘  └──────────────────────────┘
        │
  /opt/incy-svc/bin/happ-decrypt   ← Rust-бинарь happ-decrypt-universal v1.0.0
```

**Фронт**: `/var/www/vpnconv/index.html` (~236 KB), vanilla JS + Tailwind через CDN + FontAwesome. SPA с табами. Никаких node_modules в браузере.

**Хранение**: ничего. Подписки и видео не пишутся на диск (видео — во `mkdtemp`, удаляется сразу после стрима). `/api/fetch` не логирует тело ответа.

---

## 🧰 Фичи фронта

Единое SPA с навигацией по табам:

| Таб | Что делает |
|-----|------------|
| **Converter** | Берёт ссылку на подписку или Crypt-ссылку, разворачивает в плоский список нод. Поддерживает Xray-JSON, sing-box, Base64, plain-list, YAML (Clash/Mihomo). |
| **Rename** | Массовое переименование нод: regex, prefix/suffix, нумерация, добавление флагов через GeoIP (опц.), удаление эмодзи. |
| **Parser** | Парсинг сырых линков (`vless://`, `vmess://`, `ss://`, `trojan://`, `hy2://`, `tuic://`, `socks://`) с валидацией параметров. Источник Crypt: **Авто / API / Local** (см. ниже). |
| **QR** | Генерация QR-кодов из любой строки/ссылки, экспорт PNG. |
| **Decode** | URL-decode/encode (percent-encoding), Base64 туда-обратно, JSON pretty-printer. |
| **YTDL** | YouTube → `mp4`. Качество: `max` (до 4K, VP9/AV1) или `1080/720/480/360` (H.264 для совместимости). |
| **Themes** | Палитра: solid, gradient mix, custom contrast pair, lg-toggle (light-glass), 9+ пресетов. Сохраняется в `localStorage`. |
| **My IP** | Показывает текущий IP клиента (`/api/myip`), кнопка-глаз для скрытия. |

### Источник Crypt (на табе Parser/Converter)

Радиогруппа **«Источник»** контролирует, как расшифровываются `happ://crypt5/...`:

- **Авто** — сначала пробует онлайн (`happy-decoder.cc`), при фейле — локальный Rust-бинарь.
- **API** — только онлайн (`happy-decoder.cc`).
- **Local** — только Rust (`/api/incy/happ-local`). Работает офлайн, без зависимости от стороннего сервиса.

---

## 🌐 Публичный API

Все ответы CORS-открытые (`Access-Control-Allow-Origin: *`). Все ответы `Cache-Control: no-store`.

### `POST /api/fetch` — Happ-крипт через `crypto.happ.su`

Прокси-обёртка над официальным шифровальщиком ссылок Happ. Принимает обычную ссылку, отдаёт её зашифрованную форму.

**Body:** `{"url": "<любая ссылка>"}`

```bash
curl -sS -X POST https://kfwl.1337.cx/api/fetch \
  -H 'Content-Type: application/json' \
  -d '{"url":"https://example.com/sub/abc"}'
```

**Ответ:** `text/plain`, тело — то что вернул upstream (обычно одна строка с зашифрованной ссылкой). Статус-код проксируется как есть.

---

### `GET /api/fetch` — серверный прокси подписок

Забирает подписку с указанного URL, подставляя нужный `User-Agent` и опционально `x-hwid`. Решает CORS + невозможность поставить UA из браузера.

**Query:**

| Параметр | Тип | Описание |
|---|---|---|
| `url` | **required** | Прямой HTTPS/HTTP URL подписки. Приватные хосты заблокированы (см. [Безопасность](#-безопасность)). |
| `ua` | string | User-Agent. По умолчанию `Happ/3.23.0`. |
| `hwid` | string | Если задан — добавляет заголовок `x-hwid`. |
| `headers` | JSON-string | Доп. заголовки, например `{"Authorization":"Bearer ..."}`. |
| `selftest` | `1` | Проверка деплоя — вернёт `{ok:true, build, node, hasGlobalFetch}`. |

```bash
# Простое получение подписки от Happ
curl -sS "https://kfwl.1337.cx/api/fetch?url=https%3A%2F%2Fsub.example%2Fxxx&ua=Happ%2F3.24.0"

# С hwid
curl -sS "https://kfwl.1337.cx/api/fetch?url=...&ua=v2rayNG%2F1.10.6&hwid=A1B2C3"

# Selftest
curl -sS "https://kfwl.1337.cx/api/fetch?selftest=1"
```

**TLS-fallback**: если у upstream битый/самоподписанный сертификат и `fetch()` падает с `fetch failed`, прокси автоматически переключается на сырой `https.request` с `rejectUnauthorized: false`. Это позволяет тянуть подписки с IP-хостов без валидных SAN.

**Ошибки:**
- `400` — `missing url`, `bad url`, `only http/https allowed`
- `403` — `forbidden host` (приватный диапазон, см. ниже)
- `502` — `fetch failed`, `proxy error` (с полями `detail`, `cause`, `fallback`, `fallbackCause`)

---

### `GET /api/challenge` — ALTCHA challenge

Выдаёт proof-of-work задачу для антибот-капчи. Используется фронтом перед запросом `/api/dl`.

```bash
curl -sS https://kfwl.1337.cx/api/challenge
```

Ответ:

```json
{
  "algorithm": "SHA-256",
  "challenge": "<hex>",
  "maxnumber": 1000000,
  "salt": "<hex>",
  "signature": "<hmac>"
}
```

Клиент решает задачу (находит nonce, при котором `SHA-256(salt + nonce) == challenge`), отправляет ответ как Base64-JSON в параметре `altcha` следующих защищённых эндпоинтов.

---

### `GET /api/dl` — YouTube downloader

Качает YouTube-видео на сервер через `yt-dlp`, склеивает с аудио через `ffmpeg`, стримит клиенту и **сразу удаляет**.

**Query:**

| Параметр | Описание |
|---|---|
| `url` | **required** YouTube-ссылка (`youtube.com`, `youtu.be`, `youtube-nocookie.com`). Другие домены — `400`. |
| `altcha` | **required** Решённый ALTCHA-токен из `/api/challenge`. Без него `403`. |
| `q` | Качество: `max` / `1080` / `720` / `480` / `360`. По умолчанию `1080`. `max` тянет до 4K (VP9/AV1, может не играть на старых девайсах). Остальные — H.264 + mp4 для максимальной совместимости. |
| `t` | Опц. токен сессии (≤48 alnum). Если задан — сервер пишет cookie `dl_<t>=1` при начале отдачи, чтобы фронт показал прогресс. |

```bash
# через фронт обычно — но можно и руками после решения капчи:
curl -sS "https://kfwl.1337.cx/api/dl?url=https%3A%2F%2Fyoutu.be%2FdQw4w9WgXcQ&q=1080&altcha=<base64>" \
     -o video.mp4
```

**Поведение:**
- Файл стримится напрямую через `pipe()`, без хранения целиком в RAM.
- Если клиент закрывает соединение — `yt-dlp` убивается `SIGKILL`, временная папка чистится.
- `Content-Disposition: attachment; filename*=UTF-8''<urlencoded-title>` — клиент получает оригинальное имя ролика.

**Ошибки:**
- `400` — `missing url`, `Только ссылки YouTube`
- `403` — `Проверка не пройдена` (ALTCHA)
- `500` — `yt-dlp не запустился` (бинаря нет в PATH), `Не удалось скачать видео` (с последними 1000 байт `stderr`), `Файл не найден`

---

### `POST /api/incy/happ-local` — офлайн-расшифровка Crypt5

Использует локальный Rust-бинарь `happ-decrypt-universal v1.0.0` для расшифровки `happ://crypt5/...` без обращения к сторонним сервисам.

```bash
curl -sS -X POST https://kfwl.1337.cx/api/incy/happ-local \
  -H 'Content-Type: application/json' \
  -d '{"link":"happ://crypt5/xxxxxxxxxxxx"}' | python3 -m json.tool
```

**Ответ:**

```json
{
  "ok": true,
  "plain": "vless://uuid@host:443?...",
  "source": "local"
}
```

**Когда полезно:** API `happy-decoder.cc` лежит, rate-limit, или хочется обработать большой батч без зависимости от стороннего сервиса.

---

### `GET /api/incy/health`

Проверка живости sidecar.

```bash
curl -sS https://kfwl.1337.cx/api/incy/health
# {"ok":true,"binary":"/opt/incy-svc/bin/happ-decrypt","version":"1.0.0"}
```

---

## 🛡 Безопасность

### SSRF-guard на `/api/fetch`

Сервер отказывает в запросе, если хост попадает в один из приватных/служебных диапазонов:

- `localhost`, `::1`, `*.local`
- `127.0.0.0/8`
- `10.0.0.0/8`
- `172.16.0.0/12`
- `192.168.0.0/16`
- `169.254.0.0/16` (link-local)
- `0.0.0.0/8`

Это блокирует попытки протянуть прокси к внутренним сервисам инфраструктуры через user-controlled URL.

### ALTCHA на `/api/dl`

Proof-of-work капча обязательна **до запуска `yt-dlp`** — чтобы ботнет не сжёг CPU/трафик. Без валидного токена — `403`, без затрат на upstream.

### Только разрешённые схемы

`/api/fetch` принимает только `http://` и `https://`. Любые `file://`, `gopher://`, `dict://` отбрасываются `400 only http/https allowed`.

### Нет персистентного хранения

- Подписки проксируются стримом, не пишутся на диск.
- YouTube-файлы пишутся в `mkdtempSync()` и удаляются в `finally`/`on("close")`/`on("error")`.
- Сервер не логирует тела ответов. В журнале только статусы и stderr-хвосты `yt-dlp` (последние 6 KB).

### Капча перед расшифровкой

Фронт перед `/api/dl` и перед массовой расшифровкой Crypt-ссылок требует решённый ALTCHA-токен (`#captchaGate` блокирует UI до прохождения).

---

## ⏱ Лимиты и таймауты

| Эндпоинт | Лимит |
|---|---|
| `/api/fetch` upstream timeout | 20 сек |
| `/api/fetch` редиректов | до 5 hop |
| `/api/incy/*` proxy_read_timeout | 60 сек |
| `/api/incy/*` body | 4 MB |
| `/api/*` proxy_read_timeout | 600 сек (для долгих yt-dlp) |
| `/api/dl` stderr buffer | 6 KB (rolling) |
| `/api/dl` cookie `dl_<t>` | TTL 120 сек |
| `t` (download token) | до 48 alnum символов |
| ALTCHA `maxnumber` | 1 000 000 |

---

## 🚀 Deploy

### Стек
- **OS**: Fedora с SELinux
- **HTTPS**: nginx + Certbot (Let's Encrypt, авторенью через cron)
- **Backend 1**: `node server.js` (порт `127.0.0.1:3000`)
- **Backend 2**: `incy-svc` (systemd, Python + Rust бинарь, порт `127.0.0.1:8002`)

### Обновление фронта

```bash
sudo unzip -o ~/Загрузки/conv-patched.zip index.html -d /var/www/vpnconv
sudo chown xray:xray /var/www/vpnconv/index.html
sudo chcon -t httpd_sys_content_t /var/www/vpnconv/index.html
```

Затем **Ctrl+Shift+R** в браузере.

> ⚠️ В zip’е **нет** `altcha.js` (намеренно, чтобы не перезатереть актуальный с ключом). Если уж надо обновить капчу — копируй отдельно.

### Обновление sidecar (incy-svc)

```bash
sudo cp /opt/incy-svc/server.py /opt/incy-svc/server.py.bak.$(date +%s)
sudo unzip -o ~/Загрузки/incy-svc-patch.zip server.py -d /opt/incy-svc
sudo chown xray:xray /opt/incy-svc/server.py
sudo systemctl restart incy-svc
sudo systemctl status incy-svc --no-pager | head -20
```

### Установка Rust-бинаря (happ-decrypt)

```bash
sudo mkdir -p /opt/incy-svc/bin
sudo curl -L -o /opt/incy-svc/bin/happ-decrypt \
  https://github.com/amurcanov/happ-decrypt-universal/releases/download/v1.0.0/linux-x64_x86
sudo chmod +x /opt/incy-svc/bin/happ-decrypt
sudo chown -R xray:xray /opt/incy-svc/bin
sudo chcon -R -t bin_t /opt/incy-svc/bin
/opt/incy-svc/bin/happ-decrypt --help 2>&1 | head -3
```

### nginx (фрагмент `/etc/nginx/conf.d/vpnconv.conf`)

```nginx
server {
    server_name kfwl.1337.cx;
    root /var/www/vpnconv;
    index index.html;

    location / { try_files $uri $uri/ =404; }

    location /api/incy/ {
        proxy_pass http://127.0.0.1:8002;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 60;
        client_max_body_size 4M;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 600;
    }

    listen 443 ssl;
    ssl_certificate     /etc/letsencrypt/live/kfwl.1337.cx/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/kfwl.1337.cx/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}

server {
    listen 80;
    server_name kfwl.1337.cx;
    if ($host = kfwl.1337.cx) { return 301 https://$host$request_uri; }
    return 404;
}
```

### Тест после установки

```bash
curl -s http://127.0.0.1:8002/api/incy/health
curl -s -X POST http://127.0.0.1:8002/api/incy/happ-local \
  -H 'Content-Type: application/json' \
  -d '{"link":"happ://crypt5/test"}' | python3 -m json.tool

curl -s https://kfwl.1337.cx/api/fetch?selftest=1 | jq
```

---

## ❓ FAQ

**Q: Где исходники?**
A: Бинарный артефакт под прод раздаётся отдельно. Этот README — публичная документация API.

**Q: Сохраняются ли мои подписки на сервере?**
A: Нет. `/api/fetch` стримит ответ от upstream к клиенту без записи на диск. Сервер не пишет ни тело запроса, ни тело ответа.

**Q: Сохраняются ли видео с YouTube?**
A: Нет. Файл создаётся во временной директории через `mkdtempSync`, стримится клиенту, **синхронно удаляется** при `end`, `close`, `error` или дисконнекте клиента.

**Q: Можно ли использовать API напрямую без UI?**
A: Да, все эндпоинты публичные. `/api/dl` требует ALTCHA — придётся реализовать решение PoW (см. `altcha.js` в любом клиенте).

**Q: Что если `happy-decoder.cc` недоступен?**
A: Переключи источник Crypt на **Local** в UI или используй `/api/incy/happ-local` напрямую — он расшифровывает через локальный Rust-бинарь без обращения к стороннему сервису.

**Q: Почему я получаю `403 forbidden host`?**
A: SSRF-guard. URL подписки указывает на приватный диапазон (`127.x`, `10.x`, `192.168.x`, `172.16-31.x`, `localhost`, `*.local`). Подписки публичных провайдеров под него не попадают.

**Q: Почему капча на скачивании YouTube?**
A: `yt-dlp` — тяжёлая операция (CPU + сеть + диск). PoW отсекает ботнет до того, как сервер начнёт работу. Норм пользователь решает её за 1-3 секунды.

**Q: `max` качество не воспроизводится — что делать?**
A: `max` отдаёт лучший доступный поток (часто VP9/AV1 в `webm`). Старые телевизоры и плееры на iOS<14 могут не уметь. Перекачай в `1080` — это H.264 в `mp4`, играет везде.

**Q: Поддерживаются ли VK, TikTok, Twitter?**
A: Нет, только YouTube (`youtube.com`, `youtu.be`, `youtube-nocookie.com`). Регексп вшит в `dl.js`.

**Q: Можно ли поставить свой self-signed cert на upstream подписки?**
A: Да, `/api/fetch` при падении `fetch()` падает на запасной путь через `https.request({ rejectUnauthorized: false })`. TLS не проверяется во втором проходе.

---

<div align="center">

*🦆 made with duck-driven development*

</div>
