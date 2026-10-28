# Единые названия и договоренности

Этот файл — источник истины для всех артефактов (карта контекстов, C4, OpenAPI, AsyncAPI, sequence, ADR). Если название где-то расходится с этим файлом, правим артефакт, а не файл. Если нужно новое название — сначала добавляем сюда.

---

## 1. Общие правила

| Что | Стиль | Пример |
|---|---|---|
| Названия сервисов в тексте и на диаграммах | Title Case | Order Service |
| Технические имена сервисов | kebab-case | `order-service` |
| Пути REST | kebab-case, существительные во мн. числе | `/api/v1/gift-orders` |
| Поля JSON | camelCase | `santaParticipantId` |
| Значения перечислений (статусы, коды ошибок) | UPPER_SNAKE_CASE | `PAYMENT_HELD` |
| Типы событий | `<источник>.<что-произошло>`, kebab-case, прошедшее время | `order.gift-ordered` |
| operationId в OpenAPI | camelCase, глагол + существительное | `createGiftOrder` |
| Идентификаторы | UUID, поле `id` или `<сущность>Id` | `groupId` |
| Дата и время | ISO 8601 в UTC, поле с суффиксом `At` | `"createdAt": "2026-12-20T10:00:00Z"` |
| Деньги | целое число в копейках + валюта | `{ "amount": 199000, "currency": "RUB" }` = 1990,00 ₽ |

Тексты (описания, абзацы, ADR) пишем по-русски, технические имена — по-английски.

---

## 2. Глоссарий

| Термин | В коде | Значение |
|---|---|---|
| Группа | `Group` | Одна игра в Тайного Санту: участники, бюджет, дедлайны |
| Организатор | `organizer` | Участник, создавший группу; закрывает регистрацию |
| Участник | `Participant` | Пользователь, вступивший в группу |
| Пользователь | `userId` | Аккаунт человека; аутентификация вне рамок системы, `userId` приходит из токена |
| Код приглашения | `inviteCode` | Код из ссылки, по которому вступают в группу |
| Бюджет | `budget` | Максимальная цена подарка в группе |
| Жеребьевка | `Draw` | Распределение пар внутри группы |
| Пара | `Pair` | Связка «Санта → подопечный» |
| Санта | `santa` | Участник, который дарит |
| Подопечный | `recipient` | Участник, который получает подарок |
| Кольцо | — | Пары образуют один замкнутый цикл, себе дарить нельзя |
| Перестройка пар | `reassignment` | Изменение пар после выхода участника |
| Заказ подарка | `GiftOrder` | Заказ подарка подопечному на маркетплейсе |
| Товар | `Product` | Позиция из каталога маркетплейса |
| Холд | `hold` | Заморозка суммы на карте Санты |
| Списание | `capture` | Фактическое списание замороженной суммы |
| Отмена холда | `release` | Разморозка суммы без списания |
| Возврат | `refund` | Возврат уже списанных денег |
| Способ оплаты | `paymentMethodToken` | Токен карты, выданный платежным провайдером (номер карты система не хранит) |
| Сага | `Saga` | Процесс заказа подарка из нескольких шагов с компенсациями |
| Компенсация | `compensation` | Действие, отменяющее уже выполненный шаг саги |
| Ключ идемпотентности | `Idempotency-Key` | Ключ запроса, по которому повтор не выполняется второй раз |

---

## 3. Сервисы и инфраструктура

| Название | Техническое имя | База данных | Ответственный |
|---|---|---|---|
| API Gateway | `api-gateway` | — | Анзор |
| Group Service | `group-service` | `group_db` | Анзор |
| Notification Service | `notification-service` | `notification_db` | Анзор |
| Draw Service | `draw-service` | `draw_db` | Маша |
| Payment Service | `payment-service` | `payment_db` | Маша |
| Order Service | `order-service` | `order_db` | Алиса |
| Брокер сообщений | `rabbitmq` | — | — |

Все базы — PostgreSQL. В базах `group_db`, `draw_db`, `order_db` есть таблица `outbox`; у всех потребителей событий есть таблица `processed_events` (дедупликация).

### Внешние системы

