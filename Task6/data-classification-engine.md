# Движок классификации данных перед загрузкой в аналитическое хранилище

## 1. Цель решения

Цель — создать отдельный слой, который до загрузки данных в аналитическое хранилище:

- принимает пакетные выгрузки из операционных систем;
- выявляет структурные изменения входных данных;
- классифицирует поля и наборы данных по уровню конфиденциальности;
- применяет политики маскирования, токенизации, псевдонимизации и шифрования;
- изолирует нераспознанные или подозрительные данные;
- сохраняет классификацию, владельцев и Data Lineage;
- допускает в аналитические витрины только проверенные данные;
- обеспечивает масштабирование при росте объёма данных и числа пользователей.

Решение является частью целевой `Data & Analytics Platform` и реализует Privacy by Design до момента использования данных аналитиками, BI, ML и AI.

---

## 2. Основные предпосылки

Входные данные имеют следующие особенности:

- поступают пакетами;
- могут иметь разные форматы;
- могут не содержать метаданных о конфиденциальности;
- их схемы изменяются со временем;
- новые поля могут появляться без предварительного согласования;
- одно и то же поле может называться по-разному в разных системах;
- часть конфиденциальных данных должна сохраняться в аналитическом контуре;
- не все данные можно полностью анонимизировать без потери бизнес-ценности.

Примеры источников:

- CRM / Patient Registry;
- Medical Information System;
- Appointment Service;
- Payment Service;
- Laboratory Integration Service;
- 1С;
- файловые выгрузки;
- журналы событий;
- object storage.

---

## 3. Архитектурная схема обработки

```text
Источники данных
        ↓
Batch Orchestrator
        ↓
Landing Zone
        ↓
Schema Detection
        ↓
Schema Drift Detector
        ↓
Classification Engine
        ↓
Privacy Policy Enforcement
        ↓
Masking / Tokenization / Pseudonymization / Encryption
        ↓
┌───────────────────────────────────────────────┐
│ Quarantine Zone                              │
│ Restricted Zone                              │
│ Protected Zone                               │
│ Trusted Analytics Zone                       │
└───────────────────────────────────────────────┘
        ↓
BI / ML / AI / Reporting
```

---

## 4. Контейнеры решения

## 4.1. Batch Orchestrator

Назначение:

- запускает пакетные загрузки;
- управляет зависимостями;
- повторяет упавшие этапы;
- сохраняет checkpoint;
- контролирует SLA batch-процессов;
- не допускает переход на следующий этап при ошибке классификации.

Возможные инструменты:

- Apache Airflow;
- Argo Workflows;
- Dagster.

---

## 4.2. Landing Zone

Landing Zone принимает данные в исходном виде.

Требования:

- object storage;
- обязательное шифрование;
- отдельный namespace по источникам;
- короткий TTL;
- запрет прямого доступа аналитиков;
- immutable ingest-файлы;
- checksum;
- фиксация времени загрузки;
- регистрация источника и версии batch.

Пример структуры:

```text
landing/
├── patient-registry/
│   └── 2026-07-05/
├── medical-system/
│   └── 2026-07-05/
├── payment-service/
│   └── 2026-07-05/
└── 1c/
    └── 2026-07-05/
```

Landing Zone не считается аналитическим слоем. Это временный технический слой до завершения классификации.

---

## 4.3. Schema Detector

Schema Detector определяет:

- имена полей;
- типы данных;
- nullable;
- вложенность;
- массивы и коллекции;
- допустимые диапазоны;
- статистические характеристики;
- примеры значений;
- кардинальность;
- долю пустых значений.

Поддерживаемые форматы:

- CSV;
- JSON;
- Parquet;
- Avro;
- XML;
- выгрузки из БД;
- XLSX как временный legacy-формат.

Результат:

```json
{
  "source": "patient-registry",
  "dataset": "patients",
  "schemaVersion": "2026-07-05-01",
  "fields": [
    {
      "name": "phone",
      "type": "string",
      "nullable": true,
      "cardinality": 92814
    }
  ]
}
```

---

## 4.4. Schema Registry

Schema Registry хранит:

- версии схем;
- связь схемы с источником;
- дату изменения;
- совместимость;
- владельца;
- историю изменений;
- статус согласования;
- classification tags.

Для каждого dataset должна существовать активная версия схемы.

---

## 4.5. Schema Drift Detector

Schema Drift Detector сравнивает новую схему с последней зарегистрированной версией.

Выявляет:

- добавление поля;
- удаление поля;
- переименование поля;
- изменение типа;
- изменение nullable;
- изменение формата;
- изменение структуры вложенного объекта;
- резкое изменение кардинальности;
- появление новых категориальных значений.

