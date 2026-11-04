# Multi-WebApps Kubernetes Ingress

**Four independent web applications · Eight application replicas · Three-node Kind cluster · One NGINX Ingress entry point**

A hands-on DevOps project demonstrating how to containerize, deploy, expose, scale, and troubleshoot multiple web applications on Kubernetes. The project combines an NGINX-served eCommerce site with three Apache-served sites and uses **path-based HTTP routing** through the NGINX Ingress Controller.

> **Environment:** Local Ubuntu host running Docker, Kind, Kubernetes, `kubectl`, and Helm. The `localhost:88` URLs in this README are local development URLs, not publicly hosted websites.

## 1. Project objective

Deploy four separate web applications into a single Kubernetes cluster while maintaining:

- **Isolation:** Each application has its own Deployment and ClusterIP Service.
- **Availability:** Two replicas per application (eight application Pods in total).
- **Centralized access:** One Ingress Controller routes requests based on the URL path.
- **Reproducibility:** Source code, Dockerfiles, Kind configuration, and Kubernetes manifests are version controlled.
- **Observability and troubleshooting:** Validate Pods, Services, EndpointSlices, Ingress rules, HTTP responses, and controller logs.

## 2. Why build this project?

Running each application on its own host port becomes cumbersome as the number of services grows. Kubernetes offers declarative workload management, service discovery, and replica management. An Ingress Controller provides a single HTTP entry point, so clients can reach multiple applications through distinct paths instead of managing four separate ports.

This is a **local learning and portfolio environment**, not a production-ready deployment. Production deployments would additionally require TLS, DNS, access controls, image registries, secret management, persistent monitoring, resource planning, and an appropriate external traffic entry point.

## 3. Architecture

```text
Ubuntu browser / curl
        |
        | http://localhost:88/<application>/
        v
kubectl port-forward (host port 88 -> controller port 80)
        |
        v
NGINX Ingress Controller (namespace: ingress-nginx)
        |
        +-- /ecom/     --> stateless-ecommerce Service --> 2 NGINX Pods
        +-- /shop/     --> online-shopping Service     --> 2 Apache Pods
        +-- /boutique/ --> desi-boutique Service       --> 2 Apache Pods
        +-- /pizza/    --> pizza Service               --> 2 Apache Pods

Kind cluster: multi-webapps
  - 1 control-plane node
  - 2 worker nodes
Application namespace: webapps
```

**Request flow:** Client → local port-forward → Ingress Controller → path-matched ClusterIP Service → ready Pod endpoint → application web server.

> The Ingress Controller is an additional infrastructure workload; the **eight Pods** count refers only to the four applications.

## 4. Technology stack

| Technology | Role |
|---|---|
| Ubuntu / Linux | Development host and CLI environment |
| Docker | Build application container images |
| Kind | Run a multi-node Kubernetes cluster in Docker |
| Kubernetes | Deployments, replicas, Pods, Services, namespaces, and Ingress resources |
| NGINX | Serve the stateless eCommerce application |
| Apache HTTP Server | Serve the shopping, boutique, and pizza applications |
| NGINX Ingress Controller | Route HTTP requests by path |
| Helm | Install and manage the Ingress Controller |
| Git / GitHub | Version control and share project source |

## 5. Applications and routing

| Application | Deployment / Service | Container image | Container port | Public path |
|---|---|---|---:|---|
| Stateless eCommerce | `stateless-ecommerce` | `stateless-ecommerce:1.0.2` | 8080 | `/ecom/` |
| Online Shopping | `online-shopping` | `online-shopping-apache:1.0.4` | 8080 | `/shop/` |
| Desi Boutique | `desi-boutique` | `desi-boutique-apache:1.0.1` | 8080 | `/boutique/` |
| Pizza House | `pizza` | `pizza-apache:1.0.1` | 8080 | `/pizza/` |

Each Kubernetes Service listens on **port 80** and forwards to application containers on **port 8080**. Each Deployment targets **two replicas**.

## 6. Repository structure

```text
multi-webapps-k8s/
├── apps/
│   ├── stateless-ecommerce/
│   ├── online_shopping_app/
│   ├── desi-boutique-apache-k8s/
│   │   └── desi-boutique/
│   └── pizza-apache-k8s/
├── kind-config.yaml
├── four-websites.yaml
├── ingress.yaml
├── .gitignore
└── README.md
```

