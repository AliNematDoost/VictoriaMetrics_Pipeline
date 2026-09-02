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

## VMAgent

`VMAgent` is responsible for **discovering scrape targets, collecting metrics, and sending them to VictoriaMetrics**.

```yaml
serviceScrapeNamespaceSelector:
  matchNames:
    - monitoring-system
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
  - url: "http://hamamooz-vmsingle.monitoring-system.svc:8428/api/v1/write"
```

After collecting metrics, VMAgent sends them to the VMSingle instance through its Kubernetes Service.

Therefore, the flow is:

**VMServiceScrape --> VMAgent --> VMSingle**

---


everything was made using manifests but thanks to great internet connectivity I enjoyed suffering from ImagePullBackOff and similar errors preventing pods to be up!
