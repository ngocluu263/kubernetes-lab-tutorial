# Monitoring the cluster
Kubernetes provides detailed information about applications and cluster resources usage. This information allows to evaluate the application’s performance and where bottlenecks can be removed to improve overall performance of the cluster.

In Kubernetes, application monitoring does not depend on a single monitoring solution. In this section, we're going to explore some of the monitoring tools currently available.

  * [Resources usage]()
  * [cAdvisor]()
  * [Metric Server]()
  * [Autoscaling]()
  * [Prometheus]()
 
## Resources usage
When creating a pod, we can specify the amount of CPU and memory that a container requests and a limit on what it may consume. 

The following pod manifest ``requests-pod.yaml`` specifies the CPU and memory requests for its single container.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: request-pod
  namespace:
  labels:
spec:
  containers:
  - image: busybox:latest
    command: ["dd", "if=/dev/zero", "of=/dev/null"]
    name: busybox
    resources:
      requests:
        cpu: 200m
  restartPolicy: Never
```

By specifying resource requests, we specify the minimum amount of resources the pod needs. However the pod above can take more than the requested CPU and memory we requested, according to the capacity and the actual load of the working node. Each node has a certain amount of CPU and memory it can allocate to pods. When scheduling a pod, the scheduler will only consider nodes with enough unallocated resources to meet the pod requirements. If the amount of unallocated CPU or memory is less than what the pod requests, the scheduler will not consider the node, because the node can’t provide the minimum amount
required by the pod.

Create the pod above

    kubectl apply -f requests-pod.yaml

Checking the resource usage

    kubectl exec requests-pod top

    Mem: 3469164K used, 2398380K free, 310084K shrd, 2072K buff, 2264708K cached
    CPU: 18.9% usr 36.5% sys  0.1% nic 43.9% idle  0.0% io  0.0% irq  0.3% sirq
    Load average: 1.57 0.73 0.54 3/642 8
      PID  PPID USER     STAT   VSZ %VSZ CPU %CPU COMMAND
        1     0 root     R     1236  0.0   1 50.0 dd if /dev/zero of /dev/null
        5     0 root     R     1244  0.0   1  0.0 top

we see the pod taking up to 50% of total of the node CPU. On a 2 core CPU node, this corresponds to a single CPU and it's as espected because the ``dd`` command is single-thread and cannot take more than one CPU by design.

Please, note that we're not specifying the maximum amount of resources the pod can consume. If we want to limit the usage of resources, we have to limit the pod as in the following ``limited-pod.yaml`` descriptor file

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: limited-pod
  namespace:
  labels:
spec:
  containers:
  - image: busybox:latest
    command: ["dd", "if=/dev/zero", "of=/dev/null"]
    name: busybox
    resources:
      requests:
        cpu: 200m
      limits:
        cpu: 200m
  restartPolicy: Never
  terminationGracePeriodSeconds: 10
```

Create the pod

    kubectl apply -f limited-pod.yaml

Checking the resource usage

    kubectl exec limited-pod top

    Mem: 3458876K used, 2408668K free, 309892K shrd, 2072K buff, 2264840K cached
    CPU:  7.6% usr  8.5% sys  0.0% nic 83.5% idle  0.0% io  0.0% irq  0.2% sirq
    Load average: 1.97 1.14 0.74 6/621 12
      PID  PPID USER     STAT   VSZ %VSZ CPU %CPU COMMAND
        1     0 root     R     1236  0.0   0 10.5 dd if /dev/zero of /dev/null
        5     0 root     S     1244  0.0   0  0.0 top

We can see the pod taking 10% of the node CPU. On a 2 core CPU node, this corresponds to 200m of the single CPU.

Both resources requests and limits are specified for each container individually, not for the entire pod. The pod resource requests and limits are the sum of the requests and limits of all the containers contained into the pod. 

