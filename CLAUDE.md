# StonksSimulator — контекст для ИИ

## Что это

Симулятор торговли акциями на Московской бирже. Пользователь регистрируется, получает стартовый баланс 100 000 ₽ и может покупать/продавать реальные акции MOEX по реальным ценам (но без реальных денег).

## Стек

- **Фронтенд:** vanilla HTML + CSS + JS, без фреймворков, без сборщика
- **Бэкенд:** Supabase (PostgreSQL + Auth + PostgREST)
- **Данные:** MOEX ISS API (Московская биржа, публичный REST)
- **Запуск:** Live Server в VS Code / WebStorm (открывать index.html через сервер, не как file://)

## Структура файлов

```
index.html          — главная: баланс и стоимость портфеля из БД, топ акций с фильтрами
market.html         — все акции TQBR из MOEX; фильтры, поиск, пагинация (20 на стр.)
trade.html          — покупка/продажа: ?ticker=SBER&mode=buy; пишет в portfolio и transactions
portfolio.html      — текущие позиции с P&L, история последних сделок
history.html        — полная история сделок; фильтры buy/sell, поиск по тикеру, пагинация
css/
  common.css        — шапка, модалки, кнопки (общее для всех страниц)
  index.css         — стили главной (карточки, skeleton, фильтры)
  market.css        — стили страницы рынка (grid 4 кол., пагинация)
  trade.css         — стили торговли (табы buy/sell, toast, position-info)
  portfolio.css     — стили портфеля (summary cards, таблицы)
  history.css       — стили истории (summary cards, таблица, фильтры, пагинация)
js/
  components.js     — HTML-компоненты: renderHeader(), renderHomeCard(),
                      renderSkeletonCard(), renderMarketCard()
  moex.js           — MoexClient класс, обёртка над MOEX ISS API
  chart.js          — renderChart(el, prices, options) — SVG-график на чистом JS
  auth.js           — ES-модуль: вход/регистрация/выход, вставляет модалки в DOM,
                      проверяет сессию при загрузке; экспортирует в window:
                        window.supabase       — доступ к клиенту из не-модульных скриптов
                        window.refreshBalance — перезагружает баланс в шапке
                        window.onUserReady    — callback: вызывается когда юзер авторизован
                        window.onUserLogout   — callback: вызывается при выходе
  config.js         — SUPABASE_URL и SUPABASE_ANON_KEY
  index.js          — не используется
  supabaseclient/
    index.js        — экспортирует объект supabase { init, auth, from }
    client.js       — базовый HTTP-клиент, JWT в localStorage, авто-рефреш на 401
    auth.js         — signUp, signIn, signOut, getUser, onAuthStateChange
    db.js           — QueryBuilder для PostgREST: select/insert/update/delete/upsert/
                      eq/order/limit/single/execute
supabase/
  schemas/schema.sql    — источник правды схемы БД
  migrations/           — миграции (создавать через: npx supabase migration new <name>)
  seed.sql              — тестовые данные
docs/
  MOEX_CLIENT.md        — документация по MOEX API
```

## Подключение auth на новой странице

```html
<script src="js/components.js"></script>
<!-- опционально: moex.js, chart.js -->
<script>
  document.getElementById('header-mount').outerHTML = renderHeader('page-key');

  // Реагировать на вход/выход:
  window.onUserReady  = () => { /* пользователь авторизован, грузим данные */ };
  window.onUserLogout = () => { /* пользователь вышел, сбрасываем UI */ };
</script>
<script type="module" src="js/auth.js"></script>
```

`window.onUserReady` вызывается при:
- загрузке страницы, если сессия уже активна
- успешном входе через модалку
- успешной регистрации

## Паттерн торговых операций (trade.html)

Все DB-операции идут через `window.supabase` (доступен после загрузки auth.js):

```js
// Проверить авторизацию
const user = await window.supabase.auth.getUser();
if (!user) { window.openLoginModal(); return; }

// Читать
const profile   = await window.supabase.from('profiles').select('balance').single().execute();
const positions = await window.supabase.from('portfolio').select('*').execute();

// Писать (фильтры через RLS — явный eq нужен для UPDATE/DELETE)
await window.supabase.from('profiles').update({ balance: newBalance }).eq('id', userId).execute();
await window.supabase.from('portfolio').insert({ user_id, ticker, quantity, avg_buy_price }).execute();
await window.supabase.from('portfolio').update({ quantity, avg_buy_price }).eq('user_id', userId).eq('ticker', ticker).execute();
await window.supabase.from('portfolio').delete().eq('user_id', userId).eq('ticker', ticker).execute();
await window.supabase.from('transactions').insert({ user_id, ticker, type: 'buy'|'sell', quantity, price, total_amount }).execute();

// Обновить баланс в шапке после сделки
window.refreshBalance?.();
```

**Важно по QueryBuilder:** `eq()` добавляет фильтр в URL (`?ticker=eq.SBER`). Несколько `.eq()` цепочкой — оба попадают в URL (AND). Для UPDATE и DELETE без фильтра PostgREST может вернуть ошибку — всегда добавляй `.eq('id', userId)` или аналог.

## Supabase

**Настройки проекта:** `js/config.js` — вставить значения из Dashboard → Settings → API.

**Email confirmation:** должно быть **выключено** (Dashboard → Authentication → Providers → Email → Confirm email → off).

**Триггер на создание профиля удалён** — профиль создаётся вручную в `handleRegister()` в `js/auth.js`.

**Схема БД:** применять через Dashboard → SQL Editor. Файл: `supabase/schemas/schema.sql`.

### Таблицы

| Таблица | Описание |
|---|---|
| `profiles` | id (→ auth.users), username, balance (default 100000) |
| `portfolio` | user_id, ticker, quantity, avg_buy_price, updated_at |
| `transactions` | user_id, ticker, type (buy/sell), quantity, price, total_amount, created_at |
| `portfolio_snapshots` | user_id, total_value, recorded_at (не используется в UI) |
| `watchlist` | user_id, ticker (не используется в UI) |

RLS включён на всех таблицах — каждый видит только свои данные.

## MOEX API

```js
const moex = new MoexClient();

// Все акции TQBR с рыночными данными
const { securities, marketdata } = await moex.getAllStocks();
const stocks = moex.merge(securities, marketdata);  // объединить по SECID

// Одна акция
const { security, marketdata } = await moex.getStock('SBER');

// Свечи (interval: 1=1мин, 10=10мин, 60=1час, 24=1день)
const candles = await moex.getStockCandles('SBER', 24, '2026-03-01', '2026-03-28');
// candles[i] = { open, close, high, low, value, volume, begin, end }
```

Rate limit: ~2 req/sec (500 мс между запросами, встроен в `_throttle()`).

### Ключевые поля MOEX

| Поле | Описание |
|---|---|
| `SECID` | тикер (SBER, YNDX, ...) |
| `SHORTNAME` | короткое название |
| `LAST` | последняя цена |
| `LASTTOPREVPRICE` | изменение к предыдущему закрытию в % |
| `VALTODAY` | оборот за день в рублях |

## Навигация

- Карточки в market.html → `trade.html?ticker=SBER&mode=buy`
- `switchMode()` на trade.html обновляет URL через `history.replaceState`
- Если `ticker` не передан → редирект на `market.html`
- Портфель → `portfolio.html`, История → `history.html`

## Что ещё не сделано

- [ ] `portfolio_snapshots` — снапшоты не записываются; график роста портфеля не реализован
- [ ] `watchlist` — таблица есть в БД, страница/UI не реализованы
- [ ] Карточка баланса на главной — SVG-график роста (нужны снапшоты)
- [ ] trade.html — нет подтверждения (confirm dialog) перед сделкой
- [ ] Обновление цены в реальном времени (сейчас цена фиксируется при загрузке страницы)
