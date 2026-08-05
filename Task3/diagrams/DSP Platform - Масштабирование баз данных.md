@startuml DatabaseScaling
!theme plain

title DSP Platform - Масштабирование баз данных

left to right direction
skinparam shadowing false
skinparam packageStyle rectangle
skinparam linetype ortho

'=========================
' Bidding Service
'=========================

package "Bidding Service" {

database "PostgreSQL\nPrimary" as BidPrimary
database "PostgreSQL\nReplica" as BidReplica

BidPrimary --> BidReplica : Репликация
}

'=========================
' Campaign Service
'=========================

package "Campaign Service" {

database "PostgreSQL\nPrimary" as CampPrimary
database "PostgreSQL\nReplica" as CampReplica

CampPrimary --> CampReplica : Репликация
}

'=========================
' Finance Service
'=========================

package "Finance Service" {

database "PostgreSQL\nPrimary" as FinPrimary
database "PostgreSQL\nReplica" as FinReplica

FinPrimary --> FinReplica : Репликация
}

'=========================
' Statistics Service
'=========================

package "Statistics Service" {

database "ClickHouse\nShard 1" as StatShard1
database "ClickHouse\nShard 2" as StatShard2

database "Replica 1" as StatReplica1
database "Replica 2" as StatReplica2

StatShard1 --> StatReplica1 : Репликация
StatShard2 --> StatReplica2 : Репликация

}

'=========================
' Analytics Service
'=========================

package "Analytics Service" {

database "Read Replica 1" as Read1
database "Read Replica 2" as Read2

StatReplica1 --> Read1
StatReplica2 --> Read2

}

'=========================
' Notes
'=========================

note top of BidPrimary
PostgreSQL

Запись выполняется
только в Primary.

Чтение возможно
из Replica.
end note

note top of StatShard1
ClickHouse

Шардирование по:
• campaignId
или
• eventDate
end note

note top of Read1
Read Replica

Используется
для аналитических
запросов и отчетов.
end note

note bottom of StatReplica2

Кластер ClickHouse

• Репликация
• Горизонтальное масштабирование
• Высокая доступность

end note

legend right

<b>Легенда</b>

Primary
  Основной узел БД,
  принимает операции записи.

Replica
  Резервная копия,
  используется для операций чтения
  и повышения отказоустойчивости.

Shard
  Часть базы данных,
  содержащая фрагмент данных.

Read Replica
  Отдельная реплика,
  предназначенная для аналитических запросов.

Стрелка
  Репликация данных между узлами.

endlegend

@enduml