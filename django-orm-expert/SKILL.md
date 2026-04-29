---
name: django-orm-expert
description: Используй этот навык всегда для написания и ревью Django ORM запросов на PostgreSQL. Покрывает QuerySet, select_related, prefetch_related, annotate, alias, aggregate, GROUP BY и HAVING, оконные функции, F и Q выражения, database functions включая JSONObject, Subquery, Exists, ArraySubquery, фильтрацию через subquery вместо списка id, bulk_create и bulk_update через генераторы и itertools.islice или batched, batch обработку через while True, транзакции, select_for_update, raw SQL, индексы (B-tree, GIN, BRIN, partial, covering), Postgres специфику (ArrayField, JSONField, ArrayAgg, full-text, trigram), keyset пагинацию для PgBouncer с отключёнными server-side cursors, COPY, upsert, EXPLAIN. Подключай при словах оптимизируй запрос, ORM, N+1, annotate, alias, aggregate, GROUP BY, Window, F-выражение, bulk, batch, генератор, islice, select_for_update, JSON поле, массив, индекс, EXPLAIN, PgBouncer, ArraySubquery, ETL.
---

# Django ORM expert

Этот навык учит писать оптимизированные запросы для Django проекта на PostgreSQL. Уровень expert backend разработчика. Цель: запросы к таблицам в десятки и сотни миллионов строк, массовая запись, корректная работа под конкурентной нагрузкой.

База: Django 5.1+, PostgreSQL 14+. Все примеры используют Postgres-специфичные возможности там, где это даёт выигрыш.

Инфраструктурный контекст проекта:
- Между Django и Postgres стоит PgBouncer в режиме transaction pooling.
- Серверные курсоры отключены через `DISABLE_SERVER_SIDE_CURSORS = True` в настройках базы.
- Всё, что требует состояния соединения между запросами (server-side cursors, prepared statements, `LISTEN/NOTIFY`, session-level advisory locks), либо не работает, либо работает непредсказуемо.

Это влияет на выбор стратегии стриминга и на работу с длинными запросами. Подробности в разделах "Стриминг больших таблиц" и "Connection management".

## Базовые принципы оптимизации

1. ORM это удобный построитель SQL. Думаем сначала про SQL, потом про ORM.
2. Каждый запрос измеряется. Ничего не оптимизируется по интуиции. Используем `EXPLAIN ANALYZE`, `django-debug-toolbar`, `django-silk` или собственный middleware с `connection.queries`.
3. Выбираем минимум данных. Лишние столбцы, лишние строки и лишние JOIN это лишняя память, сеть и работа БД.
4. Запрос должен попадать в индекс. Если нет, либо строим индекс, либо переписываем запрос.
5. ORM запросы lazy. Запрос уходит в БД при итерации, срезе с шагом, `len()`, `bool()`, `list()`, `repr()`, сериализации.
6. Любая операция, которая делает запрос внутри цикла, это потенциальный N+1.
7. Всё, что меняет состояние, должно быть атомарным или явно неатомарным с пониманием почему.
8. Чтение и запись миллионов строк это не одна транзакция и не один запрос. Это всегда батчи.
9. ORM не закрывает все случаи. Когда выгоднее raw SQL или хранимка, мы пишем raw SQL или хранимку.

## Анатомия QuerySet

QuerySet это lazy объект. Он строит SQL и кэширует результат в памяти после первой материализации.

### Когда запрос уходит в БД

```python
queryset = Order.objects.filter(status="paid")

list(queryset)
bool(queryset)
len(queryset)
queryset.exists()
queryset.count()
queryset[5]
queryset[:10:1]

for order in queryset:
    pass

repr(queryset)
```

### Когда запрос не уходит

```python
queryset = Order.objects.filter(status="paid")
filtered_queryset = queryset.exclude(amount=0)
ordered_queryset = filtered_queryset.order_by("-created_at")
sliced_queryset = ordered_queryset[:100]
```

Цепочка `filter().exclude().order_by()[:100]` это один SQL запрос. Он уйдёт в БД при материализации последнего queryset.

### Кэш результатов

После материализации QuerySet хранит результат в `_result_cache`. Повторная итерация не делает запрос. Это важно: если код несколько раз обходит один queryset, не клонируй его.

```python
orders_queryset = Order.objects.filter(status="paid")
list(orders_queryset)
list(orders_queryset)
```

Второй `list()` не делает запрос. А вот так делает дважды:

```python
list(Order.objects.filter(status="paid"))
list(Order.objects.filter(status="paid"))
```

### Когда кэш не работает

`exists()`, `count()`, `iterator()`, `update()`, `delete()` обходят кэш. `iterator()` вообще не наполняет кэш и идёт по строкам через server-side cursor.

## select_related и prefetch_related

Самый частый источник N+1. Правила простые.

### select_related для ForeignKey и OneToOne

Делает `INNER JOIN` (или `LEFT JOIN` для nullable) и забирает все связанные поля одним запросом.

```python
orders_queryset = (
    Order.objects
    .select_related("customer", "shipping_address", "billing_address")
    .filter(status="paid")
)

for order in orders_queryset:
    customer_email = order.customer.email
    shipping_city = order.shipping_address.city
```

Цепочки через двойное подчёркивание для глубоких связей:

```python
orders_queryset = Order.objects.select_related(
    "customer__company",
    "customer__company__country",
)
```

Каждый уровень добавляет JOIN. Слишком много JOIN ухудшает план запроса. На пяти и более JOIN PostgreSQL может перейти в `geqo` режим планирования и план становится непредсказуемым. Лучше разбить.

### prefetch_related для обратных связей и ManyToMany

Делает отдельный запрос для каждой связи и склеивает в Python.

```python
orders_queryset = (
    Order.objects
    .prefetch_related("items", "tags")
    .filter(status="paid")
)

for order in orders_queryset:
    for order_item in order.items.all():
        item_total = order_item.quantity * order_item.unit_price
```

`order.items.all()` использует prefetch и не делает запрос.

### Кастомизация через Prefetch

Когда нужно отфильтровать или отсортировать связанные объекты, использовать другой queryset, или сохранить результат в отдельный атрибут, идём через `Prefetch`.

```python
from django.db.models import Prefetch

active_items_prefetch = Prefetch(
    "items",
    queryset=OrderItem.objects.filter(is_cancelled=False).select_related("product"),
    to_attr="active_items",
)

orders_queryset = Order.objects.prefetch_related(active_items_prefetch)

for order in orders_queryset:
    for active_item in order.active_items:
        product_name = active_item.product.name
```

`to_attr` важен: без него prefetch перезаписывает менеджер связи и `order.items.all()` начнёт возвращать только активные. С `to_attr` оригинальный менеджер не трогается.

### Вложенный prefetch с select_related внутри

```python
items_with_product = Prefetch(
    "items",
    queryset=OrderItem.objects.select_related("product__category"),
)

orders_queryset = (
    Order.objects
    .select_related("customer")
    .prefetch_related(items_with_product)
)
```

### Когда prefetch_related хуже select_related

Если связь это `ForeignKey` и связанных объектов мало, `select_related` всегда лучше. `prefetch_related` имеет смысл, когда:
- связь обратная или ManyToMany,
- нужен фильтр на связанных объектах,
- столбцов в связанной таблице много, и копировать их в каждую строку JOIN дорого.

### Антипаттерн: prefetch на больших коллекциях

Если у заказа сто тысяч позиций, `prefetch_related("items")` загрузит все сто тысяч в память Python. Используй `Prefetch` с фильтром или вообще отдельный запрос.

## only, defer, values, values_list

Снижение объёма выгружаемых данных.

### only и defer для модельных объектов

`only("field1", "field2")` грузит только указанные поля. `defer("field1")` грузит всё кроме указанных.

```python
orders_queryset = Order.objects.only("id", "total_amount", "status").filter(status="paid")
```

Все остальные поля при первом обращении вызывают отдельный SELECT. Это легко превращается в N+1, если использовать неаккуратно.

`only` применим, когда нужны именно модельные объекты с методами и сигналами. Когда нужны просто данные, лучше `values`.

### values и values_list

`values("field1", "field2")` возвращает queryset словарей. `values_list("field1", "field2")` возвращает queryset кортежей. `values_list("field", flat=True)` возвращает queryset скаляров.

```python
order_summaries = Order.objects.filter(status="paid").values(
    "id", "total_amount", "customer__email"
)

order_ids = list(Order.objects.filter(status="paid").values_list("id", flat=True))
```

`values` поддерживает обращение через `__` к связанным таблицам и автоматически делает JOIN. Это удобно для отчётов.

### values с агрегацией и группировкой

`.values(...).annotate(...)` это `GROUP BY` по полям из values:

```python
from django.db.models import Sum, Count

revenue_by_customer = (
    Order.objects
    .filter(status="paid")
    .values("customer_id")
    .annotate(
        total_revenue=Sum("total_amount"),
        order_count=Count("id"),
    )
    .order_by("-total_revenue")
)
```

Это `GROUP BY customer_id`. Полезно для отчётов, агрегатных таблиц, материализованных представлений.

## Аннотации и агрегации

`aggregate` возвращает словарь со скалярными значениями. `annotate` добавляет вычисленное поле к каждой строке queryset.

### aggregate

```python
from django.db.models import Sum, Count, Avg, Min, Max, StdDev, Variance

revenue_summary = Order.objects.filter(status="paid").aggregate(
    total=Sum("total_amount"),
    average=Avg("total_amount"),
    minimum=Min("total_amount"),
    maximum=Max("total_amount"),
    count=Count("id"),
)
```

Все стандартные агрегаторы: `Sum`, `Count`, `Avg`, `Min`, `Max`, `StdDev`, `Variance`. PostgreSQL дополнительно: `ArrayAgg`, `StringAgg`, `JSONBAgg`, `BitAnd`, `BitOr`, `BoolAnd`, `BoolOr`, `Corr`, `CovarPop`, `RegrAvgX`, `RegrAvgY`, `RegrCount` и другие из `django.contrib.postgres.aggregates`.

### annotate с агрегатом

```python
from django.db.models import Count

customers_with_order_count = Customer.objects.annotate(
    order_count=Count("orders"),
).filter(order_count__gte=10)
```

Аннотация добавляется в SELECT и доступна в WHERE через имя.

### distinct в Count

`Count("field", distinct=True)` это `COUNT(DISTINCT field)`. Используй, когда есть JOIN, который дублирует строки:

```python
unique_customer_count = Order.objects.aggregate(
    customers=Count("customer_id", distinct=True),
)
```

### Условные агрегации

Самый частый случай: посчитать сразу несколько метрик с разными условиями.

```python
from django.db.models import Count, Q, Sum

order_metrics = Order.objects.aggregate(
    paid_count=Count("id", filter=Q(status="paid")),
    cancelled_count=Count("id", filter=Q(status="cancelled")),
    paid_revenue=Sum("total_amount", filter=Q(status="paid")),
    refunded_revenue=Sum("total_amount", filter=Q(status="refunded")),
)
```

Аргумент `filter` на агрегате компилируется в `FILTER (WHERE ...)` Postgres. Это в десятки раз быстрее, чем делать отдельный запрос на каждую метрику.

### Case и When для условных значений

```python
from django.db.models import Case, When, Value, IntegerField

orders_with_priority = Order.objects.annotate(
    priority=Case(
        When(total_amount__gte=10000, then=Value(1)),
        When(total_amount__gte=1000, then=Value(2)),
        default=Value(3),
        output_field=IntegerField(),
    ),
).order_by("priority", "-total_amount")
```

`output_field` обязателен, если Django не может вывести тип. Это частая причина ошибок.

### GROUP BY через ORM

В Django нет явного метода `.group_by()`. Группировка возникает как побочный эффект `.values(...).annotate(...)`. Это место, где регулярно ошибаются.

