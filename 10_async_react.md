# Занятие 10. Асинхронные запросы и компонентный интерфейс React

## Цель

Освоить Promise, `async`/`await`, HTTP-запросы и компонентную модель React. Создать интерфейс задач с загрузкой исходных данных и обработкой ошибок.

## Теория

HTTP связывает клиент и сервер запросами и ответами. Метод `GET` получает данные, `POST` создаёт ресурс, `PATCH` частично изменяет его, `DELETE` удаляет. Код состояния `200` означает успешный ответ, `201` — создание ресурса, `400` — ошибку запроса, `404` — отсутствие ресурса, `500` — ошибку сервера. Тело ответа часто передаётся в формате JSON. CORS ограничивает чтение ответов между разными источниками в браузере; его разрешение настраивается сервером, а не отключается клиентским кодом.

Асинхронные операции не останавливают интерфейс на время ожидания. `fetch()` возвращает Promise; выражение `await` ожидает результат внутри асинхронной функции. После завершения операции продолжение выполняется через очередь задач JavaScript. `fetch` не отклоняет Promise автоматически для HTTP 404 и 500: статус необходимо проверять самостоятельно.

**Пример 1. Загрузка JSON с проверкой ответа.** Создать файл `src/api.js`:

```javascript
export async function loadTasks(signal) {
  const response = await fetch('/tasks.json', {
    method: 'GET',
    headers: { Accept: 'application/json' },
    signal
  });
  if (!response.ok) {
    throw new Error(`Ошибка HTTP: ${response.status}`);
  }
  const data = await response.json();
  if (!Array.isArray(data) || data.length > 100) {
    throw new Error('Неверный формат ответа');
  }
  return data;
}
```

Обработка ошибки обычно выполняется через `try`/`catch`/`finally`. Для временной локальной ошибки сети не следует заменять успешно загруженные данные пустым списком. `AbortController` позволяет отменить запрос, если компонент больше не использует его результат.

React строит интерфейс из компонентов. `props` передают данные от родительского компонента, `useState` хранит изменяемое состояние, `useEffect` запускает побочный эффект, например загрузку данных. В JSX выражения заключаются в фигурные скобки; для списка нужны устойчивые ключи (`key`). Обработчики React получают функции, а не строки кода.

**Пример 2. Компонент списка.**

```jsx
export function TaskList({ tasks, onToggle }) {
  if (tasks.length === 0) return <p>Записей нет</p>;
  return (
    <ul>
      {tasks.map(task => (
        <li key={task.id}>
          <span>{task.title}</span>
          <button type="button" onClick={() => onToggle(task.id)}>
            {task.completed ? 'Отменить выполнение' : 'Выполнить'}
          </button>
        </li>
      ))}
    </ul>
  );
}
```

React экранирует текстовое содержимое JSX. Ввод строки пользователя не должен приводить к выполнению HTML-кода. Не изменять массив состояния непосредственно через `push()` и изменение свойств объектов: для корректного обновления создавать новую коллекцию.

**Пример 3. Асинхронная загрузка в компоненте.**

```jsx
import { useEffect, useState } from 'react';
import { loadTasks } from './api.js';

export function App() {
  const [tasks, setTasks] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState('');

  useEffect(() => {
    const controller = new AbortController();
    (async () => {
      try {
        const incoming = await loadTasks(controller.signal);
        if (!controller.signal.aborted) setTasks(incoming);
      } catch (err) {
        if (!controller.signal.aborted) setError(err.message);
      } finally {
        if (!controller.signal.aborted) setLoading(false);
      }
    })();
    return () => controller.abort();
  }, []);

  if (loading) return <p>Загрузка…</p>;
  if (error) return <p role="alert">{error}</p>;
  return <p>Загружено: {tasks.length}</p>;
}
```

Компонент выше демонстрирует только загрузку; отображение, поиск и изменение задач реализуются в задании. Разные компоненты могут владеть состоянием, однако общий список удобно хранить у ближайшего общего родителя и передавать обработчики вниз.

## Подготовка

Требуются Node.js (поддерживаемая LTS-версия), npm, браузер и редактор. В новой папке выполнить:

```bash
npm create vite@latest practice-10 -- --template react
cd practice-10
npm install
npm run dev
```

В каталоге `public` создать `tasks.json` с тремя задачами, например:

```json
[
  {"id":1,"title":"Прочитать документацию","completed":false},
  {"id":2,"title":"Создать компонент","completed":true},
  {"id":3,"title":"Проверить загрузку","completed":false}
]
```

Адрес `/tasks.json` обслуживается локальным сервером разработки Vite. В `src` предусмотреть файлы `api.js`, `App.jsx` и `TaskList.jsx`. В интерфейсе выделить поле поиска, фильтр, список, сообщение о загрузке и поле ошибки. Подготовить инструменты разработчика браузера (вкладки Console и Network).

## Источники

1. MDN Web Docs. [Using the Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch).
2. MDN Web Docs. [async function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function).
3. MDN Web Docs. [Cross-Origin Resource Sharing](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS).
4. React. [State: A Component's Memory](https://react.dev/learn/state-a-components-memory).
5. React. [Synchronizing with Effects](https://react.dev/learn/synchronizing-with-effects).
6. Vite. [Getting Started](https://vite.dev/guide/).

## Задание

Разработать React-приложение «Просмотр задач» с асинхронным получением данных из локального HTTP-источника.

| Параметр | Требование |
| --- | --- |
| Исходные данные | JSON-массив, от 0 до 100 записей |
| ID | Уникальное положительное безопасное целое число |
| Название | Строка от 1 до 80 символов после `trim()` |
| Статус | `completed: boolean` |
| Поиск | До 80 символов, без учёта регистра |
| Фильтр | Все / выполненные / невыполненные |
| Ответ | Проверка HTTP-статуса и структуры JSON |

1. Реализовать `loadTasks()` в отдельном модуле, проверку ответа и отображение состояния «Загрузка».
2. Разместить отображение задач в отдельном компоненте `TaskList`. Передавать данные через `props`; использовать `key={task.id}`.
3. Добавить управляемое поле поиска и фильтр статуса с совместной работой условий. Показывать число отображённых задач.
4. Добавить действие изменения статуса задачи в памяти React с помощью `setTasks` и создания нового массива. Повторный запрос не должен добавлять дубликаты.
5. Предусмотреть кнопку «Повторить загрузку», показывать текст ошибки и уметь восстановить успешную загрузку после устранения причины.
6. Проверять каждую запись полученного JSON, в том числе повтор идентификатора и тип `completed`. При неверном ответе не выводить частично разобранный список.
7. Открыть вкладку Network, зафиксировать метод, URL, HTTP-код и время одного запроса. Объяснить различие между `fetch('/tasks.json')` и чтением файла напрямую с диска.

**Проверка:** три корректные задачи; пустой массив; поиск без результатов; двойное изменение статуса; некорректный JSON; повтор ID; HTTP 404 после временного переименования файла; исправление файла и повторная загрузка. Ошибка загрузки должна быть показана без зависания интерфейса.

**Представить:** исходный код React-проекта без `node_modules`, `public/tasks.json`, инструкцию запуска и результаты проверки.
