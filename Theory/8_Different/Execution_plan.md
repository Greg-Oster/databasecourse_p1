# План выполнения запросов

[Оптимизатор запросов](./Query_optimization.md) не всегда может дать идеальное решение. Часто бывает, что запрос необходимо переписать другим образом, чтобы увеличить скорость его работы.

Посмотреть план выполнения запроса в apex.oracle можно в sql commands на вкладке explain.

В MySQL Workbench - Query - Explain current statement

Либо в Oracle можно сделать так:

```sql
EXPLAIN PLAN FOR
  SELECT * FROM students;
```

а затем `SELECT PLAN_TABLE_OUTPUT FROM TABLE(DBMS_XPLAN.DISPLAY());`

| Operation        | Options      | Object   | Rows | Time | Cost | Bytes | FilterPredicates | AccessPredicates |
| ---------------- | ------------ | -------- | ---- | ---- | ---- | ----- | ---------------- | ---------------- |
| SELECT STATEMENT |              |          | 3    | 1    | 3    | 153   |                  |                  |
| TABLE ACCESS     | STORAGE FULL | STUDENTS | 3    | 1    | 3    | 153   |                  |                  |

В данном примере вы должны увидеть нечто похожее.

У нас на таблицу студенты есть индекс.

Давайте сделаем обычный запрос

```sql
select name from students
where n_z = 2
```

и зайдём во вкладку Explain

| Operation        | Options        | Object      | Rows | Time | Cost | Bytes | FilterPredicates | AccessPredicates |
| ---------------- | -------------- | ----------- | ---- | ---- | ---- | ----- | ---------------- | ---------------- |
| SELECT STATEMENT |                |             | 1    | 1    | 1    | 8     |                  |                  |
| TABLE ACCESS     | BY INDEX ROWID | STUDENTS    | 1    | 1    | 1    | 8     |                  |                  |
| INDEX            | UNIQUE SCAN    | STUDENTS_PK | 1    | 1    | 0    |       |                  | "N_Z" = 2        |

Можно увидеть, что поиск записи осуществляется при помощи индекса и стоимость всего запроса равна 1.

Давайте зайдём в таблицу вкладку Indexes, перейдём на индекс и отключим его. И выполним опять тот же самый запрос.

| Operation        | Options      | Object   | Rows | Time | Cost | Bytes | FilterPredicates | AccessPredicates |
| ---------------- | ------------ | -------- | ---- | ---- | ---- | ----- | ---------------- | ---------------- |
| SELECT STATEMENT |              |          | 1    | 1    | 3    | 8     |                  |                  |
| TABLE ACCESS     | STORAGE FULL | STUDENTS | 1    | 1    | 3    | 8     | "N_Z" = 2        | "N_Z" = 2        |

Стала понятно разница? Из-за того, что индексов нет приходится делать полный перебор, что, очевидно, гораздо дороже. Можете включить индексы (или сделать rebuild, чтоб включить его)

Однако индексы серьезно замедлят добавление новых записей в таблицу, т.к. он будет постоянно перестраиваться.

## Cardinality

Или Rows - Количество строк.

Количество элементов - это предполагаемое количество строк, которые будут возвращаться при каждой операции. Оптимизатор определяет количество элементов для каждой операции на основе сложного набора формул, которые используют в качестве входных данных статистику уровня таблицы и столбца (или статистику, полученную путем динамической выборки).

Одна из простейших формул используется, когда в одном запросе таблицы есть один предикат равенства. В этом случае оптимизатор предполагает равномерное распределение и вычисляет количество элементов для запроса путем деления общего числа строк в таблице на количество различных значений в столбце, используемом в предикате предложения where.

Например, таблица имеет 19 строк, в данном атрибуте есть 5 разных записей. План примерно будет вычисляться делением 19/5 = 3.8 = 4 - строки.

Очень важно, чтобы оценка количества элементов была максимально точная, но есть некоторые моменты, которые могут повлияеть на неверную оценку:

- перекос данных
- несколько предикатов одного столбца в одной таблице
- обернутые функций столбцы в предикатах where
- сложные выражения

Т.е. если в предыдущем примере будет 1 строка с уникальным значением в атрибуте, то план, если будет считать, как выше указано, выдаст неверный ответ. Но в Oracle используется гистограмма. Он атоматически определяет столбцы, для которых нужны гистограммы, на основе статистики использования столбцов и наличия перекоса данных.

