Это отличная и очень практичная лабораторная работа! Чтобы выполнить её идеально и получить высший балл, нужно строго следовать требованиям: использовать **VS Code + Git-Bash + Docker CLI**, вести **лог**, сделать **два дампа** и показать **аномалии**.

Ниже представлена подробная пошаговая инструкция, как выполнить это задание от начала до конца.

---

### Шаг 0. Подготовка окружения и запуск логирования

1. Откройте **VS Code** и откройте в нем терминал (**Git-Bash**).
2. Запустите Docker-контейнер с MySQL (если еще не запущен):
   ```bash
   docker run --name mysql-lab -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=sales_norm -p 3306:3306 -d mysql:latest
   ```
3. Подключитесь к базе данных через терминал. 
   💡 **Важно для лога:** Сразу после подключения включите команду `tee`, чтобы всё, что вы вводите и видите, автоматически сохранялось в текстовый файл!
   ```bash
   docker exec -it mysql-lab mysql -uroot -proot sales_norm
   ```
   Внутри консоли MySQL введите:
   ```sql
   \T lab_log.txt
   ```
   *(Теперь все ваши действия пишутся в файл `lab_log.txt`. Не забудьте отключить эту команду в самом конце через `\t`)*.

---

### Шаг 1. Создание «плохой» таблицы и первый дамп

Скопируйте и выполните SQL-скрипт из задания для создания и заполнения таблицы `sales_unnormalized`.

```sql
CREATE TABLE sales_unnormalized (
     sale_id       INT,
     sale_date     DATE,
     client_id     INT,
     client_name   VARCHAR(100),
     client_city   VARCHAR(100),
     product_id    INT,
     product_name  VARCHAR(100),
     unit_price    DECIMAL(10,2),
     quantity      INT
);

INSERT INTO sales_unnormalized VALUES
(5001, '2026-11-01', 101, 'Иванов Иван',     'Москва',           201, 'Монитор',    15000.00, 2),
(5001, '2026-11-01', 101, 'Иванов Иван',     'Москва',           202, 'Клавиатура',  2000.00, 1),
(5002, '2026-11-02', 102, 'Петрова Мария',   'Санкт-Петербург', 203, 'Мышь',        1000.00, 3),
(5002, '2026-11-02', 102, 'Петрова Мария',   'Санкт-Петербург', 204, 'Наушники',    3500.00, 1),
(5003, '2026-11-03', 101, 'Иванов Иван',     'Москва',           205, 'Колонки',     2500.00, 2),
(5004, '2026-11-04', 103, 'Сидоров Алексей', 'Казань',           202, 'Клавиатура',  2000.00, 5),
(5004, '2026-11-04', 103, 'Сидоров Алексей', 'Казань',           203, 'Мышь',        1000.00, 2),
(5005, '2026-11-05', 102, 'Петрова Мария',   'Санкт-Петербург', 201, 'Монитор',    15000.00, 1),
(5006, '2026-11-06', 103, 'Сидоров Алексей', 'Казань',           204, 'Наушники',    3500.00, 2),
(5006, '2026-11-06', 103, 'Сидоров Алексей', 'Казань',           205, 'Колонки',     2500.00, 1);

SELECT * FROM sales_unnormalized;
```

📸 **Сделайте скриншот** результата `SELECT *`.

💾 **Сделайте ПЕРВЫЙ дамп** (выйдите из MySQL командой `exit` и выполните в Git-Bash):
```bash
docker exec mysql-lab mysqldump -uroot -proot sales_norm > dump_before.sql
```

---

### Шаг 2. Демонстрация аномалий

Вернитесь в MySQL (`docker exec -it ...`) и покажите преподавателю, почему таблица "плохая".

**1. Аномалия обновления (Update):**
Представим, что цена Монитора (product_id = 201) изменилась. Нам придется обновлять её в нескольких строках.
```sql
-- Обновляем цену, но забываем, что она дублируется
UPDATE sales_unnormalized SET unit_price = 16000.00 WHERE product_id = 201 AND sale_id = 5001;
-- Если бы мы не указали sale_id, обновились бы все строки, но риск ошибки велик.
SELECT * FROM sales_unnormalized WHERE product_id = 201;
```

