# Noor & Naqsh — Desi Boutique

Responsive demo storefront for women's Pakistani shalwar kameez, served with Apache HTTP Server and deployed to Kind Kubernetes.

## Features
- Responsive homepage, eight sample suits, category filters
- Browser-side shopping bag with localStorage
- Demo checkout (no payments or orders)
- Demo newsletter (does not collect or transmit email addresses)
- Two Apache replicas, ClusterIP Service, probes, resource limits, non-root containers

## Requirements
Docker, Kind, kubectl on Ubuntu.

## Run locally in Docker
```bash
docker build -t desi-boutique:1.0.0 .
docker run --rm -p 8085:8080 desi-boutique:1.0.0
# Open http://localhost:8085
```

## Run in Kubernetes
```bash
kind create cluster --name boutique --config kind-config.yaml
kind load docker-image desi-boutique:1.0.0 --name boutique
kubectl config use-context kind-boutique
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/deployment.yaml
kubectl rollout status deployment/desi-boutique -n desi-boutique --timeout=180s
kubectl get pods -n desi-boutique -o wide
kubectl port-forward -n desi-boutique svc/desi-boutique 8084:80 --address=127.0.0.1
# Open http://localhost:8084
```

## Scale
```bash
kubectl scale deployment/desi-boutique --replicas=3 -n desi-boutique
```

## Delete cluster
```bash
kind delete cluster --name boutique
```

## Security notes
The demo has no backend, database, login, payment integration, or real inventory. The cart uses browser localStorage, so it is not synchronized between devices. Do not accept real orders or payment details with this project. Port-forward is bound to localhost for safety. Public hosting requires additional deployment and security controls. Google Fonts require an external connection; fallback fonts are provided.
