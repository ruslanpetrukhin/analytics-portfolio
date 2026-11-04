# Реализация аналитического хранилища с использованием dbt

## 1. Назначение dbt

dbt (Data Build Tool) используется для преобразования данных внутри аналитического хранилища.

В рамках проекта dbt отвечает за:
- Создание SQL-моделей на основе исходных данных.
- Организацию зависимостей между моделями.
- Централизацию бизнес-логики.
- Проверку качества данных.
- Инкрементальное обновление аналитических таблиц.
- Документирование моделей и их полей.

Первоначальная загрузка данных из PostgreSQL в Raw выполняется отдельным инструментом. dbt начинает работу после загрузки исходных данных.

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
│       ├── core/
│       │   ├── fct_order_items.sql
│       │   ├── fct_order_status_history.sql
│       │   ├── dim_customers.sql
│       │   ├── dim_products.sql
│       │   ├── dim_order_status.sql
│       │   ├── dim_date.sql
│       │   └── schema.yml
├── snapshots/
│   └── customers_snapshot.sql
└── tests/
    └── assert_no_invalid_order_items.sql
```

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

На этом уровне выполняются базовые преобразования: приведение типов, стандартизация названий полей и обработка технических особенностей источника.

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

На следующем этапе она используется для формирования фактовой таблицы.

**Правило учёта отменённых заказов:**

В базовой фактовой таблице сохраняются все товарные позиции, включая отменённые заказы. Это позволяет не терять исходную детализацию.

Для расчёта выручки используется единое условие включения статусов:

```sql
status IN (
    'paid',
    'processing',
    'shipped',
    'delivered'
)
```

Это правило должно быть централизовано и повторно использоваться в аналитических моделях, например через общий макрос или отдельную промежуточную модель отфильтрованных продаж.

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

Пример:

```sql
SELECT
    order_id,
    product_id,
    customer_id,
    order_date,
    quantity,
    unit_price,
    line_revenue AS revenue
FROM {{ ref('int_order_items_enriched') }}
```

Для формирования `fct_order_items` дополнительно выполняется сопоставление исходных идентификаторов с суррогатными ключами измерений.

В частности, `customer_sk` определяется по версии клиента, действовавшей на дату заказа.

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

Важно учитывать, что snapshot фиксирует изменения только во время своих запусков. Он не заменяет полноценную событийную историю или CDC, если требуется восстановить все изменения между запусками.

Текущая исходная таблица `customers` не содержит сегментов клиента. Поэтому история сегментов потребует дополнительного источника данных.

## 8. Тестирование качества данных

Для контроля качества данных используются встроенные generic tests и пользовательские SQL-тесты dbt.

### Generic tests

| Тест | Назначение |
|---|---|
| `not_null` | Проверка обязательных значений |
| `unique` | Проверка уникальности ключей |
| `relationships` | Проверка ссылочной целостности |
| `accepted_values` | Проверка допустимых значений |

Пример:

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
          - relationships:
              arguments:
                to: ref('dim_products')
                field: product_sk
```

### Пользовательский SQL-тест

Пример проверки отсутствия некорректных товарных позиций:

```sql
SELECT *
FROM {{ ref('fct_order_items') }}
WHERE quantity <= 0
   OR unit_price < 0
```

Если запрос возвращает строки, тест завершается ошибкой.

Дополнительно можно создать тест, проверяющий, что у каждого клиента не более одной текущей версии в `dim_customers`.

## 9. Инкрементальная загрузка

Для крупных фактовых таблиц предполагается использовать `materialized='incremental'`.

Это позволяет обрабатывать новые и изменённые записи без полной перезаписи всей таблицы.

Принципы реализации:
- Определение уникального ключа записи.
- Отбор новых и изменённых данных.
- Использование подходящей стратегии обновления, например `merge`, если она поддерживается целевой платформой.
- Обработка запоздавших изменений.
- Возможность повторной обработки исторических периодов.

В исходной таблице `order_items` отдельного идентификатора позиции нет. В качестве уникального ключа можно использовать комбинацию `order_id` и `product_id` либо сформировать технический ключ.

Фактический выбор стратегии зависит от возможностей целевой СУБД и требований к обновлению данных.

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

## 11. Статус реализации

**Спроектировано:**
- Структура dbt-проекта.
- Организация Sources, Staging, Intermediate и Marts.
- Принципы построения моделей через `ref()`.
- Подход к историзации SCD Type 2.
- Проверки качества данных.
- Принципы инкрементальной загрузки.

**Не реализовано:**
- Подключение dbt к физическому DWH.
- Реальные SQL-модели и Snapshot.
- Выполнение dbt-тестов.
- Автоматизированный pipeline.
- Развёртывание и расписание обновлений.

Документ описывает проектируемый подход к реализации dbt, а не действующий ETL/ELT-процесс.