| Название | Техническое имя | Для чего |
|---|---|---|
| Платежный провайдер | `payment-provider` | Холд, списание, отмена холда, возврат |
| Маркетплейс | `marketplace` | Каталог товаров, заказы, доставка, вебхуки о статусе |
| Telegram Bot API | `telegram` | Уведомления в Telegram |
| Почтовый сервер (SMTP) | `smtp` | Уведомления на почту |

### Сокращения для sequence-диаграмм (Mermaid)

```
participant U as Пользователь
participant GW as API Gateway
participant GS as Group Service
participant DS as Draw Service
participant PS as Payment Service
participant OS as Order Service
participant NS as Notification Service
participant MQ as RabbitMQ
participant PP as Платежный провайдер
participant MP as Маркетплейс
```

---

## 4. Сущности, поля и статусы

### Group Service

**Group**: `id`, `name`, `organizerParticipantId`, `inviteCode`, `budget`, `registrationDeadline`, `giftDeadline`, `status`, `createdAt`

`GroupStatus`:
- `OPEN` — идет регистрация
- `REGISTRATION_CLOSED` — регистрация закрыта, ждем жеребьевку
- `DRAWN` — жеребьевка проведена, Санты выбирают подарки (Group Service узнает об этом из события `draw.completed`)
- `FINISHED` — наступил `giftDeadline`

Отмены игры нет: ее сознательно не стали делать (см. `defense.md`, граничный случай 2).

**Participant**: `id`, `groupId`, `userId`, `displayName`, `email`, `telegramChatId`, `deliveryAddress`, `status`, `joinedAt`, `leftAt`

`ParticipantStatus`: `ACTIVE`, `LEFT`

**DeliveryAddress** (вложенный объект): `postalCode`, `city`, `street`, `house`, `apartment`, `phone`

**Копия данных из событий**: `shipped_gifts` — `giftOrderId`, `groupId`, `santaParticipantId`, `recipientParticipantId` (из `order.gift-shipped`). По ней Group Service запрещает выход участникам, чей подарок уже в доставке. Из события `draw.completed` Group Service берет только `groupId` и переводит группу в `DRAWN`, пары не сохраняет.

### Draw Service

**Draw**: `id`, `groupId`, `status`, `createdAt`, `completedAt`

`DrawStatus`: `PENDING`, `COMPLETED`, `FAILED`

**Pair**: `id`, `drawId`, `groupId`, `santaParticipantId`, `recipientParticipantId`, `status`, `createdAt`

`PairStatus`: `ACTIVE`, `CANCELLED` (пара отменена при перестройке)

### Payment Service

**Payment**: `id`, `giftOrderId`, `payerParticipantId`, `amount`, `currency`, `providerPaymentId`, `status`, `createdAt`, `updatedAt`

`PaymentStatus`:
- `CREATED` — запись создана, провайдеру еще не отправлено
- `HELD` — сумма заморожена
- `CAPTURED` — сумма списана
- `RELEASED` — холд отменен
- `REFUNDED` — деньги возвращены после списания
- `FAILED` — провайдер отказал

### Order Service

**GiftOrder**: `id`, `groupId`, `santaParticipantId`, `recipientParticipantId`, `productId`, `productName`, `price`, `paymentId`, `marketplaceOrderId`, `status`, `failureReason`, `sagaStep`, `createdAt`, `updatedAt`

`GiftOrderStatus`:
- `PENDING` — заказ создан
- `PAYMENT_HELD` — деньги заморожены
- `ORDERED` — маркетплейс принял заказ
- `PAID` — деньги списаны
- `SHIPPED` — подарок передан в доставку
- `DELIVERED` — подарок доставлен
- `FAILED` — сага не удалась, компенсации выполнены
- `CANCELLED` — заказ отменен (например, участник вышел)

`FailureReason`: `OUT_OF_STOCK`, `PAYMENT_DECLINED`, `PRICE_EXCEEDS_BUDGET`, `MARKETPLACE_UNAVAILABLE`, `PAYMENT_PROVIDER_UNAVAILABLE`, `PARTICIPANT_LEFT`, `PRICE_CHANGED`, `TIMEOUT`

`SagaStep` (шаги саги в порядке выполнения):
1. `HOLD_PAYMENT` — заморозить деньги
2. `GET_RECIPIENT_ADDRESS` — получить адрес подопечного
3. `PLACE_MARKETPLACE_ORDER` — оформить заказ на маркетплейсе
4. `CAPTURE_PAYMENT` — списать деньги
5. `COMPLETED`