Можно создать гистограмму вручную (параметры: название схемы, название таблицы, метод оптимизации):

```sql
exec DBMS_STATS.GATHER_TABLE_STATS('HR', 'EMPLOYEES', method_opt=>'FOR COLUMNS SIZE 254 JOB_ID');
```

Для определения правильности предположения оптимизатора можно использовать подобный запрос:

```sql
SELECT COUNT(*)
FROM students
WHERE n_group=1234;
```

В качестве альтернативы вы можете использовать подсказку GATHER_PLAN_STATISTICS в операторе SQL для автоматического сбора более полной статистики времени выполнения. С таким параметром оптимизатор записывает фактическое количество элементов (количество возвращаемых строк) для каждой операции при выполнении оператора. Это количество времени выполнения может затем отображаться в плане выполнения с использованием DBMS_XPLAN.DISPLAY_CURSOR (DBMS_XPLAN.DISPLAY) с параметром формата, установленным в «ALLSTATS LAST». Дополнительный столбец под названием A-Rows, который обозначает фактические возвращенные строки, появится в плане.

```sql
Explain plan for
Select /*+ GATHER_PLAN_STATISTICS */ name, n_group
from students
where n_group = 1111

SELECT plan_table_output
FROM TABLE(DBMS_XPLAN.DISPLAY(FORMAT=>'ALLSTATS LAST'));
```

Обратите внимание, что использование указания (hint)GATHER_PLAN_STATISTICS влияет на время выполнения оператора SQL, поэтому его следует использовать только для целей анализа. Подсказка GATHER_PLAN_STATISTICS не требуется для отображения столбца A-Rows, если для параметра init.ora STATISTICS_LEVEL установлено значение ALL.

## Access method

Метод доступа - показывает, как данные будут доступны из каждой таблицы (или индекса). Метод доступа показан в поле операции (Operation) плана объяснения.

| Operation        | Options      | Object     | Rows | Time | Cost | Bytes | FilterPredicates \* | AccessPredicates |
| ---------------- | ------------ | ---------- | ---- | ---- | ---- | ----- | ------------------- | ---------------- |
| SELECT STATEMENT |              |            | 5    | 1    | 3    | 90    |                     |                  |
| TABLE ACCESS     | STORAGE FULL | STUDENTS\$ | 5    | 1    | 3    | 90    | "N_GROUP" = 4032    | "N_GROUP" = 4032 |

### Full table scan

Table access - storage full.

Считывает все строки из таблицы и отфильтровывает те, которые не соответствуют предикатам условия where. При полном сканировании таблицы будет использоваться многоблочный ввод-вывод (обычно 1 Мбайт ввода-вывода). Полное сканирование таблицы выбирается в том случае, если необходимо получить доступ к большой части строк в таблице, отсутствуют индексы или нельзя использовать имеющиеся индексы или если стоимость самая низкая. Решение использовать полное сканирование таблицы также зависит от следующего:

- Параметр init.ora db_multi_block_read_count
- Параллельная степень
- Указания (точнее тут используется слово hint, хз как нормально перевести)
- Отсутствие полезных индексов
- Использование индекса - более дорогая операция

### Rowid

Table access by rowid - в строке rowid указывается файл данных, блок данных в этом файле и расположение строки в этом блоке. Oracle сначала получает значения rowid либо из предиката предложения WHERE, либо путем сканирования индекса одного или нескольких индексов таблицы. Затем Oracle находит каждую выбранную строку в таблице на основе ее rowid и осуществляет построчный доступ.

### Index unique scan

Только одна строка будет возвращена после сканирования уникального индекса. Он будет использоваться при наличии предиката равенства для уникального (B-дерева) индекса или индекса, созданного в результате ограничения первичного ключа.

```sql
Select s.name, s.n_group
from students$ s, students_hobbies$ sh
where s.n_z = 3 and s.n_z = sh.n_z
```

| Operation        | Options        | Object             | Rows | Time | Cost | Bytes | FilterPredicates \* | AccessPredicates |
| ---------------- | -------------- | ------------------ | ---- | ---- | ---- | ----- | ------------------- | ---------------- |
| SELECT STATEMENT |                |                    | 1    | 1    | 4    | 24    |                     |                  |
| NESTED LOOPS     |                |                    | 1    | 1    | 4    | 24    |                     |                  |
| TABLE ACCESS     | BY INDEX ROWID | STUDENTS\$         | 1    | 1    | 1    | 21    |                     |                  |
| INDEX            | UNIQUE SCAN    | STUDENTS\$\_PK     | 1    | 1    | 0    |       |                     | "S"."N_Z" = 3    |
| TABLE ACCESS     | STORAGE FULL   | STUDENTS_HOBBIES\$ | 1    | 1    | 3    | 3     | "SH"."N_Z" = 3      | "SH"."N_Z" = 3   |

