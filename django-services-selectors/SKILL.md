---
name: django-services-selectors
description: Используй этот навык всегда при проектировании или ревью бизнес-логики в Django проекте. Учит разделению кода на слои services и selectors, держит модели тонкими, view/endpoint тонкими, сериализаторы тонкими. Покрывает где жить бизнес-логике, как писать сервисы для записи и селекторы для чтения, как они взаимодействуют с моделями и DRF, тестирование и типичные ошибки. Подключай при словах "куда положить логику", "service", "selector", "тонкая модель", "fat model", "толстый view", "куда поместить бизнес-правило", "архитектура Django приложения", "слои", "разделить логику", "сериализатор не должен", даже если фреймворк не назван явно.
---

# Services и Selectors в Django

Этот навык описывает архитектурный подход для Django проекта: вся бизнес-логика живёт в двух слоях, `services` (запись и побочные эффекты) и `selectors` (чтение). Модели, view и сериализаторы остаются тонкими и предсказуемыми.

Подход взят из практики команды HackSoft (HackSoft Django Styleguide) и адаптирован под наши реалии: PostgreSQL под высокой нагрузкой, PgBouncer, тесты на pytest, типизация через ty.

## Зачем разделять

Django из коробки даёт несколько мест, куда можно положить логику: модель, менеджер, форма, сериализатор, view, signal handler. На маленьком проекте это удобно. На большом превращается в хаос: чтобы понять, что происходит при создании заказа, надо проверить шесть мест и помолиться.

Разделение даёт три понятных слоя:
- **Models** описывают форму данных и инварианты на уровне БД.
- **Selectors** отвечают на вопрос "как достать данные из БД для конкретного use case".
- **Services** отвечают на вопрос "что произойдёт в системе, если мы выполним эту бизнес-операцию".

Всё остальное (view, сериализатор, форма, admin, management command, Celery task, GraphQL resolver) это **транспорт**. Транспорт принимает входные данные, передаёт их в service или selector, получает результат и отдаёт пользователю. Транспорт не содержит бизнес-логики.

Выгоды от такого разделения:
- Бизнес-логика тестируется без HTTP. Тест сервиса это тест функции с аргументами, а не интеграционный тест с APIClient.
- Одна и та же логика переиспользуется в HTTP API, в admin, в management команде и в Celery таске без копипасты.
- Поиск кода по проекту становится очевидным: "где создаётся заказ" это `services/orders.py:create_order`. Не "может в модели, может в сериализаторе, может в signal".
- Замена транспорта (DRF на Ninja, REST на GraphQL) не требует переписывать бизнес-логику.
- Code review становится быстрым: смотришь только сервис, всё остальное декларативное.

## Когда не применять

Подход не бесплатный. Он добавляет файлы и слои там, где их раньше не было. Если проект:
- одноразовый прототип на пару недель,
- маленький CRUD без бизнес-правил (буквально только админка над таблицами),
- учебный пример из туториала,

то полное разделение оверхед. В таких случаях Django ORM плюс DRF ViewSet с дефолтным CRUD достаточно. Смысл слоёв появляется, когда:
- появляются нетривиальные бизнес-правила (статусы, переходы, расчёты, валидации между моделями),
- одна операция меняет несколько таблиц атомарно,
- логика вызывается из нескольких мест (HTTP, Celery, management команда),
- проект живёт больше года и в команде больше двух разработчиков.

## Структура приложения

Стандартное Django приложение `apps/orders/` под этот подход выглядит так:

```
apps/orders/
├── __init__.py
├── apps.py
├── models.py
├── selectors.py
├── services.py
├── api.py
├── serializers.py
├── tasks.py
├── signals.py
├── admin.py
├── urls.py
├── exceptions.py
└── migrations/
```

Что куда:
- `models.py` определения моделей. Поля, связи, простые свойства, минимальная валидация на уровне БД через constraints.
- `selectors.py` функции чтения. Принимают параметры фильтрации, возвращают QuerySet или его проекцию.
- `services.py` функции записи. Принимают входные данные, изменяют состояние БД, ставят задачи, отправляют события.
- `api.py` view, endpoints, ViewSet. Тонкие, только разбор запроса и вызов service/selector.
- `serializers.py` DRF сериализаторы. Только сериализация и десериализация. Без бизнес-логики.
- `tasks.py` Celery таски. Тонкие обёртки над сервисами.
- `signals.py` сигналы Django. Используются редко, только для технических нужд (логирование, аудит). Бизнес-логика в сигналы не идёт.
- `exceptions.py` доменные исключения, общие для приложения.

В большом приложении `services.py` и `selectors.py` могут стать пакетами:

```
apps/orders/services/
├── __init__.py
├── creation.py
├── fulfillment.py
├── refunds.py
└── cancellation.py
```

В `__init__.py` реэкспортируем публичные функции, чтобы `from apps.orders.services import create_order` продолжал работать.

## Тонкие модели

Модель описывает форму данных и инварианты, которые БД может проверить сама. Это всё.

### Что в модели должно быть

