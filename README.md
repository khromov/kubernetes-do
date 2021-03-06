## Kubernetes cluster with Nginx Ingress and Cert-Manager for Let's Encrypt SSL on DigitalOcean

#### Tl;dr

Assumes you have a working cluster and `kubectl` configured.

Edit `hello-kubernetes-ingress.yaml` and replace `domain1.example.org` and `domain2.example.org` with your production domains.

```
kubectl apply -f hello-kubernetes-first.yaml
kubectl apply -f hello-kubernetes-second.yaml
kubectl apply -f production_issuer.yaml
kubectl apply -f hello-kubernetes-ingress.yaml
```

After some time, you should be able to visit your two domains and see a "Hello world" message.

Base tutorial:

https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nginx-ingress-on-digitalocean-kubernetes-using-helm

#### How to

Install prerequisites, follow the tutorial and deploy your two services:

```
kubectl create -f hello-kubernetes-first.yaml
kubectl create -f hello-kubernetes-second.yaml
```

Verify services:

```bash
$ kubectl get service hello-kubernetes-first
NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
hello-kubernetes-first   ClusterIP   10.245.216.61   <none>        80/TCP    16m

$ kubectl get service hello-kubernetes-second
NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
hello-kubernetes-second   ClusterIP   10.245.219.48   <none>        80/TCP    15h
```

```bash
$ kubectl get service
NAME                            TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                      AGE
hello-kubernetes-first          ClusterIP      10.245.216.61   <none>           80/TCP                       16h
hello-kubernetes-second         ClusterIP      10.245.219.48   <none>           80/TCP                       15h
# ...
```

**Install Helm 3**

[Installing Helm](https://helm.sh/docs/intro/install/)

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

Add required charts:

```bash
helm repo add stable https://charts.helm.sh/stable
```

**Install load balancer**

```bash
helm install nginx-ingress stable/nginx-ingress --set controller.publishService.enabled=true
```

⚠ Note - The ingress controller in the tutorial appears to be deprecated, but still works.

```bash
*******************************************************************************************************
* DEPRECATED, please use https://github.com/kubernetes/ingress-nginx/tree/master/charts/ingress-nginx *
*******************************************************************************************************
```

Check when service becomes available (`EXTERNAL-IP` should no longer be `<pending>`)

```bash
kubectl get services -o wide -w nginx-ingress-controller
NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE   SELECTOR
nginx-ingress-controller   LoadBalancer   10.245.236.88   <pending>     80:31319/TCP,443:32284/TCP   70s   app.kubernetes.io/component=controller,app=nginx-ingress,release=nginx-ingress
```

Find load balancers in DO control panel if you want to inspect it.

https://cloud.digitalocean.com/networking/load_balancers


Point your two domains/subdomains in DNS to the load balancer IP and add it to `hello-kubernetes-ingress.yaml`.

Deploying Nginx ingress configuration 

```bash
kubectl create -f hello-kubernetes-ingress.yaml
```

Now your two domains listed in `hello-kubernetes-ingress.yaml > spec > host` should work without SSL.

## Set up Cert-Manager

The tutorial version uses a very old version of Cert-Manager, let's install a new one instead: [https://cert-manager.io/docs/installation/kubernetes/](https://cert-manager.io/docs/installation/kubernetes/).

```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.2.0/cert-manager.yaml
```

Set up the `ClusterIssuer` resource:

```
kubectl create -f production_issuer.yaml
```

Set up the `Ingress` configuration:

```bash
kubectl apply -f hello-kubernetes-ingress.yaml
```

```bash
kubectl describe certificate hello-kubernetes-tls

# ...
Type    Reason     Age   From          Message
  ----    ------     ----  ----          -------
  Normal  Issuing    49s   cert-manager  Issuing certificate as Secret does not exist
  Normal  Generated  47s   cert-manager  Stored new private key in temporary Secret resource "hello-kubernetes-tls-fqd6z"
  Normal  Requested  47s   cert-manager  Created new CertificateRequest resource "hello-kubernetes-tls-qcq8r"
  Normal  Issuing    22s   cert-manager  The certificate has been successfully issued
```

**SSL now works when you visit your domains! Congratulations!**

### Misc

#### Display running pods

Current namespace:

```
kubectl get pods
```

All namespaces:

```
kubectl get pods --all-namespaces
```

#### Edit a deployment in-place without applying a new yaml file

```
kubectl edit deployment/hello-kubernetes-first
```

Useful to quickly scale a service up or down.

#### Accessing cluster services on your local computer using port forwarding 

A great way to access the services inside your cluster

https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/

Exposing our "Hello world" service:

```bash
kubectl port-forward svc/hello-kubernetes-first 8080:80 -n default
```

[http://localhost:8080/](http://localhost:8080/)

#### Monitoring stack setup

How to set up and access the monitoring stack

[https://marketplace.digitalocean.com/apps/kubernetes-monitoring-stack?iframe=true](https://marketplace.digitalocean.com/apps/kubernetes-monitoring-stack?iframe=true)

How to delete the monitoring stack on existing cluster

[https://github.com/digitalocean/marketplace-kubernetes/issues/94](https://github.com/digitalocean/marketplace-kubernetes/issues/94)

Get additional metrics in DO control panel

[https://www.digitalocean.com/docs/kubernetes/how-to/monitor-advanced/](https://www.digitalocean.com/docs/kubernetes/how-to/monitor-advanced/)