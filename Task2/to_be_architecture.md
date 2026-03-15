# Task 2 — Проектирование TO-BE архитектуры (Privacy by Design)

## 1. Анализ рисков AS-IS и механизмы их устранения в TO-BE

| Риск конфиденциальности (AS-IS) | Механизм управления риском | Новый блок в C4 |
|---|---|---|
| Данные PII и медицинская тайна в Excel без шифрования | Шифрование at-rest (pgcrypto + Vault), разделение схем БД | HashiCorp Vault, PostgreSQL в МИС |
| Любой доменный пользователь имеет доступ к любым файлам | RBAC + ABAC: доступ только на основании роли и контекста | IAM (Keycloak) |
| Нет аудита доступа к данным | Централизованный неизменяемый лог всех операций | Audit Log Service |
| Данные пациента передаются без шифрования (e-mail, устно) | TLS 1.3 + mTLS для всех каналов; нет данных вне защищённых API | API Gateway, Istio |
| Результаты анализов поступают по e-mail без верификации | API-интеграция с HMAC-верификацией подлинности данных | Lab Integration Service |
| Платёжные данные (карта) не защищены в соответствии с PCI DSS | Токенизация карт, PCI DSS-сертифицированный шлюз | Payment Gateway |
| Нет согласия пациента на обработку ПДн | Consent Management при регистрации + флаг согласия в каждом запросе | IAM (Keycloak) + CRM |
| Нетипичные действия пользователей не отслеживаются | SIEM с ML-корреляцией событий, алертинг на аномалии | SIEM / Monitoring |
| Аналитика невозможна без доступа к сырым ПДн | Многоуровневое хранилище с обезличиванием (Medallion Architecture) | Data Lake / Analytical Layer |
| Нет классификации данных — непонятно, что защищать | Автоматический движок классификации перед загрузкой в хранилище | Data Classification Engine |
| Ключи и секреты в конфигах / на диске | Централизованное управление секретами с аудитом | HashiCorp Vault |
| Один физический сервер — одна точка отказа | K8s кластер + репликация PostgreSQL + резервный сервер | Kubernetes (инфраструктура) |

---

## 2. Принципы Privacy by Design в целевой архитектуре

Все семь принципов PbD встраиваются на уровне архитектуры, а не добавляются постфактум:

1. **Проактивность** — угрозы закрываются до реализации через Threat Modeling
2. **Privacy as Default** — данные закрыты по умолчанию, доступ требует явного разрешения
3. **Privacy Embedded** — шифрование, аудит, минимизация данных встроены в каждый сервис
4. **Full Functionality** — защита не ограничивает бизнес-функции
5. **End-to-End Security** — TLS везде, шифрование at-rest, mTLS между сервисами
6. **Visibility & Transparency** — аудит-лог, мониторинг, алертинг
7. **Respect for User Privacy** — consent management, право на удаление, минимизация

---

## 3. Новые блоки в C4-архитектуре TO-BE

### Уровень C1 — Контекст системы

| Актор | Взаимодействие |
|---|---|
| Пациент | Портал + Мобильное приложение |
| Врач | Медицинская информационная система (МИС) |
| Ресепшен | Портал ресепшена |
| Бухгалтерия / Касса | ERP (1С серверный режим) |
| Лаборатория | API-интеграция (Lab API) |
| Банк / Эквайер | Payment Gateway |
| Роскомнадзор / ФНС | Регуляторная отчётность |
| Голосовой робот | Notification Service |

### Уровень C2 — Контейнеры (новые блоки)

#### Блок 1: API Gateway / Edge Layer
- **Назначение**: единая точка входа для всех клиентов
- **Privacy by Design**: rate limiting, WAF, валидация JWT, фильтрация чувствительных данных из ответов
- **Технологии**: Kong / Nginx + Lua / Envoy
- **Требования**: TLS 1.3 обязателен, HTTP запрещён, HSTS включён

#### Блок 2: Identity & Access Management (IAM)
- **Назначение**: централизованная аутентификация и авторизация
- **Privacy by Design**: RBAC + ABAC, принцип минимальных привилегий, consent management
- **Технологии**: Keycloak (open-source, self-hosted, данные в РФ)
- **Роли**: PATIENT, DOCTOR, RECEPTIONIST, CASHIER, ACCOUNTANT, WAREHOUSE, IT_ADMIN, ANALYST
- **Атрибуты ABAC**: department, clearance_level, data_scope (own_patients / all_patients)

#### Блок 3: Patient Portal (Web + Mobile)
- **Назначение**: самозапись, просмотр своих данных, уведомления
- **Privacy by Design**: пациент видит только свои данные (ABAC), consent flow при регистрации
- **Технологии**: React (Web) + React Native / Flutter (Mobile), бэкенд — Java Spring Boot
- **Ключевые требования**: JWT с коротким TTL (15 мин), refresh token rotation, биометрия

#### Блок 4: Reception Portal
- **Назначение**: управление записями, напоминания пациентам, работа с журналом
- **Privacy by Design**: RBAC — только данные своей смены, маскирование ПДн в интерфейсе
- **Технологии**: React, бэкенд — Java Spring Boot

