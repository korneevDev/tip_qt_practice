# Занятие 12. Сетевое взаимодействие и обработка JSON

## Цель

Освоить выполнение HTTP-запросов и асинхронную обработку ответов в Qt.

## Теория

HTTP-запрос содержит метод, URL, заголовки и при необходимости тело. Ответ содержит код состояния, заголовки и данные. GET используется для получения ресурса, POST — для передачи данных на обработку. Формат и допустимые операции определяются контрактом API. `QUrlQuery` представляет параметры адреса.

`QNetworkAccessManager` отправляет запрос, `QNetworkRequest` содержит его параметры, `QNetworkReply` представляет ответ. Обработка выполняется по сигналу `finished`; интерфейс продолжает обрабатывать события. После получения ответа проверяются сетевая ошибка, HTTP-код и структура JSON. Объект ответа освобождается через `deleteLater()`.

Параметр `setTransferTimeout(5000)` ограничивает период отсутствия передачи данных, а не общее время загрузки.

**Пример 1. Подключение сетевого модуля.** В существующем `CMakeLists.txt` добавить `Network` к компонентам и `Qt6::Network` к библиотекам. Вместо `Catalog` указать имя цели своего проекта:

```cmake
find_package(Qt6 REQUIRED COMPONENTS Widgets Network)
target_link_libraries(Catalog PRIVATE Qt6::Widgets Qt6::Network)
```

В `mainwindow.h` подключить `<QNetworkAccessManager>` и объявить в `private`:

```cpp
QNetworkAccessManager *manager = nullptr;
bool loading = false;
void loadCatalog();
```

В конструкторе после `ui->setupUi(this)` создать менеджер и соединение кнопки:

```cpp
manager = new QNetworkAccessManager(this);
connect(ui->loadButton, &QPushButton::clicked,
        this, &MainWindow::loadCatalog);
```

**Пример 2. Асинхронный GET.** В `.cpp` подключить `<QNetworkRequest>`, `<QNetworkReply>`, `<QUrl>`, `<QJsonDocument>`, `<QJsonObject>`, `<QJsonArray>`, `<QJsonParseError>`. Обработчик демонстрирует транспортную часть и проверку корневой структуры:

```cpp
void MainWindow::loadCatalog()
{
    if (loading) return;
    const QUrl url(ui->urlEdit->text().trimmed());
    if (!url.isValid() || url.host().isEmpty()
        || (url.scheme() != "http" && url.scheme() != "https")) {
        ui->statusLabel->setText("Некорректный URL");
        return;
    }
    QNetworkRequest request(url);
    request.setRawHeader("Accept", "application/json");
    request.setTransferTimeout(5000);
    loading = true;
    ui->loadButton->setEnabled(false);
    ui->statusLabel->setText("Загрузка");
    QNetworkReply *reply = manager->get(request);
    connect(reply, &QNetworkReply::finished, this, [this, reply] {
        loading = false;
        ui->loadButton->setEnabled(true);
        const int status = reply->attribute(
            QNetworkRequest::HttpStatusCodeAttribute).toInt();
        const auto networkError = reply->error();
        const QByteArray bytes = reply->readAll();
        reply->deleteLater();
        if (networkError != QNetworkReply::NoError || status != 200) {
            ui->statusLabel->setText(QString("Ошибка запроса: HTTP %1").arg(status));
            return;
        }
        if (bytes.size() > 2097152) {
            ui->statusLabel->setText("Ответ превышает 2 МиБ");
            return;
        }
        QJsonParseError error;
        const QJsonDocument doc = QJsonDocument::fromJson(bytes, &error);
        if (error.error != QJsonParseError::NoError || !doc.isObject()) {
            ui->statusLabel->setText("Некорректный JSON");
            return;
        }
        const QJsonObject root = doc.object();
        const QJsonValue version = root.value("schemaVersion");
        if (!version.isDouble() || version.toDouble() != 1.0
            || !root.value("items").isArray()
            || root.value("items").toArray().size() > 10000) {
            ui->statusLabel->setText("Неверная схема ответа");
            return;
        }
        // Проверить каждую запись и уникальность ID во временном QList<Game>.
        // При успехе заменить games и обновить список; при ошибке оставить games.
        ui->statusLabel->setText("Корневая структура проверена");
    });
}
```

Проверку полей выполнить по образцам занятия 9. Комментарии в конце обозначают обязательную доработку, без которой ответ ещё не принят как каталог. Лимит размера проверяется перед разбором, а не во время скачивания.

**Пример 3. Параметр адреса и тело POST.** Подключить `<QUrlQuery>`; фрагменты выполняются в методах окна:

```cpp
QUrl url("http://127.0.0.1:8000/api/games");
QUrlQuery query;
query.addQueryItem("title", "Portal");
url.setQuery(query);

QNetworkRequest postRequest(QUrl("http://127.0.0.1:8000/api/echo"));
postRequest.setHeader(QNetworkRequest::ContentTypeHeader, "application/json");
postRequest.setTransferTimeout(5000);
const QByteArray body = QJsonDocument(
    QJsonObject{{"message", QString("Привет, Qt!")}}).toJson();
QNetworkReply *postReply = manager->post(postRequest, body);
```

К `postReply` подключается обработчик `finished` по приведённому образцу: проверка ошибки и HTTP-кода, чтение тела, `deleteLater()`. Для echo проверяется объект `received`, а не схема `schemaVersion/items`. Неиспользуемый ответ нельзя оставлять без освобождения.

## Подготовка

Средства разработки: C++17, Qt 6.2 или новее, Qt Creator, CMake; тип проекта — Qt Widgets Application, класс главного окна — `MainWindow`. Исходные файлы сохранять в UTF-8. Интерфейс размещать в компоновках. Имена объектов задавать через свойство `objectName`.

Подключить модуль Qt Network в CMake. Разместить `urlEdit`, `loadButton`, `statusLabel`, `searchEdit`, `gamesList`. Для испытаний запустить приложенный `data/server.py` из папки `data`: `py -3 server.py` (Windows) или `python3 server.py` (Linux/macOS). Требуется Python 3.9 или новее. Адрес каталога: `http://127.0.0.1:8000/api/games`. Маршруты `/api/empty`, `/api/broken`, `/api/error`, `/api/slow` возвращают пустой каталог, повреждённый JSON, HTTP 500 и ответ с задержкой 7 секунд. Остановка сервера — Ctrl+C.

## Источники

1. Qt Documentation. [QNetworkAccessManager](https://doc.qt.io/qt-6/qnetworkaccessmanager.html).
2. Qt Documentation. [QNetworkRequest](https://doc.qt.io/qt-6/qnetworkrequest.html).
3. Qt Documentation. [QNetworkReply](https://doc.qt.io/qt-6/qnetworkreply.html).
4. Qt Documentation. [QUrlQuery](https://doc.qt.io/qt-6/qurlquery.html).

## Задание

Разработать сетевой каталог.

1. Проверять URL: схема `http` или `https`, непустое имя сервера. Выполнять GET по нажатию кнопки, устанавливать `Accept: application/json` и тайм-аут 5000 мс.
2. Разрешать не более одного запроса каталога одновременно. На время загрузки отключать кнопку и выводить состояние; по завершении восстанавливать возможность запроса.
3. Принимать ответ при отсутствии сетевой ошибки и HTTP 200. До разбора проверять размер: не более 2097152 байт.
4. Проверять JSON: корневой объект `schemaVersion: 1`, массив `items` из 0–10000 записей. Поля: уникальный целый `id` 1–1000000; `title` 1–100 и `genre` 1–50 единиц UTF-16 после `trimmed()`; целый `rating` 0–10; логический `finished`.
5. Заменять каталог только после проверки всего ответа. При сетевой ошибке, другом HTTP-коде или неверных данных показывать причину и сохранять прежний каталог.
6. Реализовать локальный поиск по названию без учёта регистра: запрос 0–100 единиц UTF-16, пустой запрос показывает все записи. Ввод поискового текста не должен отправлять сетевой запрос.
7. Сформировать URL с параметром `title=Portal` через `QUrlQuery` и проверить серверную фильтрацию, если она предусмотрена контрактом используемого сервера.

**Пример корректного ответа:**

```json
{"schemaVersion":1,"items":[{"id":1,"title":"Portal","genre":"Головоломка","rating":9,"finished":true}]}
```

**Проверка:** успешный ответ, пустой массив, повреждённый JSON, HTTP 404/500, недоступный сервер, задержка более 5 секунд без передачи данных, повторная загрузка. Во всех случаях окно должно оставаться отзывчивым, кнопка — восстанавливаться, записи — не дублироваться.

**Дополнительное задание:** отправить JSON-объект методом POST с `Content-Type: application/json` на маршрут `http://127.0.0.1:8000/api/echo` (JSON-объект 1–4096 байт; ответ HTTP 200 с полем `received`); обработать успешный ответ и ошибку формата. Представить адреса, примеры ответов и результаты испытаний.

**Представить:** исходные файлы проекта, форму `.ui`, файл сборки и результаты проверки программы.
