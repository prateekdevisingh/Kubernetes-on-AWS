This chapter will cover various kubernetes controllers. This includes
* ReplicaSet
* ReplicationController
* Deployment
* StatefulSets
* DaemonSet
* ConfigMap
* Garbage Collection
* TTL Controller for Finished Resources
* CronJob

### Replication Controller
The Replication Controller is the original form of replication in Kubernetes.  It’s being replaced by Replica Sets, but it’s still in wide use, so it’s worth understanding what it is and how it works.

A Replication Controller is a structure that enables you to easily create multiple pods, then make sure that that number of pods always exists. If a pod does crash, the Replication Controller replaces it.

Replication Controllers also provide other benefits, such as the ability to scale the number of pods, and to update or delete multiple pods with a single command.

You can create a Replication Controller with an imperative command, or declaratively, from a file.  For example, create a new file called hello-kubernetes-rc.yml and add the following text:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-kubernetes
spec:
  replicas: 3
  selector:
      app: hello-kubernetes
  template:
    metadata:
      name: hello-kubernetes
      labels:
        app: hello-kubernetes
    spec:
      containers:
      - name: hello-kubernetes
        image: aveevadevopsr/hello-kubernetes:1.5
        ports:
        - containerPort: 8080
   ```
Now tell Kubernetes to create the Replication Controller based on that file:

```
kubectl create -f hello-kubernetes-rc.yml
replicationcontroller "hello-kubernetes" created
```

Let’s take a look at what we have using the describe command:

```
kubectl describe rc hello-kubernetes
```

As you can see, we’ve got the Replication Controller, and there are 3 replicas, of the 3 that we wanted.  All 3 of them are currently running.  You can also see the individual pods listed underneath, along with their names.  If you ask Kubernetes to show you the pods, you can see those same names show up:

```
kubectl get pods
```

Delete the replication controller

```
kubectl delete rc hello-kubernetes
```
As you can see, when you delete the Replication Controller, you also delete all of the pods that it created.

### ReplicaSet
Replica Set ensures how many replica of pod should be running. It can be considered as a replacement of replication controller. The key difference between the replica set and the replication controller is, the replication controller only supports equality-based selector whereas the replica set supports set-based selector.

Replica Sets are a sort of hybrid, in that they are in some ways more powerful than Replication Controllers, and in others they are less powerful.

Replica Sets are declared in essentially the same way as Replication Controllers, except that they have more options for the selector.  For example, we could create a Replica Set like this:

```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: hello-kubernetes
spec:
  replicas: 3
  selector:
    matchExpressions:
      - {key: app, operator: In, values: [hello-kubernetes, hello-kubernetes1]}
      - {key: teir, operator: NotIn, values: [production]}
  template:
    metadata:
      labels:
        app: hello-kubernetes
        environment: dev
    spec:
      containers:
      - name: hello-kubernetes
        image: aveevadevops/hello-kubernetes:1.5
        ports:
        - containerPort: 8080
```
Create replicaSets now using command:
```
kubectl create -f hello-kubernetes-replicasets.yml
replicaset "hello-kubernetes" created
```

Check replicasets
```
kubectl get rs ${NAME}
kubectl get hello-kubernetes
```

Describe replicasets to get details
```
kubectl describe rs ${NAME}
kubectl describe rs hello-kubernetes
```

As you can see, the output is pretty much the same as for a Replication Controller (except for the selector), and for most intents and purposes, they are similar.  The major difference is that the rolling-update command works with Replication Controllers, but won’t work with a Replica Set.  This is because Replica Sets are meant to be used as the backend for Deployments.

Let’s clean up before we move on.

```
kubectl delete rs ${NAME}
kubectl delete rs hello-kubernetes
```

### Deployment
Deployments are intended to replace Replication Controllers.  They provide the same replication functions (through Replica Sets) and also the ability to rollout changes and roll them back if necessary.

For more information about deployment, refer Chapter #3 - https://github.com/aveeva-devops/kubernetes/blob/master/docs/Manage_Deployments.md 

### StatefulSets

### DaemonSet
Like other workload objects, DaemonSets manage groups of replicated Pods. However, DaemonSets attempt to adhere to a one-Pod-per-node model, either across the entire cluster or a subset of nodes. As you add nodes to a node pool, DaemonSets automatically add Pods to the new nodes as needed.

DaemonSets use a Pod template, which contains a specification for its Pods. The Pod specification determines how each Pod should look: what applications should run inside its containers, which volumes it should mount, its labels and selectors, and more.

##### Usage patterns
DaemonSets are useful for deploying ongoing background tasks that you need to run on all or certain nodes, and which do not require user intervention. Examples of such tasks include storage daemons like ceph, log collection daemons like fluentd, and node monitoring daemons like collectd.

For example, you could have DaemonSets for each type of daemon run on all of your nodes. Alternatively, you could run multiple DaemonSets for a single type of daemon, but have them use different configurations for different hardware types and resource needs.

##### Creating DaemonSets
You can create a DaemonSet using kubectl apply or kubectl create.
The following is an example of a DaemonSet manifest file:

```
apiVersion: v1/beta2 # For Kubernetes version 1.9 and later, use apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  selector:
      matchLabels:
        name: fluentd # Label selector that determines which Pods belong to the DaemonSet
  template:
    metadata:
      labels:
        name: fluentd # Pod template's label selector
    spec:
      nodeSelector:
        type: prod # Node label selector that determines on which nodes Pod should be scheduled
                   # In this case, Pods are only scheduled to nodes bearing the label "type: prod"
      containers:
      - name: fluentd
        image: gcr.io/google-containers/fluentd-elasticsearch:1.20
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
```
In this example:

A DaemonSet named fluentd is created, indicated by the metadata: name field.
DaemonSet's Pod is labelled fluentd.
A node label selector (type: prod) declares on which labelled nodes the DaemonSet schedules its Pod.
The Pod's container pulls the fluentd-elasticsearch image at version 1.20. The container image is hosted by Container Registry.
The container requests 100m of CPU and 200Mi of memory, and limits itself to 200Mi total of memory usage.
In sum, the Pod specification contains the following instructions:

Label Pod as fluentd.
Use node label selector type: prod to schedule Pod to matching nodes, and do not schedule on nodes which do not bear the label selector. (Alternatively, omit the nodeSelector field to schedule on all nodes.)
Run fluentd-elasticsearch at version 1.20.
Request some memory and CPU resources.
For more information about DaemonSet configurations, refer to the DaemonSet API reference.

##### Updating DaemonSets
You can update DaemonSets by changing its Pod specification, resource requests and limits, labels, and annotations.

To decide how to handle updates, DaemonSet use a update strategy defined in spec: updateStrategy. There are two strategies, OnDelete and RollingUpdate:

OnDelete does not automatically delete and recreate DaemonSet Pods when the object's configuration is changed. Instead, Pods must be manually deleted to cause the controller to create new Pods that reflect your changes.

RollingUpdate automatically deletes and recreates DaemonSet Pods. With this strategy, valid changes automatically triggers a rollout. This is the default update strategy for DaemonSets.
Update rollouts can be monitored by running the following command:

kubectl rollout status ds [DAEMONSET_NAME]

### configmap
https://cloud.google.com/kubernetes-engine/docs/concepts/configmap
