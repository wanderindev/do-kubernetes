<h1 align="center">Cluster Setup for DigitalOcean Managed Kubernetes ⚓️</h1>
<p>
  <img src="https://img.shields.io/badge/version-1.0-blue.svg?cacheSeconds=2592000" />
  <a href="https://github.com/wanderindev/do-managed-kubernetes/blob/master/README.md">
    <img alt="Documentation" src="https://img.shields.io/badge/documentation-yes-brightgreen.svg" target="_blank" />
  </a>
  <a href="https://github.com/wanderindev/do-managed-kubernetes/graphs/commit-activity">
    <img alt="Maintenance" src="https://img.shields.io/badge/Maintained%3F-yes-brightgreen.svg" target="_blank" />
  </a> 
  <a href="https://github.com/wanderindev/do-managed-kubernetes/blob/master/LICENSE.md">
    <img alt="License: MIT" src="https://img.shields.io/badge/License-MIT-yellow.svg" target="_blank" />
  </a>
  <a href="https://twitter.com/JavierFeliuA">
    <img alt="Twitter: JavierFeliuA" src="https://img.shields.io/twitter/follow/JavierFeliuA.svg?style=social" target="_blank" />
  </a>
</p>

>Collection of yaml files for configuring our Kubernetes cluster.

## How to use
### Clone the repository
```sh
git clone https://github.com/wanderindev/do-managed-kubernetes.git
cd do-managed-kubernetes
```
### Create the cluster and add nodes
Fom the DigitalOcean dashboard, go to Kubernetes and create a cluster with 
two node pools. 

One pool will be for the database payloads and will have one node with specs: standard / 2 vcpu / 4GB.

The other pool will be for the workers and will have two nodes with specs: standard / 1 vcpu / 2GB.

Download the cluster configuration file and place it in the repository root.  Rename
it config.yml.

Make sure the KUBECONFIG environment variable points to the config.yml file.

Open the .profile file:
```sh
sudo vi ~/.profile
```
And paste at the end:
```sh
# set environment variable with path to kubectl config file
export KUBECONFIG="/mnt/c/Users/jfeli/version-control/do-managed-kubernetes/config.yml"
```

