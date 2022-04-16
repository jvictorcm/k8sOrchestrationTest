# Getting Started

### K8s Orchestration Test

The purpose of this test is to be testing to work inside a k8s env.

```console
minikube start
minikube addons enable metrics-server
minikube dashboard
```

Make notes that this will open a browser page with the cluster management.

`Opening http://127.0.0.1:38177/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/ in your default browser...
`

Let's create a deployment of apache httpd:

```console
kubectl create deployment apache-httpd --image=httpd
kubectl expose deployment apache-httpd --type=LoadBalancer --port=80
```

We can check if the proxy is working with

`http://127.0.0.1:38177/api/v1/namespaces/default/services/apache-httpd/proxy/`

### Food Service

#### Creating image

We will be using mockserver for this.

```console
cd food-app/image
docker build -t food-app:1 .
```

Now, we need to make this image available inside minikube. They use different registry. Let's push into minikube cache
registry.

```console
minikube cache add food-app:1
```

After this, we can see the image inside minikube

```console
minikube cache list
```

#### Deploying

We should create a new
namespace[(theory)](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) to our services:

```console
kubectl create namespace sample-ns
```

So, sample-ns should be appearing by
the [dashboard](http://127.0.0.1:38177/api/v1/namespaces/kubernetes-dashboard/services/kubernetes-dashboard/proxy/?namespace=sample-ns#/overview?namespace=sample-ns)
.

```console
cd food-app/deployment
kubectl apply -n sample-ns -f food-app-pod.yaml
```

We can check by
the [dashboard](http://127.0.0.1:38177/api/v1/namespaces/kubernetes-dashboard/services/kubernetes-dashboard/proxy/#/pod?namespace=sample-ns)
or through terminal curl:

```terminal
curl http://127.0.0.1:38177/api/v1/namespaces/sample-ns/pods/food-app-pod:1080/proxy/food-in-range
```  

response:

```json
{
  "restaurantList": [
    {
      "name": "Restaurant A",
      "foodList": [
        {
          "name": "Food A",
          "price": "2.00"
        },
        {
          "name": "Food B",
          "price": "10.00"
        }
      ]
    },
    {
      "name": "Restaurant B",
      "foodList": [
        {
          "name": "Food A",
          "price": "1.95"
        }
      ]
    },
    {
      "name": "Restaurant C",
      "foodList": [
        {
          "name": "Food A",
          "price": "2.10"
        },
        {
          "name": "Food B",
          "price": "9.00"
        }
      ]
    }
  ]
}
```

If our service stops?

```console
kubectl delete -n sample-ns pod food-app-pod
curl http://127.0.0.1:38177/api/v1/namespaces/sample-ns/pods/food-app-pod:1080/proxy/food-in-range
```

response:

```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "pods \"food-app-pod\" not found",
  "reason": "NotFound",
  "details": {
    "name": "food-app-pod",
    "kind": "pods"
  },
  "code": 404
}
```

Lets make and [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) instead of a pod
creation with [food-app-deployment.yaml](food-app/deployment/food-app-deployment.yaml). This will prevent that our
service is unavailable.

```console
kubectl apply -n sample-ns -f food-app-deployment.yaml
```

We can try to delete one of the pods but our deployment rules will return the pods. Also, different pods, different
names, how to access it without knowing the names? Let's create
a [service](https://kubernetes.io/docs/concepts/services-networking/service/). The setting is
inside [food-app-service.yaml](food-app/deployment/food-app-service.yaml)

You can check at
the [dashboard](http://127.0.0.1:46535/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/#/service/sample-ns/food-app-service?namespace=sample-ns)
that all pods are inside this service by now.

```console
kubectl apply -n sample-ns -f food-app-service.yaml
```

Let's check at
the [management](http://127.0.0.1:46535/api/v1/namespaces/sample-ns/services/food-app-service:1099/proxy/food-in-range)

```console
kubectl get -n sample-ns service
minikube tunnel
curl http://CLUSTER-IP:1099/food-in-range
```

Response:

```console
victor@victor-MS-7995:~$ curl http://10.97.194.112:1099/food-in-range
{"restaurantList": [{"name": "Restaurant A","foodList": [{"name": "Food A","price": "2.00"},{"name": "Food B","price": "10.00"}]}, {"name": "Restaurant B","foodList": [{"name": "Food A","price": "1.95"}]},{"name": "Restaurant C","foodList": [{"name": "Food A","price": "2.10"}, {"name": "Food B","price": "9.00"}]}] }
```

Inside the cluster, other pods can access our service though
internal [DNS](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/).
Let's check how that works...

```console
kubectl run -it --tty pingtest --rm --image=busybox --restart=Never -- /bin/sh

wget -qO- http://food-app-service.sample-ns.svc.cluster.local:1099/food-in-range
```

That pretty much resumes the deployment process.

#### Escaling service

Static scaling:

```console
kubectl scale -n sample-ns --replicas=1 deployment/food-app-deployment
```

Autoscale with cpu constraint:

```console
kubectl autoscale -n sample-ns deployment/food-app-deployment --min=1 --max=5 --cpu-percent=5
```

Or, maybe... we can create rules that
define [autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/) with just
like we defined our deployment and service. Take a look into [food-app-hpa.yaml](food-app/deployment/food-app-hpa.yaml)

```console
kubectl apply -n sample-ns -f food-app-hpa.yaml
```

We can even check the details of the HPA(horizontal pod autoscale) inside
the [dashboard](http://127.0.0.1:46535/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/#/deployment/sample-ns/food-app-deployment?namespace=sample-ns)
or through the terminal:

```console
kubectl describe -n sample-ns hpa food-app-hpa
```

Okay... Let's stress this service with apache (1bi requests from 1k requesters)

```console
ab -n 1000000 -c 1000 http://10.97.194.112:1099/food-in-range
```