# Занятие 12. Разработка HTTP API на Node.js и Express

## Цель

Освоить серверную среду Node.js, маршрутизацию Express и принципы REST. Разработать API для чтения и изменения списка задач с проверкой входных данных и корректными HTTP-ответами.

## Теория

### 1. Клиент, сервер и API: схема взаимодействия

В занятиях 9–11 JavaScript работал в браузере. Но браузер не должен напрямую управлять общей базой данных: в многопользовательском приложении операции выполняет сервер. **Клиент** показывает интерфейс и отправляет запрос, **сервер** проверяет данные и исполняет операцию, **API** определяет доступные адреса и формат запросов.

```text
Пользователь
    |
    v
Браузер или REST-клиент
    | HTTP GET /api/tasks
    v
Express (Node.js)
    | проверка запроса и получение массива
    v
JSON-ответ { "items": [...] }
```

На занятии данные временно будут храниться в массиве Node.js. После перезапуска процесса они исчезнут. На следующем занятии массив заменится PostgreSQL, при этом адреса API останутся прежними.

### 2. Первый сервер на чистом Node.js

Node.js запускает JavaScript-файлы так же, как браузер запускает скрипты страницы, но имеет дополнительные возможности для сети и файлов. В терминале проверьте `node --version` и `npm --version`. Создайте `hello-server.js`:

```javascript
import http from 'node:http';

const server = http.createServer((request, response) => {
  response.writeHead(200, { 'Content-Type': 'text/plain; charset=utf-8' });
  response.end('Сервер работает');
});

server.listen(3000, '127.0.0.1', () => {
  console.log('http://127.0.0.1:3000');
});
```

Для этого файла нужен режим ES-модулей, включённый через `"type": "module"` в `package.json` либо расширение `.mjs`. Откройте адрес в браузере. Сервер продолжает работать, пока его не остановить `Ctrl+C` в терминале. Express сокращает количество ручной обработки HTTP, поэтому далее используйте именно его.

### 3. npm, зависимости, модули и запуск проекта

`npm` — менеджер библиотек Node.js. Файл `package.json` описывает проект, пакет `express` устанавливается в папку `node_modules`, а `package-lock.json` фиксирует разрешённые версии зависимостей. Команда `npm install` устанавливает библиотеки; команды `npm init -y` и `npm pkg set type=module` создают и настраивают проект.

```bash
mkdir practice-12
cd practice-12
npm init -y
npm pkg set type=module
npm install express
```

В `package.json` можно задать команду запуска:

```json
{
  "type": "module",
  "scripts": { "start": "node server.js" },
  "dependencies": { "express": "^5.0.0" }
}
```

Это **пример структуры**, а не файл для копирования поверх сгенерированного npm: установленная версия `express` может отличаться; сохраняйте ту, что записал `npm install`. После добавления `scripts.start` запуск выполняется через `npm start`.

В ES-модулях `export` объявляет доступную из другого файла функцию, `import` подключает её:

```javascript
// validation.js
export function normalizeTitle(value) {
  if (typeof value !== 'string') return null;
  const result = value.trim();
  return result.length >= 1 && result.length <= 80 ? result : null;
}

```

В другом файле, `server.js`, подключите экспортированную функцию:

```javascript
import { normalizeTitle } from './validation.js';
```

### 4. HTTP-запрос и ответ подробно

URL `http://127.0.0.1:3000/api/tasks/7` состоит из схемы `http`, адреса сервера `127.0.0.1`, порта `3000` и пути `/api/tasks/7`. У HTTP-запроса есть метод, заголовки и иногда тело. JSON-тело должно сопровождаться заголовком `Content-Type: application/json`.

Пример запроса:

```http
POST /api/tasks HTTP/1.1
Host: 127.0.0.1:3000
Content-Type: application/json

{"title":"Создать макет","categoryId":2}
```

Пример ответа сервера:

```http
HTTP/1.1 201 Created
Content-Type: application/json; charset=utf-8

{"id":4,"title":"Создать макет","categoryId":2,"completed":false}
```

Методы GET, POST, PATCH, DELETE описывают назначение действия. Код `201` подходит для успешного создания, `204` — для успешного удаления без тела, `400` — для неверных данных, `404` — для отсутствующей записи, `500` — для непредвиденной ошибки приложения.

### 5. Маршрут и параметры в Express

Маршрут связывает HTTP-метод и адрес с JavaScript-функцией:

```javascript
app.get('/api/tasks/:id', (req, res) => {
  console.log(req.params.id);
  res.json({ receivedId: req.params.id });
});
```

Выражение `:id` означает **параметр пути**. Даже если URL содержит `/7`, `req.params.id` будет строкой `'7'`. Поисковые параметры после знака `?` доступны через `req.query`:

```text
GET /api/tasks?q=сайт&completed=false
```

В этом случае `req.query.q` содержит строку поиска, а `req.query.completed` — строку `'false'`, а не логическое значение `false`. Тело JSON после подключения `app.use(express.json())` становится доступно через `req.body`.

| Источник | Пример | Как прочитать |
| --- | --- | --- |
| Параметр пути | `/api/tasks/7` | `req.params.id` |
| Строка запроса | `?q=сайт` | `req.query.q` |
| Тело запроса | `{ "title": "Сайт" }` | `req.body.title` |
| Заголовок | `Content-Type: application/json` | `req.get('Content-Type')` |

### 6. Middleware: выполнение до маршрута

Middleware — функция, которая может прочитать запрос, изменить его или передать управление следующей функции через `next()`. `express.json()` — готовое middleware, которое разбирает JSON-тело. Подключать его нужно **до маршрутов**, в которых используется `req.body`.

```javascript
import express from 'express';
const app = express();
app.use(express.json({ limit: '32kb' }));
app.get('/api/health', (req, res) => res.json({ status: 'ok' }));
```

Порядок middleware важен: обработчик неизвестного маршрута размещают после объявленных маршрутов, а обработчик ошибок — в конце цепочки. При отправке ошибочного JSON необходимо отвечать `400`, а не выводить HTML с внутренними подробностями ошибки.

### 7. Проверка входных данных на стороне сервера

Проверка в браузере повышает удобство, но не защищает API: любой пользователь может отправить запрос с другим телом напрямую. Сервер обязан независимо проверять тип, диапазон и допустимые значения каждого поля.

```javascript
export function parsePositiveId(text) {
  if (typeof text !== 'string' || !/^[1-9]\d*$/.test(text)) return null;
  const id = Number(text);
  return Number.isSafeInteger(id) ? id : null;
}

export function normalizeTitle(value) {
  if (typeof value !== 'string') return null;
  const title = value.trim();
  return title.length >= 1 && title.length <= 80 ? title : null;
}
```

**Почему нельзя делать `if (req.body.completed)`?** Значение `false` — корректный статус, но условие для него не выполнится. Нужна явная проверка: `typeof req.body.completed === 'boolean'`.

**Почему нельзя применять `Number(raw)` без регулярного выражения?** `Number('')` даёт ноль, а `Number('1e2')` — сто; такие строки могут не соответствовать договорённому формату ID.

### 8. Отдельные операции CRUD

CRUD обозначает создание (Create), чтение (Read), изменение (Update), удаление (Delete). Для объекта `tasks` удобно использовать следующие структуры:

```javascript
const categories = [
  { id: 1, name: 'Учёба' },
  { id: 2, name: 'Проект' },
  { id: 3, name: 'Другое' }
];
let tasks = [];
let nextId = 1;
```

**Создание:** проверить `title` и `categoryId`, убедиться, что число записей меньше 100, создать объект с новым ID и `completed: false`, добавить его и вернуть статус `201`. **Чтение:** выбрать все или одну запись; использовать `404`, если запись по ID не существует. **Изменение:** проверить `completed`, заменить объект в массиве через `map`. **Удаление:** удалить через `filter` и ответить `res.status(204).end()` — тело отсутствует.

Например, обработчик удаления может использовать сравнение размеров:

```javascript
app.delete('/api/tasks/:id', (req, res) => {
  const id = parsePositiveId(req.params.id);
  if (id === null) return res.status(400).json({ error: 'invalid_id' });
  if (!tasks.some(task => task.id === id)) {
    return res.status(404).json({ error: 'not_found' });
  }
  tasks = tasks.filter(task => task.id !== id);
  return res.status(204).end();
});
```

### 9. Фильтрация в API

Для `GET /api/tasks` удобно начать с исходного массива, а затем последовательно применить оба фильтра. Строковые параметры нужно проверять перед использованием:

```javascript
app.get('/api/tasks', (req, res) => {
  const rawCompleted = req.query.completed;
  const q = req.query.q ?? '';

  if (rawCompleted !== undefined && !['true', 'false'].includes(rawCompleted)) {
    return res.status(400).json({ error: 'invalid_completed_filter' });
  }
  if (typeof q !== 'string' || q.length > 80) {
    return res.status(400).json({ error: 'invalid_query' });
  }

  let result = tasks;
  if (rawCompleted !== undefined) {
    const expected = rawCompleted === 'true';
    result = result.filter(task => task.completed === expected);
  }
  const needle = q.trim().toLocaleLowerCase();
  result = result.filter(task => task.title.toLocaleLowerCase().includes(needle));
  return res.json({ items: result });
});
```

