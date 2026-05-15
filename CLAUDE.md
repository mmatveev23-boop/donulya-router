# donulya-router — карта для AI-агента

> Это карта проекта для новой нейронки которая открыла репо без контекста. Прочитай эту страницу первой — даст быстрое понимание что делать.

## Что это

`donulya-router` — бэкенд-сервис распределения лидов трипваеров между клоузерами в воронке amoCRM МПП2. Связан с виджетом «🚦 Касания» дисциплины (отдельный проект [donulya-ai-auditor](https://github.com/mmatveev23-boop/donulya-ai-auditor)) — зона клоузера по светофору влияет на распределение.

**Главная фишка:** время от оплаты 99 ₽ до первого касания клоузера — менее 5 минут в рабочее время. Если клоузер не успел — лид уходит следующему без штрафа.

## Состояние

**Pre-MVP. Кода ещё нет.** Документация подготовлена, прогер берёт `README.md` для введения + `docs/SPEC.md` для технической реализации.

## Стек (планируется)

- **Backend:** Python 3.12 + FastAPI + SQLAlchemy + Alembic + asyncio workers + loguru + httpx
- **DB:** SQLite (WAL mode) для MVP, миграция на PostgreSQL при росте нагрузки (через тот же SQLAlchemy)
- **Frontend admin:** React + Vite + shadcn/ui, авторизация через amoCRM OAuth (donulya2 tenant)
- **Деплой:** systemd на `87.228.88.163` (тот же сервер где amo-partner-bridge), nginx маршрут `donula.online/router/*` → 127.0.0.1:8444 (новый порт)

## Структура

```
donulya-router/
├── README.md            ← простыми словами, для введения
├── CLAUDE.md            ← этот файл (карта для нейронки)
├── .gitignore
└── docs/
    ├── SPEC.md          ← полная техническая спека (~750 строк)
    ├── REGULATION.md    ← регламент дисциплины
    └── WIDGET.md        ← где смотреть UI-мокап
```

## Перед началом разработки

Все ответы есть в `docs/SPEC.md §11` — 13 закрытых вопросов с обоснованиями. Перечитай его перед написанием кода.

Что нужно подготовить (силами владельца / РОПа):

1. **amoCRM:**
   - Custom fields на сделке (`cf_router_assignment_id`, `cf_was_revived`, `cf_router_attempts`)
   - Custom field на пользователе (`cf_user_zone`) — `cf_user_active_count` НЕ нужен, лимит активных отменён (см. SPEC §11.I)
   - Группы пользователей: «Клоузеры» / «Охотники» / «ОС» / «РОПы»
   - Webhook на `lead.status_changed` на наш endpoint
   - Точные pipeline_id и stage_id (см. SPEC §11.K)

2. **OAuth приложение** в amoCRM Marketplace для авторизации админки (donula.online/router-admin/)

3. **Переподписка регламента** командой по новой формуле `(🔴 + 0.5×🟠) / всего × 100` с новыми порогами (см. SPEC §11.E)

## Связанные проекты

- **donulya-ai-auditor** — виджет дисциплины и дашборд РОПа (https://github.com/mmatveev23-boop/donulya-ai-auditor). UI роутера встроен в `closer-top-bar.html` (🔥 pill «Новый лид» + status-toggle + queue-pull). Live: https://mmatveev23-boop.github.io/donulya-ai-auditor/closer-top-bar.html
- **amo-partner-bridge** — бридж amoCRM (https://github.com/mmatveev23-boop/donula-partnerka). НЕ роутер, но тот же сервер `87.228.88.163`. Можно взять оттуда наработки по `amo_client.py`, OAuth, webhook handlers — переиспользовать паттерны (но не код напрямую).

## Стиль кода

При написании кода придерживаться правил amo-partner-bridge:
- `from __future__ import annotations` обязательно
- `loguru` для логов (не `logging`)
- `httpx` для HTTP
- Type hints на публичных функциях
- Короткие функции, плоская вложенность
- `pytest` для тестов

## Знание

Если ты Claude и тебе нужно больше контекста — пользователь использует Knowledge Vault в Obsidian (iCloud путь):

```
~/Library/Mobile Documents/com~apple~CloudDocs/CLAUD/knowledge-vault/Projects/donulya-ai-auditor/
├── spec-lead-router.md                        ← основной spec (этот же в docs/SPEC.md)
├── regulation-цвета-сделок-и-зоны-клоузера.md ← regulation
├── router-логика-простыми-словами.md          ← простые шаги (этот же в README.md)
├── decisions.md                                ← архитектурные решения
└── ...
```

Если ты другая нейронка без доступа к iCloud — `docs/SPEC.md` + `docs/REGULATION.md` + `README.md` достаточно. Все ключевые решения там же.

## Контакты

- **Владелец:** Максим Матвеев (mmatveev23@gmail.com)
- **GitHub:** @mmatveev23-boop
