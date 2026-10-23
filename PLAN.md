# План: MCP-сервер `tourvisor-mcp` (поиск туров на tourvisor.ru через Playwright)

> Статус: план, не реализация. Файл-план: `PLAN.md` в корне проекта.
> Дата: 23.10.2026

---

## 1. Цель и объём

Создать MCP-сервер (`stdio`-транспорт), который:

1. Через **Playwright (detached-режим)** собирает с сайта `https://tourvisor.ru/` значения **базовых справочников**:
   города вылета, страны, курорты (regions), отели, туроператоры, типы питания (meals), звёзды;
2. Предоставляет эти справочники агенту (LLM), чтобы тот **уточнил у пользователя** параметры поиска;
3. На сайте ищет: **туры** (`/search.php`), **отели** (`/poisk-otelej`), **горящие туры** (`/goryashchie-tury`), **перелёты** (расписание чартеров + перелёты внутри карточек туров);
4. **Парсит выдачу** (асинхронная загрузка результатов с прогрессом — ждать до 100%) и отдаёт структурированные данные + **изображения** отелей/туров;
5. Кеширует справочники и результаты поиска в **SQLite** (единый файл БД, Blob-кеш изображений в БД).

Явно вне объёма: бронирование/оплата, авторизация в ЛК Tourvisor, XML-шлюзы (`tourvisor.ru/xml/*`) — единственный источник данных здесь публичный сайт; XML-шлюзы позже можно добавить опциональным адаптером, если появятся оплаченные креды.

---

## 2. Архитектура

```
┌────────────┐  stdio (MCP)  ┌─────────────────┐   WS/IPC   ┌──────────────────────┐
│ LLM-клиент │◄─────────────►│  tourvisor-mcp  │◄──────────►│ browser-daemon (det.)│
│ (opencode) │               │  Node 20 + TS   │            │ Playwright chromium  │
└────────────┘               │  tools/cache    │            │ persistent profile   │
                             └───────┬─────────┘            └──────────┬───────────┘
                                     │                                 │
                              ┌──────▼──────┐                  HTTPS к сайту (UI)
                              │ SQLite      │◄────────── JSON-парсеры (src/parsers)
                              │ data/tv.db  │
                              └─────────────┘
```

### 2.1 Режим detached

