# Задание 6. Движок классификации данных

## Цель

Спроектировать слой классификации данных перед их пакетной загрузкой в аналитическое хранилище с учётом принципов Privacy by Design.

Решение должно:

- принимать данные из операционных систем;
- обнаруживать структурные изменения;
- классифицировать неразмеченные поля;
- применять защитные преобразования;
- изолировать неизвестные и рискованные данные;
- безопасно публиковать данные в аналитические слои;
- поддерживать масштабирование;
- предоставлять метрики качества классификации.

## Состав решения

- `data-classification-engine.md` — подробное описание архитектуры, алгоритма классификации, слоёв хранения, метрик и масштабирования.
- `diagrams/c4-data-classification-engine-c2.drawio` — редактируемая C4 Container Diagram уровня C2.
- `diagrams/c4-data-classification-engine-c2.png` — изображение диаграммы для быстрого просмотра.

## Основной поток данных

```text
Операционные системы
        ↓
Batch Orchestrator
        ↓
Landing Zone
        ↓
Schema Detector
        ↓
Schema Drift Detector
        ↓
Classification Engine
        ↓
Privacy Rules Service
        ↓
Privacy Transformation Service
        ↓
Quarantine / Restricted / Protected / Trusted Zones
        ↓
BI / ML / AI
```

## Ключевые контейнеры

| Контейнер | Назначение |
|---|---|
| Batch Orchestrator | Управляет пакетными загрузками, retry и checkpoint |
| Landing Zone | Принимает исходные данные в зашифрованном виде |
| Schema Detector | Определяет структуру и статистический профиль данных |
| Schema Registry | Хранит версии схем |
| Schema Drift Detector | Обнаруживает новые, удалённые и изменённые поля |
| Classification Engine | Назначает класс, semantic tag и confidence score |
| Privacy Rules Service | Хранит правила обработки и допустимые зоны |
| Privacy Transformation Service | Маскирование, токенизация, псевдонимизация и шифрование |
| Metadata Catalog & Lineage | Хранит теги, владельцев и происхождение данных |
| Audit & Monitoring | Метрики, аудит и алерты |
| Quarantine Zone | Неизвестные схемы и низкий confidence |
| Restricted Zone | Конфиденциальные данные со строгим доступом |
| Protected Zone | Очищенные и псевдонимизированные данные |
| Trusted Analytics Zone | Обезличенные и агрегированные витрины |

## Принцип классификации

Используется гибридный подход:

1. Детерминированные правила.
2. Анализ значений.
3. Контекстный анализ.
4. ML-классификатор.
5. Policy resolution.
6. Confidence-based routing.

При конфликте выбирается более строгий класс.

## Fail-closed

```text
Неизвестное или неклассифицированное поле
не может попасть в Trusted Analytics Zone.
```

Такие данные направляются в `Quarantine Zone` или остаются в `Restricted Zone` до ручной проверки.

## Слои хранения

### Landing Zone

- исходные данные;
- шифрование;
- короткий TTL;
- запрет доступа аналитиков.

### Quarantine Zone

- неизвестные схемы;
- низкий confidence;
- конфликтующие правила;
- ручная проверка.

### Restricted Zone

- медицинские, паспортные и финансовые данные;
- ABAC;
- field-level encryption;
- аудит каждого чтения.

### Protected Zone

- очищенные;
- нормализованные;
- токенизированные;
- псевдонимизированные данные.

### Trusted Analytics Zone

- обезличенные витрины;
- агрегированные наборы;
- разрешённые BI/ML/AI dataset.

## Метрики качества классификации

- Precision;
- Recall;
- F1-score;
- False Negative Rate;
- Classification Coverage;
- Manual Review Rate;
- Quarantine Rate;
- Schema Drift Detection Rate.

Наиболее критичная метрика:

```text
False Negative Rate
```

Она показывает долю конфиденциальных полей, которые движок не распознал.

## Метрики производительности

- batch duration;
- records per second;
- GB per hour;
- classification latency;
- queue depth;
- retry count;
- data freshness;
- cost per processed GB;
- quarantine resolution time.

## Масштабируемость

Решение масштабируется за счёт:

- stateless classification workers;
- object storage;
- partitioning по источнику, дате и dataset;
- Apache Spark или Kubernetes Jobs;
- autoscaling по backlog и batch duration;
- независимого масштабирования ingestion, classification и query layers;
- разделения compute и storage;
- отдельного query-cluster для BI и ML.

## Безопасность

- mTLS;
- IAM;
- Vault/KMS;
- short-lived credentials;
- encryption at rest;
- ABAC;
- DLP;
- immutable audit;
- Data Lineage;
- no PII in logs;
- запрет прямого доступа к Landing Zone.

## Результат

Подготовлены:

- проект движка классификации;
- обработка schema drift;
- модель confidence-based routing;
- специальные слои аналитического хранилища;
- метрики качества и производительности;
- стратегия масштабирования;
- C4 Container Diagram уровня C2.
