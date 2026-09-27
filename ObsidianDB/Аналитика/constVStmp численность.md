

**Задача:** Разработать функционал постоянного/временного перевода ставок на период проведения итерации

# Изменение "Предложение ЦА"

**Задача:** 
Добавить 3 колонки:
* Предложение ЦА (`ca_proposal`)
* Постоянное предложение ЦА (`const_ca_proposal`)
* Временное предложение ЦА (`tmp_ca_proposal`)

`ca_proposal = const_ca_proposal + tmp_ca_proposal`

Процесс загрузки Предложение ЦА остается таким же, изменяется лишь исходный файл, который грузит пользователь. Пример файла **ДО** и **ПОСЛЕ** представлен в **таблице 1.1.**

**Таблица 1.1**

| ДО  | ПОСЛЕ |
| --- | ----- |
|     |       |

**Алгоритм работы процесса:**

1. Пользователь загружает новый файл «Предложение ЦА».

2. Сервис `iuch-etl`:
   - сохраняет данные файла в таблицу `etl_ca_proposal.initial_data`;
   - обновляет данные в таблице `etl_model.vsp_ca_proposal`;
   - присваивает данным идентификатор текущей загрузки `batch_id`.

3. Сервис `iuch-etl` запускает pipeline передачи данных в `Kafka`.
   Данные передаются батчами примерно по 10 000 записей.

4. Сервис `iuch-metric-aggr` получает данные из `Kafka` и обрабатывает их
   в таблице `metric_aggr.aggregated_data`.

5. Сервис `iuch-metric-aggr` обнуляет значения всех предыдущих загрузок,
   для которых `ca_proposal_file_id != avro.ca_proposal_file_id`.
   Обнуляются следующие поля:
   - `ca_proposal`;
   - `const_ca_proposal`;
   - `tmp_ca_proposal`.

6. Сервис `iuch-metric-aggr` записывает значения из исходных Avro-данных
   без модификации:
   - `ca_proposal` — значение поля `ca_proposal`;
   - `const_ca_proposal` — значение поля `const_ca_proposal`;
   - `tmp_ca_proposal` — значение поля `tmp_ca_proposal`.

   Значения должны соответствовать формуле:

   `ca_proposal = const_ca_proposal + tmp_ca_proposal`


## Изменения в техническом контракте и структуре данных

В рамках задачи изменяются Avro-контракт между сервисами `iuch-etl` и
`iuch-metric-aggr`, а также таблицы, через которые передаются и хранятся
данные «Предложения ЦА».

### Соответствие бизнес-полей и технических полей

| Бизнес-поле | Поле Avro | Колонки БД |
|---|---|---|
| Предложение ЦА | `total_ftu` | `total_ftu` |
| Постоянное предложение ЦА | `const_ftu` | `const_ftu` |
| Временное предложение ЦА | `tmp_ftu` | `tmp_ftu` |

Итоговое значение рассчитывается по формуле:

```text
total_ftu = const_ftu + tmp_ftu
```

В итоговой таблице `etl_model.vsp_ca_proposals` поле `ftu` переименовывается
в `total_ftu`. Переименование связано с изменением смысла поля: теперь оно
хранит итоговое значение, рассчитанное как сумма постоянной и временной частей
предложения ЦА.

### Изменение Avro-схемы

Avro-схема является контрактом между сервисами `iuch-etl` и
`iuch-metric-aggr`.

