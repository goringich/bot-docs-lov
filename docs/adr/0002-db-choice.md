# ADR 0002 - DB choice

## Status
accepted

## Context
Нужна "обычная" реляционная БД уровня MySQL. Требуются индексы, транзакции, и быстрые выборки под экспорт Excel. В перспективе 20+ чатов и большие объемы сообщений.

## Decision
- Используем MySQL 8.x (локально через Docker).
- Драйвер SQLAlchemy: mysql+pymysql.
- Для MySQL auth caching_sha2_password используем зависимость cryptography в venv.

## Consequences
- Простое локальное поднятие через docker compose.
- Для подключения требуется cryptography (иначе PyMySQL не пройдет auth).
- В будущем можно заменить драйвер на mysqlclient или перейти на PostgreSQL без смены общей архитектуры.

