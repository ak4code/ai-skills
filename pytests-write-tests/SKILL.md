---
name: pytest-django-stack
description: Используй этот навык всегда, когда нужно писать или ревьюить тесты на pytest для проекта на стеке Django, DRF, PostgreSQL, Redis, Celery, RabbitMQ. Покрывает функциональный стиль pytest, фикстуры, параметризацию, factory_boy, тестирование сервисов и селекторов, DRF endpoints, асинхронных задач Celery, кэша Redis через fakeredis, очередей RabbitMQ, моки внешних HTTP сервисов и фиксацию времени. Подключай при словах "напиши тест", "покрой тестами", "фикстура", "factory", "тест задачи", "mock", "проверь endpoint", даже если фреймворк не назван явно.
---

# Pytest для Django стека

Этот навык учит писать тесты для проекта на Django, DRF, PostgreSQL, Redis, Celery и RabbitMQ. Все тесты пишем в функциональном стиле на pytest. Тестовые классы запрещены.

## Базовые принципы

1. Только функции. Никаких `class TestXxx`. pytest даёт всё необходимое через фикстуры и параметризацию.
2. Один тест проверяет одно поведение. Если в имени теста есть "и", тест надо разделить.
3. Имя теста описывает сценарий. Шаблон: `test_<что>_<при_каких_условиях>_<ожидаемый_результат>`. Пример: `test_create_order_when_stock_empty_raises_validation_error`.
4. Тело теста разделено на три блока: Arrange, Act, Assert. Между блоками одна пустая строка. Это видно глазами без комментариев.
5. Тест читается сверху вниз как короткий рассказ. Если читателю нужна шпаргалка, тест надо переписать.
6. Тест изолирован. Не зависит от порядка запуска и от других тестов.
7. Тест детерминирован. Никаких `datetime.now()`, `random`, `uuid4` без фиксации.
8. Тест проверяет поведение, не реализацию. Не пишем тесты, которые ломаются от рефакторинга без изменения функциональности.
9. Никаких inline комментариев в тесте. Сценарий описан в docstring по AAA.

## Формат docstring для тестов

В каждом тесте docstring пишется строго по шаблону. Никаких `:param:` и `:return:` для тестов. Только три строки: Arrange, Act, Assert.

```python
def test_create_order_succeeds() -> None:
    """
    Arrange: Подготовка данных
    Act: Выполнение действия
    Assert: Проверка результата
    """
```

Каждая строка содержит конкретное описание для своего теста, не шаблонный текст. Пример настоящего docstring:

```python
def test_create_order_persists_order_with_items(regular_user) -> None:
    """
    Arrange: Продукт с остатком на складе и корзина с одной позицией
    Act: Вызов сервиса создания заказа
    Assert: Заказ создан, итог посчитан, остаток уменьшился
    """
```

Правила:
- Каждое описание начинается с заглавной буквы.
- В конце строки точка не ставится.
- Описание короткое, одно предложение, без подробностей кода.
- Содержательное. Не "Подготовка данных", а конкретно что подготовили.

Для не тестовых функций (фикстуры, хелперы) docstring пишется в формате Sphinx/reST с полями `:param:`, `:return:`, `:raises:`.

## Структура каталога тестов

```
tests/
├── conftest.py
├── factories/
│   ├── __init__.py
│   ├── orders.py
│   └── users.py
├── unit/
│   └── orders/
│       ├── conftest.py
│       ├── test_selectors.py
│       ├── test_services.py
│       └── test_tasks.py
├── integration/
│   └── orders/
│       ├── test_api.py
│       └── test_celery_pipeline.py
└── e2e/
    └── test_checkout_flow.py
```

Правила:
- Тесты живут вне пакета `apps`. Это убирает соблазн импортировать приватные хелперы из тестов в продакшен код.
- Юнит тесты не ходят в базу. Если ходят, это уже интеграционный тест.
- Каждый каталог тестов имеет свой `conftest.py`, если нужны узкие фикстуры. Глобальные фикстуры живут в корневом `tests/conftest.py`.
- Фабрики данных лежат в одном месте и переиспользуются.

## Фикстуры

Фикстура это функция, которая готовит ресурс или данные и отдаёт их тесту. Главное правило: фикстура решает одну задачу.

### Скоупы фикстур

| Скоуп | Когда применять |
|---|---|
| `function` | По умолчанию. Для всего, что меняет состояние. |
| `module` | Дорогие ресурсы только для чтения внутри одного файла. |
| `package` | Общий ресурс для группы тестов в каталоге. |
| `session` | Контейнеры testcontainers, поднятые один раз на запуск. |

### Базовые фикстуры в корневом conftest