```python
T = TypeVar("T", bound="BaseModelAvro")

class BatchBaseModelAvro(AvroBaseModel, Generic[T]):
    id: int  # noqa: A003
    items: list[T]


class BaseModelAvro(AvroBaseModel):
    id: int  # noqa: A003
    schema_version: str
    is_deleted: bool = False
    created_at: datetime
    updated_at: datetime

    class Config:
        json_encoders = {
            datetime: lambda v: int(v.timestamp() * 1000),
        }

# Было:
class CAProposalAvro(BaseModelAvro):
    """
    Предложение ЦА
    """

    urf_code: str
    base_pos_id: int
    ftu: float
    channel: str
    ca_proposal_file_id: int

# Стало:
class CAProposalAvro(BaseModelAvro):
    """
    Предложение ЦА
    """

    urf_code: str
    base_pos_id: int
    total_ftu: float # Общая сумма Предложение ЦА (tmp_ftu + const_ftu)
    tmp_ftu: float # Временное Предложение ЦА
    const_ftu: float # Постоянное Предложение ЦА
    channel: str
    ca_proposal_file_id: int


class CAProposalBatchAvro(BatchBaseModelAvro[CAProposalAvro]):
    """
    Предложение ЦА в формате батчей
    """
```

#### Изменение таблицы `etl_ca_proposal.initial_data`

> DDL приведён для демонстрации состава и типов полей.

**Было**
```sql
CREATE TABLE etl_ca_proposal.initial_data (
    id                      INT8 PRIMARY KEY,
    ca_proposal_file_id     BIGINT       NOT NULL,
    -- ВНИМАНИЕ: ftu - legacy-поле, поэтому оно VARCHAR
    ftu                     VARCHAR  NOT NULL,
    tb_code                 VARCHAR  NOT NULL,
    gosb_code               VARCHAR  NOT NULL,
    vsp_code                VARCHAR  NOT NULL,
    etalon_pos_id           VARCHAR NOT NULL,
    urf_code                VARCHAR NOT NULL,
);
```

**Стало**
```sql
CREATE TABLE etl_ca_proposal.initial_data (
    id                      INT8 PRIMARY KEY,
    ca_proposal_file_id     BIGINT       NOT NULL,
    -- Поле не заполняется; сохраняется для доступа к предыдущим значениям
    ftu                     VARCHAR  NULL,
    const_ftu               NUMERIC  NOT NULL, -- Постоянное предложение ЦА
    tmp_ftu                 NUMERIC  NOT NULL, -- Временное предложение ЦА
    tb_code                 VARCHAR  NOT NULL,
    gosb_code               VARCHAR  NOT NULL,
    vsp_code                VARCHAR  NOT NULL,
    etalon_pos_id           VARCHAR NOT NULL,
    urf_code                VARCHAR NOT NULL,
);
```

#### Изменение таблицы `etl_model.vsp_ca_proposals`

**Было**
```sql
CREATE TABLE etl_model.vsp_ca_proposals (
    id                    BIGSERIAL    PRIMARY KEY,
    urf_code              VARCHAR(255) NOT NULL,
    base_pos_id           BIGINT       NOT NULL,
    ftu                   NUMERIC(10,3) NOT NULL,
    channel               VARCHAR(255) NOT NULL,
    ca_proposal_file_id   BIGINT       NOT NULL,
    
    ...
    
    CONSTRAINT uq_vsp_ca_proposals_urf_code_base_pos_id
        UNIQUE (urf_code, base_pos_id)
);
```

**Стало**
```sql
CREATE TABLE vsp_ca_proposals (
    id                    BIGSERIAL    PRIMARY KEY,
    urf_code              VARCHAR(255) NOT NULL,
    base_pos_id           BIGINT       NOT NULL,
    
    total_ftu             NUMERIC(10,3) NOT NULL, -- total_ftu = const_ftu + tmp_ftu
    const_ftu             NUMERIC(10,3) NOT NULL, -- Постоянное предложение ЦА
    tmp_ftu               NUMERIC(10,3) NOT NULL, -- Временное предложение ЦА
    channel               VARCHAR(255) NOT NULL,
    ca_proposal_file_id   BIGINT       NOT NULL,

    ....

    CONSTRAINT uq_vsp_ca_proposals_urf_code_base_pos_id
        UNIQUE (urf_code, base_pos_id)
);
```


# Изменение "Предложение ТБ"

**Задача:**
Добавить три поля:

* итоговое предложение ТБ (`total_ftu`);
* постоянное предложение ТБ (`const_ftu`);
* временное предложение ТБ (`tmp_ftu`).