- Отдельный **daemon-процесс** (`src/daemon/index.ts`): `chromium.launchServer({ headless, port, wsEndpoint })`, запускается `spawn({ detached: true, stdio: 'ignore' })` + `unref()`.
- Endpoint (ws://127.0.0.1:<port>/) хранится в `data/browser.json` (+ pid, health-check `/json/version`).
- MCP-сервер подключается лениво: `playwright.connect(wsEndpoint)`; daemon авто-стартует при первом обращении, если не запущен.
- `userDataDir` = `data/profile` → сохраняются cookies/сессии между рестартами сервера и **разделяются между всеми клиентами** MCP.
- Ручной запуск/стоп: `pnpm daemon start|stop|status` (CLI-обёртка).
- Fallback: `TV_BROWSER_MODE=inprocess` — обычный in-process браузер (для тестов/CI), без daemon.

### 2.2 Стек

- Node ≥ 20, TypeScript (strict), `@modelcontextprotocol/sdk` (`McpServer`, stdio), `zod` для схем аргументов.
- `playwright-core` + chromium (в devDependencies — полный `playwright` для `npx playwright install chromium`).
- `better-sqlite3` (синхронный, без гонок внутри одного процесса; WAL).
- Форматирование/линт: prettier + eslint (минимально).

---

## 3. Источники данных на сайте (Z0-разведка)

Перед кодированием — **M0: reconnaissance** через DevTools + этот Playwright-MCP (уже доступен в среде):

| Данные | Кандидаты источника | Метод |
| --- | --- | --- |
| Города вылета / страны | JS-конфиг поискового модуля на главной и `/search.php` (options селекта, либо embed-JS `tcInit({...departures, countries})`, либо JSON-файл модуля поиска) | Слушать network (HAR), вытащить XHR/конфиг |
| Курорты / subregions | опции фильтра выдачи после выбора страны; страницы каталога страны | Селекторы + перехват XHR |
| Отели | `poisk-otelej`; фильтры выдачи; каталог отелей страны/курорта | Парсинг карточек/селектов |
| Операторы / питание / звёзды | панели фильтров выдачи туров | Селекторы фильтров |
| Выдача туров | `search.php` + асинхронная подгрузка (прогресс-бар, статусы операторов); страницы результатов | Ожидание `[data-*]/класса лоадера` до пропадания; парсинг карточек |
| Горящие туры | `goryashchie-tury` (карточки со скидками) | Парсинг карточек |
| Перелёты | блоки перелёта в карточке тура (рейс/авиакомпания/аэропорты/времена); `raspisanie-charterov` | Парсинг таблицы/карточек |
| Изображения | `img.tourvisor.ru/...` в карточках и галереях отелей/tурa | `srcSet`, lazy-load (скролл/`data-src`) |

Результат M0 фиксируется как `docs/recon.md`: конечные точки XHR, имена полей, устойчивые селекторы (по возможности `data-testid`/семантические классы), статусы завершения поиска.

**Важно**: сайт — SPA-подобный поиск с прогрессом; парсер должен уметь ждать (`progress=100`/исчезновение лоадера) и снимать состояние DOM, а не «первый кадр».

---

## 4. Схема SQLite (`data/tv.db`)

```sql
-- Справочники: ключ = тип(+:аргументы), payload = JSON-массив элементов
CREATE TABLE dictionaries (
  key         TEXT PRIMARY KEY,     -- 'departures' | 'countries:2' | 'resorts:84' | ...
  payload     TEXT NOT NULL,        -- JSON
  fetched_at  INTEGER NOT NULL,     -- unixtime
  ttl_seconds INTEGER NOT NULL      -- geo: 7-30 дн; отели: 7 дн; meals/stars: 30 дн
);

-- Результаты поиска: нормализованный запрос → выдача
CREATE TABLE search_cache (
  key         TEXT PRIMARY KEY,     -- sha1(type + canonical_params_json)
  type        TEXT NOT NULL,        -- 'tours' | 'hotels' | 'hottours' | 'flights' | 'tour_detail' | 'hotel_detail'
  params      TEXT NOT NULL,
  payload     TEXT NOT NULL,        -- JSON: массив карточек (+meta: статус, minprice, timestamp)
  fetched_at  INTEGER NOT NULL,
  ttl_seconds INTEGER NOT NULL      -- выдача: 10-30 мин; detail-страницы: 1-24 ч
);

-- Изображения (Blob-кеш; URL — первичный ключ)
CREATE TABLE images (
  url        TEXT PRIMARY KEY,
  bytes      BLOB,                  -- NULL, пока не скачан (тогда только метаданные)
  mime       TEXT, width INT, height INT,
  etag       TEXT, fetched_at INTEGER,
  refs       TEXT                   -- JSON: какие key используют
);

-- Журнал поисковых сессий (диагностика/снапшот DOM для отладки парсеров)
CREATE TABLE runs (
  id INTEGER PRIMARY KEY, type TEXT, params TEXT,
  started_at INTEGER, finished_at INTEGER, status TEXT, error TEXT, snapshot TEXT
);

PRAGMA journal_mode=WAL;
```

Политика TTL: свежесть >ву ключу; вместо перезаписи — `revalidate` (фоновое обновление, старое отдать). Инвалидация: `cache_clear(scope?)`.

---

## 5. MCP-интерфейс

### 5.1 Tools (RFC-черновик)

**Справочники** (все с кешем; `forceRefresh = true` → обход кеша):

| Tool | Назначение |
| --- | --- |
| `tv_departures()` | города вылета (id, name) |
| `tv_countries(departureId?)` | страны |
| `tv_resorts(countryId)` | курорты страны (+subregions) |
| `tv_operators(countryId?)` | туроператоры |
| `tv_meals()` / `tv_stars()` | питание / звёздность |
| `tv_hotels(countryId, resortIds?, page?, onpage?)` | отели (paged) |

**Поиск:**

| Tool | Назначение |
| --- | --- |
| `tv_search_tours({ departureId, countryId, dateFrom?, dateTo?, nightsMin?, nightsMax?, adults?, children?(возрасты), stars?, meal?, resortIds?, hotelIds?, operatorIds?, priceMin?, priceMax?, hotOnly?, currency? })` | создаёт сессию поиска; ответ: `sessionId`, статус, первая страница |
| `tv_search_status(sessionId)` | прогресс (0–100), статусы операторов, hotelsfound/minprice |
| `tv_search_results(sessionId, page?, onpage?, sort?/filters?)` | страница выдачи (отели с турами, minprice по отелю, изображения) |
| `tv_search_hotels({ countryId, resortIds?, stars?, rating?, price? ... , page? })` | отдельный поток `/poisk-otelej` |
| `tv_hot_tours({ departureId?, countryId?, page? })` | горящие туры (карточки со скидкой) |
| `tv_charter_flights({ departureId, countryId?, dateFrom?, dateTo? })` | расписание/перелёты чартеров |
| `tv_tour_details(tourRef)` | детальная тура + перелёты + photos |
| `tv_hotel_details(hotelCode)` | описание отеля, удобства, галерея |
| `tv_images(refKey)` | список URL изображений варианта из кеша |
| `tv_fetch_image(url, { maxWidth? })` | скачать (из кеша or fetch) → вернуть как image-content (base64) |
| `tv_cache_clear(scope?)` / `tv_cache_stats()` | управление кешем |

### 5.2 Prompts / Resources

- **Prompt `plan_tour_search`**: инструкция агенту — сначала получить справочники, затем спросить у пользователя недостающее (город вылета, страна/курорт, даты±ночей, состав туристов, бюджет), затем вызвать `tv_search_tours`; далее с 5–10 сек поллингом `tv_search_status` до 100% и выдача топ-вариантов с ценами/фото.
- Resource `tourvisor://docs/how-it-works` — краткая справка для модели.

### 5.3 Ответы

- Единый envelope: `{ ok, data, meta: { sessionId?, page?, total?, cached: bool, fetchedAt, stale? } }`.
- Списки: `{ items: [...], page, hasMore, nextKey? }`.
- Ошибки: человеческое `message` + `code` (`selectors_changed`, `timeout`, `blocked`, `no_results`).

---

## 6. Потоки данных

### 6.1 Справочники (кеш-first)
`tool call → cache lookup (TTL ok? → return) → playwright page.goto → ожидание узла → extract → upsert dictionaries → return` (rate-limit 0.5–1.5 s между запросами к сайту).

### 6.2 Поиск туров
1. Валидация аргументов (zod) — требуются `departureId` + `countryId` (иначе ошибка «уточните у пользователя»).
2. Нормализация → `key` → если кеш живой (< TTL) → вернуть из кеша.
3. Открыть `search.php`, заполнить форму (или прямые query-параметры URL, если найдутся в M0), submit.
4. Дождаться завершения: исчезновение лоадера ИЛИ `progress=100` (таймаут 120–180 s, съём промежуточных статусов).
5. Парс карточек → {hotel, stars, rating, resort, meal, tour {price, operator, dates, nights, room, flight}, image urls}.
6. Upsert `search_cache` + регистрации изображений (только URL; скачивание — лениво по `tv_fetch_image`).

### 6.3 Горящие / отели / перелёты — те же шаги с собственными селекторами/страницей.

### 6.4 Изображения
Ленивая загрузка: `tv_fetch_image` → `images.bytes` (проверять etag; ограничить размер, конвертация не требуется). Отдавать как MCP image-контент (`base64`) или путь+mime.

---

## 7. Верификация парсеров

- **Fixtures**: в кеше хранить `snapshot` DOM (HTML выдачи) в `runs` + файлы в `test/fixtures/*.html` — unit-тесты парсеров без сайта.
- **Smoke-скрипт** `pnpm smoke` : daemon → словари → поиск Москва→Турция → печать топ-3.
- **Канарейки селекторов**: тест, который рендерит сохранённую выдачу и делает простые проверки (кол-во карточек > 0, у первой есть price и image) — быстрое обнаружение «сайт поменяла разметку».

---

## 8. Надёжность / лимиты

- Rate-limit к сайту (token-bucket ~1 req/2 s по умолчанию, jitter), ретраи ×2 с backoff.
- Таймауты: навигация 30 s, ожидание выдачи 180 s.
- Обработка: капча/бан (`blocked` → вернуть пользователю сообщение, ничего не ломать), офлайн — выдача stale-кеша с пометкой `stale: true`.
- Порядочность: один общий браузер (не ферма), учитывать `robots.txt`-дух; не мультитредить десятки параллельных сессий.
- Логи: `logs/mcp.log` (stdout MCP-сервера не замусоривать — всё в файл).

---

## 9. Структура проекта

```
tourvisor-mcp/
  PLAN.md, README.md, package.json, tsconfig.json
  src/
    index.ts            # MCP entry (stdio), регистрация tools/prompts
    daemon/             # detached playwright launcher + client.connect
    site/               # page-объекты: searchForm, resultsPage, hotelPage, ...
    parsers/            # dictionaries, tours, hotels, hottours, flights, images (чистые функции)
    cache/              # sqlite schema, getters/upserts, TTL, canonical key (sha1)
    tools/              # MCP tool-обёртки + zod-схемы
    config.ts           # TTL, лимиты, пути (env: TV_DATA_DIR, TV_TTL_*, TV_BROWSER_MODE)
  data/                 # tv.db, browser.json, profile/, logs/ (в .gitignore)
  test/                 # unit (fixtures), integration (smoke)
```

---

## 10. Этапы работ (milestones)

- **M0 — разведка сайта (0.5–1 день)**: HAR всех нужных страниц/форм; список XHR-источников справочников; устойчивые селекторы; статус завершения поиска. Выхлоп: `docs/recon.md` + черновые JSON-примеры.
- **M1 — скелет (0.5 день)**: package/tsconfig, MCP «hello tool», sqlite schema + миграции, daemon + playwright-клиент, `.gitignore`.
- **M2 — справочники (1 день)**: parsers + tools `tv_departures/countries/resorts/meals/stars/operators/hotels`, кеш с TTL, `forceRefresh`.
- **M3 — поиск туров (1.5–2 дня)**: форма/URL-параметры, ожидание прогресса, парс карточек, `tv_search_tours/status/results` + кеш; снапшоты в `runs`.
- **M4 — остальные разделы (1 день)**: `tv_search_hotels`, `tv_hot_tours`, `tv_charter_flights`, `tv_tour_details`, `tv_hotel_details`.
- **M5 — изображения (0.5 дня)**: `tv_images`, `tv_fetch_image` (blob in sqlite, etag, лимиты размера).
- **M6 — UX-обвязка (0.5 дня)**: prompt `plan_tour_search`, resources, ошибки/ретраи/логи, `tv_cache_*`.
- **M7 — качество и доки (1 день)**: unit по fixtures, smoke, README (что это, установка, конфиг MCP-клиента: пример JSON `mcpServers`), финальная проверка в opencode.</br>
  **Итого ~ 6–8 дней** ориентировочно (сольно).

---

## 11. Риски и допущения

| Риск | Митигация |
| --- | --- |
| Без авторизации сайт всё же может требовать агентские токены для полной выдачи | Проверить в M0; если публичная выдача урезана — отказаться от сайта в пользу XML-шлюза (адаптер уже спроектирован за интерфейсом `parsers`) |
| Смена разметки/эндпоинтов | fixtures-тесты + одна точка селекторов (`site/`) + понятный код ошибки `selectors_changed` |
| Долгий асинхронный поиск, MCP-вызов «зависает» | Двухфазность: сразу отвечатьsessionId + `tv_search_status` поллинг |
| Playwright тяжёлый для окружения пользователя | `TV_BROWSER_MODE=inprocess`, кеширование, минимум переиспользуемых вкладок |
| Правовые | Только публичные страницы, выгодные интервалы, без обхода авторизаций/капч |

---

## 12. Критерии приёмки

1. `pnpm daemon start` — браузер живёт независимо от MCP-клиента; рестарт клиента не роняет сессию/куки.
2. Через MCP доступны все справочники; повторный запрос идёт из SQLite (проверка по `meta.cached=true`).
3. `tv_search_tours(Москва→Турция, 7 ночей)` возвращает карточки туров (цена, отель, звёзды, питание, вылет, туроператор) и ≥1 изображение на карточку (URL; `tv_fetch_image` отдаёт бинарник).
4. Горящие туры, отели, перелёты — те же гарантии.
5. Кеш: TTL-освежение реализовано, `tv_cache_clear` работает; БД — один файл `data/tv.db`.
6. Парсеры покрыты fixtures-тестами (unit без сети — «зелёные» в CI-модели).
