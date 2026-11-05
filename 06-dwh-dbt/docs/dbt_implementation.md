# Реализация аналитического хранилища с использованием dbt

## 1. Назначение dbt

**dbt (Data Build Tool)** используется для преобразования данных внутри аналитического хранилища.

В рамках проекта dbt отвечает за:

- Создание SQL-моделей на основе исходных данных.
- Организацию зависимостей между моделями.
- Централизацию бизнес-логики.
- Проверку качества данных.
- Инкрементальное обновление аналитических таблиц.
- Документирование моделей и их полей.

Первоначальная загрузка данных из PostgreSQL в Raw выполняется отдельным инструментом ingestion. dbt начинает работу после загрузки исходных данных.

Выбранный подход — ELT (Extract, Load, Transform): сначала данные загружаются в Raw, затем преобразуются внутри DWH.

## 2. Структура проекта

Предлагаемая структура dbt-проекта:

```text
dbt_project/
├── dbt_project.yml
├── models/
│   ├── staging/
│   │   └── ecommerce/
│   │       ├── sources.yml
│   │       ├── stg_customers.sql
│   │       ├── stg_products.sql
│   │       ├── stg_orders.sql
│   │       ├── stg_order_items.sql
│   │       ├── stg_order_status.sql
│   │       └── stg_order_status_history.sql
│   ├── intermediate/
│   │   ├── int_order_items_enriched.sql
│   │   └── int_order_status_history.sql
│   └── marts/
│       └── core/
│           ├── fct_order_items.sql
│           ├── fct_order_status_history.sql
│           ├── dim_customers.sql
│           ├── dim_products.sql
│           ├── dim_order_status.sql
│           ├── dim_date.sql
│           └── schema.yml
├── macros/
│   └── revenue_status.sql
├── snapshots/
│   └── customers_snapshot.sql
└── tests/
    ├── assert_no_invalid_order_items.sql
    └── assert_one_current_customer_version.sql
```

Структура отражает разделение ответственности между слоями:

- Staging — подготовка исходных данных.
- Intermediate — объединение данных и повторно используемая бизнес-логика.
- Marts — факты и измерения для аналитики.
- Macros — повторно используемые SQL-фрагменты.
- Snapshots — сохранение истории изменений.
- Tests — проверки качества данных.

## 3. Источники данных — Sources

Исходные таблицы Raw регистрируются в dbt через `sources.yml`.

Это позволяет:

- Явно описать источники данных.
- Использовать `source()` для обращения к исходным таблицам.
- Проверять наличие и качество источников.
- Отслеживать зависимости между источниками и моделями.

Пример:

```yaml
version: 2

sources:
  - name: raw_ecommerce
    schema: raw
    tables:
      - name: customers
      - name: products
      - name: orders
      - name: order_items
      - name: order_status
      - name: order_status_history
```

В примере `orders` соответствует исходной таблице `all_orders`, переименованной при загрузке в Raw.

## 4. Staging-модели

Staging-модели формируют единый и подготовленный слой исходных данных.

Пример `stg_order_items.sql`:

```sql
SELECT
    order_id,
    product_id,
    quantity,
    unit_price,
    quantity * unit_price AS line_revenue
FROM {{ source('raw_ecommerce', 'order_items') }}
```

Пример `stg_orders.sql`:

```sql
SELECT
    order_id,
    customer_id,
    order_date,
    status
FROM {{ source('raw_ecommerce', 'orders') }}
```

На этом уровне выполняются базовые преобразования:

- Приведение типов.
- Стандартизация названий полей.
- Обработка технических особенностей источника.
- Сохранение исходной детализации данных.

Staging не должен содержать сложную бизнес-логику расчёта аналитических показателей.

## 5. Intermediate-модели

Intermediate-модели объединяют подготовленные данные и содержат повторно используемую бизнес-логику.

Пример `int_order_items_enriched.sql`:

```sql
SELECT
    oi.order_id,
    oi.product_id,
    o.customer_id,
    o.order_date,
    o.status,
    oi.quantity,
    oi.unit_price,
    oi.line_revenue
FROM {{ ref('stg_order_items') }} oi
JOIN {{ ref('stg_orders') }} o
    ON oi.order_id = o.order_id
```

Модель объединяет товарные позиции с информацией о заказах.

На следующем этапе она используется для формирования `fct_order_items`.

### Правило учёта отменённых заказов

В базовой фактовой таблице сохраняются все товарные позиции, включая позиции отменённых заказов. Это позволяет не терять исходную детализацию.

