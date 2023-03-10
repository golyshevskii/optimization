### SQL optimization tips

#### **Оптимизация структуры таблиц `CREATE TABLE`**:

- **Лучше разбить сложные таблицы (широкие, в которых много столбцов) на несколько**
> чем больше в вашей таблице столбцов и тяжелых типов `nvarchar(max)`, тем тяжелее по ней проход

- Если некоторые столбцы не всегда используются в `SELECT` с таблицей, то *лучше вынести большие столбцы в отдельные таблицы и связать их через `FK` (Foreign Key)*

- **Выбирайте правильные типы данных**
> по возможности, всегда выбирайте самый маленький тип данных столбца

- Если текстовые данные в столбце имеют разную длину, то *используйте тип данных `VARCHAR` вместо `CHAR`*
> не используйте `NVARCHAR` или `NCHAR`, если не нужно сохранить 16-разрядные символьные данные (UNICODE), так как они требуют в два раза больше места, чем `CHAR` и `VARCHAR`, что повышает расход времени на ввод-вывод (но если тусовочка с кириллицей, то без `NVARCHAR` не обойтись)

- Если нужно хранить большие строки данных и их длина меньше чем `8000` символов, то лучше использовть тип данных `NVARCHAR` вместо `TEXT`
> `TEXT` - текстовое поле, которое требуют больше ресурсов для обработки, из-за чего снижается производительность

- Любой столбец, в котором должны быть только отличные от нуля значения, нужно объявлять как `NOT NULL`

- Для любого столбца, который должнен содержать уникальные значения, стоит указать модификатор `UNIQUE`

- Хранение изображений в БД нежелательно
> лучше хранить в таблице путь к файлу (локальный путь или URL), а сам файл помещать в файловую систему сервера


#### **Оптимизация запросов `SELECT`**:

- **Не читайте больше данных, чем нужно**
> не используйте * (выборка всех столбцов), если знаете названия столбцов для выборки, лучше указать их явно

- **Как можно раньше отфильтруйте данные**
> не нужно выполнять большой тяжелый подзапрос для всех строк таблицы, сначала отфильтруйте нужные строки

- Корректно используйте соединения `JOIN`
> если есть две или более таблиц, которые часто объединяются вместе, тогда столбцы, используемые для объединений, должны иметь соответствующий индекс

- Для лучшей производительности, **столбцы, используемые в объединениях должны иметь одинаковые типы данных** 
> если возможно, то лучше использовать числовые типы данных, вместо строковых

- **Избегайте объединять таблицы по столбцам с малым числом уникальных значений**
> eсли столбцы, используемые при объединениях, имеют мало уникальных значений, то будет просмотрена вся таблица, даже если по данному столбцу существует индекс, для наилучшей производительности объединение таблиц должно производится по столбцам с уникальными индексами

- Если нужно регулярно объединять четыре или более таблиц, то попробуйте денормализовать таблицы так, чтобы число таблиц, участвующих в объединении уменьшилось
> часто, при добавлении одного или двух столбцов из одной таблицы в другую, нагрузка может быть уменьшена

- Тип `JOIN` используйте только тот, который вернет вам *НЕОБХОДИМЫЕ* данные без каких-либо дублей или лишней информации

##### *Сортировка в `SELECT`*

- **Самой ресурсоемкой сортировкой является сортировка строк**
- Если сортируете по дате создания записи, то попробуйте сортировать просто по первичному ключу

##### *Группировка в `SELECT`*

- **Используйте как можно меньше колонок для группировки**
- По возможности лучше использовать `WHERE` вместо `HAVING`
> т.к. это уменьшает количество строк для группировки на ранней стадии

- Если требуется группировка, но без использования агрегатных функций: `COUNT()`, `MIN()`, `MAX` и т.д., то разумно использовать `DISTINCT`

- Не используйте множественные вложенные группировки через подзапросы

> по возможности, ограничить использование `DISTINCT`, так как команда требует повышенного времени обработки

#### **Оптимизация `WHERE` в запросе `SELECT`**