### Deploying pods and services
Deploy all pods and associated services:
```sh
kubectl apply -f ./sites
```
Or deploy individual pods and services, for instance:
```sh
kubectl apply -f ./sites/anafeliu-web.yml
```
### Deploy an Nginx ingress controller and cert-manager
Install Tiller in the cluster:
```sh
kubectl -n kube-system create serviceaccount tiller
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account tiller
```
Install the Nginx Ingress Controller, setting the `controller.publishService.enabled` parameter to `true`:
```sh
helm install stable/nginx-ingress --name nginx-ingress --set controller.publishService.enabled=true
```
A load balancer will be created.  Check if its available with:
```sh
kubectl get services -o wide -w nginx-ingress-controller
```
Create a cert-manager custom resource definition and add label to the kube-system namespace: 
```sh
kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.8/deploy/manifests/00-crds.yaml
kubectl label namespace kube-system certmanager.k8s.io/disable-validation="true"
```
Install cert-manager:
```sh
helm repo add jetstack https://charts.jetstack.io
helm install --name cert-manager --namespace cert-manager jetstack/cert-manager
```
Create a certificate issuer:
```sh
kubectl create -f ./ingress-controller/issuer.yml
```
Create the ingress resource:
```sh
kubectl apply -f ./ingress-controller/ingress.yml
```
Wait a few minutes for the certificates to get issued.  Check progress with:
```sh
kubectl describe certificate letsencrypt-prod
```
Refer to [this source](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nginx-ingress-on-digitalocean-kubernetes-using-helm)
for more information.
### Setup ExternalDNS 
In the DigitalOcean dashboard, create an Api token and add it to `./ingress-controller/externaldns-values.yml`
Install ExternalDNS:
```sh
helm install stable/external-dns --name external-dns -f ./ingress-controller/externaldns-values.yml
```
Verify that ExternalDNS is ready:
```sh
kubectl --namespace=default get pods -l "app=external-dns,release=external-dns" -w
```
Refer to [this source](https://www.digitalocean.com/community/tutorials/how-to-automatically-manage-dns-records-from-digitalocean-kubernetes-using-externaldns)
for more information.
### Setup cluster monitoring
Install the `prometheus-operator` chart:
```sh
helm install --namespace monitoring --name doks-cluster-monitoring -f ./monitoring/values.yml stable/prometheus-operator
```
Check that all pods (6 in total) are ready:
```sh
kubectl --namespace monitoring get pods -l "release=doks-cluster-monitoring"

NAME                                                          READY   STATUS    RESTARTS   AGE
doks-cluster-monitoring-grafana-7b775d756-m8rxj               2/2     Running   0          3m9s
doks-cluster-monitoring-kube-state-metrics-85456956d7-st49f   1/1     Running   0          3m9s
doks-cluster-monitoring-pr-operator-6685fdb84-4xk4m           1/1     Running   0          3m9s
doks-cluster-monitoring-prometheus-node-exporter-tw7d7        1/1     Running   0          3m9s
doks-cluster-monitoring-prometheus-node-exporter-v7m4q        1/1     Running   0          3m9s
```
Create a port-forwarding tunnel for the Grafana service:
```sh
kubectl port-forward -n monitoring svc/doks-cluster-monitoring-grafana 8000:80

Forwarding from 127.0.0.1:8000 -> 3000
Forwarding from [::1]:8000 -> 3000
```
And access it at `http://localhost:8000`. The username is `admin` and the password is in line 16 of `monitoring/values.yml`.  When done with Grafana, close the tunnel with `CTRL-C`.

Create a port-forwarding tunnel for the Prometheus service:
```sh
kubectl port-forward -n monitoring svc/doks-cluster-monitoring-pr-prometheus 9090:9090

Forwarding from 127.0.0.1:9090 -> 9090
Forwarding from [::1]:9090 -> 9090
```
And access it at `http://localhost:9090`.

Create a port-forwarding tunnel for the Alertmanager service:
```sh
kubectl port-forward -n monitoring svc/doks-cluster-monitoring-pr-alertmanager 9093:9093

Forwarding from 127.0.0.1:9093 -> 9093
Forwarding from [::1]:9093 -> 9093
```
And access it at `http://localhost:9093`.

Refer to [this source](https://www.digitalocean.com/community/tutorials/how-to-set-up-digitalocean-kubernetes-cluster-monitoring-with-helm-and-prometheus-operator)
for more information.
### Deploy PostgreSQL
Modify, if needed, the `./postgresql/values-sample.yml` and install the PostgreSQL chart:
```sh
helm install --name wid-pg -f ./postgresql/values-sample.yml stable/postgresql
```
Get the auto generated PostgreSQL password:
```sh
kubectl get secret --namespace default wid-pg-postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode
```
### Deploy MySQL
Modify, if needed, the `./mysql/values-sample.yml` and install the MySQL chart:
```sh
helm install --name wid-mysql -f ./values-sample.yml stable/mysql
```
Get the auto generated MySQL password:
```sh
kubectl get secret --namespace default wid-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo
```

 ## Author

👤 **Javier Feliu**

* Twitter: [@JavierFeliuA](https://twitter.com/JavierFeliuA)
* Github: [@wanderindev](https://github.com/wanderindev)

## Show your support

Give a ⭐️ if this project helped you!

## 📝 License

Copyright © 2019 [Javier Feliu](https://github.com/wanderindev).<br />

This project is [MIT](https://github.com/wanderindev/hr-rest-api/blob/master/LICENSE.md) licensed.

***
_I based this README on a template generated with ❤️ by [readme-md-generator](https://github.com/kefranabg/readme-md-generator)_
 