Для расчёта выручки применяется единое бизнес-правило: учитываются только заказы со статусами:

- `paid`
- `processing`
- `shipped`
- `delivered`

Правило рекомендуется централизовать через dbt-макрос.

Пример `macros/revenue_status.sql`:

```sql
{% macro revenue_eligible_status(status_column) %}

    {{ status_column }} IN (
        'paid',
        'processing',
        'shipped',
        'delivered'
    )

{% endmacro %}
```

Использование в аналитической модели:

```sql
SELECT
    SUM(revenue) AS total_revenue
FROM {{ ref('fct_order_items') }}
WHERE {{ revenue_eligible_status('status') }}
```

Таким образом, изменение бизнес-правила выполняется централизованно, а не отдельно в каждом SQL-запросе.

Важно: поле `status` для такого расчёта должно быть доступно в соответствующей модели или получено через JOIN с данными о заказах. Сам факт `fct_order_items` не обязан хранить `status_sk`.

## 6. Marts-модели

На уровне Marts формируются фактовые таблицы и измерения, предназначенные для аналитики.

Основные модели:

- `fct_order_items`
- `fct_order_status_history`
- `dim_customers`
- `dim_products`
- `dim_order_status`
- `dim_date`

Модели связываются через `ref()`.

### Формирование `fct_order_items`

Принципиальная логика:

1. Получить товарные позиции из `int_order_items_enriched`.
2. Сопоставить `customer_id` с исторической версией клиента.
3. Сопоставить `product_id` с `dim_products`.
4. Сопоставить дату заказа с `dim_date`.
5. Рассчитать выручку товарной позиции.

Для выбора исторической версии клиента используется условие:

```sql
order_date >= valid_from
AND order_date < valid_to
```

Пример сопоставления клиента:

```sql
SELECT
    oi.order_id,
    oi.product_id,
    c.customer_sk,
    oi.quantity,
    oi.unit_price,
    oi.line_revenue AS revenue
FROM {{ ref('int_order_items_enriched') }} oi
LEFT JOIN {{ ref('dim_customers') }} c
    ON oi.customer_id = c.customer_id
    AND oi.order_date >= c.valid_from
    AND oi.order_date < c.valid_to
```

Это концептуальный пример сопоставления клиента. Полная модель дополнительно должна присоединить `dim_products` и `dim_date`, а также проверить отсутствие фактов без соответствующих измерений.

### Формирование `fct_order_status_history`

Модель строится на основе `int_order_status_history`.

Для каждой записи истории статусов сопоставляются:

- `status_id` с `dim_order_status`.
- Дата начала действия статуса с `dim_date`.
- Дата окончания действия статуса с `dim_date`.

Для анализа продолжительности статуса целесообразно сохранять исходные временные метки `status_begins` и `status_ends`, а не только календарные ключи.

## 7. Историзация данных — SCD Type 2

Для сохранения истории изменений атрибутов клиента может использоваться dbt Snapshot.

Snapshot позволяет фиксировать изменения записей источника и создавать исторические версии.

Пример концептуальной конфигурации:

```sql
{% snapshot customers_snapshot %}

{{
    config(
        target_schema='snapshots',
        unique_key='customer_id',
        strategy='check',
        check_cols=['full_name', 'email']
    )
}}

SELECT
    customer_id,
    full_name,
    email
FROM {{ source('raw_ecommerce', 'customers') }}

{% endsnapshot %}
```

Snapshot создаёт историю изменений с периодами действия версий.

При построении `dim_customers` можно использовать данные Snapshot, сопоставляя технические поля истории dbt с полями измерения:

- `dbt_valid_from` → `valid_from`
- `dbt_valid_to` → `valid_to`
- `dbt_scd_id` → технический идентификатор версии

`customer_sk` при этом выступает ключом версии клиента в аналитической модели.

Важно учитывать:

- Snapshot фиксирует изменения только во время своих запусков.
- Он не восстанавливает все промежуточные изменения между запусками.
- Для полноценной событийной истории может потребоваться CDC или отдельный журнал изменений.
- Текущая таблица `customers` не содержит сегментов клиента. История сегментов потребует дополнительного источника данных.

SCD Type 2 для `full_name` и `email` является проектным решением, а не реализованной функциональностью.

## 8. Тестирование качества данных

Для контроля качества данных используются встроенные generic tests и пользовательские SQL-тесты dbt.

### Generic tests

