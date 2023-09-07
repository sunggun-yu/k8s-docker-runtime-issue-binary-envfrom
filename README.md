# K8s Docker runtime issue

, when value of secret is binary text and it is passed as environment variable into Pod/Container with envFrom/secretRef

The issue was discovered in another environment that utilizes Rancher RKE1, which uses Docker as the container runtime. This repository is to replicate the issue in a similar environment using Minikube.

- [Prepare environment](#prepare-environment)
  - [Minikube](#minikube)
  - [Kind](#kind)
  - [Switch context](#switch-context)
- [Test](#test)
  - [case 1](#case-1)
  - [case 2](#case-2)
  - [case 3](#case-3)
  - [case 4](#case-4)
  - [case 5](#case-5)
  - [case 6](#case-6)
- [Checks the test result](#checks-the-test-result)
- [Symptom](#symptom)
- [Hypothesis](#hypothesis)
- [Difference between Minikube/Docker and Kind/Containerd](#difference-between-minikubedocker-and-kindcontainerd)
- [How to restore the node](#how-to-restore-the-node)

## Prepare environment

- Kubernetes v1.27.3

- Minikube: v1.31.2
  - Docker: version 24.0.4, build 3713ee1

  ```bash
  brew install minikube
  ```

- Kind: v0.20.0 go1.20.5 darwin/arm64
  - Containerd v1.7.1 1677a17964311325ed1c31e2c0a3589ce6d5c30d

  ```bash
  brew install kind
  ```

### Minikube

Start Minikube cluster with Docker runtime

```bash
$ minikube start --driver=docker --kubernetes-version=v1.27.3

$ kubectl get nodes -o wide

NAME       STATUS   ROLES           AGE     VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION        CONTAINER-RUNTIME
minikube   Ready    control-plane   4m34s   v1.27.3   192.168.49.2   <none>        Ubuntu 22.04.2 LTS   5.15.49-linuxkit-pr   docker://24.0.4
```

Watch the Minikube logs

```bash
minikube logs -f
```

Attach to Minikube container to run `docker` commands

```bash
docker exec -it minikube bash
# or
minikube ssh

docker ps -a
```

### Kind

Start Kind cluster for comparison with Docker and Containerd runtime

```bash
$ kind create cluster --config=configs/kind-cluster-config.yaml

$ kubectl get nodes -o wide
NAME                 STATUS   ROLES           AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION        CONTAINER-RUNTIME
kind-control-plane   Ready    control-plane   2m20s   v1.27.3   172.18.0.2    <none>        Debian GNU/Linux 11 (bullseye)   5.15.49-linuxkit-pr   containerd://1.7.1
```

Export Kind logs after testing to see `kubelet` logs

```bash
kind export logs
```

Attach to Kind container to run `crictl` commands

```bash
docker exec -it kind-control-plane bash
crictl ps -a
```

### Switch context

, between minikube and kind if necessary

```bash
brew install kubectx
```

```bash
kubectx minikube
```

```bash
kubectx kind-kind
```

## Test

deploy deployment into minikube cluster

### case 1

for more details, see [case 1](./manifests/cases/1/README.md)

  ```bash
  kustomize build manifests/cases/1 | kubectl apply -f -
  ```

### case 2

for more details, see [case 2](./manifests/cases/2/README.md)

  ```bash
  kustomize build manifests/cases/2 | kubectl apply -f -
  ```

### case 3

for more details, see [case 3](./manifests/cases/3/README.md)

  ```bash
  kustomize build manifests/cases/3 | kubectl apply -f -
  ```

### case 4

for more details, see [case 4](./manifests/cases/4/README.md)

  ```bash
  kustomize build manifests/cases/4 | kubectl apply -f -
  ```

### case 5

for more details, see [case 5](./manifests/cases/5/README.md)

  ```bash
  kustomize build manifests/cases/5 | kubectl apply -f -
  ```

### case 6

for more details, see [case 6](./manifests/cases/6/README.md)

  ```bash
  kustomize build manifests/cases/6 | kubectl apply -f -
  ```

## Checks the test result

get into minikube docker container

```bash
docker exec -it minikube bash
```

check if any container of Pod is hang

```bash
docker ps -a | awk '{print $1}' | while read ID; do echo $ID; docker inspect $ID >/dev/null; done
```

if there is not responding container for `docker inspect`, entire node will be marked as `NotReady` status

## Symptom

I created Secret with PNG images binary data that is base64 encoded text which size of `50113` bytes after decoding. also, the Deployment set it as environment variable with `envFrom` as below.

```yaml
envFrom:
  - secretRef:
      name: secret
      optional: false 
```

then, the Node that container of Pod is running will goes into `NotReady` state.

but it wasn't reproducing with ConfigMap.

```yaml
envFrom:
  - configMapRef:
      name: config
      optional: false
```

## Hypothesis

The maximum size of data may affect somehow?:

- Secret: `262144 bytes`
- [Configmap](https://kubernetes.io/docs/concepts/configuration/configmap/#motivation): `1MiB`

## Difference between Minikube/Docker and Kind/Containerd

Pod/Container status in `Kind`/`Containerd` cluster changed to `CrashLoopBackOff` while `Minikube/Docker` keep remains `ContainerCreating`

- Minikube/Docker:

  ```txt
  NAME                              READY   STATUS              RESTARTS   AGE
  case1-helloweb-64f44dbf67-wlk64   0/1     ContainerCreating   0          19m
  ```

  ```txt
  NAME       STATUS     ROLES           AGE   VERSION
  minikube   NotReady   control-plane    1h   v1.27.3
  ```

- Kind/Containerd:

  ```txt
  NAME                              READY   STATUS             RESTARTS        AGE
  case1-helloweb-766d98967f-zlbf7   0/1     CrashLoopBackOff   8 (3m30s ago)   37m
  ```

  ```txt
  NAME                 STATUS   ROLES           AGE   VERSION
  kind-control-plane   Ready    control-plane   98m   v1.27.3
  ```

## How to restore the node

delete the deployment

```bash
kustomize build manifests/cases/1 | kubectl delete -f -
```

force delete the pod

```bash
kubectl delete pod <pod-name-of-hello> --force
```

restart the docker daemon in the minikube container

```bash
sudo systemctl restart docker
```
