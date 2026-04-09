# План улучшений hh.ru-clicker

Результаты аудита кодовой базы. Проблемы сгруппированы по приоритету.

---

## P0 — Критичное

### Безопасность

| # | Проблема | Файл / строки | Решение |
|---|----------|---------------|---------|
| 1 | **Нет аутентификации** — сервис на `0.0.0.0:8000`, любой в сети может управлять ботом через WebSocket и API | `web_app.py` : `uvicorn.run(..., host="0.0.0.0")` ~5064, все `/api/*` и `/ws` | Middleware с API key / HTTP Basic; для WS — токен в query-параметре. Либо сменить на `127.0.0.1` + reverse proxy с auth |
| 2 | **`verify=False` / `ssl.CERT_NONE`** повсеместно (~15 мест) — MITM может перехватить куки, API-ключи, тела LLM-запросов | `web_app.py` : ~913, 1027, 1452, 1530, 1726, 1777, 2017, 3128, 3763, 4340, 4474, 4726, 4799, 5007 | Убрать `verify=False`; при корпоративном MITM-прокси — прокинуть свой CA bundle через переменную окружения |
| 3 | **LLM API-ключи в открытом виде** в `config.json` | `web_app.py` : `save_config()` ~310, ключи профилей ~4869–4887 | Переменные окружения / OS keyring; в конфиге только ссылка или placeholder |
| 4 | **XSS через `acc.name`** — имя аккаунта вставляется в `innerHTML` без `esc()` | `index.html` : `buildCardHTML` ~3176 | `esc(acc.name)` |
| 5 | **XSS через серверные сообщения** — `data.message` от API подставляется в `innerHTML` | `index.html` : `applyShowResult` ~4254–4258 | `esc(msg)` или `textContent` |
| 6 | **XSS в логах** — `e.time`, `e.icon` в HTML без экранирования | `index.html` : ~3497–3507, ~3577–3588 | `esc()` для всех интерполируемых значений |
| 7 | **XSS в ссылках** — `neg_id` в `href` без `encodeURIComponent` | `index.html` : ~2328, ~2482, ~3673 | `encodeURIComponent(neg_id)` |
| 8 | **XSS через `showConfirm`** — `msg` подставляется в `<p>${msg}</p>` | `index.html` : ~4924–4937 | `esc(msg)` |

### Race conditions

| # | Проблема | Файл / строки | Решение |
|---|----------|---------------|---------|
| 9 | **`accounts_data`** мутируется из воркер-потоков и API-хендлеров без общего лока | `web_app.py` : ~4015–4031, 4067–4069 | `threading.RLock` на все операции со списком или единый сервис с очередью команд |
| 10 | **`_llm_rr_index`** — обычный `int`, инкрементируется из разных потоков | `web_app.py` : ~35, используется в `generate_llm_reply` | `threading.Lock` при инкременте или `itertools.count()` |
| 11 | **`_resume_cache`** — `dict`, читается/пишется из потоков без лока | `web_app.py` : ~36–37 | `threading.Lock` вокруг чтения/записи |

---

## P1 — Важное

### Обработка ошибок

| # | Проблема | Файл / строки | Решение |
|---|----------|---------------|---------|
| 12 | **Голые `except:`** при чтении JSON-кешей — глотают `KeyboardInterrupt`, `SystemExit` | `web_app.py` : ~117–119, 126–128, 135–137 | `except (json.JSONDecodeError, OSError) as e:` + лог |
| 13 | **`except: pass`** при разборе ответа HH — теряется диагностика | `web_app.py` : ~1082–1083 | Логировать `json.JSONDecodeError` + фрагмент ответа |
| 14 | **`check_limit`: `except: return True`** — сетевая ошибка → ложное срабатывание лимита | `web_app.py` : ~1105–1106 | Разделить: сеть → retry/unknown; парсинг → отдельная ветка |
| 15 | **`load_browser_sessions`: `except: pass`** — тихая потеря сессий | `web_app.py` : ~259–266 | Лог + резервная копия файла |
| 16 | **WebSocket disconnect без лога** | `web_app.py` : ~3674–3675 | `log_error(..., exc_info=True)` |
| 17 | **`fetch_page` — одна попытка**, нет retry/backoff | `web_app.py` : ~737–747 | `tenacity` или ручной retry с jitter для 5xx/timeout |
| 18 | **`api_llm_config`: `await request.json()` без try** — невалидный JSON → 500 | `web_app.py` : ~4900–4903 | Обернуть в try как в `api_llm_profiles` |

### Производительность (бэкенд)

| # | Проблема | Файл / строки | Решение |
|---|----------|---------------|---------|
| 19 | **`asyncio.run` + новый `ClientSession`** на каждый цикл каждого воркера — лишние TLS-хэндшейки | `web_app.py` : ~2722–2815, 3119 | Переиспользовать session/connector или долгоживущий async worker |
| 20 | **`asyncio.gather` по всем URL×страницам сразу** — пики памяти, нагрузка на HH | `web_app.py` : ~3150–3175 | Батчи задач или `asyncio.Semaphore` |
| 21 | **`broadcast_loop` каждые 0.3с** — полный снапшот всем клиентам | `web_app.py` : ~5048–5056 | Инкрементальные дельты или хеш снапшота для сравнения |
| 22 | **`auto_decline_discards`** — синхронные `requests` в цикле | `web_app.py` : ~2048–2066 | async + пул или семафор |