**2. Аномалия вставки (Insert):**
Мы не можем добавить нового клиента, если он еще ничего не купил (так как `sale_id` и `product_id` фактически являются частью ключа).
```sql
-- Попытка добавить клиента без покупки (выдаст ошибку или создаст мусор)
INSERT INTO sales_unnormalized (sale_id, client_id, client_name, client_city) 
VALUES (NULL, 104, 'Новиков Петр', 'Сочи');
```

**3. Аномалия удаления (Delete):**
Если клиент вернул все товары и мы удалим его продажи, мы потеряем информацию о самом клиенте!
```sql
-- Удаляем все продажи Петровой
DELETE FROM sales_unnormalized WHERE client_id = 102;
-- Проверяем: Петровой больше нет в базе, хотя она наш клиент!
SELECT * FROM sales_unnormalized WHERE client_id = 102;
```

📸 **Сделайте скриншоты** этих аномалий.

---

### Шаг 3. Нормализация (Приведение к 3NF)

Создаем 4 правильные таблицы с первичными (PK) и внешними ключами (FK).

```sql
CREATE TABLE clients (
    client_id INT PRIMARY KEY,
    client_name VARCHAR(100) NOT NULL,
    client_city VARCHAR(100) NOT NULL
);

CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100) NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL
);

CREATE TABLE sales (
    sale_id INT PRIMARY KEY,
    sale_date DATE NOT NULL,
    client_id INT,
    FOREIGN KEY (client_id) REFERENCES clients(client_id)
);

CREATE TABLE sale_items (
    sale_id INT,
    product_id INT,
    quantity INT NOT NULL,
    PRIMARY KEY (sale_id, product_id),
    FOREIGN KEY (sale_id) REFERENCES sales(sale_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

---

### Шаг 4. Перенос данных (INSERT ... SELECT)

Используем `DISTINCT`, чтобы убрать дубликаты при переносе.

```sql
-- Для клиентов
INSERT INTO clients (client_id, client_name, client_city)
SELECT DISTINCT client_id, client_name, client_city 
FROM sales_unnormalized 
WHERE client_id IS NOT NULL;

-- Для продаж (шапок)
INSERT INTO sales (sale_id, sale_date, client_id)
SELECT DISTINCT sale_id, sale_date, client_id 
FROM sales_unnormalized 
WHERE sale_id IS NOT NULL;

-- 3. Переносим сами продажи (шапки)
INSERT INTO sales (sale_id, sale_date, client_id)
SELECT DISTINCT sale_id, sale_date, client_id 
FROM sales_unnormalized 
WHERE sale_id IS NOT NULL 
  AND client_id IS NOT NULL;
  
-- 4. Переносим позиции продаж (детализацию)
INSERT INTO sale_items (sale_id, product_id, quantity)
SELECT sale_id, product_id, quantity 
FROM sales_unnormalized 
WHERE sale_id IS NOT NULL 
  AND product_id IS NOT NULL;
```

---

### Шаг 5. Проверка, очистка и ВТОРОЙ дамп

Проверяем, что данные успешно перенеслись (сравниваем количество строк и суммы).

```sql
-- Проверяем количество записей
SELECT COUNT(*) AS bad_table_rows FROM sales_unnormalized;
SELECT COUNT(*) AS items_rows FROM sale_items;

-- Проверяем, что данные не потерялись (JOIN для проверки)
SELECT s.sale_id, c.client_name, p.product_name, si.quantity 
FROM sales s
JOIN clients c ON s.client_id = c.client_id
JOIN sale_items si ON s.sale_id = si.sale_id
JOIN products p ON si.product_id = p.product_id;
```
📸 **Сделайте скриншот** успешного переноса.

Теперь удаляем "плохую" таблицу, так как она больше не нужна:
```sql
DROP TABLE sales_unnormalized;
SHOW TABLES;
```

💾 **Выйдите из MySQL** (`exit`) и сделайте **ВТОРОЙ дамп** в Git-Bash:
```bash
docker exec mysql-lab mysqldump -uroot -proot sales_norm > dump_after.sql
```

Отключите логирование (если еще не вышли из MySQL):
```sql
\t
```

---
