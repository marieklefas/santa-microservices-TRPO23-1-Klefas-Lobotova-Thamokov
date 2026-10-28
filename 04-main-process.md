# 4. Главный процесс: заказ подарка

Ответственные: Алиса (успешный сценарий, сбои 1 и 2, диаграмма состояний), Маша (сбой 3).

Главный процесс системы — сага заказа подарка. Ее координирует Order Service (оркестрация, см. [ADR-001](05-adr/ADR-001-saga-orchestration.md)). Шаги саги:

| Шаг (`sagaStep`) | Что делает | Компенсация при сбое следующих шагов |
|---|---|---|
| `HOLD_PAYMENT` | Payment Service замораживает цену подарка на карте Санты | `RELEASE_PAYMENT` — отменить холд |
| `GET_RECIPIENT_ADDRESS` | Group Service отдает адрес подопечного | не нужна (только чтение) |
| `PLACE_MARKETPLACE_ORDER` | Order Service проверяет текущую цену товара, затем маркетплейс принимает заказ с адресом подопечного | `CANCEL_MARKETPLACE_ORDER` — отменить заказ |
| `CAPTURE_PAYMENT` | Payment Service списывает замороженную сумму | `REFUND_PAYMENT` — вернуть деньги |

Перед каждым вызовом Order Service сохраняет текущий шаг в `saga_state`, а каждый вызов идет с ключом идемпотентности (`hold-{giftOrderId}`, `order-{giftOrderId}` и т. д.). Благодаря этому после падения сага продолжается с того же шага без задвоений.

## Состояния заказа подарка

```mermaid
stateDiagram-v2
    [*] --> PENDING: createGiftOrder
    PENDING --> PAYMENT_HELD: холд успешен
    PENDING --> FAILED: оплата отклонена
    PAYMENT_HELD --> ORDERED: маркетплейс принял заказ
    PAYMENT_HELD --> FAILED: товар закончился, цена выросла или маркетплейс недоступен (холд отменен)
    ORDERED --> PAID: списание успешно
    ORDERED --> FAILED: списание не удалось (заказ отменен, холд отменен)
    PAID --> SHIPPED: вебхук ORDER_SHIPPED
    SHIPPED --> DELIVERED: вебхук ORDER_DELIVERED
    PENDING --> CANCELLED: участник вышел
    PAYMENT_HELD --> CANCELLED: участник вышел
    ORDERED --> CANCELLED: участник вышел
    PAID --> CANCELLED: участник вышел (возврат денег)
    FAILED --> [*]
    CANCELLED --> [*]
    DELIVERED --> [*]
```

После `FAILED` Санта может создать новый заказ: правило «один активный заказ» не считает `FAILED` и `CANCELLED` активными.

## Сценарий 0. Успешный заказ подарка

