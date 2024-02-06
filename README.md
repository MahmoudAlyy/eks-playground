This repo showcases a simple EKS deployment made public with ingress, with HPA using custom metrics from prometheus and a grafana dashboard to monitor node CPU & memory.

## Configuration

### 1. Configure AWS CLI

Run the following command to configure your AWS CLI with your credentials:

```sh
aws configure
```

Enter your AWS Access Key ID, Secret Access Key, and default region when prompted.

### 2. Run terraform

We use terraform to deploy EKS and multiple addons including:
* aws-load-balancer-controller
* metric service
* aws-ebs-csi-driver

Go to `tf` folder and run

```sh
terraform init
terraform apply
```

terraform will output the cluster name.

### 3. Setup kubectl locally 

```
aws eks update-kubeconfig --region us-east-2 --name <cluster-name>
```

### 4. Deploy the app and HPA

```
kubectl apply -f kubernetes/app
```

This will deploy an application and the loadbalancer.
To access the application, navigate to the AWS Console, select the 'Load Balancers' section, and use the DNS name provided there.
HPA will also be deployed but will not work at the moment becuase we have not setup custom metric endpoint yet.

### 5. How do we scale using custom metrics provided by our application?

Essentially our application [podinfo](https://github.com/stefanprodan/podinfo) reveals an `/metrics` endpoint which contains useful metrics like `http_requests_total`.
We tell prometheus to pull the metrics from this service by adding this annotations.
```
annotations:
  prometheus.io/scrape: "true"
```
Prometheus is automtiallcy configued to go the `/metrics` path so we are set.  
Prometheus Adapter retrieves metrics from Prometheus and makes them available through the Metrics API as custom metrics.  
HPA will then query the custom metrics endpoint and scale accordingly.  

### 6. Deploy prometheus and prometheus-adapter

We set alertmanager and pushgateway to false as we dont need them and we want to keep things simple.

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/prometheus --set alertmanager.enabled=false --set prometheus-pushgateway.enabled=false

helm install prometheus-adapter prometheus-community/prometheus-adapter -f kubernetes/prometheus/prometheus-adapter-values.yaml
```

You can access prometheus UI using this cmd
```
kubectl port-forward service/prometheus-server 3000:80
```
You can try searching for `http_requests_total` in the graph tab or for `podinfo-service` in the target tab.


### 7. Generate load

```
kubectl run load-generator --image=williamyeh/hey:latest --restart=Never -- -c 200 -q 100 -z 10m  http://podinfo-service.default.svc.cluster.local
```

Check HPA

```
kubectl get hpa
```

You should see this
```
➜  eks-playground git:(main) ✗ kubectl get hpa
NAME                  REFERENCE                       TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
podinfo-http-scaler   Deployment/podinfo-deployment   2468145m/10   1         3         3          37m
```

### 8. Deploy Grafana dashboard

Apply dashboard config map
```
kubectl apply -f kubernetes/grafana/node-dashboard-config.yaml
```

```
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm upgrade --install grafana grafana/grafana -f kubernetes/grafana/grafana-values.yaml
```

Get password

```
kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Access grafana dashboard on your local 
```
kubectl port-forward service/grafana 4000:80
```

You should find a dashboard that monitors node CPU & memory usage.
<img width="1213" alt="image" src="https://github.com/MahmoudAlyy/eks-playground/assets/57804225/89e28af0-9eb1-457c-9f26-06b170583b30">
