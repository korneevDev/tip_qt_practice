# Занятие 9. Основы JavaScript, DOM и локальное хранение данных

## Цель

Освоить основные конструкции JavaScript, работу с объектами и массивами, обработку событий браузера и сохранение данных. Разработать интерактивный список учебных задач.

## Теория

### 1. Как устроена программа на JavaScript

JavaScript — язык программирования, на котором описывают поведение веб-страницы. HTML отвечает за структуру (поля, кнопки, заголовки), CSS — за внешний вид, а JavaScript — за действия пользователя и обработку данных. До выполнения этой работы знать JavaScript не требуется: необходимые конструкции рассматриваются далее.

Программа может выполняться в двух средах:

- **Браузер:** можно получать элементы HTML через `document`, реагировать на нажатия и сохранять небольшие данные в `localStorage`.
- **Node.js:** можно запускать `.js`-файлы из терминала. Здесь нет браузерного `document`, но доступны средства для серверных программ и запуска проверок.

Для первого знакомства создайте `hello.js`:

```javascript
const student = 'Ирина';
const practice = 9;
console.log(`Студент: ${student}. Практическая работа №${practice}`);
```

В терминале из папки файла выполните `node hello.js`. Команда `console.log()` выводит информацию в терминал Node.js или в консоль разработчика браузера. Обратные кавычки образуют **шаблонную строку**: `${practice}` заменяется значением переменной. Однострочный комментарий начинается с `//`, многострочный заключается между `/*` и `*/`.

### 2. Переменные, типы данных и приведение типов

Переменная — именованное место для хранения значения. `const` не позволяет присвоить переменной другое значение, `let` позволяет. Для объектов `const` запрещает повторное присваивание самой переменной, но не делает её содержимое неизменяемым.

```javascript
const title = 'Лабораторная работа'; // string
let quantity = 3;                 // number
const completed = false;          // boolean
const empty = null;               // специальное пустое значение
let notAssigned;                   // undefined

quantity = quantity + 1;
console.log(typeof title);       // string
console.log(quantity);           // 4
```

Важнейшие операторы: `+`, `-`, `*`, `/`, `%` (остаток), `>`, `<`, `>=`, `<=`, `===` (строгое равенство), `!==` (строгое неравенство), `&&` (И), `||` (ИЛИ), `!` (НЕ). Строгое сравнение не пытается автоматически сделать число из строки:

```javascript
console.log(5 === '5'); // false
console.log(5 == '5');  // true, такое нестрогое сравнение лучше не применять
console.log('5' + 2);  // '52': строка + число приводит к склеиванию
console.log(Number('5') + 2); // 7
console.log(Number('abc'));   // NaN — не число
```

Поля HTML-форм **всегда возвращают строку**, даже если пользователь вводит цифры. При необходимости преобразуйте строку явно и проверьте результат:

```javascript
const rawId = '12';
const id = Number(rawId);
if (!Number.isSafeInteger(id) || id < 1) {
  console.error('Некорректный идентификатор');
} else {
  console.log('ID:', id);
}
```

| Тип | Пример | Для чего используется в журнале задач |
| --- | --- | --- |
| `string` | `'Сдать отчёт'` | Название задачи |
| `number` | `12` | Идентификатор |
| `boolean` | `false` | Выполнена ли задача |
| `object` | `{ id: 1, title: '...' }` | Одна запись |
| `Array` | `[{ id: 1 }, { id: 2 }]` | Список записей |
| `null` | `null` | Явное отсутствие значения |

### 3. Условия, циклы, функции и область видимости

Оператор `if` выбирает действие по условию. `for...of` обходит значения массива. `return` завершает функцию и возвращает результат.

```javascript
function getStatusText(completed) {
  if (completed) {
    return 'Выполнена';
  }
  return 'Не выполнена';
}

const statuses = [true, false, true];
for (const status of statuses) {
  console.log(getStatusText(status));
}
```

У функции есть параметры — имена входных значений. Если объявить переменную внутри функции, снаружи она недоступна. Отдельные функции лучше делать небольшими: одна проверяет название, другая добавляет запись, третья обновляет HTML. В JavaScript распространён и краткий синтаксис стрелочных функций: `const double = value => value * 2;`. Запись `value => ...` означает функцию с одним параметром.

Для обработки ошибок применяют `throw`, `try`, `catch`:

```javascript
function requirePositiveInteger(value) {
  if (!Number.isSafeInteger(value) || value <= 0) {
    throw new Error('Число должно быть положительным целым');
  }
  return value;
}

try {
  console.log(requirePositiveInteger(-2));
} catch (error) {
  console.log(error.message); // сообщение, а не аварийное завершение интерфейса
}
```

### 4. Массивы, объекты и методы обработки коллекций

Массив хранит значения в порядке, а объект объединяет поля одной сущности. Нумерация элементов массива начинается с нуля.

```javascript
const task = { id: 1, title: 'Изучить JavaScript', completed: false };
const tasks = [task, { id: 2, title: 'Сделать форму', completed: true }];
console.log(tasks[0].title); // Изучить JavaScript
console.log(tasks.length);   // 2
```

На этой работе понадобятся пять методов массива:

| Метод | Назначение | Пример |
| --- | --- | --- |
| `find` | Найти первый элемент | `tasks.find(t => t.id === 2)` |
| `some` | Проверить наличие совпадения | `tasks.some(t => t.completed)` |
| `filter` | Получить новый массив подходящих элементов | `tasks.filter(t => !t.completed)` |
| `map` | Преобразовать каждый элемент в новом массиве | `tasks.map(t => t.title)` |
| `forEach` | Выполнить действие для каждого элемента | `tasks.forEach(t => console.log(t.id))` |

`filter` и `map` не изменяют исходный массив. Чтобы обновить одну задачу, удобнее получить новый список:

```javascript
const updated = tasks.map(t =>
  t.id === 1 ? { ...t, completed: true } : t
);
console.log(tasks[0].completed);   // false
console.log(updated[0].completed); // true
```

Оператор `...t` копирует перечислимые свойства объекта. После копирования выражение `completed: true` заменяет одно поле, не меняя старую запись. Для удаления используют `filter(t => t.id !== requiredId)`, для добавления — `[...tasks, newTask]`.

### 5. HTML: минимальные сведения, необходимые для работы с DOM

Тег `<form>` объединяет поля ввода, `<input>` создаёт поле, `<button>` — кнопку, `<ul>` и `<li>` — список. Атрибут `id` позволяет найти элемент из JavaScript. Достаточно следующей небольшой страницы:

```html
<!doctype html>
<html lang="ru">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Учебные задачи</title>
</head>
<body>
  <h1>Журнал учебных задач</h1>
  <form id="taskForm">
    <label for="titleInput">Задача</label>
    <input id="titleInput" maxlength="80" required>
    <button type="submit">Добавить</button>
  </form>
  <p id="message" role="status"></p>
  <ul id="taskList"></ul>
  <script type="module" src="./app.js"></script>
</body>
</html>
```

Атрибут `type="module"` позволяет использовать `import`/`export`. У формы по умолчанию есть отправка с перезагрузкой страницы. Чтобы обработать её средствами JavaScript, в обработчике события вызывают `event.preventDefault()`.

### 6. Работа с DOM: чтение данных, создание элементов, события

DOM — дерево объектов, соответствующих HTML-элементам. `document.getElementById('titleInput')` получает поле по `id`; его текущее содержимое хранится в свойстве `.value`. `textContent` задаёт текст, `.append()` добавляет дочерние элементы.

```javascript
const form = document.querySelector('#taskForm');
const input = document.querySelector('#titleInput');
const list = document.querySelector('#taskList');
const message = document.querySelector('#message');

form.addEventListener('submit', event => {
  event.preventDefault();
  const title = input.value.trim();
  if (title.length === 0) {
    message.textContent = 'Введите название задачи';
    return;
  }
  const item = document.createElement('li');
  item.textContent = title;
  list.append(item);
  input.value = '';
  message.textContent = 'Задача добавлена';
});
```

Проверка: откройте страницу, введите название, нажмите Enter. Новая запись должна появиться без перезагрузки страницы. Если изменить `textContent` на `innerHTML`, строка пользователя начнёт интерпретироваться как HTML. **Для названий задач используйте только `textContent`.**

При создании кнопок у каждой записи можно положить ID в `button.dataset.id`. События `click` от дочерних кнопок всплывают к родительскому списку; поэтому один обработчик на `<ul>` может обслуживать все динамически созданные кнопки.

### 7. JSON и localStorage: что переживает перезагрузку страницы