```mermaid
sequenceDiagram
    autonumber
    actor U as Санта
    participant GW as API Gateway
    participant OS as Order Service
    participant PS as Payment Service
    participant PP as Платежный провайдер
    participant GS as Group Service
    participant MP as Маркетплейс
    participant MQ as RabbitMQ
    participant NS as Notification Service

    U->>GW: POST /api/v1/groups/{groupId}/gift-orders (Idempotency-Key)
    GW->>OS: createGiftOrder + X-User-Id
    OS->>OS: Проверки: есть пара, цена не выше бюджета, нет активного заказа
    OS->>OS: Сохранить GiftOrder: PENDING, sagaStep = HOLD_PAYMENT
    OS-->>GW: 202 Accepted, giftOrderId
    GW-->>U: 202 Accepted

    Note over OS: Дальше сага выполняется в фоне

    OS->>PS: POST /internal/payments/holds (Idempotency-Key: hold-{giftOrderId})
    PS->>PP: POST /holds
    PP-->>PS: 200, holdId
    PS-->>OS: 201, paymentId, HELD
    OS->>OS: PAYMENT_HELD, sagaStep = GET_RECIPIENT_ADDRESS

    OS->>GS: GET /internal/groups/{groupId}/participants/{recipientId}/address
    GS-->>OS: 200, адрес подопечного
    OS->>OS: sagaStep = PLACE_MARKETPLACE_ORDER

    OS->>MP: GET /products/{productId}
    MP-->>OS: 200, текущая цена, в наличии
    OS->>OS: Текущая цена не выше замороженной суммы
    OS->>MP: POST /orders (Idempotency-Key: order-{giftOrderId}, expectedPrice)
    MP-->>OS: 201, marketplaceOrderId
    OS->>OS: ORDERED, sagaStep = CAPTURE_PAYMENT

    OS->>PS: POST /internal/payments/{paymentId}/capture (amount = цена заказа)
    PS->>PP: POST /holds/{holdId}/capture
    PP-->>PS: 200
    PS-->>OS: 200, CAPTURED

    OS->>OS: Одна транзакция: PAID, sagaStep = COMPLETED, запись order.gift-ordered в outbox
    OS-)MQ: order.gift-ordered (отправка из outbox)
    MQ-)NS: order.gift-ordered
    NS->>NS: eventId нет в processed_events, обрабатываем
    NS-)U: «Подарок для подопечного заказан» (GIFT_ORDERED)

    Note over MP,OS: Через несколько дней
    MP->>OS: POST /webhooks/marketplace (ORDER_SHIPPED)
    OS->>OS: SHIPPED + outbox order.gift-shipped
    OS-->>MP: 200
    OS-)MQ: order.gift-shipped
    MQ-)NS: order.gift-shipped
    NS-)NS: Подопечному: «К вам едет подарок» (GIFT_ON_THE_WAY)
    MP->>OS: POST /webhooks/marketplace (ORDER_DELIVERED)
    OS->>OS: DELIVERED + outbox order.gift-delivered
    OS-->>MP: 200
    OS-)MQ: order.gift-delivered
    MQ-)NS: order.gift-delivered
    NS-)NS: Подопечному и Санте: GIFT_DELIVERED
```

## Сценарий сбоя 1. Товар закончился после холда

**Что ломается.** Пока Санта выбирал подарок и проходил холд, последний экземпляр товара купили. Маркетплейс отказывает в заказе, а деньги Санты уже заморожены.

**Как система компенсирует.** Order Service выполняет компенсацию шага `HOLD_PAYMENT`: отменяет холд, переводит заказ в `FAILED` с причиной `OUT_OF_STOCK` и просит Санту выбрать другой подарок. Если маркетплейс не ответил вообще, сначала идут повторы, а потом та же компенсация с причиной `MARKETPLACE_UNAVAILABLE`.

Та же компенсация срабатывает, если товар за это время подорожал и стал дороже замороженной суммы (`PRICE_CHANGED`, бизнес-правило 10): доплату без согласия Санты мы не делаем. Если товар подешевел, сбоя нет: заказ оформляется по новой цене, списывается меньше, чем заморожено, а остаток холда размораживается.

```mermaid
sequenceDiagram
    autonumber
    participant OS as Order Service
    participant PS as Payment Service
    participant PP as Платежный провайдер
    participant GS as Group Service
    participant MP as Маркетплейс
    participant MQ as RabbitMQ
    participant NS as Notification Service
    actor U as Санта

    Note over OS,PP: Холд уже выполнен: заказ PAYMENT_HELD
    OS->>GS: GET адрес подопечного
    GS-->>OS: 200, адрес
    OS->>OS: sagaStep = PLACE_MARKETPLACE_ORDER

    alt Цена выросла
        OS->>MP: GET /products/{productId}
        MP-->>OS: 200, цена выше замороженной суммы
        OS->>OS: failureReason = PRICE_CHANGED
    else Товар закончился
        OS->>MP: POST /orders (Idempotency-Key: order-{giftOrderId})
        MP-->>OS: 409 OUT_OF_STOCK
        OS->>OS: failureReason = OUT_OF_STOCK
    else Маркетплейс не отвечает
        loop 3 попытки: паузы 1, 5, 30 секунд
            OS-xMP: POST /orders (тот же Idempotency-Key)
        end
        OS->>OS: failureReason = MARKETPLACE_UNAVAILABLE
    end

    Note over OS: Компенсация шага HOLD_PAYMENT
    OS->>OS: sagaStep = RELEASE_PAYMENT
    OS->>PS: POST /internal/payments/{paymentId}/release (Idempotency-Key: release-{giftOrderId})
    PS->>PP: POST /holds/{holdId}/release
    PP-->>PS: 200
    PS-->>OS: 200, RELEASED
    OS->>OS: Одна транзакция: FAILED + outbox order.gift-failed
    OS-)MQ: order.gift-failed
    MQ-)NS: order.gift-failed
    NS-)U: «Заказ не удался, деньги разморожены, выберите подарок заново» (CHOOSE_ANOTHER_GIFT)
```

