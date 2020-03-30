## Chapter 4 : HealthCheck,Labels and PODS

This chapter will cover below topics:

* HealthCheck
* Labels
* Pods

### HealthCheck

With a few lines of YAML, you can turn your Kubernetes pods into auto healing wonders. 
The right combination of liveness and readiness probes used with Kubernetes deployments can:

```
    1. Enable zero downtime deploys
    2. Prevent deployment of broken images
    3. Ensure that failed containers are automatically restarted
```

#### Readiness probes
The kubelet uses readiness probes to know when is container is ready to start accepting traffic. 
A pod is considered ready when all of its containers are ready.
With readiness probes, Kubernetes will not send traffic to a pod until the probe is successful. 
When updating a deployment, it will also leave old replica(s) running until probes have been successful on new replica. 
That means that if your new pods are broken in some way, they’ll 
never see traffic, your old pods will continue to serve all traffic for the deployment

#### Liveness probes
In many cases, it makes sense to complement readiness probes with liveness probes. 
Despite the similarities, they actually function independently. 
While readiness probes take a more passive approach, liveness probes will actually attempt to restart a container* if it fails.

Here’s what this might look like in a real life failure scenario. Let’s say our API encounters a fatal exception when processing a request.

    1. Readiness probes fails
    2. Kubernetes stops sending traffic to the pod
    3. Liveness probes fails
    4. Due to liveness configuration, kubernetes starts the failed container.
    5. Readiness probes succeeds
    6. kubernetes starts sending traffic to the pod again

Lets take an example of yaml file with readiness probes and livenss probes

```
apiVersion: v1
kind: Service
metadata:
  name: hello-kubernetes
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: hello-kubernetes
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-kubernetes
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-kubernetes
  template:
    metadata:
      labels:
        app: hello-kubernetes
    spec:
      containers:
      - name: hello-kubernetes
        image: aveevadevopsr/hello-kubernetes:1.5
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 3
          timeoutSeconds: 30
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 3
          timeoutSeconds: 30

```

Pretty incredible, right? This is the kind of automated healing that makes Kubernetes incredible to work with.
All of the above is made possible with just a few lines of YAML.

##### Probes for HTTP Services

The most noticeable impact these probes can have will be on HTTP services. 
The readinessProbe and livenessProbe attributes should be placed at the same level 
as the name and image attributes for your containers in a deployment. 
For one of our services, the probe configuration looks something like:

```
        livenessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 3
          timeoutSeconds: 30
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 3
          timeoutSeconds: 30
```

The attributes are pretty straightforward here. The key ones to pay attention to are:
###### initialDelaySeconds 
How long to wait before sending a probe after a container starts. 
For liveness probes this should be safely longer than the time your app usually takes to start up. 
Without that, you could get stuck in a reboot loop. On the other hand, this value can be lower 
for readiness probes as you’ll likely want traffic to reach your new containers as soon as they’re ready.
###### timeoutSeconds 
How long a request can take to respond before it’s considered a failure. 
###### periodSeconds 
How often a probe will be sent. 
The value you set here depends on finding a balance between sending too many probes to your service or 
going too long without detecting a failure. 
In most cases we settle for a value between 10 and 20 seconds here.

##### Probes for Background Services
Although readiness and liveness probes are remarkably straightforward for services that expose an HTTP endpoint, 
it takes a bit more effort to probe background services. Here’s an example of probes we’re currently using for one of our background services:

```
readinessProbe:
  exec:
    command:
      - test
      - '`find alive.txt -mmin -1`'
  initialDelaySeconds: 5
  periodSeconds: 15
livenessProbe:
  exec:
    command:
      - test
      - '`find alive.txt -mmin -1`'
  initialDelaySeconds: 15
  periodSeconds: 15
```

Our background service touches alive.txt every 15 seconds, and the probes test to ensure that file has been modified within 1 minute. 
This ensures that not only is the service running, it’s continuing to function as expected.
The exec option for probes is incredibly powerful. 
We’re using a fairly simple command in the above example, but the sky’s the limit here. 
Each service is unique and this allows for a flexible way to ensure that things remain functional.

### Labels

Labels are the mechanism you use to organize Kubernetes objects. A label is a key-value pair with certain restrictions concerning length and allowed values but without any pre-defined meaning. So you’re free to choose labels as you see fit, for example, to express environments such as ‘this pod is running in production’ or ownership, like ‘department X owns that pod’.

Let’s create a pod that initially has one label (env=development):

```
kubectl create -f hello-kubernetes-label.yml
```

Get labels

```
kubectl get pods --show-labels
```
In above get pods command note the --show-labels option that output the labels of an object in an additional column.

You can add a label to the pod as:

```
kubectl label pods ${POD_NAME} owner=devops
kubectl get pods --show-labels
```

To use a label for filtering, for example to list only pods that have an owner that equals devops, use the --selector option

```
kubectl get pods --selector owner=devops
kubectl get pods -l env=development
```
In above example, the --selector option is abbreviated to -l

