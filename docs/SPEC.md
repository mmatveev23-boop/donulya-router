---
tags: [spec, donulya-ai-auditor, lead-router, distribution, architecture]
date: 2026-05-15
project: donulya-ai-auditor
audience: программист (внедрение бэкенд-роутера) + РОП (проверка логики)
status: ready · все 13 вопросов закрыты · опубликован в репо donulya-router
github_repo: https://github.com/mmatveev23-boop/donulya-router (private)
related: regulation-цвета-сделок-и-зоны-клоузера.md, router-логика-простыми-словами.md, spec-виджет-amocrm-данные-и-контроль.md, decisions.md
---

# Спецификация · Робот-диспетчер распределения лидов

Бэкенд-сервис который мгновенно отдаёт каждого оплатившего лида клоузеру кто прямо сейчас свободен и в форме. Расширяемая платформа: сейчас работает на отдел клоузеров, потом подключим охотников и сопровождение через ту же модель.

**Связано:**
- [[Projects/donulya-ai-auditor/regulation-цвета-сделок-и-зоны-клоузера|регламент цветов и зон Светофора]] — базовая семантика
- [[Projects/donulya-ai-auditor/spec-виджет-amocrm-данные-и-контроль|спецификация виджета в карточке]] — UI клоузера
- [[Projects/donulya-ai-auditor/decisions|архитектурные решения]]

---

## 1. Цель и не-цель

**Цель.** Свести время от оплаты 99 ₽ на МПП1 до первого реального касания клоузера к **< 5 минут в рабочее время**. Распределить лиды равномерно с учётом дисциплины клоузера (Светофор). Дать РОПу инструмент управления потоком: правила, on/off, ручное перераспределение.

**Не-цель.** Заменить виджет Светофора в amoCRM. Заменить дашборд аналитики РОПа. Решать что считается «реальным касанием» (это уже в regulation). Делать дозвонщика автоматического — звонит всё-таки клоузер.

---

## 2. Терминология

| Термин | Что значит |
|---|---|
| **Отдел** (Department) | Группа сотрудников с общей логикой роутинга: closers, hunters, support |
| **Лид** | Сделка в amoCRM на этапе который тригерит роутер |
| **Назначение** (Assignment) | Привязка лида к одному сотруднику + таймер реагирования |
| **Светофор сделки** | 🟢🟡🟠🔴 цвет по часам без касания (см. regulation) |
| **Зона клоузера** | 🟢🟡🟠🔴 цвет по % красных сделок у клоузера |
| **Round-Robin** | Указатель идёт по кольцу подходящих сотрудников, FIFO |
| **Окно реакции** | Таймер сколько у сотрудника есть на первое действие (5 мин) |
| **Отложенная очередь** | Накопитель лидов которые прилетели вне рабочих часов |

---

## 3. Сценарии (user stories)

### 3.1 Лид оплатил → роутер назначает клоузера

**Дано:** клиент только что оплатил 99 ₽ на МПП1, сделка автоматически создана в МПП2 (donulya2) на этапе «Новый клиент».

**Действия системы:**
1. amoCRM шлёт webhook `lead.status_changed` на `/api/v1/triggers/amo` сервиса роутера
2. Роутер находит правило: triggers где `pipeline_id=10780422 AND stage_id=<новый_клиент>`
3. Запускает матчер для отдела `closers`:
   - Берёт всех `User` с `department=closers` AND `active=true`
   - Применяет фильтры (расписание / телефония / лимит / светофор) — см. §7
   - Если кандидат есть — Round-Robin выбор → создаёт `Assignment` со статусом `ASSIGNED`, таймер 5 мин, ставит задачу в amoCRM «🔥 Звони клиенту сейчас» с responsible_user_id
   - Если нет — создаёт `Assignment` со статусом `QUEUED`, лид в очереди
4. Лог: `RoutingDecision` с указанием почему выбрали кого выбрали и кого отвергли

**Реакция клоузера:**
- Видит 🔥 pill в виджете шапки + amoCRM-задачу «🔥 Звони клиенту сейчас»
- Клик на pill = открывает карточку сделки в amoCRM (новая вкладка), **таймер НЕ сбрасывается** (это просто навигация)
- Набирает клиента из карточки → Sipuni шлёт webhook `call_initiated` → **только сейчас** роутер сбрасывает таймер, переводит `Assignment` в `ENGAGED`
- Не позвонил 5 мин (даже если открывал карточку) → таймер истекает → роутер переназначает следующему по той же формуле, статус первого `Assignment` → `TIMEOUT` (без штрафа), создаётся новый `Assignment`

### 3.2 Никого свободного → отложенная очередь

**Дано:** оплата в 14:00 пятницы, всех клоузеров отсеяли (на звонке, в красной зоне, лимит 40).

**Действия:**
1. Роутер создаёт `Assignment` со статусом `QUEUED`, пишет причину в `RoutingDecision` («все на звонке × 5, в красной × 1, лимит × 2»)
2. Worker раз в 30 сек проверяет очередь: появился свободный → берёт **самый старый** `QUEUED` → назначает по обычной формуле
3. Если `QUEUED` висит > 10 минут → пуш РОПу через TG: «Лид #12345 висит 12 мин, никто не разобрал. Разрули» + ссылка на сделку
4. Когда РОП открывает админку → видит таблицу очереди + кнопку «Назначить вручную» с выпадающим списком клоузеров (показывая статус каждого)

### 3.3 Ночь / выходные

**Дано:** оплата в субботу 17:00 (выходной) или будний день в 22:00 (ночь).

**Действия:**
1. Матчер видит: рабочий календарь = выкл → `Assignment` создаётся со статусом `DEFERRED` (отдельный от `QUEUED`), указывается `dispatch_after_ts = ближайшее_утро_9:00_рабочего_дня`
2. Worker в 9:00 рабочего дня берёт все `DEFERRED` где `dispatch_after_ts <= now`, сортирует по `created_at` (FIFO — кто первый оплатил, того первый клоузер)
3. **Pull-режим раздачи** (см. открытый вопрос §11.G):
   - В 9:00 каждый клоузер видит в виджете шапки «🔥 У тебя 3 лида в очереди, возьми» (счётчик), кнопка «Взять следующего»
   - Клик → Pop из очереди → `Assignment` со статусом `ASSIGNED`, таймер 5 мин
   - Альтернатива: push-режим (роутер сразу назначает по 1 каждому, как только клоузер не на звонке)

### 3.4 Возврат «из закрыто и нереализовано»

**Дано:** клиент в архиве, есть критерий реанимации (например, прошло > 30 дней + не было reason="отказался от услуг"). РОП нажимает «Реанимировать» в админке.

**Действия:**
1. Админка ставит сделку на этап «Новый клиент» в МПП2
2. amoCRM шлёт webhook (как в §3.1)
3. Матчер ставит флаг `is_revival=true`, **игнорирует старого клоузера** в выборе (если клиент не запросил его явно — см. открытый вопрос §11.H)
4. Назначается свежий клоузер по той же формуле

### 3.5 РОП управляет: on/off клоузер, меняет правила

**Дано:** Мифтахов в отпуске, РОП хочет временно вывести из пула.

**Действия:**
1. РОП открывает админку donula.online/router-admin/, заходит в «Клоузеры» (под отделом)
2. Видит таблицу: имя · текущий статус (✅ В пуле / ❌ Off / ⚠ Заблокирован светофором) · последнее назначение · сегодня получил N лидов
3. Тогглит Мифтахова → `User.active=false` → роутер мгновенно перестаёт его учитывать
4. Через неделю тоггл обратно → снова в пул

**Дано:** РОП хочет изменить пороги светофора (например, 🟠 не -50% а -30%).

