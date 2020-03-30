## Chapter 3 : Manage Deployments 

This chapter will cover below topics:

* Hello world deployment on kubernetes
* Troubleshoot application 
* Expose your application
* Scale up and Scale down your app
* Rolling updates and Rollback your existing deployment

### Hello world deployment on kubernetes

Create deployment hello-kubernetes.yaml file. 

```
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
```

Once file is created. Deploy application and expose it publicly using service given below. Read more about service in later part of this chapter.

```
kubectl apply -f hello-kubernetes-service.yaml
```

Service Yaml:

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
```

Validate application:
```
kubectl get pods
```

This will show output as:
```
bash-3.2$ kubectl get pods
NAME                                READY   STATUS             RESTARTS   AGE
hello-kubernetes-7bf6fbdb57-8ct4z   1/1     Running            0          10m
hello-kubernetes-7bf6fbdb57-8d6t5   1/1     Running            0          10m
hello-kubernetes-7bf6fbdb57-h5v8t   1/1     Running            0          10m

### Troubleshoot application 

Below commands will help to troubleshoot app running on kubernetes:

```

Get logs of a specific pod

```
kubectl logs ${POD_NAME}

```

Get status of a single pod

```
kubectl get -w po ${POD_NAME}
```

Get Complete details about a POD_NAME

```
kubectl describe pods ${POD_NAME}
```

Get Name of container running inside the POD_NAME

```
kubectl get pod ${POD_NAME} -o jsonpath='{.spec.containers[*].name}'
```

Get logs of a specific container running inside pod

```
kubectl logs ${POD_NAME} ${CONTAINER_NAME}
```

If your container has previously crashed, you can access the previous container’s crash log with:

```
kubectl logs --previous ${POD_NAME} ${CONTAINER_NAME}
```

Debug inside container running in pod:

```
kubectl exec ${POD_NAME} -c ${CONTAINER_NAME} -- ${CMD} ${ARG1} ${ARG2} ... ${ARGN}
Note: -c ${CONTAINER_NAME} is optional. You can omit it for pods that only contain a single container.
For example : kubectl exec ${POD_NAME} -c ${CONTAINER_NAME} -- ls -ltr /tmp
kubectl exec -it hello-world-deployment-677c9f4789-64lfn -c hello-world -- bash
```

Login inside a running container inside pod:

```
Kubectl exec -it ${POD_NAME} -c ${CONTAINER_NAME} -- bash
```

### Expose your application using service

Applications/pods inside a cluster can be exposed using service. Read https://cloud.google.com/kubernetes-engine/docs/concepts/service for more details about service and its types. Here we will use service type "LoadBalancer" to expose Hello-world using an aws LoadBalancer: 

Below YAMl file explains service, which can be deployed to expose the application "hello-kubernetes":

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
```
In above example:

* Deployment kind is "service"
* Service type is "LoadBalancer"
* LabelSelector is " app: hello-kubernetes"

Deploy this service using:
```
kubectl apply -f hello-kuberenets-service.yaml
```

This will expose deployed application using AWS ELB. Find out endpoint url using:
```
Kubectl get service
```

### Scale up and Scale down your app:

Scaling is accomplished by changing the number of replicas in a Deployment.
Scaling out a Deployment will ensure new Pods are created and scheduled to Nodes with available resources. Scaling will increase the number of Pods to the new desired state. Kubernetes also supports autoscaling of Pods, but it is outside of the scope of this tutorial. Scaling to zero is also possible, and it will terminate all Pods of the specified Deployment.
Running multiple instances of an application will require a way to distribute the traffic to all of them. Services have an integrated load-balancer that will distribute network traffic to all Pods of an exposed Deployment. Services will monitor continuously the running Pods using endpoints, to ensure the traffic is sent only to available Pods.
Scaling is accomplished by changing the number of replicas in a Deployment.
Once you have multiple instances of an Application running, you would be able to do Rolling updates without downtime. We'll cover that in the next module. Now, let's go to the online terminal and scale our application.

Update number of replicas from 3 to 5 deploy the application using. Update hello-kubernetes.yml and change number of replicas from 3 to 5. Deploy updated changes
```
Kubectl apply -f hello-kubernete.yaml 
```

Above script will increase pods from 3 to 5 replicas (pods). Lets try to delete one of POD and see if it comes automatically:

```
kubectl get pods
Kubectl delete pod ${POD_NAME}
```
We can see number of pods after some time, and number of pods will be same as before. Replication controller brought third pod back to previous state.

Manually Scale up existing deployment
```
Kubectl scale --replicas=6 -f hello-kubernete.yaml

```

Manually Scale down 
```
Kubectl scale --replicas=1 -f hello-kubernetes.yaml
```
You can only horizontally scale applications if they are stateless (No local data and session storage).
Data is stored on persistent volumes or inside a database.

Delete the deployment
```
kubectl delete -f hello-kubernetes.yaml
```

### Rolling updates and Rollback your existing deployment