Правило: `values()` ДО `annotate()` создаёт `GROUP BY`. `values()` ПОСЛЕ `annotate()` это просто проекция полей.

```python
revenue_by_status = (
    Order.objects
    .values("status")
    .annotate(total_revenue=Sum("total_amount"))
)
```

Это:

```sql
SELECT status, SUM(total_amount) AS total_revenue
FROM orders
GROUP BY status
```

Группировка по нескольким полям:

```python
revenue_by_status_and_country = (
    Order.objects
    .values("status", "customer__country")
    .annotate(
        total_revenue=Sum("total_amount"),
        order_count=Count("id"),
    )
    .order_by("-total_revenue")
)
```

#### HAVING через filter после annotate

`filter` после `annotate` это `HAVING`, а до `annotate` это `WHERE`. Эти запросы выглядят похоже, но дают разный SQL и часто разный результат.

```python
top_customers_with_having = (
    Customer.objects
    .annotate(order_count=Count("orders"))
    .filter(order_count__gte=10)
)

top_customers_with_where_then_aggregate = (
    Customer.objects
    .filter(orders__status="paid")
    .annotate(paid_order_count=Count("orders"))
    .filter(paid_order_count__gte=10)
)
```

Первый: считаем все заказы клиента, потом отфильтровываем тех, у кого их больше десяти. Второй: фильтруем заказы по статусу, потом считаем оставшиеся, потом отфильтровываем клиентов. Это разные запросы.

#### Ловушка с GROUP BY и неявными полями

```python
strange_grouping = (
    Order.objects
    .values("id", "status")
    .annotate(total=Sum("total_amount"))
)
```

`GROUP BY id, status`. Поскольку `id` уникален, группа всегда из одной строки, и `Sum` бессмысленен. Распространённая ошибка: добавили `id` в `values` для отладки, и группировка сломалась.

Решение: всегда явно указывай только те поля, по которым реально группируешь. Если нужны дополнительные поля в выводе, вынеси их в `annotate` через `F("...")` или подзапрос.

#### Группировка по выражению

```python
from django.db.models.functions import TruncMonth

revenue_by_month = (
    Order.objects
    .annotate(order_month=TruncMonth("created_at"))
    .values("order_month")
    .annotate(monthly_revenue=Sum("total_amount"))
    .order_by("order_month")
)
```

Сначала аннотируем выражение, потом группируем по нему через `values`. Обратный порядок не сработает.

### alias для промежуточных выражений

`.alias()` это `.annotate()`, который не добавляет поле в SELECT. Появился в Django 3.2.

Когда применять: нужно использовать выражение в `filter`, `order_by`, или другом `annotate`, но само значение в результате не нужно.

```python
from django.db.models import Count

popular_categories = (
    Category.objects
    .alias(active_product_count=Count("products", filter=Q(products__is_active=True)))
    .filter(active_product_count__gte=5)
    .order_by("-active_product_count")
)
```

В SELECT нет `active_product_count`. Но `filter` и `order_by` его видят. SQL будет содержать `COUNT(...) FILTER (WHERE ...)` только в `HAVING` и `ORDER BY`, без выбора в результат.

#### Чем это лучше annotate

- Меньше данных летит из Postgres в Python.
- Плану запроса не нужно вычислять выражение для каждой строки результата, только для тех, что прошли HAVING.
- Понятнее намерение: "это вспомогательное вычисление, не часть результата".

#### Комбинация alias и annotate

Часть выражений нужна только для фильтрации, часть нужна на выходе:

```python
from django.db.models import Sum, F, Q

profitable_customers = (
    Customer.objects
    .alias(
        total_revenue=Sum("orders__total_amount", filter=Q(orders__status="paid")),
        total_cost=Sum("orders__cost_amount", filter=Q(orders__status="paid")),
    )
    .annotate(profit=F("total_revenue") - F("total_cost"))
    .filter(profit__gt=10000)
    .order_by("-profit")
)
```

Тут `total_revenue` и `total_cost` нужны только для расчёта `profit`. В результат идёт только `profit`. Без `alias` пришлось бы либо тащить лишние поля, либо повторять выражение в `filter`.

#### alias не работает в .values

Если потом сделать `.values("profit")`, поле найдётся (через `annotate` оно есть). Но если попробовать `.values("total_revenue")` после `alias`, упадёт. Значение для проекции есть только у `annotate`.

## Subquery, OuterRef, Exists

Когда нужно подставить значение или проверку существования из другого запроса в текущий.

### Subquery с OuterRef

```python
from django.db.models import Subquery, OuterRef

last_order_date = Order.objects.filter(
    customer_id=OuterRef("pk"),
).order_by("-created_at").values("created_at")[:1]

customers_with_last_order = Customer.objects.annotate(
    last_order_at=Subquery(last_order_date),
)
```

Запрос компилируется в коррелированный подзапрос. Срез `[:1]` обязателен, иначе подзапрос вернёт несколько строк и упадёт.

### Exists для проверки наличия

```python
from django.db.models import Exists, OuterRef

has_paid_order = Order.objects.filter(
    customer_id=OuterRef("pk"),
    status="paid",
)

customers_annotated = Customer.objects.annotate(
    has_paid_order=Exists(has_paid_order),
).filter(has_paid_order=True)
```

`Exists` лучше, чем `Count(...) > 0`, потому что Postgres останавливается на первой найденной строке. На больших таблицах разница на порядки.

### Subquery для denormalized полей

```python
from django.db.models import Subquery, OuterRef, F

last_payment_amount = Payment.objects.filter(
    order_id=OuterRef("pk"),
).order_by("-paid_at").values("amount")[:1]

orders_with_payment = Order.objects.annotate(
    last_payment_amount=Subquery(last_payment_amount),
).filter(last_payment_amount__gt=F("total_amount"))
```

### ArraySubquery для массива значений

`Subquery` со срезом `[:1]` возвращает скаляр. Когда нужен массив значений из подзапроса, используем `ArraySubquery` из `django.contrib.postgres.expressions`. Это Postgres-специфика, компилируется в `ARRAY(SELECT ...)`.

```python
from django.contrib.postgres.expressions import ArraySubquery
from django.db.models import OuterRef

recent_order_ids = Order.objects.filter(
    customer_id=OuterRef("pk"),
    status="paid",
).order_by("-created_at").values("id")[:5]

customers_with_recent_orders = Customer.objects.annotate(
    recent_order_ids=ArraySubquery(recent_order_ids),
)

for customer in customers_with_recent_orders:
    for current_order_id in customer.recent_order_ids:
        process_recent_order(current_order_id)
```

В отличие от `ArrayAgg` через JOIN, `ArraySubquery`:
- не дублирует строки родительской таблицы,
- не требует GROUP BY на родителе,
- возвращает упорядоченный массив по ORDER BY подзапроса,
- работает с лимитом и пагинацией внутри подзапроса.

#### ArraySubquery с JSONObject для богатых данных

```python
from django.contrib.postgres.expressions import ArraySubquery
from django.db.models import OuterRef, F
from django.db.models.functions import JSONObject

recent_orders_json = Order.objects.filter(
    customer_id=OuterRef("pk"),
).annotate(
    order_data=JSONObject(
        id=F("id"),
        amount=F("total_amount"),
        status=F("status"),
    ),
).order_by("-created_at").values("order_data")[:10]

customers_with_orders = Customer.objects.annotate(
    recent_orders=ArraySubquery(recent_orders_json),
)
```

Один SELECT, на выходе массив словарей у каждого клиента. Альтернатива через ORM требует `prefetch_related` плюс ручной фильтр по дате плюс срез по 10 в Python.

### Фильтрация через queryset с values вместо списка id

Очень частая ошибка, которая удваивает количество запросов и иногда катастрофически увеличивает их объём.

#### Антипаттерн

```python
active_customer_ids = list(
    Customer.objects.filter(is_active=True).values_list("id", flat=True)
)
recent_orders = Order.objects.filter(customer_id__in=active_customer_ids)
```

Что происходит:
1. Первый запрос вытягивает все id активных клиентов в Python. На миллионе клиентов это миллион int в памяти.
2. Django подставляет этот миллион id в SQL: `WHERE customer_id IN (1, 2, 3, ..., 1000000)`. SQL получается огромный, парсится и планируется долго.
3. Postgres строит хеш из миллиона значений в памяти.

#### Правильно

```python
active_customer_ids_subquery = Customer.objects.filter(is_active=True).values("id")
recent_orders = Order.objects.filter(customer_id__in=active_customer_ids_subquery)
```

Что происходит:
1. Запрос один. Подзапрос компилируется в `WHERE customer_id IN (SELECT id FROM customers WHERE is_active = TRUE)`.
2. Postgres решает: использовать вложенный подзапрос, превратить в semi-join, в hash-join, в anti-join. План оптимизируется под данные и индексы.
3. Никаких списков в Python. Никакой передачи миллионов значений по сети.

Тонкости:
- Используем `.values("id")`, не `.values_list("id", flat=True)`. Оба работают, но `.values()` идиоматичнее для подзапросов.
- Не оборачиваем в `list()`. Оборачивание материализует и превращает обратно в антипаттерн.
- Срез работает: `.values("id")[:1000]` это `LIMIT 1000` внутри подзапроса.

#### Когда полезен список

Список оправдан, когда:
- Идентификаторы пришли извне (от пользователя, из API, из файла) и их немного.
- Нужно использовать тот же набор id в нескольких местах кода без повторного запроса в БД.
- Источник id это не Postgres (например, Redis или внешний сервис).

Граница: пара тысяч id безопасна как список. Десятки тысяч уже плохо. Сотни тысяч это всегда подзапрос или временная таблица.

#### Вариант с exclude

```python
inactive_user_ids_subquery = User.objects.filter(is_active=False).values("id")
orders_of_active_users = Order.objects.exclude(customer_id__in=inactive_user_ids_subquery)
```

Это `WHERE customer_id NOT IN (SELECT ...)`. Postgres превратит в anti-join. На больших данных это работает на порядки быстрее, чем `filter(customer_id__in=...)` со списком, потому что план запроса видит структуру.

#### Subquery в exclude через ~Exists

`NOT IN` в Postgres имеет известную проблему с NULL: если подзапрос вернёт хоть один NULL, весь NOT IN вернёт пустой набор. Безопаснее `~Exists`:

```python
from django.db.models import Exists, OuterRef

inactive_user_check = User.objects.filter(
    id=OuterRef("customer_id"),
    is_active=False,
)

orders_of_active_users = Order.objects.filter(~Exists(inactive_user_check))
```

`NOT EXISTS` корректно работает с NULL и часто даёт лучший план.

## F-выражения

`F` это ссылка на значение поля в БД. Позволяет:
- сравнивать поля одной таблицы между собой,
- делать арифметику на стороне БД без вытягивания значения в Python,
- атомарно обновлять поле без race condition.

### Сравнение полей

```python
from django.db.models import F

orders_overpaid = Order.objects.filter(paid_amount__gt=F("total_amount"))
```

### Атомарный инкремент

Без F:

```python
order = Order.objects.get(pk=order_id)
order.view_count += 1
order.save()
```

Это race condition. Между `get` и `save` другой процесс может прочитать то же значение и оба сохранят неправильное.

С F:

```python
Order.objects.filter(pk=order_id).update(view_count=F("view_count") + 1)
```

Это атомарный `UPDATE orders SET view_count = view_count + 1 WHERE id = ...`. Нет race condition.

### Арифметика на уровне БД

```python
from django.db.models import F

orders_with_margin = Order.objects.annotate(
    margin=F("total_amount") - F("cost_amount"),
    margin_percent=(F("total_amount") - F("cost_amount")) * 100 / F("total_amount"),
)
```

Все операции считаются в Postgres. В Python приходит уже результат.