**Действия:**
1. Админка → «Правила отдела clоузеров» → «Светофор» → меняет значения
2. Правило — `Rule` запись с типом `light_zone_weights` и JSON-конфигом
3. Следующий запуск матчера читает новые правила

---

## 4. Архитектура · общий вид

```
┌──────────────────────────────────────────────────────────────┐
│                        amoCRM (donulya2)                      │
│  - Воронка МПП2 (closers)                                     │
│  - Воронка МПП1 (hunters, потом)                              │
│  - Виджет в шапке (см. spec-виджет-amocrm-данные-и-контроль)  │
└────────────┬──────────────────────────────────┬───────────────┘
             │ webhook (lead.status_changed)    │ виджет шапки
             │                                  │ читает state
             ▼                                  │
┌──────────────────────────────────────┐        │
│         donulya-router               │        │
│  ┌────────────────────────────────┐  │        │
│  │ FastAPI (Python 3.12)          │  │        │
│  │  - /api/v1/triggers/amo        │◄─┼────────┘
│  │  - /api/v1/dispatch/run        │  │
│  │  - /api/v1/admin/*             │  │
│  │  - /api/v1/widget/*            │  │
│  └────────────────────────────────┘  │
│  ┌────────────────────────────────┐  │
│  │ Background workers (asyncio)   │  │
│  │  - taskQueue: 5-мин таймеры    │  │
│  │  - hourly: пересчёт зон        │  │
│  │  - 9:00 cron: ночная очередь   │  │
│  │  - 30sec: try-assign-queued    │  │
│  └────────────────────────────────┘  │
│  ┌────────────────────────────────┐  │
│  │ PostgreSQL (state)             │  │
│  │  - departments / users / rules │  │
│  │  - assignments / queue / log   │  │
│  └────────────────────────────────┘  │
└─────┬──────────────────┬─────────────┘
      │                  │
      ▼                  ▼
┌──────────────┐   ┌──────────────────────┐
│  Telephony   │   │  donula.online/      │
│  - Sipuni    │   │  router-admin/       │
│  - МегаФон   │   │  (React + Vite)      │
│  - Контур    │   │  Auth: amoCRM OAuth  │
│  Webhooks    │   └──────────────────────┘
└──────────────┘
```

**Технологии:**
- **Backend:** Python 3.12, FastAPI, SQLAlchemy + Alembic, asyncio workers, loguru, httpx (с прокси для TG если нужно)
- **DB:** PostgreSQL 16 (выбрал не SQLite потому что нужны конкурентные write'ы в queue и assignment'ах; решение vs amo-partner-bridge SQLite — см. открытый вопрос §11.L)
- **Frontend admin:** React + Vite, shadcn/ui, авторизация через amoCRM OAuth (тот же tenant)
- **Деплой:** systemd сервис на `87.228.88.163` (тот же сервер где amo-partner-bridge и hiring-backend), nginx маршрут `donula.online/router/*` → 127.0.0.1:8444 (новый порт), `donula.online/router-admin/` → статический build

---

## 5. Модель данных

```sql
-- ОТДЕЛЫ
CREATE TABLE departments (
  id           SERIAL PRIMARY KEY,
  code         TEXT UNIQUE NOT NULL,           -- 'closers' | 'hunters' | 'support'
  title        TEXT NOT NULL,                  -- 'Клоузеры МПП2'
  active       BOOLEAN DEFAULT true,
  config       JSONB NOT NULL,                  -- настройки специфичные для типа
  created_at   TIMESTAMPTZ DEFAULT NOW()
);
-- Пример config для closers (без capacity_limit — регулирование через Светофор, см. §11.I):
-- {
--   "trigger": { "pipeline_id": 10780422, "stage_id": 84657777 },
--   "reaction_window_sec": 300,
--   "queue_alert_to_rop_sec": 600,
--   "schedule": { "tz": "Europe/Moscow", "workdays": [1,2,3,4,5], "hours": [9,21] },
--   "light_weights": { "green": 1.0, "yellow": 1.0, "orange": 0.5, "red": 0.0 },
--   "active_pipeline_id": 10780422,
--   "active_stage_ids": [<думает>, <обещал>, <частичка>, <полная>]
-- }

-- СОТРУДНИКИ (клоузеры, охотники, ...)
CREATE TABLE users (
  id              SERIAL PRIMARY KEY,
  amo_user_id     INTEGER UNIQUE NOT NULL,      -- из amoCRM
  department_id   INTEGER REFERENCES departments(id),
  full_name       TEXT NOT NULL,
  active          BOOLEAN DEFAULT true,         -- toggled РОПом
  manual_block    BOOLEAN DEFAULT false,        -- временный блок (отпуск, болезнь)
  manual_block_until TIMESTAMPTZ,
  last_assigned_at TIMESTAMPTZ,                  -- для Round-Robin сортировки
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- НАЗНАЧЕНИЯ
CREATE TABLE assignments (
  id              BIGSERIAL PRIMARY KEY,
  lead_id         BIGINT NOT NULL,               -- amoCRM lead.id
  department_id   INTEGER REFERENCES departments(id),
  user_id         INTEGER REFERENCES users(id),  -- NULL если в очереди
  status          TEXT NOT NULL,                 -- 'PENDING'|'ASSIGNED'|'ENGAGED'|'TIMEOUT'|'QUEUED'|'DEFERRED'|'COMPLETED'|'CANCELLED'
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  assigned_at     TIMESTAMPTZ,
  engaged_at      TIMESTAMPTZ,                   -- момент когда сбросили таймер
  expires_at      TIMESTAMPTZ,                   -- когда истекает 5-мин таймер
  dispatch_after_ts TIMESTAMPTZ,                  -- для DEFERRED — когда запускать
  is_revival      BOOLEAN DEFAULT false,
  meta            JSONB
);
CREATE INDEX assignments_active_idx ON assignments(status, expires_at) WHERE status IN ('ASSIGNED','QUEUED','DEFERRED');

-- ЛОГ РЕШЕНИЙ матчера
CREATE TABLE routing_decisions (
  id              BIGSERIAL PRIMARY KEY,
  assignment_id   BIGINT REFERENCES assignments(id),
  decision        TEXT NOT NULL,                 -- 'ASSIGNED' | 'QUEUED' | 'DEFERRED'
  chosen_user_id  INTEGER,
  candidates      JSONB,                          -- [{user_id, accepted, reject_reason}]
  rule_snapshot   JSONB,                          -- какая версия правил применялась
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- СОБЫТИЯ от внешних систем (телефония, календарь)
CREATE TABLE user_availability_events (
  id              BIGSERIAL PRIMARY KEY,
  user_id         INTEGER REFERENCES users(id),
  source          TEXT NOT NULL,                 -- 'sipuni' | 'megafon' | 'kontur' | 'manual'
  event           TEXT NOT NULL,                 -- 'call_start' | 'call_end' | 'busy' | 'free'
  ts              TIMESTAMPTZ DEFAULT NOW(),
  raw             JSONB
);
-- Кэш текущего состояния:
CREATE TABLE user_availability_state (
  user_id         INTEGER PRIMARY KEY REFERENCES users(id),
  is_on_call      BOOLEAN DEFAULT false,
  on_call_since   TIMESTAMPTZ,
  manual_busy     BOOLEAN DEFAULT false,         -- клоузер сам себя пометил
  manual_busy_until TIMESTAMPTZ,
  updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- АГРЕГАТЫ по портфелю клоузера (обновляются hourly worker'ом)
CREATE TABLE user_pipeline_state (
  user_id         INTEGER PRIMARY KEY REFERENCES users(id),
  active_count    INTEGER DEFAULT 0,
  green_count     INTEGER DEFAULT 0,
  yellow_count    INTEGER DEFAULT 0,
  orange_count    INTEGER DEFAULT 0,
  red_count       INTEGER DEFAULT 0,
  zone            TEXT,                          -- 'green'|'yellow'|'orange'|'red'
  red_percent     NUMERIC(5,2),
  updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- ПРАВИЛА (расширяемые)
CREATE TABLE rules (
  id              SERIAL PRIMARY KEY,
  department_id   INTEGER REFERENCES departments(id),
  kind            TEXT NOT NULL,                 -- 'schedule'|'telephony_block'|'light_zone'|'custom_filter' (capacity отменён 2026-05-15, см. §11.I)
  config          JSONB NOT NULL,
  enabled         BOOLEAN DEFAULT true,
  order_idx       INTEGER DEFAULT 0,             -- порядок применения фильтров
  created_at      TIMESTAMPTZ DEFAULT NOW()
);
-- Пример: rule{kind='light_zone', config={'block_zones':['red'], 'weights':{'orange':0.5}}}
-- Пример: rule{kind='custom_filter', config={'sql':'...'}} — escape hatch для редких случаев

-- АУДИТ изменений правил (кто, когда, что изменил)
CREATE TABLE rule_changes_log (
  id              BIGSERIAL PRIMARY KEY,
  rule_id         INTEGER REFERENCES rules(id),
  changed_by_amo_user_id INTEGER,
  before_config   JSONB,
  after_config    JSONB,
  changed_at      TIMESTAMPTZ DEFAULT NOW()
);
```