#### Rolling updates: 
Rolling updates allow Deployments’ update to take place with zero downtime by incrementally updating Pods instances with new ones. Rolling updates would not be enabled by default in Kubernetes. We need to configure rolling update and rolling strategy to make zero downtime deployments.

Users expect applications to be available all the time and developers are expected to deploy new versions of them several times a day. In Kubernetes this is done with rolling updates. Rolling updates allow Deployments’ update to take place with zero downtime by incrementally updating Pods instances with new ones. The new Pods will be scheduled on Nodes with available resources.

Let’s take a simple deployment manifest.

```
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
  ```
  
  This should work fine when you execute the following command and a deployment called hello-kubernetes will be created.
  
  ```
  kubectl apply -f hello-kubernetes.yml
  ```
  
But imagine you want to update the image running in the above deployment? Then you probably have to do a kubectl edit or   just edit the yaml file as following and re-deploy using kubectl apply -f
 
 Let’s see the edited yaml file (version 2.0 of image)
 
 ```
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
        image: aveevadevopsr/hello-kubernetes:2.0
        ports:
        - containerPort: 8080
  ```
  
Once you apply this or edit this, notice that there maybe a little downtime on your application because the old pods are getting terminated and the new ones are getting created. You can easily notice a downtime if you open a new tab on your terminal and run a curl command your exposed service every second.

This happens because kubernetes doesn’t know when your new pod is ready to start accepting requests, so as soon as your new pod gets created, the old pod is terminated without waiting to see if all the necessary services, processes have started in the new pod which would then enable it to receive requests.

To do this, Kubernetes provide a config option in deployment called Readiness Probe. Readiness Probe makes sure that the new pods created are ready to take on requests before terminating the old pods. To enable this, first you need to have a route in whatever the application you want to run which would return a 200 on an HTTP GET request. (Note: you can have other HTTP request methods as well, but for this post, I’m sticking with GET method).

```
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
        image: aveevadevopsr/hello-kubernetes:2.0
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
             path: /
             port: 8080
             initialDelaySeconds: 5
             periodSeconds: 5
             successThreshold: 1
  ```
Don't worry about Readiness Probe, I will explain this in next chapter in HealthCheck section.

Another thing we should add is something called RollingUpdate strategy and it can be configured as follows.

```
strategy:
  type: RollingUpdate
  rollingUpdate:
     maxUnavailable: 25%
     maxSurge: 1
```

The above specifies the strategy used to replace old Pods by new ones. The type can be “Recreate” or “RollingUpdate”. “RollingUpdate” is the default value.

##### maxUnavailable: 
is an optional field that specifies the maximum number of Pods that can be unavailable during the update process. The value can be an absolute number (for example, 5) or a percentage of desired Pods (for example, 10%). The absolute number is calculated from percentage by rounding down. The value cannot be 0 if maxSurge is 0. The default value is 25%.

##### maxSurge 
is an optional field that specifies the maximum number of Pods that can be created over the desired number of Pods. The value can be an absolute number (for example, 5) or a percentage of desired Pods (for example, 10%). The value cannot be 0 if MaxUnavailable is 0. The absolute number is calculated from the percentage by rounding up. The default value is 25%.

With all the above, configurations, Let’s take a look at the final deployment file.

```

apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-kubernetes
spec:
  replicas: 3
  strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 25%
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
        image: aveevadevopsr/hello-kubernetes:2.0
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
             path: /
             port: 8080
             initialDelaySeconds: 5
             periodSeconds: 5
             successThreshold: 1
```

Applying this deployment manifest will ensure rolling updates without any downtime. 

#### Rollback your existing deployment

Step 1: Get deployment details and note down name of deployment you need to rollback

```
kubectl get deployment
```
Check status of deployment 

```
kubectl rollout status deployment/${deployment_name}
kubectl rollout status deployment/hello-kubernetes
```

Get deployment history
```
kubectl rollout history deployment/${deployment_name}
kubectl rollout history deployment/hello-kubernetes
```
Update deployment to new image. Below command will update deployment to version 2.0
```
kubectl set image deployment/${deployment_name} ${container_name}:${image_name}:${image_version}
kubectl set image deployment/hello-kubernetes hello-kubernetes:aveevadevopsr/hello-kubernetes:2.0
```
Check rollout history to check the updated version
```
kubectl rollout history deployment/hello-kubernetes
```
Check updated deployment, it should use the 2.0 image now
```
kubectl describe deployment/${deployment_name}
```

Rollback to previous version of deployment
```
kubectl rollout undo deployment/${deployment_name}
kubectl describe deployment/${deployment_name}
```
Rollback to a specific previous version

```
Kubectl rollout history deployment/hello-kubernetes
kubectl rollout undo deployment/${deployment_name} --to-revision=1
```

By Default kubernetes keeps 2 versions, if we need to go back to later versions, then we need to edit history limit
```
Kubectl edit deployment/${deployment_name}
```
and edit to ddd revisionHistoryLimit:100

Get deployment info

This wraps up the third chapter !!