### Bulk update через F

```python
Product.objects.filter(category_id=42).update(
    price=F("price") * Decimal("1.10"),
    updated_at=timezone.now(),
)
```

Один запрос обновляет любое количество строк.

## Q-объекты

Сложные условия с AND, OR, NOT.

```python
from django.db.models import Q

active_or_pending = Order.objects.filter(
    Q(status="active") | Q(status="pending"),
    created_at__gte=last_week,
)

not_cancelled = Order.objects.filter(~Q(status="cancelled"))

complex_filter = Order.objects.filter(
    Q(customer__country="DE") & (Q(total_amount__gte=1000) | Q(is_priority=True))
)
```

Правила:
- Позиционные Q объекты соединяются через AND.
- `|` это OR, `&` это AND, `~` это NOT.
- Оборачивай OR в скобки, иначе приоритет операторов даст неожиданный результат.
- Q объекты можно собирать программно через `functools.reduce`:

```python
from functools import reduce
from operator import or_

status_filters = [Q(status=current_status) for current_status in active_statuses]
combined_filter = reduce(or_, status_filters)
matching_orders = Order.objects.filter(combined_filter)
```

## Database functions

Функции уровня БД доступны через `django.db.models.functions`. Они работают внутри annotate, filter, order_by.

### Coalesce, Greatest, Least

```python
from django.db.models import F
from django.db.models.functions import Coalesce, Greatest, Least
from decimal import Decimal

orders_normalized = Order.objects.annotate(
    discount=Coalesce("discount_amount", Decimal("0")),
    final_price=Greatest(F("total_amount") - F("discount_amount"), Decimal("0")),
    capped_price=Least(F("total_amount"), Decimal("999999.99")),
)
```

`Coalesce` особенно нужен, когда LEFT JOIN или Subquery возвращает NULL, и его нужно заменить на значение по умолчанию.

### Cast для смены типа

```python
from django.db.models import IntegerField
from django.db.models.functions import Cast

products_with_stock_int = Product.objects.annotate(
    stock_int=Cast("stock_decimal", output_field=IntegerField()),
)
```

### Текстовые функции

```python
from django.db.models.functions import Concat, Lower, Upper, Length, Substr, Trim
from django.db.models import Value

users_with_full_name = User.objects.annotate(
    full_name=Concat("first_name", Value(" "), "last_name"),
    name_length=Length("first_name"),
    initials=Concat(Substr("first_name", 1, 1), Substr("last_name", 1, 1)),
    normalized_email=Lower("email"),
)
```

`Value` оборачивает константы. Без неё Django примет строку за имя поля и упадёт.

### Функции дат

```python
from django.db.models.functions import (
    Trunc, TruncDate, TruncMonth, TruncYear,
    Extract, ExtractYear, ExtractMonth, ExtractWeekDay,
    Now,
)

orders_by_month = (
    Order.objects
    .annotate(order_month=TruncMonth("created_at"))
    .values("order_month")
    .annotate(monthly_revenue=Sum("total_amount"))
    .order_by("order_month")
)

orders_with_age_in_days = Order.objects.annotate(
    age_days=Extract(Now() - F("created_at"), "day"),
)
```

`Trunc` обрезает дату до указанной точности. `Extract` достаёт часть даты.

### JSONObject для построения JSON в SELECT

`JSONObject` появился в Django 4.0. Собирает JSON объект из полей и выражений на стороне БД. Это лучше, чем тащить отдельные поля и склеивать в Python.

```python
from django.db.models import F
from django.db.models.functions import JSONObject

orders_with_summary = Order.objects.annotate(
    summary=JSONObject(
        order_id=F("id"),
        amount=F("total_amount"),
        customer_email=F("customer__email"),
        item_count=Count("items"),
    ),
)
```

Каждый `summary` это уже готовый dict, не модельный объект. Это удобно для:
- API ответов, когда сериализатор простой и нет смысла строить полную модель,
- экспортов в JSON,
- передачи в очередь Celery (только id и нужные поля),
- upsert в JSON колонку другой таблицы.

#### Вложенный JSONObject

```python
customer_with_metadata = Customer.objects.annotate(
    profile=JSONObject(
        id=F("id"),
        name=F("full_name"),
        contact=JSONObject(
            email=F("email"),
            phone=F("phone"),
        ),
        country=F("country__code"),
    ),
)
```

Postgres вернёт колонку `profile` сразу с вложенной структурой. Никакой работы по сборке в Python.

#### JSONObject плюс ArrayAgg для агрегатов

Очень мощная комбинация: собрать массив JSON объектов одним запросом.

```python
from django.contrib.postgres.aggregates import ArrayAgg
from django.db.models.functions import JSONObject

customers_with_orders_array = Customer.objects.annotate(
    orders_data=ArrayAgg(
        JSONObject(
            id=F("orders__id"),
            amount=F("orders__total_amount"),
            status=F("orders__status"),
        ),
        filter=Q(orders__status__in=["paid", "shipped"]),
        ordering="-orders__created_at",
    ),
)

for customer in customers_with_orders_array:
    for order_dict in customer.orders_data:
        order_id_value = order_dict["id"]
```

Один запрос, ноль JOIN-дублирования в Python, готовая структура. Альтернатива через `prefetch_related` дала бы два запроса и сборку в Python.

## Оконные функции

Оконные функции считают агрегаты по окну строк, не схлопывая результат. Это самый мощный инструмент аналитических запросов.

### Структура Window

```python
from django.db.models import Window, F
from django.db.models.functions import RowNumber

orders_ranked = Order.objects.annotate(
    row_index=Window(
        expression=RowNumber(),
        partition_by=[F("customer_id")],
        order_by=F("created_at").desc(),
    ),
)
```

Это `ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC)`. Каждый заказ получает порядковый номер внутри своего customer.

### Все доступные функции

Из `django.db.models.functions`:

- `RowNumber` номер строки в окне, начиная с 1.
- `Rank` ранг с пропусками при равенстве (1, 1, 3).
- `DenseRank` ранг без пропусков (1, 1, 2).
- `PercentRank` процентильный ранг.
- `CumeDist` накопительная дистрибуция.
- `Ntile(n)` разбивка окна на n равных частей.
- `Lag(expression, offset)` значение с предыдущей строки.
- `Lead(expression, offset)` значение со следующей строки.
- `FirstValue(expression)` значение первой строки окна.
- `LastValue(expression)` значение последней строки окна.
- `NthValue(expression, n)` значение n-й строки окна.

Любой обычный агрегат тоже работает как оконная функция: `Sum`, `Avg`, `Count`, `Min`, `Max`.

### Топ N в группе

Классическая задача: выбрать три последних заказа каждого клиента.

```python
from django.db.models import Window, F, Subquery
from django.db.models.functions import RowNumber

ranked_orders_subquery = Order.objects.annotate(
    row_index=Window(
        expression=RowNumber(),
        partition_by=[F("customer_id")],
        order_by=F("created_at").desc(),
    ),
).values("id", "row_index")

top_three_per_customer_ids = [
    row["id"]
    for row in ranked_orders_subquery
    if row["row_index"] <= 3
]

top_three_orders = Order.objects.filter(id__in=top_three_per_customer_ids)
```

Внимание: в Django нельзя фильтровать queryset по результату Window напрямую в WHERE. Это ограничение SQL стандарта (оконные функции вычисляются после WHERE). Решения:
- сделать subquery и фильтровать снаружи,
- использовать CTE через raw SQL,
- использовать `LATERAL JOIN` через raw SQL.

### Скользящие окна с frame

```python
from django.db.models import Window, F, Sum
from django.db.models.expressions import RowRange, ValueRange

revenue_with_moving_average = Order.objects.annotate(
    moving_avg_7_days=Window(
        expression=Avg("total_amount"),
        partition_by=[F("customer_id")],
        order_by=F("created_at").asc(),
        frame=RowRange(start=-6, end=0),
    ),
)
```

`RowRange(start=-6, end=0)` это `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW`. Подходит для скользящего среднего по последним N строкам.

`ValueRange(start=-6, end=0)` это `RANGE BETWEEN ... PRECEDING AND CURRENT ROW`. Использует значение поля сортировки, а не позицию строки. Полезно для интервалов времени.

### Lag и Lead для сравнения с соседом

```python
from django.db.models.functions import Lag

orders_with_previous = Order.objects.annotate(
    previous_order_amount=Window(
        expression=Lag("total_amount", offset=1),
        partition_by=[F("customer_id")],
        order_by=F("created_at").asc(),
    ),
).annotate(
    amount_diff=F("total_amount") - F("previous_order_amount"),
)
```

Lag без partition берёт предыдущую строку всего набора. С partition только в пределах группы.

### Кумулятивная сумма

```python
from django.db.models import Window, F, Sum

orders_cumulative = Order.objects.annotate(
    cumulative_revenue=Window(
        expression=Sum("total_amount"),
        partition_by=[F("customer_id")],
        order_by=F("created_at").asc(),
    ),
)
```

Без явного frame Postgres использует `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. Это и есть кумулятивная сумма.

## Условные выражения и сложные SQL

### When и Case в обновлении

```python
from django.db.models import Case, When, Value

Order.objects.update(
    priority=Case(
        When(total_amount__gte=10000, then=Value("vip")),
        When(total_amount__gte=1000, then=Value("normal")),
        default=Value("low"),
    ),
)
```

Один UPDATE с разными значениями для разных строк. Без `Case` пришлось бы делать три отдельных UPDATE.

### Conditional update с разными значениями для каждой строки

Часто нужно обновить тысячу строк, каждую своим значением. bulk_update это умеет, но для очень больших объёмов через Case удобнее одним запросом:

```python
from django.db.models import Case, When, Value, IntegerField

new_stocks_by_product_id = {1: 100, 2: 250, 3: 75}

when_clauses = [
    When(pk=product_id, then=Value(new_stock))
    for product_id, new_stock in new_stocks_by_product_id.items()
]

Product.objects.filter(pk__in=new_stocks_by_product_id.keys()).update(
    stock=Case(*when_clauses, output_field=IntegerField()),
)
```

Один SQL вместо тысячи. Но не злоупотребляй: при огромном словаре Case разрастается, и query plan становится медленным. Для больших объёмов используй `bulk_update`.

## Bulk операции

### bulk_create

```python
new_orders = [
    Order(customer_id=42, total_amount=100, status="pending"),
    Order(customer_id=43, total_amount=200, status="pending"),
    Order(customer_id=44, total_amount=300, status="pending"),
]

Order.objects.bulk_create(new_orders, batch_size=1000)
```

Один INSERT на батч вместо отдельного INSERT на объект. Для миллиона строк это разница между минутами и часами.

Параметры:
- `batch_size` сколько строк в одном INSERT. Оптимум 500-2000. Слишком маленький теряет смысл, слишком большой упирается в лимиты Postgres.
- `ignore_conflicts=True` пропустить дубликаты по уникальному ключу. Превращается в `ON CONFLICT DO NOTHING`.
- `update_conflicts=True` обновить при конфликте. Это upsert.
- `unique_fields` поля для определения конфликта.
- `update_fields` поля для обновления при конфликте.

### Upsert через bulk_create

```python
Product.objects.bulk_create(
    new_products,
    update_conflicts=True,
    unique_fields=["sku"],
    update_fields=["name", "price", "stock"],
    batch_size=1000,
)
```

Это `INSERT ... ON CONFLICT (sku) DO UPDATE SET name = EXCLUDED.name, ...`. Атомарный upsert, никаких race condition.

Ограничения:
- `bulk_create` не вызывает `save()`, не запускает сигналы `pre_save` и `post_save`. Кастомная логика в `save()` не выполнится.
- `pre_create` и `post_create` сигналов нет.
- На SQLite `bulk_create` без upsert не возвращает PK.

### bulk_update

```python
products_to_update = list(Product.objects.filter(category_id=42))
for product in products_to_update:
    product.price = product.price * Decimal("1.10")

