# Архитектура аналитического хранилища данных

Диаграмма показывает предполагаемый путь данных от операционной базы PostgreSQL `FOR CV` до аналитических витрин и BI-отчётности.

**Зелёным обозначен существующий источник данных. Синим — проектируемые слои архитектуры.**

```mermaid
flowchart TB
    subgraph SOURCE["Источник данных — реализовано"]
        PG[("PostgreSQL FOR CV")]
        T1["customers"]
        T2["products"]
        T3["all_orders"]
        T4["order_items"]
        T5["order_status"]
        T6["order_status_history"]

        PG --- T1
        PG --- T2
        PG --- T3
        PG --- T4
        PG --- T5
        PG --- T6
    end

    LOAD["Загрузка данных: Extract → Load"]

    subgraph RAW["RAW — исходные копии"]
        R1[("raw_customers")]
        R2[("raw_products")]
        R3[("raw_orders")]
        R4[("raw_order_items")]
        R5[("raw_order_status")]
        R6[("raw_order_status_history")]
    end

    subgraph STAGING["STAGING — подготовка данных"]
        S1["stg_customers"]
        S2["stg_products"]
        S3["stg_orders"]
        S4["stg_order_items"]
        S5["stg_order_status"]
        S6["stg_order_status_history"]
    end

    subgraph INTERMEDIATE["INTERMEDIATE — бизнес-логика"]
        I1["int_order_items_enriched"]
        I2["int_order_status_history"]
    end

    subgraph MARTS["MARTS — аналитическая модель"]
        F1[("fct_order_items")]
        F2[("fct_order_status_history")]
        D1[("dim_customers — SCD Type 2")]
        D2[("dim_products")]
        D3[("dim_date")]
        D4[("dim_order_status")]
    end

    BI["Power BI / SQL-аналитика / Бизнес-отчёты"]

    PG --> LOAD
    LOAD --> RAW

    R1 --> S1
    R2 --> S2
    R3 --> S3
    R4 --> S4
    R5 --> S5
    R6 --> S6

    S3 --> I1
    S4 --> I1
    S3 --> I2
    S6 --> I2
    S5 --> I2

    I1 --> F1
    I2 --> F2

    S1 --> D1
    S2 --> D2
    S5 --> D4

    D1 --> F1
    D2 --> F1
    D3 --> F1
    D4 --> F2

    F1 --> BI
    F2 --> BI

    classDef implemented fill:#dcfce7,stroke:#16a34a,color:#14532d
    classDef proposed fill:#dbeafe,stroke:#2563eb,color:#1e3a8a

    class PG,T1,T2,T3,T4,T5,T6 implemented
    class LOAD,R1,R2,R3,R4,R5,R6,S1,S2,S3,S4,S5,S6,I1,I2,F1,F2,D1,D2,D3,D4,BI proposed
```

## Обозначения

- **Source** — существующая операционная база PostgreSQL.
- **Raw** — исходные копии данных без существенных бизнес-преобразований.
- **Staging** — подготовка и стандартизация данных.
- **Intermediate** — объединение таблиц и применение общей бизнес-логики.
- **Marts** — факты и измерения для аналитики.
- **BI** — использование готовых данных в отчётности.

## Особенности проектируемой модели

- `fct_order_items` — факты по товарным позициям заказов.
- `fct_order_status_history` — история изменения статусов заказов.
- `dim_customers` — измерение клиентов с возможностью реализации SCD Type 2 на основе существующих атрибутов `full_name` и `email`.
- `dim_products` — измерение товаров.
- `dim_date` — календарное измерение.
- `dim_order_status` — справочник статусов заказов.

**Статус:** PostgreSQL `FOR CV` реализована ранее. Остальные слои описывают проектируемое решение и не являются реализованным DWH/dbt-пайплайном. Для отслеживания сегментов клиентов или рекламных кампаний потребуются дополнительные источники данных.