**Почему так:**
- `departments.config` — полиморфизм через JSON. Поля специфичные для типа отдела (триггер, расписание, лимит, веса светофора)
- `rules` отдельной таблицей — позволяет РОПу добавлять/удалять/менять порядок фильтров без миграции схемы
- `assignments` хранит всё: и `QUEUED` (ждёт назначения), и `DEFERRED` (ночные), и `ASSIGNED`, и `TIMEOUT` (для статистики), и `COMPLETED`
- `user_availability_state` — кэш для быстрого матчинга (читается на каждый dispatch)
- `user_pipeline_state` — кэш агрегатов (обновляется раз в час, чтобы матчер не дёргал amoCRM API)

---

## 6. FSM назначения

```
                           ┌──────────────┐
                           │   PENDING    │  (только что создано, ещё не обработан матчер)
                           └──────┬───────┘
                                  │ match()
                ┌─────────────────┼─────────────────┐
                ▼                 ▼                 ▼
         ┌──────────┐      ┌──────────┐      ┌──────────┐
         │ ASSIGNED │      │  QUEUED  │      │ DEFERRED │
         │ user_id  │      │ user_id  │      │ user_id  │
         │ expires_ │      │  = NULL  │      │  = NULL  │
         │ at = +5м │      └────┬─────┘      │ dispatch_│
         └─────┬────┘           │            │ after =  │
               │                │            │ 9:00     │
        ┌──────┴──────┐         │ worker     └────┬─────┘
        ▼             ▼         │ → match()       │ 9:00 cron
  ┌─────────┐   ┌──────────┐    ▼                 ▼ → match()
  │ ENGAGED │   │ TIMEOUT  │  ┌──────────┐    ┌──────────┐
  │ позвонил│   │ передаём │  │ ASSIGNED │    │ ASSIGNED │
  └────┬────┘   │ дальше   │  └──────────┘    └──────────┘
       │        └────┬─────┘
       │             │ создаём новый
       │             │ PENDING для того же lead_id
       │             ▼
       │      ┌────────────┐
       │      │   PENDING  │
       │      └────────────┘
       ▼
  ┌──────────┐
  │COMPLETED │  (сделка ушла дальше по воронке, роутер выпускает)
  └──────────┘
```

**Особенности:**
- На один `lead_id` может быть **много** `assignments` (если переназначали по TIMEOUT). Активный всегда один.
- `CANCELLED` — РОП руками отменил назначение (например, перевёл лида в архив).

---

## 7. Алгоритм матчинга

Псевдокод. Каждый шаг = отдельное `Rule` в БД, выполняется в порядке `order_idx`.

```python
async def match(lead, department):
    candidates = await all_active_users(department)
    rejected = []

    # 1. Глобальная отсечка по расписанию (rule kind='schedule')
    schedule_rule = department.rules.where(kind='schedule', enabled=True).first()
    if schedule_rule and not in_workhours(schedule_rule.config, now()):
        # Полностью вне расписания → DEFERRED
        next_workhour = calc_next_workhour(schedule_rule.config, now())
        return create_assignment(lead, status='DEFERRED', dispatch_after=next_workhour)

    # 2. Фильтры по каждому кандидату
    for rule in department.rules.where(enabled=True).order_by('order_idx'):
        if rule.kind == 'manual_block':
            candidates = [c for c in candidates if not (c.manual_block and (c.manual_block_until is None or c.manual_block_until > now()))]
        elif rule.kind == 'telephony_block':
            states = await fetch_availability_states([c.id for c in candidates])
            candidates = [c for c in candidates if not states[c.id].is_on_call]
            candidates = [c for c in candidates if not (states[c.id].manual_busy and states[c.id].manual_busy_until > now())]
        # NB: rule.kind == 'capacity' — отменён 2026-05-15 (см. §11.I).
        # Регулирование нагрузки происходит только через Светофор ниже.
        elif rule.kind == 'light_zone':
            blocked_zones = set(rule.config['block_zones'])
            for c in list(candidates):
                zone = (await get_pipeline_state(c.id)).zone
                if zone in blocked_zones:
                    rejected.append((c, f'zone={zone} blocked'))
                    candidates.remove(c)
        elif rule.kind == 'custom_filter':
            # escape hatch
            candidates = await apply_custom_filter(candidates, rule.config)

    # 3. Если никого — в очередь
    if not candidates:
        return create_assignment(lead, status='QUEUED', meta={'rejected': rejected})

    # 4. Веса по светофору
    light_rule = department.rules.where(kind='light_zone').first()
    weights = light_rule.config.get('weights', {}) if light_rule else {}
    candidates_with_weight = []
    for c in candidates:
        zone = (await get_pipeline_state(c.id)).zone
        w = weights.get(zone, 1.0)
        if w > 0:
            candidates_with_weight.append((c, w))

    # 5. Round-Robin с учётом веса
    chosen = round_robin_with_weights(candidates_with_weight, department.id)

    # 6. Создаём assignment + ставим задачу в amoCRM
    a = create_assignment(lead, user=chosen, status='ASSIGNED', expires_at=now()+5min)
    await amo_set_responsible(lead.id, chosen.amo_user_id)
    await amo_create_task(lead.id, chosen.amo_user_id, '🔥 Звони клиенту сейчас', complete_till=now()+5min)
    chosen.last_assigned_at = now()
    return a


def round_robin_with_weights(candidates_with_weight, department_id):
    """
    Round-Robin кольцо где оранжевые получают каждого второго лида.
    Реализация: храним указатель в KV (Redis или таблица rr_state), идём по кругу.
    Для оранжевых (weight=0.5) — пропускаем через раз: храним счётчик 'отдано оранжевым',
    если sum_orange_given % 2 == 0 — текущий оранжевый пропускается, иначе выбирается.
    Сортировка кандидатов: сначала по last_assigned_at ASC (давно не получали — выше),
    потом по amo_user_id для детерминированности.
    """
```

---

## 8. Триггеры и события

