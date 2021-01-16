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

## About

This repository contains a series of YML files that I used to configure all artifacts and payloads that run on my DigitalOcean managed Kubernetes cluster. 

In this README, I explained how I set up the cluster and share the commands I most frequently use to manage it.

At the end of the README, I list references to useful DigitalOcean documentation.

## Install

To use the project in your development machine, clone it, and go to the project's root:

```sh
git clone https://github.com/wanderindev/do-managed-kubernetes.git
cd do-managed-kubernetes
```

### Create a cluster

Before you can use this repository contents, you must have a Kubernetes cluster running on DigitalOcean.  You can create it following these simple steps:

1. Login to your DigitalOcean account.
2. Click on **Create > Kubernetes** in the top right.
3. Select the region closest to your target audience.
4. In the **"Choose cluster capacity"** section:
   -  Enter a pool name or leave the default.
   -  Select the number of nodes. I've used 2 for almost two years without any issues but choose something that makes sense for your use case.  You can always add or delete nodes as needed later.
   -  Select the node plan.  I use the 2.5 GB RAM / 2 vCPUs
5. Modify the cluster's name or leave the default.
6. Click **Create Cluster**.  Your new cluster will be ready in about 4 minutes. You will get redirected to a **"Getting Started"** page.

### Install management tools

