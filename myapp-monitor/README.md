# 📈 Prometheus Custom Metrics App (Kubernetes Example)

This is a simple Python Flask app that exposes a **custom Prometheus metric** called `myapp_custom_requests_total`. It's designed to run in a Kubernetes cluster and be scraped by Prometheus using a `PodMonitor`.

---

## 🚀 Features

- Prometheus-compatible `/metrics` endpoint
- Custom counter metric for incoming requests
- Deployable via Kubernetes YAML
- Works with Prometheus Operator (e.g., kube-prometheus-stack)

---

## 🧱 Prerequisites

- Docker
- Kubernetes cluster
- Prometheus Operator installed (e.g., via `kube-prometheus-stack`)
- Access to push Docker images (e.g., Docker Hub)

---

## 🐍 App: `app.py`

```python
from flask import Flask
from prometheus_client import Counter, generate_latest, CONTENT_TYPE_LATEST

app = Flask(__name__)

REQUEST_COUNT = Counter("myapp_custom_requests_total", "Total custom requests")

@app.route("/")
def home():
    REQUEST_COUNT.inc()
    return "Hello from custom metric app!"

@app.route("/metrics")
def metrics():
    return generate_latest(), 200, {'Content-Type': CONTENT_TYPE_LATEST}

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
```

---

## 🐳 Dockerfile

```Dockerfile
FROM python:3.9-slim

RUN pip install flask prometheus_client

COPY app.py /app.py

EXPOSE 8000

CMD ["python", "app.py"]
```

---

## 📦 Build & Push the Image

Replace `<your-dockerhub-username>` with your actual Docker Hub username:

```bash
docker build -t <your-dockerhub-username>/myapp:latest .
docker push <your-dockerhub-username>/myapp:latest
```

---

## ☸️ Kubernetes Deployment

Save the following as `myapp.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
    - port: 8000
      targetPort: 8000
      name: http
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: <your-dockerhub-username>/myapp:latest
          ports:
            - containerPort: 8000
```

Apply it:

```bash
kubectl apply -f myapp.yaml
```

---

## 📡 PodMonitor Configuration

Save the following as `podmonitor.yaml`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: myapp-monitor
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: myapp
  podMetricsEndpoints:
    - port: http
      path: /metrics
```

Apply it:

```bash
kubectl apply -f podmonitor.yaml
```

---

## 🔍 Test the Application

Forward the service and generate traffic:

```bash
kubectl port-forward svc/myapp 8080:8000
curl http://localhost:8080/
curl http://localhost:8080/
curl http://localhost:8080/metrics

# or just

kubectl exec -it $(kubectl get pods |grep metrics-app|awk '{print$1}'|tail -1) -- curl localhost:8080/metrics |grep myapp_custom
```

You should see:

```
# HELP myapp_custom_requests_total Total custom requests
# TYPE myapp_custom_requests_total counter
myapp_custom_requests_total 2.0
```

---

## 📈 View the Metric in Prometheus

Open the Prometheus UI and search:

```
myapp_custom_requests_total
```

You can also build a dashboard in Grafana using this metric.

---

## ✅ Done!

You've successfully:

- Deployed a custom-metric-enabled app to Kubernetes
- Set up Prometheus to scrape the metric using a `PodMonitor`
- Viewed your custom metric via Prometheus/Grafana

---

## 🛠 Optional: Clean Up

```bash
kubectl delete -f myapp.yaml
kubectl delete -f podmonitor.yaml
```
