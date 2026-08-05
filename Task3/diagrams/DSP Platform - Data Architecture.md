@startuml DataArchitecture
!theme plain

title DSP Platform - Data Architecture

left to right direction
skinparam shadowing false
skinparam packageStyle rectangle
skinparam linetype ortho

'========================
' Services
'========================

package "Microservices" {

rectangle "API Gateway" as gateway

rectangle "Bidding Service" as bidding
rectangle "Campaign Service" as campaign
rectangle "Statistics Service" as stats
rectangle "Analytics Service" as analytics
rectangle "Finance Service" as finance

}

'========================
' Databases
'========================

database "PostgreSQL\n(Bidding DB)" as bidDB
database "PostgreSQL\n(Campaign DB)" as campDB
database "ClickHouse\n(Statistics DB)" as statDB
database "ClickHouse\n(Analytics DB)" as analDB
database "PostgreSQL\n(Finance DB)" as finDB

'========================
' Infrastructure
'========================

queue "Apache Kafka" as kafka
collections "Redis Cluster" as redis

'========================
' Database per Service
'========================

bidding --> bidDB : owns

campaign --> campDB : owns

stats --> statDB : owns

analytics --> analDB : owns

finance --> finDB : owns

'========================
' Kafka
'========================

gateway --> bidding

bidding --> kafka : Bid Response

campaign --> kafka : Campaign Updated

finance --> kafka : Budget Updated

kafka --> stats

kafka --> analytics

kafka --> finance

kafka --> bidding

'========================
' Redis
'========================

bidding --> redis

campaign --> redis

finance --> redis

'========================
' Notes
'========================

note right of redis
Distributed cache

• Campaigns
• Budgets
• Bid settings
• Targeting

TTL:
30 sec – 30 min
end note

note bottom of kafka

Event streaming

• bid-request
• bid-response
• impression
• click
• auction-win
• campaign-updated
• budget-updated
• finance-event

end note

note bottom of bidDB

OLTP

ACID Transactions

end note

note bottom of statDB

OLAP

Large Event Storage

end note

legend right

== Legend ==

Rectangle  : Microservice

Database   : Service Database

Queue      : Apache Kafka

Collections: Redis Cache

Each service owns
its own database
(Database per Service)

endlegend

@enduml