Product.objects.bulk_update(products_to_update, fields=["price"], batch_size=500)
```

Под капотом строит огромный CASE WHEN. Эффективен для тысяч строк, но не для миллионов.

Когда нужно обновить миллион строк одинаково, используй `update()` с `F`:

```python
Product.objects.filter(category_id=42).update(price=F("price") * Decimal("1.10"))
```

Это один SQL, любой объём.

### Когда bulk_update не подходит

Если разных значений много, `bulk_update` строит CASE WHEN на каждую строку. Для миллиона строк план становится неуправляемым. Альтернативы:
- временная таблица + UPDATE FROM,
- COPY в staging таблицу + MERGE.

### COPY для миллионов строк

`bulk_create` тоже не справится с десятками миллионов. Postgres COPY быстрее в 10-50 раз.

```python
import csv
import io

from django.db import connection


def copy_products_from_iterable(product_rows) -> None:
    """
    Грузит большие объёмы данных в таблицу products через COPY.

    :param product_rows: итератор кортежей (sku, name, price, stock).
    :return: None.
    """
    csv_buffer = io.StringIO()
    csv_writer = csv.writer(csv_buffer)
    for product_row in product_rows:
        csv_writer.writerow(product_row)
    csv_buffer.seek(0)

    with connection.cursor() as database_cursor:
        with database_cursor.copy(
            "COPY products (sku, name, price, stock) FROM STDIN WITH CSV"
        ) as copy_stream:
            copy_stream.write(csv_buffer.read())
```

Работает с psycopg3 драйвером. Это самый быстрый способ загрузить большой объём в Postgres.

### Pattern: загрузка через staging таблицу

Для очень больших объёмов с upsert:

1. COPY в `products_staging` (без констрейнтов и индексов).
2. INSERT ... SELECT из staging в основную с ON CONFLICT.
3. TRUNCATE staging.

```python
from django.db import connection


def upsert_via_staging(staging_table_name: str, target_table_name: str) -> None:
    """
    Переносит данные из staging таблицы в целевую с upsert.

    :param staging_table_name: имя таблицы со свежими данными.
    :param target_table_name: имя целевой таблицы.
    :return: None.
    """
    upsert_sql = f"""
        INSERT INTO {target_table_name} (sku, name, price, stock)
        SELECT sku, name, price, stock FROM {staging_table_name}
        ON CONFLICT (sku) DO UPDATE SET
            name = EXCLUDED.name,
            price = EXCLUDED.price,
            stock = EXCLUDED.stock,
            updated_at = NOW()
    """
    with connection.cursor() as database_cursor:
        database_cursor.execute(upsert_sql)
        database_cursor.execute(f"TRUNCATE TABLE {staging_table_name}")
```

## Генераторы, islice и батчирование

Когда данных много, а память ограничена, идиома такая: источник это генератор, потребитель работает порциями фиксированного размера, нарезка делается через `itertools.islice` или `itertools.batched`. Это переносится один в один на `bulk_create`, `bulk_update`, удаление миллионов строк, ETL и любые batch воркеры.

### Зачем генераторы

Генератор отдаёт элементы по одному и не материализует всю последовательность. Это даёт три выгоды:
- константная память независимо от размера источника,
- ленивый pipeline: операции не выполняются пока результат не понадобится,
- легко комбинируются (filter, map, transform поверх генератора без копий списков).

Антипаттерн:

```python
all_source_rows = list(read_source())
transformed_rows = [transform_row(row) for row in all_source_rows]
new_orders = [Order(**row_data) for row_data in transformed_rows]
Order.objects.bulk_create(new_orders, batch_size=1000)
```

В памяти одновременно три копии всего массива. На миллионе строк это гигабайты.

С генераторами:

```python
source_rows_generator = read_source()
transformed_generator = (transform_row(row) for row in source_rows_generator)
orders_generator = (Order(**row_data) for row_data in transformed_generator)

for batch_orders in chunked(orders_generator, 1000):
    Order.objects.bulk_create(batch_orders)
```

В памяти одновременно одна строка плюс один батч.

### Классический паттерн: islice плюс while

`itertools.islice(iterable, n)` отдаёт первые n элементов. Для генератора это значит "достань следующие n из текущей позиции". Ничего не копирует.

```python
from itertools import islice
from typing import Iterable, Iterator


def chunked(source_iterable: Iterable, batch_size: int) -> Iterator[list]:
    """
    Нарезает любой итерируемый объект на батчи фиксированного размера.

    :param source_iterable: источник данных, любой iterable.
    :param batch_size: размер одного батча.
    :return: генератор списков длиной до batch_size, последний может быть короче.
    :raises ValueError: если batch_size не положительное число.
    """
    if batch_size <= 0:
        raise ValueError("batch_size must be positive")
    iterator = iter(source_iterable)
    while True:
        current_batch = list(islice(iterator, batch_size))
        if not current_batch:
            break
        yield current_batch
```

Логика:
- `iter()` превращает любой iterable в iterator. Для генератора это no-op.
- `islice(iterator, batch_size)` это новый итератор, который вытягивает не больше batch_size элементов из исходного.
- `list(...)` материализует только этот батч.
- Когда исходный иссяк, `list(islice(...))` вернёт пустой список и `while` прервётся.

Размер батча в памяти всегда ограничен и не зависит от размера источника.

### itertools.batched в Python 3.12+

С версии Python 3.12 в стандартной библиотеке появилась `itertools.batched`. Это та же логика без своей реализации:

```python
from itertools import batched

for batch_tuple in batched(source_generator, 1000):
    process_batch(batch_tuple)
```

Отличия от своей `chunked`:
- возвращает `tuple`, не `list`,
- доступна только с Python 3.12,
- с 3.13 поддерживает параметр `strict=True` (бросает исключение, если последний батч короче).

Если проект на 3.12+, используем `batched`. Если на 3.10/3.11, своя `chunked`. Реализации эквивалентны.

### bulk_create из генератора

`bulk_create` принимает любой iterable. Не нужно строить список миллиона объектов в памяти.

Антипаттерн:

```python
new_orders_list = []
for source_row in read_source_rows():
    new_orders_list.append(Order(**source_row))
Order.objects.bulk_create(new_orders_list, batch_size=1000)
```

Миллион объектов лежит в памяти одновременно. Дальше Django ещё разрежет внутри на батчи и вставит, но память уже съедена.

Правильно:

```python
def order_generator(source_rows):
    """
    Превращает поток строк источника в генератор Order инстансов.

    :param source_rows: iterable со строками источника.
    :return: генератор Order инстансов.
    """
    for source_row in source_rows:
        yield Order(
            sku=source_row["sku"],
            customer_id=source_row["customer_id"],
            total_amount=source_row["amount"],
        )


for batch_orders in chunked(order_generator(read_source_rows()), 1000):
    Order.objects.bulk_create(batch_orders)
```

В памяти одновременно живёт максимум 1000 объектов плюс одна строка источника. Источник может быть размером с диск.

### Зачем внешний chunked, если bulk_create имеет batch_size

`bulk_create` сам принимает `batch_size` параметр. Зачем тогда внешний `chunked`?

Причины использовать внешний цикл:
- Между батчами нужно делать промежуточную работу: логирование прогресса, проверка флага остановки, пауза для разгрузки реплики.
- Каждый батч это отдельная транзакция. Это критично под PgBouncer (короткие транзакции возвращают соединение в пул быстро).
- Легче поймать и обработать ошибку на конкретном батче, продолжив остальные.
- Если источник медленный (HTTP, файл по сети), внутренний `batch_size` Django всё равно сначала ждёт пока iterable полностью прочитается до конца текущего внутреннего батча. Внешний `chunked` даёт явный контроль.
- Не все методы Django принимают `batch_size`. Например, `update` или ручной `INSERT ... SELECT` нужно разбивать самостоятельно.

### Полный пайплайн ETL

```python
import csv
from decimal import Decimal

from django.db import transaction


def read_source_csv(file_path: str):
    """
    Читает CSV файл построчно как генератор словарей.

    :param file_path: путь к файлу.
    :return: генератор словарей с данными строки.
    """
    with open(file_path, newline="", encoding="utf-8") as source_file:
        for csv_row in csv.DictReader(source_file):
            yield csv_row


def transform_to_orders(source_rows):
    """
    Превращает строки CSV в Order инстансы, отсеивая невалидные.

    :param source_rows: генератор словарей CSV.
    :return: генератор Order инстансов.
    """
    for source_row in source_rows:
        if not source_row.get("sku"):
            continue
        yield Order(
            sku=source_row["sku"],
            total_amount=Decimal(source_row["amount"]),
            customer_id=int(source_row["customer_id"]),
        )


def import_orders_from_csv(file_path: str, batch_size: int = 1000) -> int:
    """
    Импортирует заказы из CSV файла батчами с upsert по sku.

    :param file_path: путь к CSV.
    :param batch_size: размер батча для bulk_create.
    :return: количество импортированных заказов.
    """
    source_rows_generator = read_source_csv(file_path)
    orders_generator = transform_to_orders(source_rows_generator)

    total_imported = 0
    for batch_orders in chunked(orders_generator, batch_size):
        with transaction.atomic():
            Order.objects.bulk_create(
                batch_orders,
                update_conflicts=True,
                unique_fields=["sku"],
                update_fields=["total_amount", "customer_id"],
            )
        total_imported += len(batch_orders)
    return total_imported
```

Что важно:
- Память: всегда максимум `batch_size` объектов плюс одна строка CSV.
- Каждый батч это отдельная короткая транзакция. Под PgBouncer это правильно.
- Если упадёт на середине, ранее импортированные батчи останутся в БД. Для повторного запуска источник должен быть идемпотентным или нужен чекпоинт.
- Файл может быть в десятки гигабайт. Пиковая память константная.

### while-цикл для batch обработки очереди

Когда нужно обработать всё, что есть, но количество заранее неизвестно, и каждый цикл должен закоммититься отдельно, используем явный `while` с проверкой выхода по пустому батчу.

```python
def process_pending_orders_in_batches(batch_size: int = 500) -> int:
    """
    Обрабатывает все pending заказы батчами, пока они есть.

    :param batch_size: размер батча.
    :return: количество обработанных заказов.
    """
    total_processed = 0
    while True:
        batch_ids = list(
            Order.objects
            .filter(status="pending")
            .values_list("id", flat=True)[:batch_size]
        )
        if not batch_ids:
            break
        with transaction.atomic():
            Order.objects.filter(id__in=batch_ids).update(status="processing")
            do_batch_processing(batch_ids)
            Order.objects.filter(id__in=batch_ids).update(status="processed")
        total_processed += len(batch_ids)
    return total_processed
```

Условие выхода: запрос вернул пустой список. Подходит, когда:
- источник это БД-запрос с фильтром по обновляемому статусу,
- между итерациями нужен коммит,
- источник может пополняться во время обработки (новые pending заказы появятся за счёт других процессов, и воркер их подхватит).

Отличие от keyset-обхода: тут не двигаем курсор по `id`, а каждый раз берём первые `batch_size` подходящих. Это работает, потому что после UPDATE строки меняют статус и больше не попадают в выборку. Если статус не меняется, нужен keyset, иначе будет бесконечный цикл по тем же строкам.

### Удаление миллиона строк через while

`Order.objects.filter(...).delete()` на миллионе строк это длинный лок, гигантский WAL, мёртвый VACUUM. Через батчи:

```python
def delete_old_orders_in_batches(cutoff_date, batch_size: int = 10000) -> int:
    """
    Удаляет старые заказы батчами, пока есть подходящие.

    :param cutoff_date: дата отсечения, всё старше будет удалено.
    :param batch_size: размер батча.
    :return: общее количество удалённых строк.
    """
    total_deleted = 0
    while True:
        target_ids = list(
            Order.objects
            .filter(created_at__lt=cutoff_date)
            .values_list("id", flat=True)[:batch_size]
        )
        if not target_ids:
            break
        deleted_count, _ = Order.objects.filter(id__in=target_ids).delete()
        total_deleted += deleted_count
    return total_deleted
