## Отчет по отладке приложения Жетписов Ансат Нурланович

### 1. Шаг 1: Анализ и запуск
* Что сделал: docker compose up --build
* Ожидание: приложение стартует, применяет миграции и поднимает API
* Факт: приложение не сможет корректно подключиться к БД, даже если .env правильный

* Проблема: Описание проблемы: в настройках опечатка в validation_alias, из-за чего переменная окружения DATABASE_URL не подхватывается, и приложение всегда использует дефолтное значение.
![alt text](assets/1.png)
* Решение: Исправить опечатку.

Вложение

Код до:
>config.py
```python
database_url: str = Field(
    "postgresql+asyncpg://postgres:postgres@db:5432/postgres_typo",
    validation_alias="DATABSE_URL",
)
```
Код после:
>config.py
```
database_url: str = Field(
    "postgresql+asyncpg://postgres:postgres@db:5432/postgres_typo",
    validation_alias="DATABASE_URL",
)
```
![alt text](assets/2.png)
Приложение запустилось и начался парсинг

### 2. Шаг 2: Исправление бага №2
* Что сделал: Исправляя конфиг заметил разницу

* Проблема: Даже если .env не применился/отсутствует, дефолтный DSN должен быть рабочим. Сейчас он ведёт в базу postgres_typo, которой нет (в compose создаётся POSTGRES_DB=postgres)
* Решение: Исправить DSN в конфиге чтобы он соответствовал env

Вложение

Код до:
>config.py
```python
    database_url: str = Field(
        "postgresql+asyncpg://postgres:postgres@db:5432/postgres_typo",
        validation_alias="DATABASE_URL",
    )
```
>.env
```
DATABASE_URL=postgresql+asyncpg://postgres:postgres@db:5432/postgres
```

Код после:
>config.py
```python
    database_url: str = Field(
        "postgresql+asyncpg://postgres:postgres@db:5432/postgres",
        validation_alias="DATABASE_URL",
    )
```

### 3. Шаг 3: Исправление бага №3
* Что сделал: Запуская приложение заметил, что парсинг осуществляется каждые 5 секунд, когда должен каждые 5 минут так как видел в env переменную `PARSE_SCHEDULE_MINUTES`

* Проблема: Планировщик APScheduler настроен на seconds=..., но переменная называется PARSE_SCHEDULE_MINUTES и по ТЗ интервал именно в минутах

* Решение: Открыть шедулер и поменять seconds на minutes

Вложение

Код до:
>scheduler.py
```python
def create_scheduler(job: Callable[[], Awaitable[None]]) -> AsyncIOScheduler:
    scheduler = AsyncIOScheduler()
    scheduler.add_job(
        job,
        trigger="interval",
        seconds=settings.parse_schedule_minutes,
        coalesce=True,
        max_instances=1,
    )
    return scheduler
```

Код после:
>scheduler.py
```python
def create_scheduler(job: Callable[[], Awaitable[None]]) -> AsyncIOScheduler:
    scheduler = AsyncIOScheduler()
    scheduler.add_job(
        job,
        trigger="interval",
        minutes=settings.parse_schedule_minutes,
        coalesce=True,
        max_instances=1,
    )
    return scheduler
```
Парсинг перестал лететь каждые 5 секунд

### 4. Шаг 4: Исправление бага №4

* Что сделал: Когда исправлял предыдущие баги, столкнулся с тем,
что в логах постоянно вылетала одна и та же ошибка
![alt text](image.png)

* Проблема: D данных city бывает null, а код делает item.city.name.strip() → AttributeError: 'NoneType' ... Это прямо ломает критерий “парсинг без падений”

* Решение: Добавить проверку

Вложение

Код до:
parser.py
```python
"city_name": item.city.name.strip(),
```

Код после:
parser.py
```python
"city_name": item.city.name.strip() if item.city else None,
```
Теперь ошибка не возникает
![alt text](image-1.png)

### Итог