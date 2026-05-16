# PUMP FADE — Референсный документ для Claude

**Версия файла:** v15.05 + OB/Flow module  
**Файл:** единый `index.html` (~3443 строк, Vanilla JS)  
**Деплой:** GitHub → Vercel → pump-fade.vercel.app  
**Репозиторий:** graffinme-del/pump-fade

> **Для Claude:** читай этот файл ПЕРВЫМ в каждой сессии, до того как смотреть на index.html.
> После любых правок запускай: `node --check /tmp/extracted.js` (см. раздел "Протокол правок").

---

## Стек и архитектура

```
Binance Futures WebSocket (aggTrade, depth)
    ↓
Сканер runIdeas() — каждые 3 мин, 200 монет, батчи по 12
    ↓
scoreForIdea() — технический анализ → idea object
    ↓
getAIVerdict() — Gemini / Claude → GO / WAIT / SKIP
    ↓
renderIdeas() → карточки в левой колонке
    ↓
showCoinCard() → центральная панель + OB/Flow модуль
```

**Ключевые URL-константы:**
- `FAPI = 'https://fapi.binance.com/fapi/v1'` — futures REST
- Futures data (другой base!): `https://fapi.binance.com/futures/data/...`
- WS: `wss://stream.binance.com:9443/stream?streams=<sym>@aggTrade`

---

## Глобальные константы (не менять без крайней нужды)

| Константа | Значение | Назначение |
|-----------|---------|------------|
| `FAPI` | `https://fapi.binance.com/fapi/v1` | REST endpoint |
| `CLAUDE_MODEL` | `'claude-haiku-4-5'` | модель Клода |
| `REFRESH_INTERVAL` | 30000 | обновление данных |
| `IDEAS_INTERVAL` | 3 * 60000 | сканер монет |
| `MIN_CONF` | 70 | порог confidence |
| `WR_KEY` | `'pf_wr_v2'` | localStorage winrate |
| `WR_HISTORY_KEY` | `'pf_wr_history_v1'` | localStorage история |
| `ACTIVE_KEY` | `'pf_active_v1'` | localStorage В работе |

---

## Глобальный объект S (главный стейт)

```javascript
const S = {
  sym,          // текущий символ карточки ('SOLUSDT' default)
  cds1h,        // свечи 1h текущего символа []
  cds15m,       // свечи 15m текущего символа []
  prices,       // {sym: price} кэш цен
  oi,           // open interest текущей карточки (число контрактов)
  fr,           // funding rate % (число)
  oiChange4h,   // % изменение OI за 4h (число | null) — fetchFutures()
  takerRatio,   // средний buy/sell ratio за 4h (число | null) — fetchFutures()
  ideas,        // [] все идеи от scoreForIdea()
  ideaDedup,    // {sym: timestamp} дедупликация
  volData,      // [] топ волатильных монет
  analysis,     // текущий analyzeFull() результат
  cvdData,      // [] CVD данные текущего символа
  positions,    // [] (зарезервировано)
  winrates,     // [] текущие WR записи (из localStorage)
  activeTrack,  // [] монеты "В работе": [{sym, dir, entry, sl, tp1, rr, addedAt, lastPrice, alerts}]
  scanning,     // boolean — идёт ли сканирование
  phase,        // текущая фаза (0=карточка, 1-4=фазы, 5=winrate)
  activeIdea,   // idea object текущей открытой карточки
  nextRefresh,  // timestamp следующего обновления
}
```

**Критически важно:**
- `S.oi`, `S.fr`, `S.oiChange4h`, `S.takerRatio` — заполняются в `fetchFutures()`, обнуляются при смене карточки
- `S.winrates` — загружается из localStorage при старте, не перезаписывать напрямую (только через `saveWR()`)
- `S.activeTrack` — аналогично, только через `saveActive()`

---

## Глобальный объект OB_STATE (Order Book & Flow)

```javascript
const OB_STATE = {
  // Карточка (card-only)
  cardSym,        // sym текущей карточки с OB tracking
  cardIdea,       // idea object текущей карточки
  renderTimer,    // setInterval для обновления UI
  bids, asks,     // [] снапшот стакана (±0.5% от цены)
  imbalance,      // число: bid_vol / ask_vol
  stacking,       // 'bid' | 'ask' | null
  depthSnapshots, // [{ts, bidTotal, askTotal}] история для stacking detection
  obScore,        // итоговый OB score (-10..+10)

  // Per-symbol (карточка + активные позиции)
  symState: {     // sym → объект:
    wsFlow,         // WebSocket объект aggTrade
    flowBuckets,    // [{ts, buyVol, sellVol}] — макс 1800 записей (30 мин)
    micro,          // {buyVol, sellVol, delta, velocity, direction, totalVol} — 60 сек
    short,          // аналогично — 300 сек
    mid,            // аналогично — 1800 сек
    flowScore,      // direction-aware flow score (-10..+10)
    label,          // строка описания flow
    lastAlertTs,    // timestamp последнего alert (throttle 5 мин)
  }
}
```

