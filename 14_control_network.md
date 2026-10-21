# Занятие 14. Контрольная работа №4. Сетевой каталог с избранным

## Цель

Проверить умение объединять сетевое взаимодействие, динамический интерфейс и локальное хранение пользовательского выбора.

## Теория

Сетевой ответ проверяется до изменения модели данных. Карточки и таблица создаются из проверенного контейнера. Сигнал карточки передаёт ID; главное окно отвечает за изменение избранного и сохранение файла.

Файл избранного хранит идентификаторы, а не полный каталог. При запуске восстанавливаются ID; сведения о записях появляются после загрузки сервера. Ошибка запроса не должна уничтожать последнюю успешно загруженную коллекцию.

**Пример 1. Совместное применение фильтров.** Фрагмент выполняется в методе формирования карточек. `games` — проверенный каталог, `favoriteIds` — множество избранного; `cardsLayout` предварительно очищена:

```cpp
const QString query = ui->searchEdit->text().trimmed();
int shown = 0;
for (const Game &game : games) {
    if (!game.title.contains(query, Qt::CaseInsensitive)) continue;
    if (ui->onlyFavoritesCheck->isChecked()
        && !favoriteIds.contains(game.id)) continue;
    auto *card = new GameCard(game, favoriteIds.contains(game.id),
                              ui->cardsArea->widget());
    connect(card, &GameCard::favoriteRequested,
            this, &MainWindow::toggleFavorite);
    cardsLayout->addWidget(card);
    ++shown;
}
ui->countLabel->setText(QString("Показано: %1").arg(shown));
```

Для пустого результата дополнительно выводится сообщение. Класс карточки и порядок очистки компоновки описаны в занятии 13. Методы `toggleFavorite(int)` и `render()` объявляются в `private` главного окна; `render()` обновляет карточки и счётчик.

**Пример 2. Сохранение нового набора до изменения интерфейса.** Подключить `<QSaveFile>`, `<QJsonDocument>`, `<QJsonObject>`, `<QJsonArray>`, `<QSet>`:

```cpp
static bool saveFavorites(const QString &path, const QSet<int> &ids,
                          QString &error)
{
    error.clear();
    QJsonArray array;
    for (int id : ids) array.append(id);
    const QByteArray bytes = QJsonDocument(
        QJsonObject{{"schemaVersion", 1}, {"ids", array}}).toJson();
    QSaveFile file(path);
    if (!file.open(QIODevice::WriteOnly)) {
        error = file.errorString();
        return false;
    }
    if (file.write(bytes) != bytes.size()) {
        error = file.errorString();
        file.cancelWriting();
        return false;
    }
    if (!file.commit()) {
        error = file.errorString();
        return false;
    }
    return true;
}
```

Функция принимает предварительно проверенный набор. В классе объявить `QString favoritesPath`, полученный через `QStandardPaths` и `QDir` по примеру занятия 13; для диагностики подключить `<QMessageBox>`:

```cpp
void MainWindow::toggleFavorite(int id)
{
    if (id < 1 || id > 1000000) return;
    QSet<int> candidate = favoriteIds;
    if (candidate.contains(id)) candidate.remove(id);
    else candidate.insert(id);
    if (candidate.size() > 10000) {
        QMessageBox::warning(this, "Ошибка", "Превышен размер избранного");
        return;
    }
    QString error;
    if (!saveFavorites(favoritesPath, candidate, error)) {
        QMessageBox::warning(this, "Ошибка сохранения", error);
        return;
    }
    favoriteIds = candidate;
    render();
}
```

При восстановлении сначала проверить документ и версию, затем каждый элемент массива `ids`: числовой тип, целочисленность, диапазон и отсутствие повторов. Проверку числового значения выполнять по образцу занятия 9. Текущее множество заменяется только после проверки всего массива. Отсутствие файла при первом запуске означает пустое избранное.

## Подготовка

Средства разработки: C++17, Qt 6.2 или новее, Qt Creator, CMake; тип проекта — Qt Widgets Application, класс главного окна — `MainWindow`. Исходные файлы сохранять в UTF-8. Интерфейс размещать в компоновках. Имена объектов задавать через свойство `objectName`.

