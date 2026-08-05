@startuml DataArchitecture
!theme plain

title DSP Platform - Архитектура хранения данных

left to right direction

skinparam shadowing false
skinparam packageStyle rectangle
skinparam linetype ortho

'=========================
' Микросервисы
'=========================

package "Микросервисы" {

rectangle "Bidding Service" as Bidding
rectangle "Campaign Service" as Campaign
rectangle "Statistics Service" as Statistics
rectangle "Analytics Service" as Analytics
rectangle "Finance Service" as Finance

}

'=========================
' Хранилища данных
'=========================

database "PostgreSQL\nBidding DB" as BidDB
database "PostgreSQL\nCampaign DB" as CampDB
database "ClickHouse\nStatistics DB" as StatDB
database "ClickHouse\nAnalytics DB" as AnalDB
database "PostgreSQL\nFinance DB" as FinDB

collections "Redis Cluster" as Redis
queue "Apache Kafka" as Kafka

'=========================
' Database per Service
'=========================

Bidding --> BidDB : хранение данных

Campaign --> CampDB : хранение данных

Statistics --> StatDB : хранение статистики

Analytics --> AnalDB : хранение аналитики

Finance --> FinDB : хранение финансов

'=========================
' Redis
'=========================

Bidding --> Redis
Campaign --> Redis
Finance --> Redis

'=========================
' Kafka
'=========================

Bidding --> Kafka
Campaign --> Kafka
Finance --> Kafka

Kafka --> Statistics
Kafka --> Analytics

'=========================
' Notes
'=========================

note top of BidDB

OLTP-хранилище

Транзакционные данные
и ACID-гарантии.

end note

note top of StatDB

OLAP-хранилище

Статистика показов,
кликов и ставок.

end note

note right of Redis

Redis используется
для кэширования:

• кампаний;
• бюджетов;
• настроек ставок.

Не является
основным хранилищем данных.

end note

note bottom of Kafka

Kafka обеспечивает
асинхронный обмен
событиями между сервисами.

end note

legend right

<b>Легенда</b>

Прямоугольник
  Микросервис.

Database
  Собственная база данных сервиса
  (Database per Service).

Redis
  Распределённый кэш.

Kafka
  Передача событий между сервисами.

Каждый сервис владеет
только своей базой данных
и не обращается напрямую
к базам других сервисов.

endlegend

@enduml