```markdown
# Delivery App — Event Matrix

## 1. Назначение

Event Matrix описывает события, которые публикуются сервисами Delivery App, а также потребителей этих событий и их действия после получения события.

Документ нужен для фиксации асинхронных взаимодействий между сервисами в микросервисной архитектуре.

---

## 2. Общие правила обработки событий

- Все события публикуются через Message Broker.
- События отражают уже произошедший бизнес-факт и называются в прошедшем времени.
- Все события должны содержать `eventId`, `eventType`, `occurredAt`.
- Consumers должны обрабатывать события идемпотентно.
- Повторная доставка одного и того же события не должна приводить к дублированию действий.
- При ошибке обработки события должна выполняться повторная попытка.
- Если событие не удалось обработать после нескольких попыток, оно должно попадать в Dead Letter Queue.
- Order Service является владельцем жизненного цикла заказа.
- Payment Service является владельцем платежей.
- Courier Service является владельцем данных о курьерах и назначении курьера.
- Notification Service не изменяет статус заказа, а только отправляет уведомления.

---

## 3. Event Matrix

### order_created

#### Event

- order_created

#### Publisher

- Order Service

#### Topic

- orders.events

#### Trigger

- Заказ успешно создан

#### Consumers

- Restaurant Service
- Notification Service

#### Consumer action

- Restaurant Service отображает заказ ресторану для подтверждения
- Notification Service уведомляет клиента о создании заказа

#### Payload

- eventId
- eventType
- orderId
- clientId
- restaurantId
- totalAmount
- status
- occurredAt

#### Notes

- Payment Service не является consumer события `order_created`, так как платеж создается через REST-вызов `Order Service → Payment Service`.
- Событие публикуется после успешного создания заказа в Order Service.
- Событие не должно содержать полный объект заказа.

---

### order_confirmed

#### Event

- order_confirmed

#### Publisher

- Order Service

#### Topic

- orders.events

#### Trigger

- Ресторан подтвердил заказ

#### Consumers

- Courier Service
- Notification Service

#### Consumer action

- Courier Service запускает поиск доступного курьера
- Notification Service уведомляет клиента о подтверждении заказа рестораном

#### Payload

- eventId
- eventType
- orderId
- restaurantId
- status
- estimatedCookingTimeMinutes
- occurredAt

#### Notes

- Order Service остается владельцем статуса заказа.
- Courier Service не меняет статус заказа напрямую, а только запускает процесс поиска курьера.
- Notification Service только уведомляет клиента.

---

### payment_confirmed

#### Event

- payment_confirmed

#### Publisher

- Payment Service

#### Topic

- payments.events

#### Trigger

- Платеж успешно подтвержден

#### Consumers

- Order Service
- Notification Service

#### Consumer action

- Order Service фиксирует факт успешной оплаты заказа
- Notification Service уведомляет клиента об успешной оплате

#### Payload

- eventId
- eventType
- orderId
- paymentId
- amount
- paymentStatus
- occurredAt

#### Notes

- Payment Service является владельцем платежа.
- Order Service не пересчитывает платеж, а только фиксирует результат оплаты.
- Повторная обработка события `payment_confirmed` не должна приводить к повторному изменению платежного статуса.

---

### courier_assigned

#### Event

- courier_assigned

#### Publisher

- Courier Service

#### Topic

- couriers.events

#### Trigger

- Курьер принял заказ

#### Consumers

- Order Service
- Notification Service

#### Consumer action

- Order Service сохраняет courierId и обновляет статус заказа на `assigned_to_courier`
- Notification Service уведомляет клиента о назначении курьера

#### Payload

- eventId
- eventType
- orderId
- courierId
- status
- estimatedPickupTime
- occurredAt

#### Notes

- Courier Service отвечает за выбор и назначение курьера.
- Order Service отвечает за изменение статуса заказа после получения события.
- Повторная обработка события `courier_assigned` не должна приводить к повторному назначению курьера.

---

### order_canceled

#### Event

- order_canceled

#### Publisher

- Order Service

#### Topic

- orders.events

#### Trigger

- Заказ отменен клиентом, рестораном, системой или поддержкой

#### Consumers

- Payment Service
- Courier Service
- Restaurant Service
- Notification Service

#### Consumer action

- Payment Service запускает возврат денежных средств, если платеж уже был подтвержден
- Courier Service освобождает курьера, если он был назначен на заказ
- Restaurant Service скрывает заказ или помечает его как отмененный
- Notification Service уведомляет клиента об отмене заказа

#### Payload

- eventId
- eventType
- orderId
- status
- cancelReason
- canceledBy
- occurredAt

#### Notes

- Order Service является владельцем факта отмены заказа.
- Возможные значения `canceledBy`: `client`, `restaurant`, `system`, `support`.
- Возможные причины отмены: `customer_cancelled`, `restaurant_timeout`, `courier_not_found`, `payment_failed`, `support_decision`.
- Payment Service должен проверять, был ли платеж подтвержден, прежде чем запускать возврат.
- Courier Service должен проверять, был ли курьер назначен, прежде чем освобождать его.
- Повторная обработка события `order_canceled` не должна приводить к повторному возврату денежных средств.

---

### order_delivered

#### Event

- order_delivered

#### Publisher

- Order Service

#### Topic

- orders.events

#### Trigger

- Заказ доставлен клиенту

#### Consumers

- Payment Service
- Notification Service

#### Consumer action

- Payment Service закрывает платеж/расчет по заказу
- Notification Service уведомляет клиента о доставке заказа

#### Payload

- eventId
- eventType
- orderId
- status
- deliveredAt
- occurredAt

#### Notes

- Order Service обновляет статус заказа на `delivered`.
- Payment Service не меняет статус заказа, а только закрывает финансовую часть.
- Notification Service уведомляет клиента о завершении доставки.

---

## 4. Сводка событий

### orders.events

- order_created
- order_confirmed
- order_canceled
- order_delivered

### payments.events

- payment_confirmed

### couriers.events

- courier_assigned

---

## 5. Связь с архитектурой

События передаются через Message Broker.

Основные event-потоки:

- Order Service → Message Broker → Restaurant Service
- Order Service → Message Broker → Notification Service
- Order Service → Message Broker → Courier Service
- Payment Service → Message Broker → Order Service
- Payment Service → Message Broker → Notification Service
- Courier Service → Message Broker → Order Service
- Courier Service → Message Broker → Notification Service

REST используется, когда вызывающей стороне нужен ответ сразу.

Events используются, когда нужно уведомить другие сервисы о произошедшем бизнес-факте.

---

## 6. Связь с другими артефактами

Документ связан со следующими артефактами:

- business_context.md
- user_stories_mvp.md
- bpmn_order_flow.drawio
- sequence_order.puml
- c4_context.puml
- c4_container.puml
- openapi.yaml
- data_model.md
- er_diagram.drawio
- database.sql
```
