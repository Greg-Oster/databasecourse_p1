# План работы в семестре

1. Знакомство. Сила данных. Зачем нужна СУБД, что такое СУБД? Модели данных. SQL. Нормализация, диаграмма отношений сущностей. Императивный, декларативный подход к получению данных.

   Зачем использовать СУБД?

   - избежать избыточности и несогласованности
   - богатый (декларативный) доступ к данным
   - синхронизация доступа к данным
   - восстановление после системных сбоев
   - безопасность и приватность
   - уменьшение стоимость и ресурсозатрат

   Фундаментальные концепции

   - **Системный подход к структурированию / представлению данных**
   - **Декларативные запросы и обработка запросов**:
   - Высокоуровневый язык получения данных: "Говори, что тебе надо, а не как это сделать"
   - Независимость данных
   - Компилятор, который находит оптимальный план получения данных
   - Множество низкоуровневых подходов для эффективного получения данных
   - **Согласованность / транзакции и контроль параллелизма**
   - Сложные операции можно рассматривать как одну атомарную операцию, которая либо завершается, либо завершается неудачей; без частичного состояния
   - Согласованность и изоляция - семантика параллельных операций чётко определена - эквивалентна последовательному выполнению, с учётом инвариантов во времени.
   - Долговечность - завершенные операции сохраняются после сбоя

   Компоненты СУБД
