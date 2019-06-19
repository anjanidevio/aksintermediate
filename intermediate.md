# Intermediate Lab

## Table of Contents

[Overview](#overview)

[Practice 1: Rolling Update with Kubernetes Deployments and Volumes](#practice-1-rolling-update-with-kubernetes-deployments-and-volumes)

[Practice 2: Ingress controller and Ingresses](#practice-2-ingress-controller-and-ingresses)

[Practice 3: Job and Daemonset](#practice-3-job-and-daemonset)

[Practice 4: LimitRange and Horizontal Pod Autoscaler](#practice-4-limitrange-and-horizontal-pod-autoscaler)

## Overview:

**Rolling Update**

A Kubernetes Deployment owns and manages one or more **Replica Sets**, and **Replica Set** manages the basic units in Kubernetes Pods. Kubernetes creates a new Replica Set each time after the new Deployment config is deployed and keeps the old Replica Set. So that we can rollback to the previous state with old Replica Set. And there is only one Replica Set is in active state, which means DESIRED > 0. Kuberentes Deployments also supports rollback support.

**Volumes**

A Kubernetes volume is a directory accessible to all containers running in a pod. Firstly The files present inside a container are short-lived, which can lead to issues in applications when running in containers where persistence is important. For some reason if a container crashes the kubelet will restart the container, but the files in the containers are lost ie the container starts with a clean slate. Secondly, when running containers together in a pod it is often neccessary to share files between those containers. Kubernetes Volumes solves both of these problems.

**Ingress controller**

Ingress controller is a necessary kubernetes feature which plays a vital role in functioning of **Ingress** resource. The Ingress resources deployed in the cluster are controlled by the ingress controller. Unlike other types of controllers which run as part of the ***kube-controller-manager*** binary, Ingress controller are not started automatically when a cluster is created. It must be deployed into the cluster manually and configured as per one's requirements.

**Ingresses**

Ingress is a kubernetes object that allows access to your kubernetes services from the outside of the kubernetes cluster. You can configure access by creating a set of rules that defines which inbound internet traffic must reach which kubernetes service in the cluster.

**Job**

Job is a type of kubernetes object which runs a task or a job/script inside one or more pods and ensures that a specified number of them successfully terminate. As pods finish a task described in it successfully, the Job object tracks the completion and terminates the pod. Jobs when deleted cleans up the pods it created. One can also use Job to run multiple pods in parallel which runs for the duration of the task completion. Jobs can be compared to cronjobs on vms.


**DaemonSets**

A DaemonSet is a kubernetes object which ensures every node present in the cluster runs a Pod. Whenever new nodes are added to the cluster, a pod is added to that node. As the nodes are removed from the cluster, pods are deleted and garbage collected. Once the DaemonSet is deleted the pods scheduled by it also gets deleted.

**LimitRange**

Assigning a memory request and a memory limit or a cpu request and a cpu limit to a container is called LimitRange.
A Container is guaranteed to have as much memory as it requests, but is not allowed to use more memory than its limit.

**Note** : Makesure you have the metrics server installed in your cluster before applying LimitRanges.

**Horizontal Pod Autoscaler**

Horizontal Pod Autoscaler scales the number pods in a deployment, replica controller or replica set automatically based on the CPU utilization. ( or other custom metric )


## Practice 1: Rolling Update with Kubernetes Deployments and Volumes

### Rolling Update

A Kubernetes Deployment owns and manages one or more **Replica Sets**, and **Replica Set** manages the basic units in Kubernetes Pods. Kubernetes creates a new Replica Set each time after the new Deployment config is deployed and keeps the old Replica Set. So that we can rollback to the previous state with old Replica Set. And there is only one Replica Set is in active state, which means DESIRED > 0. Kuberentes Deployments also supports rollback support.

### step 1: Create a sample Kubernetes Deployment

Let’s create a Deployment with the following deployment YAML file.

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-sample
  labels:
    service-name: deployment-sample
    environment: deployment-sample
spec:
  replicas: 3
  selector:
    matchLabels:
      service-name: deployment-sample
      environment: deployment-sample
  template:
    metadata:
      labels:
        service-name: deployment-sample
        environment: deployment-sample
    spec:
      containers: 
        - image:  ashorg/sample:v1
          name: deployment-sample
          resources:
          ports:
            - containerPort: 8089
```

You can use the command `kubectl -n <namespace-name> create -f <filename>` to create the deployment.

`kubectl -n nodejs create -f <filename.yaml>`
`deployment.apps "deployment-sample" created`

Now get the deployment using the command 

`kubectl -n <namespace-name> get deployment`

```
NAME                DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment-sample   3         3         3            3           20s
```

Now get the pods that got created as part of this deployment using the command 

`kubectl -n <namespace-name> get pods`

```
NAME                                 READY     STATUS    RESTARTS   AGE
deployment-sample-6b44657b4f-5vdnk   1/1       Running   0          6m
deployment-sample-6b44657b4f-gz2dv   1/1       Running   0          6m
deployment-sample-6b44657b4f-xwlpt   1/1       Running   0          6m
```

Now get the Replica Sets that got created as part of the deployment using the command 

`kubectl -n <namespace-name> get replicaset`

```
NAME                           DESIRED   CURRENT   READY     AGE
deployment-sample-6b44657b4f   3         3         3         17m
```

### step 2: Apply Rolling Update

In order to support rolling update, we need to configure the update strategy first.

So we add following part into **spec**

``` yaml
minReadySeconds: 5
strategy:
  # indicate which strategy we want for rolling update
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1
```

#### minReadySeconds

- **minReadySeconds** is the bootup time of your application, kubernetes waits specific time till the next pod creation.
- Kubernetes assumes that your application is available once the pod is created by default.
- If you leave this field empty, the service may be unavailable after the update process because all the application pods are not ready yet.

#### maxSurge

- **maxSurge** is number of pods that can get provisioned more than the desired number of Pods. 
- This field can be an absolute number or the percentage.
- For instance, if **maxSurge: 1** means that there will be at most 4 pods during the update process if replicas is set to 3

#### maxUnavailable

- **maxUnavailable** is the amount of pods that can be unavailable during the update process
- this fields can be a absolute number or the percentage.
- this field cannot be zero if **maxSurge** is set to 0
- For instance, if **maxUnavailable: 1** means that there will be at most 1 pod unavailable during the update process


The final deployment yaml file will look like

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-sample
  labels:
    service-name: deployment-sample
    environment: deployment-sample
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  minReadySeconds: 5
  selector:
    matchLabels:
      service-name: deployment-sample
      environment: deployment-sample
  template:
    metadata:
      labels:
        service-name: deployment-sample
        environment: deployment-sample
    spec:
      containers: 
        - image:  ashorg/sample:v1
          name: deployment-sample
          resources:
          ports:
            - containerPort: 8089
```

Now use the command `kubectl -n <namespace-name> apply -f <filename>` to apply the Rolling Update change to the Deployment.


### step 3: Editing the deployment 

Now, for example, if we want to update the docker image, we have three ways to perform the rolling update.

#### CLI

One can update the docker image of a deployment using the command

`kubectl -n <namespace-name> set image deployment <deployment> <container>=<image> --record`

**For Example:**

`kubectl -n nodejs set image deployment deployment-sample deployment-sample=ashorg/sample:v2 --record`
`deployment.apps "deployment-sample" image updated`

#### Replace

One can update the docker image by directly modifying the YAML file

```
 containers: 
    # newer image version
  - image:  ashorg/sample:v2
    name: deployment-sample
    resources:
    ports:
      - containerPort: 8089
```

Use the command `kubectl -n <namespace-name> replace -f <filename> --record`

#### Edit 

One can edit the docker image in the currently running deployment by using the command `kubectl -n <namespace-name> edit deployment <deployment-name> --record`. This will open the current deployment in a text editor, change the image and save the file. Make sure you follow the YAML syntax properly.

### step 4: Checking the Status of the Rolling Update

Check the list of pods for that deployment using the command `kubectl -n nodejs get pods` after you edit the deployment. As you can see below it has created extra pods as mentioned in maxSurge.

```
NAME                                 READY     STATUS        RESTARTS   AGE
deployment-sample-6b44657b4f-89mn6   1/1       Running       0          15s
deployment-sample-6b44657b4f-pzs7b   1/1       Running       0          15s
deployment-sample-b4db4bc54-hpswc    1/1       Running       0          14m
deployment-sample-b4db4bc54-v9z5j    1/1       Running       0          14m
deployment-sample-b4db4bc54-x2ll2    0/1       Terminating   0          13m
```

Now Let's check the status of the Rolling Update, we use the command `kubectl -n <namespace-name> rollout status deployment <deployment-name>`

**For Example:**

`kubectl -n nodejs rollout status deployment deployment-sample`
`deployment "deployment-sample" successfully rolled out`

### step 5: Pause Rolling Update

To pause a rolling update use the command `kubectl -n <namespace-name> rollout pause deployment <deployment-name>`

### step 6: Resume Rolling Update

To resume a rolling update use the command `kubectl -n <namespace-name> rollout resume deployment <deployment-name>`

### step 7: Rollback the changes to previous revision

To rollback the changes use the command `kubectl -n <namespace-name> rollout undo deployment <deployment-name>` 

**For Example:**

`kubectl -n nodejs rollout undo deployment deployment-sample` 
`deployment.apps "deployment-sample"`


 ### Volumes

A Kubernetes volume is a directory accessible to all containers running in a pod. Firstly The files present inside a container are short-lived, which can lead to issues in an important applications when running in containers. For some reason if a container crashes the kubelet will restart the container, but the files in the containers are lost ie the container starts with a clean state. Secondly, when running containers together in a pod it is often neccessary to share files between those containers. The Kubernetes Volumes solves both of these problems.

In this session we mainly focus on **hostPath** type of Volume

### step 1: HostPath

A hostPath volume mounts a directory from the host node's filesystem into the pod. For instance, some cases for a hostPath are

- Running a Container that needs to access data like docker internals; use a hostPath of /var/lib/docker

### step 2: Creating a Pod with Volume

Use the below code to create a YAML file and create a sample deployment with a **hostPath** type volume

``` yaml


apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deployment
  labels:
    app: test-app
spec:
  selector:
    matchLabels:
      app: test-app
  replicas: 1
  template:
    metadata:
      labels:
        app: test-app
    spec:
      containers:
      - name: test-container
        image: k8s.gcr.io/test-webserver
        volumeMounts:
        - mountPath: /test-pod
          name: test-volume
        ports:
        - containerPort: 80
      volumes:
        - name: test-volume
          hostPath:
            path: /usr/src/data
            type: Directory
```

Use the command `kubectl -n <namespace-name> create -f <filename>`, this will create a deployment which will provision a pod with hostPath that is mounted on to its container. 

As per above example the hostPath **/usr/src/data** in the node is mounted inside the container at **/test-pod** directory.

Thus you have successfully created a deployment with volume attached to it.

## Practice 2: Ingress controller and Ingresses
  
**Ingress controller**

Ingress controller is a necessary kubernetes feature which plays a vital role in functioning of **Ingress** resource. The Ingress resources deployed in the cluster are controlled by the ingress controller. Unlike other types of controllers which run as part of the ***kube-controller-manager*** binary, Ingress controller are not started automatically when a cluster is created. It must be deployed into the cluster manually and configured as per one's requirements.

**Use of multiple Ingress controllers**

One can deploy any number of ingress controllers within a cluster. When one creates an Ingress resource, one must annotate each Ingress resource with the appropriate **ingress.class** to indicate which Ingress Controller to use that exists within your cluster.

**NGINX Ingress Controller for Kubernetes**

The NGINX Ingress Controller for Kubernetes provides enterprise‑grade delivery of services for Kubernetes applications, with benefits of using open source NGINX. 

ingresscontroller created default with aks cluster in kube-system namespace 

`kubectl -n kube-system get all`

```
NAME                                                      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
addon-http-application-routing-default-http-backend       1         1         1            1           12d
addon-http-application-routing-external-dns               1         1         1            1           12d
addon-http-application-routing-nginx-ingress-controller   1         1         1            1           12d

```
### step 1: Get the pods of the Ingress Controller

The pods in **kube-system** namespace belong to the Ingress Controller deployments. One can view these pods using the command

`kubectl -n kube-system get pods`

```
NAME                                       READY     STATUS    RESTARTS   AGE
default-http-backend-587b7d64b5-hxk87      1/1       Running   0          2d
nginx-ingress-controller-cd8cb99f5-btdxz   1/1       Running   0          2d
```

### step 2: Get the services of the Ingress Controller

The ingress controller services in ingress-nginx namespace must be of type LoadBalancer so that it uses public Cloud specific Load Balancer resource and Internet traffic enters the cluster through it. One can view the ingress controller service using the command.

  `kubectl -n kube-system  get service`

  ```
  NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                      AGE
  default-http-backend   ClusterIP      10.96.49.199   <none>           80/TCP                       39d
  ingress-nginx          LoadBalancer   10.96.127.87   129.146.208.43   80:30735/TCP,443:30314/TCP   39d
  ```

Notice that the **ingress-nginx service** is of type **LoadBalancer** and has an External IP which is the IP associated to the Load Balancer resource that gets deployed in the public cloud.

**Ingresses**

Ingress is a kubernetes object that allows access to your kubernetes services from the outside of the kubernetes cluster. You can configure access by creating a set of rules that defines which inbound internet traffic must reach which kubernetes service in the cluster.

This lets you consolidate your routing rules into a single resource. For instance, you might want to send requests to **helloworld.com/service/v1** to the **service-1** and requests to **helloworld.com/service/v2** to the **service-2**. With an Ingress, you can easily set this up without creating a bunch of LoadBalancer type service or exposing each service on the node using NodePort type service.

For Ingress resources to work within a cluster you must deploy an Ingress Controller prior to deploying an Ingress. The Ingress controller allows the Internet traffic to enter in the kubernetes cluster. Once it is setup correctly we can create Ingress resources in the cluster and route internet traffic to the services. Note that Ingress resources are namespace specific

### step 1: Create an Ingress

First let's create a service to demonstrate how the Ingress resource routes the request to the service. We have taken a sample nodejs service and deployment as shown below

``` yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nodejs
  labels:
    app: nodejs
    nodejs: nodejs

---
apiVersion: v1
kind: Service
metadata:
  name: nodejs
  namespace: nodejs
  labels:
    service-name: nodejs
spec:
  ports:
    - port: 8089
  selector:
    service-name: nodejs
    
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs
  namespace: nodejs
  labels:
    service-name: nodejs
    environment: nodejs
spec:
  replicas: 1
  selector:
    matchLabels:
      service-name: nodejs
      environment: nodejs
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        service-name: nodejs
        environment: nodejs
    spec:
      imagePullSecrets:
      - name: regcred
      containers: 
        - image:  ashorg/sample:v1
          imagePullPolicy: Always
          name: nodejs
          ports:
          - containerPort: 8089

```
Use the above code to deploy a sample nodejs application 
 
`kubectl apply -f nodejs.yaml`

Now, Let's create an Ingress to route traffic to nodejs service. Use the following code to create a Ingress yaml and deploy the Ingress into the cluster. 

``` yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nodejs-ingress
  annotations:
    # nginx.ingress.kubernetes.io/rewrite-target: /
    # kubernetes.io/ingress.class: addon-http-application-routing
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    nginx.ingress.kubernetes.io/cors-allow-credentials: "true"
    nginx.ingress.kubernetes.io/cors-allow-methods: "*"
spec:
  rules:  
  - host: nodejs.qld.com
    http:
      paths:
      - backend:
          serviceName: nodejs
          servicePort: 8089
```

Thus you have successfully deployed an Ingress resource.

`kubectl -n <namespace-name> apply -f ingress.yaml`

### step 2 : Get the list of Ingress

To get the list of Ingress resources in a namespace use the command.

`kubectl -n <namespace-name> get ingress`

```
NAME             HOSTS            ADDRESS          PORTS     AGE
nodejs-ingress   nodejs.qld.com   129.146.208.43   80        5d
```

Now add the host **nodejs.qld.com** with the Public IP address **129.146.208.43** in the hosts file (/etc/hosts)

Once you have done the above steps access the host url defined in the ingress yaml, in this case
http://nodejs.qld.com/print/{input} (Replace {input} with any string) like http://nodejs.qld.com/print/hello and you must see the respose as follows.

```
{"message":"u have given hello"}
```

### step  3: Describe an Ingress

To describe an Ingress resource in a namespace use the command

`kubectl -n <namespace-name> describe Ingress <ingress-name>`

```
Name:             nodejs-ingress
Namespace:        nodejs
Address:          129.146.208.43
Default backend:  default-http-backend:80 (<none>)
Rules:
  Host            Path  Backends
  ----            ----  --------
  nodejs.qld.com
                     nodejs:8089 (<none>)
Annotations:
  nginx.ingress.kubernetes.io/cors-allow-origin:       *
  nginx.ingress.kubernetes.io/enable-cors:             true
  kubernetes.io/ingress.class:                         nginx
  nginx.ingress.kubernetes.io/cors-allow-credentials:  true
  nginx.ingress.kubernetes.io/cors-allow-methods:      *
Events:                                                <none>
```

### step  4: Edit an Ingress

To edit an Ingress resource in a namespace use the command

`kubectl -n <namespace-name> edit ingress <ingress-name>`

The currently deployed Ingress resource YAML will be opened in a text editor and you can edit as per the requirement then save the file, this will update the Ingress resource successfully. Make sure the YAML syntax is followed.

### step  5: Delete an Ingress

To delete an Ingress resource in a namespace use the command

`kubectl -n <namespace-name> delete ingress <ingress-name>` 

This will delete the ingress resource.

## Practice 3: Job and Daemonset

**Job**

Job is a type of kubernetes object which runs a task or a job/script inside one or more pods and ensures that a specified number of them successfully terminate. As pods finish a task described in it successfully, the Job object tracks the completion and terminates the pod. Jobs when deleted cleans up the pods it created. One can also use Job to run multiple pods in parallel which runs for the duration of the task completion.

### step 1: Create the Job resource

To create a Job one must describe a Job in a YAML file. For example, one can find a sample Job code below which you can use to deploy a job in one's cluster.

``` yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-sample
spec:
  template:
    spec:
      containers: 
        - image:  ashorg/sample:jobsleep
          name: job-sample
      restartPolicy: Never
  backoffLimit: 4
```

### step 2: Required Fields

Jobs can be described using YAML files, some of the required fields in the Job YAML file are **apiVersion**, **kind**, **metadata**. A Job also requires a **.spec** section. Under the **.spec**  the required fields are **.spec.template** and **.spec.selector**


Create a yaml file using the code shown above, and use the command below to create a Job.

`kubectl -n <namespace_name> apply -f <file-name>`
`job.batch "job-sample" created`

### step 3: Get the Job resource

To view the created job use the command 

`kubectl -n <namespace_name> get job <job-name>`

```
NAME         DESIRED   SUCCESSFUL   AGE
job-sample   1         1            17m
```

### step 4: Describe the Job resource

One can also describe a Job using the command

`kubectl -n <namespace_name> describe job <job-name>`

```
Name:           job-sample
Namespace:      nodejs
Selector:       controller-uid=e0b12da8-6d85-11e9-a023-0a580aed6956
Labels:         environment=job-sample
                Job-name=job-sample
Annotations:    <none>
Parallelism:    1
Completions:    1
Start Time:     Fri, 03 May 2019 14:58:58 +0530
Pods Statuses:  0 Running / 1 Succeeded / 0 Failed
Pod Template:
  Labels:  controller-uid=e0b12da8-6d85-11e9-a023-0a580aed6956
           environment=job-sample
           job-name=job-sample
           Job-name=job-sample
  Containers:
   job-sample:
    Image:        ashorg/sample:jobsleep
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From            Message
  ----    ------            ----  ----            -------
  Normal  SuccessfulCreate  20m   job-controller  Created pod: job-sample-2dggf
```

### step 5: Get the Pods created by the Job

To view the list of pods created by job use the command

`kubectl -n <namespace_name> get pods -o wide`

```
NAME               READY     STATUS    RESTARTS   AGE       IP            NODE
job-sample-mzhbt   1/1       Running   0          3s        10.244.2.88   10.0.2.2
```

Here one can see that the pods has completed its task and the job is succesffully completed with pod showing the status as **Completed**.

`kubectl -n <namespace_name> get pods -o wide`

```
NAME               READY     STATUS      RESTARTS   AGE       IP            NODE
job-sample-mzhbt   0/1       Completed   0          3m        10.244.2.88   10.0.2.2
```

Now you have successfully deployed a Job and the pods of that job has completed the task successfully.

### step 6: Edit a Job

To edit a Job resource in a namespace use the command `kubectl -n <namespace-name> edit job <Job-name>`

The currently deployed job resource YAML will be opened in a text editor and you can edit as per the requirement then save the file, this will update the job resource successfully. Make sure the YAML syntax is followed.


### step 7: Delete a Job

To delete the Job use the command

  `kubectl -n <namespace_name> delete job <job-name>`
  `job.batch "job-sample" deleted`

This will successfully delete the pods that got created as part of the Job and the Job resource itself.

**DaemonSets**

A DaemonSet is a kubernetes object which ensures every node present in the cluster runs a Pod. Whenever new nodes are added to the cluster, a pod is added to that node. As the nodes are removed from the cluster, pods are deleted and garbage collected. Once the DaemonSet is deleted the pods scheduled by it also gets deleted.

**Usecases of DaemonSets**

- One typical usecase would be running a DaemonSet that monitors the nodes such as Prometheus.

### step 1: Create a DaemonSet

To create a DaemonSet one must describe a DaemonSet in a YAML file. For instance, one can find a sample daemonSet code below  which can be used to deploy a daemonset in a cluster.

``` yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: daemonset-sample
  labels:
    service-name: daemonset-sample
    environment: daemonset-sample
spec:
  selector:
    matchLabels:
      service-name: daemonset-sample
      environment: daemonset-sample
  template:
    metadata:
      labels:
        service-name: daemonset-sample
        environment: daemonset-sample
    spec:
      containers: 
        - image:  ashorg/sample:v1
          name: daemonset-sample
          resources:
          ports:
            - containerPort: 8089
```

### step 2: Required Fields

DaemonSet can be described using YAML files, some of the required fields in the DaemonSet YAML file are **apiVersion**, **kind**, **metadata**. A DaemonSet also requires a **.spec** section. Under the **.spec**  the required fields are **.spec.template** and **.spec.selector**.

Create a yaml file using the code shown above, and use the command below to create a DaemonSet.

`kubectl -n <namespace_name> apply -f <file-name>`

### step 3: Get a DaemonSet resource

To view the created daemon set use the command

`kubectl -n <namespace_name> get ds <daemonSet-name>`

```
NAME                DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset-sample    3         3         3         3            3           <none>          1h
```
### step 4: Describe a DaemonSet resource

One can also describe a daemon set using the following command

`kubectl -n <namespace_name> describe ds <daemonSet-name>`
    
```
Name:           daemonset-sample
Selector:       environment=daemonset-sample,service-name=daemonset-sample
Node-Selector:  <none>
Labels:         environment=daemonset-sample
                service-name=daemonset-sample
Annotations:    <none>
Desired Number of Nodes Scheduled: 3
Current Number of Nodes Scheduled: 3
Number of Nodes Scheduled with Up-to-date Pods: 3
Number of Nodes Scheduled with Available Pods: 3
Number of Nodes Misscheduled: 0
Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  environment=daemonset-sample
           service-name=daemonset-sample
  Containers:
   daemonset-sample:
    Image:        ashorg/sample:v1
    Port:         8089/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From                  Message
  ----    ------            ----  ----                  -------
  Normal  SuccessfulCreate  30s   daemonset-controller  Created pod: daemonset-sample-2f95p
  Normal  SuccessfulCreate  30s   daemonset-controller  Created pod: daemonset-sample-88bvt
  Normal  SuccessfulCreate  30s   daemonset-controller  Created pod: daemonset-sample-7blxh
```

### step 5: Get the pods created by the DaemonSet
  
To view the list of pods created by the DaemonSet use the command

`kubectl -n <namespace_name> get pods -o wide`
    
One must see list of pods as follows 

```
NAME                     READY     STATUS    RESTARTS   AGE       IP            NODE
daemonset-sample-2f95p   1/1       Running   0          20m       10.244.1.80   10.0.0.2
daemonset-sample-88bvt   1/1       Running   0          20m       10.244.0.61   10.0.1.2
daemonset-sample-7blxh   1/1       Running   0          18m       10.244.2.83   10.0.2.2
```

Here one can see that the pod has been created in all the node (For this example a 3 node cluster is used.)

Now let's delete one of the pods of DaemonSet and observe what happens

`kubectl -n <namespace_name> delete pod daemonset-sample-7blxh`
`pod "daemonset-sample-7blxh" deleted`

Let's list the pods again

`kubectl -n <namespace_name> get pods -o wide`

```
NAME                     READY     STATUS        RESTARTS   AGE       IP            NODE
daemonset-sample-2f95p   1/1       Running       0          1h        10.244.1.80   10.0.0.2
daemonset-sample-88bvt   1/1       Running       0          1h        10.244.0.61   10.0.1.2
daemonset-sample-7blxh   0/1       Terminating   0          1h        10.244.2.83   10.0.2.2
```

One can see that the pod is started again in the same node with AGE as 1m

```
NAME                     READY     STATUS    RESTARTS   AGE       IP            NODE
daemonset-sample-2f95p   1/1       Running   0          1h        10.244.1.80   10.0.0.2
daemonset-sample-88bvt   1/1       Running   0          1h        10.244.0.61   10.0.1.2
daemonset-sample-7blxh   1/1       Running   0          1m        10.244.2.84   10.0.2.2
```

Now you have successfully deployed a DaemonSet and tested it.

### step 6: Edit a DaemonSet

To edit a DaemonSet resource in a namespace use the command `kubectl -n <namespace-name> edit daemonset <DaemonsSet-name>`

The currently deployed DaemonSet resource YAML will be opened in a text editor and you can edit as per the requirement then save the file, this will update the DaemonSet resource successfully. Make sure the YAML syntax is followed.

### step 7: Delete a DaemonSet

To delete the DaemonSet which you have created use the command

`kubectl -n <namespace_name> delete daemonset <daemonset-name>`
`daemonset.extensions "nodejs" deleted`

Now you have deleted the daemonset successfully.

## Practice 4: LimitRange and Horizontal Pod Autoscaler

### LimitRange

Assigning a memory request and a memory limit or a cpu request and a cpu limit to a container is called LimitRange.
A Container is guaranteed to have as much memory as it requests, but is not allowed to use more memory than its limit.

**Note** : Makesure you have the metrics server installed in your cluster before applying LimitRanges.

### step 1: LimitRange

### Create LimitRange

Create a LimitRange to a namespace then all the pods that exists in that namespace will get applied by the LimitRange.

1. Create a namespace using `kubectl create ns test`

2. Create and apply the below limitrange to the namespace using `kubectl create -f <file path>`

```  yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-memory
  namespace: test
spec:
  limits:
  - default:
      cpu: 250m
      memory: 300Mi
    defaultRequest:
      cpu: 100m
      memory: 200Mi
    type: Container

```

### Deploy a simple pod and service

1. Let's deploy a pod and service that creates a single container to demonstrate how default values are applied to each pod.

    ` kubectl -n test run php-apache --image=k8s.gcr.io/hpa-example --expose --port=80`

2. Get the pods and service using `kubectl -n test get pods` and `kubectl -n test get services`

```
NAME                          READY     STATUS    RESTARTS   AGE
php-apache-55c4bb8b88-bb7jp   1/1       Running   0          4m

```

```
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
php-apache   ClusterIP   10.96.225.250   <none>        80/TCP    4m

```

### Get the configuration of the pod

1. Now get the configuration of the pod using `kubectl -n test get pod <podname> -o yaml`

```
....................
....................
  containers:
  - image: k8s.gcr.io/hpa-example
    imagePullPolicy: Always
    name: php-apache
    ports:
    - containerPort: 80
      protocol: TCP
    resources:
      limits:
        cpu: 250m
        memory: 300Mi
      requests:
        cpu: 100m
        memory: 200Mi
...................
...................
...................

```
2. Get the metrics of the pods in your namespace using `kubectl top pods -n test`.

```
NAME                          CPU(cores)   MEMORY(bytes)
php-apache-55c4bb8b88-bb7jp   1m           9Mi

```

### step 2: Horizontal Pod Autoscaler

Horizontal Pod Autoscaler scales the number pods in a deployment, replica controller or replica set automatically based on the CPU utilization. ( or other custom metric )

### Apply Horizontal Pod Autoscaling

1. Apply the HPA configuartion to the existing deployment.

  ` kubectl -n test autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10`

2. Get the HPA configuration using `kubectl -n test get hpa`

```
  NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
  php-apache   Deployment/php-apache   1%/50%    1         10        1          10m

```

### Geberate Load

1. Now we will use load generator to generate some load on apache.

2. Open an additional terminal window and run the below command.

  `kubectl -n test run -i --tty load-generator --image=busybox /bin/sh`

3. Hit enter and run below command to generate load on apache.

  `while true; do wget -q -O- http://php-apache.test.svc.cluster.local; done `

  Output shoul be : `OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!`

4. Within few minutes, we should see the higher CPU load by executing `kubectl -n test get hpa`

```
NAME         REFERENCE               TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   158%/50%   1         10        4          16m
```
5. You can see the number of apache pods increased due to load on apache using `kubectl -n test get pods`

```
NAME                              READY     STATUS    RESTARTS   AGE
load-generator-5ff6784f85-7wgnm   1/1       Running   0          9m
php-apache-55c4bb8b88-2j2nl       1/1       Running   0          7m
php-apache-55c4bb8b88-bp5mf       1/1       Running   0          5m
php-apache-55c4bb8b88-jx5qr       1/1       Running   0          7m
php-apache-55c4bb8b88-kc68r       1/1       Running   0          52m
php-apache-55c4bb8b88-m4hzb       1/1       Running   0          5m
php-apache-55c4bb8b88-trvrm       1/1       Running   0          7m
```
### Stop Load

1. You can stop the load on apache by typing <Ctrl> + C on new terminal.

2. You can verify the result within a minute using `kubectl -n test get hpa`.

```
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   44%/50%   1         10        6          24m
```

**conclusion**

Congratulations! You have successfully completed the Kubernetes beginners lab. In this lab, you created Rolling Update, Volumes,Ingress controller, Ingresses, Job, DaemonSets, Limitrange, Horizontal Pod Autoscaler.

Feel free to continue exploring or start a new lab.

Thank you for taking this training lab!
