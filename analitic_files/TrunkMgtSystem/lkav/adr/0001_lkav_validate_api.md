# ADR-0001: Архитектура TrunkMgtSystem

- Статус: Согласование
- Дата: 10-06-2026
- Владелец: Бухотецкий Дмитрий

## Контекст

При реализации услуги на плече МТТ при оказании услуги наблюдается ряд проблем:

1. Длинная цепочка получения событий: биллинги МТТ - БД в МТТ- БД в МТС - другие операторы - бит-маски и обратный процесс. Данные теряются на разных этапах и для восстановления всей цепочки необходима проверка в множестве компонентов и наличия логов на каждом этапе.
2. Общая логика реализации услуги: в биллинге нет валидации услуг и соблюдения логики их включения, проблема найдется только после сдачи бага клиентом.

    Необходимо уйти от вкл/откл в биллинге на вкл/отк в продуктах, где инициатор команды знает, что передается:
    1. В тексты подписи заносят некорректные символы
    2. Вкл/выкл услуги за 1 минуту несколько раз
    3. Вкл одновременно АВ и Маркировку
    4. Разбор зависших номеров на вкл/откл из-за несброшенной бит-маски от другого клиента
    5. Вкл МАВ без Маркировки, Маркировку без МАВ

### Проблематика

На команду падает достаточно большое количество операционки и разбора клиентских проблем, что влечёт за собой недовольство сервисом и отсутствие возможности дальнейшего развития.

### Риски

Деградация качества предоставления услуги.

### Требования

- реализовать систему которая снизит риски и вероятность человеческой ошибки;
- поддерживать поддерживать консистенстность данных;

## Решение

Реализация сервиса lkav_validate_api которая возьмёт на себя роль валидатора и входной точки для включения услуги.

- Сотрудники по средством запроса к API будут осуществлять включение/отключение услуги. Тем самым мы убираем риски неверного или неполного набора параметров для включения услуги. API реализует заведомо корректную последовательность действий для включения услуги в АСР что исключает риск ошибки.
- Валидация корректности ввода Этикетки.
- Сокращение пути включения услуги: так как сейчас тригером для включения услуги является АСР система, что не совсем корректно для рахитектуры вцелом, т.к. размазывается зона ответственности для смежных продуктов.
- Единая точка интеграции для всех продуктов.

Доработка сервиса numbers-storage-api который отвечает за логиу подготовки и доставки нумерации и включения услуги в сторону встреных опреаторов: системы заказов(orders).

- Позволит отслеживать статус заказа и "не брать в работу" следйющий заказ на изменение услуги пока не завергился предидущий.
- Следить за консистентностью данных.
- Отслеживать зависшие транзакции в сторону встречных операторов.

## Компонентная диаграмма

```plantuml
@startuml LKAV_Validate_C4_Container
!NEW_C4_STYLE=1
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

LAYOUT_WITH_LEGEND()

title C4 Container Diagram — AV/MAV Order System

' --- Actor ---

Person(dks_employee, "Сотрудник ДКС", "Оператор включения/отключения услуги")

' --- External software systems ---

System_Ext(porta, "Porta", "Software System")
System_Ext(onyma, "Onyma", "Software System")
System_Ext(exolve_lkr, "Exolve LKR", "Software System")
System_Ext(nms, "NMS", "Software System")
System_Ext(dadata, "DaData", "Software System")

' --- System boundary ---

System_Boundary(av_mav_sys, "AV/MAV Order System") {
    Container(validate_api, "AV/MAV Validate API", "Golang", "Интерфейс для сотрудника ДКС")
    Container(number_storage_api, "number-storage-api", "Golang", "AV/MAV Order Service")
    ContainerDb(number_storage_db, "NumberStorageDB", "PostgreSQL")
    ContainerQueue(mts_kafka, "mts kafka", "Kafka")
    Container(mts_callback_worker, "MTS Callback Worker", "Golang")
    ContainerQueue(porta_rmq, "Porta RMQ", "RabbitMQ")
    Container(porta_sync_worker, "porta-numbers-sync-worker", "Golang")
    ContainerQueue(onyma_rmq, "Onyma RMQ", "RabbitMQ")
    Container(onyma_sync_worker, "onyma-numbers-sync-worker", "Golang")
}

' --- Relationships ---

Rel(dks_employee, validate_api, "Запрос на включение/отключение услуги")

Rel(validate_api, exolve_lkr, "Check active order")
Rel(validate_api, onyma, "Enable service")
Rel(validate_api, porta, "Enable service")
Rel(validate_api, number_storage_api, "Activate/deactivate service")
Rel(number_storage_api, validate_api, "Check active order")

Rel(number_storage_api, nms, "get numbers by account")
Rel(number_storage_api, number_storage_db, "Check active operation")
Rel(number_storage_api, dadata, "get OKVED")

Rel(onyma, onyma_rmq, "Событие изменения аккаунта")
Rel(onyma_rmq, onyma_sync_worker, "Событие изменения аккаунта")
Rel(onyma_sync_worker, number_storage_api, "Событие изменения аккаунта")

Rel(porta, porta_rmq, "Enable service")
Rel(porta_rmq, porta_sync_worker, "Enable service")
Rel(porta_sync_worker, number_storage_api, "Событие изменения аккаунта")

Rel(number_storage_api, mts_kafka, "task for operators")
Rel(number_storage_api, mts_kafka, "Событие изменения аккаунта")
Rel(mts_kafka, mts_callback_worker, "task for service")
Rel(mts_callback_worker, number_storage_api, "task for service")

@enduml
```

## Связанные артефакты