- Поля и их типы.
- Связи (`ForeignKey`, `ManyToManyField`, `OneToOneField`).
- `Meta` с индексами, уникальностью, упорядочиванием по умолчанию, constraints.
- `__str__` для удобства отладки и admin.
- Простые `@property` без обращения к БД и без побочных эффектов.

```python
from decimal import Decimal
from django.db import models


class Order(models.Model):
    class Status(models.TextChoices):
        DRAFT = "draft"
        PENDING = "pending"
        PAID = "paid"
        SHIPPED = "shipped"
        CANCELLED = "cancelled"
        REFUNDED = "refunded"

    customer = models.ForeignKey(
        "customers.Customer",
        on_delete=models.PROTECT,
        related_name="orders",
    )
    status = models.CharField(
        max_length=20,
        choices=Status.choices,
        default=Status.DRAFT,
    )
    total_amount = models.DecimalField(max_digits=12, decimal_places=2)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        indexes = [
            models.Index(fields=["customer", "-created_at"]),
            models.Index(fields=["status", "-created_at"]),
        ]
        constraints = [
            models.CheckConstraint(
                condition=models.Q(total_amount__gte=0),
                name="orders_total_amount_non_negative",
            ),
        ]

    def __str__(self) -> str:
        return f"Order #{self.pk} ({self.status})"

    @property
    def is_finalized(self) -> bool:
        return self.status in {self.Status.SHIPPED, self.Status.REFUNDED}
```

### Что в модели быть не должно

- Кастомные `save()` с побочными эффектами (отправка письма, постановка задачи, изменение других моделей).
- Методы вроде `order.process_payment()`, `order.cancel()`, `order.send_to_warehouse()`. Это бизнес-операции, их место в сервисе.
- Сложные методы выборки. `Order.objects.get_top_customers()` или `OrderManager.with_unpaid_invoices()` это селектор, его место в `selectors.py`.
- Кэширование на уровне instance. Если нужно денормализованное поле, делаем колонку и обновляем явно через сервис.
- Сигналы `post_save` с бизнес-эффектами.

### Почему именно так

Когда модель содержит бизнес-логику, возникают три проблемы.

**Скрытые побочные эффекты.** `order.save()` выглядит как простое сохранение, но если внутри отправляется письмо и ставится задача, поведение неочевидно. Любой код, который пишет `Order.objects.update(...)`, обходит эти эффекты молча. Bulk операции тоже их пропускают.

**Невозможность переиспользования.** Метод `order.cancel()` живёт на инстансе. Чтобы отменить заказ из management команды, надо сначала загрузить инстанс, потом дёрнуть метод. Если логика меняет несколько моделей, метод одной модели становится местом, где знание про чужие модели расползается по всему коду.

**Сложность тестирования.** Тест метода модели почти всегда требует реальной БД, иначе валидация и сохранение не работают. Тест сервисной функции часто можно сделать чище: фикстуры с моделями плюс прямой вызов функции.

### Свойства и методы это серая зона

`@property is_finalized` выше нормально: чистая функция от полей самой модели, без БД и побочных эффектов. Это упрощает код, который проверяет состояние.

Граница такая: если свойство делает запрос (`self.items.count()`, `self.customer.email`), оно пахнет N+1. Лучше превратить в селектор или передавать через `annotate`.

## Selectors

Селектор отвечает на вопрос "как получить данные для конкретного use case". Это функция, которая принимает параметры и возвращает данные.

### Сигнатура

```python
from django.db.models import QuerySet

from apps.orders.models import Order


def get_orders_for_customer(
    *,
    customer_id: int,
    statuses: list[str] | None = None,
) -> QuerySet[Order]:
    """
    Возвращает заказы клиента с опциональной фильтрацией по статусам.

    :param customer_id: идентификатор клиента.
    :param statuses: список статусов для фильтрации, None означает все статусы.
    :return: queryset заказов, отсортированный по дате создания убыванием.
    """
    base_queryset = (
        Order.objects
        .filter(customer_id=customer_id)
        .select_related("customer")
        .order_by("-created_at")
    )
    if statuses is not None:
        base_queryset = base_queryset.filter(status__in=statuses)
    return base_queryset
```

Правила:
- Все аргументы keyword-only через `*`. Это защищает от перепутанного порядка при чтении вызова.
- Аннотации типов обязательны.
- Селектор возвращает `QuerySet`, не `list`. QuerySet ленивый, потребитель сам решит когда материализовать. Список селектор возвращает только если так удобнее по контексту (например, агрегат для отчёта).
- Селектор не делает запись.
- Селектор не вызывает другие сервисы.
- Селектор не делает HTTP запросы и не работает с кэшем (кэш живёт в сервисе или отдельном слое, см. ниже).

### Что возвращать: QuerySet, list, dict

QuerySet универсальнее. Транспорт сам решит:
- DRF сериализатор пройдёт по QuerySet и сериализует.
- Admin отдаст QuerySet в свою пагинацию.
- Management команда сделает `iterator()` или `list()` по необходимости.

Возвращать `list` стоит, когда селектор делает агрегацию или сложную трансформацию, и QuerySet уже не отражает результат:

```python
def get_revenue_by_month(*, year: int) -> list[dict]:
    """
    Считает выручку по месяцам за указанный год.

    :param year: год для расчёта.
    :return: список словарей с полями month и revenue, отсортированный по месяцу.
    """
    aggregated_data = (
        Order.objects
        .filter(status="paid", created_at__year=year)
        .annotate(order_month=TruncMonth("created_at"))
        .values("order_month")
        .annotate(revenue=Sum("total_amount"))
        .order_by("order_month")
    )
    return list(aggregated_data)
```

### Селекторы не цепляются друг с другом

Анти-паттерн: селектор внутри селектора через ORM. Когда `get_orders_for_customer` вызывает `get_active_customers`, и тот ещё что-то, тесты становятся хрупкими и план запросов непредсказуемым.

Правильно: каждый селектор это самостоятельный запрос, оптимизированный под свой use case. Если есть общая часть, выноси её в private хелпер в том же файле:

```python
def _base_active_orders_queryset() -> QuerySet[Order]:
    """
    Возвращает базовый queryset активных заказов с предзагруженными связями.

    :return: queryset активных заказов.
    """
    return (
        Order.objects
        .exclude(status__in=["cancelled", "refunded"])
        .select_related("customer")
    )


def get_active_orders_for_customer(*, customer_id: int) -> QuerySet[Order]:
    """
    Возвращает активные заказы клиента.

    :param customer_id: идентификатор клиента.
    :return: queryset активных заказов клиента.
    """
    return _base_active_orders_queryset().filter(customer_id=customer_id)


def get_recent_active_orders(*, days: int) -> QuerySet[Order]:
    """
    Возвращает активные заказы за последние N дней.

    :param days: количество дней назад.
    :return: queryset активных заказов за период.
    """
    cutoff_date = timezone.now() - timedelta(days=days)
    return _base_active_orders_queryset().filter(created_at__gte=cutoff_date)
```

Хелпер с `_` обозначает внутреннюю кухню модуля. Он переиспользуется внутри файла, но не торчит наружу.

### Что не селектор

- Селектор не отдаёт DTO, спроектированный под конкретный API. Это работа сериализатора.
- Селектор не считает производные значения, которые нужны только в одном представлении. Если в админке нужен особый CSV экспорт, отдельная функция в `selectors.py` оправдана только если она переиспользуется или содержит сложный запрос. Простую трансформацию делает транспорт.
- Селектор не является хранилищем кэша. Если результат должен кэшироваться, есть два пути: либо отдельный кэширующий слой над селектором, либо сервис, который читает кэш и при промахе зовёт селектор.

## Services

Сервис это функция, которая выполняет бизнес-операцию. Он меняет состояние БД, может ставить задачи в Celery, отправлять события, дёргать внешние API. Сервис это центр, в котором происходит работа.

### Сигнатура и базовая структура

```python
from django.db import transaction

from apps.orders.models import Order
from apps.orders.exceptions import OrderValidationError
from apps.orders.tasks import send_order_confirmation_email
from apps.products.selectors import get_product_for_update


@transaction.atomic
def create_order(
    *,
    customer_id: int,
    items: list[dict],
    promo_code: str | None = None,
) -> Order:
    """
    Создаёт заказ, резервирует склад и ставит задачу на отправку подтверждения.

    :param customer_id: идентификатор клиента.
    :param items: список позиций с product_id и quantity.
    :param promo_code: опциональный промокод для скидки.
    :return: созданный заказ.
    :raises OrderValidationError: если корзина пуста или товара недостаточно.
    """
    if not items:
        raise OrderValidationError("Cart is empty")

    new_order = Order.objects.create(customer_id=customer_id, status=Order.Status.DRAFT)
    total_amount = _add_items_to_order(order=new_order, items=items)

    if promo_code:
        total_amount = _apply_promo_code(order=new_order, code=promo_code, subtotal=total_amount)

    new_order.total_amount = total_amount
    new_order.status = Order.Status.PENDING
    new_order.save(update_fields=["total_amount", "status"])

    transaction.on_commit(
        lambda: send_order_confirmation_email.delay(order_id=new_order.id)
    )
    return new_order
```

Правила:
- Keyword-only аргументы через `*`.
- Аннотации типов и docstring с `:raises:` для каждого исключения, которое сервис может бросить.
- Атомарность через `@transaction.atomic` или явный `with transaction.atomic():` блок.
- Постановка задач Celery через `transaction.on_commit`. Это критично: если поставить `delay()` напрямую внутри транзакции и она откатится, задача всё равно выполнится с несуществующим id.
- Сервис возвращает результат операции (созданный объект, обновлённый объект, дельту, что угодно осмысленное). Не возвращает `None`, если есть что вернуть.
- Бизнес-исключения это собственные классы из `exceptions.py`, не голые `ValueError`.

### Service vs ModelManager.create

Стандартный `Model.objects.create(...)` это просто INSERT. Это не сервис.

