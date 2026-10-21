# Занятие 6. Обработка строк и регулярные выражения

## Цель

Изучить средства обработки строк Qt и реализовать приложение для анализа текста и проверки пользовательского ввода.

## Теория

`QString` хранит текст Unicode. Методы `trimmed()` и `simplified()` удаляют внешние пробелы и нормализуют пробельные последовательности соответственно. Для разделения и объединения строк применяются `split()` и `join()`, для поиска — `contains()`, для формирования сообщений — `arg()`. Метод `size()` возвращает длину в единицах UTF-16.

`QRegularExpression::match()` находит первое совпадение, `globalMatch()` — последовательность совпадений. Границы `\A` и `\z` обеспечивают проверку всей строки. `captured(0)` возвращает полное совпадение, `captured(1)` — первую группу. Именованная группа записывается как `(?<name>...)`. В строковых литералах C++ обратную косую черту необходимо удваивать либо использовать сырую строку `R"(...)"`.

При разборе ввода калькулятора последовательно проверяются непустое значение, формат `\A[+-]?[0-9]+\z`, результат `toLongLong(&ok, 10)` и диапазон −2147483648…2147483647. Перед делением проверяется ненулевой делитель. Сложение, вычитание и умножение выполняются после преобразования операндов в `qint64`. Некорректный ввод не заменяется нулём. Основание системы счисления задаётся аргументом преобразования; само целое значение не имеет текстового основания.

**Пример 1. Обработка текста.** В обработчике кнопки текст получают из виджета, затем обрабатывают средствами `QString`. Требуются заголовки `<QStringList>` и `<QRegularExpression>`.

```cpp
const QString text = ui->sourceEdit->toPlainText();
QStringList lines = text.split('\n', Qt::KeepEmptyParts);
int nonEmpty = 0;
for (QString &line : lines) {
    if (!line.trimmed().isEmpty()) ++nonEmpty;
    line = line.simplified();
}
ui->sourceEdit->setPlainText(lines.join('\n'));
ui->resultEdit->setPlainText(
    QString("Единиц UTF-16: %1; непустых строк: %2")
        .arg(text.size()).arg(nonEmpty));
```

Ссылка `QString &line` позволяет изменить элемент списка. Пустые строки сохраняются. Для подсчёта слов используется последовательный обход совпадений:

```cpp
const QRegularExpression wordRe(R"(\p{L}+)");
auto matches = wordRe.globalMatch(text);
int words = 0;
while (matches.hasNext()) {
    matches.next();
    ++words;
}
const QRegularExpression tagRe(
    R"(\[(?<category>[A-Za-z]{2,10}):(?<number>[0-9]{3})\])");
const auto tag = tagRe.match("[BUG:101]");
if (tag.hasMatch()) {
    const QString category = tag.captured("category"); // BUG
    const QString number = tag.captured("number");     // 101
}
```

Для поиска адресов заменить шаблон `wordRe` шаблоном email из задания и сохранять `captured(0)` каждого совпадения. Для неперекрывающихся вхождений подстроки после совпадения переходить за его конец; пустой запрос проверять до цикла:

```cpp
const QString query = ui->queryEdit->text().trimmed();
int count = 0;
if (!query.isEmpty()) {
    qsizetype position = 0;
    while ((position = text.indexOf(query, position,
                                    Qt::CaseInsensitive)) != -1) {
        ++count;
        position += query.size();
    }
}
```

**Пример 2. Проверка ввода калькулятора.** Функцию разместить перед определениями методов окна в `.cpp`; подключить `<limits>`. При ошибке выходной параметр не изменяется.

```cpp
static bool parseInteger(const QString &raw, qint32 &value)
{
    const QString s = raw.trimmed();
    const QRegularExpression re(R"(\A[+-]?[0-9]+\z)");
    if (!re.match(s).hasMatch()) return false;
    bool ok = false;
    const qint64 candidate = s.toLongLong(&ok, 10);
    if (!ok || candidate < std::numeric_limits<qint32>::min()
            || candidate > std::numeric_limits<qint32>::max())
        return false;
    value = qint32(candidate);
    return true;
}
```

Применение в обработчике; `<QMessageBox>` нужен для диагностики:

```cpp
qint32 a = 0, b = 0;
if (!parseInteger("2147483647", a) || !parseInteger("2", b)) {
    QMessageBox::warning(this, "Ошибка", "Недопустимое целое число");
    return;
}
const qint64 product = qint64(a) * qint64(b);
if (b == 0) {
    QMessageBox::warning(this, "Ошибка", "Деление на ноль");
    return;
}
const qint64 quotient = qint64(a) / qint64(b);
```

Умножение и деление выполняются в расширенном типе; целочисленное деление отбрасывает дробную часть. Преобразование после умножения было бы запоздалым: переполнение могло бы возникнуть раньше.

## Подготовка

Средства разработки: C++17, Qt 6.2 или новее, Qt Creator, CMake; тип проекта — Qt Widgets Application, класс главного окна — `MainWindow`. Исходные файлы сохранять в UTF-8. Интерфейс размещать в компоновках. Имена объектов задавать через свойство `objectName`.

Разместить `sourceEdit` и `resultEdit` (`QPlainTextEdit`), `queryEdit` и `loginEdit` (`QLineEdit`), кнопки `analyzeButton`, `normalizeButton`, `loginButton`. Для `resultEdit` установить `readOnly = true`.

## Источники

1. Qt Documentation. [QString](https://doc.qt.io/qt-6/qstring.html).
2. Qt Documentation. [QRegularExpression](https://doc.qt.io/qt-6/qregularexpression.html).
3. Qt Documentation. [QPlainTextEdit](https://doc.qt.io/qt-6/qplaintextedit.html).

## Задание

Разработать приложение «Анализатор текста».

1. По кнопке анализа вывести длину текста в единицах UTF-16, число слов, число непустых строк, число вхождений запроса и найденные адреса электронной почты.
2. Словом считать непрерывную последовательность букв Unicode; использовать шаблон `\p{L}+`. Непустой считать строку, для которой `trimmed().isEmpty()` возвращает `false`.
3. Искать запрос как подстроку без учёта регистра; считать неперекрывающиеся вхождения. Пустой запрос означает отсутствие поиска и даёт ноль совпадений. Длина запроса — 0–100 единиц UTF-16.
4. Для извлечения email использовать шаблон `[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}`. Сохранять порядок и повторения. Данный шаблон определяет учебный формат адреса.
5. По кнопке нормализации применить `simplified()` отдельно к каждой строке, сохранив переводы строк и пустые строки.
6. Проверить имя пользователя: длина 4–16, первый символ — латинская буква, остальные — латинские буквы, цифры или `_`. Использовать шаблон всей строки; при ошибке вывести причину.
7. Дополнить анализ извлечением меток вида `[BUG:101]`: категория — 2–10 латинских букв, номер — три цифры. Вывести категорию и номер через группы регулярного выражения.

**Проверка:** для текста `Qt Qt\n\nТекст` (где `\n` обозначает перевод строки) получить 12 единиц UTF-16, 3 слова и 2 непустые строки; запрос `qt` даёт 2 совпадения. `User_7` — допустимое имя, `7User` и `Ab` — недопустимые. Повторный анализ не накапливает результаты. Для примера калькулятора составить таблицу проверки ввода: пустая строка, `0`, `12a`, `2147483648`, `-2147483648` и деление на ноль.

**Представить:** исходные файлы проекта, форму `.ui`, файл сборки и результаты проверки программы.
