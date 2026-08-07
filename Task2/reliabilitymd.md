# reliability.md

# Reliability Strategy — RTB Bidding Service

## 1\. Назначение

RTB-платформа должна обеспечивать стабильную работу при высокой нагрузке и частичных отказах компонентов.

Bidding Service является критическим компонентом системы, так как участвует в режиме реального времени в процессе аукциона. Основные требования:

- время ответа Bid Request: **P95 ≤ 100 мс**;
- отсутствие единой точки отказа;
- сохранение работоспособности при деградации зависимых сервисов;
- предотвращение финансовых ошибок;
- поддержка горизонтального масштабирования.

Для обеспечения надёжности используются:

- развёртывание Active-Active;
- Circuit Breaker;
- Retry с ограничением для hot path;
- идемпотентность операций;
- Fallback стратегии;
- Kafka Cluster с отказоустойчивой конфигурацией.
- Redis Cluster failover.

---

## 2\. Общая архитектурная схема

```text
                              								DSP
                                                             │
                                                             ▼
                                                       API Gateway
                                                             │
                                                ┌────────────┴────────────┐
                                                ▼                         ▼
                                        Bidding Instance A        Bidding Instance B
                                                │                         │
                                                └────────────┬────────────┘
                                                             │
                                      ┌──────────────────────┼──────────────────────┐
                                      ▼                      ▼                      ▼
                               Circuit Breaker       Circuit Breaker       Circuit Breaker
                                      │                      │                      │
                                      ▼                      ▼                      ▼
                               Campaign Service      Budget Service      Targeting Service
                                      │                      │                      │
                                      └──────────────┬───────┴──────────────┬───────┘
                                                     ▼
                                             Fallback Strategy
                                        ├── Redis Cache
                                        ├── Default Bid
                                        └── No Bid
                              
                                      Impression / Click / Conversion
                                                   │
                                                   ▼
                                             Kafka Cluster
                                      RF=3 / minISR=2 / acks=all
```

---

## 3\. Failover стратегия

### 3\.1 Bidding Service — Active-Active

Bidding Service работает в режиме **Active-Active**.
Все экземпляры сервиса одновременно принимают запросы ставок и могут независимо обработать любой Bid Request.

Основные принципы:

- сервис является stateless;
- состояние хранится во внешних системах;
- экземпляры равноправны;
- масштабирование выполняется горизонтально;
- отказ одного экземпляра не влияет на доступность сервиса.

---

### 3\.2 Поведение при отказе Bidding Instance

При отказе одного экземпляра:

1. Health Check обнаруживает проблему.
2. API Gateway прекращает маршрутизацию запросов.
3. Трафик перераспределяется между оставшимися экземплярами.
4. Оркестратор запускает новый экземпляр сервиса.

Целевые показатели восстановления:


|Метрика|Значение|
|:---|---:|
|Detection Time|≤ 5 секунд|
|Traffic Rerouting|≤ 1 секунда|
|Instance Recovery|≤ 30 секунд|

---

## 4\. Failover зависимых сервисов

### 4\.1 Campaign Service

Campaign Service предоставляет данные о кампаниях, таргетинге и креативах.
При недоступности сервиса Bidding Service использует кэш Redis.


|Параметр|Значение|
|:---|:---|
|Cache TTL|60 сек|
|Max Data Age|5 мин|

При отсутствии актуальных данных возвращается **No Bid**.

---

### 4\.2 Budget Service

Budget Service отвечает за контроль расходов рекламных кампаний.

Отказ Budget Service является критичным, так как может привести к превышению бюджета.
При отказе Budget Service используется заранее зарезервированный Reserved Budget Pool. После его исчерпания Bidding Service возвращает **No Bid**.
Это предотвращает неконтролируемое расходование бюджета.

### 4\.3 Целевые показатели доступности и восстановления


|Показатель|Значение|
|:---|---:|
|Availability|≥ 99.9%|
|RTO|≤ 30 сек|
|RPO (финансовые операции)|0|
|RPO (аналитические события Kafka)|≤ 1 событие|

---

## 5\. Kafka Reliability Configuration

Kafka используется для асинхронной обработки событий:

* Impression;
* Click;
* Conversion;
* Billing events.

Kafka не используется в Bid Request hot path, так как запись события не должна влиять на время ответа аукциона.

---

### 5\.1 Конфигурация Kafka

```yaml
topic: ad-events
partitions: 12
replication.factor: 3
min.insync.replicas: 2

acks: all
enable.idempotence: true
max.in.flight.requests.per.connection: 1
retries: 3
delivery.timeout.ms: 30000
```

Конфигурация обеспечивает:

- хранение трёх копий сообщений;
- запись только при наличии минимум двух синхронных реплик;
- идемпотентную запись;
- устойчивость к отказу одного брокера.

---

## 6\. Redis Cluster Configuration

Redis используется для хранения данных, необходимых для быстрого принятия решения:

* активные кампании;
* targeting rules;
* budget counters;
* резервные данные (fallback data).


|Параметр|Значение|
|:---|---:|
|Cluster Mode|Enabled|
|Nodes|6|
|Masters|3|
|Replicas|3|
|Automatic Failover|Enabled|
|Replica Migration|Enabled|
|Quorum|Majority|
|Cluster Bus|Enabled|
|Persistence|AOF \+ RDB|

---

При отказе Master:

```text
Master failure
        |
Replica Promotion
        |
New Master
```

---

## 7\. Circuit Breaker

### 7\.1 Назначение

Circuit Breaker предотвращает каскадные отказы при недоступности зависимых сервисов.

Используется для:


|Вызов|Circuit Breaker|
|:---|:---|
|Bidding → Campaign|Да|
|Bidding → Budget|Да|
|Bidding → Targeting|Да|
|API Gateway → Bidding|Да|

---

### 7\.2 Состояния Circuit Breaker


|Состояние|Поведение|
|:---|:---|
|Closed|Все запросы проходят к сервису|
|Open|Запросы блокируются, используется Fallback|
|Half-Open|Выполняются 10 тестовых запросов|

---

## 7\.3 Circuit Breaker Thresholds


|Параметр|Значение|
|:---|:---|
|Failure Rate Threshold|50%|
|Slow Call Threshold|50%|
|Slow Call Duration|50 ms|
|Timeout Rate|20%|
|Minimum Calls|100|
|Open Timeout|5 s|
|Half-Open Requests|10|
|Success Threshold|90%|

---

## 8\. Deadline и Retry Policy

### 8\.1 Общий Deadline

RTB SLA:

```text
P95 <= 100 ms
```

Для внутренних вызовов используется:

```text
Total Deadline = 80 ms
```

Оставшиеся 20 мс используются для:

* API Gateway;
* сетевой задержки;
* сериализации;
* формирования ответа.

**Распределение**:


|Компонент|Deadline|
|:---|---:|
|API Gateway|5 мс|
|Bidding Logic|30 мс|
|Campaign Lookup|15 мс|
|Budget Check|10 мс|
|Creative Lookup|10 мс|
|Response Processing|10 мс|

---

### 8\.2 Retry Policy для Hot Path

Стандартный retry:

```text
100 ms
200 ms
400 ms
```

не применяется.

Причина: первый retry превышает допустимый SLA.

Для RTB используется:

```yaml
maxRetries: 1
retryDelay: 5-10 ms
jitter: enabled
```

Retry разрешён только для:

* временных сетевых ошибок;
* UNAVAILABLE;
* RESOURCE_EXHAUSTED.

Retry запрещён для:

* INVALID_ARGUMENT;
* NOT_FOUND;
* бизнес-ошибок.

---

### 8\.3 Ограничение по Deadline

Повторная попытка выполняется только в том случае, если после первой ошибки остаётся достаточно времени до истечения общего deadline (80 мс).

Если оставшееся время недостаточно, Bidding Service немедленно использует Fallback либо возвращает **No Bid**.

Это предотвращает нарушение SLA P95 ≤ 100 мс.

---

## 9\. Идемпотентность операций

Для финансовых операций используется **Idempotency-Key**, который хранится в Redis (TTL — 24 часа) и защищается ограничением `UNIQUE` в базе данных.
При повторном поступлении запроса возвращается результат уже выполненной операции без повторного резервирования бюджета.

---

### 9\.1 Database Constraint

Пример:

```sql
CREATE TABLE budget_reservation (
    id UUID,
    campaign_id UUID,
    amount DECIMAL,
    idempotency_key VARCHAR(128),
    created_at TIMESTAMP,
    UNIQUE(idempotency_key)
);
```

Повторный запрос:

```text
Same Idempotency-Key
        |
Find existing operation
        |
Return previous result
```

Повторное резервирование бюджета не выполняется.

---

## 10\. Fallback Strategies


|Сценарии|Поведение|
|:---|:---|
|Campaign недоступен|Redis Cache|
|Budget недоступен|Reserved Budget Pool|
|Невозможно вычислить ставку|Default Bid|
|Данные отсутствуют|No Bid|

Это предотвращает финансовые ошибки.

---

## 11\. Monitoring

Для контроля надёжности собираются метрики:


|Метрика|Назначение|
|:---|:---|
|Error Rate|количество ошибок|
|Retry Count|количество повторных запросов|
|Circuit Breaker State|состояние защиты|
|Fallback Usage|частота fallback|
|Service Availability|доступность|
|P95 Response Time|соблюдение SLA|
|Redis Availability|состояние кэша|
|Kafka Consumer Lag|Контроль задержки обработки событий|
|Redis Hit Ratio|Эффективность использования кэша|

---

## Обоснование архитектурных решений

- **Active-Active Bidding Service** обеспечивает отсутствие единой точки отказа, позволяет выполнять горизонтальное масштабирование и сохраняет доступность RTB hot path при отказе отдельных экземпляров сервиса.
- **Circuit Breaker** предотвращает каскадные отказы, ограничивая обращения к недоступным зависимым сервисам и позволяя использовать резервные сценарии обработки.
- **Ограниченная Retry Policy** (не более одной повторной попытки с задержкой 5–10 мс) учитывает требования SLA **P95 ≤ 100 мс** и не увеличивает время обработки Bid Request.
- **Kafka** используется только для асинхронной обработки событий (Impression, Click, Conversion и Billing) и не участвует в критическом пути обработки ставок, что исключает влияние событийной обработки на задержку ответа.
- **Kafka replication.factor = 3** и **min.insync.replicas = 2** обеспечивают сохранность событий и устойчивость к отказу отдельных брокеров без потери данных.
- **Redis Cluster** обеспечивает быстрый доступ к критичным данным (кампании, таргетинг, бюджетные счётчики) и поддерживает автоматическое восстановление при отказе узлов.
- **Idempotency Key** совместно с ограничением **UNIQUE** гарантирует однократное выполнение финансовых операций и предотвращает повторное резервирование бюджета при повторной доставке запросов.
- **Fallback-стратегии** (Redis Cache, Default Bid и No Bid) позволяют сохранить корректную работу платформы при частичной недоступности зависимых сервисов, минимизируя влияние отказов на обработку Bid Request и соблюдение SLA.

---