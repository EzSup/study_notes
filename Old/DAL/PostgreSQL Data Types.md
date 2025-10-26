## Common data types

| Data Type             | Description                                  | Example                                  |
| --------------------- | -------------------------------------------- | ---------------------------------------- |
| `SMALLINT`            | Small-range integer                          | `32767`                                  |
| `INTEGER` / `INT`     | Standard integer                             | `2147483647`                             |
| `BIGINT`              | Large-range integer                          | `9223372036854775807`                    |
| `DECIMAL` / `NUMERIC` | Exact numeric with precision                 | `1234.567`                               |
| `VARCHAR(n)`          | Variable-length character string (limit `n`) | `'example'`                              |
| `TEXT`                | Unlimited variable-length text               | `'long text'`                            |
| `DATE`                | Calendar date                                | `'2025-01-01'`                           |
| `TIMESTAMP`           | Date and time                                | `'2025-01-01 12:00:00'`                  |
| `BOOLEAN`             | Logical true/false                           | `TRUE` / `FALSE`                         |
| `UUID`                | Universally unique identifier                | `'550e8400-e29b-41d4-a716-446655440000'` |
| `JSON` / `JSONB`      | JSON data                                    | `'{"key": "value"}'`                     |
| `BYTEA`               | Binary data (e.g., file content)             | `'\xDEADBEEF'`                           |

## Data types modifiers

| Modifier                                              | Description                                                                                                                                        | Example                                                                              |
| ----------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| `NOT NULL`                                            | Створює обмеження, яке не дозволяє зберігати значення `NULL` для цього стовпця.                                                                    | `CREATE TABLE users (id SERIAL NOT NULL, name VARCHAR(100));`                        |
| `NULL`                                                | За замовчуванням, стовпець дозволяє зберігати значення `NULL` (якщо не вказано інше).                                                              | `CREATE TABLE users (id SERIAL, name VARCHAR(100));`                                 |
| `DEFAULT`                                             | Встановлює значення за замовчуванням, яке буде використовуватися, якщо при вставці не вказано значення для цього стовпця.                          | `CREATE TABLE users (id SERIAL PRIMARY KEY, name VARCHAR(100) DEFAULT 'Anonymous');` |
| `PRIMARY KEY`                                         | Визначає стовпець як первинний ключ (унікальний і не може бути NULL).                                                                              | `CREATE TABLE users (id SERIAL PRIMARY KEY, name VARCHAR(100));`                     |
| `UNIQUE`                                              | Встановлює обмеження, яке гарантує, що всі значення в стовпці будуть унікальними.                                                                  | `CREATE TABLE users (id SERIAL PRIMARY KEY, email VARCHAR(100) UNIQUE);`             |
| `CHECK`                                               | Додає обмеження для значень стовпця (наприклад, перевірка діапазону значень).                                                                      | `CREATE TABLE products (price DECIMAL CHECK (price >= 0));`                          |
| `REFERENCES`                                          | Встановлює зовнішній ключ, який вказує на інший стовпець в іншій таблиці.                                                                          | `CREATE TABLE orders (user_id INTEGER REFERENCES users(id));`                        |
| `ON DELETE CASCADE`                                   | Встановлює правило для зовнішнього ключа, яке автоматично видаляє рядки в залежній таблиці, коли відповідний рядок в основній таблиці видаляється. | `CREATE TABLE orders (user_id INTEGER REFERENCES users(id) ON DELETE CASCADE);`      |
| `ON UPDATE CASCADE`                                   | Встановлює правило для зовнішнього ключа, яке автоматично оновлює значення в залежній таблиці, коли значення в основній таблиці змінюється.        | `CREATE TABLE orders (user_id INTEGER REFERENCES users(id) ON UPDATE CASCADE);`      |
| `AUTO_INCREMENT` (PostgreSQL: `SERIAL` / `BIGSERIAL`) | Створює стовпець, значення якого автоматично збільшується при кожному новому рядку.                                                                | `CREATE TABLE users (id SERIAL PRIMARY KEY, name VARCHAR(100));`                     |
| `COLLATE`                                             | Визначає правила сортування для текстових стовпців (наприклад, з урахуванням регістру чи без).                                                     | `CREATE TABLE users (name VARCHAR(100) COLLATE "en_US");`                            |

## Detailed data types
### 1. Numeric Data Types
| Data Type             | Description                                              | Storage Size | Range/Example                                           |
| --------------------- | -------------------------------------------------------- | ------------ | ------------------------------------------------------- |
| `SMALLINT`            | Small-range integer                                      | 2 bytes      | -32,768 to 32,767                                       |
| `INTEGER` / `INT`     | Standard integer                                         | 4 bytes      | -2,147,483,648 to 2,147,483,647                         |
| `BIGINT`              | Large-range integer                                      | 8 bytes      | -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807 |
| `DECIMAL` / `NUMERIC` | Exact numeric type with user-defined precision and scale | Variable     | `10.50`, `1234.567`                                     |
| `REAL`                | Single-precision floating point                          | 4 bytes      | Approximate value: `3.14159`                            |
| `DOUBLE PRECISION`    | Double-precision floating point                          | 8 bytes      | Approximate value: `3.14159265358979`                   |
| `SERIAL`              | Auto-incrementing integer (4 bytes)                      | 4 bytes      | `1, 2, 3...`                                            |
| `BIGSERIAL`           | Auto-incrementing integer (8 bytes)                      | 8 bytes      | `1, 2, 3...`                                            |