Сервис нужен, когда:
- операция требует валидации сложнее, чем одно поле,
- меняется больше одной модели,
- есть побочные эффекты (письма, задачи, события, внешние API),
- атомарность критична,
- бизнес-правило может эволюционировать (новые поля, статусы, проверки).

Если объект создаётся одним INSERT и больше ничего не происходит, ORM `create` достаточно. Не плоди сервис ради сервиса.

### Сервис меняет несколько моделей

Сервис это идеальное место для координации между моделями. Он видит все участвующие сущности и держит их согласованными.

```python
@transaction.atomic
def cancel_order(*, order_id: int, reason: str) -> Order:
    """
    Отменяет заказ, возвращает резерв на склад, ставит задачу на возврат оплаты.

    :param order_id: идентификатор заказа.
    :param reason: причина отмены для аудита.
    :return: обновлённый заказ.
    :raises OrderValidationError: если заказ нельзя отменить в текущем статусе.
    """
    target_order = (
        Order.objects
        .select_for_update()
        .select_related("customer")
        .get(pk=order_id)
    )

    if target_order.status not in {Order.Status.PENDING, Order.Status.PAID}:
        raise OrderValidationError(
            f"Cannot cancel order in status {target_order.status}"
        )

    for order_item in target_order.items.select_for_update():
        _release_stock_reservation(product_id=order_item.product_id, quantity=order_item.quantity)

    target_order.status = Order.Status.CANCELLED
    target_order.cancellation_reason = reason
    target_order.save(update_fields=["status", "cancellation_reason"])

    if target_order.status == Order.Status.PAID:
        transaction.on_commit(
            lambda: process_refund.delay(order_id=target_order.id)
        )

    return target_order
```

Что важно:
- `select_for_update` блокирует строку до конца транзакции. Это защищает от гонок: пока мы отменяем заказ, никто не может параллельно его отгрузить.
- Логика переходов статусов живёт в сервисе. Модель про это не знает.
- Возврат оплаты идёт через Celery после коммита. Внешний API не должен дёргаться внутри транзакции (см. раздел про PgBouncer в navыке django-orm-expert).

### Сервис вызывает другие сервисы

Это нормально. Большая операция собирается из маленьких:

```python
@transaction.atomic
def checkout_cart(*, cart_id: int, payment_method_id: int) -> Order:
    """
    Превращает корзину в оплаченный заказ.

    :param cart_id: идентификатор корзины.
    :param payment_method_id: идентификатор метода оплаты.
    :return: оплаченный заказ.
    :raises CartValidationError: если корзина пуста или не валидна.
    :raises PaymentError: если оплата не прошла.
    """
    target_cart = get_cart_with_items(cart_id=cart_id)
    new_order = create_order(
        customer_id=target_cart.customer_id,
        items=[item.to_dict() for item in target_cart.items.all()],
    )
    paid_order = process_payment(order_id=new_order.id, method_id=payment_method_id)
    clear_cart(cart_id=cart_id)
    return paid_order
```

Внешний `@transaction.atomic` гарантирует, что либо всё прошло, либо ничего не сохранилось. Внутренние вызовы `@transaction.atomic` создают savepoints и работают корректно вложенно.

### Сервис не вызывает view, сериализатор и transport

Поток зависимостей строго направленный:

```
view  →  service  →  models
view  →  selector →  models
service →  selector →  models
service →  service →  models
```

Селектор не зовёт сервис. Сервис не импортирует view или сериализатор. Модель не знает ничего из верхних слоёв.

Если возникает соблазн вызвать сериализатор из сервиса (например, чтобы получить готовый JSON для отправки в очередь), это знак, что нужен отдельный модуль или функция конвертации. Сериализатор DRF это часть транспорта, его привязка к HTTP может ломаться вне HTTP контекста.

### Idempotency и повторы

Сервисы, которые вызываются из Celery или из внешних webhook'ов, должны быть идемпотентными. Вызвать дважды то же самое не должно ломать систему.

Способы:
- `update_or_create` с уникальным ключом операции.
- `bulk_create(update_conflicts=True)` с уникальным индексом.
- Явная проверка "уже сделано" перед действием.
- Запись в таблицу `processed_events(event_id)` с уникальным индексом и отказ при повторе.

```python
@transaction.atomic
def apply_payment_webhook(*, webhook_event_id: str, order_id: int, amount: Decimal) -> Order:
    """
    Применяет событие оплаты от платёжного провайдера. Идемпотентно по event_id.

    :param webhook_event_id: уникальный id события от провайдера.
    :param order_id: идентификатор заказа.
    :param amount: сумма платежа.
    :return: обновлённый заказ.
    """
    _, was_created = ProcessedWebhookEvent.objects.get_or_create(
        event_id=webhook_event_id,
        defaults={"received_at": timezone.now()},
    )
    if not was_created:
        return Order.objects.get(pk=order_id)

    target_order = Order.objects.select_for_update().get(pk=order_id)
    target_order.paid_amount = amount
    target_order.status = Order.Status.PAID
    target_order.save(update_fields=["paid_amount", "status"])
    return target_order
```

## Тонкие view