### Index range scan

Сканирование диапазона индекса. Oracle обращается к смежным элементам индекса, а затем использует значения ROWID в индексе для извлечения соответствующих строк из таблицы. Сканирование диапазона индекса может быть ограниченным или неограниченным. Он будет использоваться, когда оператор имеет предикат равенства для неуникального ключа индекса или предикат неравенства или диапазона для уникального ключа индекса. (=, <,>, Как, если не на переднем крае). Данные возвращаются в порядке возрастания столбцов индекса.

```sql
Select s.name, s.n_group
from students$ s, students_hobbies$ sh
where s.n_z < 3 and s.n_z = sh.n_z
```

| Operation        | Options        | Object             | Rows | Time | Cost | Bytes | FilterPredicates \* | AccessPredicates       |
| ---------------- | -------------- | ------------------ | ---- | ---- | ---- | ----- | ------------------- | ---------------------- |
| SELECT STATEMENT |                |                    | 2    | 1    | 5    | 48    |                     |                        |
| HASH JOIN        |                |                    | 2    | 1    | 5    | 48    |                     | "S"."N_Z" = "SH"."N_Z" |
| TABLE ACCESS     | BY INDEX ROWID | STUDENTS\$         | 2    | 1    | 2    | 42    |                     |                        |
| INDEX            | RANGE SCAN     | STUDENTS\$\_PK     | 2    | 1    | 1    |       |                     | "S"."N_Z"<3            |
| TABLE ACCESS     | STORAGE FULL   | STUDENTS_HOBBIES\$ | 3    | 1    | 3    | 9     | "SH"."N_Z"<3        | "SH"."N_Z"<3           |

### Index range scan descending

Концептуально тоже самое, что и сканирование диапазона индекса, но он используется, когда индексом может быть удовлетворено предложение ORDER BY .. DESCENDING.

```sql
Select s.n_z, s.name, s.n_group
from students$ s, students_hobbies$ sh
where s.n_z < 3 and s.n_z = sh.n_z
order by s.n_z desc
```

| Operation        | Options               | Object             | Rows | Time | Cost | Bytes | FilterPredicates \*                     | AccessPredicates                        |
| ---------------- | --------------------- | ------------------ | ---- | ---- | ---- | ----- | --------------------------------------- | --------------------------------------- |
| SELECT STATEMENT |                       |                    | 2    | 1    | 6    | 48    |                                         |                                         |
| NESTED LOOPS     |                       |                    | 2    | 1    | 6    | 48    |                                         |                                         |
| TABLE ACCESS     | BY INDEX ROWID        | STUDENTS\$         | 2    | 1    | 2    | 42    |                                         |                                         |
| INDEX            | RANGE SCAN DESCENDING | STUDENTS\$\_PK     | 2    | 1    | 1    |       |                                         | "S"."N_Z"<3                             |
| TABLE ACCESS     | STORAGE FULL          | STUDENTS_HOBBIES\$ | 1    | 1    | 2    | 3     | "SH"."N_Z"<3 AND "S"."N_Z" = "SH"."N_Z" | "SH"."N_Z"<3 AND "S"."N_Z" = "SH"."N_Z" |

### Index skip scan

Обычно для использования индекса в запросе указывается префикс ключа индекса (передний край индекса). Однако, если все остальные столбцы в индексе упоминаются в операторе, кроме первого столбца, Oracle может выполнить сканирование с пропуском индекса, чтобы пропустить первый столбец индекса и использовать оставшуюся часть. Это может быть полезно, если в начальном столбце составного индекса имеется несколько отдельных значений и много значений в не ведущем ключе индекса.

### Full index scan

Полное сканирование индекса.

Полное сканирование индекса не читает каждый блок в структуре индекса, вопреки тому, что предполагает его имя. При полном сканировании индекса обрабатываются все листовые блоки индекса, но достаточно только блоков ветвлений, чтобы найти первый листовой блок. Он используется, когда все столбцы, необходимые для удовлетворения оператора, находятся в индексе, и это дешевле, чем сканирование таблицы. Он использует одиночные блоки ввода-вывода. Может использоваться в любой из следующих ситуаций:

- Предложение ORDER BY содержит все столбцы индекса, и порядок такой же, как в индексе (также может содержать подмножество столбцов в индексе).
- Запрос требует сортировки слиянием, и все столбцы, на которые есть ссылки в запросе, находятся в индексе.
- Порядок столбцов, на которые есть ссылки в запросе, соответствует порядку ведущих столбцов индекса.
- Предложение GROUP BY присутствует в запросе, а столбцы предложения GROUP BY присутствуют в индексе.

```sql
Select s.name, s.n_group
from students$ s, students_hobbies$ sh
where s.n_group = 4032 and s.n_z = sh.n_z
```

| Operation        | Options        | Object             | Rows | Time | Cost | Bytes | FilterPredicates \*    | AccessPredicates       |
| ---------------- | -------------- | ------------------ | ---- | ---- | ---- | ----- | ---------------------- | ---------------------- |
| SELECT STATEMENT |                |                    | 6    | 1    | 6    | 144   |                        |                        |
| MERGE JOIN       |                |                    | 6    | 1    | 6    | 144   |                        |                        |
| TABLE ACCESS     | BY INDEX ROWID | STUDENTS\$         | 5    | 1    | 2    | 105   | "S"."N_GROUP" = 4032   |                        |
| INDEX            | FULL SCAN      | STUDENTS\$\_PK     | 19   | 1    | 1    |       |                        |                        |
| SORT             | JOIN           |                    | 14   | 1    | 4    | 42    | "S"."N_Z" = "SH"."N_Z" | "S"."N_Z" = "SH"."N_Z" |
| TABLE ACCESS     | STORAGE FULL   | STUDENTS_HOBBIES\$ | 14   | 1    | 3    | 42    |                        |                        |

### Fast full index scan

Это альтернатива полному просмотру таблицы, когда индекс содержит все столбцы, необходимые для запроса, и хотя бы один столбец в ключе индекса имеет ограничение NOT NULL. Его нельзя использовать для исключения операции сортировки, поскольку доступ к данным не следует за индексным ключом. Он также будет читать все блоки в индексе, используя многоблочные чтения, в отличие от полного сканирования индекса.

### Index join

Это объединение нескольких индексов в одной таблице, которые в совокупности содержат все столбцы, на которые есть ссылки в запросе из этой таблицы. Если используется соединение по индексу, то доступ к таблице не требуется, поскольку все соответствующие значения столбца могут быть получены из соединенных индексов. Индексное соединение не может использоваться для устранения операции сортировки.

### Bitmap index

Bitmap индекс использует набор битов для каждого значения ключа и функцию отображения, которая преобразует каждую битовую позицию в rowid. Oracle может эффективно объединять bitmap индексы, которые соответствуют нескольким предикатам в предложении WHERE, используя логические операции для разрешения условий И и ИЛИ.

## Join Method

Метод объединения описывает, как данные двух операторов, производящих данные, будут объединены. Вы можете определить методы соединения, используемые в операторе SQL, посмотрев столбец операций в плане объяснения.

### Hash joins

Хеш-объединения используются для объединения больших наборов данных. Оптимизатор использует меньшую из двух таблиц или источников данных для построения хеш-таблицы на основе ключа соединения в памяти. Затем он сканирует таблицу большего размера и выполняет тот же алгоритм хеширования для столбцов столбцов соединения. Затем он проверяет ранее созданную хеш-таблицу для каждого значения и, если они совпадают, возвращает строку.

```sql
Select s.name, s.n_group
from students$ s, students_hobbies$ sh
where s.n_z < 3 and s.n_z = sh.n_z
```

| Operation        | Options        | Object             | Rows | Time | Cost | Bytes | FilterPredicates \* | AccessPredicates       |
| ---------------- | -------------- | ------------------ | ---- | ---- | ---- | ----- | ------------------- | ---------------------- |
| SELECT STATEMENT |                |                    | 2    | 1    | 5    | 48    |                     |                        |
| HASH JOIN        |                |                    | 2    | 1    | 5    | 48    |                     | "S"."N_Z" = "SH"."N_Z" |
| TABLE ACCESS     | BY INDEX ROWID | STUDENTS\$         | 2    | 1    | 2    | 42    |                     |                        |
| INDEX            | RANGE SCAN     | STUDENTS\$\_PK     | 2    | 1    | 1    |       |                     | "S"."N_Z"<3            |
| TABLE ACCESS     | STORAGE FULL   | STUDENTS_HOBBIES\$ | 3    | 1    | 3    | 9     | "SH"."N_Z"<3        | "SH"."N_Z"<3           |