```python
import pytest
from rest_framework.test import APIClient

from tests.factories.users import UserFactory


@pytest.fixture
def user_password() -> str:
    """
    Возвращает пароль по умолчанию для тестового пользователя.

    :return: строка с паролем.
    """
    return "test-password-123"


@pytest.fixture
def regular_user(db, user_password):
    """
    Создаёт обычного пользователя в базе.

    :param db: фикстура pytest-django, открывающая доступ к БД.
    :param user_password: пароль из фикстуры.
    :return: экземпляр модели пользователя.
    """
    return UserFactory(password=user_password)


@pytest.fixture
def staff_user(db, user_password):
    """
    Создаёт сотрудника с правом на админку.

    :param db: фикстура pytest-django.
    :param user_password: пароль из фикстуры.
    :return: экземпляр модели пользователя с is_staff=True.
    """
    return UserFactory(password=user_password, is_staff=True)


@pytest.fixture
def api_client() -> APIClient:
    """
    Возвращает неаутентифицированный клиент DRF.

    :return: экземпляр APIClient.
    """
    return APIClient()


@pytest.fixture
def authenticated_client(api_client, regular_user) -> APIClient:
    """
    Возвращает клиент DRF с активной сессией обычного пользователя.

    :param api_client: чистый клиент.
    :param regular_user: пользователь для входа.
    :return: тот же APIClient после force_authenticate.
    """
    api_client.force_authenticate(user=regular_user)
    return api_client
```

### Yield фикстуры с очисткой

Когда фикстура захватывает ресурс, освобождай его после теста. Через `yield` это выглядит просто.

```python
import pytest
from django.core.cache import cache


@pytest.fixture
def clean_cache():
    """
    Очищает кэш до и после теста.

    :return: None, тест получает чистый кэш.
    """
    cache.clear()
    yield
    cache.clear()
```

### autouse фикстуры

Используй экономно. Каждая autouse фикстура замедляет все тесты. Хороший случай: сброс почтового outbox.

```python
import pytest
from django.core import mail


@pytest.fixture(autouse=True)
def reset_mail_outbox():
    """
    Очищает почтовый outbox перед каждым тестом.

    :return: None.
    """
    mail.outbox.clear()
    yield
```

### Фикстуры с параметрами через request

```python
import pytest

from tests.factories.users import UserFactory


@pytest.fixture
def user_with_role(db, request):
    """
    Создаёт пользователя с ролью, указанной через indirect параметризацию.

    :param db: фикстура pytest-django.
    :param request: pytest request с параметром роли.
    :return: пользователь с указанной ролью.
    """
    role_name = request.param
    return UserFactory(role=role_name)
```

Использование:

```python
@pytest.mark.parametrize(
    "user_with_role",
    ["admin", "manager", "viewer"],
    indirect=True,
)
def test_dashboard_access_depends_on_role(user_with_role, api_client) -> None:
    """
    Arrange: Пользователь с конкретной ролью
    Act: Запрос дашборда от его имени
    Assert: Статус ответа соответствует правам роли
    """
    api_client.force_authenticate(user=user_with_role)

    response = api_client.get("/api/dashboard/")

    expected_status = 200 if user_with_role.role in {"admin", "manager"} else 403
    assert response.status_code == expected_status
```

## Параметризация

Параметризация заменяет копипасту. Один тест проверяет десятки случаев.

```python
import pytest

from apps.billing.services import calculate_discount


@pytest.mark.parametrize(
    ("subtotal", "promo_code", "expected_discount"),
    [
        pytest.param(100, "", 0, id="no-promo"),
        pytest.param(100, "SAVE10", 10, id="percent-promo"),
        pytest.param(100, "FIXED5", 5, id="fixed-promo"),
        pytest.param(50, "MIN100", 0, id="below-minimum"),
        pytest.param(1000, "VIP", 200, id="vip-cap-applied"),
    ],
)
def test_calculate_discount_returns_expected_value(
    subtotal: int,
    promo_code: str,
    expected_discount: int,
) -> None:
    """
    Arrange: Сумма заказа и промокод из параметров
    Act: Вызов расчёта скидки
    Assert: Возвращённая скидка равна ожидаемой
    """
    actual_discount = calculate_discount(subtotal=subtotal, promo_code=promo_code)

    assert actual_discount == expected_discount
```

Правила:
- Каждый набор параметров получает явный `id`. Без него отчёт превращается в кашу.
- Кортеж параметров идёт первым аргументом. Имена выровнены с аргументами теста.
- Для длинных списков выноси данные в константу модуля.

### Параметризация на нескольких уровнях

```python
@pytest.mark.parametrize("currency", ["USD", "EUR", "GBP"])
@pytest.mark.parametrize("amount", [0, 100, 999_999])
def test_format_money_handles_all_combinations(currency: str, amount: int) -> None:
    """
    Arrange: Декартово произведение валют и сумм
    Act: Форматирование суммы
    Assert: Строка содержит код валюты и сумму
    """
    formatted_value = format_money(amount=amount, currency=currency)

    assert currency in formatted_value
    assert str(amount) in formatted_value
```

## Фабрики тестовых данных через factory_boy

Не создавай объекты вручную через `Model.objects.create(...)`. Используй фабрики. Они задают разумные дефолты, поддерживают связи и работают с `pytest-factoryboy`.

