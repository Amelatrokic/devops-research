## Blackbox export of application metrics in Kubernetes using Grok Exporter and Prometheus

### Intro

In order to gain better understanding on how our applications behaves in the realtime on specific environment and to be able to pinpoint and alert possible problems that may occur either with system or the application itself, we must continually create telemetry which is, in plain words, process of automatic generation and transmission of data to some system where it will be monitored and analyzed. One of the preferred monitoring tools is Prometheus.

In the context of application monitoring usually, we think about exporting certain application metrics from the application source code itself. This method is often referred to as a white box monitoring. While this is usually prefered way of doing things, sometimes this approach is not possible (in other words, changes in the code itself are not acceptable, you don't have access to source code, you are using third party service which you depend upon..etc.)

In these situations, only way to get some knowledge about behaviour of the application is by observing application logs and use them to acquire metrics. This approach is usually referred to as a black box monitoring. Fortunately, this is also possible using Prometheus with additional metric exporters. One of the most popular is Grok exporter.

These days, more and more applications are running as microservices in Kubernetes ecosystem which is becoming de-facto standard for orchestration of containerized applications. In this article, I will focus on explaining how you can export metrics from the application logs using Grok exporter and Prometheus in Kubernetes and also explain difference between exporting metrics from the running application in contrast to exporting metrics from application running as a cron job (this especially becomes a bit more challenging to implement when working in Kubernetes ecosystem).

### Deploying Prometheus in Kubernetes

Before we go into the application part, we must first make sure that we have Prometheus running in Kubernetes. This can be done using following kubernetes resources:

- [prometheus-configmap](https://github.com/ATLANTBH/devops-research/blob/master/blackbox_app_metrics_exporter_with_prometheus_and_k8s/prometheus/prometheus-configmap.yaml) - contains prometheus config file which will be dynamically loaded into the running Prometheus pod. As you can see, it contains one scrape config defined which points to the grok service running in Kubernetes (later, we will deploy grok exporter in Kubernetes also)
- [prometheus-deployment](https://github.com/ATLANTBH/devops-research/blob/master/blackbox_app_metrics_exporter_with_prometheus_and_k8s/prometheus/prometheus-deployment.yaml) - this is Kubernetes deployment resource which defined one Prometheus pod replica that will be deployed. For the sake of simplicity, we will not take into the consideration persistence of Prometheus data, but this is something that should definitely be done in production usage. In that case, usually, statefulset will be better fit than deployment which would contain persistence volume solution (some nfs storage, local storage, block storage...etc.)
- [prometheus-service](https://github.com/ATLANTBH/devops-research/blob/master/blackbox_app_metrics_exporter_with_prometheus_and_k8s/prometheus/prometheus-service.yaml) - this is Kubernetes NodePort service resource which exposes external Prometheus port on all Kubernetes cluster node through which external access to Prometheus dashboard will be available. In the production environment, you would put load balancer/ingress in front of Prometheus application and enable SSL termination but that is out of the scope for this article

We will first create namespace called `app-monitoring` and then apply resources from above:
```
$ kubectl create namespace app-monitoring

cd ./prometheus
prometheus $ kubectl apply prometheus-configmap.yaml
prometheus $ kubectl apply prometheus-deployment.yaml
prometheus $ kubectl apply prometheus-service.yaml

prometheus $ kubectl get deployment,pod,svc -n app-monitoring
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/prometheus   1/1     1            1           65s

NAME                              READY   STATUS    RESTARTS   AGE
pod/prometheus-5dc98b454f-4kzwv   1/1     Running   0          53s

NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/prometheus      NodePort    10.105.102.60   <none>        9090:31100/TCP   22d
```

When we apply these resources, you should be able to access Prometheus dashboard through your browser: http://<HOST>:31100

### Example application

Now that we have Prometheus up and running, next thing to do is to consider how are we going to run application from which we will scrape metrics using Grok exporter and pull them from Prometheus.

For the purposes of making this article clearer and focused on the problem of blackbox monitoring, I've created small "application" which, by default, logs information which we will scrape to get metrics. Idea is to dockerize this application and run it in kubernetes environment.

Format of logged metrics looks like this:
```
| ACNTST_C: 66 | ACNTST_C_HVA: 53 | ACTIVE_PINGS: 21 | B_WRITERS: 55 |
```

Our assignment is to scrape values for each 4 metrics and push them to the Prometheus but then have ability on the Prometheus to show them as Prometheus GAUGE metric. In contrast to COUNTER, which is cumulative metric, GAUGE is a metric that represents a single numerical value that can arbitrarily go up and down. This way, we would be able to track how metric values are changing in the function of time.
There are two possible scenarios in which this application is running: continually running application and application running as a cron job. This article will show both approaches in context of exporting metrics from the logs.

### Grok exporter

As previously said, to export metrics in the blackbox fashion, we are using [Grok Exporter tool](https://github.com/fstab/grok_exporter) which is a generic Prometheus exporter that extracts metrics from unstructured log data. Grok exporter uses Grok patterns for parsing log lines (Grok was originally developed as part of Logstash to provide log data as input for ElasticSearch).

Before we dockerize and deploy example application in Kubernetes, we must first apply following Grok exporter kubernetes resources:
- [grok-exporter-configmap](https://github.com/ATLANTBH/devops-research/blob/master/blackbox_app_metrics_exporter_with_prometheus_and_k8s/grok_exporter/grok-exporter-configmap.yaml) - contains configuration file for Grok exporter which has following key information: 
  - input - path to the log that will be scraped
  - grok - configures location of grok pattern definitions
  - metrics - defines which metrics we want to scrape from application log
  - server - configures HTTP port
- [grok-exporter-service](https://github.com/ATLANTBH/devops-research/blob/master/blackbox_app_metrics_exporter_with_prometheus_and_k8s/grok_exporter/grok-exporter-service.yaml) - Kubernetes ClusterIP service that will be exposed so that Grok exporter is available. Notice that we expose only ClusterIP service which means that service is available only inside Kubernetes cluster which is what we need (after all, Prometheus is also running in Kubernetes and it needs to know URL of the Grok exporter service)

We can apply these resources using following kubectl commands:
```
cd ./grok_exporter
grok_exporter $ kubectl apply grok-exporter-configmap.yaml
grok_exporter $ kubectl apply grok-exporter-service.yaml
```

### Scraping metrics from the continually running application

We will first look at how to scrape metrics from running application. We need to do following:
- run application which is writing data to the application log file
- run grok exporter which takes data from the application log file, and based on the rules that we define, makes data available to Prometheus
- we already have Prometheus running and listening to the Grok exporter service for incomming metrics

Application can be found [here](https://github.com/ATLANTBH/devops-research/blob/master/blackbox_app_metrics_exporter_with_prometheus_and_k8s/example_application/example_application.rb) which, for the sake of simulation, just logs in format from above. Application takes two arguments: number of time it will generate output with random values, output log where data will be stored. 
Dockerfile for this application is defined [here](https://github.com/ATLANTBH/devops-research/blob/master/blackbox_app_metrics_exporter_with_prometheus_and_k8s/example_application/Dockerfile)

First thing that we need to do is to dockerize our application by simply doing this: 
```
cd ./example_application
example_application $ docker build -t example-application .
```

Now that we have application docker image, it is time to use it in Kubernetes deployment resource defined in file: [example-application-deployment](https://github.com/ATLANTBH/devops-research/blob/master/blackbox_app_metrics_exporter_with_prometheus_and_k8s/example_application/example-application-deployment.yaml)

If we observe this file more carefully, we will see that pod contains two containers: **example-application** and **grok-exporter**. The usual pattern in Kubernetes where one pod has multiple containers is the case where one of them is a sidecontainer which needs to take output of the main running application container and do something with it (in our case, it needs to take log output and pass it to grok exporter application which will, based on the rules defined, scrape metrics for Prometheus)

To be able to do this, important thing to know is that **containers running in the same pod are sharing same volume** which means that we can mount volume on each container and thus, enable sharing of the files between them. For this purpose we are using Volume called **emptyDir** which is first created when a Pod is assigned to a Node, and exists as long as that Pod is running on that node. As the name says, it is initially empty. Containers in the Pod can all read and write the same files in the emptyDir volume, although that volume can be mounted at the same or different paths in each Container. This basically means that while example-application is logging information in the output log, that log is simultaneosly read by the grok-exporter.

Let's apply this resource in kubernetes:
```
cd ./example_application
example_application $ kubectl apply -f example-application-deployment.yaml
example_application $ kubectl get pods -n app-monitoring | grep example-application
example-application-5d594c6bd-8c8ml   2/2     Running   0          71s
```

If we open Prometheus dashboard http://<HOST>:31100, we can search 4 metrics that we created from application logs. You should be able to see them and show values in the graph. For example, for metric ACNTST_C, graph can look like this:

![](https://github.com/ATLANTBH/devops-research/edit/master/blackbox_app_metrics_exporter_with_prometheus_and_k8s/prometheus_graph_1.png)

### Scraping metrics from the application running as cron job

You are probably asking a question: What is the difference in scraping metrics between continually running application vs. application running as cron job since, after all, both of them are logging output in some log which is scraped by Grok exporter, converted into the metrics and pulled from the Prometheus side? You are right. It shouldn't make any difference. However, since application is running in Kubernetes, this adds couple of complexities which I will discuss in this part of an article.
