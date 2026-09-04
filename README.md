# Task Report 

## Implementing Metrics in Django application 
---
At first all metrics added to code based on their functionality. Metrics are added to `./clusterproject/metrics.py` and they are used where they should to change the metrics while the code is running.
Added metrics are exposed on /metrics.

Django project repo : `https://github.com/AliNematDoost/Django-Project-Hamamouz`

## Deploying application on KIND cluster 
---
Then deployed the application on KIND local cluster and exposed endpoints on localhost. now /metrics is accessible using http://localhost/api/metrics on localhost.

## Deploying VictoriaMetrics Pipeline
---

### Concepts I learned here
Before starting anything, I preferred to read more about concepts in this part, so I got these:

1. We are going to first install VictoriaMetrics operator. but what exactly iis operator here?

   Operator is a Controller placed in k8s that manages and creates resources for CRDs we define. Kubenetes will know that we have new types of kind and accepts them but does not know what to do with them.
   at this time operator comes in and manages new objects with new kinds defined earlier and create kubenetes resources for them as they need. ( Deployment also has a controller that creates replicaset for
   deployment objects )

2. Got deeper about components of VictoriaMetrics and what they gonna do. like:
   - VMServiceScrape: This is going to identify which targets we are going to collect their metrics
   - VMAgent: This is going to collect metrics from targets and remote write them
   - VMSingle: This is going to store collected metrics is storage

Now for deploying the Pipeline I followed this structure: 

django /metrics --> django service <-- VNServiceScrape matches and targets this <--VMAgent collects metrics and writes <-- VMSingle stores metrics on storage


### VMSingle

`VMSingle` is the VictoriaMetrics component responsible for **storing the collected metrics**.

Metrics are retained for 4 days. Data older than this is automatically removed.

```yaml
storage:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

This requests a **1 GiB persistent volume** for storing the metrics. `ReadWriteOnce` means the volume can be mounted for read/write by one node.

---

### VMAgent

`VMAgent` is responsible for **discovering scrape targets, collecting metrics, and sending them to VictoriaMetrics**.

```yaml
serviceScrapeNamespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: monitoring-system
```

This tells VMAgent to look for `VMServiceScrape` resources in the `monitoring-system` namespace which we have made VmServiceScrape in it.

```yaml
serviceScrapeSelector:
  matchLabels: {}
```

An empty selector means that VMAgent accepts all matching `VMServiceScrape` objects in the selected namespace.

```yaml
replicaCount: 1
scrapeInterval: 15s
```

There is one VMAgent replica, and it collects metrics every 15 seconds.

```yaml
remoteWrite:
  - url: "http://vmsingle-hamamooz-vmsingle.monitoring-system.svc:8429/api/v1/write"
```

After collecting metrics, VMAgent sends them to the VMSingle instance through its Kubernetes Service.
Found the name and port of VMSingle service as below:
```
k get svc -n monitoring-system
NAME                                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
vm-operator-victoria-metrics-operator   ClusterIP   10.43.142.87    <none>        8080/TCP,9443/TCP   17h
vmagent-hamamooz-vmagent                ClusterIP   10.43.175.31    <none>        8429/TCP            17h
vmsingle-hamamooz-vmsingle              ClusterIP   10.43.130.242   <none>        8429/TCP,8428/TCP   17h
```

**no matter to use which port for connecting to VMSingle service because both of them route the traffic to the same port of VMSingle pod:**
```
k describe svc vmsingle-hamamooz-vmsingle -n monitoring-system
Name:                     vmsingle-hamamooz-vmsingle
Namespace:                monitoring-system
Labels:                   app.kubernetes.io/component=monitoring
                          app.kubernetes.io/instance=hamamooz-vmsingle
                          app.kubernetes.io/name=vmsingle
                          managed-by=vm-operator
Annotations:              <none>
Selector:                 app.kubernetes.io/component=monitoring,app.kubernetes.io/instance=hamamooz-vmsingle,app.kubernetes.io/name=vmsingle,managed-by=vm-operator
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.43.130.242
IPs:                      10.43.130.242
Port:                     http  8429/TCP
TargetPort:               8429/TCP
Endpoints:                10.42.1.67:8429
Port:                     http-alias  8428/TCP
TargetPort:               8429/TCP
Endpoints:                10.42.1.67:8429
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>
```

Therefore, the flow is:

**VMServiceScrape --> VMAgent --> VMSingle**

---

### VMServiceScrape

`VMServiceScrape` tells VMAgent **which Kubernetes Service to scrape and where the metrics endpoint is located**.

```yaml
namespaceSelector:
  matchNames:
    - application
```

This tells VMAgent to look for the target Service in the `application` namespace.

```yaml
selector:
  matchLabels:
    service: django
```

It selects the Django Service using its label:

```yaml
service: django
```
For that reason I modified the django-service manifest and added a label to its metadata section.

```yaml
endpoints:
  - port: metrics
    path: /api/metrics
```
For that reason I modified the django-service manifest and added a name to port section.

This tells VMAgent to scrape the selected Service on the port named `metrics` and request:

```text
/api/metrics
```

So this resource creates the connection:

```text
Django Service --> VMServiceScrape --> VMAgent
```

Together with the other components, the complete pipeline is:

```text
Django /api/metrics --> Django Service --> VMServiceScrape --> VMAgent --> VMSingle
```

**Note**
The final version of django-service after updates:
```
apiVersion: v1
kind: Service
metadata:
  name: django-service
  namespace: application
  labels:
    service: django
spec:
  selector:
    app: django
  ports:
    - name: metrics
      port: 8000
      targetPort: 8000
```



## Problems and Challenges Solved 

### Ingress creation and VMUI api call

First of all I created a new Ingress Rule for exposing VMUI on host I already have. so I created this :
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vm-ingress
  namespace: monitoring-system
spec:
  ingressClassName: traefik
  rules:
    - host: nematdoust.osdl.ir
      http:
        paths:
          - path: /vmui
            pathType: Prefix
            backend:
              service:
                name: vmsingle-hamamooz-vmsingle
                port:
                  number: 8429
```

**Also in this step I learned that Kubernetes interprets that the Backend service specified in ingress.yaml is placed in the same namespace as ingress is created in it. So here Kunernetes searches for 
service called `vmsingle-hamamooz-vmsingle` in monitoring-system namespace. As a result of this, it would be better to create the ingress rule in the same namespace as service**

After creating this ingress VVMUI was available on `http://nematdoust.osdl.ir/vmui` but searching query did not work. I checked the network tab of browser inspect and found out that the queries api call is on a different path : `http://nematdoust.osdl.ir/prometheus/api/v1/query_range`

and we configured ingress to only accept paths with prefix /vmui. 

So I configured ingress rule as below:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vm-ingress
  namespace: monitoring-system
spec:
  ingressClassName: traefik
  rules:
    - host: nematdoust.osdl.ir
      http:
        paths:
          - path: /vmui
            pathType: Prefix
            backend:
              service:
                name: vmsingle-hamamooz-vmsingle
                port:
                  number: 8429
    - host: nematdoust.osdl.ir
      http:
        paths:
          - path: /prometheus
            pathType: Prefix
            backend:
              service:
                name: vmsingle-hamamooz-vmsingle
                port:
                  number: 8429
```

And created a new rule to also accept /prometheus which is used for query API calles in vmui.
