<div align="center">

# 🔁 kfwl.1337.cx

**VPN Config Converter** — конвертер подписок, парсер прокси-линков, расшифровщик happ-крипт и YouTube-даунлоадер в одной веб-морде.

[![status](https://img.shields.io/badge/status-production-22c55e)]()
[![host](https://img.shields.io/badge/host-kfwl.1337.cx-7c3aed)]()
[![license](https://img.shields.io/badge/license-Proprietary-red)]()

[🌐 Открыть сайт](https://kfwl.1337.cx)

</div>

---

## ⚠️ Дисклеймер

Этот репозиторий — публичная документация публичного API сервиса [`kfwl.1337.cx`](https://kfwl.1337.cx). Исходный код сервера и фронта — проприетарный и в открытый доступ не выкладывается. Все конфиги, ссылки, ключи в примерах — синтетика.

Сервис предоставляется **as-is**, без SLA, для личного и ознакомительного использования. Использование сторонних подписок — на ответственности пользователя; их содержимое не сохраняется.

---

## 📋 Содержание

- [Что это](#-что-это)
- [Фичи](#-фичи)
- [Публичный API](#-публичный-api)
  - [POST /api/fetch — Happ-крипт через crypto.happ.su](#post-apifetch--happ-крипт-через-cryptohappsu)
  - [GET /api/fetch — серверный прокси подписок](#get-apifetch--серверный-прокси-подписок)
  - [GET /api/challenge — ALTCHA challenge](#get-apichallenge--altcha-challenge)
  - [GET /api/dl — YouTube downloader](#get-apidl--youtube-downloader)
  - [POST /api/incy/happ-local — офлайн-расшифровка Crypt5](#post-apiincyhapp-local--офлайн-расшифровка-crypt5)
  - [GET /api/incy/health](#get-apiincyhealth)
  - [POST /api/incy/parser/auto](#post-apiincyparserauto--автопарсер-серверный)
  - [POST /api/incy/decrypt](#post-apiincydecrypt--расшифровка-incycrypt1)
- [Парсер](#-парсер)
- [Безопасность](#-безопасность)
- [Лимиты и таймауты](#-лимиты-и-таймауты)
- [FAQ](#-faq)

---

## 🎯 Что это

`kfwl.1337.cx` — однофайловый веб-конвертер для прокси/VPN-инфраструктуры. Решает четыре типовых боли одним сайтом:

1. **Подписки не открываются в браузере** (CORS, невозможно выставить `User-Agent`, `x-hwid`) → серверный прокси `/api/fetch`.
2. **Happ Crypt-ссылки нечитаемы** (`happ://crypt5/...`) → две независимые расшифровки: онлайн (`happy-decoder.cc`) и локальная через Rust-бинарь на сервере.
3. **Нужно быстро перепарсить/переименовать/пересобрать линки** → парсер с экспортом в `Selector` / `URLTest` / Base64 / Subscription JSON.
4. **YouTube без рекламы и без VPN на клиенте** → `/api/dl` с серверным `yt-dlp` + `ffmpeg` (файл не хранится).

Всё это под одним HTTPS-доменом, без аккаунтов, без БД, без логов содержимого.

---

## 🧰 Фичи

Единое SPA с навигацией по табам:

| Таб | Что делает |
|-----|------------|
| **Converter** | Берёт ссылку на подписку или Crypt-ссылку, разворачивает в плоский список нод. Поддерживает Xray-JSON, sing-box, Base64, plain-list, YAML (Clash/Mihomo). |
| **Rename** | Массовое переименование нод: regex, prefix/suffix, нумерация, добавление флагов через GeoIP, удаление эмодзи. |
| **Parser** | Тянет подписку с любого сервера, расшифровывает обёртки, достаёт ключи. Три режима: **🤖 Авто** (сервер сам подбирает клиент), **⚙️ Свой UA**, **∅ Без UA**. Поддерживает Happ / V2RayTUN / V2RayNG / Clash Meta / sing-box / NekoBox / Throne / INCY. Подробнее → [Парсер](#-парсер). |
| **QR** | Генерация QR-кодов из любой строки/ссылки, экспорт PNG. |
| **Decode** | URL-decode/encode (percent-encoding), Base64 туда-обратно, JSON pretty-printer. |
| **YTDL** | YouTube → `mp4`. Качество: `max` (до 4K, VP9/AV1) или `1080/720/480/360` (H.264 для совместимости). |
| **Themes** | Палитра: solid, gradient mix, custom contrast pair, light-glass toggle, 9+ пресетов. Сохраняется в `localStorage`. |
| **My IP** | Показывает текущий IP клиента, кнопка-глаз для скрытия. |

### Источник Crypt (на табах Parser/Converter)

Радиогруппа **«Источник»** контролирует, как расшифровываются `happ://crypt5/...`:

- **Авто** — сначала онлайн (`happy-decoder.cc`), при фейле — локальный Rust-бинарь.
- **API** — только онлайн (`happy-decoder.cc`).
- **Local** — только Rust (`/api/incy/happ-local`). Работает без зависимости от стороннего сервиса.

---

## 🌐 Публичный API

Все ответы CORS-открытые (`Access-Control-Allow-Origin: *`). Все ответы `Cache-Control: no-store`. Base URL: `https://kfwl.1337.cx`.

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
| `selftest` | `1` | Проверка живости — вернёт `{ok:true, build, node, hasGlobalFetch}`. |

```bash
# Простое получение подписки от Happ
curl -sS "https://kfwl.1337.cx/api/fetch?url=https%3A%2F%2Fsub.example%2Fxxx&ua=Happ%2F3.24.0"

# С hwid
curl -sS "https://kfwl.1337.cx/api/fetch?url=...&ua=v2rayNG%2F1.10.6&hwid=A1B2C3"

# Selftest
curl -sS "https://kfwl.1337.cx/api/fetch?selftest=1"
```

**TLS-fallback**: если у upstream битый/самоподписанный сертификат и штатный `fetch()` падает, прокси автоматически переключается на сырой HTTPS-клиент без проверки цепочки. Это позволяет тянуть подписки с IP-хостов без валидных SAN.

**Ошибки:**
- `400` — `missing url`, `bad url`, `only http/https allowed`
- `403` — `forbidden host` (приватный диапазон)
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
- `500` — `yt-dlp не запустился`, `Не удалось скачать видео` (с последними 1000 байт `stderr`), `Файл не найден`

---

### `POST /api/incy/happ-local` — офлайн-расшифровка Crypt5

Использует локальный Rust-бинарь [`happ-decrypt-universal`](https://github.com/amurcanov/happ-decrypt-universal) для расшифровки `happ://crypt5/...` без обращения к сторонним сервисам.

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

**Когда полезно:** API `happy-decoder.cc` лежит, rate-limit, или хочется обработать батч без зависимости от стороннего сервиса.

---

### `GET /api/incy/health`

Проверка живости sidecar.

```bash
curl -sS https://kfwl.1337.cx/api/incy/health
# {"ok":true,"binary":"happ-decrypt","version":"1.0.0"}
```

---

### `POST /api/incy/parser/auto` — автопарсер (серверный)

Принимает URL подписки, сам подбирает клиент-эмулятор (UA + HWID + заголовки), снимает HTML-обёртки, раскрывает Happ-JSON-форматы, выдаёт чистый текст с ключами. Используется фронтом в режиме **🤖 Авто**.

**Body:** `{"url": "<URL подписки>"}`

```bash
curl -sS -X POST https://kfwl.1337.cx/api/incy/parser/auto \
  -H 'Content-Type: application/json' \
  -d '{"url":"https://sub.example.com/abc"}' | python3 -m json.tool
```

**Ответ (успех):**

```json
{
  "ok": true,
  "content": "vless://...\nvless://...\n",
  "suggested_hwid": "A1B2C3D4E5F6...",
  "hwid_format": "alnum16",
  "log": [
    {"t":"12:34:56.001","msg":"→ HEAD https://sub.example.com/abc"},
    {"t":"12:34:56.220","msg":"detected: Happ marker"},
    {"t":"12:34:56.221","msg":"→ GET ua=Happ/3.24.0/Android x-hwid=..."},
    {"t":"12:34:56.701","msg":"✓ 12 ключей"}
  ]
}
```

**Особые случаи:**
- `remnawave_stats: true` — попалась страница статистики Remnawave (там нет подписки, только usage). Нужна ссылка-подписка из кнопки «Подписка» приложения.
- `ok: false` + `error: "..."` — все попытки исчерпаны, в `log` — пошаговая диагностика.

---

### `POST /api/incy/decrypt` — расшифровка `incy://crypt1/...`

Локальный сайдкар расшифровывает INCY-крипт (AES-256-GCM) без обращения к внешним сервисам.

```bash
curl -sS -X POST https://kfwl.1337.cx/api/incy/decrypt \
  -H 'Content-Type: application/json' \
  -d '{"link":"incy://crypt1/..."}'
# {"ok":true,"url":"https://sub.example.com/abc"}
```

Если расшифрованное значение — URL подписки, фронт сам подтянет содержимое через `/api/fetch`. Если это уже plain-список ключей — возвращается как есть.

---

## 🔬 Парсер

Парсер — самая толстая часть UI. Его задача: из любой ссылки/строки достать список нод, даже если сервер подписки прячет её за HTML/JSON/UA-проверками/шифрованием.

### Что он умеет принимать

- **HTTPS-подписки** любых провайдеров (Marzban, Marzneshin, 3x-ui, x-ui, Remnawave, custom)
- **Happ Crypt5**: `happ://crypt5/...`
- **INCY Crypt1**: `incy://crypt1/...` (AES-256-GCM)
- **Base64-блобы** (с/без padding, urlsafe/standard)
- **YAML-подписки** Clash / Mihomo (`proxies:`/`Proxy:`)
- **Plain-text** список линков: `vless` / `vmess` / `ss` / `trojan` / `hy2` / `hysteria2` / `tuic` / `wireguard` / `naive`
- **HTML-странички**, в которых подписка спрятана за `<pre>`, `<br>` или deeplink-кнопкой
- **JSON-обёртки** Happ: поля `sub`, `configs`, `servers`, `data`, `proxies`

### Три режима

| Режим | Кнопка | Что делает |
|-------|--------|------------|
| **🤖 Авто** | `data-pm="auto"` | Отправляет URL на `/api/incy/parser/auto` — сервер сам подбирает UA, HWID, заголовки, разбирает HTML, раскрывает JSON. |
| **⚙️ Свой UA** | `data-pm="custom"` | Ручной User-Agent, HWID (или галка «генерировать случайно»), доп. заголовки. Запрос идёт через `/api/fetch`. |
| **∅ Без UA** | `data-pm="noua"` | Запрос без `User-Agent` вообще. Для серверов, которые запрещают браузерные UA. |

### Авто-режим: как именно работает

Под капотом **🤖 Авто** — двухступенчатая цепь: сервер пытается сделать все эвристики, фронт добивает оставшиеся кейсы.

#### 🔹 Шаг 1. Сервер (`POST /api/incy/parser/auto`)

- HEAD + GET на исходный URL, разбор `Content-Type` и тела.
- Поиск маркера клиента в HTML (`Happ`, `V2RayTUN`, `V2RayNG`, `Clash Meta`, `sing-box`, `NekoBox`, `Throne`, `INCY`).
- При находке — переход на UA этого клиента и повтор запроса с правильным `x-hwid` + специфичными заголовками.
- Извлечение deeplink-URL из HTML (вида `happ://add-sub/...` или `clash://install-config?url=...`) и парсинг именно из него.
- Ответ: `{ok, content, suggested_hwid, hwid_format, log}` или `{ok:false, error, log}`.

#### 🔹 Шаг 2. Фронт-постобработка (9 слоёв)

Даже после «успеха» сервера фронт прогоняет ответ через доп. слои:

1. **`unwrapHtmlSub`** — снимает HTML-обёртку (`<pre>`, `<br>`, entities) и достаёт плоский текст.
2. **`detectAppFE`** — ищет маркер клиента уже на клиенте (если сервер не справился) → берёт UA+HWID этого клиента → ре-фетчит через `/api/fetch`.
3. **UA-fallback chain** — если маркеров нет, последовательно пробует: **Happ → INCY → V2RayTUN → Clash Meta**. Для каждого: подкидывает соответствующий UA, HWID нужного формата. Для INCY — ещё весь букет `x-app-version`, `x-client`, `x-device-os`, `x-ver-os`, `x-device-model`. Если ответ HTML — следующий, если JSON — раскрывает поля, если plain — проверяет на ключи.
4. **`resolveIncy`** — если в тексте найден `incy://crypt1/...`, дёргает `/api/incy/decrypt`.
5. **`resolveHapp`** — если найден `happ://crypt5/...`, расшифровывает через выбранный источник (**Авто / API / Local**). Если результат — URL, рекурсивно тянет его как подписку.
6. **`decodeSub`** — если строка без `://`, пробует base64-decode.
7. **`autoParse`** — основной парсер: вытаскивает все валидные ноды, валидирует UUID/порты/обязательные поля.
8. **Regex-fallback** — если `autoParse` пустой, прогоняет регэксп `(?:vless|vmess|trojan|ss|hy2|hysteria2|tuic|wireguard):\/\/\S+` по всему тексту.
9. **Base64-fallback** — последняя попытка: декодит всю строку как base64 и ещё раз парсит.

Пройдя все 9 слоёв вхолостую — ошибка с предложением открыть **🐛 логи**.

### UA + HWID по клиентам

Когда парсер «эмулирует» клиент, он использует следующие комбинации:

| Клиент | User-Agent | HWID-формат | Доп. заголовки |
|--------|-----------|-------------|----------------|
| **Happ** | `Happ/3.24.0/Android` | 16-знач. alnum | `x-hwid` |
| **INCY** | `INCY/3.2.2/android Dalvik/2.1.0` | UUID | `x-hwid`, `x-app-version: 3.2.2`, `x-client: INCY`, `x-device-os: Android`, `x-ver-os: 13`, `x-device-model: Pixel 7` |
| **V2RayTUN** | `v2raytun/5.23.74/android` | UUID | — |
| **V2RayNG** | `v2rayNG/1.10.7` | 16-знач. hex | — |
| **Clash Meta** | `ClashMetaForAndroid/2.11.7.Meta` | UUID | — |
| **sing-box** | `sing-box/1.11.0` | UUID | — |
| **NekoBox** | `NekoBox/1.3.7` | 16-знач. hex | — |
| **Throne** | `Throne/1.0.0` | UUID | — |

В режиме **⚙️ Свой UA** в выпадашке есть заранее заготовленные строки (включая Happ-варианты с уже вшитым HWID — отмечены `✓`).

### Логи парсера (🐛)

Кнопка-баг под полем ввода открывает чёрную панель с пошаговым логом:

```
12:34:56.001  → POST /api/incy/parser/auto · https://sub.example.com/abc
12:34:56.220  получено 14821 симв., preview: <!doctype html><html lang="ru"><head>...
12:34:56.221  ⚠ маркеры не найдены — пробую UA-фолбэк (Happ→INCY→V2RayTUN→Clash)
12:34:56.222  → UA=Happ/3.24.0/Android, HWID=A1B2C3D4E5F6G7H8 (alnum16)
12:34:56.701  ← happ: 1842 симв., preview: vless://uuid@host:443?...
12:34:56.702  ✓ UA=happ дал валидную подписку
12:34:56.703  после decodeSub+resolveHapp: 1842 симв., preview: vless://...
12:34:56.710  ✓ парсинг ОК: 12 ключей
```

Это бесценно когда подписка «не парсится» — сразу видно, на каком слое отвалилось.

### Что делать, если не парсится

1. Открой 🐛 логи и посмотри последнюю строку.
2. «маркеры не найдены» + «всё ещё HTML» → сервер отдаёт SPA, нужен правильный UA. Переключись на **⚙️ Свой UA**, выбери `Happ/3.24.0/Android` или `v2rayNG/1.10.7`, включи галку HWID auto.
3. «forbidden host» → ссылка указывает на приватный IP. Используй внешний домен.
4. «remnawave_stats» → в приложении возьми настоящий subscription URL вида `https://.../sub/<token>`, а не страницу статы.
5. Если HTML с deeplink-кнопкой и без маркеров — скопируй сам deeplink (`happ://add-sub/...`) и вставь его — парсер раскроет.

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

`/api/fetch` принимает только `http://` и `https://`. Любые `file://`, `gopher://`, `dict://` отбрасываются как `400 only http/https allowed`.

### Нет персистентного хранения

- Подписки проксируются стримом, не пишутся на диск.
- YouTube-файлы пишутся в `mkdtempSync()` и удаляются в `finally`/`on("close")`/`on("error")`.
- Сервер не логирует тела ответов. В журнале только статусы и stderr-хвосты `yt-dlp` (последние 6 KB).

---

## ⏱ Лимиты и таймауты

| Лимит | Значение |
|---|---|
| `/api/fetch` upstream timeout | 20 сек |
| `/api/fetch` редиректов | до 5 hop |
| `/api/incy/*` request body | 4 MB |
| `/api/dl` stderr buffer | 6 KB (rolling) |
| `/api/dl` cookie `dl_<t>` | TTL 120 сек |
| `t` (download token) | до 48 alnum символов |
| ALTCHA `maxnumber` | 1 000 000 |

---

## ❓ FAQ

**Q: Где исходники?**
A: Сервер и фронт — проприетарные и в открытый доступ не выкладываются. Репозиторий — публичная документация API.

**Q: Сохраняются ли мои подписки на сервере?**
A: Нет. `/api/fetch` стримит ответ от upstream к клиенту без записи на диск. Сервер не пишет ни тело запроса, ни тело ответа.

**Q: Сохраняются ли видео с YouTube?**
A: Нет. Файл создаётся во временной директории через `mkdtempSync`, стримится клиенту, **синхронно удаляется** при `end`, `close`, `error` или дисконнекте клиента.

**Q: Можно ли использовать API напрямую без UI?**
A: Да, все эндпоинты публичные. `/api/dl` требует ALTCHA — придётся реализовать решение PoW на клиенте.

**Q: Парсер ничего не находит, хотя в браузере подписка открывается.**
A: Скорее всего сервер подписки требует определённый UA. Переключи парсер в **⚙️ Свой UA**, выбери `Happ/3.24.0/Android` или `v2rayNG/1.10.7`. Включи галку **«Генерировать HWID случайно»**. Не помогло — открой 🐛 логи и посмотри, что отдаётся в `preview`.

**Q: В чём разница между Авто и Свой UA?**
A: **🤖 Авто** идёт через серверный сайдкар (`/api/incy/parser/auto`), который HEAD-ит URL, читает HTML, ищет маркеры, перебирает UA-цепочку. **⚙️ Свой UA** — прямой `/api/fetch` с указанными тобой параметрами, без эвристик. Авто работает в ~95% случаев, Свой UA — когда нужна тонкая настройка или приложения нет в списке.

**Q: Что если `happy-decoder.cc` недоступен?**
A: Переключи источник Crypt на **Local** в UI или используй `/api/incy/happ-local` напрямую — он расшифровывает через локальный Rust-бинарь без обращения к стороннему сервису.

**Q: Почему я получаю `403 forbidden host`?**
A: SSRF-guard. URL подписки указывает на приватный диапазон (`127.x`, `10.x`, `192.168.x`, `172.16-31.x`, `localhost`, `*.local`). Подписки публичных провайдеров под него не попадают.

**Q: Почему капча на скачивании YouTube?**
A: `yt-dlp` — тяжёлая операция (CPU + сеть + диск). PoW отсекает ботнет до того, как сервер начнёт работу. Норм пользователь решает её за 1-3 сек��нды.

**Q: `max` качество не воспроизводится — что делать?**
A: `max` отдаёт лучший доступный поток (часто VP9/AV1 в `webm`). Старые телевизоры и плееры на iOS<14 могут не уметь. Перекачай в `1080` — это H.264 в `mp4`, играет везде.

**Q: Поддерживаются ли VK, TikTok, Twitter?**
A: Нет, только YouTube (`youtube.com`, `youtu.be`, `youtube-nocookie.com`).

**Q: Можно ли поставить свой self-signed cert на upstream подписки?**
A: Да. `/api/fetch` при падении штатного `fetch()` переключается на запасной путь без проверки цепочки сертификатов.

---

<div align="center">

*🦆 made with duck-driven development*

</div>
