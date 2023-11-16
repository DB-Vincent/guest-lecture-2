# Kubectl Cheat Sheet

## Basics

| Command | Description |
| ------- | ----------- |
| `kubectl --help` | Get Help |
| `kubectl api-resources` | View Kubernetes API Resources |
| `kubectl cluster-info` | Get Cluster Info |

## Pods

| Command | Description |
| ------- | ----------- |
| `kubectl get pods` | List Pods |
| `kubectl get pods -A` | List Pods in all namespaces |
| `kubectl get pods -n <namespace_name>` | List Pods in a specific namespace |
| `kubectl describe pod <pod_name>` | Describe Pod |
| `kubectl create -f <pod_manifest.yaml>` | Create Pod |
| `kubectl delete pod <pod_name>` | Delete Pod |

## Services

| Command | Description |
| ------- | ----------- |
| `kubectl get services` | List Services |
| `kubectl describe service <service_name>` | Describe Service |

## Deployments

| Command | Description |
| ------- | ----------- |
| `kubectl get deployments` | List Deployments |
| `kubectl scale deployment <deployment_name> --replicas=<replica_count>` | Scale Deployment |
| `kubectl rollout restart deployment <deployment_name>` | Rolling Restart Deployment |

## ConfigMaps and Secrets

| Command | Description |
| ------- | ----------- |
| `kubectl create configmap <configmap_name> --from-file=<path_to_file>` | Create ConfigMap |
| `kubectl create secret generic <secret_name> --from-literal=<key>=<value>` | Create Secret |

## Namespace

| Command | Description |
| ------- | ----------- |
| `kubectl get namespaces` | List Namespaces |
| `kubectl config set-context --current --namespace=<namespace_name>` | Switch Namespace |

## Context

| Command | Description |
| ------- | ----------- |
| `kubectl config get-contexts` | List Contexts |
| `kubectl config use-context <context_name>` | Switch Context |

## Other Useful Commands

| Command | Description |
| ------- | ----------- |
| `kubectl port-forward <pod_name> <local_port>:<pod_port>` | Port Forwarding |
| `kubectl exec -it <pod_name> -- <command>` | Run a Command in a Pod |
| `kubectl logs <pod_name>` | Logs |
| `kubectl apply -f <manifest_file.yaml>` | Apply Configuration |
| `kubectl get <resource_type> <resource_name> -o yaml` | Get YAML Representation of Resource |
| `kubectl delete <resource_type> <resource_name>` | Delete Resource |