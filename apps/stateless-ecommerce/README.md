# Northstar Market — Stateless Kubernetes eCommerce Demo

A responsive, static eCommerce storefront (HTML/CSS/JavaScript) served by NGINX. Product catalog, search, categories, and in-memory shopping cart. **No backend, database, authentication, persistent cart, actual orders, or payment processing.** Refreshing the page clears the cart; each browser keeps its own cart in memory.

## Prerequisites

- Docker, kubectl, and access to a Kubernetes cluster
- Git for version control
- For Kind, the `kind` CLI and a running cluster

## Git

```bash
git init
git add .
git commit -m "Initial stateless ecommerce app"
# Optional: create an empty GitHub repository, then:
git branch -M main
git remote add origin https://github.com/YOUR_USER/stateless-ecommerce.git
git push -u origin main
```

## Build and test Docker locally

```bash
docker build -t ecommerce:1.0.0 .
docker run --rm -p 8085:8080 ecommerce:1.0.0
# In a second terminal: curl http://localhost:8085/healthz
```

Open http://localhost:8085. Stop with Ctrl+C.

## Deploy to Kind

```bash
kind get clusters
# Replace CLUSTER_NAME with the actual name from the previous command.
kind load docker-image ecommerce:1.0.0 --name CLUSTER_NAME
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/deployment.yaml
kubectl rollout status deployment/ecommerce -n ecommerce
kubectl get pods,svc,endpoints -n ecommerce
```

**Important:** Kind loads images into a specific cluster; loading is needed after rebuilding. On a kubeadm/other non-Kind cluster, use a registry accessible to every worker node, e.g. tag and push to Docker Hub, and replace the `image:` in `k8s/deployment.yaml` with `YOUR_USER/ecommerce:1.0.0` before deploying. Do not assume local Docker images are visible to Kubernetes worker nodes.

## Access from Ubuntu host

```bash
kubectl port-forward -n ecommerce svc/ecommerce 8084:80
```

Open http://localhost:8084. Leave this terminal running.

## Access from another PC on the same LAN

Stop the previous port-forward (Ctrl+C) and start:

```bash
kubectl port-forward -n ecommerce svc/ecommerce 8084:80 --address=0.0.0.0
```

On the other PC open `http://192.168.68.110:8084` **if that is still your Ubuntu machine's LAN IP** (verify using `ip -4 addr show`). Your firewall and Wi-Fi isolation settings must allow it. If UFW is enabled, restrict access to your trusted subnet:

```bash
sudo ufw allow from 192.168.68.0/24 to any port 8084 proto tcp
```

Port-forward is a temporary development method. For permanent exposure, use a properly configured Ingress/Gateway or NodePort; Kind requires additional Docker port mapping or host networking configuration for external NodePort access.

## Updating

Change files in `app/`, commit with Git, then rebuild using a **new image tag** (e.g. `ecommerce:1.0.1`). Load that tag into Kind, then:

```bash
kubectl set image deployment/ecommerce ecommerce=ecommerce:1.0.1 -n ecommerce
kubectl rollout status deployment/ecommerce -n ecommerce
```

For registry-backed clusters, push the new tag before updating the Deployment. Never reuse a tag and expect all nodes to pick up the change automatically.

## Troubleshooting

```bash
kubectl get pods -n ecommerce -o wide
kubectl get endpoints ecommerce -n ecommerce
kubectl describe pods -n ecommerce
kubectl logs -n ecommerce deployment/ecommerce
sudo ss -ltnp 'sport = :8084'
```

If pods show `ImagePullBackOff`, check image availability on all nodes and `imagePullPolicy`. If the Service has no endpoints, check pod readiness and selector labels. If the host port is occupied, stop the old port-forward or use another local port.
