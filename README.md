## Intro

A **service** is an object (network abstraction) that declares a common policy to access a set of **pods**
Exposes the pods under a single IP address and a single DNS name.

When you create a service, a corresponding DNS entry is automatically created  `<service_name>.<namespace>.svc.cluster.local` by the cluster's DNS system (e.g. CoreDNS). `kubectl get po -n kube-system | grep dns`

Services are assigned DNS A and/or AAAA records, depending on the IP family or families of the Service

[CoreDNS](https://kubernetes.io/docs/concepts/services-networking/service/#dns) is cluster-aware DNS server that watches the Kubernete API for new Services and creates a set of DNS records for each one, it uses `cluster.local` domain (by default).
For cloud providers check DNS config: `kubectl get configmap coredns -n kube-system -o yaml`

Flow: Pods (use /etc/resolv.conf) ---to query---> CoreDNS (CoreDNS resolves Kubernetes Service names i.e. for local setup `my-service.default.svc.cluster.local`)

Once your services have been established in Kubernetes, you likely want to get external traffic into your cluster. One way to achieve it is to use [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

## Getting external traffic in the cluster: Setup

Ingress manages manages external access to the services, it can provide load balancing, SSL termination and name-based virtual hosting.

You will need to have in your cluster first and Ingress controller and only afterwards you can createa a Ingress resource. Ingress resources are used to expose HTTP and HTTPS routes from outside the cluster to services within the cluster.  Traffic routing is controlled by rules defined on the Ingress resource. An Ingress controller is responsible for fulfilling the Ingress, usually with a load balancer, though it may also configure your edge router or additional frontends to help handle the traffic.

**A ingress resource without a ingress controller has no effect.**
We need a Ingress CONTROLLER (i.e. NGNIX ingress controller) and a Ingress OBJECT (Routing Rules)
There are multiple Ingress [controller flavours](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)

Install NGNIX ingress CONTROLLER (helm or with manifests), check doc [page](https://docs.nginx.com/nginx-ingress-controller/installation/installing-nic/). Install Ingress controller (e.g. [ingress-nginx](https://kubernetes.github.io/ingress-nginx/deploy/#docker-desktop)) on local cluster

```bash
kubectl apply -f ingress_controller.yaml

# create ns and resources
namespace/ingress-nginx created
serviceaccount/ingress-nginx created
serviceaccount/ingress-nginx-admission created
role.rbac.authorization.k8s.io/ingress-nginx created
....
deployment.apps/ingress-nginx-controller created
ingressclass.networking.k8s.io/nginx created
validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission created
```

Ingress resource, get the ingressClassName `kubectl get ingressclasses.networking.k8s.io`

## Demo: ingress with ngnix

```bash
# create deployment
kubectl create deployment demo --image=httpd --port=80

# create cluster svc for deployment (default port 80)
kubectl expose deployment demo --name=demo-svc

# create ingress resource: imperative
kubectl create ingress demo-localhost --class=nginx  --rule="localhost/*=demo:80"

# create ingress resource: declarative
kubectl apply -f ingress_for_nginx.yaml

# test setup: forward a local port to the ingress controller
kubectl port-forward --namespace=ingress-nginx service/ingress-nginx-controller 8080:80

# override DNS resolution for demo.localdev.me
curl --resolve demo.localdev.me:8080:127.0.0.1 http://demo.localdev.me:8080

# cleanup
kubectl delete deploy demo
kubectl delete svc demo-svc
kubectl delete ingress demo-localhost
```

## Demo: ingress with 2 services 

```bash
# start apps on local machine
docker run -p 5555:8888 dejanualex/pythonhello:1.0
docker run -p 8888:8888 dejanualex/gohello:1.0

# create deployment for apps
kubectl create deployment pythonapp --image=dejanualex/pythonhello:1.0 --replicas=2
kubectl create deployment goapp --image=dejanualex/gohello:1.0 --replicas=2

# create services for apps
kubectl expose deployment pythonapp --name=pythonapp-svc --port=8081 --target-port=8888
kubectl expose deployment goapp --name=goapp-svc --port=8082 --target-port=8888

# check svc using naked pod with curl 
kubectl  run --rm curlopenssl -i --tty --image=dejanualex/curlopenssl:1.0  -- sh
curl pythonapp-svc.default.svc.cluster.local:8081
curl goapp-svc.default.svc.cluster.local:8082

# install ingress controller
kubectl apply -f ingress_for_apps.yaml

# test setup
# forward a local port to the ingress controller
kubectl port-forward --namespace=ingress-nginx service/ingress-nginx-controller 8080:80

# browser LOCALHOST
http://localhost:8080/
http://localhost:8080/python
http://localhost:8080/go

## one go
kubectl apply -f deploy_nginx.yaml

# cleanup
kubectl delete -f ingress_for_apps.yaml
kubectl delete deployments.apps pythonapp goapp
kubectl delete svc goapp-svc pythonapp-svc
```