The application directories include source files and Docker build instructions. Some also contain standalone Kubernetes manifests retained from their individual projects. For this combined deployment, use the **top-level** `four-websites.yaml` and `ingress.yaml` rather than applying all nested manifests indiscriminately.

## 7. Prerequisites

Install and verify the following on Ubuntu:

- Docker Engine, with permission to run `docker` commands.
- Kind (`kind`).
- Kubernetes CLI (`kubectl`).
- Helm (`helm`).
- Git (`git`).

```bash
Docker --version 2>/dev/null || docker --version
kind version
kubectl version --client
helm version --short
git --version
```

Make sure Docker is running:

```bash
sudo systemctl status docker --no-pager
```

> These commands assume the repository is already cloned or available locally. Use the same Linux account for `kind`, `kubectl`, and `helm`, or explicitly configure the correct kubeconfig. In particular, root and a regular user may have different Kubernetes contexts.

## 8. Execution: deploy from source

### Step 1 — Clone the repository

```bash
git clone https://github.com/harisamjad0158/multi-webapps-kubernetes-ingress.git
cd multi-webapps-kubernetes-ingress
```

If working in the existing local project directory, use:

```bash
cd /home/adminx/Downloads/multi-webapps-k8s
```

### Step 2 — Create the Kind cluster

Review the cluster configuration:

```bash
cat kind-config.yaml
```

Create the cluster:

```bash
kind create cluster --name multi-webapps --config kind-config.yaml
kubectl config use-context kind-multi-webapps
kubectl get nodes -o wide
```

**Expected topology:** one control-plane node and two worker nodes, all `Ready`.

If the cluster already exists, **do not create it again**. Select its context instead:

```bash
kind get clusters
kubectl config use-context kind-multi-webapps
```

### Step 3 — Build Docker images

Run from the repository root:

```bash
docker build -t stateless-ecommerce:1.0.2 \
  ./apps/stateless-ecommerce

docker build -t online-shopping-apache:1.0.4 \
  ./apps/online_shopping_app

docker build -t desi-boutique-apache:1.0.1 \
  ./apps/desi-boutique-apache-k8s/desi-boutique

docker build -t pizza-apache:1.0.1 \
  ./apps/pizza-apache-k8s
```

These image names and tags must match the `image:` entries in `four-websites.yaml`. If the Dockerfiles change, rebuild and use an updated image tag.

Check the images:

```bash
docker images | grep -E 'stateless-ecommerce|online-shopping-apache|desi-boutique-apache|pizza-apache'
```

### Step 4 — Load images into Kind

Kind nodes use their own container runtime; locally built images are not automatically available to the cluster.

```bash
kind load docker-image stateless-ecommerce:1.0.2 --name multi-webapps
kind load docker-image online-shopping-apache:1.0.4 --name multi-webapps
kind load docker-image desi-boutique-apache:1.0.1 --name multi-webapps
kind load docker-image pizza-apache:1.0.1 --name multi-webapps
```

For locally loaded images, ensure the application manifests do not require an unavailable remote registry image. Use a suitable `imagePullPolicy` such as `IfNotPresent` for non-`latest` tags.

### Step 5 — Deploy all four applications

Review the manifest before applying:

```bash
kubectl apply --dry-run=client -f four-websites.yaml
kubectl apply -f four-websites.yaml
```

The manifest is intended to create the `webapps` namespace, four Deployments, and four ClusterIP Services. If it does **not** define the namespace, create it first:

```bash
kubectl create namespace webapps --dry-run=client -o yaml | kubectl apply -f -
```

Then reapply the workload manifest if necessary:

```bash
kubectl apply -f four-websites.yaml
```

Wait for the Deployments:

```bash
kubectl rollout status deployment/stateless-ecommerce -n webapps --timeout=180s
kubectl rollout status deployment/online-shopping -n webapps --timeout=180s
kubectl rollout status deployment/desi-boutique -n webapps --timeout=180s
kubectl rollout status deployment/pizza -n webapps --timeout=180s
```

Verify:

```bash
kubectl get deployments,pods,services -n webapps -o wide
kubectl get endpointslices -n webapps
```

