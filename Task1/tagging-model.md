# Модель классификации и тегирования данных

## 1. Цель

Модель тегирования предназначена для автоматического определения конфиденциальных данных, применения правил доступа и контроля их перемещения между системами компании «Медикаменте».

Тег должен сопровождать набор данных на всём жизненном цикле:

```text
Сбор → Обработка → Передача → Хранение → Аналитика → Архивирование → Удаление
```

Для централизованного хранения метаданных предлагается использовать каталог метаданных **OpenMetadata**. В нём регистрируются источники данных, владельцы, классификации, теги, сроки хранения и связи Data Lineage.

## 2. Уровни конфиденциальности

Каждый объект данных должен иметь один обязательный тег уровня конфиденциальности.

| Тег | Описание | Примеры |
|---|---|---|
| `CLASS.PUBLIC` | Открытые данные | Адрес филиала, перечень услуг |
| `CLASS.INTERNAL` | Внутренняя информация | Регламенты, справочники |
| `CLASS.CONFIDENTIAL` | Персональные, финансовые и договорные данные | Телефон, email, договор, платёж |
| `CLASS.RESTRICTED` | Медицинские и критичные кадровые данные | Диагноз, анализы, медицинская карта |

Если объект содержит данные разных уровней, ему присваивается наиболее строгий уровень.

## 3. Категории данных

| Категория | Теги |
|---|---|
| Идентификационные данные | `PII.IDENTITY.FULL_NAME`, `PII.IDENTITY.BIRTH_DATE`, `PII.IDENTITY.DOCUMENT` |
| Контактные данные | `PII.CONTACT.PHONE`, `PII.CONTACT.EMAIL`, `PII.CONTACT.ADDRESS` |
| Медицинские данные | `SENSITIVE.HEALTH.DIAGNOSIS`, `SENSITIVE.HEALTH.ANAMNESIS`, `SENSITIVE.HEALTH.LAB_RESULT`, `SENSITIVE.HEALTH.PRESCRIPTION` |
| Финансовые данные | `SENSITIVE.FINANCE.PAYMENT`, `SENSITIVE.FINANCE.RECEIPT`, `SENSITIVE.FINANCE.SALARY` |
| Кадровые данные | `SENSITIVE.HR.EMPLOYEE`, `SENSITIVE.HR.PAYROLL`, `SENSITIVE.HR.TAX` |
| Договорные данные | `LEGAL.CONTRACT`, `LEGAL.CONSENT`, `LEGAL.SIGNATURE` |
| Технические данные | `TECHNICAL.AUDIT`, `TECHNICAL.SECRET`, `TECHNICAL.TOKEN` |

## 4. Дополнительные обязательные теги

Кроме категории и уровня конфиденциальности объект должен содержать управляющие теги.

| Группа | Примеры | Назначение |
|---|---|---|
| Владелец | `OWNER.MEDICAL`, `OWNER.RECEPTION`, `OWNER.ACCOUNTING` | Ответственный за данные |
| Цель обработки | `PURPOSE.TREATMENT`, `PURPOSE.APPOINTMENT`, `PURPOSE.PAYMENT`, `PURPOSE.ANALYTICS` | Допустимая цель использования |
| Срок хранения | `RETENTION.1Y`, `RETENTION.5Y`, `RETENTION.LEGAL` | Политика архивирования и удаления |
| Территория хранения | `LOCATION.RU` | Ограничение размещения данных |
| Защита | `PROTECTION.ENCRYPT`, `PROTECTION.MASK`, `PROTECTION.PSEUDONYMIZE`, `PROTECTION.ANONYMIZE` | Обязательные методы защиты |
| Ограничение экспорта | `EXPORT.DENY`, `EXPORT.APPROVAL_REQUIRED` | Контроль выгрузок |
| Аудит | `AUDIT.READ`, `AUDIT.WRITE`, `AUDIT.EXPORT`, `AUDIT.DELETE` | Журналируемые действия |

## 5. Формат метаданных

Пример карточки медицинского результата:

```yaml
asset:
  name: laboratory_result
  system: medical-records
  owner: medical-department

classification:
  level: RESTRICTED
  tags:
    - SENSITIVE.HEALTH.LAB_RESULT
    - PII.IDENTITY.BIRTH_DATE

processing:
  purpose:
    - TREATMENT
  retention: LEGAL
  location: RU

protection:
  encryption_at_rest: true
  encryption_in_transit: true
  masking: true
  pseudonymization: true
  anonymization_for_analytics: true

access:
  model: ABAC
  policy: doctor_of_patient_or_medical_admin

audit:
  - READ
  - WRITE
  - EXPORT
  - DELETE

export:
  policy: APPROVAL_REQUIRED
```

## 6. Механизм присвоения тегов

### 6.1. Автоматическое тегирование

Автоматический классификатор анализирует:

- названия таблиц, колонок и файлов;
- типы данных;
- регулярные выражения;
- словари терминов;
- содержимое документов;
- источник данных и бизнес-контекст.

Примеры правил:

```text
column_name in ("phone", "mobile", "telephone")
    → PII.CONTACT.PHONE

column_name in ("diagnosis", "anamnesis", "chronic_disease")
    → SENSITIVE.HEALTH.DIAGNOSIS
    → CLASS.RESTRICTED

file_path contains "/patients/"
    → OWNER.MEDICAL
    → EXPORT.APPROVAL_REQUIRED
```

Для неструктурированных PDF, JPG и Excel-файлов классификация выполняется при загрузке в контролируемое хранилище. До миграции рядом с файлом допускается sidecar-файл метаданных:

```text
conclusion.pdf
conclusion.pdf.metadata.yaml
```

### 6.2. Ручная валидация

Автоматически назначенные теги подтверждает владелец данных или Data Steward.

Ручная проверка обязательна для:

- медицинских данных;
- паспортных реквизитов;
- новых интеграций;
- наборов для BI, ML и AI;
- данных с неопределённой категорией.

### 6.3. Наследование тегов

Теги наследуются по цепочке Data Lineage.

Пример:

```text
medical_record.diagnosis
    ↓
laboratory_export.patient_diagnosis
    ↓
analytics_raw.diagnosis
```

Если поле-источник имеет тег `CLASS.RESTRICTED`, производные наборы сохраняют этот уровень до подтверждённого обезличивания.

## 7. Применение тегов в политиках доступа

Теги должны использоваться не только для описания, но и для автоматического применения политик.

| Условие | Действие |
|---|---|
| `CLASS.RESTRICTED` | Обязательное шифрование и ABAC |
| `SENSITIVE.HEALTH.*` | Доступ только медицинскому персоналу, связанному с пациентом |
| `PII.CONTACT.*` | Маскирование для пользователей без расширенных прав |
| `PURPOSE.ANALYTICS` | Использование только обезличенных данных |
| `EXPORT.DENY` | Запрет выгрузки |
| `EXPORT.APPROVAL_REQUIRED` | Выгрузка после согласования и с записью в аудит |
| `RETENTION.*` | Автоматическое архивирование или удаление |
| `AUDIT.*` | Регистрация соответствующих операций |

Пример правила:

```text
DENY READ
IF data.tag = "SENSITIVE.HEALTH.LAB_RESULT"
AND user.role != "DOCTOR"
AND user.role != "MEDICAL_ADMIN"
```

## 8. Использование тегов в CI/CD

При разработке новых сервисов контракты и схемы данных должны проверяться автоматически.

Проверки выполняются для:

- OpenAPI-контрактов;
- DTO;
- схем баз данных;
- Kafka-событий;
- SQL-миграций;
- аналитических витрин.

Пример аннотации поля:

```yaml
patientPhone:
  type: string
  x-data-classification:
    level: CONFIDENTIAL
    tags:
      - PII.CONTACT.PHONE
    protection:
      - MASK
      - ENCRYPT
```

Pipeline должен завершаться ошибкой, если:

- поле содержит конфиденциальные данные, но не имеет тега;
- `RESTRICTED`-данные передаются без защищённого канала;
- внешний API возвращает внутренний идентификатор пациента;
- аналитическая витрина содержит PII без обезличивания;
- данные выгружаются без политики аудита и срока хранения.

## 9. Ответственность

| Роль | Ответственность |
|---|---|
| Data Owner | Определяет цель обработки и допустимый доступ |
| Data Steward | Подтверждает классификацию и качество тегов |
| Security Officer | Определяет обязательные меры защиты |
| Разработчик | Указывает теги в контрактах и схемах |
| DevOps | Подключает проверки в CI/CD и инфраструктуре |
| Аналитик | Использует только разрешённые и обезличенные наборы |
| Аудитор | Проверяет полноту тегирования и соблюдение политик |

## 10. Контроль качества тегирования

Необходимо регулярно проверять:

- долю активов без классификации;
- долю полей PII без владельца;
- количество конфликтующих тегов;
- наличие `RESTRICTED`-данных в тестовых и аналитических средах;
- выгрузки с тегом `EXPORT.DENY`;
- данные с истёкшим сроком хранения;
- разрывы в Data Lineage.

Целевой показатель:

```text
100% объектов с CONFIDENTIAL и RESTRICTED данными
имеют владельца, цель обработки, срок хранения
и обязательный набор тегов защиты.
```

## 11. Отображение на DFD

На защищённых диаграммах потоки маркируются следующим образом:

```text
[CONFIDENTIAL][PII.CONTACT][TLS][AUDIT]

[RESTRICTED][SENSITIVE.HEALTH]
[mTLS][PSEUDONYMIZE][ABAC][AUDIT]

[ANONYMIZED][PURPOSE.ANALYTICS]
```

Возле хранилищ указываются:

```text
Encryption at Rest
Retention Policy
Access Control
Audit
Metadata Catalog
```

## 12. Вывод

Тегирование должно быть частью архитектуры, а не отдельным реестром.

Каталог метаданных хранит описание и Data Lineage, автоматические сканеры назначают первичные теги, владельцы данных подтверждают классификацию, а политики доступа, CI/CD, DLP и аудит используют эти теги для автоматического применения мер защиты.
