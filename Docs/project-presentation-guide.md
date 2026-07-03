# Restaurant Project Presentation Guide

Use this guide when presenting the project from the outside in:

1. user network
2. gateway network
3. service-to-service network
4. database network
5. Kafka event network
6. CI/CD and GitOps network

This project matches the course ideas from:

- Chapter 6 Docker and Kubernetes, merged PDF pages 264-383
- Chapter 7 DevOps and GitOps, merged PDF pages 384-455
- Chapter 9 API Gateway, merged PDF pages 469-538
- Chapter 10 service design details, merged PDF pages 594-664
- Chapter 14 Saga and transaction, merged PDF pages 789-839
- Chapter 15 microservice implementation, merged PDF pages 840-905

## 1. One-Sentence Project Summary

This is a restaurant microservices demo where the frontend talks to one API gateway, the gateway routes to backend services, each service owns its own MySQL schema, and order completion happens through Kafka events between `order-service` and `payment-service`.

## 2. What Runs In This Project

Application repos:

- `frontend/`
- `restaurant-app/`
- `restaurant-gitops/`

Runtime components in Kubernetes:

- `gateway-service`
- `menu-service`
- `order-service`
- `payment-service`
- `mysql`
- `kafka`
- `phpmyadmin`

Optional:

- Istio overlay in `restaurant-gitops/overlays/istio-dev`

## 3. Start With The Big Picture

Say this first:

> The system has one frontend, one gateway, four backend services, one MySQL server split into three service-owned databases, and Kafka for asynchronous status updates.

Use this simple diagram:

```text
Browser UI
   |
   v
gateway-service
   |        |         |
   |        |         +--> payment-service --> restaurant_payment DB
   |        |
   |        +------------> order-service ----> restaurant_order DB
   |
   +---------------------> menu-service -----> restaurant_menu DB

order-service -- order.events --> Kafka
payment-service -- payment.events --> Kafka
order-service <-- payment.events -- Kafka
```

## 4. Network By Network

## Network 1: User To Frontend

What happens:

- user opens the Vite React app in `frontend/`
- the browser UI sends HTTP requests to `http://localhost:8080`
- that URL is usually a port-forward to Kubernetes `gateway-service`

Presentation line:

> The browser never talks directly to `menu-service`, `order-service`, or `payment-service`. It only talks to the gateway.

Main frontend API file:

- `frontend/src/api/client.ts`

Important frontend endpoints:

- `GET /api/menu/items`
- `POST /api/orders`
- `GET /api/orders/{id}`
- `PATCH /api/orders/{id}/cancel`
- `POST /api/payments/mock-confirm`

## Network 2: Frontend To Gateway

Course mapping:

- Chapter 9 API Gateway says the gateway should be the single entry point and should not contain business logic.

This project does exactly that:

- `gateway-service` receives public requests
- it routes by path
- it forwards a correlation ID header
- it does not decide order or payment business rules

Gateway routes are in:

- `restaurant-app/services/gateway-service/src/main/resources/application.yml`

Current routes:

- `/api/menu/**` -> `menu-service`
- `/api/orders/**` -> `order-service`
- `/api/payments/**` -> `payment-service`

Correlation ID behavior:

- implemented in `restaurant-app/services/gateway-service/src/main/java/com/example/restaurant/gateway/filter/CorrelationIdFilter.java`
- if request has no `X-Correlation-Id`, the gateway creates one
- the same value is returned in the response header

Presentation line:

> The gateway is like the front desk of the restaurant. Every customer request enters there first, then the gateway sends it to the right internal service.

## Network 3: Gateway To Backend Services

Inside Kubernetes, services call each other by Kubernetes service name:

- `http://menu-service:8080`
- `http://order-service:8080`
- `http://payment-service:8080`

These internal URLs are configured in:

- `restaurant-gitops/base/services/gateway-service/configmap.yaml`

Why this matters:

- the frontend does not need to know internal pod IPs
- Kubernetes `Service` objects provide stable network names
- this follows Chapter 6 Kubernetes service concepts and Chapter 8 service communication

Presentation line:

> Kubernetes gives each backend a stable service name, so the gateway can route to service names instead of hardcoding pod IP addresses.

## Network 4: Service To Database

Each service has its own database ownership boundary.

Actual databases created by MySQL init:

- `restaurant_menu`
- `restaurant_order`
- `restaurant_payment`

Defined in:

- `restaurant-gitops/base/platform/mysql/init-configmap.yaml`

This follows the course idea of service separation and data ownership:

- menu data belongs to `menu-service`
- order data belongs to `order-service`
- payment data belongs to `payment-service`

Important rule for presenting:

> Services do not directly write into each other’s tables. They communicate through APIs or events.

### menu-service

Responsibility:

- read menu items
- expose read-only menu endpoints

Code:

- controller: `restaurant-app/services/menu-service/src/main/java/com/example/restaurant/menu/web/MenuController.java`
- schema: `restaurant-app/services/menu-service/src/main/resources/db/migration/V1__init_schema.sql`

Key tables:

- `menu_category`
- `menu_item`

### order-service

Responsibility:

- create orders
- read orders
- cancel orders
- track Saga progress
- consume payment events and update final order status

Code:

- controller: `restaurant-app/services/order-service/src/main/java/com/example/restaurant/order/web/OrderController.java`
- service: `restaurant-app/services/order-service/src/main/java/com/example/restaurant/order/service/OrderService.java`
- listener: `restaurant-app/services/order-service/src/main/java/com/example/restaurant/order/service/OrderSagaEventListener.java`
- schema: `restaurant-app/services/order-service/src/main/resources/db/migration/V1__init_schema.sql`

Key tables:

- `customer_order`
- `order_item`
- `saga_state`

### payment-service

Responsibility:

- simulate payment approval or failure
- store payment authorization result
- publish payment result event to Kafka

Code:

- controller: `restaurant-app/services/payment-service/src/main/java/com/example/restaurant/payment/web/PaymentController.java`
- service: `restaurant-app/services/payment-service/src/main/java/com/example/restaurant/payment/service/PaymentService.java`
- schema: `restaurant-app/services/payment-service/src/main/resources/db/migration/V1__init_schema.sql`

Key tables:

- `payment_authorization`
- `payment_refund`

## Network 5: Kafka Event Network

This is the most important part for explaining why order status changes later instead of immediately.

Course mapping:

- Chapter 14 Saga and transaction
- Chapter 15 implementation with Kafka event messaging

This project uses choreography-style event flow, not an orchestrator service.

Important honesty point for presentation:

- `order-service` does publish `OrderCreated`
- but the current classroom implementation does not automatically make `payment-service` consume `order.events` and process payment by itself
- payment confirmation is still manually triggered by the frontend through `POST /api/payments/mock-confirm`
- after that manual trigger, the payment result returns to `order-service` through Kafka

So the current project is:

> a partial Saga demo with real asynchronous order-status completion, but manual payment initiation.

Current topics:

- `order.events`
- `payment.events`

Configured in:

- `restaurant-gitops/base/services/order-service/configmap.yaml`
- `restaurant-gitops/base/services/payment-service/configmap.yaml`

### Order creation event flow

1. frontend calls `POST /api/orders`
2. gateway forwards to `order-service`
3. `order-service` writes the order into `customer_order`
4. `order-service` writes `saga_state` as `ORDER_CREATED`
5. `order-service` publishes `OrderCreated` to `order.events`
6. the frontend then asks the user to confirm payment manually

### Payment confirmation event flow

1. frontend calls `POST /api/payments/mock-confirm`
2. gateway forwards to `payment-service`
3. `payment-service` stores `AUTHORIZED` or `FAILED`
4. `payment-service` publishes `PaymentAuthorized` or `PaymentFailed` to `payment.events`
5. `order-service` consumes that payment event
6. `order-service` updates `customer_order.status`
7. `order-service` updates `saga_state`
8. if success, `order-service` also publishes `OrderCompleted`

Presentation line:

> Payment confirmation does not directly change the order row by calling the order database. It changes the order through Kafka event consumption, which is the Saga-style asynchronous step.

### Why Orders Can Stay PENDING

If Kafka is down:

- payment may already be stored as `AUTHORIZED`
- but `order-service` will not receive `PaymentAuthorized`
- so `customer_order.status` stays `PENDING`

That means:

> A stuck `PENDING` order is usually an event-chain problem, not a frontend problem.

## Network 6: Kubernetes Deployment Network

The manifests are in:

- `restaurant-gitops/base/`

How Kubernetes objects are used:

- `Deployment` for stateless app services
- `Service` for stable internal networking
- `StatefulSet` for `mysql` and `kafka`
- `ConfigMap` for environment values
- `Secret` for MySQL root password
- `Kustomization` to group everything

Main file:

- `restaurant-gitops/base/kustomization.yaml`

Presentation line:

> In Kubernetes, Deployments run the service pods, Services give them stable names, and StatefulSets are used for stateful infrastructure like MySQL and Kafka.