### Типы изменений

| Тип | Пример | Реакция |
|---|---|---|
| Совместимое | Добавлено nullable-поле | Запустить классификацию нового поля |
| Потенциально опасное | `string` → `object` | Quarantine и ручная проверка |
| Несовместимое | Удалено обязательное поле | Остановить pipeline |
| Privacy-sensitive | Появилось поле `diagnosis` | Немедленная классификация как `RESTRICTED` |
| Неопределённое | Поле `value_42` | Quarantine |

### Принцип fail closed

Если новое поле не классифицировано, batch не может автоматически попасть в Trusted Analytics Zone.

---

## 4.6. Classification Engine

Classification Engine определяет:

- тип данных;
- категорию;
- уровень конфиденциальности;
- допустимые способы использования;
- требуемые меры защиты;
- confidence score.

Классификация выполняется на уровне:

- dataset;
- таблицы;
- колонки;
- поля JSON;
- файла;
- отдельного объекта при необходимости.

### Результат классификации

```json
{
  "field": "passport_number",
  "semanticType": "PII.IDENTITY",
  "classification": "RESTRICTED",
  "confidence": 0.99,
  "requiredAction": "TOKENIZE",
  "allowedZones": [
    "RESTRICTED",
    "PROTECTED"
  ],
  "allowedPurposes": [
    "LEGAL_REPORTING"
  ]
}
```

---

## 5. Модель классификации

## 5.1. Классы конфиденциальности

| Класс | Описание | Примеры |
|---|---|---|
| `PUBLIC` | Открытые данные | Перечень услуг |
| `INTERNAL` | Внутренние данные компании | Технические справочники |
| `CONFIDENTIAL` | Персональные и финансовые данные | Ф. И. О., телефон, email |
| `RESTRICTED` | Медицинские, паспортные, кадровые данные | Диагноз, анализы, паспорт |
| `SECRET` | Секреты и ключевой материал | Tokens, credentials, private keys |

## 5.2. Семантические теги

```text
PII.IDENTITY
PII.CONTACT
PII.LOCATION
SENSITIVE.HEALTH
SENSITIVE.FINANCE
SENSITIVE.HR
PAYMENT.TRANSACTION
SECURITY.CREDENTIAL
BUSINESS.INTERNAL
ANALYTICS.AGGREGATED
```

## 5.3. Дополнительные атрибуты

Для каждого поля сохраняются:

```text
classification
semanticType
confidence
sourceSystem
datasetOwner
processingPurpose
retentionPeriod
allowedZones
maskingRule
encryptionRequired
manualReviewRequired
schemaVersion
lineageId
```

---

## 6. Алгоритм классификации

Движок использует гибридный подход.

## 6.1. Этап 1. Детерминированные правила

Проверяются:

- имя поля;
- regex;
- формат;
- тип данных;
- источник;
- путь в структуре;
- словарь доменных терминов.

Примеры:

```text
phone, mobile, contact_phone
    → PII.CONTACT
    → CONFIDENTIAL

diagnosis, anamnesis, icd_code
    → SENSITIVE.HEALTH
    → RESTRICTED

password, api_key, token
    → SECURITY.CREDENTIAL
    → SECRET
```

## 6.2. Этап 2. Анализ значений

Используются безопасные профилирующие функции:

- regex matching;
- entropy;
- length distribution;
- date pattern;
- checksum;
- value distribution;
- uniqueness;
- language and domain dictionaries.

Примеры:

```text
+7 999 123-45-67
    → phone

4510 123456
    → passport-like value

A00.1
    → ICD-like medical code
```

Полные значения не должны сохраняться в логах классификатора.

## 6.3. Этап 3. Контекстный анализ

Учитываются:

- название dataset;
- соседние поля;
- источник;
- бизнес-домен;
- Data Lineage;
- используемый процесс;
- историческая классификация похожих полей.

Пример:

```text
field = result
dataset = laboratory_results
neighbors = patient_id, test_code

classification:
SENSITIVE.HEALTH / RESTRICTED
```

## 6.4. Этап 4. ML-классификатор

ML-модель используется, если правила и контекст не дали однозначного результата.

Вход модели:

- имя поля;
- описание;
- dataset;
- тип;
- статистический профиль;
- соседние поля;
- источник.

Выход:

- semantic class;
- confidentiality class;
- confidence score.

ML не является единственным механизмом принятия решения для критичных данных.

## 6.5. Этап 5. Policy Resolution

Результаты правил и ML объединяются.

Пример:

