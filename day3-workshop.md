# Introduction to Kubernetes / Deutsche Bank / 26, 27, 29 May 2025

## Day 3: Advanced Concepts and Best Practices

### Schedule - Day 3

- 10:00 - 10:30: Days 1-2 Recap
- 10:30 - 12:00: Storage and WordPress Example
- 12:00 - 12:15: Break
- 12:15 - 13:30: Resource Management
- 13:30 - 14:30: Lunch Break
- 14:30 - 16:00: Node Management and Scheduling
- 16:00 - 16:15: Break
- 16:15 - 18:00: Production Best Practices and Wrap-up

### Exercise 11

- Download `https://gist.githubusercontent.com/lbogdan/a3c5a7e2d85a89b10a1b469cc950e017/raw/353d1da4ab3f5c08e6ebc84750833da6b5bceb29/mysql.yaml` and `https://gist.github.com/lbogdan/a3c5a7e2d85a89b10a1b469cc950e017/raw/353d1da4ab3f5c08e6ebc84750833da6b5bceb29/wordpress.yaml` to your home folder.

- Apply the resources in the two files to the cluster.

- Change the ingress you created above to point to the `wordpress` service and configure Wordpress.

- Check what happens if you delete and recreate the deployments.

# 5. Advanced Concepts

## 5.1. Resources Requests And Limits

```yaml
# pod.yaml
# [...]
spec:
  containers:
    - name: nginx
      image: nginx:1.23.2
      resources:
        requests:
          memory: 64Mi
          cpu: 250m
        limits:
          memory: 256Mi
          cpu: 1
```

> If a Container exceeds its memory limit, it might be terminated. If it is restartable, the kubelet will restart it, as with any other type of runtime failure.

> If a Container exceeds its memory request, it is likely that its Pod will be evicted whenever the node runs out of memory.