## Network 7: CI/CD And GitOps Network

Course mapping:

- Chapter 7 DevOps and GitOps

App CI workflow:

- `restaurant-app/.github/workflows/ci.yml`

Pipeline behavior:

1. run Maven tests for all four services
2. on `main`, build Docker images
3. push images to Docker Hub
4. update image tags in `restaurant-gitops`
5. push GitOps changes

That means this project separates:

- application source repo: `restaurant-app`
- deployment manifests repo: `restaurant-gitops`

Presentation line:

> This follows GitOps style because deployment state is stored in a separate manifests repository, and the app pipeline updates that desired state automatically.

## 5. Service By Service Script

Use this short script during the presentation.

### gateway-service

> `gateway-service` is the single external entry point. It routes `/api/menu` to `menu-service`, `/api/orders` to `order-service`, and `/api/payments` to `payment-service`. It also generates or forwards `X-Correlation-Id` for tracing.

### menu-service

> `menu-service` is the simplest service. It reads menu data from the `restaurant_menu` schema and exposes read-only endpoints for menu items.

### order-service

> `order-service` is the core business service. It creates orders, stores order items, tracks the Saga state, and waits for Kafka payment events before marking the order completed or cancelled.

### payment-service

> `payment-service` is a mock payment service for classroom demo purposes. It stores a payment authorization result, then emits Kafka events so `order-service` can update the real final order status.

### mysql

> MySQL is shared as one server in infrastructure, but logically split into separate databases so each service owns its own data.

### kafka

> Kafka is the event backbone. It decouples payment completion from order status update and demonstrates Saga-style asynchronous processing.

### phpmyadmin

> phpMyAdmin is not part of the business flow. It is only there to inspect the real MySQL data during demo and debugging.

## 6. Best Demo Order

If you need to present a live demo, use this order:

1. open frontend
2. load menu
3. create one order
4. show order is `PENDING`
5. confirm payment
6. explain that frontend may briefly show `UPDATING`
7. fetch order again or wait for refresh
8. show status becomes `COMPLETED`
9. open phpMyAdmin and show `customer_order` and `payment_authorization`

## 7. Very Short 2-Minute Explanation

If time is short, say this:

> This project is a small restaurant microservices system. The browser calls only one gateway service. The gateway routes requests to menu, order, and payment services. Each backend owns its own MySQL database schema. Creating an order is synchronous, but completing the order is asynchronous: payment-service publishes a Kafka event, then order-service consumes it and updates the final order status. Deployment is done with Kubernetes manifests stored in a separate GitOps repository, and the CI pipeline builds images and updates those manifests automatically.

## 8. Likely Questions And Good Answers

### Why use a gateway?

Because the course architecture uses a single entry point for routing, cross-cutting concerns, and cleaner frontend access.

### Why not let payment-service update order table directly?

Because that would break service ownership. Order data belongs to `order-service`.

### Why use Kafka here?

To model asynchronous Saga-style communication between services and avoid tight coupling.

### Is the whole Saga fully automated?

Not yet. Order creation publishes an event, but payment is still manually confirmed from the frontend. The asynchronous part that is fully real today is the payment-result event coming back through Kafka and updating the order status.

### Is this choreography or orchestration?

This project is choreography because services react to events directly. There is no separate orchestrator service.

### Why is there one MySQL server if this is microservices?

For classroom simplicity. Logically the data is still separated by schema and service ownership.

### What happens if Kafka is down?

Payment can be stored, but order status may remain `PENDING` because the event was not consumed.

## 9. Files To Show On Screen

If you want to prove your explanation during the presentation, open these files:

- `README.md`
- `restaurant-app/services/gateway-service/src/main/resources/application.yml`
- `restaurant-app/services/order-service/src/main/java/com/example/restaurant/order/service/OrderService.java`
- `restaurant-app/services/payment-service/src/main/java/com/example/restaurant/payment/service/PaymentService.java`
- `restaurant-app/services/order-service/src/main/java/com/example/restaurant/order/service/OrderSagaEventListener.java`
- `restaurant-gitops/base/kustomization.yaml`
- `restaurant-gitops/base/services/order-service/configmap.yaml`
- `restaurant-gitops/base/platform/mysql/init-configmap.yaml`
- `restaurant-gitops/base/platform/kafka/kafka-statefulset.yaml`

## 10. Final Mental Model

Remember this sentence:

> HTTP handles the request path, MySQL stores the service-owned data, Kafka carries the cross-service status changes, and Kubernetes gives every component a stable place to run.