Переменная JavaScript существует лишь пока работает программа. Для простого локального приложения данные можно сохранять в `localStorage` браузера. Там хранятся **только строки**, поэтому применяют JSON:

```javascript
const sample = [{ id: 1, title: 'Пример', completed: false }];
const text = JSON.stringify(sample);
console.log(text); // '[{"id":1,"title":"Пример","completed":false}]'
const restored = JSON.parse(text);
console.log(restored[0].title); // Пример
```

Разница между `null` и пустым массивом существенна: `localStorage.getItem('key') === null` означает, что значение ещё не сохранено; строка `'[]'` — сохранённый пустой список. Запись `JSON.parse()` способна выбросить исключение при повреждённых данных. Успешный разбор не гарантирует корректность типов и уникальность ID.

```javascript
function readStoredTasks() {
  const raw = localStorage.getItem('student-tasks-v1');
  if (raw === null) return [];
  try {
    const result = JSON.parse(raw);
    if (!Array.isArray(result)) throw new Error('Ожидался массив');
    return result;
  } catch (error) {
    throw new Error(`Невозможно восстановить сохранение: ${error.message}`);
  }
}
```

Это лишь первый уровень проверки. В задании дополнительно проверьте поля каждой записи, число элементов и повторяющиеся ID. Не очищайте повреждённый ключ автоматически: для пользователя это единственная копия данных.

### 8. Небольшая последовательность для самостоятельного освоения

1. Запустите `hello.js` через Node.js и поменяйте значение переменной `practice`.
2. Создайте массив из трёх задач, выведите только невыполненные через `filter`.
3. Напишите `normalizeTitle`, которая возвращает `null` для пустой строки и строки длиннее 80 символов.
4. Создайте HTML-страницу и обработайте отправку формы, не перезагружая страницу.
5. Запишите массив в `localStorage`, обновите страницу и восстановите записи через `JSON.parse`.
6. Откройте инструменты разработчика (`F12`): в **Console** находятся сообщения и ошибки, в **Elements** — HTML, в **Application → Local Storage** — сохранённые значения.

На занятии следует довести до рабочего состояния добавление, вывод и переключение статуса. Поиск, фильтрацию и тщательную проверку повреждённого хранилища можно завершить как самостоятельную часть.

### 9. Частые ошибки начинающих

| Сообщение или ситуация | Возможная причина | Как исправить |
| --- | --- | --- |
| `ReferenceError: x is not defined` | Опечатка в имени переменной или переменная вне области видимости | Проверить имена и место объявления |
| `Cannot read properties of null` | Элемент с указанным `id` не найден | Сверить HTML и селектор, проверить подключение скрипта |
| `Unexpected token ... in JSON` | Хранилище содержит невалидный JSON | Показать сообщение; не очищать данные самовольно |
| Страница обновляется при отправке формы | Нет `event.preventDefault()` | Добавить его в обработчик `submit` |
| После обновления страницы задачи пропали | Изменения не записываются в `localStorage` | Вызывать сохранение после успешной операции |
| Строка `'12'` не равна числу `12` | Поле ввода возвращает строку | Явно преобразовать и проверить число |
| `Cannot use import statement outside a module` | Отсутствует `type="module"` или неверный запуск | Подключить модуль правильно и использовать локальный сервер |

### Дополнительные примеры и пояснения


JavaScript выполняется в браузере и в среде Node.js. Браузер предоставляет объект `document` для работы со страницей, а Node.js позволяет выполнять программы без браузера. Переменные объявляются через `const` и `let`; при обработке чисел важно явно проверять тип и диапазон, а при сравнении использовать `===`.

Объект объединяет связанные свойства, массив хранит последовательность элементов. Функции выделяют самостоятельные операции. Методы `filter`, `map` и `find` позволяют обрабатывать коллекции без повторения кода. Оператор `...` используется для копирования объекта при обновлении записи.

**Пример 1. Модель задач и функции обработки данных.** Создать файл `model.js`:

```javascript
export function normalizeTitle(value) {
  if (typeof value !== 'string') return null;
  const title = value.trim();
  return title.length >= 1 && title.length <= 80 ? title : null;
}

export function addTask(tasks, title, id) {
  const validTitle = normalizeTitle(title);
  if (!validTitle || !Number.isSafeInteger(id) || id < 1) {
    throw new Error('Некорректные данные задачи');
  }
  if (tasks.some(task => task.id === id)) {
    throw new Error('Идентификатор уже существует');
  }
  return [...tasks, { id, title: validTitle, completed: false }];
}

export function countCompleted(tasks) {
  return tasks.filter(task => task.completed).length;
}
```

