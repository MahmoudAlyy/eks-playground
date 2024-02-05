run terraform output to get cluster name and region

aws eks update-kubeconfig --region us-east-2 --name eks-playground-LesRzzhf


kubectl apply -f kubernetes/sampleWebService.yaml

kubectl autoscale deployment podinfo-deployment --name=podinfo-deployment-austoscaler --cpu-percent=50 --min=1 --max=5


Test autoscaling by genrating load

kubectl run load-generator \
  --image=williamyeh/hey:latest \
  --restart=Never -- -c 200 -q 100 -z 10m  http://podinfo-service.default.svc.cluster.local

kubectl delete pod load-generator

---


---
i faced a problem when i tried to make node node communcatiin (trying to run curl from one node to another for hpa testing)
curl could not reach  http://podinfo-service.default.svc.cluster.local
after debugging turns out the security group was the problem as it only allows ports (1025 - 65535) and podinfo on node was hosted on port 80
to fix another rule to the sg to allow node to node ingress on port 80


----------------

Custom metric autoscaler

https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/user-guides/getting-started.md

install promethus
```
curl -sL https://github.com/prometheus-operator/prometheus-operator/releases/download/v0.71.2/bundle.yaml | kubectl create -f -
```

create service account and service monitor
kubectl apply -f kubernetes/prometheusRole.yaml
kubectl apply -f kubernetes/prometheusService.yaml



challenges faced:
trying to make ingres rewrite target to host prometheus

---
another way



helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus-adapter prometheus-community/prometheus-adapter -f kubernetes/prometheus-adapter-values.yaml

helm install prometheus prometheus-community/prometheus --set alertmanager.enabled=false --set prometheus-pushgateway.enabled=false


  export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=prometheus,app.kubernetes.io/instance=prometheus" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace default port-forward $POD_NAME 9090


delete the other scaler

In a few minutes you should be able to list metrics using the following command(s):
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1/namespaces/default


kubectl get --raw /apis/external.metrics.k8s.io/v1beta1/namespaces/default |jq

* Issue faced:
sum(rate(http_requests_total{namespace!=""}[1m])) by (namespace)
prometheus is scraping data every 1m so in turn, this query returns nothing.


----------------
## Grafana deployment
```
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm upgrade --install grafana grafana/grafana -f kubernetes/grafana-values.yaml
```

kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo



     export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")
     kubectl --namespace default port-forward $POD_NAME 3000



kubectl port-forward service/grafana 3000:80



in this code loki is just setup but not really utilized


# TODO
create folders in kubetenes folder
split the dashboard definition in a sep file