| Источник | Событие | Endpoint роутера | Что делает |
|---|---|---|---|
| amoCRM webhook | `lead.status_changed` | `POST /api/v1/triggers/amo` | Проверяет: попадает ли в `Department.config.trigger` → match() |
| amoCRM webhook | `lead.responsible_changed` | `POST /api/v1/triggers/amo` | Если меняли руками → освобождаем старого, начисляем counter новому |
| Sipuni webhook | `call_started` (extension X) | `POST /api/v1/triggers/telephony` | `user.is_on_call=true`, если активен `Assignment` → `ENGAGED` (сброс таймера) |
| Sipuni webhook | `call_ended` | `POST /api/v1/triggers/telephony` | `user.is_on_call=false` |
| МегаФон | `call_started`/`call_ended` | то же | (см. открытый вопрос §11.B) |
| Контур API poll | `meeting_status=active` | (worker раз в N сек) | (см. открытый вопрос §11.B) |
| amoCRM widget | клик «Я свободен / занят» | `POST /api/v1/widget/self-status` | `user.manual_busy` toggle |
| cron 30 sec | (внутренний) | — | Worker сканит `QUEUED`, пробует match() |
| cron 60 sec | (внутренний) | — | Worker сканит `ASSIGNED` где `expires_at < now`, переводит в `TIMEOUT`, создаёт новый `PENDING` |
| cron hourly | (внутренний) | — | Worker пересчитывает `user_pipeline_state` для всех клоузеров (через amoCRM API GET leads) |
| cron 9:00 будни | (внутренний) | — | Worker берёт `DEFERRED` где `dispatch_after_ts <= now`, FIFO → match() |
| cron 10 мин | (внутренний) | — | Worker ищет `QUEUED` старше 10 минут → notify РОПа |

---

## 9. API endpoints

### 9.1 Внутренний (для админки)

```
GET    /api/v1/admin/departments                    список отделов
POST   /api/v1/admin/departments                    создать отдел
PATCH  /api/v1/admin/departments/{id}               обновить config
DELETE /api/v1/admin/departments/{id}               soft delete (active=false)

GET    /api/v1/admin/departments/{id}/users         клоузеры отдела
POST   /api/v1/admin/departments/{id}/users/sync    подтянуть из amoCRM
PATCH  /api/v1/admin/users/{id}                     active/manual_block toggle

GET    /api/v1/admin/departments/{id}/rules         все правила
POST   /api/v1/admin/departments/{id}/rules         создать
PATCH  /api/v1/admin/rules/{id}                     обновить config
DELETE /api/v1/admin/rules/{id}
PATCH  /api/v1/admin/rules/{id}/order               изменить order_idx

GET    /api/v1/admin/queue                          текущая очередь (QUEUED + DEFERRED)
POST   /api/v1/admin/queue/{assignment_id}/assign   ручное назначение РОПом
DELETE /api/v1/admin/queue/{assignment_id}          отменить (отправить в архив)

GET    /api/v1/admin/decisions                      лог решений матчера (с фильтрами)
GET    /api/v1/admin/users/{id}/load                сколько лидов сегодня / неделю получил
```

### 9.2 Для виджета клоузера в amoCRM

```
GET   /api/v1/widget/me                  мой статус, моя очередь, моя зона
POST  /api/v1/widget/self-status         { busy: true, until: '+30min' }
POST  /api/v1/widget/take-next           pull-режим: возьми следующего из очереди
GET   /api/v1/widget/team-status         светофор команды (для rank-badge)
```

### 9.3 Внешние webhooks (без auth, но IP whitelist)

```
POST  /api/v1/triggers/amo               от amoCRM
POST  /api/v1/triggers/sipuni            от Sipuni
POST  /api/v1/triggers/megafon           от МегаФон
```

### 9.4 «График работы операторов» — отдельная страница админки

**UX-эталон** (по референсу владельца — см. скриншот в чате 2026-05-14):

```
┌──────────────────────────────────────────────────────────────────────────┐
│ График работы операторов                                                 │
│ Настройка рабочих смен. Клик по ячейке — редактирование.                 │
│ В перспективе: звонки распределяются только агентам у которых сейчас     │
│ рабочий интервал.                                                        │
├──────────────────────────────────────────────────────────────────────────┤
│ ← [11.05 — 17.05 2026] →    [Сегодня]                                    │
├────────────────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬───────┤
│ ОПЕРАТОР       │ ПН   │ ВТ   │ СР   │ ЧТ   │ ПТ   │ СБ   │ ВС   │ ИТОГО │
│                │ 11.05│ 12.05│ 13.05│ 14.05│ 15.05│ 16.05│ 17.05│       │
├────────────────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┼───────┤
│ Антипов Дмитрий│ ─    │ ─    │ ─    │ ─    │09:00-│ ─    │ ─    │ 9 ч   │
│ ext 110        │      │      │      │      │18:00 │      │      │ 1 смена│
├────────────────┼──────┼──────┼──────┼──────┼──────┼──────┼──────┼───────┤
│ Брехт Данила   │ ─    │ ─    │ ─    │ ─    │ ─    │ ─    │ ─    │ 0 ч   │
│ ext 105        │      │      │      │      │      │      │      │ 0 смен │
└────────────────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┴───────┘
```

**Что делает РОП:**
- Открывает страницу `donula.online/router-admin/schedule`
- Видит таблицу: операторы × 7 дней недели (текущая неделя)
- Клик на ячейке → попап «Установить смену: 09:00 — 18:00» (или «Удалить смену»)
- Стрелки слева/справа — листать недели; кнопка «Сегодня» — вернуться
- ИТОГО справа = сумма часов + количество смен за неделю
- Цветовая подсветка: ячейка с активной сменой = зелёная; пустая = серая; СБ/ВС = красным заголовок

**Endpoints:**
```
GET   /api/v1/admin/schedules?department_id=X&week=2026-W20
      → [{user_id, weekday, start_time, end_time, active}, ...]

PUT   /api/v1/admin/schedules
      Body: {user_id, weekday, start_time, end_time}
      → создаёт или обновляет смену на день

DELETE /api/v1/admin/schedules/{user_id}/{weekday}
      → удаляет смену

GET   /api/v1/admin/overrides?user_id=X&date_from=...&date_to=...
      → отпуска/больничные в диапазоне

POST  /api/v1/admin/overrides
      Body: {user_id, date_from, date_to, reason, note}
      → добавить отпуск/больничный
```

**Дополнительно для виджета шапки клоузера:**
```
GET   /api/v1/widget/my-schedule
      → расписание ЭТОГО клоузера на текущую неделю (read-only)
```

Клоузер видит свой график в виджете (read-only), не может редактировать — это право РОПа.

---

## 10. Расширяемость на охотников и сопровождение

Дизайн полиморфен через `Department.config` и `Rule.kind`. Чтобы добавить новый отдел — НЕ нужны изменения схемы БД:

### 10.1 Охотники (воронка «Охотники», МПП1)

**Контекст:** охотники продают трипваер 99 ₽ холодному/тёплому трафику. Главный KPI — **конверсия `лид → оплата трипваера`**. После оплаты клиент уходит в МПП2 к клоузеру.

**Метрика зон** (см. §11.N с обоснованием выбора):
- 🟢 ≥ 25% конверсии — +20% к потоку лидов
- 🟡 18-25% — норма
- 🟠 12-18% — −30% к потоку
- 🔴 < 12% — 0 новых лидов (звонит только своим)

Окно расчёта: скользящие **7 дней** (сглаживает дневные колебания). Минимум **20 взятых лидов** в окне — иначе охотник считается «новичком» (зона = 🟡 по умолчанию).

```sql
INSERT INTO departments (code, title, config) VALUES (
  'hunters',
  'Охотники · воронка трипваера',
  '{
    "trigger": { "pipeline_id": <id_охотников>, "stage_id": <первичный_лид> },
    "reaction_window_sec": 300,
    "queue_alert_to_rop_sec": 600,
    "schedule": { "tz": "Europe/Moscow", "workdays": [1,2,3,4,5,6], "hours": [10, 22] },
    "zone_metric": "tripwire_conversion",
    "zone_thresholds": { "green": 25, "yellow": 18, "orange": 12 },
    "zone_window_days": 7,
    "zone_min_leads": 20,
    "zone_weights": { "green": 1.2, "yellow": 1.0, "orange": 0.7, "red": 0.0 }
  }'
);

-- Правила:
INSERT INTO rules (department_id, kind, config) VALUES
  (<hunters_id>, 'schedule', '{...из config...}'),
  (<hunters_id>, 'telephony_block', '{}'),
  (<hunters_id>, 'hunter_zone', '{"window_days": 7, "min_leads": 20}');  -- новый kind для охотников
```