Функция `addTask` возвращает новый массив. Это упрощает проверку результата и отделяет обработку данных от интерфейса. Пример вызова:

```javascript
import { addTask, countCompleted } from './model.js';
let tasks = addTask([], 'Подготовить проект', 1);
console.log(tasks[0].title, countCompleted(tasks));
```

DOM — объектное представление HTML-документа. Элементы находят методами `querySelector` и `getElementById`, события обрабатывают через `addEventListener`. Для пользовательского текста применяется `textContent`, а не вставка через `innerHTML`: так строка не интерпретируется как HTML-разметка. При делегировании событий обработчик назначается общему контейнеру, а нужная кнопка определяется через `closest`.

**Пример 2. Безопасный вывод списка и обработка событий.** В `index.html` предусмотрен элемент `<ul id="taskList"></ul>`:

```javascript
const taskList = document.querySelector('#taskList');

function render(tasks) {
  taskList.replaceChildren();
  for (const task of tasks) {
    const item = document.createElement('li');
    const title = document.createElement('span');
    title.textContent = task.title;

    const button = document.createElement('button');
    button.type = 'button';
    button.dataset.id = String(task.id);
    button.textContent = task.completed ? 'Вернуть' : 'Выполнено';
    item.append(title, button);
    taskList.append(item);
  }
}

taskList.addEventListener('click', event => {
  const button = event.target.closest('button[data-id]');
  if (!button || !taskList.contains(button)) return;
  const id = Number(button.dataset.id);
  // Найти задачу, изменить её статус, сохранить данные и вызвать render().
});
```

Объект `localStorage` хранит строковые значения между перезапусками страницы в рамках одного источника. Объекты преобразуются в текст с помощью `JSON.stringify`, восстанавливаются с помощью `JSON.parse`. Разбор JSON может завершиться ошибкой, а корректный синтаксис не гарантирует правильную структуру данных.

**Пример 3. Сохранение данных и проверка при загрузке.**

```javascript
const STORAGE_KEY = 'student-tasks-v1';

function saveTasks(tasks) {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(tasks));
}

function loadTasks() {
  const raw = localStorage.getItem(STORAGE_KEY);
  if (raw === null) return [];
  const parsed = JSON.parse(raw); // Ошибку разобрать в вызывающем коде.
  if (!Array.isArray(parsed) || parsed.length > 100) {
    throw new Error('Неверный формат хранилища');
  }
  const ids = new Set();
  for (const task of parsed) {
    if (!task || !Number.isSafeInteger(task.id) || task.id < 1 ||
        ids.has(task.id) || normalizeTitle(task.title) !== task.title ||
        typeof task.completed !== 'boolean') {
      throw new Error('Повреждённая запись');
    }
    ids.add(task.id);
  }
  return parsed;
}
```

При повреждённом сохранении программа должна показать сообщение об ошибке и не перезаписывать данные до явного действия пользователя. Область хранения и выполнение JavaScript различаются у локальных файлов и HTTP-сайтов; ES-модули следует открывать через локальный сервер.

## Подготовка

Установить актуальную поддерживаемую LTS-версию Node.js, Git и редактор кода. Проверить команды `node --version`, `npm --version`, `git --version`. Создать каталог `practice-09` с файлами `index.html`, `model.js`, `app.js`. В `index.html` подготовить форму с полем `titleInput`, кнопкой `addButton`, полем поиска `searchInput`, фильтром `statusFilter`, элементом `taskList` и счётчиком `countLabel`. Подключить `app.js` через `<script type="module" src="./app.js"></script>`.

Запускать проект через локальный HTTP-сервер, например `npx --yes http-server . -p 8080` из каталога проекта, и открывать `http://127.0.0.1:8080`. При отсутствии возможности загрузить утилиту допускается сервер, предоставляемый редактором кода. Проверить консоль браузера и вкладку Application → Local Storage.

## Источники

1. MDN Web Docs. [JavaScript Guide](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide).
2. MDN Web Docs. [Introduction to the DOM](https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model/Introduction).
3. MDN Web Docs. [EventTarget.addEventListener()](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener).
4. MDN Web Docs. [Window.localStorage](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage).
5. MDN Web Docs. [JSON.parse()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/parse).

