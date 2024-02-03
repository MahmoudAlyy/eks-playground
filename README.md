run terraform output to get cluster name and region

aws eks update-kubeconfig --region us-east-2 --name eks-playground-LesRzzhf


kubectl apply -f kubernetes/sampleWebService.yaml

kubectl autoscale deployment nginx-deployment --name=nginx-deployment-austoscaler --cpu-percent=50 --min=1 --max=5


Test autoscaling by genrating load

kubectl run load-generator \
  --image=williamyeh/hey:latest \
  --restart=Never -- -c 200 -q 100 -z 10m  http://nginx-service.default.svc.cluster.local

kubectl delete pod load-generator

---


---
i faced a problem when i tried to make node node communcatiin (trying to run curl from one node to another for hpa testing)
curl could not reach  http://nginx-service.default.svc.cluster.local
after debugging turns out the security group was the problem as it only allows ports (1025 - 65535) and nginx on node was hosted on port 80
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
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1