---

## Объект idea (создаётся в scoreForIdea())

Каждая idea содержит:

```javascript
{
  sym,          // 'BTCUSDT'
  dir,          // 'LONG' | 'SHORT'
  isPump,       // boolean
  conf,         // 0-100 confidence
  entry,        // цена входа
  sl,           // стоп-лосс
  tp1,          // тейк-профит
  rr,           // R/R ratio
  sigs,         // [] строки сигналов
  verdict,      // 'GO' | 'WAIT' | 'SKIP' (от AI)
  verdictConf,  // число
  verdictReason,// строка
  why,          // строка причины
  pumpPct,      // % пампа за 8h
  pumpSpeed,    // % изменение за 2h
  volRatio,     // объём / среднее
  rsiNow,       // текущий RSI
  atrRatio,     // ATR ratio
  cvdFalling,   // boolean
  cvdDiv,       // boolean — CVD дивергенция
  allPatterns,  // [{name, dir:'bear'|'bull', w, tf:'1h'|'15m'}] — свечные паттерны
  patScore,     // суммарный score паттернов
  trend4h,      // 'bull' | 'bear' | 'neutral'
  cds1h,        // свечи 1h (массив)
  cds15m,       // свечи 15m (массив)
  quoteVol24h,  // объём за 24h в USDT
  shortConf, shortSigs, longConf, longDir, longSigs, earlyPumpConf,
}
```

---

## DOM IDs — кто что обновляет

### Карточка монеты (showCoinCard + updateFuturesDisplay)

| ID | Элемент | Кто обновляет |
|----|---------|---------------|
| `cc-oi` | Open Interest значение | `updateFuturesDisplay()` |
| `cc-oi-change` | % изменение OI за 4h | `updateFuturesDisplay()` |
| `cc-oi-sub` | Подпись под OI | `updateFuturesDisplay()` |
| `cc-fr` | Funding Rate значение | `updateFuturesDisplay()` |
| `cc-fr-warn` | Предупреждение FR | `updateFuturesDisplay()` |
| `cc-taker` | Taker ratio метка | `updateFuturesDisplay()` |
| `cc-taker-sub` | Taker ratio число | `updateFuturesDisplay()` |
| `cc-cvd-trend` | CVD тренд | `updateFuturesDisplay()` |
| `cc-cvd-div` | CVD дивергенция | `updateFuturesDisplay()` |
| `cc-candles` | Свечные паттерны | `showCoinCard()` — сразу после innerHTML |
| `ob-container` | OB/Flow блок | `obRenderContainer()` |
| `oi-fr` | Мини-топбар FR | `updateFuturesDisplay()` |
| `dynamic-zone` | Вся центральная панель | `showCoinCard()`, `showWinrates()`, `showPhaseContent()` |

### Графики

| ID | Тип | Кто управляет |
|----|-----|---------------|
| `chart-1h` | Свечной 1h | `drawCandles()` |
| `chart-15m` | Свечной 15m | `drawCandles()` |
| `cvd-chart` | CVD 15m | `drawCVD()` |
| `vol-chart` | Volume 1h | `drawVolume()` |

### Топбар

| ID | Назначение |
|----|-----------|
| `oi-fr` | FR мини-индикатор |
| `btc-price` | BTC цена |
| `btc-chg` | BTC изменение |
| `live-dot` | Статус LIVE/SCAN |

---

## Карта функций по доменам

### Технический анализ

| Функция | Назначение |
|---------|-----------|
| `ema(arr, p)` | Exponential Moving Average |
| `rsi(c, p=14)` | RSI |
| `macd(c)` | MACD → {ml, sig, h} |
| `atrCalc(cds, p=14)` | ATR |
| `cvdFromCandles(cds)` | CVD из свечей |
| `analyzeFull(cds1h, cds15m)` | Полный анализ фаз → {phase, exSigs, bkSigs, ...} |
| `scoreForIdea(sym)` | Главная функция скоринга → idea object |
| `tfOrient(cds1h, cds15m, trend4h)` | **Возвращает {m5, m15, h1, h4}** — НЕ строку! |

