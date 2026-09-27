

**Задача:** Разработать функционал постоянного/временного перевода ставок на период проведения итерации

## Изменение предложение ЦА

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


**Change**

#### Изменение avro-схемы (контракт между **iuch-etl** и **metric-aggr**)

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
    total_ftu: float
    tmp_ftu: float
    const_ftu: float
    channel: str
    ca_proposal_file_id: int


class CAProposalBatchAvro(BatchBaseModelAvro[CAProposalAvro]):
    """
    Предложение ЦА в формате батчей
    """
```

#### Изменение таблицы `etl_ca_proposal.initial_data`

>DDL является демонстрационным

**Было**
```sql
CREATE TABLE etl_ca_proposal.initial_data (
    id                      INT8 PRIMARY KEY,
    ca_proposal_file_id     BIGINT       NOT NULL,
    // ВНИМАНИЕ ftu VARCHAR - LEGACY
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
    ftu                     VARCHAR  NULL, // ПОЛЕ НЕ ЗАПОЛНЯЕТСЯ
    const_ftu               NUMERIC  NOT NULL, // Постоянное "Предложение ЦА"
    tmp_ftu                 NUMERIC  NOT NULL, // Временное  "Предложение ЦА"
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
CREATE TABLE vsp_ca_proposals (
    id                    BIGSERIAL    PRIMARY KEY,
    urf_code              VARCHAR(255) NOT NULL,
    base_pos_id           BIGINT       NOT NULL,
    ftu                   NUMERIC(10,3) NOT NULL,
    channel               VARCHAR(255) NOT NULL,
    ca_proposal_file_id   BIGINT       NOT NULL,
    -- если ETLModelModel добавляет timestamps:
    created_at            TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at            TIMESTAMPTZ  NOT NULL DEFAULT now(),
    CONSTRAINT uq_vsp_ca_proposals_urf_code_base_pos_id
        UNIQUE (urf_code, base_pos_id)
);
```

Стало
```
CREATE TABLE vsp_ca_proposals (
    id                    BIGSERIAL    PRIMARY KEY,
    urf_code              VARCHAR(255) NOT NULL,
    base_pos_id           BIGINT       NOT NULL,
    ftu                   NUMERIC(10,3) NOT NULL,
    const_ftu              NUMERIC  NOT NULL, // Постоянное "Предложение ЦА"
    tmp_ftu                 NUMERIC  NOT NULL, // Временное  "Предложение ЦА"
    channel               VARCHAR(255) NOT NULL,
    ca_proposal_file_id   BIGINT       NOT NULL,
    -- если ETLModelModel добавляет timestamps:
    created_at            TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at            TIMESTAMPTZ  NOT NULL DEFAULT now(),
    CONSTRAINT uq_vsp_ca_proposals_urf_code_base_pos_id
        UNIQUE (urf_code, base_pos_id)
);
```