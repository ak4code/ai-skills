---
name: django-pydantic-dto
description: Используй этот навык для проектирования и ревью кода в Django-проектах, где Pydantic v2 применяется как DTO для межслойного и межсервисного взаимодействия. Покрывает изоляцию слоёв, входные и выходные DTO, model_validate, model_dump, TypeAdapter, JSON-контракты, Celery-задачи, RabbitMQ-сообщения, HTTP-клиенты и интеграцию с сервисами. Подключай при словах "создай DTO", "контракт между сервисами", "Pydantic DTO", "TypeAdapter", "валидация payload", "межсервисное взаимодействие", "валидация структуры", "слоистая архитектура в Django".
---

# Pydantic DTO в Django

Этот навык определяет стандарт использования Pydantic v2 как Data Transfer Object в Django-проектах.

DTO нужны не для замены Django Models или DRF serializers, а для явного описания структуры данных, которые проходят между слоями приложения и между сервисами.

Основные сценарии:

- передача данных из API-слоя в сервис;
- возврат результата из сервиса;
- формирование payload для другого сервиса;
- валидация ответа внешнего сервиса;
- передача данных в Celery-задачи;
- публикация и чтение сообщений из RabbitMQ;
- фиксация контрактов между backend-сервисами.

## Базовые принципы

1. DTO это только данные.

   В Pydantic-моделях не должно быть сохранения в БД, отправки писем, HTTP-запросов, публикации сообщений или другой бизнес-логики.

2. DTO фиксирует контракт.

   Если один сервис отправляет другому JSON, структура этого JSON должна быть описана через DTO. Это делает контракт явным, типизированным и проверяемым тестами.

3. Валидация выполняется на границах.

   Pydantic проверяет форму данных: типы, обязательные поля, формат email, UUID, datetime, вложенные структуры. Бизнес-валидация остаётся в сервисах.

4. Сервисы принимают DTO, а не сырые dict.

   Сервисный слой не должен разбирать произвольные словари. На вход сервиса должны попадать уже провалидированные данные.

5. Используется только Pydantic v2 API.

   Используй `model_validate`, `model_dump`, `model_validate_json`, `ConfigDict`, `TypeAdapter`. Не используй `parse_obj`, `from_orm`, `dict`, `json` из Pydantic v1.

6. DTO не должен зависеть от Django.

   DTO не импортирует модели, QuerySet, request, response, settings и другие инфраструктурные объекты. DTO можно безопасно использовать в API, Celery, HTTP-клиентах и тестах.

## Где хранить DTO

DTO описываются в `dtos.py` внутри приложения, к которому относится контракт.

Пример структуры:

```text
apps/
  orders/
    dtos.py
    services.py
    selectors.py
    api.py
    tasks.py
    clients.py
```

Если DTO относится к интеграции с внешним сервисом, допустимо выделять отдельный модуль:

```text
apps/
  integrations/
    some_service/
      dtos.py
      client.py
      services.py
```

## Базовый пример DTO

```python
from datetime import datetime
from uuid import UUID

from pydantic import BaseModel, ConfigDict, EmailStr, Field


class UserDTO(BaseModel):
    model_config = ConfigDict(extra='forbid')

    id: UUID
    email: EmailStr
    full_name: str = Field(..., max_length=255)
    is_active: bool
    created_at: datetime
```

`extra='forbid'` запрещает лишние поля. Для межсервисных контрактов это полезно, потому что неожиданные поля часто означают рассинхрон версий контракта.

## DTO для входа в сервис

Входной DTO описывает команду или действие.

```python
from uuid import UUID

from pydantic import BaseModel, ConfigDict, Field


class CreateOrderDTO(BaseModel):
    model_config = ConfigDict(extra='forbid')

    user_id: UUID
    product_id: UUID
    quantity: int = Field(..., ge=1)
```

Сервис принимает DTO:

```python
from apps.orders.dtos import CreateOrderDTO, OrderDTO


def order_create(dto: CreateOrderDTO) -> OrderDTO:
    ...
```

Не передавай в сервис `request.data`, `serializer.validated_data` или произвольный `dict`, если структура данных уже является частью бизнес-контракта.

## DTO для результата сервиса

Выходной DTO описывает результат операции.

```python
from datetime import datetime
from uuid import UUID

from pydantic import BaseModel, ConfigDict


class OrderDTO(BaseModel):
    model_config = ConfigDict(extra='forbid')

    id: UUID
    user_id: UUID
    status: str
    total_price: str
    created_at: datetime
```

Сервис возвращает DTO, а API-слой сам решает, как превратить его в HTTP response.

```python
result = order_create(dto)
return Response(result.model_dump(mode='json'), status=201)
```

## Интеграция с DRF

DRF может использоваться для HTTP-уровня: permissions, authentication, parsers, renderers, status codes.

Pydantic DTO используется для передачи данных в сервисный слой.

```python
from pydantic import ValidationError
from rest_framework.response import Response
from rest_framework.views import APIView

from apps.orders.dtos import CreateOrderDTO
from apps.orders.services import order_create


class OrderCreateAPIView(APIView):
    def post(self, request):
        try:
            dto = CreateOrderDTO.model_validate(request.data)
        except ValidationError as error:
            return Response(error.errors(), status=400)

        result = order_create(dto)

        return Response(result.model_dump(mode='json'), status=201)
```

Правило: view отвечает за HTTP, сервис отвечает за бизнес-логику, DTO отвечает за форму данных.

## Межсервисное взаимодействие через HTTP

Для исходящего запроса используется отдельный DTO запроса.

Для входящего ответа используется отдельный DTO ответа.

```python
from uuid import UUID

from pydantic import BaseModel, ConfigDict, Field


class ContractStatusRequestDTO(BaseModel):
    model_config = ConfigDict(extra='forbid')

    contract_id: UUID
    organization_id: UUID


class ContractStatusResponseDTO(BaseModel):
    model_config = ConfigDict(extra='forbid')

    contract_id: UUID
    status: str
    edo_status: str | None = None
```

HTTP-клиент формирует payload через `model_dump(mode='json')` и валидирует ответ через `model_validate`.

```python
import httpx

from apps.contracts.dtos import (
    ContractStatusRequestDTO,
    ContractStatusResponseDTO,
)


class ContractClient:
    def __init__(self, base_url: str, timeout: float) -> None:
        self.base_url = base_url
        self.timeout = timeout

    def get_status(self, dto: ContractStatusRequestDTO) -> ContractStatusResponseDTO:
        response = httpx.post(
            f'{self.base_url}/api/contracts/status/',
            json=dto.model_dump(mode='json'),
            timeout=self.timeout,
        )
        response.raise_for_status()

        return ContractStatusResponseDTO.model_validate(response.json())
```

Такой подход защищает сервис от неявного изменения внешнего API. Если внешний сервис начал отдавать другую структуру, ошибка будет поймана на границе интеграции.

## Межсервисное взаимодействие через RabbitMQ

Сообщение в брокере тоже является контрактом.

Для каждого события должен быть отдельный DTO.

```python
from datetime import datetime
from uuid import UUID

from pydantic import BaseModel, ConfigDict


class ContractStatusChangedEventDTO(BaseModel):
    model_config = ConfigDict(extra='forbid')

    event_id: UUID
    contract_id: UUID
    organization_id: UUID
    status: str
    changed_at: datetime
```

Публикация события:

```python
import json

from apps.contracts.dtos import ContractStatusChangedEventDTO


def contract_status_changed_publish(dto: ContractStatusChangedEventDTO) -> None:
    payload = dto.model_dump(mode='json')

    producer.publish(
        json.dumps(payload),
        exchange='contracts',
        routing_key='contract.status.changed',
        content_type='application/json',
    )
```

Обработка события:

```python
from pydantic import ValidationError

from apps.contracts.dtos import ContractStatusChangedEventDTO
from apps.contracts.services import contract_status_changed_process


def on_contract_status_changed(payload: dict) -> None:
    try:
        dto = ContractStatusChangedEventDTO.model_validate(payload)
    except ValidationError:
        # логирование, reject или отправка в DLQ зависит от инфраструктурного соглашения
        raise

    contract_status_changed_process(dto)
```

Правило: consumer не должен передавать сырой payload в бизнес-сервис.

## Celery-задачи

В Celery нельзя передавать Django-модели и сложные Python-объекты. В задачу передаётся JSON-совместимый словарь.

```python
from celery import shared_task

from apps.notifications.dtos import SendEmailDTO
from apps.notifications.services import email_send


@shared_task
def send_email_task(payload: dict) -> None:
    dto = SendEmailDTO.model_validate(payload)
    email_send(dto)
```

Вызов задачи:

```python
from apps.notifications.dtos import SendEmailDTO
from apps.notifications.tasks import send_email_task


def welcome_email_schedule(dto: SendEmailDTO) -> None:
    send_email_task.delay(dto.model_dump(mode='json'))
```

`mode='json'` важен, потому что приводит `datetime`, `UUID`, `Decimal` и другие типы к JSON-совместимому виду.

## TypeAdapter

`TypeAdapter` используется, когда нужно валидировать или сериализовать не одну `BaseModel`, а составной тип:

- `list[DTO]`;
- `dict[str, DTO]`;
- `list[UUID]`;
- `list[EmailStr]`;
- `str | int`;
- вложенные JSON-структуры без отдельной корневой модели.

Для одного DTO используй `model_validate`:

```python
user = UserDTO.model_validate(payload)
```

Для коллекции DTO используй `TypeAdapter`:

```python
from pydantic import TypeAdapter

from apps.users.dtos import UserDTO

UserListAdapter = TypeAdapter(list[UserDTO])


def users_parse(payload: list[dict]) -> list[UserDTO]:
    return UserListAdapter.validate_python(payload)
```

Адаптер создаётся один раз на уровне модуля. Не создавай `TypeAdapter` внутри горячего кода.

## TypeAdapter в межсервисных контрактах

`TypeAdapter` особенно полезен, когда другой сервис возвращает массив объектов без корневого поля.

Например, внешний сервис отдаёт такой JSON:

```json
[
  {
    "id": "7f82e4d4-3f72-4a52-aed0-fb0d9e69ef74",
    "email": "user@example.com",
    "full_name": "Ivan Petrov",
    "is_active": true,
    "created_at": "2026-04-29T10:00:00Z"
  }
]
```

Для такого ответа не нужно создавать искусственный `UsersResponseDTO` только ради поля `items`.

```python
import httpx
from pydantic import TypeAdapter

from apps.users.dtos import UserDTO

UserListAdapter = TypeAdapter(list[UserDTO])


class UserClient:
    def __init__(self, base_url: str, timeout: float) -> None:
        self.base_url = base_url
        self.timeout = timeout

    def get_users(self) -> list[UserDTO]:
        response = httpx.get(
            f'{self.base_url}/api/users/',
            timeout=self.timeout,
        )
        response.raise_for_status()

        return UserListAdapter.validate_python(response.json())
```

Если response body доступен как bytes, можно валидировать JSON напрямую:

```python
return UserListAdapter.validate_json(response.content)
```

## TypeAdapter для входящих списков

`TypeAdapter` удобен для endpoint-ов, которые принимают список значений.

```python
from uuid import UUID

from pydantic import TypeAdapter, ValidationError
from rest_framework.response import Response
from rest_framework.views import APIView

SignatureIdListAdapter = TypeAdapter(list[UUID])


class SignatureContentAPIView(APIView):
    def post(self, request):
        try:
            signature_ids = SignatureIdListAdapter.validate_python(request.data)
        except ValidationError as error:
            return Response(error.errors(), status=400)

        ...
```