**КРИТИЧНО по `tfOrient`:** функция принимает 3 параметра и возвращает объект. Не упрощать до строки.

### Сканер и идеи

| Функция | Назначение |
|---------|-----------|
| `runIdeas(manual=false)` | Запуск сканера 200 монет |
| `renderIdeas()` | Рендер карточек в левой колонке |
| `setIdeasTab(tab)` | Переключение вкладок (Сигналы/Наблюдение/В работе/...) |
| `updateTabCounts()` | Обновление счётчиков на вкладках |
| `selectIdea(i)` | Выбор идеи по индексу |
| `openSym(sym)` | Открыть монету по символу |
| `loadSym(sym, idea=null)` | Загрузить данные символа (свечи, CVD, графики) |

### Карточка монеты

| Функция | Назначение |
|---------|-----------|
| `showCoinCard(idea, analysis)` | Рендер всей карточки в dynamic-zone |
| `fetchFutures(sym)` | REST: OI + FR + OI history 4h + Taker ratio 4h → обновляет S.oi, S.fr, S.oiChange4h, S.takerRatio |
| `updateFuturesDisplay()` | Обновляет DOM: cc-oi, cc-oi-change, cc-fr, cc-fr-warn, cc-taker, cc-taker-sub, cc-cvd-* |

**КРИТИЧНО:** `fetchFutures()` вызывает `updateFuturesDisplay()` после всех 4 запросов. Не переставлять порядок.

### OB / Flow модуль

| Функция | Назначение |
|---------|-----------|
| `obGetOrCreateSymState(sym)` | Инициализация per-symbol стейта |
| `obProcessAggTrade(data, sym)` | Обработка aggTrade события → 1-секундные бакеты |
| `obComputeWindow(buckets, windowSec)` | Rolling window → {buyVol, sellVol, delta, velocity, direction, totalVol} |
| `obUpdateWindows(sym)` | Обновляет micro(60с)/short(300с)/mid(1800с) |
| `obComputeFlowScore(idea, micro, short, mid)` | Direction-aware score (-10..+10), 8 состояний |
| `obFetchSnapshot(sym)` | REST depth100 → OB_STATE.bids/asks |
| `obComputeImbalance(sym)` | OBI ±0.5% + stacking/pulling detection |
| `obGetOBScore(idea)` | OB score из imbalance + stacking |
| `obRenderContainer()` | Рендер HTML в #ob-container |
| `obConnectFlowStream(sym)` | aggTrade WS с auto-reconnect |
| `obDisconnectFlowStream(sym)` | Закрытие WS |
| `startOBTracking(idea)` | Запуск OB tracking для карточки |
| `stopCardOBTracking()` | Остановка OB tracking карточки |
| `startActiveFlowTracking(sym)` | Запуск flow WS для активной позиции |
| `stopActiveFlowTracking(sym)` | Остановка flow WS активной позиции |
| `checkActiveFlowAlerts()` | Toast-алерты, throttle 5 мин/символ |

**Flow windows:** ТОЛЬКО rolling (не candle-reset). micro=60с, short=300с, mid=1800с.  
**Velocity:** нормализована по totalVol. Delta velocity = (rDelta - eDelta) / totalVol.

### Winrate

| Функция | Назначение |
|---------|-----------|
| `showWinrates()` | Рендер таблицы (dynamic-zone). Вкладки: today / **yesterday** / week / month / all |
| `scheduleWinrateCheck(idea)` | Запланировать проверку GO сигнала |
| `checkPendingWinrates()` | Проверить все pending на hit TP/SL |
| `recheckAllWinrates()` | Пересчитать все записи |
| `loadWR() / saveWR()` | localStorage `pf_wr_v2` |
| `loadWRHistory() / saveWRHistory()` | localStorage `pf_wr_history_v1` |
| `checkMidnightReset()` | Архивировать S.winrates в историю в 00:00 МСК |

**Вкладка "Вчера":** фильтрует по МСК-времени (UTC+3), только история предыдущего дня. `S.winrates` не включается (они принадлежат сегодня).

### AI (Gemini / Claude)