```

Короткие транзакции, маленькие коммиты, VACUUM успевает работать, реплики не отстают.

### bulk_update из генератора

`bulk_update` принимает iterable. Паттерн идентичен:

```python
def update_prices_in_batches(updated_products_generator, batch_size: int = 500) -> int:
    """
    Применяет обновления цен батчами через bulk_update.

    :param updated_products_generator: генератор Product с новыми ценами.
    :param batch_size: размер батча.
    :return: количество обновлённых продуктов.
    """
    total_updated = 0
    for batch_products in chunked(updated_products_generator, batch_size):
        Product.objects.bulk_update(batch_products, fields=["price"])
        total_updated += len(batch_products)
    return total_updated
```

Источник `updated_products_generator` может приходить откуда угодно: из другого SQL запроса, из API, из CSV, из Redis. Память не растёт.

### Комбинация генераторов keyset + chunked

Keyset обход уже сам генератор. Можно скормить его в `chunked` и пайплайнить дальше:

```python
def archive_old_orders_in_batches(cutoff_date, batch_size: int = 1000) -> int:
    """
    Архивирует старые заказы батчами через keyset обход.

    :param cutoff_date: дата отсечения.
    :param batch_size: размер батча.
    :return: количество перенесённых заказов.
    """
    source_orders_generator = (
        order
        for order in iterate_orders_by_keyset(batch_size=batch_size)
        if order.created_at < cutoff_date
    )
    archive_records_generator = (
        OrderArchive(
            original_id=current_order.id,
            customer_id=current_order.customer_id,
            payload=current_order.to_archive_dict(),
        )
        for current_order in source_orders_generator
    )

    total_archived = 0
    for batch_archives in chunked(archive_records_generator, batch_size):
        with transaction.atomic():
            OrderArchive.objects.bulk_create(batch_archives)
        total_archived += len(batch_archives)
    return total_archived
```

Источник лениво читает БД через keyset. Фильтрация ленива. Трансформация ленива. Запись батчами. Память константная независимо от размера таблицы.

### Постановка задач в Celery батчами

Часто нужно поставить миллион задач в очередь. Прямой цикл `for ... task.delay(...)` забивает сеть и брокер. Решение: батчевая постановка через `task.chunks` или ручная батчевая отправка.

```python
from celery import group


def schedule_processing_in_batches(batch_size: int = 100) -> int:
    """
    Ставит задачи обработки pending заказов группами в Celery.

    :param batch_size: количество id в одной задаче.
    :return: количество поставленных задач.
    """
    pending_ids_generator = iterate_pending_order_ids(batch_size=5000)

    total_jobs_scheduled = 0
    for batch_ids in chunked(pending_ids_generator, batch_size):
        process_orders_batch.delay(order_ids=batch_ids)
        total_jobs_scheduled += 1
    return total_jobs_scheduled
```

Одна Celery задача обрабатывает сразу 100 заказов. Накладные расходы брокера амортизируются, скорость растёт в десятки раз.

### Подводные камни

- **Генератор проходится один раз.** После исчерпания пустой. Если нужно несколько проходов, либо `itertools.tee` (с осторожностью, кэширует пройденное в памяти), либо пересоздавай генератор.
- **Исключение в середине обработки оставляет частично сделанную работу.** Если это недопустимо, оборачивай весь pipeline в одну транзакцию (но это убивает идею батчей и плохо для PgBouncer). Чаще правильно: каждая транзакция атомарна на уровне батча, добавляем idempotency (upsert вместо insert) и retry на уровне батча.
- **Размер батча.** Слишком маленький теряет смысл (накладные расходы на транзакцию выше работы). Слишком большой ест память и держит транзакцию долго. Эмпирический коридор: 500-5000 для bulk_create, 100-1000 для bulk_update, 5000-50000 для values_list скаляров.
- **Не путаем размер батча приложения и `batch_size` параметр Django методов.** Внешний `chunked` контролирует поток и транзакции, внутренний `batch_size` методов управляет размером отдельного INSERT внутри одной транзакции.
- **Lazy evaluation плюс closure.** Генератор-выражение захватывает переменные по ссылке. Если в pipeline есть `(x for x in ...)` с переменной из внешнего цикла, проверь что переменная не переопределилась к моменту итерации.
- **Дублирование при пополняющемся источнике.** Если в `process_pending_orders_in_batches` другой процесс параллельно меняет статус, возможна гонка. Защищаемся через `select_for_update(skip_locked=True)` (см. раздел Транзакции и блокировки).

## Стриминг больших таблиц

`Order.objects.all()` на 50 миллионов строк положит процесс. Нужен стриминг.

### Важно: server-side cursors отключены

В обычном Postgres стриминг делается через `iterator()`, который создаёт server-side cursor (`DECLARE ... CURSOR`). У нас это не работает.

Причина: PgBouncer в режиме transaction pooling возвращает соединение в пул после каждой транзакции. Server-side cursor живёт в рамках сессии, а не транзакции. После возврата соединения в пул следующий клиент получает соединение с чужим открытым курсором или потеряет свой. Поэтому в настройках стоит `DISABLE_SERVER_SIDE_CURSORS = True`.

Что делает Django при отключённых server-side cursors:

```python
for order in Order.objects.all().iterator(chunk_size=2000):
    process_single_order(order)
```

Это превращается не в `DECLARE CURSOR`, а в обычный SELECT, который тянет всё в клиент. Драйвер psycopg всё равно загрузит весь результат в память. `chunk_size` управляет только тем, как Django нарезает результат при отдаче в Python, не тем, сколько Postgres присылает за раз. Память клиента всё равно вырастет.

Вывод: `iterator()` в нашей конфигурации это НЕ стриминг. Использовать его для больших выборок бесполезно и опасно.

### Единственный надёжный способ: keyset пагинация

Идём батчами по индексированному полю. Каждый батч это отдельный короткий запрос, который коммитится и не держит соединение долго.

```python
def iterate_orders_by_keyset(batch_size: int = 1000):
    """
    Итерирует все заказы батчами через keyset пагинацию.

    :param batch_size: размер батча.
    :return: генератор заказов.
    """
    last_seen_id = 0
    while True:
        batch_orders = list(
            Order.objects
            .filter(id__gt=last_seen_id)
            .order_by("id")[:batch_size]
        )
        if not batch_orders:
            break
        yield from batch_orders
        last_seen_id = batch_orders[-1].id
```

Преимущества:
- Каждый запрос короткий, транзакция короткая, PgBouncer счастлив.
- Время на батч константное независимо от глубины. `OFFSET 1000000` так не умеет.
- Память клиента ограничена `batch_size`.
- Легко перезапустить с любого места: храни `last_seen_id` где угодно.

Подводные камни:
- Поле для keyset должно быть монотонно растущее и иметь индекс. Обычно это `id`. Если данные пишутся с разными `id` асинхронно, можно пропустить строки. Тогда добавляй `created_at` как вторичный ключ.
- Если строки удаляются параллельно с итерацией, дубликатов не будет, но прогресс может казаться неровным.
- Сортировка обязательна. Без `order_by` Postgres вернёт строки в произвольном порядке, и `id__gt` не даст полного обхода.

### Keyset с составным ключом

Когда `id` не подходит (например, данные шардированы по `tenant_id`):

```python
def iterate_events_by_keyset(tenant_id: int, batch_size: int = 1000):
    """
    Итерирует события одного тенанта по составному keyset.

    :param tenant_id: идентификатор тенанта.
    :param batch_size: размер батча.
    :return: генератор событий.
    """
    last_seen_created_at = None
    last_seen_id = 0
    while True:
        base_queryset = Event.objects.filter(tenant_id=tenant_id)
        if last_seen_created_at is not None:
            base_queryset = base_queryset.filter(
                Q(created_at__gt=last_seen_created_at)
                | Q(created_at=last_seen_created_at, id__gt=last_seen_id)
            )
        batch_events = list(
            base_queryset.order_by("created_at", "id")[:batch_size]
        )
        if not batch_events:
            break
        yield from batch_events
        last_seen_created_at = batch_events[-1].created_at
        last_seen_id = batch_events[-1].id
```

Индекс `(tenant_id, created_at, id)` обязателен.

### values_list для скаляров

Когда нужны только id, не тяни модели:

```python
def iterate_pending_order_ids(batch_size: int = 5000):
    """
    Итерирует id всех pending заказов через keyset.

    :param batch_size: размер батча.
    :return: генератор id.
    """
    last_seen_id = 0
    while True:
        batch_ids = list(
            Order.objects
            .filter(status="pending", id__gt=last_seen_id)
            .order_by("id")
            .values_list("id", flat=True)[:batch_size]
        )
        if not batch_ids:
            break
        yield from batch_ids
        last_seen_id = batch_ids[-1]
```

Память на батч: список целых чисел. Можно ставить `batch_size=50000` без проблем.

### Транзакции и долгие итерации

Долгая транзакция с открытым соединением блокирует VACUUM в Postgres и ведёт к раздуванию таблицы. Долгая транзакция через PgBouncer не возвращает соединение в пул, что забивает пул. Поэтому каждый батч идёт в своей короткой транзакции (по умолчанию Django так и делает в режиме `AUTOCOMMIT`).

Если внутри батча есть несколько изменений, оборачивай их в `atomic` (определение `chunked` смотри в разделе "Генераторы, islice и батчирование"):

```python
for batch_ids_chunk in chunked(iterate_pending_order_ids(), 500):
    with transaction.atomic():
        for current_order_id in batch_ids_chunk:
            process_and_update_order(current_order_id)
```

Для очень тяжёлой обработки выноси работу в Celery и в очередь кладите только id:

```python
for current_order_id in iterate_pending_order_ids():
    process_order.delay(order_id=current_order_id)
```

### Аналитические выборки на реплике

Если действительно нужно прочитать гигантский набор как один запрос (отчёт, экспорт), это идёт на реплику, не через PgBouncer основной нагрузки, и не через Django. Варианты:
- Postgres `COPY ... TO STDOUT` напрямую через psql или скрипт с прямым подключением.
- Materialized view с обновлением по расписанию.
- Прямое подключение к реплике в обход PgBouncer с включёнными server-side cursors.

## Транзакции и блокировки

### atomic

```python
from django.db import transaction

with transaction.atomic():
    order = Order.objects.create(...)
    Payment.objects.create(order=order, ...)
    update_inventory(order)
```

Если внутри блока возникнет исключение, всё откатится. Декоратор `@transaction.atomic` делает то же для всей функции.

### Savepoints

Вложенные `atomic` создают savepoint. Внутренний блок может откатиться без отката внешнего:

```python
with transaction.atomic():
    create_main_record()
    try:
        with transaction.atomic():
            create_optional_record()
    except IntegrityError:
        log_warning_about_duplicate()
```

### on_commit

Отправлять задачу в Celery надо после коммита, иначе воркер может прочитать заказ, которого ещё нет в БД.

```python
from django.db import transaction


def create_order_and_notify(customer, items):
    """
    Создаёт заказ и ставит задачу уведомления после коммита.

    :param customer: пользователь, оформляющий заказ.
    :param items: позиции заказа.
    :return: созданный заказ.
    """
    with transaction.atomic():
        new_order = Order.objects.create(customer=customer)
        OrderItem.objects.bulk_create(
            [OrderItem(order=new_order, **item) for item in items]
        )
        transaction.on_commit(lambda: send_notification.delay(new_order.id))
    return new_order