**Что нужно докодить:**
- Новый `Rule.kind = 'hunter_zone'` — считает конверсию из leads, классифицирует в зону
- Worker hourly пересчитывает `user_hunter_metrics` (как `user_pipeline_state` у клоузеров)
- В таблице state: `leads_taken_7d`, `tripwires_sold_7d`, `conversion_pct`, `zone`

### 10.2 Сопровождение (воронка «Сбор документов»)

**Контекст:** после задачи «Передать в сопровождение» в этапе «Полная продажа» сделка переезжает в воронку «Сбор документов» → этап «Поступил в ОС». Триггер именно по этому переходу.

**Особенности отдела:**
- Не срочно (5 мин реакция не нужна) — клиент уже оплатил, должен ждать своего ОС
- Долгий цикл (недели-месяцы)
- Нет «конверсии в продажу» — нечего продавать
- Метрика для зон: качество работы (`avg_days_per_stage`, `% дел в просрочке`)

**Стратегия выбора: НЕ Round-Robin, а weighted_workload** — меньше открытых дел = больше шансов получить. Балансировка нагрузки, не равенство по входу.

```sql
INSERT INTO departments (code, title, config) VALUES (
  'support',
  'Сопровождение · ОС',
  '{
    "trigger": { "pipeline_id": <id_сбора_документов>, "stage_id": <поступил_в_ОС> },
    "reaction_window_sec": 7200,                   -- 2 часа (не срочно)
    "queue_alert_to_rop_sec": 14400,                -- 4 часа
    "schedule": { "tz": "Europe/Moscow", "workdays": [1,2,3,4,5], "hours": [10, 19] },
    "assignment_strategy": "weighted_workload",
    "zone_metric": "support_quality",
    "zone_thresholds": { "green_max_overdue": 0, "yellow_max_overdue": 2 }
  }'
);
```

**Что нужно докодить:**
- Новая `assignment_strategy = 'weighted_workload'` — сортирует кандидатов по `open_cases_count ASC` (меньше дел — выше шансы)
- Новый `Rule.kind = 'support_zone'` — считает зависшие/просроченные дела
- В таблице state: `open_cases`, `overdue_cases`, `avg_stage_days`, `zone`

### 10.3 Что общее для всех 3 отделов

- Полная схема БД (`departments`, `users`, `rules`, `assignments`, `routing_decisions`)
- FSM (PENDING → ASSIGNED → ENGAGED/TIMEOUT/QUEUED/DEFERRED)
- Алгоритм матчинга (фильтры → выбор → создание assignment + amoCRM task)
- Админка `donula.online/router-admin/` (CRUD для всех 3 отделов)
- amoCRM webhook receiver
- Воркеры (таймеры, очередь, агрегаты)

### 10.4 Что специфично для каждого отдела

| Аспект | clоузеры | hunters | support |
|---|---|---|---|
| Триггер | МПП2/Новый клиент | МПП1/первичный лид | СбДок/Поступил в ОС |
| Reaction window | 5 мин | 5 мин | 2 часа |
| Регулирование нагрузки | через Светофор (зона + веса) | через Светофор-охотник (конверсия 7d) | через weighted_workload (open_cases) |
| Стратегия выбора | Round-Robin | Round-Robin | Weighted по нагрузке |
| Метрика зон | % грехов сделок | конверсия трипваера 7d | overdue cases |
| Уведомление | amoCRM task | amoCRM task | amoCRM task (без срочности) |

### 10.3 Что нужно докодить для каждого нового типа отдела

- Если новые `Rule.kind` — добавить branch в `match()` функцию (это **расширение** функции, не изменение схемы)
- Если новая `assignment_strategy` (не Round-Robin) — добавить функцию в matchers/ модуль
- В админке — UI для редактирования config (можно начать с JSON-editor, потом сделать формы)

---

## 11. Открытые вопросы

Список того что **не однозначно** в исходных требованиях. Каждый требует решения владельца перед началом разработки.

### A. Расписание клоузеров ✅ ЗАКРЫТО 2026-05-14

**Решение:** **индивидуальный график на каждого**, настраивается в виджете админки. РОП заходит в «График работы операторов» (UI-мокап от владельца: таблица оператор × день недели, клик на ячейке = выставить интервал).

**Дизайн БД:**
```sql
CREATE TABLE user_schedules (
  id            SERIAL PRIMARY KEY,
  user_id       INTEGER REFERENCES users(id),
  weekday       SMALLINT NOT NULL,           -- 1=пн ... 7=вс
  start_time    TIME NOT NULL,                -- '09:00:00'
  end_time      TIME NOT NULL,                -- '21:00:00'
  active        BOOLEAN DEFAULT true,
  UNIQUE(user_id, weekday)
);

-- Отпуска / больничные / отгулы (overrides)
CREATE TABLE user_schedule_overrides (
  id            SERIAL PRIMARY KEY,
  user_id       INTEGER REFERENCES users(id),
  date_from     DATE NOT NULL,
  date_to       DATE NOT NULL,
  reason        TEXT,                          -- 'vacation' | 'sick' | 'personal'
  note          TEXT,
  created_at    TIMESTAMPTZ DEFAULT NOW()
);
```

**Алгоритм проверки «на смене сейчас»:**
1. Проверить overrides на сегодня → если есть, клоузер не на смене
2. Проверить `user_schedules` для текущего weekday → если есть запись и `now ∈ [start_time, end_time]` → на смене
3. Иначе → не на смене (вне расписания)

**UI «График работы операторов»** (см. §9.4 ниже).

### E. Формула % красных ✅ ЗАКРЫТО — CANONICAL новая

**Решение:** **canonical = новая формула.** Подтверждено владельцем 2026-05-14:

```
% грехов = (🔴 + 0.5 × 🟠) / общее_число_сделок_в_работе × 100
```

Знаменатель = только активные сделки клоузера (этапы «Думает», «Обещал оплатить», «Оплатил частичку», «Полная продажа» в МПП2). Архивные, на паузе, неактивные — НЕ считаются.

**Пороги зон:**
| Зона | % грехов | Поток новых лидов | Round-Robin вес |
|---|---|---|---|
| 🟢 Зелёная | **ровно 0%** | 100% | 1.0 |
| 🟡 Жёлтая | 0% < x ≤ 10% | 100% (+ флаг РОПу) | 1.0 |
| 🟠 Оранжевая | 10% < x ≤ 20% | **50%** | 0.5 |
| 🔴 Красная | > 20% | **0%** (исключён из пула) | 0.0 |

**⚠ Расхождение с regulation:** старая формула в [[regulation-цвета-сделок-и-зоны-клоузера]] (🟢≤10% / 🟡10-20% / 🟠20-30% / 🔴>30%, формула без коэффициента 0.5 для оранжевых) теперь **deprecated**. Regulation должен быть переподписан командой с новой формулой. Я отметил это в самом regulation файле.

### F. Канал уведомления «🔥 Звони сейчас» ✅ ЗАКРЫТО

**Решение:** **amoCRM task + виджет шапки**.

- **amoCRM task** ставится через `POST /api/v4/leads/{id}/tasks` с `responsible_user_id=клоузер`, `complete_till=now+5min`, текст «🔥 Звони клиенту сейчас». Это нужно для отчётности и SLA-контроля.
- **Виджет шапки** показывает огненный pill «🔥 Новый лид: Петров К.» с обратным таймером 5:00 → 0:00. Клик = открывает карточку сделки в amoCRM, **НЕ сбрасывает таймер**. Сброс таймера только реальным `Sipuni call_initiated` (см. §11.C).