View это транспорт. Он:
- разбирает HTTP запрос (заголовки, query, body),
- валидирует входные данные через сериализатор,
- зовёт сервис или селектор,
- сериализует результат,
- возвращает HTTP ответ.

Никакой бизнес-логики в view нет.

### DRF ViewSet тонкий

```python
from rest_framework import status, viewsets
from rest_framework.decorators import action
from rest_framework.response import Response

from apps.orders import selectors, services
from apps.orders.serializers import (
    OrderCreateRequestSerializer,
    OrderDetailSerializer,
    OrderListSerializer,
)


class OrderViewSet(viewsets.GenericViewSet):
    permission_classes = [IsAuthenticated]

    def list(self, request):
        orders_queryset = selectors.get_orders_for_customer(
            customer_id=request.user.id,
        )
        page = self.paginate_queryset(orders_queryset)
        response_serializer = OrderListSerializer(page, many=True)
        return self.get_paginated_response(response_serializer.data)

    def retrieve(self, request, pk: int):
        target_order = selectors.get_order_for_customer(
            customer_id=request.user.id,
            order_id=int(pk),
        )
        response_serializer = OrderDetailSerializer(target_order)
        return Response(response_serializer.data)

    def create(self, request):
        request_serializer = OrderCreateRequestSerializer(data=request.data)
        request_serializer.is_valid(raise_exception=True)

        try:
            new_order = services.create_order(
                customer_id=request.user.id,
                items=request_serializer.validated_data["items"],
                promo_code=request_serializer.validated_data.get("promo_code"),
            )
        except OrderValidationError as validation_exception:
            return Response(
                {"detail": str(validation_exception)},
                status=status.HTTP_400_BAD_REQUEST,
            )

        response_serializer = OrderDetailSerializer(new_order)
        return Response(response_serializer.data, status=status.HTTP_201_CREATED)

    @action(detail=True, methods=["post"])
    def cancel(self, request, pk: int):
        request_serializer = OrderCancelRequestSerializer(data=request.data)
        request_serializer.is_valid(raise_exception=True)

        try:
            cancelled_order = services.cancel_order(
                order_id=int(pk),
                reason=request_serializer.validated_data["reason"],
            )
        except OrderValidationError as validation_exception:
            return Response(
                {"detail": str(validation_exception)},
                status=status.HTTP_400_BAD_REQUEST,
            )

        response_serializer = OrderDetailSerializer(cancelled_order)
        return Response(response_serializer.data)
```

Видно, что в view нет ничего, кроме разбора запроса и вызова. Любую логику видно по имени функции в `services` или `selectors`.

### Без ModelViewSet

`ModelViewSet` соблазнительно прост: на одной строчке полный CRUD. Но он смешивает транспорт, сериализатор и логику записи. На реальных бизнес-правилах быстро превращается в свалку из переопределённых методов.

В этом подходе мы не используем `ModelViewSet`. Берём `GenericViewSet` или `ViewSet` и вручную вызываем сервисы. Скучнее на старте, понятнее на дистанции.

### Permissions это часть транспорта

Проверка "имеет ли пользователь право вызвать этот endpoint" живёт в `permission_classes`. Это часть HTTP контракта. Сервис об этом не знает: ему передают `customer_id`, и он работает с этим клиентом.

Бизнес-правила вроде "только владелец заказа может его отменить" могут жить в двух местах:
- если правило строго привязано к HTTP, в `permissions.py` рядом с view,
- если правило фундаментально для домена и нужно везде, проверка идёт в сервисе через явный аргумент типа `acting_user_id` и сравнение в коде сервиса.

Чаще правильно второе: HTTP это один из способов вызвать операцию, и доменные правила должны выполняться при любом способе.

## Тонкие сериализаторы

Сериализатор это функция перевода между нашим внутренним представлением и форматом транспорта. Только это.

### Что в сериализаторе должно быть

- Поля и их типы.
- Простая валидация одного поля (длина, формат, диапазон).
- Перевод типов (decimal в строку, datetime в isoformat).
- Вложенные сериализаторы для связанных объектов.

### Что в сериализаторе быть не должно

- Создание объектов в БД через `create()` или `update()` метод.
- Бизнес-валидация, которая трогает другие модели или БД.
- Побочные эффекты (отправка письма, постановка задачи).
- Сложная логика вычисления полей.

#### Антипаттерн: толстый ModelSerializer

```python
class OrderCreateSerializer(serializers.ModelSerializer):
    class Meta:
        model = Order
        fields = ["customer", "items", "promo_code"]

    def validate(self, attrs):
        for item in attrs["items"]:
            product = Product.objects.get(pk=item["product_id"])
            if product.stock < item["quantity"]:
                raise serializers.ValidationError("Not enough stock")
        return attrs

    def create(self, validated_data):
        items = validated_data.pop("items")
        order = Order.objects.create(**validated_data)
        for item in items:
            OrderItem.objects.create(order=order, **item)
        send_order_confirmation_email.delay(order_id=order.id)
        return order
```

