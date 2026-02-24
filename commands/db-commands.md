sudo net start/stop MySQL80
sudo net start/stop postgresql-x64-18

C://Program Files/MySQL/MySQL Serser 8.0/bin
C://Program Files/PostgreSQL/18/bin

mysql -u root -p
psql -U postgres



INT                 -- Целое число (-2^31 до 2^31-1)
INT UNSIGNED        -- Положительное целое (0 до 2^32-1)
BIGINT              -- Большое целое
DECIMAL(10, 2)      -- Точные числа (10 цифр, 2 после запятой)
FLOAT               -- Числа с плавающей точкой
DOUBLE              -- Double precision
BOOLEAN             -- Логический тип (TRUE/FALSE)


VARCHAR(255)        -- Переменная длина (до 65535 символов)
CHAR(10)            -- Фиксированная длина (дополняется пробелами)
TEXT                -- Большой текст (до 64KB)
LONGTEXT            -- Очень большой текст (до 4GB)
ENUM('Y', 'N')      -- Перечисление значений
SET('a', 'b', 'c')  -- Набор значений


DATE                -- Дата (YYYY-MM-DD)
TIME                -- Время (HH:MM:SS)
DATETIME            -- Дата и время (YYYY-MM-DD HH:MM:SS)
TIMESTAMP           -- Timestamp (до 2038 года)
YEAR                -- Год (1901-2155)


BLOB                -- Бинарные данные (до 64KB)
LONGBLOB            -- Большие бинарные данные (до 4GB)
BINARY(16)          -- Фиксированные бинарные данные
VARBINARY(255)      -- Переменные бинарные данные


NOT NULL            -- Запрет NULL значений
NULL                -- Разрешение NULL значений
DEFAULT value       -- Значение по умолчанию
AUTO_INCREMENT      -- Автоматическая инкрементация
UNIQUE              -- Уникальные значения
PRIMARY KEY         -- Первичный ключ
CHECK (condition)   -- Проверка условия (MySQL 8.0+)


PRIMARY KEY (col1, col2)        -- Составной первичный ключ
UNIQUE (col1, col2)             -- Составной уникальный ключ
FOREIGN KEY (col) REFERENCES other_table(col) -- Внешний ключ
CHECK (col1 > 0 AND col2 != '') -- Проверка условия


ENGINE = InnoDB     -- По умолчанию (транзакции, FK)
ENGINE = MyISAM     -- Быстрый, без транзакций
ENGINE = MEMORY     -- Хранится в памяти
ENGINE = CSV        -- CSV формат


CHARACTER SET utf8mb4          -- Кодировка
COLLATE utf8mb4_unicode_ci     -- Сортировка
DEFAULT CHARSET = utf8mb4      -- Кодировка по умолчанию




CREATE DATABASE db_name;        -- Создает новую базу данных
CREATE TABLE table_name (...);  -- Создает новую таблицу
CREATE INDEX idx_name ON table_name(column); -- Создает индекс


ALTER TABLE table_name ...;     -- Изменяет структуру таблицы
ALTER TABLE ADD column ...;     -- Добавляет колонку
ALTER TABLE MODIFY column ...;  -- Изменяет колонку
ALTER TABLE DROP column ...;    -- Удаляет колонку


DROP DATABASE db_name;          -- Удаляет базу данных
DROP TABLE table_name;          -- Удаляет таблицу
DROP INDEX index_name;          -- Удаляет индекс
TRUNCATE TABLE table_name;      -- Очищает таблицу (быстрее DELETE)


INSERT INTO table VALUES (...); -- Добавляет новые записи
UPDATE table SET ... WHERE ...; -- Изменяет существующие записи
DELETE FROM table WHERE ...;    -- Удаляет записи


SELECT ... FROM ... WHERE ...;  -- Выбирает данные из таблицы
SELECT * FROM table;            -- Выбирает все колонки
SELECT column1, column2 FROM table; -- Выбирает конкретные колонки


SELECT ... FROM ...;            -- Основная команда выборки
WHERE condition;                -- Фильтрация записей
ORDER BY column;                -- Сортировка результатов
GROUP BY column;                -- Группировка данных
HAVING condition;               -- Фильтрация после GROUP BY


INNER JOIN ... ON ...;          -- Внутреннее соединение
LEFT JOIN ... ON ...;           -- Левое соединение
RIGHT JOIN ... ON ...;          -- Правое соединение
FULL OUTER JOIN ... ON ...;     -- Полное внешнее соединение
CROSS JOIN ...;                 -- Декартово произведение


GRANT privilege ON ... TO user; -- Дает права пользователю
REVOKE privilege ON ... FROM user; -- Забирает права


START TRANSACTION;              -- Начинает транзакцию
COMMIT;                         -- Подтверждает изменения
ROLLBACK;                       -- Отменяет изменения
SAVEPOINT point_name;           -- Создает точку сохранения
ROLLBACK TO point_name;         -- Откат к точке сохранения


COMMIT;     -- Применяет изменения
ROLLBACK;   -- Откатывает изменения
SET TRANSACTION ...; -- Настраивает параметры транзакции


DISTINCT;           -- Уникальные значения
WHERE;              -- Условие фильтрации
ORDER BY;           -- Сортировка
GROUP BY;           -- Группировка
HAVING;             -- Условие для сгруппированных данных
LIMIT;              -- Ограничение количества результатов
OFFSET;             -- Смещение для пагинации


COUNT();            -- Подсчет записей
SUM();              -- Сумма значений
AVG();              -- Среднее значение
MIN();              -- Минимальное значение
MAX();              -- Максимальное значение


UNION;              -- Объединение результатов (уникальные)
UNION ALL;          -- Объединение результатов (все)
INTERSECT;          -- Пересечение результатов
EXCEPT;             -- Разность результатов


USE database_name;  -- Выбор базы данных
SHOW DATABASES;     -- Показать все базы данных
SHOW TABLES;        -- Показать таблицы в текущей БД
DESCRIBE table_name;-- Показать структуру таблицы


SHOW CREATE TABLE table_name; -- Показать SQL создания таблицы
SHOW INDEX FROM table_name;   -- Показать индексы таблицы
SHOW PROCESSLIST;             -- Показать активные соединения


CONCAT();           -- Объединение строк
SUBSTRING();        -- Подстрока
LENGTH();           -- Длина строки
UPPER(); LOWER();   -- Регистр символов
TRIM();             -- Удаление пробелов


ROUND();            -- Округление
ABS();              -- Модуль числа
CEIL(); FLOOR();    -- Округление вверх/вниз
RAND();             -- Случайное число


NOW();              -- Текущая дата и время
CURDATE();          -- Текущая дата
CURTIME();          -- Текущее время
DATE_FORMAT();      -- Форматирование даты
DATEDIFF();         -- Разница между датами


PRIMARY KEY;        -- Первичный ключ
FOREIGN KEY;        -- Внешний ключ
UNIQUE;             -- Уникальное значение
NOT NULL;           -- Запрет NULL значений
CHECK;              -- Проверка условия
DEFAULT;            -- Значение по умолчанию


AS alias;           -- Псевдоним для колонки или таблицы
/* comment */;      -- Многострочный комментарий
-- comment;         -- Однострочный комментарий
# comment;          -- Однострочный комментарий (MySQL)


EXPLAIN SELECT ...; -- План выполнения запроса
SHOW WARNINGS;      -- Показать предупреждения
SHOW ERRORS;        -- Показать ошибки
SET @variable = ...;-- Установка переменной
SELECT @variable;   -- Чтение переменной