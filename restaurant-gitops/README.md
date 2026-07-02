# Restaurant GitOps

Simple GitOps overlay repo for the 4-service restaurant system.

## What It Deploys

- MySQL
- phpMyAdmin
- Kafka
- `gateway-service`
- `menu-service`
- `order-service`
- `payment-service`

## Overlays

- `overlays/dev`
- `overlays/demo`
- `overlays/istio-dev`

## Quick Start

```bash
kubectl apply -k restaurant-gitops/overlays/dev
```

phpMyAdmin access:

```bash
kubectl port-forward service/phpmyadmin -n restaurant-demo 8081:80
```

Open `http://localhost:8081` and sign in with:

- server: `mysql`
- username: `restaurant_app`
- password: `change-me`

Istio demo:

```bash
istioctl install --set profile=demo -y
kubectl apply -k restaurant-gitops/overlays/istio-dev
```
