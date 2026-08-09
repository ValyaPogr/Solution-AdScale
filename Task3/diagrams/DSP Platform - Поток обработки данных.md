@startuml DataFlow
!theme plain

title DSP Platform - Поток обработки данных

left to right direction
skinparam shadowing false
skinparam packageStyle rectangle
skinparam linetype ortho

'=========================
' Внешние системы
'=========================

actor "Пользователь" as User
rectangle "API Gateway" as Gateway
rectangle "Ad Server" as AdServer

'=========================
' Микросервисы
'=========================

rectangle "Bidding Service" as Bidding
rectangle "Campaign Service" as Campaign
rectangle "Statistics Service" as Statistics
rectangle "Analytics Service" as Analytics
rectangle "Finance Service" as Finance

'=========================
' Инфраструктура
'=========================

queue "Apache Kafka" as Kafka
collections "Redis Cluster" as Redis

database "Campaign DB" as CampDB
database "Statistics DB" as StatDB
database "Analytics DB" as AnalDB
database "Finance DB" as FinDB

'=========================
' Поток обработки
'=========================

User --> Gateway : Запрос

Gateway --> Bidding : Bid Request

Bidding --> Redis : Чтение кампаний\nи настроек ставок
Campaign --> Redis : Обновление кэша
Finance --> Redis : Обновление бюджета

Bidding --> Kafka : Bid Response

Campaign --> Kafka : Campaign Updated
Finance --> Kafka : Budget Updated

AdServer --> Kafka : Impression
AdServer --> Kafka : Click

Kafka --> Statistics
Kafka --> Analytics
Kafka --> Finance
Kafka --> Bidding

Statistics --> StatDB
Analytics --> AnalDB
Finance --> FinDB
Campaign --> CampDB

'=========================
' Notes
'=========================

note top of Kafka

Apache Kafka используется
для асинхронного обмена событиями
между микросервисами.

Основные события:

• Bid Request
• Bid Response
• Campaign Updated
• Budget Updated
• Impression
• Click

end note

note right of Redis

Redis используется
для хранения часто
запрашиваемых данных:

• кампании;
• бюджеты;
• настройки ставок.

Позволяет уменьшить
нагрузку на базы данных
и сократить время ответа.

end note

note bottom of Statistics

Сервис статистики
сохраняет события
для последующего анализа.

end note

note bottom of Analytics

Сервис аналитики
формирует отчеты
и аналитические панели
на основе накопленной статистики.

end note

legend right

<b>Легенда</b>

Пользователь
  Внешний клиент системы.

Прямоугольник
  Микросервис.

Database
  База данных сервиса.

Kafka
  Передача событий
  между сервисами.

Redis
  Распределенный кэш.

Стрелки
  Поток данных
  между компонентами.

endlegend

@enduml