**Expected:** four Deployments with `2/2` ready, eight running application Pods, and four Services with ready endpoints.

### Step 6 — Install the NGINX Ingress Controller

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=ClusterIP \
  --wait --timeout 5m
```

Verify:

```bash
kubectl get pods,services -n ingress-nginx
kubectl rollout status deployment/ingress-nginx-controller \
  -n ingress-nginx --timeout=180s
```

The controller should be running and ready.

### Step 7 — Configure path-based routing

Apply the repository's Ingress manifest:

```bash
kubectl apply -f ingress.yaml
kubectl get ingress -n webapps
kubectl describe ingress multi-webapps-ingress -n webapps
```

The rules route these paths to the matching Services:

```text
/ecom      -> stateless-ecommerce:80
/shop      -> online-shopping:80
/boutique  -> desi-boutique:80
/pizza     -> pizza:80
```

**Important:** The images are configured to serve content under these URL prefixes. Do **not** add a blanket `nginx.ingress.kubernetes.io/rewrite-target` annotation unless the application configurations are changed to expect rewritten paths. Otherwise, CSS, JavaScript, or other assets may break.

### Step 8 — Access all four websites through one port

Start a port-forward in a dedicated terminal:

```bash
kubectl port-forward -n ingress-nginx \
  deployment/ingress-nginx-controller \
  88:80 --address=127.0.0.1
```

Keep that terminal open. From another terminal or a browser **on the Ubuntu host**, open:

- **eCommerce:** http://localhost:88/ecom/
- **Online Shopping:** http://localhost:88/shop/
- **Desi Boutique:** http://localhost:88/boutique/
- **Pizza House:** http://localhost:88/pizza/

The port-forward is temporary: stopping its process closes the local entry point. `localhost` on a different computer refers to that other computer, not the Ubuntu host.

## 9. Validation and smoke tests

Test all four application paths:

```bash
for path in ecom shop boutique pizza; do
  curl -sS --max-time 10 -o /dev/null \
    -w "$path: HTTP %{http_code}, time %{time_total}s\n" \
    "http://127.0.0.1:88/$path/"
done
```

**Expected for healthy sites:** `HTTP 200` for all four paths.

Inspect Pods and replicas:

```bash
kubectl get pods -n webapps -o wide
kubectl get deployments -n webapps
```

Inspect routing and endpoints:

```bash
kubectl get ingress -n webapps
kubectl describe ingress multi-webapps-ingress -n webapps
kubectl get endpointslices -n webapps
```

Inspect Ingress access/error logs:

```bash
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller --tail=100
```

### Demonstrate self-healing

Delete one application Pod (replace the placeholder with a real Pod name):

```bash
kubectl get pods -n webapps
kubectl delete pod -n webapps <POD_NAME>
kubectl get pods -n webapps -w
```

The Deployment's ReplicaSet should create a replacement Pod to restore the desired replica count. Do not run this demonstration during important live use.

### Demonstrate scaling

Temporarily increase an application's replicas:

```bash
kubectl scale deployment/stateless-ecommerce -n webapps --replicas=3
kubectl rollout status deployment/stateless-ecommerce -n webapps
kubectl get pods -n webapps
```

Restore the desired state:

```bash
kubectl scale deployment/stateless-ecommerce -n webapps --replicas=2
```

If you reapply the original manifest, it should also restore its declared replica count.

## 10. Troubleshooting guide

| Symptom | Likely cause | Check / remedy |
|---|---|---|
| `HTTP 000` or curl timeout | Stale or unhealthy port-forward, or connectivity issue | Inspect port-forward terminal; retry with a fresh forward on an unused port |
| `bind: address already in use` | Another process owns local port 88 | `sudo ss -ltnp 'sport = :88'`; stop only the identified process |
| `kubectl` tries `localhost:8080` | Missing/incorrect kubeconfig or context | `kubectl config current-context`; use the correct account and `kind-multi-webapps` context |
| `ImagePullBackOff` | Kind cannot find the image | Confirm image tags; rebuild and `kind load docker-image` |
| `CrashLoopBackOff` | Container fails at startup | `kubectl logs -n webapps <POD_NAME> --previous`; inspect server config and ports |
| Service has no endpoints | Pod labels do not match Service selector, or Pods are unready | `kubectl describe svc -n webapps <SERVICE>` and `kubectl get pods --show-labels -n webapps` |
| `404` through Ingress | Wrong path or Ingress rule | Inspect `ingress.yaml` and `kubectl describe ingress` |
| HTML works but assets fail | Application base path or web-server alias mismatch | Check asset URLs and Apache/NGINX configuration; avoid unplanned path rewriting |
| `503` from Ingress | No healthy backend endpoint or backend unavailable | Inspect Pods, readiness, Service selectors, and EndpointSlices |

If port 88 behaves unexpectedly, test with an unused local port without changing Kubernetes:

```bash
kubectl port-forward -n ingress-nginx \
  deployment/ingress-nginx-controller 8188:80