TG не используем — все коммуникации внутри amoCRM (решение из roadmap-внедрения-системы-дожима-сделок 2026-05-13).

### K. Smoke test реальности — что нужно сделать в amoCRM ✅ ЗАКРЫТО

**Решение:** «что надо — сделаем». Чек-лист подготовительных действий на стороне amoCRM:

**Custom fields на сделке (МПП2):**
- `cf_router_assignment_id` — id назначения в роутере (для трекинга)
- `cf_was_revived` — для статистики, выставляет Salebot при реанимации
- `cf_router_attempts` — сколько раз переназначался

**Custom fields на пользователе:**
- `cf_user_zone` — `green` / `yellow` / `orange` / `red` (роутер пишет hourly из агрегата). Используется виджетом шапки и для аудита.
<!-- `cf_user_active_count` — был для отображения лимита, но capacity_limit отменён 2026-05-15 (см. §11.I). Не создаём. -->

**Группы пользователей (amoCRM):**
- Создать группу «Клоузеры» — список user_id для синка с `users` таблицей роутера
- Создать группу «Охотники» — то же для отдела hunters
- Создать группу «ОС» — для сопровождения
- Создать группу «РОПы» (или один user marked is_admin) — для прав в админке

**Этапы воронок (нужны точные stage_id):**
- МПП2: «Новый клиент» (триггер), «Думает», «Обещал оплатить», «Оплатил частичку», «Полная продажа» (для расчёта знаменателя % грехов в светофоре, см. §11.I)
- МПП1 («Охотники»): первичный лид, оплачен трипваер (для конверсии)
- «Сбор документов»: «Поступил в ОС» (триггер для сопровождения)

**Webhooks:** настроить amoCRM → POST в `donula.online/router/api/v1/triggers/amo` на события `lead.status_changed` для всех 3 воронок.

### L. PostgreSQL vs SQLite ✅ ЗАКРЫТО

**Решение:** начинаем с **SQLite + WAL mode** (как amo-partner-bridge). Если упрёмся в конкуренцию write'ов — мигрируем на Postgres через SQLAlchemy без боли (ORM не меняется, только URL подключения).

**Почему SQLite ОК для MVP:**
- Объём ~200-500 лидов/день = ~5-10 write'ов на dispatch + worker'ы
- WAL mode даёт read-write конкуренцию (читатели не блокируют писателя)
- Деплой проще: один файл `data/router.db` рядом с бинарём, бэкап через `rsync`

**Когда мигрируем на Postgres:** если QPS > 20 write/sec в пиках, или если потребуется JSON-индексирование (Postgres JSONB).

### M. Права РОПа в админке ✅ ЗАКРЫТО

**Решение:** многоуровневая модель доступа от простого к сложному.

**MVP (Фаза 1):**
- Авторизация через amoCRM OAuth (тот же tenant donulya2)
- В `.env` хардкод: `ROUTER_ADMIN_USER_IDS=12345,67890` (amoCRM user_id РОПов)
- Любой залогиненный РОП = полный доступ к админке

**Phase 5 (расширенная админка):**
- Таблица `permissions` (user_id × resource × action)
- Resources: `departments`, `users`, `rules`, `assignments`, `analytics`
- Actions: `read`, `write`, `delete`
- РОП клоузеров может править только department='closers', не трогает hunters/support
- Можно создать роль «Старший РОП» с полным доступом

### B. Источник «занят сейчас» ✅ ЗАКРЫТО 2026-05-14

**Решение владельца:** в идеале — автоматика (webhook от телефоний), но т.к. интеграции с 3 источниками = много работы, **MVP делаем на ручных статусах клоузера** в виджете шапки amoCRM: 🟢 «Свободен» / 📞 «На линии» / ☕ «Перерыв».

**Дизайн гибрид (готов к подключению автоматики позже):**
- `user_availability_state.manual_status` enum: `available` / `on_call` / `break` — задаёт сам клоузер
- `user_availability_state.auto_status` — заполняется webhook'ом от Sipuni когда подключим (опционально)
- Effective busy = `manual_status != 'available' OR auto_status = 'on_call'`

В виджете шапки добавим контрол (toggle 3 кнопки), позволяющий клоузеру быстро переключаться. UX: при выборе «На линии» / «Перерыв» — спросить «На сколько?» (15/30/60 мин или до ручного снятия). Когда time-up — авто-возврат в `available`.

В будущем когда подключим Sipuni — в матчере проверяем оба поля. Manual всё ещё работает (клоузер может пометить «перерыв» в любой момент).

### C. Что считать «звонил» для сброса 5-мин таймера ✅ ЗАКРЫТО · 2026-05-15 пересмотрено

**Решение:** **ТОЛЬКО факт исходящего звонка через Sipuni** (`call_started` или `call_initiated` webhook на номер лида). Без длительности — главное чтобы клоузер начал работать с лидом за 5 минут.

**⚠ Важное уточнение от директора 2026-05-15:** ранее в спеке был fallback «кнопка в виджете "Я взял" если Sipuni не интегрирована» — это **отменено**. Кнопка в виджете создавала **лайфхак для обхода системы**: клоузер мог нажать → таймер сброшен → отчётность ОК, при этом реально не позвонить. Особенно опасно для метрики «время до первого касания клиента».

**Новая модель поведения 🔥 pill в виджете:**

- Клик на 🔥 pill = **открывает карточку сделки в amoCRM** в новой вкладке (как обычный клик на плашку сделки в развёрнутой части)
- **Таймер НЕ сбрасывается** при клике на pill — это просто навигация «иди работай с этим лидом»
- Таймер сбрасывается **только реальным `call_initiated` от Sipuni**

Это значит:
- Клоузер видит pill → жмёт → попадает в карточку amoCRM → должен набрать номер
- Если набрал (Sipuni шлёт webhook) → таймер сброшен, статус `ENGAGED`
- Если не набрал за 5 мин (был в YouTube или забыл) → лид уходит другому, **даже если карточку открывал**

**Sipuni обязательна с Фазы 1.** Без неё запускать роутер нет смысла — главная метрика «реальное касание < 5 мин» без Sipuni не верифицируется.

«Зачёт касания» для светофора сделки — это **другая метрика** (≥90 сек + продвижение в CRM), считается отдельно AI-пайплайном.

### D. Round-Robin vs «давно не получал» ✅ ЗАКРЫТО

**Решение:** Вариант А — **чистый Round-Robin** с FIFO-указателем.

Реализация: сортировка кандидатов по `last_assigned_at ASC` (давно не получал — выше). При равенстве — по `amo_user_id` для детерминированности. Берём первого. Это даёт равномерность естественно, без random'а.

🟠 (вес 0.5) — пропуск раз через раз. Храним в таблице `rr_state` счётчик «отдано оранжевым» per department. Если sum_orange_given чётное — текущий 🟠 пропускается, лид идёт следующему по очереди (не оранжевому). Если нечётное — берём.

### E. Формула «% красных»

**КОНФЛИКТ с regulation.**
- В [[regulation-цвета-сделок-и-зоны-клоузера]] (актуальный): `% красных = (🔴 / все_активные) × 100`, пороги 🟢≤10% / 🟡10-20% / 🟠20-30% / 🔴>30%
- В этом запросе: `% красных = (🔴 + 0.5×🟠) / все_активные × 100`, пороги 🟢=0% / 🟡 0-10% / 🟠 10-20% / 🔴 >20%

Разница серьёзная — нужно подписать какая версия canonical. Если новая, regulation требует ревизии (он подписывался командой).

### F. Канал уведомления «🔥 Звони сейчас»

