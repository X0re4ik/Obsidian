

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

5. Для определения актуальной загрузки значение `batch_id` записывается
   в поле `ca_proposal_file_id`.

6. Сервис `iuch-metric-aggr` обнуляет значения всех предыдущих загрузок,
   для которых `ca_proposal_file_id != batch_id`.
   Обнуляются следующие поля:
   - `ca_proposal`;
   - `const_ca_proposal`;
   - `tmp_ca_proposal`.

7. Сервис `iuch-metric-aggr` записывает значения из исходных Avro-данных
   без модификации:
   - `ca_proposal` — значение поля `ca_proposal`;
   - `const_ca_proposal` — значение поля `const_ca_proposal`;
   - `tmp_ca_proposal` — значение поля `tmp_ca_proposal`.

   Значения должны соответствовать формуле:

   `ca_proposal = const_ca_proposal + tmp_ca_proposal`


**Change**

Изменение avro-схемы (контракт между **iuch-etl** и **metric-aggr**)

**Было:**
```
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

class CAProposalAvro(BaseModelAvro):
    """
    Предложение ЦА
    """

    urf_code: str
    base_pos_id: int
    ftu: float
    channel: str
    ca_proposal_file_id: int


class CAProposalBatchAvro(BatchBaseModelAvro[CAProposalAvro]):
    """
    Предложение ЦА в формате батчей
    """
```