We can check the usage of the resources at node level by describing the node

    kubectl describe node kubew00

    ...

      Namespace            Name                 CPU Requests  CPU Limits  Memory Requests  Memory Limits
      ---------            ----                 ------------  ----------  ---------------  -------------
      default              limited-pod          200m (10%)    200m (5%)   0 (0%)           0 (0%)
      default              request-pod          200m (10%)    0 (0%)      0 (0%)           0 (0%)

    Allocated resources:

      Resource  Requests     Limits
      --------  --------     ------
      cpu       400m (20%)   200m (10%)
      memory    0 (0%)       0 (0%)


## cAdvisor
The resource usage is provided by the **cAdvisor** agent running into kubelet binary and exposed externally to the port 4194 on the worker node. This is an unsecure port and can be closed. If not closed, we can start a simple web UI of the cAdvisor agent by using a web browser. The cAdvisor auto-discovers all containers running on the node and collects CPU, memory, filesystem, and network usage statistics. It also provides the overall machine usage by analyzing the root container.

## Metric Server
The **Metric Server** is an additional component running as pod in the cluster, making centrally accessible all the metrics collected by all the cAdvisor agents running on each worker nodes. Once installed, the metric server makes it possible to obtain resource usages for nodes and individual pods through the ``kubectl top`` command.

To see how much CPU and memory is being used on the worker nodes, run the command: 

    kubectl top nodes
    
    NAME          CPU(cores)   CPU%      MEMORY(bytes)   MEMORY%   
    kubew03       366m         18%       2170Mi          38%       
    kubew04       102m          6%       2170Mi          38%   
    kubew05       708m         40%       2170Mi          38%   

This shows the actual, current CPU and memory usage of all the pods running on all the nodes.

To see how much each individual pod is using, use the command:

    kubectl top pods

    NAME                    CPU(cores)   MEMORY(bytes)   
    curl                    0m           0Mi             
    limited-pod             200m         0Mi             
    request-pod             999m         0Mi        

To see resource usages across individual containers instead of pods, use the ``--containers`` option

    kubectl top pod limited-pod --containers

    POD           NAME      CPU(cores)   MEMORY(bytes)   
    limited-pod   busybox   200m         0Mi             

Metrics are also exposed as API by the kubernetes API server at ``http://cluseter.local/apis/metrics.k8s.io/v1beta1`` address.

### Setup the Metric Server
The purpose of the Metric Server is to provide a stable, versioned API that other kubernetes components can rely on. Metric Server is part of the so-called *core metrics pipeline*.

In order to get a resource metrics server up and running, we first need to configure the *aggregation layer* on the cluster. The aggregation layer is a general feature of the API server, allowing other custom API servers to register themselves to the main API server. This is accomplished by configuring the *kube-aggregator* on the main API server. The aggregator is basically a proxy that forwards requests coming from clients to the custom API servers.

![](../img/aggregator.png?raw=true)

Configuring the aggregation layer involves setting a number of flags on the API Server

     --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
     --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
     --requestheader-allowed-names=front-proxy-client
     --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
     --requestheader-extra-headers-prefix=X-Remote-Extra-
     --requestheader-group-headers=X-Remote-Group
     --requestheader-username-headers=X-Remote-User
     --enable-aggregator-routing=true

using some additional certificates. See [here](https://github.com/kubernetes-incubator/apiserver-builder/blob/master/docs/concepts/auth.md) for details.

The Metric Server is one of the custom API server that can be configured with the aggregator. To install it, configure the API server to enable the aggregator and then deploy it in the ``kube-system`` namespace from the manifest files:

     kubectl apply -f auth-delegator.yaml
     kubectl apply -f auth-reader.yaml
     kubectl apply -f resource-reader.yaml
     kubectl apply -f metrics-apiservice.yaml
     kubectl apply -f metrics-server-sa.yaml
     kubectl apply -f metrics-server-deployment.yaml
     kubectl apply -f metrics-server-service.yaml

The metric server will be deployed as pod and exposed as an internal service.

    get deploy metrics-server

    NAME             DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    metrics-server   1         1         1            1           2h

    kubectl get svc metrics-server

    NAME             TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)         AGE
    metrics-server   ClusterIP   10.32.0.19    <none>        443/TCP         2h

