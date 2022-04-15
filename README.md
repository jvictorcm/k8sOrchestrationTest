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