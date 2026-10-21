# Занятие 7. Текстовые файлы и функции обработки данных

## Цель

Освоить чтение и сохранение текстовых файлов, выделение обработки данных в функции и контроль несохранённых изменений.

## Теория

`QFile` представляет файловое устройство, `QTextStream` выполняет текстовый ввод и вывод. Для Qt 6 кодировка задаётся через `setEncoding(QStringConverter::Utf8)`. `readLine()` читает строку без её окончания; оператор `>>` извлекает значения, разделённые пробельными символами. После операций проверяется состояние потока.

Функция возвращает значение через `return`; тип `void` обозначает отсутствие возвращаемого значения. Параметр `const QString &` предназначен для чтения, `QString &` позволяет изменять аргумент. В функции `bool readText(const QString &path, QString &result, QString &error)` логическое значение обозначает успех, а ссылки — выходные данные. Значения параметров по умолчанию указываются после обязательных параметров.

`QFileDialog` возвращает путь; пустой путь означает отмену. `QSaveFile` записывает временный файл и заменяет целевой после успешного `commit()`. Флаг `QTextDocument::isModified()` используется для контроля изменений документа.

**Пример 1. Чтение UTF-8.** Функция размещается в `.cpp` вне класса. Требуются `<QFile>`, `<QTextStream>`, `<QStringConverter>`. Лимит проверяется перед чтением; этот пример предназначен для обычного локального файла.

```cpp
static bool readText(const QString &path, QString &result, QString &error)
{
    error.clear();
    QFile file(path);
    if (!file.open(QIODevice::ReadOnly | QIODevice::Text)) {
        error = file.errorString();
        return false;
    }
    if (file.size() > 2097152) {
        error = "Файл превышает 2 МиБ";
        return false;
    }
    QTextStream stream(&file);
    stream.setEncoding(QStringConverter::Utf8);
    const QString temporary = stream.readAll();
    if (stream.status() != QTextStream::Ok) {
        error = "Ошибка чтения текста";
        return false;
    }
    result = temporary;
    return true;
}
```

`result` изменяется только при успехе. Объект `QFile` закрывает файл при выходе из функции. В обработчике открытия используются `<QFileDialog>` и `<QMessageBox>`:

```cpp
const QString path = QFileDialog::getOpenFileName(
    this, "Открыть текст", QString(), "Текст (*.txt)");
if (path.isEmpty()) return;
QString text, error;
if (!readText(path, text, error)) {
    QMessageBox::warning(this, "Ошибка", error);
    return;
}
ui->sourceEdit->setPlainText(text);
ui->sourceEdit->document()->setModified(false);
```

Перед этим фрагментом необходимо выполнить проверку несохранённых изменений по условию задания. Для обращения к документу подключить `<QTextDocument>`.

**Пример 2. Запись текста.** Подключить `<QSaveFile>`. `flush()` передаёт буфер потока устройству до проверки и `commit()`.

```cpp
static bool writeText(const QString &path, const QString &text, QString &error)
{
    error.clear();
    QSaveFile file(path);
    if (!file.open(QIODevice::WriteOnly | QIODevice::Text)) {
        error = file.errorString();
        return false;
    }
    QTextStream stream(&file);
    stream.setEncoding(QStringConverter::Utf8);
    stream << text;
    stream.flush();
    if (stream.status() != QTextStream::Ok) {
        error = "Ошибка записи текста";
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

Экспорт отчёта вызывает эту функцию с `ui->resultEdit->toPlainText()`, сохранение исходного документа — с `ui->sourceEdit->toPlainText()`. Признак изменения исходника сбрасывается только после его успешного сохранения.

**Пример 3. Возвращаемый результат и выходной параметр.** При разборе строки сначала проверяется количество полей, затем их содержимое:

```cpp
QStringList fields = line.split(';', Qt::KeepEmptyParts);
if (fields.size() != 3) {
    error = "Ожидалось три поля";
    return false; // Внутри функции bool parseTicket(...).
}
for (QString &field : fields) field = field.trimmed();
```

Здесь `line` — входной параметр, `error` — ссылка на строку диагностики. Далее проверяются номер, email и описание; очищенные поля передаются вызывающему коду только при успехе.

## Подготовка

Средства разработки: C++17, Qt 6.2 или новее, Qt Creator, CMake; тип проекта — Qt Widgets Application, класс главного окна — `MainWindow`. Исходные файлы сохранять в UTF-8. Интерфейс размещать в компоновках. Имена объектов задавать через свойство `objectName`.

Разместить поля `sourceEdit`, `resultEdit`, `queryEdit`; кнопки `openButton`, `saveButton`, `analyzeButton`, `exportButton`. Отчёт должен быть доступен только для чтения. Подготовить текстовый файл UTF-8 с несколькими строками, кириллицей и пустой строкой.

## Источники

1. Qt Documentation. [QFile](https://doc.qt.io/qt-6/qfile.html).
2. Qt Documentation. [QTextStream](https://doc.qt.io/qt-6/qtextstream.html).
3. Qt Documentation. [QSaveFile](https://doc.qt.io/qt-6/qsavefile.html).
4. Qt Documentation. [QFileDialog](https://doc.qt.io/qt-6/qfiledialog.html).

## Задание

Разработать приложение для редактирования и анализа текстового файла.

1. Реализовать отдельные функции `readText(path, result, error)` и `writeText(path, text, error)`, возвращающие `bool`. Диалоги и сообщения размещать в обработчиках окна, а не в этих функциях.
2. Открывать файлы UTF-8 размером 0–2097152 байт. При ошибке чтения сохранять прежний текст; новый текст устанавливать только после успешного завершения операции.
3. Сохранять исходный текст и отчёт по выбранным пользователем путям. Проверять открытие, состояние потока и результат завершения записи. Отмена диалога не изменяет документ.
4. Формировать отчёт: длина текста, число непустых строк, число строк, содержащих запрос, и сами найденные строки с физическими номерами. Искать без учёта регистра. Запрос — 0–100 единиц UTF-16; при пустом запросе поиск не выполнять.
5. После изменения исходного текста очищать устаревший отчёт. Сохранение отчёта не должно сбрасывать признак изменения исходного документа.
6. Перед открытием другого файла и закрытием изменённого документа предлагать сохранение, отказ от сохранения или отмену действия. Ошибка сохранения и отмена выбора пути должны отменять исходное действие.
7. Разработать функцию разбора строки `номер;email;описание`. Требовать ровно три поля, сохранять пустые поля при разделении. Номер — 100–999, email — 6–254 единицы UTF-16 и соответствие шаблону адреса из занятия 6, описание — 1–500 единиц после `trimmed()`. Возвращать поля и причину первой ошибки через выходные параметры.

**Проверка:** пустой файл открывается без ошибки; кириллица сохраняется; запрос `ТЕКСТ` совпадает с `текст`; отмена открытия и закрытия сохраняет текущий документ. В разборе записи проверить неверный номер, пустое описание и лишний разделитель. Представить исходный файл и экспортированный отчёт.

**Представить:** исходные файлы проекта, форму `.ui`, файл сборки и результаты проверки программы.