The Metric Server can be used to enable the pods autoscaling feature.

## Autoscaling
Applications running in pods can be scaled out manually by increasing the replicas field of the Replica Set, Deploy, or Stateful Set. However, kubernetes can monitor the pods and scale them up automatically as soon as it detects an increase in the CPU usage or some other metric. To achieve this, we need to configure an autoscaler object. We can have multiple autoscalers, each one controlling a separated set of pods.

The pods autoscaling process is implemented as a control loop that can be split into three steps:

 1. Obtain metrics of all the pods managed by the scaled resource object.
 2. Calculate the number of pods required to bring the metrics close to a target value.
 3. Update the replicas field of the scaled resource.

The autoscaler controller doesn’t perform the collection of the pod metrics itself. Instead, it gets the metrics from the Metric Server through REST calls.

Once the autoscaler has all the metrics for all the pods of the target resource, it can use those metrics to return the number of replicas to bring the metrics close to the target.

When the autoscaler is configured to consider only a single metric, calculating the required replica count is simple: sum the metrics values of all the pods, divide that by the target value and then round it up to the next integer. For example, if we set the target value to be *50%* of requested CPU and we have 3 pods consuming, respectively, *60%*, *90%*, and *50%* of CPU, then the resulting number is *(60+90+50)/50=4* replicas.

The final step of the autoscaler is updating the desired replica count field on the resource object, e.g. the Deploy, and then letting it take care of spinning up additional pods or deleting excess ones.

*Note that pods autoscaling does not apply to objects that can’t be scaled, for example, Daemon Sets.*

### Autoscaling based on CPU usage
In general, speaking about *"CPU usage"* we can refer to any of the following:

 * Amount of the consumed node's CPU from the single pod.
 * Amount of the consumed node's CPU from all the pods.
 * Amount of the requested CPU, as specified by the pod descriptor.
 * Amount of the limited CPU, as specified by the pod descriptor.

The most common used metric for pods autoscaling is the node's CPU consumed by all the pods controlled by the autoscaler. Those values are collected from the Metric Server and evaluated as an average.

The target parameter used by the autoscaler to determine the number of replicas is the requested CPU specified by the pod descriptor.

In this section, we're going to configure the pods autoscaler for a set on nginx pods.

Create a deploy as in the following ``nginx-deploy.yaml`` descriptor file

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
  name: nginx
  namespace:
spec:
  replicas: 3
  selector:
    matchLabels:
      run: nginx
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - image: nginx:1.12
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
        resources:
          requests:
            cpu: 100m
```

Make sure to configure the CPU requests to 100 millicore. This means that each pod needs for the 5% of node's CPU to be scheduled, considering a 2 CPU node.

### Autoscaling based on memory usage

### Autoscaling based on custom metrics
But one metric does not fit all use cases and for different kind of applications, the metric might vary. For example, for a message queue, the number of messages in waiting state might be the appropriate metric. For memory intensive applications, memory consumption might be that metric. If you have a business application which handles about 1000 transactions per second for a given capacity pod, then you might want to use that metric and scale out when the TPS in pod reach above 850.

So far we have only considered the scale-up part, but when the workload usage drops, there should be a way to scale down gracefully and without causing interruption to existing requests being processed.


### Configuring the autoscaler
The period of the autoscaler is controlled by the ``--horizontal-pod-autoscaler-sync-period`` flag of controller manager. The default value is 30 seconds. The delay between two scale up operations is controlled by using the flag  ``--horizontal-pod-autoscaler-upscale-delay``. The default value is 3 minutes. Similarly, the delay between two scale down operations is adjustible with flag  ``--horizontal-pod-autoscaler-downscale-delay``. The default value is 5 minutes. 