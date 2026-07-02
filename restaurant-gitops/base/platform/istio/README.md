# Istio Base

Simple Istio mesh resources for the classroom demo.

This mesh setup keeps only 4 application services inside the focused Istio path:

- `gateway-service`
- `menu-service`
- `order-service`
- `payment-service`

Resources included here:

- `PeerAuthentication`: enables mesh mTLS inside `restaurant-demo`
- `Gateway`: exposes the mesh through the Istio ingress gateway
- `VirtualService`: forwards `/api/menu`, `/api/orders`, and `/api/payments` to `gateway-service`
- `DestinationRule`: applies simple round-robin plus `ISTIO_MUTUAL` traffic policy for the 4 services

The pod sidecar injection is enabled per deployment annotation in `restaurant-app/platform/services/*/deployment.yaml`, which keeps MySQL and Kafka out of the mesh path.

## Install Istio First

These mesh resources assume Istio is already installed in the cluster.

Simple classroom path:

```bash
istioctl install --set profile=demo -y
kubectl get pods -n istio-system
```

After Istio is ready, apply the focused mesh overlay:

```bash
kubectl apply -k restaurant-gitops/overlays/istio-dev
```