Это бизнес-логика, размазанная по сериализатору. Проблемы:
- Логика создания недоступна вне HTTP. Чтобы создать заказ из management команды, придётся либо инстанцировать сериализатор без `request`, либо дублировать код.
- Тесты на эту логику требуют APIClient или ручной инстанциации сериализатора.
- Валидация в `validate()` делает запрос в БД на каждый item. На большом запросе это N запросов.
- Никакой атомарности: если письмо упадёт, заказ уже создан.

#### Правильно: request/response сериализаторы

Разделяем сериализатор на два: request для приёма и валидации формата запроса, response для отрисовки данных в ответ.

```python
class OrderCreateRequestSerializer(serializers.Serializer):
    items = serializers.ListField(
        child=serializers.DictField(),
        min_length=1,
        max_length=100,
    )
    promo_code = serializers.CharField(max_length=50, required=False, allow_blank=True)


class OrderItemResponseSerializer(serializers.Serializer):
    id = serializers.IntegerField()
    product_id = serializers.IntegerField()
    quantity = serializers.IntegerField()
    unit_price = serializers.DecimalField(max_digits=10, decimal_places=2)


class OrderDetailSerializer(serializers.Serializer):
    id = serializers.IntegerField()
    status = serializers.CharField()
    total_amount = serializers.DecimalField(max_digits=12, decimal_places=2)
    created_at = serializers.DateTimeField()
    items = OrderItemResponseSerializer(many=True)
```

Не `ModelSerializer`, а обычный `Serializer`. Поля описываются явно. Это:
- независимость от модели: меняем БД без поломки контракта API,
- явный список полей: никаких "случайно вылез internal_notes наружу",
- никакой логики в сериализаторе: его функция только перевод формата,
- никакой связи с HTTP контекстом, можно использовать вне view.

Бизнес-валидация (хватает ли товара, корректен ли промокод) живёт в сервисе, не в сериализаторе. Сериализатор проверяет только что `quantity` это положительное целое.

### Inline валидация одного поля это нормально

```python
class OrderCreateRequestSerializer(serializers.Serializer):
    items = serializers.ListField(child=serializers.DictField(), min_length=1)
    promo_code = serializers.CharField(max_length=50, required=False, allow_blank=True)

    def validate_promo_code(self, code_value: str) -> str:
        """
        Проверяет формат промокода.

        :param code_value: значение промокода.
        :return: нормализованное значение в верхнем регистре.
        """
        if code_value and not code_value.isalnum():
            raise serializers.ValidationError("Only alphanumeric characters allowed")
        return code_value.upper()
```

Такая валидация чисто синтаксическая. Она про формат, не про бизнес-смысл. Существование промокода в БД и его срок действия проверяет сервис.

## Где живут типичные вещи

### Кэш

Кэш это техническая забота. Хорошее место это отдельный тонкий слой над селектором, либо явное использование внутри сервиса.

```python
def get_top_products_cached() -> list[dict]:
    """
    Возвращает топ продуктов с кэшированием на 5 минут.

    :return: список словарей с данными топ продуктов.
    """
    cached_value = cache.get("top_products")
    if cached_value is not None:
        return cached_value

    fresh_value = list(selectors.get_top_products())
    cache.set("top_products", fresh_value, timeout=300)
    return fresh_value
```

Можно положить такую обёртку либо в `selectors.py` рядом с базовым селектором, либо в отдельный `caches.py`. Главное, что чистый селектор остаётся без знания про кэш и его легко тестировать.

### Сигналы Django

Сигналы это легко превращающийся в магию инструмент. Бизнес-логике в `post_save` не место. Когда что-то должно случиться после создания заказа, явно делаем это в сервисе.

Сигналы оправданы для:
- технических задач (логирование, аудит, метрики),
- интеграции со сторонним кодом, который не знает про наши сервисы (например, `django-allauth` шлёт сигналы при регистрации).

Если ловишь себя на том, что пишешь signal handler, который делает реальную работу, остановись и подумай, не должен ли это быть прямой вызов сервиса.

### Celery таски

Celery таска это тонкая обёртка над сервисом:

```python
from celery import shared_task

from apps.orders import services


@shared_task(bind=True, max_retries=3, default_retry_delay=60)
def process_order_payment(self, order_id: int) -> None:
    """
    Celery таска оплаты заказа. Идемпотентна.

    :param self: контекст celery таски.
    :param order_id: идентификатор заказа.
    :return: None.
    """
    try:
        services.process_payment(order_id=order_id)
    except TemporaryPaymentError as temporary_error:
        raise self.retry(exc=temporary_error)
```

Таска отвечает за интеграцию с Celery (retry, очередь, приоритет). Бизнес-логика оплаты в сервисе. Это разделение позволяет тестировать сервис без Celery и переиспользовать его из синхронного кода.

### Management команды

```python
from django.core.management.base import BaseCommand

from apps.orders import services


class Command(BaseCommand):
    help = "Cancels stale draft orders older than the given number of days."

    def add_arguments(self, parser) -> None:
        parser.add_argument("--days", type=int, default=7)

    def handle(self, *args, **options) -> None:
        days_threshold = options["days"]
        total_cancelled = services.cancel_stale_draft_orders(days=days_threshold)
        self.stdout.write(f"Cancelled {total_cancelled} stale orders")
```