Оба фильтра действуют совместно, а исходный массив `tasks` остаётся неизменным. Если параметры `q` или `completed` переданы несколькими значениями, их следует отклонить как некорректные.

### 10. Проверка API без пользовательского интерфейса

API можно проверить из терминала. В Windows PowerShell используйте `curl.exe`, чтобы избежать возможных различий с псевдонимом `curl`.

```powershell
curl.exe -i http://127.0.0.1:3000/api/health
curl.exe -i http://127.0.0.1:3000/api/tasks
curl.exe -i -X POST http://127.0.0.1:3000/api/tasks -H "Content-Type: application/json" -d '{"title":"Отчёт","categoryId":1}'
```

В разных оболочках правила экранирования кавычек могут различаться. Если передача JSON через `-d` вызывает проблему в PowerShell, сохраните тело в файл `request.json` и используйте `--data-binary "@request.json"`:

```powershell
curl.exe -i -X POST http://127.0.0.1:3000/api/tasks -H "Content-Type: application/json" --data-binary "@request.json"
```

Содержимое `request.json`: `{"title":"Отчёт","categoryId":1}`. Для проверки изменения используйте `-X PATCH`, для удаления — `-X DELETE`. После каждого запроса фиксируйте HTTP-статус и фактическое тело ответа. `204` означает, что тело отсутствует — это нормальное поведение.

### 11. Типичные ошибки сервера

| Ошибка | Причина | Проверка |
| --- | --- | --- |
| `Cannot find package 'express'` | Зависимость не установлена | Выполнить `npm install` в правильной папке |
| `Cannot use import statement outside a module` | Не включён ES-модуль | Указать `"type": "module"` |
| `EADDRINUSE` | Порт уже занят другим процессом | Остановить старый сервер или использовать другой порт |
| `req.body` равно `undefined` | Нет middleware или неверный Content-Type | Подключить `express.json()` и заголовок |
| `Cannot GET /api/...` | Нет соответствующего маршрута | Проверить адрес, метод и порядок объявления |
| Сервер отвечает `500` для пустого поля | Нет валидации или обработчика ошибок | Добавить проверку и ответ `400` |
| При `204` клиент не может прочитать JSON | У ответа нет тела | Не вызывать `response.json()` для `204` |

Перед основным заданием выполните три мини-упражнения: получите JSON от `/api/health`, добавьте маршрут с параметром ID, отправьте JSON в тестовый POST и выведите `req.body` в консоль. Только после этого разрабатывайте полный контракт API.

### Дополнительные примеры и пояснения


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

### Порядок выполнения

**Шаг 1.** Создайте Node.js-проект, установите Express и включите ES-модули. Создайте `server.js`, `validation.js`, `.gitignore`, `README.md`. Убедитесь, что `GET /api/health` возвращает статус `200` и объект `{ "status": "ok" }`.

**Шаг 2.** Определите массив категорий и задач, а также переменную следующего ID. Сначала реализуйте только маршруты чтения: `/api/categories`, `/api/tasks`, `/api/tasks/:id`. Убедитесь, что строковый ID превращается в безопасное целое число.

**Шаг 3.** Реализуйте POST с проверкой тела и кодом `201`. Испытайте корректную задачу, пустое название, неверную категорию и строковый `categoryId`.

**Шаг 4.** Добавьте PATCH и DELETE. После PATCH повторный GET должен показывать обновлённый статус; после DELETE GET удалённого ID должен вернуть `404`.

**Шаг 5.** Реализуйте фильтры `q` и `completed`. При неверном значении фильтра возвращайте `400`; при отсутствии совпадений — `200` с пустым массивом `items`.

**Шаг 6.** Добавьте обработку неизвестного маршрута и ошибочного JSON. Создайте README с точными командами. Проведите испытания, не ограничиваясь одним успешным запросом.

**Минимальный результат на занятии:** сервер запускается, отдаёт категории и задачи, принимает POST с проверкой. **Полный результат:** реализован весь контракт, фильтрация и отрицательные сценарии.

### Контрольные вопросы

1. Как различаются функции браузера и Node.js?
2. Что хранится в `package.json`, а что в `node_modules`?
3. В чём разница между параметром пути `:id` и параметром запроса `?q=...`?
4. Для чего нужен `express.json()`?
5. Почему `req.body` нельзя считать достоверным без проверки?
6. Когда возвращать `201`, `204`, `400`, `404`?
7. Почему запрос `GET /api/tasks?completed=false` нельзя проверить условием `if (req.query.completed)`?
8. Почему список в памяти сервера исчезает после перезапуска?