| Функция | Назначение |
|---------|-----------|
| `getAIVerdict(idea)` | Роутер: Claude если есть ключ, иначе Gemini |
| `callClaudeVerdict(idea)` | Вердикт через Claude API |
| `getGeminiVerdict(idea)` | Вердикт через Gemini API |
| `callGemini(prompt, maxTokens)` | Универсальный вызов Gemini |
| `callClaude(prompt, maxTokens)` | Универсальный вызов Claude |
| `sendChat(userMsg)` | Чат в правой колонке |
| `askAIAboutIdea(sym)` | Анализ монеты через AI |
| `buildContext()` | Контекст для AI чата |

**Приоритет моделей:** Claude (если есть `claude_api_key`) → Gemini.  
**Модель Claude:** `claude-haiku-4-5` (константа `CLAUDE_MODEL`).  
**Системный промпт Gemini:** disciplined trader, <30% GO rate, NEVER GO против 4h тренда.

### Активные позиции ("В работе")

| Функция | Назначение |
|---------|-----------|
| `toggleActive(sym)` | Добавить/убрать из В работе → запускает/останавливает flow WS |
| `removeActiveTrack(sym)` | Убрать из В работе |
| `runActiveMonitor()` | Главный монитор (15 сек) — проверяет CVD, EMA, объём, свечи |
| `renderActiveTab(list)` | Рендер вкладки В работе |
| `openActiveCard(sym)` | Открыть карточку активной позиции |

**Сигналы разворота** (требуют подтверждения на следующей проверке):
- CVD разворот против позиции >15% — `_cvdPending`
- EMA20 пробита против — `_emaPending`
- Объём ×2+ при движении против — `_volPending`

### Вспомогательные

| Функция | Назначение |
|---------|-----------|
| `fmt(n, d=2)` | Форматирование числа (авто K/M/B) |
| `fmtAge(ts)` | Возраст в с/м/ч |
| `fmtElapsed(min)` | Elapsed time |
| `fmtSignalTime(ts)` | Время сигнала |
| `showToast(msg)` | Toast уведомление |
| `notify(title, body)` | Push уведомление |
| `fetchCds(sym, iv, limit=120)` | REST klines → [{time,open,high,low,close,volume}] |
| `fetchPrice(sym)` | REST ticker/price → число |
| `getCapLabel(v)` | Категория капитализации по объёму → {label, color, icon} |
| `connectWS()` | Главный WS (markPrice потоки) |
| `checkBTC()` | Обновление BTC тренда (каждые 5 мин) |
| `playAlert()` | Звуковой алерт (только критические события) |

---

## Поток данных fetchFutures()

```
fetchFutures(sym)
  ├── /fapi/v1/openInterest?symbol=sym          → S.oi
  ├── /fapi/v1/fundingRate?symbol=sym&limit=1   → S.fr
  ├── /futures/data/openInterestHist             → S.oiChange4h (% за 4h)
  │   period=30m&limit=9 (9×30мин = 4 часа)
  └── /futures/data/takerlongshortRatio          → S.takerRatio (avg за 4h)
      period=1h&limit=4
  
  → updateFuturesDisplay() → обновляет все cc-* DOM элементы
```

---

## Скоринг (scoreForIdea)

```
final_score = strategy_score + orderbook_score + flow_score + absorption_score
```

- **strategy_score:** детекторы L1-L5 (LONG) + S1-S6 (SHORT)
- **OB score:** imbalance (±5-8) + stacking/pulling (±4-7) → вес 15-20%
- **Flow score:** direction-aware confluence (-10..+10), 8 состояний
- **MIN_CONF:** 70 — порог для отображения

**SL/TP (структурный):**
- LONG: `sl = min(swingLow_1h - ATR*0.3, EMA50 - ATR*0.2)`, `tp = max(slDist*2, entry*0.05)`
- SHORT: `sl = max(swingHigh_1h + ATR*0.3, EMA50 + ATR*0.2)`, `tp = max(slDist*2, entry*0.05)`

---

## MTF (Multi-Timeframe) — tfOrient()

```javascript
// ПРАВИЛЬНАЯ СИГНАТУРА:
tfOrient(cds1h, cds15m, trend4h) → {m5, m15, h1, h4}

// ПРАВИЛЬНЫЙ ВЫЗОВ в showCoinCard():
const tf = tfOrient(idea.cds1h||S.cds1h||[], idea.cds15m||S.cds15m||[], idea.trend4h);

// m5  — последняя треть 15m свечей (если < 22 свечей → берёт m15)
// m15 — полные cds15m через orient()
// h1  — cds1h через orient()
// h4  — idea.trend4h (уже вычислен в scoreForIdea)
```

