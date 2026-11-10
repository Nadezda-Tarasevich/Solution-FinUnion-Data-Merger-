### README: Архитектура данных FinUnion Group — объединение с RetailBank



##### Обзор проекта

Данный документ описывает целевую архитектуру данных после слияния FinUnion Group с RetailBank. Проект направлен на объединение клиентских данных, продуктовой линейки, отчётности и каналов обслуживания без остановки клиентских сервисов.



##### Ключевые цели:

* Получить единый профиль клиента (Customer 360)
* Определить мастер-системы для ключевых сущностей
* Синхронизировать данные между системами в переходный период
* Обеспечить корректную управленческую и регуляторную отчётность
* Разделить операционные и аналитические нагрузки
* Выполнить миграцию без остановки клиентских сервисов



##### Сроки:



Промежуточное состояние — 3 месяца
Целевое состояние — 12 месяцев



###### Документы проекта

\---

README.md - содержит краткий путеводитель по документам проекта
master-systems.md
data-ownership.md
data-flows.md
data-flow-management.md
data-dependencies.md
diagrams/



##### Содержание документов



**master-systems.md**
Для каждой ключевой бизнес-сущности определены мастер-системы в промежуточном (3 месяца) и целевом (12 месяцев) состояниях. При определении использованы результаты маппинга BIAN.



|Сущность|Мастер-система (3 месяца)|Мастер-система (12 месяцев)|BIAN-домен|
|:-:|:-:|:-:|:-:|
|Клиент|MDM (Customer Hub)|MDM (Customer Hub)|Party Reference Data Directory|
|Договор|FU Core (новые) + ESB (синхронизация RB)|FU Core|Product Access|
|Счёт|FU Core (новые) + ESB (синхронизация RB)|FU Core|Current Account / Savings Account / Loan Account|
|Карта|FU Cards (новые) + ESB (статусы из RB)|FU Cards|Card Management|
|Транзакция|Fed DWH (единая витрина)|Data Lakehouse + Fed DWH|Transaction Authorization / Clearing / Settlement|
|Платёж|FU Core (единый шлюз)|FU Core|Payment Execution|
|Продукт|MDM (единый справочник)|MDM|Product Catalog + Product Design|
|Обращение|FU CRM (единая CRM)|FU CRM|Customer Relationship Management|
|Сегмент|MDM (справочник) + FU DWH (правила)|MDM + Data Lakehouse|Segment Management|
|Сотрудник|HR-система (Active Directory)|HR-система + IAM|Employee Management|



**data-ownership.md**
Для каждой ключевой сущности определены роли и ответственность:

|Сущность|Data Owner|Data Steward|Producer|Consumer|
|:-:|:-:|:-:|:-:|:-:|
|Клиент|Data Office + Руководители розницы/МСБ|Data Steward по клиентским данным|FU CRM, RB CRM, MDM|FU CRM, RB CRM, DWH, BI, Контакт-центр|
|Договор|Руководитель продуктового департамента|Data Steward по продуктовым данным|FU Core, RB Core|FU Core, DWH, BI, Риск-менеджмент|
|Счёт|Финансовый департамент|Data Steward по счётным данным|FU Core, RB Core|DWH, BI, Риск-менеджмент, Казначейство|
|Карта|Руководитель карточного департамента|Data Steward по карточным данным|FU Cards, RB Cards|DWH, BI, Фрод-мониторинг|
|Транзакция|Фрод-мониторинг + Финансовый департамент|Data Steward по транзакционным данным|FU Core, FU Cards, RB Core, RB Cards|DWH, BI, Фрод-мониторинг|
|Платёж|Руководитель РКО|Data Steward по платёжным данным|FU Core, RB Core|DWH, BI, Финмониторинг|
|Продукт|Руководитель продуктового департамента|Data Steward по продуктовым справочникам|MDM|FU Core, RB Core, CRM, DWH|
|Обращение|Руководитель контакт-центра|Data Steward по обращениям|FU CRM, RB CRM|DWH, BI, Риск-менеджмент|
|Сегмент|Руководитель маркетинга|Data Steward по сегментации|MDM, Data Lakehouse|CRM, DWH, BI|
|Сотрудник|HR + ИБ|Data Steward по сотрудникам|HR-система (AD)|CRM, Core, DWH, IAM|





**data-flows.md/data-flows.pdf** - контекстная диаграмма потоков данных

Для промежуточного состояния (3 месяца) построена контекстная диаграмма потоков данных.

|Цвет|Тип системы|Компоненты|
|:-:|:-:|:-:|
|Голубой|Системы FinUnion (As-Is)|FU CRM, FU Core, FU Cards, FU DWH, FU Data Lake|
|Оранжевый|Системы RetailBank (As-Is)|RB CRM, RB Core, RB Cards, RB DWH|
|Зелёный|Целевые компоненты (3 мес.)|ESB, Fed DWH, BI|
|Красный|Мастер-системы|MDM (Customer Hub)|

Основные потоки данных:



|№|Источник|Приёмник|Сущности|Тип действия|Режим|
|:-:|:-:|:-:|:-:|:-:|:-:|
|1|Клиент|FU CRM, RB CRM|Клиент|Create/Update|Синхронный|
|2|CRM|Core (АБС)|Клиент, Договор, Счёт|Create/Update|Синхронный|
|3|Core|Cards|Счёт, Карта|Create/Update|Синхронный|
|4|Core, Cards|DWH|Счёт, Договор, Транзакция|Read (batch)|Batch|
|5|Core, Cards, CRM|Data Lake|Транзакции, События, Документы|Create|Streaming/Batch|
|6|Data Lake|Fed DWH|Исторические данные|Read/Enrich|Batch|
|7|CRM|ESB|Клиент (события)|Create/Update|Async|
|8|Core|ESB|Договор, Счёт (события)|Create/Update|Async|
|9|ESB|MDM|Клиент (матчинг)|Create/Update/Deduplicate|Async|
|10|MDM|ESB|Клиент (golden\_id), Продукт|Enrich/Read|Async|
|11|MDM|CRM|Клиент (golden\_id)|Update|Async|
|12|ESB|Fed DWH|Обогащённые данные|Enrich/Create|Async|
|13|FU DWH, RB DWH|Fed DWH|Все сущности (агрегированные)|Read/Validate/Aggregate|Batch|
|14|Fed DWH|BI|Витрины данных|Read/Aggregate|Синхронный|
|15|Fed DWH|Регулятор|Формы ЦБ, МСФО|Read/Validate|Batch|



**data-flow-management.md**- Управление потоками данных
Таблица управления потоками данных включает основные потоки с описанием:



|Поток|Формат|Частота|Тип обмена|SLA|Владелец|
|:-:|:-:|:-:|:-:|:-:|:-:|
|Клиент → CRM|JSON|Онлайн|Синхронный|< 2 сек|Владелец CRM|
|CRM → Core|JSON/XML|Онлайн|Синхронный|< 3 сек|Владелец АБС|
|Core → Cards|JSON/XML|Онлайн|Синхронный|< 2 сек|Владелец карточного процессинга|
|Core → DWH|CSV/Parquet|Ежедневно|Batch|30 мин|Data Steward по счетам|
|Cards → DWH|CSV/Parquet|Ежедневно|Batch|1 час|Data Steward по картам|
|Core, Cards, CRM → Data Lake|Avro/JSON/Binary|Streaming + Ежедневно|Async + Batch|< 5 мин (stream)|Data Steward по Data Lake|
|Data Lake → Fed DWH|Parquet|Еженедельно|Batch|4 часа|Data Steward по Fed DWH|
|CRM → ESB|JSON (CloudEvents)|Near real-time|Async|< 10 сек|Владелец ESB|
|Core → ESB|JSON (CloudEvents)|Near real-time|Async|< 10 сек|Владелец ESB|
|ESB → MDM|JSON (CloudEvents)|Near real-time|Async|< 5 мин|Data Office|
|MDM → ESB|JSON (CloudEvents)|Near real-time|Async|< 2 мин|Data Office|
|MDM → CRM|JSON|Near real-time|Async|< 10 мин|Data Office + Владельцы CRM|
|ESB → Fed DWH|JSON (CloudEvents)|Near real-time|Async|< 5 мин|Data Steward по Fed DWH|
|DWH → Fed DWH|SQL|Ежедневно|Batch|2 часа|Data Steward по Fed DWH|
|Fed DWH → BI|SQL|По запросу|Синхронный|< 10 сек|Data Steward по BI|
|Fed DWH → Регулятор|XML (ЦБ) + XBRL|Ежемесячно|Batch|4 часа|Финансовый департамент|



**data-dependencies.md**- Проблемные зависимости и риски
Документ описывает основные 5 проблем зависимостей или рисков на уровне данных для промежуточного состояния через три месяца.

|№|Риск|Критичность|Затронутые сущности|Влияние на бизнес|
|:-:|:-:|:-:|:-:|:-:|
|1|Разные идентификаторы клиента|Критическая|Клиент, Договор, Счёт, Карта, Кредитный лимит, Обращение, Сегмент|Невозможен единый профиль клиента, дубли в отчётности, сбой кросс-продаж, неверный скоринг|
|2|Несогласованные справочники продуктов|Высокая|Продукт, Договор, Счёт, Карта|Ошибки в финансовой отчётности, жалобы клиентов на тарифы, судебные риски|
|3|Зависимость от ESB (единая точка отказа)|Критическая|Все сущности (через ESB)|Остановка синхронизации данных, расхождения в отчётности, накопление дублей|
|4|Ошибки в отчётности из-за разных форматов|Высокая|Счёт, Транзакция, Кредитный лимит, Договор|Штрафы ЦБ до 1 млн руб., неверные управленческие решения, ручные сверки|
|5|Дублирование обращений клиентов|Высокая|Обращение, Клиент|Снижение NPS, нарушение сроков ЦБ по жалобам, потеря обращений|