```python
import factory
from factory.django import DjangoModelFactory
from factory.fuzzy import FuzzyChoice

from apps.orders.models import Order, OrderItem
from apps.users.models import User


class UserFactory(DjangoModelFactory):
    class Meta:
        model = User
        django_get_or_create = ("email",)

    email = factory.Sequence(lambda counter: f"user-{counter}@example.com")
    full_name = factory.Faker("name")
    is_active = True
    password = factory.PostGenerationMethodCall("set_password", "test-password-123")

    class Params:
        admin = factory.Trait(is_staff=True, is_superuser=True)


class OrderFactory(DjangoModelFactory):
    class Meta:
        model = Order

    customer = factory.SubFactory(UserFactory)
    status = FuzzyChoice(["pending", "paid", "shipped"])
    total_amount = factory.Faker(
        "pydecimal", left_digits=4, right_digits=2, positive=True
    )


class OrderItemFactory(DjangoModelFactory):
    class Meta:
        model = OrderItem

    order = factory.SubFactory(OrderFactory)
    product_name = factory.Faker("word")
    quantity = factory.Faker("pyint", min_value=1, max_value=10)
    unit_price = factory.Faker(
        "pydecimal", left_digits=3, right_digits=2, positive=True
    )
```

Применение:

```python
def test_order_total_matches_sum_of_items(db) -> None:
    """
    Arrange: Заказ с тремя позициями по две штуки и цене десять
    Act: Пересчёт итога через сервис
    Assert: Итог равен шестидесяти
    """
    order = OrderFactory(total_amount=0)
    OrderItemFactory.create_batch(3, order=order, quantity=2, unit_price=10)

    recalculated_total = recalculate_order_total(order_id=order.id)

    assert recalculated_total == 60
```

Главные правила:
- `Sequence` для уникальных полей. Защищает от дубликатов.
- `SubFactory` для связей `ForeignKey`. Не создавай родителя руками.
- `Trait` для частых модификаций. Делает фабрику читаемой.
- `PostGenerationMethodCall` для паролей и других пост обработок.
- `django_get_or_create` для справочников.
- `create_batch` для массового создания.
- `build` без сохранения в БД, когда модель не нужна в базе.

### pytest-factoryboy для авторегистрации

```python
from pytest_factoryboy import register

from tests.factories.orders import OrderFactory, OrderItemFactory
from tests.factories.users import UserFactory

register(UserFactory)
register(OrderFactory)
register(OrderItemFactory)
```

После регистрации фикстуры `user`, `order`, `order_item`, `user_factory`, `order_factory` доступны во всех тестах автоматически.

## Тестирование селекторов

Селектор это функция чтения из БД. Тест проверяет:
- что возвращает правильные данные при разных входах,
- что не делает лишних запросов.

```python
import pytest

from apps.orders.selectors import get_orders_with_items_for_customer
from tests.factories.orders import OrderFactory, OrderItemFactory


@pytest.mark.django_db
def test_get_orders_with_items_for_customer_returns_only_owned_orders(
    regular_user,
) -> None:
    """
    Arrange: Два заказа целевого пользователя и один чужой
    Act: Вызов селектора по id целевого пользователя
    Assert: Получены ровно два заказа, оба принадлежат целевому пользователю
    """
    own_orders = OrderFactory.create_batch(2, customer=regular_user)
    OrderFactory()

    fetched_orders = list(
        get_orders_with_items_for_customer(customer_id=regular_user.id)
    )

    assert len(fetched_orders) == 2
    assert {order.id for order in fetched_orders} == {order.id for order in own_orders}


@pytest.mark.django_db
def test_get_orders_with_items_for_customer_does_not_have_n_plus_one(
    regular_user,
    django_assert_num_queries,
) -> None:
    """
    Arrange: Пять заказов с тремя позициями в каждом
    Act: Вызов селектора и обход позиций каждого заказа
    Assert: Количество SQL запросов не растёт от числа заказов
    """
    orders = OrderFactory.create_batch(5, customer=regular_user)
    for order in orders:
        OrderItemFactory.create_batch(3, order=order)

    with django_assert_num_queries(2):
        fetched_orders = list(
            get_orders_with_items_for_customer(customer_id=regular_user.id)
        )
        for order in fetched_orders:
            list(order.items.all())
```

Что важно:
- `django_assert_num_queries` ловит регрессии N+1.
- Тестируем выборку, не строение запроса. Если селектор сменил `select_related` на `Prefetch` с тем же результатом, тест должен пройти.
- Создаём чужой объект, чтобы убедиться, что селектор фильтрует.

## Тестирование сервисов

Сервис меняет состояние. Тест проверяет:
- успешный сценарий и результат,
- провал по бизнес правилу с исключением,
- побочные эффекты: записи в БД, отправку писем, постановку задач.

```python
import pytest
from decimal import Decimal

from apps.orders.services import create_order, OrderValidationError
from tests.factories.products import ProductFactory


@pytest.mark.django_db
def test_create_order_persists_order_with_items(regular_user) -> None:
    """
    Arrange: Продукт с остатком десять и корзина с двумя единицами
    Act: Вызов сервиса создания заказа
    Assert: Заказ создан, итог равен двумстам, остаток уменьшился до восьми
    """
    product = ProductFactory(stock=10, price=Decimal("100.00"))
    cart_items = [{"product_id": product.id, "quantity": 2}]

    created_order = create_order(customer=regular_user, items=cart_items)

    assert created_order.pk is not None
    assert created_order.total_amount == Decimal("200.00")
    assert created_order.items.count() == 1
    product.refresh_from_db()
    assert product.stock == 8


@pytest.mark.django_db
def test_create_order_when_stock_insufficient_raises_validation_error(
    regular_user,
) -> None:
    """
    Arrange: Продукт с одним экземпляром на складе и запрос на два
    Act: Вызов сервиса создания заказа
    Assert: Поднимается OrderValidationError, заказ не создан
    """
    product = ProductFactory(stock=1)
    cart_items = [{"product_id": product.id, "quantity": 2}]

    with pytest.raises(OrderValidationError, match="Недостаточно товара"):
        create_order(customer=regular_user, items=cart_items)

    assert regular_user.orders.count() == 0


@pytest.mark.django_db
def test_create_order_schedules_confirmation_email(regular_user, mocker) -> None:
    """
    Arrange: Подменена задача отправки письма
    Act: Создание заказа через сервис
    Assert: Задача поставлена в очередь с id заказа
    """
    send_email_mock = mocker.patch(
        "apps.orders.services.send_order_confirmation_email.delay"
    )
    product = ProductFactory(stock=10)

    created_order = create_order(
        customer=regular_user,
        items=[{"product_id": product.id, "quantity": 1}],
    )

    send_email_mock.assert_called_once_with(order_id=created_order.id)
```

