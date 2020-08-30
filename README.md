# chaos-monitoring
Repo for chaos monitoring setups.

# Monitor Chaos on Sock-Shop

Chaos experiments on sock-shop app with grafana dashboard to monitor it. 

## Step-0: Obtain the demo artefacts

- Clone the sock-shop repo

  ```
  git clone https://github.com/ishangupta-ds/chaos-monitoring.git
  cd chaos-monitoring
  ```


## Step-1: Setup Sock-Shop Microservices Application

- Create sock-shop namespace on the cluster

  ```
  kubectl create ns sock-shop
  ```

- Apply the sock-shop microservices manifests

  ```
  kubectl apply -f sample-application/sock-shop/deploy/sock-shop/
  ```

- Wait until all services are up. Verify via `kubectl get pods -n sock-shop`

## Step-2: Setup the LitmusChaos Infrastructure

- Install the litmus chaos operator and CRDs 

  ```
  kubectl apply -f https://litmuschaos.github.io/litmus/litmus-operator-v1.7.0.yaml
  ```

- Install the litmus-admin serviceaccount for centralized/admin-mode of chaos execution

  ```
  kubectl apply -f https://litmuschaos.github.io/litmus/litmus-admin-rbac.yaml
  ```

- Install the chaos experiments in admin(litmus) namespace

  ```
  kubectl apply -f https://hub.litmuschaos.io/api/chaos/1.7.0?file=charts/generic/experiments.yaml -n litmus 
  ```

- Install the chaos experiment metrics exporter and chaos event exporter

  ```
  kubectl apply -f sample-application/sock-shop/deploy/litmus-metrics/01-event-router-cm.yaml
  kubectl apply -f sample-application/sock-shop/deploy/litmus-metrics/02-event-router.yaml
  kubectl apply -f sample-application/sock-shop/deploy/litmus-metrics/03-chaos-exporter.yaml
  ```

## Step-3: Setup the Monitoring Infrastructure

  ```
  kubectl create ns monitoring
  ```

- Create the operator to instanciate all CRDs
  ```
  kubectl -n monitoring apply -f deploy/prometheus-operator/
  ```

- Deploy monitoring components
  ```
  kubectl -n monitoring apply -f deploy/node-exporter/
  kubectl -n monitoring apply -f deploy/kube-state-metrics/
  kubectl -n monitoring apply -f deploy/alertmanager
  ```

- Deploy prometheus instance and all the service monitors for targets
  ```
  kubectl -n monitoring apply -f deploy/prometheus-cluster-monitoring/
  ```

- Apply the grafana manifests in specified order after deploying prometheus for all metrics.

  ```
  kubectl apply -f sample-application/sock-shop/deploy/monitoring/01-grafana-deployment.yaml
  kubectl apply -f sample-application/sock-shop/deploy/monitoring/02-grafana-svc.yaml
  ```

- Access the grafana dashboard via the NodePort (or loadbalancer) service IP or via a port-forward operation on localhost

  Note: To change the service type to Loadbalancer, perform a `kubectl edit svc prometheus-k8s -n monitoring` and replace 
  `type: NodePort` to `type: LoadBalancer`

  ```
  kubectl get svc -n monitoring 
  ```

  Default username/password credentials: `admin/admin`

- Add the prometheus datasources from monitoring namespace as DS_PROMETHEUS and litmus-monitoring namespace as Prometheus for Grafana via the Grafana Settings menu

- Import the grafana dashboard "Sock-Shop Performance" provided [here](https://raw.githubusercontent.com/ksatchit/sock-shop/master/deploy/monitoring/10-grafana-dashboard.json)

  ![image](https://user-images.githubusercontent.com/21166217/87426547-f28d5300-c5fc-11ea-95da-e091fb07f1b5.png)

## Step-4: Execute the Chaos Experiments


- For the sake of illustration, let us execute a CPU hog experiment on the `catalogue` microservice & a Memory Hog experiment on 
  the `orders` microservice in a staggered manner
 

  ```
  kubectl apply -f chaos/catalogue/catalogue-pod-cpu-hog.yaml
  ```

  Wait for ~60s

  ```
  kubectl apply -f chaos/orders/orders-pod-memory-hog.yaml
  ```

  Wait for ~60s

  ```
  kubectl apply -f chaos/catalogue/catalogue-node-cpu-hog.yaml
  ```

  Wait for ~60s

  ```
  kubectl apply -f chaos/orders/orders-node-memory-hog.yaml
  ```
  
- Verify execution of chaos experiments

  ```
  kubectl describe chaosengine catalogue-cpu-hog -n litmus
  kubectl describe chaosengine orders-node-memory-hog -n litmus
  ```
  
## Step-5: Visualize Chaos Impact

- Observe the impact of chaos injection through increased Latency & reduced QPS (queries per second) on the microservices 
  under test. 

  ![image](https://user-images.githubusercontent.com/21166217/87426747-4d26af00-c5fd-11ea-8d82-dabf6bc9048a.png)

  ![image](https://user-images.githubusercontent.com/21166217/87426820-6cbdd780-c5fd-11ea-88de-1fe8a1b5b503.png)