```text
Rule confidence: 0.92
Context confidence: 0.88
ML confidence: 0.76

Final classification:
RESTRICTED
Final confidence:
0.94
```

Для безопасности используется правило:

```text
При конфликте выбирается более строгий класс.
```

---

## 7. Confidence score и маршрутизация

| Confidence | Решение |
|---:|---|
| `>= 0.95` | Автоматическая классификация и обработка |
| `0.80–0.94` | Обработка только в Restricted/Protected Zone |
| `0.60–0.79` | Quarantine и ручная проверка |
| `< 0.60` | Pipeline останавливается для dataset |

Пороговые значения настраиваются по классам.

Для медицинских и секретных данных порог должен быть строже, чем для `INTERNAL`.

---

## 8. Privacy Rules Service

Privacy Rules Service хранит версии правил:

- классификации;
- допустимых зон;
- маскирования;
- токенизации;
- шифрования;
- retention;
- допустимых целей использования;
- требований ручного согласования.

Пример правила:

```yaml
ruleId: health-diagnosis
match:
  fieldNames:
    - diagnosis
    - icd_code
    - anamnesis
classification: RESTRICTED
semanticType: SENSITIVE.HEALTH
actions:
  - ENCRYPT
  - PSEUDONYMIZE_PATIENT_ID
allowedZones:
  - RESTRICTED
  - PROTECTED
deniedZones:
  - TRUSTED
```

Все изменения правил должны версионироваться и проходить code review.

---

## 9. Слои аналитического хранилища

## 9.1. Landing Zone

Содержит исходные batch-файлы.

Особенности:

- зашифрована;
- короткий TTL;
- доступ только ingestion-service;
- запрещена для BI/ML;
- данные ещё не считаются проверенными.

## 9.2. Quarantine Zone

Содержит:

- неизвестные схемы;
- поля с низким confidence;
- ошибки типов;
- dataset без владельца;
- данные с конфликтующей классификацией;
- batch с нарушенными policy checks.

Особенности:

- изолирована;
- доступ только Data Steward и Security;
- запрещена для аналитики;
- каждый объект имеет причину помещения в quarantine;
- после исправления batch запускается повторно.

## 9.3. Restricted Zone

Хранит данные, которые необходимо сохранить в конфиденциальном виде.

Примеры:

- медицинские записи;
- паспортные данные;
- HR;
- точные финансовые операции.

Меры:

- отдельный encryption key;
- ABAC;
- audit каждого чтения;
- query approval для чувствительных наборов;
- запрет массового экспорта;
- row/column-level security;
- ограниченный retention;
- no direct BI access.

## 9.4. Protected Zone

Хранит:

- очищенные данные;
- нормализованные данные;
- псевдонимизированные идентификаторы;
- токенизированные значения;
- данные без прямых идентификаторов.

Пример:

```text
patient_id → analytics_subject_id
phone → masked or removed
passport_number → token
diagnosis → сохранён только при разрешённой цели
```

## 9.5. Trusted Analytics Zone

Содержит данные, разрешённые для BI, ML и AI:

- агрегированные витрины;
- обезличенные dataset;
- согласованные метрики;
- минимально необходимый набор полей;
- версии, прошедшие quality gate.

Прямой перенос данных из Landing Zone в Trusted Analytics Zone запрещён.

---

## 10. Дополнительные меры защиты

Кроме разделения на зоны, применяются:

- encryption at rest;
- mTLS;
- field-level encryption;
- tokenization;
- pseudonymization;
- anonymization;
- masking;
- RBAC и ABAC;
- Data Lineage;
- retention;
- DLP;
- immutable audit;
- query audit;
- approval workflow;
- row-level security;
- column-level security;
- purpose-based access;
- запрет production PII в test;
- запрет выгрузки секретов в аналитический контур.

---

## 11. Metadata Catalog и Data Lineage

Metadata Catalog хранит:

- источники;
- владельцев;
- classification tags;
- версии схем;
- lineage;
- правила обработки;
- допустимые зоны;
- retention;
- consumer systems;
- историю ручных решений.

Пример lineage:

```text
CRM.patients.phone
    ↓
Landing.patient.phone
    ↓ MASK
Protected.patient.phone_masked
    ↓ DROP
Trusted.patient_statistics
```

Возможный инструмент:

- OpenMetadata.

---

## 12. Ручная проверка

Некоторые данные требуют участия Data Steward.

Причины:

- низкий confidence;
- конфликт правил;
- новое неизвестное поле;
- изменение смысла существующего поля;
- неполное описание источника;
- новое бизнес-назначение dataset.