Правила:
- `pytest.raises(..., match="...")` проверяет тип и сообщение исключения.
- Побочный эффект мокается на месте использования, не на месте определения. Патчим `apps.orders.services.send_order_confirmation_email`, не `apps.notifications.tasks.send_order_confirmation_email`.
- В одном тесте одна Act строчка. Если их несколько, тест проверяет несколько вещей и должен быть разделён.

## Тестирование DRF endpoints

Тестируем поведение API: статус, структуру ответа, изменения в БД. Используем `APIClient` из `rest_framework.test`.

### Успешный сценарий

```python
import pytest
from rest_framework import status

from apps.orders.models import Order
from tests.factories.orders import OrderFactory
from tests.factories.products import ProductFactory


@pytest.mark.django_db
def test_list_orders_returns_only_owned_orders(
    authenticated_client, regular_user
) -> None:
    """
    Arrange: Три заказа пользователя и два чужих
    Act: GET запрос на список заказов
    Assert: В ответе ровно три заказа, все принадлежат пользователю
    """
    OrderFactory.create_batch(3, customer=regular_user)
    OrderFactory.create_batch(2)

    response = authenticated_client.get("/api/orders/")

    assert response.status_code == status.HTTP_200_OK
    response_payload = response.json()
    assert len(response_payload["results"]) == 3
    assert all(
        item["customer_id"] == regular_user.id
        for item in response_payload["results"]
    )


@pytest.mark.django_db
def test_create_order_returns_201_and_persists(authenticated_client) -> None:
    """
    Arrange: Валидный payload с одной позицией
    Act: POST на endpoint создания заказа
    Assert: Статус 201, в ответе id, заказ сохранён в БД
    """
    product = ProductFactory(stock=5)
    request_payload = {"items": [{"product_id": product.id, "quantity": 1}]}

    response = authenticated_client.post(
        "/api/orders/",
        data=request_payload,
        format="json",
    )

    assert response.status_code == status.HTTP_201_CREATED
    response_body = response.json()
    assert "id" in response_body
    assert Order.objects.filter(id=response_body["id"]).exists()
```

### Авторизация и доступ

```python
@pytest.mark.django_db
def test_create_order_unauthorized_returns_401(api_client) -> None:
    """
    Arrange: Неаутентифицированный клиент
    Act: POST на endpoint создания заказа
    Assert: Статус 401, заказы не созданы
    """
    response = api_client.post("/api/orders/", data={}, format="json")

    assert response.status_code == status.HTTP_401_UNAUTHORIZED
    assert Order.objects.count() == 0


@pytest.mark.django_db
def test_get_other_user_order_returns_404(authenticated_client) -> None:
    """
    Arrange: Чужой заказ существует в базе
    Act: GET запрос на чужой заказ от имени пользователя
    Assert: Статус 404, без утечки информации о существовании
    """
    other_user_order = OrderFactory()

    response = authenticated_client.get(f"/api/orders/{other_user_order.id}/")

    assert response.status_code == status.HTTP_404_NOT_FOUND
```

### Валидация payload

```python
@pytest.mark.django_db
def test_create_order_with_empty_payload_returns_400(authenticated_client) -> None:
    """
    Arrange: Payload без обязательного поля items
    Act: POST на endpoint создания заказа
    Assert: Статус 400, в ответе ошибка про items
    """
    response = authenticated_client.post(
        "/api/orders/",
        data={},
        format="json",
    )

    assert response.status_code == status.HTTP_400_BAD_REQUEST
    assert "items" in response.json()


@pytest.mark.django_db
@pytest.mark.parametrize(
    ("invalid_payload", "expected_field"),
    [
        pytest.param({"items": []}, "items", id="empty-items"),
        pytest.param({"items": [{"quantity": 1}]}, "product_id", id="missing-product"),
        pytest.param(
            {"items": [{"product_id": 1, "quantity": 0}]},
            "quantity",
            id="zero-quantity",
        ),
    ],
)
def test_create_order_validation_errors(
    authenticated_client,
    invalid_payload: dict,
    expected_field: str,
) -> None:
    """
    Arrange: Невалидный payload и ожидаемое имя поля с ошибкой
    Act: POST на endpoint создания заказа
    Assert: Статус 400, ошибка содержит ожидаемое поле
    """
    response = authenticated_client.post(
        "/api/orders/",
        data=invalid_payload,
        format="json",
    )

    assert response.status_code == status.HTTP_400_BAD_REQUEST
    assert expected_field in str(response.json())
```