### Nested loops joins

Соединения с вложенными циклами полезны, когда объединяются небольшие подмножества данных и если существует эффективный способ доступа ко второй таблице (например, поиск по индексу). Для каждой строки в первой таблице (внешней таблице) Oracle обращается ко всем строкам во второй таблице (внутренней таблице). Рассмотрим это как два встроенных цикла FOR. В Oracle Database 11g внутренняя реализация для соединений с вложенными циклами была изменена, чтобы уменьшить общую задержку для физического ввода-вывода, поэтому возможно, что вы увидите два соединения NESTED LOOPS в столбце операций плана, где ранее вы видели только одно в более ранних версиях Oracle.

```sql
Select s.n_z, s.name, s.n_group
from students$ s, students_hobbies$ sh
where s.n_z < 3 and s.n_z = sh.n_z
order by s.n_z desc
```

| Operation        | Options               | Object             | Rows | Time | Cost | Bytes | FilterPredicates \*                     | AccessPredicates                        |
| ---------------- | --------------------- | ------------------ | ---- | ---- | ---- | ----- | --------------------------------------- | --------------------------------------- |
| SELECT STATEMENT |                       |                    | 2    | 1    | 6    | 48    |                                         |                                         |
| NESTED LOOPS     |                       |                    | 2    | 1    | 6    | 48    |                                         |                                         |
| TABLE ACCESS     | BY INDEX ROWID        | STUDENTS\$         | 2    | 1    | 2    | 42    |                                         |                                         |
| INDEX            | RANGE SCAN DESCENDING | STUDENTS\$\_PK     | 2    | 1    | 1    |       |                                         | "S"."N_Z"<3                             |
| TABLE ACCESS     | STORAGE FULL          | STUDENTS_HOBBIES\$ | 1    | 1    | 2    | 3     | "SH"."N_Z"<3 AND "S"."N_Z" = "SH"."N_Z" | "SH"."N_Z"<3 AND "S"."N_Z" = "SH"."N_Z" |

### Sort merge joins

Объединения с сортировкой слиянием полезны, когда условие соединения между двумя таблицами является условием, например, <, <=,> или> =. Объединения с сортировкой слиянием могут работать лучше, чем объединения с вложенными циклами, для больших наборов данных. Объединение состоит из двух этапов:

- Операция сортировки соединения: оба входа отсортированы по ключу соединения.
- Операция объединения слиянием: отсортированные списки объединяются вместе.

Соединение сортировки слиянием более вероятно будет выбрано, если в одной из таблиц есть индекс, который исключит одну из сортировок.

```sql
Select s.name, s.n_group
from students$ s, students_hobbies$ sh
where s.n_group = 4032 and s.n_z = sh.n_z
```

| Operation        | Options        | Object             | Rows | Time | Cost | Bytes | FilterPredicates \*    | AccessPredicates       |
| ---------------- | -------------- | ------------------ | ---- | ---- | ---- | ----- | ---------------------- | ---------------------- |
| SELECT STATEMENT |                |                    | 6    | 1    | 6    | 144   |                        |                        |
| MERGE JOIN       |                |                    | 6    | 1    | 6    | 144   |                        |                        |
| TABLE ACCESS     | BY INDEX ROWID | STUDENTS\$         | 5    | 1    | 2    | 105   | "S"."N_GROUP" = 4032   |                        |
| INDEX            | FULL SCAN      | STUDENTS\$\_PK     | 19   | 1    | 1    |       |                        |                        |
| SORT             | JOIN           |                    | 14   | 1    | 4    | 42    | "S"."N_Z" = "SH"."N_Z" | "S"."N_Z" = "SH"."N_Z" |
| TABLE ACCESS     | STORAGE FULL   | STUDENTS_HOBBIES\$ | 14   | 1    | 3    | 42    |                        |                        |

### Cartesian join

Оптимизатор объединяет каждую строку из одного источника данных с каждой строкой из другого источника данных, создавая декартово произведение двух наборов. Как правило, это выбирается только в том случае, если участвующие таблицы являются маленькими или если одна или несколько таблиц не имеют условий соединения с какой-либо другой таблицей в операторе. Декартовы объединения не распространены, поэтому они могут быть признаком проблемы с оценками мощности, если они выбраны по любой другой причине. Строго говоря, декартово произведение не является объединением.

