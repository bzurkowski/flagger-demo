# Flagger Demo

- [Prerequisites](#prerequisites)
- [Setup](#setup)
  - [Kubernetes](#kubernetes)
  - [Metrics Server](#metrics-server)
  - [Istio](#istio)
    - [Prometheus](#prometheus)
    - [Grafana](#grafana)
    - [Kiali](#kiali)
  - [Flagger](#flagger)
  - [Test application](#test-application)
  - [Fortio](#fortio)
  - [Chaos Mesh](#chaos-mesh)
  - [Port forwarding](#port-forwarding)
  - [Terminal](#terminal)
- [Demo](#demo)
  - [Canary setup](#canary-setup)
  - [Load generation](#load-generation)
  - [Automated canary promotion](#automated-canary-promotion)
  - [Automated canary rollback](#automated-canary-rollback)
- [Cleanup](#cleanup)
  - [Test application](#test-application-1)
  - [All](#all)

## Prerequisites

Install required dependencies:

```bash
brew install kind kubectl helm watch
```
## Setup

### Kubernetes

Create kind cluster:

```bash
kind create cluster --name flagger-demo --config cluster-config.yaml
```

### Metrics Server

Add Helm repository:

```
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
```

Deploy Metrics Server:

```
helm upgrade metrics-server metrics-server/metrics-server \
    --install \
    --namespace kube-system \
    --set args={--kubelet-insecure-tls}
```

### Istio

Download Istio release:

```
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.15.3 sh -
```

Add `istioctl` binary to the system path:

```
sudo mv ./istio-1.15.3/bin/istioctl /usr/local/bin/
```

Deploy Istio control plane:

```
istioctl manifest install --set profile=default
```

#### Prometheus

Deploy Prometheus addon:

```
kubectl -n istio-system apply -f ./istio-1.15.3/samples/addons/prometheus.yaml
```

#### Grafana

Deploy Grafana addon:

```
kubectl -n istio-system apply -f ./istio-1.15.3/samples/addons/grafana.yaml
```

#### Kiali

Deploy Kiali addon:

```
kubectl -n istio-system apply -f ./istio-1.15.3/samples/addons/kiali.yaml
```

### Flagger

Add Helm repository:

```
helm repo add flagger https://flagger.app
```

Install Flagger CRDs:

```
kubectl apply -f https://raw.githubusercontent.com/fluxcd/flagger/main/artifacts/flagger/crd.yaml
```

Deploy Flagger for Istio:

```
helm upgrade flagger flagger/flagger \
    --install \
    --namespace=istio-system \
    --set crd.create=false \
    --set meshProvider=istio \
    --set metricsServer=http://prometheus:9090
```

### Test application

Create namespace with Istio proxy-injection enabled:

```
kubectl create ns test
kubectl label namespace test istio-injection=enabled
```

Deploy sample application:

```
kubectl -n test apply -k https://github.com/fluxcd/flagger//kustomize/podinfo?ref=main
```

### Fortio

Clone repository:

```
git clone https://github.com/openrca/orca-testapps.git
```

Deploy Fortio:

```
helm install fortio ./orca-testapps/fortio --namespace test
```

### Chaos Mesh

Add Helm repository:

```
helm repo add chaos-mesh https://charts.chaos-mesh.org
```

Deploy Chaos Mesh:

```
helm upgrade chaos-mesh chaos-mesh/chaos-mesh \
    --version 2.4.2 \
    --install \
    --namespace=chaos \
    --create-namespace \
    --set chaosDaemon.runtime=containerd \
    --set chaosDaemon.socketPath=/run/containerd/containerd.sock \
    --set dashboard.create=true \
    --set dashboard.securityMode=false
```

### Port forwarding

Port-forward [Grafana](http://localhost:3000):

```
kubectl -n istio-system port-forward svc/grafana 3000
```

Port-forward [Kiali](http://localhost:20001):

```
kubectl -n istio-system port-forward svc/kiali 20001
```

Port-forward [Fortio](http://localhost:8082/fortio):

```
kubectl -n test port-forward svc/fortio 8082
```

### Terminal

![](/assets/images/terminal-setup.png)

Setup watch on test application Deployments:

```
watch kubectl -n test get deploy
```

Setup watch on test application Pods:

```
watch kubectl -n test get pods
```

Setup watch on test application Services:

```
watch kubectl -n test get svc
```

Setup watch on canary events:

```
kubectl -n test get events \
    --watch \
    --field-selector involvedObject.kind=canary \
    --field-selector involvedObject.name=podinfo |grep canary/podinfo
```

Setup watch on Istio configuration for test application service:

```
watch 'kubectl -n test describe vs podinfo |grep Spec -A 50'
```

## Demo

### Canary setup

1. Create canary configuration:

    ```
    kubectl -n test apply -f ./canary-demo.yaml
    ```

2. Inspect created resources (Deployments, Services, Istio configuration).

### Load generation

1. Port-forward [test application](http://localhost:9898):

    ```
    kubectl -n test port-forward svc/podinfo 9898
    ```

2. Ensure the [test application](http://localhost:9898) is working.

3. Generate traffic to test application using [Fortio](http://localhost:8082/fortio)
web interface:

    ![](/assets/images/fortio-loadtest-promotion.png)

4. Check service graph in [Kiali](http://localhost:20001).

5. Check service metrics in [Grafana](http://localhost:3000).

### Automated canary promotion

1. Trigger canary deployment by updating the container image:

    ```
    kubectl -n test set image deployment/podinfo \
        podinfod=ghcr.io/stefanprodan/podinfo:6.0.1
    ```

2. Check service graph in [Kiali](http://localhost:20001):

    ![](/assets/images/kiali-service-graph-promotion.png)

3. Check service metrics in [Grafana](http://localhost:3000):

    ![](/assets/images/grafana-service-metrics-promotion.png)

4. Observe canary processing in terminal:

    ```
    New revision detected! Scaling up podinfo.test
    Starting canary analysis for podinfo.test
    Advance podinfo.test canary weight 10
    Advance podinfo.test canary weight 20
    Advance podinfo.test canary weight 30
    Advance podinfo.test canary weight 40
    Advance podinfo.test canary weight 50
    Copying podinfo.test template spec to podinfo-primary.test
    Routing all traffic to primary
    Promotion completed! Scaling down podinfo.test
    ```

### Automated canary rollback

1. Trigger canary deployment by updating the container image:

    ```
    kubectl -n test set image deployment/podinfo \
        podinfod=ghcr.io/stefanprodan/podinfo:6.0.2
    ```

2. Wait until Flagger shifts portion of the traffic to the canary:

    ```
    New revision detected! Scaling up podinfo.test
    Starting canary analysis for podinfo.test
    Advance podinfo.test canary weight 10
    ```

3. Generate traffic to application endpoint responding with `500 Internal Error`:

    ![](/assets/images/fortio-loadtest-rollback.png)

4. Check service graph in [Kiali](http://localhost:20001):

    ![](/assets/images/kiali-service-graph-rollback.png)

5. Check service metrics in [Grafana](http://localhost:3000):

    ![](/assets/images/grafana-service-metrics-rollback.png)

6. Observe canary processing in terminal:

    ```
    New revision detected! Scaling up podinfo.test
    Starting canary analysis for podinfo.test
    Advance podinfo.test canary weight 10
    Advance podinfo.test canary weight 20
    Halt advancement no values found for istio metric request-duration probably podinfo.test is not receiving traffic
    Halt advancement no values found for istio metric request-success-rate probably podinfo.test is not receiving traffno values found","canary":"podinfo.test"}
    Advance podinfo.test canary weight 30
    Advance podinfo.test canary weight 40
    Halt podinfo.test advancement success rate 54.23% < 99%
    Halt podinfo.test advancement success rate 49.88% < 99%
    Halt podinfo.test advancement success rate 47.11% < 99%
    Rolling back podinfo.test failed checks threshold reached 5
    Canary failed! Scaling down podinfo.test
    ```

## Cleanup

### Test application

Delete canary configuration:

```
kubectl -n test delete -f ./canary-demo.yaml
```

Delete Fortio:

```
helm -n test delete fortio
```

Delete test application:

```
kubectl -n test delete -k https://github.com/fluxcd/flagger//kustomize/podinfo?ref=main
```

Delete application namespace:

```
kubectl delete ns test
```

### All

Delete kind cluster:

```
kind delete cluster --name flagger-demo
```