## Задание

Разработать браузерное приложение «Журнал учебных задач» с разделением функций работы с данными и кода интерфейса.

| Параметр | Требование |
| --- | --- |
| Идентификатор | Уникальное положительное безопасное целое число |
| Название задачи | Строка из 1–80 символов после `trim()` |
| Статус | Логическое значение `completed` |
| Количество задач | 0–100 |
| Поиск | Подстрока названия без учёта регистра, от 0 до 80 символов |
| Фильтр | Все / выполненные / невыполненные |
| Хранение | JSON-массив в `localStorage`, ключ `student-tasks-v1` |

1. Создать функции добавления, изменения статуса, удаления и фильтрации задач в `model.js`. Не использовать в них DOM-объекты.
2. Реализовать создание задачи через форму и нажатие Enter. Пустые строки, превышение длины и превышение числа записей отклонять с понятным сообщением.
3. Отобразить список задач с названием, статусом и кнопками изменения статуса и удаления. Добавить счётчик выполненных задач и делегирование событий списка.
4. Реализовать совместную работу строки поиска и фильтра статуса; изменение фильтров не должно удалять исходные данные.
5. Сохранять данные после успешного изменения. При запуске восстанавливать список, проверять структуру и отсутствие повторяющихся идентификаторов.
6. Проверить безопасное отображение строки `<img src=x onerror=alert(1)>`: она должна выводиться как текст, без выполнения кода.

**Проверка:** добавить три записи; создать и удалить задачу; изменить статус дважды; проверить поиск без совпадений; перезагрузить страницу; испытать пустой ввод, название из 81 символа, некорректный JSON и повтор ID в хранилище. Для испорченного хранилища убедиться, что приложение сообщает об ошибке, не уничтожая содержимое автоматически.

**Представить:** `index.html`, `model.js`, `app.js`, краткую инструкцию запуска и таблицу фактических результатов проверок.

### Порядок выполнения

**Этап 1. Модель данных.** Сначала создайте и проверьте `normalizeTitle`, `addTask`, `toggleTask`, `removeTask`, `getVisibleTasks` в `model.js`. Например, вызов `addTask([], '  Подготовить доклад  ', 1)` должен вернуть массив с одним объектом и названием без пробелов по краям. После удаления отсутствующего ID исходный список должен остаться прежним.

**Этап 2. Отображение.** Добавьте на страницу форму, список, поисковую строку, фильтр статуса, счётчик и элемент для сообщения об ошибке. Напишите `render()`, который сначала очищает контейнер, а потом строит список из текущего массива. В каждую запись добавьте две кнопки: «Изменить статус» и «Удалить». Для обработки действий используйте `data-id` и делегирование события.

**Этап 3. Действия.** При отправке формы проверяйте название, выбирайте следующий неиспользованный ID, вызывайте функцию модели, сохраняйте массив и затем обновляйте интерфейс. При поиске не изменяйте исходные данные — фильтруйте только видимый список.

**Этап 4. Сохранение.** После корректных изменений вызывайте `saveTasks`. На старте выполняйте `loadTasks` в `try/catch`; при ошибке отображайте понятное предупреждение, сохраняя повреждённую строку в `localStorage` для разбора. Только явное решение пользователя может заменить повреждённое сохранение.

**Этап 5. Проверка.** Для каждого случая из раздела «Проверка» запишите входные данные, ожидаемый результат и фактический результат. Отдельно испытайте ввод строки с HTML-тегами, чтобы убедиться в безопасном выводе.

**Минимальный результат к концу аудиторной работы:** страница открывается без ошибок, позволяет добавить и показать задачу, переключить статус и сохранить текущий список. **Для завершённой работы:** реализовать все требования задания, фильтры и отрицательные сценарии.

### Контрольные вопросы

1. Чем отличаются `const` и `let`?
2. Почему `Number('abc')` нельзя использовать без проверки?
3. Что возвращает `filter`, а что — `find`?
4. Для чего нужен `event.preventDefault()`?
5. Почему пользовательский текст лучше вставлять через `textContent`?
6. Почему `JSON.parse` нужно сопровождать проверкой структуры данных?
7. Что делает обработчик события, установленный на общем контейнере списка?