```

Колбэк `on_commit` вызывается после успешного COMMIT. При откате не вызывается. В тестах с `@pytest.mark.django_db` без `transaction=True` колбэк не сработает, потому что транзакция не коммитится.

### select_for_update пессимистичная блокировка

```python
from django.db import transaction


def deduct_stock(product_id: int, quantity: int) -> None:
    """
    Списывает товар со склада с блокировкой строки.

    :param product_id: идентификатор товара.
    :param quantity: количество к списанию.
    :return: None.
    :raises InsufficientStockError: если на складе недостаточно товара.
    """
    with transaction.atomic():
        product = Product.objects.select_for_update().get(pk=product_id)
        if product.stock < quantity:
            raise InsufficientStockError()
        product.stock -= quantity
        product.save(update_fields=["stock"])
```

`SELECT ... FOR UPDATE` блокирует строку до конца транзакции. Другие транзакции, пытающиеся прочитать ту же строку с FOR UPDATE, ждут.

Опции:
- `select_for_update(nowait=True)` бросает исключение, если строка занята.
- `select_for_update(skip_locked=True)` пропускает занятые строки. Полезно для очередей в Postgres.
- `select_for_update(of=("self",))` блокирует только основную таблицу, не связанные.

`select_for_update` работает только внутри `atomic`.

### Очередь задач через skip_locked

Postgres как очередь без RabbitMQ:

```python
from django.db import transaction


def claim_next_job():
    """
    Забирает следующую задачу из очереди, пропуская занятые.

    :return: захваченная задача или None.
    """
    with transaction.atomic():
        next_job = (
            Job.objects
            .select_for_update(skip_locked=True)
            .filter(status="pending")
            .order_by("priority", "created_at")
            .first()
        )
        if next_job is None:
            return None
        next_job.status = "in_progress"
        next_job.save(update_fields=["status"])
        return next_job
```

`skip_locked` гарантирует, что воркеры не дерутся за одну задачу.

### Оптимистичная блокировка через version поле

```python
from django.db.models import F


def update_order_optimistically(order_id: int, new_status: str, expected_version: int) -> bool:
    """
    Обновляет заказ при условии, что версия не изменилась.

    :param order_id: идентификатор заказа.
    :param new_status: новый статус.
    :param expected_version: ожидаемая версия записи.
    :return: True если обновление прошло, False если версия устарела.
    """
    updated_count = Order.objects.filter(
        pk=order_id,
        version=expected_version,
    ).update(
        status=new_status,
        version=F("version") + 1,
    )
    return updated_count == 1
```

Без блокировок. Если другая транзакция обновила запись первой, наш UPDATE вернёт ноль строк. На очень конкурентных нагрузках это масштабируется лучше FOR UPDATE.

### Уровни изоляции

Postgres по умолчанию `READ COMMITTED`. Для денежных операций иногда нужен `SERIALIZABLE` или `REPEATABLE READ`.

```python
from django.db import transaction


@transaction.atomic
def transfer_funds(from_account_id: int, to_account_id: int, amount: Decimal) -> None:
    """
    Переводит деньги между счетами в SERIALIZABLE изоляции.

    :param from_account_id: счёт отправителя.
    :param to_account_id: счёт получателя.
    :param amount: сумма перевода.
    :return: None.
    """
    with transaction.atomic():
        from django.db import connection
        with connection.cursor() as database_cursor:
            database_cursor.execute(
                "SET TRANSACTION ISOLATION LEVEL SERIALIZABLE"
            )
        execute_transfer(from_account_id, to_account_id, amount)
```

`SERIALIZABLE` ловит конфликты сериализации и кидает `SerializationFailure`. Код должен ретраить такую транзакцию.

## Raw SQL когда нужен

ORM хорош, но не покрывает всё. CTE, LATERAL JOIN, RECURSIVE, специфичные оконные конструкции иногда проще написать сразу на SQL.

### Model.objects.raw()

```python
top_customers = Customer.objects.raw(
    """
    SELECT customer.*, SUM(orders.total_amount) AS calculated_revenue
    FROM customers customer
    JOIN orders ON orders.customer_id = customer.id
    WHERE orders.status = %s
    GROUP BY customer.id
    ORDER BY calculated_revenue DESC
    LIMIT %s
    """,
    ["paid", 100],
)

for customer in top_customers:
    revenue_value = customer.calculated_revenue
```

Возвращает экземпляры модели. Доступны все методы модели.

### connection.cursor для произвольного SQL

```python
from django.db import connection


def get_orders_per_day_with_cte(start_date, end_date):
    """
    Считает заказы в день с заполнением пропусков через CTE generate_series.

    :param start_date: начало периода.
    :param end_date: конец периода.
    :return: список словарей с днём и количеством заказов.
    """
    sql_query = """
        WITH date_series AS (
            SELECT generate_series(%s::date, %s::date, '1 day'::interval)::date AS report_date
        )
        SELECT
            date_series.report_date,
            COALESCE(COUNT(orders.id), 0) AS orders_count
        FROM date_series
        LEFT JOIN orders ON DATE(orders.created_at) = date_series.report_date
        GROUP BY date_series.report_date
        ORDER BY date_series.report_date
    """
    with connection.cursor() as database_cursor:
        database_cursor.execute(sql_query, [start_date, end_date])
        column_names = [column.name for column in database_cursor.description]
        return [
            dict(zip(column_names, row_values))
            for row_values in database_cursor.fetchall()
        ]
```

Правила:
- Параметры передаются всегда через `%s`. Никогда не склеивай SQL строки руками.
- Имена таблиц и столбцов в SQL не передаются как параметры. Их валидируй белым списком в Python.

### RawSQL и RawSQL внутри ORM

Иногда нужен фрагмент SQL внутри annotate:

```python
from django.db.models.expressions import RawSQL

orders_with_custom_sort = Order.objects.annotate(
    custom_sort_key=RawSQL(
        "CASE WHEN status = %s THEN 1 ELSE 2 END",
        ["urgent"],
    ),
).order_by("custom_sort_key", "-created_at")
```

Используй редко. Большинство случаев покрываются `Case/When`.

## Индексы

Запрос без индекса на большой таблице это полный seq scan. Индексы решают.

### Объявление в Meta

```python
from django.db import models
from django.contrib.postgres.indexes import (
    BTreeIndex, GinIndex, GistIndex, BrinIndex, HashIndex,
)


class Order(models.Model):
    customer = models.ForeignKey("Customer", on_delete=models.PROTECT)
    status = models.CharField(max_length=20)
    total_amount = models.DecimalField(max_digits=10, decimal_places=2)
    created_at = models.DateTimeField(auto_now_add=True)
    metadata = models.JSONField(default=dict)
    tags = ArrayField(models.CharField(max_length=50), default=list)

    class Meta:
        indexes = [
            models.Index(fields=["status", "-created_at"]),
            models.Index(
                fields=["customer", "created_at"],
                name="orders_customer_recent_idx",
                condition=models.Q(status__in=["pending", "paid"]),
            ),
            GinIndex(fields=["metadata"]),
            GinIndex(fields=["tags"]),
            BrinIndex(fields=["created_at"]),
        ]
```

### Виды индексов в Postgres

- **B-tree** дефолтный, для равенства, диапазонов, сортировки. Покрывает 95 процентов случаев.
- **GIN** для массивов, JSONB, полнотекстового поиска, trigram. Записывается медленно, читается быстро.
- **GIST** для геоданных, диапазонов, trigram.
- **BRIN** для очень больших таблиц, отсортированных по индексируемому полю (типично created_at). Занимает в тысячи раз меньше места, чем B-tree.
- **HASH** только для равенства. Редко имеет смысл, B-tree почти всегда не хуже.

### Композитный индекс и порядок столбцов

Композитный индекс `(status, created_at)` работает для:
- WHERE status = 'paid',
- WHERE status = 'paid' AND created_at > '2024-01-01',
- WHERE status = 'paid' ORDER BY created_at.

Не работает для:
- WHERE created_at > '2024-01-01' (без условия на status).

Самое селективное поле или поле для равенства идёт первым. Поле для диапазона или сортировки последним.

### Partial index с condition

Если 99 процентов заказов в статусе `completed`, индекс на статус не поможет искать редкие статусы. Partial index решает:

```python
models.Index(
    fields=["created_at"],
    name="orders_pending_idx",
    condition=models.Q(status="pending"),
)
```

Индекс содержит только pending заказы. Маленький, быстрый, эффективен для запросов с этим условием.

### Functional index

```python
from django.db.models.functions import Lower

models.Index(
    Lower("email"),
    name="users_email_lower_idx",
)
```

Поддерживает запросы `WHERE LOWER(email) = ...` и нечувствительный к регистру поиск.

### Covering index с include

PostgreSQL 11+ поддерживает covering indexes:

```python
models.Index(
    fields=["customer_id", "status"],
    include=["total_amount", "created_at"],
    name="orders_customer_status_covering_idx",
)
```

Дополнительные поля хранятся в индексе, но не участвуют в условиях. Запрос может полностью обслужиться индексом без обращения к таблице (index-only scan).

### Когда индекс лишний

Каждый индекс замедляет INSERT, UPDATE, DELETE на эту таблицу. На таблице с миллиардами вставок в день нельзя ставить десять индексов. Индексы ставятся под конкретные запросы. Неиспользуемые индексы убиваем.

Проверка использования:

```sql
SELECT
    indexrelname AS index_name,
    idx_scan AS scans
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan ASC;
```

Индексы с нулевым `idx_scan` за месяц можно дропать.

### CREATE INDEX CONCURRENTLY

Обычный CREATE INDEX блокирует таблицу. На продакшене используем CONCURRENTLY:

```python
from django.db import migrations
from django.contrib.postgres.operations import AddIndexConcurrently
from django.db.models import Index


class Migration(migrations.Migration):
    atomic = False

    dependencies = [...]

    operations = [
        AddIndexConcurrently(
            model_name="order",
            index=Index(fields=["customer_id", "created_at"], name="orders_customer_created_idx"),
        ),
    ]
```

`atomic = False` обязателен. CONCURRENTLY не работает в транзакции.

## Postgres специфика

### ArrayField

```python
from django.contrib.postgres.fields import ArrayField


class Article(models.Model):
    title = models.CharField(max_length=200)
    tags = ArrayField(models.CharField(max_length=50), default=list)
```

Запросы:

```python
articles_with_python_tag = Article.objects.filter(tags__contains=["python"])
articles_with_any_tag = Article.objects.filter(tags__overlap=["python", "django"])
articles_with_exact_tags = Article.objects.filter(tags=["python", "django"])
articles_with_three_tags = Article.objects.filter(tags__len=3)
articles_with_first_tag = Article.objects.filter(tags__0="python")
```

Для эффективности обязателен GIN индекс.

### JSONField

```python
class Event(models.Model):
    payload = models.JSONField(default=dict)


events_with_user_id = Event.objects.filter(payload__user_id=42)
events_with_nested = Event.objects.filter(payload__metadata__source="web")
events_with_key = Event.objects.filter(payload__has_key="email")
events_with_keys = Event.objects.filter(payload__has_keys=["email", "phone"])
events_with_any = Event.objects.filter(payload__has_any_keys=["email", "phone"])
events_contained = Event.objects.filter(payload__contains={"status": "active"})
```

GIN индекс с операторами `jsonb_path_ops` для contains запросов:

```python
GinIndex(fields=["payload"], opclasses=["jsonb_path_ops"], name="events_payload_gin_idx")
```

Извлечение значения:

```python
from django.db.models.expressions import RawSQL

events_with_extracted = Event.objects.annotate(
    user_email=RawSQL("payload ->> 'email'", []),
)
```

### Postgres-специфичные агрегаты

```python
from django.contrib.postgres.aggregates import (
    ArrayAgg, StringAgg, JSONBAgg, BoolAnd, BoolOr,
)

