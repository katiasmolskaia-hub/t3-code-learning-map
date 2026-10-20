# T3 Code · как запрос становится работой агента

От окна чата до запуска внешнего агента, сохранения истории и состояния файлов. Изучаем организацию работы на зафиксированном снимке проекта.

Снимок: 4b1024dafe4766ea7c59546936c21ce0fab8acb3. Модель и приложение не запускались.

## Вся система

Центр — T3 как управляющее приложение. Внешний агент и файлы проекта показаны как окружение; окно чата, сервер и хранилище раскрываем отдельными островами.

Масштаб: Приложение и его рабочая среда.

### T3 Code

Вывод; Центр: приложение управления.

Связывает пользовательский интерфейс, историю работы, файлы проекта и внешние программы-агенты.

**Пример:** Можно открыть чат с телефона, но рабочие файлы и процесс агента остаются в среде сервера.

**Код:** README называет T3 agent harness control surface. Сервер Node принимает RPC, организует команды и события; адаптеры обращаются к внешним provider runtime. Собственный нейросетевой движок из этого не следует.

**Неизвестно:** Насколько выбранные сценарии работают в конкретной установке со всеми провайдерами.

**Как узнать:** Проверить отдельный сценарий на выбранном provider и клиенте, сохранив версии.

Источники: [README.md:1](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/README.md#L1-L21), [docs/internals/overview.md:1](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/docs/internals/overview.md#L1-L19), [docs/internals/overview.md:53](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/docs/internals/overview.md#L53-L73).

### Окно чата

Наблюдение; Интерфейс пользователя.

Собирает ввод и показывает состояние. Общая логика клиентов помогает нескольким интерфейсам разговаривать с сервером.

**Пример:** Нажать «отправить» — значит передать намерение серверу; это ещё не значит, что агент уже закончил работу.

**Код:** startThreadTurn создаёт message.dispatch с commandId, threadId, текстом и параметрами доставки. Ветки очереди и управления активным ходом зависят от capabilities.

**Неизвестно:** Совпадение поведения всех кнопок и всех клиентов на практике.

**Как узнать:** Пройти один сценарий в web, desktop и mobile. В этом разборе они не запускались.

Источники: [packages/client-runtime/src/operations/commands.ts:749](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/packages/client-runtime/src/operations/commands.ts#L749-L766), [docs/internals/overview.md:1](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/docs/internals/overview.md#L1-L19).

### Сервер среды

Наблюдение; Подсистема T3.

Принимает обращения клиентов и координирует работу на машине с проектом.

**Пример:** Удалённое окно управляет средой сервера, а не подменяет её своей файловой системой.

**Код:** WebSocket/RPC handler передаёт command в ThreadMessageIntake. Документация явно относит Git, provider-процессы, терминалы и файлы к environment сервера.

**Неизвестно:** Полнота проверки прав во всех RPC и способах подключения.

**Как узнать:** Провести отдельный аудит выбранного метода с разными правами доступа.

Источники: [docs/internals/overview.md:1](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/docs/internals/overview.md#L1-L19), [apps/server/src/ws.ts:1793](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/ws.ts#L1793-L1827).

### Внешний агент

Наблюдение; Среда T3: исполнитель.

Выполняет работу через собственный runtime. T3 разговаривает с ним через адаптер и общий контракт.

**Пример:** Два агента могут поддерживать разные способы остановки, продолжения и управления активным запросом.

**Код:** ProviderTurnStartService выбирает session.startTurn или compactThread и передаёт подготовленный текст. Это граница между управлением T3 и внешним исполнением.

**Неизвестно:** Какие гарантии даёт конкретная версия каждого внешнего агента.

**Как узнать:** Читать его контракт и тестировать сценарий с выбранным адаптером; не переносить результат одного провайдера на все.

Источники: [README.md:1](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/README.md#L1-L21), [apps/server/src/orchestration-v2/ProviderTurnStartService.ts:1149](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/ProviderTurnStartService.ts#L1149-L1184).

### SQLite

Наблюдение; Хранилище T3.

Сохраняет события и производные представления состояния, чтобы работа не зависела только от открытого окна.

**Пример:** Чат можно закрыть, а затем снова прочитать сохранённое состояние. Это отличается от сохранения живого процесса агента.

**Код:** События, projection, receipt и outbox фиксируются в транзакции. SQLite настроена на WAL и synchronous=NORMAL. Комментарий прямо различает падение приложения и потерю последних commits при сбое ОС или питания.

**Неизвестно:** Какие записи переживут конкретный аварийный сбой и как быстро восстановится клиент.

**Как узнать:** Испытать выбранную точку прерывания на копии тестовых данных.

Источники: [apps/server/src/orchestration-v2/EventSink.ts:559](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/EventSink.ts#L559-L624), [apps/server/src/orchestration-v2/EventSink.ts:239](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/EventSink.ts#L239-L258), [apps/server/src/persistence/Sqlite.ts:17](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/persistence/Sqlite.ts#L17-L39).

### Файлы проекта

Наблюдение; Среда работы агента.

Здесь находятся результаты действий над кодом. Их состояние нужно согласовывать с историей разговора.

**Пример:** История может говорить, что файл менялся, а файлы на диске уже могли измениться ещё раз вне T3.

**Код:** Environment владеет файлами и Git. Checkpoint — отдельный механизм снимка workspace; он не равен событию в SQLite и не фиксирует всю внешнюю среду.

**Неизвестно:** Какие сторонние изменения попадут в конкретный снимок и что останется вне его.

**Как узнать:** Проверить набор файлов и Git-статусы на тестовом проекте перед и после снимка.

Источники: [docs/internals/overview.md:1](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/docs/internals/overview.md#L1-L19), [apps/server/src/orchestration-v2/CheckpointService.ts:381](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/CheckpointService.ts#L381-L426).

## Путь запроса

Центр — один пользовательский запрос. Схема показывает главные границы, а не каждый вызов функций.

Масштаб: Команда → сохранение → действие → отображение.

### Команда

Наблюдение; Данные намерения.

Описывает действие с идентификатором и нужными параметрами. Команда проходит серверный путь принятия.

**Пример:** Одна и та же повторная отправка с тем же commandId должна соотноситься с уже сохранённым решением.

**Код:** message.dispatch содержит commandId, threadId, messageId, текст и dispatchMode. EventSink резервирует receipt; существующий commandId возвращает сохранённый результат.

**Неизвестно:** Поведение при каждом сочетании потери соединения и повторной отправки со стороны клиента.

**Как узнать:** Разорвать соединение в выбранной точке и проверить commandId, receipt и историю.

Источники: [packages/client-runtime/src/operations/commands.ts:749](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/packages/client-runtime/src/operations/commands.ts#L749-L766), [apps/server/src/orchestration-v2/EventSink.ts:559](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/EventSink.ts#L559-L624).

### Оркестратор

Наблюдение; Подсистема решения.

Сопоставляет команду с состоянием беседы и определяет следующий переход.

**Пример:** Запрос во время активной работы может означать очередь или управление текущим ходом — это разные переходы.

**Код:** В обзорной документации описаны сериализация команд и решение событий без provider/filesystem work. Внешние действия передаются effect worker после фиксации намерения. Это описание ключевого пути, а не доказательство чистоты каждой строки огромного файла.

**Неизвестно:** Полнота инвариантов для всех гонок и комбинаций состояний.

**Как узнать:** Выбрать один инвариант и проверить соответствующий обработчик и целевой тест.

Источники: [docs/internals/overview.md:53](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/docs/internals/overview.md#L53-L73), [packages/client-runtime/src/operations/commands.ts:749](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/packages/client-runtime/src/operations/commands.ts#L749-L766).

### Фиксация решения

Наблюдение; Подсистема сохранения.

Сохраняет принятое решение и задания для дальнейших действий совместно.

**Пример:** Квитанция означает «намерение записано». Она не равна «внешний агент всё сделал».

**Код:** commitCommand пишет events, обновляет projections, ставит effects в outbox и сохраняет receipt. commitThenPublish выполняет транзакцию и публикует события после commit.

**Неизвестно:** Что произошло во внешней программе между выполнением действия и получением подтверждения.

**Как узнать:** Сопоставить устойчивые идентификаторы и журнал провайдера при контролируемом прерывании. Одна SQLite-транзакция это неизвестное не закрывает.

Источники: [apps/server/src/orchestration-v2/EventSink.ts:559](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/EventSink.ts#L559-L624), [apps/server/src/orchestration-v2/EventSink.ts:239](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/EventSink.ts#L239-L258), [docs/internals/overview.md:53](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/docs/internals/overview.md#L53-L73).

### Исполнитель эффектов

Наблюдение; Подсистема исполнения.

Берёт сохранённые задания и выполняет действия: например, запуск хода агента.

**Пример:** Программа сначала записала «нужно запустить», затем отдельный исполнитель действительно делает запуск.

**Код:** Ветка provider-turn.start обращается к providerTurnStart.start; результаты внешней работы возвращаются в orchestration через команды и события.

**Неизвестно:** Поведение каждого типа эффекта при потере процесса на своей границе исполнения.

**Как узнать:** Проверять конкретный effect и recovery-путь; нельзя считать всю внешнюю работу автоматически exactly-once.

Источники: [apps/server/src/orchestration-v2/EffectWorker.ts:144](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/EffectWorker.ts#L144-L156), [docs/internals/overview.md:53](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/docs/internals/overview.md#L53-L73).

### Внешний агент

Наблюдение; Среда T3: исполнитель.

Выполняет работу через собственный runtime. T3 разговаривает с ним через адаптер и общий контракт.

**Пример:** Два агента могут поддерживать разные способы остановки, продолжения и управления активным запросом.

**Код:** ProviderTurnStartService выбирает session.startTurn или compactThread и передаёт подготовленный текст. Это граница между управлением T3 и внешним исполнением.

**Неизвестно:** Какие гарантии даёт конкретная версия каждого внешнего агента.

**Как узнать:** Читать его контракт и тестировать сценарий с выбранным адаптером; не переносить результат одного провайдера на все.

Источники: [README.md:1](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/README.md#L1-L21), [apps/server/src/orchestration-v2/ProviderTurnStartService.ts:1149](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/ProviderTurnStartService.ts#L1149-L1184).

### Приём событий агента

Наблюдение; Подсистема обратной связи.

Принимает сообщения внешнего агента и приводит их к внутреннему виду для сохранения и показа.

**Пример:** Кусочек текста, сообщение о файле и завершение хода — разные виды поступающих данных.

**Код:** ProviderEventIngestor преобразует message.updated и turn_item.updated в domain events с thread/run/node привязкой. Детали нормализации остаются зависимыми от адаптера.

**Неизвестно:** Как обработается каждое необычное или позднее событие конкретного provider.

**Как узнать:** Воспроизвести последовательность событий адаптера на фиксированной версии.

Источники: [apps/server/src/orchestration-v2/ProviderEventIngestor.ts:620](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/ProviderEventIngestor.ts#L620-L641).

### Состояние для клиента

Наблюдение; Данные отображения.

Показывает клиенту нужное представление работы, а не заставляет окно заново понимать весь журнал.

**Пример:** После подключения окно получает состояние выбранного thread и дальнейшие обновления.

**Код:** EventSink применяет stored events к projection. Shared client-runtime использует subscribeThread и отдельные пути начальной загрузки, resume и применения событий.

**Неизвестно:** Согласованность при каждом варианте reconnect, fork и rollback.

**Как узнать:** Проверить выбранную последовательность сбоев и сравнить восстановленный экран с сохранёнными событиями.

Источники: [apps/server/src/orchestration-v2/EventSink.ts:559](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/EventSink.ts#L559-L624), [packages/client-runtime/src/state/threads.ts:768](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/packages/client-runtime/src/state/threads.ts#L768-L789), [docs/internals/overview.md:1](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/docs/internals/overview.md#L1-L19).

### Окно чата

Наблюдение; Интерфейс пользователя.

Собирает ввод и показывает состояние. Общая логика клиентов помогает нескольким интерфейсам разговаривать с сервером.

**Пример:** Нажать «отправить» — значит передать намерение серверу; это ещё не значит, что агент уже закончил работу.

**Код:** startThreadTurn создаёт message.dispatch с commandId, threadId, текстом и параметрами доставки. Ветки очереди и управления активным ходом зависят от capabilities.

**Неизвестно:** Совпадение поведения всех кнопок и всех клиентов на практике.

**Как узнать:** Пройти один сценарий в web, desktop и mobile. В этом разборе они не запускались.

Источники: [packages/client-runtime/src/operations/commands.ts:749](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/packages/client-runtime/src/operations/commands.ts#L749-L766), [docs/internals/overview.md:1](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/docs/internals/overview.md#L1-L19).

## История и восстановление

Центр — устойчивое состояние T3. История, квитанция и внешнее действие имеют разные гарантии.

Масштаб: Хранение и жизненный цикл.

### SQLite

Наблюдение; Хранилище T3.

Сохраняет события и производные представления состояния, чтобы работа не зависела только от открытого окна.

**Пример:** Чат можно закрыть, а затем снова прочитать сохранённое состояние. Это отличается от сохранения живого процесса агента.

**Код:** События, projection, receipt и outbox фиксируются в транзакции. SQLite настроена на WAL и synchronous=NORMAL. Комментарий прямо различает падение приложения и потерю последних commits при сбое ОС или питания.

**Неизвестно:** Какие записи переживут конкретный аварийный сбой и как быстро восстановится клиент.

**Как узнать:** Испытать выбранную точку прерывания на копии тестовых данных.

Источники: [apps/server/src/orchestration-v2/EventSink.ts:559](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/EventSink.ts#L559-L624), [apps/server/src/orchestration-v2/EventSink.ts:239](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/EventSink.ts#L239-L258), [apps/server/src/persistence/Sqlite.ts:17](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/persistence/Sqlite.ts#L17-L39).

### Фиксация решения

Наблюдение; Подсистема сохранения.

Сохраняет принятое решение и задания для дальнейших действий совместно.

**Пример:** Квитанция означает «намерение записано». Она не равна «внешний агент всё сделал».

**Код:** commitCommand пишет events, обновляет projections, ставит effects в outbox и сохраняет receipt. commitThenPublish выполняет транзакцию и публикует события после commit.

**Неизвестно:** Что произошло во внешней программе между выполнением действия и получением подтверждения.

**Как узнать:** Сопоставить устойчивые идентификаторы и журнал провайдера при контролируемом прерывании. Одна SQLite-транзакция это неизвестное не закрывает.

Источники: [apps/server/src/orchestration-v2/EventSink.ts:559](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/EventSink.ts#L559-L624), [apps/server/src/orchestration-v2/EventSink.ts:239](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/EventSink.ts#L239-L258), [docs/internals/overview.md:53](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/docs/internals/overview.md#L53-L73).

### Состояние для клиента

Наблюдение; Данные отображения.

Показывает клиенту нужное представление работы, а не заставляет окно заново понимать весь журнал.

**Пример:** После подключения окно получает состояние выбранного thread и дальнейшие обновления.

**Код:** EventSink применяет stored events к projection. Shared client-runtime использует subscribeThread и отдельные пути начальной загрузки, resume и применения событий.

**Неизвестно:** Согласованность при каждом варианте reconnect, fork и rollback.

**Как узнать:** Проверить выбранную последовательность сбоев и сравнить восстановленный экран с сохранёнными событиями.

Источники: [apps/server/src/orchestration-v2/EventSink.ts:559](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/EventSink.ts#L559-L624), [packages/client-runtime/src/state/threads.ts:768](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/packages/client-runtime/src/state/threads.ts#L768-L789), [docs/internals/overview.md:1](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/docs/internals/overview.md#L1-L19).

### Исполнитель эффектов

Наблюдение; Подсистема исполнения.

Берёт сохранённые задания и выполняет действия: например, запуск хода агента.

**Пример:** Программа сначала записала «нужно запустить», затем отдельный исполнитель действительно делает запуск.

**Код:** Ветка provider-turn.start обращается к providerTurnStart.start; результаты внешней работы возвращаются в orchestration через команды и события.

**Неизвестно:** Поведение каждого типа эффекта при потере процесса на своей границе исполнения.

**Как узнать:** Проверять конкретный effect и recovery-путь; нельзя считать всю внешнюю работу автоматически exactly-once.

Источники: [apps/server/src/orchestration-v2/EffectWorker.ts:144](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/EffectWorker.ts#L144-L156), [docs/internals/overview.md:53](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/docs/internals/overview.md#L53-L73).

### Продолжение после обновления

Наблюдение; Операция восстановления.

Для некоторых прерванных работ может организовать продолжение с сохранёнными данными. Это не тождественно оживлению прежнего процесса.

**Пример:** Записанное намерение продолжить помогает заново начать работу, но старый запущенный процесс не становится от этого бессмертным.

**Код:** RestartContinuation проверяет continueThreadsAfterServerUpdate, более новую активность и ограничение цепочки. Весь набор recovery-механизмов шире этого одного выбранного пути.

**Неизвестно:** Как будет вести себя конкретный незавершённый инструмент после реального обновления.

**Как узнать:** Проверить именно этот сценарий с заданной версией адаптера и журналами внешнего действия.

Источники: [apps/server/src/orchestration-v2/RestartContinuation.ts:188](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/RestartContinuation.ts#L188-L244), [docs/internals/overview.md:53](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/docs/internals/overview.md#L53-L73).

### Остановка хода

Наблюдение; Операция управления.

Передаёт команду прерывания агенту и обрабатывает итог в рамках поддерживаемого протокола.

**Пример:** Кнопка остановки обращается к внешнему исполнителю; время реакции может различаться между провайдерами.

**Код:** ProviderTurnControlService обрабатывает активную сессию и вызов interrupt. Возможности и момент подтверждения зависят от provider и текущего состояния.

**Неизвестно:** Гарантия прекращения всех порождённых внешних действий.

**Как узнать:** Испытать выбранный инструмент, дочерние процессы и адаптер, сопоставив UI с фактическим состоянием.

Источники: [apps/server/src/orchestration-v2/ProviderTurnControlService.ts:184](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/ProviderTurnControlService.ts#L184-L254), [apps/server/src/orchestration-v2/ProviderTurnStartService.ts:1149](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/ProviderTurnStartService.ts#L1149-L1184).

## Что получает агент

Центр — перенос исторического контекста в другой запуск. Архив, выбранный текст и фактическое чтение не отождествляются.

Масштаб: Преобразование и доставка информации.

### Передача контекста

Наблюдение; Подсистема подготовки входа.

Выбирает исторические материалы, которые будут переданы агенту как контекст. Архив приложения и реальный вход агента различаются.

**Пример:** В истории можно хранить много событий, а в один вход поместить лишь часть. Сохранить сообщение и реально передать его исполнителю — разные действия.

**Код:** historicalMessage принимает некоторые виды turn item; другие возвращают null. selectHistory выбирает целые сообщения по бюджету и сообщает о пропусках. Вложения и native tool/reasoning state не переигрываются как прежнее выполнение.

**Неизвестно:** Какая информация окажется существенной для качества следующего ответа.

**Как узнать:** Сравнить сохранённую историю, фактически доставленный вход и результат на конкретном сценарии.

Источники: [packages/provider-core/src/server/handoffBudget.ts:138](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/packages/provider-core/src/server/handoffBudget.ts#L138-L178), [packages/provider-core/src/server/handoffBudget.ts:223](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/packages/provider-core/src/server/handoffBudget.ts#L223-L276).

### Преобразование истории

Наблюдение; Этап передачи контекста.

Не все записи приложения превращаются в сообщения для внешнего агента. Для некоторых типов сохраняется упрощённое текстовое описание.

**Пример:** У file_change в этом преобразовании остаётся имя файла. Это ещё не полный diff файла.

**Код:** historicalMessage оставляет текст user/assistant, команду с кодом выхода и output, ошибку, interrupt result, имя изменённого файла, план. Необработанные типы возвращают null.

**Неизвестно:** Потеряется ли полезное ограничение в конкретной истории из-за такого отбора.

**Как узнать:** Построить небольшую историю с известным важным условием и проверить преобразованные сообщения и фактический вход. Это гипотеза для проверки, не найденный сбой T3.

Источники: [packages/provider-core/src/server/handoffBudget.ts:138](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/packages/provider-core/src/server/handoffBudget.ts#L138-L178).

### Бюджет истории

Наблюдение; Этап передачи контекста.

Ограничивает объём переносимой истории. Приоритеты выбора определены алгоритмом, а не нашим впечатлением о важности текста.

**Пример:** Длинное старое сообщение может не поместиться, даже если содержит важную для человека мысль. Это возможный случай, а не наблюдавшаяся ошибка.

**Код:** Пробуются последнее user, последнее assistant, первое user, затем остальные в обратном порядке. Берутся только полностью помещающиеся элементы; итог возвращается в исходном порядке. Цена оценивается по байтам сериализации с накладными расходами.

**Неизвестно:** Совпадение формального бюджета с фактическим tokenizer и смысловая достаточность выбранных элементов.

**Как узнать:** Сохранить рассчитанный бюджет и точный вход; посчитать токены выбранной моделью и проверить сохранение нужного условия.

Источники: [packages/provider-core/src/server/handoffBudget.ts:223](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/packages/provider-core/src/server/handoffBudget.ts#L223-L276).

### Доставка контекста

Наблюдение; Этап передачи контекста.

Различает подготовленную историю и подтверждённый способ её передачи в native thread.

**Пример:** Когда неизвестно, принял ли внешний runtime данные, повторная отправка может создать дубликат. Код поэтому хранит промежуточный статус.

**Код:** Перед native injection сохраняется pending; при успехе — injected. Повторный pending в том же nativeThreadId даёт UncertainError. Есть inline-путь и отдельный fresh-thread путь при неопределённой прошлой доставке.

**Неизвестно:** Поведение всей цепочки при аварии в конкретный момент передачи.

**Как узнать:** Прервать процесс между отправкой и записью статуса; проверить новую сессию, фактический вход и отсутствие непреднамеренного повторного действия.

Источники: [apps/server/src/orchestration-v2/ContextHandoffDelivery.ts:112](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/ContextHandoffDelivery.ts#L112-L148), [apps/server/src/orchestration-v2/ProviderTurnStartService.ts:673](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/ProviderTurnStartService.ts#L673-L711), [apps/server/src/orchestration-v2/ProviderTurnStartService.ts:1149](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/ProviderTurnStartService.ts#L1149-L1184).

### Дочитывание истории

Наблюдение; Интерфейс к сохранённым данным.

Даёт агенту путь запросить недостающие материалы. Но наличие инструмента не означает, что агент обязательно им воспользуется.

**Пример:** Вместо передачи всего архива можно указать, как дочитать пропущенное. Получится ли полезный ответ, зависит и от поведения исполнителя.

**Код:** handoffCoverage описывает activity view, afterPosition и отдельный textOffset для itemId. В показанной text-функции offset применяется к строковому slice; это курсор текста, не token offset модели.

**Неизвестно:** Дочитает ли исполнитель нужный материал в интересующем сценарии.

**Как узнать:** Записать реальные обращения t3_thread_read и сверить прочитанные диапазоны с вопросом.

Источники: [packages/provider-core/src/server/handoffBudget.ts:223](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/packages/provider-core/src/server/handoffBudget.ts#L223-L276), [apps/server/src/mcp/OrchestratorMcpService.ts:861](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/mcp/OrchestratorMcpService.ts#L861-L896).

### Внешний агент

Наблюдение; Среда T3: исполнитель.

Выполняет работу через собственный runtime. T3 разговаривает с ним через адаптер и общий контракт.

**Пример:** Два агента могут поддерживать разные способы остановки, продолжения и управления активным запросом.

**Код:** ProviderTurnStartService выбирает session.startTurn или compactThread и передаёт подготовленный текст. Это граница между управлением T3 и внешним исполнением.

**Неизвестно:** Какие гарантии даёт конкретная версия каждого внешнего агента.

**Как узнать:** Читать его контракт и тестировать сценарий с выбранным адаптером; не переносить результат одного провайдера на все.

Источники: [README.md:1](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/README.md#L1-L21), [apps/server/src/orchestration-v2/ProviderTurnStartService.ts:1149](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/ProviderTurnStartService.ts#L1149-L1184).

## Файлы и откат

Центр — согласование состояния workspace и разговора с provider. Git-снимок и журнал SQLite различаются.

Масштаб: Файлы и внешняя сессия.

### Файлы проекта

Наблюдение; Среда работы агента.

Здесь находятся результаты действий над кодом. Их состояние нужно согласовывать с историей разговора.

**Пример:** История может говорить, что файл менялся, а файлы на диске уже могли измениться ещё раз вне T3.

**Код:** Environment владеет файлами и Git. Checkpoint — отдельный механизм снимка workspace; он не равен событию в SQLite и не фиксирует всю внешнюю среду.

**Неизвестно:** Какие сторонние изменения попадут в конкретный снимок и что останется вне его.

**Как узнать:** Проверить набор файлов и Git-статусы на тестовом проекте перед и после снимка.

Источники: [docs/internals/overview.md:1](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/docs/internals/overview.md#L1-L19), [apps/server/src/orchestration-v2/CheckpointService.ts:381](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/CheckpointService.ts#L381-L426).

### Снимок файлов

Наблюдение; Состояние workspace.

Фиксирует файловое состояние через Git, чтобы можно было сравнивать и восстанавливать. Его успех нужно смотреть отдельно от завершения хода.

**Пример:** Агент завершил ответ, но снимок файлов может иметь статус missing или error.

**Код:** CheckpointService выставляет missing для неподходящего Git workspace и error при ошибке capture. CheckpointCaptureService может записать run.status=completed вместе с checkpointId; это не доказательство ready-снимка.

**Неизвестно:** Полнота восстановимого состояния в конкретном проекте.

**Как узнать:** Проверить checkpoint.status, состав файлов и восстановление на отдельной копии проекта.

Источники: [apps/server/src/orchestration-v2/CheckpointService.ts:381](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/CheckpointService.ts#L381-L426), [apps/server/src/orchestration-v2/CheckpointCaptureService.ts:201](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/CheckpointCaptureService.ts#L201-L216).

### Откат

Наблюдение; Операция согласования.

Должен согласовать историю внешнего агента и выбранный файловый снимок. Эти операции находятся по разные стороны границ.

**Пример:** Вернуть только файлы может быть недостаточно, если агент продолжает опираться на разговор о более поздних изменениях.

**Код:** В показанном участке вызывается session.rollbackThread, затем checkpoints.restore. По этой последовательности нельзя заключить единую атомарную транзакцию через runtime и файловую систему.

**Неизвестно:** Итог при сбое между этими действиями для выбранного provider.

**Как узнать:** Воспроизвести прерывание между откатом разговора и восстановлением файлов на тестовом проекте.

Источники: [apps/server/src/orchestration-v2/CheckpointRollbackService.ts:281](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/CheckpointRollbackService.ts#L281-L300), [docs/internals/overview.md:1](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/docs/internals/overview.md#L1-L19).

### Внешний агент

Наблюдение; Среда T3: исполнитель.

Выполняет работу через собственный runtime. T3 разговаривает с ним через адаптер и общий контракт.

**Пример:** Два агента могут поддерживать разные способы остановки, продолжения и управления активным запросом.

**Код:** ProviderTurnStartService выбирает session.startTurn или compactThread и передаёт подготовленный текст. Это граница между управлением T3 и внешним исполнением.

**Неизвестно:** Какие гарантии даёт конкретная версия каждого внешнего агента.

**Как узнать:** Читать его контракт и тестировать сценарий с выбранным адаптером; не переносить результат одного провайдера на все.

Источники: [README.md:1](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/README.md#L1-L21), [apps/server/src/orchestration-v2/ProviderTurnStartService.ts:1149](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/ProviderTurnStartService.ts#L1149-L1184).

### SQLite

Наблюдение; Хранилище T3.

Сохраняет события и производные представления состояния, чтобы работа не зависела только от открытого окна.

**Пример:** Чат можно закрыть, а затем снова прочитать сохранённое состояние. Это отличается от сохранения живого процесса агента.

**Код:** События, projection, receipt и outbox фиксируются в транзакции. SQLite настроена на WAL и synchronous=NORMAL. Комментарий прямо различает падение приложения и потерю последних commits при сбое ОС или питания.

**Неизвестно:** Какие записи переживут конкретный аварийный сбой и как быстро восстановится клиент.

**Как узнать:** Испытать выбранную точку прерывания на копии тестовых данных.

Источники: [apps/server/src/orchestration-v2/EventSink.ts:559](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/EventSink.ts#L559-L624), [apps/server/src/orchestration-v2/EventSink.ts:239](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/orchestration-v2/EventSink.ts#L239-L258), [apps/server/src/persistence/Sqlite.ts:17](https://github.com/pingdotgg/t3code/blob/4b1024dafe4766ea7c59546936c21ce0fab8acb3/apps/server/src/persistence/Sqlite.ts#L17-L39).

## Границы

Вопрос: как T3 организует работу внешнего агента и сохраняет состояние. Источники: локальный снимок 17.10.2026, чтение выбранных участков кода и документации. T3, внешние агенты, соединения и рабочие сценарии не запускались.

Язык карты: выбранная рабочая редакция «Архитектурное ядро 0.3». Отдельные виды явно меняют масштаб. Среда зависит от выбранной границы; база данных здесь относится к внутреннему состоянию T3.

Наблюдения содержат ссылки на конкретный commit. Связи и обобщения — выводы по этим наблюдениям. Контрпример или воспроизведение на другой версии способны изменить вывод. Карта не заменяет аудит авторизации, всех адаптеров, сценариев потери связи и миграций.

Альтернатива: изучать только UI или только runtime конкретного provider. Это даст другую карту и другой набор неизвестных. Гипотетическая потеря ограничения в контексте не выдаётся за воспроизведённый дефект T3.

## Авторство и источники

Исходный проект: T3 Code. Copyright (c) 2026 T3 Tools Inc. Фрагменты исходников приведены по лицензии MIT; ссылки ведут на зафиксированную версию проекта.

Учебный замысел, вопросы и направление работы — Катя. Понятийная рамка — «Архитектурное ядро», выбранное для этого учебного проекта.

Объяснения, код интерактивной карты и проверка привязки фрагментов к исходникам подготовлены с помощью Codex — ИИ-помощника OpenAI — в диалоге с Катей.

Это неофициальный учебный материал. Он не является официальной документацией T3 Tools или публикацией OpenAI.

Основание разбора: снимок исходников от 17.10.2026, commit 4b1024dafe4766ea7c59546936c21ce0fab8acb3. Изучались выбранные участки кода; T3 и внешние агенты в этом исследовании не запускались.

Лицензия фрагментов T3: [MIT](LICENSE.T3Code.txt).