### 2. Character Data Types
| Data Type    | Description                                                   | Storage Size | Example              |
| ------------ | ------------------------------------------------------------- | ------------ | -------------------- |
| `CHAR(n)`    | Fixed-length character string (padded with spaces if shorter) | n bytes      | `'abc'`              |
| `VARCHAR(n)` | Variable-length character string                              | n+1 bytes    | `'example'`          |
| `TEXT`       | Unlimited variable-length text                                | Variable     | `'long text string'` |

### 3. Date/Time Data Types
| Data Type     | Description                  | Storage Size | Range/Example                 |
| ------------- | ---------------------------- | ------------ | ----------------------------- |
| `DATE`        | Calendar date                | 4 bytes      | `'2025-01-01'`                |
| `TIME`        | Time of day (no timezone)    | 8 bytes      | `'13:45:30'`                  |
| `TIMESTAMP`   | Date and time (no timezone)  | 8 bytes      | `'2025-01-01 13:45:30'`       |
| `TIMESTAMPTZ` | Date and time with time zone | 8 bytes      | `'2025-01-01 13:45:30+02:00'` |
| `INTERVAL`    | Time span                    | 16 bytes     | `'1 year 2 months 3 days'`    |

### 4. Boolean Data Type
| Data Type | Description        | Storage Size | Example                 |
| --------- | ------------------ | ------------ | ----------------------- |
| `BOOLEAN` | Logical true/false | 1 byte       | `TRUE`, `FALSE`, `NULL` |

### 5. UUID Data Type
| Data Type | Description                   | Storage Size | Example                                  |
| --------- | ----------------------------- | ------------ | ---------------------------------------- |
| `UUID`    | Universally unique identifier | 16 bytes     | `'550e8400-e29b-41d4-a716-446655440000'` |

### 6. JSON Data Types
| Data Type       | Description                                             | Storage Size   | Example                              |
|------------------|---------------------------------------------------------|----------------|--------------------------------------|
| `JSON`          | Stores JSON data                                        | Variable       | `'{"key": "value"}'`                |
| `JSONB`         | Binary format of JSON data (more efficient for indexing)| Variable       | `'{"key": "value"}'`                |

### 7. Arrays
| Data Type       | Description                                             | Storage Size   | Example                              |
|------------------|---------------------------------------------------------|----------------|--------------------------------------|
| `ARRAY`         | One-dimensional or multi-dimensional array              | Variable       | `'{1, 2, 3}'` or `'{{1,2},{3,4}}'` |

### 8. Special Data Types
| Data Type       | Description                                             | Storage Size   | Example                              |
|------------------|---------------------------------------------------------|----------------|--------------------------------------|
| `BYTEA`         | Binary data (e.g., file content)                        | Variable       | `'\xDEADBEEF'`                      |
| `CIDR`          | IPv4 or IPv6 network                                    | Variable       | `'192.168.100.128/25'`              |
| `INET`          | IPv4 or IPv6 host address                               | Variable       | `'192.168.100.128'`                 |
| `MACADDR`       | MAC address                                             | 6 bytes        | `'08:00:2b:01:02:03'`               |

### 9. Geometric Data Types
| Data Type | Description           | Example                    |
| --------- | --------------------- | -------------------------- |
| `POINT`   | Point in a 2D plane   | `(1, 2)`                   |
| `LINE`    | Infinite line         | `{1, 2, 3}`                |
| `LSEG`    | Line segment          | `[(0, 0), (1, 1)]`         |
| `BOX`     | Rectangular box       | `((0, 0), (1, 1))`         |
| `CIRCLE`  | Circle                | `<(1, 1), 5>`              |
| `POLYGON` | Closed geometric path | `((0, 0), (1, 1), (2, 0))` |

### 10. Range Types
| Data Type       | Description                                             | Example                              |
|------------------|---------------------------------------------------------|--------------------------------------|
| `INT4RANGE`     | Range of integers                                       | `'[1,10)'`                          |
| `NUMRANGE`      | Range of numeric values                                 | `'[1.5,2.5]'`                       |
| `TSRANGE`       | Range of timestamps                                     | `'[2025-01-01,2025-02-01)'`         |
| `DATERANGE`     | Range of dates                                          | `'[2025-01-01,2025-01-31)'`         |
| `TSTZRANGE`     | Range of timestamps with timezone                       | `'[2025-01-01 00:00+02,2025-02-01)'`|