Правила:
- Используем `format="json"` для DRF клиента. Без него `dict` уйдёт как form-data.
- Статусы берём из `rest_framework.status.HTTP_*`. Магические числа запрещены.
- Для каждого endpoint минимум четыре теста: успех, не авторизован, нет прав, невалидный payload.
- Структуру ответа проверяем по ключевым полям, не сравниваем весь JSON целиком.
- Проверяем не только ответ, но и состояние в БД после запроса.

## Тестирование Celery задач

Базовый принцип: задачу проверяй как обычную функцию через её код, а саму инфраструктуру отдельно.

### Юнит тест задачи

В тестовых настройках включён `CELERY_TASK_ALWAYS_EAGER = True`. Любой `task.delay()` или `task.apply_async()` выполняется синхронно. Это позволяет вызывать задачу как функцию.

```python
import pytest
from celery.exceptions import Retry

from apps.orders.tasks import process_payment
from tests.factories.orders import OrderFactory


@pytest.mark.django_db
def test_process_payment_marks_order_as_paid(mocker) -> None:
    """
    Arrange: Платёжный шлюз подменён на успешный ответ
    Act: Прямой вызов задачи как функции
    Assert: Заказ переведён в статус paid
    """
    order = OrderFactory(status="pending")
    mocker.patch(
        "apps.orders.tasks.payment_gateway.charge",
        return_value={"status": "ok", "transaction_id": "tx-1"},
    )

    process_payment(order_id=order.id)

    order.refresh_from_db()
    assert order.status == "paid"


@pytest.mark.django_db
def test_process_payment_retries_on_temporary_gateway_error(mocker) -> None:
    """
    Arrange: Платёжный шлюз кидает временную ошибку
    Act: Вызов задачи
    Assert: Задача планирует Retry
    """
    order = OrderFactory(status="pending")
    mocker.patch(
        "apps.orders.tasks.payment_gateway.charge",
        side_effect=ConnectionError("timeout"),
    )

    with pytest.raises(Retry):
        process_payment(order_id=order.id)
```

### Сервис, который ставит задачу

Сервис вызывает `task.delay()`. Eager режим выполнит задачу синхронно. Это позволяет проверить весь пайплайн в одном тесте.

```python
@pytest.mark.django_db
def test_create_order_triggers_full_pipeline(regular_user) -> None:
    """
    Arrange: Продукт в наличии
    Act: Создание заказа через сервис
    Assert: Заказ оплачен, отправлено письмо, обновлена аналитика
    """
    product = ProductFactory(stock=5)

    created_order = create_order(
        customer=regular_user,
        items=[{"product_id": product.id, "quantity": 1}],
    )

    created_order.refresh_from_db()
    assert created_order.status == "paid"
    assert len(mail.outbox) == 1
    assert AnalyticsEvent.objects.filter(order_id=created_order.id).exists()
```

### Тестирование цепочек, групп и хордов

Когда нужно проверить именно цепочку, патч высокого уровня не годится. Используй `pytest-celery` с реальным воркером.

```python
import pytest
from celery import chain

from apps.orders.tasks import fetch_invoice, send_invoice
from tests.factories.orders import OrderFactory


@pytest.mark.celery
def test_invoice_pipeline_executes_in_order(celery_session_worker, db) -> None:
    """
    Arrange: Цепочка из двух задач и созданный заказ
    Act: Запуск цепочки и ожидание результата
    Assert: Каждая задача выполнилась, итог сохранён
    """
    order = OrderFactory()

    pipeline_result = chain(
        fetch_invoice.s(order_id=order.id),
        send_invoice.s(),
    ).apply_async()
    pipeline_result.get(timeout=10)

    order.refresh_from_db()
    assert order.invoice_sent_at is not None
```

### Когда mocking лучше eager

Если задача делает HTTP вызов или работает с внешним API, мокай вызов, не задачу. Это проверяет код задачи, а не код её запуска.

## Тестирование Redis через fakeredis

Redis мокается через `fakeredis`. Это in-memory подмена клиента, которая поддерживает почти все команды настоящего Redis. Тесты остаются быстрыми и не требуют запущенного сервиса.

### Базовая фикстура

```python
import pytest
from fakeredis import FakeStrictRedis


@pytest.fixture
def fake_redis(mocker) -> FakeStrictRedis:
    """
    Подменяет реальный Redis клиент на fakeredis.

    :param mocker: фикстура pytest-mock.
    :return: фейковый клиент с включённым decode_responses.
    """
    fake_client = FakeStrictRedis(decode_responses=True)
    mocker.patch("apps.realtime.services.redis_client", fake_client)
    return fake_client
```

Главное правило: патчим именно ту переменную, которую импортирует наш код. Если в `apps.realtime.services` написано `from apps.shared.redis import redis_client`, то патчим `apps.realtime.services.redis_client`, не `apps.shared.redis.redis_client`.

### Тест записи

