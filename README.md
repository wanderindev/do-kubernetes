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
two node pool. 

One pool will be for the database payloads and will have one node.

The other pool will be for the workers and will have two nodes.

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

### Add labels to the nodes
List the nodes in the cluster:
```sh
kubectl get nodes
```
The output will be similar to this:
```sh
NAME                    STATUS   ROLES    AGE   VERSION
wid-db-jatk             Ready    <none>   35d   v1.14.1
wid-workers-2cpu-xzb4   Ready    <none>   14d   v1.14.1
wid-workers-2cpu-xmri   Ready    <none>   14d   v1.14.1
```
Add labels:
```sh
kubectl label nodes wid-db-jatk type=db
kubectl label nodes wid-workers-2cpu-xzb4 type=worker
kubectl label nodes wid-workers-2cpu-xmri type=worker
```
###Deploying pods and services
Deploy all pods and associated services:
```sh
kubectl apply -f ./sites
```
Or deploy individual pods and services, for instance:
```sh
kubectl apply -f ./sites/anafeliu-web.yml
```
###Deploy an Nginx ingress controller and cert-manager
Create mandatory resources and load balancer:
```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.24.1/deploy/mandatory.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.24.1/deploy/provider/cloud-generic.yaml
```
Create a cert-manager custom resource definition and add label to the kube-system namespace: 
```sh
kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.8/deploy/manifests/00-crds.yaml
kubectl label namespace kube-system certmanager.k8s.io/disable-validation="true"
```
Install Tiller in the cluster
```sh
kubectl -n kube-system create serviceaccount tiller
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account tiller
```
Install cert-manager:
```sh
helm install --name cert-manager --namespace kube-system jetstack/cert-manager --version v0.8.0
```
Create a certificate issuer:
```sh
kubectl create -f ./ingress-controller/issuer.yml
```
Create the ingress resource:
```sh
kubectl apply -f ./ingress-controller/ingress.yml
```
Refer to [this source](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nginx-ingress-with-cert-manager-on-digitalocean-kubernetes)
for more information.
###Deploy PostgreSQL
Modify, if needed, the `./postgresql/values-sample.yml` and install the PostgreSQL chart:
```sh
helm install --name wid-pg -f ./postgresql/values-sample.yml stable/postgresql
```
Get the auto generated PostgreSQL password:
```sh
kubectl get secret --namespace default wid-pg-postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode
```
###Deploy MySQL
Modify, if needed, the `./mysql/values-sample.yml` and install the MySQL chart:
```sh
helm install --name wid-mysql -f ./values-sample.yml stable/mysql
```
Get the auto generated MySQL password:
```sh
kubectl get secret --namespace default wid-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo
```
## Running tests
From PyCharm, right-click in the `tests/system` or `tests/unit` directory and select 
"Run tests with Coverage" to run either system or unit tests.

## Testing with Postman

Clone the [postman-hr-rest-api](https://github.com/wanderindev/postman-hr-rest-api)
repository:
```sh
git clone https://github.com/wanderindev/postman-hr-rest-api.git
``` 
Open Postman and import `hr-rest-api.json` and `hr-rest-api-environment.json`.

Click on Runner.

In the window that pops open, select the `hr-rest-api` collection and the `hr-rest-api`
environment.

Click on Run.

## Deployment
Modify `rest/Dockerfile_prod`, adding the correct values for the environment variables.

Build the container image and push to Docker Hub:
```sh
cd rest
docker build -t wanderindev/hr-rest -f Dockerfile_prod .
docker push wanderindev/hr-rest
``` 
 
 Go to the [do-managed-kubernetes](https://github.com/wanderindev/do-managed-kubernetes) 
 repository and re-deploy the pod.
 ```sh
kubectl delete deployment hr-rest
kubectl apply -f ./sites/hr-rest.yml
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
 