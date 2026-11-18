# Занятие 13. PostgreSQL, тестирование и подготовка веб-приложения к запуску

## Цель

Освоить создание связанных таблиц PostgreSQL и безопасное обращение к ним из Node.js. Реализовать постоянное хранение данных, автоматизированные проверки и воспроизводимую конфигурацию приложения.

## Теория

Реляционная база данных хранит связанные записи в таблицах. Первичный ключ (`PRIMARY KEY`) идентифицирует запись, внешний ключ (`REFERENCES`) устанавливает связь, `NOT NULL`, `UNIQUE` и `CHECK` ограничивают допустимые данные. SQL-команды `SELECT`, `INSERT`, `UPDATE`, `DELETE` выполняют чтение и изменение записей. `JOIN` объединяет связанные таблицы, а сортировка и фильтрация задаются через `ORDER BY` и `WHERE`.

**Пример 1. Схема для задач и категорий.** Создать `sql/001_schema.sql`:

```sql
CREATE TABLE IF NOT EXISTS categories (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name varchar(40) NOT NULL UNIQUE,
    CONSTRAINT category_name_length CHECK (length(btrim(name)) BETWEEN 1 AND 40)
);

CREATE TABLE IF NOT EXISTS tasks (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title varchar(80) NOT NULL,
    category_id integer NOT NULL REFERENCES categories(id),
    completed boolean NOT NULL DEFAULT false,
    CONSTRAINT task_title_length CHECK (length(btrim(title)) BETWEEN 1 AND 80)
);

INSERT INTO categories (name)
VALUES ('Учёба'), ('Проект'), ('Другое')
ON CONFLICT (name) DO NOTHING;
```

После создания схемы получить список с названиями категорий:

```sql
SELECT t.id, t.title, t.completed,
       c.id AS category_id, c.name AS category_name
FROM tasks AS t
JOIN categories AS c ON c.id = t.category_id
ORDER BY t.id;
```

Модуль `pg` предоставляет пул соединений `Pool`. Значения пользователя передаются отдельными параметрами `$1`, `$2`, а не вставляются в SQL-строку. Это необходимо для правильной обработки данных и защиты от SQL-инъекций.

**Пример 2. Параметризованный запрос.** Создать `db.js`:

```javascript
import pg from 'pg';

export const pool = new pg.Pool({
  connectionString: process.env.DATABASE_URL,
  max: 5,
  connectionTimeoutMillis: 3000
});

export async function createTask(title, categoryId) {
  const result = await pool.query(
    `INSERT INTO tasks (title, category_id)
     VALUES ($1, $2)
     RETURNING id, title, category_id AS "categoryId", completed`,
    [title, categoryId]
  );
  return result.rows[0];
}
```

Запрос выполняется после проверки входных данных на уровне API. Если база недоступна, сервер должен вернуть контролируемый ответ `503`, а не бесконечно ожидать соединения. Оператор `RETURNING` позволяет получить созданную запись. Связанные изменения нескольких таблиц выполняют в транзакции `BEGIN` → команды → `COMMIT`; при ошибке выполняют `ROLLBACK` через одно и то же соединение из пула.

Миграция — сохранённое изменение схемы, применяемое в определённом порядке. Для учебного приложения достаточно фиксированного SQL-файла с воспроизводимым созданием таблиц. Не удалять существующие данные при каждом запуске сервера. Учётные данные хранятся в переменных окружения, не в исходном коде и не в Git.

Проверка программ включает модульные тесты функций, интеграционные тесты HTTP API и отрицательные сценарии. В Node.js есть встроенный модуль `node:test`. Тест следует делать независимым от порядка выполнения других проверок.

**Пример 3. Модульный тест в `test/validation.test.js`.** Предполагается, что функция `normalizeTitle` из `validation.js` возвращает нормализованную строку или `null`:

```javascript
import test from 'node:test';
import assert from 'node:assert/strict';
import { normalizeTitle } from '../validation.js';

test('обрезка пробелов в названии', () => {
  assert.equal(normalizeTitle('  Задача  '), 'Задача');
});

test('пустое название недопустимо', () => {
  assert.equal(normalizeTitle('   '), null);
});

test('длина названия ограничена', () => {
  assert.equal(normalizeTitle('А'.repeat(81)), null);
});
```

В процессе подготовки приложения к запуску проверяют зависимости, отсутствие секретов в репозитории, доступность сервера, корректность сборки интерфейса и README. CI автоматически выполняет тесты при изменениях; контейнеризация описывает окружение приложения как воспроизводимую конфигурацию. В рамках занятия достаточно понимать назначение этих средств, а не создавать промышленную инфраструктуру.