```python
from apps.realtime.services import push_notification


def test_push_notification_writes_to_user_channel(fake_redis) -> None:
    """
    Arrange: Фейковый Redis подменён в сервисе
    Act: Push уведомления пользователю с id 42
    Assert: В списке notifications:42 появилась запись с сообщением
    """
    push_notification(user_id=42, message="hello")

    stored_value = fake_redis.lindex("notifications:42", 0)
    assert "hello" in stored_value
```

### Тест чтения

```python
from apps.realtime.services import get_pending_notifications


def test_get_pending_notifications_returns_all_messages(fake_redis) -> None:
    """
    Arrange: В Redis заранее положены три сообщения для пользователя
    Act: Чтение всех ожидающих уведомлений
    Assert: Возвращён список из трёх сообщений в порядке добавления
    """
    fake_redis.rpush("notifications:42", "first", "second", "third")

    pending_messages = get_pending_notifications(user_id=42)

    assert pending_messages == ["first", "second", "third"]
```

### Тест кэша с инвалидацией

```python
from apps.catalog.services import get_or_set_top_products


def test_get_or_set_top_products_caches_result(fake_redis, db, mocker) -> None:
    """
    Arrange: Слежение за вызовом БД через spy, кэш пуст
    Act: Два последовательных вызова сервиса
    Assert: Запрос в БД случился ровно один раз
    """
    spy_database_call = mocker.spy(Product.objects, "filter")
    ProductFactory.create_batch(5, is_top=True)

    first_result = get_or_set_top_products()
    second_result = get_or_set_top_products()

    assert first_result == second_result
    assert spy_database_call.call_count == 1
```

### Изоляция между тестами

`FakeStrictRedis` создаётся в фикстуре с дефолтным `function` скоупом. Каждый тест получает чистый клиент. Если фикстура поднимается выше по скоупу, добавь `flushall` после теста через yield.

```python
@pytest.fixture
def shared_fake_redis(mocker):
    """
    Подменяет Redis клиент на общий fakeredis для модуля.

    :param mocker: фикстура pytest-mock.
    :return: фейковый клиент.
    """
    fake_client = FakeStrictRedis(decode_responses=True)
    mocker.patch("apps.realtime.services.redis_client", fake_client)
    yield fake_client
    fake_client.flushall()
```

### Когда нужны команды с временем

`fakeredis` уважает `EXPIRE` и `TTL`. Для тестов с таймаутами комбинируй с `freezegun`.

```python
from freezegun import freeze_time


def test_session_expires_after_one_hour(fake_redis) -> None:
    """
    Arrange: Записан ключ сессии с TTL час
    Act: Сдвиг времени на два часа вперёд
    Assert: Ключ больше не существует
    """
    with freeze_time("2026-01-01 12:00:00"):
        fake_redis.set("session:1", "data", ex=3600)

    with freeze_time("2026-01-01 14:00:00"):
        stored_value = fake_redis.get("session:1")

    assert stored_value is None
```

## Тестирование RabbitMQ

Прямые тесты publisher и consumer. Поднимаем брокер через `testcontainers` и помечаем тесты маркером `rabbitmq`. Запускаем только в интеграционной стадии CI.

```python
import json
import pytest
from testcontainers.rabbitmq import RabbitMqContainer

from apps.messaging.publisher import publish_event
from apps.messaging.consumer import consume_one_message


@pytest.fixture(scope="session")
def rabbitmq_container():
    """
    Поднимает RabbitMQ в контейнере на сессию.

    :return: запущенный контейнер.
    """
    with RabbitMqContainer("rabbitmq:3.13-management") as container:
        yield container


@pytest.fixture
def rabbitmq_url(rabbitmq_container) -> str:
    """
    Возвращает URL подключения к контейнеру RabbitMQ.

    :param rabbitmq_container: запущенный контейнер.
    :return: URL формата amqp://...
    """
    return rabbitmq_container.get_connection_url()


@pytest.mark.rabbitmq
def test_publish_event_message_is_consumable(rabbitmq_url) -> None:
    """
    Arrange: Publisher и consumer указывают на тестовый брокер
    Act: Публикация события и чтение одного сообщения
    Assert: Тело сообщения совпадает с опубликованным
    """
    event_payload = {"order_id": 1, "type": "created"}

    publish_event(broker_url=rabbitmq_url, queue="orders", payload=event_payload)
    received_body = consume_one_message(
        broker_url=rabbitmq_url, queue="orders", timeout=5
    )

    assert json.loads(received_body) == event_payload
```

Когда брокер не нужен, мокай `kombu.Connection` или клиент `aio-pika`. Контейнер дорогой, поднимай только под маркером `rabbitmq`.

## Mocking через pytest-mock

Используем фикстуру `mocker` из `pytest-mock`. Она автоматически отменяет патчи после теста, в отличие от голого `unittest.mock`.

### Базовый патч

```python
def test_payment_service_uses_external_gateway(mocker) -> None:
    """
    Arrange: Внешний шлюз подменён на успешный ответ
    Act: Вызов сервиса оплаты
    Assert: Шлюз вызван с нужными аргументами, результат проброшен
    """
    gateway_mock = mocker.patch("apps.billing.services.payment_gateway.charge")
    gateway_mock.return_value = {"status": "ok"}

    payment_result = process_payment(amount=100, customer_id=1)

    gateway_mock.assert_called_once_with(amount=100, customer_id=1)
    assert payment_result["status"] == "ok"
```