Команда тонкая. Парсинг аргументов и вызов сервиса.

### Admin actions

То же самое: action в admin это вызов сервиса, не отдельная реализация.

```python
@admin.action(description="Cancel selected orders")
def cancel_orders_action(modeladmin, request, queryset):
    """
    Admin action для отмены выделенных заказов через сервисный слой.

    :param modeladmin: экземпляр ModelAdmin.
    :param request: HTTP запрос.
    :param queryset: queryset выделенных заказов.
    :return: None.
    """
    for target_order in queryset:
        try:
            services.cancel_order(
                order_id=target_order.id,
                reason="Cancelled from admin",
            )
        except OrderValidationError as validation_exception:
            messages.error(request, f"Order {target_order.id}: {validation_exception}")
```

## Исключения

Бизнес-исключения свои, не голый `ValueError`. Это позволяет ловить их адресно в транспорте и переводить в нужный HTTP статус.

```python
class OrderValidationError(Exception):
    """Исключение валидации бизнес-правил заказа."""


class OrderNotFoundError(Exception):
    """Заказ не найден или недоступен запрашивающему."""


class InsufficientStockError(OrderValidationError):
    """На складе недостаточно товара для выполнения заказа."""
```

Иерархия осмысленная. Транспорт может ловить общий `OrderValidationError` и переводить в 400, а селектор бросает `OrderNotFoundError` в 404.

DRF позволяет повесить глобальный exception handler, который переводит доменные исключения в HTTP ответы. Это ещё один способ держать view тонкими: в сервисе бросаем `OrderValidationError`, в settings указан handler, который переводит его в `{"detail": "..."}` и 400. View вообще не знает про try/except.

```python
def domain_exception_handler(exception_value, context):
    """
    Глобальный DRF handler для перевода доменных исключений в HTTP ответы.

    :param exception_value: пойманное исключение.
    :param context: контекст DRF.
    :return: Response или None если исключение не наше.
    """
    if isinstance(exception_value, OrderNotFoundError):
        return Response({"detail": str(exception_value)}, status=404)
    if isinstance(exception_value, OrderValidationError):
        return Response({"detail": str(exception_value)}, status=400)
    return drf_exception_handler(exception_value, context)
```

В `settings.REST_FRAMEWORK = {"EXCEPTION_HANDLER": "..."}`. После этого view становится ещё тоньше:

```python
def create(self, request):
    request_serializer = OrderCreateRequestSerializer(data=request.data)
    request_serializer.is_valid(raise_exception=True)

    new_order = services.create_order(
        customer_id=request.user.id,
        items=request_serializer.validated_data["items"],
        promo_code=request_serializer.validated_data.get("promo_code"),
    )

    response_serializer = OrderDetailSerializer(new_order)
    return Response(response_serializer.data, status=status.HTTP_201_CREATED)
```

## Тестирование

Слои дают чистую пирамиду тестов.

### Тесты сервисов

Чистая функция: подаём вход, проверяем выход и побочные эффекты. Тест с `pytest.mark.django_db`, без HTTP.

```python
@pytest.mark.django_db
def test_create_order_creates_order_with_items(regular_user) -> None:
    """
    Arrange: продукт с остатком на складе и корзина с двумя позициями
    Act: вызов сервиса создания заказа
    Assert: заказ создан со статусом pending, итог посчитан, остаток уменьшился
    """
    target_product = ProductFactory(stock=10, price=Decimal("100.00"))
    cart_items = [{"product_id": target_product.id, "quantity": 2}]

    created_order = services.create_order(
        customer_id=regular_user.id,
        items=cart_items,
    )

    assert created_order.status == Order.Status.PENDING
    assert created_order.total_amount == Decimal("200.00")
    target_product.refresh_from_db()
    assert target_product.stock == 8
```

Тест короткий и стабильный. Не зависит от URL роутинга, аутентификации, формата сериализатора. Если роуты переедут с DRF на Ninja, тест продолжит работать.

### Тесты селекторов

Аналогично: подаём данные через фабрики, вызываем селектор, проверяем результат и количество запросов.

```python
@pytest.mark.django_db
def test_get_orders_for_customer_returns_only_owned_orders(
    regular_user,
    django_assert_num_queries,
) -> None:
    """
    Arrange: три заказа целевого пользователя и два чужих
    Act: вызов селектора по id целевого пользователя
    Assert: получены ровно три заказа, запросов не больше двух
    """
    own_orders = OrderFactory.create_batch(3, customer_id=regular_user.id)
    OrderFactory.create_batch(2)

    with django_assert_num_queries(2):
        fetched_orders = list(
            selectors.get_orders_for_customer(customer_id=regular_user.id)
        )

    assert {order.id for order in fetched_orders} == {order.id for order in own_orders}
```

### Тесты view

Тонкие тесты на контракт API: статус, форма ответа, авторизация. Бизнес-логику здесь не дублируем, она проверяется в сервисе.