Компенсации: `RELEASE_PAYMENT` (отменить холд), `CANCEL_MARKETPLACE_ORDER` (отменить заказ на маркетплейсе), `REFUND_PAYMENT` (вернуть списанные деньги).

**Копии данных из событий** (локальные таблицы Order Service): `group_members` и `group_budgets` (из `group.registration-closed`), `assignments` — текущие пары (из `draw.completed` и `draw.reassigned`).

### Notification Service

**Notification**: `id`, `participantId`, `channel`, `type`, `status`, `sourceEventId`, `createdAt`, `sentAt`

`NotificationChannel`: `EMAIL`, `TELEGRAM`

`NotificationType`:
- `WELCOME` — вы вступили в группу
- `DRAW_RESULT` — ваш подопечный: …
- `RECIPIENT_CHANGED` — ваш подопечный изменился
- `GIFT_ORDERED` — Санте: заказ оформлен
- `GIFT_ON_THE_WAY` — подопечному: к вам едет подарок
- `GIFT_DELIVERED` — подопечному: подарок доставлен
- `CHOOSE_ANOTHER_GIFT` — Санте: заказ не удался, выберите другой подарок
- `GIFT_CANCELLED` — заказ отменен

`NotificationStatus`: `PENDING`, `SENT`, `FAILED`

**Копия данных из событий**: `contacts` — контакты участников (из `group.participant-joined`), чтобы уведомления работали, даже когда Group Service недоступен.

---

## 5. REST API (для OpenAPI)

Внешние запросы идут через API Gateway с префиксом `/api/v1`. Вызовы между сервисами — с префиксом `/internal`, через Gateway они недоступны.

### Заголовки

| Заголовок | Где | Зачем |
|---|---|---|
| `Authorization: Bearer <token>` | Внешние запросы | Аутентификация (проверяет Gateway) |
| `X-User-Id` | Gateway → сервисы | `userId` из токена |
| `X-Request-Id` | Все запросы | Сквозной идентификатор для логов |
| `Idempotency-Key` | Все POST, которые создают деньги или заказы | Защита от повторов |

Ключи идемпотентности, которые формирует Order Service: `hold-{giftOrderId}` (холд), `capture-{giftOrderId}`, `release-{giftOrderId}`, `refund-{giftOrderId}`, `order-{giftOrderId}` (заказ на маркетплейсе). Ключ для `createGiftOrder` генерирует клиент (UUID).

### Group Service

| Метод и путь | operationId | Что делает |
|---|---|---|
| `POST /api/v1/groups` | `createGroup` | Создать группу (создатель становится организатором) |
| `GET /api/v1/groups/{groupId}` | `getGroup` | Данные группы |
| `POST /api/v1/groups/{groupId}/participants` | `joinGroup` | Вступить (в теле `inviteCode`, контакты, адрес) |
| `GET /api/v1/groups/{groupId}/participants` | `listParticipants` | Список участников (только имена) |
| `PUT /api/v1/groups/{groupId}/participants/me/address` | `updateMyAddress` | Изменить свой адрес |
| `DELETE /api/v1/groups/{groupId}/participants/me` | `leaveGroup` | Выйти из группы |
| `POST /api/v1/groups/{groupId}/close-registration` | `closeRegistration` | Закрыть регистрацию (только организатор) |
| `GET /internal/groups/{groupId}/participants/{participantId}/address` | `getParticipantAddress` | Адрес подопечного (вызывает только Order Service) |

### Draw Service

| Метод и путь | operationId | Что делает |
|---|---|---|
| `GET /api/v1/groups/{groupId}/draw` | `getDrawStatus` | Статус жеребьевки |
| `GET /api/v1/groups/{groupId}/draw/my-recipient` | `getMyRecipient` | Мой подопечный (только имя, без адреса) |

### Payment Service (только внутренний)

| Метод и путь | operationId | Что делает |
|---|---|---|
| `POST /internal/payments/holds` | `holdPayment` | Заморозить сумму (`Idempotency-Key` обязателен) |
| `POST /internal/payments/{paymentId}/capture` | `capturePayment` | Списать (можно меньше замороженной суммы, в теле `amount`) |
| `POST /internal/payments/{paymentId}/release` | `releasePayment` | Отменить холд |
| `POST /internal/payments/{paymentId}/refund` | `refundPayment` | Вернуть деньги |
| `GET /internal/payments/{paymentId}` | `getPayment` | Статус платежа |

