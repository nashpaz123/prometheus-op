https://docs.google.com/spreadsheets/d/1-fDViIl5xWK8z-1OumJXYKtPVQARR6OMMST7GRWtGPI/edit?gid=72479198#gid=72479198

https://surveys.segmanta.com/c2l9b0/q

https://drive.google.com/drive/folders/1ad9btfYgUhoQFr3Fo1f2b22H7ut4Wqj8

Lets go over:

https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack

https://github.com/prometheus-operator/prometheus-operator

# Letss start by installing Promethous:
The below procedure will install Prometheous , Grafana & AlertManager along with NodeExporter.


1. Install helm and metricserver
~~~
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
~~~
To check if Metric server is already installed run:
~~~
kubectl get deployments/metrics-server -n kube-system
~~~
install it using the following if missing:
~~~
kubectl apply -f https://raw.githubusercontent.com/yanivomc/seminars/K8S/K8S/advanced/hpa/metricserver/components.yaml
~~~
VERIFY:
~~~
kubectl api-resources | grep metrics
(expected results)
nodes  metrics.k8s.io/v1beta1  false   NodeMetrics
pods  metrics.k8s.io/v1beta1 true PodMetrics

~~~
2. Add Promethous Repo:
~~~
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
~~~
Let's see what we'll get:
~~~
helm show values prometheus-community/kube-prometheus-stack
~~~
3. Install Prometheous without custom Values

~~~
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack 
~~~

but clusterip so

~~~
cat <<EOF > prometheus-values.yaml
grafana:
  service:
    type: LoadBalancer

prometheus:
  service:
    type: LoadBalancer

alertmanager:
  service:
    type: LoadBalancer
EOF

helm upgrade prometheus prometheus-community/kube-prometheus-stack -f prometheus-values.yaml
~~~

4. Check SVC (this may take a while)
~~~
ubuntu@ip-172-31-6-140:~$ kubectl get svc | grep LoadBalancer
prometheus-grafana                        LoadBalancer   100.64.222.amazonaws.com   80:32171/TCP       63m
prometheus-kube-prometheus-alertmanager   LoadBalancer   100.68.186.amazonaws.com   9093:31184/TCP     63m
prometheus-kube-prometheus-prometheus     LoadBalancer   100.71.57.amazonaws.com   9090:31877/TCP     63m
~~~

5. Browse Grafana:
~~~
export ALBHOST="http://"$(kubectl  get svc prometheus-grafana --output jsonpath='{.status.loadBalancer.ingress[:1].hostname}'):80/dashboards && echo $ALBHOST
~~~

USER: admin Password: prom-operator

5. Browse Prometheous:
~~~
export ALBHOST="http://"$(kubectl  get svc prometheus-kube-prometheus-prometheus --output jsonpath='{.status.loadBalancer.ingress[:1].hostname}'):9090 && echo $ALBHOST
~~~

5. Browse Alertmanger:
~~~
kubectl  get svc -A |grep alertm #dont forget to add port 9093
~~~

in prometheus try a query, graph:
~~~
sum by (namespace, pod) (rate(container_cpu_usage_seconds_total{container!=""}[1m])) * 1000
~~~
How about a few more queries: https://github.com/nashpaz123/prometheus-op/blob/main/cheetsheets/README.md 

check [my-custom-rules.yaml](https://raw.githubusercontent.com/nashpaz123/prometheus-op/refs/heads/main/my-custom-rules.yaml)
~~~
kubectl apply -f https://raw.githubusercontent.com/nashpaz123/prometheus-op/refs/heads/main/my-custom-rules.yaml #edit NotSoHighCPUUsage
#on the node:
( for i in {1..4}; do while :; do :; done & done; wait ) #to load the cpus
#or as pod:
kubectl run stress-cpu-visible --image=alpine --restart=Never -it -- \
  sh -c 'echo "Starting CPU stress loop..."; while true; do echo "Stressing CPU..."; done'
~~~

 PodMonitor:
~~~
kubectl apply -f https://raw.githubusercontent.com/yanivomc/seminars/K8S/K8S/advanced/monitoring/examples/pod-metrics.yaml
~~~

but kubectl get prometheus prometheus-kube-prometheus-prometheus -oyaml podmonitor matches some basic propmetheus label
~~~
cat <<EOF > prometheus-values.yaml
grafana:
  service:
    type: LoadBalancer

prometheus:
  service:
    type: LoadBalancer
  prometheusSpec:
    podMonitorSelector:
      matchLabels:
        team: frontend

alertmanager:
  service:
    type: LoadBalancer
EOF

helm upgrade prometheus prometheus-community/kube-prometheus-stack -f prometheus-values.yaml
~~~

or for several matches:
~~~
cat <<EOF > prometheus-values.yaml
grafana:
  service:
    type: LoadBalancer

prometheus:
  service:
    type: LoadBalancer
  prometheusSpec:
    podMonitorSelector:
      matchExpressions:
        - key: team
          operator: In
          values:
            - frontend
        - key: release
          operator: In
          values:
            - prometheus

alertmanager:
  service:
    type: LoadBalancer
EOF

helm upgrade prometheus prometheus-community/kube-prometheus-stack -f prometheus-values.yaml

~~~