### Производительность (фронтенд)

| # | Проблема | Файл / строки | Решение |
|---|----------|---------------|---------|
| 23 | **`renderAll` на каждое WS-сообщение** (каждые 0.3с) даже на неактивных вкладках | `index.html` : ~2997–3003 | Проверка `document.hidden` или хеш снапшота |
| 24 | **Поиск в логе/applied/db без debounce** — `oninput` пересобирает DOM на каждый символ | `index.html` : ~902, 929, 988 | `debounce(renderLog, 200)` |
| 25 | **`llmInterviewsLoad` делает двойной fetch** к `/api/interviews` | `index.html` : ~2278–2357, 2365–2366 | Передавать уже загруженные данные в `llmRenderAccStats` |
| 26 | **На вкладке views — `loadViews()` при каждом снапшоте** | `index.html` : ~3041–3048 | Кешировать, загружать по требованию |

### JS-баги (потенциальные TypeError)

| # | Проблема | Файл / строки | Решение |
|---|----------|---------------|---------|
| 27 | `snap.global_stats.total_found` без проверки | `index.html` : `renderHeader` ~3103–3108 | `snap.global_stats?.total_found ?? 0` |
| 28 | `acc.status.toUpperCase()` при undefined | `index.html` : ~3295–3297 | `(acc.status ?? 'unknown').toUpperCase()` |
| 29 | `acc.action_history.length` без проверки | `index.html` : ~3487–3488 | `acc.action_history?.length` |
| 30 | `snap.recent_responses.length` без проверки | `index.html` : ~3573–3574 | `snap.recent_responses?.length` |
| 31 | `rows.map(...)` после `res.json()` без `Array.isArray` | `index.html` : ~2314–2340, 3713, 3832 | `if (!Array.isArray(rows)) return;` |
| 32 | `o.vacancyNames.slice(0,3)` без проверки | `index.html` : ~3687–3691 | optional chaining |

---

## P2 — Желательное

### WebSocket и соединение

| # | Проблема | Файл / строки | Решение |
|---|----------|---------------|---------|
| 33 | При переподключении старый WS не закрывается явно, `reconnectTimer` не отменяется | `index.html` : ~2987–3024 | `State.ws?.close()` перед созданием нового; `clearTimeout(State.reconnectTimer)` |
| 34 | `sendCmd` при обрыве соединения тихо не отправляет — нет обратной связи | `index.html` : ~3026–3029 | Тост/статус «Нет соединения с сервером» |

### UX

| # | Проблема | Файл / строки | Решение |
|---|----------|---------------|---------|
| 35 | Вкладки — `<div>`, нет `role="tab"`, `tabindex`, `aria-selected` | `index.html` : ~867–877 | Семантичная разметка для accessibility |
| 36 | Вкладка LLM не попала в горячие клавиши (1–9, а вкладок 10) | `index.html` : `TAB_KEYS` ~5008–5016 | Добавить `0` → LLM или перенумеровать |
| 37 | `Notification.requestPermission()` вызывается автоматически через 3с — раздражает | `index.html` : ~5066–5067 | Запрашивать по жесту пользователя (кнопка в UI) |
| 38 | Несогласованные подписи кнопки «Поднять резюме» — «📤 Поднять» vs «📤 Сейчас» | `index.html` : ~2879 vs ~3243 | Унифицировать текст |
| 39 | `llmGlobalToggle`: `catch(e) {}` — сбой без обратной связи | `index.html` : ~2196 | Тост с ошибкой |
| 40 | Дублируется CSS-селектор `.hh-account-block` | `index.html` : ~379–387 | Объединить в один блок правил |

### Архитектура

| # | Проблема | Файл / строки | Решение |
|---|----------|---------------|---------|
| 41 | **Монолит ~5000 строк** — `web_app.py` содержит бот-логику, LLM, API, парсинг, конфиг | `web_app.py` | Выделить модули: `bot/`, `llm/`, `api/`, `config.py` |
| 42 | Смешение `threading` + `asyncio` без чётких границ | `web_app.py` : воркеры в потоках вызывают `asyncio.run` | Выбрать: полностью async или async только для I/O с чётким API |
| 43 | **Неатомарная запись** `applied_vacancies.json` и `test_required_vacancies.json` | `web_app.py` : `_save_applied_async` ~143–148 | Атомарная запись через `.tmp` + `rename` (как уже сделано для interviews) |
| 44 | Нет rate-limiting на API | `web_app.py` | `slowapi` или nginx `limit_req` |
| 45 | Нет CORS middleware | `web_app.py` | `CORSMiddleware` с явными origins на случай выноса фронта |