#### Блок 5: Medical Information System (МИС)
- **Назначение**: электронные медицинские карты (ЭМК), назначения, результаты анализов
- **Privacy by Design**: врач видит только своих пациентов (ABAC), ЭМК шифруется на уровне БД
- **Технологии**: Java Spring Boot, PostgreSQL с pgcrypto
- **Интеграции**: Lab Integration Service, Notification Service

#### Блок 6: Lab Integration Service
- **Назначение**: API-интеграция с внешней лабораторией анализов
- **Privacy by Design**: передача минимальных данных (только patient_token, не ФИО), mTLS, HMAC-верификация результатов
- **Технологии**: Java Spring Boot, Kafka (для асинхронной доставки результатов)

#### Блок 7: Payment Gateway Service
- **Назначение**: обработка платежей, интеграция с банком-эквайером и ККМ
- **Privacy by Design**: токенизация карт (PAN не хранится), минимальные ПДн при платеже
- **Технологии**: Java Spring Boot, интеграция с сертифицированным Payment Processor
- **Соответствие**: PCI DSS

#### Блок 8: CRM Service
- **Назначение**: управление клиентской базой, история обращений
- **Privacy by Design**: consent tracking (что разрешено, что нет), Data Minimization (нет лишних полей)
- **Технологии**: Java Spring Boot, PostgreSQL

#### Блок 9: Notification Service
- **Назначение**: SMS, Push, e-mail, голосовые напоминания
- **Privacy by Design**: шаблоны без ПДн в теле (только «у вас запись завтра»), минимальные данные в payload
- **Технологии**: Java Spring Boot + интеграции с Twilio/SMS-провайдером, Firebase FCM

#### Блок 10: Audit Log Service
- **Назначение**: централизованный журнал всех действий с данными
- **Privacy by Design**: полная прозрачность — кто, что, когда, откуда; алертинг на аномалии
- **Технологии**: Kafka → Elasticsearch / OpenSearch + Kibana, Victoria Metrics для метрик
- **Политика**: лог immutable (только append), хранение 5 лет (требование 152-ФЗ)

#### Блок 11: Secret Management (HashiCorp Vault)
- **Назначение**: управление ключами шифрования, секретами, сертификатами
- **Privacy by Design**: ключи шифрования не хранятся в коде или конфигах, ротация ключей автоматически
- **Технологии**: HashiCorp Vault (self-hosted)
- **Интеграция**: все сервисы получают секреты через Vault Agent / Sidecar

#### Блок 12: Data Classification Engine
- **Назначение**: автоматическая классификация данных перед загрузкой в аналитику
- **Детали**: Task 6
- **Технологии**: Python / Java, Apache Spark, Apache Kafka

#### Блок 13: Data Lake / Analytical Layer
- **Назначение**: аналитика, BI, ML/AI на основе данных системы
- **Детали**: секция 3 данного документа и Task 6
- **Технологии**: ClickHouse, Apache Spark

#### Блок 14: SIEM / Monitoring
- **Назначение**: мониторинг инцидентов, алертинг, отслеживание нетипичных действий
- **Privacy by Design**: алертинг при несанкционированном доступе к `MEDICAL_SECRET`
- **Технологии**: Victoria Metrics (метрики), Elasticsearch (логи), Grafana (дашборды), AlertManager

#### Блок 15: ERP / 1С (мигрированная)
- **Назначение**: бухгалтерия, склад, HR (заменяет файловый режим)
- **Privacy by Design**: 1С переведена в серверный режим с СУБД (PostgreSQL), роли внутри 1С
- **Технологии**: 1С:Предприятие серверный режим + PostgreSQL

---

## 4. Схема взаимодействия блоков (C2)

> Полная C4 Container-диаграмма TO-BE архитектуры представлена в файле [to_be_architecture.drawio](to_be_architecture.drawio).

---

## 5. Меры Privacy by Design в каждом блоке

| Блок | Мера PbD | Инструмент |
|---|---|---|
| API Gateway | WAF, rate limiting, фильтрация чувствительных полей из ответов | Kong, ModSecurity |
| IAM | Consent management, RBAC/ABAC, принцип min. привилегий | Keycloak |
| Patient Portal | Scope = только свои данные, согласие при регистрации | JWT scope, Keycloak |
| МИС | Шифрование ЭМК на уровне колонок, аудит доступа к карте | pgcrypto, Vault |
| Lab Integration | Минимальные данные (patient_token), mTLS, HMAC | mTLS, Kafka |
| Payment Gateway | Токенизация карты, нет PAN в хранилище | PCI DSS tokenization |
| Notification Svc | Нет ПДн в теле сообщения | Template engine |
| Audit Log | Неизменяемый лог (immutable), алертинг на аномалии | Kafka, Elasticsearch |
| Vault | Ротация ключей, аудит обращений к секретам | HashiCorp Vault |
| Data Lake | Обезличивание, отдельные слои по чувствительности | Spark, ClickHouse |