Это лучше, чем принимать `list[str]` и потом вручную приводить элементы к UUID в сервисе.

## TypeAdapter для сериализации

`TypeAdapter` умеет не только валидировать, но и сериализовать составные типы.

```python
payload = UserListAdapter.dump_python(users, mode='json')
payload_json = UserListAdapter.dump_json(users)
```

`dump_python(..., mode='json')` возвращает JSON-совместимую Python-структуру.

`dump_json` возвращает `bytes`. Это важно учитывать при передаче результата в HTTP response, брокер сообщений или S3.

## DTO для версионирования событий

Для событий и внешних контрактов желательно явно хранить версию схемы.

```python
from datetime import datetime
from typing import Literal
from uuid import UUID

from pydantic import BaseModel, ConfigDict


class ContractBlockedEventV1DTO(BaseModel):
    model_config = ConfigDict(extra='forbid')

    schema_version: Literal['1.0']
    event_id: UUID
    contract_id: UUID
    organization_id: UUID
    reason: str
    occurred_at: datetime
```

Версия нужна, чтобы consumer мог безопасно отличать старый формат сообщения от нового.

## DTO и обратная совместимость

При изменении межсервисного контракта нельзя просто удалить или переименовать поле.

Безопасные изменения:

- добавить необязательное поле с дефолтом;
- добавить новое значение enum, если consumer готов его обработать;
- расширить DTO новой версией события;
- поддержать старый и новый формат на период миграции.

Опасные изменения:

- удалить обязательное поле;
- переименовать поле;
- поменять тип поля;
- изменить смысл существующего поля;
- начать отправлять `null`, если раньше поле всегда было строкой, числом или объектом.

## model_dump для внешних контрактов

Для внешних JSON-контрактов почти всегда используй:

```python
dto.model_dump(mode='json')
```

Не используй обычный `model_dump()` для payload, который уйдёт в HTTP, Celery или RabbitMQ, если внутри есть `UUID`, `datetime`, `date`, `time`, `Decimal` или другие нестандартные JSON-типы.

## model_validate_json

Если данные уже пришли в виде JSON-строки или bytes, можно валидировать без предварительного `json.loads`.

```python
class WebhookDTO(BaseModel):
    model_config = ConfigDict(extra='forbid')

    event_id: str
    event_type: str


def webhook_parse(raw_body: bytes) -> WebhookDTO:
    return WebhookDTO.model_validate_json(raw_body)
```

Для списка или другого составного типа используй `TypeAdapter.validate_json`.

```python
WebhookListAdapter = TypeAdapter(list[WebhookDTO])


def webhooks_parse(raw_body: bytes) -> list[WebhookDTO]:
    return WebhookListAdapter.validate_json(raw_body)
```

## Ошибки валидации

Ошибки Pydantic нельзя отдавать наружу бездумно во всех случаях.

Для публичного API можно вернуть нормализованный ответ 400.

Для внутреннего consumer-а ошибка валидации означает нарушение контракта. Обычно её нужно логировать с `event_id`, `routing_key`, `message_id`, `x-request-id` и отправлять сообщение в DLQ или retry-механизм, если это предусмотрено инфраструктурой.

```python
from pydantic import ValidationError


def payload_validate(payload: dict) -> ContractStatusChangedEventDTO:
    try:
        return ContractStatusChangedEventDTO.model_validate(payload)
    except ValidationError as error:
        logger.warning(
            'Invalid contract status event payload',
            extra={'errors': error.errors()},
        )
        raise
```

## Что не должно попадать в DTO

Не добавляй в DTO:

- Django Model;
- QuerySet;
- request;
- response;
- serializer;
- database connection;
- settings object;
- logger;
- Celery task instance;
- HTTP client;
- producer или consumer брокера.

DTO должен оставаться простым объектом данных.

## Антипаттерны