amoCRM task (отображается в карточке сделки) или дополнительно TG-пуш или виджет в шапке?
**Моё предложение:** amoCRM task (responsible_user=клоузер) + видимая push-notification в виджете шапки. TG не нужен — usability через виджет.

### G. Раздача ночной очереди утром ✅ ЗАКРЫТО

**Решение:** Pull-режим (вариант С). Контекст: за выходные приходит ~5 лидов, часть набирают сами клоузеры до понедельника. Не нужно push-распределение.

В 9:00 рабочего дня worker не назначает автоматически. В виджете шапки появляется пилюля «🔥 В очереди для тебя: N. Взять следующего →». Клик → роутер делает обычный `match()` (выбирает из очереди FIFO самый старый `DEFERRED`), назначает этому клоузеру, запускает 5-мин таймер.

В дневном режиме — push (роутер сам назначает на webhook от amoCRM).

### H. Возврат «закрыто/нереализовано» ✅ ЗАКРЫТО

**Решение:** реанимация **автоматическая через Salebot**, не cron в роутере.

Процесс:
1. Salebot периодически шлёт триггеры в архивные сделки (со своими условиями)
2. Клиент отвечает → Salebot закрывает на созвон → клиент даёт согласие
3. Salebot создаёт **новый лид** в МПП2, этап «Новый клиент»
4. amoCRM webhook → роутер видит обычное событие как §3.1 → назначает **свежего клоузера**
5. Старый клоузер игнорируется (даже если был в истории по этому контакту)

Жалоба клиента «верните Анну» — единичный случай, **РОП обрабатывает руками** через админку (`PATCH /api/v1/admin/assignments/{id}/user`).

**Что нужно от роутера:** ничего специального. Это просто обычный новый лид. Если хочется отличать в логах — Salebot может добавлять note или custom_field `was_revived=true` для статистики.

### I. Лимит активных сделок ✅ ОТМЕНЁН 2026-05-15

**Решение:** **жёсткий лимит количества сделок отменён**. Регулирование нагрузки происходит **только через Светофор** (зона клоузера по % грехов).

**Почему так:**

Изначально была идея «лимит 40 активных сделок — больше не давать». Но это **наказывает трудолюбивых**:

- Боря с 41 🟢-сделкой (все дожимает, портфель чистый) → попал бы под лимит → не получает лидов
- Аня с 38 сделок где 5 🔴 (забросила половину) → не достигла лимита → получает лидов и хоронит ещё

Это **переворачивает мотивацию**: выгоднее держать маленький грязный портфель чем большой чистый.

**Светофор сам становится регулятором:**

- Боря набрал 60 сделок → если справляется (все 🟢/🟡) → зона остаётся 🟢 → дальше получает
- Если не успевает дожимать → начинают появляться 🟠 → 🔴 → его % грехов растёт → переходит в 🟠 зону → автоматически получает только 50% потока → может догнать
- Если совсем завалил → 🔴 зона → 0 новых лидов → пока не разгребёт

Это **самокорректирующаяся система с одним контуром обратной связи** (вместо двух — лимит и светофор).

**Что убрано из дизайна:**
- ❌ Поле `departments.config.capacity_limit` — больше не используется
- ❌ Правило `kind='capacity'` — удалено из списка `Rule.kind`
- ❌ Custom field `cf_user_active_count` на пользователе — не нужен (не показываем лимит)
- ❌ Фильтр в `match()` `WHERE active_count <= 40` — удалён

**Что осталось из дизайна:**
- ✅ Светофор клоузера (зона + веса) — единственный регулятор нагрузки
- ✅ `user_pipeline_state` таблица — продолжаем считать active_count / red_count и т.д. для отображения в виджете и для расчёта % грехов
- ✅ Все 4 этапа МПП2 («Думает», «Обещал», «Частичка», «Полная продажа») остаются в знаменателе формулы % грехов

**Минус варианта (зафиксирован):** теоретический stamp-overload в первые часы — клоузер может набрать 100+ сделок до того как первые начнут протухать. Защита: hourly worker пересчитывает зону, через ~24-36 часов первые забытые сделки уйдут в 🟡/🟠 → автоматическое снижение потока. Если этот сценарий станет проблемой в реальной работе — добавим **rate-limit на частоту назначений** (например, max 1 лид в 5 минут одному клоузеру). **Сейчас не делаем** — добавим только если увидим проблему в production.

### J. Профиль будущих отделов ✅ УТОЧНЕНО

**Охотники (МПП1)** — раскрыто отдельно: см. §10.1 и §11.N (метрика зон для охотников).

**Сопровождение (ОС)** — раскрыто: триггер = переход в воронку «Сбор документов», этап «Поступил в ОС». См. §10.2 и §11.O.

### K. Smoke test реальности — что уже есть в amoCRM

- Поле `closer.svetofor_zone` — реальное поле в amoCRM или агрегат у нас? **Гипотеза:** агрегат у нас (`user_pipeline_state`), пишем обратно в amoCRM custom_field раз в час для виджета.
- Список клоузеров — берём через amoCRM API `/api/v4/users` + фильтр по группе?
- Расписание сменности — где сейчас живёт? В заметке РОПа? Если нет места — будем хранить у себя.

### L. PostgreSQL vs SQLite

**Конфликт:** amo-partner-bridge на SQLite, я предлагаю PostgreSQL для роутера.
- Причина: конкурентные write'ы в `assignments` и `queue` (Workers + webhook handler одновременно)
- Альтернатива: SQLite с WAL mode — для нашего объёма (200-500 лидов/день) хватит, и проще деплой.
- Решение: начнём с SQLite, при нагрузке мигрируем на Postgres. Это допустимо благодаря SQLAlchemy.

### M. Хранение прав РОПа (кто может менять правила)

Авторизация в админке через amoCRM OAuth — берём `amo_user_id` + проверяем `is_admin_group`. Если РОП один — захардкодим список user_id. Если несколько — добавим `permissions` таблицу.

### N. Метрика зон для охотников ✅ РЕКОМЕНДАЦИЯ 2026-05-14 (ожидает подтверждения)

**Контекст вопроса:** владелец предложил 4 варианта метрики для зон охотников: (A) конверсия лид→трипваер, (B) кол-во продаж/день, (C) качество звонка AI, (D) комбинация. Какой использовать?

**Моя рекомендация: вариант A — конверсия за скользящие 7 дней, с защитами**

#### Почему конверсия лучше «кол-ва продаж в день»

Пример:
- **Аня:** 3 продажи в день (план), взяла 60 лидов = **5% конверсия**
- **Боря:** 2 продажи в день (минус 33%), взял 5 лидов = **40% конверсия**

По «кол-ву продаж» Аня = 🟢, Боря = 🟡. **На самом деле Боря в 8× эффективнее**. Система перевернёт мотивацию: «бери побольше лидов и хоть 3 продай — будут хвалить».

Конверсия = справедливая метрика: не зависит от потока, измеряет качество работы охотника.

#### Почему конверсия лучше «качества звонка AI»

AI может оценивать как охотник говорит, но не отражает **результат**. Можно быть вежливым роботом по скрипту и не закрывать. Конверсия отражает результат напрямую.

AI остаётся как **корректирующий слой** (см. ниже).

#### Конкретная конфигурация

**Окно расчёта:** скользящие **7 дней** (не сегодня) — сглаживает дневные колебания (плохой день / случайно горячий лид).

**Стартовые пороги:**
| Зона | Конверсия | Поток новых лидов |
|---|---|---|
| 🟢 Зелёная | ≥ 25% | **+20%** (получает больше) |
| 🟡 Жёлтая | 18% — 25% | норма |
| 🟠 Оранжевая | 12% — 18% | **−30%** (получает меньше, лиды дороже) |
| 🔴 Красная | < 12% | **0** (звонит только своим текущим) |