Решение Data Steward сохраняется как новое правило, чтобы уменьшить количество повторных ручных проверок.

---

## 13. Метрики качества классификации

## 13.1. Precision

```text
Precision =
правильно найденные конфиденциальные поля
/
все поля, классифицированные как конфиденциальные
```

Высокий Precision снижает количество ложных срабатываний и ручной работы.

## 13.2. Recall

```text
Recall =
правильно найденные конфиденциальные поля
/
все реально конфиденциальные поля
```

Recall особенно важен, потому что пропущенное чувствительное поле может попасть в Trusted Zone.

## 13.3. F1-score

```text
F1 =
2 × Precision × Recall
/
(Precision + Recall)
```

Используется для общей оценки качества классификации.

## 13.4. False Negative Rate

```text
FNR =
пропущенные конфиденциальные поля
/
все конфиденциальные поля
```

Это наиболее критичная метрика.

Цель:

```text
FNR для RESTRICTED и SECRET → максимально близко к 0
```

## 13.5. Classification Coverage

```text
Coverage =
классифицированные поля
/
все входные поля
```

Показывает долю данных, для которых принято решение.

## 13.6. Manual Review Rate

```text
Manual Review Rate =
поля, отправленные на ручную проверку
/
все поля
```

Снижение показателя означает улучшение правил и модели.

## 13.7. Quarantine Rate

```text
Quarantine Rate =
batch в quarantine
/
все batch
```

Резкий рост может означать:

- изменение источника;
- ошибку схемы;
- плохое качество данных;
- устаревшие правила.

## 13.8. Schema Drift Detection Rate

Показывает, сколько изменений схемы выявлено до загрузки в хранилище.

## 13.9. Метрики трансформации

- доля замаскированных полей;
- доля токенизированных полей;
- доля удалённых прямых идентификаторов;
- доля dataset, доступных в Trusted Zone;
- число policy violations;
- число несанкционированных попыток доступа.

---

## 14. Метрики производительности

| Метрика | Назначение |
|---|---|
| Batch duration | Время полного pipeline |
| Records per second | Пропускная способность |
| GB per hour | Производительность по объёму |
| Classification latency | Время классификации поля/dataset |
| Queue depth | Накопившийся backlog |
| Failed records | Ошибки обработки |
| Retry count | Стабильность pipeline |
| Data freshness | Задержка появления данных в Trusted Zone |
| Cost per processed GB | Стоимость обработки |
| Worker utilization | Эффективность ресурсов |
| Quarantine resolution time | Скорость устранения проблем |

---

## 15. Как метрики используются для оптимизации

### Если падает Precision

Необходимо:

- уточнить правила;
- улучшить словари;
- добавить контекст;
- убрать слишком широкие regex;
- переобучить модель.

### Если падает Recall

Необходимо:

- расширить набор эталонных данных;
- добавить новые semantic classes;
- повысить строгость fail-closed;
- анализировать новые источники;
- увеличить долю ручной проверки.

### Если растёт Manual Review Rate

Необходимо:

- превращать ручные решения в новые правила;
- улучшать Schema Registry;
- добавлять source-specific policies;
- применять active learning.

### Если растёт Batch Duration

Необходимо:

- увеличить число workers;
- оптимизировать partitioning;
- вынести тяжёлые модели в отдельный pool;
- кэшировать правила;
- использовать columnar formats;
- уменьшить повторное чтение данных.

### Если растёт Quarantine Rate

Необходимо:

- проверить schema drift;
- связаться с владельцем источника;
- валидировать контракты;
- обновить classification rules.

---

## 16. Масштабируемость

## 16.1. Горизонтальное масштабирование

Classification workers должны быть stateless.

Масштабирование выполняется по:

- queue depth;
- batch backlog;
- объёму данных;
- CPU;
- memory;
- времени обработки.

```text
1 worker
    ↓
N workers
    ↓
parallel partitions
```

## 16.2. Partitioning

Данные разбиваются:

- по источнику;
- по дате;
- по филиалу;
- по dataset;
- по partition key.

Пример:

```text
source=medical-system/date=2026-07-05/branch=tomsk
```

## 16.3. Разделение compute и storage

Object storage масштабируется независимо от вычислительных workers.

Это позволяет:

- не хранить state локально;
- повторно запускать batch;
- обрабатывать большие объёмы;
- масштабировать compute только на время загрузки.

## 16.4. Распределённая обработка

Для больших объёмов может использоваться:

- Apache Spark;
- Apache Flink для near-real-time сценариев;
- Kubernetes Jobs;
- autoscaling worker pools.