Обязательные средства: Qt Network, `QNetworkAccessManager`, `QNetworkReply`, `QList<Game>`, `QSet<int>`, собственный класс `GameCard` и сигнал с ID, `QScrollArea`, `QTableWidget`, `QStandardPaths`, `QSaveFile`, JSON. Запустить `data/server.py` командой `py -3 server.py` или `python3 server.py` из папки `data`. Адрес: `http://127.0.0.1:8000/api/games`. Подготовить в `data/games.json` не менее пяти записей с двумя категориями и обоими статусами. Для ошибок использовать `/api/broken`, `/api/error`, `/api/slow`, для пустого ответа — `/api/empty`.

## Источники

1. Qt Documentation. [QNetworkAccessManager](https://doc.qt.io/qt-6/qnetworkaccessmanager.html).
2. Qt Documentation. [Signals & Slots](https://doc.qt.io/qt-6/signalsandslots.html).
3. Qt Documentation. [QTableWidget](https://doc.qt.io/qt-6/qtablewidget.html).
4. Qt Documentation. [QStandardPaths](https://doc.qt.io/qt-6/qstandardpaths.html).
5. Qt Documentation. [QSaveFile](https://doc.qt.io/qt-6/qsavefile.html).

## Задание

Разработать приложение «Сетевой каталог».

| Параметр | Требование |
|---|---|
| Запрос | GET; URL `http`/`https` с непустым именем сервера |
| Успех | HTTP 200, отсутствие сетевой ошибки, корректный JSON |
| Ожидание | 5000 мс без передачи данных; не более одного запроса каталога |
| JSON | Не более 2097152 байт; `schemaVersion: 1`, массив `items` |
| Каталог | 0–10000 записей |
| Поля | Уникальный целый ID 1–1000000; название 1–100, категория 1–50 единиц UTF-16 после `trimmed()`; целая оценка 0–10; логический статус |
| Поиск | 0–100 единиц UTF-16 после `trimmed()`, без учёта регистра |
| Избранное | 0–10000 уникальных целых ID 1–1000000 |

1. Реализовать асинхронную загрузку с индикацией состояния и блокировкой повторного запроса. При любой ошибке восстанавливать кнопку и сохранять прежние данные.
2. Показать карточки с пятью полями записи и кнопкой избранного. Изменения передавать через собственный сигнал с ID. Повторная загрузка не создаёт дубликатов.
3. Совместно применять локальный поиск по названию и флажок «Только избранное». Пустой запрос не ограничивает выборку; при отсутствии результатов выводить сообщение и нулевой счётчик.
4. Сохранять выбор после каждого успешного изменения в `favorites.json` в `AppLocalDataLocation`, восстанавливать при запуске. Формат: `{"schemaVersion":1,"ids":[...]}`. При ошибке записи сохранять прежний набор; при повреждённом файле выводить диагностику. Отсутствующие в текущем каталоге ID автоматически не удалять.
5. Открывать таблицу всех загруженных записей с шестью столбцами: ID, название, категория, оценка, статус, избранное. Фильтры карточек на таблицу не влияют. Запретить редактирование и сортировку; двойным щелчком выводить ID строки в заголовок. Пустой каталог — ноль строк с заголовками.
6. Выполнить одну индивидуальную функцию по назначенному варианту: 1 — фильтр оценки не ниже 8; 2 — фильтр незавершённых; 3 — поиск по названию или категории; 4 — минимальная оценка из `QSpinBox` 0–10; 5 — счётчик завершённых среди показанных карточек. Функция должна учитывать действующие фильтры.

**Проверка:** успешная и повторная загрузка; поиск без совпадений; совместная фильтрация; сохранение и восстановление избранного; пустой ответ; неверный JSON; HTTP 500; остановленный сервер; тайм-аут; повреждённый файл избранного; повторное открытие таблицы. Представить исходный набор сервера, файл избранного и таблицу результатов испытаний.

**Представить:** исходные файлы проекта, форму `.ui`, файл сборки и результаты проверки программы.
