# Занятие 9. Хранение структурированных данных в JSON и двоичном формате

## Цель

Изучить сериализацию данных и реализовать загрузку и сохранение каталога в JSON и двоичном формате.

## Теория

JSON представляет данные объектами, массивами и значениями. В Qt используются `QJsonDocument`, `QJsonObject`, `QJsonArray`, `QJsonValue`. Успешный синтаксический разбор не гарантирует соответствия схеме: необходимо проверять обязательные поля, типы и диапазоны.

`QDataStream` записывает типизированные значения. При чтении должны совпадать порядок полей, типы, версия потока и порядок байтов. Сигнатура идентифицирует формат, версия — его структуру. Текстовый и двоичный файлы состоят из байтов; способ представления данных определяется форматом.

Данные загружаются во временный контейнер. Текущий каталог заменяется после успешного чтения и проверки всех записей.

**Пример 1. Представление записи.** Объявление поместить в `game.h`, подключаемый из файлов обработки данных:

```cpp
#pragma once
#include <QString>
struct Game {
    int id = 0;
    QString title;
    QString genre;
    int rating = 0;
    bool finished = false;
};
```

В `.cpp` подключить `game.h`, `<QList>`, `<QJsonDocument>`, `<QJsonObject>`, `<QJsonArray>`. Следующая функция сериализует уже проверенные записи:

```cpp
static QByteArray toJson(const QList<Game> &games)
{
    QJsonArray items;
    for (const Game &g : games) {
        items.append(QJsonObject{{"id", g.id}, {"title", g.title},
                                {"genre", g.genre}, {"rating", g.rating},
                                {"finished", g.finished}});
    }
    return QJsonDocument(QJsonObject{{"schemaVersion", 1}, {"items", items}})
        .toJson(QJsonDocument::Indented);
}
```

**Пример 2. Проверка числового поля JSON.** `toInt()` не заменяет проверку исходного типа и целочисленности. Подключить `<cmath>`; функция подходит для ограничений ID и оценки:

```cpp
static bool readInteger(const QJsonValue &value, int minimum,
                        int maximum, int &result)
{
    if (!value.isDouble()) return false;
    const double number = value.toDouble();
    if (!std::isfinite(number) || std::floor(number) != number
        || number < minimum || number > maximum)
        return false;
    result = int(number);
    return true;
}
```

При чтении документа использовать `QJsonDocument::fromJson(bytes, &error)` с `QJsonParseError`; затем проверять `isObject()`, поля версии и массива. Для строк использовать `isString()`, для статуса — `isBool()`. Повтор ID проверять множеством `QSet<int>`. Изменять текущий каталог только после проверки всех объектов.

**Пример 3. Двоичный поток в памяти.** Подключить `<QDataStream>` и `<QIODevice>`. Формат записи воспроизводится при чтении в том же порядке:

```cpp
QByteArray bytes;
QDataStream output(&bytes, QIODevice::WriteOnly);
output.setVersion(QDataStream::Qt_6_0);
output.setByteOrder(QDataStream::BigEndian);
output << quint32(0x51474331) << quint16(1) << quint32(1);
output << qint32(1) << QString("Portal") << QString("Головоломка")
       << qint32(9) << true;

QDataStream input(bytes);
input.setVersion(QDataStream::Qt_6_0);
input.setByteOrder(QDataStream::BigEndian);
quint32 signature = 0, count = 0;
quint16 version = 0;
input >> signature >> version >> count;
if (input.status() != QDataStream::Ok || signature != 0x51474331
    || version != 1 || count > 10000) {
    return; // Фрагмент внутри обработчика void.
}
qint32 id = 0, rating = 0;
QString title, genre;
bool finished = false;
input >> id >> title >> genre >> rating >> finished;
if (input.status() != QDataStream::Ok) return;
```

Для каталога чтение записи помещается в цикл по `count`; значения проверяются по таблице ограничений. Вместо массива байтов потоку можно передать адрес открытого `QFile` или `QSaveFile`. Перед фиксацией записи проверяются состояние потока и результат `commit()`.

## Подготовка

Средства разработки: C++17, Qt 6.2 или новее, Qt Creator, CMake; тип проекта — Qt Widgets Application, класс главного окна — `MainWindow`. Исходные файлы сохранять в UTF-8. Интерфейс размещать в компоновках. Имена объектов задавать через свойство `objectName`.

Создать структуру `Game` с полями `id`, `title`, `genre`, `rating`, `finished` и контейнер `QList<Game>`. Разместить список записей, поле сообщений и кнопки загрузки и сохранения для JSON и BIN. Подключить классы JSON, `QDataStream`, `QFileDialog`, `QSaveFile`.

## Источники

1. Qt Documentation. [QJsonDocument](https://doc.qt.io/qt-6/qjsondocument.html).
2. Qt Documentation. [QJsonValue](https://doc.qt.io/qt-6/qjsonvalue.html).
3. Qt Documentation. [QDataStream](https://doc.qt.io/qt-6/qdatastream.html).
4. Qt Documentation. [QSaveFile](https://doc.qt.io/qt-6/qsavefile.html).

## Задание

Разработать приложение хранения игровой коллекции.

| Данные | Допустимые значения |
|---|---|
| Количество записей | 0–10000 |
| `id` | Целое 1–1000000, уникальное |
| `title` | Строка 1–100 единиц UTF-16 после `trimmed()` |
| `genre` | Строка 1–50 единиц UTF-16 после `trimmed()` |
| `rating` | Целое 0–10 |
| `finished` | Логическое значение |
| Размер загружаемого файла | 0–2097152 байт; пустой файл не соответствует формату |

1. Сформировать не менее четырёх записей с двумя категориями и обоими значениями статуса. Вывести все поля в интерфейсе.
2. Реализовать JSON-формат: корневой объект с числовым полем `schemaVersion`, равным 1, и массивом `items`. Объект каждой записи содержит пять указанных полей. Пустой массив допустим; отсутствие поля, неверный тип, дробный ID или рейтинг, повтор ID — ошибка всего документа.
3. Реализовать двоичный формат: `quint32`-сигнатура `0x51474331`, `quint16`-версия 1, `quint32`-число записей; далее для каждой записи — `qint32 id`, `QString title`, `QString genre`, `qint32 rating`, `bool finished`. Установить `QDataStream::Qt_6_0` и `BigEndian` при записи и чтении.
4. Проверять статус потока, сигнатуру, версию, число записей и значения полей. Не принимать усечённый файл. При ошибке сохранять прежний каталог и выводить причину.
5. Разместить функции сериализации отдельно от обработчиков интерфейса. Выбор пути выполнять через файловый диалог; отмена не считается ошибкой.

**Проверка:** сохранить каталог в обоих форматах, очистить и загрузить поочерёдно; все поля и порядок должны восстановиться. Проверить пустой каталог, повреждённый JSON, отсутствующее поле, рейтинг 11, повтор ID, неверную сигнатуру и усечённый BIN. Представить оба сохранённых файла.

**Представить:** исходные файлы проекта, форму `.ui`, файл сборки и результаты проверки программы.