| Тест | Назначение |
|---|---|
| `not_null` | Проверка обязательных значений |
| `unique` | Проверка уникальности ключей |
| `relationships` | Проверка ссылочной целостности |
| `accepted_values` | Проверка допустимых значений |

Пример `schema.yml`:

```yaml
version: 2

models:
  - name: dim_customers
    columns:
      - name: customer_sk
        data_tests:
          - not_null
          - unique

      - name: customer_id
        data_tests:
          - not_null

  - name: fct_order_items
    columns:
      - name: product_sk
        data_tests:
          - not_null
          - relationships:
              arguments:
                to: ref('dim_products')
                field: product_sk

      - name: customer_sk
        data_tests:
          - not_null
          - relationships:
              arguments:
                to: ref('dim_customers')
                field: customer_sk
```

Примечание: синтаксис `data_tests` и `arguments` зависит от версии dbt. Для проекта необходимо зафиксировать версию и использовать соответствующий ей синтаксис.

### Пользовательский SQL-тест

Пример проверки отсутствия некорректных товарных позиций:

```sql
SELECT *
FROM {{ ref('fct_order_items') }}
WHERE quantity <= 0
   OR unit_price < 0
```

Если запрос возвращает строки, тест завершается ошибкой.

### Проверка единственной текущей версии клиента

Пример теста `assert_one_current_customer_version.sql`:

```sql
SELECT
    customer_id
FROM {{ ref('dim_customers') }}
WHERE is_current = TRUE
GROUP BY customer_id
HAVING COUNT(*) > 1
```

Тест должен завершаться успешно только при наличии не более одной текущей версии для каждого клиента.

Дополнительно необходимо контролировать пересечение периодов действия версий одного клиента. Это предотвращает неоднозначное сопоставление заказа с исторической версией клиента.

## 9. Инкрементальная загрузка

Для крупных фактовых таблиц предполагается использовать:

```sql
{{ config(
    materialized='incremental',
    unique_key=['order_id', 'product_id']
) }}
```

Инкрементальная материализация позволяет обрабатывать новые и изменённые записи без полной перезаписи всей таблицы.

Основные принципы:

- Определение уникального ключа записи.
- Отбор новых и изменённых данных.
- Использование подходящей стратегии обновления, например `merge`, если она поддерживается целевой платформой.
- Обработка запоздавших изменений.
- Возможность повторной обработки исторических периодов.

В исходной таблице `order_items` отдельного идентификатора позиции нет. В качестве уникального ключа используется комбинация `order_id` и `product_id`.

**Ограничение источника:** в текущей структуре не подтверждено наличие поля `updated_at`. Поэтому нельзя без дополнительных механизмов гарантировать обнаружение всех изменённых записей при инкрементальной загрузке.

Для полноценной реализации необходимо выбрать механизм отслеживания изменений:

- Добавить техническое поле времени обновления, если это возможно.
- Использовать CDC.
- Применять повторную обработку ограниченного исторического периода, если бизнес-условия и доступные данные позволяют это сделать.

Выбор стратегии зависит от целевой СУБД, объёма данных и требований к актуальности.

## 10. Зависимости и запуск моделей

dbt автоматически определяет зависимости моделей через `ref()` и источников через `source()`.

Это позволяет строить граф выполнения моделей и соблюдать последовательность преобразований.

Основные команды:

```bash
dbt debug
dbt run
dbt test
dbt snapshot
dbt docs generate
dbt docs serve
```

Назначение:

- `dbt debug` — проверка подключения и конфигурации.
- `dbt run` — выполнение моделей.
- `dbt test` — запуск тестов качества данных.
- `dbt snapshot` — обновление исторических снимков.
- `dbt docs generate` — генерация документации.
- `dbt docs serve` — локальный просмотр документации.

В рабочем процессе запуск моделей, тестов и Snapshot должен выполняться в согласованном порядке. Расписание и оркестрация могут быть организованы отдельным инструментом.

## 11. Статус реализации

**Спроектировано:**

- Структура dbt-проекта.
- Организация Sources, Staging, Intermediate и Marts.
- Принципы построения моделей через `ref()`.
- Централизация бизнес-правила расчёта выручки.
- Подход к историзации SCD Type 2.
- Проверки качества данных.
- Принципы инкрементальной загрузки.

**Не реализовано:**

- Подключение dbt к физическому DWH.
- Реальные SQL-модели и Snapshot.
- Выполнение dbt-тестов.
- Автоматизированный ELT pipeline.
- Развёртывание и расписание обновлений.

Документ описывает проектируемый подход к реализации dbt, а не действующий ETL/ELT-процесс.