**ЗАПРЕЩЕНО:** возвращать строку из tfOrient(). Это ломает весь MTF блок карточки.

---

## localStorage ключи

| Ключ | Данные | Лимит |
|------|--------|-------|
| `pf_wr_v2` | Текущие WR записи | max 50 |
| `pf_wr_history_v1` | Архив WR | max 2000 |
| `pf_active_v1` | Монеты В работе | — |
| `gemini_api_key` | Ключ Gemini | — |
| `gemini_model_override` | Выбранная модель Gemini | — |
| `claude_api_key` | Ключ Claude | — |
| `muted` | Звук вкл/выкл | — |
| `wr_reset_date` | Дата сброса WR | — |

---

## Критические инварианты — НИКОГДА НЕ НАРУШАТЬ

1. **flow = rolling windows** — не привязывать к свечам, не делать candle reset
2. **tfOrient()** — всегда принимает 3 аргумента, всегда возвращает объект `{m5, m15, h1, h4}`
3. **fetchFutures()** — обязательно вызывает `updateFuturesDisplay()` в конце, после всех 4 fetch
4. **showCoinCard()** — после `dz.innerHTML=...` сначала заполняет `#cc-candles` из `idea.allPatterns`, потом вызывает `fetchFutures()`, потом `stopCardOBTracking()` + `startOBTracking(idea)`
5. **OB_STATE.symState** — per-symbol, не сбрасывать при смене карточки (активные позиции могут использовать тот же WS)
6. **S.winrates / S.activeTrack** — только через `saveWR()` / `saveActive()`, не localStorage напрямую
7. **MIN_CONF = 70** — порог для GO-сигналов, не снижать без явного запроса
8. **Звук** — только для критических алертов (SL пробит, CVD разворот подтверждён, EMA пробита)

---

## Порядок вызовов при открытии карточки

```
openSym(sym) / selectIdea(i)
  └── loadSym(sym, idea)
        ├── fetchCds(1h + 15m)
        ├── drawCandles() + drawCVD() + drawVolume()
        ├── scoreForIdea() [если нет идеи]
        └── showCoinCard(idea, analysis)
              ├── tfOrient(cds1h, cds15m, trend4h)  ← объект {m5,m15,h1,h4}
              ├── dz.innerHTML = `...карточка...`
              ├── ccCandles.innerHTML = idea.allPatterns...
              ├── fetchFutures(sym) → updateFuturesDisplay()
              ├── stopCardOBTracking()
              └── startOBTracking(idea)
                    ├── obConnectFlowStream(sym)
                    ├── obFetchSnapshot(sym) + obComputeImbalance(sym)
                    └── renderTimer = setInterval(obRenderContainer, 2000)
```

---

## Протокол правок для Claude

### Начало сессии:
1. Прочитать этот файл
2. Прочитать нужные секции index.html (НЕ весь файл)
3. Понять что именно меняется и какие функции это затронет

### При правках:
1. Использовать точечный Edit (не перезаписывать весь файл)
2. После каждого Edit проверять что не задеты соседние строки
3. Запускать проверку синтаксиса:
```bash
python3 -c "
import re
content = open('/sessions/.../mnt/outputs/index.html').read()
scripts = re.findall(r'<script[^>]*>(.*?)</script>', content, re.DOTALL)
open('/tmp/extracted.js', 'w').write('\n'.join(scripts))
"
node --check /tmp/extracted.js && echo "✅ OK" || echo "❌ ERROR"
```
4. Проверить grep'ом что новые ID/переменные не конфликтуют с существующими

### Чего НЕ делать:
- Не перезаписывать весь index.html через Write (только через Edit)
- Не добавлять новые глобальные переменные с именами похожими на существующие
- Не менять сигнатуру tfOrient()
- Не менять порядок вызовов в showCoinCard() после innerHTML
- Не добавлять candle-reset логику в flow модуль

---

## История изменений

| Версия | Изменение |
|--------|-----------|
| v15.05 base | Базовый файл от пользователя |
| +OB/Flow | Добавлен полный OB/Flow модуль (401 строк): OB_STATE, aggTrade WS, rolling windows, confluence scoring, obRenderContainer |
| +Fixes | Исправлены: tfOrient() → объект, fetchFutures() + OI history + taker ratio, updateFuturesDisplay() + cc-oi-change + cc-taker-sub, showCoinCard() → #cc-candles из allPatterns, showWinrates() + вкладка Вчера |
