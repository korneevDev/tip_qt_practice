# Занятие 13. Динамические виджеты и табличное представление

## Цель

Освоить создание виджетов во время выполнения, собственные сигналы и представление коллекции в таблице.

## Теория

Динамический виджет создаётся программно и включается в компоновку. Родитель управляет временем жизни дочерних объектов. Удаление элемента компоновки не означает удаления виджета; отложенное удаление выполняется через `deleteLater()`.

Класс карточки передаёт идентификатор записи собственным сигналом. Главное окно обрабатывает событие и изменяет данные. Для избранного применяется `QSet<int>`; хранение ID позволяет сохранять выбор при пересоздании интерфейса.

`QTableWidget` хранит строки и столбцы, `QTableWidgetItem` — содержимое ячейки. `setRowCount()` не создаёт элементы ячеек; они передаются через `setItem()`. До чтения результата `item()` требуется проверка на `nullptr`. Таблица владеет переданными ей элементами.

**Пример 1. Класс динамической карточки.** В `gamecard.h` разместить объявление; структура `Game` находится в `game.h`:

```cpp
#pragma once
#include <QFrame>
#include "game.h"
class GameCard : public QFrame
{
    Q_OBJECT
public:
    explicit GameCard(const Game &game, bool favorite,
                      QWidget *parent = nullptr);
signals:
    void favoriteRequested(int id);
};
```

В `gamecard.cpp` подключить `<QVBoxLayout>`, `<QLabel>`, `<QPushButton>` и `gamecard.h`:

```cpp
GameCard::GameCard(const Game &game, bool favorite, QWidget *parent)
    : QFrame(parent)
{
    auto *layout = new QVBoxLayout(this);
    auto *title = new QLabel(game.title, this);
    title->setTextFormat(Qt::PlainText);
    layout->addWidget(title);
    auto *button = new QPushButton(
        favorite ? "Убрать из избранного" : "В избранное", this);
    layout->addWidget(button);
    connect(button, &QPushButton::clicked, this, [this, id = game.id] {
        emit favoriteRequested(id);
    });
}
```

Добавить оба файла в цель сборки; включить `CMAKE_AUTOMOC` до создания цели, если он ещё не включён. ID захватывается по значению, поэтому сигнал не зависит от времени жизни параметра `game`. Остальные поля карточки выводятся дополнительными `QLabel`.

При пересоздании карточек очистить компоновку `cardsLayout`:

```cpp
while (QLayoutItem *item = cardsLayout->takeAt(0)) {
    if (QWidget *widget = item->widget()) {
        widget->hide();
        widget->deleteLater();
    }
    delete item;
}
```

Здесь `cardsLayout` — `QVBoxLayout`, созданный для содержимого `QScrollArea`. После очистки создавать карточки только для записей, прошедших фильтры.

**Пример 2. Табличное представление.** В методе главного окна используются контейнер `games` и множество `favoriteIds`. Подключить `<QDialog>`, `<QVBoxLayout>`, `<QTableWidget>`:

```cpp
QDialog dialog(this);
QVBoxLayout layout(&dialog);
QTableWidget table(&dialog);
table.setColumnCount(6);
table.setHorizontalHeaderLabels(
    {"ID", "Название", "Категория", "Оценка", "Статус", "Избранное"});
table.setRowCount(int(games.size()));
table.setEditTriggers(QAbstractItemView::NoEditTriggers);
table.setSortingEnabled(false);
for (int row = 0; row < int(games.size()); ++row) {
    const Game &g = games.at(row);
    const QStringList values{QString::number(g.id), g.title, g.genre,
        QString::number(g.rating), g.finished ? "Да" : "Нет",
        favoriteIds.contains(g.id) ? "Да" : "Нет"};
    for (int column = 0; column < values.size(); ++column)
        table.setItem(row, column, new QTableWidgetItem(values.at(column)));
    table.item(row, 0)->setData(Qt::UserRole, g.id);
}
layout.addWidget(&table);
connect(&table, &QTableWidget::cellDoubleClicked, &dialog,
        [&table, &dialog](int row, int) {
    const auto *item = table.item(row, 0);
    if (!item) return;
    dialog.setWindowTitle(QString("ID %1").arg(item->data(Qt::UserRole).toInt()));
});
dialog.resize(800, 400);
dialog.exec();
```