**Что если не удалась и сама компенсация** (Payment Service или провайдер недоступны при отмене холда)? Шаг `RELEASE_PAYMENT` сохранен в `saga_state`, фоновый обработчик повторяет его, пока не получится. Отмена холда идемпотентна, поэтому повтор безопасен. В худшем случае холд снимет сам банк по истечении срока, но деньги в любом случае не списываются.

## Сценарий сбоя 2. Order Service упал посреди саги

**Что ломается.** Order Service отправил заказ на маркетплейс, маркетплейс заказ принял, но Order Service упал до того, как сохранил ответ. После перезапуска он не знает, создан ли заказ. Если просто отправить заказ еще раз, подопечный получит два подарка, а Санта заплатит за один.

**Как система компенсирует.** Состояние саги хранится в базе, а не в памяти. После перезапуска фоновый обработчик находит незавершенные саги и повторяет текущий шаг с тем же ключом идемпотентности. Маркетплейс по ключу узнает повтор и возвращает уже созданный заказ, а не создает новый. Сага продолжается со списания денег.

```mermaid
sequenceDiagram
    autonumber
    participant OS as Order Service
    participant DB as order_db
    participant MP as Маркетплейс
    participant PS as Payment Service
    participant MQ as RabbitMQ

    Note over OS,DB: Заказ PAYMENT_HELD, sagaStep = PLACE_MARKETPLACE_ORDER
    OS->>MP: POST /orders (Idempotency-Key: order-{giftOrderId})
    MP->>MP: Заказ создан, ключ сохранен
    MP--xOS: 201, marketplaceOrderId (ответ потерян)
    Note over OS: Order Service упал

    Note over OS: Перезапуск через несколько секунд
    OS->>DB: Найти саги, где sagaStep не COMPLETED
    DB-->>OS: giftOrderId, sagaStep = PLACE_MARKETPLACE_ORDER
    OS->>MP: POST /orders (тот же Idempotency-Key: order-{giftOrderId})
    MP->>MP: Ключ уже был, новый заказ не создается
    MP-->>OS: 200, тот же marketplaceOrderId
    OS->>DB: ORDERED, sagaStep = CAPTURE_PAYMENT
    OS->>PS: POST /internal/payments/{paymentId}/capture (Idempotency-Key: capture-{giftOrderId})
    PS-->>OS: 200, CAPTURED
    OS->>DB: Одна транзакция: PAID, COMPLETED + outbox order.gift-ordered
    OS-)MQ: order.gift-ordered
```

Что еще важно в этом сценарии:

- **Повтор запроса от клиента.** Если Санта, не дождавшись ответа, нажал «Заказать» второй раз, запрос придет с тем же `Idempotency-Key`, и Order Service вернет уже созданный заказ.
- **Вебхуки, пришедшие во время падения**, не теряются: маркетплейс повторяет их, пока не получит `200`.
- **Событие не потеряется**, даже если сервис упал сразу после записи `PAID`: событие лежит в `outbox` и будет отправлено после перезапуска. Если оно уйдет дважды, Notification Service отбросит повтор по `eventId`.
- **Холд не зависнет навсегда**: если сага стоит дольше 30 минут, она завершается компенсацией с причиной `TIMEOUT`.

## Сценарий сбоя 3. Участник вышел после жеребьевки

**Что ломается.** Жеребьевка проведена, кольцо пар замкнуто: … → Санта S → X → R → … Участник X выходит из игры. Теперь у S нет подопечного, а у R нет Санты. Кроме того, у X и S уже могут быть оформленные заказы: X мог заказать подарок для R, а S — для X.