### Толстые DTO

Плохо:

```python
class SendEmailDTO(BaseModel):
    email: str

    def send(self) -> None:
        ...
```

Хорошо:

```python
class SendEmailDTO(BaseModel):
    email: str


def email_send(dto: SendEmailDTO) -> None:
    ...
```

### Сырые dict в сервисах

Плохо:

```python
def order_create(data: dict) -> None:
    product_id = data['product_id']
```

Хорошо:

```python
def order_create(dto: CreateOrderDTO) -> OrderDTO:
    product_id = dto.product_id
```

### Бизнес-валидация внутри DTO

Плохо:

```python
class CreateUserDTO(BaseModel):
    email: str

    @model_validator(mode='after')
    def validate_unique_email(self):
        if User.objects.filter(email=self.email).exists():
            raise ValueError('Email already exists')
        return self
```

Хорошо:

```python
def user_create(dto: CreateUserDTO) -> UserDTO:
    if user_email_exists(dto.email):
        raise UserAlreadyExistsError

    ...
```

### Создание TypeAdapter внутри функции

Плохо:

```python
def users_parse(payload: list[dict]) -> list[UserDTO]:
    return TypeAdapter(list[UserDTO]).validate_python(payload)
```

Хорошо:

```python
UserListAdapter = TypeAdapter(list[UserDTO])


def users_parse(payload: list[dict]) -> list[UserDTO]:
    return UserListAdapter.validate_python(payload)
```

### Передача DTO напрямую в Celery

Плохо:

```python
send_email_task.delay(dto)
```

Хорошо:

```python
send_email_task.delay(dto.model_dump(mode='json'))
```

## Тестирование DTO-контрактов

DTO-контракты нужно тестировать отдельно от бизнес-логики.

```python
from apps.contracts.dtos import ContractStatusChangedEventDTO


def test_contract_status_changed_event_valid_payload():
    payload = {
        'event_id': '7f82e4d4-3f72-4a52-aed0-fb0d9e69ef74',
        'contract_id': '2e6f9d0e-33b1-4e5d-82c9-82a7156c7581',
        'organization_id': 'b4d8c6d1-47cb-4a83-86f1-01aa708a6dcb',
        'status': 'active',
        'changed_at': '2026-04-29T10:00:00Z',
    }

    dto = ContractStatusChangedEventDTO.model_validate(payload)

    assert dto.status == 'active'
```

Для исходящих сообщений полезно тестировать JSON-совместимый dump.

```python
def test_contract_status_changed_event_dump_json_mode():
    dto = ContractStatusChangedEventDTO.model_validate(payload)

    dumped = dto.model_dump(mode='json')

    assert isinstance(dumped['event_id'], str)
    assert isinstance(dumped['changed_at'], str)
```

## Чек-лист код-ревью

- [ ] DTO используется как объект данных, а не как место для бизнес-логики.
- [ ] DTO не импортирует Django Models, QuerySet, request, response или инфраструктурные клиенты.
- [ ] Сервис принимает DTO, а не сырой `dict`, если структура данных является контрактом.
- [ ] Входящие payload из HTTP, RabbitMQ, Celery и webhook валидируются на границе.
- [ ] Исходящие payload формируются через `model_dump(mode='json')`.
- [ ] Ответы внешних сервисов валидируются через DTO перед передачей в бизнес-логику.
- [ ] Для списков и других составных типов используется `TypeAdapter`.
- [ ] `TypeAdapter` создаётся на уровне модуля, а не внутри горячего кода.
- [ ] DTO для событий имеют понятное имя и при необходимости поле версии схемы.
- [ ] Изменения межсервисного контракта сохраняют обратную совместимость или оформлены новой версией DTO.
- [ ] Ошибки валидации внутренних сообщений логируются как нарушение контракта.
- [ ] Не используется устаревший синтаксис Pydantic v1: `parse_obj`, `from_orm`, `dict`, `json`.