```

Then test `http://127.0.0.1:8188/ecom/` and the other paths. A successful response on 8188 but not 88 points toward the local forwarding process or port rather than the application manifests.

## 11. What I learned

1. **Multi-node Kubernetes:** Created and operated a three-node Kind cluster and verified node readiness.
2. **Containerization:** Built versioned Docker images for applications served by NGINX and Apache.
3. **Image distribution in Kind:** Learned why locally built images must be explicitly loaded into Kind nodes.
4. **Declarative deployments:** Used Deployments to maintain two replicas of each application and manage rollouts.
5. **Service discovery and traffic distribution:** Used ClusterIP Services and endpoint discovery to reach healthy Pods without relying on Pod IPs.
6. **Ingress and HTTP routing:** Configured a single NGINX Ingress Controller to route `/ecom/`, `/shop/`, `/boutique/`, and `/pizza/` to different backends.
7. **Web-server path configuration:** Understood how URL prefixes, Apache aliases, NGINX configuration, and frontend asset paths interact.
8. **Helm operations:** Installed and verified an Ingress Controller using Helm.
9. **Troubleshooting:** Investigated port-forward timeouts, occupied local ports, kubeconfig mismatches, readiness, endpoint routing, and application logs.
10. **Git workflows:** Consolidated separate application sources into a single repository and practiced Git authentication, commits, and remote-history integration.

## 12. Kubernetes concepts demonstrated

- **Pod:** The running unit containing an application container.
- **Deployment:** Declares and maintains the desired Pod replicas; supports updates and rollbacks.
- **ReplicaSet:** Helps a Deployment maintain its requested Pod count.
- **Service (ClusterIP):** Provides a stable in-cluster address and routes traffic to matching ready Pods.
- **EndpointSlice:** Tracks network endpoints behind a Service.
- **Ingress:** Declares HTTP routing rules; requires an Ingress Controller to enforce them.
- **Namespace:** Logically groups application and infrastructure resources.
- **Readiness probe:** Helps prevent traffic from being routed to an unready application.
- **Liveness probe:** Allows Kubernetes to restart containers that fail a configured health check.
- **Port-forward:** Creates a temporary local tunnel for development or debugging; it is not a production load balancer.

## 13. Possible future improvements

- Automate builds and validation with GitHub Actions.
- Push images to a container registry and pin them by immutable digest.
- Add automated smoke tests and application unit tests.
- Introduce HTTPS, DNS, and certificate management for an externally accessible environment.
- Add Prometheus/Grafana metrics, centralized logs, and alerting.
- Configure resource requests/limits, security contexts, and network policies consistently.
- Add Helm charts or Kustomize overlays for repeatable environment management.
- Configure a persistent local ingress exposure mechanism rather than relying on a terminal port-forward.

## 14. Cleanup

Remove only the application resources:

```bash
kubectl delete -f ingress.yaml
kubectl delete -f four-websites.yaml
```

To remove the entire local Kind cluster and all workloads running in it:

```bash
kind delete cluster --name multi-webapps
```

**Warning:** Deleting the cluster removes its Kubernetes resources. Do this only when the lab is no longer needed.

## 15. Repository

GitHub: https://github.com/harisamjad0158/multi-webapps-kubernetes-ingress

**Project focus:** Kubernetes orchestration, Docker image lifecycle, multi-application hosting, NGINX Ingress, Apache/NGINX web servers, deployment reliability, and Linux-based troubleshooting.
