theme: Next, 3

# Docker and Kubernetes

### By Robert Brown

---

# Docker

---

# Docker

* A portable store of a single component and its dependencies
* Not a VM
* Boots in under a second

---

# Docker Hub

---

# Docker Hub

* Like GitHub but for containers.
* Default registry.
* Can have your own Docker registries.
* Amazon offers Elastic Cloud Registry (ECR).

---

# Docker Hub: Common Images

[.column]

* `debian`
* `alpine` 
* `nginx`

[.column]

* `redis`
* `erlang`
* `elixir`

---

# Container Versions

From [official Elixir container](https://hub.docker.com/_/elixir):

[.column]

* `1.10.2`, `1.10`, `latest`
* `1.10.2-slim`, `1.10-slim`, `slim`
* `1.10.2-alpine`, `1.10-alpine`, `alpine`

[.column]

* `1.9.4`, `1.9`
* `1.9.4-slim`, `1.9-slim`
* `1.9.4-alpine`, `1.9-alpine`

---

# Image Variants

* Default images typically use Debian.
* Slim images have only minimal dependencies included.
* Alpine images are also small but include Busybox.
* Busybox includes common CLI commands with minified functionality.

^ Prefer Alpine over slim images.

---

# Dockerfile

---

# Dockerfile

* Repeatable set of instructions to make a container.
* Each command is a layer.
* When rebuiling, previous layers don't need to run if nothing changes.
* Can include more than one image for multi-stage builds.

---

# Dockerfile 

# Example

---

# Dockerfile: Part 1/4

```dockerfile
# The version of Alpine to use for the final image
# This should match the version of Alpine that the `elixir:1.10-alpine` image uses
ARG ALPINE_VERSION=3.11

FROM elixir:1.10-alpine AS builder

# Private hex.pm key
ARG HEX_KEY

# By convention, /opt is typically used for applications
WORKDIR /opt/app

# This step installs all the build tools we'll need
RUN apk update && \
  apk upgrade --no-cache && \
  apk add --no-cache \
    git \
    build-base && \
  mix local.rebar --force && \
  mix local.hex --force
```

---

# Dockerfile: Part 2/4

```dockerfile
# Cache the dependency fetching
COPY mix.exs mix.exs
COPY mix.lock mix.lock
COPY VERSION VERSION

RUN mix hex.organization auth tubitv --key $HEX_KEY
RUN mix deps.get
RUN MIX_ENV=test mix deps.compile

COPY lib lib
COPY test test

RUN mix test
```

---

# Dockerfile: Part 3/4

```dockerfile
RUN MIX_ENV=prod mix deps.compile
RUN MIX_ENV=prod mix compile

ENV MIX_ENV=prod

RUN \
  mkdir -p /opt/built && \
  mix release && \
  cp _build/${MIX_ENV}/*.tar.gz /opt/built/built.tar.gz && \
  cd /opt/built && \
  tar -xzf *.tar.gz && \
  rm *.tar.gz
```

---

# Dockerfile: Part 4/4

```dockerfile
# From this line onwards, we're in a new image, which will be the image used in production
FROM alpine:${ALPINE_VERSION}

RUN apk update && \
    apk add --no-cache \
    bash \
    openssl-dev

WORKDIR /opt/app

COPY --from=builder /opt/built .

EXPOSE 18001

CMD /opt/app/bin/crm start
```

---

# Why Multi-stage Builds?

* Building code requires toolchains and other dependencies.
* These aren't needed when running the build artifacts.
* Especially true when bundling the Erlang VM.
* Smaller, faster containers.
* Smaller attack surface.

^ The downside is fewer debugging tool.

^ Alpine includes Busybox which has slimmed down versions of common Unix CLI tools.

---

![fit](img/docker_images.png)

---

# Kubernetes

---

# Kubernetes

* Open source container-orchestration system built by Google.
* Greek for "helmsman" or "pilot"
* Often called k8s (k-eight letters-s)

---

# Terminology

---

# Pods

* A collection of containers and volumes.
* Typcially one container per pod.
* All containers in a pod share an IP address.
* Erlang's EPMD has problems with multiple containers per volume.
* Planned to fix in future version of Erlang.

^ EPMD = Erlang Port Mapper Daemon

---

# Pods

![inline](./img/pods.png)

[.footer: Image: https://kubernetes.io/docs/tutorials/kubernetes-basics/explore/explore-intro/]

--- 

# Node

* A worker machine in the cluster
* Runs pods
* May be physical or virtual
* Runs Kubelet to communicate with Kubernetes Master
* Run a container runtime (ex. Docker, rkt)

---

# Node

![inline](./img/node.png)

[.footer: Image: https://kubernetes.io/docs/tutorials/kubernetes-basics/explore/explore-intro/]

---

# Replica Sets

* Maintains a stable set of pods.

---

# Deployment

* Controls and updates pods and replica sets.
* Prefered to working with pods and replica sets directly.
* Allows for rolling back to a previous deployment.

---

# Service

* A logical set of pods and a policy by which to access them.
* Publicly exposes a pod or deployment.
* Balances load between pods.

---

# Service

![inline](./img/service.png)

[.footer: https://kubernetes.io/docs/tutorials/kubernetes-basics/expose/expose-intro/]

---

# Namespaces

* A logical grouping of resources.
* When a namespace is deleted, so is everything in the namespace.
* Great for ensuring everything is cleaned up.

---

# Configuration

---

# Configuration Options

* YAML (preferred)
* JSON
* `kubectl`

--- 

# Example: Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: crm
```

---

# Example: Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: crm-pod
  labels:
    app: crm-pod
spec:
  containers:
    - name: crm-pod
      image: crm
      ports:
        - containerPort: 18001
```

---

# Example: Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: crm-node
spec:
  type: NodePort
  ports:
  - port: 18001
    protocol: TCP
    targetPort: 18001
    nodePort: 30080
  selector:
    app: crm-node
```

---

# Demo

---

## Create Deployment

---

![fit](img/k8s_create_deployment.png)

---

## Create Service

---

![fit](img/k8s_create_service.png)

---

## Scale Up

---

![fit](img/k8s_scale_up_1.png)

---

![fit](img/k8s_scale_up_2.png)

---

## Scale Down

---

![fit](img/k8s_scale_down.png)

---

## Autoscale

---

![fit](img/k8s_autoscale.png)

---

## Update Container

---

![fit](img/k8s_update.png)

---

## Rollback

---

![fit](img/k8s_rollback_1.png)

---

![fit](img/k8s_rollback_2.png)

---

# Questions?

---

# Resources

* [Best practices for writing Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
* [Docker multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/)
* [Learn Kubernetes Basics](https://kubernetes.io/docs/tutorials/kubernetes-basics/)

---
---
---
---

# Bash Examples Created with Carbon

^ Terminal UI images: https://carbon.now.sh/?bg=rgba(0%2C0%2C0%2C0)&t=seti&wt=none&l=application%2Fx-sh&ds=false&dsyoff=20px&dsblur=68px&wc=true&wa=false&pv=56px&ph=56px&ln=false&fl=1&fm=Hack&fs=14px&lh=133%25&si=false&es=2x&wm=false&code=

---

```bash
$ docker build --build-arg HEX_KEY='REDACTED' -t crm -f docker_crm/Dockerfile .

$ docker images
REPOSITORY    TAG                 IMAGE ID            CREATED             SIZE
crm           latest              a28e197ebd00        10 minutes ago      24.1MB
<none>        <none>              6d44057c3e65        10 minutes ago      335MB
```

---

```bash
$ kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
minikube   Ready    master   19s   v1.17.3

$ kubectl create deployment kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1
deployment.apps/kubernetes-bootcamp created

$ kubectl get deployments
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   1/1     1            1           10s

$ kubectl get replicasets
NAME                             DESIRED   CURRENT   READY   AGE
kubernetes-bootcamp-69fbc6f4cf   1         1         1       76s

$ kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
kubernetes-bootcamp-69fbc6f4cf-4kp5l   1/1     Running   0          91s
```

---

```bash
$ kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080
service/kubernetes-bootcamp exposed

$ kubectl get services
NAME                  TYPE       CLUSTER-IP    EXTERNAL-IP  PORT(S)          AGE
kubernetes            ClusterIP  10.96.0.1     <none>       443/TCP          28s
kubernetes-bootcamp   NodePort   10.102.72.34  <none>       8080:32073/TCP   15s

$ curl $(minikube ip):32073
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-765bf4c7b4-hzrmd
```

---

```bash
$ kubectl scale deployments/kubernetes-bootcamp --replicas=4
deployment.apps/kubernetes-bootcamp scaled

$ kubectl get replicasets
NAME                             DESIRED   CURRENT   READY   AGE
kubernetes-bootcamp-765bf4c7b4   4         4         4       104s

$ kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
kubernetes-bootcamp-765bf4c7b4-2h2rm   1/1     Running   0          13s
kubernetes-bootcamp-765bf4c7b4-jwgdj   1/1     Running   0          13s
kubernetes-bootcamp-765bf4c7b4-wpmgr   1/1     Running   0          13s
kubernetes-bootcamp-765bf4c7b4-wxck4   1/1     Running   0          104s
```

---

```bash
$ curl $(minikube ip):32073
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-765bf4c7b4-2h2rm

$ curl $(minikube ip):32073
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-765bf4c7b4-wpmgr

$ curl $(minikube ip):32073
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-765bf4c7b4-jwgdj

$ curl $(minikube ip):32073
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-765bf4c7b4-wxck4

$ curl $(minikube ip):32073
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-765bf4c7b4-wpmgr
```

---

```bash
$ kubectl scale deployments/kubernetes-bootcamp --replicas=2
deployment.apps/kubernetes-bootcamp scaled

$ kubectl get deployments
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   2/2     2            2           2m44s

$ kubectl get pods
NAME                                   READY   STATUS        RESTARTS   AGE
kubernetes-bootcamp-765bf4c7b4-5zvvx   1/1     Terminating   0          2m11s
kubernetes-bootcamp-765bf4c7b4-hhhjz   1/1     Running       0          2m58s
kubernetes-bootcamp-765bf4c7b4-rwmqc   1/1     Terminating   0          2m11s
kubernetes-bootcamp-765bf4c7b4-z8jm4   1/1     Running       0          2m11s

$ kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
kubernetes-bootcamp-765bf4c7b4-hhhjz   1/1     Running   0          3m20s
kubernetes-bootcamp-765bf4c7b4-z8jm4   1/1     Running   0          2m33s
```

---

```bash
$ kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2
deployment.apps/kubernetes-bootcamp image updated

$ kubectl get pods
NAME                                   READY   STATUS              RESTARTS  AGE
kubernetes-bootcamp-765bf4c7b4-kcfz2   1/1     Running             0         4m
kubernetes-bootcamp-765bf4c7b4-p95kp   1/1     Terminating         0         4m
kubernetes-bootcamp-765bf4c7b4-pc2cp   1/1     Terminating         0         4m
kubernetes-bootcamp-765bf4c7b4-z9g95   1/1     Terminating         0         4m
kubernetes-bootcamp-7d6f8694b6-g287r   0/1     ContainerCreating   0         1s
kubernetes-bootcamp-7d6f8694b6-ksh4j   0/1     ContainerCreating   0         1s
kubernetes-bootcamp-7d6f8694b6-pjvk8   1/1     Running             0         4s
kubernetes-bootcamp-7d6f8694b6-t2s8q   1/1     Running             0         4s

$ kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
kubernetes-bootcamp-7d6f8694b6-g287r   1/1     Running   0          55s
kubernetes-bootcamp-7d6f8694b6-ksh4j   1/1     Running   0          55s
kubernetes-bootcamp-7d6f8694b6-pjvk8   1/1     Running   0          58s
kubernetes-bootcamp-7d6f8694b6-t2s8q   1/1     Running   0          58s
```

---

```bash
$ kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=gcr.io/google-samples/kubernetes-bootcamp:v10
deployment.apps/kubernetes-bootcamp image updated

$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
kubernetes-bootcamp-7d6f8694b6-g287r   1/1     Running            0          92s
kubernetes-bootcamp-7d6f8694b6-ksh4j   1/1     Terminating        0          92s
kubernetes-bootcamp-7d6f8694b6-pjvk8   1/1     Running            0          95s
kubernetes-bootcamp-7d6f8694b6-t2s8q   1/1     Running            0          95s
kubernetes-bootcamp-886577c5d-2zlt9    0/1     ImagePullBackOff   0          5s
kubernetes-bootcamp-886577c5d-rsxmp    0/1     ErrImagePull       0          5s
```

---

```bash
$ kubectl rollout undo deployments/kubernetes-bootcamp
deployment.apps/kubernetes-bootcamp rolled back

$ kubectl get pods
NAME                                   READY   STATUS              RESTARTS  AGE
kubernetes-bootcamp-7d6f8694b6-fj2md   0/1     ContainerCreating   0         2s
kubernetes-bootcamp-7d6f8694b6-g287r   1/1     Running             0         2m
kubernetes-bootcamp-7d6f8694b6-pjvk8   1/1     Running             0         2m
kubernetes-bootcamp-7d6f8694b6-t2s8q   1/1     Running             0         2m
kubernetes-bootcamp-886577c5d-2zlt9    0/1     Terminating         0         73s
kubernetes-bootcamp-886577c5d-rsxmp    0/1     Terminating         0         73s

$ kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
kubernetes-bootcamp-7d6f8694b6-fj2md   1/1     Running   0          24s
kubernetes-bootcamp-7d6f8694b6-g287r   1/1     Running   0          3m2s
kubernetes-bootcamp-7d6f8694b6-pjvk8   1/1     Running   0          3m5s
kubernetes-bootcamp-7d6f8694b6-t2s8q   1/1     Running   0          3m5s
```

---

```bash
$ kubectl autoscale deployment/kubernetes-bootcamp --min=2 --max=10 --cpu-percent=70
horizontalpodautoscaler.autoscaling/kubernetes-bootcamp autoscaled

$ kubectl get deployments
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   2/2     2            2           2m58s

$ kubectl get replicasets
NAME                             DESIRED   CURRENT   READY   AGE
kubernetes-bootcamp-765bf4c7b4   2         2         2       3m

$ kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
kubernetes-bootcamp-765bf4c7b4-nqf8w   1/1     Running   0          84s
kubernetes-bootcamp-765bf4c7b4-psszr   1/1     Running   0          3m5s
```
