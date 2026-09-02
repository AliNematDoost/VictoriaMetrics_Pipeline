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
  
everything was made using manifests but thanks to great internet connectivity I enjoyed suffering from ImagePullBackOff and similar errors preventing pods to be up!