##### Set based selectors

Kubernetes objects also support set based selectors. Lets launch another pod that has two labels (env=production and owner=devops)

```
kubectl create -f hello-kubernetes-label-set.yml
```

Get pods with env=production and owner=devops
```
kubectl get pods -l 'env in (production, development)'
```

To delete pods with specified label
```
kubectl delete pods -l 'env in (production, development)'
```

### PODS

##### Overview of a Pod
A Kubernetes pod is a group of containers that are deployed together on the same host. If you frequently deploy single containers, you can generally replace the word "pod" with "container" and accurately understand the concept.

Pods operate at one level higher than individual containers because it's very common to have a group of containers work together to produce an artifact or process a set of work.

For example, consider this pair of containers: a caching server and a cache "warmer". You could build these two functions into a single container, but now they can each be tailored to the specific task and shared between different projects/teams.

Another example is an app server pod that contains three separate containers: the app server itself, a monitoring adapter, and a logging adapter. The logging and monitoring containers could be shared across all projects in your organization. These adapters could provide an abstraction between different cloud monitoring vendors or other destinations.

Any project requiring logging or monitoring can include these containers in their pods, but not have to worry about the specific logic. All they need to do is send logs from the app server to a known location within the pod. How does that work? Let's walk through it.

##### Shared Namespaces, Volumes and Secrets
By design, all of the containers in a pod are connected to facilitate intra-pod communication, ease of management and flexibility for application architectures. If you've ever fought with connecting two raw containers together, the concept of a pod will save you time and is much more powerful. The hostname is set to the pod’s Name for the application containers within the pod

#### Shared Network
All containers share the same network namespace & port space. Communication over localhost is encouraged. Each container can also communicate with any other pod or service within the cluster.

##### Shared Volumes
Volumes attached to the pod may be mounted inside of one or more containers. In the logging example above, a volume named logs is attached to the pod. The app server would log output to logs mounted at /volumes/logs and the logging adapter would have a read-only mount to the same volume. If either of these containers needed to restarted, the log data is preserved instead of being lost.

There are many types of volumes supported by Kubernetes, including native support for mounting Github repos, network disks (EBS, NFS, etc), local machine disks, and a few special volume types, like secrets.

Here's an example pod:
```
apiVersion: v1
kind: Pod
metadata:
  name: example-app
  labels:
    app: example-app
    version: v1
    role: backend
spec:
  containers:
  - name: java
    image: companyname/java
    ports:
    - containerPort: 443
    volumeMounts:
    - mountPath: /volumes/logs
      name: logs
  - name: logger
    image: companyname/logger:v1.2.3
    ports:
    - containerPort: 9999
    volumeMounts:
    - mountPath: /logs
      name: logs
  - name: monitoring
    image: companyname/monitoring:v4.5.6
    ports:
    - containerPort: 1234
   
   ```

##### Resources
Resource limits such as CPU and RAM are shared between all containers in the pod.

##### Creating Pods
Pods are considered ephemeral "cattle" and won't survive a machine failure and may be terminated for machine maintenance. For high resiliency, pods are managed by a replication controller, which creates and destroys replicas of pods as needed. Individual pods can also be created outside of a replication controller, but this isn't a common practice.

Kubernetes services should always be used to expose pod(s) to the rest of the cluster in order to provide the proper level of abstraction since individual pods will come and go.

Replication controllers and services use the pod labels to select a group of pods that they interact with. Your pods will typically have labels for the application name, role, environment, version, etc. Each of these can be combined in order to select all pods with a certain role, a certain application, or a more complex query. The label system is extremely flexible by design and experimentation is encouraged to establish the practices that work best for your company or team.

##### Pod Template

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']
```

To create a pod, use below command

```
kubectl apply -f hello-kubernetes-pod.yml
kubectl get pods
```



##### Termination of Pods

Because pods represent running processes on nodes in the cluster, it is important to allow those processes to gracefully terminate when they are no longer needed (vs being violently killed with a KILL signal and having no chance to clean up). Users should be able to request deletion and know when processes terminate, but also be able to ensure that deletes eventually complete. When a user requests deletion of a pod, the system records the intended grace period before the pod is allowed to be forcefully killed, and a TERM signal is sent to the main process in each container. Once the grace period has expired, the KILL signal is sent to those processes, and the pod is then deleted from the API server. If the Kubelet or the container manager is restarted while waiting for processes to terminate, the termination will be retried with the full grace period.

To delete the pod, use below command

```
kubectl delete -f hello-kubernetes-pod.yml
```

By default, all deletes are graceful within 30 seconds. The kubectl delete command supports the --grace-period=<seconds> option which allows a user to override the default and specify their own value. The value 0 force deletes the pod. In kubectl version >= 1.5, you must specify an additional flag --force along with --grace-period=0 in order to perform force deletions.

##### Pods lifecycle

Refer https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/ to know more about Pod lifecycle