```sql
Select s.n_z, s.name, s.n_group
from students$ s, students_hobbies$ sh
```

| Operation        | Options                | Object                 | Rows | Time | Cost | Bytes | FilterPredicates \* | AccessPredicates |
| ---------------- | ---------------------- | ---------------------- | ---- | ---- | ---- | ----- | ------------------- | ---------------- |
| SELECT STATEMENT |                        |                        | 266  | 1    | 9    | 5,586 |                     |                  |
| MERGE JOIN       | CARTESIAN              |                        | 266  | 1    | 9    | 5,586 |                     |                  |
| TABLE ACCESS     | STORAGE FULL           | STUDENTS\$             | 19   | 1    | 3    | 399   |                     |                  |
| BUFFER           | SORT                   |                        | 14   | 1    | 6    |       |                     |                  |
| INDEX            | STORAGE FAST FULL SCAN | STUDENTS_HOBBIES\$\_PK | 14   | 1    | 0    |       |                     |                  |

## Join types

Inner, outer, left, right - можно почитать в другом разделе.

## Join Order

Порядок соединения - это порядок, в котором таблицы объединяются в многостоловом операторе SQL. Чтобы определить порядок соединения в плане выполнения, посмотрите на отступ таблиц в столбце операций. В таблице ниже идёт объекдинение студентов со студентами хобби, а затем с таблицей хобби.

| Operation                      | Options        | Object             | Rows | Time | Cost | Bytes | FilterPredicates \* | AccessPredicates           |
| ------------------------------ | -------------- | ------------------ | ---- | ---- | ---- | ----- | ------------------- | -------------------------- |
| SELECT STATEMENT               |                |                    | 2    | 1    | 7    | 92    |                     |                            |
| \_\_HASH JOIN                  |                |                    | 2    | 1    | 7    | 92    |                     | "SH"."HOBBY_ID" = "H"."ID" |
| \_\_\_NESTED LOOPS             |                |                    | 2    | 1    | 7    | 92    |                     |                            |
| \_\_\_\_NESTED LOOPS           |                |                    | 2    | 1    | 7    | 92    |                     |                            |
| \_\_\_\_\_STATISTICS COLLECTOR |                |                    |      |      |      |       |                     |                            |
| \_\_\_\_\_\_HASH JOIN          |                |                    | 2    | 1    | 5    | 54    |                     | "S"."N_Z" = "SH"."N_Z"     |
| \_\_\_\_\_\_\_TABLE ACCESS     | BY INDEX ROWID | STUDENTS\$         | 2    | 1    | 2    | 42    |                     |                            |
| \_\_\_\_\_\_\_\_INDEX          | RANGE SCAN     | STUDENTS\$\_PK     | 2    | 1    | 1    |       |                     | "S"."N_Z"<3                |
| \_\_\_\_\_\_\_TABLE ACCESS     | STORAGE FULL   | STUDENTS_HOBBIES\$ | 3    | 1    | 3    | 18    | "SH"."N_Z"<3        | "SH"."N_Z"<3               |
| \_\_\_\_\_INDEX                | UNIQUE SCAN    | HOBBIES\$\_PK      | 1    | 1    | 0    |       |                     | "SH"."HOBBY_ID" = "H"."ID" |
| \_\_\_\_TABLE ACCESS           | BY INDEX ROWID | HOBBIES\$          | 1    | 1    | 1    | 19    |                     |                            |
| \_\_\_TABLE ACCESS             | STORAGE FULL   | HOBBIES\$          | 1    | 1    | 1    | 19    |                     |                            |

В более сложной инструкции SQL может быть не так просто определить порядок соединения, просматривая отступы таблиц в столбце операций. В этих случаях может быть проще использовать параметр FORMAT в процедурах DBMS_XPLAN для отображения информации схемы для плана, который будет содержать порядок соединения. Например, для генерации информации плана для плана, показанного в таблице, можно использовать следующую опцию формата DBMS_XPLAN;

`DBMS_XPLAN.DISPLAY_CURSOR(FORMAT=>'TYPICAL + OUTLINE'));` - в apex.oracle.com нет прав для запуска DISPLAY_CURSOR.

## Partitioning

## Parallel Execution