Для текущего batch-сценария основной вариант:

```text
Object Storage + Orchestrator + Spark/Kubernetes Jobs
```

## 16.5. Независимое масштабирование компонентов

Отдельно масштабируются:

- ingestion;
- schema detection;
- classification;
- transformation;
- metadata catalog;
- audit and monitoring;
- manual review UI.

## 16.6. Масштабирование по пользователям

Рост числа аналитиков не должен увеличивать нагрузку на ingestion.

Для этого:

- BI работает только с Trusted Zone;
- используются read replicas или отдельный query engine;
- вводятся workload groups;
- ограничиваются тяжёлые запросы;
- применяются query quotas;
- используются materialized views;
- выполняется autoscaling query-cluster.

---

## 17. Обработка ошибок

| Ошибка | Реакция |
|---|---|
| Неизвестная схема | Quarantine |
| Низкий confidence | Manual review |
| Нарушение policy | Pipeline stop |
| Ошибка токенизации | Retry, затем quarantine |
| Недоступен Metadata Catalog | Не публиковать данные |
| Недоступен KMS | Остановить обработку |
| Ошибка записи в Trusted Zone | Rollback batch |
| Повторный batch | Idempotent processing |
| Частично обработанный batch | Resume from checkpoint |
| Повреждён файл | Reject + alert |

---

## 18. Идемпотентность и повторная обработка

Каждый batch получает:

```text
batch_id
source_id
schema_version
checksum
processing_version
classification_rule_version
```

Повторная обработка не должна создавать дубли.

Пример ключа:

```text
source_id + batch_id + checksum
```

---

## 19. Аудит

Журналируются:

- загрузка batch;
- версия схемы;
- результат drift detection;
- классификация каждого поля;
- confidence;
- применённые правила;
- трансформации;
- ручные решения;
- публикация в зоны;
- попытки доступа;
- экспорт;
- удаление;
- изменение правил.

Audit Log хранится отдельно и защищён от изменения.

---

## 20. Безопасность самого движка

- service-to-service mTLS;
- short-lived credentials;
- Vault/KMS;
- отдельные service accounts;
- no PII in logs;
- encryption at rest;
- network policies;
- least privilege;
- signed container images;
- dependency scanning;
- audit rule changes;
- separation of duties между Data Engineer, Data Steward и Security.

---

## 21. Рекомендуемый технологический набор

| Компонент | Возможный инструмент |
|---|---|
| Orchestration | Apache Airflow / Argo Workflows |
| Object Storage | S3-compatible storage |
| Distributed Processing | Apache Spark |
| Schema Registry | Confluent Schema Registry / custom registry |
| Metadata Catalog | OpenMetadata |
| Classification Rules | Собственный rules service |
| ML Classification | Python service / model serving |
| Tokenization | Vault Transform / dedicated tokenization service |
| Secrets and Keys | Vault + KMS/HSM |
| Monitoring | Prometheus + Grafana |
| Logs and Audit | OpenSearch / SIEM |
| Policy Enforcement | OPA |
| Data Quality | Great Expectations / custom checks |

---

## 22. Минимальный MVP

В первую версию входят:

1. Batch Orchestrator.
2. Landing Zone.
3. Schema Detector.
4. Schema Drift Detector.
5. Rules-based Classification Engine.
6. Confidence score.
7. Quarantine Zone.
8. Restricted Zone.
9. Protected Zone.
10. Trusted Analytics Zone.
11. Metadata Catalog.
12. Audit и базовые метрики.

ML-классификатор добавляется после накопления размеченного набора данных и статистики ручных решений.

---

## 23. Критерии готовности

Решение считается готовым, если:

- все входные dataset регистрируются;
- все новые поля проходят классификацию;
- неизвестные поля не попадают в Trusted Zone;
- schema drift выявляется до публикации;
- `RESTRICTED` и `SECRET` поля не доступны аналитикам напрямую;
- все transformations сохраняются в Data Lineage;
- pipeline идемпотентен;
- manual review поддерживается;
- метрики качества доступны;
- система горизонтально масштабируется;
- batch можно повторно обработать;
- audit содержит полную историю принятого решения.

---

## 24. Итог

Предлагаемый движок обеспечивает классификацию и защиту данных до их попадания в аналитическое хранилище.

Ключевые свойства решения:

- fail-closed;
- hybrid classification;
- schema drift detection;
- confidence-based routing;
- отдельные зоны хранения;
- controlled access;
- Data Lineage;
- автоматизированные метрики;
- горизонтальное масштабирование;
- безопасная работа BI, ML и AI с конфиденциальными данными.
