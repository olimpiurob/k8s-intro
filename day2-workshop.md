# Introduction to Kubernetes / Deutsche Bank / 26, 27, 29 May 2025

## Day 2: Service Networking and Configuration Management

### Schedule - Day 2

- 10:00 - 10:30: Day 1 Recap
- 10:30 - 12:00: Ingress and External Access
- 12:00 - 12:15: Break
- 12:15 - 13:30: ConfigMaps & Secrets
- 13:30 - 14:30: Lunch Break
- 14:30 - 16:00: Deployments and ReplicaSets
- 16:00 - 16:15: Break
- 16:15 - 18:00: More Deployment Strategies & DaemonSets

## 4.6. Ingress

> An API object that manages external access to the services in a cluster, typically HTTP(S). Ingress may provide load balancing, SSL termination and name-based virtual hosting. - [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

- you must have an [ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) to satisfy an Ingress resource, only creating an Ingress resource has no effect
- we're using [`ingress-nginx`](https://kubernetes.github.io/ingress-nginx/deploy/)

### Exercise 5

Expose the service created in exercise 4 on `http://$username.65.109.43.242.nip.io/`. Test that you can access it from your browser.

```yaml
# ingress.yaml
# ⚠️ replace $username with your username ⚠️
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webserver-ing
spec:
  ingressClassName: nginx
  rules:
  - host: $username.65.109.43.242.nip.io
    http:
      paths:
      - backend:
          service:
            name: webserver-svc
            port:
              number: 80
        path: /
        pathType: Prefix
```

### Exercise 6

Enable SSL/TLS for the ingress created in exercise 5. Test that accessing the `http://` URL redirects to `https://` and inspect the certificate.

> Note: The TLS certificate is created by the [`cert-manager`](https://cert-manager.io/docs/) app which is installed in our cluster, by leveraging [Let's Encrypt](https://letsencrypt.org/getting-started/) free TLS certificate generation service.


> Note: The `cert-manager` app is installed in the cluster and configured to use the `letsencrypt` issuer. You can check the status of the issuer with `kubectl get issuer -n cert-manager`.

> Note: Since there is a limit on the number of certificates that can be created for a domain, please don't delete the ingress resource created in this excercise.

```yaml
# merge with ingress.yaml
# ⚠️ replace $username with your username ⚠️
[...]
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
[...]
spec:
  [...]
  tls:
  - hosts:
    - $username.65.109.43.242.nip.io
    secretName: $username.65.109.43.242.nip.io-tls
```

## 4.7. ConfigMaps & Secrets

> A ConfigMap is an API object used to store non-confidential data in key-value pairs. Pods can consume ConfigMaps as environment variables, command-line arguments, or as configuration files in a volume. A ConfigMap allows you to decouple environment-specific configuration from your container images, so that your applications are easily portable. - [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)

Using a ConfigMap with environment variables (not updated when the ConfigMap changes!):

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-env
data:
  databaseHost: mysql
  databasePort: "3306"

# somepod.yaml
# [...]
spec:
  containers:
  - name: some-name
    image: some-image
    env:
    - name: DATABASE_HOST
      valueFrom:
        configMapKeyRef:
          name: config-env
          key: databaseHost
    - name: DATABASE_PORT
      valueFrom:
        configMapKeyRef:
          name: config-env
          key: databasePort
# or use all key-values from a ConfigMap
    envFrom:
    - configMapRef:
        name: config-env
```

Using a ConfigMap with files (updated):

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-mnt
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
      <title>Welcome to nginx!</title>
    </head>
    <body>
      <h1>Mounted from a ConfigMap!</h1>
    </body>
    </html>

# pod.yaml
# [...]
spec:
  containers:
  - name: nginx
    image: nginx:1.28.0
    volumeMounts:
    - name: config-volume
      mountPath: /usr/share/nginx/html
  volumes:
  - name: config-volume
    configMap:
      name: config-mnt
```

ConfigMaps can also be created from the command line using `kubectl`, see `kubectl create configmap --help`.

> Kubernetes Secrets let you store and manage sensitive information, such as passwords, OAuth tokens, and ssh keys. Storing confidential information in a Secret is safer and more flexible than putting it verbatim in a Pod definition or in a container image. - [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-secret
type: Opaque
data:
  # base64-encoded
  username: cm9vdA==
  password: dG9wc2VjcmV0
# or
stringData:
  username: root
  password: topsecret
```

You can use secrets the same as configmaps, either as environment variables or mount them inside a pod:

```yaml
# somepod.yaml
# [...]
spec:
  containers:
    - name: some-name
      image: some-image
      env:
      - name: DB_USERNAME
        valueFrom:
          secretKeyRef:
            name: database-secret
            key: username
# or use all key-values
      envFrom:
      - secretRef:
          name: database-secret
# or mount
# [...]
    volumeMounts:
    - name: database-volume
      mountPath: /usr/share/nginx/html
  volumes:
  - name: database-volume
    secret:
      secretName: database-secret
```

To `base64` encode / decode a string, use
```sh
# encode
echo -n someString | base64
# decode
echo someBase64EncodedString | base64 -d
```

Secrets can also be created from the command line using `kubectl`, see `kubectl create secret generic --help`.

### Exercise 7

Experiment with configmaps and secrets, using them as both environment variables and mounted inside a pod. Check what happens when they're updated.

### Exercise 7 Solution

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-env
data:
  databaseHost: mysql2
  databasePort: "3306"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-mnt
data:
  databaseHost: mysql2
  databasePort: "3306"
---
apiVersion: v1
kind: Secret
metadata:
  name: database-secret
type: Opaque
data:
  # base64-encoded
  username: cm9vdA==
  password: dG9wc2VjcmV0
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: config-test
  name: config-test
spec:
  containers:
  - image: busybox:1.36.0
    command: ["sleep", "86400"]
    name: busybox
    # env:
    # - name: DATABASE_HOST
    #   valueFrom:
    #     configMapKeyRef:
    #       name: config-env
    #       key: databaseHost
    # - name: DATABASE_PORT
    #   valueFrom:
    #     configMapKeyRef:
    #       name: config-env
    #       key: databasePort
  #   volumeMounts:
  #   - name: config-volume
  #     mountPath: /config
  # volumes:
  # - name: config-volume
  #   configMap:
  #     name: config-mnt
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: database-secret
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: database-secret
          key: password
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-html
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
      <title>Welcome to nginx!</title>
    </head>
    <body>
      <h1>Hello Kubernetes from a ConfigMap!</h1>
    </body>
    </html>
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: webserver
  name: webserver
spec:
  containers:
  - image: nginx:1.28.0
    name: nginx
    volumeMounts:
    - name: config-volume
      mountPath: /usr/share/nginx/html
  volumes:
  - name: config-volume
    configMap:
      name: nginx-html
```

## 4.8. Deployments and ReplicaSets

> A ReplicaSet ensures that a specified number of pod replicas are running at any given time. However, a Deployment is a higher-level concept that manages ReplicaSets and provides declarative updates to Pods along with a lot of other useful features. [...] This actually means that you may never need to manipulate ReplicaSet objects: **use a Deployment** instead, and define your application in the spec section. - [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)

> A Deployment provides declarative updates for Pods and ReplicaSets. You describe a desired state in a Deployment, and the Deployment Controller changes the actual state to the desired state at a controlled rate. - [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webserver
  labels:
    app: webserver
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webserver
  template:
    metadata: $podMetadata
    spec: $podSpec
```

### Exercise 8

- Create a deployment named `demo` with one replica using the `lbogdan/docker-demo:v1` image, expose it using a service named `demo-svc` (the server inside that image listens on port 8080) and the ingress you created above by changing the service it points to.

- Experiment with scaling the deployment up / down (also to 0) (`kubectl scale --replicas 1 deploy/demo`).

- Experiment with changing the image tag to `v2` to see how pods are replaced.

> Note: We're reusing the ingress so that we don't create new TLS certificates, which have a creation rate-limit per domain.

> Note: The [`docker-demo`](https://github.com/ehazlett/docker-demo) app is a simple app with a UI that sends an API request to itself every second and shows the version and the hostname of the container that responded to the request - or an error. It helps visualizing how scaling deployments works.

Useful commands:

```sh
# output a simple deployment blueprint
kubectl create deployment $name --image $image --dry-run=client -o yaml

# set a deployment replicas
kubectl scale --replicas $number deployment/$name

# show a deployment rollout status
kubectl rollout status deployment/$name

# restart a deployment
kubectl rollout restart deployment/$name
```

### Exercise 8 Solution

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo
  labels:
    app: demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - image: lbogdan/docker-demo:v1
        name: demo
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: demo
  name: demo-svc
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: demo
```

### Exercise 9

[Blue-Green Deployments](https://codefresh.io/learn/software-deployment/what-is-blue-green-deployment/)

- Delete the previous `demo` deployment;

- Create a similar deployment, named `demo-blue`, with the `lbogdan/docker-demo:v1` image and an extra `color: blue` label; add the label to the service's selector;

- (Deploy new version) Create a similar deployment, named `demo-green`, with the `lbogdan/docker-demo:v2` image and the `color` label set to `green`; check which version is serving traffic;

- (Switch to new version) Change the `color` to `green` in the service's selector; check which version is serving traffic;

- (rollback) Change the `color` back to `blue` in the service's selector; check which version is serving traffic;

- Switch back to `v2` (`green`), delete `demo-blue` deployment; check the app is still serving traffic;

- Cleanup.

## 4.9. DaemonSets & StatefulSets

> A DaemonSet ensures that all (or some) Nodes run a copy of a Pod. As nodes are added to the cluster, Pods are added to them. As nodes are removed from the cluster, those Pods are garbage collected. Deleting a DaemonSet will clean up the Pods it created. - [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)

> StatefulSet is the workload API object used to manage stateful applications. Manages the deployment and scaling of a set of Pods, and provides guarantees about the ordering and uniqueness of these Pods. Like a Deployment, a StatefulSet manages Pods that are based on an identical container spec. Unlike a Deployment, a StatefulSet maintains a sticky identity for each of their Pods. These pods are created from the same spec, but are not interchangeable: each has a persistent identifier that it maintains across any rescheduling. - [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)

## 4.10. Jobs & CronJobs

> A Job creates one or more Pods and ensures that a specified number of them successfully terminate. As pods successfully complete, the Job tracks the successful completions. When a specified number of successful completions is reached, the task (ie, Job) is complete. Deleting a Job will clean up the Pods it created. - [Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)

> A CronJob creates Jobs on a repeating schedule. One CronJob object is like one line of a crontab (cron table) file. It runs a job periodically on a given schedule, written in Cron format. - [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)

## 4.11. PersistentVolumeClaims & PersistentVolumes

> A PersistentVolume (PV) is a piece of storage in the cluster that has been provisioned by an administrator or **dynamically provisioned using Storage Classes**. PVs [...] have a lifecycle independent of any individual Pod that uses the PV. This API object captures the details of the implementation of the storage, be that NFS, iSCSI, or a cloud-provider-specific storage system.

> A PersistentVolumeClaim (PVC) is a request for storage by a user. It is similar to a Pod. Pods consume node resources and PVCs consume PV resources. Pods can request specific levels of resources (CPU and Memory). Claims can request specific size and access modes (e.g., they can be mounted ReadWriteOnce, ReadOnlyMany or ReadWriteMany. - [Persistent Volumes] (https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  storageClassName: ceph-block
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: test
  name: test
spec:
  containers:
  - image: busybox:1.36.0
    command: ["sleep", "86400"]
    name: test
    volumeMounts:
    - name: storage-volume
      mountPath: /storage
  volumes:
  - name: storage-volume
    persistentVolumeClaim:
      claimName: test-pvc
```

### Exercise 10

- Create the PVC above; check it is bound;

- Create the pod above; exec into it; check the mounts (`mount`); create a file inside the volume (`echo persist! >/storage/data.txt`); check that the file is created (`ls -l /storage; cat /storage/data.txt`);

- Delete the pod and recreate it; exec into it and check the file is still there;

- Cleanup the PVC and the pod.
