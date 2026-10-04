# Проверка проектных решений

## Основное

## Адреса

| Вопрос | Ответ и свидетельство |
| --- | --- |
| Где версия? | В обоих servers.url: /v1/; paths не дублируют версию |
| Какие ресурсы? | Books, Readers, Loans; множественные существительные |
| Где ID? | /books/{id}, /readers/{id}, /loans/{id}; переиспользуемый EntityId |
| Где фильтры? | Query коллекций; book-id/reader-id не спрятаны в body |
| Что с именованием? | JSON snake_case; query/page-size/due-before/registered-after — kebab-case |
| Как возвращается книга? | POST /loans/{id}/return, а не DELETE |

## Методы и модель данных

| Вопрос | Ответ и свидетельство |
| --- | --- |
| Есть ли CRUD? | Books/Readers: Create POST, Read GET списка/id, Update PATCH, Delete DELETE. PUT не является обязательным условием CRUD и сознательно отсутствует |
| Есть ли все Loan операции? | POST/GET коллекции, GET ID, POST возврата: 4 операции |
| Правильны ли статусы? | 201+Location для создания; 200 для чтения/изменения/возврата; 204 без content для удаления |
| Есть ли 404/409/422? | Явные responses, единый ErrorResponse; дополнительно 400/401/415 |
| Отделён ли ввод от вывода? | BookCreation/BookChanges, ReaderRegistration/ReaderChanges, LoanApplication отдельно от ресурсов |
| Можно ли записать серверные поля? | Нет: они отсутствуют в request properties, additionalProperties=false |
| Как работает PATCH? | minProperties=1; отсутствующие поля сохраняются, null запрещён; обычный application/json |
| Как описан null? | returned_at: type=[string,null], а не nullable:true |
| Как считается остаток? | total_copies минус число Loan с returned_at=null; readOnly, только response |
| Что с уменьшением фонда? | Сервер атомарно не допускает total_copies ниже активных выдач |
| Сохраняется ли история? | Логическое удаление каталога/читателя, исторические ID в Loan сохраняются |

## Списки, отказы, повторы

| Вопрос | Ответ и свидетельство |
| --- | --- |
| Фильтры во всех коллекциях? | Books q/isbn/year-from/year-to/in-stock; Readers q/email/registered-after; Loans status/book-id/reader-id/due-before |
| Сортировка во всех коллекциях? | CSV массив sort; разрешённые токены, запрет повторов, ID tie-break |
| Пагинация едина? | PageSize+Cursor, items+page; default=25, max=50; next_cursor/has_next согласованы |
| Есть ли точный total? | Нет, сознательное свойство cursor-контракта; задание не требует total |
| Прописан ли overdue? | GET /loans?status=overdue; null returned_at и due_date<дата UTC; сегодня ещё не overdue |
| Один формат ошибок? | Плоский ErrorResponse: code/message/request_id/violations |
| Все обязательные коды? | INVALID_REQUEST, NOT_FOUND, VALIDATION_ERROR, CONFLICT, BUSINESS_RULE_VIOLATION, IDEMPOTENCY_MISMATCH. UNAUTHORIZED добавлен для JWT, обязательные коды не заменены |
| Есть ли 422 example? | PastDueDate и UnknownInput, с location/pointer/rule |
| Выдача при нуле? | 409 BUSINESS_RULE_VIOLATION/stock_exhausted |
| Шестая активная выдача? | 409 BUSINESS_RULE_VIOLATION/active_loan_quota; overdue входит в лимит |
| Дата новой выдачи? | Строго позже даты UTC; динамическая проверка после reserve нового ключа |
| Запрет удаления с активными выдачами? | Оба DELETE: 409 BUSINESS_RULE_VIOLATION/active_loan_dependency |
| Идемпотентность обязательна? | Header OperationKey required=true на POST Loans |
| Полный replay? | Статус, байты тела, все исходные HTTP-заголовки; timestamp/request_id исходные |
| Другое тело? | 409 IDEMPOTENCY_MISMATCH, включая in-progress |
| Конкурентный одинаковый повтор? | Ожидание до 2 секунд, затем replay либо 409+Retry-After:2, без новой выдачи |
| TTL? | 24 часа от completion, без продления, граница истечения определена |
| Сбой во время обработки? | In-progress не очищается TTL завершённых; восстановление транзакции, не слепой повтор |
| Конкурентные разные ключи? | Атомарные бизнес-проверки по книге/читателю независимо от ключа |

## Документация и проверяемость

| Вопрос | Ответ и свидетельство |
| --- | --- |
| Tags? | Books/Readers/Loans у всех 14 операций |
| Summary/description? | Каждая операция описана, operationId уникальны |
| Минимум три examples? | 16 именованных примеров, включая ошибки и возврат |
| Переиспользуемые компоненты? | Схемы, параметры, responses, examples через внутренние $ref |
| YAML и ссылки? | Проверены при сборке, все $ref разрешаются |
| Схемы и examples? | Локальные проверки JSON Schema 2020-12 с format; не полноценный OAS dialect validator |
| Негативные проверки? | Серверные поля, неизвестные поля, пустой PATCH, null, диапазоны, неверная дата/email/enum |
| Генератор/Swagger/mock? | Не запускались; README не утверждает обратное |
| Бонус? | Bearer JWT объявлен и глобально применён, 401 описан в каждом endpoint |

## Итог по рубрикатору

| Критерий | Самооценка | Что подтверждает |
| --- | --- | --- |
| REST | Полностью в проекте | 14 операций, версия URL, коды, возврат командой |
| Схемы | Полностью в проекте | Раздельный ввод/вывод, закрытые requests, computed поле |
| Ошибки и коллекции | Полностью в проекте | Шесть кодов + UNAUTHORIZED, фильтры/sort/cursor во всех списках |
| Правила и идемпотентность | Полностью в проекте | Нормативное описание POST Loans и связанных изменений |
| Документация | Полностью | Tags/описания/examples |
| STYLE/REVIEW/README | Полностью | Все три документа, 10 правил стиля |
| Дополнительное задание | Полностью в контракте | Bearer JWT + global security + 401 |