customer_orders_array = Customer.objects.annotate(
    order_ids=ArrayAgg("orders__id", filter=Q(orders__status="paid")),
    order_statuses_csv=StringAgg("orders__status", delimiter=","),
    orders_data=JSONBAgg("orders__id"),
    all_paid=BoolAnd("orders__status", filter=Q(orders__status="paid")),
)
```

`ArrayAgg(distinct=True, ordering="-created_at")` поддерживает уникальность и сортировку внутри агрегата.

### Полнотекстовый поиск

```python
from django.contrib.postgres.search import (
    SearchVector, SearchQuery, SearchRank, SearchHeadline,
)

search_query = SearchQuery("python tutorial", config="english")

articles_search_results = Article.objects.annotate(
    search_vector=SearchVector("title", weight="A") + SearchVector("body", weight="B"),
    rank=SearchRank(F("search_vector"), search_query),
    headline=SearchHeadline("body", search_query),
).filter(search_vector=search_query).order_by("-rank")
```

Для производительности храни SearchVector в отдельном поле с GIN индексом и обновляй триггером:

```python
from django.contrib.postgres.search import SearchVectorField
from django.contrib.postgres.indexes import GinIndex


class Article(models.Model):
    title = models.CharField(max_length=200)
    body = models.TextField()
    search_vector = SearchVectorField(null=True)

    class Meta:
        indexes = [GinIndex(fields=["search_vector"])]
```

### Trigram для нечёткого поиска

```python
from django.contrib.postgres.search import TrigramSimilarity

products_fuzzy_match = Product.objects.annotate(
    similarity=TrigramSimilarity("name", user_input),
).filter(similarity__gt=0.3).order_by("-similarity")
```

Требует расширения `pg_trgm` и GIN индекса с `gin_trgm_ops`:

```python
from django.contrib.postgres.indexes import GinIndex

GinIndex(fields=["name"], name="products_name_trgm_idx", opclasses=["gin_trgm_ops"])
```

## EXPLAIN и профилирование

### QuerySet.explain

```python
queryset = Order.objects.filter(status="paid").select_related("customer")
print(queryset.explain(analyze=True, buffers=True, verbose=True))
```

Ключи:
- `analyze=True` реально выполняет запрос и показывает фактическое время.
- `buffers=True` показывает обращения к buffer cache.
- `verbose=True` подробный вывод.
- `format="json"` или `"yaml"` для машинной обработки.

### Что искать в плане

- `Seq Scan` на большой таблице это плохо. Нужен индекс.
- `Bitmap Heap Scan` нормально, но если `Recheck Cond` отбраковывает много строк, индекс плохо подходит.
- `Index Scan` хорошо. `Index Only Scan` идеально.
- `Nested Loop` хорош для маленьких выборок, плох для больших. Если строк много, должен быть `Hash Join` или `Merge Join`.
- `Sort` с `external merge Disk` означает, что Postgres не уместил сортировку в память. Увеличить `work_mem` или добавить индекс с правильным порядком.
- `rows estimated` сильно расходится с `rows actual` это устаревшая статистика. `ANALYZE table_name`.

### Логирование запросов в dev

```python
LOGGING = {
    "version": 1,
    "loggers": {
        "django.db.backends": {
            "level": "DEBUG",
            "handlers": ["console"],
        },
    },
    "handlers": {
        "console": {"class": "logging.StreamHandler"},
    },
}
```

В продакшене это убивает производительность. Используем `django-silk` или сэмплирование.

### connection.queries в тестах

```python
from django.db import connection, reset_queries
from django.test.utils import override_settings


@override_settings(DEBUG=True)
def measure_queries():
    """
    Считает запросы в блоке кода.

    :return: список SQL запросов.
    """
    reset_queries()
    do_some_work()
    return connection.queries
```

В тестах удобнее `django_assert_num_queries`:

```python
with django_assert_num_queries(3):
    list(Order.objects.select_related("customer").filter(status="paid"))
```

### PgBouncer в режиме transaction pooling

В нашем проекте между Django и Postgres стоит PgBouncer в режиме `transaction pooling`. Это означает: PgBouncer выдаёт серверное соединение клиенту на одну транзакцию. После `COMMIT` или `ROLLBACK` соединение возвращается в пул и достаётся следующему клиенту.

Это даёт огромную экономию: 1000 Django воркеров могут шарить 50 серверных соединений к Postgres. Но накладывает ограничения на то, что можно делать.

#### Настройки Django под PgBouncer

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "mydb",
        "HOST": "pgbouncer.internal",
        "PORT": 6432,
        "CONN_MAX_AGE": 0,
        "DISABLE_SERVER_SIDE_CURSORS": True,
        "OPTIONS": {
            "options": "-c statement_timeout=30000",
        },
    }
}
```

Разбор:

- `CONN_MAX_AGE = 0` обязательно. Persistent connections в Django поверх PgBouncer убивают весь смысл пула. Каждое Django соединение к PgBouncer удерживает место в его пуле и не отдаёт.
- `DISABLE_SERVER_SIDE_CURSORS = True` обязательно. Server-side cursors живут в сессии. Между транзакциями PgBouncer передаст соединение другому клиенту, и cursor либо потеряется, либо станет утечкой.
- `statement_timeout` защита от запросов, которые держат соединение часами.

#### Что не работает или ломается

- **Server-side cursors.** Уже разобрали в разделе про стриминг. `iterator()` работает как обычный SELECT.
- **Prepared statements.** Они кэшируются в сессии. На transaction pooling после возврата соединения prepared statement пропадает. psycopg3 умеет их правильно готовить заново (`prepare_threshold`), но проще выключить целиком.
- **`LISTEN/NOTIFY`.** Требуют постоянной сессии. Не работают через PgBouncer transaction pooling. Если нужны pub/sub, идём в Redis или RabbitMQ.
- **Session-level advisory locks.** `pg_advisory_lock(key)` без префикса `xact` берёт лок до конца сессии. После возврата соединения в пул лок останется висеть на чужом коннекте. Используем только `pg_advisory_xact_lock(key)`, который автоматически снимается на `COMMIT`.
- **Обычный `SET`.** Меняет настройку для сессии. Следующий клиент, получивший то же соединение, унаследует. Используем только `SET LOCAL`, который действует до конца транзакции.
- **Временные таблицы (`CREATE TEMP TABLE`)** живут в сессии. На transaction pooling следующий клиент увидит чужие временные таблицы. Используем `CREATE TEMP TABLE ... ON COMMIT DROP`.
- **Cursor-based ResultSet возвраты из хранимых процедур** (типа `OPEN cursor`) тоже привязаны к сессии.

#### Что работает

- Обычные SELECT, INSERT, UPDATE, DELETE.
- `BEGIN ... COMMIT` блоки.
- `pg_advisory_xact_lock` для распределённых блокировок.
- `SET LOCAL search_path = ...` и подобное внутри транзакции.
- `CREATE TEMP TABLE ... ON COMMIT DROP` внутри транзакции.

#### Правильные advisory locks через PgBouncer

```python
from django.db import connection, transaction


def with_advisory_lock(lock_key: int) -> None:
    """
    Захватывает advisory lock на время текущей транзакции.

    :param lock_key: целочисленный ключ блокировки.
    :return: None.
    :raises RuntimeError: если вызван вне транзакции.
    """
    if not connection.in_atomic_block:
        raise RuntimeError("Advisory lock requires an active transaction")
    with connection.cursor() as database_cursor:
        database_cursor.execute("SELECT pg_advisory_xact_lock(%s)", [lock_key])


@transaction.atomic
def process_with_distributed_lock(lock_key: int, target_id: int) -> None:
    """
    Выполняет работу под распределённой блокировкой.

    :param lock_key: ключ блокировки.
    :param target_id: id цели обработки.
    :return: None.
    """
    with_advisory_lock(lock_key)
    do_critical_work(target_id)
```

`pg_advisory_xact_lock` снимется автоматически на коммите. PgBouncer вернёт чистое соединение в пул.

#### Длинные транзакции это враг

В transaction pooling соединение возвращается в пул только после `COMMIT`. Транзакция длиной в минуту удерживает серверное соединение всю эту минуту. На пуле в 50 соединений 50 одновременных длинных транзакций исчерпают пул, и весь сервис встанет.

Правила:
- Никогда не делать сетевые вызовы внутри `transaction.atomic`.
- Никогда не итерировать большой queryset внутри `atomic`.
- Тяжёлые отчёты с долгим подсчётом не должны быть в транзакциях с записью.
- Внешние HTTP вызовы выносим за пределы транзакции, использовать `transaction.on_commit` для пост-обработки.

#### Мониторинг пула

Что смотрим в `SHOW POOLS` PgBouncer:
- `cl_waiting` клиенты в очереди на соединение. Если стабильно больше нуля, пул мал или транзакции долгие.
- `sv_active` активные серверные соединения. Если близко к `pool_size`, упираемся в потолок.
- `maxwait` максимальное время ожидания клиента. Если растёт, проблема.

### CONN_MAX_AGE без PgBouncer

Если в каком-то отдельном сервисе (например, миграционный скрипт) используется прямое подключение к Postgres без PgBouncer:

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "mydb",
        "HOST": "postgres-primary.internal",
        "PORT": 5432,
        "CONN_MAX_AGE": 60,
        "CONN_HEALTH_CHECKS": True,
    }
}
```

`CONN_MAX_AGE` держит соединение живым между запросами. `CONN_HEALTH_CHECKS=True` проверяет его перед использованием. Защищает от мёртвых соединений после рестарта Postgres.

### Multiple databases и роутинг

```python
DATABASES = {
    "default": {...},
    "replica": {...},
    "analytics": {...},
}

DATABASE_ROUTERS = ["myapp.routers.PrimaryReplicaRouter"]


class PrimaryReplicaRouter:
    """
    Роутер для разделения чтения и записи между primary и replica.
    """

    def db_for_read(self, model, **hints):
        """
        Возвращает базу для чтения.

        :param model: модель Django.
        :param hints: подсказки роутинга.
        :return: имя базы.
        """
        return "replica"

    def db_for_write(self, model, **hints):
        """
        Возвращает базу для записи.

        :param model: модель Django.
        :param hints: подсказки роутинга.
        :return: имя базы.
        """
        return "default"

    def allow_relation(self, first_object, second_object, **hints):
        """
        Разрешает связи между объектами.

        :param first_object: первый объект.
        :param second_object: второй объект.
        :param hints: подсказки.
        :return: True если связь разрешена.
        """
        return True

    def allow_migrate(self, db_name, app_label, model_name=None, **hints):
        """
        Разрешает миграции.

        :param db_name: имя базы.
        :param app_label: приложение.
        :param model_name: модель.
        :param hints: подсказки.
        :return: True если миграция разрешена.
        """
        return db_name == "default"
```

Явное переключение через `using`:

```python
Order.objects.using("replica").filter(status="paid")
```

Внимание: чтение с реплики после записи на primary может вернуть старые данные из-за репликационной задержки. После критичной записи читай с primary.

## Антипаттерны и трюки

### count vs exists vs len

```python
if Order.objects.filter(status="paid").count() > 0: ...
if Order.objects.filter(status="paid").exists(): ...
```

`exists()` останавливается на первой строке. `count()` пробегает весь набор. Разница на больших таблицах огромная.

```python
orders_list = list(Order.objects.filter(status="paid"))
len(orders_list)
Order.objects.filter(status="paid").count()
```

Если queryset уже материализован в Python, `len()` бесплатен. `count()` делает новый запрос.

### refresh_from_db

После `update()` через ORM или после внешнего изменения объект в Python устарел:

```python
order = Order.objects.get(pk=42)
Order.objects.filter(pk=42).update(status="paid")
order.refresh_from_db(fields=["status"])
```

`fields` ограничивает что перечитываем, чтобы не вытягивать всю запись.

### update_or_create vs get_or_create

```python
profile, was_created = Profile.objects.get_or_create(
    user_id=user_id,
    defaults={"timezone": "UTC"},
)

