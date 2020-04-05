# Fn Project Helm Chart

The [Fn project](http://fnproject.io) is an open source, container native, and cloud agnostic serverless platform. It’s easy to use, supports every programming language, and is extensible and performant.

## Introduction

This chart deploys a fully functioning instance of the [Fn](https://github.com/fnproject/fn) platform on a Kubernetes cluster using the [Helm](https://helm.sh/) package manager.

## Prerequisites

- persistent volume provisioning support in the underlying infrastructure (for persistent data, see below )

- Install [Helm](https://github.com/kubernetes/helm#install)

- Initialize Helm by installing Tiller, the server portion of Helm, to your Kubernetes cluster

- [Ingress controller](https://github.com/helm/charts/tree/master/stable/nginx-ingress)

- [Cert manager](https://medium.com/oracledevs/secure-your-kubernetes-services-using-cert-manager-nginx-ingress-and-lets-encrypt-888c8b996260)


```bash
helm init --upgrade
```

## Preparing chart values

### Minimum configuration

In order to get a working deployment please pay attention to what you have in your chart values.
[Here](fn/values.yaml) is the bare minimum chart configuration to deploy a working Fn cluster.

### Exposing Fn services

#### Ingress controller

If you are installing Fn behind an ingress controller, you'll need to have a single DNS sub-domain that will act as your ingress controllers IP resolution.

Important: An ingress controller works as a proxy, so you can use the ingress IP address as an HTTP proxy:

```bash
curl -x http://<ingress-controller-endpoint>:80 api.fn.internal 
{"goto":"https://github.com/fnproject/fn","hello":"world!"}
```


#### LoadBalancer

In order to natively expose the Fn services, you'll need to modify the Fn API, Runner, and UI service definitions:

 - at `fn_api` node values, modify `fn_api.service.type` from `ClusterIP` to `LoadBalancer`
 - at `fn_lb_runner` node values, modify `fn_lb_runner.service.type` from `ClusterIP` to `LoadBalancer`
 - at `ui` node values, modify `ui.service.type` from `ClusterIP` to `LoadBalancer`


#### DNS names

In an Fn deployment with LoadBalancer service types, you'll need 3 DNS names:

 - one for an API service (i.e., `api.fn.mydomain.com`)
 - one for runner LB service (i.e., `lb.fn.mydomain.com`)
 - one for UI service (i.e., `ui.fn.mydomain.com`)

Upon successful deployment, you'll have three public IP addresses -- one for each service.
However, the IP address for the API and LB services will be identical since they are exposed as a single service.
You'll have two IP addresses, but three DNS names.

Please keep in mind the best way for exposing services is an **ingress controller**.

## Installing the Chart

Clone the fn-helm repo:

```bash
git clone https://github.com/fnproject/fn-helm.git && cd fn-helm
```

Install chart dependencies from [requirements.yaml](./fn/requirements.yaml):

```bash
helm dep build fn
```

The default chart will install fn as a private service inside your cluster with ephemeral storage, to configure a public endpoint and persistent storage you should look at [values.yaml](fn/values.yaml) and modify the default settings.
To install the chart with the release name `my-release`:

```bash
helm install --name my-release fn
```

> Note: if you do not pass the --name flag, a release name will be auto-generated. You can view releases by running helm list (or helm ls, for short).

## Working with Fn 

#### Ingress controller

Please ensure that your ingress controller is running and has a public-facing IP address.
An ingress controller acts as a proxy between your internal and public networks.
Therefore in order to talk to your Fn Deployment, you'll need to set the `HTTP_PROXY` environment variable or use cURL like so:

```bash
curl -x http://<ingress-controller-endpoint>:80 api.fn.internal
{"goto":"https://github.com/fnproject/fn","hello":"world!"}
```

## Uninstalling the Fn Helm Chart

Assuming your release is named `my-release`:

```bash
helm delete --purge my-release
```

The command removes all the Kubernetes components associated with the chart and deletes the release.

## Configuration 

For detailed configuration, please see [default chart values](fn/values.yaml).

 ## Configuring Database Persistence 
 
Fn persists application data in MySQL. This is configured using the MySQL Helm Chart.

By default this uses container storage. To configure a persistent volume, set `mysql.*` values in the chart values to that which corresponds to your storage requirements.

e.g. to use an existing persistent volume claim for MySQL storage:

```bash 
helm install --name testfn --set mysql.persistence.enabled=true,mysql.persistence.existingClaim=tc-fn-mysql fn
```

```
helm install fnserver --namespace public-service fn
```

```
NAME: fnserver
LAST DEPLOYED: Sun Apr  5 18:56:27 2020
NAMESPACE: public-service
STATUS: deployed
REVISION: 6
NOTES:
Your K8s cluster MUST have configured with NGINX ingress controller.
See https://github.com/helm/charts/tree/master/stable/nginx-ingress.

For SSL/TLS support please take a look at:
https://medium.com/oracledevs/secure-your-kubernetes-services-using-cert-manager-nginx-ingress-and-lets-encrypt-888c8b996260

The Fn service can be accessed within your cluster at:

 - http://fnserver.api.fn.k8scloud.site:32001

Then you need to update your environment variable `HTTP_HOST` with the corresponding ingress controller endpoint or an IP address:

    export {HTTP,HTTPS}_PROXY=$(kubectl get svc <ingress-nginx-ingress-controller> -o jsonpath='http://{.status.loadBalancer.ingress[0].ip}:{.spec.ports[0].port}')

For instance:

    kubectl get svc | grep ingress

    ingress-nginx-ingress-controller        10.96.224.0     XXX.XXX.XXX.XXX   80:31698/TCP,443:30947/TCP   105d
    ingress-nginx-ingress-default-backend   10.96.140.84    <none>            80/TCP                       105d

So, you'd need `ingress-nginx-ingress-controller` service. The command from above will evaluate an HTTP endpoint:

    kubectl get svc ingress-nginx-ingress-controller -o jsonpath='http://{.status.loadBalancer.ingress[0].ip}:{.spec.ports[0].port}'

    http://XXX.XXX.XXX.XXX:80

or HTTPS endpoint:

    kubectl get svc ingress-nginx-ingress-controller -o jsonpath='https://{.status.loadBalancer.ingress[0].ip}:{.spec.ports[1].port}'

    http://XXX.XXX.XXX.XXX:443


Then you need to create Fn CLI context:

    fn create context fnserver-fn --api-url http://fnserver.api.fn.k8scloud.site:32001 --provider default --registry <your-docker-registry>
    fn use context fnserver-fn

Next thing would be to test the connectivity between your host and the deployment using simple API query:

    fn create app first-app
    fn list apps

And the last thing would be to check whether you can call a functions:

    fn init --runtime go --trigger http first-fn
    cd first-fn
    fn --verbose deploy --app first-app
    fn inspect fn first-app first-fn

You need to check an `invokeEndpoint` annotation, it suppose to point to http://fnserver.lb.fn.k8scloud.site:32002.
So, you are ready to go:

    echo -e '{"name":"john"}' | fn invoke first-app first-fn
    curl http://fnserver.lb.fn.k8scloud.site:32002/t/first-app/first-fn
```
