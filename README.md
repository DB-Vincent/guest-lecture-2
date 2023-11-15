# 2nd guest lecture @ Thomas More

This repository contains all the source files for the second DXC guest lecture at Thomas More.

Inspiration has been taken from these sources:
- HorizontalPodScaler walkthrough: [https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
- Setting up & using MetalLB: [https://vincentdeborger.be/blog/exploring-metallb-for-load-balancing-in-scaled-workloads-in-kubernetes](https://vincentdeborger.be/blog/exploring-metallb-for-load-balancing-in-scaled-workloads-in-kubernetes)

## Step 1: setting up a pod

A basic pod YAML would look something like this:

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
- We'll be using the `nginx:latest` container image for our Pod.
- Add a `ports` section that sets the container port to `80`.
- Change the label so you have a label with the value "app" and the value "demo". 

Once your Pod manifest has been adjusted, apply it using the `kubectl apply -f <filename>` command. You should now see your pod starting up:

```shell
[vdeborger@node-01 ~]$ kubectl get pods demo-app
NAME       READY   STATUS              RESTARTS   AGE
demo-app   0/1     ContainerCreating   0          3s
```

After some time (usually 30 seconds to a minute), the container image should be pulled and your container should be in the "Running" status. If that's the case, great news, you can continue to step 2! If that's not the case, it's time to open Google and check the Pod's error by executing the `kubectl describe pods demo-app` command.

## Step 2: creating replica's for our pod

Pods on their own are already pretty usefull, but what do you do when you need multiple instances of your application? Well, that's when you start to use Deployments or ReplicaSets. We'll just be focusing on Deployments as they're the most extensive option of the two.

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
- Change the labels so you have a label with the value "app" and the value "demo". 

Once you've configured the Deployment, it's time to apply it using the `kubectl apply -f <filename>` command. This should start up a Deployment (of which you can check the status using the `kubectl get deployments demo-app` command):

```shell
[vdeborger@node-01 ~]$ kubectl get deployments demo-app
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
demo-app   1/1     1            1           5s
```

If - for some reason - the deployment does not start up, try to debug it by checking the events using the `kubectl describe deployments demo-app` command.