### Order Service

| Метод и путь | operationId | Что делает |
|---|---|---|
| `GET /api/v1/groups/{groupId}/catalog/products` | `searchProducts` | Товары маркетплейса в пределах бюджета группы |
| `POST /api/v1/groups/{groupId}/gift-orders` | `createGiftOrder` | Заказать подарок (в теле `productId`, `paymentMethodToken`; `Idempotency-Key` обязателен) |
| `GET /api/v1/groups/{groupId}/gift-orders/my` | `getMyGiftOrder` | Мой заказ подарка в этой группе |
| `GET /api/v1/gift-orders/{giftOrderId}` | `getGiftOrder` | Заказ по id |
| `POST /webhooks/marketplace` | `handleMarketplaceWebhook` | Статусы доставки от маркетплейса |

### Notification Service

REST API нет, сервис только потребляет события.

### Внешние API (что мы вызываем)

- **Платежный провайдер**: `POST /holds`, `POST /holds/{id}/capture`, `POST /holds/{id}/release`, `POST /payments/{id}/refund`
- **Маркетплейс**: `GET /products`, `GET /products/{id}` (текущая цена и наличие), `POST /orders` (с `Idempotency-Key` и ожидаемой ценой `expectedPrice`), `POST /orders/{id}/cancel`; вебхуки `ORDER_SHIPPED`, `ORDER_DELIVERED`

### Формат ошибки

```json
{ "code": "OUT_OF_STOCK", "message": "Товар закончился", "requestId": "..." }
```

| Код | HTTP | Когда |
|---|---|---|
| `GROUP_NOT_FOUND` | 404 | Нет такой группы (или пользователь в ней не состоит) |
| `GIFT_ORDER_NOT_FOUND` | 404 | Нет такого заказа подарка |
| `PAYMENT_NOT_FOUND` | 404 | Нет такого платежа |
| `INVALID_STATE` | 409 | Операция недопустима в текущем статусе |
| `GIFT_ALREADY_SHIPPED` | 409 | Нельзя выйти: подарок от участника или для него уже в доставке |
| `TOO_FEW_PARTICIPANTS` | 409 | Нельзя выйти: останется меньше 2 участников |
| `INVALID_INVITE_CODE` | 403 | Неверный код приглашения |
| `ALREADY_JOINED` | 409 | Уже участник |
| `REGISTRATION_CLOSED` | 409 | Регистрация закрыта |
| `NOT_ENOUGH_PARTICIPANTS` | 409 | Меньше 3 участников при закрытии регистрации |
| `NOT_ORGANIZER` | 403 | Действие доступно только организатору |
| `DRAW_NOT_COMPLETED` | 409 | Жеребьевки еще не было |
| `PRICE_EXCEEDS_BUDGET` | 422 | Товар дороже бюджета |
| `GIFT_ORDER_ALREADY_EXISTS` | 409 | У Санты уже есть активный заказ в этой группе |
| `PAYMENT_DECLINED` | 402 | Провайдер отклонил оплату |
| `OUT_OF_STOCK` | 409 | Товар закончился |
| `IDEMPOTENCY_KEY_REUSED` | 422 | Тот же ключ, но другое тело запроса |
| `SERVICE_UNAVAILABLE` | 503 | Зависимость недоступна, повторите позже |

---

## 6. События (для AsyncAPI)

### Брокер

- Exchange на каждый сервис-источник, тип `topic`: `group.events`, `draw.events`, `order.events`.
- Routing key = тип события, например `group.participant-left`.
- Очередь потребителя: `<сервис-потребитель>.<exchange>`, например `notification-service.order.events`.
- Для каждой очереди есть очередь «мертвых писем»: `<имя очереди>.dlq`.

### Конверт события (общий для всех)

```json
{
  "eventId": "uuid",
  "eventType": "order.gift-ordered",
  "occurredAt": "2026-12-20T10:00:00Z",
  "producer": "order-service",
  "correlationId": "uuid",
  "data": { }
}
```

`eventId` используется для дедупликации у потребителя, `correlationId` связывает все события одного процесса.

### Список событий

