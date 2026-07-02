# Restaurant Assignment Run Guide

This workspace contains three parts that work together:

- `restaurant-app/`
- `restaurant-gitops/`
- `frontend/`

The current system scope is:

- `gateway-service`
- `menu-service`
- `order-service`
- `payment-service`
- platform dependencies: MySQL, Kafka, phpMyAdmin

## What Each Part Does

- `restaurant-app/` contains the Spring Boot services and Dockerfiles
- `restaurant-gitops/` contains the Kubernetes manifests and overlays
- `frontend/` contains the student UI that calls `gateway-service`

## Recommended Run Order

1. Build the backend services from `restaurant-app/`
2. Build or push the Docker images you want Kubernetes to run
3. Deploy `restaurant-gitops/`
4. Port-forward `gateway-service`
5. Start the `frontend/` app

Why this order:

- the frontend depends on the gateway
- the gateway depends on the backend services
- order completion depends on Kafka being healthy

## Build Backend Services

From `restaurant-app/`:

```bash
mvn clean package
docker build -t docker.io/khcodenet1008/gateway-service:dev services/gateway-service
docker build -t docker.io/khcodenet1008/menu-service:dev services/menu-service
docker build -t docker.io/khcodenet1008/order-service:dev services/order-service
docker build -t docker.io/khcodenet1008/payment-service:dev services/payment-service
```

## Deploy To Kubernetes

Run these commands from the workspace root:

```bash
kubectl create namespace restaurant-demo --dry-run=client -o yaml | kubectl apply -f -
kubectl -n restaurant-demo create secret generic mysql-secret \
  --from-literal=root-password=root1234 \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -k restaurant-gitops/overlays/dev
```

If you are already inside `restaurant-app/`, use:

```bash
kubectl apply -k ../restaurant-gitops/overlays/dev
```

Check that the main pods are healthy:

```bash
kubectl get pods -n restaurant-demo
```

You should see these components running:

- `gateway-service`
- `menu-service`
- `order-service`
- `payment-service`
- `mysql`
- `kafka`
- `phpmyadmin`

## Quick Backend Test

```bash
kubectl port-forward deployment/gateway-service -n restaurant-demo 8080:8080
curl http://localhost:8080/api/menu/items
```

## Frontend UI

The simple student UI lives in `frontend/`.

Start it only after the gateway is reachable on `http://localhost:8080`:

```bash
cd frontend
npm install
npm run dev
```

If `npm run dev` says `vite: command not found`, it means dependencies were not installed yet. Run:

```bash
npm install
```

Optional environment file:

```bash
cp .env.example .env
```

Default frontend API base URL:

```bash
VITE_GATEWAY_API_BASE_URL=http://localhost:8080
```

## Order And Payment Status Flow

The frontend is not supposed to invent final order states.

- creating an order stores a real order in MySQL through `order-service`
- confirming payment stores a real payment authorization in MySQL through `payment-service`
- `payment-service` publishes a Kafka event
- `order-service` consumes that event and updates the real order status in MySQL

Expected status behavior:

- `PENDING` means the order was created but not completed yet
- `UPDATING` in the UI means the payment request was sent and the frontend is waiting for the backend status to change
- `COMPLETED` means `order-service` consumed `PaymentAuthorized` and updated the order row in MySQL
- `FAILED` or `CANCELLED` means the backend stored a failure outcome

Important note:

- if Kafka is down, payment may be stored as authorized while the order stays `PENDING`
- this is a backend event-chain issue, not just a UI issue

## Recent Orders In The Frontend

The frontend keeps a recent-orders list in browser `localStorage` so the table survives page reloads in the same browser.

- the list itself is UI-side persistence
- the order status still comes from the backend when the UI refreshes or refetches
- MySQL remains the source of truth for real order and payment data

## Inspect MySQL With phpMyAdmin

Port-forward phpMyAdmin:

```bash
kubectl port-forward service/phpmyadmin -n restaurant-demo 8081:80
```

Open:

```text
http://localhost:8081
```

Login:

- server: `mysql`
- username: `restaurant_app`
- password: `change-me`

Useful schemas:

- `restaurant_order`
- `restaurant_payment`
- `restaurant_menu`

## Kafka Note

Order completion depends on Kafka.

If confirmed orders stay `PENDING` for a long time, check Kafka first:

```bash
kubectl get pods -n restaurant-demo
kubectl logs statefulset/kafka -n restaurant-demo
```

When Kafka is healthy, `payment-service` can publish `PaymentAuthorized` or `PaymentFailed`, and `order-service` can update the real order status.

## Istio Demo

```bash
istioctl install --set profile=demo -y
kubectl apply -k restaurant-gitops/overlays/istio-dev
```
