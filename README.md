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

Once your Pod manifest has been adjusted, apply it using the `kubectl apply -f <filename>` command. You should now see your pod starting up:

```shell
[vdeborger@node-01 ~]$ kubectl get pods demo-app
NAME       READY   STATUS              RESTARTS   AGE
demo-app   0/1     ContainerCreating   0          3s
```

After some time (usually 30 seconds to a minute), the container image should be pulled and your container should be in the "Running" status. If that's the case, great news, you can continue to step 2! If that's not the case, it's time to open Google and check the Pod's error by executing the `kubectl describe pods demo-app` command.