profile, was_created = Profile.objects.update_or_create(
    user_id=user_id,
    defaults={"timezone": "Europe/Berlin"},
)
```

`get_or_create` либо создаёт, либо возвращает существующий без изменений. `update_or_create` либо создаёт, либо обновляет существующий.

Оба не атомарны на уровне SQL без уникального индекса. При гонке двух процессов возможен `IntegrityError`. Оборачивай в try/except или используй `bulk_create(update_conflicts=True)`.

### .first() vs [:1] vs [0]

```python
first_order = Order.objects.filter(status="paid").first()
first_order = Order.objects.filter(status="paid").order_by("id")[:1].first()
first_order = Order.objects.filter(status="paid")[0]
```

`first()` без `order_by` возвращает любую строку, что часто баг. `[0]` бросает `IndexError` на пустой выборке. Безопасно: `.order_by("id").first()`.

### in_bulk

```python
products_by_id = Product.objects.in_bulk([1, 2, 3, 4, 5])
selected_product = products_by_id[3]
```

Возвращает словарь `{pk: instance}`. Удобно для джойнов на стороне Python.

С полем кроме pk:

```python
products_by_sku = Product.objects.in_bulk(["SKU-A", "SKU-B"], field_name="sku")
```

### Удаление миллиона строк

`Order.objects.filter(...).delete()` загружает все строки в память, обходит сигналы, обрабатывает каскады в Python. На миллионе строк это часы.

Альтернативы:

1. Если нет каскадов и сигналов: один сырой `DELETE` через `connection.cursor`.
2. Батчами через `while` с проверкой пустого результата (полный пример в разделе "Генераторы, islice и батчирование").
3. Для очень больших объёмов: партиционирование таблицы по дате и `DROP PARTITION`.

```python
from django.db import connection

with connection.cursor() as database_cursor:
    database_cursor.execute(
        "DELETE FROM orders WHERE status = 'cancelled' AND created_at < %s",
        [cutoff],
    )
```

### only ловушка с N+1

```python
orders_only_id = Order.objects.only("id")
for order in orders_only_id:
    print(order.status)
```

`order.status` не загружен через `only`. Каждое обращение делает отдельный SELECT. Это N+1 в чистом виде. `only` использовать осознанно.

### values + annotate ловушка с лишним GROUP BY

```python
revenue_by_status = (
    Order.objects
    .values("status")
    .annotate(total=Sum("total_amount"))
)
```

Это `GROUP BY status`, что мы и хотим. Но если случайно добавить `.values("id", "status")`, GROUP BY станет по двум полям, и группировки не будет.

Порядок имеет значение: `values()` ДО `annotate()` создаёт GROUP BY, ПОСЛЕ это просто проекция.

### distinct на разных диалектах

`distinct()` без аргументов это `SELECT DISTINCT`. С аргументами `distinct("field")` это Postgres-only `DISTINCT ON (field)`. Возвращает первую строку для каждого уникального значения поля.

```python
latest_order_per_customer = (
    Order.objects
    .order_by("customer_id", "-created_at")
    .distinct("customer_id")
)
```

Поле в `distinct` должно быть первым в `order_by`. Это требование Postgres.

### Избегай union без необходимости

```python
combined = queryset_a.union(queryset_b)
```

`UNION` дедуплицирует. Если дубликатов быть не может, `union(all=True)` это `UNION ALL`, в разы быстрее.

Лучшая альтернатива: один queryset с OR в фильтре.

### Q-фильтр с пустым списком

```python
matching_orders = Order.objects.filter(id__in=potentially_empty_list)
```

Если `potentially_empty_list` пустой, Django делает запрос `WHERE id IN ()` который Postgres не любит. Защита:

```python
if potentially_empty_list:
    matching_orders = Order.objects.filter(id__in=potentially_empty_list)
else:
    matching_orders = Order.objects.none()
```

`Model.objects.none()` это пустой queryset без обращения к БД.

### __in с большим списком

`WHERE id IN (...)` на 100 тысячах значений работает плохо. Postgres строит хеш-таблицу из IN списка, расход памяти растёт.

Решения:
- разбить на батчи по 1000-5000,
- временная таблица + JOIN,
- Postgres specific: `ANY(ARRAY[...])`.

### prefetch_related дубль через related_query_name

Если ту же связь читаешь и через `prefetch_related`, и через annotate, Django может сделать запрос дважды. Избегай:

```python
from django.db.models import Prefetch

orders_with_items = Order.objects.prefetch_related(
    Prefetch("items", queryset=OrderItem.objects.select_related("product")),
).annotate(items_count=Count("items"))
```

`Count("items")` делает JOIN, `prefetch_related` делает отдельный запрос. Два разных пути к items, оба валидны.

### connection.cursor для полностью кастомных запросов

Когда нужен SQL, который Django не построит:
- RECURSIVE CTE для дерева,
- LATERAL JOIN для топ N в группе,
- MERGE для сложного upsert,
- хранимая процедура,
- `pg_advisory_xact_lock` для распределённой блокировки на время транзакции.

```python
from django.db import connection, transaction


def try_acquire_xact_lock(lock_key: int) -> bool:
    """
    Пробует захватить advisory блокировку на время текущей транзакции.

    :param lock_key: целочисленный ключ блокировки.
    :return: True если блокировка захвачена, False если занята.
    :raises RuntimeError: если вызван вне транзакции.
    """
    if not connection.in_atomic_block:
        raise RuntimeError("Advisory xact lock requires an active transaction")
    with connection.cursor() as database_cursor:
        database_cursor.execute("SELECT pg_try_advisory_xact_lock(%s)", [lock_key])
        result_row = database_cursor.fetchone()
        return result_row[0]


@transaction.atomic
def process_with_lock_or_skip(lock_key: int, target_id: int) -> bool:
    """
    Выполняет работу под блокировкой, если она свободна.

    :param lock_key: ключ блокировки.
    :param target_id: id обрабатываемой цели.
    :return: True если работа выполнена, False если блокировка занята.
    """
    if not try_acquire_xact_lock(lock_key):
        return False
    do_critical_work(target_id)
    return True
```

Вариант с `xact` снимется автоматически на коммите. Session-level `pg_try_advisory_lock` через PgBouncer transaction pooling использовать нельзя (см. раздел Connection management).

## Чек-лист производительности запроса

Перед коммитом запроса в продакшен:

- [ ] План запроса (`EXPLAIN ANALYZE`) использует индекс, а не Seq Scan.
- [ ] Нет N+1: `select_related` для FK, `prefetch_related` для обратных и M2M.
- [ ] Выгружается минимум полей через `only`, `values` или `JSONObject`.
- [ ] Для чтения миллионов строк используется keyset пагинация. `iterator()` не считается стримингом, потому что server-side cursors отключены.
- [ ] Для записи больших объёмов используется `bulk_create` с `batch_size`, или COPY.
- [ ] Атомарные обновления через `F` без race condition.
- [ ] Запросы в цикле проверены на N+1 через `django_assert_num_queries`.
- [ ] Long-running транзакции разбиты на короткие батчи. Никаких сетевых вызовов внутри `atomic`.
- [ ] `select_for_update` обёрнут в `atomic` и использует `skip_locked` или `nowait`, где это уместно.
- [ ] Каскадное удаление миллионов строк заменено на raw DELETE или партиционирование.
- [ ] Индексы под новые запросы добавлены через `CONCURRENTLY`.
- [ ] Тяжёлые отчётные запросы вынесены на реплику или в материализованное представление.
- [ ] Источник данных для bulk операций это генератор, не список. Источник нарезается на батчи через `chunked` или `itertools.batched`.
- [ ] Bulk обработка очереди или удаление миллионов строк идёт через `while True` с выходом по пустому батчу.
- [ ] Промежуточные выражения для filter и order_by вынесены в `alias`, не в `annotate`.
- [ ] Фильтрация по результату другого запроса использует subquery (`.values("id")`), а не `list(...)`.
- [ ] Advisory locks только `pg_advisory_xact_lock`, никогда не session-level.
- [ ] Группировка через `values().annotate()` проверена: в `values` только поля группировки, без случайных `id`.

## Краткая шпаргалка по выбору инструмента

| Задача | Инструмент |
|---|---|
| Загрузка нескольких связанных объектов через FK | `select_related` |
| Загрузка обратных связей и M2M | `prefetch_related` |
| Фильтр или сортировка внутри prefetch | `Prefetch(queryset=...)` |
| Атомарный инкремент или арифметика | `F("field") + value` |
| Сложные условия | `Q(...) \| Q(...)`, `~Q(...)` |
| Агрегация по группам (GROUP BY) | `.values(<поля группировки>).annotate(<агрегаты>)` |
| Фильтр по агрегату (HAVING) | `.annotate(...).filter(<имя аннотации>=...)` |
| Несколько метрик с разными условиями | агрегат с `filter=Q(...)` |
| Промежуточное выражение только для filter/order_by | `.alias(...)` |
| Условное значение в строке | `Case(When(..., then=...), default=...)` |
| Денормализованное поле из подзапроса | `Subquery + OuterRef` |
| Массив значений из подзапроса | `ArraySubquery(...)` |
| Проверка существования | `Exists(...)` |
| Отсутствие связанной строки | `~Exists(...)` (а не `NOT IN`) |
| JSON объект из полей в SELECT | `JSONObject(field1=F(...), ...)` |
| Массив JSON объектов вместо prefetch | `ArraySubquery(... .values(JSONObject(...)))` |
| Фильтр по результату другого запроса | `filter(id__in=other_queryset.values("id"))`, НЕ `list(...)` |
| Топ N в группе, скользящие, кумулятивные | `Window(expression=..., partition_by=..., order_by=..., frame=...)` |
| Bulk вставка | `bulk_create(batch_size=1000)` |
| Bulk вставка большого источника | генератор + `chunked(...)` или `itertools.batched(...)` |
| Bulk апсёрт | `bulk_create(update_conflicts=True, unique_fields=...)` |
| Bulk обновление одинаково | `update(field=F(...))` |
| Bulk обновление разными значениями | `bulk_update(fields=...)` |
| Обработка очереди или batch воркер | `while True` плюс выход по пустому батчу |
| Удаление миллионов строк | `while True` с `values_list("id")[:N]` плюс `delete()` |
| ETL из файла, API, внешнего источника | генератор-источник плюс генератор-трансформер плюс `chunked` плюс `bulk_create` |
| Постановка миллиона задач в Celery | батчирование `chunked` плюс одна задача на батч id |
| Чтение миллионов строк под PgBouncer | keyset пагинация по индексированному полю |
| Глубокая пагинация | keyset, никогда `OFFSET` |
| Пессимистичная блокировка | `select_for_update` внутри `atomic` |
| Очередь в Postgres | `select_for_update(skip_locked=True)` |
| Оптимистичная блокировка | поле version плюс UPDATE с проверкой |
| Распределённая блокировка через PgBouncer | `pg_advisory_xact_lock` внутри `atomic` |
| Полнотекстовый поиск | `SearchVector + SearchQuery` плюс GIN индекс |
| Нечёткий поиск | `TrigramSimilarity` плюс GIN индекс с trgm |
| Поиск по JSON | `payload__key`, GIN индекс с jsonb_path_ops |
| Массивы | `tags__contains`, `tags__overlap`, GIN индекс |
| Долгие транзакции | разбить на короткие через keyset, выкинуть сетевые вызовы |
| RECURSIVE, LATERAL, MERGE | `connection.cursor()` |