**Как система компенсирует.**

1. Draw Service вырезает X из кольца: пары S → X и X → R отменяются, создается пара S → R. Кольцо снова замкнуто, и задвоений нет.
2. Order Service отменяет незавершенные заказы, связанные с X: заказ S для X (подарок больше некому дарить) и заказ X для R (X ушел). Деньги возвращаются: холд отменяется или списанные деньги возвращаются.
3. S получает уведомление о новом подопечном и выбирает подарок для R.

```mermaid
sequenceDiagram
    autonumber
    actor X as Участник X
    participant GW as API Gateway
    participant GS as Group Service
    participant MQ as RabbitMQ
    participant DS as Draw Service
    participant OS as Order Service
    participant PS as Payment Service
    participant MP as Маркетплейс
    participant NS as Notification Service

    X->>GW: DELETE /api/v1/groups/{groupId}/participants/me
    GW->>GS: leaveGroup
    GS->>GS: Проверки по своей базе: подарков с участием X нет в shipped_gifts, после выхода останется хотя бы 2 участника
    alt Проверка не пройдена
        GS-->>X: 409 GIFT_ALREADY_SHIPPED или TOO_FEW_PARTICIPANTS
    else Проверка пройдена
        GS->>GS: Одна транзакция: X = LEFT + outbox group.participant-left
        GS-->>X: 204
        GS-)MQ: group.participant-left
    end

    par Draw Service
        MQ-)DS: group.participant-left
        DS->>DS: Найти пары S → X и X → R
        DS->>DS: Одна транзакция: обе пары CANCELLED, новая пара S → R + outbox draw.reassigned
        DS-)MQ: draw.reassigned
        MQ-)OS: draw.reassigned
        OS->>OS: Обновить копию пар: S дарит R
        MQ-)NS: draw.reassigned
        NS-)NS: Санте S: «Ваш новый подопечный — R» (RECIPIENT_CHANGED)
    and Order Service
        MQ-)OS: group.participant-left
        OS->>OS: Найти незавершенные заказы, где X — Санта или подопечный
        loop Для каждого такого заказа
            alt Заказ PAYMENT_HELD
                OS->>PS: release (Idempotency-Key: release-{giftOrderId})
            else Заказ ORDERED или PAID
                OS->>MP: POST /orders/{marketplaceOrderId}/cancel
                OS->>PS: release или refund (Idempotency-Key: refund-{giftOrderId})
            end
            OS->>OS: CANCELLED, failureReason = PARTICIPANT_LEFT + outbox order.gift-cancelled
            OS-)MQ: order.gift-cancelled
        end
        MQ-)NS: order.gift-cancelled
        NS-)NS: Санте отмененного заказа: GIFT_CANCELLED
    end
```

**Пояснения к сценарию.**

- События `draw.reassigned` и `group.participant-left` могут прийти в Order Service в любом порядке. Это не страшно: одно из них меняет копию пар, другое отменяет заказы, и они не мешают друг другу.
- **Подарок уже в доставке.** Отменить его нельзя, поэтому выйти из группы в этом случае тоже нельзя (бизнес-правило 8). Group Service проверяет это по своей копии `shipped_gifts`, которую заполняет из события `order.gift-shipped`, без синхронного вызова Order Service. Значит, выход работает, даже если Order Service недоступен.
- **Редкая гонка.** Если X вышел в те несколько секунд, когда подарок уже отправлен, а событие `order.gift-shipped` еще не дошло до Group Service, Order Service увидит заказ в статусе `SHIPPED` и не станет его отменять. Деньги при этом не теряются и не задваиваются: каждый Санта платит только за свой подарок. В худшем случае кто-то получит лишний подарок. Мы сознательно принимаем этот риск, чтобы не делать выход из группы зависимым от Order Service.
- **Двое оставшихся.** Если в группе осталось 2 участника, выйти нельзя (бизнес-правило 9): иначе последнему некому дарить. Перестройка кольца из трех участников дает пару «дарим друг другу», и это нормально.
- Если X выходит до жеребьевки, Draw Service и Order Service ничего не делают: пар еще нет. X просто не попадет в список участников события `group.registration-closed`.