Пороги нужно **калибровать на реальных данных** через 2-3 недели работы — посмотрим распределение по команде и сдвинем.

**Защиты от манипуляций:**

1. **Минимум 20 взятых лидов** за окно — иначе зона по умолчанию 🟡 (новичкам шанс)

2. **AI-проверка качества разговора:** если конверсия 🟢 но средний AI-балл < 5/10 — зона не повышается выше 🟡. Защищает от «выезда на инерции маркетинга»: были горячие лиды, охотник тупо взял трубку, клиенты сами купили.

3. **Отказы засчитываются:** если охотник может «отложить» лида (не взять) — считаем как «взял и не продал». Иначе будут отбирать только горячих.

#### План продаж — отдельная мотивационная метрика

«3 продажи в день» — **не для роутера**, для **премии РОПа**. Чтобы мотивировать охотника не «зависать» на сложных и брать больше попыток. Влияет на зарплату, не на распределение.

#### Что нужно от amoCRM

- Поле сделки `responsible_user_id` — кто из охотников взял
- Поле сделки `status` или этап = «оплачен трипваер» — для конверсии
- Возможность фильтровать `WHERE assigned_to=X AND created_at BETWEEN now-7d AND now` через `/api/v4/leads`

### O. Сопровождение — детали ✅ УТОЧНЕНО 2026-05-14

**Триггер:** задача «Передать в сопровождение» в этапе «Полная продажа» (МПП2). Эта задача автоматически переводит сделку в воронку «Сбор документов» / этап «Поступил в ОС».

**Webhook на роутер:** `lead.status_changed` где `pipeline_id=<сбор документов>` AND `stage_id=<поступил_в_ОС>`. То есть роутер видит сделку только когда она УЖЕ в новой воронке.

**Что не делаем:** не сидим на МПП2 и не слушаем «задача создана». Слушаем только finalный переход в ОС-воронку.

**Особенности отдела (vs клоузеры):**
- Не Round-Robin а **weighted_workload**: кандидаты сортируются по `open_cases ASC` (меньше открытых дел — выше шансы). Балансировка нагрузки.
- Reaction window: **2 часа** (не 5 мин — клиент уже оплатил, не горит)
- Capacity: **80** (больше чем у клоузеров, потому что цикл дольше)
- Светофор: не % красных а % просроченных дел (зависшие > N дней)

Метрики зон ОС — нужно отдельное обсуждение когда будем подключать сопровождение в Фазе 6+. Для MVP — можно начать БЕЗ зон (все равны, weight=1.0).

---

## 12. План реализации

### Фаза 0 · Согласование (1-2 дня)
- [ ] Владелец отвечает на §11 A-M
- [ ] Подписываем canonical формулу % красных и пороги зон
- [ ] Фиксируем какие телефонии и в каком статусе интеграции

### Фаза 1 · MVP «один отдел, один триггер» (3 недели · с Sipuni)
- [ ] `donulya-router` репо на GitHub (private)
- [ ] FastAPI + SQLite (WAL) + Alembic, основные таблицы (departments, users, schedules, schedule_overrides, assignments, rules, decisions, availability_state, pipeline_state)
- [ ] amoCRM custom fields подготовлены (см. §11.K)
- [ ] Группа «Клоузеры» в amoCRM, sync user_id с роутером
- [ ] amoCRM webhook receiver → match() для отдела closers
- [ ] Алгоритм матчинга: schedule + manual_busy + light_zone(новая формула, новые пороги) + Round-Robin (без capacity-фильтра — см. §11.I)
- [ ] amoCRM task creation на ASSIGNED («🔥 Звони клиенту сейчас», complete_till=+5min)
- [ ] Worker для 5-мин таймера (TIMEOUT → reassign, без штрафа первому)
- [ ] Worker hourly: пересчёт `pipeline_state` (включая `% грехов` по новой формуле)
- [ ] Виджет шапки клоузера: 🔥 pill «Новый лид → Взять» + toggle статуса 🟢🔴☕
- [ ] Минимальная админка (`donula.online/router-admin/`, Vite + React + shadcn):
  - Логин через amoCRM OAuth
  - Таблица отделов (read-only пока 1 отдел)
  - Таблица клоузеров с on/off
  - **График работы операторов** (см. §9.4) — основной UI настройки
  - Текущая очередь + ручное назначение РОПом
- [ ] **Sipuni-интеграция** (перенесена из бывшей Фазы 2):
  - Webhook receiver `POST /api/v1/triggers/sipuni`
  - На `call_started` от клоузера X на номер лида Y → найти активный
    `Assignment(user=X, lead=Y, status=ASSIGNED)` → перевести в `ENGAGED`,
    сбросить таймер. Это **единственный** способ сброса таймера.
  - На `call_started`/`call_ended` обновлять `user_availability_state.auto_status`
    (для будущей автоматики статуса 📞 «На линии»)
- [ ] Деплой на 87.228.88.163, nginx маршрут `donula.online/router/*` → 127.0.0.1:8444

### Фаза 2 · Доп. телефонии и видеовстречи (3-5 дней)
- [ ] МегаФон webhook (если есть параллельная телефония) — приоритет за Sipuni, МегаФон fallback
- [ ] Контур (видеовстречи) — sync через ical (worker 1 раз в 5 мин) или Kontur API

### Фаза 3 · Очередь + ночной режим (3-5 дней)
- [ ] DEFERRED статус + 9:00 cron
- [ ] QUEUED + worker 30-sec retry
- [ ] Виджет в шапке amoCRM: pull-кнопка «Взять следующего»
- [ ] Уведомление РОПу при `queue_age > 10мин`

### Фаза 4 · Hourly агрегаты + Светофор (3-5 дней)
- [ ] Worker hourly: GET leads из amoCRM → пересчёт user_pipeline_state
- [ ] Write обратно в amoCRM custom_field зоны клоузера
- [ ] Endpoint для виджета шапки (donulya-ai-auditor)

### Фаза 5 · Полноценная админка (1-2 недели)
- [ ] CRUD правил (drag-drop порядок, JSON-editor для config)
- [ ] Лог решений с фильтрами (по дате / отделу / клоузеру)
- [ ] Аналитика: загрузка по клоузерам, среднее время реакции, частота TIMEOUT
- [ ] Аудит изменений правил

### Фаза 6 · Расширение на охотников (1 неделя)
- [ ] Создаём department='hunters' через админку
- [ ] Тестируем что общий код работает без правок
- [ ] Если нужны новые `Rule.kind` (например, tripwire_score) — добавляем обработчик в match()

---

## 13. Чек-лист готовности к запуску в прод

- [ ] §11 закрыт (все открытые вопросы)
- [ ] Регламент в [[regulation-цвета-сделок-и-зоны-клоузера]] синхронизирован с canonical формулой
- [ ] Интеграция с Sipuni работает в стейджинге (хотя бы dry-run на 24 ч)
- [ ] План отката: если робот сходит с ума, как быстро РОП вернёт ручное распределение? Кнопка «Disable Router» в админке → все новые лиды без `responsible_user_id` → РОП назначает руками.
- [ ] Лог `routing_decisions` пишется в каждый матч (для post-mortem споров «почему именно мне дали»)
- [ ] Метрики dashboard: queue_size, avg_dispatch_time, timeout_rate, top_loaded_closer — на отдельной странице админки или в Grafana

---

## Следующие шаги

1. **Ревью этого документа** владельцем
2. **Ответы на §11** (открытые вопросы)
3. **Решение о расхождении regulation** — пересмотреть пороги или оставить старые
4. **Создание репо** `donulya-router` (private)
5. **Старт Фазы 1** с MVP

Когда стейк-холдеры (владелец + РОП) подпишут эту спеку — переходим к коду.