### side_effect для последовательности

```python
def test_retry_logic_succeeds_on_third_attempt(mocker) -> None:
    """
    Arrange: Первые два вызова падают по таймауту, третий успешен
    Act: Запуск функции с ретраями и максимумом три попытки
    Assert: Получен успешный ответ, было ровно три вызова
    """
    gateway_mock = mocker.patch("apps.billing.services.payment_gateway.charge")
    gateway_mock.side_effect = [
        ConnectionError("timeout"),
        ConnectionError("timeout"),
        {"status": "ok"},
    ]

    final_result = charge_with_retry(amount=100, max_attempts=3)

    assert final_result["status"] == "ok"
    assert gateway_mock.call_count == 3
```

### autospec против опечаток

`autospec=True` защищает от моков с несуществующими атрибутами. Если рефакторинг переименовал метод, тест упадёт.

```python
gateway_mock = mocker.patch(
    "apps.billing.services.payment_gateway",
    autospec=True,
)
```

### spy для наблюдения без подмены

```python
def test_repository_save_called_once(mocker) -> None:
    """
    Arrange: Слежение за реальным методом без подмены поведения
    Act: Создание заказа через сервис
    Assert: Метод репозитория вызван один раз
    """
    spy_save = mocker.spy(OrderRepository, "save")

    create_order(customer_id=1, items=[])

    assert spy_save.call_count == 1
```

Правила:
- Патчим там, где имя используется, не там, где оно определено.
- Чем выше уровень патча, тем хрупче тест. Патч одного метода стабильнее, чем патч целого модуля.
- Не мокай то, что владеешь, без необходимости. Если метод сервиса быстрый и без побочек, вызови его честно.

## Внешние HTTP вызовы

Используй `responses` для `requests` и `respx` для `httpx`. Это стабильнее, чем мокать клиент целиком.

```python
import pytest
import responses

from apps.integrations.shipping import (
    fetch_shipping_rate,
    ShippingServiceUnavailable,
)


@responses.activate
def test_fetch_shipping_rate_returns_parsed_value() -> None:
    """
    Arrange: Внешний API замокан со 200 ответом и тарифом 12.5
    Act: Запрос тарифа доставки на два килограмма
    Assert: Возвращена сумма из ответа API
    """
    responses.add(
        responses.GET,
        "https://api.shipping.example/rates",
        json={"rate": 12.5, "currency": "USD"},
        status=200,
    )

    shipping_rate = fetch_shipping_rate(weight_kg=2)

    assert shipping_rate == 12.5
    assert len(responses.calls) == 1


@responses.activate
def test_fetch_shipping_rate_raises_on_server_error() -> None:
    """
    Arrange: Внешний API возвращает 503
    Act: Запрос тарифа доставки
    Assert: Поднимается ShippingServiceUnavailable
    """
    responses.add(
        responses.GET,
        "https://api.shipping.example/rates",
        status=503,
    )

    with pytest.raises(ShippingServiceUnavailable):
        fetch_shipping_rate(weight_kg=2)
```

## Работа со временем

Никаких голых `datetime.now()` в коде, который тестируем. Всё, что зависит от времени, замораживаем через `freezegun`.

```python
import pytest
from datetime import datetime, timezone
from freezegun import freeze_time

from apps.subscriptions.services import is_subscription_active
from tests.factories.subscriptions import SubscriptionFactory


@pytest.mark.django_db
@freeze_time("2026-01-15 12:00:00")
def test_is_subscription_active_returns_true_inside_window() -> None:
    """
    Arrange: Подписка активна с 1 января по 1 февраля 2026
    Act: Проверка активности 15 января
    Assert: Возвращено True
    """
    subscription = SubscriptionFactory(
        starts_at=datetime(2026, 1, 1, tzinfo=timezone.utc),
        ends_at=datetime(2026, 2, 1, tzinfo=timezone.utc),
    )

    is_active = is_subscription_active(subscription_id=subscription.id)

    assert is_active is True
```

Для модели с `auto_now_add` оборачивай создание в `freeze_time`, иначе время выставит БД или Python в момент сохранения.

## Маркер django_db и его варианты

Маркер открывает доступ к базе. Без него тест с обращением в БД упадёт.

| Что нужно | Маркер | Скорость |
|---|---|---|
| Чтение и запись в одной транзакции | `@pytest.mark.django_db` | быстрая |
| Тест ловит коммит, тестирует сигналы on_commit | `@pytest.mark.django_db(transaction=True)` | медленная |
| Несколько баз | `@pytest.mark.django_db(databases=["default", "analytics"])` | медленная |

```python
@pytest.mark.django_db(transaction=True)
def test_signal_after_commit_runs_task(mocker) -> None:
    """
    Arrange: Подменена задача, которую сигнал ставит после коммита
    Act: Создание заказа через фабрику
    Assert: Задача поставлена один раз
    """
    task_mock = mocker.patch("apps.orders.signals.notify_warehouse.delay")

    OrderFactory()

    task_mock.assert_called_once()
```

`@pytest.mark.django_db` оборачивает каждый тест в транзакцию и откатывает её. Это быстро, но `transaction.on_commit` колбэки не выполняются. Для тестов сигналов с `on_commit` обязательно `transaction=True`.