```python
@pytest.mark.django_db
def test_create_order_endpoint_returns_201(authenticated_client, mocker) -> None:
    """
    Arrange: сервис create_order замокан, payload валидный
    Act: POST на endpoint создания заказа
    Assert: статус 201, сервис вызван с правильными аргументами
    """
    create_order_mock = mocker.patch(
        "apps.orders.api.services.create_order",
        return_value=OrderFactory.build(id=42),
    )
    request_payload = {"items": [{"product_id": 1, "quantity": 1}]}

    response = authenticated_client.post("/api/orders/", data=request_payload, format="json")

    assert response.status_code == status.HTTP_201_CREATED
    create_order_mock.assert_called_once()
```

В тестах view сервис мокается, потому что мы тестируем именно транспорт, а не повторно бизнес-логику.

## Антипаттерны

- **Толстая модель.** Метод `order.process_refund()` в `models.py`. Перенести в `services.py` как `process_refund(order_id=...)`.
- **Логика в `save()`.** Сигнатура `save` подразумевает только сохранение. Любая дополнительная работа в `save` это магия для всех bulk операций и ORM `update`.
- **Логика в `signals.py`.** Особенно `post_save`, который что-то отправляет. Сложно отлаживать, обходится bulk операциями.
- **Толстый ModelSerializer.** `create()` или `update()` метод сериализатора с реальной работой. Бизнес-операция должна быть сервисом.
- **Логика в view.** Переходы статусов, расчёты, цепочки вызовов прямо в view. Перенести в сервис.
- **Селектор зовёт сервис.** Сломанный поток зависимостей. Селектор только читает, сервис только пишет.
- **Сервис зовёт сериализатор.** Сериализатор это часть транспорта. Если нужна структура данных, сделай отдельный конвертер или dataclass.
- **Несколько мест для одной операции.** Создание заказа через сериализатор `OrderSerializer.create`, через метод модели `Order.create_with_items`, через фабрику `OrderFactory.create_real`. Должно быть одно место: сервис.
- **`Order.objects.update(status=...)` вместо сервиса.** Если изменение статуса это бизнес-операция (триггерит письма, задачи, проверки), голый `update` обходит логику. Только сервис.
- **HTTP вызов внутри `transaction.atomic`.** Сетевая задержка держит соединение из пула PgBouncer открытым на секунды. Внешние API через `transaction.on_commit` плюс Celery.
- **`ValueError` как бизнес-исключение.** Сложно ловить адресно. Свои классы из `exceptions.py`.

## Чек-лист перед коммитом

- [ ] Бизнес-операция вызывается из одного места: сервиса.
- [ ] Сервис обёрнут в `transaction.atomic`, постановки в Celery идут через `on_commit`.
- [ ] Селектор не делает запись и не зовёт сервис.
- [ ] View разбирает запрос, валидирует через сериализатор, зовёт сервис или селектор, возвращает ответ. Точка.
- [ ] Сериализатор не делает запросы в БД и не имеет `create`/`update` методов с логикой.
- [ ] Модель содержит только описание данных, инварианты БД и простые свойства без побочек.
- [ ] Бизнес-исключения свои, не `ValueError` или `Exception`.
- [ ] Аргументы сервиса и селектора keyword-only.
- [ ] Все публичные функции имеют аннотации типов и docstring с `:param`, `:return`, `:raises`.
- [ ] На сервис написан тест, который вызывает его напрямую без HTTP.
- [ ] На селектор написан тест с `django_assert_num_queries`.
- [ ] Тест view проверяет контракт, не дублирует бизнес-логику.

## Краткая шпаргалка

| Где должна жить логика | Куда положить |
|---|---|
| Создание объекта с побочными эффектами | `services.py` |
| Простой INSERT без логики | `Model.objects.create` напрямую в сервисе |
| Сложный SELECT с фильтрами и JOIN | `selectors.py` |
| Простой `Model.objects.get(pk=...)` | напрямую в сервисе |
| Атомарное обновление поля без логики | `Model.objects.filter().update()` в сервисе |
| Переход статуса с проверками | `services.py` |
| Расчёт производного значения для вывода | в сериализаторе через `SerializerMethodField`, если просто; в селекторе через `annotate`, если сложно |
| Кэширование результата | обёртка над селектором или отдельный `caches.py` |
| Реакция на событие БД (логирование, аудит) | `signals.py` |
| Реакция на событие БД (бизнес-логика) | прямой вызов сервиса в нужном месте |
| Авторизация по HTTP контексту | `permissions.py` |
| Доменное правило доступа | в сервисе через явный аргумент |
| Создание из management команды | вызов сервиса из `Command.handle` |
| Создание из admin action | вызов сервиса из admin action |
| Создание из Celery таски | вызов сервиса из таски |
| Перевод доменного исключения в HTTP | глобальный DRF exception handler |
| Сериализация для API ответа | `serializers.py` (Response serializer) |
| Валидация формата запроса | `serializers.py` (Request serializer) |
| Бизнес-валидация (другие модели, БД) | в сервисе |
| Идемпотентность таски | в сервисе через таблицу processed events |