To manage your cluster, you will need ```kubectl``` installed in your terminal.  Visit [this link](https://kubernetes.io/docs/tasks/tools/install-kubectl/) for instructions on how to install it.

You will also need the Helm package manager for Kubernetes.  Visit [this link](https://helm.sh/docs/intro/install/) for installation instructions.

### Download the config file

Your cluster comes with a configuration file.  You need to download it and place it on the root of the project.  This file allows you to connect to the cluster and issue commands with ```kubectl```.

On the **"Getting Started"** page, go to step 3, and click on **download the cluster configuration file**.  Place the file in the root of the project and rename it ```config.yml```.

Make sure you keep this file out of version control.  Anyone with access to the file contents can connect to your cluster.  Before going any further, add ```config.yml``` to your ```.gitignore``` file.

Finally, you have to make sure there's a KUBECONFIG environment variable pointing to the ```config.yml``` file.  If you use Ubuntu, like me, you can accomplish this by opening the ```.profile``` file:

```sh
sudo vi ~/.profile
```

And pasting at the end:

```sh
# set environment variable with path to kubectl config file
export KUBECONFIG="/mnt/c/Users/jfeli/version-control/do-managed-kubernetes/config.yml"
```
making sure you replace the value with the path to your ```config.yml``` file.

Try connecting to the cluster to make sure everything is all right:

```sh
kubectl get nodes
```

```sh
Output

NAME                 STATUS   ROLES    AGE   VERSION
workers-pool-31ety   Ready    <none>   14h   v1.19.3
workers-pool-2a7xn   Ready    <none>   14h   v1.19.3
```

## Set up an Ingress Controller

A Kubernetes Ingress exposes HTTP / HTTPS routes from outside the cluster to services within the cluster.  The rules defined in the Ingress resource control the traffic routing.

The file ```ingress-controller/ingress.yml``` in this repository defines our Ingress resource.  If you look at the first rule, you see this:

```sh
  rules:
  - host: anafeliu.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: anafeliu-web
            port:
              number: 80
```

Which says: "Route all traffic arriving at anafeliu.com to the port 80 of service anafeliu-web." You can have as many rules as necessary in your Ingress resource.

The easiest way to set up an Ingress Controller for your cluster is to use the one-click-app-install available in the DigitalOcean marketplace.  Installing it will add a Pod running Nginx to your nodes that will route traffic according to the rules specified in our Ingress resource.

Note that adding the Ingress will also create a load balancer, which will cost $10 per month.

To add an Ingress controller, login to your DigitalOcean account and follow these steps:

1. Visit the [NGINX Ingress Controller app](https://cloud.digitalocean.com/marketplace/5dcc8254e2339e33d74decd7?i=eedacf).  
2. Click Install App
3. Select your cluster.

You will be redirected to your cluster's overview page.  The installation will take a few minutes.  Once ready, check to make sure the new pod it is running:

```sh
kubectl get pods --all-namespaces -l app.kubernetes.io/name=ingress-nginx
```

```sh
Output

NAMESPACE       NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx   ingress-nginx-admission-create-chk64        0/1     Completed   0          2m47s
ingress-nginx   ingress-nginx-admission-patch-trjmk         0/1     Completed   0          2m47s
ingress-nginx   ingress-nginx-controller-56c75d774d-9nsd8   1/1     Running     0          2m49s
```

And you can check that the new load balancer is ready and what is its IP by running:

```sh
kubectl get svc -n ingress-nginx
```

```sh
Output

NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP       PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.245.84.127    175.108.120.222   80:31646/TCP,443:31596/TCP   6m
ingress-nginx-controller-admission   ClusterIP      10.245.146.170   <none>            443/TCP                      6m
```

Note the external IP since you will need it later.

We need to add some annotations to our load balancer to allow Pod to Pod communication within the DigitalOcean Kubernetes cluster.  The file ```ingress-controller/load_balancer.yml```
has been already annotated with a reference to a hostname, ```lb.feliu.io```:

```sh
metadata:
  annotations:
    service.beta.kubernetes.io/do-loadbalancer-enable-proxy-protocol: 'true'
    service.beta.kubernetes.io/do-loadbalancer-hostname: "lb.feliu.io"
```

Where ```feliu.io``` is a domain I already own and that its is managed by DigitalOcean.  Replace with a domain of your own.

Apply the changes to the load balancer by running:

```sh
kubectl apply -f ingress-controller/load_balancer.yml
```

```sh
Output

service/ingress-nginx-controller configured
```

Finally, go to the **Networking** section in your DigitalOcean dashboard and add an A record for the subdomain.domain you used for the annotation (in my case ```lb.feliu.io```) pointing to the load balancer's external IP.

## Installing Cert-Manager

Cert-manager is a Kubernetes add-on that provisions TLS certificates from a certificate authority like Let's Encrypt.  Adding cert-manager to your cluster provides worry-free management of the TLS certificates' lifecycles for all your hosts.

The first thing you need to do is install cert-manager by running:

```sh
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.1.0/cert-manager.yaml
```

```sh
Output

customresourcedefinition.apiextensions.k8s.io/certificaterequests.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/certificates.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/challenges.acme.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/clusterissuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/issuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/orders.acme.cert-manager.io created
namespace/cert-manager created
serviceaccount/cert-manager-cainjector created
serviceaccount/cert-manager created
serviceaccount/cert-manager-webhook created
clusterrole.rbac.authorization.k8s.io/cert-manager-cainjector created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-issuers created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-clusterissuers created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-certificates created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-orders created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-challenges created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-ingress-shim created
clusterrole.rbac.authorization.k8s.io/cert-manager-view created
clusterrole.rbac.authorization.k8s.io/cert-manager-edit created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-cainjector created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-issuers created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-clusterissuers created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-certificates created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-orders created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-challenges created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-ingress-shim created
role.rbac.authorization.k8s.io/cert-manager-cainjector:leaderelection created
role.rbac.authorization.k8s.io/cert-manager:leaderelection created
role.rbac.authorization.k8s.io/cert-manager-webhook:dynamic-serving created
rolebinding.rbac.authorization.k8s.io/cert-manager-cainjector:leaderelection created
rolebinding.rbac.authorization.k8s.io/cert-manager:leaderelection created
rolebinding.rbac.authorization.k8s.io/cert-manager-webhook:dynamic-serving created
service/cert-manager created
service/cert-manager-webhook created
deployment.apps/cert-manager-cainjector created
deployment.apps/cert-manager created
deployment.apps/cert-manager-webhook created
mutatingwebhookconfiguration.admissionregistration.k8s.io/cert-manager-webhook created
validatingwebhookconfiguration.admissionregistration.k8s.io/cert-manager-webhook created
```

Once installed, check that the cert-manager pods are running:

```sh
kubectl get pods --namespace cert-manager
```

```sh
Output

NAME                                      READY   STATUS    RESTARTS    AGE
cert-manager-cainjector-fc6c5a17db-wbtt6   1/1     Running   0          2m10s
cert-manager-d9937d4d7-xxv2z               1/1     Running   0          2m9s
cert-manager-webhook-845d1578bf-nqqfw      1/1     Running   0          2m9s
```

### Creating a certificate issuer

The next step is to add to the cluster a certificate Issuer.  The Issuer defines the authority which will sign your certificates.   We will use Let's Encrypt.

The file ```ingress-controller/issuer.yml``` contains our Issuer.  It looks like this:

```sh
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  namespace: cert-manager
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: javier@wanderin.dev
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class: nginx
```

In it, we define a ClusterIssuer named ```letsencrypt-prod``` which will use the Let's Encrypt server to generate certificates registered to my email.  The acount's private key will be store in the Kubernetes secret ```letsencrypt-prod```.

You can add it to the cluster by running:

```sh
kubectl create -f ./ingress-controller/issuer.yml
```

```sh
Output

clusterissuer.cert-manager.io/letsencrypt-prod created
```

### Adding a ```tls``` section to our Ingress

Cert-manager knows which certificates it needs to get by looking at the ```tls``` section in the Ingress resource, which, in our case, looks like this:

```sh
spec:
  tls:
  - hosts:
    - anafeliu.com
    - www.anafeliu.com
    - felaro.org
    - www.felaro.org
    - feliu.io
    - www.feliu.io
    - calcfina.com
    - www.calcfina.com
    - api.calcfina.com
    - uslpanama.com
    - www.uslpanama.com
    - wallcouture.com.pa
    - www.wallcouture.com.pa

    secretName: letsencrypt-prod
```

Cert-manager will take care of managing the certificates for all the hosts listed there.  The ```secretName``` will hold the TLS private key and issued certificates.

### Pointing your hosts to the load balancer

Before you apply the Ingress resource to your cluster, you need to modify the A records for all your hosts so they point to the load balancer's IP.  You can accomplish this from the **Networking** section in your **DigitalOcean dashboard**.

### Adding Pods and Services for your hosts

We also need to add to the cluster the Pods that contain your hosts' actual software and the associated Services.  The ```python``` and ```sites``` folders in this repository contain many yml files with Deployments and related Services.  

A Deployment defines how to create a Pod with a specific workload and how many Pod replicas should be running in the cluster.

You can add the Pods and Services for the hosts by running:

```sh
kubectl apply -f ./sites/anafeliu-web.yml
kubectl apply -f ./sites/felaro-web.yml
kubectl apply -f ./sites/feliuio-web.yml
kubectl apply -f ./sites/calcfina-web.yml
kubectl apply -f ./python/api-calcfina.yml
kubectl apply -f ./sites/usl-web.yml
kubectl apply -f ./sites/wallcouture-web.yml
```

```sh
Output

service/anafeliu-web created
deployment.apps/anafeliu-web created
service/felaro-web created
deployment.apps/felaro-web created
service/feliuio-web created
deployment.apps/feliuio-web created
service/calcfina-web created
deployment.apps/calcfina-web created
service/api-calcfina created
deployment.apps/api-calcfina created
service/usl-web created
deployment.apps/usl-web created
service/wallcouture-web created
deployment.apps/wallcouture-web created
```

You can check that the Pods and Services were created, like so:

```sh
kubectl get pods
```

```sh
Output

NAME                              READY   STATUS    RESTARTS   AGE
anafeliu-web-7ddb7f75bb-6xh2k     1/1     Running   0          76s
api-calcfina-68dd859fcb-4v5mz     1/1     Running   0          43s
api-calcfina-68dd859fcb-xcrmt     1/1     Running   0          43s
calcfina-web-55f6679c8b-hngkb     1/1     Running   0          51s
calcfina-web-55f6679c8b-rrjhq     1/1     Running   0          51s
felaro-web-5bdf476664-kphrq       1/1     Running   0          66s
feliuio-web-695b4dcbf8-lcn4s      1/1     Running   0          58s
usl-web-77b979f75f-qbtmb          1/1     Running   0          35s
wallcouture-web-fff96dcfb-88q59   1/1     Running   0          27s
```

```sh
kubectl get svc
```

```sh
Output

NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
anafeliu-web      ClusterIP   10.245.11.234    <none>        80/TCP    86s
api-calcfina      ClusterIP   10.245.80.179    <none>        80/TCP    53s
calcfina-web      ClusterIP   10.245.15.196    <none>        80/TCP    61s
felaro-web        ClusterIP   10.245.125.206   <none>        80/TCP    76s
feliuio-web       ClusterIP   10.245.195.59    <none>        80/TCP    68s
kubernetes        ClusterIP   10.245.0.1       <none>        443/TCP   28m
usl-web           ClusterIP   10.245.174.105   <none>        80/TCP    45s
wallcouture-web   ClusterIP   10.245.224.25    <none>        80/TCP    37s
```

### Add the Ingress resource

Finally, we can add the Ingress resource by running:

```sh
kubectl apply -f ./ingress-controller/ingress.yml
```

```sh
Output

ingress.networking.k8s.io/wid-ingress created
```

Cert-manager will request the TLS certificates for all the hosts listed in the Ingress resource.  Also, all external traffic received for the hosts will be routed to the appropriate Service as defined in the Ingress resource.

Wait a few minutes for the certificates to get issued.  Check progress with:

```sh
kubectl describe certificate letsencrypt-prod
```

```sh
output

Name:         letsencrypt-prod
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Certificate
Metadata:
  Creation Timestamp:  2021-01-16T00:46:38Z
  Generation:          1
  Managed Fields:
    API Version:  cert-manager.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:ownerReferences:
          .:
          k:{"uid":"2639c261-e7d2-47ef-afa5-7f6ffbf8ca9b"}:
            .:
            f:apiVersion:
            f:blockOwnerDeletion:
            f:controller:
            f:kind:
            f:name:
            f:uid:
      f:spec:
        .:
        f:dnsNames:
        f:issuerRef:
          .:
          f:group:
          f:kind:
          f:name:
        f:privateKey:
        f:secretName:
      f:status:
        .:
        f:conditions:
        f:notAfter:
        f:notBefore:
        f:renewalTime:
        f:revision:
    Manager:    controller
    Operation:  Update
    Time:       2021-01-16T01:44:30Z
  Owner References:
    API Version:           extensions/v1beta1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  Ingress
    Name:                  wid-ingress
    UID:                   2639c261-e7d2-47ef-afa5-7f6ffbf8ca9b
  Resource Version:        17717
  Self Link:               /apis/cert-manager.io/v1/namespaces/default/certificates/letsencrypt-prod
  UID:                     b02bcad7-22ca-4238-854d-8b95b16344e3
Spec:
  Dns Names:
    anafeliu.com
    www.anafeliu.com
    felaro.org
    www.felaro.org
    feliu.io
    www.feliu.io
    calcfina.com
    www.calcfina.com
    api.calcfina.com
    uslpanama.com
    www.uslpanama.com
    wallcouture.com.pa
    www.wallcouture.com.pa
  Issuer Ref:
    Group:      cert-manager.io
    Kind:       ClusterIssuer
    Name:       letsencrypt-prod
  Secret Name:  letsencrypt-prod
Status:
  Conditions:
    Last Transition Time:  2021-01-16T01:44:30Z
    Message:               Certificate is up to date and has not expired
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2021-04-16T00:44:30Z
  Not Before:              2021-01-16T00:44:30Z
  Renewal Time:            2021-03-17T00:44:30Z
  Revision:                1
Events:
  Type    Reason   Age    From          Message
  ----    ------   ----   ----          -------
  Normal  Issuing  3m52s  cert-manager  The certificate has been successfully issued
```



## References

- [DigitalOcean Kubernetes screencast]()
- [Documentation: ```kubeclt```](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [Documentation: cert-manager](https://cert-manager.io/docs/concepts/)
- [Repository: cert-manager](https://github.com/jetstack/cert-manager)  
- [Tutorial: How to Set Up an Nginx Ingress with Cert-Manager on DigitalOcean Kubernetes](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nginx-ingress-with-cert-manager-on-digitalocean-kubernetes)
- [Tutorial: How to Create, Edit, and Delete DNS Records](https://www.digitalocean.com/docs/networking/dns/how-to/manage-records/)
- [Documentation: Kubernetes Managing Resources Guide](https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment)

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
 