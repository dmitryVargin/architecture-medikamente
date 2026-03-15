# Task 6 — Движок классификации данных (Data Classification Engine)

## 1. Контекст и постановка задачи

Данные поступают в аналитическое хранилище из разных источников (PostgreSQL микросервисов,
1С (ERP), исторические Excel-файлы) и имеют два ключевых свойства:
- **Структура данных изменяется** без предупреждения (schema drift)
- **Поля не размечены** по классам конфиденциальности

Движок должен автоматически классифицировать каждое поле перед загрузкой в хранилище
и маршрутизировать данные в соответствующий слой на основании результатов классификации.

---

## 2. C2-диаграмма: Движок классификации данных

> Визуальная C2-диаграмма движка классификации представлена в файле [classification_engine.drawio](classification_engine.drawio).

### Компоненты движка

**Источники данных:** PostgreSQL (CDC via Debezium), 1С ERP (batch export), Excel (one-time migration), Lab Integration Service (API)

**Ingestion Layer:** Apache Kafka (TLS-шифрование топиков, retention 7 дней, RF=3)

**Schema Detector:** обнаружение schema drift, сравнение с Confluent Schema Registry, алерт в SIEM при изменении

**Classification Engine (Core):**
- Rule-based Classifier — regex-правила: email (RFC 5322), телефон (+7XXXXXXXXXX), ИНН, СНИЛС, паспорт, ФИО (NER), МКБ-10, дата рождения, банковские реквизиты
- ML-based Classifier — spaCy / BERT-multilingual, обучен на примерах ПДн из российских мед. документов, порог уверенности 0.85
- Ensemble Decision — Rule HIGH → принять; Rule LOW → ML; ML < 0.85 → UNKNOWN → Human review

**Metadata Store:** Apache Atlas / OpenMetadata — хранит field_name → classification_tag + confidence + timestamp, REST API, версионирование

**Policy Engine:** на основе тегов применяет трансформации:
- PII / PII_SPECIAL → псевдонимизация (ФИО → UUID)
- MEDICAL_SECRET → шифрование (AES-256-GCM) + ограниченный доступ
- FINANCIAL → токенизация / маскирование
- HR_SENSITIVE → шифрование + изоляция в отдельной схеме
- CONFIDENTIAL → шифрование колонок
- INTERNAL / PUBLIC → без изменений
- UNKNOWN → блокировка загрузки + алерт в SIEM

**Routing Service:** все данные проходят последовательно через Medallion (Landing → Bronze → Silver → Gold), теги определяют трансформации на каждом переходе и максимальную глубину распространения данных. UNKNOWN → Quarantine Zone + Human review queue.

---

## 3. Слои хранилища данных

### Landing Zone (Зона приземления)
- Все сырые данные поступают сюда в исходном формате (JSON, CSV, Parquet)
- Данные MEDICAL_SECRET и PII_SPECIAL дополнительно шифруются на уровне объектов
- Шифрование: AES-256 (MinIO SSE)
- Доступ: только Classification Engine + Data Engineer (с MFA)
- Срок хранения: 90 дней → архив → удаление по политике retention

### Bronze Layer (Первичная обработка)
- Данные PII: псевдонимизированы (ФИО → UUID), остальные поля сохранены
- Данные FINANCIAL: суммы сохранены, идентификаторы карт — токенизированы
- Данные MEDICAL_SECRET: зашифрованы на уровне колонок (ClickHouse encryption)
- Данные HR_SENSITIVE: зашифрованы, изолированы в отдельной схеме, не проходят в Silver
- Маппинг patient_uuid → реальные ПДн хранится только в оперативной БД
- Доступ: DATA_ENGINEER (read/write), DATA_SCIENTIST (read only — ограниченные колонки)

### Silver Layer (Аналитическая обработка)
- Убраны все прямые и косвенные идентификаторы (UUID не передаётся)
- Данные агрегированы или статистически обработаны
- MEDICAL_SECRET и HR_SENSITIVE данные не попадают в этот слой
- Доступ: BI_ANALYST, DATA_SCIENTIST (full read)

### Gold Layer (Витрины для BI)
- Полностью обезличенные агрегаты (Data Marts)
- Автоматическая проверка на наличие PII перед публикацией (regex scan)
- Проверка k-anonymity: в выборке не менее 5 записей
- Доступ: ANALYST, MANAGER (через Superset / Metabase)

### Quarantine Zone (Карантин)
- Данные с тегом UNKNOWN или confidence < 0.85
- Не загружаются в хранилище до ручной разметки
- Human review queue (интерфейс для Data Engineer)
- Алерт в SIEM при появлении данных в карантине

---

## 4. Метрики эффективности классификации

### Метрики качества классификации

