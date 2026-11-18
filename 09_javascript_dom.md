# Занятие 9. Основы JavaScript, DOM и локальное хранение данных

## Цель

Освоить основные конструкции JavaScript, работу с объектами и массивами, обработку событий браузера и сохранение данных. Разработать интерактивный список учебных задач.

## Теория

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