## Snapshot тестирование

Подходит для сложных JSON ответов, отрендеренных писем и больших структур. Используем `syrupy`.

```python
def test_order_summary_response_matches_snapshot(snapshot, regular_user) -> None:
    """
    Arrange: Заказ с известным составом и двумя позициями
    Act: Рендер summary через сервис
    Assert: Ответ совпадает с сохранённым снимком
    """
    order = OrderFactory(customer=regular_user, total_amount=100)
    OrderItemFactory.create_batch(2, order=order)

    summary_payload = render_order_summary(order_id=order.id)

    assert summary_payload == snapshot
```

Снимки коммитятся в репозиторий. При изменении формата запускаем `pytest --snapshot-update` и ревьюим diff.

## Маркеры pytest

Маркеры группируют тесты для выборочного запуска. Все маркеры регистрируются в конфигурации pytest через `markers = [...]`.

```python
@pytest.mark.unit
def test_pure_function() -> None:
    ...

@pytest.mark.integration
@pytest.mark.django_db
def test_with_database() -> None:
    ...

@pytest.mark.slow
def test_long_running_simulation() -> None:
    ...

@pytest.mark.skip(reason="ждём фикс библиотеки X")
def test_known_failure() -> None:
    ...

@pytest.mark.skipif(sys.platform == "win32", reason="нет fsync на Windows")
def test_unix_only_behavior() -> None:
    ...

@pytest.mark.xfail(reason="баг подтверждён, ждём исправления")
def test_known_buggy_behavior() -> None:
    ...
```

## conftest.py: иерархия и лучшие практики

`conftest.py` подгружается автоматически из всех родительских каталогов до корневого. Это даёт чистый способ скоупить фикстуры без импортов.

Правила:
- Корневой `tests/conftest.py` хранит общие фикстуры: клиенты, базовых пользователей, очистку кэша.
- Подкаталог имеет свой `conftest.py`, если фикстура нужна только в нём.
- Импорт фикстуры между файлами через `from X import Y` плохая практика. Если фикстура нужна шире, поднимаем её в `conftest.py` ближайшего общего родителя.
- Регистрация factory_boy фабрик через `pytest_factoryboy.register` живёт в корневом conftest.

```python
import pytest


def pytest_collection_modifyitems(config, items) -> None:
    """
    Помечает все тесты в каталоге integration маркером integration.

    :param config: конфигурация pytest.
    :param items: собранные тесты.
    :return: None.
    """
    for collected_item in items:
        if "/integration/" in str(collected_item.fspath):
            collected_item.add_marker(pytest.mark.integration)
```

## Антипаттерны

- Тесты в классах. Запрещено. Используем функции.
- Inline комментарии в теле теста. Сценарий описан в docstring по AAA, остальное говорит сам код.
- Один тест на пять Assert разных сущностей. Разделить на пять тестов.
- `assert response.status_code == 200 and response.json()["x"] == 1`. Два независимых assert лучше одного составного. Сообщение об ошибке понятнее.
- Голый `Mock()` без `spec`. Защита от опечаток в имени метода ломается.
- Патч на уровне сторонней библиотеки. Делает тест зависимым от её внутренностей. Патчим на уровне нашего кода.
- Создание объектов через `Model.objects.create(...)` в тесте. Используем фабрику.
- Жёсткие ожидания на текстах сообщений вроде `assert msg == "Order #42 created"`. Хрупко. Проверяй структуру и ключевые поля.
- `time.sleep()` в тестах. Заменяем на `freeze_time` или ожидание состояния.
- Зависимость от порядка тестов. Pytest-randomly выявит. Изолируем тесты.
- Реальные HTTP вызовы. Используем `responses` или `respx`.
- Реальный Redis. Используем `fakeredis`.
- Реальный SMTP. Используем `EMAIL_BACKEND = locmem`.
- `print` вместо assert. Ничего не проверяет.
- Слишком умные фикстуры с десятком ветвлений. Сделай несколько узких фикстур.
- Docstring с шаблонным текстом "Arrange: Подготовка данных". Описывай конкретно, что подготовлено и что проверяем.

## Чек-лист перед коммитом

- [ ] Каждый тест это функция, не класс.
- [ ] Имя теста описывает сценарий и результат.
- [ ] В docstring три строки: Arrange, Act, Assert с конкретным описанием.
- [ ] Один тест проверяет одно поведение.
- [ ] Используются фабрики, не прямые вызовы ORM.
- [ ] Внешние HTTP сервисы замоканы через responses.
- [ ] Redis замокан через fakeredis.
- [ ] Время заморожено там, где это важно.
- [ ] `django_assert_num_queries` стоит на критических селекторах.
- [ ] Тесты проходят в случайном порядке через pytest-randomly.

## Краткая шпаргалка по приоритетам

1. Юнит тесты сервисов и селекторов покрывают бизнес правила. Это основа пирамиды.
2. Интеграционные тесты DRF endpoints покрывают контракты API и сериализацию.
3. Тесты задач Celery с моками внешних API проверяют код задачи.
4. Тесты с реальным брокером RabbitMQ запускаются по маркеру в CI на отдельной стадии.
5. e2e сценарии добавляются точечно для критичных бизнес флоу.