| Тип события | Источник | Потребители | Поля `data` |
|---|---|---|---|
| `group.participant-joined` | Group | Notification | `groupId`, `participantId`, `userId`, `displayName`, `email`, `telegramChatId` |
| `group.participant-left` | Group | Draw, Order, Notification | `groupId`, `participantId`, `leftAt` |
| `group.registration-closed` | Group | Draw, Order | `groupId`, `participants[]` (`participantId`, `userId`, `displayName`), `budget` |
| `draw.completed` | Draw | Order, Notification, Group | `groupId`, `drawId`, `pairs[]` (`santaParticipantId`, `recipientParticipantId`, `recipientDisplayName`) |
| `draw.reassigned` | Draw | Order, Notification | `groupId`, `drawId`, `leftParticipantId`, `santaParticipantId`, `newRecipientParticipantId`, `newRecipientDisplayName` |
| `order.gift-ordered` | Order | Notification | `giftOrderId`, `groupId`, `santaParticipantId`, `recipientParticipantId`, `productName` |
| `order.gift-failed` | Order | Notification | `giftOrderId`, `groupId`, `santaParticipantId`, `failureReason` |
| `order.gift-cancelled` | Order | Notification | `giftOrderId`, `groupId`, `santaParticipantId`, `recipientParticipantId`, `failureReason` |
| `order.gift-shipped` | Order | Group, Notification | `giftOrderId`, `groupId`, `santaParticipantId`, `recipientParticipantId` |
| `order.gift-delivered` | Order | Notification | `giftOrderId`, `groupId`, `santaParticipantId`, `recipientParticipantId` |

Payment Service событий не публикует: Order Service вызывает его синхронно.

---

## 7. Бизнес-правила (на них ссылаемся в диаграммах и ADR)

1. В группе минимум 3 участника.
2. Цена подарка не больше `budget` группы.
3. У Санты в группе одновременно только один активный заказ (`PENDING`, `PAYMENT_HELD`, `ORDERED`, `PAID`, `SHIPPED`).
4. Санта никогда не видит адрес подопечного; адрес получает только Order Service и передает маркетплейсу.
5. Холд действует 30 минут; если сага не дошла до списания, холд отменяется (`TIMEOUT`).
6. При выходе участника X: Санта участника X начинает дарить подопечному участника X (`draw.reassigned`); активные заказы, где X — Санта или подопечный, отменяются с возвратом денег.
7. Повторы при недоступности внешних систем: 3 попытки с паузами 1, 5, 30 секунд, затем компенсация.
8. Выйти из группы нельзя, если подарок от участника или для него уже передан в доставку (`GIFT_ALREADY_SHIPPED`). Group Service узнает об этом из события `order.gift-shipped`.
9. Выйти из группы нельзя, если после выхода останется меньше 2 активных участников (`TOO_FEW_PARTICIPANTS`). Двое оставшихся дарят подарки друг другу.
10. Перед заказом на маркетплейсе Order Service проверяет текущую цену товара. Если цена снизилась или не изменилась, заказ оформляется по текущей цене, и списывается ровно она (остаток холда размораживается). Если цена выросла, сага завершается компенсацией с причиной `PRICE_CHANGED`, и Санта выбирает подарок заново.

---

## 8. Сценарии для sequence-диаграмм

| № | Название | Ответственный |
|---|---|---|
| 0 | Успешный заказ подарка | Алиса |
| 1 | Товар закончился (маркетплейс отказал после холда) | Алиса |
| 2 | Order Service упал посреди саги | Алиса |
| 3 | Участник вышел после жеребьевки | Маша |

## 9. ADR

| Файл | Название | Ответственный |
|---|---|---|
| `ADR-001-saga-orchestration.md` | Оркестрация саги заказа подарка в Order Service | Алиса |
| `ADR-002-message-broker.md` | RabbitMQ в качестве брокера сообщений | Анзор |
| `ADR-003-outbox.md` | Transactional Outbox для публикации событий | Анзор |
| `ADR-004-idempotency.md` | Идемпотентность платежей, заказов и обработчиков событий | Маша |
| `ADR-005-hold-capture.md` | Холд и списание вместо мгновенной оплаты | Маша |

Структура каждого ADR: **Статус** (Принято) → **Контекст** → **Рассмотренные варианты** → **Решение** → **Последствия** (плюсы и минусы).