## Подготовка

Потребуются Node.js, npm, локальный PostgreSQL или выданная преподавателем учебная база. Создать отдельную пустую базу `tip_tasks` и пользователя с правами на неё. В каталоге сервера занятия 12 установить `pg`:

```bash
npm install pg
```

Создать `sql/001_schema.sql`, `db.js`, `validation.js`, `test/validation.test.js`, `.env.example`, `.gitignore`. В `.env.example` указать строку-заготовку без настоящего пароля:

```dotenv
DATABASE_URL=postgresql://USER:PASSWORD@127.0.0.1:5432/tip_tasks
PORT=3000
```

Создать собственный `.env` и исключить его из Git. Для запуска с загрузкой файла окружения в поддерживаемой версии Node.js использовать `node --env-file=.env server.js`. Проверить доступность `psql`, применить схему командой `psql -d tip_tasks -f sql/001_schema.sql` (при необходимости указать `-U` и `-h`) и запустить тесты командой `node --test`.

## Источники

1. PostgreSQL. [Data Definition](https://www.postgresql.org/docs/current/ddl.html).
2. PostgreSQL. [Queries](https://www.postgresql.org/docs/current/queries.html).
3. node-postgres. [Queries](https://node-postgres.com/features/queries).
4. node-postgres. [Transactions](https://node-postgres.com/features/transactions).
5. Node.js. [Test runner](https://nodejs.org/api/test.html).
6. Node.js. [Environment variables](https://nodejs.org/api/environment_variables.html).
7. GitHub Docs. [Understanding GitHub Actions](https://docs.github.com/en/actions/get-started/understand-github-actions).

## Задание

Расширить сервер «API учебных задач» постоянным хранением в PostgreSQL и базовыми проверками качества.

| Объект | Требование |
| --- | --- |
| `categories` | Уникальный ID, уникальное название 1–40 символов |
| `tasks` | Уникальный ID, название 1–80 символов, внешний ключ категории, логический статус |
| Связь | Каждая задача относится ровно к одной существующей категории |
| SQL | Параметры запросов `$1`, `$2`, … для пользовательских значений |
| Конфигурация | `DATABASE_URL` и `PORT` через переменные окружения |
| Тесты | Не менее четырёх модульных и трёх HTTP-проверок |

1. Создать таблицы и ограничения по схеме. Подготовить не менее трёх категорий и четырёх задач. Проверить невозможность вставки задачи с несуществующим `category_id`.
2. Подключить `pg.Pool` к Express-приложению. Заменить хранение задач в массиве на SQL-запросы для `GET /api/tasks`, `POST /api/tasks`, `PATCH /api/tasks/:id` и `DELETE /api/tasks/:id`. Маршрут категорий должен читать категории из базы.
3. В ответе списка задач возвращать `{ "items": [...] }` с полями `id`, `title`, `categoryId`, `categoryName`, `completed`. Сортировать по ID; сохранить поддержку фильтров `q` и `completed`.
4. Для создания и изменения данных использовать параметры SQL. Проверять входные значения до выполнения запроса. Для нарушения внешнего ключа возвращать `400`, для отсутствующей записи — `404`, для недоступной БД — `503`.
5. Реализовать модульные тесты проверки названия, ID и допустимой категории: не менее четырёх сценариев, включая отрицательные.
6. Выполнить не менее трёх HTTP-проверок: успешное создание; запрос несуществующего ID; отклонение ошибочного тела запроса. Зафиксировать ожидаемый и фактический HTTP-код.
7. Перезапустить сервер и убедиться, что ранее созданные записи сохранились. Проверить инструкцию развёртывания на чистой учебной базе, не публиковать `.env`.
8. Описать в README порядок установки зависимостей, создания базы, применения SQL и запуска тестов. Объяснить назначение миграций, транзакций, автоматической сборки и контейнеризации.

**Проверка:** выборка с названием категории; добавление и изменение задачи; удаление; поиск и фильтрация; ошибочный ID; неизвестная категория; повторный запуск; неработающая БД; проверка параметризованного запроса строкой `x' OR '1'='1` как обычным текстом названия. Выполнить `node --test` и записать количество успешных проверок.

**Представить:** код сервера, SQL-файл, тесты, `.env.example`, `package.json`, `package-lock.json`, README и результаты испытаний.
