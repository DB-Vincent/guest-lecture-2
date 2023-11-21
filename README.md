# Autoscaling and Load balancing in Kubernetes

This repository contains all the source files for the second DXC guest lecture at Thomas More.

We'll delve into the world of scalability and efficient load distribution in Kubernetes. Our focus will be on two crucial aspects: Autoscaling through the HorizonalPodAutoscaler, enabling our applications to flexibly adapt to changing workloads, and Load Balancing with services (and MetalLB), ensuring that the load gets distributed to the various replica's. 

Throughout this guest lecture, we'll be working with Kubernetes manifests, which are YAML files defining resources.
Keep a record of your manifest files after each step! For example, create a new directory and copy the required manifests as a starting point for the next step.
Note that you can update the kubernetes resources by applying manifests of the same kind and same name.

---

## Table of contents

- [Step 1: setting up a pod](#step-1-setting-up-a-pod)
- [Step 2: creating replica's for our pod](#step-2-creating-replicas-for-our-pod)
- [Step 3: accessing the deployments using a service](#step-3-accessing-the-deployments-using-a-service)
- [Step 4: identifying which pod we're accessing](#step-4-identifying-which-pod-were-accessing)
- [Step 5: automatically scale a Deployment](#step-5-automatically-scale-a-deployment)
  - [Step 5a: installing metrics-server](#step-5a-installing-metrics-server)
  - [Step 5b: setting up horizontal scaling for our Deployment](#step-5b-setting-up-horizontal-scaling-for-our-deployment)
  - [Step 5c: generating load and seeing the HPA in action](#step-5c-generating-load-and-seeing-the-hpa-in-action)
- [Bonus: using MetalLB to make our application available outside the cluster](#bonus-using-metallb-to-make-our-application-available-outside-the-cluster)
  - [Installation](#installation)
  - [Configuration](#configuration)

## Step 1: setting up a pod

**[`^ back to top ^`](#autoscaling-and-load-balancing-in-kubernetes)**

Let's start off with the basics. Before we can create Deployments in our cluster, we'll first create a single pod. For a Pod, a Kubernetes manifest file would look something like this:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: some-cool-pod-name
  # Defines the labels for the Pod
  labels:
    a-cool-label-name: a-cool-label-value
spec:
  containers:
  - name: a-cool-container-name
    image: container-image:image-tag
```

Now, change the example above to match the following settings:
- The Pod's name should be "demo-app"
- The container's name should be "nginx"
- We'll be using the nginx:latest container image for our Pod.
- Add a ports section that sets the container port to 80.
- Change the label so you have a label with the key "app" and the value "demo".

Once your Pod manifest has been adjusted, apply it using the `kubectl apply -f <filename>` command. You should now see your pod starting up:

```shell
[vdeborger@node-01 ~]$ kubectl get pods demo-app
NAME       READY   STATUS              RESTARTS   AGE
demo-app   0/1     ContainerCreating   0          3s
```

After some time (usually 30 seconds to a minute), the container image should be pulled and your container should be in the "Running" status. If that's the case, great news, you can remove the Pod and continue to step 2! If that's not the case, it's time to open Google and check the Pod's error by executing the `kubectl describe pods demo-app` command.

Remove the Pod by executing `kubectl delete -f <filename>`.


## Step 2: creating replica's for our pod

**[`^ back to top ^`](#autoscaling-and-load-balancing-in-kubernetes)**

Pods on their own are already pretty useful, but what do you do when you need multiple instances of your application? Well, that's when you start to use Deployments or ReplicaSets. We'll just be focusing on Deployments as they're the most extensive option of the two.

A basic Deployment manifest looks a bit like a Pod manifest, just with some additional settings. Here, take a look at an example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: some-cool-deployment-name
  # Defines the labels for the Deployment
  labels:
    a-cool-label-name: a-cool-label-value
spec:
  replicas: 3
  # Defines the labels which the underlying ReplicaSet will use to find the Pods that are created for this Deployment. Must match the labels in the template below.
  selector:
    matchLabels:
      a-cool-label-name: a-cool-label-value
  template:
    # Defines the labels for the Pod(s) created by the Deployment
    metadata:
      labels:
        a-cool-label-name: a-cool-label-value
    spec:
      containers:
      - name: a-cool-container-name
        image: container-image:image-tag
```

Now, adjust this manifest in the same way as you did for the Pod:
- The Deployment's name should be "demo-app"
- The container's name should be "nginx"
- We'll be using the `nginx:latest` container image for our Deployment.
- Add a `ports` section that sets the container port to `80`.
- Change the labels so you have a label with the key "app" and the value "demo".

Once you've configured the Deployment, it's time to apply it using the `kubectl apply -f <filename>` command. This should start up a Deployment (of which you can check the status using the `kubectl get deployments demo-app` command):

```shell
[vdeborger@node-01 ~]$ kubectl get deployments demo-app
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
demo-app   3/3     3            3           5s
```

When you now list all the pods for that Deployment (which uses the label with key "app" and value "demo"), you should see 3 pods running:

```shell
[vdeborger@node-01 ~]$ kubectl get pods --selector=app=demo
NAME                       READY   STATUS    RESTARTS   AGE
demo-app-8dd45dff6-25nnz   1/1     Running   0          54s
demo-app-8dd45dff6-tkd8s   1/1     Running   0          54s
demo-app-8dd45dff6-zpp9v   1/1     Running   0          54s
```

If - for some reason - the deployment does not start up, try to debug it by checking the events using the `kubectl describe deployments demo-app` command.

## Step 3: accessing the deployments using a service

**[`^ back to top ^`](#autoscaling-and-load-balancing-in-kubernetes)**

Now that we have 3 replicas of our application running, it's time to add a service so it can be accessed. For this, we'll use a Kubernetes service which not only enables us to access our application, but this also automatically load balances over our replicas.

Let's start with a basic Service manifest:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: some-cool-deployment-name
spec:
  type: some-service-type
  # Defines the labels which the Service will use to find the Pods that are created for this Deployment. Must match the labels in the template section of the Deployment.
  selector:
    a-cool-label-name: a-cool-label-value
  ports:
  - protocol: TCP
    port: 8080
```

This basic manifest is pretty much all we need, but there are a couple of things that need changing before we can apply it:
- To keep the same naming convention, let's name the Service "demo-app".
- We want our application to be available inside the cluster, so set the type to "ClusterIP".
- The selector should match the Pod labels defined in the Deployment.
- The port defined in the Service should match the port defined in the Deployment.

Once that's done, apply the Service manifest using the `kubectl apply -f <filename>` command. After executing the command, you should be able to retrieve its status like so:

```shell
[vdeborger@node-01 ~]$ kubectl get services demo-app
NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
demo-app   ClusterIP   10.99.237.222   <none>        80/TCP    5s
```

As you can see, the service has an IP address, but that IP address is an internal Kubernetes service address. This means we can't access it from outside the cluster. In order to check out our application from inside, we can start a temporary Pod in which we can execute commands. Busybox is an ideal container for this use, start a busybox pod using the following command: `kubectl run -i --tty cluster-access --rm --image=busybox:latest --restart=Never -- /bin/sh`. Once the Pod has started, you should see a shell (`/ #`). In this shell, you can now send an HTTP request to our application using wget: `wget -q -O- <service_address>`. You should now see a response from NGINX! Exiting out of the busybox Pod can be done by typing `exit` and hitting Enter.

> *Tip*: the service address has a standard convention in kubernetes: "<service_name>.<namespace>.svc.cluster.local"

## Step 4: identifying which pod we're accessing

**[`^ back to top ^`](#autoscaling-and-load-balancing-in-kubernetes)**

When you executed the `wget` command in the previous step, you got load balanced to one of our 3 replicas in the Deployment we created earlier. In order to get a view which Pod we actually landed on, we'll add a couple of identifiers to the `index.html` file. For this, we'll use an initContainer. These containers are executed before the containers in the `containers` section are started. This means we can create an index file, put it on a volume and mount that volume to the NGINX container.

So, to get started, add a `initContainers` section to your Deployment's spec section on the `template spec` level. In the `initContainers` section, add a container that uses the busybox image and executes `echo -en "Running $POD_NAME ($POD_IP) on $NODE_NAME\n" > /nginx-temp/index.html`.

> *Tip*: executing commands when a container starts up can be done using the `command` and `args` parameters inside a container's definition. More information can be found [here](https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/#run-a-command-in-a-shell).

The `$POD_NAME`, `$POD_IP` and `$NODE_NAME` environment variables aren't set by default, so we'll need to make sure the container has these set. To do that, you can add an `env` list to your initContainer container. I'll spare you the trouble of looking up the specific parameters needed to retrieve the values for these variables, these are the environment variable definitions:

```yaml
env:
- name: NODE_NAME
  valueFrom:
    fieldRef:
      fieldPath: spec.nodeName
- name: POD_NAME
  valueFrom:
    fieldRef:
      fieldPath: metadata.name
- name: POD_IP
  valueFrom:
    fieldRef:
      fieldPath: status.podIP
```

As you might have also noticed, we're trying to write to an `index.html` file in the `/nginx-temp/` directory. This directory should be a volume. For the sake of simplicity, we'll be using an emptyDir volume which you can define like so (change the names where appropriate):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  ...
spec:
  ...
  template:
    spec:
      initContainers:
      - name: a-cool-initcontainer-name
        image: container-image:image-tag
        ...
        volumeMounts:
        - name: name-of-your-volume
          mountPath: "/a-cool-dir-name"
      containers:
      - name: a-cool-initcontainer-name
        image: container-image:image-tag
        ...
        volumeMounts:
        - name: name-of-your-volume
          mountPath: "/another-cool-dir-name"
      volumes:
      - name: name-of-your-volume
        emptyDir: {}
```

Now that our index file gets generated during the initialization phase of the Pod, we als need to make sure it ends up on our NGINX pod. To do that, you need to add a `volumeMount` to your NGINX container which mounts the emptyDir volume to the `/usr/share/nginx/html` path on your NGINX container.

After all that's done, you can start up your Deployment using the `kubectl apply -f <filename>` command and watch the pods initialize:

```shell
[vdeborger@node-01 ~]$ kubectl get pods --selector=app=demo
NAME                       READY   STATUS            RESTARTS   AGE
demo-app-d9f6d5bd5-f9tt4   0/1     PodInitializing   0          4s
demo-app-d9f6d5bd5-fsb8d   0/1     PodInitializing   0          4s
demo-app-d9f6d5bd5-zrpl6   0/1     PodInitializing   0          4s
```

At first, these pods will be in the "PodsInitializing" status; this means that Kubernetes is starting up our initContainer and is executing the commands we defined. After some time, we should see the status change to "Init:0/1" and eventually to "Running". Once that's the case, you can start up our temporary busybox Pod again and execute the `wget -q -O- <service_address>` command a couple of times. You should now see that the requests get distributed across our 3 pods.

## Step 5: automatically scale a Deployment

**[`^ back to top ^`](#autoscaling-and-load-balancing-in-kubernetes)**

In Kubernetes, there is a specific resource that allows you to horizontally scale a Deployment of ReplicaSet based on specific metrics. These metrics need to be retrieved, in order to make those available, we need to install a component called [metrics-server](https://github.com/kubernetes-sigs/metrics-server). 

### Step 5a: installing metrics-server

The installation can be done by applying the latest manifest: `kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml`. You might notice that the deployment does not become healthy (by executing `kubectl get deployment metrics-server --namespace kube-system`), that is because our kubelet service has been set up without being signed by the cluster's certificate authority. In order to configure metrics-server to allow insecure TLS, execute the following command which adds a specific argument to the startup command: 
```shell
kubectl patch deployment metrics-server --namespace kube-system --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/args", "value": [
  "--cert-dir=/tmp",
  "--secure-port=4443",
  "--kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname",
  "--kubelet-use-node-status-port",
  "--metric-resolution=15s",
  "--kubelet-insecure-tls"
]}]'
```

After a minute or two, you should see that the metrics-server is healthy with a running pod.

### Step 5b: setting up horizontal scaling for our Deployment

Now that the metrics-server has been installed, we can create [a Horizontal Pod Autoscaler (HPA)](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) which will use the metrics exposed by the metrics-server to decide whether extra replica's are required. To do so, we'll start off from a basic template and adjust it to our needs.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: some-cool-autoscaler-name
  # Defines the labels for the HorizontalPodAutoscaler
  labels:
    a-cool-label-name: a-cool-label-value
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: some-cool-deployment-name
  minReplicas: a-number-of-replicas
  maxReplicas: a-bigger-number-of-replicas
  metrics:
  - type: Resource
    resource:
      name: some-resource-metric
      target:
        type: a-source-metric-type
        averageUtilization: a-specific-threshold
```

We'll take it step by step, so we'll start off with the usual stuff;
- Our HorizontalPodAutoscaler needs a name, give it a recognizable name, e.g. `demo-hpa`.
- The HorizontalPodAutoscaler should target the deployment we created earlier, so make sure the name of the Deployment in `scaleTargetRef` matches the one you set earlier.
- Change the labels so you have a label with the key "app" and the value "demo".

Once that's done, we'll set up the auto scaling part:
- `minReplicas` defines the lowest number of replica's that should be running, let's say `3`.
- `maxReplicas` defines the maximum amount of replica's that can run, let's put that at `10`.
- The resource metric name can be (by default) one of two values: `cpu` or `memory`. For this exercise, we'll fill in `cpu`.
- The metric type should be set to `Utilization` in order to get the actual utilization.
- `averageUtilization` can be set to any number betwee 0 and 100, but in this exercise, we'll put it at `25` so we can see some autoscaling action with relatively low usage.

Alright, that's it! Let's apply the Kubernetes manifest and see it in action. After executing the `kubectl apply -f <filename>` command, you can check the status of the HPA using the `kubectl get hpa <hpa-name>` command.

As you might notice, the HPA reports `<unknown>/25%` in the Targets column. This is because we didn't set any resource requests/limits inside our Deployment. In Kubernetes, a resource request is the amount of CPU or memory that a container initially requests to run, ensuring that the cluster places it on a node with sufficient capacity. Resource limits, on the other hand, define the maximum amount of CPU or memory a container is allowed to consume, preventing it from taking up all the resources and potentially affecting the stability of other containers on the same node. In order to set a resource request/limit on the Deployment, edit the Kubernetes manifest so it includes an `resources` section in the container specification.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  ...
spec:
    ...
    spec:
      ...
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 10m
            memory: 64Mi
          requests:
            cpu: 100m
            memory: 256Mi
        volumeMounts:
        - name: workdir
          mountPath: /usr/share/nginx/html
        ...
```

Once you've changed the Kubernetes manifest for the Deployment, apply it and check the status of the HPA. Quite soon it should show something like this:

```shell
[vdeborger@node-01 ~]$ kubectl get hpa demo-hpa
NAME       REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
demo-hpa   Deployment/demo-app   0%/25%    3         10        3          5m
```

### Step 5c: generating load and seeing the HPA in action

Okay, our Deployment is running and the HPA is monitoring the Pods' CPU usage. Everything's ready to actually see the HPA in action. We'll need to create some load on the Pods in order to trigger a scaling event. Let's use a custom I built which uses [baton](https://github.com/americanexpress/baton) to send requests to our Pods: `kubectl run -i --tty cluster-access --rm --image=vincentdebo/baton:latest --restart=Never -- -u http://demo-app -c 4 -t 60`. When you check the status of the HPA now (with `kubectl get hpa demo-hpa --watch` so it auto-updates), you should see it scale up once the CPU utilization goes over the threshold and scale back down once the CPU utilization goes back under the threshold.

```shell
[vdeborger@node-01 ~]$ kubectl get hpa demo-hpa --watch
NAME       REFERENCE             TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
demo-hpa   Deployment/demo-app   0%/25%     3         10        3          21m
demo-hpa   Deployment/demo-app   166%/25%   3         10        3          22m
demo-hpa   Deployment/demo-app   213%/25%   3         10        6          22m
demo-hpa   Deployment/demo-app   190%/25%   3         10        10         22m
demo-hpa   Deployment/demo-app   61%/25%    3         10        10         22m
demo-hpa   Deployment/demo-app   20%/25%    3         10        10         23m
demo-hpa   Deployment/demo-app   0%/25%     3         10        10         23m
demo-hpa   Deployment/demo-app   0%/25%     3         10        10         27m
demo-hpa   Deployment/demo-app   0%/25%     3         10        10         28m
demo-hpa   Deployment/demo-app   0%/25%     3         10        3          28m
```

While the application is under load and scaled up, you can also see the replica's being created in the Pod overview (`kubectl get pods --selector=app=demo`).

## Bonus: using MetalLB to make our application available outside the cluster

**[`^ back to top ^`](#autoscaling-and-load-balancing-in-kubernetes)**

MetalLB is an open-source load balancer designed for Kubernetes clusters without native integration with cloud provider load balancers. In contrary to most load balancers, MetalLB has been created for on-premise Kubernetes clusters. MetalLB allows you to provision external IP addresses, providing external access and load balancing for applications running in your Kubernetes cluster.

### Installation

Before we can install MetalLB, we need to make sure that kube-proxy (Kubernetes networking component for service abstraction and load balancing) is running in "strictARP" mode. To edit the kube-proxy config of your cluster, execute `kubectl edit configmap -n kube-system kube-proxy` and set "strictARP" to true:

```yaml
apiVersion: v1
data:
  config.conf: |-
    ...
    ipvs:
      strictARP: true
    ...
```

That should be it for kube-proxy, you can always double check the value by executing the `kubectl get configmap -n kube-system kube-proxy -o jsonpath="{.data['config\.conf']}" | grep strictARP` command.

We can now install the MetalLB Kubernetes manifest:
```shell
export LATEST_VERSION=$(curl -s https://api.github.com/repos/metallb/metallb/releases/latest | grep \"tag_name\" | cut -d : -f 2,3 | tr -d \" | tr -d , | tr -d " ")
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/$LATEST_VERSION/config/manifests/metallb-native.yaml
```

This will create a couple of resources in your cluster, in the metallb-system namespace. Some of the most noteworthy resources created by the manifest are the following;

- A deployment called "controller"; this is the cluster-wide component that’s responsible for allocating IP addresses, configuring the load balancer, dynamically updating configurations and performing health checks.
- A daemonset called "speaker"; this component is deployed on each node and is responsible for ensuring that external traffic can reach the services within the Kubernetes cluster.
- A couple of service accounts along with RBAC permissions which are necessary for the components to function.

You can verify the deployment of the components by executing the following command:

```shell
[vdeborger@node-01 ~]$ kubectl get pods -n metallb-system
NAME                          READY   STATUS    RESTARTS   AGE
controller-595f88d88f-52vzq   1/1     Running   0          1m13s
speaker-fr8xk                 1/1     Running   0          1m13s
speaker-qs45k                 1/1     Running   0          1m13s
speaker-z9rvx                 1/1     Running   0          1m13s
```

If the components are in a stable "Running" state, the deployment of MetalLB is complete. 

### Configuration

One of the things we need to configure is an IPAddressPool. This resource serves as a configuration that defines a range of IP addresses which can be used for allocating to services with the "LoadBalancer" type.

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: lecture
  namespace: metallb-system
spec:
  addresses:
  - <ip_address_range>
```

Once you’ve created an MetalLB IP address pool, it’s time to make sure that we can access the services from the IP addresses provided by MetalLB. This is done by creating an "Advertisement" for the IP address pool.

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: external-access
  namespace: metallb-system
spec:
  ipAddressPools:
  - lecture
```

Once the IPAddressPool and Advertisement have been configured, we can define a service which uses MetalLB to load balance its traffic. There are 2 important items in a Kubernetes service with a MetalLB load balancer;

- The type of the service should be set to "LoadBalancer"
- An annotation called "metallb.universe.tf/address-pool" has to be added.

You can find an example of a Kubernetes service manifest below;

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-app
  annotations:
    metallb.universe.tf/address-pool: lecture
spec:
  type: LoadBalancer
  selector:
    app: demo
  ports:
  - protocol: TCP
    port: 80
```

Once the service has been created, you can verify that the service has received an IP address using the kubectl get service command.

```shell
[vdeborger@node-01 ~]$ kubectl get service demo-app
NAME       TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)        AGE
demo-app   LoadBalancer   10.106.252.134   x.x.x.x          80:30821/TCP   1m4s
```

As you can see in the "EXTERNAL-IP" column, the service has received the IP address "x.x.x.x", which should be the first IP adress in your IPAddressPool range. You should now be able to access the application from your local machine using the external IP address.

---

Sources:
- Kubernetes HorizontalPodScaler: [https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
- Kubernetes services: [https://kubernetes.io/docs/concepts/services-networking/service/](https://kubernetes.io/docs/concepts/services-networking/service/)
- Setting up & using MetalLB: [https://metallb.universe.tf/](https://metallb.universe.tf/)