> A Container might or might not be allowed to exceed its CPU limit for extended periods of time. However, it will not be killed for excessive CPU usage. - [How Pods with resource limits are run](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#how-pods-with-resource-limits-are-run)


### Exercise 12

Create a deployment using the `gcr.io/kuar-demo/kuard-amd64:blue` image, expose it using a service (the server running in the image is listening on port `8080`), and point your ingress to it.

- Check what happens when you set requests larger than node allocatable resources (> 3800m CPU, > 7.5Gi memory);

- Lower the requests so that the pod can start;

- Go to the `kuard` UI -> KeyGen Workload and enable it; check what happens to the pod used CPU (`watch kubectl top pod`);

- Set the CPU limit to `100m` and repeat last step; remove the CPU limit

- Go to the `kuard` UI -> Memory; allocate memory until the pod restarts (you see the lightning icon); check why it restarted (describe the pod and the node it runs on);

- Set the memory limit to `800Mi` and repeat last step;

- Set `ephemeral-storage` limit to 100Mi; exec into the pod, create a 200MB file (`dd if=/dev/zero of=/tmp/test.bin bs=1M count=200`); wait until pod is terminated; check why it was terminated (describe pod).

## 5.2. Taints, Tolerations, nodeSelector, (Anti-)Affinity

> Node affinity, is a property of Pods that attracts them to a set of nodes (either as a preference or a hard requirement). Taints are the opposite -- they allow a node to repel a set of pods.

> Tolerations are applied to pods, and allow (but do not require) the pods to schedule onto nodes with matching taints.

> Taints and tolerations work together to ensure that pods are not scheduled onto inappropriate nodes. One or more taints are applied to a node; this marks that the node should not accept any pods that do not tolerate the taints. - [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)

```yaml
# node.yaml
# [...]
spec:
  taints:
  - key: key
    value: value # optional
    effect: NoSchedule
```

```yaml
# pod.yaml
# [...]
spec:
  tolerations:
  - key: key
    operator: Equal # Exists, if value is not set
    value: value # optional, if set in the taint
    effect: NoSchedule
#or tolerate all taints
  - operator: Exists
    effect: NoSchedule
```

> You can constrain a Pod to only be able to run on particular Node(s), or to prefer to run on particular nodes. There are several ways to do this, and the recommended approaches all use label selectors to make the selection. [...] there are some circumstances where you may want more control on a node where a pod lands, for example to ensure that a pod ends up on a machine with an SSD attached to it, or to co-locate pods from two different services that communicate a lot into the same availability zone. - [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)

```yaml
# pod.yaml
# [...]
spec:
  nodeSelector:
    label: value
```

> Node affinity is conceptually similar to nodeSelector -- it allows you to constrain which nodes your pod is eligible to be scheduled on, based on labels on the node. - [Affinity and anti-affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity)

> Inter-pod affinity and anti-affinity allow you to constrain which nodes your pod is eligible to be scheduled based on labels on pods that are already running on the node rather than based on labels on nodes. - [Inter-pod affinity and anti-affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity)

### Exercise 13

- Use `nodeSelector` to schedule a pod to the control-plane node `jsl-cp-0` (`kubernetes.io/hostname: jsl-cp-0`); check if it's running; check why it's not running;

- Check the control-plane node taint (`kubectl get no jsl-cp-0 -o yaml`); add a toleration for the control-plane taint:

  ```yaml
  tolerations: # under spec
  - key: node-role.kubernetes.io/control-plane
    effect: NoSchedule
    operator: Exists
  ```

- Comment out the toleration, replace `nodeSelector` with a preferred (vs. required) node affinity; check where the pod is running;

  ```yaml
  affinity: # under spec
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - jsl-cp-0
  ```

- Uncomment the toleration; check where the pod is running.

## 5.3. Security Context

> To specify security settings for a Pod, include the securityContext field in the Pod specification. The securityContext field is a PodSecurityContext object. The security settings that you specify for a Pod apply to all Containers in the Pod. - [Set the security context for a Pod](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)

```yaml
# pod.yaml
# [...]
spec:
  securityContext:
    runAsUser: 1000 # uid
    runAsGroup: 1000 # gid
```

### Exercise 14

- Create a deployment using the `busybox:1.36.0` image; exec into the pod, check what user the container is running as (`id`, `ps a`); check what users are defined in the image (`cat /etc/passwd`);

- Add a `securityContext` to the pod to run its container as the `nobody (65534)` user; exec into the pod, check what user the container is running as (`id`, `ps a`); create a file (`touch /tmp/testfile`) and check what user and group is it created with (`ls -al /tmp/testfile`).

## 5.4. RBAC

ServiceAccount <-> RoleBinding <-> Role

```yaml
# rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: view-pod
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-and-pod-logs-reader
rules:
- apiGroups: [""]
  resources: ["pods"] # "pods/log"
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: view-pod-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-and-pod-logs-reader
subjects:
- kind: ServiceAccount
  name: view-pod
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-rbac
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-rbac
  template:
    metadata:
      labels:
        app: test-rbac
    spec:
      containers:
      - image: bitnami/kubectl:1.23.12
        name: kubectl
        command: ["sleep", "86400"]
      serviceAccountName: view-pod
```

### Exercise 15

- Create the above resources; exec into the pod, list the pods; try to list services; try to show the logs of a pod;

- Add the missing resource in the error above, when trying to show the logs, to the `Role`'s' `resources`; try to show the logs again;

- Change the deployment image to `curlimages/curl:7.86.0`; exec into the pod, do a raw API request to the kubernetes API server (⚠️ replace `$username` with your username ⚠️):

  ```sh
  curl -i --cacert /run/secrets/kubernetes.io/serviceaccount/ca.crt -H "Authorization: Bearer $(cat /run/secrets/kubernetes.io/serviceaccount/token)" https://10.96.0.1/api/v1/namespaces/$username/pods
  ```

## 5.5. Liveness & Readiness Probes

> The kubelet uses liveness probes to know when to restart a container. For example, liveness probes could catch a deadlock, where an application is running, but unable to make progress. Restarting a container in such a state can help to make the application more available despite bugs.

> The kubelet uses readiness probes to know when a container is ready to start accepting traffic. A Pod is considered ready when all of its containers are ready. One use of this signal is to control which Pods are used as backends for Services. When a Pod is not ready, it is removed from Service load balancers. - [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)

```yaml
# pod.yaml
# [...]
spec:
  containers:
    - image: some-image
      name: some-name
      livenessProbe:
        exec:
          command:
          - cat
          - /tmp/healthy
        # or
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 5
        failureThreshold: 3
```

### Exercise 16

- Create a deployment using the `kuard` image;

- Add a http `livenessProbe` to `/healthy`; go to the `kuard` UI -> Liveness Probe, set it to fail; check what happens to the pod (describe pod, watch events - `kubectl get events --watch-only`):

  ```yaml
  livenessProbe: # under containers
  httpGet:
    path: /healthy
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
  ```

- Set the deployment replicas to 2;

- Add a http `readinessProbe` to `/ready`; go to the `kuard` UI -> Readiness Probe; watch the pods (`kubectl get po -w`) and events; set the probe to fail for next 5 calls; refresh the UI and check which pod is serving the requests;

  ```yaml
  readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 12
  failureThreshold: 3
  ```

## 5.6. Network Policies

> A network policy is a specification of how groups of pods are allowed to communicate with each other and other network endpoints. NetworkPolicy resources use labels to select pods and define rules which specify what traffic is allowed to the selected pods. - [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

```yaml
# networkpolicy.yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: frontend-mysql-allow
spec:
  podSelector:
    matchLabels:
      app: wordpress
      tier: mysql
  ingress:
  - from:
      - podSelector:
          matchLabels:
            app: wordpress
            tier: frontend
```

### Exercise 17

- Create a mysql client pod:

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    labels:
      app: mysql-client
    name: mysql-client
  spec:
    containers:
    - image: ghcr.io/lbogdan/mysql:5.6
      name: mysql-client
      command: ["sleep", "86400"]
      env:
        MYSQL_HOST: wordpress-mysql
        MYSQL_USER: root
        MYSQL_PASSWD: changeme
  ```

- Exec into it, try to connect to the WordPress mysql server: `mysql --password=$MYSQL_PASSWD wordpress`; check connection details (`\s`)

- Create the above network policy;

- Exec into the `mysql-client` pod again, try to connect to mysql; check that the WordPress is still working.

### Exercise 18 - kapp

Deploy WordPress using `kapp`.

- Make sure you have `mysql.yaml` and `wordpress.yaml`, and only them, in a folder; also add the network policy you created in the previous exercise to `networkpolicy.yaml`;

- Deploy from the folder:

  ```sh
  kapp deploy -a wordpress-app -f .
  ```

- Show all deployed (and related) resources:

  ```sh
  kapp inspect -a wordpress-app -t
  ```

- Comment out the deployment in `wordpress.yaml` and check what happens if you deploy again.

### Exercise 19 - sealed secrets

- Define a secret resource in `secret.yaml` using `stringData`; give it a name that doesn't exist;

- Encrypt the secret to a sealed secret:

  ```sh
  kubeseal -o yaml <secret.yaml >sealedsecret.yaml
  ```

- Inspect and apply `sealedsecret.yaml`; describe it;

- Check the generated secret in the cluster; check that it contains the right data;

- Use the secret as an environment variable inside a test pod; exec into the pod and check the environment variable is properly set.

### Exercise 20 - helm

Deploy WordPress from the [Bitnami Helm chart](https://github.com/bitnami/charts/tree/main/bitnami/wordpress/).

- Remove old WordPress deployment:

  ```sh
  kapp delete -a wordpress-app
  ```

- Add the Bitnami repository;

  ```sh
  helm repo add bitnami https://charts.bitnami.com/bitnami
  ```

- First, install with default values; figure out why it doesn't work;

  ```sh
  helm install wordpress bitnami/wordpress
  ```

- Uninstall;

  ```sh
  helm uninstall wordpress
  # this PVC is created by the StatefulSet, so it's not deleted by helm
  kubectl delete pvc data-wordpress-mariadb-0
  ```

- Create a `values.yaml` files that configures:
  - passwords for WordPress user, MariaDB `root`, MariaDB user
  - storage class and size of PVCs for WordPress and MariaDB
  - service type to `ClusterIP`
  - ingress for `$username.65.109.43.242.nip.io`

  See the chart docs for the configuration parameters.

- Install using the `values.yaml` file:

  ```sh
  helm install wordpress bitnami/wordpress --values values.yaml
  ```

Useful commands:

```sh
# render the chart to stdout
helm template wordpress bitnami/wordpress --values values.yaml | less

# show release status
helm status wordpress

# show all resources associated with the release
helm get manifest wordpress | kubectl get -f -
```

# Additional Resources

## Additional Kubernetes Components

...installed in the workshop cluster to handle networking, ingresses, storage, GitOps etc.:

- [calico](https://docs.projectcalico.org/getting-started/kubernetes/self-managed-onprem/onpremises) - Kubernetes network plugin

- [metrics-server](https://github.com/kubernetes-sigs/metrics-server) - Kubernetes CPU & memory metrics collector

- [nginx Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
  - [nginx Ingress Controller Sticky Sessions](https://kubernetes.github.io/ingress-nginx/examples/affinity/cookie/)

- [cert-manager](https://cert-manager.io/docs/) - SSL/TLS certificate manager; automatically generates [Let's Encrypt](https://letsencrypt.org/) SSL/TLS certificates for ingresses

- [Rook Ceph](https://rook.io/docs/rook/v1.7/) - Kubernetes storage operator / controller

- [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) - GitOps tool; syncs Kubernetes resources between a git repository and the cluster

- [Kubernetes Dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)

## Kubernetes User Tools

...that I used throughout the workshop:

- [Visual Studio Code](https://code.visualstudio.com/)

- [VSCode Remote SSH extension](https://code.visualstudio.com/docs/remote/ssh)

- [VSCode Kubernetes extension](https://code.visualstudio.com/docs/azure/kubernetes#_install-the-kubernetes-extension)

- [VSCode Kubernetes Support extension](https://marketplace.visualstudio.com/items?itemName=ipedrazas.kubernetes-snippets) - Kubernetes resources YAML snippets / templates

- [kubectl](https://kubectl.docs.kubernetes.io/)

- [kail](https://github.com/boz/kail) - Stream logs from all matched pods

- [k9s](https://k9scli.io/) - Kubernetes terminal-based client UI

- [Lens](https://k8slens.dev/) - Kubernetes graphical client UI

- [octant](https://octant.dev/) - Another Kubernetes graphical client UI

- [kapp](https://carvel.dev/kapp/) - Deploy and view groups of Kubernetes resources as "applications"

- [kustomize](https://kustomize.io/) - Kubernetes native configuration management

- [helm](https://helm.sh/) - The package manager for Kubernetes

- [kubeval](https://kubeval.instrumenta.dev/) - Validate Kubernetes YAML resource files

- [Excalidraw](http://excalidraw.com/) - Diagram sketching app

## Other Resources

...in no particular order:

- [etcd](https://etcd.io/) - Distributed, reliable key-value store used by Kubernetes to store its state

- [CNCF Cloud Native Interactive Landscape](https://landscape.cncf.io/)

- [Kubernetes 1.22 API Reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.22/)

- [▶️ The future of Linux Containers](https://www.youtube.com/watch?v=wW9CAH9nSLs) - At PyCon Solomon Hykes shows docker to the public for the first time

- [📖 Kubernetes Up and Running](https://k8s.vmware.com/kubernetes-up-and-running/)

- [📖 Kubernetes Patterns](https://k8spatterns.io/)

- [NetworkPolicy Editor](https://editor.cilium.io/)

- [▶️ TGI Kubernetes](https://tgik.io/) ([GitHub repo](https://github.com/vmware-tanzu/tgik)) - Weekly live video stream about all things Kubernetes

- [Kubernetes Reddit Community](https://www.reddit.com/r/kubernetes/)

- [OCI - Open Container Image Spec](https://github.com/opencontainers/image-spec)

- Official Kubernetes Certifications:
  - [Certified Kubernetes Administrator (CKA)](https://www.cncf.io/certification/cka/)
  - [Certified Kubernetes Application Developer (CKAD)](https://www.cncf.io/certification/ckad/)
  - [Certified Kubernetes Security Specialist (CKS)](https://www.cncf.io/certification/ckad/)
