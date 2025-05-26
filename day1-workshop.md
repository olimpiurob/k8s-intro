# Introduction to Kubernetes / Deutsche Bank / 26, 27, 29 May 2025

## Day 1: Kubernetes Fundamentals

### Schedule - Day 1

- 10:00 - 10:30: Introduction and Environment Setup
- 10:30 - 12:00: Containers Recap & Kubernetes Overview
- 12:00 - 12:15: Break
- 12:15 - 13:30: Kubernetes Objects & kubectl
- 13:30 - 14:30: Lunch Break
- 14:30 - 16:00: YAML, Namespaces, and Pods
- 16:00 - 16:15: Break
- 16:15 - 18:00: Events and Services

## Format

- I go (quickly) over the theory
- I solve an exercise
- you solve the exercise
- we go over and solve any encountered issue together so that everyone finishes the exercise

## Getting To Know Each Other

Hello, I'm Olimpiu!

- I've been programming and doing some sysadmining for the past ~20 years
- backend - python
- dev&ops @[Eau we Web](https://eaudeweb.ro/)
- sysadmin & networking
- Kubernetes instructor @[JSLeague](https://www.jsleague.ro/)

What about you?

- Technical background?
- Experience with Linux, terminal, Docker / containers, Kubernetes?
- Expectations from the workshop?

## 0. Containers Recap

But first, a bit of history! [The History of Virtualization and Containerization](https://kubernetestutorials.com/kubernetes-beginners-tutorials/the-history-of-virtualization-and-containerization/)

> A container is a **sandboxed process** running on a host machine that is isolated from all other processes running on that host machine. That isolation leverages kernel namespaces and cgroups, features that have been in Linux for a long time. Docker makes these capabilities approachable and easy to use. - [Images and containers](https://docs.docker.com/get-started/#images-and-containers)

> A container **runs natively on Linux** and shares the kernel of the host machine with other containers. It runs a discrete process, taking no more memory than any other executable, making it lightweight. - [Containers and (vs.) virtual machines](https://docs.docker.com/guides/docker-concepts/the-basics/what-is-a-container/  #containers-versus-virtual-machines-vms)

To achieve isolation, (Docker) containers use the following Linux kernel features:

- namespaces: PID, MNT, NET, UTS (hostname), IPC
- control groups (`cgroups`): share available hardware resources
and, if required, set up limits and constraints - memory, CPU, disk I/O
- secure computing mode (`seccomp`):  restrict the actions available within a container down
to the granularity of a single system call
  > Docker's default seccomp profile is a whitelist of calls that are allowed and blocks over 50 different syscalls. [...] The default bounding set of capabilities inside a Docker container is less than half the total capabilities assigned to a Linux process.
  > `root` inside a container is different than `root` on the host"

## 1. Kubernetes Overview & Architecture

> Kubernetes is a portable, extensible, open-source platform for managing containerized workloads and services, that facilitates both declarative configuration and automation. It has a large, rapidly growing ecosystem. [...] The name Kubernetes originates from Greek, meaning helmsman or pilot. Google open-sourced the Kubernetes project in 2014. [...] Kubernetes provides the building blocks for building developer platforms, but preserves user choice and flexibility where it is important. - [What is Kubernetes?](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/)

Kubernetes distributions:
- [minikube](https://minikube.sigs.k8s.io/docs/), [kind](https://kind.sigs.k8s.io/) - development
- [kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/), [kubespray](https://kubernetes.io/docs/setup/production-environment/tools/kubespray/), [rke](https://rancher.com/docs/rke/latest/en/), [k3s](https://k3s.io/) - bare-metal clusters
- [Rancher](https://rancher.com/), [Red Hat OpenShift](https://www.openshift.com/learn/topics/kubernetes/) - enterprise
- [GKE](https://cloud.google.com/kubernetes-engine), [EKS](https://aws.amazon.com/eks/), [AKS](https://azure.microsoft.com/en-in/services/kubernetes-service/) - infrastructure provider-managed
- many others

[Kubernetes components](https://kubernetes.io/docs/concepts/overview/components/) - drawing board time!

## 2. Kubernetes Objects

> Kubernetes objects are persistent entities in the Kubernetes system. Kubernetes uses these entities to represent the state of your cluster. Specifically, they can describe:
> - What containerized applications are running (and on which nodes)
> - The resources available to those applications
> - The policies around how those applications behave, such as restart policies, upgrades, and fault-tolerance - [Understanding Kubernetes Objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)

Object fields:
- `apiVersion` - Which version of the Kubernetes API you're using to create this object
- `kind` - What kind of object you want to create
- `metadata` - Data that helps uniquely identify the object, including a `name` string, `UID`, and optional `namespace`
- `spec` - What state you desire for the object
- `status` - Used by Kubernetes to report back the resource status

## 3. kubectl ([Documentation](https://kubectl.docs.kubernetes.io/))

`kubectl` is a tool to interact with the Kubernetes API server from the command line.

Useful commands:

```sh
# get cluster info (not that useful)
kubectl cluster-info

# show all available resource types / API versions in the cluster
kubectl api-resources
kubectl api-versions

# get the list of resources
# available options:
# -A: get resources from all namespaces
# -n $name: get resources from namespace $name
# -o wide: show more details
# -w: watch for changes
# -l foo=bar: list resources which have label foo with value bar
# for more options run kubectl get --help
# replace $kind with the resource type, e.g. pod
kubectl get $kind

# get a single resource
# available options:
# -o yaml: show detailed yaml resource definition
kubectl get $kind $name

# get more details about a resource
kubectl describe $kind $name

# show resource fields documentation
# available options:
# --recursive: show the full field hierarchy
kubectl explain $kind[.$field[.$subfield[...]]]

# show kubectl config
kubectl config view
```

### Exercise 1

Explore the cluster using the commands above. Specifically, look for:
- Kubernetes version
- available resource types and API versions
- how many nodes, node details (Linux distro, CPU, memory etc.)
- namespaces, pods
- what fields does a `pod`'s `spec` field have?

### Exercise 1 Solution

```sh
# control plane components versions
kubectl -n kube-system describe pod kube-apiserver-jsl-cp-0 | grep -i image
kubectl -n kube-system describe pod kube-controller-manager-jsl-cp-0 | grep -i image
kubectl -n kube-system describe pod kube-scheduler-jsl-cp-0 | grep -i image
kubectl -n kube-system describe pod etcd-jsl-cp-0 | grep -i image

# kubelet versions
kubectl get nodes -o wide

# resources and api versions
kubectl api-resources
kubectl api-versions

# node details
kubectl describe node jsl-cp-0

kubectl get namespaces
kubectl get pods -A -o wide

kubectl explain pod.spec [--recursive]
```

## 4.1. YAML overview

```yaml
key: value
list:
  - 'item 1'
  - "item 2"
altIndentList:
- foo
- bar
number: 123
bool: true
listOfObjects:
  - id: 1
    name: foo
  - id: 2
    name: bar
multilineString: |
  a very very very
  very very very
  long multiline string
```

## 4.2. Namespaces

> Kubernetes supports multiple virtual clusters backed by the same physical cluster. These virtual clusters are called namespaces. - [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)

- provide a scope for names: names of resources need to be unique within a namespace, but not across namespaces
- Kubernetes starts with four initial namespaces: `default`, `kube-system`, `kube-public`, `kube-node-lease`
- if no namespace is set in the `kubectl` config, `default` is used
- some resource types are not namespaced (which is an obvious one? how can we find out which resources are namespaced and which are not?)
- they are not a security boundary by default (workloads in one namespace can access services in another)

## 4.3. Pods

> Pods are the smallest deployable units of computing that can be created and managed in Kubernetes.

> A Pod (as in a pod of whales or pea pod) is **a group of one or more containers, with shared storage and network resources**, and a specification for how to run the containers. A Pod's contents are always co-located and co-scheduled, and run in a shared context. A Pod models an application-specific "logical host" - it contains one or more application containers which are relatively tightly coupled. In non-cloud contexts, applications executed on the same physical or virtual machine are analogous to cloud applications executed on the same logical host. - [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/)

For practical reasons we'll use one container per pod, but be aware that a pod can contain more than one containers.

### Exercise 2

- Create a simple pod named `webserver` with one container using the `nginx:1.28.0` image.

- Check that it's running, get its IP address and use `wget` to test that it's working.

- Check the pod logs.

Hints:
- to output a simple pod blueprint
  ```sh
  kubectl run $podName --image $imageSpec --dry-run=client -o yaml
  ```
  (see `kubectl run --help`)
- to redirect the output of a command to a file use
  ```sh
  command >filename
  ```
- to send resources from a YAML file to Kubernetes use
  ```sh
  kubectl apply -f filename.yaml
  ```
- to get the pod IP use
  ```sh
  kubectl get pods -o wide
  ```
- to get the pod logs use
  ```sh
  kubectl logs $podName
  # follow logs
  kubectl logs -f $podName
  ```
- to output the `wget` response use
  ```sh
  # run this from a cluster node
  wget -qO- http://$ipAddress
  ```

### Exercise 2 Solution

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: webserver
  name: webserver
spec:
  containers:
  - image: nginx:1.23.4
    name: nginx
```

### Exercise 3

Create a new pod named `debug` using the `busybox:1.36.0` image, exec into it, and use `wget` to access the `nginx` pod.

Hints:
- use `command: ["sleep", "86400"]` to keep the pod running (see `kubectl explain pod.spec.containers.command`)
- to delete a pod use
  ```sh
  kubectl delete pod $podName --grace-period 1
  ```
- to exec (run a shell) into a pod use
  ```sh
  kubectl exec -it $podName -- /bin/sh
  ```
- to see a diff between Kubernetes state and an YAML file use
  ```sh
  kubectl diff -f filename.yaml [ | cdiff]
  ```

### Exercise 3 Solution

```yaml
# append to pod.yaml
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: debug
  name: debug
spec:
  containers:
  - image: busybox:1.36.0
    command: ["sleep", "86400"]
    name: busybox
```

```sh
kubectl exec debug -- wget -qO- http://$webserverIP
```

## 4.4. Events

> Kubernetes events are objects that show you what is happening inside a cluster, such as what decisions were made by the scheduler or why some pods were evicted from the node. All core components and extensions (operators) may create events through the API Server. - [Understanding Kubernetes cluster events](https://banzaicloud.com/blog/k8s-cluster-logging/)

- are only kept for 1h by default
- `kubectl get events [-A] [-w] [--watch-only]`
- related events are shown at the end of `kubectl describe` output

## 4.5. Services

> An abstract way to expose an application running on a set of Pods as a network service.

> With Kubernetes you don't need to modify your application to use an unfamiliar service discovery mechanism. Kubernetes gives Pods their own IP addresses and a single DNS name for a set of Pods, and can load-balance across them. - [Service](https://kubernetes.io/docs/concepts/services-networking/service/)

### Exercise 4

- Expose the `webserver` pod using a service named `webserver-svc`.

- Get its IP address and test that it works using `wget` (from the `debug` pod).

- Create another `webserver-2` pod similar to `webserver` (use the same labels, so that it's selected by the service) and test that the service load-balances the requests by looking at the pods logs.

- Test that you can access the service using its DNS name (`webserver-svc` / `webserver-svc.$username`).

- Make the service `NodePort` and try to access it from your computer: `http://65.109.43.242.nip.io:$nodePort/`.

Hints:
- to get the blueprint for a service use
  ```sh
  kubectl expose pod $podName --port $port --dry-run=client -o yaml
  ```
- to check the endpoints the service balances the traffic to use
  ```sh
  kubectl describe service $serviceName
  ```
- to show the logs of multiple pods that match a label use [`kail`](https://github.com/boz/kail)
  ```sh
  kail -l $label=$value -n $username
  ```

### Exercise 4 Solution

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: webserver-svc
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: webserver
```

```yaml
# append to pod.yaml
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: webserver
  name: webserver-2
spec:
  containers:
  - image: nginx:1.23.4
    name: webserver-2
```

```sh
kubectl describe service webserver-svc

kubectl exec debug -- wget -qO- http://webserver-svc
```
