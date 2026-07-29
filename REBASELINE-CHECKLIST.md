# Jobich API — чек-лист rebaseline (2026-07-29)

Источники изменений:
- `Jobich/jobich-api-contract.pdf` — текущий контракт API (deploy 2026-07-28)
- `Jobich/jobich-subscribe-06-rebaseline.pdf` — детальный ре-бейзлайн для папки 06 (`/api/subscribe`)

Причина: QA Round 1 (коммит `258e326`) усилил валидацию входа и добавил method-guards. Часть
negative/edge тестов писалась до этого коммита и ожидала устаревшее поведение.

Как гонять: открой коллекцию в Postman → выбери environment **Jobich - Prod** → запускай перечисленные
запросы либо вручную, либо через Collection Runner по конкретной папке.

---

## Collection: `Jobich.ch API - Negative & Edge Cases`

### 📁 06_Subscription /api/subscribe - edge cases

| # | Запрос | Было | Стало | Проверить |
|---|--------|------|-------|-----------|
| 1 | **Missing subscription_type** | ожидал `400` | ожидает `200 {ok:true}` | тип по умолчанию `query`, `query` передан → должен пройти успешно |
| 2 | **subscription_type=keyword but query missing** | `[400,200]` (нестрого) | строго `400`, error содержит `Invalid subscription_type` | падает именно на невалидном типе, а не на отсутствии `query` |
| 3 | **subscription_type=cv but cv_embedding missing** | `[400,200]` (нестрого) | строго `400`, error содержит `CV embedding` | |
| 4 | **Duplicate subscribe (first call)** | body слал `subscription_type: "keyword"` (невалидный тип!) | body теперь `"query"`, ответ строго `200 {ok:true}` | раньше тест был сломан — реально получал бы 400 |
| 5 | **Duplicate subscribe - second identical call** | допускал `[200,201,409]` | body `"query"`, строго `200 {ok:true}` | идемпотентность через `ON CONFLICT DO UPDATE`, 409 никогда не бывает |

### 📁 03_Public API /api/translate - edge cases

| # | Запрос | Было | Стало | Проверить |
|---|--------|------|-------|-----------|
| 6 | **Missing to param (defaults to en)** — было *«Missing required to param»* | ожидал `400` | ожидает `200` + поле `translated` | `to` теперь опционален, дефолт `en` |

### 📁 05_CV Match /api/match/analyze - edge cases

| # | Запрос | Было | Стало | Проверить |
|---|--------|------|-------|-----------|
| 7 | **Missing jobTitle only** | `[400,200]` (нестрого) | строго `400`, error содержит `jobTitle` | `cvText`+`jobTitle` обязательны вместе |
| 8 | **Empty cvText** | `[400,200]` (нестрого) | строго `400` | пустой `cvText` = слишком короткий |

### 📁 09_Partner API /jobs/changes - edge cases

| # | Запрос | Было | Стало | Проверить |
|---|--------|------|-------|-----------|
| 9 | **since date in the future** | допускал `200` с пустым списком | строго `400`, error содержит `future` | контракт: `"'since' cannot be in the future."` |

### 📁 11_HTTP method mismatches (405 checks)

| # | Запрос | Было | Стало | Проверить |
|---|--------|------|-------|-----------|
| 10 | **POST to /api/jobs without a recognized action** — было *«POST to GET-only /api/jobs»* | строго `405` | `[400,405]` | `/api/jobs` на самом деле принимает POST для `?action=report_vacancy`/`?action=feedback`, «GET-only» — неверная предпосылка |

---

## Collection: `Jobich.ch API - Happy Path`

### 📁 04_Subscription API

| # | Запрос | Было | Стало | Проверить |
|---|--------|------|-------|-----------|
| 11 | **12 - Subscribe by keyword** | body слал `subscription_type: "keyword"` | body теперь `"query"` | раньше запрос реально падал бы с 400 вместо успеха |

### 📁 02_Public API

| # | Запрос | Было | Стало | Проверить |
|---|--------|------|-------|-----------|
| 12 | **03 - Search jobs (basic query)** | проверял поле `total` | проверяет `totalCount` | имя поля в ответе — `totalCount`, не `total` |
| 13 | **08 - Translate EN to DE** — было *«...- does not work»* | проверял `j.to === 'de'` | проверяет только наличие `translated` | ответ translate не содержит поля `to`, только `{translated}` |

---

## Не тронуто (проверено, но найдено соответствующим контракту)

`00_Health`, `01_Public API /api/jobs` (кроме отмеченного), `02_Public API /api/salary`,
`04_CV Match /api/match`, `07_Partner API - auth`, `08_Partner /jobs/search`, `10_Partner /jobs/:id`,
`12_Rate limiting` (пустая папка-плейсхолдер) — assertions уже соответствуют контракту.

## Найдено отдельно, не входит в этот rebaseline

- `01_Public API /api/jobs` → **"limit = 0"** — тест называется «limit = 0», но реально шлёт `limit=2` в URL.
  Отдельная задача (не связана с контрактом), вынесена в отдельный таск.
- `Jobich.ch API - Happy Path` → **"13 - Subscribe by CV"** — `cv_embedding` содержит массив из 3 чисел,
  а контракт требует длину 1024. Отдельная задача.
