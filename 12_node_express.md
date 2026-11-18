# Занятие 12. Разработка HTTP API на Node.js и Express

## Цель

Освоить серверную среду Node.js, маршрутизацию Express и принципы REST. Разработать API для чтения и изменения списка задач с проверкой входных данных и корректными HTTP-ответами.

## Теория

Node.js выполняет JavaScript вне браузера, предоставляет доступ к сети и файловой системе. Пакет `express` облегчает создание HTTP-сервера. Сервер принимает запрос, сопоставляет его с маршрутом, обрабатывает параметры и возвращает ответ. `middleware` — функция промежуточной обработки запроса. Вызов `express.json()` позволяет разобрать JSON-тело до обращения к маршрутам.

Для коллекции задач используется адрес `/api/tasks`, для отдельной задачи — `/api/tasks/:id`. Метод `GET` читает, `POST` создаёт, `PATCH` изменяет и `DELETE` удаляет данные. Рекомендуемые ответы: `200` с данными, `201` после создания, `204` при удалении без тела, `400` при ошибке формата, `404` при отсутствии записи. Сервер проверяет ввод независимо от проверок браузера.

**Пример 1. Создание сервера и получение данных.** В `package.json` необходимо указать `"type": "module"`:

```javascript
import express from 'express';

const app = express();
app.use(express.json({ limit: '32kb' }));

let tasks = [
  { id: 1, title: 'Составить план', categoryId: 1, completed: false }
];

app.get('/api/health', (req, res) => {
  res.json({ status: 'ok' });
});

app.get('/api/tasks', (req, res) => {
  res.json({ items: tasks });
});

app.listen(3000, '127.0.0.1', () => {
  console.log('API: http://127.0.0.1:3000');
});
```

Для числового ID недостаточно применять `Number()` без проверки: `Number('')` возвращает 0, а дробные значения и отрицательные числа не соответствуют формату идентификатора. Для строкового параметра использовать проверку регулярным выражением и `Number.isSafeInteger`.

**Пример 2. Проверка ID и добавление записи.** Фрагмент помещается до `app.listen`:

```javascript
function parseId(raw) {
  if (!/^[1-9]\d*$/.test(raw)) return null;
  const id = Number(raw);
  return Number.isSafeInteger(id) ? id : null;
}

function validateTaskInput(body) {
  if (!body || typeof body !== 'object' || Array.isArray(body)) return null;
  if (typeof body.title !== 'string') return null;
  const title = body.title.trim();
  if (title.length < 1 || title.length > 80) return null;
  if (!Number.isSafeInteger(body.categoryId) || body.categoryId < 1) return null;
  return { title, categoryId: body.categoryId };
}

let nextId = 2;
app.post('/api/tasks', (req, res) => {
  const data = validateTaskInput(req.body);
  if (!data || ![1, 2, 3].includes(data.categoryId)) {
    return res.status(400).json({ error: 'invalid_task' });
  }
  const task = { id: nextId++, ...data, completed: false };
  tasks = [...tasks, task];
  return res.status(201).json(task);
});
```

В полноценной версии допустимые категории должны проверяться по массиву `categories`, а идентификатор создаваться сервером. Ответ об ошибке лучше представлять стабильным объектом `{ "error": "..." }`, а не необработанным HTML исключения. Не раскрывать клиенту стеки вызовов, пароли и данные конфигурации.

**Пример 3. Изменение статуса.**

```javascript
app.patch('/api/tasks/:id', (req, res) => {
  const id = parseId(req.params.id);
  if (id === null || typeof req.body?.completed !== 'boolean') {
    return res.status(400).json({ error: 'invalid_request' });
  }
  const found = tasks.find(task => task.id === id);
  if (!found) return res.status(404).json({ error: 'not_found' });
  tasks = tasks.map(task =>
    task.id === id ? { ...task, completed: req.body.completed } : task
  );
  return res.json(tasks.find(task => task.id === id));
});
```

Клиентский React-проект и сервер, запущенные на разных портах, принадлежат разным источникам. Для последующей интеграции можно настроить прокси в Vite: браузер обращается к `/api`, а сервер Vite перенаправляет запросы на Express. Это позволяет избежать изменения политики браузера и ненужного открытия CORS.

## Подготовка

Проверить работу Node.js и npm. Создать каталог `practice-12`, выполнить:

```bash
npm init -y
npm pkg set type=module
npm install express
```

Создать `server.js`, файлы `.gitignore` и `README.md`. В `.gitignore` добавить `node_modules/`, `.env`, `dist/`. Сервер запускать командой `node server.js`, проверять в браузере `http://127.0.0.1:3000/api/health`. Для запросов с телом использовать REST Client, Postman или `curl`; в PowerShell обращаться к внешней утилите как `curl.exe`. Данные в этом занятии хранятся только в памяти, после перезапуска сервера восстанавливается стартовый набор.

## Источники

1. Node.js. [Introduction to Node.js](https://nodejs.org/en/learn/getting-started/introduction-to-nodejs).
2. Express. [Routing](https://expressjs.com/en/guide/routing.html).
3. Express. [Using middleware](https://expressjs.com/en/guide/using-middleware.html).
4. MDN Web Docs. [HTTP request methods](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods).
5. MDN Web Docs. [HTTP response status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status).

## Задание

Разработать сервер «API учебных задач».

| Поле / параметр | Допустимое значение |
| --- | --- |
| ID задачи | Уникальное положительное безопасное целое число |
| `title` | Строка из 1–80 символов после `trim()` |
| `categoryId` | Целое: 1, 2 или 3 |
| `completed` | `true` или `false` |
| Количество задач | 0–100 |
| Поисковый параметр `q` | 0–80 символов |
| Параметр `completed` | Отсутствует, `true` или `false` |

Необходимые маршруты:

| Метод | Адрес | Результат |
| --- | --- | --- |
| GET | `/api/health` | `{ "status": "ok" }` |
| GET | `/api/categories` | Массив трёх объектов `{ id, name }` |
| GET | `/api/tasks` | `{ "items": [...] }` с необязательными фильтрами `q`, `completed` |
| GET | `/api/tasks/:id` | Одна задача или `404` |
| POST | `/api/tasks` | Создание задачи, `201` и созданный объект |
| PATCH | `/api/tasks/:id` | Изменение `completed`, `200` и обновлённый объект |
| DELETE | `/api/tasks/:id` | Удаление, `204` без тела |

1. Реализовать сервер Express, JSON-middleware, перечисленные маршруты и единообразные ответы об ошибках.
2. Организовать отдельные функции проверки ID, названия и тела запроса. При неизвестной категории, неверном типе поля или превышении лимита возвращать `400` без изменения массива.
3. Реализовать поиск по подстроке без учёта регистра и фильтрацию по статусу; оба фильтра могут применяться совместно.
4. Для несуществующего ID возвращать `404`; при удалении возвращать `204`, не отправляя JSON-тело.
5. Реализовать `404` для неизвестного адреса и обработку ошибочного JSON с кодом `400`.
6. Добавить в README команды запуска и примеры запросов. Создать минимум три задачи с разными категориями и статусами.

**Проверка:** выполнить все семь маршрутов; создать запись; изменить её статус; удалить её; убедиться в `404` при повторном удалении. Проверить пустое название, `categoryId: "1"`, неизвестную категорию, дробный ID, некорректный JSON, фильтр без совпадений и одновременное применение фильтров. Зафиксировать HTTP-коды и тела ответов.

**Представить:** исходные файлы сервера, `package.json`, `package-lock.json`, `.gitignore`, README и таблицу результатов HTTP-запросов.