- Если `WHERE` состоит из условий, объединенных `AND`, то они должны располагаться в порядке возрастания вероятности истинности данного условия
> т.е. чем быстрее мы получим `false` в одном из условий, тем меньше условий будет обработано и тем быстрее будет выполнен запрос

- Если `WHERE` состоит из условий, объединенных `OR`, то они должны располагаться в порядке уменьшения вероятности истинности данного условия
> т.е. чем быстрее мы получим `true` в одном из условий, тем меньше условий будет обработано и тем быстрее будет выполнен запрос

- Исопльзуйте `IN` вместо `OR`
> операция `IN` работает гораздо быстрее, чем серия `OR`

- Используйте `EXISTS` вместо `count > 0` в подзапросах 
```
where exists(select id from t1 where id = t.id)
вместо 
where count(select id from t1 where id = t.id) > 0
```

> `LIKE` эту команду следует использовать только при крайней необходимости, много кушает

#### **Работа с индексами `CREATE INDEX`**

- *Советы по созданию кластерных индексов*
> Кластерные индексы идеальны для запросов, где есть выбор по диапазону или необходима сортировка. Так происходит, потому что данные в кластерном индексе физически отсортированы по какому-то столбцу. Запросы, получающие выгоду от кластерных индексов, обычно включают в себя операторы `BETWEEN, <, >, GROUP BY, ORDER BY`, и агрегативные операторы типа `MAX, MIN, и COUNT`
   
> Кластерные индексы хороши для запросов, которые ищут запись с уникальным значением (типа id) и когда нужно вернуть большую часть данных из записи или всю запись (потому что запрос покрывается индексом)

> Кластерные индексы хороши для запросов, которые обращаются к столбцам с ограниченным числом значений, например столбцы, содержащие данные о странах или штатах. Но если данные столбца мало отличаются, например, значения типа "да/нет", "мужчина/женщина", то такие столбцы вообще не должны индексироваться

> Кластерные индексы хороши для запросов, которые используют операторы `GROUP BY` или `JOIN`

> Кластерные индексы хороши для запросов, которые возвращают много записей, потому что данные находятся в индексе, и нет необходимости искать их где-то еще

> Избегайте помещать кластерный индекс в столбцы, в которых содержатся постоянно возрастающие величины, например, даты, подверженные частым вставкам в таблицу через `INSERT`. Причина в сортировки данных в индексе, т.е. кластерный индекс на инкрементирующемся столбце вынуждает новые данные вставляться в определенную часть страницы индекса, что создает "горячую зону в таблице (происходит перестроение индекса)" и приводит к большому объему дискового ввода-вывода

- *Советы по выбору некластерных индексов*

> Некластерные индексы лучше подходят для запросов, которые возвращают немного записей (включая только одну запись) и где индекс имеет хорошую селективность `более чем 95%`

> Если столбец в таблице не содержит по крайней мере `95%` уникальных значений, тогда очень вероятно, что Оптимизатор Запроса SQL сервера не будет использовать некластерный индекс, основанный на этом столбце. Поэтому добавляйте некластерные индексы к столбцам, которые имеют `хотя бы 95% уникальных записей`. Например, столбец с "Да" или "Нет" не имеет 95% уникальных записей.

> Постарайтесь сделать ваши индексы как можно меньшего размера (особенно для многостолбцовых индексов), что уменьшит число чтений, необходимых для использования индекса

> Если возможно, создавайте индексы на столбцах, которые имеют целочисленные значения. Целочисленные значения имеют меньше потерь производительности, чем, например, строки

> Индекс полезен для запроса только в том случае, если оператор `WHERE` запроса соответствует столбцу (столбцам), которые являются крайними левыми в индексе. Так, если Вы создаете составной индекс, типа `(City, State)`, тогда запрос `WHERE City = 'Хьюстон'` будет использовать индекс, но запрос `WHERE State = 'TX'` не будет использовать индекс


##### *Лучшие кандидаты на установку индекса*
- Это поля, по которым идет `JOIN`
- Поля связи, участвующие в подзапросах
- Поля, по которым идет фильтрация в `WHERE`
- Поля, по которым выполняется сортировка `ORDER BY`