Диалог, компоновка и таблица — локальные объекты; они уничтожаются в обратном порядке. Для них не вызывается `delete`. Таблица удаляет элементы ячеек самостоятельно. `exec()` используется для модального просмотра, а сетевые запросы обрабатываются асинхронно.

**Пример 3. Получение пути сохранения.** Подключить `<QStandardPaths>` и `<QDir>`:

```cpp
const QString directory = QStandardPaths::writableLocation(
    QStandardPaths::AppLocalDataLocation);
if (directory.isEmpty() || !QDir().mkpath(directory)) return;
const QString path = QDir(directory).filePath("favorites.json");
```

При отказе создания каталога обработчик должен показать ошибку. Содержимое множества сериализуется в JSON-массив, затем сохраняется через `QSaveFile`; пример записи и изменения набора приведён в занятии 14.

## Подготовка

Средства разработки: C++17, Qt 6.2 или новее, Qt Creator, CMake; тип проекта — Qt Widgets Application, класс главного окна — `MainWindow`. Исходные файлы сохранять в UTF-8. Интерфейс размещать в компоновках. Имена объектов задавать через свойство `objectName`.

Использовать сетевой каталог. Добавить `QScrollArea`, флажок `onlyFavoritesCheck`, счётчик и кнопку `tableButton`. Создать класс `GameCard`, производный от `QFrame`, с макросом `Q_OBJECT`. Для таблицы подготовить `QDialog` и `QTableWidget`.

## Источники

1. Qt Documentation. [QObject](https://doc.qt.io/qt-6/qobject.html).
2. Qt Documentation. [Signals & Slots](https://doc.qt.io/qt-6/signalsandslots.html).
3. Qt Documentation. [QScrollArea](https://doc.qt.io/qt-6/qscrollarea.html).
4. Qt Documentation. [QTableWidget](https://doc.qt.io/qt-6/qtablewidget.html).
5. Qt Documentation. [QStandardPaths](https://doc.qt.io/qt-6/qstandardpaths.html).

## Задание

Реализовать отображение каталога карточками и таблицей.

1. Выводить в карточке название, категорию, оценку 0–10, статус и кнопку избранного. Собственный сигнал `favoriteRequested(int id)` должен передавать ID выбранной записи.
2. Хранить избранные ID в `QSet<int>`. По нажатию добавлять или удалять ID; обновлять надпись кнопки и счётчик.
3. Совместно применять поиск по названию и фильтр избранного. При пустом результате выводить сообщение. Повторная загрузка не должна создавать дубликаты карточек и соединений.
4. При обновлении извлекать элементы компоновки, скрывать и удалять прежние карточки, затем создавать актуальные. Сохранять ограничения данных сетевого каталога: до 10000 записей, ID 1–1000000, длины названия/категории 100/50, запроса 100 единиц UTF-16.
5. По кнопке открывать таблицу всех загруженных записей независимо от фильтров карточек. Столбцы: ID, название, категория, оценка, статус, избранное. Статус и избранное выводить как «Да»/«Нет». Редактирование и сортировку отключить.
6. Хранить ID в `Qt::UserRole` первой ячейки. Двойной щелчок по строке должен выводить ID в заголовок диалога. На пустом каталоге сохранять заголовки и ноль строк.
7. Сохранять избранное в `favorites.json` каталога `QStandardPaths::AppLocalDataLocation`; создавать каталог при необходимости. Формат: `{"schemaVersion":1,"ids":[1,4]}`. Допустимы 0–10000 уникальных целых ID 1–1000000. Использовать `QSaveFile`; восстанавливать набор при запуске. Ошибка записи не должна подтверждать изменение как сохранённое.

**Проверка:** отметить запись, применить оба фильтра, открыть и закрыть таблицу трижды, перезапустить приложение и загрузить каталог. ID должны восстановиться, строки — не дублироваться. Проверить пустой каталог и повреждённый файл избранного. ID, временно отсутствующий в ответе сервера, сохранять.

**Представить:** исходные файлы проекта, форму `.ui`, файл сборки и результаты проверки программы.
