# Kubecon Europe 2020 - virtual event

[Mesh in a Mesh: A Model for Stronger Multi-tenancy of Kubernetes Workloads - Nitish Malhotra & Akash Baid, Affirmed Networks](https://sched.co/Zelb)

## Schedule 

Tuesday, August 18 • 14:30 - 15:05

## Slides
https://drive.google.com/file/d/1Ag_tvDawCTg9sL1r1GB_3mC9_ZMVi5oa/view?usp=sharing

## Demo - Setup (MacOS)

### Pre-requisites

#### Kubernetes Cluster

##### KinD

```bash
kind create cluster --config $PWD/kind.config --name kubecon
kubectl cluster-info --context kind-kubecon
```

##### Docker for Mac / Minikube / Standard Cluster

https://docs.docker.com/docker-for-mac/kubernetes/
https://kubernetes.io/docs/tasks/tools/install-minikube/

#### Istioctl

https://istio.io/latest/docs/setup/getting-started/#download

### Deploy

#### Istio control-plane

```bash
istioctl manifest apply --set profile=demo --set values.gateways.istio-ingressgateway.enabled=false --set values.gateways.istio-egressgateway.enabled=false --set values.global.jwtPolicy=first-party-jwt
```

#### BookInfo App

##### Tenant-1

```bash
kubectl create ns tenant-one
kubectl label namespace tenant-one istio-injection=enabled
kubectl apply -k deploy/multitenancy/tenant-one
```

##### Tenant-2

```bash
kubectl create ns tenant-two
kubectl label namespace tenant-two istio-injection=enabled
kubectl apply -k deploy/multitenancy/tenant-two
```

##### Sleep (for KinD only)

```bash
kubectl create ns user
kubectl apply -k deploy/sleep
```

### Expose Gateways

#### Docker for Mac / Minikube / Standard Cluster

`istio-ingressgateway` service in each tenant namespace has the HTTP2 and HTTPS `NodePort` (you can use `LoadBalancer` on clusters that support externalIPs) set to default values,

##### Tenant-1

+ 31080 (`http2`)
+ 31443 (`https`)

##### Tenant-2

+ 32080 (`http2`)
+ 32443 (`https`)

#### KinD

> `ClusterIP` is reachable only from within the cluster.
>
> We will use the `sleep` container in namespace `user` to try out the applications

Get the ClusterIP for both tenant ingress gateways

```bash
CLUSTER_IP_ONE=$(kubectl -n tenant-one get service istio-ingressgateway -o jsonpath='{.spec.clusterIP}')
CLUSTER_IP_TWO=$(kubectl -n tenant-two get service istio-ingressgateway -o jsonpath='{.spec.clusterIP}')
```

### Try it out

#### Docker for Mac / Minikube / Standard Cluster

[![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/a16e7d6bee6ab779e456)

Follow the screenshot below to setup the client certificates

![Image of postman](assets/images/postman.png)
#### KinD

##### Tenant-1 client to server - Success

```bash
curl -v --resolve "tenant-one.example.com:443:$CLUSTER_IP_ONE" --cacert /etc/certs/cacert --
cert /etc/certs/t1-cert.pem  --key /etc/certs/t1-key.pem  https://tenant-one.example.com:443/api/v1/products
```

##### Tenant-2 client to server - Success

```bash
curl -v --resolve "tenant-two.example.com:443:$CLUSTER_IP_TWO" --cacert /etc/certs/cacert --
cert /etc/certs/t2-cert.pem  --key /etc/certs/t2-key.pem  https://tenant-two.example.com:443//api/v1/products
```

##### Tenant-1 client to Tenant-2 server - Failure

```bash
curl -v --resolve "tenant-two.example.com:443:$CLUSTER_IP_TWO" --cacert /etc/certs/cacert --
cert /etc/certs/t1-cert.pem  --key /etc/certs/t1-key.pem  https://tenant-two.example.com:443//api/v1/products
```