| Метрика | Формула | Целевое значение | Как помогает |
|---|---|---|---|
| **Accuracy** | (TP + TN) / Total | ≥ 0.95 | Общая точность классификатора |
| **Precision** (по каждому классу) | TP / (TP + FP) | ≥ 0.90 | Минимизация ложных тревог |
| **Recall** (по каждому классу) | TP / (TP + FN) | ≥ 0.95 для MEDICAL_SECRET | Для критических классов важно не пропустить |
| **F1-score** | 2 × (P × R) / (P + R) | ≥ 0.92 | Баланс между precision и recall |
| **False Negative Rate (FNR)** | FN / (TP + FN) | < 0.02 для PII и MEDICAL | Пропущенные чувствительные данные — критический риск |
| **Quarantine Rate** | UNKNOWN / Total | < 0.05 | Слишком высокий = неэффективный классификатор |
| **Human Review Time** | Avg. время разметки UNKNOWN | < 24 часа | SLA на обработку карантина |

### Метрики производительности

| Метрика | Целевое значение | Инструмент |
|---|---|---|
| Throughput (пакетный режим) | ≥ 50 000 записей / мин | Apache Spark (batch) |
| Throughput (потоковый режим) | ≥ 5 000 событий / сек | Apache Kafka + Flink |
| Latency (p99) | < 2 секунды на запись | Victoria Metrics |
| Availability | ≥ 99.9% | Kubernetes health checks |
| Schema drift detection time | < 5 минут | Schema Registry polling |

### Метрики покрытия данных

| Метрика | Целевое значение |
|---|---|
| Доля классифицированных полей | 100% (блокировка при UNKNOWN) |
| Доля таблиц с тегами в Metadata Store | ≥ 95% в течение 30 дней после запуска |
| Доля данных, прошедших re-classification после schema drift | 100% |

### Как метрики помогают в оптимизации:
- **Высокий FNR для MEDICAL_SECRET** → добавить новые regex-правила или дообучить ML-модель
- **Высокий Quarantine Rate** → расширить обучающий датасет или снизить порог confidence
- **Высокая Latency** → увеличить Spark executor или параллелизм Kafka consumer
- **Падение Accuracy после schema drift** → алерт + re-training пайплайн

---

## 5. Масштабируемость

### Горизонтальное масштабирование

**Apache Kafka:**
- Увеличение партиций топиков при росте нагрузки
- Добавление брокеров в кластер без даунтайма
- Consumer group автоматически перераспределяет партиции

**Apache Spark (batch классификация):**
- Динамическое выделение ресурсов (dynamic allocation)
- Увеличение числа executor при росте объёма данных
- При 5x росте данных: увеличить executor с 4 до 20 → линейный прирост производительности

**Classification Engine (ML-модели):**
- Контейнеризация в Kubernetes: HPA (Horizontal Pod Autoscaler) по CPU/RPS
- Stateless сервисы → горизонтальное масштабирование без ограничений
- ML-модель кешируется в памяти пода (нет обращений к диску при инференсе)

**ClickHouse (аналитическое хранилище):**
- Шардирование: данные распределяются по нескольким нодам
- Репликация: каждый шард реплицируется (ReplicatedMergeTree)
- При добавлении нод — автоматическое перебалансирование (через ZooKeeper)

### Вертикальное масштабирование
- Landing Zone (MinIO): добавление дисков в кластер без перезапуска
- PostgreSQL: read replicas для аналитических запросов

### Масштабирование при росте числа пользователей
- API Gateway: автоскейл подов IAM/Keycloak
- Audit Log Service: Kafka буферизует пики нагрузки, Elasticsearch масштабируется горизонтально

### Расширение при новых источниках данных (новые филиалы)
- Добавить новый Kafka topic для филиала
- Classification Engine автоматически подхватит новый topic через topic subscription pattern
- Новые типы данных → добавить правила в Rule Engine (без перезапуска через hot-reload конфига)

### Пример: сценарий роста 5x

| Компонент | Сейчас (1 офис) | После роста (5 филиалов) | Действие |
|---|---|---|---|
| Kafka | 3 брокера, 12 партиций | 3+ брокера, 60 партиций | Добавить партиции |
| Spark executors | 4 | 20 | Увеличить через Kubernetes |
| ClickHouse nodes | 1 shard × 2 replicas | 5 shards × 2 replicas | Добавить ноды |
| Classification API pods | 2 | 10 | HPA auto-scale |
| Kafka topics | 4 (по процессам) | 20 (по процессам × филиалам) | Добавить через конфиг |

---

## 6. Дополнительные меры (помимо слоёв хранилища)

### Data Lineage (Родословная данных)
- Apache Atlas автоматически строит граф: источник → классификация → слой хранилища → BI
- При любом инциденте (утечка) — можно мгновенно найти, откуда данные попали в систему

### Continuous Re-classification
- Еженедельный Spark job повторно классифицирует данные в Bronze Layer
- Выявляет данные, которые не были корректно классифицированы при первой загрузке

### Consent-aware Loading
- Перед загрузкой данных пациента в аналитику: проверка наличия согласия в CRM
- Если согласия нет → данные не загружаются в Bronze и выше

### Right to be Forgotten (Право на удаление)
- При запросе пациента: пометить patient_uuid как deleted
- Spark job удаляет все записи с этим UUID из Bronze Layer
- В Gold Layer данные уже обезличены — удалять нечего

### Integrity Checks
- Checksums (SHA-256) для каждого батча данных при загрузке в Landing
- Верификация контрольной суммы при переходе между слоями
- Несоответствие → блокировка + алерт