В начале итерации временное предложение отсутствует, поэтому:

```text
tmp_ftu = 0
total_ftu = const_ftu
```

Во время итерации в ВСП могут временно входить и выходить ставки. Поэтому
`tmp_ftu` хранит не количество переводов, а их суммарное изменение в ПШЕ.

Правила расчёта временного изменения:

* для ставки, временно вошедшей в ВСП, её ПШЕ учитываются со знаком «плюс»;
* для ставки, временно вышедшей из ВСП, её ПШЕ учитываются со знаком «минус».

Таким образом, `tmp_ftu` рассчитывается как чистое сальдо временных переводов:

```text
tmp_ftu = сумма ПШЕ вошедших ставок - сумма ПШЕ вышедших ставок
```

`total_ftu` рассчитывается как сумма ПШЕ всех актуальных ставок, находящихся
в ВСП после применения временных переводов:

```text
total_ftu = сумма ПШЕ всех актуальных ставок ВСП
```

`const_ftu` определяется как часть итогового предложения, не относящаяся
к временным переводам:

```text
const_ftu = total_ftu - tmp_ftu
```

Итоговая проверка выполняется по формуле:

```text
total_ftu = const_ftu + tmp_ftu
```

Например, если из ВСП временно вышла ставка с параметром 1 ПШЕ,
то `tmp_ftu = -1`. При `total_ftu = 3` постоянная часть рассчитывается так:

```text
const_ftu = 3 - (-1) = 4
```

Если в ВСП вошли ставки на 2 ПШЕ, а вышла ставка на 1 ПШЕ,
то чистое временное изменение составит:

```text
tmp_ftu = 2 - 1 = 1
```
Если в ВСП вошли две новыые ставки, то tmp_ftu = 2, const_ftu = 4, total_ftu = 6


Алгоритм расчёта const_ftu и tmp_ftu внтри сервиса: расчитать total_ftu как обычное (сумма всех ставок внутри ВСП), найти все временные ставки ВОШЕДШИЕ В ВСП = tmp_ftu, тогда const_ftu = total_ftu - tmp_ftu, найти все временные ставки, которые вышли из ВСП = -tmp_ftu, тогда const_ftu = total_ftu - (-tmp_ftu) =  const_ftu = total_ftu + tmp_ftu


**Change**

Изменение в avro схеме

```python

from typing import Literal

from dataclasses_avroschema.pydantic import AvroBaseModel

# Было:
class MyProposalByPosAvro(AvroBaseModel):
    """
    Мое предложение по основной эталонной должности
    """

    id: int  # noqa: A003
    urf_code: str  # Идентификатор ВСП
    et_main_pos_id: int  # Идентификатор эталонных должностей
    ftu: float  # Общее число ПШЕ
    diff_ftu: float  # (legacy) Разница со занчением на момент старта итераци

# Стало:
class MyProposalByPosAvro(AvroBaseModel):
    """
    Мое предложение по основной эталонной должности
    """

    id: int  # noqa: A003
    urf_code: str  # Идентификатор ВСП
    et_main_pos_id: int  # Идентификатор эталонных должностей
    total_ftu: float  # Общее число ПШЕ (tmp_ftu + const_ftu)
    const_ftu: float  # Постоянное число ПШЕ
    tmp_ftu: float  # Временное число ПШЕ

```

Добавить таблицу с фиксацией временного перевода ставки

```sql
CREATE TABLE tmp_employee_position_transfer (
    id                    BIGSERIAL    PRIMARY KEY,
    proposal_id INT4, -- Идентификатор предложения
    from_vsp_id INT4, -- ВСП откуда времено переевли ставку
    to_vsp_id INT4, -- ВСП куда временное перевели ставку
    employee_position_id INT4 -- Идентификатор ставки, которую переевли
    ....

    CONSTRAINT uq_vsp_ca_proposals_urf_code_base_pos_id
        UNIQUE (urf_code, base_pos_id)
);
```
